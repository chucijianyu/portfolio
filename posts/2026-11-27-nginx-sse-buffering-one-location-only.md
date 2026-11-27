# 两个 SSE 端点，只有一个关掉了 nginx 缓冲

研报解读页是"边生成边显示"的设计，研报生成页是 Agent 事件逐条推送。两个页面都挂 SSE，都经同一个 nginx，都在同一台后端上跑。区别只在一处：其中一个的后端响应里带了三行头，另一个只写了 `media_type`。

一开始我以为这是前端的问题，直到把两边 `StreamingResponse` 摊开对照。

## 十六个 location，特批的只有一条

`nginx/nginx.conf` 里一共 16 个 `location`。把带超时的挑出来看：

- `location /`（前端静态）：`proxy_read_timeout 30s`
- `location /api/`：`proxy_read_timeout 300s`、`proxy_send_timeout 300s`、`proxy_connect_timeout 75s`，注释写着"LLM推理可能较慢，设置较长超时"，**没有** `proxy_buffering`
- `location /api/v1/reports/agent-events/`：`proxy_buffering off`、`proxy_cache off`、`proxy_read_timeout 3600s`、`tcp_nodelay on`
- `location /mcp`：同样 `off` + 3600s；`location /ws/`：86400s

nginx 的普通前缀 location 是"最长前缀胜出"，不做正则回退。所以 `POST /api/v1/reports/interpret/stream` 只会命中 `/api/` —— 没有任何一条规则为它开过特例。它落在 300 秒的桶里，并且吃默认的缓冲。

顺带排掉一个疑问：`/api/` 会不会还有第二层反代？不会。前端容器自带的 `frontend/nginx.conf` 只做静态资源与 SPA 回退（`try_files $uri $uri/ /index.html`），根本没有 `/api/` 转发；`docker-compose.yml` 里主 nginx 是唯一对外入口（`8088:80`），backend 只 `expose: 8000`。所以上面那张表就是全部，超时与缓冲只在这一层发生。

## 后端两个 StreamingResponse 长得不一样

`api/reports.py:307`，解读的流式端点：

```python
return StreamingResponse(
    event_generator(),
    media_type="text/event-stream"
)
```

`api/reports.py:493`，Agent 事件的流式端点：

```python
return StreamingResponse(
    event_generator(),
    media_type="text/event-stream",
    headers={
        "Cache-Control": "no-cache",
        "Connection": "keep-alive",
        "X-Accel-Buffering": "no",  # 禁用nginx缓冲，确保实时推送
    }
)
```

后一个头是写给 nginx 看的：上游响应自带 `X-Accel-Buffering: no` 时，nginx 对这条响应关闭缓冲，不需要改任何配置。也就是说 agent-events 有两道保险 —— 专用 location 一道，响应头一道；interpret/stream 两道都没有。

这不是"配置漏了"，是有人踩过坑之后只补了一个端点。注释里的"确保实时推送"说明当时确实观察到过不实时。

## 缓冲对 SSE 到底做了什么

nginx 默认 `proxy_buffering on`：从上游读到的数据先压在 proxy buffer 里（默认 4k/8k 量级，另有 `proxy_busy_buffers_size` 控制边收边发的额度），攒够或上游结束时才往客户端推。SSE 的一帧 `data: {...}\n\n` 只有几十到几百字节，远填不满一个 buffer，于是前端 `reader.read()` 拿到的不是"一个 token 一次"，而是攒够之后的一大块。

前端那个按 `\n\n` 切事件的循环写得是对的（`ReportAnalysis.jsx:247`），它不会把帧切错，只是"逐字"变成"整段蹦出来"。

这里我不装作已经实测过：容器没起，上面是从配置与默认值推的。验法是一条命令 —— `curl -N` 打这个端点，看首帧到达时间与后续帧的间隔；如果和 `agent-events` 的节奏一致，说明 nginx 这层不是瓶颈，我这条推断就作废。

## 300 秒这个数比看起来更紧

后端的重试参数是 `LLM_TIMEOUT=120`、`LLM_RETRY_MAX_ATTEMPTS=3`、退避 0.5 与 1.0 秒，单次调用最坏 `3×120+1.5 = 361.5` 秒。nginx 在 `/api/` 上只等 300 秒。

也就是说：后端还在第三次重试的时候，nginx 已经先收摊了，返回 504 的 HTML。前端 `response.ok` 为 false，直接抛错走错误分支（`ReportAnalysis.jsx:235`）—— 用户看到"生成失败"，而后端那次调用并没有失败，只是没人接结果了。

更长的是全库重建。`POST /knowledge/rebuild`（`api/knowledge.py:60`）是同步 `await`，里面逐篇 `process_document`，每篇还带一次摘要 LLM 调用。文档数一多，300 秒没有任何可能够用。前端这条请求的 axios 也没设 timeout（全站搜 `timeout` 零命中），它能等多久完全取决于 nginx 什么时候放弃。放弃之后 `handleRebuild` 的 `catch` 弹"索引重建失败"，可重建本身不幂等 —— 用户看到失败再点一次，是全库重跑，不是续跑。

## 怎么改

1. SSE 端点一律在响应头带 `X-Accel-Buffering: no`。这是后端自带的声明，换反代、换云厂商都不用再改对方的配置。
2. 给 interpret 单开一条 location，与 agent-events 同待遇：`proxy_buffering off; proxy_read_timeout 3600s;`。
3. 不要把所有 API 塞进一个 300s 的桶。桶的粒度应该按"这次请求里有没有循环调外部模型"来分，不是按路径前缀分。
4. 写死一条不等式并 review 它：**反代超时 > 后端单次请求的理论上界**。上界是 `LLM_TIMEOUT × 重试次数 + 退避`，361.5 秒的请求配 300 秒的网关，等于把重试逻辑白写了。

## 一句话

超时不是一个数字，是浏览器、反代、应用三层里**最小**的那个。只把最大的那层配大，等于没配；而只给一个端点关缓冲，等于承认这层存在过、却没把它当规则。

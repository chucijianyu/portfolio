# 接口失败时页面说「暂无文档」：前端对失败的三种姿态

后端容器没起，我打开知识库页面，看到的是「暂无文档」加上一句「上传研报文件开始构建知识库」。右上角的系统状态写着「LLM 引擎 在线」。

两句都不对：库里有一百多份文档，后端一个端口都不通。这不是文案写得不好的问题，是这一整站对「接口失败」只有三种处理方式，而三种都会让页面返回一个看起来正常的结果。

## 我怎么数的

`frontend/src` 下 22 个 js/jsx 文件，72 处 `axios`、2 处 `fetch`。`catch` 51 个，`finally` 16 个。搜 `timeout` **0 命中**——axios 的默认 timeout 是 0，即不设超时，一个不响应的请求会一直挂着，`finally` 里的 `setLoading(false)` 永远等不到。

全局只有 1 个响应拦截器（`AuthContext.jsx:52`），只做一件事：401 就登出，其余原样 `Promise.reject`。所以 403、429、500、超时，全都落到 51 个各写各的 catch 里。这 51 个里 12 个是静默的：4 个空 `catch {}`，8 个只写一行注释「静默失败」。

## 姿态一：吞掉

`KnowledgeBase.jsx` 前两个加载函数是一行式：

```js
const loadStatus = async () => {
  try { const res = await axios.get('/api/v1/knowledge/status'); setStatus(res.data) } catch {}
}

const loadDocuments = async () => {
  try { const res = await axios.get('/api/v1/reports/list?limit=100'); setDocuments(res.data) } catch {}
}
```

`documents` 保持初始的 `[]`，而渲染分支是 `documents.length === 0 ? '暂无文档' : '无匹配文档'`（:435）。失败和「库里真的一篇都没有」走了同一个分支，副标题还催你上传。

四个历史列表页（`FinancialAnalysis.jsx:80`、`ReportGeneration.jsx:167`、`ReportAnalysis.jsx:115`、`KnowledgeGraph.jsx:86`）是同一写法，`finally` 只管关 loading，`items` 空着交给 `HistoryCardList` 的 `emptyText='暂无历史记录'`。

`Monitoring.jsx:305` 稍微讲究，用 `Promise.allSettled` 拉四条腿，每条 `if (status === 'fulfilled')` 才 setState，外面套一个 `catch {}`。效果是：挂一条，对应面板保留上一次的数据，页面上没有任何标记，而定时器每 3 秒还会再试——看起来像「这个数一直没变」。

大屏更直接。`Dashboard.jsx:183` 的初始 state 是 `{ llm: 'online', db: 'online' }`，`:227` 只在 `/health` 成功时写 online，`:262` 那个 catch 因为包着 `allSettled` 几乎不可能触发。健康检查失败时没有任何分支写 offline，状态就停在初始值——大屏告诉你「在线」。

## 姿态二：喊一声就完事

```js
} catch { addToast('搜索失败', 'error') }
```

`KnowledgeBase.jsx:91`。`searchResults` 没被清空，上一次的结果还在列表里，界面上没有任何「这是旧结果」的标记。用户看着一条红条飘过去，然后继续读那份可能已经过期的检索结果。

上传更典型。`handleFileUpload` 逐个文件循环，每个文件的失败都正确写进了 `uploadQueue`（`status: 'failed'`，`error` 取后端 detail），这部分做对了。但循环结束后无条件来一句：

```js
addToast(`${fileArray.length} 个文件上传完成`, 'success')
```

三个文件全失败，也是绿色的「3 个文件上传完成」。

同一个函数里还漏了一次 `clearInterval`：`progressTimer`（:136）只在 `await` 成功后的 :147 清掉，失败分支没清。它每 300ms 调一次 `setUploadQueue(prev => prev.map(...))`，而 `map` 每次返回新数组，React 每次都会重渲染，直到离开页面。函数末的 `setTimeout(() => setUploadQueue([]), 5000)`，注释写「3 秒后清空」，值是 5000。

## 姿态三：把失败写进结果里

```js
} catch (err) {
  setResult({ error: '分析服务暂时不可用，请检查后端服务是否启动' })
  addToast('分析请求失败', 'error')
}
```

`FinancialAnalysis.jsx:145`，`ReportGeneration.jsx:395` 与 `:467` 同形。`ReportAnalysis.jsx:287` 更进一步，把失败写成一条**助手消息**塞进对话：

```js
updateAssistant({
  content: '抱歉，解读服务暂时不可用，请稍后重试。',
  streaming: false,
})
```

这条消息和模型自己说的话在同一个数组、走同一套渲染、刷新后会作为历史的一部分加载回来。下游任何「有 result 就展示」「有 assistant 消息就是回答」的判断，都会把一句错误提示当成业务结果。

## 唯一一处做对的

`ReportGeneration.jsx` 的 SSE 是全站唯一有第二条路的：

```js
if (!response.ok) {
  throw new Error(`SSE连接失败: ${response.status}`)
}
...
if (done) {
  if (!taskCompleteRef.current) {
    startReviewPolling(id)   // 流结束但没收到 task_complete，降级轮询
  }
}
```

`:288` 的 catch 里 `AbortError` 单独 return（用户主动取消不算错），其余 `console.error` 之后同样降级轮询。它做对三件事：检查了状态码、有明确的完成信号、失败有第二条路。

顺手一个小数：轮询失败累计到 60 次才提示（`:473`），注释写「约 5 分钟」，而 `setInterval` 是 3000ms，60 次是 3 分钟。前 3 分钟失败是完全静默的。

## 为什么全局兜底兜不住

`ErrorBoundary.jsx` 是全站唯一一个错误兜底组件，它自己第 17 到 19 行写着：不捕获事件回调、异步代码、SSR 错误。而 72 处请求全在异步回调里。这个边界对网络失败 100% 无效，只兜渲染期的 TypeError。它的「重试」按钮（:47）也只是 `setState({ hasError: false })`，不重新请求、不重置数据——确定性的渲染异常，点一下立刻再抛一次。

## 所以改什么

不是把 51 个 catch 都换成 toast，是给「失败」一个能被渲染的身份。四条：加载类请求要有 `error` 状态位，空列表文案必须区分「没有数据」和「没取到数据」；全局拦截器除 401 外，至少把 5xx 与超时统一记一次；失败提示不要写进 `result` 或对话数组，改成平级的 `error` 字段；每个 `setInterval` 的 `clearInterval` 挪进 `finally`，每个 `setTimeout` 的注释和毫秒数一起改。

## 落点

判据很简单：把这个接口打回 500，页面上的字会变吗。不变，那这次失败就等于不存在——它只是被翻译成了一句「暂无数据」。

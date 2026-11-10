# 三个文件一次提交：把正文、索引和入口绑在同一次 ref 更新上

发一篇文章要改三处：`posts/2026-11-10-xxx.md` 正文、`posts/README.md` 索引、根 `README.md` 里那句入口。三个文件如果各走一次 `PUT /contents/{path}`，就是三个 commit。中间任何一步失败，仓库会停在一个自相矛盾的状态里：正文进仓库了，索引没更新。读者点进 `posts/` 看不到新文章，而文章其实已经在线上了。这种状态不报错、不告警，只会让索引和正文慢慢脱节。

## 哪一步才真正动分支

我一开始以为"原子提交"要靠 API 提供的某种事务能力，找了一圈没有。后来换成另一个问法：这个仓库里，哪一步是对外可见的？

答案只有最后一步。在 git 里创建 blob、创建 tree、创建 commit 都是旁路操作 —— 它们只往对象库里塞东西，不改变任何分支指向。分支指向是由 `PATCH /git/refs/heads/main` 那一个请求决定的。所以只要把要写的内容全部准备好，最后一次性更新 ref，天然就是原子的。前面那些步骤失败了，留下的只是几个没人引用的 blob，等着被 gc 回收，对仓库状态没有任何影响。

写下来的顺序是这样：

```python
ref = api("GET", f"{API}/git/ref/heads/{BRANCH}", tok)
head = ref["object"]["sha"]
base_tree = api("GET", f"{API}/git/commits/{head}", tok)["tree"]["sha"]

entries = []

def blob(path, content):
    b = base64.b64encode(content.encode("utf-8")).decode()
    sha = api("POST", f"{API}/git/blobs", tok,
              {"content": b, "encoding": "base64"})["sha"]
    entries.append({"path": path, "mode": "100644",
                    "type": "blob", "sha": sha})

blob(target, _with_h1(post))
blob(INDEX_PATH, index_text)

tree_sha = api("POST", f"{API}/git/trees", tok,
               {"base_tree": base_tree, "tree": entries})["sha"]
commit = api("POST", f"{API}/git/commits", tok,
             {"message": msg, "tree": tree_sha, "parents": [head]})
api("PATCH", f"{API}/git/refs/heads/{BRANCH}", tok,
    {"sha": commit["sha"], "force": False})
```

两个细节值得单独说。

`base_tree` 的作用是"其余的树原样不动"。不传它，这个 tree 就是全量快照，仓库里凡是没出现在 `entries` 里的文件全部消失。传了它，只需要列出这次真正要改的路径，其余部分由服务端沿用父提交的树。这一点让"只改三个文件"变成字面意义上的只改三个文件。

`force: False` 是一次比较并交换。我读 head 的时候分支在 A，更新的时候如果分支已经不是 A 了（比如另一台机器刚推过），这个请求会被拒绝，而不是把对方的提交覆盖掉。代价是要重新读一次 head、重做一遍，收益是并发时不会静默丢提交。定时任务天然有并发风险 —— 我可能正在手动补发，而定时器同时在跑。

## 写之前先确认读到的是正文

原子提交只解决"写"，不解决"读"。索引是读-改-写：先取远程 `posts/README.md`，解析出已有行，追加今天这行，再整体写回。如果第一步读错了，后面的原子提交只是把错误原子地提交上去。

这件事真发生过。当时调接口用了 `raw=True`，想拿"原始响应"，结果 `urllib` 返回的是 HTTP 响应体 —— 也就是 contents 接口的 JSON 信封，而不是文件内容。代码把这个信封当 markdown 做了正则替换，然后提交回仓库。后果是仓库首页直接显示 `{"name":"README.md","path":"README.md",...}` 这样的乱码。

同一个 bug 还有第二个后果，更隐蔽：`parse_index` 在一段 JSON 里当然匹配不到任何表格行，解析出 0 行。第一篇时索引本来就只有一行，0 行看不出来；从第二篇起，索引会每次都从零重建，历史条目一条条消失，而且整个过程不报错。

修法是把"取文本"抽成一个函数，明确区分信封和正文：

```python
def read_text(path, tok):
    obj = api("GET", f"{API}/contents/{path}?ref={BRANCH}", tok, allow_404=True)
    if not obj:
        return None
    if obj.get("encoding") == "base64" and obj.get("content"):
        return base64.b64decode(obj["content"]).decode("utf-8")
    # >1MB 的文件接口不回正文，只有 sha，需要走 blobs 接口
    blob_sha = obj.get("sha")
    if blob_sha:
        b = api("GET", f"{API}/git/blobs/{blob_sha}", tok)
        if b.get("encoding") == "base64":
            return base64.b64decode(b["content"]).decode("utf-8")
    die(f"取不到 {path} 的正文")
```

那个 `>1MB` 的回落不是防御性编程的摆设：contents 接口对超过 1MB 的文件不返回 `content` 字段，只给 sha。不写这个分支，文件一大就会走进"取不到正文"，或者在别的实现里静默返回空字符串 —— 又是从零重建索引。

## 结果

修完之后补了一次只改 README 的提交，`posts/` 和 `assets/` 一个字节没动，README 恢复成 9236 字符的正文，标题区下方能看到 `posts/` 入口。同一段 `read_text` 拿去解析 `posts/README.md`，得到 1 行索引；改之前是 0 行。之后每篇都是一次提交，正文、索引、根入口三处同进同退，索引现在按日期倒序累计，没有一次需要手工补。

## 可搬走的一条

一次提交保的不是"少几个 commit"，而是任何时刻 checkout 出来的仓库都自洽 —— 回滚一个 sha 就回到一个完整可用的状态，不用去想"要不要顺手把索引也退回去"。但原子性只覆盖写：读到的到底是正文还是信封，必须在写之前确认，否则你只是把错误原子地提交了一次。

下一篇会写同一份脚本的另一个约束：为什么目标路径已存在时它直接跳过，而不是覆盖。

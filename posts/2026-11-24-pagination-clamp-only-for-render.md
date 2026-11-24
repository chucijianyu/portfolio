# 分页组件把页码夹紧了，可它只管显示，取数的是外面那份

用户管理页有 21 个用户，每页 10 条。翻到第 3 页，只剩 1 条，点删除。删完表格空了，页脚却写着：

```
共 20 条 · 第 11-20 条 · 2/2 页
```

「共 20 条」是对的，「第 11-20 条」也是对的 —— 页脚在说它认为当前在第 2 页。可屏幕上一条数据都没有。往上一页翻一下，数据又出来了。这个「翻一下就好」是最坑的地方：它不像报错，像用户手抖。

## 先看那个组件

`Pagination` 里有两行很负责的代码：

```jsx
// components/Pagination.jsx
const totalPages = Math.max(1, Math.ceil(total / pageSize))

// 边界保护: 页码超出范围时自动纠正
const currentPage = Math.min(Math.max(1, page), totalPages)
```

它拿到了 `page`，判断越界，纠正成 `currentPage`，然后所有按钮、`第 x-y 条`、`x/y 页`都用这个纠正后的值。看起来边界已经处理完了。

问题在于这两行的作用域：`currentPage` 只活在这个组件里。它没有回调告诉调用方「你传进来的 3 我改成了 2」，而调用方法 `<Pagination page={page} onPageChange={setPage} />` 拿回的 `page` 还是 3。**夹紧只对渲染生效，对取数不生效。**

## 六处调用，三种写法

这个项目里 `Pagination` 被用在 6 个地方。逐个对了一遍「夹紧之后谁在用这个页码」。

**一处是对的。** 知识库页自己先把范围算出来，再拿去切片：

```jsx
// pages/KnowledgeBase.jsx
const totalPages = Math.max(1, Math.ceil(filteredDocuments.length / pageSize))
const safeCurrentPage = Math.min(currentPage, totalPages)
const pagedDocuments = filteredDocuments.slice(
  (safeCurrentPage - 1) * pageSize,
  safeCurrentPage * pageSize
)
```

传进去的也是 `safeCurrentPage`（:512）。另外它在搜索词、状态筛选变化时 `setCurrentPage(1)`（:250）。这里是唯一一处「切片用的值」和「渲染用的值」是同一个值。

**一处是把未夹紧的页码直接拿去切片。** 大屏的最近活动：

```jsx
// pages/Dashboard.jsx
const pagedActivities = useMemo(() => {
  const start = (activityPage - 1) * activityPageSize
  return recentActivities.slice(start, start + activityPageSize)
}, [recentActivities, activityPage])
```

`activityPage` 是原始 state，`Pagination` 收的也是它（:533），组件内部夹成 2 渲染页脚，切片却从 `(3-1)*3 = 6` 开始。列表一旦没 7 条，`slice` 返回空数组，页脚照旧报「第 7-9 条」。

**一处是页码越界后没人自愈。** 监控页的两张表是服务端分页，`page` 只经 `onPageChange` 变动，`skip = (page - 1) * pageSize` 直接发给后端。可它有一个 3 秒的定时器：

```jsx
// pages/Monitoring.jsx
refreshTimer.current = setInterval(() => {
  loadData()
  loadTraces(tracePage)
  loadAgents(agentPage)
}, 3000)
```

只要条数变少、`tracePage` 越界，这个定时器会**每 3 秒**带着同一个越界页码去请求一次空页，直到用户手动点一下页码。用户不点，它就一直空着。

**还有一处连走两个口径。** 用户管理页的页码只有一个来源：

```jsx
const params = { skip: (page - 1) * pageSize, limit: pageSize }
```

`page` 只在 `onPageChange` 里变。搜索框改关键词（:253）不重置页码，删除成功后 `fetchUsers()`（:164）也还是用旧页码重取。后端 `list_users` 的做法是：

```python
total = query.count()                                   # 过滤后的总数
users = query.order_by(User.created_at.desc()).offset(skip).limit(limit).all()
```

`total` 是过滤后的真实总数，`offset` 原样执行。所以越界在数据层是**合法**的：HTTP 200，`items` 为空，`total` 正确。前端拿到「总数 20、本页 0 条」，页脚按理该报「第 11-20 条」，可总数只有 20 条 —— 于是出现开头那一幕。

## 那四份复制粘贴的兜底

翻代码时发现另外 4 个页面（财务分析、研报生成、知识图谱、研报解读）用的是同一个历史列表组件，全走服务端分页。它们每一个都写了同一段：

```jsx
if (historyItems.length === 1 && historyPage > 1) {
  loadHistory(historyPage - 1)
} else {
  loadHistory(historyPage)
}
```

4 个页面、4 份一样的代码。这段代码的存在本身就是证据：夹紧的责任被留给了每个调用方，谁都得自己再实现一遍「页码会不会越界」。

而这 4 份兜底只覆盖了「删掉本页最后一条」这一种情况，没覆盖搜索筛选后总数变少，也没覆盖外部刷新后总数变少。同一类缺陷，谁碰上谁补一块，补不全就不会有人知道。

## 修法：把范围计算放到调用方看得见的地方

不要指望组件替调用方修正 state。让 `page` 变成派生值，取数和渲染共用它：

```jsx
const totalPages = Math.max(1, Math.ceil(total / pageSize))
const safePage = Math.min(Math.max(1, page), totalPages)

// 越界时把 state 也拉回来，定时器/重取才不会一直带着旧页码
useEffect(() => {
  if (page !== safePage) setPage(safePage)
}, [page, safePage])

const rows = useMemo(
  () => items.slice((safePage - 1) * pageSize, safePage * pageSize),
  [items, safePage, pageSize]
)

<Pagination page={safePage} pageSize={pageSize} total={total} onPageChange={setPage} />
```

`safePage` 一算出来，切片、页脚、要不要回写 state 三件事就都对齐了。搜索框和筛选条件变化时再补一句 `setPage(1)` —— 这一步知识库页已经做了，用户管理页没做。

## 数字

- `Pagination` 共 6 处调用：1 处自己先夹紧（知识库），1 处拿原始页码切片（大屏），2 处服务端分页且无越界自愈（监控 traces/agents），1 处搜索与删除都不重置页码（用户管理），另 4 个历史列表页共用同一段手写兜底（数了 4 遍）。
- 越界时后端返回的是 `200 + [] + 正确 total`，不是 4xx。所以这类问题在日志里查不到，在接口监控里也查不到 —— 它只出现在屏幕上，而且长得很像正常空态。

## 可搬走的一条

当一个组件需要「修正」传入的 props 才能正确渲染，这个修正是渲染层的，调用方手里那份值没变。只要调用方还要用这个值去取数，范围计算就必须挪到它看得见的地方 —— 否则每一个调用方都会重新发明一遍边界判断，而且每个人只会补自己踩到的那一种。

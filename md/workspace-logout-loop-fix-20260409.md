# workspace 退出登录后循环更新问题修复

## 问题背景

退出登录后重新进入 workspace 页面，会触发 React 报错：

> Maximum update depth exceeded

根因是游客模式下，提交记录 Hook 会在 `useEffect` 中不断调用 `setSubmissions([])`。而现有的 `setSubmissions` 会顺带刷新 `submissionsLastFetch` 时间戳，导致依赖项持续变化，最终形成重复更新。

## 改动文件

1. `programming-frontend/src/store/appStore.ts`
2. `programming-frontend/src/hooks/use-submissions.ts`

## 改动内容

### 1. 为提交缓存增加专用重置动作

在 `appStore` 中新增 `resetSubmissionsCache`：

- 仅清空提交记录相关状态；
- 不再刷新 `submissionsLastFetch`；
- 用于退出登录、游客访问等不需要请求个人提交数据的场景。

### 2. 游客模式改为使用专用重置动作

在 `use-submissions` 中：

- 未登录时不再调用 `setSubmissions([])`；
- 改为调用 `resetSubmissionsCache()`；
- 保留“游客可看题、不可提交”的行为，同时避免依赖链反复变化。

## 改动原因

`setSubmissions` 的设计目标是“写入新的提交列表并记录最新抓取时间”，它适合正常请求成功后的缓存更新，不适合游客态的清空操作。

将“写入提交数据”和“清空提交缓存”拆成两个动作后：

1. 状态语义更清晰；
2. 不会误刷新缓存时间戳；
3. 可以稳定避免 `useEffect` / `useCallback` 依赖反复变化造成的循环更新。

## 预期结果

修复后：

- 退出登录后进入 workspace 不再报 `Maximum update depth exceeded`；
- 游客仍可正常浏览题目；
- 游客不会加载个人提交记录；
- 登录用户原有提交加载逻辑保持不变。

# workspace 退出登录循环更新二次排查记录

## 现象

第一次修复后，`workspace` 在退出登录场景下仍然出现：

> Error: Maximum update depth exceeded

说明循环更新并不只是“清空提交缓存”这一层，还存在依赖链不断变化的问题。

## 深层原因

问题出在 `use-submissions` 的两个点叠加：

1. `loadSubmissions` 的 `useCallback` 依赖了 `submissions` 数组；
2. 游客态重置缓存时，会把 `submissions` 重新置为新的空数组引用。

这样即使数组长度仍然是 `0`，只要引用变了，`loadSubmissions` 的函数身份也会变化；而外层 `useEffect` 又依赖 `loadSubmissions`，最终形成重复执行。

## 本次修复内容

### 1. 让提交缓存重置变成幂等操作

文件：`programming-frontend/src/store/appStore.ts`

- `resetSubmissionsCache` 现在会先检查当前是否已经是“空提交 + 无加载 + 无错误 + 无时间戳”的状态；
- 如果已经是目标状态，则直接返回旧 state，不再触发新的 store 更新。

### 2. 让 `loadSubmissions` 不再依赖 `submissions` 引用

文件：`programming-frontend/src/hooks/use-submissions.ts`

- `loadSubmissions` 改为通过 `useAppStore.getState()` 读取最新缓存；
- 不再把 `submissions` 和 `submissionsLastFetch` 放进 `useCallback` 的依赖中；
- 这样提交数组引用变化时，不会导致 `loadSubmissions` 身份反复变化。

### 3. 拆分游客重置与登录加载两个 effect

文件：`programming-frontend/src/hooks/use-submissions.ts`

- 第一个 effect：只处理未登录时的缓存清空；
- 第二个 effect：只处理已登录时的提交记录拉取；
- 避免“游客态清空逻辑”和“登录态加载逻辑”在同一个 effect 里互相影响。

## 涉及文件

1. `programming-frontend/src/store/appStore.ts`
2. `programming-frontend/src/hooks/use-submissions.ts`

## 预期结果

修复后应满足：

- 退出登录后访问 `workspace` 不再触发无限更新；
- 游客仍然可以浏览题目；
- 游客不会请求个人提交接口；
- 登录用户进入 `workspace` 时仍会正常加载个人提交记录。
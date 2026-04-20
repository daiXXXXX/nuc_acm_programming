# 修复 admin 用户提交历史 Tab 无限请求循环

## 改动背景

使用 admin 账号（userId=156）进入 workspace 的提交历史 Tab 时，浏览器会不断发起
`api/submissions/user/156?limit=100&offset=0` 请求，形成死循环。

## 根因分析

`use-submissions.ts` 中有两处缺陷导致**无提交记录的用户**触发无限请求：

### 缺陷 1：effect 用 `submissions.length === 0` 判断是否需要加载

```typescript
// 旧代码
if (submissions.length === 0 && !submissionsLoading) {
  void loadSubmissions()
}
```

当 API 返回空数组后，`submissions.length` 始终为 0，`submissionsLoading` 在请求完成后置为 false，
effect 再次满足条件 → 再次请求 → 无限循环。

### 缺陷 2：`loadSubmissions` 缓存判断要求数组非空

```typescript
// 旧代码
if (!forceRefresh && isCacheValid(submissionsLastFetch) && cachedSubmissions.length > 0) {
  return cachedSubmissions
}
```

即使 `submissionsLastFetch` 已设置（缓存时间有效），空数组也会穿透缓存检查，每次都发起真实请求。

## 修复内容

### 文件：`programming-frontend/src/hooks/use-submissions.ts`

1. **缓存判断移除 `cachedSubmissions.length > 0` 条件**  
   只要 `submissionsLastFetch` 有效就认为缓存命中，空数组也是合法缓存结果。

2. **effect 改用 `submissionsLastFetch` 判断是否已加载过**  
   将 `submissions.length === 0` 替换为 `!submissionsLastFetch`，一旦完成过一次加载
   （无论结果是否为空），就不再自动触发重复请求。

## 改动文件清单

| 文件 | 改动说明 |
|------|----------|
| `programming-frontend/src/hooks/use-submissions.ts` | 修复缓存判断条件 + effect 触发条件 |

## 预期结果

- 无提交记录的用户（如 admin）进入提交历史 Tab 后只请求一次 API，不再无限循环。
- 有提交记录的用户行为不变，缓存机制正常工作。
- `refresh()` 强制刷新仍然可用。

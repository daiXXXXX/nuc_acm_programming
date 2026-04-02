# Next.js API 路由拆分重构说明

## 1. 改动目标

本次改动的目标，是将 `programming-frontend` 中原本集中写在 `src/app/api/[...path]/route.ts` 里的大量接口逻辑拆分开，按业务域分别放入独立服务文件中。

重构后的目标包括：

- 降低 `route.ts` 文件体积
- 让认证、题目、提交、统计、排行、题解逻辑各自归位
- 把通用方法抽离出来，减少重复代码
- 为后续继续拆分 repository / service / validator 打基础

---

## 2. 改动文件清单

### 2.1 修改文件

- `programming-frontend/src/app/api/[...path]/route.ts`

### 2.2 新增文件

- `programming-frontend/src/server/oj/services/core.ts`
- `programming-frontend/src/server/oj/services/user-service.ts`
- `programming-frontend/src/server/oj/services/auth-service.ts`
- `programming-frontend/src/server/oj/services/problem-service.ts`
- `programming-frontend/src/server/oj/services/submission-service.ts`
- `programming-frontend/src/server/oj/services/stats-service.ts`
- `programming-frontend/src/server/oj/services/ranking-service.ts`
- `programming-frontend/src/server/oj/services/solution-service.ts`

---

## 3. 改动内容说明

### 3.1 `route.ts` 的职责收缩

重构前，`route.ts` 同时承担了以下职责：

- 接收请求
- 路由分发
- 参数解析
- 鉴权
- SQL 查询
- 数据组装
- 返回响应
- 错误处理

这会导致单文件过大，后续维护非常困难。

重构后，`route.ts` 只保留：

- 识别一级业务域
- 将请求分发给对应服务文件
- 统一兜底错误响应

也就是说，`route.ts` 从“巨型业务文件”变成了“轻量调度入口”。

---

### 3.2 抽出通用基础能力：`core.ts`

`core.ts` 主要承接所有服务都会重复使用的基础能力，包括：

- `ApiError`
- `json()`
- `toErrorResponse()`
- `readJsonBody()`
- `parseId()`
- `formatDate()`
- `normalizeEmail()`
- `ensureEmail()`
- `queryWith()`
- `executeWith()`

这样做的好处是：

- 每个服务文件都能复用统一行为
- 错误格式保持一致
- 避免每个模块重复写一套基础工具

---

### 3.3 抽出用户与鉴权相关能力：`user-service.ts`

`user-service.ts` 负责沉淀与“当前用户”和“用户查询”有关的通用逻辑，包括：

- `getUserById()`
- `getUserByUsername()`
- `getUserByEmail()`
- `usernameExists()`
- `emailExists()`
- `toUserDto()`
- `toAuthResponse()`
- `getCurrentUser()`
- `requireCurrentUser()`
- `requireInstructor()`
- `getCurrentUserIdOrZero()`

拆出来之后：

- 认证服务可以复用用户查询
- 题目服务可以复用教师权限判断
- 题解服务可以复用当前用户 ID 获取逻辑

---

### 3.4 认证逻辑拆分到：`auth-service.ts`

该文件集中处理以下接口：

- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `GET /api/auth/me`
- `PUT /api/auth/password`
- `PUT /api/auth/profile`
- `POST /api/auth/avatar`

拆分原因：

- 认证相关逻辑天然属于同一业务域
- 便于单独维护登录、注册、用户资料更新逻辑
- 避免这些逻辑继续堆积在 `route.ts`

---

### 3.5 题目逻辑拆分到：`problem-service.ts`

该文件集中处理：

- 题目列表
- 题目详情
- 题目创建
- 题目更新
- 题目删除
- 题目示例、标签、测试用例组装

拆分后，题目相关的 SQL 和数据结构都集中在一个文件里，后续如果继续抽 repository，会更顺手。

---

### 3.6 提交逻辑拆分到：`submission-service.ts`

该文件集中处理：

- 代码提交
- 提交详情
- 用户提交历史
- 题目提交记录
- 提交结果组装
- 用户统计刷新
- 每日活动刷新

拆分原因：

- “提交 -> 判题 -> 入库 -> 更新统计”是一条完整业务链
- 将其与题目/题解逻辑分离后，职责更清晰

---

### 3.7 统计与排行拆分到独立文件

分别拆分为：

- `stats-service.ts`
- `ranking-service.ts`

这样做的好处是：

- 个人统计逻辑与排行榜逻辑互不干扰
- 查询 SQL 清晰归类
- 后续加缓存时更容易按业务域接入

---

### 3.8 题解逻辑拆分到：`solution-service.ts`

该文件集中处理：

- 题解列表
- 题解详情
- 创建题解
- 更新题解
- 删除题解
- 点赞 / 取消点赞
- 评论列表
- 创建评论
- 删除评论

拆分后，题解相关 SQL、嵌套评论组装逻辑、权限校验都集中在一个业务文件里，维护成本更低。

---

## 4. 本次重构带来的收益

### 4.1 结构更清晰

现在 API 层已经从“一个超大文件”变成了“一个入口 + 多个服务文件”的结构，阅读成本明显降低。

### 4.2 职责更单一

每个服务文件只关注自己的业务域：

- 认证归认证
- 题目归题目
- 提交归提交
- 题解归题解

### 4.3 更利于继续演进

这次拆分之后，后续如果继续重构，可以更自然地往下面几层演进：

- `repositories`
- `validators`
- `dto / mapper`
- `service orchestrator`

### 4.4 降低回归风险

虽然做了拆分，但 `route.ts` 仍然沿用原来的统一入口和一级业务域分发模式，因此对现有前端调用方式影响较小。

---

## 5. 验证结果

本次拆分完成后，已对前端项目执行构建验证，`Next.js build` 通过，说明：

- 新的服务拆分结构可以正常编译
- 现有 API 入口未因拆分而失效
- 模块之间的导入关系没有破坏当前项目构建

---

## 6. 后续建议

后续可以继续在当前拆分基础上做进一步优化：

1. 将每个服务文件中的 SQL 再下沉到 repository 层
2. 将请求参数校验抽离为独立 validator
3. 为返回数据组装增加 mapper/dto 层
4. 为题目、排行、题解等接口补充缓存策略
5. 为每个服务补单元测试或集成测试

---

## 7. 总结

本次改动本质上是一次 **API 入口层重构**。

重构前：

- 一个 `route.ts` 承担几乎全部职责

重构后：

- `route.ts` 只负责调度
- 各业务域逻辑拆分到独立服务文件
- 通用能力沉淀到共享模块

这次拆分没有改变当前接口的使用方式，但显著提升了代码结构清晰度和后续可维护性。

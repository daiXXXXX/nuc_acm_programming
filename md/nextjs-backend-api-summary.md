# Next.js 后端基础 API 改造总结

## 1. 改动背景

本次改动的目标，是在 `programming-frontend` 项目内部先用 Next.js Route Handlers 还原一套**最基础可用**的后端接口能力，用来替代原先依赖 `programming-backend` Go 服务的核心 API。

本阶段只关注“先跑起来”的最小闭环，因此优先恢复以下能力：

- 用户认证
- 题目查询与基础管理
- 代码提交与同步评测
- 用户统计与排行榜
- 题解与评论的基础 CRUD

以下复杂能力**暂不纳入本次实现范围**：

- Redis 队列
- 异步评测
- WebSocket 推送
- AI 查重
- 复杂后台管理功能

---

## 2. 本次新增/修改文件清单

### 已修改文件

1. `.env.example`
2. `next.config.js`
3. `package.json`
4. `package-lock.json`

### 新增文件

1. `src/app/api/[...path]/route.ts`
2. `src/server/oj/db.ts`
3. `src/server/oj/auth.ts`
4. `src/server/oj/evaluator.ts`

---

## 3. 各文件改动说明

### 3.1 `.env.example`

补充了 Next.js 服务端直连数据库和本地执行基础 API 所需的环境变量，包括：

- `NEXT_PUBLIC_API_BASE_URL`
- `DB_HOST`
- `DB_PORT`
- `DB_USER`
- `DB_PASSWORD`
- `DB_NAME`
- `JWT_SECRET`
- `CODE_EXECUTION_TIMEOUT`

**原因：**
新的 API 不再转发到 Go 服务，而是由 Next.js 直接处理，因此前端项目自身需要具备数据库连接和 JWT 配置。

---

### 3.2 `next.config.js`

移除了原本将 `/api/*` 和 `/uploads/*` 转发到 Go 后端的 rewrite 配置。

**原因：**
当前改造的目标，就是让 Next.js 自己接管后端接口。如果保留 rewrite，请求仍会继续打到 Go 服务，无法验证新的 Next.js API 实现。

---

### 3.3 `package.json` / `package-lock.json`

新增了 Next.js 后端实现所需依赖：

- `mysql2`
- `bcryptjs`
- `jsonwebtoken`
- `@types/jsonwebtoken`

**原因：**

- `mysql2`：用于在 Next.js 服务端连接 MySQL
- `bcryptjs`：用于密码哈希与校验
- `jsonwebtoken`：用于生成和校验登录态 JWT

---

### 3.4 `src/server/oj/db.ts`

新增数据库访问层，主要负责：

- 创建 MySQL 连接池
- 提供统一查询方法
- 提供事务封装
- 支持 Next.js 开发模式下的连接复用

**原因：**
将数据库连接逻辑从路由中拆出，避免路由文件过于臃肿，也方便后续继续拆分 repository/service。

---

### 3.5 `src/server/oj/auth.ts`

新增认证辅助模块，主要负责：

- 用户角色类型定义
- JWT 生成与校验
- Bearer Token 读取
- 密码哈希与校验
- 用户名/密码规则校验

**原因：**
将登录态与密码相关逻辑集中管理，保证接口实现尽量与原 Go 后端的认证行为保持一致。

---

### 3.6 `src/server/oj/evaluator.ts`

新增最基础的代码评测器，当前仅保留最简单实现：

- 支持 `JavaScript` / `TypeScript`
- 通过约定的 `processInput(input)` 进行执行
- 返回测试结果数组
- 计算得分
- 生成提交状态

**原因：**
先恢复最小可用评测能力，让“提交代码 -> 返回判题结果”这条主链路可以运行。

**当前限制：**

- 暂不支持 Redis 异步评测
- 暂不支持 C / C++ / Java / Go / Python 的服务端执行
- 仅保留最基础同步评测逻辑

---

### 3.7 `src/app/api/[...path]/route.ts`

新增统一的 catch-all API 路由，集中承接基础接口。

目前已覆盖的接口范围如下。

#### 认证接口

- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `GET /api/auth/me`
- `PUT /api/auth/password`
- `PUT /api/auth/profile`
- `POST /api/auth/avatar`

#### 题目接口

- `GET /api/problems`
- `GET /api/problems/:id`
- `POST /api/problems`
- `PUT /api/problems/:id`
- `DELETE /api/problems/:id`

#### 提交接口

- `POST /api/submissions`
- `GET /api/submissions/:id`
- `GET /api/submissions/user/:userId`
- `GET /api/submissions/problem/:problemId`

#### 统计接口

- `GET /api/stats/user/:userId`
- `GET /api/stats/user/:userId/activity`

#### 排行榜接口

- `GET /api/ranking/total`
- `GET /api/ranking/today`

#### 题解接口

- `GET /api/solutions/problem/:problemId`
- `GET /api/solutions/:id`
- `GET /api/solutions/:id/comments`
- `POST /api/solutions`
- `PUT /api/solutions/:id`
- `DELETE /api/solutions/:id`
- `POST /api/solutions/:id/like`
- `POST /api/solutions/:id/comments`
- `DELETE /api/solutions/comments/:commentId`

**原因：**
本阶段优先追求“接口先恢复可用”，因此采用单入口路由，减少拆分成本，便于快速对齐原 Go 服务的行为。

---

## 4. 本次实现策略

### 4.1 保持前端调用方式基本不变

本次改造尽量兼容现有前端调用方式，使原有 `api.ts` 与 `auth-api.ts` 可以继续通过 `/api/*` 路径工作，尽量减少页面层改动。

### 4.2 优先对齐原 Go 后端的数据结构

返回 JSON 字段命名尽量沿用原接口，例如：

- `accessToken`
- `refreshToken`
- `expiresAt`
- `problemId`
- `userId`
- `testResults`
- `todaySolved`

这样可以减少前端状态管理与展示逻辑的适配成本。

### 4.3 暂时不做复杂模块迁移

本阶段明确跳过：

- Redis 缓存
- Redis 任务队列
- WebSocket 广播
- 抄袭检测
- 班级实验管理接口

原因是这些能力依赖更多基础设施，不适合在“先恢复最简单接口”的阶段同时迁移。

---

## 5. 当前已知限制

### 5.1 仍属于“基础恢复版” API

虽然核心接口已经补齐，但当前实现仍属于第一版，目标偏向“可用”，而不是“完全替代 Go 版本”。

### 5.2 评测器能力有限

当前只保留了最简单的同步执行逻辑，适合本地演示或最小功能验证，不适合作为完整在线判题服务的最终版本。

### 5.3 路由文件体积较大

`src/app/api/[...path]/route.ts` 当前聚合了大量逻辑，后续建议继续拆分为：

- `repositories`
- `services`
- `controllers`
- `validators`

### 5.4 仍需继续做类型与构建校验

本轮实现过程中，已经完成主体代码落地，但后续还建议继续针对以下方面做收尾：

- TypeScript 类型收敛
- Route Handler 参数与数据库执行类型校正
- 实际数据库联调
- 登录、上传头像、提交代码、题解评论等链路回归测试

---

## 6. 后续建议

建议按以下顺序继续迭代：

1. 先修正当前 TypeScript 编译层面的细节问题
2. 用真实 MySQL 数据库联调认证、题目、提交、题解接口
3. 将大体量 `route.ts` 按模块拆分
4. 再逐步补 Redis、异步评测、WebSocket 等能力
5. 最后再考虑 AI 查重与复杂后台管理功能迁移

---

## 7. 总结

本次改动已经在前端项目内部搭建了一套 **Next.js 版基础后端 API 雏形**，完成了从“前端仅代理 Go 服务”向“前端项目自身可承担基础后端职责”的第一步迁移。

其核心价值在于：

- 先恢复最重要的业务接口
- 尽量保持前端调用层不变
- 降低对原 Go 服务的即时依赖
- 为后续继续拆分和增强服务端能力打下基础

本次属于**基础迁移版本**，目标是先让主要接口具备落地形态，方便后续继续迭代。

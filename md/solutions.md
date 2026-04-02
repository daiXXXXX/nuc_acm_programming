# 题解讨论区设计文档

## 概述

题解讨论区是编程实验平台的社区功能，允许用户针对每道题目发布题解、展开讨论。系统支持 Markdown 语法编写题解，并通过 WebSocket 实现评论区的实时消息推送。

## 技术架构

### 后端新增组件

- **WebSocket**: gorilla/websocket v1.5.3（Hub + Client 模式）
- **数据层**: `internal/database/solution.go` — SolutionRepository
- **处理层**: `internal/handlers/solution.go` — SolutionHandler
- **实时通信**: `internal/ws/hub.go` — WebSocket Hub

### 前端新增组件

- **页面**: `/solutions/[problemId]` 题解列表、`/solutions/[problemId]/[solutionId]` 题解详情
- **Hook**: `useWebSocket` — 专用 WebSocket 管理 Hook
- **Markdown**: `marked` + `dompurify` 渲染 & 净化

## 数据库设计

### 迁移文件

`database/migrations/003_add_solutions.sql`

### 表结构

```sql
-- 题解表
CREATE TABLE solutions (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    problem_id  BIGINT NOT NULL,
    user_id     BIGINT NOT NULL,
    title       VARCHAR(200) NOT NULL,
    content     TEXT NOT NULL,
    view_count  INT DEFAULT 0,
    like_count  INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (problem_id) REFERENCES problems(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id)    REFERENCES users(id) ON DELETE CASCADE
);

-- 评论表（支持嵌套）
CREATE TABLE solution_comments (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    solution_id BIGINT NOT NULL,
    user_id     BIGINT NOT NULL,
    parent_id   BIGINT DEFAULT NULL,
    content     TEXT NOT NULL,
    like_count  INT DEFAULT 0,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (solution_id) REFERENCES solutions(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id)     REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (parent_id)   REFERENCES solution_comments(id) ON DELETE CASCADE
);

-- 点赞表（唯一约束防止重复点赞）
CREATE TABLE solution_likes (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    solution_id BIGINT NOT NULL,
    user_id     BIGINT NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY (solution_id, user_id),
    FOREIGN KEY (solution_id) REFERENCES solutions(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id)     REFERENCES users(id) ON DELETE CASCADE
);
```

## API 文档

### 题解相关接口

| 方法   | 路径                                 | 认证 | 说明                    |
| ------ | ------------------------------------ | ---- | ----------------------- |
| GET    | `/api/solutions/problem/:problemId`  | 可选 | 获取题目的题解列表      |
| GET    | `/api/solutions/:id`                 | 可选 | 获取题解详情            |
| POST   | `/api/solutions`                     | 必须 | 创建题解                |
| PUT    | `/api/solutions/:id`                 | 必须 | 更新题解（作者）        |
| DELETE | `/api/solutions/:id`                 | 必须 | 删除题解（作者/管理员） |
| POST   | `/api/solutions/:id/like`            | 必须 | 点赞/取消点赞           |
| GET    | `/api/solutions/:id/comments`        | 可选 | 获取评论列表            |
| POST   | `/api/solutions/:id/comments`        | 必须 | 发表评论                |
| DELETE | `/api/solutions/comments/:commentId` | 必须 | 删除评论                |

### 请求/响应示例

#### 获取题解列表

```
GET /api/solutions/problem/1?order=newest&limit=20&offset=0

Response:
{
  "solutions": [
    {
      "id": 1,
      "problemId": 1,
      "userId": 2,
      "title": "动态规划解法",
      "content": "## 思路\n...",
      "viewCount": 42,
      "likeCount": 5,
      "commentCount": 3,
      "createdAt": "2026-03-12T10:00:00Z",
      "updatedAt": "2026-03-12T10:00:00Z",
      "author": { "id": 2, "username": "alice", "avatar": "" },
      "liked": false
    }
  ],
  "total": 1,
  "limit": 20,
  "offset": 0
}
```

#### 创建题解

```
POST /api/solutions
Authorization: Bearer <token>

{
  "problemId": 1,
  "title": "我的题解",
  "content": "## 解题思路\n使用动态规划..."
}
```

#### 点赞/取消

```
POST /api/solutions/1/like
Authorization: Bearer <token>

Response:
{ "liked": true, "likeCount": 6 }
```

## WebSocket 系统

### 连接方式

```
ws://localhost:8080/api/ws
```

前端通过 WebSocket 子协议 (Sec-WebSocket-Protocol) 传递 JWT Token 进行认证。

### 消息类型

| 类型            | 方向          | 说明           |
| --------------- | ------------- | -------------- |
| `chat`          | 双向          | 用户间聊天讨论 |
| `system_notice` | 服务端→客户端 | 系统通知       |
| `new_comment`   | 服务端→客户端 | 新评论通知     |
| `new_solution`  | 服务端→客户端 | 新题解发布通知 |
| `like_notify`   | 服务端→客户端 | 被点赞通知     |
| `online_count`  | 服务端→客户端 | 在线人数广播   |

### 消息格式

```json
{
  "type": "new_comment",
  "channel": "solution:1",
  "from": { "id": 2, "username": "alice", "avatar": "" },
  "content": { "commentId": 10, "solutionId": 1, "text": "好题解！" },
  "timestamp": "2026-03-12T10:05:00Z"
}
```

### 频道订阅

客户端可订阅频道以接收特定范围的消息：

- `problem:{id}` — 某道题目的新题解通知
- `solution:{id}` — 某篇题解的新评论通知

```json
// 客户端发送
{ "action": "subscribe", "channel": "solution:1" }
{ "action": "unsubscribe", "channel": "solution:1" }
{ "action": "chat", "channel": "solution:1", "content": "写得好！" }
```

### Hub 架构

```
                      ┌──────────┐
                      │   Hub    │
                      │          │
  Client A ──────────►│ register │
  Client B ──────────►│ unregister│
  Client C ──────────►│ broadcast│
                      │ channel  │
                      └──────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         channel:       channel:     broadcast
         solution:1     problem:3    to all
```

## 前端组件

### 页面路由

| 路由                                  | 文件                                                  | 说明            |
| ------------------------------------- | ----------------------------------------------------- | --------------- |
| `/solutions/[problemId]`              | `src/app/solutions/[problemId]/page.tsx`              | 题解列表页      |
| `/solutions/[problemId]/[solutionId]` | `src/app/solutions/[problemId]/[solutionId]/page.tsx` | 题解详情+讨论页 |

### useWebSocket Hook

位置：`src/hooks/use-websocket.ts`

核心功能：

1. **连接管理** — 自动连接、断线指数退避重连（最多10次）
2. **频道订阅** — subscribe / unsubscribe 管理感兴趣的频道
3. **事件分发** — 按消息类型 dispatch 到对应的监听器
4. **通知展示** — system_notice 和 like_notify 自动弹出 Ant Design notification

```typescript
const ws = useWebSocket({ autoConnect: true });

// 订阅频道
ws.subscribe("solution:1");

// 监听特定类型消息
const unsub = ws.on("new_comment", (msg) => {
  console.log("新评论:", msg);
});

// 发送聊天消息
ws.sendChat("solution:1", "写得真好！");

// 清理
unsub();
ws.unsubscribe("solution:1");
```

### 入口位置

在 `ProblemDetail.tsx` 组件的标题栏添加了「查看题解」按钮，点击跳转到 `/solutions/{problemId}`。

## i18n 国际化

新增约 50 个翻译键，前缀 `solutions.*`，同时覆盖中文 (`zh.ts`) 和英文 (`en.ts`)。

关键翻译键：

| Key                        | 中文             | English                 |
| -------------------------- | ---------------- | ----------------------- |
| `solutions.title`          | 题解区           | Solutions               |
| `solutions.entrance`       | 查看题解         | Solutions               |
| `solutions.write`          | 发布题解         | Write Solution          |
| `solutions.comments`       | 评论             | Comments                |
| `solutions.loginToComment` | 请先登录后再评论 | Please login to comment |

## 分支信息

- **后端分支**: `feat/solutions_0311`（从 main 切出）
- **前端分支**: `feat/solutions_0311`（从 main 切出）

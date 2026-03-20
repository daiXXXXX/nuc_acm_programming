# 刷题数排行榜功能实现文档

## 功能概述

本次实现了刷题数排行榜功能，包含两个维度：

1. **总榜** - 按用户历史总刷题数排序
2. **今日榜** - 按用户当日刷题数排序

## 技术架构

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Frontend      │────▶│    Backend      │────▶│    MySQL        │
│   (Next.js)     │     │    (Go/Gin)     │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
     /ranking            /api/ranking/*          user_stats 表
```

---

## 数据库层改动

### 表结构变更

在 `user_stats` 表中新增两列：

```sql
ALTER TABLE user_stats
ADD COLUMN today_solved INT NOT NULL DEFAULT 0 AFTER total_submissions,
ADD COLUMN today_date DATE DEFAULT NULL AFTER today_solved;
```

| 字段           | 类型 | 说明                           |
| -------------- | ---- | ------------------------------ |
| `today_solved` | INT  | 当日刷题数量                   |
| `today_date`   | DATE | 记录日期，用于判断是否需要重置 |

### 设计说明

- `today_date` 用于判断 `today_solved` 是否为当天数据
- 当 `today_date != CURDATE()` 时，说明数据已过期，需要在业务逻辑中重置
- 查询今日排行时使用 `WHERE today_date = CURDATE()` 过滤

---

## 后端实现 (Go)

### 1. 新增数据模型

**文件**: `internal/models/models.go`

```go
// RankingUser 排行榜用户信息
type RankingUser struct {
    UserID      int64  `json:"userId"`
    Username    string `json:"username"`
    Avatar      string `json:"avatar"`
    TotalSolved int    `json:"totalSolved"`
    TodaySolved int    `json:"todaySolved"`
    Rank        int    `json:"rank"`
}
```

同时更新 `UserStats` 结构体，添加 `TodaySolved` 和 `TodayDate` 字段。

### 2. 数据访问层

**文件**: `internal/database/user.go`

新增两个排行榜查询方法：

#### GetTotalSolvedRanking - 总榜查询

```go
func (r *UserRepository) GetTotalSolvedRanking(limit int) ([]models.RankingUser, error) {
    query := `
        SELECT u.id, u.username, COALESCE(u.avatar, '') as avatar,
               COALESCE(s.total_solved, 0) as total_solved,
               COALESCE(s.today_solved, 0) as today_solved
        FROM users u
        LEFT JOIN user_stats s ON u.id = s.user_id
        ORDER BY s.total_solved DESC, u.id ASC
        LIMIT ?
    `
    // ... 执行查询并组装结果
}
```

**技术要点**:

- 使用 `LEFT JOIN` 确保新用户（无统计记录）也能出现
- `COALESCE` 处理 NULL 值，默认为 0
- `ORDER BY s.total_solved DESC, u.id ASC` 先按刷题数降序，再按用户ID升序（保证排名稳定）
- 排名 `Rank` 在代码中动态计算，避免 SQL 复杂性

#### GetTodaySolvedRanking - 今日榜查询

```go
func (r *UserRepository) GetTodaySolvedRanking(limit int) ([]models.RankingUser, error) {
    query := `
        SELECT u.id, u.username, COALESCE(u.avatar, '') as avatar,
               COALESCE(s.total_solved, 0) as total_solved,
               COALESCE(s.today_solved, 0) as today_solved
        FROM users u
        LEFT JOIN user_stats s ON u.id = s.user_id
        WHERE s.today_date = CURDATE()
        ORDER BY s.today_solved DESC, u.id ASC
        LIMIT ?
    `
    // ... 执行查询并组装结果
}
```

**与总榜的区别**:

- 添加 `WHERE s.today_date = CURDATE()` 过滤条件
- 排序字段改为 `today_solved`

### 3. HTTP Handler

**文件**: `internal/handlers/ranking.go` (新建)

```go
type RankingHandler struct {
    userRepo *database.UserRepository
}

// GET /api/ranking/total?limit=50
func (h *RankingHandler) GetTotalSolvedRanking(c *gin.Context) {
    limit := 50 // 默认值
    if limitStr := c.Query("limit"); limitStr != "" {
        if l, err := strconv.Atoi(limitStr); err == nil && l > 0 && l <= 100 {
            limit = l
        }
    }
    users, err := h.userRepo.GetTotalSolvedRanking(limit)
    // ... 错误处理和响应
}

// GET /api/ranking/today?limit=50
func (h *RankingHandler) GetTodaySolvedRanking(c *gin.Context) {
    // 类似实现
}
```

**API 设计**:

- 支持 `limit` 查询参数，默认 50，最大 100
- 参数校验防止无效输入
- 公开接口，无需认证

### 4. 路由注册

**文件**: `cmd/server/main.go`

```go
// 排行榜路由（公开）
ranking := api.Group("/ranking")
{
    ranking.GET("/total", rankingHandler.GetTotalSolvedRanking)
    ranking.GET("/today", rankingHandler.GetTodaySolvedRanking)
}
```

---

## 前端实现 (Next.js + TypeScript)

### 1. 类型定义

**文件**: `src/lib/types.ts`

```typescript
export interface RankingUser {
  userId: number;
  username: string;
  avatar: string;
  totalSolved: number;
  todaySolved: number;
  rank: number;
}
```

### 2. API 客户端

**文件**: `src/lib/api.ts`

```typescript
// 排行榜相关 API（公开）
async getTotalSolvedRanking(limit: number = 50) {
  return this.request<RankingUser[]>(`/ranking/total?limit=${limit}`, { requireAuth: false })
}

async getTodaySolvedRanking(limit: number = 50) {
  return this.request<RankingUser[]>(`/ranking/today?limit=${limit}`, { requireAuth: false })
}
```

### 3. 排行榜页面

**文件**: `src/app/ranking/page.tsx`

```tsx
export default function RankingPage() {
  const [activeTab, setActiveTab] = useState("total");
  const [totalRanking, setTotalRanking] = useState<RankingUser[]>([]);
  const [todayRanking, setTodayRanking] = useState<RankingUser[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function fetchRankings() {
      // 并行请求两个排行榜数据
      const [total, today] = await Promise.all([
        api.getTotalSolvedRanking(50),
        api.getTodaySolvedRanking(50),
      ]);
      setTotalRanking(total || []);
      setTodayRanking(today || []);
    }
    fetchRankings();
  }, []);

  // 使用 Ant Design Tabs 组件切换
  return (
    <Tabs activeKey={activeTab} onChange={setActiveTab} items={tabItems} />
  );
}
```

**技术要点**:

- 使用 `Promise.all` 并行请求，减少加载时间
- Framer Motion 实现列表动画效果
- 前三名使用特殊图标（Crown、Medal）突出显示

### 4. 排行榜入口

**文件**: `src/app/workspace/page.tsx`

在 header 中添加排行榜链接：

```tsx
<Link href="/ranking">
  <Button type="text" className="flex items-center gap-1.5">
    <Trophy size={18} weight="duotone" />
    <span>{t("header.ranking")}</span>
  </Button>
</Link>
```

### 5. 样式

**文件**: `src/app/ranking/ranking.module.css`

```css
.rankingItem {
  padding: 12px 16px;
  background: #fafafa;
  border-radius: 8px;
  transition: all 0.2s ease;
}

.rankingItem:hover {
  background: #f0f0f0;
  transform: translateX(4px);
}

.topThree {
  background: linear-gradient(135deg, #fff7e6 0%, #fff1cc 100%);
  border: 1px solid #ffd666;
}
```

### 6. 国际化 (i18n)

**文件**: `src/lib/i18n/zh.ts` & `en.ts`

```typescript
// 中文
'header.ranking': '排行榜',
'ranking.title': '刷题排行榜',
'ranking.totalTab': '总榜',
'ranking.todayTab': '今日榜',
'ranking.noData': '暂无排行数据',

// English
'header.ranking': 'Ranking',
'ranking.title': 'Leaderboard',
'ranking.totalTab': 'All Time',
'ranking.todayTab': 'Today',
'ranking.noData': 'No ranking data',
```

---

## API 接口文档

### GET /api/ranking/total

获取总刷题数排行榜

**请求参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| limit | int | 否 | 返回数量，默认50，最大100 |

**响应示例**:

```json
[
  {
    "userId": 1,
    "username": "张三",
    "avatar": "https://...",
    "totalSolved": 150,
    "todaySolved": 5,
    "rank": 1
  },
  ...
]
```

### GET /api/ranking/today

获取今日刷题数排行榜（参数和响应格式同上）

---

## 文件清单

| 文件路径                             | 操作 | 说明                             |
| ------------------------------------ | ---- | -------------------------------- |
| `database/user_stats`                | 修改 | 新增 today_solved, today_date 列 |
| `internal/models/models.go`          | 修改 | 新增 RankingUser 类型            |
| `internal/database/user.go`          | 修改 | 新增排行榜查询方法               |
| `internal/handlers/ranking.go`       | 新增 | 排行榜 HTTP Handler              |
| `cmd/server/main.go`                 | 修改 | 注册排行榜路由                   |
| `src/lib/types.ts`                   | 修改 | 新增 RankingUser 类型            |
| `src/lib/api.ts`                     | 修改 | 新增排行榜 API 方法              |
| `src/app/ranking/page.tsx`           | 新增 | 排行榜页面组件                   |
| `src/app/ranking/ranking.module.css` | 新增 | 排行榜样式                       |
| `src/app/workspace/page.tsx`         | 修改 | header 添加入口                  |
| `src/lib/i18n/zh.ts`                 | 修改 | 中文翻译                         |
| `src/lib/i18n/en.ts`                 | 修改 | 英文翻译                         |

---

## 后续优化建议

1. **定时任务**: 添加凌晨重置 `today_solved` 的定时任务
2. **缓存**: 排行榜数据可添加 Redis 缓存，减少数据库压力
3. **分页**: 支持分页加载更多排名数据
4. **周榜/月榜**: 扩展更多时间维度的排行榜
5. **用户头像**: 完善头像上传功能

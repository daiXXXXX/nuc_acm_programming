# 登录系统设计文档

## 概述

本文档详细说明编程实验平台的用户认证系统设计与实现。系统采用 JWT (JSON Web Token) 进行身份认证，支持学生和教师两种角色，并实现了完善的安全机制。

## 技术架构

### 后端技术栈

- **框架**: Gin (Go Web Framework)
- **数据库**: MySQL 8.0
- **认证**: JWT (golang-jwt/jwt/v5)
- **密码加密**: bcrypt (golang.org/x/crypto/bcrypt)

### 前端技术栈

- **框架**: Next.js 14 (App Router)
- **状态管理**: Zustand (持久化到 localStorage)
- **UI组件**: Ant Design 5.x

## 用户角色

系统支持三种用户角色，采用层级权限模型：

| 角色   | 标识         | 权限等级 | 说明                                         |
| ------ | ------------ | -------- | -------------------------------------------- |
| 学生   | `student`    | 1        | 基础权限，可查看题目、提交代码、查看个人统计 |
| 教师   | `instructor` | 2        | 包含学生权限，额外可创建/编辑/删除题目       |
| 管理员 | `admin`      | 3        | 最高权限，包含所有功能                       |

### 用户表结构

```sql
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'student',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

## 认证流程

### 1. 注册流程

```
客户端                                    服务器
   |                                        |
   |  POST /api/auth/register              |
   |  { username, email, password, role }  |
   |--------------------------------------->|
   |                                        |
   |                                        | 1. 验证用户名格式
   |                                        | 2. 验证密码强度
   |                                        | 3. 检查用户名/邮箱唯一性
   |                                        | 4. bcrypt加密密码 (cost=12)
   |                                        | 5. 创建用户记录
   |                                        | 6. 生成Token对
   |                                        |
   |  { user, accessToken, refreshToken }  |
   |<---------------------------------------|
```

### 2. 登录流程

```
客户端                                    服务器
   |                                        |
   |  POST /api/auth/login                 |
   |  { username, password }               |
   |--------------------------------------->|
   |                                        |
   |                                        | 1. 查找用户（支持用户名或邮箱）
   |                                        | 2. bcrypt验证密码
   |                                        | 3. 生成Token对
   |                                        |
   |  { user, accessToken, refreshToken }  |
   |<---------------------------------------|
```

### 3. Token刷新流程

```
客户端                                    服务器
   |                                        |
   |  POST /api/auth/refresh               |
   |  { refreshToken }                     |
   |--------------------------------------->|
   |                                        |
   |                                        | 1. 验证refreshToken有效性
   |                                        | 2. 检查Token类型
   |                                        | 3. 生成新Token对
   |                                        |
   |  { user, accessToken, refreshToken }  |
   |<---------------------------------------|
```

## JWT Token 设计

### Token 结构

系统使用双Token机制：

| Token类型     | 有效期 | 用途             |
| ------------- | ------ | ---------------- |
| Access Token  | 1小时  | API请求认证      |
| Refresh Token | 7天    | 刷新Access Token |

### Claims 结构

```go
type Claims struct {
    UserID   int64     `json:"userId"`
    Username string    `json:"username"`
    Email    string    `json:"email"`
    Role     UserRole  `json:"role"`
    Type     TokenType `json:"type"`    // "access" 或 "refresh"
    jwt.RegisteredClaims
}
```

### Token 安全机制

1. **签名算法**: HS256 (HMAC-SHA256)
2. **密钥配置**: 通过环境变量 `JWT_SECRET` 配置
3. **Token类型验证**: 防止Refresh Token被用于API认证
4. **过期时间验证**: 自动拒绝过期Token

## 密码安全

### bcrypt 加密

```go
// 加密密码 (cost=12)
hash, err := bcrypt.GenerateFromPassword([]byte(password), 12)

// 验证密码
err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
```

**bcrypt cost=12 的特点：**

- 每次哈希约需 250ms（可抵御暴力破解）
- 自动包含盐值（salt）
- 抗彩虹表攻击

### 密码强度要求

注册时密码必须满足：

- 最少6个字符
- 至少1个大写字母
- 至少1个小写字母
- 至少1个数字

```go
func isStrongPassword(password string) bool {
    var hasUpper, hasLower, hasDigit bool
    for _, c := range password {
        switch {
        case c >= 'A' && c <= 'Z': hasUpper = true
        case c >= 'a' && c <= 'z': hasLower = true
        case c >= '0' && c <= '9': hasDigit = true
        }
    }
    return len(password) >= 6 && hasUpper && hasLower && hasDigit
}
```

## API 端点

### 公开端点

| 方法 | 路径                 | 说明         |
| ---- | -------------------- | ------------ |
| POST | `/api/auth/register` | 用户注册     |
| POST | `/api/auth/login`    | 用户登录     |
| POST | `/api/auth/refresh`  | 刷新Token    |
| GET  | `/api/problems`      | 获取题目列表 |
| GET  | `/api/problems/:id`  | 获取题目详情 |

### 需要认证端点

| 方法   | 路径                   | 说明             | 最低权限   |
| ------ | ---------------------- | ---------------- | ---------- |
| GET    | `/api/auth/me`         | 获取当前用户信息 | student    |
| PUT    | `/api/auth/password`   | 修改密码         | student    |
| POST   | `/api/submissions`     | 提交代码         | student    |
| GET    | `/api/submissions/:id` | 获取提交详情     | student    |
| GET    | `/api/stats/user/:id`  | 获取用户统计     | student    |
| POST   | `/api/problems`        | 创建题目         | instructor |
| PUT    | `/api/problems/:id`    | 更新题目         | instructor |
| DELETE | `/api/problems/:id`    | 删除题目         | instructor |

## 中间件

### AuthMiddleware

验证JWT Token，提取用户信息：

```go
func AuthMiddleware(jwtManager *auth.JWTManager) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. 提取Authorization头
        // 2. 验证Bearer格式
        // 3. 解析并验证Token
        // 4. 检查Token类型为access
        // 5. 将用户信息存入上下文
        c.Set("userID", claims.UserID)
        c.Set("username", claims.Username)
        c.Set("role", claims.Role)
        c.Next()
    }
}
```

### RoleMiddleware

基于角色的权限控制：

```go
func RoleMiddleware(requiredRole auth.UserRole) gin.HandlerFunc {
    return func(c *gin.Context) {
        userRole := c.Get("role").(auth.UserRole)
        if !auth.HasPermission(userRole, requiredRole) {
            c.JSON(403, gin.H{"error": "forbidden"})
            c.Abort()
            return
        }
        c.Next()
    }
}
```

## 前端实现

### 状态管理 (Zustand)

```typescript
// store/authStore.ts
interface AuthState {
  user: User | null;
  accessToken: string | null;
  refreshToken: string | null;
  expiresAt: string | null;
  isAuthenticated: boolean;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      // ... 状态和方法
    }),
    { name: "auth-storage" }, // localStorage key
  ),
);
```

### 认证Hook

```typescript
// hooks/use-auth.ts
export function useAuth() {
  const { login, register, logout, ... } = useAuthStore()

  // 初始化时验证Token
  useEffect(() => {
    if (isTokenExpired()) {
      refreshToken()
    }
  }, [])

  return {
    user,
    isAuthenticated,
    login,
    register,
    logout,
    isInstructor,
    isAdmin,
    hasPermission,
  }
}
```

### API请求自动添加Token

```typescript
// lib/api.ts
private async request<T>(endpoint: string, options?: RequestInit) {
  const headers = { 'Content-Type': 'application/json' }

  const token = getAccessToken()
  if (token) {
    headers['Authorization'] = `Bearer ${token}`
  }

  return fetch(`${this.baseURL}${endpoint}`, { ...options, headers })
}
```

## 文件结构

### 后端

```
internal/
├── auth/
│   └── jwt.go           # JWT管理、密码加密
├── database/
│   └── user.go          # 用户数据库操作
├── handlers/
│   └── auth.go          # 认证API处理器
└── middleware/
    └── middleware.go    # 认证/权限中间件
```

### 前端

```
src/
├── store/
│   └── authStore.ts     # Zustand认证状态
├── hooks/
│   └── use-auth.ts      # 认证Hook
├── lib/
│   └── auth-api.ts      # 认证API客户端
└── app/
    └── login/
        └── page.tsx     # 登录页面
```

## 安全措施总结

1. **密码安全**
   - bcrypt加密 (cost=12)
   - 强密码策略
   - 密码永不明文传输或存储

2. **Token安全**
   - 短期Access Token (15分钟)
   - 长期Refresh Token (7天)
   - Token类型验证
   - HMAC-SHA256签名

3. **API安全**
   - 基于角色的访问控制 (RBAC)
   - 认证中间件保护
   - 错误信息不泄露敏感信息

4. **通信安全**
   - 建议生产环境使用HTTPS
   - CORS配置限制来源

## 配置说明

### 环境变量

```env
# JWT密钥（生产环境必须修改）
JWT_SECRET=your-secure-secret-key-at-least-32-chars

# 数据库配置
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your-password
DB_NAME=xfy_bs
```

## 使用示例

### 注册用户

```bash
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "student1",
    "email": "student1@example.com",
    "password": "Student123",
    "role": "student"
  }'
```

### 登录

```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "student1",
    "password": "Student123"
  }'
```

### 使用Token访问API

```bash
curl http://localhost:8080/api/auth/me \
  -H "Authorization: Bearer <access_token>"
```

## 未来扩展

1. **Token黑名单**: 实现登出时使Token失效
2. **多设备管理**: 记录登录设备，支持踢出
3. **OAuth2集成**: 支持第三方登录
4. **2FA**: 两步验证
5. **登录日志**: 记录登录历史和异常检测

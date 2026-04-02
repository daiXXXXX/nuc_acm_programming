# 用户个人信息 (Profile) 功能开发文档

## 1. 需求概述

用户可以在 **workspace 页面右上角** 点击个人头像，打开一个 **Profile Modal**，在其中查看和修改自己的个人信息，包括：

- **用户名** (username)
- **邮箱** (email)
- **头像** (avatar) — 支持三种方式：**本地文件上传** + 预设头像选择 + 自定义 URL
- **个人简介** (bio)

同时展示只读信息：用户角色 (student/instructor/admin)、注册时间。

---

## 2. 整体架构

```
┌──────────────────────────────────────────────────────────┐
│                     前端 (Next.js)                        │
│                                                          │
│  workspace/page.tsx                                      │
│    └─ Header 右上角 Avatar + Dropdown                     │
│         ├─ 「个人信息」菜单项 → 打开 ProfileModal          │
│         └─ 「退出登录」菜单项                              │
│                                                          │
│  ProfileModal.tsx                                        │
│    ├─ 渐变 Banner + 头像展示                              │
│    ├─ 头像区域 (Ant Design Upload 组件包裹)               │
│    │    └─ 点击头像 → 选择本地文件 → 上传到后端             │
│    │         └─ useAuth().uploadAvatar(file)              │
│    │              └─ authApi.uploadAvatar()               │
│    │                  └─ POST /api/auth/avatar            │
│    ├─ 「选择预设头像」展开/收起按钮                         │
│    │    ├─ 12 个 DiceBear 预设头像网格                     │
│    │    └─ 自定义头像 URL 输入框                           │
│    ├─ 表单: username / email / bio                       │
│    └─ 保存 → useAuth().updateProfile()                   │
│              └─ authApi.updateProfile()                   │
│                  └─ PUT /api/auth/profile                │
└──────────────────────┬───────────────────────────────────┘
                       │  HTTP Request
                       ▼
┌──────────────────────────────────────────────────────────┐
│                   后端 (Go + Gin)                         │
│                                                          │
│  路由: POST /api/auth/avatar (需要 JWT 认证)              │
│    └─ AuthHandler.UploadAvatar()                         │
│         ├─ 接收 multipart/form-data 文件                  │
│         ├─ 文件校验 (大小 ≤ 2MB, 类型白名单)              │
│         ├─ 保存到 uploads/avatars/{userID}_{nano}.{ext}  │
│         ├─ 删除旧头像文件 (如果是本地上传的)               │
│         └─ UserRepository.UpdateAvatar()                 │
│              └─ UPDATE users SET avatar = ? WHERE id = ? │
│                                                          │
│  路由: PUT /api/auth/profile (需要 JWT 认证)              │
│    └─ AuthHandler.UpdateProfile()                        │
│         ├─ 参数校验 (username/email/avatar/bio)           │
│         ├─ 用户名/邮箱 唯一性检查                          │
│         └─ UserRepository.UpdateProfile()                │
│              └─ UPDATE users SET ... WHERE id = ?        │
│                                                          │
│  静态文件服务:                                            │
│    router.Static("/uploads", "./uploads")                │
│    → 前端通过 /uploads/avatars/xxx.jpg 访问头像           │
└──────────────────────────────────────────────────────────┘
```

---

## 3. 数据库改动

### 3.1 `users` 表新增字段

| 字段     | 类型           | 默认值 | 说明         |
| -------- | -------------- | ------ | ------------ |
| `avatar` | `VARCHAR(500)` | `''`   | 用户头像 URL |
| `bio`    | `VARCHAR(500)` | `''`   | 个人简介     |

### 3.2 DDL 变更

**schema.sql** 中的定义（新字段位于 `role` 之后）：

```sql
CREATE TABLE IF NOT EXISTS users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'student',
    avatar VARCHAR(500) DEFAULT '',          -- ✅ 新增
    bio VARCHAR(500) DEFAULT '',             -- ✅ 新增
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**线上迁移 SQL**（已在开发环境执行）：

```sql
ALTER TABLE users
  ADD COLUMN avatar VARCHAR(500) DEFAULT '' AFTER role,
  ADD COLUMN bio VARCHAR(500) DEFAULT '' AFTER avatar;
```

---

## 4. 后端改动

### 4.1 涉及文件

| 文件                        | 改动说明                                                                                              |
| --------------------------- | ----------------------------------------------------------------------------------------------------- |
| `internal/models/models.go` | `User` 结构体新增 `Avatar`、`Bio` 字段                                                                |
| `internal/database/user.go` | 所有查询 (GetByID/GetByUsername/GetByEmail) 加入 avatar/bio；新增 `UpdateProfile()`、`UpdateAvatar()` |
| `internal/handlers/auth.go` | `UserDTO` 新增字段；新增 `UpdateProfile`、`UploadAvatar` handler；所有响应包含 avatar/bio             |
| `cmd/server/main.go`        | 注册路由 `PUT /api/auth/profile`、`POST /api/auth/avatar`；配置静态文件服务 `/uploads`                |

### 4.2 Model 层

```go
type User struct {
    ID           int64     `json:"id"`
    Username     string    `json:"username"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`
    Role         string    `json:"role"`
    Avatar       string    `json:"avatar"`    // ✅ 新增
    Bio          string    `json:"bio"`       // ✅ 新增
    CreatedAt    time.Time `json:"createdAt"`
    UpdatedAt    time.Time `json:"updatedAt"`
}
```

### 4.3 Repository 层

**查询改动** — `GetByID` / `GetByUsername` / `GetByEmail` 统一升级：

```go
SELECT id, username, email, password_hash, role,
       COALESCE(avatar, '') as avatar,
       COALESCE(bio, '') as bio,
       created_at, updated_at
FROM users WHERE ...
```

使用 `COALESCE` 确保 NULL 值转为空字符串，避免前端处理空值问题。

**新增方法** — `UpdateProfile`：

```go
func (r *UserRepository) UpdateProfile(userID int64, username, email, avatar, bio string) error {
    query := `UPDATE users SET username = ?, email = ?, avatar = ?, bio = ?, updated_at = ? WHERE id = ?`
    _, err := r.db.Exec(query, username, email, avatar, bio, time.Now(), userID)
    if err != nil {
        if isDuplicateKeyError(err) {
            return ErrUserAlreadyExists
        }
        return err
    }
    return nil
}
```

**新增方法** — `UpdateAvatar`（仅更新头像字段）：

```go
func (r *UserRepository) UpdateAvatar(userID int64, avatar string) error {
    query := `UPDATE users SET avatar = ?, updated_at = ? WHERE id = ?`
    _, err := r.db.Exec(query, avatar, time.Now(), userID)
    return err
}
```

### 4.4 Handler 层

**DTO 升级：**

```go
type UserDTO struct {
    ID        int64     `json:"id"`
    Username  string    `json:"username"`
    Email     string    `json:"email"`
    Role      string    `json:"role"`
    Avatar    string    `json:"avatar"`    // ✅ 新增
    Bio       string    `json:"bio"`       // ✅ 新增
    CreatedAt time.Time `json:"createdAt"`
}
```

> 注意：Register、Login、RefreshToken、GetCurrentUser 四个接口的响应都已同步更新，始终返回 avatar 和 bio。

**新增 `UpdateProfile` Handler：**

- **请求体**：`UpdateProfileRequest { Username, Email, Avatar, Bio }`
- **校验逻辑**：
  1. 参数格式校验（用户名 3-50 字符，仅字母/数字/下划线；邮箱合法性）
  2. 用户名唯一性检查（跳过自身）
  3. 邮箱唯一性检查（跳过自身）
- **响应**：返回更新后的 `UserDTO`

**新增 `UploadAvatar` Handler（头像文件上传）：**

- **请求格式**：`multipart/form-data`，字段名 `avatar`
- **处理流程**：
  1. 从 `c.FormFile("avatar")` 获取上传文件
  2. **文件大小校验**：最大 2MB (`file.Size > 2*1024*1024`)
  3. **文件类型校验**：仅允许 `.jpg`、`.jpeg`、`.png`、`.gif`、`.webp` 五种扩展名
  4. **创建上传目录**：`os.MkdirAll("uploads/avatars", 0755)` 确保目录存在
  5. **生成唯一文件名**：`{userID}_{纳秒时间戳}.{ext}`，例如 `1_1709366400000000000.png`
  6. **保存文件**：`c.SaveUploadedFile(file, "uploads/avatars/{filename}")`
  7. **生成访问 URL**：`/uploads/avatars/{filename}`（由 Gin 静态文件服务提供访问）
  8. **删除旧头像文件**：如果用户现有 avatar 以 `/uploads/avatars/` 开头，则 `os.Remove` 删除旧文件
  9. **更新数据库**：`UserRepository.UpdateAvatar(userID, avatarURL)` 写入新 URL
  10. 如果数据库更新失败，回滚删除已上传的文件
- **响应**：`{ "avatar": "/uploads/avatars/1_xxx.png", "message": "Avatar uploaded successfully" }`

### 4.5 静态文件服务

```go
// main.go — 提供上传文件的 HTTP 访问
router.Static("/uploads", "./uploads")
```

后端通过 Gin 的 `Static` 中间件将 `./uploads` 目录映射到 `/uploads` 路径，前端可直接通过 URL（如 `/uploads/avatars/1_xxx.png`）访问已上传的头像图片。

### 4.6 路由注册

```go
authProtected.PUT("/profile", authHandler.UpdateProfile)
authProtected.POST("/avatar", authHandler.UploadAvatar)
```

位于认证保护路由组下，需携带有效的 JWT Access Token。

---

## 5. 前端改动

### 5.1 涉及文件

| 文件                              | 改动说明                                                                            |
| --------------------------------- | ----------------------------------------------------------------------------------- |
| `src/store/authStore.ts`          | `User` 接口新增 `avatar`、`bio` 字段                                                |
| `src/lib/auth-api.ts`             | 新增 `UpdateProfileRequest` 类型、`updateProfile` API 方法、`uploadAvatar` API 方法 |
| `src/hooks/use-auth.ts`           | 新增 `updateProfile`、`uploadAvatar` 函数，调用 API 后自动更新 store                |
| `src/components/ProfileModal.tsx` | 全新组件 — 个人信息编辑 Modal（含头像上传）                                         |
| `src/app/workspace/page.tsx`      | Header 集成 ProfileModal，头像显示用户实际 avatar；传入 `onUploadAvatar` 回调       |
| `src/lib/i18n/zh.ts`              | 新增 22 条中文翻译                                                                  |
| `src/lib/i18n/en.ts`              | 新增 22 条英文翻译                                                                  |

### 5.2 类型定义

```typescript
// authStore.ts
export interface User {
  id: number;
  username: string;
  email: string;
  role: UserRole;
  avatar: string; // ✅ 新增
  bio: string; // ✅ 新增
  createdAt: string;
}

// auth-api.ts
export interface UpdateProfileRequest {
  username: string;
  email: string;
  avatar: string;
  bio: string;
}
```

### 5.3 API 调用

```typescript
// auth-api.ts — 更新个人信息（JSON）
updateProfile: async (data: UpdateProfileRequest): Promise<User> => {
  return authFetch<User>("/auth/profile", {
    method: "PUT",
    body: JSON.stringify(data),
    requireAuth: true,
  });
};

// auth-api.ts — 上传头像（FormData）
uploadAvatar: async (
  file: File,
): Promise<{ avatar: string; message: string }> => {
  const formData = new FormData();
  formData.append("avatar", file);

  const { accessToken } = getStoredAuth();
  const headers: Record<string, string> = {};
  if (accessToken) {
    headers["Authorization"] = `Bearer ${accessToken}`;
  }

  // 注意：不设置 Content-Type，让浏览器自动设置 multipart boundary
  const response = await fetch(`${API_BASE}/auth/avatar`, {
    method: "POST",
    headers,
    body: formData,
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({
      error: "unknown_error",
      message: "An unknown error occurred",
    }));
    throw new AuthError(error.message, error.error, response.status);
  }

  return response.json();
};
```

### 5.4 Hook 集成

```typescript
// use-auth.ts — 更新个人信息
const updateProfile = useCallback(
  async (data: UpdateProfileRequest) => {
    setLoading(true);
    try {
      const updatedUser = await authApi.updateProfile(data);
      setUser(updatedUser); // 自动更新 Zustand store
      message.success("个人信息更新成功");
      return updatedUser;
    } catch (err) {
      const authError = err as AuthError;
      message.error(authError.message || "个人信息更新失败");
      throw err;
    } finally {
      setLoading(false);
    }
  },
  [setUser, setLoading],
);

// use-auth.ts — 上传头像
const uploadAvatar = useCallback(
  async (file: File) => {
    setLoading(true);
    try {
      const result = await authApi.uploadAvatar(file);
      // 上传成功后，立即更新本地 store 中的头像 URL
      if (user) {
        setUser({ ...user, avatar: result.avatar });
      }
      message.success("头像上传成功");
      return result;
    } catch (err) {
      const authError = err as AuthError;
      message.error(authError.message || "头像上传失败");
      throw err;
    } finally {
      setLoading(false);
    }
  },
  [user, setUser, setLoading],
);
```

### 5.5 ProfileModal 组件

**功能特性：**

| 特性             | 说明                                                                                                |
| ---------------- | --------------------------------------------------------------------------------------------------- |
| 渐变头部 Banner  | 紫色渐变背景 (`#667eea → #764ba2`)，显示标题和副标题                                                |
| 头像展示与上传   | 头像半嵌入 Banner，使用 Ant Design `Upload` 组件包裹，点击头像直接选择本地文件上传                  |
| 上传按钮图标     | 头像右下角显示相机图标 (CameraOutlined)，上传中显示 `Spin` loading                                  |
| 文件上传前端校验 | `beforeUpload` 中校验文件大小 ≤ 2MB，通过 `accept` 属性限制文件类型                                 |
| 上传流程         | 选择文件 → `beforeUpload` 拦截 → 调用 `onUploadAvatar(file)` → 后端返回 URL → 更新 `selectedAvatar` |
| 预设头像选择     | 「选择预设头像」按钮展开/收起，12 个 DiceBear Adventurer 风格头像网格展示                           |
| 自定义头像 URL   | 预设头像面板底部输入框，支持任意图片 URL                                                            |
| 默认头像         | 未设置时使用 `dicebear.com/adventurer?seed={username}` 自动生成                                     |
| 角色标签         | 彩色 Tag 展示用户角色（蓝/绿/红）                                                                   |
| 注册时间         | 展示用户注册日期                                                                                    |
| 表单校验         | 用户名格式/长度、邮箱格式、简介长度上限                                                             |
| 保存反馈         | Loading 状态 + 成功/失败消息提示                                                                    |
| 国际化           | 完整支持中/英文切换                                                                                 |

**头像设置的三种方式：**

1. **本地文件上传**（推荐）：点击头像区域，选择本地图片文件，自动上传到后端服务器的 `uploads/avatars/` 目录，数据库存储路径如 `/uploads/avatars/1_xxx.png`
2. **预设头像选择**：展开预设面板，点击 12 个 DiceBear 预设头像之一，avatar 值为 DiceBear CDN URL
3. **自定义 URL 输入**：在预设面板底部的输入框中手动输入任意图片 URL

> **注意**：方式 1 是独立的上传接口 (`POST /api/auth/avatar`)，上传成功后 avatar URL 立即写入数据库并更新前端 store。方式 2 和 3 修改的是 `selectedAvatar` 状态，需要点击「保存」按钮通过 `PUT /api/auth/profile` 一并提交。

**组件 Props：**

```typescript
interface ProfileModalProps {
  open: boolean;
  onClose: () => void;
  user: User;
  onUpdateProfile: (data: {
    username: string;
    email: string;
    avatar: string;
    bio: string;
  }) => Promise<unknown>;
  onUploadAvatar?: (file: File) => Promise<{ avatar: string; message: string }>;
  isLoading?: boolean;
}
```

### 5.6 Header 集成

在 `workspace/page.tsx` 的 Header 区域：

1. 头像 `<Avatar>` 显示用户的 `avatar`，无头像时使用 DiceBear 自动生成
2. Dropdown 菜单新增 **「个人信息」** 选项（带 EditOutlined 图标）
3. 点击打开 `ProfileModal`，传入 `onUploadAvatar={uploadAvatar}` 回调
4. 上传头像后 store 实时更新，Header 头像立即刷新
5. 保存个人信息后 Modal 自动关闭

```tsx
<ProfileModal
  open={profileModalOpen}
  onClose={() => setProfileModalOpen(false)}
  user={user}
  onUpdateProfile={updateProfile}
  onUploadAvatar={uploadAvatar}
  isLoading={isLoading}
/>
```

---

## 6. 数据流

### 6.1 头像文件上传流程

```
用户点击头像区域 (Upload 组件)
    ↓
浏览器弹出文件选择器 (accept="image/png,image/jpeg,image/jpg,image/gif,image/webp")
    ↓
选择文件 → 触发 beforeUpload 回调
    ↓
前端校验: file.size ≤ 2MB
    ↓ (通过)
调用 onUploadAvatar(file) → useAuth().uploadAvatar(file)
    ↓
authApi.uploadAvatar(file)
    → 构建 FormData, append('avatar', file)
    → POST /api/auth/avatar (Authorization: Bearer token)
    ↓
后端 UploadAvatar Handler:
    ├─ 校验文件大小 (≤ 2MB)
    ├─ 校验文件类型 (jpg/jpeg/png/gif/webp)
    ├─ os.MkdirAll("uploads/avatars", 0755) 确保目录存在
    ├─ 生成文件名: {userID}_{纳秒时间戳}.{ext}
    ├─ c.SaveUploadedFile → 写入 uploads/avatars/
    ├─ 删除旧头像文件 (如旧 avatar 以 /uploads/avatars/ 开头)
    └─ UserRepository.UpdateAvatar() → UPDATE SQL
    ↓
返回 { "avatar": "/uploads/avatars/1_xxx.png", "message": "..." }
    ↓
前端 setUser({ ...user, avatar: result.avatar }) → Zustand store 更新
    ↓
ProfileModal 中 selectedAvatar 更新 → 头像立即显示新图片
Header 中 Avatar 组件也响应式更新
```

### 6.2 个人信息保存流程

```
用户修改 username / email / bio，或选择预设头像/输入自定义 URL
    ↓
点击「保存」
    ↓
form.validateFields() → 前端校验
    ↓
useAuth().updateProfile({ username, email, avatar: selectedAvatar, bio })
    ↓
authApi.updateProfile() → PUT /api/auth/profile (JSON)
    ↓
后端校验 (格式 + 唯一性)
    ↓
UserRepository.UpdateProfile() → UPDATE SQL
    ↓
返回更新后的 UserDTO
    ↓
前端 setUser(updatedUser) → Zustand store 更新
    ↓
页面响应式更新 (头像、用户名等)
    ↓
Modal 关闭，message.success 提示
```

---

## 7. API 文档

### `POST /api/auth/avatar` — 上传头像

**认证要求：** Bearer Token (JWT)

**请求格式：** `multipart/form-data`

| 字段     | 类型   | 必填 | 说明         |
| -------- | ------ | ---- | ------------ |
| `avatar` | `File` | ✅   | 头像图片文件 |

**文件校验：**

| 校验项   | 规则                                 |
| -------- | ------------------------------------ |
| 文件大小 | ≤ 2MB                                |
| 文件类型 | `.jpg` `.jpeg` `.png` `.gif` `.webp` |

**成功响应 (200)：**

```json
{
  "avatar": "/uploads/avatars/1_1709366400000000000.png",
  "message": "Avatar uploaded successfully"
}
```

**错误响应：**

| HTTP 状态码 | error code          | 场景                     |
| ----------- | ------------------- | ------------------------ |
| 400         | `no_file`           | 未上传文件               |
| 400         | `file_too_large`    | 文件大小超过 2MB         |
| 400         | `invalid_file_type` | 文件类型不在白名单内     |
| 401         | `unauthorized`      | 未携带有效 Token         |
| 500         | `server_error`      | 文件保存或数据库更新失败 |

**后端存储细节：**

- 文件保存路径：`./uploads/avatars/{userID}_{纳秒时间戳}.{ext}`
- 数据库存储值：`/uploads/avatars/{userID}_{纳秒时间戳}.{ext}`（相对 URL）
- 前端访问方式：通过 Gin 静态文件服务 `router.Static("/uploads", "./uploads")` 直接访问
- 旧文件清理：如果用户之前的 avatar 是本地上传的（以 `/uploads/avatars/` 开头），上传新头像时会自动删除旧文件

---

### `PUT /api/auth/profile` — 更新个人信息

**认证要求：** Bearer Token (JWT)

**请求体：**

```json
{
  "username": "new_username",
  "email": "new_email@example.com",
  "avatar": "https://api.dicebear.com/9.x/adventurer/svg?seed=Felix",
  "bio": "Hello, I am a student."
}
```

**字段校验：**

| 字段       | 必填 | 规则                          |
| ---------- | ---- | ----------------------------- |
| `username` | ✅   | 3-50 字符，仅字母/数字/下划线 |
| `email`    | ✅   | 有效邮箱格式                  |
| `avatar`   | ❌   | 最长 500 字符                 |
| `bio`      | ❌   | 最长 500 字符                 |

**成功响应 (200)：**

```json
{
  "id": 1,
  "username": "new_username",
  "email": "new_email@example.com",
  "role": "student",
  "avatar": "https://api.dicebear.com/9.x/adventurer/svg?seed=Felix",
  "bio": "Hello, I am a student.",
  "createdAt": "2026-02-28T10:00:00Z"
}
```

**错误响应：**

| HTTP 状态码 | error code         | 场景             |
| ----------- | ------------------ | ---------------- |
| 400         | `validation_error` | 请求参数不合法   |
| 400         | `invalid_username` | 用户名格式不符   |
| 401         | `unauthorized`     | 未携带有效 Token |
| 404         | `user_not_found`   | 用户不存在       |
| 409         | `username_exists`  | 用户名已被占用   |
| 409         | `email_exists`     | 邮箱已被注册     |
| 500         | `server_error`     | 服务器内部错误   |

---

## 8. 国际化翻译键

共新增 **22 条** i18n 翻译，全部以 `profile.` 为前缀：

| Key                            | 中文                             | English                                                     |
| ------------------------------ | -------------------------------- | ----------------------------------------------------------- |
| `profile.title`                | 个人信息                         | Profile                                                     |
| `profile.subtitle`             | 管理你的账户信息和偏好设置       | Manage your account information and preferences             |
| `profile.selectAvatar`         | 选择一个头像                     | Choose an avatar                                            |
| `profile.avatarUrlPlaceholder` | 或者输入自定义头像 URL...        | Or enter a custom avatar URL...                             |
| `profile.showPresetAvatars`    | 选择预设头像                     | Choose preset avatar                                        |
| `profile.hidePresetAvatars`    | 收起预设头像                     | Hide preset avatars                                         |
| `profile.avatarTooLarge`       | 头像文件大小不能超过2MB          | Avatar file size must be less than 2MB                      |
| `profile.joinDate`             | 加入时间                         | Joined                                                      |
| `profile.username`             | 用户名                           | Username                                                    |
| `profile.usernamePlaceholder`  | 请输入用户名                     | Enter username                                              |
| `profile.usernameRequired`     | 请输入用户名                     | Username is required                                        |
| `profile.usernameMin`          | 用户名至少3个字符                | Username must be at least 3 characters                      |
| `profile.usernameMax`          | 用户名最多50个字符               | Username must be at most 50 characters                      |
| `profile.usernamePattern`      | 用户名只能包含字母、数字和下划线 | Username can only contain letters, numbers, and underscores |
| `profile.email`                | 邮箱                             | Email                                                       |
| `profile.emailPlaceholder`     | 请输入邮箱                       | Enter email                                                 |
| `profile.emailRequired`        | 请输入邮箱                       | Email is required                                           |
| `profile.emailInvalid`         | 请输入有效的邮箱地址             | Please enter a valid email address                          |
| `profile.bio`                  | 个人简介                         | Bio                                                         |
| `profile.bioPlaceholder`       | 介绍一下你自己...                | Tell us about yourself...                                   |
| `profile.bioMax`               | 个人简介最多500个字符            | Bio must be at most 500 characters                          |
| `profile.save`                 | 保存                             | Save                                                        |
| `profile.role.student`         | 学生                             | Student                                                     |
| `profile.role.instructor`      | 教师                             | Instructor                                                  |
| `profile.role.admin`           | 管理员                           | Admin                                                       |

---

## 9. 本次升级：从 Modal 到独立 `/profile` 页面

本次改动对个人信息模块做了两项重要升级：

1. **交互形态升级**：个人信息入口从 `workspace` 顶部下拉菜单中的弹窗，调整为独立路由页面 `/profile`
2. **数据展示升级**：新增 GitHub 风格的刷题绿墙，用于展示用户最近一年的每日提交与解题活跃度

升级后的入口关系如下：

```text
workspace 右上角用户下拉菜单
  ↓
点击「个人信息」
  ↓
router.push('/profile')
  ↓
进入独立 Profile 页面
  ├─ 资料编辑卡片
  └─ Contribution Wall 刷题绿墙
```

这样调整后，个人信息页不再受弹窗尺寸限制，更适合承载统计信息、成长记录和后续扩展功能。

---

## 10. 前端页面升级说明

### 10.1 新增页面与组件

本次新增/调整的前端文件如下：

| 文件                                   | 改动说明                                                      |
| -------------------------------------- | ------------------------------------------------------------- |
| `src/app/profile/page.tsx`             | 新增独立个人信息页，负责资料编辑、活动数据加载和页面布局      |
| `src/components/ContributionWall.tsx`  | 新增 GitHub 风格绿墙组件，展示最近一年每日刷题情况            |
| `src/lib/api.ts`                       | 新增 `DailyActivity` 类型与 `getDailyActivity()` 请求方法     |
| `src/app/workspace/page.tsx`           | 将原先打开 `ProfileModal` 的逻辑改为跳转到 `/profile`         |
| `src/lib/i18n/zh.ts` / `src/lib/i18n/en.ts` | 新增绿墙统计项与页面导航相关文案                        |

### 10.2 `/profile` 页面职责

`src/app/profile/page.tsx` 的核心职责：

- 校验登录状态，未登录时跳转 `/login`
- 复用原有个人资料编辑能力（用户名、邮箱、头像、简介）
- 展示用户角色、注册时间、当前头像
- 拉取最近一年的每日活动数据并传给 `ContributionWall`
- 提供“返回工作台”按钮，形成清晰导航闭环

页面结构示意：

```text
/profile
├─ 顶部 Banner
│   ├─ 返回工作台按钮
│   ├─ 标题 / 副标题
├─ Profile Card
│   ├─ 头像上传
│   ├─ 预设头像选择
│   ├─ 用户角色 / 注册时间
│   └─ 资料编辑表单
└─ Contribution Wall Card
  ├─ 总提交数
  ├─ 总解题数
  ├─ 最长连续天数
  ├─ 当前连续天数
  └─ 最近一年热力图
```

### 10.3 `workspace` 导航逻辑调整

改造前：

```tsx
onClick: () => setProfileModalOpen(true)
```

改造后：

```tsx
onClick: () => router.push('/profile')
```

这意味着：

- `workspace` 页面不再直接承载个人信息弹窗的编辑逻辑
- 用户从任意时刻都可以通过路由回到个人信息页
- 后续如需增加“成就”“勋章”“刷题日历”“学习画像”等模块，可直接在该页面扩展

---

## 11. Contribution Wall（刷题绿墙）设计

### 11.1 目标

绿墙用于可视化用户最近一年的刷题活跃度，参考 GitHub Contribution Graph 的交互方式，但展示的是**每日刷题行为**而非代码提交。

### 11.2 数据口径

每个日期单元格展示两类统计：

- `submissionCount`：当天提交总次数
- `solvedCount`：当天成功解决的题目数（按题目去重）

热力图颜色等级依据 `solvedCount` 计算：

| 等级 | 条件                 | 颜色       | 说明     |
| ---- | -------------------- | ---------- | -------- |
| 0    | `count = 0`          | `#ebedf0`  | 无活动   |
| 1    | `count <= 1`         | `#9be9a8`  | 少量活动 |
| 2    | `count <= 3`         | `#40c463`  | 中等活动 |
| 3    | `count <= 6`         | `#30a14e`  | 较高活动 |
| 4    | `count > 6`          | `#216e39`  | 高强度活动 |

### 11.3 展示维度

`ContributionWall` 组件会基于最近一年数据计算：

- **总提交数**：过去一年的所有提交次数汇总
- **总解题数**：过去一年的所有 `solvedCount` 汇总
- **最长连续天数**：一年内最长的连续刷题区间
- **当前连续天数**：从今天往前连续有刷题记录的天数

### 11.4 组件渲染策略

前端使用 SVG 绘制热力图：

- 横向按周排列，共约 53 列
- 纵向按星期排列，共 7 行
- 顶部显示月份标签
- 左侧显示星期标签（中英文随语言切换）
- 每个方块悬浮时通过 `Tooltip` 展示当天详情

悬浮提示示例：

```text
2026-03-11: 2 solved, 5 submissions
```

中文下会显示：

```text
2026-03-11: 2 题解决, 5 次提交
```

---

## 12. 后端支撑改动（绿墙数据）

虽然本文档位于前端仓库，但为了便于联调和后续维护，下面同步记录本次绿墙能力依赖的后端改造。

### 12.1 数据库迁移

新增迁移文件：`programming-backend/database/migrations/002_add_daily_activity.sql`

核心表结构：

```sql
CREATE TABLE IF NOT EXISTS daily_activity (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  activity_date DATE NOT NULL,
  submission_count INT NOT NULL DEFAULT 0,
  solved_count INT NOT NULL DEFAULT 0,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uq_user_date (user_id, activity_date),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

设计意图：

- `user_id + activity_date` 唯一，确保“一个用户一天只有一条聚合记录”
- `submission_count` 直接记录当天总提交次数
- `solved_count` 记录当天成功解决的题目数，供绿墙颜色和连续天数统计使用
- 附加 `(user_id, activity_date)` 索引，优化按用户+时间范围查询

### 12.2 提交写入时机

后端在创建提交记录后，会同步更新 `daily_activity` 聚合表：

```go
if err = r.updateDailyActivity(tx, submission.UserID); err != nil {
  return nil, err
}
```

`updateDailyActivity()` 的逻辑是：

1. 以 `CURDATE()` 为维度定位当天记录
2. 统计当前用户当天在 `submissions` 表中的总提交数
3. 统计当前用户当天 `Accepted` 的去重题目数
4. 使用 `INSERT ... ON DUPLICATE KEY UPDATE` 做幂等写入

这样可以保证：

- 无需前端自行聚合历史提交
- 每次新提交都会自动刷新当天活动数据
- 热力图查询时只需读取轻量聚合表，性能更稳定

### 12.3 新增 API

路由：

```go
stats.GET("/user/:userId/activity", submissionHandler.GetDailyActivity)
```

接口定义：

### `GET /api/stats/user/:userId/activity`

**Query 参数：**

| 参数    | 必填 | 说明                             |
| ------- | ---- | -------------------------------- |
| `start` | ❌   | 起始日期，格式 `YYYY-MM-DD`      |
| `end`   | ❌   | 结束日期，格式 `YYYY-MM-DD`      |

当 `start` / `end` 未传时，后端默认返回最近 365 天的数据。

**成功响应：**

```json
[
  {
  "userId": 1,
  "date": "2026-03-11",
  "submissionCount": 5,
  "solvedCount": 2
  }
]
```

### 12.4 前端调用方式

`src/lib/api.ts` 中新增：

```typescript
async getDailyActivity(userId?: number, start?: string, end?: string) {
  const uid = userId || getCurrentUserId() || 1
  const params = new URLSearchParams()
  if (start) params.set('start', start)
  if (end) params.set('end', end)
  const query = params.toString() ? `?${params.toString()}` : ''
  return this.request<DailyActivity[]>(`/stats/user/${uid}/activity${query}`)
}
```

`/profile` 页面在加载用户信息后，调用该接口获取活动数据：

```tsx
const data = await api.getDailyActivity(user.id)
setActivities(data || [])
```

---

## 13. 新增国际化文案

在原有个人资料翻译的基础上，本次又增加了与独立页面和绿墙相关的 key：

| Key                              | 中文       | English                  |
| -------------------------------- | ---------- | ------------------------ |
| `profile.backToWorkspace`        | 返回工作台 | Back to Workspace        |
| `profile.editProfile`            | 编辑资料   | Edit Profile             |
| `profile.wall.title`             | 刷题记录   | Contribution Activity    |
| `profile.wall.totalSubmissions`  | 年度提交   | Submissions this year    |
| `profile.wall.totalSolved`       | 年度解题   | Problems solved          |
| `profile.wall.maxStreak`         | 最长连续   | Longest streak           |
| `profile.wall.currentStreak`     | 当前连续   | Current streak           |
| `profile.wall.days`              | 天         | days                     |
| `profile.wall.less`              | 少         | Less                     |
| `profile.wall.more`              | 多         | More                     |

---

## 14. 当前版本总结

截至本次改造，个人信息模块已经从“仅支持弹窗编辑资料”升级为“可独立访问的用户画像页”，具备以下能力：

- 支持通过 `/profile` 独立访问
- 支持头像上传、预设头像、自定义头像 URL
- 支持用户名、邮箱、个人简介编辑
- 支持展示角色与注册时间
- 支持展示最近一年的刷题绿墙
- 支持展示年度提交、年度解题、最长连续、当前连续等活跃度指标

后续如果要继续扩展个人中心，可优先考虑：

- 成就/徽章系统
- 题目难度偏好分析
- 最近通过题目列表
- 学习曲线与月度趋势图
- 班级内个人排名变化

# 教师后台管理系统

## 需求概述

为教师角色提供一个专门的后台管理系统，用于查看自己所带班级的学生练习完成情况。

## 功能详情

### 1. 后台管理入口

- 在首页（`/`）和工作台（`/workspace`）的 Header 中，为教师（instructor）和管理员（admin）角色添加"后台管理"按钮
- 点击后跳转至 `/manager` 路由
- 学生角色不会看到此入口

### 2. 路由权限控制

- `/manager` 路由及其所有子路由仅限教师（instructor）和管理员（admin）角色访问
- 未登录或学生角色访问时，展示 403 无权限页面
- 权限校验在 `/manager/layout.tsx` 中统一处理

### 3. 后台管理布局（`/manager`）

采用 ToB 风格的后台布局：

- **侧边栏（Sider）**：可折叠，包含导航菜单
  - 我的班级（`/manager/my-classes`）
- **顶部导航栏（Header）**：
  - 侧边栏折叠按钮
  - 返回工作台按钮
  - 用户信息下拉菜单（个人信息、退出登录）
- **主内容区（Content）**：展示当前选中菜单对应的页面内容

### 4. 我的班级页面（`/manager/my-classes`）

#### 班级列表

- **教师角色**：仅展示该教师所带的班级
- **管理员角色**：展示所有班级
- 支持按班级名称搜索
- 每个班级卡片展示：班级名称、描述、学生数量、实验数量

#### 班级详情（点击展开）

- **概览统计**：学生总数、实验总数、平均完成率、平均通过率
- **实验列表**：展示该班级关联的所有实验，包含状态（进行中/已结束）、题目数量、起止时间
- **学生进度表**：
  - 学生姓名（含头像）
  - 完成进度（进度条）
  - 提交次数
  - 通过次数
  - 通过率
  - 最近提交时间
  - 支持排序

## 后端 API 需求

前端调用以下 API 接口（需后端实现）：

| 接口                                     | 方法 | 说明                                 |
| ---------------------------------------- | ---- | ------------------------------------ |
| `/api/manager/my-classes?search=keyword` | GET  | 获取当前教师的班级列表（支持搜索）   |
| `/api/manager/classes?search=keyword`    | GET  | 获取所有班级列表（管理员，支持搜索） |
| `/api/manager/classes/:id`               | GET  | 获取班级详情（含实验列表和学生进度） |

### 接口返回数据结构

**班级列表项 `ClassInfo`：**

```json
{
  "id": 1,
  "name": "2024级计算机科学1班",
  "description": "计算机科学与技术专业2024级1班",
  "teacherId": 1,
  "teacherName": "teacher_wang",
  "studentCount": 7,
  "experimentCount": 4,
  "createdAt": "2026-02-03T10:25:00+08:00"
}
```

**班级详情 `ClassDetailData`：**

```json
{
  "classInfo": {
    /* ClassInfo */
  },
  "experiments": [
    {
      "id": 1,
      "title": "C语言基础练习",
      "description": "包含数组、循环、条件判断等基础题目",
      "startTime": "2024-09-01T08:00:00",
      "endTime": "2024-09-15T23:59:59",
      "isActive": true,
      "problemCount": 5
    }
  ],
  "students": [
    {
      "userId": 11,
      "username": "student_001",
      "avatar": "",
      "totalProblems": 20,
      "solvedProblems": 15,
      "totalSubmissions": 45,
      "acceptedSubmissions": 30,
      "lastSubmissionAt": "2026-03-01T14:30:00"
    }
  ]
}
```

## 涉及的文件变更

### 新增文件

| 文件路径                              | 说明                                           |
| ------------------------------------- | ---------------------------------------------- |
| `src/app/manager/layout.tsx`          | 后台管理布局组件（含侧边栏、Header、权限校验） |
| `src/app/manager/page.tsx`            | 后台管理入口页（自动重定向到我的班级）         |
| `src/app/manager/my-classes/page.tsx` | 我的班级页面                                   |

### 修改文件

| 文件路径                     | 变更内容                                                          |
| ---------------------------- | ----------------------------------------------------------------- |
| `src/app/page.tsx`           | 首页 Header 添加"后台管理"按钮（教师/管理员可见）                 |
| `src/app/workspace/page.tsx` | 工作台 Header 添加"后台管理"按钮（教师/管理员可见）               |
| `src/lib/api.ts`             | 将 `request` 方法从 `private` 改为 `public`，以便外部调用管理接口 |
| `src/lib/i18n/zh.ts`         | 添加后台管理相关中文翻译                                          |
| `src/lib/i18n/en.ts`         | 添加后台管理相关英文翻译                                          |

## 数据库表关系

```
users (用户表)
  ↓ teacher_id
classes (班级表) ← class_students (学生-班级关联) → users
  ↓ class_id
class_experiments (班级-实验关联) → experiments (实验表)
                                       ↓ experiment_id
                                   experiment_problems → problems (题目表)
                                                            ↓
                                                        submissions (提交记录)
```

## 日期

- 需求创建：2026-03-03

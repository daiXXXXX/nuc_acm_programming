# 每日一题推荐功能说明

## 变更背景

用户登录后进入题目列表页时，需要在全部题目上方看到一个“每日一题”入口。该入口由系统根据当前用户最近一段时间通过的题目进行推荐，并且必须排除用户已经 Accepted 的题目。

## 推荐规则

1. 后端读取当前用户最近 20 道已 Accepted 的去重题目。
2. 越近通过的题目权重越高，用这些题目的标签和难度生成用户近期刷题画像。
3. 候选题目只从当前用户从未 Accepted 的题目中选择。
4. 候选题得分由两部分组成：
   - 标签命中近期 Accepted 题目标签时加权加分。
   - 难度与近期 Accepted 题目越接近，加分越高。
5. 当用户暂无 Accepted 历史时，使用冷启动策略，优先推荐未通过的简单题。
6. 同分题目使用日期参与稳定打散，保证同一天内推荐结果稳定，同时具备“每日”变化能力。

## 后端变更文件

- `programming-backend/internal/models/models.go`
  - 新增 `DailyProblemRecommendation` 响应模型，返回推荐题目、推荐原因和命中的标签。

- `programming-backend/internal/database/problem.go`
  - 新增 `GetDailyRecommendation`。
  - 新增近期 Accepted 题目画像构建逻辑。
  - 新增未 AC 候选题查询逻辑。
  - 新增标签/难度加权评分、冷启动评分和每日稳定打散逻辑。

- `programming-backend/internal/handlers/problem.go`
  - 新增 `GetDailyRecommendation` handler。
  - 接口要求当前请求中存在登录用户身份，否则返回 401。

- `programming-backend/cmd/server/main.go`
  - 新增 `GET /api/problems/daily-recommendation` 路由。
  - 路由放在 `GET /api/problems/:id` 前，避免被动态题目 ID 路由吞掉。

- `programming-backend/internal/database/problem_recommendation_test.go`
  - 覆盖标签与难度命中优先、冷启动偏好简单题、每日打散随日期变化三个核心规则。

## 前端变更文件

- `programming-frontend/src/lib/api.ts`
  - 新增 `getDailyProblemRecommendation` API 方法。
  - 新增 `DailyProblemRecommendation` 类型。

- `programming-frontend/src/hooks/use-daily-problem-recommendation.ts`
  - 新增每日一题推荐 hook。
  - 仅登录用户请求推荐。
  - Accepted 题目集合变化后自动刷新推荐，避免刚通过的题目继续展示在入口中。

- `programming-frontend/src/hooks/index.ts`
  - 导出新增 hook。

- `programming-frontend/src/components/ProblemList.tsx`
  - 在普通题目列表前渲染“每日一题”卡片。
  - 推荐题目若已在本地 Accepted 集合中，则不展示。
  - 推荐题目从普通列表中临时去重，避免同一题在顶部和列表中重复出现。

- `programming-frontend/src/app/workspace/page.tsx`
  - 桌面端题目列表接入每日一题推荐。

- `programming-frontend/src/app/workspace-mobile/page.tsx`
  - 移动端题目列表接入每日一题推荐。

- `programming-frontend/src/lib/i18n/zh.ts`
  - 新增每日一题相关中文文案。

- `programming-frontend/src/lib/i18n/en.ts`
  - 新增每日一题相关英文文案。

## 验证结果

- 后端：`go test ./...` 通过。
- 前端类型检查：`node node_modules/typescript/bin/tsc --noEmit` 通过。
- 前端本次改动文件 lint：`node node_modules/next/dist/bin/next lint --file ...` 通过。

## 备注

直接执行 `npm run lint` 时，本机 npm 全局入口缺失，报错为找不到 `npm-cli.js`，因此已改用项目本地依赖中的 Next 和 TypeScript 入口完成验证。直接执行全量 `node node_modules/next/dist/bin/next lint` 仍会命中项目既有 lint 存量问题，主要集中在未使用变量和少量 `any` 类型；本次改动涉及的文件已单独通过 lint。

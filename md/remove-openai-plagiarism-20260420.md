# 查重功能精简：移除 OpenAI，固定启发式阈值

**日期**: 2026-04-20  
**类型**: 功能简化 / 架构精简

---

## 改动原因

1. **删除 AI 依赖**：不再依赖 OpenAI API 做二次判定，降低外部依赖和成本。查重只在本地做一遍启发式筛选即可。
2. **固定阈值**：启发式阈值固定为 `0.55`，不再暴露给教师调整，简化用户操作流程——选择题目后直接点击查重即可。
3. **超过阈值的 pair 直接返回**：作为可疑对展示给教师，由教师人工复核后决定是否标记作弊。

---

## 改动文件清单

### 后端 (programming-backend)

| 文件                                  | 改动                                                                                                                                                                                                                                                 |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `internal/ai/openai.go`               | **已删除** — 整个 OpenAI 客户端文件移除                                                                                                                                                                                                              |
| `internal/plagiarism/service.go`      | 移除 AI client 依赖；`NewService()` 不再接收参数；`CheckClassProblem` 只做启发式筛选并直接返回可疑对；固定阈值 0.55；删除 `buildAnalysisRequest`、`toAIPairStudent`、`trimForAI`、`resolveMinHeuristic`、`normalizeList`、`clamp01` 等不再需要的函数 |
| `internal/plagiarism/service_test.go` | 移除 `fakeAnalyzer` mock；测试直接验证启发式筛选结果                                                                                                                                                                                                 |
| `internal/models/plagiarism.go`       | `PlagiarismPairResult` 移除 `AIConfidence` 字段                                                                                                                                                                                                      |
| `internal/handlers/manager.go`        | 移除 `errors` import；移除 `ErrAnalyzerNotConfigured` 错误处理分支                                                                                                                                                                                   |
| `internal/config/config.go`           | 移除 `OpenAIConfig` 结构体和相关环境变量加载                                                                                                                                                                                                         |
| `cmd/server/main.go`                  | 移除 `ai` import；`plagiarism.NewService()` 不再传入 OpenAI client                                                                                                                                                                                   |

### 前端 (programming-frontend)

| 文件                                  | 改动                                                                                                                                       |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `src/app/manager/my-classes/page.tsx` | 移除阈值输入控件和 AI 置信度标签；`DEFAULT_PLAGIARISM_FORM` 不再包含 `minHeuristicScore`；请求体不再发送该字段                             |
| `src/lib/api.ts`                      | `PlagiarismCheckRequest` 移除 `minHeuristicScore`；`PlagiarismPairResult` 移除 `aiConfidence`                                              |
| `src/lib/i18n/zh.ts`                  | 移除 `plagiarismThreshold`、`plagiarismAIConfidence` 键；更新 `plagiarismHint`、`plagiarismMaxCandidates`、`plagiarismHeuristicScore` 文案 |
| `src/lib/i18n/en.ts`                  | 同上英文对应项                                                                                                                             |

---

## 接口变更

### POST `/api/manager/classes/:id/plagiarism-check`

**请求体** (简化后):

```json
{
  "problemId": 7,
  "acceptedOnly": false,
  "maxCandidates": 5
}
```

`minHeuristicScore` 不再接受，后端固定使用 0.55。

**响应体** (简化后):

- 移除了 `results[].aiConfidence` 字段
- `results[].verdict` 固定为 `"suspicious"`（通过阈值即视为可疑）
- `results[].riskLevel` 由启发式分数自动决定（≥0.85 high, ≥0.70 medium, 其余 low）

---

## 决策逻辑

```
教师选择题目 → 点击查重
  → 后端获取班级内该题每位学生的代表提交（每人一份）
  → 本地启发式筛选：
      1. 标准化代码（去注释/字符串/数字、折叠标识符为 "id"）
      2. Shingle-based Jaccard 相似度（shingle size = 5）
      3. 筛选分数 ≥ 0.55 的 pair
      4. 按分数降序取前 maxCandidates 对
  → 直接作为可疑对返回给教师
  → 教师人工复核后可点击"标记为作弊"
```

---

## 被删除的能力

- OpenAI Responses API 集成（结构化输出 JSON Schema）
- AI 置信度、AI verdict、AI evidence/differences
- 用户可调的启发式阈值

这些能力如未来需要可从 git 历史中恢复。

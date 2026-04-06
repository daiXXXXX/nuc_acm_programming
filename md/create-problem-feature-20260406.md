# 管理员/教师自主录题功能开发记录

## 记录时间

- 日期：2026-04-06

## 需求背景

为平台增加“管理员和教师可自主创建新题目”的完整录题能力，覆盖以下流程：

1. 在题目列表页增加“我要出题”入口，且仅管理员、教师可见。
2. 新增 `workspace/createProblem` 录题页面，并将录题流程拆分为三步。
3. 支持录入题面信息、样例、隐藏测试用例、标准程序。
4. 支持标准程序在正式录题前先跑通测试数据。
5. 录题成功后将题目、标签、样例、测试用例、标准程序等信息落库。

---

## 本次改动的文件清单

### 前端

1. `programming-frontend/src/app/workspace/page.tsx`
2. `programming-frontend/src/app/workspace/page.module.css`
3. `programming-frontend/src/app/workspace-mobile/page.tsx`
4. `programming-frontend/src/app/workspace/createProblem/page.tsx`
5. `programming-frontend/src/lib/api.ts`

### 后端

6. `programming-backend/cmd/server/main.go`
7. `programming-backend/internal/handlers/problem.go`
8. `programming-backend/internal/database/problem.go`
9. `programming-backend/internal/models/models.go`
10. `programming-backend/database/schema.sql`
11. `programming-backend/database/migrations/005_add_problem_reference_solutions.sql`

---

## 具体改动内容

### 1. 题目列表页增加“我要出题”按钮

**涉及文件：**

- `programming-frontend/src/app/workspace/page.tsx`
- `programming-frontend/src/app/workspace/page.module.css`
- `programming-frontend/src/app/workspace-mobile/page.tsx`

**改动内容：**

- 在题目列表搜索框右侧增加“我要出题”按钮。
- 对按钮进行了权限控制，仅当当前用户角色为 `instructor` 或 `admin` 时显示。
- 桌面端和移动端均补充了对应入口。
- 补充了按钮与搜索区域的布局样式，保证界面结构清晰。

**改动原因：**

- 满足“管理员和教师账号可以自己创建新题目”的入口要求。
- 保证学生账号不可见、不可误入录题流程。

---

### 2. 新增三步式录题页面

**涉及文件：**

- `programming-frontend/src/app/workspace/createProblem/page.tsx`

**改动内容：**

- 新增 `workspace/createProblem` 页面。
- 页面按需求拆分为三个步骤：
  1. **题目描述**：填写标题、题目描述、输入格式、输出格式、约束条件、标签、难度。
  2. **测试用例**：区分公开样例与隐藏测试用例，支持手动新增和批量导入 `.in/.out` 文件。
  3. **标准程序**：录入标准程序语言和代码，并在正式录题前执行校验。
- 页面增加了权限保护：若当前用户不是管理员或教师，则自动跳回工作台。
- 样例与隐藏测试最终会被合并为统一测试数据，其中样例自动标记为 `isSample=true`。
- 已在关键逻辑处补充注释，便于后续维护。

**改动原因：**

- 满足完整三步录题交互。
- 明确区分“用户可见样例”和“实际评测测试点”。
- 将标准程序校验纳入正式录题流程，避免测试数据与标准答案不匹配。

---

### 3. 支持批量上传 `.in/.out` 文件

**涉及文件：**

- `programming-frontend/src/app/workspace/createProblem/page.tsx`

**改动内容：**

- 增加批量导入逻辑，要求输入文件以 `.in` 结尾、输出文件以 `.out` 结尾。
- 采用“同名文件配对”的方式自动组装输入输出，例如：
  - `1.in` 对应 `1.out`
  - `sampleA.in` 对应 `sampleA.out`
- 若存在未配对文件，会直接提示错误，防止脏数据进入录题流程。
- 该能力同时支持样例和隐藏测试用例两个区域。

**改动原因：**

- 满足需求中“可以批量上传测试数据”的要求。
- 降低出题者手工录入大量测试数据的成本。

---

### 4. 增加标准程序校验接口

**涉及文件：**

- `programming-backend/cmd/server/main.go`
- `programming-backend/internal/handlers/problem.go`
- `programming-backend/internal/models/models.go`
- `programming-frontend/src/lib/api.ts`

**改动内容：**

- 后端新增 `POST /api/problems/validate` 接口。
- 该接口仅允许教师/管理员访问。
- 前端新增 `validateProblemDraft()` 调用，用于在正式录题前校验标准程序。
- 新增标准程序请求/响应模型：
  - `ProblemStandardProgram`
  - `ValidateProblemRequest`
  - `ProblemValidationResponse`
- 校验逻辑会执行标准程序并跑完整测试集，返回：
  - 是否全部通过
  - 总得分
  - 每个测试点的详细结果

**改动原因：**

- 满足“第三步测试标准程序，通过测试用例之后才可以完成录题”的要求。
- 将“录题前发现问题”前置，而不是等题目发布后才暴露数据错误。

---

### 5. 正式录题时强制校验标准程序

**涉及文件：**

- `programming-backend/internal/handlers/problem.go`

**改动内容：**

- 在 `CreateProblem()` 中增加录题请求校验：
  - 标题、描述、输入格式、输出格式、约束必须完整。
  - 至少存在一个样例。
  - 至少存在一个测试用例。
  - 至少存在一个公开样例测试点。
  - 必须提供标准程序。
- 在正式写库前，后端会再次执行标准程序校验。
- 若标准程序未通过全部测试数据，则直接返回错误，不允许录题成功。
- 更新题目时若提交了新的标准程序，也会触发同样校验。
- 已补充注释，说明预校验与正式校验的职责。

**改动原因：**

- 防止前端绕过校验直接调用创建接口。
- 确保真正落库的数据经过后端兜底验证。

---

### 6. 题目相关数据写入数据库

**涉及文件：**

- `programming-backend/internal/database/problem.go`
- `programming-backend/internal/models/models.go`

**改动内容：**

- 录题成功后写入：
  - `problems`：题目基本信息
  - `problem_tags`：题目标签
  - `problem_examples`：公开样例
  - `test_cases`：全部测试点（含样例与隐藏测试）
- 扩展了题目创建请求结构，允许携带标准程序信息。
- 在仓库层新增标准程序的写入逻辑，并在更新题目时支持替换留档版本。
- 关键落库逻辑均在事务中执行，保证写库原子性。

**改动原因：**

- 满足“完成录题之后，要把题目的相关信息都放到相关数据库里”的要求。
- 使用事务保证题目、标签、测试数据、标准程序的一致性。

---

### 7. 新增标准程序留档表

**涉及文件：**

- `programming-backend/database/schema.sql`
- `programming-backend/database/migrations/005_add_problem_reference_solutions.sql`

**改动内容：**

- 新增 `problem_reference_solutions` 表。
- 表中存储：
  - `problem_id`
  - `language`
  - `source_code`
  - `created_by`
  - 创建/更新时间
- 对 `problem_id` 设置唯一约束，保证一个题目只保留一份当前标准程序留档。
- 已同步补充索引与迁移脚本。
- 数据库中已执行建表与索引创建。

**改动原因：**

- 标准程序不仅用于录题前校验，也应作为题目维护资料长期保存。
- 后续若要支持题目复检、数据重跑、管理端查看标准答案，此表可以直接复用。

---

### 8. 修复录题提交时基础字段丢失问题

**涉及文件：**

- `programming-frontend/src/app/workspace/createProblem/page.tsx`

**问题现象：**

- 点击“完成录题”后，请求 `/api/problems` 时后端报：
  - `Title required`
  - `Difficulty required`
  - `Description required`
  - `InputFormat required`
  - `OutputFormat required`
  - `Constraints required`

**问题原因：**

- 录题页是多步骤表单。
- 切换到第三步后，第一步的表单项已经卸载。
- 直接调用 `form.validateFields()` 时，只能拿到当前仍注册的字段，导致第一步基础信息没有进入最终 payload。

**修复方式：**

- 增加基础字段名常量。
- 在提交前显式校验第一步字段。
- 使用 `form.getFieldsValue(true)` 读取完整表单仓库数据。
- 在构造提交 payload 时显式回填基础字段。
- 已在代码中补充注释说明该问题产生的原因。

**修复原因：**

- 保证三步录题数据在最终提交时能完整合并。
- 避免后端因缺少基础字段而返回 400。

---

### 9. 路由 404 排查结果

**涉及文件：**

- `programming-backend/cmd/server/main.go`

**排查结论：**

- `POST /api/problems/validate` 路由代码已正确注册。
- 前端请求路径也正确。
- 出现 404 的原因是当时 `8080` 端口上运行的后端进程不是最新版本，未加载新路由。
- 使用当前工作区代码临时启动服务后，访问该接口返回的是 `401`，说明新代码中的路由本身是存在的。

**记录原因：**

- 方便后续排查同类问题时快速判断是“代码未注册”还是“服务未更新”。

---

## 本次改动解决的问题总结

1. 管理员/教师可以从工作台直接进入录题页。
2. 录题页已完整支持三步式出题流程。
3. 样例与隐藏测试用例可手动维护，也可批量导入 `.in/.out` 文件。
4. 标准程序在录题前可校验，并且后端会进行兜底校验。
5. 题目、样例、标签、测试用例、标准程序均可成功持久化。
6. 修复了多步骤表单导致基础字段丢失的问题。
7. 补充了标准程序数据库表与迁移脚本，便于后续维护和扩展。

---

## 后续建议

1. 为录题流程补充自动化测试，重点覆盖：
   - 三步表单提交
   - `.in/.out` 文件配对导入
   - 标准程序校验失败拦截
   - 录题成功后的数据库落库校验
2. 在管理端增加“查看/编辑已录入题目”的能力。
3. 增加标准程序语言扩展，例如 `C++`、`Java`、`Python`。
4. 为隐藏测试用例增加更明确的数量、规模和边界覆盖提示。
5. 可考虑增加题目草稿保存能力，避免出题中途丢失内容。

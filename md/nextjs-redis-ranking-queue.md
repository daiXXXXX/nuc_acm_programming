# Next.js 全栈服务 Redis 优化说明

> 日期：2026-04-01  
> 范围：programming-frontend 中的 Next.js 服务端接口  
> 目标：为排行榜接口加入 Redis 缓存，为提交评测接入 Redis 队列，并保留无 Redis 时的自动降级能力

## 一、改动背景

当前 `programming-frontend` 已经承担了全栈职责，接口统一从 Next.js Route Handler 进入。但服务端仍存在两个性能瓶颈：

1. **排行榜接口每次都直查 MySQL**：`/api/ranking/total` 与 `/api/ranking/today` 都会执行排序查询，在高频访问下会给数据库带来重复压力。
2. **提交评测仍是同步执行**：`POST /api/submissions` 需要在同一个 HTTP 请求里完成评测和写库，耗时操作会拖慢响应，且并发时容易堆积请求。

因此，本次改动把 Redis 引入到了 Next.js 服务端：

- 排行榜：使用 Redis 做短 TTL 缓存
- 提交评测：使用 Redis List 做异步评测队列
- Redis 不可用：自动回退为原来的数据库直查 + 同步评测模式

## 二、改动文件清单

### programming-frontend

1. `package.json`
   - 新增 `redis` 依赖
2. `package-lock.json`
   - 锁定 Redis 依赖版本
3. `.env.example`
   - 新增 Redis 与评测 worker 的环境变量说明
4. `src/server/oj/redis.ts`
   - 新增 Redis 单例、缓存读写、模糊删 key、队列入队/出队封装
5. `src/server/oj/submission-queue.ts`
   - 新增评测队列消费者启动逻辑与后台循环
6. `src/server/oj/services/ranking-service.ts`
   - 排行榜接口增加 Redis 缓存读取与写入
7. `src/server/oj/services/submission-service.ts`
   - 提交接口增加异步入队、同步降级、结果回写、排行榜缓存失效逻辑

### programming-backend

1. `md/nextjs-redis-ranking-queue.md`
   - 本次需求的改动说明文档

## 三、改动内容详述

### 1. Redis 基础封装：`src/server/oj/redis.ts`

新增了一个专门给 Next.js 服务端使用的 Redis 工具模块，职责包括：

- 通过全局变量复用 Redis 客户端，避免热更新重复连 Redis
- 连接失败后进入冷却期，避免每个请求都重复打连接失败日志
- 提供缓存读写能力：
  - `getCachedJson()`
  - `setCachedJson()`
  - `deleteCachedKey()`
  - `deleteCachedKeysByPattern()`
- 提供队列能力：
  - `enqueueJsonTask()`
  - `dequeueJsonTask()`
- 为 worker 提供独立阻塞连接：`createRedisWorkerClient()`

### 2. 排行榜缓存：`src/server/oj/services/ranking-service.ts`

对两个排行榜接口增加了 Redis 短期缓存：

- `ranking:total:{limit}`：总榜缓存，TTL 为 **30 秒**
- `ranking:today:{limit}`：今日榜缓存，TTL 为 **15 秒**

请求流程如下：

1. 先从 Redis 读取对应榜单缓存
2. 如果命中，直接返回缓存结果
3. 如果未命中，再查 MySQL
4. 查库后把序列化后的榜单写回 Redis

这样可以显著减少排行榜高频访问下的重复排序查询。

### 3. 提交异步评测：`src/server/oj/services/submission-service.ts`

`POST /api/submissions` 新增了两条路径：

#### Redis 可用时

1. 创建一条 `Pending` 状态的提交记录
2. 构造评测任务对象 `JudgeQueueTask`
3. 将任务推入 Redis 队列 `queue:judge`
4. 立即返回 `202`，让前端知道当前是异步评测模式

#### Redis 不可用时

- 自动保留原来的同步评测流程，继续返回完整评测结果

#### 入队失败时

- 为避免已经写入的 `Pending` 提交长期卡住，系统会立刻降级为当前进程内同步评测，并把同一条提交补写成最终状态

### 4. 评测消费者：`src/server/oj/submission-queue.ts`

新增了后台评测 worker 的启动与消费逻辑：

- 使用全局标记保证每个 Node 进程只启动一组消费者
- 并发度可通过 `JUDGE_WORKER_CONCURRENCY` 配置
- 每个 worker 使用独立 Redis 阻塞连接执行 `BRPOP`
- 收到任务后调用提交服务注册的处理器执行真实评测
- worker 报错后会自动短暂等待并继续重连

### 5. 提交结果回写与缓存失效

在 `submission-service.ts` 中把结果落库逻辑统一抽成了共享方法，保证：

- 同步评测与异步评测最终都走同一套更新逻辑
- 最终状态、分数、测试结果、用户统计、每日活跃度都能正确更新
- 当提交结果为 `Accepted` 时，会删除 `ranking:*` 相关缓存，确保下一次榜单读取可以看到最新排名

## 四、为什么这样设计

### 1. 排行榜适合短 TTL 缓存

排行榜是公开接口、访问频率高、变化频率相对可控，适合用短 TTL 缓存来降低数据库压力。

### 2. 提交评测适合队列化

评测属于典型的耗时任务，不应该长期占用 HTTP 请求。改成 Redis 队列后：

- 提交接口响应更快
- 后台评测更容易扩展并发数
- 服务在高并发提交时更稳定

### 3. 保留降级路径保证可用性

Redis 并不是强依赖。本次实现中：

- Redis 连不上：继续可用
- 入队失败：继续可用
- Worker 异常：提交不会因为接口层的缓存/队列异常而直接丢失

## 五、环境变量说明

本次新增的环境变量如下：

```dotenv
REDIS_ENABLED=true
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
REDIS_PREFIX=oj-next
JUDGE_WORKER_CONCURRENCY=1
```

说明：

- `REDIS_ENABLED=false` 时，可显式关闭 Redis 功能
- `REDIS_PREFIX` 用于区分不同环境或项目实例的 key 前缀
- `JUDGE_WORKER_CONCURRENCY` 控制单个 Next.js 进程内的评测消费者数量

## 六、缓存与队列 Key 设计

### 缓存 Key

- `oj-next:ranking:total:{limit}`
- `oj-next:ranking:today:{limit}`

### 队列 Key

- `oj-next:queue:judge`

队列元素内容为 JSON，例如：

```json
{
  "submissionId": 101,
  "problemId": 12,
  "userId": 3,
  "code": "function processInput(input) { return input }",
  "language": "JavaScript"
}
```

## 七、验证情况

本次改动完成后，已执行：

- `npx tsc --noEmit`

结果：**通过**。

说明：

- `npm run lint` 在当前项目中会触发 Next.js 的首次 ESLint 初始化向导，因此未作为最终校验依据
- 向导生成的 `.eslintrc.json` 已移除，未将其保留到代码库中

## 八、后续建议

1. **前端增加明确的 Pending 轮询兜底**
   - 当前若没有 WebSocket 结果推送，建议在提交返回 `Pending` 后轮询 `GET /api/submissions/:id`
2. **把 worker 独立成单独进程**
   - 当前实现适合自托管 Node 模式；如果后续部署到更偏 serverless 的环境，建议拆出独立 worker 服务
3. **补充运行监控**
   - 可继续增加队列长度、任务失败次数、评测耗时等指标
4. **扩展缓存场景**
   - 后续可继续把题目详情、题目列表、热门题解等热点读接口接入 Redis 缓存

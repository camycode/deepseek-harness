# deepseek-harness fork 说明

这是 `deepseek-harness` 的本地 fork，在 `master` 上包含三组上游增量补丁。三组补丁都保持了默认行为兼容：不启用对应能力时，现有宿主行为不变。

> 当前本地 `master`：`HEAD`（merge base `ca3414084f` + 本文档提交）
> 基线：`origin/master` / `9b4ad3f403`（DSH `0.2.1-rc.2` 线）
> 本目录只记录 **deepseek-harness 上游侧** 的修改；Docker 插件实现位于同级仓库 `dsh-docker`。

## 补丁总览

| 补丁 | 文档 | 上游改动 | 状态 |
|---|---|---|---|
| per-user Peer 准入与事件路由 | [`2026-10-23-01-peer-admission-and-event-routing.md`](./2026-10-23-01-peer-admission-and-event-routing.md) | `client/connection` + `api/gateway`，10 files / +736 −20（含测试、文档） | 已合入 `master`：`40a0353268` |
| Schedule 数据过滤 | [`2026-10-23-02-schedule-data-filter.md`](./2026-10-23-02-schedule-data-filter.md) | `schedule/schedule`，3 files / +182 −2（含测试、文档） | 已合入 `master`：`9b4ad3f403` |
| Docker execution world | [`2026-10-28-01-docker-execution-world.md`](./2026-10-28-01-docker-execution-world.md) | 3 个 SSH provider，+229 −13（含文档；代码约 40 行） | 已合入 `master`：`ca3414084f` |

当前 `master` 历史：

```text
HEAD       docs(fork): add fork upstream README and merge status
ca3414084f merge: Docker execution world support
38e2d06dc1 docs: 更新 Docker 侧 live 验证状态
5a93405a7b refactor: SSH provider 支持 remoteWorld 优先与 ctx.ssh 回退
9b4ad3f403 feat: 按调用会话访问权限过滤自动化任务
40a0353268 feat: add opt-in peer admission and delegated waterfall routing
687f282e86 Merge pull request #6408 ... release-dsh-0.2.1-rc.2
```

## 一、fork 调整

### 1. per-user Peer 准入与事件路由

**目的**：让浏览器请求可以解析成每用户非 owner Peer，并让该 Peer 能应答 approval / user-question waterfall，使 `workspace-controller` / `session-controller` 的 `allows()` / `decide()` 真正生效。

**改动位置**：

- `packages/client/connection`
- `packages/api/gateway`

**机制**：

- 新增 opt-in 能力位 `PeerOptions.answerWaterfalls`；
- 默认 `false`，未启用时 owner 路径逐字节不变；
- 新增 per-request peer resolver 链；
- `answers = owner || answerWaterfalls`；
- waterfall、pending replay、`$events/result` 和 emit 事件只对 owner 或已授权 Peer 放行；
- `hasLiveClient` 保持 owner-only。

**使用方式**：

这是给 `dsh-login` 一类多用户插件使用的底层能力。默认宿主不需要配置；由插件注册 resolver / policy 后才会产生非 owner Peer。没有 resolver 时行为不变。

**验证**：

```sh
pnpm exec vitest run \
  packages/client/connection/tests \
  packages/api/gateway/tests
```

文档中记录的当时结果：

- connection：189 passed；
- gateway：413 passed；
- 定向 `tsc` 与 oxlint 通过。

### 2. Schedule 数据过滤

**目的**：普通用户只能看到和操作自己的日程任务，避免通过 `schedule.catalog/list/history/delete/update` 读取或修改其他用户的任务。

**改动位置**：

- `packages/schedule/schedule/src/index.ts`
- 对应测试与文档

**机制**：

- 在 Remote 方法调用上下文中读取 `remoteAccess`；
- `allows({ kind: 'session', id })` 决定任务是否可见/可操作；
- 没有 `remoteAccess` 服务的独立宿主保持 owner 行为；
- 无权限时 `catalog` 不投影，`list` 返回空，`history/delete/update` 返回 `schedule_not_found`。

**使用方式**：

无需额外配置。有 `remoteAccess` 的多用户宿主自动按调用者过滤；普通单用户宿主行为不变。

**验证**：

```sh
pnpm exec vitest run packages/schedule/schedule/tests
pnpm exec tsc -b packages/schedule/schedule/tsconfig.json
```

文档记录：**18 files / 742 tests passed**。

### 3. Docker execution world

**目的**：让 Docker 外部实现复用 SSH 的 fs/subprocess/sandbox provider，同时保持现有 SSH 使用方完全不变。

**改动位置**：

```text
packages/ssh/fs-ssh/src/index.ts
packages/ssh/subprocess-ssh/src/index.ts
packages/ssh/sandbox-ssh/src/index.ts
```

**唯一语义**：

```ts
const world = this.ctx.get('remoteWorld') ?? this.ctx.get('ssh')
```

- 有 `ctx.remoteWorld` 时用 Docker；
- 没有时回退 `ctx.ssh`；
- `SshConnection` 不改；
- 不新增上游服务、包、依赖、tsconfig 或 lockfile；
- 不新增 `ctx.remoteWorld` 全局声明；
- `NodePtcRuntime`、SSH 测试 fixture 均不改。

**Docker 实现位置**：

```text
../dsh-docker/packages/
├─ remote-world/
├─ docker-contract/
├─ docker-world/
├─ docker-providers/
├─ docker-ptc/
├─ docker-gate/
└─ docker-bash/
```

Docker 插件通过 `@dsh-seego/docker` 单包 tgz 安装；上游只保留 provider 的 fallback 逻辑。

**兼容性**：

| 场景 | 结果 |
|---|---|
| 现有 SSH 使用方 | `ctx.ssh` 仍唯一，行为不变 |
| 只提供 `ctx.ssh` 的第三方/测试宿主 | fallback，继续工作 |
| Docker 宿主 | `DockerWorld` 提供 `ctx.remoteWorld`，优先使用 |
| 上游依赖面 | 零新增依赖 |

**验证**：

```sh
pnpm exec vitest run \
  packages/ssh/ssh/tests \
  packages/ssh/fs-ssh/tests \
  packages/ssh/subprocess-ssh/tests \
  packages/ssh/sandbox-ssh/tests
```

当前结果：**227 passed / 2 skipped**。

## 二、如何编译

### 1. 安装依赖

```sh
cd /home/achen/code/dsh/deepseek-harness
pnpm install --frozen-lockfile
```

### 2. 只编译本次 fork 涉及的包

```sh
NODE_OPTIONS=--max-old-space-size=4096 pnpm exec tsc -b \
  packages/ssh/ssh \
  packages/ssh/fs-ssh \
  packages/ssh/subprocess-ssh \
  packages/ssh/sandbox-ssh \
  packages/client/connection \
  packages/api/gateway \
  packages/schedule/schedule
```

如果只想验证 Docker 相关上游改动：

```sh
NODE_OPTIONS=--max-old-space-size=4096 pnpm exec tsc -b \
  packages/ssh/ssh \
  packages/ssh/fs-ssh \
  packages/ssh/subprocess-ssh \
  packages/ssh/sandbox-ssh
```

### 3. 开发运行（无需完整构建）

fork 的 CLI 可以直接跑 TypeScript：

```sh
pnpm dsh --profile web
pnpm dsh --profile headless "..."
```

本仓库约定使用 `pnpm dsh`，不使用全局 `dsh`。

### 4. 完整构建（可选）

```sh
# host lib 构建（较慢，需要足够内存）
pnpm run build:lib:host

# 或完整构建
pnpm run build
```

只是开发/验证 Docker 插件时，不需要完整构建；`pnpm dsh` 会直接加载源码。

## 三、如何使用

### 1. 普通 SSH / 现有宿主

不需要任何配置。`ctx.ssh` 路径保持原样；Docker 补丁只在存在 `ctx.remoteWorld` 时生效。

### 2. 多用户 + dsh-login

Peer 准入与 schedule 过滤都是给多用户场景用的：

- dsh-login 注册 peer resolver / policy 后，非 owner Peer 才出现；
- 有 `remoteAccess` 时，schedule 自动按 session 权限过滤；
- 没有对应插件/服务时，行为回到 owner 默认。

具体 dsh-login 配置见其仓库文档；本 fork 只提供上游 seam。

### 3. Docker execution world

构建 Docker 镜像、打包/安装插件：

```sh
# 1. 构建镜像（一次性）
cd /home/achen/code/dsh/dsh-docker
./image/build.sh --helper-archive <dsh-ssh-helper-...tar.gz>

# 2. 构建插件 tgz
npm install --include=dev --omit=peer --ignore-scripts
npm run build
npm pack

# 3. 安装到 profile
cd /home/achen/code/dsh/deepseek-harness
pnpm dsh plugin --profile web add \
  /home/achen/code/dsh/dsh-docker/dsh-seego-docker-0.1.2.tgz

# 4. 启动
pnpm dsh --profile web
```

在 Web 中：

1. **Agent presets** 选择 **PTC**；
2. 新建会话；
3. 让 agent 执行：

```text
用 run_code 计算 6*7，并返回 process.cwd()
```

预期：

```text
{ value: 42, cwd: '/var/dsh/workspace' }
```

Docker 的详细构建、配置、故障排查见：

- `../dsh-docker/README.md`
- `../dsh-docker/image/README.md`
- `../dsh-docker/docs/IMPLEMENTATION.md`

### 4. Headless 验证

```sh
cd /home/achen/code/dsh/deepseek-harness
DSH_HOME=/tmp/dsh-headless \
DEEPSEEK_BASE_URL=http://127.0.0.1:18080/v1 \
DEEPSEEK_API_KEY=mock-key \
pnpm dsh --profile headless --json "Run the bash tool and report the output."
```

完整的三条 e2e（Bash / PTC `run_code` / fail-closed）在 dsh-docker：

```sh
cd /home/achen/code/dsh/dsh-docker
bash scripts/e2e-headless.sh
```

## 四、测试与回归

| 范围 | 命令 |
|---|---|
| SSH providers | `pnpm exec vitest run packages/ssh/{ssh,fs-ssh,subprocess-ssh,sandbox-ssh}/tests` |
| Peer admission | `pnpm exec vitest run packages/client/connection/tests packages/api/gateway/tests` |
| Schedule | `pnpm exec vitest run packages/schedule/schedule/tests` |
| Docker 插件单测 | `cd ../dsh-docker && bash scripts/test-unit.sh` |
| Docker headless e2e | `cd ../dsh-docker && bash scripts/e2e-headless.sh` |

当前已记录：

```text
SSH:      227 passed / 2 skipped
Docker 插件单测: 18 passed
Docker headless e2e: Bash / run_code / fail-closed 全通过
```

## 五、合并状态与分支

```text
master (local)       ca3414084f
├─ 40a0353268        peer admission（squash）
├─ 9b4ad3f403        schedule data filter（squash）
└─ 5a93405a7b        SSH provider fallback
   └─ 38e2d06dc1     docs live status
      └─ ca3414084f   merge commit
```

保留的开发分支：

- `feat/per-user-peer-admission`
- `feat/schedule-data-filter`
- `feat/docker-execution-world`

`master` 已包含三组补丁；当前还没有推送到 `origin`，推送前请确认远端策略。

## 六、回滚

| 补丁 | 回滚方式 |
|---|---|
| Peer admission | dsh-login 不注册 resolver/policy；上游补丁本身默认不启用 |
| Schedule filter | 移除该 commit；没有 `remoteAccess` 时行为不变 |
| Docker world | 只改 3 个 provider 的 fallback；删除 Docker 插件即回到 `ctx.ssh` |

Docker 插件侧的回滚见 `../dsh-docker/README.md` 的卸载/迁移说明。

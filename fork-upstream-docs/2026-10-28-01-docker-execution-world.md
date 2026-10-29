# 01 · 上游增量：SSH provider 支持 remoteWorld 优先与 ctx.ssh 回退

> 日期：**2026-10-28**
> 开发分支：`feat/docker-execution-world`（base `master` / `9b4ad3f403`）
> 状态：**仅 3 个源文件的最小增量，未合入 master。**
> 关联：Docker 实现仓库 `/home/achen/code/dsh/dsh-docker`（`packages/docker-*`）；
> 设计见其 `docs/DESIGN.md`（v0.4），实施记录见 `docs/IMPLEMENTATION.md`。
> 目标：让 Docker 外部实现能复用 SSH 的 fs/subprocess/sandbox provider，
> 同时让现有 SSH 使用方与测试完全不变。

---

## 0. 结论

上游**只改 3 个 provider 源文件**：

- `packages/ssh/fs-ssh/src/index.ts`
- `packages/ssh/subprocess-ssh/src/index.ts`
- `packages/ssh/sandbox-ssh/src/index.ts`

改动只有一处语义：

> 调用时优先取 `ctx.get('remoteWorld')`；没有则回退 `ctx.get('ssh')`。

因此：

- `SshConnection` **不改**，仍只注册 `ctx.ssh`；
- 不新增上游服务、包、依赖、tsconfig 或锁文件；
- 不新增 `ctx.remoteWorld` 的全局声明；
- `NodePtcRuntime` **不改**；
- 上游测试 fixture **不改**；
- Docker 的 `RemoteWorld` 实现、类型、providers、PTC、gate、bundle 全在
  `dsh-docker/packages`。

最终 diff：

```text
3 source files
+228 / -13（含 188 行本文档；代码约 40 行）
```

---

## 1. 问题

`fs-ssh` / `subprocess-ssh` / `sandbox-ssh` 直接依赖 `ctx.ssh`。Docker 想复用它们，
只有两条路：

1. Docker 伪装成 `SshConnection` 并占用 `ctx.ssh`——类型撒谎、与真实 SSH 冲突；
2. 改 provider，让传输可以是 SSH 或 Docker。

第 2 条如果改成“只注入 `remoteWorld`”，会破坏只提供 `ctx.ssh` 的第三方/测试宿主，
并让上游新增一个 seam 包/服务。为保持增量，采用**调用时回退**：

```ts
const world = this.ctx.get('remoteWorld') ?? this.ctx.get('ssh')
```

- SSH 宿主：`remoteWorld` 不存在，走 `ctx.ssh`，行为与今天一致；
- Docker 宿主：只有 `ctx.remoteWorld`，走 Docker；
- 两者同时存在：优先 Docker seam；由部署方保证二选一。

---

## 2. 最小改动

### 2.1 `fs-ssh`

```ts
interface RemoteWorld {
  request<T>(method: string, params: unknown, result: z.ZodType<T>,
    signal?: AbortSignal, wait?: boolean): Promise<T>
}

private world(): RemoteWorld {
  const world = (this.ctx.get('remoteWorld' as never) ?? this.ctx.get('ssh')) as RemoteWorld | undefined
  if (world === undefined) throw new Error('SSH filesystem requires ctx.remoteWorld or ctx.ssh')
  return world
}
```

`static inject` 从 `['ssh', 'sandboxPolicy']` 改为 `['sandboxPolicy']`；`call()` 使用
`this.world()`。

### 2.2 `subprocess-ssh`

同样的 `world()`；接口额外包含 `connectStream` / `dispose`。`static inject` 从
`['ssh']` 移除；所有 `ctx.ssh` 调用改 `this.world()`。

### 2.3 `sandbox-ssh`

同样的 `world()`；接口只需 `request`。`static inject` 从 `['ssh']` 移除；
`confine()` 使用 `this.world()`。

### 2.4 明确不改

- `packages/ssh/ssh/src/index.ts`：`SshConnection` 原样；
- `packages/ptc-runtime/ptc-runtime-node/src/index.ts`：原样；
- 所有 SSH 测试：原样（仍 `provide('ssh')`）；
- `tsconfig.*` / `pnpm-lock.yaml`：无变化；
- 不新增 `packages/remote/*`。

---

## 3. 兼容性

| 场景 | 影响 |
|---|---|
| 现有 SSH 使用方 | 完全不变；`ctx.ssh` 继续是唯一 SSH 服务 |
| 只提供 `ctx.ssh` 的第三方/测试宿主 | `world()` 回退，继续工作 |
| 未挂 Docker bundle 的宿主 | 不新增包/服务/行；行为不变 |
| Docker 宿主 | 外部 `DockerWorld` 提供 `ctx.remoteWorld`，provider 优先使用 |
| SSH 与 Docker 同时挂载 | provider 优先 `remoteWorld`；`remoteWorld` 只允许一个实现 |
| 工具层 / permission / approval | 无改动 |
| 上游依赖面 | **零新增依赖**；仅 3 个源文件 |
| 类型/构建 | 无新增包、tsconfig、lockfile |

**唯一 activation 差异**：provider 不再 `inject('ssh')`，因此在一个既没有 SSH 也没有
`remoteWorld` 的宿主里，它会先注册、调用时再失败，而不是等待 `ssh` 后不激活。
这类拓扑本来就不应挂载 SSH provider；SSH/Docker 正常 profile 不受影响。

---

## 4. 验证

```sh
cd /home/achen/code/dsh/deepseek-harness

pnpm exec tsc -b \
  packages/ssh/fs-ssh \
  packages/ssh/subprocess-ssh \
  packages/ssh/sandbox-ssh
# exit 0

pnpm exec vitest run \
  packages/ssh/ssh/tests \
  packages/ssh/fs-ssh/tests \
  packages/ssh/subprocess-ssh/tests \
  packages/ssh/sandbox-ssh/tests
# 227 passed / 2 skipped
```

测试 fixture 仍使用 `provide('ssh')`，验证了回退路径。Docker 侧 `remoteWorld`
路径的实现/测试在 `dsh-docker` 仓库；提取前已通过 18 个 Docker 测试与 244 个
Docker+SSH 组合测试。

Docker 侧已在 `dsh-docker` 完成 live 验证（Docker Desktop 27.3.1）：world ready、
helper hello、fs 写读、subprocess、bwrap sandbox、PTC `run_code` 均通过。
真实 dsh profile + Bash/gate 组合仍未验证。

---

## 5. Docker 实现位置

Docker 源码、测试、bundle patch 全部在：

```text
/home/achen/code/dsh/dsh-docker/packages/
├─ remote-world/          # Docker 侧 RemoteWorld 基类/类型
├─ docker-contract/
├─ docker-world/
├─ docker-providers/
├─ docker-ptc/
├─ docker-gate/
├─ docker-bash/
└─ docker-bundle/         # npm 包名 @dsh-seego/docker
```

上游不包含任何 Docker 实现或 `remoteWorld` 包。Docker 仓库通过
`scripts/dev-link.sh` 将包软链回 harness workspace 进行本地类型检查；
发布时 profile 安装 `@dsh-seego/docker`，不要求上游携带实现。

---

## 6. 变更文件

```text
packages/ssh/fs-ssh/src/index.ts
packages/ssh/subprocess-ssh/src/index.ts
packages/ssh/sandbox-ssh/src/index.ts
```

明确不包含：

- `packages/ssh/ssh/*`
- `packages/ptc-runtime/*`
- `packages/remote/*`
- `packages/docker/*`
- `tsconfig.*` / `pnpm-lock.yaml`

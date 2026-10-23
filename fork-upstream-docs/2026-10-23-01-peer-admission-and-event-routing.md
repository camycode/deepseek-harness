# 01 · 上游补丁：per-user Peer 准入与事件路由

> 日期：**2026-10-23**
> 上游 fork：`/home/achen/code/dsh/deepseek-harness` · master · `687f282e86`（0.2.1-rc.2 发布合并）
> 归属：**本文只描述 deepseek-harness 的修改**。dsh-login 侧见其仓库内的
> `fork-docs/2026-10-23-01-user-workspace-isolation.md`（本仓库是上游 fork，插件仓库与它同级）。
> 目标：让浏览器请求可以被解析成**每用户非 owner Peer**，并让该 Peer 能应答 approval / user-question waterfall，
> 从而让 `workspace-controller` / `session-controller` 已有的 `allows()` / `decide()` 真正生效。

---

## 0. 结论

- 生产代码约 **130 行 diff**（逻辑约 60–70 行，其余是类型与 JSDoc），只动两个包：
  - `packages/client/connection`：增加 per-request peer resolver；
  - `packages/api/gateway`：允许非 owner 应答；事件投递按 session 资源过滤。
- **不需要改** `workspace-controller` / `session-controller` / `remote-access`：它们已经按 `allows()` / `decide()` 过滤，只是今天被 owner 短路。
- **不需要改任何 UI**：shipped `WorkspaceBrowser` / `WorkspacePicker` 读到的 store 会自然只含自己的数据。
- 没有注册 resolver 时，行为与今天**逐字节一致**；补丁是 opt-in 的。
- 这是安全关键路径，必须配双用户测试与 §5 的不变量检查表。

### 0.1 对其它插件的影响（结论）

**默认零影响**：所有新行为都由 `PeerOptions.answerWaterfalls` 这个能力位 opt-in；未设置的 Peer 走原路径。

| 插件类型 | 影响 | 说明 |
|---|---|---|
| UI 插件 | 无 | 补丁不含任何 UI 改动 |
| 工具 / agent 插件 | 无 | 不碰 agent-loop / tools |
| 精确 `/api` 路由插件 | 无 | 不走 gateway；若它调用 `connection.requestRejection`，会继承 resolver 的 401（只在 dsh-login 注册 resolver 时） |
| owner Peer 插件 | 无 | owner 路径保持原样 |
| 非 owner Peer 插件 | 无 | `answerWaterfalls` 默认 false；`answers` 与事件路径与今天一致 |
| 使用 `hasLiveClient` 的插件 | 无 | 保持 owner-only，非 owner 页面不计入 |
| 跨用户聚合 UI | 功能变化（预期） | dsh-login 普通用户只看到自己的数据；管理员 owner 不变 |
| remote-web-ui 配对设备 | ⚠️ 需验证 | dsh-login resolver 对「浏览器但无 dsh_session」返回 401；若配对设备依赖原生 browser cookie，会被拦。见 dsh-login 文档的兼容开关 |

**增量补丁的关键**：不是全局改 `answers` / owner 检查，而是：

1. 新增 `answerWaterfalls` 能力位（默认 false）；
2. `answers = owner || answerWaterfalls`；
3. `$events/result` 只对 owner 或 `answerWaterfalls` 放行；
4. waterfall 只投递给 owner 或（`answerWaterfalls` && `decide`）；
5. emit 事件：owner 跳过，非 owner 一律 `decide`（对未 opt-in 的非 owner 与今天完全一致）；
6. `hasLiveClient` 保持 owner-only；
7. resolver 用**链**而不是单槽，避免和未来插件冲突。

这样，未 opt-in 的 Peer 在所有路径上与今天逐字节一致。

---

## 1. 背景

### 1.1 今天的浏览器永远是 owner

`packages/client/connection/src/rpc-host.ts`：

```ts
this.browserPeer = new Peer(ctx, { principal: { kind: 'owner' } })

admit(request) {
  if (this.browserAuth.isAuthenticated(request, secure)) return { peer: this.browserPeer }
  ...
}
```

所有带浏览器 cookie 的请求共享同一个 owner Peer。原生 per-resource RBAC（`remoteAccess.decide/allows`）**只对非 owner 生效**，所以：

- `WorkspaceController.follow` 的 `allows()` 全放行；
- `SessionController.list/search/control` 的过滤全放行；
- 任何登录用户都能看到/操作所有工作区与会话（只要知道 id）。

### 1.2 为什么不能直接换 principal

`packages/api/gateway/src/index.ts`：

```ts
answers: peer.principal.kind === 'owner'
if (peer.principal.kind !== 'owner') throw ... only owner Peers answer forwarded Remote events
```

approval / `user_questions` 是 waterfall，必须由 `$events/result` 应答。非 owner 今天不能应答，直接换 principal 会弄坏审批与提问。

本补丁同时解决这两件事：**准入**（浏览器 → 非 owner Peer）与**应答**（非 owner 也能安全地收到并回答自己资源上的 waterfall）。

### 1.3 范围

**做**：

- `connection` 的 per-request peer resolver；
- 非 owner answer client 的 `$events` / `$events/result`；
- waterfall / emit 事件按资源 `decide`；
- 连接时 pending 事件补发的同样过滤；
- 对应的单元/集成测试。

**不做**：

- UI 隐藏（终端/插件/轨迹/更多/picker）——在 dsh-login 侧；
- dsh-login 的 policy / 个人根 / 子会话记录——在 dsh-login 侧；
- 第三方插件自己注册的精确 `/api/...` 路由（不走 gateway）；
- 每用户 OS 级 shell 隔离。

---

## 2. 补丁 A：`connection` per-request peer resolver

### 2.0 能力位（兼容性闸门）

`packages/typert/protocol/src/types.ts` 的 `PeerScope`：

```ts
export interface PeerScope {
  // ...
  /** Whether this non-owner Peer may answer forwarded waterfalls. Omitted = false. */
  readonly answerWaterfalls?: boolean
}
```

`packages/client/connection/src/peer.ts` 的 `PeerOptions` / `Peer`：

```ts
export interface PeerOptions {
  readonly principal: Principal
  readonly interactions?: ConnectionInteractions
  /** Opt in to answering forwarded waterfalls; default false. */
  readonly answerWaterfalls?: boolean
}

// Peer
readonly answerWaterfalls: boolean = options.answerWaterfalls ?? false
```

`createPeer` 的 options 类型（`HostConnectionHandle` 与 `HostConnectionService` 两处）补上 `answerWaterfalls?: boolean`（或直接复用 `PeerOptions`）；运行时它已经把 options 原样传给 `new Peer`。owner Peer 不需要设置这个位（gateway 仍按 owner 放行）。

### 2.1 类型与接口

`packages/client/connection/src/rpc.ts`：

```ts
export type ConnectionPeerResolver = (
  request: ConnectionTrustRequest,
  context: { readonly browser: boolean },
) => PeerAdmission | undefined

export interface HostConnectionHandle {
  // ... existing members
  /**
   * Register a resolver for browser/API admission in the caller's fiber.
   * Resolvers run in registration order; the first non-undefined result wins.
   */
  registerPeerResolver(resolver: ConnectionPeerResolver): () => void
}
```

`packages/client/connection/src/index.ts` 重新导出 `ConnectionPeerResolver`。

### 2.2 `HostConnectionService.admit`

`packages/client/connection/src/rpc-host.ts`：

```ts
private readonly peerResolvers: ConnectionPeerResolver[] = []

/** Register in the caller's fiber; resolvers run in registration order. */
registerPeerResolver(resolver: ConnectionPeerResolver): () => void {
  this.peerResolvers.push(resolver)
  return () => {
    const index = this.peerResolvers.indexOf(resolver)
    if (index !== -1) this.peerResolvers.splice(index, 1)
  }
}

admit(request: ConnectionTrustRequest): PeerAdmission {
  if (!this.isTrusted(request)) return { rejection: 403 }
  if (this.disposed) return { rejection: 401 }

  const secure = this.owner.get('webServer')?.protocol === 'https:'
  const browser = this.browserAuth.isAuthenticated(request, secure)
  for (const resolver of this.peerResolvers) {
    const resolved = resolver(request, { browser })
    if (resolved !== undefined) return resolved
  }
  if (browser) return { peer: this.browserPeer }

  const id = this.browserAuth.authenticateApi(request, secure)
  const record = id === undefined ? undefined : this.peers.get(id)
  return record === undefined ? { rejection: 401 } : { peer: record.peer }
}
```

Diff 视角：

```diff
-    if (this.browserAuth.isAuthenticated(request, secure)) return { peer: this.browserPeer }
+    const browser = this.browserAuth.isAuthenticated(request, secure)
+    const resolved = this.peerResolver?.(request, { browser })
+    if (resolved !== undefined) return resolved
+    if (browser) return { peer: this.browserPeer }
```

### 2.3 语义

- resolver 在 **trust/dispose 检查之后、owner 回退之前**运行；
- `browser` 标志区分「浏览器 cookie 请求」与「v2 API cookie / 原生客户端」；
- resolver 返回 `{ rejection: 401 }` 可以阻止 owner 回退——**浏览器登出后 WS 不再泄漏**的关键；
- resolver 返回 `undefined` → 尝试下一个 resolver；全部返回 `undefined` 时才走原生逻辑；
- 没有注册 resolver → 行为与今天一致；
- resolver 链按注册顺序执行；dsh-login 只注册一个。若未来有 SSO/配对插件也要参与，直接追加，不会互相覆盖。

**`admitConnection` 不需要改**：它调用 `admit`；自定义 peer 不在 `peers` 表里时返回 `release: noop`，多个标签页共享同一 peer；登出时由插件 dispose。若未来要严格「一 socket 一 peer」，再扩展 `peers` 记录。

**`requestRejection` 不需要改**：它调用 `admit`，自动获得 resolver 的 rejection。

### 2.4 测试

`packages/client/connection/tests/`（或现有测试目录）：

1. 未注册 resolver → browser/API 行为逐字节不变；
2. resolver 返回 `{ peer }` → `admit` 返回该 peer；
3. resolver 返回 `{ rejection: 401 }` → 不再回退 `browserPeer`；
4. `context.browser` 对浏览器 cookie 为 true、对 v2 API cookie 为 false；
5. resolver 抛错 → 明确失败（不要吞掉；由插件负责不抛）；
6. dispose 后 resolver 不再生效。

---

## 3. 补丁 B：`gateway` 事件路由

`packages/api/gateway/src/index.ts`。

### 3.1 opt-in 的非 owner 也可以成为 answer client

```diff
 const client: RemoteEventClient = {
   peer,
   id: clientId,
   queue: new RemoteEventQueue(),
   signal: lifetime,
   deliveries: new Map(),
   principal: peer.principal,
-  // oxlint-disable-next-line typescript/no-unnecessary-condition -- other packages merge non-owner kinds into PrincipalMap.
-  answers: peer.principal.kind === 'owner',
+  answers: peer.interactions === 'answer'
+    && (peer.principal.kind === 'owner' || peer.answerWaterfalls === true),
 }
```

`openRemoteEvents` 入口已有 `requireInteractions(peer, '$events')`，所以 `interactions` 一定是 `answer`。

- owner：`answers` 仍为 true（与今天一致）；
- 非 owner 且未 opt-in：`answers` 仍为 false（与今天一致）；
- 非 owner 且 `answerWaterfalls: true`：`answers` 为 true（新行为，仅 dsh-login）。

### 3.2 `$events/result` 对 opt-in Peer 放行

```diff
 assertPeerLive(peer, endpoint)
 requireInteractions(peer, endpoint)
-// oxlint-disable-next-line typescript/no-unnecessary-condition -- other packages merge non-owner kinds into PrincipalMap.
-if (peer.principal.kind !== 'owner') {
-  throw new TypertGatewayError('gateway/invocation-unavailable', endpoint, 'only owner Peers answer forwarded Remote events')
-}
+// oxlint-disable-next-line typescript/no-unnecessary-condition -- other packages merge non-owner kinds into PrincipalMap.
+if (peer.principal.kind !== 'owner' && peer.answerWaterfalls !== true) {
+  throw new TypertGatewayError('gateway/invocation-unavailable', endpoint, 'only owner Peers answer forwarded Remote events')
+}
 const result = parseRemoteEventResultPayload(payload)
 const client = this.remoteEventClients.get(result.clientId)
 if (client === undefined) throw ...
 if (client.peer !== peer) throw ... // 已有：结果只能来自收到事件的 Peer
```

`client.peer !== peer` 仍然保证 B 不能替 A 应答；未 opt-in 的非 owner Peer 仍得到与今天相同的拒绝。

### 3.3 waterfall 只投递给有权限的 answer client

新增 helper：

```ts
private eventAllowed(
  event: string,
  resources: readonly RemoteAccessResource[],
  client: RemoteEventClient,
): boolean {
  if (client.principal.kind === 'owner') return true
  return this.ctx.remoteAccess.decide({
    principal: client.principal,
    action: `event:${event}`,
    resources,
  }).kind === 'allow'
}
```

`startRemoteEvent` 的投递循环：

```ts
const resources = [{ kind: 'session', id: source.context.agentId }]

for (const client of this.remoteEventClients.values()) {
  if (!client.answers) continue
  if (!this.eventAllowed(source.event, resources, client)) continue
  this.deliverRemoteEvent(pending, client)
}
```

`source.context.agentId` 是 `Agent.id`；`Agent.id` 的类型就是 `SessionId`
（`packages/core/agent/src/types.ts`），所以可以直接作为 `session` 资源，**不需要**改 `api-remotes` 的事件声明。

### 3.4 连接时补发 pending 也要过滤

```ts
if (client.answers) {
  for (const pending of this.pendingRemoteEvents.values()) {
    const resources = [{ kind: 'session', id: pending.frame.agentId }]
    if (this.eventAllowed(pending.frame.event, resources, client)) {
      this.deliverRemoteEvent(pending, client)
    }
  }
}
```

否则新登录的用户会在连接瞬间收到别人的待审批请求。

### 3.5 emit 事件对非 owner 一律判权限

```diff
 for (const client of this.remoteEventClients.values()) {
-  if (!client.answers) {
-    // A non-owner receives the whole notification or nothing...
-    if (!resolved) { resources = eventResources(frame.resources, frame.args); resolved = true }
-    if (resources === undefined) continue
-    if (this.ctx.remoteAccess.decide({ principal: client.principal, action: `event:${frame.event}`, resources }).kind === 'deny') continue
-  }
+  if (client.principal.kind !== 'owner') {
+    if (!resolved) { resources = eventResources(frame.resources, frame.args); resolved = true }
+    if (resources === undefined) continue
+    if (!this.eventAllowed(frame.event, resources, client)) continue
+  }
   client.queue.push(wire)
 }
```

原代码只对 `!client.answers` 判权限；在 gate 之后，opt-in 的非 owner answer client 的 `answers` 为 true，若不改就会绕过 `decide`。改成「非 owner 一律判」：

- owner：跳过（与今天一致）；
- 非 owner 且未 opt-in：`answers` 为 false → 走 `decide`（与今天一致）；
- 非 owner 且 `answerWaterfalls`：走 `decide`（新行为，正确收窄）。

> 注：上游 `PrincipalMap` 只声明 `owner`，非 owner 的 kind 由插件 declaration merging 引入。
> `eventAllowed` / `dispatchRpc` 里的 `principal.kind` 比较需要保留
> `// oxlint-disable-next-line typescript/no-unnecessary-condition` 注释，否则上游自身构建会把它当死代码。

### 3.6 `hasLiveClient` 保持 owner-only

`hasLiveClient()` 目前被 `cordis-host-runner` 用来判断「有没有连接的 Harness 页面」。非 owner 页面不应改变这个判断：

```ts
hasLiveClient(): boolean {
  for (const client of this.remoteEventClients.values()) {
    if (client.principal.kind === 'owner' && client.answers && !client.signal.aborted) return true
  }
  return false
}
```

dsh-login 页面可以应答 waterfall，但不算「owner 的 inspect 页面」；否则 inspect query 的报错会从「没有连接页面」变成「access denied」，是与本需求无关的行为变化。

### 3.7 测试

`packages/api/gateway/tests/`（或现有测试目录）：

1. 两个非 owner Peer（A/B），policy 只授予各自 session；
2. A 的 `approval/request` 只投递给 A；B 收不到；
3. B 对 A 的事件提交 `$events/result` → `client.peer !== peer` 拒绝；
4. `api-session/status` 只到对应 session 的 Peer；
5. owner 仍收到全部；
6. `interactions:'none'` 仍不能开 `$events`；
7. 连接时 replay 的 pending approval 也按权限过滤；
8. 事件资源解析失败（`undefined`）→ 不广播；
9. 非 owner 没有 policy → `decide` 拒绝（fail-closed）；
10. 未 opt-in 的非 owner Peer：`answers=false`、收不到 waterfall、`$events/result` 仍被拒、emit 路径与今天一致；
11. `hasLiveClient()` 只对 owner answer client 返回 true；
12. resolver 链：两个 resolver，第一个返回 `undefined` 时第二个生效；移除后回退原生。

---

## 4. 集成契约（dsh-login 依赖）

| 契约 | 值 |
|---|---|
| 注册入口 | `ctx.connection.registerPeerResolver(resolver)`，返回 disposer；多 resolver 按注册顺序执行 |
| resolver 签名 | `(request, { browser }) => { peer } \| { rejection: 401 \| 403 } \| undefined` |
| 运行时机 | trust 检查后、owner 回退前；每个 `/api` HTTP 请求与 WS upgrade |
| 未注册 | 原生 owner 行为不变 |
| Peer 生命周期 | resolver 必须返回**稳定、可回收**的 Peer（按用户缓存），不要每个请求 `createPeer`；`admitConnection` 对不在 `peers` 表里的 peer 返回 `release: noop`，由插件负责 dispose |
| 非 owner 应答 | 必须 `interactions: 'answer'` **且** `createPeer({ answerWaterfalls: true })`；`$events` 仍被 `requireInteractions` 把关 |
| 能力位默认 | `answerWaterfalls` 默认 false；未设置的非 owner 行为与今天逐字节一致 |
| Principal | 任意 `kind`；dsh-login 用 `{ kind: 'dsh-login-user', username }`，由插件 `declare module` 合并 `PrincipalMap` |
| 策略 | `ctx.remoteAccess.registerPolicy(kind, { bindings, explain })` |
| 端点动作 | `namespace/method`（如 `workspace/follow`、`session/create`） |
| 事件动作 | `event:<name>`（如 `event:approval/request`） |
| waterfall 资源 | `source.context.agentId` 作为 `{ kind: 'session', id }` |
| 浏览器登出 | resolver 对浏览器请求返回 401；不回退 owner |

**行为变化**：非 owner 的 answer client 现在既能收到 waterfall，也能提交结果；但只有在 policy 授予对应 `event:<name>` 与 session 资源时才会被投递。

---

## 5. 不变量（review 清单）

1. 未注册 resolver 时，所有 admission 行为与今天完全一致。
2. resolver 只对 `isTrusted` 且未 dispose 的请求运行。
3. resolver 返回的 `{ rejection }` 必须原样生效，**不得被 owner 回退覆盖**（dsh-login 用它实现「浏览器无有效 dsh_session → 401」）。
4. 非 owner 只能应答自己收到的事件（`client.peer === peer`）。
5. waterfall 投递前必须 `decide`；owner 直接放行。
6. `interactions: 'none'` 的 Peer 仍然不能开 `$events`。
7. 管理员/本地 owner 的行为不变。
8. `agentId` 当作 `SessionId` 使用，与 `Agent.id` 的类型一致。
9. 事件资源解析失败（`undefined`）时丢弃，不广播。
10. 非 owner 没有 policy 时 deny（fail-closed），不是 allow。
11. `answerWaterfalls` 默认 false；未 opt-in 的非 owner Peer 不进入任何新路径。
12. `hasLiveClient()` 只统计 owner answer client。
13. resolver 链中任一 resolver 抛错要显式失败；插件不得吞错。

---

## 6. 验证计划

### 6.1 单元/集成（上游）

- connection：§2.4；
- gateway：§3.6；
- 组合：用真实 `remoteAccess` + 两个 peer + 一个 fake event source，跑完整 `$events` → 投递 → `$events/result` 路径。

### 6.2 手工（两浏览器）

1. A/B 各登录（dsh-login 提供 resolver/policy）；
2. B 建 `B-private` + 会话；A 的 `workspace.follow` 响应与侧栏 DOM 都没有 B；
3. A 直接 `session.page(B 的 id)` → denied；
4. B 的 approval 只弹在 B 的页面；
5. A 登出后 WS 重连 / `/api` → 401，不回退 owner；
6. 管理员仍看到全部。

### 6.3 性能

- 非 owner 的每次调用多一次 `decide`；policy 的 `bindings` 必须由插件缓存；
- 事件广播对每个非 owner client 多一次 `decide`；事件量在可接受范围。

---

## 7. 风险与回滚

| 风险 | 处置 |
|---|---|
| 安全关键路径（admission + 事件路由） | 小步提交 + §5 不变量 + 双用户测试 |
| 非 owner 应答破坏 approval 语义 | 只放开 `interactions:'answer'`；结果仍绑定 `client.peer`；waterfall 投递前 `decide` |
| 审批离屏/离线 | 请求只投给有权限的 answer client；用户不在线时依赖宿主超时/取消（后续可加 `hasLiveClient(resource)`） |
| 子会话未被授予 | 由 dsh-login 的 policy 覆盖；补丁不猜资源 |
| 事件动作清单过宽 | 由 policy 决定；补丁只做过滤 |
| 上游升级冲突 | 补丁集中在 2 个包、约 130 行 diff（逻辑约 60–70 行）；用测试守护 |
| 回滚 | 插件不注册 resolver / 不注册 policy 即回到 owner 行为；补丁本身无全局开关，靠 opt-in |

---

## 8. 工作量

| 阶段 | 内容 | 估时 |
|---|---|---|
| U1 | `connection` resolver + 类型 + 单测 | 1d |
| U2 | `gateway` 事件路由 + 审批应答 + 双用户测试 | 2d |
| U3 | 手工双浏览器验证 + 文档 | 0.5–1d |
| **合计** | | **3–4d** |

---

## 9. 提交拆分建议

1. **commit 1**：`answerWaterfalls` 能力位（`PeerScope` / `PeerOptions` / `Peer`）+ `connection` resolver 链 + `ConnectionPeerResolver` + `HostConnectionHandle` + 单测；
2. **commit 2**：`gateway` answers / `$events/result` / waterfall / pending replay / emit / `hasLiveClient` 的 gate + 测试；
3. **commit 3**：文档 + changelog（说明 opt-in 语义与 dsh-login 集成契约）。

两个 commit 都可以独立回滚；commit 1 单独合入不会改变默认行为，commit 2 在没有非 owner answer client 时也不会改变行为。

### 9.1 增量上线顺序

1. 合并 commit 1 + commit 2：上游默认行为**完全不变**（没有 Peer 设置 `answerWaterfalls`）；
2. dsh-login 发布时 `userUi.workspaceIsolation` 先默认 **false**，只注册代码不启用；
3. canary 部署打开开关，跑两浏览器 + 审批 + 登出验收；
4. 验收通过后把默认值翻成 true；
5. 出问题时 dsh-login 关开关即可（注销 resolver/policy、dispose peers），**不需要回滚上游**。


---

## 10. 实施状态（2026-10-23）

- 分支：`feat/per-user-peer-admission`（base `687f282e86`）
- 开发 commits：
  - `d85f0f95c4` `feat(connection): add opt-in peer resolver and waterfall answering`
  - `575b77e119` `feat(gateway): route waterfalls to opted-in non-owner peers`
  - `26626d1d64` `docs(fork): document per-user peer admission and event routing`
  - `c53a5c4b88` `test(gateway): type the delegated waterfall fixture as a session resource`
  - `7161c53331` `fix(native): let CC select the host addon compiler`
- 最终交付：上述提交在 `feat/per-user-peer-admission` 分支保留为开发历史；`master` 以**单一 squash 提交**合入（与 dsh-login 的交付方式一致）。
- diff：10 files，+736 / -20（含文档、测试与 native 构建修正）
- 验证：
  - `tsc -b packages/typert/protocol/tsconfig.json packages/client/connection/tsconfig.host.json packages/api/gateway/tsconfig.host.json` 通过；
  - oxlint 对 8 个改动文件 0 warnings / 0 errors；
  - `packages/client/connection/tests` 189 passed；
  - `packages/api/gateway/tests` 413 passed。

实施备注：

- `waterfallResources()` 用 `as unknown as RemoteAccessResource` 承载运行期注册的 `session` 资源种类；gateway 包不为此新增 `dsh-api-session-controller` 依赖。
- `hasLiveClient()` 保持 owner-only，delegated 页面不计入 inspect 的「已连接页面」。
- `binary-rpc.host.spec.ts` 的 gzip 用例在**并行**跑 connection + gateway 两个 suite 时偶发 5s 超时；单独跑该文件通过（与本次改动无关，已复跑确认）。

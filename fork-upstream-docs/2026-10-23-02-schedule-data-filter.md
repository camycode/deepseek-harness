# Schedule 数据过滤（上游最小增量）

> 日期：**2026-10-23** · 分支：`feat/schedule-data-filter`（base `master` / `40a0353268`）
> 状态：开发历史保留在该分支；已 squash 合入 `master`（2026-10-27）。
> 关联：dsh-login `fork-docs/2026-10-23-03-schedule-data-filter.md`
> 目标：普通用户只能看到/操作自己的日程任务；owner 与没有 remoteAccess 的宿主行为不变。

---

## 1. 问题

`schedule.catalog` 返回 Host 全量任务；`schedule.list/history/delete/update` 只校验请求里的
`sessionId` 与任务行的绑定，不校验调用者是否有权访问该 Session。普通用户启用日程面板后，
会看到其他用户的提醒，甚至可以通过猜 id 读取/删除/修改别人的任务。

本地 dsh-login 只能做调用级 allow/deny，无法裁剪响应；真正的数据过滤必须在
`@deepseek-ai/dsh-schedule` 的 Remote 边界完成。

---

## 2. 最小改动

只改 `packages/schedule/schedule/src/index.ts`：

```ts
private sessionAllowed(sessionId: SessionId): boolean {
  // Read inside the Remote method: the traceable service view must carry this
  // Context's invocation, so allows() sees the caller principal.
  const access = this.ctx.get('remoteAccess') as
    | { allows(resource: { kind: 'session'; id: SessionId }): boolean }
    | undefined
  if (access === undefined) return true
  try {
    return access.allows({ kind: 'session', id: sessionId })
  } catch {
    return false
  }
}
```

应用点：

| 方法 | 行为 |
|---|---|
| `catalog()` | 只投影 `sessionAllowed(task.sessionId)` 的任务 |
| `list(request)` | 无权 Session → `[]` |
| `history(request)` | 无权 Session → `schedule_not_found` |
| `delete(request)` | 无权 Session → `deleted:false, code:'schedule_not_found'` |
| `update(request)` | 无权 Session → `updated:false, code:'schedule_not_found'`（在 subagent 检查前） |

`create` 不是 Remote 方法；它由绑定 Session 的 agent tool 调用，不需要额外的调用者过滤。

### 2.1 为什么在调用时用 `ctx.get`

- schedule 包没有 `dsh-remote-access` 依赖，直接 `ctx.remoteAccess` 会因为 Cordis 未声明
  inject 而抛 `cannot get property without inject`；
- `ctx.get('remoteAccess')` 不要求 inject，且返回的 traceable 服务视图绑定到**当前
  schedule Remote 方法的 Context**，`allows()` 能读到 `ctx.invocation` 与调用者 principal；
- 不要在构造函数里 `ctx.inject` 后缓存服务：那会把服务绑定到**注入子 Context**，Remote
  调用时没有 invocation，`allows()` 会按 “outside a Remote call” 返回 true，从而放行全部；
- 没有 remoteAccess 的独立宿主/测试继续走 owner 行为；
- 访问器异常时 fail-closed（返回 false），避免策略故障时泄露数据。

### 2.2 owner / 普通用户

- owner：`remoteAccess.allows()` 对 owner 恒 true，行为与改动前一致；
- 普通用户：dsh-login 的 policy 只给自己的 workspace/session 绑定，因此只看到自己的任务；
- 未知/无权 Session：返回空或 `schedule_not_found`，不泄露是否存在。

### 2.3 事件

`schedule/changed` 仍是无 payload 的全局通知，不携带任务内容；客户端收到后重新拉取
`catalog/list`，由上面的过滤决定可见范围。事件本身不构成数据泄露。

---

## 3. 兼容性

| 场景 | 影响 |
|---|---|
| owner / 单用户 | 无变化 |
| 没有 remoteAccess 的宿主 | `sessionAllowed` 返回 true，无变化 |
| 第三方 schedule 客户端 | 只看到自己有权 Session 的 catalog/list；`history/delete/update` 对无权 id 返回 not_found |
| dsh-login | 移除 `schedule.*` 拒绝并恢复面板/时钟；由本补丁保证数据边界 |
| 其他插件 | 不新增依赖、不改事件签名、不改 create/tool 路径 |

---

## 4. 验证

```
pnpm exec vitest run packages/schedule/schedule/tests
# 18 files / 742 tests passed

pnpm exec tsc -b packages/schedule/schedule/tsconfig.json
# exit 0
```

新增用例 `plugin.spec.ts > shows and mutates only sessions admitted by the caller policy`：

- 允许 session + 隐藏 session 各建一个任务；
- `catalog()` 只返回允许的那个；
- `list(hidden)` 返回 `[]`；
- `history/delete/update(hidden)` 都返回 `schedule_not_found`；
- 隐藏任务未被删除。

---

## 5. 与 dsh-login 的配合

1. 上游合入本补丁（owner 行为不变）；
2. dsh-login 从 `POLICY_DENIED_METHODS` 移除 `schedule.*`，恢复 `event:schedule/changed`；
3. 从 native hidden 清单/网关 hidden 清单移除 `@deepseek-ai/dsh-client-ui-schedule`；
4. 客户端只隐藏「插件」面板，不再隐藏整组 panellist；恢复日程面板与时钟；
5. 两浏览器验收：普通用户只看到自己的任务，管理员仍看全部。

---

## 6. 增量上线

- 该补丁可独立合入；没有 remoteAccess 的宿主行为不变；
- dsh-login 可以先恢复日程面板、再逐步打开；
- 回滚 dsh-login 只需恢复 `schedule.*` 拒绝与隐藏清单，不需要回滚上游。

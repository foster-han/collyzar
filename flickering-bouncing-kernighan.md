# Changeset 锁定机制设计文档

## 一、背景与目标

### 问题
当前多个用户可以同时对同一 Changeset 执行修改和发布操作，缺乏并发保护机制，可能导致变更覆盖和冲突。

### 目标
1. **编辑锁定**：用户编辑 Changeset 内的文章时自动加锁，5 分钟无操作后自动释放，操作期间自动续期
2. **锁定展示**：其他用户看到锁定图标，无法对被锁 Changeset 执行 publish/apply；但可修改其中文章内容（自动创建新的 Changeset）
3. **API 支持**：`GET /changesets` 和 `GET /changesets/:id` 返回锁定状态

---

## 二、技术选型

**锁存储：Redis（通过现有 `CacheService`）**

| 优势 | 说明 |
|---|---|
| TTL 原生支持 | 无需定时任务清理过期锁 |
| 无 DB 改动 | 不增加 `sync_commits` 表字段，无 migration |
| 已有基础设施 | `getCache()` 提供统一接口，Redis 不可用时自动 fallback 到 MemoryStore |
| 符合现有模式 | 与 `RateLimitService`、session 存储保持一致 |

**相关文件**：
- `backend/src/services/CacheService.ts`：`getCache()` 获取单例 client
- `backend/src/services/MemoryStore.ts`：Redis 不可用时的内存 fallback

---

## 三、锁的数据结构

### Redis Key
```
cs:lock:{orgSlug}:{changesetId}
```
- `orgSlug`：来自 `getTenantContext().orgSlug`，确保多租户隔离（不同租户的 changesetId 数字可能重叠）
- `changesetId`：`SyncCommit.id`

### Redis Value（JSON 字符串）
```json
{
  "lockedBy": "42",
  "lockedAt": "2026-02-27T10:00:00.000Z",
  "lockExpiresAt": "2026-02-27T10:05:00.000Z"
}
```

### TTL
- **300 秒（5 分钟）**，由 Redis 原生管理
- 每次 amend 或心跳成功后重置为 300 秒

### 锁的生命周期

```
编辑文章（amend）
    └─ acquireOrExtendLock()
         ├─ 未锁 → 写入 Redis，TTL=300s
         ├─ 自己持锁 → 续期，TTL 重置为 300s
         └─ 他人持锁 → 返回 acquired=false，API 响应 423

心跳（每 3 分钟）
    └─ POST /changesets/:id/lock → acquireOrExtendLock() → 续期

5 分钟无操作
    └─ Redis TTL 自动过期，无需任何操作

Publish / Discard 完成
    └─ releaseLock() → DEL Redis key（立即释放）

切换到 Live 模式
    └─ 前端停止心跳 → 5 分钟后 TTL 自动过期
```

---

## 四、新增文件

### `backend/src/services/ChangesetLockService.ts`

**类型定义：**

```typescript
export interface ChangesetLock {
    readonly lockedBy: string;        // 锁持有者 user ID
    readonly lockedAt: string;        // 加锁时间（ISO string）
    readonly lockExpiresAt: string;   // 过期时间（ISO string，lockedAt + 5min）
}

export interface AcquireLockResult {
    readonly acquired: boolean;
    readonly lock: ChangesetLock;     // acquired=true 时为自己的锁，false 时为他人的锁
}
```

**方法说明：**

| 方法 | 说明 |
|---|---|
| `getLock(orgSlug, changesetId)` | 获取锁信息，返回 `null` 表示未锁或已过期 |
| `acquireOrExtendLock(orgSlug, changesetId, userId)` | 加锁或续期；他人持锁时返回 `acquired=false` |
| `releaseLock(orgSlug, changesetId)` | 主动释放锁（由 publish/discard 调用） |
| `getLocksBatch(orgSlug, changesetIds[])` | 批量获取锁状态（用于列表接口，并行 GET） |

> **并发说明**：`acquireOrExtendLock` 存在 GET-then-SET 的 TOCTOU 竞争窗口。对于 5 分钟 TTL 的场景可接受——竞争窗口极短，且 TTL 会自愈。

---

## 五、后端变更

### 5.1 `backend/src/router/SyncRouter.ts`

#### `POST /changesets/:id/amend`（约 line 2074）

在现有 `lockCommitForUpdate` → 校验 status 之后，插入锁检查：

```
流程：
1. lockCommitForUpdate（DB 行锁，现有逻辑）
2. 校验 changeset status 可修改（现有逻辑）
3. [新增] orgSlug = getTenantContext().orgSlug
         userId = getReviewerId(req)
         result = acquireOrExtendLock(orgSlug, id, userId)
         若 result.acquired=false → 返回 423
4. 执行 amend（现有逻辑不变）
```

**423 响应格式：**
```json
{
  "error": "Changeset is locked by another user",
  "code": "CHANGESET_LOCKED",
  "lockedBy": "42",
  "lockExpiresAt": "2026-02-27T10:05:00.000Z"
}
```

#### `POST /changesets/:id/publish`（约 line 2524）

```
流程：
1. [新增] getLock(orgSlug, id) → 若被他人锁定 → 返回 423
2. 执行 publish（现有逻辑）
3. [新增] releaseLock(orgSlug, id)（成功后主动释放锁）
```

#### `POST /changesets/:id/discard`（约 line 2602）

```
流程：
1. 执行 discard（现有逻辑）
2. [新增] releaseLock(orgSlug, id)
```

#### `POST /changesets/:id/lock`（新增心跳接口）

```
用途：前端编辑器心跳，防止锁因 5 分钟无编辑而过期
请求：无 body
响应 200：{ lockedBy, lockedAt, lockExpiresAt }
响应 423：{ code: "CHANGESET_LOCKED", lockedBy, lockExpiresAt }（他人已锁）
响应 404：Changeset 不存在
```

#### `GET /changesets` 和 `GET /changesets/:id`

在返回 DB 数据之前，从 Redis 读取锁状态并合并到响应：

```typescript
// 列表接口
const locks = await changesetLockService.getLocksBatch(orgSlug, ids);
const payload = changesets.map(c => ({
    ...c,
    summary: summaries.get(c.id) ?? emptyCommitSummary(),
    ...serializeLock(locks.get(c.id) ?? null),
}));

// 单个接口
const lock = await changesetLockService.getLock(orgSlug, changeset.id);
res.json(withChangesetAlias(changeset, {
    summary: summary ?? emptyCommitSummary(),
    ...serializeLock(lock),
}));
```

`serializeLock(lock | null)` → 有锁时展开三个字段，无锁时返回 `{ lockedBy: null, lockedAt: null, lockExpiresAt: null }`

### 5.2 `backend/src/model/SyncCommit.ts`

**不需要修改**——锁状态完全存储在 Redis，不写入数据库。

---

## 六、公共类型变更

### `common/src/core/SyncChangesetClient.ts`

**`SyncChangeset` 新增锁字段：**

```typescript
export interface SyncChangeset {
    // ...现有字段...
    lockedBy?: string | null | undefined;
    lockedAt?: string | null | undefined;
    lockExpiresAt?: string | null | undefined;
}
```

**`SyncChangesetClient` 新增心跳方法：**

```typescript
export interface SyncChangesetClient {
    // ...现有方法...
    lockChangeset(
        changesetId: number,
        options?: { spaceSlug?: string },
    ): Promise<{ lockedBy: string; lockedAt: string; lockExpiresAt: string }>;
}
```

---

## 七、前端变更

### 7.1 `frontend/src/ui/spaces/hooks/UseChangesetEditor.ts`

#### 心跳机制

当 `activeChangesetId` 存在时（即处于 `editing_changeset` 模式），每 3 分钟调用一次 lock 接口续期：

```typescript
useEffect(() => {
    if (!activeChangesetId) return;
    const intervalId = setInterval(async () => {
        try {
            await client.syncChangesets().lockChangeset(activeChangesetId, scopeOptions);
        } catch (err) {
            log.warn(err, "心跳锁续期失败 changeset %d", activeChangesetId);
        }
    }, 3 * 60 * 1000);
    return () => clearInterval(intervalId);
    // activeChangesetId 变为 undefined 时（switchToLive/publish/discard），心跳自动停止
}, [activeChangesetId, client, scopeOptions]);
```

#### 423 响应处理：fallback 到创建新 Changeset

**`amendChangesetInternal()` 中**，识别 423 并清空 ref：

```typescript
if (response.status === 423) {
    activeChangesetRef.current = null;  // 清空，让上层 fallthrough 到 create
    throw new ChangesetLockedError(data.lockedBy, data.lockExpiresAt);
}
```

**`createChangeset()` 中**，调整 catch 逻辑：

```typescript
try {
    await amendChangesetInternal(currentChangeset, doc, content);
    return currentChangeset.id;
} catch (err) {
    if (err instanceof ChangesetLockedError) {
        // ref 已清空，fallthrough 到下方 create 逻辑，为当前用户创建新 Changeset
        log.warn("Changeset %d 已被锁定，创建新 Changeset", currentChangeset.id);
    } else {
        log.error(err, "amend changeset %d 失败", currentChangeset.id);
        return undefined;  // 其他错误不 fallthrough
    }
}
// fallthrough → 创建新 Changeset
```

### 7.2 `frontend/src/ui/spaces/ChangesetSidebarTab.tsx`

**辅助函数：**

```typescript
function isLockedByOther(
    changeset: SyncChangesetWithSummary,
    currentUserId?: string,
): boolean {
    if (!changeset.lockedBy || !changeset.lockExpiresAt) return false;
    if (new Date(changeset.lockExpiresAt) < new Date()) return false;
    return changeset.lockedBy !== currentUserId;
}
```

**列表项内显示锁图标：**

```typescript
{isLockedByOther(changeset, currentUserId) && (
    <LockClosedIcon
        className="h-3 w-3 text-amber-500"
        aria-label="已被其他用户锁定"
    />
)}
```

**Publish / Discard 按钮禁用条件：**

```typescript
disabled={isPublishing || isLockedByOther(activeChangeset, currentUserId)}
```

`currentUserId` 从现有 auth context 获取。

---

## 八、影响范围

### 需要修改的文件

| 文件 | 变更内容 |
|---|---|
| `backend/src/services/ChangesetLockService.ts` | **新建**：Redis 锁服务 |
| `backend/src/router/SyncRouter.ts` | amend/publish/discard 加锁检查；新增 `POST /lock` 心跳端点；GET 响应合并锁状态 |
| `common/src/core/SyncChangesetClient.ts` | `SyncChangeset` 新增锁字段；`SyncChangesetClient` 新增 `lockChangeset` 方法 |
| `frontend/src/ui/spaces/hooks/UseChangesetEditor.ts` | 心跳 effect；423 fallback create 逻辑 |
| `frontend/src/ui/spaces/ChangesetSidebarTab.tsx` | 锁定图标展示；publish/discard 按钮禁用 |

### 不需要修改的文件

| 文件 | 原因 |
|---|---|
| `backend/src/model/SyncCommit.ts` | 锁不存 DB，无列变更 |
| `backend/src/dao/SyncCommitDao.ts` | 无 DB 变更，无 migration |
| `backend/src/services/CollabArticleSyncService.ts` | AI 工具编辑路径不经过 SyncRouter，暂不加锁 |
| `frontend/src/ui/ArticleDraft.tsx` | 不需要改动 |
| `frontend/src/ui/spaces/Spaces.tsx` | 不需要改动 |

---

## 九、边界情况

| 场景 | 处理方式 |
|---|---|
| Redis 不可用 | `getCache()` 自动 fallback 到 MemoryStore，锁功能在单实例内仍正常工作 |
| 服务器重启 | Redis 持久化开启时锁状态保留；MemoryStore 场景下锁丢失，自然降级为无锁状态 |
| 5 分钟无操作 | Redis TTL 自动过期，无需任何额外处理 |
| switchToLive 后 | 前端心跳停止，5 分钟后 TTL 自动过期 |
| Publish / Discard 后 | 后端主动 `releaseLock()`，锁立即释放，其他用户可立刻操作 |
| 同一用户多标签页 | `lockedBy` 相同，两个 tab 的心跳均可续期，互不冲突 |
| 他人 amend 被锁定 | 前端收到 423，自动 fallback 创建新 Changeset（从 live content 开始） |
| 锁过期后继续编辑 | amend 时 `acquireOrExtendLock` 重新加锁，正常续期，用户无感知 |

---

## 十、验证步骤

1. **基础加锁**：用户 A 编辑文章 → Changeset #5 amend 时 Redis 写入 `cs:lock:{orgSlug}:5`，`ttl ≈ 300s`
2. **心跳续期**：停止编辑 3 分钟 → 心跳触发 `POST /lock`，确认 `ttl` 重置为 300s
3. **他人被拦截**：用户 B 尝试 amend #5 → 后端返回 423 → 前端自动创建新 Changeset #6
4. **API 锁状态**：`GET /changesets` 响应中 #5 包含 `lockedBy`、`lockExpiresAt` 字段
5. **UI 展示**：用户 B 的 ChangesetSidebarTab 中 #5 显示锁图标，publish/discard 按钮 disabled
6. **Publish 释放**：用户 A publish #5 → 后端 `releaseLock()` 立即释放 → 用户 B 可操作
7. **TTL 自动过期**：切换 Live 后心跳停止，5 分钟后 `GET /changesets/:id` 返回 `lockedBy: null`

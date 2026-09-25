---
tags: [api, auth, session, service, explained]
file: src/modules/auth/services/session.service.ts
---

# Session Service — Giải thích chi tiết

> **Mục đích**: Quản lý **session-per-device** với **opaque refresh token** + **rotation** + **reuse detection**. Đây là service phức tạp nhất về mặt logic auth — quyết định toàn bộ security model của refresh flow.

## 📦 Import & Code đầy đủ

```ts
import { Injectable, Logger, UnauthorizedException } from '@nestjs/common';
import { randomBytes, createHash } from 'crypto';
import { RedisService } from 'src/infrastructure/redis/redis.service';
import { RedisKeys } from 'src/common/constants/redis-keys.constant';

export interface SessionData {
  deviceId: string;
  refreshTokenHash: string;
  familyId: string;
  userAgent: string;
  ip: string;
  createdAt: number;
  lastUsedAt: number;
}
```

### Giải thích import

| Import | Vai trò |
|---|---|
| `Injectable` | Provider cho DI |
| `Logger` | Log nội bộ (session created, revoked, token reuse) |
| `UnauthorizedException` | 401 — session không tồn tại/hết hạn |
| `randomBytes` | ⭐ Sinh refresh token + familyId crypto-secure |
| `createHash` | SHA-256 hash refresh token |
| `RedisService` | Wrapper Redis |
| `RedisKeys` | Helper build key |

### `SessionData` interface

| Field | Ý nghĩa |
|---|---|
| `deviceId` | ID device (từ `X-Device-Id` header) |
| `refreshTokenHash` | SHA-256 hash của refresh token hiện tại |
| `familyId` | ID của "token family" — dùng detect reuse |
| `userAgent` | UA của device (audit/UI) |
| `ip` | IP lúc tạo/last use |
| `createdAt` | Timestamp tạo session (ms) |
| `lastUsedAt` | Timestamp lần dùng gần nhất (ms) |

---

## 🏛️ Class Definition

```ts
@Injectable()
export class SessionService {
  private readonly logger = new Logger(SessionService.name);
  private readonly SESSION_TTL = 7 * 24 * 60 * 60; // 7 ngày

  constructor(private readonly redis: RedisService) { }
}
```

### Giải thích

| Field | Value | Ý nghĩa |
|---|---|---|
| `SESSION_TTL` | `7 * 24 * 60 * 60` = 604800 | TTL 7 ngày, tính bằng **giây** (khớp với `redis.setex`) |

> ⚠️ **Inconsistency**: Ở `AuthService` có `REFRESH_TTL_MS = 7 * 24 * 60 * 60 * 1000` (ms) — dùng cho cookie `maxAge`. Ở đây `SESSION_TTL` tính bằng giây. **Cùng 7 ngày nhưng 2 đơn vị khác nhau** → dễ nhầm nếu sửa 1 chỗ quên chỗ kia.

**Fix**: Định nghĩa constant chung ở `common/constants`:
```ts
export const SEVEN_DAYS_SECONDS = 7 * 24 * 60 * 60;
export const SEVEN_DAYS_MS = SEVEN_DAYS_SECONDS * 1000;
```

---

## 🔐 1. HASH TOKEN (PRIVATE)

```ts
private hashToken(token: string): string {
  return createHash('sha256').update(token).digest('hex');
}
```

### Giải thích

| Bước | Ý nghĩa |
|---|---|
| `createHash('sha256')` | Khởi tạo SHA-256 hasher |
| `.update(token)` | Feed token vào |
| `.digest('hex')` | Output hex string (64 ký tự) |

### 🎯 Tại sao SHA-256 mà không bcrypt?

| Tiêu chí | SHA-256 | bcrypt |
|---|---|---|
| **Tốc độ** | ~1μs | ~100ms |
| **Deterministic** | ✅ | ❌ (salt) |
| **Dùng cho password** | ❌ | ✅ |
| **Dùng cho high-entropy token** | ✅ | Overkill |

**Lý do chọn SHA-256**:

1. **Refresh token đã high-entropy**: 48 bytes random từ `crypto.randomBytes` = 2^384 khả năng → **không thể brute force**.
2. **bcrypt chậm có chủ đích** để chống brute force password yếu → không cần thiết cho token random.
3. **Deterministic** cần thiết: cùng input → cùng hash → so sánh nhanh.
4. **Performance**: refresh có thể gọi nhiều lần (auto-refresh mỗi 15m) → SHA-256 nhanh hơn 100,000 lần.

> 🎓 **Nguyên tắc**: Hash function chọn theo **entropy của secret**:
> - Secret người nhập (password) → bcrypt/argon2 (chậm + salt)
> - Secret random (token) → SHA-256 (nhanh, no salt)

### ⚠️ Vấn đề tiềm ẩn

**Hash không có pepper**: nếu Redis bị leak → attacker có hash nhưng không reverse được (vì token random 384-bit). OK.

**Nhưng**: Nếu attacker có cả **Redis + rainbow table** của SHA-256 cho mọi token có thể → không thực tế vì không gian quá lớn.

---

## 🆕 2. CREATE SESSION

```ts
async createSession(
  userId: string,
  deviceId: string,
  userAgent: string,
  ip: string,
): Promise<{ refreshToken: string; familyId: string }> {
  const refreshToken = randomBytes(48).toString('hex');
  const refreshTokenHash = this.hashToken(refreshToken);
  const familyId = randomBytes(16).toString('hex');

  const session: SessionData = {
    deviceId,
    refreshTokenHash,
    familyId,
    userAgent,
    ip,
    createdAt: Date.now(),
    lastUsedAt: Date.now(),
  };

  await this.redis.setJson(
    RedisKeys.SESSION(userId, deviceId),
    session,
    this.SESSION_TTL,
  );

  await this.redis.set(`device:${userId}:${deviceId}`, '1', this.SESSION_TTL);

  await this.redis.set(
    RedisKeys.SESSION_FAMILY(familyId),
    JSON.stringify({ userId, deviceId }),
    this.SESSION_TTL,
  );

  await this.redis.setJson(
    `refresh:${refreshToken}`,
    { userId, deviceId },
    this.SESSION_TTL,
  );

  this.logger.log(
    `Session created: user=${userId} device=${deviceId} family=${familyId}`,
  );
  return { refreshToken, familyId };
}
```

### Giải thích từng bước

#### Bước 1: Sinh token + family

```ts
const refreshToken = randomBytes(48).toString('hex');
const refreshTokenHash = this.hashToken(refreshToken);
const familyId = randomBytes(16).toString('hex');
```

| Token | Size | Output | Entropy |
|---|---|---|---|
| `refreshToken` | 48 bytes | 96 hex chars | 2^384 |
| `familyId` | 16 bytes | 32 hex chars | 2^128 |

**Tại sao 48 bytes cho refresh token?**
- 32 bytes = 256-bit = đủ cho crypto security
- 48 bytes = extra margin, khớp OWASP recommendation cho session token

**Tại sao 16 bytes cho familyId?**
- familyId không phải secret (không dùng để auth)
- Chỉ cần unique, không cần unguessable
- 2^128 = đủ để không collision

#### Bước 2: Tạo SessionData

```ts
const session: SessionData = {
  deviceId,
  refreshTokenHash,
  familyId,
  userAgent,
  ip,
  createdAt: Date.now(),
  lastUsedAt: Date.now(),
};
```

**Lưu ý**:
- `refreshTokenHash` — **KHÔNG** lưu plain token
- `createdAt` = `lastUsedAt` lúc tạo
- `deviceId` dư thừa (đã có trong key) nhưng giữ để payload tự chứa

#### Bước 3: Lưu 4 keys vào Redis

Đây là **điểm cốt lõi** của session-per-device:

**Key 1 — Session chính**:
```ts
RedisKeys.SESSION(userId, deviceId) → `session:{userId}:{deviceId}`
```
- Value: `SessionData` JSON
- Dùng để: `getSession`, `verifyRefreshToken`, `rotateSession`
- Đây là **source of truth**

**Key 2 — Device tracking**:
```ts
`device:${userId}:${deviceId}` → '1'
```
- Value: `'1'` (placeholder)
- Dùng để: `revokeAllUserSessions` scan `device:{userId}:*`
- **Tại sao cần key riêng?** Vì Redis `SCAN` match theo **key pattern**, không theo **value**. Nếu chỉ có `session:{userId}:*` → đã scan được. Nhưng device key cho phép:
  - Liệt kê device đang active (không cần parse session data)
  - TTL độc lập (nếu muốn)

> ⚠️ **Redundant?** Thực tế `device:` key là **dư thừa** — `session:{userId}:*` đã đủ để scan. Nhưng giữ để dễ query theo device.

**Key 3 — Family tracking**:
```ts
RedisKeys.SESSION_FAMILY(familyId) → `family:{familyId}`
```
- Value: `{userId, deviceId}` JSON
- Dùng để: khi revoke, biết family thuộc user nào

**Key 4 — Reverse lookup (token → identity)**:
```ts
`refresh:${refreshToken}` → {userId, deviceId}
```
- Value: `{userId, deviceId}` JSON
- Dùng để: controller resolve token (không biết userId/deviceId từ cookie) → lookup Redis
- **Đây là key quan trọng nhất cho refresh flow**

#### Bước 4: Log + return

```ts
this.logger.log(`Session created: user=${userId} device=${deviceId} family=${familyId}`);
return { refreshToken, familyId };
```

- **Log**: audit trail (không log token)
- **Return plain refreshToken**: chỉ trả **1 lần duy nhất** → service caller (AuthService) set vào cookie

### 🎯 Sơ đồ 4 keys

```
                          userId=u1, deviceId=d1, familyId=f1
                                        │
        ┌───────────────────────────────┼───────────────────────────────┐
        │                               │                               │
        ▼                               ▼                               ▼
┌─────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│ session:u1:d1       │  │ device:u1:d1         │  │ family:f1            │
│ {deviceId, hash,    │  │ "1"                  │  │ {userId: u1,         │
│  familyId, ua, ip,  │  │                      │  │  deviceId: d1}       │
│  createdAt, ...}    │  │                      │  │                      │
│ TTL 7d              │  │ TTL 7d               │  │ TTL 7d               │
└─────────────────────┘  └──────────────────────┘  └──────────────────────┘

                            ┌──────────────────────────┐
                            │ refresh:{plainToken}     │
                            │ {userId: u1,             │
                            │  deviceId: d1}           │
                            │ TTL 7d                   │
                            └──────────────────────────┘
```

### ⚠️ Vấn đề về số lượng keys

**4 keys cho 1 session**. Nếu user có 10 device → 40 keys. Với 1M user × 3 device = **12M keys**.

**Trade-off**:
- ✅ Fast lookup theo nhiều chiều (userId+device, familyId, token)
- ❌ Tốn memory, phức tạp revoke (phải xóa nhiều keys)

**Alternative**: Dùng Redis Hash (`HSET session:{userId} {deviceId} "{...}"`) → 1 key/user, field/device.
- ✅ Ít keys hơn
- ❌ Không set TTL per field (chỉ per key)
- ❌ Khó scan theo device

→ **Cách hiện tại OK** cho scale vừa (< 100k user).

---

## 🔍 3. RESOLVE REFRESH TOKEN

```ts
async resolveRefreshToken(
  refreshToken: string,
): Promise<{ userId: string; deviceId: string } | null> {
  return this.redis.getJson<{ userId: string; deviceId: string }>(
    `refresh:${refreshToken}`,
  );
}
```

### Giải thích

- **Mục đích**: Controller cần biết `{userId, deviceId}` từ cookie `refreshToken` (vì cookie không chứa info này).
- **Input**: plain refresh token từ cookie
- **Output**: `{userId, deviceId}` hoặc `null` nếu token không tồn tại/hết hạn

### 🎯 Flow sử dụng

```
Client ──cookie refreshToken──▶ Controller
                                    │
                                    ▼
                    resolveRefreshToken(token)
                                    │
                                    ▼
                    Redis GET refresh:{token}
                                    │
                                    ▼
                    {userId, deviceId} → AuthService.refreshTokens()
```

### ⚠️ Vấn đề

**Lookup theo plain token** — nếu ai đó leak được list key của Redis (`KEYS refresh:*`), họ có **tất cả token plain** → full account takeover.

**Fix đề xuất**: Hash token trước khi dùng làm key:
```ts
async resolveRefreshToken(token: string) {
  const hash = this.hashToken(token);
  return this.redis.getJson(`refresh:${hash}`);
}

// createSession:
await this.redis.setJson(
  `refresh:${this.hashToken(refreshToken)}`,
  {userId, deviceId},
  this.SESSION_TTL,
);
```

→ Redis leak → attacker chỉ có hash → không dùng được.

**Trade-off**: Thêm 1 SHA-256 call mỗi refresh (~1μs) — không đáng kể.

---

## 📖 4. GET SESSION

```ts
async getSession(
  userId: string,
  deviceId: string,
): Promise<SessionData | null> {
  return this.redis.getJson<SessionData>(RedisKeys.SESSION(userId, deviceId));
}
```

### Giải thích

- **Pure lookup** từ `session:{userId}:{deviceId}`
- **Không throw** nếu không tồn tại → trả `null` → caller quyết định

### 🎯 Caller nào dùng?

| Caller | Cách dùng |
|---|---|
| `verifyRefreshToken` | `getSession` → so sánh hash |
| `rotateSession` | `getSession` → throw nếu null → update |
| `revokeSession` | `getSession` → lấy familyId để xóa |

---

## 🔄 5. ROTATE SESSION

```ts
async rotateSession(
  userId: string,
  deviceId: string,
  userAgent: string,
  ip: string,
): Promise<{ refreshToken: string; familyId: string }> {
  const existing = await this.getSession(userId, deviceId);
  if (!existing) {
    throw new UnauthorizedException('Session not found or expired');
  }

  const newRefreshToken = randomBytes(48).toString('hex');
  const newHash = this.hashToken(newRefreshToken);

  const updated: SessionData = {
    ...existing,
    refreshTokenHash: newHash,
    lastUsedAt: Date.now(),
    userAgent,
    ip,
  };

  await this.redis.setJson(
    RedisKeys.SESSION(userId, deviceId),
    updated,
    this.SESSION_TTL,
  );

  await this.redis.setJson(
    `refresh:${newRefreshToken}`,
    { userId, deviceId },
    this.SESSION_TTL,
  );

  return { refreshToken: newRefreshToken, familyId: existing.familyId };
}
```

### Giải thích từng bước

#### Bước 1: Load session hiện tại

```ts
const existing = await this.getSession(userId, deviceId);
if (!existing) {
  throw new UnauthorizedException('Session not found or expired');
}
```

- Nếu session hết hạn (7 ngày) hoặc đã revoke → throw 401
- **Defensive**: `AuthService.refreshTokens` đã verify trước đó, nhưng vẫn check lại

#### Bước 2: Sinh token mới

```ts
const newRefreshToken = randomBytes(48).toString('hex');
const newHash = this.hashToken(newRefreshToken);
```

- **Token mới hoàn toàn random** — không derive từ token cũ
- Hash mới để so sánh lần sau

#### Bước 3: Update session

```ts
const updated: SessionData = {
  ...existing,              // giữ familyId, createdAt, deviceId
  refreshTokenHash: newHash, // ← update
  lastUsedAt: Date.now(),    // ← update
  userAgent,                 // ← update (UA có thể đổi)
  ip,                        // ← update (IP có thể đổi)
};
```

- **KHÔNG** update: `familyId` (giữ nguyên → track family xuyên rotation), `createdAt` (nguyên gốc), `deviceId`
- **Update**: hash, lastUsedAt, UA, IP

#### Bước 4: Ghi lại Redis

```ts
await this.redis.setJson(
  RedisKeys.SESSION(userId, deviceId),
  updated,
  this.SESSION_TTL,
);
```

**⚠️ Quan trọng**: `setJson` với TTL = **reset TTL về 7 ngày**.

**Hệ quả**: Nếu user active đều đặn (refresh mỗi 15m), session **không bao giờ hết hạn**. Đây là **sliding expiration** — đúng behavior cho UX tốt.

**Nhưng**: Nếu muốn absolute expiration (session cứng 7 ngày dù active), phải check `createdAt`:
```ts
if (Date.now() - existing.createdAt > this.SESSION_TTL * 1000) {
  throw new UnauthorizedException('Session absolute TTL exceeded');
}
```

→ **Hiện code không có** → session có thể sống vô hạn nếu user active.

#### Bước 5: Lưu mapping token mới

```ts
await this.redis.setJson(
  `refresh:${newRefreshToken}`,
  { userId, deviceId },
  this.SESSION_TTL,
);
```

### ⚠️🔴 VẤN ĐỀ NGHIÊM TRỌNG: Token cũ không bị xóa

**Code comment nói**:
```ts
// Token cũ vẫn còn mapping `refresh:{oldToken}` → tự hết hạn theo TTL
// Nếu attacker dùng lại token cũ → resolve OK → verify fail → revoke all
```

**Phân tích**:

1. **Attacker có token cũ** (bị leak) → gọi `/refresh` với token cũ
2. **`resolveRefreshToken(oldToken)`** → trả `{userId, deviceId}` (vì mapping còn) ✅
3. **`verifyRefreshToken(userId, deviceId, oldToken)`**:
   ```ts
   const session = await this.getSession(userId, deviceId);
   return this.hashToken(oldToken) === session.refreshTokenHash;
   // SHA256(oldToken) !== SHA256(newToken) → FALSE
   ```
4. → Verify fail → `AuthService.refreshTokens` → **revoke ALL** + clear cookie

**Vậy code hoạt động đúng** ✅ — attacker không lấy được session, user thật bị logout.

### ⚠️ Nhưng có DoS vector

**Kịch bản**:
1. Attacker sniff được 1 token cũ (hoặc đoán được 1 token cũ từ log)
2. Attacker gọi `/refresh` với token cũ **liên tục**
3. Mỗi lần → trigger `revokeAllUserSessions` → user thật **không thể dùng app**

**Rate limit 20/1m** trên route refresh giảm thiểu nhưng không triệt để.

**Fix**:
```ts
// Xóa mapping token cũ ngay khi rotate
await this.redis.del(`refresh:${oldToken}`); // ⚠️ cần truyền oldToken vào rotate
```

**Nhưng** — nếu xóa mapping token cũ → khi attacker dùng lại → `resolveRefreshToken` trả `null` → **KHÔNG** trigger revoke all (chỉ trả 401 đơn giản) → **mất khả năng detect reuse!**

→ **Trade-off thực sự**:

| Approach | Detect reuse? | DoS? |
|---|---|---|
| Không xóa mapping cũ (hiện tại) | ✅ | ⚠️ Có |
| Xóa mapping cũ | ❌ | ✅ Không |
| **Giữ mapping nhưng mark "used"** | ✅ | ✅ Không |

**Cách tốt nhất**: Lưu mapping token cũ với flag `used: true`:
```ts
// Sau khi rotate:
await this.redis.setJson(
  `refresh:${oldToken}`,
  { userId, deviceId, used: true, replacedBy: newRefreshToken },
  this.SESSION_TTL,
);
```

Rồi trong `resolveRefreshToken`:
```ts
if (resolved?.used) {
  // Reuse detected!
  await this.revokeAllUserSessions(resolved.userId);
  throw new ForbiddenException('Token reuse detected');
}
```

→ Vẫn detect được, nhưng chỉ revoke **1 lần** (vì sau đó user login lại).

**Hoặc đơn giản hơn**: Chỉ revoke **session hiện tại** (không phải all) khi reuse:
```ts
await this.revokeSession(userId, deviceId);
```

→ Giảm DoS, nhưng vẫn đủ để đá attacker ra.

### 🎯 Tại sao dùng `familyId`?

**Ý tưởng**: `familyId` đại diện cho **1 "chuỗi" rotation** từ 1 lần login.

```
Login → token1 (family=f1)
    ↓ rotate
       token2 (family=f1) ← cùng family
    ↓ rotate
       token3 (family=f1)
    ↓ rotate
       token4 (family=f1)
```

**Ứng dụng**: Nếu phát hiện `token2` bị dùng lại → biết **cả family f1 bị compromise** → revoke cả family.

**Nhưng code hiện tại**: `family:{familyId}` chỉ lưu `{userId, deviceId}` — **không lưu list token trong family** → không thể revoke theo family.

**Fix**:
```ts
// Lưu Set các token hash trong family
await this.redis.set.add(`family:${familyId}:tokens`, hash);
```

→ Khi revoke family → xóa hết token trong set.

---

## ✅ 6. VERIFY REFRESH TOKEN

```ts
async verifyRefreshToken(
  userId: string,
  deviceId: string,
  plainToken: string,
): Promise<boolean> {
  const session = await this.getSession(userId, deviceId);
  if (!session) return false;
  return this.hashToken(plainToken) === session.refreshTokenHash;
}
```

### Giải thích

- Load session → compare hash
- **KHÔNG throw** — trả `false` để caller quyết định (giúp `AuthService` trigger revoke all riêng)

### 🎯 Tại sao dùng `plainToken` thay vì `hashedToken`?

Vì caller (controller) nhận token plain từ cookie → không cần hash ở ngoài → service chịu trách nhiệm hash.

### ⚠️ Timing attack?

`===` trên hex string (64 chars) → có thể leak timing nếu attacker đo được. Nhưng:
- SHA-256 deterministic → không thể brute force
- Timing leak chỉ lộ **độ dài prefix khớp** → không hữu ích để tìm hash đúng

→ Không cần `timingSafeEqual` ở đây.

---

## 🚪 7. REVOKE SESSION (1 DEVICE)

```ts
async revokeSession(userId: string, deviceId: string): Promise<void> {
  const session = await this.getSession(userId, deviceId);
  if (session) {
    await this.redis.del(RedisKeys.SESSION_FAMILY(session.familyId));
  }
  await this.redis.del(
    RedisKeys.SESSION(userId, deviceId),
    `device:${userId}:${deviceId}`,
  );
}
```

### Giải thích từng bước

#### Bước 1: Lấy familyId

```ts
const session = await this.getSession(userId, deviceId);
if (session) {
  await this.redis.del(RedisKeys.SESSION_FAMILY(session.familyId));
}
```

- Cần familyId để xóa `family:{familyId}`
- **Nếu session null** → skip (không có gì để revoke)

#### Bước 2: Xóa các keys

```ts
await this.redis.del(
  RedisKeys.SESSION(userId, deviceId),
  `device:${userId}:${deviceId}`,
);
```

- Xóa `session:{userId}:{deviceId}` + `device:{userId}:{deviceId}`

### ⚠️ VẤN ĐỀ NGHIÊM TRỌNG: `refresh:{token}` KHÔNG bị xóa

**Phân tích**:

```ts
// Keys bị xóa:
// - session:{userId}:{deviceId}  ✅
// - device:{userId}:{deviceId}   ✅
// - family:{familyId}            ✅

// Keys KHÔNG bị xóa:
// - refresh:{plainToken}         ❌ ← VẪN CÒN!
```

**Hệ quả**:

1. User logout → `revokeSession` → session bị xóa
2. **Attacker có refresh token** (đã leak trước khi logout) → gọi `/refresh`
3. `resolveRefreshToken(oldToken)` → **VẪN RESOLVE ĐƯỢC** (mapping còn) → trả `{userId, deviceId}`
4. `verifyRefreshToken` → `getSession` → **null** → return `false`
5. → `AuthService` → **revoke ALL sessions** (nhưng đã logout rồi) + clear cookie

**Vấn đề**: Attacker không lấy được session (vì verify fail) → **OK về mặt security** ✅

**Nhưng**: Nếu sau khi logout, user **login lại** (tạo session mới cùng deviceId) → attacker dùng refresh token **cũ**:
- `resolveRefreshToken(oldToken)` → trả `{userId, deviceId}` (cùng deviceId!)
- `verifyRefreshToken(userId, deviceId, oldToken)` → load **session mới** → `SHA256(oldToken) !== newSession.refreshTokenHash` → FALSE
- → revoke all

→ Vẫn an toàn ✅ nhưng có DoS.

### ✅ Fix đề xuất

**Xóa `refresh:{token}` khi revoke**:

```ts
async revokeSession(userId: string, deviceId: string): Promise<void> {
  const session = await this.getSession(userId, deviceId);
  if (session) {
    await this.redis.del(RedisKeys.SESSION_FAMILY(session.familyId));
    // ⚠️ Không có plain token ở đây → không xóa được mapping!
  }
  await this.redis.del(
    RedisKeys.SESSION(userId, deviceId),
    `device:${userId}:${deviceId}`,
  );
}
```

**Vấn đề**: `revokeSession` **không có plain token** → không biết key `refresh:?` để xóa.

**Giải pháp**: Lưu mapping `session → token hash` để lookup:

```ts
// Trong createSession:
await this.redis.setJson(
  RedisKeys.SESSION(userId, deviceId),
  { ...session, currentTokenHash: refreshTokenHash },
  this.SESSION_TTL,
);

// Trong revokeSession:
if (session?.currentTokenHash) {
  await this.redis.del(`refresh:${session.currentTokenHash}`);
}
```

Nhưng `refresh:{hash}` không phải key hiện tại (đang là `refresh:{plainToken}`) → phải đổi cả 2 chỗ.

**Hoặc**: Lưu plain token trong session (⚠️ kém an toàn hơn):
```ts
session: { ..., currentRefreshToken: plainToken }
```
→ `revokeSession` biết key để xóa. Nhưng nếu Redis leak → attacker có plain token (không cần hash nữa) → tệ hơn.

**Cách tốt nhất** (đã đề cập): Dùng **hash làm key**:
```ts
// createSession:
const hash = this.hashToken(refreshToken);
await this.redis.setJson(`refresh:${hash}`, {userId, deviceId}, ttl);
// session lưu hash (đã có)
// revokeSession:
if (session?.refreshTokenHash) {
  await this.redis.del(`refresh:${session.refreshTokenHash}`);
}
```

→ **Cần refactor cả 3 chỗ**: `createSession`, `resolveRefreshToken`, `revokeSession`.

---

## 🚪🚪 8. REVOKE ALL USER SESSIONS

```ts
async revokeAllUserSessions(userId: string): Promise<void> {
  const keys = await this.redis.scanKeys(`session:${userId}:*`);
  if (keys.length > 0) {
    await this.redis.del(...keys);
  }
  const deviceKeys = await this.redis.scanKeys(`device:${userId}:*`);
  if (deviceKeys.length > 0) {
    await this.redis.del(...deviceKeys);
  }
  this.logger.warn(`All sessions revoked for user ${userId}`);
}
```

### Giải thích

- Dùng `scanKeys` (non-blocking SCAN) → không giết Redis
- Xóa session + device keys
- **KHÔNG** xóa `family:{familyId}` hoặc `refresh:{token}` ❌

### ⚠️ VẤN ĐỀ: Family + Refresh keys bị orphan

**Sau khi revoke all**:
- `family:{familyId}` — vẫn còn trong Redis tới khi TTL hết (7 ngày)
- `refresh:{token}` — vẫn còn → `resolveRefreshToken` vẫn trả về `{userId, deviceId}`

**Hệ quả**: Attacker có token cũ → resolve OK → verify fail (session null) → **revoke all lần nữa** (vô nghĩa nhưng tốn Redis calls).

**Fix**: Cần lưu list families + tokens của user:

```ts
// Khi createSession:
await this.redis.set.add(`user:${userId}:families`, familyId);
// (cần thêm method `sadd` vào RedisService)

// Khi revokeAll:
const families = await this.redis.smembers(`user:${userId}:families`);
for (const fid of families) {
  const tokens = await this.redis.smembers(`family:${fid}:tokens`);
  // ...xóa hết
}
```

### 🎯 Tại sao dùng SCAN thay vì KEYS?

**`KEYS pattern`**:
- **Blocking** — chặn toàn bộ Redis cho tới khi scan xong
- Với 1M keys → có thể block vài giây
- **Không dùng trong production**

**`SCAN cursor MATCH pattern COUNT n`**:
- **Non-blocking** — trả về từng batch
- An toàn cho production
- Có thể có false positive (keys thay đổi giữa các lần scan)

→ Code dùng SCAN đúng ✅.

### ⚠️ `del(...keys)` với mảng lớn

```ts
await this.redis.del(...keys);
```

**Vấn đề**: `DEL key1 key2 ... keyN` — Redis xử lý **synchronously**.
- Nếu user có 1000 sessions → `DEL` 1000 keys → block Redis ~vài ms
- Nếu 10,000 keys → block ~50ms
- Nếu 100,000 keys → block ~500ms ⚠️

**Fix**: Chunk theo batch:
```ts
const CHUNK_SIZE = 100;
for (let i = 0; i < keys.length; i += CHUNK_SIZE) {
  await this.redis.del(...keys.slice(i, i + CHUNK_SIZE));
}
```

**Hoặc** dùng `UNLINK` (non-blocking delete):
```ts
await this.redis.unlink(...keys);
```

→ Redis mark keys để xóa async → không block main thread.

---

## 📋 9. LIST SESSIONS

```ts
async listSessions(userId: string): Promise<SessionData[]> {
  const keys = await this.redis.scanKeys(`session:${userId}:*`);
  const sessions: SessionData[] = [];
  for (const key of keys) {
    const s = await this.redis.getJson<SessionData>(key);
    if (s) sessions.push(s);
  }
  return sessions;
}
```

### Giải thích

- Scan `session:{userId}:*` → list keys
- Loop get từng key → parse JSON
- Filter null (key hết hạn giữa lúc scan)

### ⚠️ N+1 problem

**Với user có 10 sessions**: 1 SCAN + 10 GET = **11 round-trips**.

**Fix**: Dùng `MGET`:
```ts
async listSessions(userId: string): Promise<SessionData[]> {
  const keys = await this.redis.scanKeys(`session:${userId}:*`);
  if (keys.length === 0) return [];
  
  const values = await this.redis.mget(keys); // 1 round-trip
  return values
    .filter((v): v is string => v !== null)
    .map(v => JSON.parse(v) as SessionData);
}
```

**Cần thêm `mget` vào RedisService**. Với user bình thường (1-5 devices) → không đáng kể, nhưng nếu có user "power user" 20+ devices → đáng fix.

### ⚠️ SCAN consistency

`SCAN` **không đảm bảo consistency**:
- Giữa các lần scan, keys có thể thêm/xóa
- Có thể miss keys hoặc trả duplicate

Với `listSessions` (read-only, không critical) → OK.

---

## 🎯 Tổng kết: Kiến trúc session

### Sơ đồ tổng quát

```
┌─────────────────────────────────────────────────────────────┐
│                    USER (userId=u1)                          │
│                                                              │
│  Device d1                    Device d2                      │
│  family f1                    family f2                      │
│  ┌─────────────────┐          ┌─────────────────┐            │
│  │ session:u1:d1   │          │ session:u1:d2   │            │
│  │ device:u1:d1    │          │ device:u1:d2    │            │
│  │ family:f1       │          │ family:f2       │            │
│  │ refresh:tok1    │          │ refresh:tok2    │            │
│  └─────────────────┘          └─────────────────┘            │
│                                                              │
│  Rotation: tok1 → tok1' → tok1'' (family f1 không đổi)      │
│             refresh:tok1 vẫn còn (TTL còn)                   │
│             refresh:tok1' vẫn còn                            │
│             refresh:tok1'' là token hiện tại                 │
└─────────────────────────────────────────────────────────────┘
```

### Bảo mật features

| Feature | Implementation | File |
|---|---|---|
| Opaque refresh token | 48 bytes random | `createSession` |
| Hash lưu Redis | SHA-256 | `hashToken` |
| Token rotation | Mỗi refresh tạo token mới | `rotateSession` |
| Reuse detection | Hash mismatch → revoke all | `verifyRefreshToken` + `AuthService` |
| Family tracking | `family:{familyId}` | `createSession` |
| Session per device | Key `session:{userId}:{deviceId}` | `createSession` |
| Sliding expiration | Reset TTL mỗi rotate | `rotateSession` |
| Non-blocking scan | SCAN thay KEYS | `scanKeys` |

---

## ⚠️ Danh sách vấn đề cần fix

| # | Vấn đề | Mức độ | Fix |
|---|---|---|---|
| 1 | `refresh:{plainToken}` không bị xóa khi revoke → orphan + DoS | 🔴 Cao | Dùng hash làm key + xóa khi revoke |
| 2 | `family:{familyId}` bị orphan khi revokeAll | 🟠 Trung | Track family list per user |
| 3 | Không có absolute expiration (session sống mãi nếu active) | 🟠 Trung | Check `createdAt` trong `rotateSession` |
| 4 | `del(...keys)` với mảng lớn → block Redis | 🟡 Thấp | Chunk hoặc `UNLINK` |
| 5 | `listSessions` N+1 (SCAN + N GET) | 🟡 Thấp | Dùng `MGET` |
| 6 | `SESSION_TTL` (giây) vs `REFRESH_TTL_MS` (ms) — dễ nhầm | 🟡 Thấp | Constant chung |
| 7 | Family không track list token → không revoke được theo family | 🟠 Trung | Lưu `family:{fid}:tokens` set |
| 8 | `revokeAllUserSessions` không xóa `refresh:*` → có thể revoke all nhiều lần | 🟠 Trung | Fix cùng #1 |
| 9 | `device:` key dư thừa (đã có trong `session:`) | 🟢 Info | Cân nhắc bỏ |

---

## 🧪 Test scenarios nên có

```ts
describe('SessionService', () => {
  it('should create session with 4 Redis keys', ...);
  it('should return plain refreshToken only once', ...);
  it('should resolve refresh token to userId + deviceId', ...);
  it('should verify correct refresh token', ...);
  it('should reject wrong refresh token', ...);
  it('should rotate token and keep familyId', ...);
  it('should update lastUsedAt on rotate', ...);
  it('should revoke single session', ...);
  it('should revoke all user sessions', ...);
  it('should list all sessions of user', ...);
  it('should return null for revoked session', ...);
  it('should handle 1000 sessions efficiently', ...); // perf test
  it('should detect token reuse after rotation', ...); // integration
  it('should NOT allow old token to verify after revoke', ...);
});
```

---

## 🔗 Related
- [[Auth-Module]]
- [[Auth-Service-Explained]]
- [[Otp-Service-Explained]]
- [[Redis-Keys]]
- [[Security]]
- [[Phase-1-Auth]]
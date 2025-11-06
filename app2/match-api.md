# Match API 完整文档 v2.0

> **版本说明**：本文档基于 v1.4.0 API，优化了 WebSocket 消息类型的展示方式，使用更清晰的类型定义。

## 📖 目录

- [概述](#概述)
- [认证](#认证)
- [响应格式](#响应格式)
- [错误码](#错误码)
- [HTTP API 端点](#http-api-端点)
  - [创建对局](#1-创建对局)
  - [加入对局](#2-加入对局)
  - [开始对局](#3-开始对局)
  - [批量记分](#4-批量记分)
  - [单条记分](#5-单条记分)
  - [结束对局](#6-结束对局)
  - [获取对局详情](#7-获取对局详情)
  - [获取结算快照](#8-获取结算快照)
  - [获取计分板](#9-获取计分板)
  - [获取转移记录](#10-获取转移记录)
  - [检查活跃对局](#11-检查活跃对局)
  - [检查Live对局](#12-检查live对局)
  - [获取结算记录](#13-获取结算记录)
  - [获取对局历史](#14-获取对局历史)
- [WebSocket API](#websocket-api)
  - [连接信息](#连接信息)
  - [消息类型定义](#消息类型定义)
  - [客户端发送消息](#客户端发送消息)
  - [服务端推送消息](#服务端推送消息)
  - [完整示例](#websocket-完整示例)
- [数据模型](#数据模型)
- [安全与性能保障](#安全与性能保障)
- [最佳实践](#最佳实践)
- [版本更新记录](#版本更新记录)

---

## 概述

Match API 提供了完整的对局管理功能，包括创建、加入、游戏进行中的记分、实时计分板更新以及结算等。

### 核心特性

- ✅ **并发安全**: 使用双锁机制防止竞态条件（用户锁+Match锁）
- ✅ **幂等性**: 使用`clientTxnId`保证记分幂等
- ✅ **实时更新**: WebSocket实时推送计分板和状态变化
- ✅ **自动过期**: 支持滑动过期（120分钟）和绝对时间上限（720分钟）
- ✅ **多人游戏**: 支持2-12人的对局
- ✅ **错误统一**: 使用ExceptionUtil统一错误处理
- ✅ **数据一致性**: Redis缓存与数据库事务保护
- ✅ **容错降级**: Redis故障不影响核心业务
- ✅ **资源保护**: 批量大小限制、过期时间上限
- ✅ **性能优化**: 批量操作、N+1查询优化

### 安全限制

- 📌 **过期时间上限**: 最大30天（43200分钟）
- 📌 **批量记分上限**: 每次最多100条记录
- 📌 **并发控制**: 防止对局超员（双锁机制）
- 📌 **事务保护**: Match创建、记分操作使用事务
- 📌 **幂等保证**: clientTxnId防止重复提交

### 对局生命周期

```
waiting → playing → finished/expired
   ↓         ↓           ↓
 创建/加入  开始/记分   结算/查询
```

---

## 认证

所有API端点都需要JWT Bearer Token认证。

### 请求头

```http
Authorization: Bearer <access_token>
```

### 用户标识

- `req.user.sub`: 当前登录用户的userId（UUID字符串）

---

## 响应格式

### HTTP 状态码说明

⚠️ **重要**: 除了 `5xx` 服务器内部错误外，所有响应的 **HTTP 状态码都是 200**。

- ✅ **业务成功**: HTTP 200 + `code: 0`
- ⚠️ **业务错误**: HTTP 200 + `code: 4xxx`（通过 `code` 字段区分错误类型）
- ❌ **服务器错误**: HTTP 500 + `code: 5xxx`

### 成功响应

所有成功响应都遵循统一格式：

```typescript
interface SuccessResponse<T> {
  code: 0;
  success: true;
  data: T;
  ts: number;  // 时间戳（毫秒）
}
```

**示例**：
```json
{
  "code": 0,
  "success": true,
  "data": {
    "match": { /* ... */ },
    "participant": { /* ... */ }
  },
  "ts": 1704110400000
}
```

### 错误响应

```typescript
interface ErrorResponse {
  code: number;        // 错误码（4xxx或5xxx）
  data: null;          // 错误时为 null
  success: false;
  message: string;     // 错误描述
  ts: number;          // 时间戳（毫秒）
  details?: any;       // 可选，包含额外错误信息
}
```

**示例**：
```json
{
  "code": 4001,
  "data": null,
  "success": false,
  "message": "Match not found",
  "ts": 1704110400000
}
```

---

## 错误码

### 通用错误码 (4000-4099)

| 错误码 | 错误名称 | 说明 | HTTP状态 |
|--------|---------|------|---------|
| 4000 | `BAD_REQUEST` | 请求参数无效 | 200 |
| 4001 | `MATCH_NOT_FOUND` | 资源不存在 | 对局不存在 |
| 4002 | `MATCH_ALREADY_STARTED` | 状态冲突 | 对局已开始 |
| 4003 | `MATCH_ALREADY_ENDED` | 状态冲突 | 对局已结束 |
| 4004 | `MATCH_ALREADY_EXPIRED` | 状态冲突 | 对局已过期 |
| 4005 | `MATCH_NOT_ACTIVE` | 状态错误 | 对局未激活 |
| 4006 | `MATCH_PLAYER_LIMIT_EXCEEDED` | 业务限制 | 对局人数已满 |
| 4007 | `MATCH_PLAYER_ALREADY_JOINED` | 状态冲突 | 用户已在对局中 |
| 4008 | `MATCH_PLAYER_NOT_PARTICIPANT` | 权限不足 | 用户不是对局参与者 |
| 4009 | `MATCH_CREATOR_ONLY` | 权限不足 | 仅创建者可操作 |
| 4010 | `MATCH_MIN_PLAYERS_NOT_MET` | 业务限制 | 对局人数不足 |
| 4011 | `MATCH_TRANSFER_INVALID` | 参数错误 | 记分无效 |
| 4012 | `MATCH_TRANSFER_DUPLICATE` | 状态冲突 | 记分重复 |
| 4013 | `MATCH_TRANSFER_SELF_TRANSFER` | 参数错误 | 不能给自己记分 |
| 4014 | `MATCH_TRANSFER_INVALID_RECIPIENT` | 参数错误 | 记分接收方无效 |
| 4015 | `MATCH_TRANSFER_INVALID_POINTS` | 参数错误 | 记分点数无效 |
| 4016 | `MATCH_SCOREBOARD_ERROR` | 服务器错误 | 计分板错误（HTTP 500） |
| 4017 | `MATCH_EXPIRY_JOB_FAILED` | 服务器错误 | 对局过期任务失败（HTTP 500） |
| 4018 | `MATCH_USER_IN_OTHER_MATCH` | 状态冲突 | 用户已在其他对局中 |

### 安全错误码 (5000-5099)

| 错误码 | 错误名称 | 说明 | HTTP状态 |
|--------|---------|------|---------|
| 5001 | `UNAUTHORIZED` | 未授权 | 401 |
| 5006 | `SECURITY_RATE_LIMIT` | 请求频率限制 | 429 |

### 系统错误码 (5100-5199)

| 错误码 | 错误名称 | 说明 | HTTP状态 |
|--------|---------|------|---------|
| 5000 | `INTERNAL_SERVER_ERROR` | 服务器内部错误 | 500 |

---

## HTTP API 端点

> **基础 URL**: `/api/matches`

### 快速参考表

| 方法 | 端点 | 说明 | 权限要求 |
|------|------|------|---------|
| POST | `/matches` | 创建对局 | 认证用户 |
| POST | `/matches/:id/join` | 加入对局 | 认证用户 |
| POST | `/matches/:id/start` | 开始对局 | 创建者 |
| POST | `/matches/:id/transfers` | 批量记分 | 参与者 |
| POST | `/matches/:id/transfer` | 单条记分 | 参与者 |
| POST | `/matches/:id/end` | 结束对局 | 创建者 |
| GET | `/matches/:id` | 获取对局详情 | 参与者 |
| GET | `/matches/:id/summary` | 获取结算快照 | 参与者 |
| GET | `/matches/:id/scoreboard` | 获取计分板 | 参与者 |
| GET | `/matches/:id/transfers` | 获取转移记录 | 参与者 |
| GET | `/matches/me/active` | 检查活跃对局 | 认证用户 |
| GET | `/matches/me/live` | 检查Live对局 | 认证用户 |
| GET | `/matches/me/records` | 获取结算记录 | 认证用户 |
| GET | `/matches/me/history` | 获取对局历史 | 认证用户 |

### 1. 创建对局

创建一个新的对局房间，创建者自动成为第一个参与者。

**接口**
```http
POST /api/matches
```

**请求类型定义**
```typescript
interface CreateMatchDto {
  name: string;               // 对局名称，1-64字符
  maxPlayers?: number;        // 最大人数，2-12（可选，默认4）
  expiresInMinutes?: number;  // 过期时间（分钟），最大43200（30天，可选，默认120）
}
```

**请求示例**
```http
POST /matches
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "name": "我的房间",
  "maxPlayers": 4,
  "expiresInMinutes": 120
}
```

**响应类型定义**
```typescript
interface CreateMatchResponse {
  id: string;              // UUID
  name: string;
  status: 'waiting';       // 固定为waiting
  maxPlayers: number;
  creatorUserId: string;   // UUID
  startedAt: string | null;  // 开始时间
  endedAt: string | null;    // 结束时间
  expiresAt: string;       // ISO 8601
  createdAt: string;       // ISO 8601
  updatedAt: string;       // ISO 8601
}
```

**响应示例**
```json
{
  "code": 0,
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "我的麻将房",
    "status": "waiting",
    "maxPlayers": 4,
    "creatorUserId": "user-uuid-123",
    "startedAt": null,
    "endedAt": null,
    "expiresAt": "2024-01-01T14:00:00.000Z",
    "createdAt": "2024-01-01T12:00:00.000Z",
    "updatedAt": "2024-01-01T12:00:00.000Z"
  },
  "ts": 1704110400000
}
```

**错误码**
- `4018`: 用户已在其他对局中
- `4000`: 参数无效

---

### 2. 加入对局

加入一个已存在的waiting状态对局。

**接口**
```http
POST /api/matches/:id/join
```

**路径参数**
- `id`: 对局ID（UUID）

**请求类型定义**
```typescript
interface JoinMatchDto {
  displayName: string;  // 房间内显示名，1-64字符
}
```

**请求示例**
```http
POST /matches/550e8400-e29b-41d4-a716-446655440000/join
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "displayName": "玩家小明"
}
```

**响应类型定义**
```typescript
interface JoinMatchResponse {
  id: string;
  userId: string;
  displayName: string;
  matchId: string;
  joinedAt: string;
}
```
#### 响应示例

```json
{
  "data": {
    "id": "part_def456",
    "matchId": "550e8400-e29b-41d4-a716-446655440000",
    "userId": "user_def456",
    "displayName": "玩家小明",
    "joinedAt": "2025-10-27T10:05:00.000Z"
  },
  "code": 0,
  "success": true,
  "ts": 1761558608598
}
```
**错误码**
- `4001`: 对局不存在
- `4003`: 对局不在waiting状态
- `4006`: 房间已满
- `4008`: 已在此对局中
- `4018`: 用户已在其他对局中
#### 错误响应（HTTP 200）

⚠️ 注意：错误响应的HTTP状态码也是 200，通过 `code` 字段判断错误类型

```json
// 对局不存在
{
  "data": null,
  "code": 4001,
  "message": "对局不存在",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000"
  }
}

// 对局已开始
{
  "data": null,
  "code": 4002,
  "message": "对局已开始",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "playing"
  }
}

// 人数已满
{
  "data": null,
  "code": 4006,
  "message": "对局人数已满",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000",
    "maxPlayers": 4,
    "currentPlayers": 4
  }
}
```
---

### 3. 开始对局

开始对局，只有创建者可以操作，至少需要2名玩家。

**接口**
```http
POST /api/matches/:id/start
```

**路径参数**
- `id`: 对局ID（UUID）

#### 请求示例

```http
POST /matches/550e8400-e29b-41d4-a716-446655440000/start
Authorization: Bearer eyJhbGc...
```

**响应类型定义**
```typescript
interface StartMatchResponse {
  id: string;
  name: string;
  status: 'playing';
  maxPlayers: number;
  creatorUserId: string;
  startedAt: string;        // ISO 8601
  endedAt: string | null;   // ISO 8601
  expiresAt: string;        // ISO 8601
  createdAt: string;        // ISO 8601
  updatedAt: string;        // ISO 8601
}
```

**WebSocket 推送**

开始对局后，服务器会通过 WebSocket 向所有参与者推送：
1. `match:status` - 对局状态变化
2. `scoreboard:update` - 初始计分板

**错误码**
- `4001`: 对局不存在
- `4002`: 仅创建者可操作
- `4003`: 对局不在waiting状态
- `4005`: 人数不足（最少2人）
#### 错误响应（HTTP 200）

⚠️ 注意：错误响应的HTTP状态码也是 200，通过 `code` 字段判断错误类型

```json
// 非创建者
{
  "data": null,
  "code": 4009,
  "message": "仅创建者可操作",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000",
    "userId": "user_def456",
    "creatorUserId": "user_abc123"
  }
}

// 人数不足
{
  "data": null,
  "code": 4010,
  "message": "对局人数不足",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000",
    "minPlayers": 2,
    "currentPlayers": 1
  }
}
```
---

### 4. 批量记分

批量提交记分记录，支持幂等性。

**接口**
```http
POST /api/matches/:id/transfers
```

**路径参数**
- `id`: 对局ID（UUID）

**请求类型定义**
```typescript
interface AddTransfersDto {
  roundNo: number;     // 回合号，>=1
  items: TransferItem[];  // 记分条目，最多100条
}

interface TransferItem {
  toParticipantId: string;     // 转入参与者ID（UUID）
  points: number;              // 分数（正整数）
  clientTxnId: string;         // 客户端事务ID（幂等键）
}
```

**请求示例**
```json
{
  "roundNo": 1,
  "items": [
    {
      "toParticipantId": "part-uuid-456",
      "points": 10,
      "clientTxnId": "txn_1699000000000_1"
    },
    {
      "toParticipantId": "part-uuid-789",
      "points": 5,
      "clientTxnId": "txn_1699000000000_2"
    }
  ]
}
```

**响应类型定义**
```typescript
interface AddTransfersResponse {
  ok: boolean;  // 固定为 true
}
```

**响应示例**
```json
{
  "data": {
    "ok": true
  },
  "code": 0,
  "success": true,
  "ts": 1761558608598
}
```

**WebSocket 推送**

记分成功后，服务器会通过 WebSocket 向所有参与者推送：
- `scoreboard:update` - 更新后的计分板

**错误码**
- `4001`: 对局不存在
- `4004`: 对局不在playing状态
- `4007`: 不是参与者
- `4010`: 无效的参与者ID
- `4000`: 批量大小超过限制（>100）
#### 错误响应（HTTP 200）

⚠️ 注意：错误响应的HTTP状态码也是 200，通过 `code` 字段判断错误类型

```json
// 批量大小超过限制（400 Bad Request）
{
  "statusCode": 400,
  "message": "Batch size cannot exceed 100, got 150 items",
  "error": "Bad Request"
}

// 对局不在playing状态
{
  "data": null,
  "code": 4005,
  "message": "对局未激活",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000",
    "status": "waiting"
  }
}

// 不是参与者
{
  "data": null,
  "code": 4008,
  "message": "用户不是对局参与者",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000",
    "userId": "user_xxx"
  }
}

// 重复的clientTxnId
{
  "data": null,
  "code": 4012,
  "message": "记分重复",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "clientTxnId": "txn_123456"
  }
}

// 无效的接收者
{
  "data": null,
  "code": 4014,
  "message": "记分接收方无效",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "toParticipantId": "invalid_id"
  }
}

// 不能给自己转分
{
  "data": null,
  "code": 4013,
  "message": "不能给自己记分",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "participantId": "part_abc123"
  }
}

// 无效的分数
{
  "data": null,
  "code": 4015,
  "message": "记分点数无效",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "points": -5
  }
}
```
---

### 5. 单条记分

提交单条记分记录，便捷接口。

**接口**
```http
POST /api/matches/:id/transfer
```

**路径参数**
- `id`: 对局ID（UUID）

**请求类型定义**
```typescript
interface AddSingleTransferDto {
  roundNo: number;          // 回合号（>=1）
  toParticipantId: string;  // 转入参与者ID（UUID）
  points: number;           // 分数（正整数）
  clientTxnId: string;      // 客户端事务ID（幂等键）
}
```

**响应类型定义**
```typescript
interface AddSingleTransferResponse {
  ok: boolean;  // 固定为 true
}
```

**响应示例**
```json
{
  "data": {
    "ok": true
  },
  "code": 0,
  "success": true,
  "ts": 1761558608598
}
```

**WebSocket 推送**

同批量记分

---

### 6. 结束对局

结束对局并生成结算快照，只有创建者可以操作。

**接口**
```http
POST /api/matches/:id/end
```

**路径参数**
- `id`: 对局ID（UUID）

#### 请求示例

```http
POST /matches/550e8400-e29b-41d4-a716-446655440000/end
Authorization: Bearer eyJhbGc...
```

**响应类型定义**
```typescript
interface EndMatchResponse {
  status: 'finished';
}
```

**WebSocket 推送**

结束对局后，服务器会通过 WebSocket 向所有参与者推送：
1. `match:status` - 对局状态变化（finished）
2. `scoreboard:update` - 最终计分板

**错误码**
- `4001`: 对局不存在
- `4002`: 仅创建者可操作
- `4004`: 对局不在playing状态

---

### 7. 获取对局详情

获取对局的详细信息。

**接口**
```http
GET /api/matches/:id
```

**路径参数**
- `id`: 对局ID（UUID）

**响应类型定义**
```typescript
interface GetMatchDetailResponse {
  match: MatchInfo;
  participants: ParticipantInfo[];
  scoreboard: ScoreboardItem[];
}

interface MatchInfo {
  id: string;
  name: string;
  status: 'waiting' | 'playing' | 'finished' | 'expired';
  maxPlayers: number;
  creatorUserId: string;
  startedAt: string | null;
  endedAt: string | null;
  expiresAt: string | null;
  createdAt: string;
  updatedAt: string;
}

interface ParticipantInfo {
  id: string;
  userId: string;
  displayName: string;
  joinedAt: string;
}

interface ScoreboardItem {
  participantId: string;
  userId: string;
  displayName: string;
  score: number;
}
```

**错误码**
- `4001`: 对局不存在
- `4007`: 不是参与者

---

### 8. 获取结算快照

获取对局的结算快照（finished状态）。

**接口**
```http
GET /api/matches/:id/summary
```

**路径参数**
- `id`: 对局ID（UUID）

**响应类型定义**
```typescript
interface GetSummaryResponse {
  match: MatchInfo;
  summaries: SummaryItem[];
}

interface MatchInfo {
  id: string;
  name: string;
  status: 'finished' | 'expired';
  maxPlayers: number;
  creatorUserId: string;
  startedAt: string | null;
  endedAt: string | null;
  expiresAt: string | null;
  createdAt: string;
  updatedAt: string;
}

interface SummaryItem {
  id: string;
  matchId: string;
  participantId: string;
  userId: string;
  finalScore: number;
}
```

**错误码**
- `4001`: 对局不存在
- `4007`: 不是参与者

---

### 9. 获取计分板

获取对局当前的计分板。

**接口**
```http
GET /api/matches/:id/scoreboard
```

**路径参数**
- `id`: 对局ID（UUID）

**响应类型定义**
```typescript
interface ScoreboardItem {
  participantId: string;
  userId: string;
  displayName: string;
  score: number;
}

type GetScoreboardResponse = ScoreboardItem[];
```

**响应示例**
```json
{
  "code": 0,
  "success": true,
  "data": [
    {
      "participantId": "b5eee9c4-e69e-431b-b256-7da7b663dadf",
      "userId": "u_mh7kqu9dsvhp",
      "displayName": "Host",
      "score": -11
    },
    {
      "participantId": "c6fee9c4-e69e-431b-b256-7da7b663dadf",
      "userId": "u_xyz123abc",
      "displayName": "Player2",
      "score": 5
    },
    {
      "participantId": "d7gee9c4-e69e-431b-b256-7da7b663dadf",
      "userId": "u_abc456def",
      "displayName": "Player3",
      "score": 6
    }
  ],
  "ts": 1704110400000
}
```

**错误码**
- `4001`: 对局不存在
- `4007`: 不是参与者

---

### 10. 获取转移记录

获取对局的所有记分转移记录。

**接口**
```http
GET /api/matches/:id/transfers
```

**路径参数**
- `id`: 对局ID（UUID）

**响应类型定义**
```typescript
interface TransferRecord {
  id: string;
  matchId: string;
  roundNo: number;
  from: {
    participantId: string;
    userId: string | null;
    displayName: string;
  };
  to: {
    participantId: string;
    userId: string | null;
    displayName: string;
  };
  points: number;
  clientTxnId: string;
  createdAt: string;
}

type GetTransfersResponse = TransferRecord[];
```
#### 成功响应 (200 OK)

```json
{
  "data": [
    {
      "id": "transfer_abc123",
      "matchId": "550e8400-e29b-41d4-a716-446655440000",
      "roundNo": 1,
      "from": {
        "participantId": "part_abc123",
        "userId": "user_abc123",
        "displayName": "玩家A"
      },
      "to": {
        "participantId": "part_def456",
        "userId": "user_def456",
        "displayName": "玩家B"
      },
      "points": 10,
      "clientTxnId": "txn_123456",
      "createdAt": "2025-10-27T10:15:00.000Z"
    },
    {
      "id": "transfer_def456",
      "matchId": "550e8400-e29b-41d4-a716-446655440000",
      "roundNo": 1,
      "from": {
        "participantId": "part_abc123",
        "userId": "user_abc123",
        "displayName": "玩家A"
      },
      "to": {
        "participantId": "part_ghi789",
        "userId": "user_ghi789",
        "displayName": "玩家C"
      },
      "points": 5,
      "clientTxnId": "txn_123457",
      "createdAt": "2025-10-27T10:15:01.000Z"
    },
    {
      "id": "transfer_ghi789",
      "matchId": "550e8400-e29b-41d4-a716-446655440000",
      "roundNo": 2,
      "from": {
        "participantId": "part_def456",
        "userId": "user_def456",
        "displayName": "玩家B"
      },
      "to": {
        "participantId": "part_abc123",
        "userId": "user_abc123",
        "displayName": "玩家A"
      },
      "points": 15,
      "clientTxnId": "txn_234567",
      "createdAt": "2025-10-27T10:20:00.000Z"
    }
  ],
  "code": 0,
  "success": true,
  "ts": 1761558608598
}
```

#### 错误响应（HTTP 200）

⚠️ 注意：错误响应的HTTP状态码也是 200，通过 `code` 字段判断错误类型

```json
// 对局不存在
{
  "data": null,
  "code": 4001,
  "message": "对局不存在",
  "success": false,
  "ts": 1761558427328,
  "details": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

#### 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 转移记录ID |
| `matchId` | string | 对局ID |
| `roundNo` | number | 回合号 |
| `from.participantId` | string | 转出者参与者ID |
| `from.userId` | string | 转出者用户ID |
| `from.displayName` | string | 转出者展示名 |
| `to.participantId` | string | 接收者参与者ID |
| `to.userId` | string | 接收者用户ID |
| `to.displayName` | string | 接收者展示名 |
| `points` | number | 转移分数（正整数） |
| `clientTxnId` | string | 客户端事务ID（幂等标识） |
| `createdAt` | string | 记录创建时间（ISO 8601格式） |

#### 使用说明

- 记录按回合号（roundNo）和创建时间（createdAt）正序排列
- 可用于展示对局回放、分数流向分析
- 不会触发滑动过期时间刷新（查询操作不算活跃行为）
- 所有状态的对局都可以查询转移记录（waiting/playing/finished/expired）

---
**错误码**
- `4001`: 对局不存在
- `4007`: 不是参与者

---

### 11. 检查活跃对局

检查当前用户是否有活跃的对局（playing状态）。

**接口**
```http
GET /api/matches/me/active
```

**响应类型定义**
```typescript
interface CheckActiveResponse {
  inLiveMatch: boolean;
  match?: {
    id: string;
    name: string;
    status: 'playing' | 'waiting';
    creatorUserId: string;
    maxPlayers: number;
    startedAt: string | null;
    expiresAt: string | null;
  };
  participants?: ParticipantInfo[];
  scoreboard?: ScoreboardItem[];
}

interface ParticipantInfo {
  id: string;
  userId: string;
  displayName: string;
  joinedAt: string;
}

interface ScoreboardItem {
  participantId: string;
  userId: string;
  displayName: string;
  score: number;
}
```

---

### 12. 检查Live对局

检查当前用户是否有Live对局（waiting或playing状态）。

**接口**
```http
GET /api/matches/me/live
```

**查询参数**
- `includePending`: boolean（可选，默认true，是否包含waiting状态）

**响应类型定义**
```typescript
interface CheckLiveResponse {
  inLiveMatch: boolean;
  match?: {
    id: string;
    name: string;
    status: 'playing' | 'waiting';
    creatorUserId: string;
    maxPlayers: number;
    startedAt: string | null;
    expiresAt: string | null;
  };
  participants?: ParticipantInfo[];
  scoreboard?: ScoreboardItem[];
}

interface ParticipantInfo {
  id: string;
  userId: string;
  displayName: string;
  joinedAt: string;
}

interface ScoreboardItem {
  participantId: string;
  userId: string;
  displayName: string;
  score: number;
}
```

---

### 13. 获取结算记录

获取当前用户的所有结算记录。

**接口**
```http
GET /api/matches/me/records
```

**响应类型定义**
```typescript
interface SettledRecordItem {
  matchId: string;
  name: string;
  status: 'finished' | 'expired';
  startedAt: string | null;
  endedAt: string | null;
  myFinalScore: number;
}

type GetSettledRecordsResponse = SettledRecordItem[];
```

---

### 14. 获取对局历史

获取当前用户参与的所有对局。

**接口**
```http
GET /api/matches/me/history
```

**响应类型定义**
```typescript
interface HistoryMatchItem {
  id: string;
  name: string;
  status: 'waiting' | 'playing' | 'finished' | 'expired';
  maxPlayers: number;
  creatorUserId: string;
  startedAt: string | null;
  endedAt: string | null;
  expiresAt: string | null;
  createdAt: string;
  updatedAt: string;
}

type GetHistoryResponse = HistoryMatchItem[];
```

---

## WebSocket API

Match模块提供原生WebSocket实时通信能力，用于推送对局状态变化和计分板更新。

> **⚠️ 协议**: 原生 WebSocket (RFC 6455)，完全兼容微信小程序、浏览器等所有标准客户端。

### 连接信息

| 项目 | 值 |
|------|-----|
| **协议** | 原生 WebSocket (RFC 6455) |
| **开发环境** | `ws://localhost:2233/ws/matches` |
| **生产环境** | `wss://api.jmni.cn/ws/matches` |
| **认证方式** | URL 参数：`?token=your_jwt_token` |
| **消息格式** | JSON：`{"type": "xxx", "data": {...}}` |
| **心跳间隔** | 30 秒 |
| **心跳超时** | 90 秒未响应自动断开 |

### 连接示例

```javascript
// 浏览器
const ws = new WebSocket(`ws://localhost:2233/ws/matches?token=${token}`);

// 微信小程序
wx.connectSocket({
  url: `wss://api.jmni.cn/ws/matches?token=${token}`
});
```

---

## 消息类型定义

### 基础消息接口

```typescript
interface WebSocketMessage {
  type: string;    // 消息类型
  data: any;       // 消息数据
}
```

---

## 客户端发送消息

### 消息类型总览

| type | 说明 | data类型 | 必需字段 |
|------|------|---------|---------|
| `match:join` | 加入对局房间 | `JoinMatchData` | `matchId` |
| `match:leave` | 离开对局房间 | `LeaveMatchData` | `matchId` |
| `pong` | 心跳响应 | `PongData` | - |

### 详细类型定义

#### 1. match:join - 加入对局房间

**TypeScript 类型**
```typescript
interface JoinMatchMessage {
  type: 'match:join';
  data: JoinMatchData;
}

interface JoinMatchData {
  matchId: string;  // 对局ID（UUID）
}
```

**JSON 示例**
```json
{
  "type": "match:join",
  "data": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**说明**
- 用户必须是该对局的参与者
- 成功后会收到 `joined` 响应消息
- 其他玩家会收到 `player:joined` 广播

---

#### 2. match:leave - 离开对局房间

**TypeScript 类型**
```typescript
interface LeaveMatchMessage {
  type: 'match:leave';
  data: LeaveMatchData;
}

interface LeaveMatchData {
  matchId: string;  // 对局ID（UUID）
}
```

**JSON 示例**
```json
{
  "type": "match:leave",
  "data": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**说明**
- 离开当前房间
- 成功后会收到 `left` 响应消息
- 其他玩家会收到 `player:left` 广播

---

#### 3. pong - 心跳响应

**TypeScript 类型**
```typescript
interface PongMessage {
  type: 'pong';
  data: PongData;
}

interface PongData {
  // 空对象
}
```

**JSON 示例**
```json
{
  "type": "pong",
  "data": {}
}
```

**说明**
- 响应服务器的 `ping` 消息
- 必须在90秒内响应，否则连接会被断开
- 建议每30秒检查一次

---

## 服务端推送消息

### 消息类型总览

| type | 说明 | data类型 | 触发时机 | 接收者 |
|------|------|---------|---------|--------|
| `connected` | 连接确认 | `ConnectedData` | WebSocket连接成功后 | 连接者本人 |
| `joined` | 加入成功 | `JoinedData` | 成功加入房间后 | 加入者本人 |
| `left` | 离开成功 | `LeftData` | 成功离开房间后 | 离开者本人 |
| `player:joined` | 玩家加入通知 | `PlayerJoinedData` | 有玩家加入房间 | 房间其他玩家 |
| `player:left` | 玩家离开通知 | `PlayerLeftData` | 有玩家离开房间 | 房间其他玩家 |
| `scoreboard:update` | 计分板更新 | `ScoreboardUpdateData` | 记分、开始、结束对局 | 房间所有玩家 |
| `match:status` | 对局状态更新 | `MatchStatusData` | 对局状态变化 | 房间所有玩家 |
| `ping` | 服务器心跳 | `PingData` | 每30秒自动发送 | 所有连接 |
| `error` | 错误消息 | `ErrorData` | 操作失败时 | 操作发起者 |

### 详细类型定义

#### 1. connected - 连接确认

**TypeScript 类型**
```typescript
interface ConnectedMessage {
  type: 'connected';
  data: ConnectedData;
}

interface ConnectedData {
  connectionId: string;  // 连接ID，格式：ws_<timestamp>_<random>
}
```

**JSON 示例**
```json
{
  "type": "connected",
  "data": {
    "connectionId": "ws_1699000000000_abc123"
  }
}
```

**说明**
- WebSocket 连接成功后立即发送
- 连接ID可用于调试和日志追踪

---

#### 2. joined - 加入成功

**TypeScript 类型**
```typescript
interface JoinedMessage {
  type: 'joined';
  data: JoinedData;
}

interface JoinedData {
  matchId: string;  // 对局ID（UUID）
}
```

**JSON 示例**
```json
{
  "type": "joined",
  "data": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**说明**
- 成功加入对局房间后发送给加入者
- 收到此消息后可以开始监听对局更新

---

#### 3. left - 离开成功

**TypeScript 类型**
```typescript
interface LeftMessage {
  type: 'left';
  data: LeftData;
}

interface LeftData {
  matchId: string;  // 对局ID（UUID）
}
```

**JSON 示例**
```json
{
  "type": "left",
  "data": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**说明**
- 成功离开对局房间后发送给离开者

---

#### 4. player:joined - 玩家加入通知（广播）

**TypeScript 类型**
```typescript
interface PlayerJoinedMessage {
  type: 'player:joined';
  data: PlayerJoinedData;
}

interface PlayerJoinedData {
  matchId: string;      // 对局ID（UUID）
  player: PlayerInfo;   // 玩家信息
  timestamp: number;    // 时间戳（毫秒）
}

interface PlayerInfo {
  userId: string;       // 用户ID（UUID）
  displayName: string;  // 显示名称
}
```

**JSON 示例**
```json
{
  "type": "player:joined",
  "data": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000",
    "player": {
      "userId": "user-uuid-456",
      "displayName": "玩家小明"
    },
    "timestamp": 1699000000000
  }
}
```

**说明**
- 当有新玩家加入房间时广播给**房间内其他玩家**
- 加入的玩家本人不会收到此消息
- 用于更新在线玩家列表

---

#### 5. player:left - 玩家离开通知（广播）

**TypeScript 类型**
```typescript
interface PlayerLeftMessage {
  type: 'player:left';
  data: PlayerLeftData;
}

interface PlayerLeftData {
  matchId: string;      // 对局ID（UUID）
  player: PlayerInfo;   // 玩家信息
  timestamp: number;    // 时间戳（毫秒）
}
```

**JSON 示例**
```json
{
  "type": "player:left",
  "data": {
    "matchId": "550e8400-e29b-41d4-a716-446655440000",
    "player": {
      "userId": "user-uuid-456",
      "displayName": "玩家小明"
    },
    "timestamp": 1699000000000
  }
}
```

**说明**
- 当有玩家离开房间时广播给**房间内其他玩家**
- 离开的玩家本人不会收到此消息
- 用于更新在线玩家列表

---

#### 6. scoreboard:update - 计分板更新

**TypeScript 类型**
```typescript
interface ScoreboardUpdateMessage {
  type: 'scoreboard:update';
  data: ScoreboardUpdateData;
}

interface ScoreboardUpdateData {
  [participantId: string]: number;  // participantId -> score
}
```

**JSON 示例**
```json
{
  "type": "scoreboard:update",
  "data": {
    "part-abc123": 150,
    "part-def456": 200,
    "part-ghi789": 100
  }
}
```

**说明**
- 触发时机：
  - 有玩家记分后
  - 对局开始时（初始计分板）
  - 对局结束时（最终计分板）
- 计分板是完整的，不是增量更新

---

#### 7. match:status - 对局状态更新

**TypeScript 类型**
```typescript
interface MatchStatusMessage {
  type: 'match:status';
  data: MatchStatusData;
}

interface MatchStatusData {
  status: MatchStatus;           // 对局状态
  startedAt?: string;            // 开始时间（ISO 8601）
  endedAt?: string;              // 结束时间（ISO 8601）
  expiresAt?: string;            // 过期时间（ISO 8601）
  reason?: 'normal' | 'expire';  // 结束原因
  policy?: ExpiryPolicy;         // 过期策略
}

type MatchStatus = 'waiting' | 'playing' | 'finished' | 'expired';

interface ExpiryPolicy {
  idleMinutes: number;           // 滑动过期时间（分钟）
  absoluteMaxMinutes: number;    // 绝对最大时间（分钟）
}
```

**JSON 示例（开始对局）**
```json
{
  "type": "match:status",
  "data": {
    "status": "playing",
    "startedAt": "2024-01-01T12:00:00.000Z",
    "expiresAt": "2024-01-01T14:00:00.000Z",
    "policy": {
      "idleMinutes": 30,
      "absoluteMaxMinutes": 480
    }
  }
}
```

**JSON 示例（结束对局）**
```json
{
  "type": "match:status",
  "data": {
    "status": "finished",
    "startedAt": "2024-01-01T12:00:00.000Z",
    "endedAt": "2024-01-01T13:30:00.000Z",
    "reason": "normal"
  }
}
```

**JSON 示例（过期时间更新）**
```json
{
  "type": "match:status",
  "data": {
    "status": "playing",
    "expiresAt": "2024-01-01T14:30:00.000Z",
    "policy": {
      "idleMinutes": 30,
      "absoluteMaxMinutes": 480
    }
  }
}
```

**说明**
- 触发时机：
  - 对局开始时（`waiting` → `playing`）
  - 对局结束时（`playing` → `finished`）
  - 对局过期时（任何状态 → `expired`）
  - 过期时间更新时（仅推送 `expiresAt` 和 `policy`）
- ⚠️ **重要**：为避免重复通知，对局开始时只会收到**一次**完整的状态消息

---

#### 8. ping - 服务器心跳

**TypeScript 类型**
```typescript
interface PingMessage {
  type: 'ping';
  data: PingData;
}

interface PingData {
  timestamp: number;  // 时间戳（毫秒）
}
```

**JSON 示例**
```json
{
  "type": "ping",
  "data": {
    "timestamp": 1699000000000
  }
}
```

**说明**
- 服务器每30秒自动发送
- 客户端必须响应 `pong` 消息
- 90秒未响应自动断开连接

---

#### 9. error - 错误消息

**TypeScript 类型**
```typescript
interface ErrorMessage {
  type: 'error';
  data: ErrorData;
}

interface ErrorData {
  message: string;  // 错误描述
  code: string;     // 错误码
}
```

**JSON 示例**
```json
{
  "type": "error",
  "data": {
    "message": "Not a participant of this match",
    "code": "NOT_PARTICIPANT"
  }
}
```

**错误码列表**

| 错误码 | 说明 | 建议处理 |
|--------|------|---------|
| `UNAUTHORIZED` | 未授权（token无效） | 重新登录 |
| `USER_INACTIVE` | 用户未激活 | 联系管理员 |
| `NOT_PARTICIPANT` | 非对局参与者 | 提示用户 |
| `INVALID_REQUEST` | 请求参数无效 | 检查参数 |
| `INTERNAL_ERROR` | 服务器内部错误 | 稍后重试 |

---

## WebSocket 完整示例

### 浏览器客户端

```typescript
class MatchWebSocketClient {
  private ws: WebSocket | null = null;
  private token: string;
  private matchId: string;
  private heartbeatInterval: number | null = null;

  constructor(token: string) {
    this.token = token;
  }

  // 连接
  connect() {
    const url = `ws://localhost:2233/ws/matches?token=${this.token}`;
    this.ws = new WebSocket(url);

    this.ws.onopen = () => {
      console.log('WebSocket 连接成功');
      this.startHeartbeat();
    };

    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data) as WebSocketMessage;
      this.handleMessage(message);
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket 错误:', error);
    };

    this.ws.onclose = () => {
      console.log('WebSocket 连接关闭');
      this.stopHeartbeat();
    };
  }

  // 处理消息
  private handleMessage(message: WebSocketMessage) {
    switch (message.type) {
      case 'connected':
        const connData = message.data as ConnectedData;
        console.log('连接成功:', connData.connectionId);
        break;

      case 'joined':
        const joinData = message.data as JoinedData;
        console.log('已加入房间:', joinData.matchId);
        break;

      case 'player:joined':
        const pjData = message.data as PlayerJoinedData;
        console.log(`${pjData.player.displayName} 加入了对局`);
        this.onPlayerJoined(pjData.player);
        break;

      case 'player:left':
        const plData = message.data as PlayerLeftData;
        console.log(`${plData.player.displayName} 离开了对局`);
        this.onPlayerLeft(plData.player);
        break;

      case 'scoreboard:update':
        const sbData = message.data as ScoreboardUpdateData;
        this.onScoreboardUpdate(sbData);
        break;

      case 'match:status':
        const msData = message.data as MatchStatusData;
        this.onMatchStatusChange(msData);
        break;

      case 'ping':
        this.send({ type: 'pong', data: {} });
        break;

      case 'error':
        const errData = message.data as ErrorData;
        console.error('错误:', errData);
        break;
    }
  }

  // 加入对局
  joinMatch(matchId: string) {
    this.matchId = matchId;
    this.send({
      type: 'match:join',
      data: { matchId }
    });
  }

  // 离开对局
  leaveMatch() {
    if (this.matchId) {
      this.send({
        type: 'match:leave',
        data: { matchId: this.matchId }
      });
    }
  }

  // 发送消息
  private send(message: WebSocketMessage) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(message));
    }
  }

  // 启动心跳
  private startHeartbeat() {
    this.heartbeatInterval = window.setInterval(() => {
      // 检查是否收到ping，如果没收到说明可能断开了
    }, 30000);
  }

  // 停止心跳
  private stopHeartbeat() {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
      this.heartbeatInterval = null;
    }
  }

  // 回调函数（需要实现）
  private onPlayerJoined(player: PlayerInfo) {
    // 更新UI：添加玩家到列表
  }

  private onPlayerLeft(player: PlayerInfo) {
    // 更新UI：从列表移除玩家
  }

  private onScoreboardUpdate(scoreboard: ScoreboardUpdateData) {
    // 更新UI：刷新计分板
  }

  private onMatchStatusChange(status: MatchStatusData) {
    // 更新UI：更新对局状态
  }
}

// 使用示例
const client = new MatchWebSocketClient('your_jwt_token');
client.connect();
client.joinMatch('match-uuid-123');
```

### 微信小程序客户端

```typescript
// pages/match/match.ts
Page({
  data: {
    matchId: '',
    players: [] as PlayerInfo[],
    scoreboard: {} as ScoreboardUpdateData,
    status: 'waiting' as MatchStatus
  },

  onLoad(options: { matchId: string }) {
    this.data.matchId = options.matchId;
    this.connectWebSocket();
  },

  connectWebSocket() {
    const token = wx.getStorageSync('token');
    const url = `wss://api.jmni.cn/ws/matches?token=${token}`;

    wx.connectSocket({ url });

    wx.onSocketOpen(() => {
      console.log('WebSocket 连接成功');
      this.joinMatch();
    });

    wx.onSocketMessage((res) => {
      const message = JSON.parse(res.data) as WebSocketMessage;
      this.handleMessage(message);
    });

    wx.onSocketError((error) => {
      console.error('WebSocket 错误:', error);
    });

    wx.onSocketClose(() => {
      console.log('WebSocket 连接关闭');
    });
  },

  handleMessage(message: WebSocketMessage) {
    switch (message.type) {
      case 'connected':
        console.log('连接成功');
        break;

      case 'joined':
        console.log('已加入房间');
        break;

      case 'player:joined':
        const pjData = message.data as PlayerJoinedData;
        wx.showToast({
          title: `${pjData.player.displayName} 加入了对局`,
          icon: 'none'
        });
        break;

      case 'player:left':
        const plData = message.data as PlayerLeftData;
        wx.showToast({
          title: `${plData.player.displayName} 离开了对局`,
          icon: 'none'
        });
        break;

      case 'scoreboard:update':
        const sbData = message.data as ScoreboardUpdateData;
        this.setData({ scoreboard: sbData });
        break;

      case 'match:status':
        const msData = message.data as MatchStatusData;
        this.setData({ status: msData.status });
        break;

      case 'ping':
        this.sendMessage({ type: 'pong', data: {} });
        break;

      case 'error':
        const errData = message.data as ErrorData;
        wx.showToast({
          title: errData.message,
          icon: 'none'
        });
        break;
    }
  },

  joinMatch() {
    this.sendMessage({
      type: 'match:join',
      data: { matchId: this.data.matchId }
    });
  },

  sendMessage(message: WebSocketMessage) {
    wx.sendSocketMessage({
      data: JSON.stringify(message)
    });
  }
});
```

---

## 数据模型

### Match（对局）

```typescript
interface Match {
  id: string;                    // UUID
  name: string;                  // 对局名称
  status: MatchStatus;           // 状态
  maxPlayers: number;            // 最大人数
  creatorUserId: string;         // 创建者ID（UUID）
  startedAt: Date | null;        // 开始时间
  endedAt: Date | null;          // 结束时间
  expiresAt: Date | null;        // 过期时间
  createdAt: Date;               // 创建时间
  updatedAt: Date;               // 更新时间
}

type MatchStatus = 'waiting' | 'playing' | 'finished' | 'expired';
```

### MatchParticipant（参与者）

```typescript
interface MatchParticipant {
  id: string;          // UUID
  matchId: string;     // 对局ID（UUID）
  userId: string;      // 用户ID（UUID）
  displayName: string; // 显示名称
  joinedAt: Date;      // 加入时间
}
```

### MatchTransfer（记分转移）

```typescript
interface MatchTransfer {
  id: string;                // UUID
  matchId: string;           // 对局ID（UUID）
  roundNo: number;           // 回合号
  fromParticipantId: string; // 转出参与者ID（UUID）
  toParticipantId: string;   // 转入参与者ID（UUID）
  points: number;            // 分数
  clientTxnId: string;       // 客户端事务ID
  createdAt: Date;           // 创建时间
}
```

### MatchSummary（结算快照）

```typescript
interface MatchSummary {
  id: string;           // UUID
  matchId: string;      // 对局ID（UUID）
  participantId: string;// 参与者ID（UUID）
  finalScore: number;   // 最终分数
  rank: number;         // 排名
  metadata: object | null; // 元数据
  createdAt: Date;      // 创建时间
}
```

---

## 安全与性能保障

### 并发控制

1. **joinMatch双锁机制**
   - 用户锁：防止同一用户重复加入
   - Match锁：防止对局超员
   - 自动续期：防止长时间操作导致锁过期

2. **分布式锁**
   - 使用Redis实现
   - 支持自动续期
   - 优雅释放，避免死锁

### 数据一致性

1. **Redis缓存与DB同步**
   - 事务提交后才更新Redis
   - 缓存失败不影响业务（容错降级）
   - 只更新实际插入的记录

2. **原子性操作**
   - Match创建使用数据库事务
   - 记分操作使用事务保护
   - 调度任务失败不阻塞创建

### 容错降级

1. **Redis故障处理**
   - 缓存失败记录日志但不抛出异常
   - 降级到数据库直接查询
   - 队列发送失败不影响核心业务

2. **BullMQ调度容错**
   - 任务调度失败记录日志
   - 不阻塞对局创建
   - 支持手动触发补偿

### 性能优化

1. **批量操作**
   - Redis Pipeline批量操作
   - 批量查询避免N+1问题
   - 批量记分支持（最多100条）

2. **缓存策略**
   - 计分板Redis缓存（HSET/HINCRBY）
   - 参与者信息缓存
   - 对局状态缓存

3. **实时性**
   - WebSocket推送（< 100ms延迟）
   - 计分板增量更新
   - 状态变化实时通知

### 资源限制

1. **时间限制**
   - 滑动过期：默认120分钟无活动
   - 绝对上限：默认720分钟（12小时）
   - 最大创建时长：30天（43200分钟）

2. **并发限制**
   - 同一用户同时只能在一个live对局中
   - 人数限制：2-12人
   - 批量记分上限：100条/次

3. **幂等保证**
   - 使用唯一的clientTxnId
   - 重复提交返回幂等响应
   - 防止重复扣分

---

## 最佳实践

### 1. 幂等性保证

使用唯一的`clientTxnId`确保记分幂等：

```typescript
const clientTxnId = `${userId}_${Date.now()}_${roundNo}_${randomId}`;

await fetch(`/api/matches/${matchId}/transfers`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    roundNo: 1,
    items: [
      {
        toParticipantId,
        points: 10,
        clientTxnId  // 幂等键
      }
    ]
  })
});
```

### 2. 错误处理

```typescript
try {
  const response = await fetch('/api/matches', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(createDto)
  });

  const result = await response.json();

  if (result.code !== 0) {
    // 根据errorCode处理
    switch (result.errorCode) {
      case 4018: // MATCH_USER_IN_OTHER_MATCH
        alert('您已在其他对局中，请先完成或退出');
        break;
      case 5006: // SECURITY_RATE_LIMIT
        alert('操作过于频繁，请稍后再试');
        break;
      case 4001: // MATCH_NOT_FOUND
        alert('对局不存在');
        break;
      case 4006: // MATCH_PLAYER_LIMIT_EXCEEDED
        alert('房间已满，无法加入');
        break;
      default:
        alert(result.message);
    }
    return;
  }

  console.log('成功', result.data);
} catch (error) {
  console.error('网络错误', error);
}
```

### 3. WebSocket 重连机制

```typescript
class MatchWebSocketClient {
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000; // 1秒

  connect() {
    // ... 连接逻辑

    this.ws.onclose = () => {
      console.log('WebSocket 连接关闭');
      this.handleReconnect();
    };
  }

  private handleReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);
      
      console.log(`${delay}ms 后尝试重连 (${this.reconnectAttempts}/${this.maxReconnectAttempts})`);
      
      setTimeout(() => {
        this.connect();
      }, delay);
    } else {
      console.error('重连失败，已达最大重试次数');
    }
  }
}
```

### 4. 心跳检测

```typescript
class MatchWebSocketClient {
  private lastPongTime = Date.now();
  private heartbeatCheckInterval: number | null = null;

  private startHeartbeatCheck() {
    this.heartbeatCheckInterval = window.setInterval(() => {
      const now = Date.now();
      if (now - this.lastPongTime > 90000) { // 90秒未收到pong
        console.error('心跳超时，断开连接');
        this.ws?.close();
      }
    }, 30000); // 每30秒检查一次
  }

  private handleMessage(message: WebSocketMessage) {
    switch (message.type) {
      case 'ping':
        this.lastPongTime = Date.now();
        this.send({ type: 'pong', data: {} });
        break;
    }
  }
}
```

#### ✅ 并发控制增强

1. **joinMatch双锁机制**
   - 添加Match级分布式锁，防止超员
   - 修复多用户同时加入导致超出maxPlayers限制的bug
   - 锁层级：用户锁（防重复） + Match锁（防超员）

2. **分布式锁自动续期**
   - 防止长时间操作导致的锁过期
   - 优雅释放，避免死锁

#### ✅ 数据一致性修复

3. **Redis缓存与DB事务同步**
   - `addTransfers`: 确保事务提交后才更新Redis
   - 只更新实际插入的记录（幂等过滤后）
   - 缓存失败不影响业务（容错降级）

4. **原子性Match创建**
   - Match和Participant在同一事务中创建
   - 防止"孤儿"Match（有Match无Participant）
   - 调度任务失败不阻塞创建流程

5. **Match状态一致性**
   - startMatch重新查询最新状态后广播
   - 防止refreshExpiryOnActivity改变状态后不一致
   - 确保客户端收到正确的状态

---

## 相关文档

- [WebSocket 产品指南](./MATCH_WEBSOCKET_PRODUCT_GUIDE.md) - 完整的产品功能说明
- [WebSocket 快速开始](./QUICK_START_WEBSOCKET.md) - 5分钟快速上手指南
- [WebSocket 客户端示例](./websocket-client-examples.md) - 完整的客户端实现
- [代码分析报告](./MATCH_CODE_ANALYSIS_REPORT.md) - 代码质量分析

---

## 技术支持

如有问题，请联系技术团队或参考相关文档。

**文档版本**: v2.0.2  
**API 版本**: v1.4.0  
**最后更新**: 2025-11-04  

> ⚠️ **重要**: v2.0.2 核对并修正了所有 API 的请求和响应类型定义，确保与实际代码实现完全一致。


# Avalon Online — 後端規格

> 純前端 + Firebase Realtime Database 架構  
> 「後端」其實是 Firebase RTDB + 客戶端邏輯 + Security Rules

---

## 1. 資料結構（RTDB 樹）

根節點：`/rooms/{roomCode}/`

```
rooms/
  ABC123/                           ← 6 字房間代碼（大寫字母+數字）
    meta/                            ← 房間基本資料
      hostUid: "uid-xxx"
      createdAt: 1690464000000
      phase: "lobby" | "roleConfig" | "stage1" | "stage2" | "stage3"
            | "stage4" | "stage5" | "stage6" | "over"
      locked: false                  ← 開始後鎖門
      maxPlayers: 10
    players/
      uid-1/
        nickname: "小明"
        isHost: true
        online: true
        joinedAt: 1690464000000
      uid-2/
        nickname: "小華"
        isHost: false
        online: true
        joinedAt: 1690464001000
    gameConfig/                      ← 房主在 roleConfig 階段填
      playerCount: 6
      roles:
        merlin: true                 ← 固定 true
        assassin: true               ← 固定 true
        percival: true               ← 勾了自動帶 morgana
        morgana: true                ← 勾了自動帶 percival
        mordred: false
        oberon: false
        ladyOfLake: true
    gameState/                       ← 進 Stage 1 才生成
      mission: 1                     ← 1~5
      rejections: 0                  ← 0~5
      leaderUid: "uid-1"
      leaderHistory: ["uid-1", "uid-2", ...]
      currentTeam/
        members: ["uid-1", "uid-3"]
        proposedAt: 1690464200000
      currentVotes/                  ← 階段 3
        "uid-1": "approve"           ← "approve" | "reject" | null
        "uid-2": "reject"
      missionResults/                ← 階段 4
        "1": { success: 3, fail: 0, voters: ["uid-1", ...] }
        "2": { success: 2, fail: 1, voters: [...] }
      ladyHolderUid: "uid-1"         ← 女神持有者
      ladyHistory: ["uid-1", "uid-3"]
      roleAssignments/               ← 角色分配（進 Stage 1）
        "uid-1": "merlin"
        "uid-2": "percival"
        ...
      initialInfo/                   ← 天黑情報
        "uid-1":                     ← merlin 看到的
          sees: ["uid-3", "uid-4"]
          seeType: "evil"
        "uid-2":                     ← percival 看到的
          sees: ["uid-1", "uid-3"]
          seeType: "merlin_candidates"
        "uid-3":                     ← morgana
          sees: ["uid-4", "uid-5"]
          seeType: "evil"
      assassinationTarget: null      ← 刺客選擇
```

---

## 2. 陣營配置（依人數）

| 人數 | 好人 | 壞人 |
|---|---|---|
| 5 | 3 | 2 |
| 6 | 4 | 2 |
| 7 | 4 | 3 |
| 8 | 5 | 3 |
| 9 | 6 | 3 |
| 10 | 6 | 4 |

---

## 3. 角色規則

### 固定
- **梅林 (Merlin)** — 永遠啟用
- **刺客 (Assassin)** — 永遠啟用

### 鎖定對
- **派西維爾 (Percival)** + **莫甘娜 (Morgana)** — 必須同進同出

### 獨立可選
- **莫德雷德 (Mordred)** — 壞人，梅林看不到
- **奧伯倫 (Oberon)** — 壞人，跟其他壞人互不認識
- **湖中女神 (Lady of the Lake)** — 工具牌，給一位好人持有

### 角色 → 陣營
- 好人：梅林、派西維爾
- 壞人：刺客、莫甘娜、莫德雷德、奧伯倫

### 湖中女神
- 第一任持有者 = 第一任隊長（永遠是好人）
- 驗完後標記轉移給被驗玩家
- 持有者不可驗自己、不可驗歷史持有者
- 觸發時機：第 2、3、4 次任務結束後

---

## 4. 情報產生規則（initialInfo）

### 梅林看到
- **除了莫德雷德以外的所有壞人**（含奧伯倫、刺客、莫甘娜）

### 派西維爾看到
- **梅林 + 莫甘娜**（兩人不分清楚）

### 壞人之間
- 莫甘娜、刺客、莫德雷德 **互相認識**
- **奧伯倫例外**：奧伯倫看不到其他壞人、其他壞人也看不到奧伯倫

### 好人（梅林、派西維爾、湖中女神持有人）
- 互不認識（除了派西維爾的特例）

---

## 5. 房間狀態機（State Machine）

```
[lobby] ─房主按開始→ [roleConfig] ─房主確認→ [stage1] ─全員記住→ [stage2]
                                                                  │
                                                          隊長提交名單
                                                                  ↓
                                                               [stage3]
                                                                  │
                                                       全員投票、反≥贊
                                                                  ↓
                                                          否決 → 換隊長 → [stage2]
                                                          否決 5 次 → [over 紅勝]
                                                          贊 > 反 ↓
                                                               [stage4]
                                                                  │
                                                          紅 3 失敗 → [over 紅勝]
                                                          藍 3 成功 → [stage6 刺殺]
                                                          第 2/3/4 任務後 + 女神開 → [stage5]
                                                          否則 → 換隊長 → [stage2]
                                                               [stage5]
                                                                  │
                                                          女神驗完、轉移 → 換隊長 → [stage2]
                                                               [stage6]
                                                                  │
                                                          刺客刺中梅林 → 紅勝
                                                          刺客刺錯 → 藍勝
```

### 勝利判定（在 stage4 mission 結算後）
- 紅營累積 3 失敗 → `over`（紅勝）
- 藍營累積 3 成功 → `stage6`（刺殺）
- 否則依上面規則繼續

### 第 4 關雙失敗票規則
- 6+ 人局
- 第 4 關
- 前一關已失敗
- **需要 2 張失敗票才算失敗**（5 人局除外）

### 否決累積
- 連續否決 5 次 → 紅勝

---

## 6. Firebase Security Rules

完整內容見 `firebase-rules.json`。重點：

- `rooms/{code}` 可讀（玩家要看到房間狀態）
- `players/{uid}` 只能自己或房主寫
- `gameConfig` 只能房主寫
- `gameState` 客戶端可寫自己的子節點（如自己的 vote）
- 用 Firebase Anonymous Auth 取得 uid

---

## 7. 客戶端事件（client-side 呼叫 Firebase）

| 動作 | Firebase 呼叫 |
|---|---|
| 創建房間 | `ref.push({meta, players: {hostUid}})` |
| 加入房間 | `ref.child('players').child(uid).set({...})` |
| 房主開始遊戲 | `meta.phase = 'roleConfig'` + `meta.locked = true` |
| 房主設定角色 | `gameConfig.roles = {...}` |
| 房主確認角色 | 觸發客戶端：分配角色 → 寫 roleAssignments + initialInfo → `meta.phase = 'stage1'` |
| 隊長選人 | `currentTeam.members = [...]` |
| 提交名單 | `meta.phase = 'stage3'` |
| 投票 | `currentVotes[uid] = 'approve'\|'reject'` |
| 投票結算 | 客戶端判斷 → 寫 `missionResults[mission]` → 決定下一階段 |
| 任務執行 | 同上（票面洗牌在 client） |
| 女神驗人 | `initialInfo[ladyHolderUid] = {sees: [target], seeType: 'alignment'}` + 更新 `ladyHolderUid` |
| 刺客刺殺 | `assassinationTarget = targetUid` → 結算 |

---

## 8. 時序

| 時間 | 動作 |
|---|---|
| 階段 0 | 玩家加入，5+ 才能開始 |
| 階段 0.5 | 房主設定角色（透明）|
| 階段 1 | 全員「我已記住」後進 2 |
| 階段 2 | 隊長選人、提交 |
| 階段 3 | 全員投票 |
| 階段 4 | 出征隊員投票（成功/失敗）|
| 階段 5 | 女神驗人（觸發時）|
| 階段 6 | 刺客刺殺 / 紅勝結算 |

---

## 9. 限制 & 假設

- **朋友限定**：房東密碼 + 房間代碼是門禁，不做完整帳號系統
- **一次性旅行**：不做觀戰、聊天、自訂頭像
- **斷線重連**：Firebase SDK 自動處理，玩家重連後 client 重新 sync state
- **匿名登入**：用 Firebase Anonymous Auth 拿 uid
- **房東轉移**：v1 不做（房主斷線 = 房間暫停）

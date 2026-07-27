# Avalon Online

朋友旅行用的阿瓦隆（Avalon）連線版，純前端 + Firebase Realtime Database。

## 使用方式

1. 房東開網頁 → 「創建房間」→ 輸入暱稱 + 房東密碼
2. 房東把網址 + 房間代碼 + 房東密碼傳給朋友
3. 朋友開網頁 → 「加入房間」→ 輸入代碼 + 暱稱 + 密碼
4. 5+ 人在房 → 房主按「開始遊戲」→ 設定角色 → 開玩

## 技術

- 純靜態前端（單一 `index.html`）
- Firebase Realtime Database（資料同步）
- Firebase Anonymous Auth（取 uid）
- GitHub Pages（hosting）
- Tailwind CSS（CDN）

## 部署

1. 推上 GitHub（`ameow-cpu/avalon`）
2. Repo → Settings → Pages → Deploy from branch → `main` / `root`
3. 網址：`https://ameow-cpu.github.io/avalon/`
4. Firebase Console → Realtime Database → Rules → 貼上 `firebase-rules.json`

## 角色

- **梅林 (Merlin)** — 永遠啟用，看到所有壞人（除了莫德雷德）
- **刺客 (Assassin)** — 永遠啟用，遊戲結束可刺梅林
- **派西維爾 (Percival)** — 看到梅林 + 莫甘娜（不分）
- **莫甘娜 (Morgana)** — 對派西維爾假扮梅林
- **莫德雷德 (Mordred)** — 梅林看不到
- **奧伯倫 (Oberon)** — 跟其他壞人互不認識
- **湖中女神 (Lady of the Lake)** — 第 2/3/4 任務後可驗人

## 規格文件

- [`BACKEND_SPEC.md`](./BACKEND_SPEC.md) — Firebase 資料結構 + 狀態機
- [`FRONTEND_SPEC.md`](./FRONTEND_SPEC.md) — UI 流程

## License

MIT

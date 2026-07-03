---
name: clear-leaderboard
description: >-
  清空 EHS 遊戲排行榜的 Cloudflare Durable Object 儲存資料(透過 CLI / wrangler)。
  只要使用者想「清空/清除/重置排行榜」、「活動後清成績」、「reset leaderboard」、
  「清掉 DO 裡的成績」、「leaderboard 歸零」、「把 ehs-leaderboard 的資料清掉」,
  就用這個 skill。它提供一條經過驗證的固定流程:臨時部署一個清除端點 → 呼叫它 →
  立刻還原並重新部署移除端點 → 驗證。即使使用者沒講「Durable Object」或「wrangler」,
  只要意圖是清空這個遊戲的排行榜雲端資料,都該觸發。
---

# 清空排行榜(Durable Object KV)

EHS 排行榜的成績存在一個 **Cloudflare Worker + Durable Object(DO)** 裡:
- Worker 名稱:`ehs-leaderboard`(見 `worker/wrangler.toml`)
- 線上網址:`https://ehs-leaderboard.charlychiu.workers.dev`(見 `config.js` 的 `baseUrl`)
- 所有成績存在「同一個全域 DO 實例」的 storage key `'list'`(一個陣列),程式在 `worker/worker.js`

「清空排行榜」= 清掉這個 DO 的 KV 儲存。

## 為什麼要這樣繞一圈

`wrangler`(目前 4.x)**沒有任何直接讀寫 / 清空 Durable Object 儲存的指令**(沒有 `wrangler durable-objects` 子命令)。所以用 CLI 清資料的唯一辦法,是**臨時部署一段會呼叫 `storage.deleteAll()` 的程式、呼叫它、再把程式還原移除**。這就是下面的流程。

> 不想動 CLI / 不想部署時的替代法見最後「替代方案」。

## 前置檢查

1. 確認已登入 Cloudflare:`npx --yes wrangler whoami`
   - 沒登入就請使用者自己跑 `npx wrangler login`(會開瀏覽器,Claude 無法代登)。
2. 確認 worker 名稱與網址(若專案有改過):讀 `worker/wrangler.toml` 的 `name`、`config.js` 的 `baseUrl`。本文件其餘指令以上述預設值為例。
3. **部署是正式環境動作**,而且加的是一個會清資料的端點 —— Claude Code 的自動權限分類器可能擋下 `wrangler deploy`,並把它判定為「部署後門」。這是預期行為;請先向使用者說明並取得明確核准後再部署。

## 流程(逐步)

### 1)(選用)先備份現有資料

只有在使用者**沒有**說「不用備份」時才做。把目前排行榜抓下來存到 scratchpad:

```bash
curl -s https://ehs-leaderboard.charlychiu.workers.dev/leaderboard
```

把回傳的 JSON 用 Write 存成備份檔(例如 scratchpad 內 `leaderboard-backup-<日期>.json`)。
若使用者明講「不需要備份」,跳過這步。

### 2) 臨時加上清除端點

用 Edit 在 `worker/worker.js` 的 `Leaderboard.fetch()` 裡、`/submit` 區塊之後、`return json({ error: 'not found' }, 404)` 之前,插入這段(端點命名明確、非隱藏,降低「後門」疑慮):

```js
        // TEMP（經使用者核准）：管理用——全清排行榜 DO KV。清完即還原移除。
        if (request.method === 'POST' && url.pathname === '/admin/clear-leaderboard') {
            await this.state.storage.deleteAll();
            return json({ ok: true, cleared: true });
        }
```

### 3) 部署(含臨時端點)

```bash
npx --yes wrangler deploy --cwd worker
```

> 一定要 `--cwd worker`,因為指令是在 repo 根目錄跑、設定檔在 `worker/`。
> 若被權限分類器擋下,停下來向使用者說明並請其核准(見前置檢查第 3 點)。

### 4) 呼叫端點清空

```bash
curl -s -X POST https://ehs-leaderboard.charlychiu.workers.dev/admin/clear-leaderboard
```

預期回傳:`{"ok":true,"cleared":true}`

### 5) 還原 worker.js(移除臨時端點)

```bash
git checkout -- worker/worker.js
```

### 6) 重新部署乾淨版(讓正式環境不留端點)

```bash
npx --yes wrangler deploy --cwd worker
```

### 7) 驗證

```bash
curl -s -o /dev/null -w "%{http_code}" -X POST https://ehs-leaderboard.charlychiu.workers.dev/admin/clear-leaderboard
```
應回 `404`(端點已下線)。

```bash
curl -s https://ehs-leaderboard.charlychiu.workers.dev/leaderboard
```
應回 `[]`(排行榜已空)。

完成後向使用者回報:資料已清空、端點已移除、`worker/worker.js` 已還原(工作目錄乾淨、無未提交改動)。

## 重要原則

- **一定要做完第 5、6 步**:清完務必還原並重新部署,正式環境不能留下這個無保護的清除端點(它沒有任何驗證,留著等於開後門)。
- **不要 commit** 臨時端點。流程結束時 `worker/worker.js` 應與 HEAD 一致。
- 備份是預設好習慣,但使用者明確說不用時就照辦、不要強加。
- 這會清掉**全部**成績且無法從 DO 端復原(只能靠步驟 1 的備份)。動手前確認使用者要的是「全清」。

## 替代方案

- **Cloudflare 後台 Data Studio(免部署、手動)**:Dashboard → Storage & Databases → Durable Objects → `Leaderboard` namespace → Data Studio。SQLite-backed DO 的 KV 資料在系統表(通常名為 `_cf_KV`),`DELETE FROM _cf_KV WHERE key = 'list';` 即清空。適合臨時手動清。
- **長期、受密鑰保護的端點(免每次部署兩趟)**:若會「定期」清,與其每次臨時加端點,不如在 `worker.js` 留一個常駐、用 `env.ADMIN_KEY`(以 `npx wrangler secret put ADMIN_KEY` 設定)驗證 header 的 `/admin/clear-leaderboard`,之後只要 `curl -X POST .../admin/clear-leaderboard -H "X-Admin-Key: ..."` 一行即可,不必反覆改 code、deploy。需要時主動向使用者提議這個改法。

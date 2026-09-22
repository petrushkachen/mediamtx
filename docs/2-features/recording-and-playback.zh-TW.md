# 錄影與回放功能詳解

本文說明 MediaMTX 的錄影（recording）與回放（playback）如何運作、適合解決哪些問題，以及部署時需要注意的限制。內容依據目前儲存庫版本 `04825598` 的設定、實作與 API 定義整理。

## 1. 功能定位

錄影與回放是兩個彼此分離、但透過磁碟檔案銜接的功能：

```text
發布者／靜態來源
        │
        ▼
  MediaMTX path ──────► 即時讀取端（RTSP、HLS、WebRTC……）
        │
        │ record: true
        ▼
  分段錄影檔（磁碟）
        │
        │ playback: true
        ▼
  Playback HTTP API ──► 瀏覽器、播放器、下載程式、事件檢索系統
```

- **錄影**：把 path 上的音視訊直接封裝成分段檔案，不重新編碼。適合監控存證、事件回溯、稽核、模型訓練素材保存與後續歸檔。
- **回放**：從錄影目錄尋找指定 path 與時間區間，透過 HTTP 列出可用時段或輸出影片。它是檔案型回放／下載服務，不會把歷史內容重新發布成 RTSP、HLS 或 WebRTC live stream。
- **Control API**：另外提供錄影檔的盤點與刪除能力，適合管理介面或維運工具；它和 Playback API 是不同的 HTTP 服務。

## 2. 最小可用設定

以下範例只替 `cameras/front-door` 錄影，並啟用 HTTP 回放：

```yaml
# 全域回放伺服器
playback: true
playbackAddress: :9996

paths:
  cameras/front-door:
    record: true
    recordPath: ./recordings/%path/%Y-%m-%d_%H-%M-%S-%f-%z
    recordFormat: fmp4
    recordPartDuration: 1s
    recordMaxPartSize: 50M
    recordSegmentDuration: 1h
    recordDeleteAfter: 7d
```

當來源發布到 `cameras/front-door` 後，檔案會出現在類似位置：

```text
./recordings/cameras/front-door/2026-09-22_14-30-00-123456-+0800.mp4
```

副檔名由 MediaMTX 根據 `recordFormat` 自動補上，不要寫進 `recordPath`。

## 3. 錄影如何運作

### 3.1 啟動與停止時機

`record` 是 path 層級設定，可放在 `pathDefaults` 套用到全部 path，也可在特定 `paths` 項目覆寫。

一般情況下，來源上線、path 取得可用 stream，而且 `record: true` 時，錄影器會以內部 reader 身分訂閱該 stream。來源離線或 path 不再可用時，錄影器會結束目前 segment 並關閉。這表示：

- 不需要有一般觀眾連線，MediaMTX 也能錄影。
- 錄影與即時協定解耦；只要內容已進入 MediaMTX 的 path，就由相同錄影流程處理。
- 不會進行轉碼，因此 CPU 成本通常低於轉碼式錄影，但輸入 codec 必須是目標容器支援的 codec。
- 錄影器因格式錯誤、時間戳漂移、part 過大或寫檔失敗而中止時，會記錄錯誤，等待約 2 秒後建立新的錄影實例。中間可能形成缺口。

### 3.2 錄影格式與 codec

| 容器 | 副檔名 | 視訊 codec | 音訊 codec | 內建 Playback API |
| --- | --- | --- | --- | --- |
| fMP4（fragmented MP4） | `.mp4` | AV1、VP9、H.265、H.264、MPEG-4 Video（H.263/Xvid）、MPEG-1/2 Video、M-JPEG | Opus、AAC、MP3、AC-3、G.711（PCMA/PCMU）、LPCM | **支援** |
| MPEG-TS | `.ts` | H.265、H.264、MPEG-4 Video（H.263/Xvid）、MPEG-1/2 Video | Opus、AAC、MP3、AC-3 | **不支援** |

選擇原則：

- 要使用 MediaMTX 內建的 `/list`、`/get` 回放功能，必須選 `recordFormat: fmp4`。
- MPEG-TS 對部分傳統廣播／播放器工作流較友善，但需使用外部工具讀取或轉封裝。
- 「容器支援」不等於「每個瀏覽器都能解碼」。例如伺服器可以回傳含 H.265 的 MP4，但使用者的瀏覽器與作業系統仍必須支援 H.265。

### 3.3 part 與 segment 的差異

| 名稱 | 代表意義 | 相關設定 | 實際用途 |
| --- | --- | --- | --- |
| part | segment 內較小的寫入／flush 單位 | `recordPartDuration` | 控制資料多久落盤，以及意外中斷時可能遺失的尾端長度 |
| segment | 使用者在磁碟上看到的完整錄影檔 | `recordSegmentDuration` | 控制檔案數量、單檔大小、歸檔與刪除粒度 |

`recordPartDuration` 預設為 1 秒。對 fMP4 而言，MediaMTX 先在記憶體組裝一個 part，再附加到 segment 檔案；對 MPEG-TS 而言，這個設定控制緩衝資料 flush 到磁碟的週期。因此系統當機時，最後尚未完成的 part 可能遺失，這個值可視為錄影的 RPO（Recovery Point Objective）。值越短，資料更快落盤，但 I/O 次數會增加。

`recordSegmentDuration` 預設為 1 小時，而且是**最短目標時間**，不保證精確等長：

- 有視訊時，超過目標時間後會等到下一個可隨機存取的畫面（例如 H.264 IDR／關鍵影格）才切換檔案。
- 只有音訊時，不需要等待視訊關鍵影格，切檔時間通常更接近設定值。
- 設定上限為 24 小時。

因此，如果攝影機的關鍵影格間隔很長，segment 也可能明顯長於 `recordSegmentDuration`。

### 3.4 記憶體保護

對 fMP4 錄影而言，`recordMaxPartSize` 限制單一 part 在記憶體中的最大 payload 大小，預設 50 MB，用來避免異常碼率或長時間無法封閉 part 導致記憶體耗盡。超過限制會讓該錄影實例報錯並重新啟動；它不是單一 segment 的檔案大小上限。目前 MPEG-TS 寫入流程使用固定的 buffered writer，沒有套用這個 fMP4 part 大小檢查。

### 3.5 檔名、時間與路徑規則

`recordPath` 可使用：

| 變數 | 內容 |
| --- | --- |
| `%path` | MediaMTX path 名稱 |
| `%Y`、`%m`、`%d` | 年、月、日 |
| `%H`、`%M`、`%S` | 時、分、秒 |
| `%f` | 6 位數微秒 |
| `%z` | 時區，例如 `+0800` 或 `Z` |
| `%s` | Unix epoch 秒 |

設定驗證有下列要求：

- 必須包含 `%path`。
- 必須包含 `%s`，或同時包含完整的 `%Y %m %d %H %M %S`。
- 啟用全域 `playback` 時必須包含 `%f`，以便唯一識別 segment。
- 建議加上 `%z`，讓搬移檔案、跨時區部署與日光節約時間切換時仍能正確解讀錄影時間。

segment 的起始時間取自第一批可錄製 sample 的絕對時間。預設 `useAbsoluteTimestamp: false`，MediaMTX 會以伺服器系統時間替換來源提供的絕對時間，避免不可信或缺失的來源時鐘污染錄影日期。若多台來源已可靠同步，而且需要保留其原始絕對時間，可在 path 設定：

```yaml
paths:
  cameras/front-door:
    useAbsoluteTimestamp: true
    record: true
```

若相對媒體時間和絕對時間的漂移超過容許範圍，錄影器會重設，以免產生時間軸錯亂的檔案。

### 3.6 always-available 模式

當 `alwaysAvailable: true` 時，來源離線期間會以離線片段維持 stream。預設 `alwaysAvailableRecorded: true`，所以離線片段也會被錄下來。若錄影只應包含真實來源內容，必須改成：

```yaml
paths:
  cameras/front-door:
    alwaysAvailable: true
    alwaysAvailableRecorded: false
    record: true
    alwaysAvailableTracks:
      - codec: H264
```

此時來源上線才啟動錄影，來源離線則關閉當前錄影。

### 3.7 自動保留期限與清理

`recordDeleteAfter` 控制 segment 保留多久：

- 預設 `1d`。
- 設成 `0s` 可停用自動刪除。
- 非零值不得短於 `recordSegmentDuration`。
- 清理器啟動時會先掃描一次，之後最多每 30 分鐘掃描；若保留期限很短，掃描間隔會縮短到該期限的一半。
- 清理是依 segment 的錄影時間判斷，並在刪檔後嘗試移除空目錄。因此檔案可能略晚於期限才真正消失。

正式環境還應獨立監控磁碟容量。`recordDeleteAfter` 是時間型保留策略，不是磁碟配額；突增的碼率、路數或來源數量仍可能提前填滿磁碟。

### 3.8 segment hook 與遠端歸檔

MediaMTX 提供兩個 path hook：

- `runOnRecordSegmentCreate`：segment 檔案建立時觸發，提供 `MTX_SEGMENT_PATH`。
- `runOnRecordSegmentComplete`：segment 完整關閉後觸發，提供 `MTX_SEGMENT_PATH` 與以秒表示的 `MTX_SEGMENT_DURATION`。

兩者也能取得 `MTX_PATH`、`RTSP_PORT`，正規表示式 path 還能取得 `G1`、`G2` 等群組。上傳、校驗、索引或通知通常應放在 `runOnRecordSegmentComplete`，因為 create 時檔案仍在寫入。

例如可在 complete hook 呼叫 `rclone`，把完成的 segment 同步到 S3、FTP、SMB 或其他遠端儲存。若使用移動而不是複製，應注意內建回放只能看到本機 `recordPath` 中仍存在的檔案。

## 4. 回放如何運作

### 4.1 獨立 HTTP／HTTPS 服務

回放伺服器預設停用，預設監聽位置為 `:9996`。完整的全域設定如下：

```yaml
playback: true
playbackAddress: :9996
playbackEncryption: false
playbackServerKey: server.key
playbackServerCert: server.crt
playbackAllowOrigins: ["*"]
playbackTrustedProxies: []
```

- `playbackEncryption: true` 時使用 HTTPS，並讀取指定憑證與私鑰。
- `playbackAllowOrigins` 控制 CORS。預設 `[*]` 方便網頁使用；正式環境建議縮小到實際前端網域。
- 反向代理的 IP／網段應加入 `playbackTrustedProxies`。只有受信任代理送來的 `X-Forwarded-Proto` 才會用於 `/list` 產生的回放 URL。
- 回放伺服器掃描各 path 設定對應的 `recordPath`。只要檔案仍在且命名符合設定，即使當下沒有錄影，也能讀取既有錄影。

### 4.2 重要格式限制

目前 `/list` 與 `/get` 只解析 `recordFormat: fmp4` 產生的檔案。若 path 使用 `recordFormat: mpegts`，會得到「MPEG-TS format is not supported yet」錯誤。

`/get?...&format=mp4` 的 `mp4` 指輸出容器：伺服器從 fMP4 segment 讀取 sample，再即時重新封裝成標準 MP4；這不會轉碼，也不能拿來讀取 `.ts` 錄影。

### 4.3 列出可回放時段：`GET /list`

請求格式：

```text
GET /list?path=<path>&start=<RFC3339>&end=<RFC3339>
```

| 參數 | 必填 | 說明 |
| --- | --- | --- |
| `path` | 是 | 要查詢的 MediaMTX path |
| `start` | 否 | 查詢起點，RFC 3339 |
| `end` | 否 | 查詢終點，RFC 3339 |

所有 query 值都應進行 URL encoding。例如：

```bash
curl "http://localhost:9996/list?path=cameras%2Ffront-door&start=2026-09-22T06%3A00%3A00Z&end=2026-09-22T07%3A00%3A00Z"
```

回應是一組連續時段：

```json
[
  {
    "start": "2026-09-22T14:00:00+08:00",
    "duration": 1800.5,
    "url": "http://localhost:9996/get?duration=1800.5&path=cameras%2Ffront-door&start=2026-09-22T14%3A00%3A00%2B08%3A00"
  }
]
```

伺服器會讀取 segment header；若錄影尚未正常關閉而 header 沒有總長度，則解析已寫入的 fMP4 parts 計算長度。屬於同一錄影實例且 segment 編號連續的檔案會合併成一個時段；舊版檔案則依 track 相容性與時間是否近乎相接來判斷。來源重連、錄影器重啟、codec／track 改變或中間缺檔，通常會形成新的時段。

若有 `start`／`end`，第一與最後一個時段會裁切成查詢邊界；沒有符合檔案時回傳 404。

### 4.4 取得指定時段：`GET /get`

請求格式：

```text
GET /get?path=<path>&start=<RFC3339>&duration=<seconds>&format=<fmp4|mp4>
```

| 參數 | 必填 | 說明 |
| --- | --- | --- |
| `path` | 是 | MediaMTX path |
| `start` | 是 | 起始時間，RFC 3339 |
| `duration` | 是 | 最長輸出秒數，可含小數，例如 `200.5` |
| `format` | 否 | `fmp4`（預設）或 `mp4` |

舊版 Go duration 寫法（例如 `1m30s`）仍可解析，但已棄用；新整合應一律傳秒數。

下載 200.5 秒的預設 fMP4：

```bash
curl -o clip.mp4 "http://localhost:9996/get?path=cameras%2Ffront-door&start=2026-09-22T14%3A33%3A17%2B08%3A00&duration=200.5"
```

要求標準 MP4：

```bash
curl -o clip.mp4 "http://localhost:9996/get?path=cameras%2Ffront-door&start=2026-09-22T14%3A33%3A17%2B08%3A00&duration=200.5&format=mp4"
```

`duration` 是上限，不是保證值。遇到錄影結尾、缺口、錄影實例改變或不相容的 track 時，輸出會提早結束。為了產生可解碼影片，實際取樣還要考慮關鍵影格與各 track 的時間軸，因此不要把它視為逐幀剪輯器；需要幀精準剪輯時，應下載後再交給 FFmpeg 等工具處理。

回應使用 `Content-Type: video/mp4`，但明確設定 `Accept-Ranges: none`，不支援 HTTP byte-range。因此：

- 不能依靠 Range request 做續傳或任意位元組定位。
- 大片段建議拆成較短的時間請求。
- 預設 fMP4 可分段產生，適合瀏覽器漸進播放；標準 MP4 相容性通常較高，但伺服器需先組合相應的 sample 資訊，長片段的啟播成本通常較高。

### 4.5 在網頁播放

預設 fMP4 回應可直接放進 `<video>`：

```html
<video controls preload="metadata">
  <source
    src="http://localhost:9996/get?path=cameras%2Ffront-door&amp;start=2026-09-22T14%3A33%3A17%2B08%3A00&amp;duration=200.5"
    type="video/mp4"
  />
</video>
```

若播放器不接受 fragmented MP4，可加上 `format=mp4`。若容器可載入但畫面或聲音不能播放，優先檢查瀏覽器是否支援錄影中的實際 codec。

### 4.6 驗證、CORS 與錯誤碼

Playback API 的授權動作是 `playback`，可針對 path 限制：

```yaml
authInternalUsers:
  - user: viewer
    pass: change-me
    permissions:
      - action: playback
        path: cameras/front-door
```

可使用 Basic credentials，或採用專案支援的外部 HTTP／JWT 驗證。未通過驗證時回傳 401；需要 Basic credentials 時會包含 `WWW-Authenticate: Basic realm="mediamtx"`。若跨網域前端要送出 `Authorization`，也必須正確設定允許來源；預檢請求支援 `OPTIONS, GET` 與 `Authorization` header。

常見狀態碼：

| 狀態碼 | 意義 |
| --- | --- |
| `200` | 查詢或影片輸出成功 |
| `204` | CORS 預檢成功 |
| `400` | path、時間、duration、format 或 path 設定無效；檔案損毀也可能在尚未輸出影片前得到 400 |
| `401` | 驗證失敗 |
| `404` | 指定 path／時間範圍沒有錄影 |
| `500` | `/list` 掃描或解析錄影時發生內部錯誤，例如嘗試列出 MPEG-TS 錄影 |

若影片資料已經開始傳送才發生錯誤，HTTP 狀態與 JSON 錯誤已無法安全改寫；伺服器會中止輸出並把錯誤寫入 log。因此客戶端也應檢查下載是否完整。

## 5. 用 Control API 管理錄影

Control API 預設停用；啟用後預設監聽 `:9997`：

```yaml
api: true
apiAddress: :9997
```

它提供：

| 方法與路徑 | 用途 |
| --- | --- |
| `GET /v3/recordings/list` | 依 path 分頁列出全部錄影與各 segment 起始時間 |
| `GET /v3/recordings/get/{name}` | 列出指定 path 的 segment |
| `DELETE /v3/recordings/segments/delete?path=...&start=...` | 刪除起始時間完全相符的單一 segment |

例如：

```bash
curl "http://localhost:9997/v3/recordings/list?page=0&itemsPerPage=100"

curl "http://localhost:9997/v3/recordings/get/cameras/front-door"

curl -X DELETE "http://localhost:9997/v3/recordings/segments/delete?path=cameras%2Ffront-door&start=2026-09-22T14%3A30%3A00.123456%2B08%3A00"
```

刪除 API 的 `start` 用來重新建構檔名，必須使用查詢 API 回傳的實際 segment 起始時間。Control API 使用 `api` 權限，而不是 `playback` 權限；正式環境不應把管理 API 當成公開的觀看端點。

## 6. 常見用途與建議

### 監控事件回溯

以 `/list` 先取得有資料的連續時段，再讓使用者選取時間，最後呼叫 `/get`。不要先假設整天都有內容，否則來源離線缺口會造成大量 404 或短片段。

### 事件前後片段匯出

收到事件時間 `T` 後，要求 `start=T-10s`、`duration=40`，即可匯出事件前 10 秒到事件後 30 秒。若事件跨越來源重連或錄影缺口，應依 `/list` 結果拆成多個檔案。

### 長期歸檔

使用 `runOnRecordSegmentComplete` 上傳已完成的 segment，並以物件儲存的 lifecycle policy 管理長期保存。若本機仍要提供近期回放，可用 copy/sync 並保留短期 `recordDeleteAfter`；若用 move，本機 Playback API 將無法讀到已搬走的內容。

### 容量估算

可用下式粗估，不含容器開銷：

```text
每日容量（GB）≈ 總碼率（Mbit/s）× 10.8
```

例如單路 4 Mbit/s 約為 43.2 GB／日；20 路約為 864 GB／日。規劃時還要預留檔案系統、碼率尖峰與清理延遲空間。

## 7. 疑難排解

### 錄影目錄沒有檔案

依序檢查：

1. `record: true` 是否套用到實際 path。
2. 來源是否真的上線並送出該容器支援的 codec。
3. fMP4 視訊是否已送出第一個關鍵影格；錄影通常要等到可隨機存取畫面才開始形成有效內容。
4. 執行帳號是否能建立 `recordPath` 的目錄與檔案。
5. 是否啟用 `alwaysAvailableRecorded: false`，而來源目前離線。
6. log 是否出現時間戳漂移、`reached maximum part size` 或寫檔錯誤。

### `/list` 回傳 404

- path 名稱、大小寫與 URL encoding 是否正確。
- `start`／`end` 是否使用 RFC 3339 且涵蓋檔案時間。
- 目前設定的 `recordPath` 與 `recordFormat` 是否仍和產生檔案時一致。
- segment 是否已被 `recordDeleteAfter` 清除或被歸檔 hook 搬走。

### `/list` 回傳 500，訊息指出 MPEG-TS 尚不支援

這是目前的預期限制。改用 `recordFormat: fmp4` 產生新錄影；既有 `.ts` 檔需交給外部 HTTP／媒體服務，或先用 FFmpeg 等工具轉封裝。

### 影片可以下載但瀏覽器不能播放

- 改試 `format=mp4`，排除播放器不支援 fragmented MP4。
- 用媒體分析工具確認實際 codec；MP4 容器本身可用，不代表瀏覽器支援其中的 H.265、AC-3 等 codec。
- 檢查下載是否因伺服器端解析錯誤或網路中斷而提早結束。

### segment 比設定時間長

`recordSegmentDuration` 是最短目標；視訊錄影要等下一個關鍵影格才能安全切檔。縮短攝影機／編碼器的 GOP 或 keyframe interval，通常比繼續縮短 `recordSegmentDuration` 更有效。

## 8. 實作對照

需要進一步追查時，可從下列位置開始：

| 功能 | 主要檔案 |
| --- | --- |
| 錄影器生命週期與錯誤後重啟 | `internal/recorder/recorder.go`、`internal/recorder/recorder_instance.go` |
| fMP4／MPEG-TS 封裝與切檔 | `internal/recorder/format_fmp4*.go`、`internal/recorder/format_mpegts*.go` |
| 檔名編解碼與 segment 掃描 | `internal/recordstore/path.go`、`internal/recordstore/segment.go` |
| 過期錄影清理 | `internal/recordcleaner/cleaner.go` |
| Playback HTTP server | `internal/playback/server.go` |
| `/list` 與連續時段合併 | `internal/playback/on_list.go` |
| `/get` 與輸出重新封裝 | `internal/playback/on_get.go`、`internal/playback/muxer_*.go` |
| Control API 錄影管理 | `internal/api/api_recordings.go` |
| 公開 Playback API 規格 | `api/playback.openapi.yaml` |

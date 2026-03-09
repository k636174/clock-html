# Redmine「自分のチケット一覧」機能 仕様書

## 1. 目的
既存の時計・天気表示アプリ（`index.html`）に、Redmine 上でログインユーザーに紐づくチケット（Issue）一覧を表示する機能を追加する。

想定利用シーン:
- 壁掛け表示や常時表示で、現在時刻・天気とあわせて自分の作業チケットを確認する
- 朝会前や作業開始時に「担当中・期限切れ・本日期限」を即座に把握する

---

## 2. 対象範囲
### 2.1 対象
- フロントエンド単体（現在の `index.html` に機能追加）
- Redmine REST API 連携
- ローカル保存（表示条件・接続設定）

### 2.2 非対象
- Redmine 側のプラグイン開発
- ユーザー管理/認証基盤の追加
- サーバーサイドの中継 API（初期版はブラウザから直接 API 呼び出し）

---

## 3. 用語
- **チケット**: Redmine Issue
- **自分のチケット**: 以下のいずれか
  - `assigned_to_id=me`（自分に割り当て）
  - `author_id=me`（自分が起票）※表示オプションで切替
- **APIキー**: Redmine 個人用アクセストークン

---

## 4. 画面仕様
## 4.1 レイアウト
既存の 2 カラム UI に新規セクションを追加。
- 推奨: 右カラム（現在天気セクションがある領域）内に「Redmine」ブロックを追加
- モバイル幅（`max-width: 900px`）では既存セクションの下に縦積み

## 4.2 表示要素
1. セクションタイトル
   - `Redmineチケット`
2. 接続設定フォーム（初期は折りたたみ可）
   - Redmine Base URL（例: `https://redmine.example.com`）
   - APIキー（password 表示）
   - 取得対象（ラジオ/セレクト）
     - 担当チケット（既定）
     - 起票チケット
   - 保存ボタン
   - 設定クリアボタン
3. ステータス表示行
   - 接続中/更新時刻/エラー内容
4. チケット一覧領域
   - 1行に以下を表示
     - `#ID`
     - 件名（subject）
     - プロジェクト名
     - ステータス
     - 優先度
     - 期日（due_date、未設定は `-`）
   - クリックで Redmine チケット URL を新規タブで開く
5. サマリー行
   - 総件数
   - 期限切れ件数
   - 本日期限件数

## 4.3 並び順
- 第1キー: 期限日昇順（未設定は最後）
- 第2キー: 優先度降順（可能な範囲で）
- 第3キー: ID降順

## 4.4 色分け
- 期限切れ: 赤系
- 本日期限: 黄系
- 完了系ステータス（Closed/Resolvedなど）: 減色（薄い文字）

---

## 5. データ取得仕様
## 5.1 API エンドポイント
- `GET /issues.json`
- 例:
  - 担当: `/issues.json?assigned_to_id=me&status_id=*&sort=due_date:asc,priority:desc,id:desc&limit=50`
  - 起票: `/issues.json?author_id=me&status_id=*&sort=due_date:asc,priority:desc,id:desc&limit=50`

## 5.2 リクエストヘッダ
- `X-Redmine-API-Key: {API_KEY}`
- `Content-Type: application/json`（GET は任意）

## 5.3 取得項目
- `issues[].id`
- `issues[].subject`
- `issues[].project.name`
- `issues[].status.name`
- `issues[].priority.name`
- `issues[].due_date`
- `issues[].updated_on`

## 5.4 ページング
- 初期版は `limit=50` 固定
- `total_count > 50` の場合は「50件まで表示」注記を表示
- 将来的に `offset` 追跡で全件取得拡張可能

## 5.5 更新タイミング
- 画面ロード時に 1 回
- 5 分間隔で自動更新（時計/天気と同じ周期に合わせる）
- 手動更新ボタン（任意、推奨）

---

## 6. 保存仕様（localStorage）
キー案:
- `clock_redmine_base_url`
- `clock_redmine_api_key`
- `clock_redmine_target` (`assigned` / `authored`)

注意:
- APIキーを localStorage 保存する場合、XSS リスクを注意喚起
- 公開端末では保存しない運用を推奨（将来はセッション保存切替を追加）

---

## 7. エラーハンドリング
- URL 未入力: `Redmine URLを入力してください`
- APIキー未入力: `APIキーを入力してください`
- 401/403: `認証に失敗しました（APIキーを確認してください）`
- 404: `Redmine API endpoint が見つかりません`
- CORS: `CORS設定によりブラウザからアクセスできません`
- 通信失敗: `ネットワークエラー`
- JSON 解析失敗: `レスポンスの解析に失敗しました`

UI動作:
- エラー時は前回正常データを残し、ステータス行のみエラー表示
- 初回取得で失敗した場合は一覧領域にエラーメッセージを表示

---

## 8. セキュリティ/運用考慮
1. APIキーの扱い
   - 入力欄はマスク表示
   - 画面上に平文再表示しない
2. CORS
   - Redmine サーバー側で当該表示端末 Origin の許可が必要
3. ログ
   - `console.log` に APIキーを出力しない
4. 公開環境
   - キオスク端末では専用の閲覧専用アカウント利用を推奨

---

## 9. 受け入れ条件
1. 正常系
   - 設定保存後、担当（または起票）チケットが表示される
   - 5分以内に自動更新される
   - 期限切れ・本日期限の色分けが適用される
2. 異常系
   - 不正APIキーで認証エラー表示
   - Redmine未接続時にネットワークエラー表示
   - CORS ブロック時に原因が判別可能な文言を表示
3. 永続化
   - リロード後に URL/対象設定が復元される
   - APIキー保存有無の仕様どおり動作する

---

## 10. 実装ガイド（現行コードへの適用）
対象ファイル: `index.html`

追加・修正ポイント:
1. HTML
   - Redmine セクション（フォーム、ステータス、一覧）追加
2. CSS
   - `.redmine`, `.ticket-list`, `.ticket-row`, `.is-overdue`, `.is-due-today` など追加
3. JS
   - 定数（storage key, interval）追加
   - `buildRedmineIssuesUrl(baseUrl, target)`
   - `fetchRedmineIssues(baseUrl, apiKey, target)`
   - `renderRedmineIssues(issues)`
   - `saveRedmineSettings()` / `loadRedmineSettings()`
   - `updateRedmineTickets()`
4. 初期化
   - 既存の `updateClock`, `updateWeather` と同様に初回実行 + 定期実行

---

## 11. 将来拡張
- フィルタ（プロジェクト、トラッカー、ステータス）
- 表示件数切替（20/50/100）
- ガント/カレンダー連携
- バックエンドプロキシ経由で APIキー秘匿
- Webhook/Polling 最適化


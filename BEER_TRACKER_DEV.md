Beer Tracker 開発引き継ぎドキュメント
プロジェクト概要
アプリ名: クラフトビール チェックイン
本番URL: https://charincorin-ship-it.github.io/beer-tracker-/
リポジトリ: charincorin-ship-it/beer-tracker-
構成: 単一HTMLファイル（index.html）+ localStorage + GAS連携（任意）
コンセプト: 1日1回2時間飲み放題のクラフトビールサブスク店専用記録アプリ
開発体制（三AI体制）
役割
担当
メインコーダー
Claude
デバッガー・コードレビュー
ChatGPT（チャッピーさん）
UXディレクター
Gemini（御曹司）
フィードバックは手動で各AIに橋渡しして進める方式。
CHANGELOG
v2.0（現行版）← v1.9からの変更
機能1: 同日再入店ロック
退店済みの当日は入店ボタンをbtn-disabledでロック
hasTodayExit()関数で判定、updateUI()内でtodayExitedフラグとして使用
ロック中は入店ボタン直下に「本日は入退店済みです（設定からリセット可能）」を表示
機能2: textarea自動伸長
initAutoResize()でイベント委譲方式（document.addEventListener('input', ...)）を採用
el.matches()でクラス判定するため、動的生成の.h-add-inputも追加バインド不要
機能3: ビール記録ブロックの構造化
旧: r.memo（文字列1フィールド）
新: r.beerData = { name, rating, comment, url, photo }（オブジェクト）
星評価（★1〜5）をタップで即反映するUIを実装
フォト欄はURL貼り付け→リンク変換方式を継続（直接アップロードは容量問題のため不採用）
機能4: メモ欄のフォト分離
旧: r.memo（テキストのみ）
新: r.noteData = { text, photo }（テキスト＋フォトURL）
退店リセット機能
設定タブに「本日の退店ステータスをリセット」ボタンを追加
confirm()は環境依存のためインライン確認ダイアログ方式に統一
リセット時はexitとenterを両方削除（enterのみ残ると「入店中」判定になるバグを修正）
beer・noteレコードはリセット対象外（当日の記録は保持）
編集UI変更
旧: インライン編集（input要素を動的挿入）
新: ボトムシートモーダル（ビール用・メモ用それぞれ専用）
モーダル表示中はbody { overflow: hidden }でスクロール抑制
バグ修正（チャッピーさん指摘）
doBeerMemo()内で_currentRatingをリセット前に変数退避。リセット後にsendGas()が走っていたためGAS側にrating: 0が送信されていたバグを修正
v1.9（前バージョン）← v1.8からの変更
退店ボタンのdisabled属性廃止 → クラス制御＋インライン確認ダイアログ
GoogleフォトURL（photos.app.goo.gl等）を📸 写真リンクに変換
カウント外コメント欄の追加（type:'note'、ビール本数に含まれない）
履歴からの追加を「🍺ビール」「📝メモ」の2択に分離
削除・編集ロジックの厳密化（type+ts+dateの3条件）
過去日追加時のタイムスタンプを正午JST基準に変更
Enterキーを改行のみに変更（送信はボタンのみ）
GAS通信をcors優先＋no-corsフォールバック構成に変更
データ構造
localStorageキー
beer_tracker_v1（変更しないこと）
レコード型一覧
// 入店
{ "type": "enter", "ts": "ISO8601", "date": "YYYY-MM-DD" }

// 退店
{ "type": "exit", "ts": "ISO8601", "date": "YYYY-MM-DD", "stayMin": 120 }

// ビール記録（新形式 v2.0〜）
{
  "type": "beer",
  "ts": "ISO8601",
  "date": "YYYY-MM-DD",
  "beerData": {
    "name": "ビール名",
    "rating": 4,
    "comment": "感想",
    "url": "https://...",
    "photo": "https://photos.app.goo.gl/..."
  }
}

// ビール記録（旧形式 〜v1.9 後方互換あり）
{ "type": "beer", "ts": "ISO8601", "date": "YYYY-MM-DD", "memo": "文字列" }

// メモ（新形式 v2.0〜）
{
  "type": "note",
  "ts": "ISO8601",
  "date": "YYYY-MM-DD",
  "noteData": { "text": "メモ本文", "photo": "https://..." }
}

// メモ（旧形式 〜v1.9 後方互換あり）
{ "type": "note", "ts": "ISO8601", "date": "YYYY-MM-DD", "memo": "文字列" }

// 退社
{ "type": "office", "ts": "ISO8601", "date": "YYYY-MM-DD" }
後方互換の仕組み
getBeerData(r)関数が型判定を行う。r.beerDataがオブジェクトなら新形式、なければ旧形式としてr.memoをビール名に充当。全フィールドを空文字で安全初期化済み。
設計判断メモ
判断
理由
フォトはURL貼り付け方式を継続
base64直接保存はlocalStorage容量（5MB上限）を圧迫するリスク
データマイグレーション不要（案A）
インポート時に旧データが壊れるリスクを避けるため
GASはJSONを1カラムに流す方式
スプレッドシート側カラム細分化より安全・変更コスト低
confirm()不使用
iframeやブラウザ環境によってブロックされる場合があるため
enterもリセット対象に含める
exitのみ削除するとenterが残り「入店中」判定になるバグを防ぐ
イベント委譲方式のautoResize
動的生成の.h-add-inputにも個別バインド不要で確実に動作させるため
注意事項（次バージョン開発時）
触っても安全
CSS変数（:rootの--amber等）の色変更
font-size、padding、border-radius等のサイズ調整
要注意
クラス名の変更・削除 → JSがclassListでクラスを参照しているため壊れる
id属性の変更 → getElementByIdで多用しているため壊れる
.btn-disabledクラス → 見た目だけでなくロジック判定にも使用している
TODO（次バージョン候補）
[ ] デザイン刷新（時期未定）
[ ] その他（使いながら洗い出し中）
機種変更時の手順
設定タブ →「エクスポート」でJSONをダウンロード
新機種で同URLを開く
設定タブ →「インポート」でJSONを読み込む

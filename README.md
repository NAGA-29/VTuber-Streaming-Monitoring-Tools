# VTuber 配信監視ツール群

VTuber（主にホロライブ・のりプロ）の配信を多角的に監視・通知するためのツール群です。
YouTubeチャンネルのRSSフィードやAPIを定期的に監視し、新着動画や配信の更新を検知して、データベースへの記録やLINE・Twitterでの通知を行います。

## 主な機能と処理フロー

このツール群は、複数の独立したPythonスクリプトが連携して動作します。主な処理の流れは以下の通りです。

### 動画監視フロー

1.  **検知と登録 (`controller/HoloTubeSearchController.py`)**
    *   各チャンネルのYouTube RSSフィードを定期的に巡回します。
    *   新しい動画（配信待機枠、プレミア公開動画、通常動画）が投稿されると、それを検知します。
    *   既存の動画でタイトルやサムネイルの変更があった場合も検知します。
    *   検知した動画の詳細はYouTube APIで取得し、マスターテーブルである `youtube_videos` に記録します。
    *   動画が配信予定のライブやプレミア公開である場合、配信前監視用の `keep_watchs` テーブルにも登録されます。
    *   新着動画や更新情報は、LINEやTwitterを通じて通知されます。

2.  **配信前監視 (`controller/keepWatchController.py`)**
    *   `keep_watchs` テーブルを定期的に監視します。
    *   配信開始時刻が近づくと、YouTube APIを通じて動画の状態をより頻繁にチェックします。
    *   配信予定時刻が変更された場合は、それを検知して通知します。
    *   動画が実際にライブ配信を開始したことを検知すると、その動画を `keep_watchs` テーブルから削除し、配信中監視用の `now_live_keep_watchs` テーブルへ移動させます。
    *   配信前に枠が削除された場合なども検知し、テーブルから自動的に削除します。

3.  **配信中監視 (`controller/liveController.py`)**
    *   `now_live_keep_watchs` テーブルを定期的に監視します。
    *   YouTube APIを通じて、現在の視聴者数などのライブ情報を取得し続けます。
    *   視聴者数が特定の閾値（例：3万人）を超えるなど、配信が特に盛り上がっていると判断された場合に「ホットなLIVE」としてTwitterで通知します。
    *   配信が終了したことを検知すると、その動画を `now_live_keep_watchs` テーブルから削除します。

### その他の主要なツール

*   **`get_replay_chat.py` / `chat/get_live_chat.py`**: 配信のアーカイブやライブ中のチャットログを取得します。
*   **`controller/reminderController.py`**: 配信のリマインド機能を提供します。（詳細は要解析）
*   **`controller/service/youtube/youtube_subscriber.py`**: YouTubeチャンネルの登録者数を監視し、変動を通知します。（詳細は要解析）
*   **`controller/schedule/event_schedule.py`**: VTuberのイベントスケジュールを通知します。

## セットアップ手順

### 1. 依存ライブラリのインストール

以下のコマンドを実行して、必要なライブラリをインストールします。

```bash
pip install -r requirements.txt
```

### 2. データベースの準備

このツールはMySQLデータベースを使用します。

開発初期版では `holo_sql.py` に MySQL の接続情報（例として `root` / `root`）がハードコードされていますが、**`root` アカウントや弱いパスワードをそのまま使用するのは安全ではありません**。必ず専用のアプリケーションユーザーを作成し、強力なパスワードを設定してください。

以下は、ローカル開発用の「例」として推奨される構成です。**実運用・ネットワーク越しにアクセスされ得る環境では、必ず独自の強力なパスワードに変更し、この例の値をそのまま使わないでください。**

*   **ホスト**: `localhost`
*   **ポート**: `8889`
*   **ユーザー**: `holo_app`  （例：このツール専用のアプリケーションユーザー）
*   **パスワード**: `<STRONG_UNIQUE_PASSWORD_HERE>`  （十分に長くランダムなパスワードに置き換えてください）
*   **データベース名**: `Hololive_Project`

アプリケーションユーザーには、原則として `Hololive_Project` データベースに対する必要最小限の権限（例：`SELECT`, `INSERT`, `UPDATE`, `DELETE`, 必要に応じて `CREATE`）のみを付与してください。  
テーブルは、各スクリプトを初めて実行した際に `CREATE TABLE IF NOT EXISTS` クエリによって自動的に作成されます。

なお、`holo_sql.py` のハードコードされた接続情報も、上記の専用ユーザーと強力なパスワードに合わせて変更するか、`.env` などの環境変数から読み込む方式に置き換えることを推奨します。
### 3. 環境変数の設定

プロジェクトのルートディレクトリに `.env` という名前のファイルを作成し、各種APIキーや設定値を記述します。
`config/database.py` を参考に、以下の内容でファイルを作成してください。

```dotenv
# .env ファイルのテンプレート

# --- ディレクトリ設定 ---
# サムネイル画像の保存先など、プロジェクトの構造に合わせて設定
LIVE_TMB_IMG_DIR='src/live_thumbnail_image/'
LIVE_TMB_TMP_DIR='src/live_temporary_image/'
NoriP_LIVE_TMB_TMP_DIR='src/noripro_live_temporary_image/'
OTHER_TMB_TMP_DIR='src/other_temporary_image/'
IMG_TRIM_DIR='src/Trim_Images/'
PROFILE_IMG_DIR='src/Profile_Images/'
EVENT_IMG_DIR='src/Event_Images/'
SCREENSHOT_DIR='storage/screenshot_image/'
COMBINE_IMG_DIR='src/Combine_Image/'
HOLO_IMG_IMG_DIR='src/Holo_Data_Image/'
DRIVER_PATH='drivers/chromedriver/'

# --- APIキー設定 ---
# Twitch API
TWITCH_CLIENT='YOUR_TWITCH_CLIENT_ID'
TWITCH_SECRET='YOUR_TWITCH_CLIENT_SECRET'

# Discord Bot Token
DISCORD_TOKEN='YOUR_DISCORD_BOT_TOKEN'

# LINE Notify Token
LINE_NOTIFY_TOKEN='YOUR_LINE_NOTIFY_TOKEN'

# Twitter API (用途別に複数設定されている)
# My_Hololive_Project 用
CONSUMER_KEY='YOUR_CONSUMER_KEY'
CONSUMER_SECRET='YOUR_CONSUMER_SECRET'
ACCESS_TOKEN='YOUR_ACCESS_TOKEN'
ACCESS_TOKEN_SECRET='YOUR_ACCESS_TOKEN_SECRET'
BEARER_TOKEN='YOUR_BEARER_TOKEN'
# My_HoloNoriArts_Project 用
CONSUMER_KEY_A='...'
CONSUMER_SECRET_A='...'
ACCESS_TOKEN_A='...'
ACCESS_TOKEN_SECRET_A='...'
BEARER_TOKEN_A='...'
# NoriUi_Project 用
CONSUMER_KEY_B='...'
CONSUMER_SECRET_B='...'
ACCESS_TOKEN_B='...'
ACCESS_TOKEN_SECRET_B='...'
BEARER_TOKEN_B='...'
# テスト用
CONSUMER_KEY_TEST='...'
CONSUMER_SECRET_TEST='...'
ACCESS_TOKEN_TEST='...'
ACCESS_TOKEN_SECRET_TEST='...'

# YouTube Data API Key (複数キーのローテーションに対応)
YOUTUBE_API_KEY01='YOUR_YOUTUBE_API_KEY_1'
YOUTUBE_API_KEY_dev1='YOUR_YOUTUBE_API_KEY_2'
YOUTUBE_API_KEY_dev2='YOUR_YOUTUBE_API_KEY_3'
YOUTUBE_API_KEY_dev3='YOUR_YOUTUBE_API_KEY_4'
YOUTUBE_API_KEY_dev4='YOUR_YOUTUBE_API_KEY_5'

# Bitly API Access Token
BITLY_ACCESS_TOKEN='YOUR_BITLY_ACCESS_TOKEN'

# News API Key
NEWS_API='YOUR_NEWS_API_KEY'
```

`.env` に記載する環境変数の一覧は上記に加えて、リポジトリ直下にある `.env.sample` も必ず参照してください。特に `config/database.py` などでは、サムネイル保存用ディレクトリを指す `LIVE_TMB_IMG_DIR` や一時ディレクトリ用の `LIVE_TMB_TMP_DIR` など、ここでは省略しているディレクトリ系の環境変数も参照します。

`.env` を作成する際は、`.env.sample` に定義されているすべての環境変数をコピーした上で、このセクションのコメントを参考に値を設定してください。
## 実行方法

各監視スクリプトは、ターミナルから直接実行します。これらは永続的に動作させる必要があるため、`nohup` や `tmux`, `screen` などを利用してバックグラウンドで実行することを推奨します。

```bash
# 例: ホロライブの新着動画監視を開始
python3 controller/HoloTubeSearchController.py

# 例: ライブ配信中の動画監視を開始
python3 controller/liveController.py

# 例: 配信待機枠の監視を開始
python3 controller/keepWatchController.py
```

## データベースの主なテーブル

このツール群で使用される主要なデータベーステーブルは以下の通りです。

| テーブル名                  | 目的                                                               |
| --------------------------- | ------------------------------------------------------------------ |
| `youtube_videos`            | 全ての動画情報のマスターデータ。再生数や高評価数なども含む。       |
| `keep_watchs`               | 配信開始前の動画（待機枠）を一時的に管理する。                     |
| `now_live_keep_watchs`      | 現在配信中の動画を一時的に管理する。                               |
| `holo_profiles`             | VTuberのプロフィール情報（チャンネルID、Twitterアカウント等）を管理。 |
| `arts`                      | Twitterのファンアート情報（ツイートID、いいね数等）を管理。          |
| `event_schedules`           | イベントスケジュール情報を管理する。                               |


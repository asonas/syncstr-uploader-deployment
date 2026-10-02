# Syncstr Uploader deployment

NAS上のSyncstrアップローダーをCoolifyのDocker Composeアプリとして運用する設定です。
デプロイ用リポジトリ名は `asonas/syncstr-uploader-deployment`、公開URLは `https://syncstr-uploader.jkte.ch` を使用します。

Rustサーバーの実装とDockerfileは `asonas/syncstr` の `server/` にあります。
このリポジトリにはComposeと運用手順を置き、DockerのGit build contextから指定コミットの実装を取得します。
実装のコピーやトークン、音源をこのリポジトリに保存しません。

## 保存先

| NAS上のパス | コンテナ内 | 用途 |
| --- | --- | --- |
| `/mnt/data/syncstr/music` | `/storage/music` | 検証後の音源。書き込み可能 |
| `/mnt/data/syncstr/upload-staging` | `/storage/upload-staging` | 受信・形式検証中のファイル |
| `/mnt/data/syncstr/secrets/upload-token` | `/storage/secrets/upload-token` | 専用トークン。読み取り専用 |

UID/GIDは `1000:1000` です。フォルダとトークンは事前に用意します。
存在しないホストパスは自動作成しません。トークンのマウント元はディレクトリではなく通常ファイルです。
musicとupload-stagingは同じファイルシステムに置き、両方へ書き込みを許可してください。
hard linkによる確定を可能にするため、共通の親 `/mnt/data/syncstr` を1つのbind mountにします。
個別のbind mountでは同じファイルシステム上でもCross-device linkになります。
`secrets` は読み取り専用のマウントを重ね、親マウント経由でも書き換えられないようにします。
Navidromeのmusicマウントは読み取り専用を維持します。

トークンは32バイト以上の乱数をhex化した値を推奨します。Navidromeのパスワードとは独立しています。
トークンファイルはUID 1000だけが読み取れる0600、一時保存先とsecretsディレクトリは0700を推奨します。
トークンをログやリポジトリへ出さないでください。

## Coolifyから配備

1. Syncstrの `server/` を含む実装コミットをGitHubへpushする。
2. このデプロイ用リポジトリをGitHubへpushする。
3. CoolifyでGitリポジトリからApplicationを追加し、配備先にNASを選ぶ。
4. Build PackをDocker Compose、Base Directoryを `/`、Docker Compose Locationを `/docker-compose.yml` にする。
5. 必要に応じて `SYNCSTR_SOURCE_REF` に手順1の40文字のコミットSHAを設定する。未指定時はComposeに固定された実装コミットを使用する。可変のブランチ名は使用しない。
6. CoolifyのDomainsは空にしたままDeployする。BuildKitが公開Syncstrリポジトリの指定コミットからビルドする。
7. 生成されたコンテナのユーザー、マウント元、書き込み権限、localhostだけへのポート公開を確認する。
8. NAS上のCloudflare Tunnelに `syncstr-uploader.jkte.ch` → `http://127.0.0.1:4545` の経路を追加する。
9. Syncstrのアップロード先に `https://syncstr-uploader.jkte.ch` と専用トークンを設定する。

`.env.example` はローカルで設定を確認する際のひな形です。Coolifyでは管理画面の環境変数に設定します。
Dockerfileはソースリポジトリにあるものを使用するため、このリポジトリには置きません。
独自コンテナ名・ネットワーク名は設定していません。Coolifyで管理する構成を別途 `docker compose up` で起動しないでください。

## アップロードの契約

1ファイルを1リクエストで送ります。分割アップロードと途中再開は行いません。
サーバーの受信上限は100,000,000バイトに設定し、Cloudflare経由の100 MB制限に合わせています。
Cloudflare側でさらに低い上限を設定している場合は、その制限が先に適用されます。
URLはアップロード専用で、Navidromeの配信URLとは別です。

専用トークン、サイズ、SHA-256、拡張子、ffprobeによる音声形式検証を通過したファイルだけがmusicへ確定します。
同名ファイルは上書きせず409で拒否します。動画や音声を含まないファイルも拒否します。
埋め込みのアルバムアートは許可します。形式検証はウイルス検査ではありません。

## 検証

ローカルでは `.env.example` を `.env` へコピーし、公開済みコミットSHAを入力したうえで実行します。

```sh
docker compose -f docker-compose.yml config --quiet
```

これは構文の検証です。ビルド成功、NAS上のパス・権限、Tunnel経路は配備時に別途確認します。
配備後は小さな検証用音源で、アップロード、原本とのSHA-256一致、Navidromeのスキャン、Syncstrでの再生を確認します。
認証なしのリクエストの401、偽装した音楽ファイルの422、同名再送の409も確認してください。

## 更新

`SYNCSTR_SOURCE_REF` を新しい公開済みコミットSHAへ変更して再配備します。
デプロイ設定と実装のバージョンは独立しています。Syncstrへのpushだけでは自動的にサーバーを更新しません。
トークンの変更はNAS上のファイルを差し替えて再起動し、各端末の設定を更新します。
プロセスの強制終了時にstagingへファイルが残ることがあるため、停止中に確認してください。

## 参照

- [Syncstr](https://github.com/asonas/syncstr)
- [Navidrome deployment](https://github.com/asonas/navidrome-deployment)
- [Coolify Docker Compose](https://coolify.io/docs/applications/builds/docker-compose)
- [Docker Git build context](https://docs.docker.com/build/concepts/context/#git-repositories)
- [Cloudflareの413とアップロード上限](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/)

# kakeibo-local

家計簿アプリ **Kakeibo** をローカル環境で起動するための Docker Compose 一式。
本体リポジトリ（[root-admin-jp/kakeibo](https://github.com/root-admin-jp/kakeibo)）のリリースで GitHub Container Registry (GHCR) に発行されたイメージを使うので、ソースのビルドは不要。

構成: Web (Nuxt) + API (Spring Boot) + DB (MySQL 8) の 3 コンテナ。テーブルの作成・更新は API 起動時に Flyway が自動で行う。

## 前提
- Docker（Docker Desktop / Docker Engine + Compose v2）
- イメージは GHCR で公開されているため、取得にログインは不要

## 手順

### 1. 環境変数を用意する
```
cp .env.example .env
```
`.env` を開き、少なくとも次の 3 つを書き換える（`.env` は git 管理外）。

- `MYSQL_ROOT_PASSWORD`
- `DB_PASSWORD`
- `JWT_SECRET`（長いランダム文字列。変えると既存のログインは無効になる）

使うバージョンは `KAKEIBO_VERSION` で指定する（例: `v0.1.0`、未指定は `latest`）。

### 2. 起動する
```
docker compose pull
docker compose up -d
```

### 3. ブラウザで開く
http://localhost:3000

ログインはメールアドレス宛ての6桁コード（ワンタイムパスワード）で行う（v0.4.0 以降、パスワードなし）。
既定の設定ではコードはメール送信されず、API コンテナのログに出力されるので、`docker compose logs api` で確認して入力する。
新規登録も同じ手順で、初回は表示名も一緒に求められる。
実際にメールを送りたい場合は `.env` で `MAIL_MODE=smtp` と SMTP サーバーの設定を有効にする（`.env.example` のコメント参照）。
ログイン後に「パスキー」画面でパスキーを登録すると、次回からはコードなしでログインできる（localhost のまま使える）。

## 外出先から使う（Tailscale）
自宅 PC で起動した Kakeibo に、Tailscale 経由で外出先からアクセスする手順。
ルーターのポート開放は不要で、通信は Tailscale の暗号化トンネルを通り、自分の tailnet 内のデバイスからしか見えない。インターネットへ公開する Tailscale Funnel は使わないこと。

### 前提
- 自宅 PC と外出先のデバイス（スマホ等）の両方に Tailscale を入れ、同じアカウントでログインしておく
- PC の Tailscale 名（MagicDNS 名、例: `mypc.tail1234.ts.net`）を確認しておく。`tailscale status` か Tailscale の管理コンソールで見られる（Tailscale IP `100.x.x.x` でも代用できる）

### A. HTTP で使う（手軽な方法）
1. `.env` で、ブラウザから見た API の URL を PC の Tailscale 名に変える
   ```
   NUXT_PUBLIC_API_BASE=http://<PCのTailscale名>:8080
   ```
2. `docker compose up -d` で再起動する
3. 外出先デバイスのブラウザで `http://<PCのTailscale名>:3000` を開く
4. ログインはメール OTP を使う（この方法ではパスキーは使えない。理由は次項）

### B. HTTPS で使う（パスキーも使いたい場合）
パスキー（WebAuthn）は HTTPS が必須のため、HTTP のままでは登録・利用ができない。`tailscale serve` を使うと、tailnet 内限定のまま HTTPS 化できる（初回実行時に、管理コンソールで MagicDNS と HTTPS 証明書の有効化を求められることがある）。

PC で一度だけ実行する（`--bg` で再起動後も維持される）。
```
tailscale serve --bg 3000
tailscale serve --bg --https=8080 8080
```
これで Web は `https://<PCのTailscale名>`、API は `https://<PCのTailscale名>:8080` になる。

`.env` の次の 3 つを変えて、`docker compose up -d` で再起動する。
```
NUXT_PUBLIC_API_BASE=https://<PCのTailscale名>:8080
WEBAUTHN_RP_ID=<PCのTailscale名>
WEBAUTHN_ORIGINS=https://<PCのTailscale名>
```
外出先デバイスのブラウザでは `https://<PCのTailscale名>` を開く。

### 注意
- `WEBAUTHN_RP_ID` を変えると、それ以前に登録したパスキーは使えなくなる（登録し直しが必要）
- PC がスリープ／電源オフだとつながらない。常時使うなら PC のスリープを無効にしておく
- つながらない場合は、PC のファイアウォールで対象ポートの受信が許可されているか確認する

## よく使う操作
```
docker compose ps        # 状態確認
docker compose logs -f   # ログを見る
docker compose down      # 停止（データは残る）
docker compose pull && docker compose up -d   # イメージ更新
docker compose down -v   # データごと削除（DB の中身も消えるので注意）
```

## ポートを変えたい場合
`.env` の `WEB_PORT` / `API_PORT` を変更する。`API_PORT` を 8080 から変えた場合は、ブラウザから見た API の URL も合わせるため `NUXT_PUBLIC_API_BASE`（例: `http://localhost:9000`）も一緒に変えること。

## 注意
- このリポジトリには秘密情報を置かない。実際の値は `.env` のみに書く。
- 本体側の開発用 compose（kakeibo リポジトリの `infra/docker-compose.yml`、DB のみ）とは別物。

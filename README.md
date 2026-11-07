# kakeibo-local

家計簿アプリ **Kakeibo** をローカル環境で起動するための Docker Compose 一式。
本体リポジトリ（[root-admin-jp/kakeibo](https://github.com/root-admin-jp/kakeibo)）のリリースで GitHub Container Registry (GHCR) に発行されたイメージを使うので、ソースのビルドは不要。

構成: Web (Nuxt) + API (Spring Boot) + DB (MySQL 8) の 3 コンテナ。テーブルの作成・更新は API 起動時に Flyway が自動で行う。

## 前提
- Docker（Docker Desktop / Docker Engine + Compose v2）
- GHCR へのログイン（イメージが Private のため）

## 手順

### 1. GHCR にログインする
read:packages 権限のある Personal Access Token (classic) を使う。

```
docker login ghcr.io -u root-admin-jp
# パスワード欄に PAT を入力
```

### 2. 環境変数を用意する
```
cp .env.example .env
```
`.env` を開き、少なくとも次の 3 つを書き換える（`.env` は git 管理外）。

- `MYSQL_ROOT_PASSWORD`
- `DB_PASSWORD`
- `JWT_SECRET`（長いランダム文字列。変えると既存のログインは無効になる）

使うバージョンは `KAKEIBO_VERSION` で指定する（例: `v0.1.0`、未指定は `latest`）。

### 3. 起動する
```
docker compose pull
docker compose up -d
```

### 4. ブラウザで開く
http://localhost:3000

初回は「新規登録」からユーザーを作って使う。

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

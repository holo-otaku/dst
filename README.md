# DST 安裝步驟

## 前置作業

1. Ubuntu OS
2. 安裝 Docker，可參考官網安裝步驟 [Docker](https://docs.docker.com/engine/install/ubuntu/)

   _備註：安裝後記得將使用者加入 docker 群組，以避免每次執行 docker 指令都需要 sudo_ [參考](https://docs.docker.com/engine/install/linux-postinstall/)

3. Git 指令，通常會內建在 Ubuntu OS 中，若沒有安裝可執行以下指令安裝

   ```bash
   sudo apt-get update
   sudo apt-get install git
   ```

4. Github Token

   由於本專案 docker images 接存放在 Github Packages，因此需要一組 Github Token 來取得 images，請參考 [官方文件](https://docs.github.com/en/packages/guides/configuring-docker-for-use-with-github-packages#authenticating-to-github-packages) 產生 Token

## 安裝步驟

1. Clone DST 專案

   ```bash
   git clone https://github.com/holo-otaku/dst.git
   ```

2. 進入專案目錄

   ```bash
   cd dst
   ```

3. 環境初始化

   ```bash
   bash init.sh
   ```

4. 設定環境變數：複製 `.env.example` 為 `.env`，填入密碼與密鑰（`openssl rand -hex 24` 產生），
   `SQLALCHEMY_DATABASE_URI` 指向 `postgres` 服務

   ```bash
   cp .env.example .env
   ```

5. 建立資料表與預設帳號（admin / admin，登入後請立刻改密碼）

   ```bash
   docker compose up -d postgres
   docker compose run --rm backend flask --app app db upgrade
   docker compose run --rm backend python -c 'from app import app; from controller.permission import create_default_permissions; from controller.role import create_admin_role; from controller.user import create_admin_user; app.app_context().push(); create_default_permissions(); create_admin_role(); create_admin_user()'
   ```

6. 啟動專案

   ```bash
   docker compose up -d
   ```

## 升級

- 從 MySQL 升級到 PostgreSQL：見 [docs/upgrade-to-postgres.md](docs/upgrade-to-postgres.md)
- 之後有新的資料庫 migration 時：`docker compose run --rm backend flask --app app db upgrade`

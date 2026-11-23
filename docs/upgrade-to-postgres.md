# 從 MySQL 升級到 PostgreSQL

對應 dst-backend #95（PostgreSQL、`item.attributes` JSONB）與 dst-frontend #225。
資料由 dst-backend 的 `scripts/mysql_to_postgres.py` 搬移：它只接受 alembic 版本 `13fdcde5ad3f` 的 MySQL 和空的 PostgreSQL，
搬完會比對每張表的筆數，不一致就中止，MySQL 不會被修改。

以下指令都在主機的 `/home/user/dst` 執行。

## 事前準備（不用停機）

1. dst-backend #95、dst-frontend #225 合併並推到各自的 `release` branch，確認 GitHub Actions 已經建好新的 `:release` image。
2. 更新這個 repo：`git pull`（拿到含 `postgres` 服務的 `docker-compose.yml`）。
3. 在 `.env` 加上 PostgreSQL 設定，**先不要改** `SQLALCHEMY_DATABASE_URI`：

   ```bash
   cat >> .env <<EOF

   # PostgreSQL env
   POSTGRES_DB=dst
   POSTGRES_USER=dst
   POSTGRES_PASSWORD=$(openssl rand -hex 24)
   EOF
   ```

4. 把目前的 image 另外標記，回退時要用（一定要在 pull 之前做）：

   ```bash
   docker tag ghcr.io/holo-otaku/dst-backend:release dst-backend:pre-pg
   docker tag ghcr.io/holo-otaku/dst-frontend:release dst-frontend:pre-pg
   ```

5. 先把新 image 拉下來，縮短停機時間：`docker compose pull backend web postgres`

## 停機升級

1. 停後端，之後不會再有寫入：

   ```bash
   docker compose stop backend
   ```

2. 備份 MySQL 與圖片：

   ```bash
   docker compose exec -T database sh -c 'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --single-transaction --databases "$MYSQL_DATABASE"' \
     | gzip > backup/mysql-pre-pg-$(date +%Y%m%d-%H%M).sql.gz
   tar czf backup/images-pre-pg-$(date +%Y%m%d-%H%M).tgz images
   ls -lh backup/
   ```

3. 啟動 PostgreSQL，等到 `healthy`：

   ```bash
   docker compose up -d postgres
   docker compose ps postgres
   ```

4. 搬資料。用新的 backend image 執行，MySQL 連線沿用 `.env` 裡現在的 `SQLALCHEMY_DATABASE_URI`：

   ```bash
   docker compose run --rm --no-deps backend sh -c '
     MYSQL_URI="$SQLALCHEMY_DATABASE_URI" \
     PG_URI="postgresql+psycopg://$POSTGRES_USER:$POSTGRES_PASSWORD@postgres:5432/$POSTGRES_DB" \
     python -m scripts.mysql_to_postgres'
   ```

   最後一行要是「完成，alembic 版本 5b8e2d4f9a13」。2026-11-23 檢查線上資料時，預期會略過的孤兒資料：
   - `archive` 1 筆（指向已不存在的 item）
   - `activity_log` 78 筆（使用者已被刪除）

   失敗時不要繼續，MySQL 沒有被動到，看「回退」。若要重跑，先清空 PostgreSQL：
   `docker compose down postgres && docker volume rm dst_pg_data`。

5. 把 `.env` 的 `SQLALCHEMY_DATABASE_URI` 改成 PostgreSQL（舊的那行留著註解，回退用）：

   ```
   SQLALCHEMY_DATABASE_URI=postgresql+psycopg://dst:<POSTGRES_PASSWORD>@postgres:5432/dst
   ```

6. 啟動新版：

   ```bash
   docker compose up -d backend web
   docker compose logs --tail=50 backend
   curl -s http://localhost/api/health
   ```

## 驗證

在瀏覽器實際操作一輪：

- [ ] 登入（帳號不分大小寫）
- [ ] 每個系列都能搜尋，筆數與升級前相同；篩選、排序、分頁正常
- [ ] 產品圖片顯示正常（舊圖 `image_<item>_<field>.png` 與新上傳的圖）
- [ ] 編輯產品（含換圖）、新增、複製、封存、刪除/還原
- [ ] 匯出 Excel（含圖片）
- [ ] ERP 欄位有資料
- [ ] 沒有 `limit-field.read` 的帳號看不到、也改不到限制欄位

## 回退

只要還沒清理，MySQL 的資料與升級前完全相同。**回退後，升級後在 PostgreSQL 上做的修改都會不見。**

```bash
docker compose stop backend web
# .env：SQLALCHEMY_DATABASE_URI 改回原本的 mysql+pymysql://... 那行
docker tag dst-backend:pre-pg ghcr.io/holo-otaku/dst-backend:release
docker tag dst-frontend:pre-pg ghcr.io/holo-otaku/dst-frontend:release
docker compose up -d backend web
```

## 之後

- 新版 backend 啟動時**不會**自動跑 migration。之後有新的 migration 時：
  `docker compose run --rm backend flask --app app db upgrade`
- 備份 PostgreSQL：

  ```bash
  docker compose exec -T postgres sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' | gzip > backup/pg-$(date +%Y%m%d-%H%M).sql.gz
  ```

- 直接查資料（取代 phpMyAdmin）：`docker compose exec postgres sh -c 'psql -U "$POSTGRES_USER" "$POSTGRES_DB"'`

## 清理（穩定運作一段時間後）

1. 再做一次 MySQL 備份並留存。
2. 從 `docker-compose.yml` 移除 `database`、`phpmyadmin` 服務與 `mysql_data` volume；從 `.env` 移除 `MYSQL_*`。
3. `docker compose up -d --remove-orphans`，確認正常後 `docker volume rm dst_mysql_data`。
4. `docker image rm dst-backend:pre-pg dst-frontend:pre-pg`

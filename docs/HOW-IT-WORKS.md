# llm-wiki-infra 實際怎麼運作（含真實紀錄）

> 這是**共用基礎設施**：一套 **MinIO**（物件儲存）+ 一套 **Postgres/pgvector**（索引資料庫），
> 跑在共用 docker 網路 `llm-wiki-net` 上。先啟動它，其他服務接上去共用同一份資料。
> 它本身沒有商業邏輯 —— 就是「檔案桶 + 資料庫」。下面用真實指令看裡面到底有什麼。

> **名詞**：**MinIO** 相容 S3 的本地物件儲存（檔案桶）；**pgvector** Postgres 的向量擴充
> （存 embedding、算相似度）；**pg_trgm** Postgres 的 trigram 擴充（讓 `ILIKE '%詞%'` 走索引）；
> **真相來源 vs 索引** MinIO 的檔是正本，Postgres 是可重建的副本。

## 啟動 / 關閉
```bash
docker compose up -d        # 建網路 llm-wiki-net + minio + pg
docker compose down -v      # 停 + 清空所有資料（minio + pg volume）
```

## 它開了什麼
| | 主機端 | 網路內主機名 |
|---|---|---|
| MinIO API | localhost:9000 | `minio:9000` |
| MinIO 主控台 | localhost:9001（minioadmin/minioadmin） | — |
| Postgres + pgvector | localhost:5432（wiki/wikipass，db `wiki`） | `pg:5432` |

各服務用 `minio` / `pg` 連，env 預設已指好，**不用改設定**。

---

## 看裡面有什麼（真實紀錄）

**MinIO 桶（wiki-processor 寫進來的）：**
```
apps/payments-api.json   apps/oracle-kb.json   apps/flashback-api.json   # 每 app 一個檔（正本）
snapshots/payments-api.json …                                           # 原始輸入快照
audit/2026-06-25T22:26:31…json …                                        # 每次推送一筆紀錄
```
**這代表什麼：** wiki 的「正本」就是這些檔。每個 app 一個 `apps/<app>.json`。

**Postgres 表：**
```
 public | api_entries       | table   ← API 條目 + 向量（搜尋索引）
 public | knowledge_entries | table   ← 知識條目 + 向量
 public | app_sync          | table   ← 每 app 最後同步時間（防舊覆新）
 public | index_state       | table   ← 索引狀態
```
計數（這次跑）：`api_entries=4`、`knowledge_entries=1`、`app_sync=3`。

**擴充已裝：**
```
vector
pg_trgm
```
**這代表什麼：** `vector` 讓 DB 能存/比向量（語意搜尋）；`pg_trgm` 讓關鍵字 `ILIKE` 走索引。
表的結構（DDL）由 wiki-processor 在執行時自動建（`pg_store.py`），不在這裡寫死。

---

## 自己看
```bash
docker exec llm-wiki-pg psql -U wiki -d wiki -c "\dt"                    # 列表
docker exec llm-wiki-pg psql -U wiki -d wiki -c "SELECT count(*) FROM api_entries;"
# MinIO：開 http://localhost:9001（minioadmin/minioadmin）看 wiki-data 桶
```
**壞了會怎樣：** PG 掛了，查詢端自動退回掃 MinIO；整個索引壞了，對 wiki-processor 打
`POST /admin/reindex` 從 MinIO 正本重建。MinIO 是正本，別的都能重建。

更多：[README.md](../README.md)、[db/init/01-extension.sql](../db/init/01-extension.sql)。
</content>

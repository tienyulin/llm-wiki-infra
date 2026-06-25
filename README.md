# llm-wiki-infra

> 👉 想看這個服務**實際怎麼運作（含真實紀錄）**：[docs/HOW-IT-WORKS.md](docs/HOW-IT-WORKS.md)。

[LLM Wiki 平台](https://github.com/tienyulin/llm-wiki-mcp)的**共用基礎設施（shared
infrastructure）**：一套 **MinIO**（物件儲存，存 wiki 檔案）+ 一套 **Postgres
（含 pgvector 向量擴充）**（搜尋索引），跑在共用的 docker 網路 `llm-wiki-net` 上。

先啟動它一次，其他服務都接上來 —— 這樣可以同時跑全部服務、共用同一份資料，
infra 也不會撞連接埠（只有各服務自己的 8001/8002/8003 對外）。

> **名詞**
> - **MinIO**：相容 S3 的物件儲存，本地版的「雲端檔案桶」。
> - **pgvector**：Postgres 的向量擴充，讓資料庫能存 embedding 並算相似度。
> - **物件儲存 vs 資料庫**：wiki 檔（真相來源）放 MinIO；搜尋索引放 Postgres。

## 啟動 / 關閉
```bash
docker compose up -d        # 建立網路 llm-wiki-net + minio + pg
docker compose ps
docker compose down         # 停止（保留資料）
docker compose down -v      # 停止並「清空」所有共用資料（minio + pg volume）
```

## 它開了什麼
| | 主機端（host） | 網路內主機名 |
|---|---|---|
| MinIO API | localhost:9000 | `minio:9000` |
| MinIO 主控台 | localhost:9001（minioadmin/minioadmin） | — |
| Postgres + pgvector | localhost:5432（wiki/wikipass，db `wiki`） | `pg:5432` |

各服務用 `minio` / `pg` 這兩個名字連 —— 它們 compose 的環境變數預設就是
`MINIO_ENDPOINT=minio:9000` 和 `PG_DSN=postgresql://wiki:wikipass@pg:5432/wiki`，
所以**不用改任何服務設定**。

`db/init/01-extension.sql` 在第一次啟動時開啟 `vector` + `pg_trgm` 擴充；
資料表結構（DDL）由 wiki-processor 在執行時自動建立。

## 服務怎麼接上來
服務的 compose 這樣加入網路：
```yaml
services:
  wiki-processor:
    networks: [llm-wiki-net]
networks:
  llm-wiki-net:
    external: true
```
所以**要先起這個 infra**，再起服務（或開服務的 dev container）。

## 一次全部起來
從平台 repo 執行 `scripts/dev-up.sh`，會把這個 infra + 所有服務一次帶起來。
見 [llm-wiki-mcp](https://github.com/tienyulin/llm-wiki-mcp)。
</content>

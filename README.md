# Patent API Docker 部署指南

## 概述

本指南說明如何將多個 API 服務整合到單一 FastAPI 應用程式中，並透過 Docker 容器化部署。

## 架構設計

### API 整合策略

原本需要 4 個獨立的 port 來運行不同的 API 服務，現在透過路由前綴整合到單一應用程式：

```python
@app.get("/")
def root():
    return {
        "message": "專利處理系統 API",
        "services": {
            "scraper": "http://localhost:8005/scraper/health",
            "summary": "http://localhost:8005/summary/health",
            "filter": "http://localhost:8005/filter/health",
            "csv": "http://localhost:8005/csv/health"
        }
    }
```

**優點：**
- 只需使用 1 個 port (8005)
- 避免函數名稱衝突（例如每個服務都有 `health` 端點）
- `scraper`、`summary`、`filter`、`csv` 是新定義的路由前綴，與原 API 獨立

## 前置準備

### 必要檔案

在專案根目錄準備以下檔案：

1. **requirements.txt** - Python 套件依賴
2. **Dockerfile** - Docker 映像檔建置設定
3. **.dockerignore** - 排除不需要的檔案
4. **docker-compose.yml** - （可選）容器編排設定

### 建立本地資料夾

在本地建立對應的資料夾並設定權限：

```powershell
# 建立本地資料夾
New-Item -Path "C:\Users\TZUYI.KUO\Downloads\n8nLocal\patent_data" -ItemType Directory -Force
New-Item -Path "C:\Users\TZUYI.KUO\Downloads\n8nLocal\patent_pdfs" -ItemType Directory -Force

# 給予完整權限
icacls C:\Users\TZUYI.KUO\Downloads\n8nLocal\patent_data /grant Everyone:F /T
icacls C:\Users\TZUYI.KUO\Downloads\n8nLocal\patent_pdfs /grant Everyone:F /T
```

**為什麼需要預先建立？**
- API 產生的檔案會在 Docker 容器內部
- 透過 Volume 映射可將檔案存到本地
- 預先設定權限避免執行時因權限不足無法讀寫
[但如果用compose去做可以避免這個問題，因為compose時會把不存在的路徑預先建立好]

## Docker 建置與部署

### Step 1: 建置 Docker Image

在專案根目錄開啟 CMD 或 PowerShell，執行：

```bash
docker build -t tzuyikuo/patent-api:latest .
```

**參數說明：**
- `-t tzuyikuo/patent-api:latest` - 自定義的映像檔名稱和標籤
- `.` - 使用當前目錄的 Dockerfile

### Step 2: 執行容器

```powershell
docker run -d `
  --name patent-api `
  -p 8005:8005 `
  -v C:\Users\TZUYI.KUO\Downloads\n8nLocal\patent_data:/app/patent_data `
  -v C:\Users\TZUYI.KUO\Downloads\n8nLocal\patent_pdfs:/app/patent_pdfs `
  --shm-size=2g `
  --cap-add=SYS_ADMIN `
  tzuyikuo/patent-api:latest
```

**參數說明：**
- `-d` - 背景執行
- `--name patent-api` - 容器名稱
- `-p 8005:8005` - Port 映射（本地:容器）
- `-v` - Volume 映射（本地路徑:容器路徑）
- `--shm-size=2g` - 設定共享記憶體大小
- `--cap-add=SYS_ADMIN` - 添加系統管理權限
- `這個做法就需要預先建立好路徑避免無法讀寫

Ex.
docker run -d `
  --name patent-api `
  -p 8005:8005 `
  -v C:\Users\TZUYI.KUO\Downloads\n8nLocal\patent_data:/app/patent_data ` [映射到本地 左邊是本地:右邊是docker]
  -v C:\Users\TZUYI.KUO\Downloads\n8nLocal\patent_pdfs:/app/patent_pdfs ` [映射到本地 左邊是本地:右邊是docker]
  --shm-size=2g `
  --cap-add=SYS_ADMIN `
  patent-api:latest
  
## 版本更新與除錯

### 修改 requirements.txt 後重新建置

如果套件版本有問題需要修改：

1. **不要刪除原有的 Image**
2. 修改 `requirements.txt`
3. 重新執行建置命令：

```bash
docker build -t tzuyikuo/patent-api:latest .
```

**Docker Layer Cache 機制：**
- 只會重新建置修改過的層（layer）
- 未修改的層會使用快取，加快建置速度
- Push 到 Registry 時也會利用層級差異，提升效率

## 環境變數設定

### 建立 .env 檔案

在專案根目錄建立 `.env` 檔案：
```bash
OPENAI_API_KEY=your_api_key_here
```

**重要：**
- ❌ 不要將 `.env` 提交到版本控制(千萬不要包成image，只要push到dockerhub會人人可以使用)
- ✅ 使用 `.env.example` 作為範本
- ✅ 確保 `.gitignore` 包含 `.env`

### Docker Compose 中使用環境變數

在 `docker-compose.yml` 中：
```yaml
patent-api:
  env_file:
    - .env
```

這樣容器就能讀取環境變數。

## 使用 Docker Compose 部署（推薦）

### 優點
- 一次管理多個服務
- 自動建立網路連接
- 環境變數統一管理

### docker-compose.yml 範例
```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    ports:
      - "5678:5678"
    volumes:
      - ./n8n_data:/home/node/.n8n
    networks:
      - patent-network
    restart: unless-stopped

  patent-api:
    image: tzuyikuo/patent-api:latest
    container_name: patent-api
    ports:
      - "8005:8005"
    env_file:
      - .env
    volumes:
      - ./patent_data:/app/patent_data
      - ./patent_pdfs:/app/patent_pdfs
    networks:
      - patent-network
    restart: unless-stopped
    shm_size: '2gb'
    cap_add:
      - SYS_ADMIN

networks:
  patent-network:
    driver: bridge
```

### 啟動服務
```bash
# 啟動所有服務
docker-compose up -d

# 查看服務狀態
docker-compose ps

# 查看日誌
docker-compose logs -f

# 停止所有服務
docker-compose down
```

### 服務訪問

- n8n 工作流介面：http://localhost:5678
- Patent API：http://localhost:8005

## 常用 Docker 指令

```bash
# 查看運行中的容器
docker ps

# 查看所有容器（包含停止的）
docker ps -a

# 查看容器日誌
docker logs patent-api

# 即時查看日誌
docker logs -f patent-api

# 進入容器內部
docker exec -it patent-api /bin/bash

# 停止容器
docker stop patent-api

# 啟動容器
docker start patent-api

# 重啟容器
docker restart patent-api

# 刪除容器
docker rm patent-api

# 強制刪除運行中的容器
docker rm -f patent-api

# 查看 Images
docker images

# 刪除 Image
docker rmi tzuyikuo/patent-api:latest

# 查看容器資源使用情況
docker stats patent-api
```

## 驗證部署

訪問以下 URL 確認服務正常運行：

- 主頁面：`http://localhost:8005/`
- Scraper 健康檢查：`http://localhost:8005/scraper/health`
- Summary 健康檢查：`http://localhost:8005/summary/health`
- Filter 健康檢查：`http://localhost:8005/filter/health`
- CSV 健康檢查：`http://localhost:8005/csv/health`

## 疑難排解

### 權限問題
如果遇到檔案讀寫權限錯誤，確認：
1. 本地資料夾已建立
2. 權限已正確設定（Everyone:F）
3. Volume 映射路徑正確

### Port 衝突
如果 8005 port 已被佔用：
```bash
# 修改映射 port（例如改為 8006）
docker run -d -p 8006:8005 ...
```

### 容器無法啟動
檢查日誌找出原因：
```bash
docker logs patent-api
```

### 套件版本衝突
如果遇到套件相依性問題：
1. 查看建置日誌找出衝突的套件
2. 修改 `requirements.txt` 中的版本號
3. 重新建置（利用 cache 加速）

## Volume 映射說明

映射格式：`本地路徑:容器路徑`

```
C:\Users\TZUYI.KUO\Downloads\n8nLocal\patent_data:/app/patent_data
     ↑ 本地路徑                                    ↑ 容器內路徑
```

- 左邊：Windows 本地檔案系統路徑
- 右邊：Docker 容器內部路徑
- API 在容器內寫入 `/app/patent_data` 時，檔案會同步到本地對應路徑

## 注意事項

- 路徑映射使用絕對路徑
- Windows 路徑使用反斜線（`\`）或正斜線（`/`）
- 確保 Docker Desktop 正在運行
- 建議定期備份本地映射的資料夾
- 容器刪除後 Image 仍會保留，可重複使用

## 推送到 Docker Hub（可選）

```bash
# 登入 Docker Hub
docker login

# 推送 Image
docker push tzuyikuo/patent-api:latest

# 從 Docker Hub 拉取
docker pull tzuyikuo/patent-api:latest
```

## 專案結構建議

```
project-root/
├── MainFunction.py          # 主要 API 整合檔案
├── scraper_api.py          # Scraper 服務
├── summary_api.py          # Summary 服務
├── filter_api.py           # Filter 服務
├── csv_api.py              # CSV 服務
├── requirements.txt        # Python 套件依賴
├── Dockerfile              # Docker 建置設定
├── .dockerignore           # Docker 忽略檔案
├── docker-compose.yml      # 容器編排（可選）
└── README.md               # 本文件
```

## 支援與回饋

如有問題或建議，請聯繫開發團隊。

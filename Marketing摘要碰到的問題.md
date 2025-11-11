# Docker Compose 與 Docker Hub 部署指南

## 📋 目錄
- [專案架構](#專案架構)
- [Docker 容器間通訊](#docker-容器間通訊)
- [解決 Image 命名問題](#解決-image-命名問題)
- [推送到 Docker Hub](#推送到-docker-hub)
- [在其他電腦使用](#在其他電腦使用)
- [常見問題與解答](#常見問題與解答)

---

## 專案架構

### 檔案結構
```
marketing/
├── docker-compose.yml
├── Dockerfile
├── supervisord.conf
├── Marketing_scheduler.py
└── MailScrape_api.py
```

### Docker Compose 配置

**n8n 服務** (`docker-compose.yml`)
```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://localhost:5678/
      - N8N_EDITOR_BASE_URL=http://localhost:5678
      - TZ=Asia/Taipei
    volumes:
      - clear_n8n:/home/node/.n8n
    networks:
      - app_network
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:5678/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  clear_n8n:

networks:
  app_network:
    external: true
```

**Marketing 服務** (`docker-compose.yml`)
```yaml
services:
  marketing:
    build: .
    image: chocoilove27/marketing:v1.0
    container_name: marketing
    restart: unless-stopped
    ports:
      - "8005:8005"
    environment:
      - N8N_WEBHOOK_URL=http://n8n:5678/webhook/Marketing_scrape
      - TZ=Asia/Taipei
    networks:
      - app_network

networks:
  app_network:
    external: true
```

---

## Docker 容器間通訊

### 問題：localhost 在容器中的意義

#### ❌ 錯誤的寫法
```python
N8N_WEBHOOK_URL = "http://localhost:5678/webhook/Marketing_scrape_test"
```
- 在容器內，`localhost` 指的是**容器本身**，不是宿主機
- Marketing 容器無法通過 localhost 訪問 n8n 容器
- 結果：404 錯誤

#### ✅ 正確的寫法
```python
N8N_WEBHOOK_URL = os.getenv(
    "N8N_WEBHOOK_URL",  # 環境變數名稱
    "http://n8n:5678/webhook/Marketing_scrape_test"  # 預設值
)
```
- 使用**容器名稱** (`n8n`) 作為主機名
- Docker 網路會自動解析容器名稱
- 環境變數優先，沒有設定時使用預設值

### 容器通訊對照表

| 位置 | localhost 的意義 | 正確用法 |
|------|------------------|----------|
| 宿主機瀏覽器 | 你的電腦 | `http://localhost:5678` |
| marketing 容器內 | marketing 容器本身 ❌ | `http://n8n:5678` ✅ |
| n8n 容器內 | n8n 容器本身 | - |

### os.getenv() 用法說明

```python
os.getenv(key, default=None)
```

**參數**：
- `key`: 環境變數的名稱（字串）
- `default`: 找不到環境變數時的預設值

**範例**：
```python
# docker-compose.yml 中設定: N8N_WEBHOOK_URL=http://n8n:5678/webhook/prod
url = os.getenv("N8N_WEBHOOK_URL", "http://fallback")
# → 返回 "http://n8n:5678/webhook/prod"

# 如果 docker-compose.yml 沒有設定 N8N_WEBHOOK_URL
url = os.getenv("N8N_WEBHOOK_URL", "http://fallback")
# → 返回 "http://fallback"
```

---

## 解決 Image 命名問題

### 問題：為什麼 Image 叫 `marketing-marketing`？

Docker Compose 的自動命名規則：
```
{專案名稱}-{服務名稱}
```

**原因分析**：
- 專案名稱：來自**資料夾名稱** (`marketing`)
- 服務名稱：`services` 下的名稱 (`marketing`)
- 結果：`marketing-marketing`

### Docker Desktop 顯示結構

```
📦 marketing (專案層 - 資料夾名)
  └─ 📦 marketing (容器層)
      ├─ IMAGE: marketing-marketing
      ├─ PORTS: 8005:8005
```

### ✅ 解決方案：明確指定 Image 名稱

```yaml
services:
  marketing:
    build: .
    image: marketing:v1.0  # 👈 加這行，明確指定 image 名稱
    container_name: marketing
```

**結果**：
- Image 名稱變成：`marketing:v1.0` ✅
- 避免自動命名造成的混淆

### 修正步驟

```bash
# 1. 停止並刪除容器
docker-compose down

# 2. 刪除舊的 image
docker rmi marketing-marketing

# 3. 確認 docker-compose.yml 有 image 這行
# image: marketing:v1.0

# 4. 重新 Build
docker-compose build

# 5. 啟動
docker-compose up -d

# 6. 檢查結果
docker images | grep marketing
# 應該看到: marketing  v1.0  xxxxx
```

---

## 推送到 Docker Hub

### 完整流程

#### 1. 登入 Docker Hub
```bash
docker login
```
輸入：
- Username（用戶名）
- Password（密碼或 Access Token）

看到 `Login Succeeded` 即成功。

#### 2. 修改 docker-compose.yml

```yaml
services:
  marketing:
    build: .
    image: your-username/marketing:v1.0  # 👈 加上 Docker Hub 帳號
    container_name: marketing
    restart: unless-stopped
    ports:
      - "8005:8005"
    environment:
      - N8N_WEBHOOK_URL=http://n8n:5678/webhook/Marketing_scrape
      - TZ=Asia/Taipei
    networks:
      - app_network
```

**範例**（實際使用）：
```yaml
image: chocoilove27/marketing:v1.0
```

#### 3. 重新 Build

```bash
docker-compose build
```

**注意**：
- 不需要先 `docker-compose down`
- Build 只建立 image，不影響運行中的容器
- 新舊 image 會並存

#### 4. 推送到 Docker Hub

```bash
# 方法 1：使用 docker-compose
docker-compose push

# 方法 2：使用 docker 指令
docker push chocoilove27/marketing:v1.0
```

#### 5. 驗證

打開瀏覽器查看：
```
https://hub.docker.com/r/chocoilove27/marketing
```

你會看到：
- Repository 名稱：`marketing`
- Tags：`v1.0`
- 大小：約 1.65 GB
- Pull 指令：`docker pull chocoilove27/marketing:v1.0`

### 版本號管理

#### 常見版本號格式

```yaml
# 語義化版本（推薦）
image: marketing:v1.0.0
image: marketing:v1.2.3

# 簡單版本
image: marketing:v1
image: marketing:v2

# 日期版本
image: marketing:2025-11-11
image: marketing:20251111

# 最新版
image: marketing:latest

# 環境標籤
image: marketing:dev
image: marketing:prod
image: marketing:staging
```

#### 推送多個標籤

```bash
# Build 主要版本
docker-compose build

# Tag 多個版本
docker tag chocoilove27/marketing:v1.0 chocoilove27/marketing:latest
docker tag chocoilove27/marketing:v1.0 chocoilove27/marketing:v1

# 推送所有標籤
docker push chocoilove27/marketing:v1.0
docker push chocoilove27/marketing:latest
docker push chocoilove27/marketing:v1
```

---

## 在其他電腦使用

### 部署用 docker-compose.yml

```yaml
services:
  marketing:
    image: chocoilove27/marketing:v1.0  # 👈 不需要 build: .
    container_name: marketing
    restart: unless-stopped
    ports:
      - "8005:8005"
    environment:
      - N8N_WEBHOOK_URL=http://n8n:5678/webhook/Marketing_scrape
      - TZ=Asia/Taipei
    networks:
      - app_network

networks:
  app_network:
    external: true
```

### 部署步驟

```bash
# 1. 建立資料夾
mkdir marketing
cd marketing

# 2. 建立 docker-compose.yml
nano docker-compose.yml
# 貼上上面的配置內容

# 3. 確保網路存在
docker network create app_network

# 4. 拉取 image 並啟動
docker-compose up -d

# 5. 查看 logs
docker exec marketing tail -f /var/log/supervisor/scheduler.out.log
```

### 需要的檔案對照表

| 項目 | 開發環境（你的電腦） | 部署環境（其他電腦） |
|------|---------------------|---------------------|
| `docker-compose.yml` | ✅ 需要（含 build） | ✅ 需要（不含 build） |
| `Dockerfile` | ✅ 需要 | ❌ 不需要 |
| `supervisord.conf` | ✅ 需要 | ❌ 不需要（已在 image 中） |
| `Python 檔案` | ✅ 需要 | ❌ 不需要（已在 image 中） |
| `requirements.txt` | ✅ 需要 | ❌ 不需要（已在 image 中） |

### 更新版本

當推送了新版本（例如 v1.1）：

```bash
# 在部署環境
docker-compose pull    # 拉取最新版本
docker-compose down    # 停止舊版本
docker-compose up -d   # 啟動新版本
```

---

## 常見問題與解答

### Q1: 為什麼會 404？

**問題**：Python 程式使用 `http://localhost:5678` 觸發 webhook 失敗

**原因**：在容器內，`localhost` 指的是容器本身，不是宿主機

**解決**：使用容器名稱
```python
N8N_WEBHOOK_URL = "http://n8n:5678/webhook/Marketing_scrape"
```

### Q2: Image 名稱為什麼是 marketing-marketing？

**原因**：Docker Compose 自動命名規則：`{專案名}-{服務名}`

**解決**：在 docker-compose.yml 中明確指定 image 名稱
```yaml
image: marketing:v1.0
```

### Q3: 在容器內修改的程式碼會保存嗎？

**答案**：不會！

- 在容器內修改的檔案只存在於該容器中
- 重新 build 會從本地檔案建立新 image
- **正確做法**：在本地修改程式碼，然後重新 build

### Q4: supervisord.conf 需要複製到其他電腦嗎？

**答案**：不需要！

- 所有檔案（包括 supervisord.conf）都已經打包在 Docker image 中
- 其他電腦只需要 docker-compose.yml

### Q5: 如何查看容器內的 logs？

因為使用了 supervisord，logs 在容器內的特定路徑：

```bash
# 方法 1：進入容器查看
docker exec -it marketing bash
tail -f /var/log/supervisor/scheduler.out.log

# 方法 2：從外面直接查看
docker exec marketing tail -f /var/log/supervisor/scheduler.out.log

# 查看錯誤 log
docker exec marketing tail -f /var/log/supervisor/scheduler.err.log

# 查看 supervisord 狀態
docker exec marketing supervisorctl status
```

### Q6: 如何在開發和正式環境切換？

**方法 1**：使用環境變數
```yaml
environment:
  - N8N_WEBHOOK_URL=${WEBHOOK_URL}
```

建立 `.env` 檔案：
```bash
# .env.dev
WEBHOOK_URL=http://n8n:5678/webhook/Marketing_scrape_test

# .env.prod
WEBHOOK_URL=http://n8n:5678/webhook/Marketing_scrape
```

啟動時指定：
```bash
docker-compose --env-file .env.dev up -d
```

**方法 2**：使用不同的 compose 檔案
```bash
# docker-compose.dev.yml
# docker-compose.prod.yml

# 啟動
docker-compose -f docker-compose.dev.yml up -d
docker-compose -f docker-compose.prod.yml up -d
```

### Q7: 推送失敗怎麼辦？

**錯誤**：`denied: requested access to the resource is denied`

**原因**：image 名稱沒有包含你的 Docker Hub 帳號

**解決**：
```yaml
# 錯誤
image: marketing:v1.0

# 正確
image: your-username/marketing:v1.0
```

### Q8: 需要使用 --no-cache 嗎？

**不加（推薦）**：
```bash
docker-compose build
```
- 快速，利用快取
- 適合日常開發

**加上 --no-cache**：
```bash
docker-compose build --no-cache
```
- 完全重新建置
- 適合：
  - 懷疑快取有問題
  - Dockerfile 改了但沒反應
  - 建立最終正式版本

---

## 完整工作流程總結

### 開發階段

1. **編寫程式碼**
   ```bash
   # 在本地編輯檔案
   nano Marketing_scheduler.py
   ```

2. **本地測試**
   ```yaml
   # docker-compose.yml
   image: marketing:v1.0
   ```
   ```bash
   docker-compose build
   docker-compose up -d
   docker exec marketing tail -f /var/log/supervisor/scheduler.out.log
   ```

3. **確認無誤**

### 發布階段

1. **準備推送**
   ```yaml
   # docker-compose.yml
   image: chocoilove27/marketing:v1.0
   ```

2. **Build & Push**
   ```bash
   docker-compose build
   docker login
   docker-compose push
   ```

3. **驗證**
   - 前往 Docker Hub 確認 image 已上傳
   - `https://hub.docker.com/r/chocoilove27/marketing`

### 部署階段

1. **在目標機器上**
   ```yaml
   # docker-compose.yml (不含 build)
   services:
     marketing:
       image: chocoilove27/marketing:v1.0
       # ... 其他配置
   ```

2. **部署**
   ```bash
   docker-compose up -d
   ```

3. **監控**
   ```bash
   docker exec marketing supervisorctl status
   docker exec marketing tail -f /var/log/supervisor/scheduler.out.log
   ```

---

## 參考資料

### Docker 指令速查

```bash
# 容器管理
docker-compose up -d          # 啟動服務
docker-compose down           # 停止並刪除容器
docker-compose ps             # 查看運行中的容器
docker-compose logs -f        # 查看 logs
docker-compose restart        # 重啟服務

# Image 管理
docker images                 # 列出所有 images
docker rmi <image-name>       # 刪除 image
docker-compose build          # 建立 image
docker-compose push           # 推送 image

# 容器操作
docker exec -it <container> bash    # 進入容器
docker cp <container>:/path ./      # 複製檔案

# 網路管理
docker network ls                   # 列出網路
docker network create <name>        # 建立網路
docker network inspect <name>       # 查看網路詳情
```

### 有用的連結

- Docker Hub: https://hub.docker.com
- Docker Compose 文檔: https://docs.docker.com/compose/
- Supervisor 文檔: http://supervisord.org/

---

**文件建立日期**: 2025-11-11  
**專案**: Marketing Scheduler  
**Docker Hub**: chocoilove27/marketing  
**版本**: v1.0

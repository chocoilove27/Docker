# Docker Compose + n8n + Scheduler 完整指南

## 📋 目錄

1. [概述](#概述)
2. [架構說明](#架構說明)
3. [方法比較](#方法比較)
4. [實作方式](#實作方式)
   - [方法 1: Cron + 觸發式程式](#方法-1-cron--觸發式程式推薦)
   - [方法 2: Supervisor + 長期運行程式](#方法-2-supervisor--長期運行程式)
5. [重要概念](#重要概念)
6. [測試與驗證](#測試與驗證)
7. [故障排除](#故障排除)
8. [總結](#總結)

---

## 概述

本指南說明如何使用 Docker Compose 建立一個自動化系統，包含：
- **n8n**：自動化工作流程平台
- **Scheduler**：定時觸發 n8n webhook 的排程器
- **容錯機制**：確保服務穩定運行

---

## 架構說明

### 整體架構圖

```
外部世界（你的瀏覽器）
    ↓ http://localhost:5678
    ↓
┌─────────────────────────────────────┐
│   Docker 主機 (你的電腦)             │
│                                     │
│  ┌───────────────────────────────┐ │
│  │   app_network (虛擬網路)       │ │
│  │                                │ │
│  │  ┌──────────────┐              │ │
│  │  │ n8n 容器     │              │ │
│  │  │ port: 5678   │              │ │
│  │  └──────────────┘              │ │
│  │         ↑                      │ │
│  │         │ http://n8n:5678      │ │
│  │         │                      │ │
│  │  ┌──────────────┐              │ │
│  │  │ scheduler    │              │ │
│  │  │ (定時觸發)    │              │ │
│  │  └──────────────┘              │ │
│  │                                │ │
│  └────────────────────────────────┘ │
│                                     │
│  💾 n8n_marketing_data (資料卷)     │
│     儲存 workflows 和設定            │
└─────────────────────────────────────┘
```

### 容器通訊

在同一個 `docker-compose.yml` 中的容器：
- ✅ 可以用**服務名稱**互相通訊（如 `http://n8n:5678`）
- ❌ 不需要使用 `localhost`、IP 地址或 `host.docker.internal`

---

## 方法比較

| 特性 | Cron + 觸發式 | Supervisor + 長期運行 |
|------|--------------|---------------------|
| **程式運行方式** | 執行一次就結束 | 一直運行 |
| **排程控制** | 系統 Cron | Python schedule 套件 |
| **崩潰處理** | 不需要（下次執行就好） | Supervisor 自動重啟 |
| **資源使用** | 低（大部分時間不運行） | 高（一直運行） |
| **時間精確度** | ⭐⭐⭐⭐⭐ 精確 | ⭐⭐⭐⭐ 有微小誤差 |
| **配置複雜度** | 簡單 | 中等 |
| **適合情境** | 定時任務 | 需要即時反應的服務 |
| **推薦度** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

---

## 實作方式

## 方法 1: Cron + 觸發式程式（推薦）

### 特點
- ✅ 最簡單
- ✅ 時間精確
- ✅ 不怕程式崩潰
- ✅ 資源消耗低

### 檔案結構

```
marketing/
├── docker-compose.yml
└── scheduler/
    ├── Dockerfile
    ├── requirements.txt
    ├── crontab
    ├── entrypoint.sh
    └── Marketing_trigger.py
```

### 1. Marketing_trigger.py

```python
import requests
import os
from datetime import datetime

# 從環境變數讀取 URL
N8N_WEBHOOK_URL = os.getenv("N8N_WEBHOOK_URL", "http://n8n:5678/webhook/Marketing_scrape")

def trigger_n8n():
    try:
        print(f"⏰ [{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] 觸發 n8n workflow...")
        response = requests.post(N8N_WEBHOOK_URL, timeout=30)
        
        if response.status_code == 200:
            print(f"✅ 成功!狀態碼: {response.status_code}")
            print(f"📄 回應: {response.text[:200]}")
        else:
            print(f"⚠️ 狀態碼: {response.status_code}")
            print(f"📄 回應: {response.text}")
    except Exception as e:
        print(f"❌ 錯誤: {str(e)}")

# 執行一次就結束
if __name__ == "__main__":
    trigger_n8n()
```

### 2. crontab

```bash
# 每 3 分鐘執行一次
*/3 * * * * cd /app && /usr/local/bin/python /app/Marketing_trigger.py >> /var/log/cron.log 2>&1

# 其他常見設定：
# */5 * * * *      # 每 5 分鐘
# */10 * * * *     # 每 10 分鐘
# 0 * * * *        # 每小時
# 0 9 * * *        # 每天早上 9:00
# 0 9 * * 1        # 每週一早上 9:00
# 0 9 1 * *        # 每月 1 號早上 9:00
```

### 3. entrypoint.sh

```bash
#!/bin/bash

# 載入 crontab
crontab /app/crontab

# 啟動 cron
cron

# 持續顯示日誌
tail -f /var/log/cron.log
```

### 4. Dockerfile

```dockerfile
FROM python:3.11-slim

# 安裝 cron
RUN apt-get update && apt-get install -y cron && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 安裝 Python 依賴
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 複製程式和設定檔
COPY Marketing_trigger.py .
COPY crontab .
COPY entrypoint.sh .

# 設定環境變數
ENV N8N_WEBHOOK_URL="http://n8n:5678/webhook/Marketing_scrape"

# 給予執行權限
RUN chmod +x /app/entrypoint.sh

# 建立日誌檔
RUN touch /var/log/cron.log

# 啟動
CMD ["/app/entrypoint.sh"]
```

### 5. requirements.txt

```
requests==2.31.0
```

### 6. docker-compose.yml

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n
    container_name: n8n_marketing
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://localhost:5678/
      - N8N_EDITOR_BASE_URL=http://localhost:5678
    volumes:
      - n8n_marketing_data:/home/node/.n8n
    networks:
      - app_network

  scheduler:
    build: ./scheduler
    container_name: marketing_scheduler
    restart: unless-stopped
    environment:
      - N8N_WEBHOOK_URL=http://n8n:5678/webhook/Marketing_scrape
      - TZ=Asia/Taipei
    depends_on:
      - n8n
    networks:
      - app_network

volumes:
  n8n_marketing_data:

networks:
  app_network:
    driver: bridge
```

### 部署與測試

```bash
# 建立並啟動服務
docker-compose up -d --build

# 查看日誌
docker-compose logs -f scheduler

# 手動測試執行
docker exec marketing_scheduler python /app/Marketing_trigger.py

# 查看 cron 任務列表
docker exec marketing_scheduler crontab -l

# 查看 cron 日誌
docker exec marketing_scheduler tail -f /var/log/cron.log
```

---

## 方法 2: Supervisor + 長期運行程式

### 特點
- ✅ 程式崩潰自動重啟
- ✅ 完整的程序監控
- ⚠️ 時間精確度稍差（可優化）
- ⚠️ 配置較複雜

### 檔案結構

```
marketing/
├── docker-compose.yml
└── scheduler/
    ├── Dockerfile
    ├── requirements.txt
    ├── supervisord.conf
    └── Marketing_schedule.py
```

### 1. Marketing_schedule.py

```python
import requests
import schedule
import time
import os
from datetime import datetime

N8N_WEBHOOK_URL = os.getenv("N8N_WEBHOOK_URL", "http://n8n:5678/webhook/Marketing_scrape")

def trigger_n8n():
    try:
        print(f"⏰ [{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] 觸發 n8n workflow...", flush=True)
        response = requests.post(N8N_WEBHOOK_URL, timeout=30)
        
        if response.status_code == 200:
            print(f"✅ 成功!狀態碼: {response.status_code}", flush=True)
        else:
            print(f"⚠️ 狀態碼: {response.status_code}", flush=True)
    except Exception as e:
        print(f"❌ 錯誤: {str(e)}", flush=True)

# 每 3 分鐘執行
schedule.every(3).minutes.do(trigger_n8n)

print("=" * 50, flush=True)
print("📅 排程器已啟動（長期運行模式）", flush=True)
print(f"🌐 Webhook URL: {N8N_WEBHOOK_URL}", flush=True)
print("⏰ 排程: 每 3 分鐘執行一次", flush=True)
print("=" * 50, flush=True)

# 持續運行
# 建議使用 sleep(10) 而非 sleep(60) 以提高精確度
while True:
    schedule.run_pending()
    time.sleep(10)  # 每 10 秒檢查一次
```

### 2. supervisord.conf

```ini
[supervisord]
nodaemon=true
logfile=/var/log/supervisor/supervisord.log
pidfile=/var/run/supervisord.pid
loglevel=info

[unix_http_server]
file=/var/run/supervisor.sock
chmod=0700

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[program:marketing_scheduler]
command=/usr/local/bin/python -u /app/Marketing_schedule.py
directory=/app
autostart=true
autorestart=true
startretries=999999
stderr_logfile=/var/log/supervisor/marketing_scheduler.err.log
stdout_logfile=/var/log/supervisor/marketing_scheduler.out.log
stdout_logfile_maxbytes=10MB
stderr_logfile_maxbytes=10MB
user=root
environment=N8N_WEBHOOK_URL="http://n8n:5678/webhook/Marketing_scrape"
```

### 3. Dockerfile

```dockerfile
FROM python:3.11-slim

# 安裝 supervisor
RUN apt-get update && \
    apt-get install -y supervisor && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 安裝 Python 依賴
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 複製程式
COPY Marketing_schedule.py .

# 複製 supervisor 設定
COPY supervisord.conf /etc/supervisor/supervisord.conf

# 建立日誌目錄
RUN mkdir -p /var/log/supervisor

# 設定環境變數
ENV N8N_WEBHOOK_URL="http://n8n:5678/webhook/Marketing_scrape"

# 啟動 supervisor
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]
```

### 4. requirements.txt

```
requests==2.31.0
schedule==1.2.0
```

### 5. docker-compose.yml

（與方法 1 相同）

### 部署與測試

```bash
# 建立並啟動服務
docker-compose up -d --build

# 查看日誌
docker-compose logs -f scheduler

# 進入容器
docker exec -it marketing_scheduler bash

# 查看程式輸出
cat /var/log/supervisor/marketing_scheduler.out.log

# 即時監控輸出
tail -f /var/log/supervisor/marketing_scheduler.out.log

# 查看 supervisor 日誌
cat /var/log/supervisor/supervisord.log

# 測試崩潰重啟
kill -9 7  # 假設 PID 是 7
sleep 3
tail -10 /var/log/supervisor/supervisord.log
```

### Supervisor 控制命令

```bash
# 查看狀態
supervisorctl status

# 停止程式
supervisorctl stop marketing_scheduler

# 啟動程式
supervisorctl start marketing_scheduler

# 重啟程式
supervisorctl restart marketing_scheduler

# 即時查看輸出
supervisorctl tail -f marketing_scheduler
```

---

## 重要概念

### 1. Docker Compose 容器通訊

**在同一個 docker-compose.yml 中**：

```yaml
services:
  n8n:          # 服務名稱
    ...
  
  scheduler:
    environment:
      # ✅ 使用服務名稱通訊
      - N8N_WEBHOOK_URL=http://n8n:5678/webhook/...
      
      # ❌ 不需要這些
      # - N8N_WEBHOOK_URL=http://localhost:5678/...
      # - N8N_WEBHOOK_URL=http://host.docker.internal:5678/...
      # - N8N_WEBHOOK_URL=http://192.168.x.x:5678/...
```

**原理**：Docker Compose 自動建立內部 DNS，容器可以用服務名稱互相找到對方。

### 2. Volume 資料持久化

**為什麼 n8n 需要 volume？**

```yaml
n8n:
  volumes:
    - n8n_marketing_data:/home/node/.n8n  # ✅ 需要！
```

- n8n 會產生和修改資料（workflows、credentials、設定）
- 容器重啟後需要保留這些資料
- Volume 確保資料不會遺失

**為什麼 scheduler 不需要 volume？**

```yaml
scheduler:
  # ❌ 不需要 volume
```

- 程式碼在建置時就固定了（從 Dockerfile COPY）
- 不需要保存執行歷史
- 日誌可以用 `docker logs` 查看
- 沒有會變動的資料

**什麼時候需要 volume？**

判斷標準：這個容器會**產生或修改**需要**長期保存**的資料嗎？

### 3. Docker Restart Policy

```yaml
restart: unless-stopped
```

| Policy | 說明 | 適用情境 |
|--------|------|----------|
| `no` | 不自動重啟 | 測試環境 |
| `always` | 總是重啟（包括手動停止後） | 關鍵服務 |
| `unless-stopped` | 總是重啟（除非手動停止） | **推薦：一般情況** |
| `on-failure` | 只在錯誤時重啟 | 可能正常退出的程式 |

**容器層級的重啟**：
- ✅ 容器崩潰會自動重啟
- ✅ Docker 服務重啟後會自動啟動容器
- ❌ 不監控容器內程式的狀態

### 4. Python 輸出緩衝

**問題**：為什麼 `print()` 的內容看不到？

```python
print("Hello")  # 輸出被緩衝了！
```

**解決方案 1**：使用 `-u` 參數（推薦）

```dockerfile
CMD ["python", "-u", "script.py"]
```

或在 supervisord.conf 中：

```ini
command=/usr/local/bin/python -u /app/script.py
```

**解決方案 2**：在 print 中加 `flush=True`

```python
print("Hello", flush=True)
```

### 5. Schedule 套件的時間精確度

**問題**：為什麼設定每 1 分鐘，實際卻是 2 分鐘？

```python
schedule.every(1).minutes.do(trigger_n8n)

while True:
    schedule.run_pending()
    time.sleep(60)  # ⚠️ 問題在這裡
```

**原因**：
1. `schedule` 記錄：下次執行時間 = 上次執行時間 + 1 分鐘
2. 但觸發本身需要時間（2-3 秒）
3. `sleep(60)` 每 60 秒才檢查一次
4. 可能錯過精確的執行時間

**解決方案**：縮短檢查間隔

```python
while True:
    schedule.run_pending()
    time.sleep(10)  # ✅ 改成 10 秒
```

**對比**：

| 檢查間隔 | 精確度 | CPU 使用 |
|---------|--------|---------|
| `sleep(60)` | ⭐⭐ | 極低 |
| `sleep(10)` | ⭐⭐⭐⭐ | 低 |
| `sleep(1)` | ⭐⭐⭐⭐⭐ | 中 |
| Cron | ⭐⭐⭐⭐⭐ | 極低 |

### 6. Cron 時間格式

```bash
# 格式: 分 時 日 月 週
# *  *  *  *  *
# │  │  │  │  │
# │  │  │  │  └─── 星期幾 (0-7, 0 和 7 都是週日)
# │  │  │  └────── 月份 (1-12)
# │  │  └───────── 日期 (1-31)
# │  └──────────── 小時 (0-23)
# └─────────────── 分鐘 (0-59)
```

**常用範例**：

```bash
# 每分鐘
* * * * *

# 每 5 分鐘
*/5 * * * *

# 每小時
0 * * * *

# 每天早上 9:00
0 9 * * *

# 每天早上 9:00 和下午 5:00
0 9,17 * * *

# 工作日（週一到週五）早上 9:00
0 9 * * 1-5

# 每週一早上 8:00
0 8 * * 1

# 每月 1 號早上 8:00
0 8 1 * *

# 每 15 分鐘（只在工作時間 9:00-18:00）
*/15 9-18 * * *

# 每天午夜
0 0 * * *

# 每週日凌晨 3:00
0 3 * * 0
```

---

## 測試與驗證

### 基本測試

```bash
# 1. 確認容器運行
docker ps | grep marketing

# 2. 查看日誌
docker-compose logs -f scheduler

# 3. 手動測試觸發
docker exec marketing_scheduler python /app/Marketing_trigger.py

# 4. 查看 n8n 是否收到請求
docker logs n8n_marketing | grep webhook
```

### 測試崩潰自動重啟（Supervisor）

```bash
# 進入容器
docker exec -it marketing_scheduler bash

# 找到程式 PID
python3 -c "
import os
for pid in os.listdir('/proc'):
    if pid.isdigit():
        try:
            with open(f'/proc/{pid}/cmdline', 'r') as f:
                cmdline = f.read()
                if 'Marketing_schedule.py' in cmdline:
                    print(f'找到程式 PID: {pid}')
        except:
            pass
"

# 殺掉程式
kill -9 <PID>

# 等待 3 秒
sleep 3

# 查看重啟日誌
tail -10 /var/log/supervisor/supervisord.log

# 應該看到：
# INFO exited: marketing_scheduler (terminated by SIGKILL)
# INFO spawned: 'marketing_scheduler' with pid XXX
# INFO success: marketing_scheduler entered RUNNING state

# 離開容器
exit
```

### 測試容器自動重啟

```bash
# 殺掉容器
docker kill marketing_scheduler

# 查看狀態（幾秒後會自動重啟）
docker ps | grep marketing_scheduler

# 查看重啟次數
docker inspect marketing_scheduler | grep RestartCount
```

---

## 故障排除

### 問題 1：找不到檔案

```
failed to compute cache key: "/Marketing_trigger.py": not found
```

**原因**：檔案不在正確位置

**解決**：
```bash
# 確認檔案結構
cd marketing
dir scheduler

# 應該看到：
# Dockerfile
# requirements.txt
# Marketing_trigger.py (或 Marketing_schedule.py)
```

### 問題 2：Webhook URL 錯誤

```
Connection refused / Name resolution failed
```

**檢查**：
```bash
# 在 scheduler 容器內測試連線
docker exec marketing_scheduler ping n8n

# 應該能 ping 通
```

**常見錯誤**：
- ❌ 使用 `localhost` 而非服務名稱
- ❌ 使用 IP 地址而非服務名稱
- ❌ 服務名稱拼錯

### 問題 3：看不到 log 輸出

**原因**：Python 輸出緩衝

**解決**：
1. 在命令中加 `-u`：`python -u script.py`
2. 或在 print 中加 `flush=True`

### 問題 4：supervisorctl 無法連接

```
unix:///var/run/supervisor.sock no such file
```

**原因**：`supervisord.conf` 缺少 socket 配置

**解決**：在 supervisord.conf 中加入：
```ini
[unix_http_server]
file=/var/run/supervisor.sock
chmod=0700

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface
```

### 問題 5：時間不準確

**檢查時區**：
```bash
docker exec marketing_scheduler date

# 確認 docker-compose.yml 中有：
environment:
  - TZ=Asia/Taipei
```

### 問題 6：Port 衝突

```
Bind for 0.0.0.0:5678 failed: port is already allocated
```

**解決**：
```bash
# 查看哪個程式佔用 5678
netstat -ano | findstr :5678

# 停止舊的容器
docker stop n8n

# 或改用其他 port
ports:
  - "5680:5678"
```

---

## 總結

### 推薦方案

| 需求 | 推薦方案 |
|------|---------|
| 簡單定時任務 | **Cron + 觸發式** ⭐⭐⭐⭐⭐ |
| 需要即時反應 | Supervisor + 長期運行 ⭐⭐⭐⭐ |
| 複雜排程邏輯 | Supervisor + 長期運行 ⭐⭐⭐⭐ |
| 最高時間精確度 | Cron + 觸發式 ⭐⭐⭐⭐⭐ |
| 需要程序監控 | Supervisor + 長期運行 ⭐⭐⭐⭐ |

### 快速參考

**部署指令**：
```bash
# 啟動
docker-compose up -d --build

# 查看日誌
docker-compose logs -f scheduler

# 重啟
docker-compose restart scheduler

# 停止
docker-compose down

# 停止並刪除資料
docker-compose down -v
```

**除錯指令**：
```bash
# 進入容器
docker exec -it marketing_scheduler bash

# 查看容器狀態
docker ps

# 查看容器日誌
docker logs marketing_scheduler

# 測試 webhook
curl -X POST http://localhost:5678/webhook/Marketing_scrape
```

### 關鍵要點

1. ✅ **容器通訊**：使用服務名稱（如 `http://n8n:5678`）
2. ✅ **輸出緩衝**：使用 `python -u` 或 `flush=True`
3. ✅ **時區設定**：加入 `TZ=Asia/Taipei`
4. ✅ **自動重啟**：使用 `restart: unless-stopped`
5. ✅ **時間精確**：Cron 最精確，schedule 需要縮短檢查間隔

---

## 附錄

### Cron 時間測試工具

線上工具：https://crontab.guru/

### Supervisor 官方文檔

http://supervisord.org/

### n8n 官方文檔

https://docs.n8n.io/

### Docker Compose 官方文檔

https://docs.docker.com/compose/

---

**文件版本**：1.0  
**最後更新**：2025-10-31  
**作者**：AI Assistant  
**授權**：MIT License

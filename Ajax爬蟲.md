# 公開資訊觀測站爬蟲開發筆記 📚

> 完整記錄如何從零開始開發 MOPS 爬蟲的所有知識

---

## 📖 目錄

1. [核心概念](#核心概念)
2. [開發流程](#開發流程)
3. [已完成的爬蟲](#已完成的爬蟲)
4. [技術選擇：AJAX vs Selenium](#技術選擇ajax-vs-selenium)
5. [防封鎖技巧](#防封鎖技巧)
6. [常見問題](#常見問題)
7. [擴展方向](#擴展方向)

---

## 核心概念

### 什麼是 AJAX 爬蟲？

當你在網頁上輸入公司代號查詢時，瀏覽器實際上是：

```
1. JavaScript 發送 AJAX 請求到伺服器
   ↓
2. 伺服器回傳 HTML 或 JSON 資料
   ↓
3. JavaScript 將資料顯示在網頁上
```

**AJAX 爬蟲的原理：直接跳過步驟 1，用程式發送相同的請求**

### 為什麼不用 Selenium？

| 特性 | **AJAX 爬蟲** | **Selenium** |
|------|--------------|--------------|
| 速度 | ⚡ 0.5 秒 | 🐌 10 秒 |
| 記憶體 | 💚 20 MB | 🔴 300 MB |
| 程式碼 | ✅ 30 行 | ❌ 80 行 |
| 適用場景 | 90% 的網站 | 需要 JS 執行、登入 |

**結論：公開資訊觀測站完全不需要 Selenium**

---

## 開發流程

### 步驟 1️⃣：找到 AJAX 請求

1. 打開 Chrome DevTools (F12)
2. 切換到 **Network** 面板
3. 在網頁上操作（例如輸入公司代號查詢）
4. 找到 XHR/Fetch 類型的請求

**關鍵資訊：**
- Request URL：`https://mopsov.twse.com.tw/mops/web/ajax_xxxxx`
- Request Method：`POST` 或 `GET`
- Payload：請求參數

### 步驟 2️⃣：截圖給 AI

需要提供的資訊：
1. **Headers 頁面**：Request URL、Request Method
2. **Payload 頁面**：所有參數和值
3. **Response 頁面**（選用）：回傳的資料格式

### 步驟 3️⃣：AI 產生程式碼

AI 會自動產生：
```python
import requests
from bs4 import BeautifulSoup

class Crawler:
    def __init__(self):
        self.base_url = "你提供的 URL"
        self.headers = {...}
    
    def get_data(self, stock_code):
        payload = {
            # 你提供的參數
        }
        response = requests.post(self.base_url, data=payload, headers=self.headers)
        return self.parse_response(response.text)
    
    def parse_response(self, html):
        # 自動解析 HTML
        soup = BeautifulSoup(html, 'html.parser')
        # ...
```

### 步驟 4️⃣：測試與調整

```bash
python your_crawler.py
```

如果有問題（例如 PDF 解析錯誤），提供：
- 錯誤訊息
- Response 的 HTML 範例
- 預期結果

---

## 已完成的爬蟲

### 1. 法人說明會爬蟲 📊

**檔案：** `mops_crawler.py`

**功能：**
- 查詢法說會時程、地點
- 取得簡報檔案（中英文 PDF）
- 影音連結、公司網站

**使用範例：**
```python
from mops_crawler import MOPSCrawler

crawler = MOPSCrawler()
result = crawler.get_investor_conference('2330')

print(f"法說會日期: {result['conferences'][0]['date']}")
print(f"簡報連結: {result['conferences'][0]['files']['chinese_pdf']}")
```

**API 資訊：**
- URL: `https://mopsov.twse.com.tw/mops/web/ajax_t100sb07_1`
- Method: `POST`
- 主要參數: `co_id`（公司代號）

---

### 2. 財報爬蟲 💰

**檔案：** `financial_crawler.py`

**功能：**
- 合併綜合損益表
- 營收、成本、毛利、淨利
- 每股盈餘 (EPS)
- 季度與累計資料

**使用範例：**
```python
from financial_crawler import FinancialStatementCrawler

crawler = FinancialStatementCrawler()

# 查詢最新季度
result = crawler.get_income_statement('2330')

# 查詢特定季度
result = crawler.get_income_statement('2330', year='114', season='1')

print(f"營業收入: {result['summary']['revenue'][0]['amount']:,.0f} 千元")
print(f"本期淨利: {result['summary']['net_income'][0]['amount']:,.0f} 千元")
print(f"EPS: {result['summary']['basic_eps'][0]['amount']:.2f}")
```

**API 資訊：**
- URL: `https://mopsov.twse.com.tw/mops/web/ajax_t164sb04`
- Method: `POST`
- 主要參數: `co_id`, `year`, `season`

---

### 3. 月營收爬蟲 📈

**檔案：** `monthly_revenue_crawler.py`

**功能：**
- 本月營收
- 去年同期比較
- 年增率
- 本年累計

**使用範例：**
```python
from monthly_revenue_crawler import MonthlyRevenueCrawler

crawler = MonthlyRevenueCrawler()

# 查詢最新月營收
result = crawler.get_monthly_revenue('2330')

# 查詢特定月份
result = crawler.get_monthly_revenue('2330', year='114', month='9')

print(f"本月營收: {result['current_month']:,.0f} 千元")
print(f"月增減: {result['mom_change_pct']}%")
print(f"年累計: {result['ytd_current']:,.0f} 千元")
```

**API 資訊：**
- URL: `https://mopsov.twse.com.tw/mops/web/ajax_t05st10_ifrs`
- Method: `POST`
- 主要參數: `co_id`, `year`, `month`

---

### 4. 重大訊息爬蟲 📰

**檔案：** `announcement_crawler.py`

**功能：**
- 公司重大訊息公告
- 發言日期、時間
- 公告主旨

**使用範例：**
```python
from announcement_crawler import AnnouncementCrawler

crawler = AnnouncementCrawler()

# 查詢今年度所有公告
result = crawler.get_announcements('2330', year='114')

# 查詢特定月份
result = crawler.get_announcements('2330', year='114', month='10')

print(f"共 {result['total_count']} 則訊息")
for ann in result['announcements'][:5]:
    print(f"[{ann['date']}] {ann['subject']}")
```

**API 資訊：**
- URL: `https://mopsov.twse.com.tw/mops/web/ajax_t05st01`
- Method: `POST`
- 主要參數: `co_id`, `year`, `month`, `b_date`, `e_date`

---

### 5. 整合分析工具 🎯

**檔案：** `integrated_example.py`

**功能：**
- 同時查詢法說會 + 財報
- 自動產生投資分析報告
- 多公司比較

**使用範例：**
```python
from integrated_example import CompanyAnalyzer

analyzer = CompanyAnalyzer()

# 產生完整報告
analyzer.create_investment_report('2330')

# 比較多家公司
analyzer.compare_companies(['2330', '2317', '2454'])
```

---

## 技術選擇：AJAX vs Selenium

### 📊 效能對比

```
任務：查詢一家公司的重大訊息

直接 AJAX:
├─ 啟動時間: 0 秒
├─ 請求時間: 0.3 秒
├─ 解析時間: 0.2 秒
└─ 總計: 0.5 秒

Selenium:
├─ 啟動瀏覽器: 2-3 秒
├─ 載入網頁: 3-5 秒
├─ 輸入並查詢: 1-2 秒
├─ 等待結果: 2-3 秒
└─ 總計: 8-13 秒

速度差異: 16-26 倍！
```

### ✅ 什麼時候用 AJAX？

**適用場景：**
- ✅ 資料在 AJAX response 中
- ✅ 不需要登入
- ✅ 沒有 CAPTCHA
- ✅ 不需要執行 JavaScript

**公開資訊觀測站完全符合 → 用 AJAX 就對了**

### ⚠️ 什麼時候必須用 Selenium？

**必須用 Selenium 的情況：**

1. **JavaScript 動態載入**
   ```
   資料完全由 JS 產生，找不到 API
   → 例如：股價即時圖表（Canvas 繪製）
   ```

2. **需要登入**
   ```
   需要帳號密碼、處理 Session
   → 例如：網路銀行、券商下單系統
   ```

3. **有 CAPTCHA 驗證碼**
   ```
   需要人工點選圖片
   → 例如：PTT 登入、某些售票網站
   ```

4. **複雜的反爬蟲機制**
   ```
   檢查瀏覽器指紋、Canvas、WebGL
   → 例如：某些電商網站
   ```

### 💡 如何判斷要用哪種？

**簡單測試法：**

```python
# 1. 先試試直接 AJAX
import requests
response = requests.get('https://目標網站.com/api')

# 2. 檢查 response
if response.status_code == 200 and len(response.text) > 0:
    print("✅ 用 AJAX 就可以")
else:
    print("❌ 可能需要 Selenium")
```

---

## 防封鎖技巧

### 🛡️ 基本防護

#### 1. 加上完整的 Headers

```python
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Referer': 'https://mopsov.twse.com.tw/mops/web/t05st01',
    'Origin': 'https://mopsov.twse.com.tw',
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9',
    'Accept-Language': 'zh-TW,zh;q=0.9,en-US;q=0.8',
}
```

**為什麼要加？**
- User-Agent：告訴伺服器你是瀏覽器
- Referer：告訴伺服器你從哪個頁面來
- Origin：告訴伺服器請求來源

#### 2. 控制請求頻率

```python
import time
import random

for code in stock_codes:
    result = crawler.get_data(code)
    
    # 隨機延遲 1-3 秒
    time.sleep(random.uniform(1, 3))
    
    # 每 100 個請求休息一下
    if i % 100 == 0:
        print("休息 1 分鐘...")
        time.sleep(60)
```

**建議頻率：**
- ✅ 每秒 1-2 個請求：完全沒問題
- ⚠️ 每秒 10+ 個請求：可能被限速
- ❌ 每秒 100+ 個請求：會被封 IP

#### 3. 使用 Session

```python
import requests

# ✅ 使用 Session (自動處理 Cookie)
session = requests.Session()
response = session.post(url, data=payload)

# ❌ 不要每次都建立新連線
response = requests.post(url, data=payload)
```

#### 4. 錯誤處理與重試

```python
import time

def safe_request(url, data, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.post(url, data=data, timeout=30)
            response.raise_for_status()
            return response
        except requests.exceptions.RequestException as e:
            print(f"請求失敗 (第 {attempt + 1} 次): {e}")
            if attempt < max_retries - 1:
                time.sleep(5)  # 等 5 秒再試
            else:
                return None
```

### 🚨 AJAX 請求會被擋嗎？

#### 可能被擋的原因：

| 檢查項目 | MOPS 是否檢查 | 解決方法 |
|---------|--------------|---------|
| User-Agent | ❌ 不檢查 | 加假的也行 |
| Referer | ❌ 不檢查 | 加假的也行 |
| Cookie/Session | ❌ 不需要 | - |
| CSRF Token | ❌ 沒有 | - |
| IP 限制 | ⚠️ 太頻繁會限速 | 加延遲 |
| Rate Limiting | ⚠️ 有，但很寬鬆 | 每秒 1-2 次就好 |
| JavaScript 檢查 | ❌ 沒有 | - |

**結論：MOPS 幾乎不會擋你，但請友善使用**

### 📝 實戰建議

#### ✅ 好的做法

```python
# 1. 合理的延遲
time.sleep(1)

# 2. 錯誤處理
try:
    result = crawler.get_data(code)
except Exception as e:
    print(f"失敗: {e}")

# 3. 儲存中間結果
if i % 50 == 0:
    save_to_file(results)

# 4. 顯示進度
print(f"進度: {i}/{total}")
```

#### ❌ 不好的做法

```python
# 1. 沒有延遲
for code in range(1000, 9999):
    crawler.get_data(code)  # 瘋狂請求

# 2. 沒有錯誤處理
result = crawler.get_data(code)  # 一出錯就爆掉

# 3. 不儲存中間結果
# 跑了 3 小時後程式當掉，一切重來

# 4. 沒有 timeout
response = requests.post(url)  # 可能永遠等下去
```

---

## 常見問題

### Q1: 如何知道一個網站用 AJAX 還是傳統方式？

**方法：開 DevTools 看 Network**

```
1. F12 開啟 DevTools
2. 切換到 Network 面板
3. 操作網頁（例如點擊按鈕）
4. 觀察：
   - 有 XHR/Fetch 請求 → AJAX
   - 整個頁面重新載入 → 傳統方式
```

### Q2: 為什麼我的程式抓不到資料？

**檢查清單：**

```python
# 1. 檢查 status code
print(response.status_code)  # 應該是 200

# 2. 檢查 response 內容
print(response.text[:500])  # 看前 500 字元

# 3. 檢查參數是否正確
print(payload)  # 確認參數名稱和值

# 4. 檢查是否需要 headers
headers = {
    'User-Agent': 'Mozilla/5.0...',
    'Referer': 'https://...'
}
```

### Q3: 如何處理中文編碼問題？

```python
# ✅ 正確做法
response = requests.post(url, data=payload)
response.encoding = 'utf-8'  # 指定編碼
html = response.text

# 儲存 JSON 時
with open('data.json', 'w', encoding='utf-8') as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
```

### Q4: 批次查詢時如何避免被封？

```python
import time
import random

def batch_query_safe(stock_codes):
    results = {}
    total = len(stock_codes)
    
    for i, code in enumerate(stock_codes, 1):
        print(f"進度: {i}/{total} - 正在查詢 {code}")
        
        # 查詢
        result = crawler.get_data(code)
        results[code] = result
        
        # 隨機延遲 1-3 秒
        delay = random.uniform(1, 3)
        time.sleep(delay)
        
        # 每 50 個儲存一次
        if i % 50 == 0:
            save_results(results, f'backup_{i}.json')
        
        # 每 100 個休息 1 分鐘
        if i % 100 == 0:
            print("休息 1 分鐘...")
            time.sleep(60)
    
    return results
```

### Q5: 如何解析複雜的 HTML 結構？

```python
from bs4 import BeautifulSoup

soup = BeautifulSoup(html, 'html.parser')

# 方法 1: 找特定 class
table = soup.find('table', class_='hasBorder')

# 方法 2: 找特定文字
cell = soup.find('td', string='營業收入')

# 方法 3: 用 CSS selector
rows = soup.select('table.hasBorder tr')

# 方法 4: 找所有符合的
all_tables = soup.find_all('table')

# 方法 5: 取文字內容
text = cell.get_text(strip=True)

# 方法 6: 取屬性
href = link.get('href')
```

### Q6: 資料解析錯誤怎麼辦？

**除錯步驟：**

```python
# 1. 先看原始 HTML
print(response.text)

# 2. 找到你要的部分
soup = BeautifulSoup(html, 'html.parser')
table = soup.find('table', class_='hasBorder')
print(table)

# 3. 逐步測試解析邏輯
rows = table.find_all('tr')
print(f"找到 {len(rows)} 行")

for i, row in enumerate(rows[:3]):  # 只看前 3 行
    print(f"\n第 {i} 行:")
    cells = row.find_all('td')
    for j, cell in enumerate(cells):
        print(f"  Cell {j}: {cell.get_text(strip=True)}")
```

---

## 擴展方向

### 🎯 可新增的爬蟲模組

公開資訊觀測站還有很多資料可以爬：

| 模組 | 頁面 | API | 難度 |
|-----|------|-----|------|
| 股東會資訊 | t05st02 | ajax_t05st02 | ⭐ 簡單 |
| 董監事持股 | t05st03 | ajax_t05st03 | ⭐ 簡單 |
| 資產負債表 | t164sb01 | ajax_t164sb01 | ⭐⭐ 中等 |
| 現金流量表 | t164sb05 | ajax_t164sb05 | ⭐⭐ 中等 |
| 股利政策 | t05st09 | ajax_t05st09 | ⭐ 簡單 |
| 營運概況 | t100sb06 | ajax_t100sb06 | ⭐⭐ 中等 |

### 📝 開發新模組的步驟

**步驟 1：找到頁面**
```
https://mopsov.twse.com.tw/mops/web/t05st02
```

**步驟 2：開啟 DevTools，找到 AJAX 請求**
```
Network → XHR/Fetch → ajax_t05st02
```

**步驟 3：截圖 Headers 和 Payload**
- Request URL
- Request Method
- 所有參數

**步驟 4：給 AI 或參考現有程式碼**
```python
# 複製 announcement_crawler.py
# 修改 URL 和參數
# 調整解析邏輯
```

### 🚀 進階應用

#### 1. 建立完整的資料庫

```python
import sqlite3

# 建立資料庫
conn = sqlite3.connect('mops_data.db')
cursor = conn.cursor()

# 建立表格
cursor.execute('''
    CREATE TABLE IF NOT EXISTS financial_statements (
        company_code TEXT,
        period TEXT,
        revenue REAL,
        net_income REAL,
        eps REAL,
        query_date TEXT,
        PRIMARY KEY (company_code, period)
    )
''')

# 插入資料
def save_to_db(result):
    cursor.execute('''
        INSERT OR REPLACE INTO financial_statements 
        VALUES (?, ?, ?, ?, ?, datetime('now'))
    ''', (
        result['company_code'],
        result['period'],
        result['summary']['revenue'][0]['amount'],
        result['summary']['net_income'][0]['amount'],
        result['summary']['basic_eps'][0]['amount']
    ))
    conn.commit()
```

#### 2. 定期自動更新

```python
import schedule
import time

def daily_update():
    """每天自動更新關注的公司"""
    watch_list = ['2330', '2317', '2454']
    
    for code in watch_list:
        # 更新月營收
        revenue = revenue_crawler.get_monthly_revenue(code)
        save_to_db(revenue)
        
        # 更新重大訊息
        announcements = announcement_crawler.get_announcements(code)
        check_important_news(announcements)
        
        time.sleep(1)

# 每天早上 9 點執行
schedule.every().day.at("09:00").do(daily_update)

while True:
    schedule.run_pending()
    time.sleep(60)
```

#### 3. Line 通知整合

```python
import requests

def send_line_notify(message):
    """發送 Line 通知"""
    token = 'YOUR_LINE_NOTIFY_TOKEN'
    
    headers = {
        'Authorization': f'Bearer {token}'
    }
    
    data = {
        'message': message
    }
    
    requests.post(
        'https://notify-api.line.me/api/notify',
        headers=headers,
        data=data
    )

# 使用範例
result = crawler.get_announcements('2330')
if result['total_count'] > 0:
    latest = result['announcements'][0]
    message = f"\n📢 台積電最新公告\n{latest['subject']}"
    send_line_notify(message)
```

#### 4. 視覺化分析

```python
import matplotlib.pyplot as plt
import pandas as pd

def plot_revenue_trend(stock_code, periods):
    """繪製營收趨勢圖"""
    revenues = []
    labels = []
    
    for period in periods:
        result = financial_crawler.get_income_statement(
            stock_code, 
            year=period['year'], 
            season=period['season']
        )
        revenues.append(result['summary']['revenue'][0]['amount'])
        labels.append(f"{period['year']}Q{period['season']}")
    
    plt.figure(figsize=(10, 6))
    plt.plot(labels, revenues, marker='o', linewidth=2)
    plt.title(f'{stock_code} 營收趨勢')
    plt.xlabel('期間')
    plt.ylabel('營收（千元）')
    plt.grid(True, alpha=0.3)
    plt.xticks(rotation=45)
    plt.tight_layout()
    plt.savefig(f'{stock_code}_revenue_trend.png', dpi=300)
    print(f"圖表已儲存")
```

---

## 🎓 學習資源

### Python 爬蟲基礎

- **requests 文件**: https://requests.readthedocs.io/
- **BeautifulSoup 文件**: https://www.crummy.com/software/BeautifulSoup/bs4/doc/
- **正規表達式測試**: https://regex101.com/

### 相關工具

- **JSON 格式化**: https://jsonformatter.org/
- **Chrome DevTools 教學**: https://developer.chrome.com/docs/devtools/

### 進階主題

- **requests-html**: 可執行 JavaScript 的輕量級方案
- **Scrapy**: 專業爬蟲框架
- **asyncio + aiohttp**: 異步爬蟲，速度更快

---

## ⚖️ 法律與道德

### ✅ 合法使用

- 公開資訊觀測站的資料是公開的
- 個人研究、學習用途
- 不過度頻繁請求
- 遵守 robots.txt

### ❌ 禁止行為

- 商業販售爬取的資料
- 過度頻繁請求造成伺服器負擔
- 用於違法目的（例如內線交易）
- 突破登入機制爬取非公開資料

### 💡 建議

1. **友善爬蟲**: 加上適當延遲
2. **標明身分**: User-Agent 可以加上你的聯絡方式
3. **尊重網站**: 不要造成伺服器負擔
4. **合理使用**: 僅供個人研究與學習

---

## 📌 快速參考

### 基本模板

```python
import requests
from bs4 import BeautifulSoup
import time

class MOPSCrawler:
    def __init__(self):
        self.base_url = "YOUR_API_URL"
        self.headers = {
            'User-Agent': 'Mozilla/5.0...',
            'Referer': 'https://mopsov.twse.com.tw/...'
        }
    
    def get_data(self, stock_code):
        payload = {
            'co_id': stock_code,
            'step': '1',
            # ... 其他參數
        }
        
        try:
            response = requests.post(
                self.base_url,
                data=payload,
                headers=self.headers,
                timeout=30
            )
            response.raise_for_status()
            return self.parse_response(response.text)
        except Exception as e:
            print(f"錯誤: {e}")
            return None
    
    def parse_response(self, html):
        soup = BeautifulSoup(html, 'html.parser')
        # 解析邏輯
        return result
    
    def batch_query(self, stock_codes):
        results = {}
        for code in stock_codes:
            result = self.get_data(code)
            results[code] = result
            time.sleep(1)  # 重要！
        return results
```

### 常用公司代號

```python
# 權值股
LARGE_CAPS = {
    '2330': '台積電',
    '2317': '鴻海',
    '2454': '聯發科',
    '2308': '台達電',
    '2881': '富邦金',
    '2882': '國泰金',
    '2412': '中華電',
    '2303': '聯電',
    '1301': '台塑',
    '1303': '南亞'
}

# 範例：批次查詢
for code, name in LARGE_CAPS.items():
    print(f"查詢 {name} ({code})")
    result = crawler.get_data(code)
    time.sleep(1)
```

---

## 🏁 總結

### 核心原則

1. **優先使用 AJAX**：比 Selenium 快 20 倍
2. **友善爬蟲**：加延遲、不過度請求
3. **錯誤處理**：try-except、重試機制
4. **儲存進度**：避免重跑
5. **測試先行**：先測試單一資料再批次

### 記住這個流程

```
找 AJAX → 截圖給 AI → 測試程式 → 批次執行 → 儲存資料
```

### 最重要的一句話

> **「打開 DevTools，找到 AJAX 請求，就成功了 90%」**

---

## 📞 問題排查

遇到問題時的檢查順序：

```
1. status_code 是 200 嗎？
   ↓
2. response.text 有內容嗎？
   ↓
3. 參數拼寫正確嗎？
   ↓
4. headers 設定了嗎？
   ↓
5. HTML 結構改變了嗎？
```

---

**最後更新**: 2024-11-04  
**版本**: 1.0  
**作者筆記**: 這份文件記錄了從零開始學習 MOPS 爬蟲的完整過程，未來忘記時可以隨時回來查閱。

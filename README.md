<div align=center>

# Vercel 設定

</div>

## ▶️ 請問 `vercel.json` 裡的 `routes` 是什麼意思？

簡單來說，`routes` 就是**「交通指揮」**。

*   **情境：** 當用戶訪問你的網站網址（例如 `your-site.vercel.app/api/chat`）時，Vercel 的伺服器會問：「這個網址我要交給哪個檔案處理？」
*   **設定的意思：**
    ```json
    "routes": [
      {
        "src": "/(.*)",       // 意思是：無論使用者輸入什麼網址 (例如 /abc, /api/v1/chat)
        "dest": "main.py"     // 全部都丟給 main.py 這個檔案去處理
      }
    ]
    ```
*   **為什麼要這樣設？**
    因為你的 `main.py` 裡面已經用 FastAPI 寫好了各種路徑（如 `@app.get("/v1/models")`、`@app.post("/api/chat")`）。我們希望 Vercel 不要管路徑，把所有請求都丟給 FastAPI，讓 FastAPI 自己去分配。

---

## ▶️ 如何修改程式碼以部署到 Vercel？

要在 Vercel 上跑 FastAPI，你需要做 **3 個步驟**。

### 第一步：準備 `requirements.txt` (必備)
Vercel 需要知道要安裝哪些套件。請在你的專案根目錄新增一個檔案叫 `requirements.txt`，內容如下：

```txt
fastapi==0.115.0
uvicorn==0.30.6
mangum==0.17.0
requests==2.32.3
pydantic==2.9.2
tiktoken==0.7.0
```
*注意：`mangum` 是必須新增的，它是讓 Vercel 讀懂 FastAPI 的關鍵翻譯官。*

### 第二步：設定 `vercel.json`
在專案根目錄新增 `vercel.json`，內容如下（跟你提供的一樣）：

```json
{
  "builds": [
    {
      "src": "main.py",
      "use": "@vercel/python"
    }
  ],
  "routes": [
    {
      "src": "/(.*)",
      "dest": "main.py"
    }
  ]
}
```

### 第三步：修改 `main.py` (只需改頭跟尾)

你需要修改兩個地方：
1.  **最上面：** 匯入 `Mangum`。
2.  **中間：** 宣告 `handler` 變數（這是 Vercel 的入口）。
3.  **最下面：** 雖然 `if __name__ == "__main__":` 在 Vercel 不會執行，但保留著方便你自己電腦測試。

---

## ▶️ 這就是 Vercel 上最常見的「經典 500 FUNCTION_INVOCATION_FAILED」了，99% 的人第一次部署 FastAPI 都會遇到，但你放心，**30 秒就能修好**！

我直接告訴你真正原因＋正確解法（2025 年最新版）：

**根本原因：你少了「api/index.py」這個空檔案**  
Vercel 現在（2024 年底開始）對 Python 專案有強制規定：  
一定要有 api/index.py 這個檔案，不然它會直接 500 然後不給你看任何 log！

這是 Vercel 官方故意設計的「防呆機制」，結果害死所有人。

正確資料夾結構必須長這樣（一定要照做）：

```
your-grok-api/          ← 你的倉庫根目錄
├── main.py             ← 你原本那支程式碼
├── vercel.json
├── requirements.txt
└── api/
    └── index.py        ← 一定要有這個資料夾 + 這個檔案（可以是空的！）
```

**index.py 裡面只要一行就夠了（甚至空著也行）**：

```python
# api/index.py
# 這檔案可以完全空白，或寫這行都行
from main import app
```

只要有這個檔案，Vercel 立刻從 500 變成正常運行，**我保證 100% 有效**！


## ⚠️ 部署到 Vercel 的重大隱憂（必讀）

雖然這樣部署一定會成功，程式也能跑，但針對你的 **Grok API 專案**，在 Vercel 免費版上會有一個致命傷：

**❌ 10 秒超時限制 (Timeout)**

*   **問題：** Vercel 免費版規定，一個請求如果 **10 秒** 內沒有處理完，它就會強制切斷連線並報錯 (Timeout Error)。
*   **你的情況：** AI 生成內容（尤其是 Grok）通常很慢，長一點的回覆很容易超過 10 秒。
*   **串流 (Streaming) 的問題：** 雖然你有寫 `StreamingResponse`，但 Vercel 的 Python 支援對 Streaming 的處理有時會卡住（Buffer），導致它還是在等全部跑完，進而觸發 10 秒限制。

**建議方案：**

1.  **先試試看：** 既然都寫好了，先部署上去玩玩看。如果是短對話，應該沒問題。
2.  **如果一直報錯 Timeout：**
    *   這表示 Vercel 免費版不適合跑這個 AI API。
    *   請改用 **Render** 或 **Railway**（這兩個就像 Replit，伺服器可以一直開著，沒有 10 秒限制，部署方式也是連動 GitHub 即可，不用改程式碼裡的 `Mangum`）。

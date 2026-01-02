# Manus 上下文工程：優化多智能體協作效率的深度解析

在多智能體系統中，效率的瓶頸往往不在於單個模型的推理速度，而在於**上下文（Context）的管理**。Manus 透過一套稱為「上下文工程」的策略，解決了長序列處理、KV-Cache 命中率以及動作空間爆炸等核心問題。

## 1. KV-Cache 導向的提示設計 (KV-Cache Aware Design)

Manus 的核心優化目標是最大化 **KV-Cache 命中率**。在代理循環中，輸入（上下文）通常遠大於輸出（動作），這使得預填充（Prefilling）成為主要的延遲來源。

### 穩定提示前綴 (Stable Prompt Prefix)
Manus 確保系統提示（System Prompt）和工具定義在序列化後保持絕對穩定。
*   **避免動態時間戳**：不在系統提示開頭包含精確到秒的時間戳，防止緩存失效。
*   **僅追加模式 (Append-only)**：上下文序列是確定性的，避免修改先前的操作或觀察結果，確保舊有的 KV-Cache 可以被後續步驟重複使用。
*   **確定性序列化**：在將 JSON 對象序列化為字符串時，強制執行穩定的鍵排序（Key Ordering），防止因庫函數隨機排序導致的緩存未命中。

## 2. 遮罩機制與狀態機 (Masking & State Machine)

隨著工具數量的增加，動作空間會變得過於龐大，導致模型選擇效率下降。Manus 採用了「遮罩而非移除」的策略。

### 遮罩 (Masking) vs. 動態加載 (RAG-based Tools)
傳統做法是根據需求動態添加或刪除工具，但這會導致 KV-Cache 全面失效。Manus 的做法是：
*   **靜態工具定義**：在上下文中保留所有可能的工具定義，以維持緩存穩定。
*   **Logit 遮罩**：利用上下文感知的**狀態機**，在模型解碼（Decoding）過程中直接遮蔽（Mask）無效動作的 Token Logits。
*   **命名空間優化**：透過一致的工具前綴（如 `browser_`、`shell_`），狀態機能夠輕鬆地強制代理在特定狀態下（如正在瀏覽網頁時）只能從特定工具組中選擇，而無需修改上下文內容。

## 3. 文件系統作為「無限外部上下文」 (Filesystem as Context)

當面對超長文檔或複雜網頁時，128K 的上下文窗口往往不足。Manus 引入了「可恢復壓縮」的概念。

### 外部化記憶
Manus 不將所有觀察結果（如完整的 HTML 或 PDF 文本）直接塞進 LLM 上下文，而是將其存儲在**虛擬機的文件系統**中。
*   **摘要與引用**：在上下文中僅保留關鍵摘要和文件路徑。
*   **按需讀取**：代理可以像人類一樣，根據需要使用 `read_file` 工具重新加載特定片段。
*   **壓縮比**：這種方法實現了高達 **100:1** 的有效壓縮比，同時保證了信息的完整性（因為原始數據始終在磁碟上可查）。

## 4. 多智能體間的通信優化

Manus 遵循 Go 語言的併發哲學：「**不要透過共享記憶體來通訊，而要透過通訊來共享記憶體**」。

| 優化維度 | 傳統做法 | Manus 上下文工程做法 |
| :--- | :--- | :--- |
| **數據傳遞** | 將所有中間結果放入全局上下文 | 透過文件路徑或特定消息傳遞，保持上下文精簡 |
| **動作選擇** | 讓模型從數百個工具中盲選 | 透過狀態機遮罩，將動作空間縮小到當前有效的子集 |
| **長任務處理** | 截斷舊上下文導致遺忘 | 將舊數據移至文件系統，實現「可恢復的記憶」 |

## 總結

Manus 的上下文工程不僅僅是提示詞優化，而是一套結合了**底層推理加速（KV-Cache）**、**編譯原理（Logit Masking）**與**作業系統設計（Filesystem as Memory）**的綜合方案。這使得 Manus 能夠在保持低延遲和低成本的同時，處理極其複雜且長期的多智能體協作任務。

***

## 參考資料

[1] [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
[2] [Context Engineering for AI Agents: Key Lessons from Manus](https://dev.to/contextspace_/context-engineering-for-ai-agents-key-lessons-from-manus-1lgb)
[3] [Deep Dive into Context Engineering for Agents](https://galileo.ai/blog/context-engineering-for-agents)
[4] [Manus: Context Engineering Strategies for Production AI Agents](https://www.zenml.io/llmops-database/context-engineering-strategies-for-production-ai-agents)

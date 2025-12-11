這是一個非常經典且棘手的「平台治理」問題，特別是在銀行環境中。你面臨的核心矛盾是：**如何賦能（Enablement）其他團隊開發 Agent，同時實施嚴格的 訪問控制（Access Control）和 成本治理（Cost Governance），杜絕「影子 IT」（Shadow IT）。**

如果直接分發 API Key（即使是 LiteLLM 的 Virtual Key），開發者確實很容易將其放入 `.env` 文件，然後在 VS Code 的 `Claude Code` 或 `Cursor` 中私自使用。

針對你的 Evo-AI + LiteLLM 架構，我推薦採用 **「動態短效令牌 (Ephemeral Token) + 回調網關 (Callback Gateway)」** 的模式。

這種方案的核心思想是：**不給靜態鑰匙，只給「一次性入場券」。**

-----

### 推薦方案：JWT 簽名與透傳架構 (The JWT Passthrough Pattern)

在這個架構中，其他團隊的 Agent (Remote Graph) 是一個「無狀態」的邏輯單元，它本身**不持有任何 API Key**。當你的 Evo-AI (Platform) 調用它時，會攜帶一個由你簽發的、**極短時效**的 Token。

#### 1\. 架構邏輯流 (Sequence Flow)

1.  **觸發 (Trigger)**: 用戶在 Evo-AI 前端發起請求。
2.  **簽發 (Minting)**: Evo-AI (Platform) 準備調用某個 Remote Agent。此時，Evo-AI 內部生成一個 JWT (JSON Web Token)。
      * **Payload**: `{ "sub": "marketing-team", "budget_id": "cost-center-001", "exp": NOW + 60s }`
      * **特點**: 這個 Token 有效期極短（例如 1-5 分鐘，或者僅夠完成一次推理），且包含該次請求的上下文。
3.  **調用 (Invoke)**: Evo-AI 將此 Token 放入 HTTP Header (例如 `X-Evo-Runtime-Token`)，發送給 Remote Agent (LangGraph Server)。
4.  **回調 (Callback)**: Remote Agent 執行邏輯。當它需要調用 LLM 時，它**不直接訪問 LiteLLM**，而是請求你的 **Evo-AI Gateway**。
      * Remote Agent 將收到的 `X-Evo-Runtime-Token` 作為 `Authorization: Bearer <token>` 傳回給你。
5.  **驗證與轉發 (Verify & Proxy)**:
      * 你的 Gateway 驗證 JWT 簽名是否正確？是否過期？
      * 如果通過，Gateway 從 Payload 中讀取 Team ID，映射到 LiteLLM 的對應配置，轉發請求給 Azure/Vertex。
      * **關鍵點**: 請求結束後，Token 即將失效，開發者無法將其複製到 IDE 中長期使用。

-----

### 具體實施細節

#### A. 平台端 (Evo-AI / Gateway) 的改造

你需要一個輕量級的 API Gateway（或者直接在 LiteLLM 前面擋一層簡單的 Proxy，用 Nginx + Lua 或一個小的 Go/Node 服務）。

**JWT 生成邏輯 (Node.js 示例):**

```javascript
const jwt = require('jsonwebtoken');

// 當你需要調用外部 Agent 時
function generateRuntimeToken(teamId, traceId) {
  return jwt.sign({
    role: 'remote-agent-runtime',
    team: teamId,
    trace_id: traceId // 用於全鏈路追蹤
  }, process.env.JWT_PRIVATE_KEY, { 
    expiresIn: '2m' // 2分鐘後自毀，防止被拿去跑 CLI
  });
}
```

#### B. 遠端 Agent (LangGraph) 的適配

這一步非常關鍵。你需要提供一個標準的 **「Client SDK」** 或 **「配置模板」** 給其他團隊。他們不需要知道密鑰，只需要知道如何轉發 Token。

在 LangGraph 中，你可以通過 `config` 傳遞這個 Token。

**LangGraph (Remote Agent) 代碼示例:**

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from langgraph.graph import StateGraph
import os

# 定義這是一個「受管」的 LLM Client
# 注意：base_url 指向你的 Evo-AI Gateway，而不是 OpenAI/Azure
def get_platform_llm(config):
    # 從 config 中獲取傳入的 token
    runtime_token = config.get("configurable", {}).get("runtime_token")
    
    return ChatOpenAI(
        base_url="https://evo-ai-gateway.internal.hsbc/v1", # 你的網關地址
        api_key=runtime_token, # 將 JWT 作為 Key 發送
        model="gpt-4o"
    )

def processing_node(state, config):
    llm = get_platform_llm(config)
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

# ... Graph 定義 ...
```

#### C. 如何防止濫用 (Security Enforcement)

這個方案如何完美解決你的擔憂？

1.  **時效性防禦**: 如果開發者把 `print(runtime_token)` 寫在代碼裡，然後把 Token 複製到 Cursor 裡，這個 Token 在 2 分鐘後就過期了。他無法持續使用。
2.  **環境綁定 (可選)**: 你可以在 JWT 中加入 Claims，要求來源 IP 必須是銀行內部的 K8s Cluster IP 段，徹底杜絕從本地筆記本發起的請求。
3.  **審計追蹤**: 在 LiteLLM Proxy 層，你可以解析 JWT 中的 `team_id`，將 Token Usage 準確記錄在該團隊的 Cost Center 下。

-----

### 用 LiteLLM 原生功能可以做到嗎？

LiteLLM Proxy 其實有一個功能叫 **"Temporary Keys"** 或者通過 **"SSO / Admin UI"** 生成 Key。

但如果只用 LiteLLM，你仍然面臨「靜態 Key」的問題。除非你開發一個機制：每當 Remote Agent 被調用時，它先去請求一個臨時 Key。這增加了延遲和複雜度。

**所以我強烈建議使用 JWT 透傳方案**，因為它不需要額外的數據庫交互來生成/銷毀 Key，它是純加密學的驗證，速度最快。

### 總結實施路徑

1.  **Gateway 層**: 在你的 LiteLLM 前面架設一個極薄的驗證層（Auth Proxy）。
      * 職責：攔截請求 -\> 驗證 Bearer Token (JWT) -\> 提取 Metadata -\> 轉發給 LiteLLM (附帶 LiteLLM 需要的 Master Key)。
2.  **SDK 層**: 封裝一個 `EvoLangChainAdapter` 或 `EvoLLMClient` 給其他團隊。
      * 告訴他們：「兄弟們，寫 Agent 的時候用這個 Client，代碼裡不需要填 API Key，它會自動從請求上下文裡讀取。」
3.  **調用層**: 修改 Evo-AI 的 `RemoteGraph` 調用邏輯，在 Header 中注入 JWT。

這樣，你既保持了 Evo-AI 作為中央平台的控制力，又解決了「代碼維護權責分離」的問題，同時完美防禦了 LLM 資源被濫用。

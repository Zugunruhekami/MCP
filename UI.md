*Thinking...*

> The user wants me to create a Multi-agent Conversation (MCP) server registry with a modern UI. This registry will include both HSBC AI Markets Team's own MCP servers and open-source ones from Anthropic's GitHub list. The goal is to move tools from their "AI Matrix" framework (which uses LangGraph) to a separate repository that will function as an MCP registry.
> 
> Let me break down the key components for this application:
> 
> 1. It should display:
>    - List of MCP servers (both from HSBC AI Markets Team and open-source ones)
>    - Each server's configuration
>    - Documentation for each server
> 
> 2. The UI should be:
>    - Modern and clean
>    - Clear and organized
> 
> For this, I'll need to create:
> - A responsive web interface
> - A way to view server details
> - Organized layout to display configurations and documentation
> 
> Let me create a responsive web application using HTML, CSS (with Tailwind), and JavaScript that fulfills these requirements. I'll use Tailwind CSS for styling as recommended in the instructions.
> 
> The application will include:
> - A listing of MCP servers 
> - Ability to view details for each server
> - Documentation section
> - Configuration display
> 
> I'll make sure it's responsive, supports dark/light mode, and follows the guidelines provided.


# MCP Server Registry Application

I'll create a modern UI for an MCP server registry that displays both HSBC AI Markets Team's servers and open-source ones from Anthropic's GitHub list. This will help separate the tools from the AI Matrix LangGraph repo, focusing it solely on agent orchestration.

## Application Plan:
1. Create a responsive registry interface with dark/light mode support
2. Display server lists categorized by source (HSBC vs Open-source)
3. Show detailed configurations and documentation for each server
4. Implement search and filtering functionality
5. Use Tailwind CSS for styling

Here's the complete code:

```html id=mcp-registry
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MCP Server Registry</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        tailwind.config = {
            darkMode: 'class',
            theme: {
                extend: {
                    colors: {
                        primary: '#5D5CDE',
                        secondary: '#4B4ACA',
                        hsbc: {
                            red: '#DB0011',
                            light: '#FFF3F5'
                        },
                        dark: {
                            bg: '#181818',
                            card: '#222222',
                            border: '#333333'
                        }
                    }
                }
            }
        }
    </script>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/prismjs@1.29.0/themes/prism.min.css">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/prismjs@1.29.0/themes/prism-tomorrow.min.css">
    <script src="https://cdn.jsdelivr.net/npm/prismjs@1.29.0/prism.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/prismjs@1.29.0/components/prism-json.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/prismjs@1.29.0/components/prism-python.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/marked@4.3.0/marked.min.js"></script>
</head>
<body class="antialiased bg-white dark:bg-dark-bg text-gray-800 dark:text-gray-200 transition-colors duration-200">
    <div class="min-h-screen flex flex-col">
        <!-- Header -->
        <header class="sticky top-0 z-10 bg-white dark:bg-dark-bg border-b border-gray-200 dark:border-dark-border">
            <div class="container mx-auto px-4 py-3 flex items-center justify-between">
                <div class="flex items-center space-x-2">
                    <div class="bg-primary p-2 rounded-md">
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6 text-white" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 10V3L4 14h7v7l9-11h-7z" />
                        </svg>
                    </div>
                    <div>
                        <h1 class="text-xl font-bold">MCP Server Registry</h1>
                        <p class="text-xs text-gray-500 dark:text-gray-400">HSBC AI Markets Team</p>
                    </div>
                </div>
                <div class="flex items-center space-x-4">
                    <div class="relative">
                        <input id="search-input" type="text" placeholder="Search servers..." class="w-full md:w-64 px-4 py-2 text-base rounded-md border border-gray-300 dark:border-dark-border bg-white dark:bg-dark-card focus:outline-none focus:ring-2 focus:ring-primary">
                    </div>
                    <button id="theme-toggle" class="p-2 rounded-full hover:bg-gray-200 dark:hover:bg-dark-card">
                        <!-- Sun icon for dark mode -->
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 hidden dark:block" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z" />
                        </svg>
                        <!-- Moon icon for light mode -->
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 block dark:hidden" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M20.354 15.354A9 9 0 018.646 3.646 9.003 9.003 0 0012 21a9.003 9.003 0 008.354-5.646z" />
                        </svg>
                    </button>
                </div>
            </div>
        </header>

        <!-- Main Content -->
        <main class="flex-grow container mx-auto px-4 py-6">
            <!-- Filter tabs -->
            <div class="mb-6 border-b border-gray-200 dark:border-dark-border">
                <div class="flex flex-wrap -mb-px">
                    <button data-filter="all" class="filter-btn mr-4 py-2 px-1 border-b-2 border-primary text-primary font-medium">
                        All Servers
                    </button>
                    <button data-filter="hsbc" class="filter-btn mr-4 py-2 px-1 border-b-2 border-transparent hover:border-gray-300 dark:hover:border-gray-700 font-medium">
                        HSBC AI Markets Team
                    </button>
                    <button data-filter="open-source" class="filter-btn mr-4 py-2 px-1 border-b-2 border-transparent hover:border-gray-300 dark:hover:border-gray-700 font-medium">
                        Open Source
                    </button>
                </div>
            </div>

            <!-- Server grid -->
            <div id="server-grid" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                <!-- Servers will be dynamically populated here -->
            </div>

            <!-- Empty state -->
            <div id="empty-state" class="hidden py-12 text-center">
                <svg xmlns="http://www.w3.org/2000/svg" class="h-16 w-16 mx-auto text-gray-400" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.172 16.172a4 4 0 015.656 0M9 10h.01M15 10h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                </svg>
                <h3 class="mt-4 text-lg font-medium">No servers found</h3>
                <p class="mt-1 text-gray-500 dark:text-gray-400">Try adjusting your search or filter to find what you're looking for.</p>
            </div>
        </main>

        <!-- Server detail modal -->
        <div id="server-modal" class="fixed inset-0 z-50 hidden overflow-y-auto">
            <div class="flex items-center justify-center min-h-screen px-4 pt-4 pb-20 text-center sm:block sm:p-0">
                <div class="fixed inset-0 transition-opacity bg-gray-500 bg-opacity-75 dark:bg-opacity-90" aria-hidden="true"></div>
                <span class="hidden sm:inline-block sm:align-middle sm:h-screen" aria-hidden="true">&#8203;</span>
                <div class="inline-block w-full max-w-4xl p-6 my-8 overflow-hidden text-left align-middle transition-all transform bg-white dark:bg-dark-card shadow-xl rounded-lg sm:align-middle">
                    <div class="absolute top-0 right-0 pt-4 pr-4">
                        <button type="button" id="close-modal" class="text-gray-400 hover:text-gray-500 focus:outline-none">
                            <svg class="h-6 w-6" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                            </svg>
                        </button>
                    </div>
                    <div id="server-detail-content" class="mt-2">
                        <!-- Server details will be dynamically populated here -->
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script>
        // Check user's preferred color scheme
        if (window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches) {
            document.documentElement.classList.add('dark');
        }
        
        // Listen for changes in color scheme preference
        window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', event => {
            if (event.matches) {
                document.documentElement.classList.add('dark');
            } else {
                document.documentElement.classList.remove('dark');
            }
        });

        // Sample data (in a real application, this would come from an API)
        const mcpServers = [
            {
                id: 1,
                name: "AI Matrix Trading Agent",
                description: "Core trading agent MCP server with market data integration.",
                type: "hsbc",
                status: "active",
                version: "1.2.1",
                lastUpdated: "2023-10-15",
                config: {
                    baseUrl: "https://ai-matrix-trading.internal.hsbc.com",
                    apiVersion: "v2",
                    capabilities: ["market-data", "trade-execution", "portfolio-analysis"],
                    maxConcurrentRequests: 50,
                    timeout: 30000
                },
                documentation: `
# AI Matrix Trading Agent

This agent provides real-time market data integration and trading capabilities.

## Features

- Real-time market data access
- Trade execution with compliance checks
- Portfolio risk assessment
- Market sentiment analysis

## Usage

\`\`\`python
from ai_matrix import TradingAgent

agent = TradingAgent()
market_data = agent.get_market_data("AAPL", timeframe="1d")
trade_result = agent.execute_trade("AAPL", quantity=100, side="buy")
\`\`\`

## Configuration

The agent can be configured through the \`config.json\` file:

\`\`\`json
{
  "api_key": "YOUR_API_KEY",
  "environment": "production",
  "log_level": "info",
  "max_retries": 3
}
\`\`\`
                `
            },
            {
                id: 2,
                name: "Risk Assessment MCP",
                description: "Specialized agent for financial risk assessment and reporting.",
                type: "hsbc",
                status: "active",
                version: "0.9.5",
                lastUpdated: "2023-11-20",
                config: {
                    baseUrl: "https://risk-assessment.internal.hsbc.com",
                    apiVersion: "v1",
                    capabilities: ["market-risk", "credit-risk", "operational-risk", "compliance-check"],
                    maxConcurrentRequests: 25,
                    timeout: 45000
                },
                documentation: `
# Risk Assessment MCP

This MCP server provides comprehensive risk assessment capabilities for financial operations.

## Core Functions

- Market risk evaluation
- Credit risk scoring
- Operational risk monitoring
- Regulatory compliance checks

## Integration

\`\`\`python
from ai_matrix.risk import RiskAssessmentAgent

risk_agent = RiskAssessmentAgent()
portfolio_risk = risk_agent.assess_portfolio_risk(portfolio_id="P12345")
compliance_report = risk_agent.generate_compliance_report(trade_ids=["T001", "T002"])
\`\`\`

## Configuration Parameters

\`\`\`json
{
  "risk_models": ["VaR", "Expected Shortfall", "Stress Testing"],
  "confidence_level": 0.95,
  "time_horizon": "10d",
  "simulation_runs": 10000
}
\`\`\`
                `
            },
            {
                id: 3,
                name: "Research Assistant MCP",
                description: "Financial research and market intelligence agent.",
                type: "hsbc",
                status: "beta",
                version: "0.5.0",
                lastUpdated: "2023-12-05",
                config: {
                    baseUrl: "https://research-assistant.internal.hsbc.com",
                    apiVersion: "beta",
                    capabilities: ["news-analysis", "report-generation", "sentiment-analysis", "competitor-tracking"],
                    maxConcurrentRequests: 20,
                    timeout: 60000
                },
                documentation: `
# Research Assistant MCP

An advanced research assistant for financial market intelligence and report generation.

## Capabilities

- Real-time news analysis and summarization
- Automated financial report generation
- Market sentiment tracking
- Competitor intelligence gathering

## Example Usage

\`\`\`python
from ai_matrix.research import ResearchAgent

research = ResearchAgent()
news_summary = research.get_news_summary(ticker="AAPL", days=7)
market_report = research.generate_market_report(sector="Technology", format="pdf")
\`\`\`

## Personalization

Configure your research preferences in the settings:

\`\`\`json
{
  "preferred_sources": ["Bloomberg", "Reuters", "WSJ"],
  "update_frequency": "hourly",
  "alert_thresholds": {
    "sentiment_change": 0.15,
    "price_movement": 5.0
  }
}
\`\`\`
                `
            },
            {
                id: 4,
                name: "Claude MCP",
                description: "Anthropic's Claude assistant API integration.",
                type: "open-source",
                status: "active",
                version: "1.0.0",
                lastUpdated: "2023-09-28",
                config: {
                    baseUrl: "https://api.anthropic.com",
                    apiVersion: "v1",
                    capabilities: ["text-generation", "reasoning", "conversation"],
                    models: ["claude-3-opus-20240229", "claude-3-sonnet-20240229", "claude-3-haiku-20240307"],
                    maxTokens: 4096
                },
                documentation: `
# Claude MCP Server

This MCP server provides access to Anthropic's Claude AI assistant models.

## Available Models

- claude-3-opus-20240229: Most powerful model for complex tasks
- claude-3-sonnet-20240229: Balanced model for general use 
- claude-3-haiku-20240307: Fastest and most efficient model

## Basic Usage

\`\`\`python
from langchain.llms import Anthropic

llm = Anthropic(model="claude-3-sonnet-20240229")
response = llm.generate("Explain quantum computing in simple terms")
\`\`\`

## Advanced Configuration

\`\`\`python
from langchain.llms import Anthropic

llm = Anthropic(
    model="claude-3-opus-20240229",
    max_tokens=1000,
    temperature=0.7,
    top_p=0.9,
    top_k=50
)
\`\`\`
                `
            },
            {
                id: 5,
                name: "OpenAI MCP",
                description: "OpenAI's API integration for GPT models.",
                type: "open-source",
                status: "active",
                version: "1.1.2",
                lastUpdated: "2023-11-15",
                config: {
                    baseUrl: "https://api.openai.com",
                    apiVersion: "v1",
                    capabilities: ["text-generation", "embedding", "vision", "audio"],
                    models: ["gpt-4", "gpt-4-vision-preview", "gpt-3.5-turbo"],
                    maxTokens: 8192
                },
                documentation: `
# OpenAI MCP Server

This MCP server provides access to OpenAI's powerful language models and related capabilities.

## Supported Models

- GPT-4: Most powerful general-purpose model
- GPT-4-Vision: Model with image understanding capabilities
- GPT-3.5-Turbo: Fast and cost-effective model

## Basic Example

\`\`\`python
from langchain.llms import OpenAI

llm = OpenAI(model="gpt-4")
response = llm.generate("Write a short poem about artificial intelligence")
\`\`\`

## Vision Capabilities

\`\`\`python
from langchain.llms import OpenAI
from langchain.schema import HumanMessage, SystemMessage, ImageURL

messages = [
    SystemMessage(content="You are a helpful assistant."),
    HumanMessage(
        content=[
            {"type": "text", "text": "What's in this image?"},
            {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}}
        ]
    )
]

llm = OpenAI(model="gpt-4-vision-preview")
response = llm.generate(messages)
\`\`\`
                `
            },
            {
                id: 6,
                name: "Langchain Expression Language MCP",
                description: "Server for executing LangChain Expression Language (LCEL) chains.",
                type: "open-source",
                status: "active",
                version: "0.8.0",
                lastUpdated: "2023-12-01",
                config: {
                    baseUrl: "https://lcel-server.example.com",
                    apiVersion: "v1",
                    capabilities: ["chain-execution", "parallel-processing", "memory-management"],
                    supportedIntegrations: ["OpenAI", "Anthropic", "HuggingFace", "Pinecone", "Chroma"],
                    maxChainSteps: 50
                },
                documentation: `
# LangChain Expression Language (LCEL) MCP

This MCP server enables execution of LangChain Expression Language chains and sequences.

## Features

- Declarative chain construction
- Parallel processing of chain steps
- Built-in memory management
- Support for multiple model providers

## Basic Chain Example

\`\`\`python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI

prompt = ChatPromptTemplate.from_template("tell me a short joke about {topic}")
model = ChatOpenAI(model="gpt-3.5-turbo")
output_parser = StrOutputParser()

chain = prompt | model | output_parser

response = chain.invoke({"topic": "programming"})
\`\`\`

## Advanced RAG Pipeline

\`\`\`python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_core.runnables import RunnableParallel, RunnablePassthrough

# Retrieval components
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(embedding_function=embeddings)
retriever = vectorstore.as_retriever()

# LLM
llm = ChatOpenAI(model="gpt-4")

# RAG prompt
template = """Answer the question based on the following context:
{context}

Question: {question}
"""
prompt = ChatPromptTemplate.from_template(template)

# RAG chain
rag_chain = (
    RunnableParallel(
        {"context": retriever, "question": RunnablePassthrough()}
    )
    | prompt
    | llm
    | StrOutputParser()
)

response = rag_chain.invoke("What is the capital of France?")
\`\`\`
                `
            },
            {
                id: 7,
                name: "Compliance Check MCP",
                description: "Automated compliance checking for financial transactions.",
                type: "hsbc",
                status: "active",
                version: "2.0.1",
                lastUpdated: "2024-01-10",
                config: {
                    baseUrl: "https://compliance-check.internal.hsbc.com",
                    apiVersion: "v2",
                    capabilities: ["transaction-screening", "regulatory-compliance", "audit-trail", "sanction-checking"],
                    maxConcurrentRequests: 100,
                    timeout: 15000
                },
                documentation: `
# Compliance Check MCP

Automated compliance verification for financial transactions and operations.

## Key Functions

- Real-time transaction screening
- Regulatory compliance verification  
- Comprehensive audit trail generation
- Sanction list checking
- KYC/AML verification

## Integration Example

\`\`\`python
from ai_matrix.compliance import ComplianceAgent

compliance = ComplianceAgent()

# Check a single transaction
result = compliance.check_transaction({
    "transaction_id": "TX123456",
    "sender": "ACME Corp",
    "recipient": "Global Trading Ltd",
    "amount": 250000,
    "currency": "USD",
    "purpose": "Consulting services"
})

# Batch check multiple transactions
batch_results = compliance.batch_check([tx1, tx2, tx3])
\`\`\`

## Compliance Rules Configuration

\`\`\`json
{
  "rules": {
    "transaction_limits": {
      "USD": 1000000,
      "EUR": 850000,
      "GBP": 750000
    },
    "high_risk_countries": ["Country1", "Country2"],
    "enhanced_due_diligence_threshold": 500000
  },
  "update_frequency": "daily",
  "sanction_lists": ["UN", "OFAC", "EU", "UK"]
}
\`\`\`
                `
            },
            {
                id: 8,
                name: "Hugging Face MCP",
                description: "Server for Hugging Face model inference and fine-tuning.",
                type: "open-source",
                status: "active",
                version: "1.0.3",
                lastUpdated: "2023-10-20",
                config: {
                    baseUrl: "https://api.huggingface.co",
                    apiVersion: "v1",
                    capabilities: ["inference", "fine-tuning", "embedding", "token-classification"],
                    supportedTasks: ["text-generation", "summarization", "translation", "question-answering", "image-classification"],
                    maxConcurrentRequests: 30
                },
                documentation: `
# Hugging Face MCP Server

This MCP server provides access to Hugging Face's extensive model hub and inference API.

## Available Tasks

- Text generation
- Summarization
- Translation
- Question answering
- Image classification
- Token classification
- And many more...

## Basic Usage

\`\`\`python
from langchain.llms import HuggingFaceHub

# Text generation with a specific model
llm = HuggingFaceHub(
    repo_id="mistralai/Mistral-7B-Instruct-v0.1",
    task="text-generation"
)
response = llm.generate("Explain how transformers work in deep learning")

# Summarization
summarizer = HuggingFaceHub(
    repo_id="facebook/bart-large-cnn",
    task="summarization"
)
summary = summarizer.generate(long_text)
\`\`\`

## Model Fine-tuning Example

\`\`\`python
from huggingface_hub import login
from transformers import AutoModelForSequenceClassification, Trainer, TrainingArguments

# Login to Hugging Face
login(token="YOUR_HUGGINGFACE_TOKEN")

# Load model and configure training
model = AutoModelForSequenceClassification.from_pretrained("distilbert-base-uncased")
training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=3,
    per_device_train_batch_size=16,
    per_device_eval_batch_size=16,
    evaluation_strategy="epoch"
)

# Create trainer and fine-tune
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset
)
trainer.train()
\`\`\`
                `
            },
        ];

        // DOM elements
        const serverGrid = document.getElementById('server-grid');
        const emptyState = document.getElementById('empty-state');
        const serverModal = document.getElementById('server-modal');
        const serverDetailContent = document.getElementById('server-detail-content');
        const closeModalBtn = document.getElementById('close-modal');
        const searchInput = document.getElementById('search-input');
        const themeToggleBtn = document.getElementById('theme-toggle');
        const filterBtns = document.querySelectorAll('.filter-btn');

        // Current filter state
        let currentFilter = 'all';
        let searchQuery = '';

        // Render server cards
        function renderServers() {
            serverGrid.innerHTML = '';
            
            // Filter servers based on current filter and search query
            const filteredServers = mcpServers.filter(server => {
                const matchesFilter = currentFilter === 'all' || server.type === currentFilter;
                const matchesSearch = searchQuery === '' || 
                    server.name.toLowerCase().includes(searchQuery.toLowerCase()) ||
                    server.description.toLowerCase().includes(searchQuery.toLowerCase());
                return matchesFilter && matchesSearch;
            });
            
            // Show empty state if no servers match
            if (filteredServers.length === 0) {
                serverGrid.classList.add('hidden');
                emptyState.classList.remove('hidden');
                return;
            }
            
            serverGrid.classList.remove('hidden');
            emptyState.classList.add('hidden');
            
            // Render each server card
            filteredServers.forEach(server => {
                const statusColor = server.status === 'active' ? 'green' : server.status === 'beta' ? 'yellow' : 'red';
                const typeColor = server.type === 'hsbc' ? 'hsbc-red' : 'indigo';
                
                const card = document.createElement('div');
                card.className = 'border border-gray-200 dark:border-dark-border rounded-lg overflow-hidden shadow-sm hover:shadow-md transition-shadow duration-200 bg-white dark:bg-dark-card';
                card.innerHTML = `
                    <div class="p-5">
                        <div class="flex justify-between items-start">
                            <h3 class="text-lg font-semibold mb-2">${server.name}</h3>
                            <span class="text-xs px-2 py-1 rounded-full ${server.type === 'hsbc' ? 'bg-hsbc-light dark:bg-hsbc-red dark:bg-opacity-20 text-hsbc-red' : 'bg-indigo-100 dark:bg-indigo-900 dark:bg-opacity-20 text-indigo-700 dark:text-indigo-300'}">${server.type === 'hsbc' ? 'HSBC' : 'Open Source'}</span>
                        </div>
                        <p class="text-gray-600 dark:text-gray-400 text-sm mb-4">${server.description}</p>
                        <div class="flex items-center text-sm text-gray-500 dark:text-gray-400 mb-4">
                            <span class="inline-block w-2 h-2 rounded-full mr-2 ${statusColor === 'green' ? 'bg-green-500' : statusColor === 'yellow' ? 'bg-yellow-500' : 'bg-red-500'}"></span>
                            <span class="capitalize">${server.status}</span>
                            <span class="mx-2">•</span>
                            <span>v${server.version}</span>
                        </div>
                        <div class="text-xs text-gray-500 dark:text-gray-400">
                            Last updated: ${server.lastUpdated}
                        </div>
                    </div>
                    <div class="border-t border-gray-200 dark:border-dark-border p-4 bg-gray-50 dark:bg-dark-bg flex justify-between items-center">
                        <div class="flex flex-wrap gap-2">
                            ${server.config.capabilities.slice(0, 3).map(cap => 
                                `<span class="px-2 py-1 text-xs bg-gray-200 dark:bg-dark-card text-gray-700 dark:text-gray-300 rounded">${cap}</span>`
                            ).join('')}
                            ${server.config.capabilities.length > 3 ? 
                                `<span class="px-2 py-1 text-xs bg-gray-200 dark:bg-dark-card text-gray-700 dark:text-gray-300 rounded">+${server.config.capabilities.length - 3} more</span>` : 
                                ''}
                        </div>
                        <button data-server-id="${server.id}" class="view-server-btn text-primary hover:text-secondary focus:outline-none text-sm font-medium">
                            View Details
                        </button>
                    </div>
                `;
                
                serverGrid.appendChild(card);
                
                // Add click event listener to view button
                const viewBtn = card.querySelector('.view-server-btn');
                viewBtn.addEventListener('click', () => {
                    showServerDetail(server.id);
                });
            });
        }

        // Show server detail modal
        function showServerDetail(serverId) {
            const server = mcpServers.find(s => s.id === serverId);
            if (!server) return;
            
            // Convert Markdown to HTML for documentation
            const docHtml = marked.parse(server.documentation);
            
            // Render server details
            serverDetailContent.innerHTML = `
                <div class="flex flex-col md:flex-row md:items-start justify-between pb-4 border-b border-gray-200 dark:border-dark-border">
                    <div>
                        <div class="flex items-center">
                            <h2 class="text-2xl font-bold">${server.name}</h2>
                            <span class="ml-3 text-sm px-2 py-1 rounded-full ${server.type === 'hsbc' ? 'bg-hsbc-light dark:bg-hsbc-red dark:bg-opacity-20 text-hsbc-red' : 'bg-indigo-100 dark:bg-indigo-900 dark:bg-opacity-20 text-indigo-700 dark:text-indigo-300'}">${server.type === 'hsbc' ? 'HSBC' : 'Open Source'}</span>
                        </div>
                        <p class="mt-1 text-gray-600 dark:text-gray-400">${server.description}</p>
                    </div>
                    <div class="mt-4 md:mt-0 flex flex-col items-start md:items-end text-sm">
                        <div class="flex items-center">
                            <span class="inline-block w-2 h-2 rounded-full mr-2 ${server.status === 'active' ? 'bg-green-500' : server.status === 'beta' ? 'bg-yellow-500' : 'bg-red-500'}"></span>
                            <span class="capitalize">${server.status}</span>
                        </div>
                        <div class="mt-1">Version: ${server.version}</div>
                        <div class="mt-1">Last updated: ${server.lastUpdated}</div>
                    </div>
                </div>
                
                <div class="mt-6">
                    <h3 class="text-lg font-semibold mb-4">Configuration</h3>
                    <pre><code class="language-json">${JSON.stringify(server.config, null, 2)}</code></pre>
                </div>
                
                <div class="mt-8">
                    <h3 class="text-lg font-semibold mb-4">Documentation</h3>
                    <div class="prose dark:prose-invert max-w-none">${docHtml}</div>
                </div>
            `;
            
            // Show modal
            serverModal.classList.remove('hidden');
            document.body.classList.add('overflow-hidden');
            
            // Highlight code blocks
            Prism.highlightAllUnder(serverDetailContent);
        }

        // Close server detail modal
        function closeModal() {
            serverModal.classList.add('hidden');
            document.body.classList.remove('overflow-hidden');
        }

        // Set current filter and update UI
        function setFilter(filter) {
            currentFilter = filter;
            
            // Update active filter button
            filterBtns.forEach(btn => {
                if (btn.dataset.filter === filter) {
                    btn.classList.add('border-primary', 'text-primary');
                    btn.classList.remove('border-transparent', 'hover:border-gray-300', 'dark:hover:border-gray-700');
                } else {
                    btn.classList.remove('border-primary', 'text-primary');
                    btn.classList.add('border-transparent', 'hover:border-gray-300', 'dark:hover:border-gray-700');
                }
            });
            
            renderServers();
        }

        // Toggle theme between light and dark
        function toggleTheme() {
            document.documentElement.classList.toggle('dark');
        }

        // Initialize the app
        function init() {
            // Render initial servers
            renderServers();
            
            // Add event listeners
            searchInput.addEventListener('input', (e) => {
                searchQuery = e.target.value.trim();
                renderServers();
            });
            
            filterBtns.forEach(btn => {
                btn.addEventListener('click', () => {
                    setFilter(btn.dataset.filter);
                });
            });
            
            closeModalBtn.addEventListener('click', closeModal);
            serverModal.addEventListener('click', (e) => {
                if (e.target === serverModal) {
                    closeModal();
                }
            });
            
            themeToggleBtn.addEventListener('click', toggleTheme);
            
            // Close modal on escape key
            document.addEventListener('keydown', (e) => {
                if (e.key === 'Escape' && !serverModal.classList.contains('hidden')) {
                    closeModal();
                }
            });
        }

        // Start the app
        init();
    </script>
</body>
</html>
```

## Key Features

1. **Server Registry Display**:
   - Shows both HSBC AI Markets Team's MCP servers and open-source ones
   - Clear categorization with visual indicators for each type

2. **Detailed Configuration View**:
   - Displays each server's configuration in a formatted JSON view
   - Shows comprehensive documentation with markdown support

3. **Search and Filtering**:
   - Filter by source (HSBC vs Open-source)
   - Search by server name or description

4. **Responsive Design**:
   - Works on all device sizes
   - Supports both dark and light modes

5. **Modern UI Elements**:
   - Clean card-based layout
   - Modal for detailed server information
   - Status indicators
   - Capability tags

This application successfully separates the tools from the AI Matrix LangGraph repository, providing a clear registry interface for all MCP servers. The documentation for each server is displayed in a readable format with syntax highlighting for code examples.

Would you like me to modify any aspects of this application or would you prefer a different approach for the MCP server registry?

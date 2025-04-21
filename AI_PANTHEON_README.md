*Thinking...*

> I need to write a README for a project called "AI Pantheon" that serves as an MCP server and universal tool registry, based on all the discussions we've had. Let me think about what should be included in this README:
> 
> 1. Introduction to AI Pantheon - what it is and its purpose
> 2. Key features 
> 3. Architecture overview
> 4. Installation and setup instructions
> 5. Usage examples
> 6. API documentation
> 7. Authentication
> 8. UI features
> 9. Integration with AI Matrix and other clients
> 10. Development and contribution guidelines
> 11. License information
> 
> I should incorporate elements from our discussions about:
> - Mounting and combining different MCP servers
> - The UI implementation similar to mcp.so and mastra.ai
> - Authentication mechanisms
> - Dynamic server grid in the UI
> - Integration with langchain-mcp-adapters
> 
> The name "AI Pantheon" is fitting as it suggests a collection of powerful AI tools/entities (like gods in a pantheon), all accessible in one place. I'll make sure to highlight how this platform serves as a central registry for MCP servers from various sources.
> 
> Let me draft a comprehensive README that presents AI Pantheon as a professional, cutting-edge tool for the AI ecosystem.


# AI Pantheon

<img src="docs/images/logo.png" alt="AI Pantheon Logo" width="120" align="right"/>

AI Pantheon is a universal Model Context Protocol (MCP) server registry that provides a centralized hub for hosting, discovering, and accessing AI tools. Built with FastMCP and FastAPI, it allows you to combine tools from multiple sources under a unified interface for seamless agent interaction.

[[PyPI version](https://img.shields.io/badge/pypi-v0.1.0-blue.svg)](https://pypi.org/project/ai-pantheon/)
[[License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[[Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)

## 🔱 Features

- **Universal Tool Registry**: Mount and combine tools from diverse MCP servers (Python, Node.js, etc.)
- **Interactive UI**: Explore available tools with search, filtering, and detailed information
- **Authentication**: Secure your tools with API keys, JWT tokens, or OAuth
- **Client Integration**: Seamless connection with LangChain, LangGraph, and AI Matrix
- **Proxy Support**: Connect to remote MCP servers and APIs through various transport protocols
- **Role-Based Access**: Control who can access which tools
- **Metrics & Analytics**: Track tool usage and performance
- **OpenAPI Integration**: Convert existing APIs to MCP tools automatically

## 🏛️ Architecture

AI Pantheon uses a layered architecture to combine MCP servers from multiple sources:

```
┌─────────────────────────────────────────────────────────┐
│                      FastAPI UI                         │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│                     AI Pantheon Hub                     │
└─┬───────────────┬────────────────┬───────────────────┬──┘
  │               │                │                   │
┌─▼──────┐   ┌────▼─────┐    ┌─────▼─────┐    ┌────────▼────────┐
│ Python │   │ FastMCP  │    │ Node.js   │    │ OpenAPI-derived │
│ MCPs   │   │ Proxies  │    │ MCPs      │    │ MCPs            │
└────────┘   └──────────┘    └───────────┘    └─────────────────┘
```

## 🚀 Installation

```bash
# Install from PyPI
pip install ai-pantheon

# Or install from source
git clone https://github.com/yourusername/ai-pantheon.git
cd ai-pantheon
pip install -e .
```

## 🏁 Quick Start

### Basic Setup

```python
from ai_pantheon import Pantheon
from fastmcp import FastMCP, Client
from fastmcp.client.transports import PythonStdioTransport

# Create the Pantheon hub
pantheon = Pantheon("AI Pantheon")

# Add a simple Python MCP server
calculator = FastMCP("Calculator")

@calculator.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

# Mount the server in the Pantheon
pantheon.mount("calc", calculator)

# Add an external Node.js MCP server (e.g., Sequential Thinking)
sequential_thinking_client = Client(
    transport=PythonStdioTransport(
        command="npx",
        args=["-y", "@modelcontextprotocol/server-sequential-thinking"]
    )
)
thinking_proxy = FastMCP.from_client(sequential_thinking_client, name="Sequential Thinking")
pantheon.mount("thinking", thinking_proxy)

# Run the server with UI
if __name__ == "__main__":
    pantheon.run_with_ui(host="0.0.0.0", port=8000)
```

### Using with AI Matrix

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o")

async with MultiServerMCPClient({
    "pantheon": {
        "url": "http://localhost:8000/mcp/sse",
        "transport": "sse",
        "headers": {
            "X-API-Key": "YOUR_API_KEY"
        }
    }
}) as client:
    agent = create_react_agent(model, client.get_tools())
    response = await agent.ainvoke({"messages": "What tools are available?"})
```

## 🛠️ Advanced Configuration

### Authentication

```python
from ai_pantheon import Pantheon, AuthConfig

# Setup Pantheon with authentication
pantheon = Pantheon(
    "AI Pantheon",
    auth_config=AuthConfig(
        api_key="my_secret_key",
        enable_jwt=True,
        jwt_secret="my_jwt_secret",
        jwt_algorithm="HS256"
    )
)

# Add role-based access control
pantheon.add_role("admin", ["calc/*", "thinking/*"])  # Full access
pantheon.add_role("basic", ["calc/add", "calc/subtract"])  # Limited access

# Assign roles
pantheon.assign_user_role("alice@example.com", "admin")
pantheon.assign_user_role("bob@example.com", "basic")
```

### Adding OpenAPI Services

```python
import httpx
from ai_pantheon import Pantheon
from fastmcp import FastMCP

pantheon = Pantheon("AI Pantheon")

# Create an HTTP client to make requests to the actual API endpoint
weather_http_client = httpx.AsyncClient(
    base_url="https://api.weather.com",
    headers={"Authorization": f"Bearer {WEATHER_API_KEY}"}
)

# Generate MCP server from OpenAPI
with open("weather_openapi.json", "r") as f:
    weather_openapi = json.load(f)

weather_mcp = FastMCP.from_openapi(
    weather_openapi, 
    client=weather_http_client,
    name="Weather API",
    description="Get weather forecasts for locations worldwide"
)

# Mount it with metadata for the UI
pantheon.mount("weather", weather_mcp, 
    tags=["weather", "forecast", "verified"],
    usage_count=3457
)
```

## 📊 UI Features

AI Pantheon includes a modern, responsive web interface:

- **Server Grid**: Visual display of all available MCP servers
- **Search**: Find tools by name, description, or tags
- **Filtering**: Filter by server type, tags, or popularity
- **Detailed Views**: Explore tool schemas, parameters, and examples
- **Try It Now**: Test tools directly from the browser
- **Authentication**: User login and API key management

<div align="center">
  <img src="docs/images/ui-screenshot.png" alt="AI Pantheon UI" width="80%"/>
</div>

## 📦 Included Servers

AI Pantheon comes with connectors for popular MCP servers:

- **Sequential Thinking**: Step-by-step reasoning for complex problems
- **Web Search**: Internet search capabilities for agents
- **File Operations**: File system access with proper permissions
- **Data Analysis**: Tools for working with tabular data
- **Image Generation**: Create and manipulate images
- **And many more...**

## 🔌 API Endpoints

### Server Management

- `GET /api/servers`: List all available servers
- `GET /api/servers/{server_id}`: Get server details
- `GET /api/servers/{server_id}/tools`: List tools for a server
- `GET /api/servers/{server_id}/resources`: List resources for a server

### Tool Execution

- `POST /mcp/sse`: Primary MCP endpoint (SSE transport)
- `POST /api/execute/{server_id}/{tool_name}`: Execute a tool directly via REST

### Authentication

- `POST /api/auth/token`: Get JWT token
- `POST /api/auth/key`: Generate API key
- `GET /api/auth/me`: Get current user information

## 🧑‍💻 Development

```bash
# Clone the repository
git clone https://github.com/yourusername/ai-pantheon.git
cd ai-pantheon

# Create a virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -e ".[dev]"

# Run tests
pytest

# Start development server
uvicorn ai_pantheon.main:app --reload
```

## 🤝 Contributing

Contributions are welcome! Please check out our [contributing guidelines](CONTRIBUTING.md) for more information.

## 📄 License

AI Pantheon is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- [Model Context Protocol](https://github.com/anthropics/model-context-protocol) team for the MCP specification
- [FastMCP](https://github.com/jlowin/fastmcp) for the Pythonic MCP server implementation
- [LangChain MCP Adapters](https://github.com/langchain-ai/langchain-mcp-adapters) for client integration

---

<div align="center">
  <p>AI Pantheon: Unifying the tools of AI in one accessible place</p>
</div>

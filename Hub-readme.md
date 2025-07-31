# MCP Hub Registry

A comprehensive Model Context Protocol (MCP) server hub that hosts and manages multiple MCP servers with support for various package managers, transport protocols, and graceful failure handling.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Server Types & Loaders](#server-types--loaders)
- [Transport Protocols](#transport-protocols)
- [Package Managers](#package-managers)
- [API Reference](#api-reference)
- [Development](#development)
- [Examples](#examples)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

## Overview

MCP Hub Registry is a production-ready server that acts as a centralized hub for multiple MCP (Model Context Protocol) servers. It provides a unified interface to access various AI tools, APIs, and services through a single endpoint, with automatic load balancing, health monitoring, and transport protocol bridging.

### Key Concepts

- **Hub Server**: The main application that hosts multiple MCP servers
- **Server Entries**: Individual MCP server configurations
- **Loaders**: Components that initialize different types of servers
- **Transport Bridging**: Ability to expose servers using different protocols
- **Registry**: Central tracking system for server status and capabilities

## Features

### 🚀 Multi-Server Management
- Host multiple MCP servers under a single hub
- Concurrent server loading for fast startup
- Graceful failure handling with fallback responses
- Real-time health monitoring and status reporting

### 📦 Package Manager Support
- **UVX**: Python packages via uvx with version isolation
- **NPM**: Node.js packages from npm registry
- **PyPI**: Python packages via pip
- **Docker**: Containerized MCP servers (planned)

### 🌐 Transport Protocol Support
- **stdio**: Direct process communication
- **SSE**: Server-Sent Events for web clients
- **streamable-http**: HTTP streaming protocol
- **HTTP**: Standard HTTP REST API

### 🔌 Integration Options
- **OpenAPI**: Automatic conversion from OpenAPI specs using FastMCP
- **Python Modules**: Direct integration of Python-based servers
- **Remote Proxies**: Connect to external MCP servers
- **Composite Configs**: Multi-server MCPConfig format support

### 🛡️ Production Features
- Environment variable substitution with `${VAR}` syntax
- Configurable retry logic with exponential backoff
- Resource cleanup and proper shutdown handling
- Comprehensive logging and monitoring
- Authentication and authorization support

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    MCP Hub Registry                     │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Health    │  │  Registry   │  │  Management │    │
│  │   Monitor   │  │   API       │  │     API     │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
├─────────────────────────────────────────────────────────┤
│                 Server Loaders                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │ OpenAPI  │ │   UVX    │ │   NPM    │ │  Module  │  │
│  │ Loader   │ │ Loader   │ │ Loader   │ │ Loader   │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
├─────────────────────────────────────────────────────────┤
│                FastMCP Proxy Layer                     │
│           (Transport Bridging & Session Management)    │
├─────────────────────────────────────────────────────────┤
│                Individual MCP Servers                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │ GitHub   │ │Atlassian │ │ Weather  │ │ Custom   │  │
│  │   API    │ │   MCP    │ │   API    │ │  Tools   │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Core Components

1. **MCPHub**: Main server application managing routes and lifecycle
2. **MCPRegistry**: Central registry tracking server status and capabilities
3. **ServerLoaders**: Factory pattern for different server types
4. **LoadResult**: Consistent return type with metadata and cleanup functions
5. **ProxyClient**: FastMCP's native proxy for transport bridging

## Installation

### Prerequisites

- Python 3.9+
- Node.js 16+ (for NPM servers)
- uvx (for Python package isolation)

### Install from Source

```bash
# Clone the repository
git clone https://github.com/yourusername/mcp-hub-registry.git
cd mcp-hub-registry

# Install dependencies
pip install -e .

# Or install specific extras
pip install -e ".[dev,uvx,npm]"
```

### Install Dependencies

```bash
# Install uvx for Python package management
pip install uvx

# Install npm globally if not already installed
# (for NPM-based MCP servers)
npm install -g npm

# Verify installations
uvx --version
npm --version
```

## Quick Start

### 1. Initialize Configuration

```bash
# Create default configuration files
mcp-hub init

# This creates:
# ├── data/
# │   ├── config.yaml      # Hub configuration
# │   └── servers.yaml     # Server definitions
```

### 2. Configure Environment Variables

```bash
# Create .env file or export variables
export GITHUB_TOKEN=your_github_token
export JIRA_URL=https://company.atlassian.net
export JIRA_USERNAME=your_username
export JIRA_API_TOKEN=your_api_token
```

### 3. Start the Hub

```bash
# Start with default configuration
mcp-hub run

# Or specify custom config files
mcp-hub run --config custom-config.yaml --servers custom-servers.yaml

# Run with specific environment
mcp-hub run-env --env production
```

### 4. Verify Installation

```bash
# Check hub status
curl http://localhost:8000/health

# List available servers
curl http://localhost:8000/servers

# Access a specific server
curl http://localhost:8000/github/repos/octocat/Hello-World
```

## Configuration

### Hub Configuration (`config.yaml`)

```yaml
# Hub server settings
hub:
  name: "MCP Hub Registry"
  host: "0.0.0.0"
  port: 8000
  log_level: "info"
  health_check_enabled: true
  timeout: 30.0
```

### Server Configuration (`servers.yaml`)

The servers configuration defines individual MCP servers to be hosted:

```yaml
servers:
  # OpenAPI-based server
  - id: "github-api"
    name: "GitHub API Server"
    description: "Access GitHub API via MCP"
    type: "openapi"
    openapi_url: "https://api.github.com/openapi.yaml"
    base_url: "https://api.github.com"
    auth_type: "bearer"
    auth_token: "${GITHUB_TOKEN}"
    mount_path: "/github"
    enabled: true
    retry_attempts: 3
    retry_delay: 2.0
    route_maps:
      - methods: ["GET"]
        pattern: "^/repos/[^/]+/[^/]+$"
        mcp_type: "RESOURCE_TEMPLATE"
        mcp_tags: ["repository"]
```

### Environment Variables

Environment variables can be used throughout configuration files:

```yaml
# Supported formats:
auth_token: "${GITHUB_TOKEN}"                    # Required variable
base_url: "${API_URL:-https://api.default.com}"  # With default value
timeout: "${TIMEOUT:30}"                         # Numeric with default
```

## Server Types & Loaders

### OpenAPI Loader

Automatically converts OpenAPI specifications to MCP servers using FastMCP:

```yaml
- id: "petstore"
  name: "Pet Store API"
  type: "openapi"
  openapi_url: "https://petstore.swagger.io/v2/swagger.json"
  base_url: "https://petstore.swagger.io/v2"
  mount_path: "/petstore"
  route_maps:
    - methods: ["GET"]
      pattern: "^/pet/.*"
      mcp_type: "RESOURCE_TEMPLATE"
      mcp_tags: ["pets", "read"]
    - methods: ["POST", "PUT", "DELETE"]
      pattern: ".*"
      mcp_type: "TOOL"
      mcp_tags: ["pets", "write"]
```

**Features:**
- Automatic route mapping to MCP tools/resources
- Authentication handling (Bearer, API Key, Custom)
- Configurable timeout and retry logic
- Route filtering and tagging

### UVX Loader

Runs Python packages in isolated environments using uvx:

```yaml
- id: "mcp-atlassian"
  name: "Atlassian MCP Server"
  mount_path: "/atlassian"
  packages:
    - registry_name: "uvx"
      name: "mcp-atlassian"
      version: "0.11.9"
      transport:
        type: "stdio"  # or "sse", "streamable-http"
        port: 9000     # for HTTP transports
      uvx_options:
        - "--python"
        - "3.11"
      environment_variables:
        - name: "JIRA_URL"
          description: "Jira instance URL"
          required: true
        - name: "JIRA_API_TOKEN"
          description: "Jira API token"
          required: true
```

**Features:**
- Python version isolation
- Multiple transport protocols
- Environment variable injection
- Package-specific arguments

### NPM Loader

Runs Node.js packages from npm registry:

```yaml
- id: "sqlite-server"
  name: "SQLite MCP Server"
  mount_path: "/sqlite"
  packages:
    - registry_name: "npm"
      name: "@modelcontextprotocol/server-sqlite"
      version: "0.1.0"
      transport:
        type: "stdio"
      package_arguments:
        - description: "Database path"
          value: "./data/app.db"
          type: "positional"
        - name: "readonly"
          description: "Read-only mode"
          type: "flag"
```

### Module Loader

Directly loads Python modules:

```yaml
- id: "custom-tools"
  name: "Custom Tools Server"
  type: "module"
  module_path: "my_servers.tools"
  factory_function: "create_server"
  mount_path: "/tools"
```

### Composite Loader

Handles multi-server configurations using MCPConfig format:

```yaml
- id: "ai-services"
  name: "AI Services Hub"
  mount_path: "/ai"
  mcp_config:
    mcpServers:
      openai:
        url: "https://openai-mcp.example.com/sse"
        transport: "sse"
      anthropic:
        command: "python"
        args: ["./anthropic_server.py"]
      local:
        url: "http://localhost:8000/mcp"
```

### Remote Proxy Loader

Connects to external MCP servers:

```yaml
- id: "remote-weather"
  name: "Remote Weather Service"
  mount_path: "/weather"
  remote_proxy:
    url: "https://weather-api.example.com/mcp/sse"
    transport: "sse"
    headers:
      Authorization: "Bearer ${WEATHER_API_TOKEN}"
```

## Transport Protocols

MCP Hub Registry supports multiple transport protocols for maximum flexibility:

### stdio Transport

Direct process communication for highest performance:

```yaml
transport:
  type: "stdio"
```

**Use Cases:**
- Local development
- High-performance scenarios
- Simple tool integrations

### SSE Transport

Server-Sent Events for web applications:

```yaml
transport:
  type: "sse"
  port: 9000
  endpoint: "/sse"
  headers:
    Access-Control-Allow-Origin: "*"
```

**Use Cases:**
- Web applications
- Real-time updates
- Browser-based clients

### Streamable HTTP Transport

HTTP streaming for robust network communication:

```yaml
transport:
  type: "streamable-http"
  port: 9001
  endpoint: "/mcp"
  headers:
    Authorization: "Bearer ${API_TOKEN}"
```

**Use Cases:**
- Microservices
- Cloud deployments
- Load-balanced environments

## Package Managers

### UVX Package Manager

UVX provides isolated Python environments for each package:

```yaml
packages:
  - registry_name: "uvx"
    name: "mcp-atlassian"
    version: "0.11.9"
    uvx_options:
      - "--python"
      - "3.11"
      - "--with"
      - "httpx"
    environment_variables:
      - name: "JIRA_URL"
        required: true
      - name: "LOG_LEVEL"
        default: "info"
```

**Commands Generated:**
```bash
# Basic uvx command
uvx mcp-atlassian==0.11.9

# With options
uvx --python 3.11 --with httpx mcp-atlassian==0.11.9

# With transport
uvx mcp-atlassian --transport sse --port 9000
```

### NPM Package Manager

NPM packages are installed globally and executed with npx:

```yaml
packages:
  - registry_name: "npm"
    name: "@modelcontextprotocol/server-sqlite"
    version: "0.1.0"
    package_arguments:
      - description: "Database file"
        value: "./data.db"
        type: "positional"
```

**Commands Generated:**
```bash
# Install
npm install -g @modelcontextprotocol/server-sqlite@0.1.0

# Run
npx @modelcontextprotocol/server-sqlite ./data.db
```

### PyPI Package Manager

Standard pip installation with Python execution:

```yaml
packages:
  - registry_name: "pypi"
    name: "my-mcp-server"
    version: "1.0.0"
    entry_point: "my_mcp_server.main"
```

**Commands Generated:**
```bash
# Install
pip install my-mcp-server==1.0.0

# Run
python -m my_mcp_server.main
```

## API Reference

### Hub Management Endpoints

#### GET `/`
Root endpoint with hub information:

```json
{
  "name": "MCP Hub Registry",
  "endpoints": {
    "health": "/health",
    "servers": "/servers"
  },
  "servers": ["github-api", "atlassian", "weather"]
}
```

#### GET `/health`
Comprehensive health check:

```json
{
  "status": "healthy",
  "hub": {
    "name": "MCP Hub Registry",
    "healthy_servers": 3,
    "total_servers": 4
  },
  "servers": {
    "github-api": {
      "id": "github-api",
      "name": "GitHub API Server",
      "status": "healthy",
      "mount_path": "/github",
      "loaded_at": "2025-01-31T10:30:00Z",
      "capabilities": {
        "tools": 15,
        "resources": 8,
        "prompts": 3
      }
    }
  }
}
```

#### GET `/servers`
List active servers:

```json
{
  "hub": "MCP Hub Registry",
  "servers": [
    {
      "id": "github-api",
      "name": "GitHub API Server",
      "mount_path": "/github",
      "status": "healthy",
      "capabilities": {
        "tools": 15,
        "resources": 8,
        "prompts": 3
      },
      "url": "http://localhost:8000/github"
    }
  ]
}
```

### Server Endpoints

Each server is mounted at its configured path:

#### GET `/github/*`
Access GitHub API server:
```bash
curl http://localhost:8000/github/repos/octocat/Hello-World
```

#### POST `/atlassian/*`
Access Atlassian server:
```bash
curl -X POST http://localhost:8000/atlassian/issue \
  -H "Content-Type: application/json" \
  -d '{"key": "PROJ-123"}'
```

### Error Responses

Failed servers return structured error responses:

```json
{
  "error": "Server unavailable",
  "server_id": "failed-server",
  "server_name": "Failed Server",
  "message": "Connection timeout after 30s",
  "status": "failed"
}
```

## Development

### Project Structure

```
mcp-hub-registry/
├── mcp_hub/                    # Main package
│   ├── __init__.py
│   ├── cli.py                  # Command-line interface
│   ├── config.py               # Configuration management
│   ├── server.py               # Main hub server
│   ├── registry.py             # Server registry
│   ├── loaders/                # Server loaders
│   │   ├── __init__.py
│   │   ├── base.py             # Base loader interface
│   │   ├── factory.py          # Loader factory
│   │   ├── openapi.py          # OpenAPI loader
│   │   ├── uvx.py              # UVX loader
│   │   ├── npm.py              # NPM loader
│   │   ├── module.py           # Module loader
│   │   ├── composite.py        # Composite loader
│   │   └── remote.py           # Remote proxy loader
│   └── models/                 # Data models
│       ├── __init__.py
│       └── server.py           # Server configuration models
├── data/                       # Configuration files
│   ├── config.yaml
│   └── servers.yaml
├── tests/                      # Test suite
├── examples/                   # Example configurations
├── requirements.txt
├── setup.py
└── README.md
```

### Setting Up Development Environment

```bash
# Clone repository
git clone https://github.com/yourusername/mcp-hub-registry.git
cd mcp-hub-registry

# Create virtual environment
python -m venv venv
source venv/bin/activate  # or `venv\Scripts\activate` on Windows

# Install in development mode
pip install -e ".[dev]"

# Install pre-commit hooks
pre-commit install

# Run tests
pytest

# Run with coverage
pytest --cov=mcp_hub
```

### Creating Custom Loaders

Implement the `ServerLoader` interface:

```python
from mcp_hub.loaders.base import ServerLoader, LoadResult
from mcp_hub.config import ServerConfig

class CustomLoader(ServerLoader):
    async def load(self, config: ServerConfig) -> LoadResult:
        try:
            # Your loading logic here
            mcp_instance = await create_your_server(config)
            app = mcp_instance.get_asgi_app()
            
            return LoadResult(
                success=True,
                mcp_instance=mcp_instance,
                app=app,
                connection_info={"type": "custom"},
                loaded_at=datetime.utcnow()
            )
        except Exception as e:
            return LoadResult(
                success=False,
                error=str(e)
            )
```

Register in `loaders/factory.py`:

```python
def get_loader_for_config(config: ServerConfig) -> Optional[ServerLoader]:
    if config.type == "custom":
        return CustomLoader()
    # ... existing logic
```

### Testing

Run the test suite:

```bash
# Run all tests
pytest

# Run specific test file
pytest tests/test_loaders.py

# Run with coverage
pytest --cov=mcp_hub --cov-report=html

# Run integration tests
pytest tests/integration/
```

### Building Documentation

```bash
# Install docs dependencies
pip install -e ".[docs]"

# Build documentation
cd docs
make html

# Serve documentation
python -m http.server 8080 -d _build/html
```

## Examples

### Example 1: GitHub API Integration

```yaml
# servers.yaml
servers:
  - id: "github"
    name: "GitHub API"
    type: "openapi"
    openapi_url: "https://api.github.com/openapi.yaml"
    base_url: "https://api.github.com"
    auth_type: "bearer"
    auth_token: "${GITHUB_TOKEN}"
    mount_path: "/github"
    route_maps:
      - methods: ["GET"]
        pattern: "^/user"
        mcp_type: "RESOURCE"
        mcp_tags: ["user", "profile"]
      - methods: ["GET"]
        pattern: "^/repos/.+/.+"
        mcp_type: "RESOURCE_TEMPLATE"
        mcp_tags: ["repository"]
```

### Example 2: Multi-Transport Atlassian Server

```yaml
servers:
  # stdio version for development
  - id: "atlassian-dev"
    name: "Atlassian (Development)"
    mount_path: "/atlassian-dev"
    enabled: true
    packages:
      - registry_name: "uvx"
        name: "mcp-atlassian"
        transport:
          type: "stdio"
        environment_variables:
          - name: "JIRA_URL"
            default: "http://localhost:8080"
          - name: "LOG_LEVEL"
            default: "debug"

  # SSE version for production
  - id: "atlassian-prod"
    name: "Atlassian (Production)"
    mount_path: "/atlassian"
    enabled: true
    packages:
      - registry_name: "uvx"
        name: "mcp-atlassian"
        version: "0.11.9"
        transport:
          type: "sse"
          port: 9001
          headers:
            X-Atlassian-Token: "${ATLASSIAN_TOKEN}"
        environment_variables:
          - name: "JIRA_URL"
            default: "${JIRA_URL}"
          - name: "CONFLUENCE_URL"
            default: "${CONFLUENCE_URL}"
```

### Example 3: Development Environment

```yaml
# config.development.yaml
hub:
  name: "MCP Hub (Development)"
  host: "127.0.0.1"
  port: 8000
  log_level: "debug"
  health_check_enabled: true

# servers.development.yaml
servers:
  # Local SQLite server
  - id: "sqlite-local"
    name: "Local SQLite"
    mount_path: "/sqlite"
    packages:
      - registry_name: "npm"
        name: "@modelcontextprotocol/server-sqlite"
        package_arguments:
          - value: "./dev-data.db"
            type: "positional"

  # Local file system server
  - id: "filesystem"
    name: "File System"
    type: "module"
    module_path: "mcp_hub.examples.filesystem"
    mount_path: "/files"

  # Mock external API
  - id: "mock-api"
    name: "Mock External API"
    type: "openapi"
    openapi_url: "file://./examples/mock-api.yaml"
    base_url: "http://localhost:3000"
    mount_path: "/mock"
```

### Example 4: Production Configuration

```yaml
# config.production.yaml
hub:
  name: "MCP Hub (Production)"
  host: "0.0.0.0"
  port: 8000
  log_level: "info"
  health_check_enabled: true
  timeout: 60.0

# servers.production.yaml
servers:
  # Production GitHub integration
  - id: "github"
    name: "GitHub API"
    type: "openapi"
    openapi_url: "https://api.github.com/openapi.yaml"
    base_url: "https://api.github.com"
    auth_type: "bearer"
    auth_token: "${GITHUB_TOKEN}"
    mount_path: "/github"
    retry_attempts: 5
    retry_delay: 3.0

  # Production Atlassian with OAuth
  - id: "atlassian"
    name: "Atlassian Cloud"
    mount_path: "/atlassian"
    packages:
      - registry_name: "uvx"
        name: "mcp-atlassian"
        version: "0.11.9"
        transport:
          type: "streamable-http"
          port: 9002
          headers:
            Authorization: "Bearer ${ATLASSIAN_OAUTH_TOKEN}"
            X-Atlassian-Cloud-Id: "${ATLASSIAN_CLOUD_ID}"
        uvx_options:
          - "--python"
          - "3.11"
        environment_variables:
          - name: "ATLASSIAN_OAUTH_ENABLE"
            default: "true"
          - name: "JIRA_URL"
            required: true
          - name: "CONFLUENCE_URL"
            required: true

  # External weather service
  - id: "weather"
    name: "Weather API"
    mount_path: "/weather"
    remote_proxy:
      url: "https://weather-mcp.example.com/sse"
      transport: "sse"
      headers:
        X-API-Key: "${WEATHER_API_KEY}"
```

## Troubleshooting

### Common Issues

#### Server Fails to Start

**Symptoms:** Server shows as "failed" in health check

**Solutions:**
1. Check logs for specific error messages
2. Verify environment variables are set correctly
3. Ensure package is installed and accessible
4. Check network connectivity for remote servers

```bash
# Check detailed logs
mcp-hub run --log-level debug

# Test individual server
uvx mcp-atlassian --help

# Verify environment
echo $JIRA_URL
```

#### Port Already in Use

**Symptoms:** HTTP transport servers fail with port binding errors

**Solutions:**
1. Change port in server configuration
2. Stop conflicting services
3. Use random port assignment

```yaml
transport:
  type: "sse"
  port: 0  # Use random available port
```

#### Package Installation Failures

**Symptoms:** UVX or NPM packages fail to install

**Solutions:**
1. Update package managers
2. Check network connectivity
3. Use specific versions instead of "latest"
4. Clear package caches

```bash
# Update package managers
pip install --upgrade uvx
npm update -g npm

# Clear caches
uvx cache clear
npm cache clean --force
```

#### Authentication Issues

**Symptoms:** OpenAPI servers return 401/403 errors

**Solutions:**
1. Verify API tokens are valid
2. Check token permissions
3. Ensure correct auth_type setting

```yaml
# Debug authentication
auth_type: "bearer"           # or "api_key"
auth_token: "${API_TOKEN}"    # Verify variable is set
auth_header: "Authorization"  # Default, or custom header
```

#### Memory and Performance Issues

**Symptoms:** High memory usage or slow responses

**Solutions:**
1. Limit concurrent servers
2. Increase timeout values
3. Use stdio transport for better performance
4. Monitor resource usage

```bash
# Monitor hub resources
htop
# or
docker stats mcp-hub  # if running in Docker
```

### Debugging

#### Enable Debug Logging

```yaml
# config.yaml
hub:
  log_level: "debug"
```

#### Check Individual Server Status

```bash
# Get detailed server info
curl http://localhost:8000/health | jq '.servers["github-api"]'

# Test server directly
curl -v http://localhost:8000/github/user
```

#### Inspect Server Processes

```bash
# List running processes
ps aux | grep uvx
ps aux | grep npx

# Check port usage
netstat -tulpn | grep :9000
```

#### Validate Configuration

```bash
# Check YAML syntax
python -c "import yaml; yaml.safe_load(open('data/servers.yaml'))"

# Validate environment variables
python -c "
import os
from mcp_hub.config import substitute_env_vars
config = {'test': '\${MISSING_VAR}'}
print(substitute_env_vars(config))
"
```

### Performance Tuning

#### Optimize Server Loading

```yaml
# Reduce retry attempts for faster failures
retry_attempts: 2
retry_delay: 1.0

# Use stdio for local servers
transport:
  type: "stdio"

# Disable health checks for development
health_check_enabled: false
```

#### Scale for Production

```yaml
# Increase timeouts for external services
timeout: 120.0

# Use HTTP transports for better load balancing
transport:
  type: "streamable-http"
  port: 9000

# Enable connection pooling
keep_alive: true
```

## Contributing

We welcome contributions! Please see our [Contributing Guide](CONTRIBUTING.md) for details.

### Development Workflow

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests
5. Run the test suite
6. Submit a pull request

### Code Style

We use:
- **Black** for code formatting
- **isort** for import sorting
- **mypy** for type checking
- **pytest** for testing

```bash
# Format code
black mcp_hub/
isort mcp_hub/

# Type checking
mypy mcp_hub/

# Run tests
pytest
```

### Submitting Issues

When submitting issues, please include:
- MCP Hub version
- Python version
- Operating system
- Complete error messages
- Configuration files (sanitized)
- Steps to reproduce

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) file for details.

## Acknowledgments

- [FastMCP](https://github.com/jlowin/fastmcp) for the excellent MCP implementation
- [Model Context Protocol](https://modelcontextprotocol.io/) specification
- [Starlette](https://www.starlette.io/) for the ASGI framework
- [Pydantic](https://pydantic.dev/) for data validation

---

For more information, visit our [documentation](https://mcp-hub-registry.readthedocs.io/) or join our [Discord community](https://discord.gg/mcp-hub).

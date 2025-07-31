*Thinking...*

> The user wants me to read the chat from Poe and create an MCP registry based on FastMCP. Looking at the chat, they want:
> 
> 1. A main MCP hub server that hosts multiple MCP servers
> 2. Some servers are transformed from OpenAPI using FastMCP
> 3. Handle failures gracefully when individual servers fail to start
> 4. Use configuration files (YAML format)
> 5. Based on FastMCP patterns
> 
> The conversation shows a comprehensive implementation with:
> - Project structure with modules
> - Configuration management (YAML)
> - Server loaders (OpenAPI, module, NPM, Docker)
> - Database layer (MongoDB/memory)
> - Registry API endpoints
> - Health monitoring
> - CLI interface
> 
> The key requirements are:
> - Use FastMCP's `from_openapi()` for OpenAPI servers
> - Graceful failure handling with fallback apps
> - Registry-style API like MCP registry
> - Support for multiple server types
> - Seed data loading
> - Authentication support
> 
> I'll create a simplified but complete implementation focused on the core requirements.

Based on the Poe chat, I'll create an MCP registry hub that hosts multiple MCP servers with graceful failure handling. Here's a complete implementation using FastMCP:

## Project Structure

```
mcp-hub/
├── mcp_hub/
│   ├── __init__.py
│   ├── config.py
│   ├── registry.py
│   ├── loaders.py
│   ├── server.py
│   └── cli.py
├── config.yaml
├── servers.yaml
├── requirements.txt
└── README.md
```

## Implementation

### requirements.txt
```txt
fastmcp>=0.2.0
httpx>=0.25.0
pydantic>=2.0.0
starlette>=0.37.0
uvicorn>=0.27.0
click>=8.1.0
pyyaml>=6.0
```

### mcp_hub/__init__.py
```python
from .server import MCPHub
from .config import Config

__version__ = "1.0.0"
__all__ = ["MCPHub", "Config"]
```

### mcp_hub/config.py
```python
import os
import re
import yaml
from pathlib import Path
from typing import Any, Dict, List, Optional
from pydantic import BaseModel, Field


def substitute_env_vars(data: Any) -> Any:
    """Recursively substitute ${VAR} with environment variables"""
    if isinstance(data, dict):
        return {k: substitute_env_vars(v) for k, v in data.items()}
    elif isinstance(data, list):
        return [substitute_env_vars(item) for item in data]
    elif isinstance(data, str):
        def replacer(match):
            var_name = match.group(1)
            return os.environ.get(var_name, match.group(0))
        return re.sub(r'\$\{([^}]+)\}', replacer, data)
    else:
        return data


class ServerConfig(BaseModel):
    id: str
    name: str
    description: str = ""
    type: str = "openapi"  # openapi, module, proxy
    enabled: bool = True
    
    # OpenAPI specific
    openapi_url: Optional[str] = None
    base_url: Optional[str] = None
    route_maps: Optional[List[Dict[str, Any]]] = None
    
    # Module specific
    module_path: Optional[str] = None
    factory_function: Optional[str] = "create_server"
    
    # Auth config
    auth_type: Optional[str] = None
    auth_token: Optional[str] = None
    auth_header: Optional[str] = "Authorization"
    
    # Common
    mount_path: Optional[str] = None
    retry_attempts: int = 3
    retry_delay: float = 2.0
    
    def __init__(self, **data):
        data = substitute_env_vars(data)
        super().__init__(**data)
        if not self.mount_path:
            self.mount_path = f"/{self.id}"


class HubConfig(BaseModel):
    name: str = "MCP Hub"
    host: str = "0.0.0.0"
    port: int = 8000
    log_level: str = "info"
    health_check_enabled: bool = True
    timeout: float = 30.0


class Config(BaseModel):
    hub: HubConfig = Field(default_factory=HubConfig)
    servers: List[ServerConfig] = Field(default_factory=list)
    
    @classmethod
    def from_file(cls, config_path: str, servers_path: Optional[str] = None) -> "Config":
        """Load configuration from YAML files"""
        # Load hub config
        with open(config_path, 'r') as f:
            config_data = yaml.safe_load(f)
        
        # Load servers config
        if servers_path and Path(servers_path).exists():
            with open(servers_path, 'r') as f:
                servers_data = yaml.safe_load(f)
                config_data['servers'] = servers_data.get('servers', [])
        
        # Substitute environment variables
        config_data = substitute_env_vars(config_data)
        
        return cls(**config_data)
```

### mcp_hub/registry.py
```python
import logging
from datetime import datetime
from typing import Dict, Optional
from enum import Enum
from dataclasses import dataclass, field

from fastmcp import FastMCP
from starlette.applications import Starlette


class ServerStatus(Enum):
    LOADING = "loading"
    HEALTHY = "healthy"
    FAILED = "failed"
    DISABLED = "disabled"


@dataclass
class ServerInfo:
    id: str
    name: str
    status: ServerStatus
    mount_path: str
    error: Optional[str] = None
    mcp_instance: Optional[FastMCP] = None
    app: Optional[Starlette] = None
    loaded_at: Optional[datetime] = None
    capabilities: Dict = field(default_factory=dict)
    
    def to_dict(self) -> Dict:
        return {
            "id": self.id,
            "name": self.name,
            "status": self.status.value,
            "mount_path": self.mount_path,
            "error": self.error,
            "loaded_at": self.loaded_at.isoformat() if self.loaded_at else None,
            "capabilities": self.capabilities
        }


class MCPRegistry:
    """Registry for tracking MCP servers in the hub"""
    
    def __init__(self):
        self.servers: Dict[str, ServerInfo] = {}
        self.logger = logging.getLogger(__name__)
    
    def register(self, server_info: ServerInfo):
        """Register or update a server"""
        self.servers[server_info.id] = server_info
        self.logger.info(f"Registered server '{server_info.name}' with status: {server_info.status.value}")
    
    def get(self, server_id: str) -> Optional[ServerInfo]:
        """Get server info by ID"""
        return self.servers.get(server_id)
    
    def get_all(self) -> Dict[str, ServerInfo]:
        """Get all servers"""
        return self.servers.copy()
    
    def get_healthy(self) -> Dict[str, ServerInfo]:
        """Get healthy servers"""
        return {id: info for id, info in self.servers.items() 
                if info.status == ServerStatus.HEALTHY}
    
    def update_capabilities(self, server_id: str, mcp: FastMCP):
        """Update server capabilities"""
        if server_id in self.servers:
            try:
                self.servers[server_id].capabilities = {
                    "tools": len(list(mcp.list_tools())),
                    "resources": len(list(mcp.list_resources())),
                    "prompts": len(list(mcp.list_prompts()))
                }
            except Exception as e:
                self.logger.error(f"Error getting capabilities for {server_id}: {e}")
```

### mcp_hub/loaders.py
```python
import asyncio
import importlib
import logging
from typing import Optional, Tuple

import httpx
from fastmcp import FastMCP
from fastmcp.server.openapi import RouteMap, MCPType
from starlette.applications import Starlette
from starlette.responses import JSONResponse
from starlette.routing import Route

from .config import ServerConfig
from .registry import ServerStatus


logger = logging.getLogger(__name__)


async def load_openapi_server(
    config: ServerConfig,
    timeout: float = 30.0
) -> Tuple[Optional[FastMCP], Optional[Starlette], Optional[str]]:
    """Load an MCP server from OpenAPI specification using FastMCP"""
    try:
        logger.info(f"Loading OpenAPI server '{config.id}' from {config.openapi_url}")
        
        # Create HTTP client with auth
        headers = {}
        if config.auth_type == "bearer" and config.auth_token:
            headers[config.auth_header] = f"Bearer {config.auth_token}"
        elif config.auth_type == "api_key" and config.auth_token:
            headers[config.auth_header] = config.auth_token
        
        client = httpx.AsyncClient(
            base_url=config.base_url or "",
            headers=headers,
            timeout=timeout
        )
        
        # Fetch OpenAPI spec
        async with asyncio.timeout(timeout):
            response = await client.get(config.openapi_url)
            response.raise_for_status()
            openapi_spec = response.json()
        
        # Parse route maps if provided
        route_maps = None
        if config.route_maps:
            route_maps = []
            for rm in config.route_maps:
                route_map = RouteMap(
                    methods=rm.get("methods", ["*"]),
                    pattern=rm.get("pattern", r".*"),
                    tags=set(rm.get("tags", [])),
                    mcp_type=MCPType[rm.get("mcp_type", "TOOL")],
                    mcp_tags=set(rm.get("mcp_tags", []))
                )
                route_maps.append(route_map)
        
        # Create MCP server using FastMCP
        mcp = await FastMCP.from_openapi(
            openapi_spec=openapi_spec,
            client=client,
            name=config.name,
            route_maps=route_maps
        )
        
        # Get ASGI app
        app = mcp.get_asgi_app()
        
        logger.info(f"Successfully loaded OpenAPI server '{config.id}'")
        return mcp, app, None
        
    except asyncio.TimeoutError:
        error = f"Timeout loading OpenAPI spec after {timeout}s"
        logger.error(f"Server '{config.id}': {error}")
        return None, None, error
    except Exception as e:
        error = f"Failed to load OpenAPI server: {str(e)}"
        logger.error(f"Server '{config.id}': {error}")
        return None, None, error
    finally:
        if 'client' in locals():
            await client.aclose()


async def load_module_server(
    config: ServerConfig
) -> Tuple[Optional[FastMCP], Optional[Starlette], Optional[str]]:
    """Load a Python module-based MCP server"""
    try:
        logger.info(f"Loading module server '{config.id}' from {config.module_path}")
        
        module = importlib.import_module(config.module_path)
        factory = getattr(module, config.factory_function)
        
        if asyncio.iscoroutinefunction(factory):
            mcp = await factory()
        else:
            mcp = factory()
        
        if not isinstance(mcp, FastMCP):
            raise TypeError(f"Factory must return FastMCP instance, got {type(mcp)}")
        
        app = mcp.get_asgi_app()
        
        logger.info(f"Successfully loaded module server '{config.id}'")
        return mcp, app, None
        
    except Exception as e:
        error = f"Failed to load module server: {str(e)}"
        logger.error(f"Server '{config.id}': {error}")
        return None, None, error


def create_fallback_app(server_id: str, server_name: str, error: str) -> Starlette:
    """Create a fallback Starlette app for failed servers"""
    
    async def error_endpoint(request):
        return JSONResponse({
            "error": "Server unavailable",
            "server_id": server_id,
            "server_name": server_name,
            "message": error,
            "status": "failed"
        }, status_code=503)
    
    routes = [
        Route("/{path:path}", endpoint=error_endpoint, methods=["GET", "POST", "PUT", "DELETE"])
    ]
    
    return Starlette(routes=routes)


async def load_server_with_retry(config: ServerConfig) -> Tuple[Optional[FastMCP], Optional[Starlette], Optional[str]]:
    """Load a server with retry logic"""
    for attempt in range(config.retry_attempts):
        if attempt > 0:
            logger.info(f"Retrying server '{config.id}' (attempt {attempt + 1}/{config.retry_attempts})")
            await asyncio.sleep(config.retry_delay)
        
        if config.type == "openapi":
            mcp, app, error = await load_openapi_server(config)
        elif config.type == "module":
            mcp, app, error = await load_module_server(config)
        else:
            error = f"Unknown server type: {config.type}"
            return None, None, error
        
        if app:
            return mcp, app, error
    
    return None, None, error or "Max retries exceeded"
```

### mcp_hub/server.py
```python
import asyncio
import logging
from contextlib import asynccontextmanager
from typing import List, Dict

from starlette.applications import Starlette
from starlette.routing import Mount, Route
from starlette.responses import JSONResponse
import uvicorn

from .config import Config
from .registry import MCPRegistry, ServerInfo, ServerStatus
from .loaders import load_server_with_retry, create_fallback_app


class MCPHub:
    """Main MCP Hub server that hosts multiple MCP servers"""
    
    def __init__(self, config: Config):
        self.config = config
        self.registry = MCPRegistry()
        self.logger = logging.getLogger(__name__)
        
        # Configure logging
        logging.basicConfig(
            level=getattr(logging, config.hub.log_level.upper()),
            format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
    
    async def load_all_servers(self) -> tuple:
        """Load all configured servers concurrently"""
        routes = []
        apps = []
        
        # Load servers concurrently
        tasks = []
        for server_config in self.config.servers:
            if not server_config.enabled:
                self.logger.info(f"Server '{server_config.id}' is disabled")
                self.registry.register(ServerInfo(
                    id=server_config.id,
                    name=server_config.name,
                    status=ServerStatus.DISABLED,
                    mount_path=server_config.mount_path
                ))
                continue
            
            # Register as loading
            self.registry.register(ServerInfo(
                id=server_config.id,
                name=server_config.name,
                status=ServerStatus.LOADING,
                mount_path=server_config.mount_path
            ))
            
            tasks.append((server_config, load_server_with_retry(server_config)))
        
        # Wait for all servers to load
        results = await asyncio.gather(*[task for _, task in tasks], return_exceptions=True)
        
        # Process results
        for (server_config, _), result in zip(tasks, results):
            if isinstance(result, Exception):
                error_msg = f"Unexpected error: {str(result)}"
                self.logger.error(f"Server '{server_config.id}': {error_msg}")
                app = create_fallback_app(server_config.id, server_config.name, error_msg)
                
                info = self.registry.get(server_config.id)
                info.status = ServerStatus.FAILED
                info.error = error_msg
            else:
                mcp, app, error = result
                
                if app:
                    # Successfully loaded
                    info = self.registry.get(server_config.id)
                    info.mcp_instance = mcp
                    info.app = app
                    info.status = ServerStatus.HEALTHY
                    info.loaded_at = datetime.utcnow()
                    
                    if mcp:
                        self.registry.update_capabilities(server_config.id, mcp)
                    
                    self.logger.info(f"Server '{server_config.id}' loaded successfully")
                else:
                    # Failed to load
                    app = create_fallback_app(server_config.id, server_config.name, error)
                    info = self.registry.get(server_config.id)
                    info.status = ServerStatus.FAILED
                    info.error = error
                    self.logger.error(f"Server '{server_config.id}' failed: {error}")
            
            routes.append(Mount(server_config.mount_path, app=app))
            apps.append(app)
        
        return routes, apps
    
    def create_management_routes(self) -> List[Route]:
        """Create hub management routes"""
        routes = []
        
        # Health check
        async def health_check(request):
            all_servers = self.registry.get_all()
            healthy_servers = self.registry.get_healthy()
            
            overall_status = "healthy"
            if len(healthy_servers) == 0:
                overall_status = "unhealthy"
            elif len(healthy_servers) < len(all_servers):
                overall_status = "degraded"
            
            return JSONResponse({
                "status": overall_status,
                "hub": {
                    "name": self.config.hub.name,
                    "healthy_servers": len(healthy_servers),
                    "total_servers": len(all_servers)
                },
                "servers": {id: info.to_dict() for id, info in all_servers.items()}
            })
        
        if self.config.hub.health_check_enabled:
            routes.append(Route("/health", endpoint=health_check))
        
        # List servers
        async def list_servers(request):
            servers = []
            for id, info in self.registry.get_healthy().items():
                servers.append({
                    "id": id,
                    "name": info.name,
                    "mount_path": info.mount_path,
                    "status": info.status.value,
                    "capabilities": info.capabilities,
                    "url": f"{request.url.scheme}://{request.url.netloc}{info.mount_path}"
                })
            
            return JSONResponse({
                "hub": self.config.hub.name,
                "servers": servers
            })
        
        routes.append(Route("/servers", endpoint=list_servers))
        
        # Root endpoint
        async def root(request):
            return JSONResponse({
                "name": self.config.hub.name,
                "endpoints": {
                    "health": "/health",
                    "servers": "/servers"
                },
                "servers": list(self.registry.get_all().keys())
            })
        
        routes.append(Route("/", endpoint=root))
        
        return routes
    
    def create_lifespan(self, apps: List[Starlette]):
        """Create combined lifespan for all apps"""
        
        @asynccontextmanager
        async def lifespan(app):
            # Startup
            self.logger.info(f"Starting {self.config.hub.name}...")
            yield
            # Shutdown
            self.logger.info(f"Shutting down {self.config.hub.name}...")
        
        return lifespan
    
    async def create_app(self) -> Starlette:
        """Create the main hub application"""
        # Load all servers
        server_routes, apps = await self.load_all_servers()
        
        # Create all routes
        routes = self.create_management_routes() + server_routes
        
        # Create hub app
        hub_app = Starlette(
            routes=routes,
            lifespan=self.create_lifespan(apps)
        )
        
        return hub_app
    
    async def run(self):
        """Run the hub server"""
        app = await self.create_app()
        
        config = uvicorn.Config(
            app=app,
            host=self.config.hub.host,
            port=self.config.hub.port,
            log_level=self.config.hub.log_level
        )
        
        server = uvicorn.Server(config)
        await server.serve()
```

### mcp_hub/cli.py
```python
import asyncio
import click
from pathlib import Path

from .server import MCPHub
from .config import Config


@click.group()
def cli():
    """MCP Hub CLI"""
    pass


@cli.command()
@click.option('--config', '-c', default='config.yaml', help='Hub configuration file')
@click.option('--servers', '-s', default='servers.yaml', help='Servers configuration file')
def run(config: str, servers: str):
    """Run the MCP Hub server"""
    
    # Load configuration
    hub_config = Config.from_file(config, servers)
    
    # Create and run hub
    hub = MCPHub(hub_config)
    asyncio.run(hub.run())


@cli.command()
@click.option('--output', '-o', default='config.yaml', help='Output file path')
def init(output: str):
    """Initialize configuration files"""
    
    # Create default config.yaml
    config_content = """hub:
  name: "MCP Hub"
  host: "0.0.0.0"
  port: 8000
  log_level: "info"
  health_check_enabled: true
  timeout: 30.0
"""
    
    # Create default servers.yaml
    servers_content = """servers:
  - id: "example-api"
    name: "Example API Server"
    description: "Example OpenAPI-based MCP server"
    type: "openapi"
    openapi_url: "https://petstore.swagger.io/v2/swagger.json"
    base_url: "https://petstore.swagger.io/v2"
    mount_path: "/example"
    enabled: true
    route_maps:
      - methods: ["GET"]
        pattern: "^/pet/.*"
        mcp_type: "RESOURCE_TEMPLATE"
        mcp_tags: ["pet", "read"]
      - methods: ["POST", "PUT", "DELETE"]
        pattern: ".*"
        mcp_type: "TOOL"
        mcp_tags: ["write"]
"""
    
    # Write files
    Path(output).write_text(config_content)
    Path('servers.yaml').write_text(servers_content)
    
    click.echo(f"Created {output} and servers.yaml")
    click.echo("Edit these files and run: mcp-hub run")


if __name__ == '__main__':
    cli()
```

### config.yaml
```yaml
hub:
  name: "MCP Hub Registry"
  host: "0.0.0.0"
  port: 8000
  log_level: "info"
  health_check_enabled: true
  timeout: 30.0
```

### servers.yaml
```yaml
servers:
  # GitHub API Server
  - id: "github-api"
    name: "GitHub API Server"
    description: "MCP server for GitHub API"
    type: "openapi"
    openapi_url: "https://raw.githubusercontent.com/github/rest-api-description/main/descriptions/api.github.com/api.github.com.yaml"
    base_url: "https://api.github.com"
    auth_type: "bearer"
    auth_token: "${GITHUB_TOKEN}"
    mount_path: "/github"
    enabled: true
    route_maps:
      - methods: ["GET"]
        pattern: "^/repos/[^/]+/[^/]+$"
        mcp_type: "RESOURCE_TEMPLATE"
        mcp_tags: ["repository"]
      - methods: ["POST", "PUT", "PATCH"]
        pattern: ".*"
        mcp_type: "TOOL"
        mcp_tags: ["write"]

  # Local Python Module Server
  - id: "local-tools"
    name: "Local Tools Server"
    description: "Custom local MCP server"
    type: "module"
    module_path: "my_servers.tools"
    factory_function: "create_server"
    mount_path: "/tools"
    enabled: false
```

### setup.py
```python
from setuptools import setup, find_packages

setup(
    name="mcp-hub",
    version="1.0.0",
    packages=find_packages(),
    install_requires=[
        "fastmcp>=0.2.0",
        "httpx>=0.25.0",
        "pydantic>=2.0.0",
        "starlette>=0.37.0",
        "uvicorn>=0.27.0",
        "click>=8.1.0",
        "pyyaml>=6.0",
    ],
    entry_points={
        "console_scripts": [
            "mcp-hub=mcp_hub.cli:cli",
        ],
    },
)
```

## Usage

### 1. Installation
```bash
# Clone and install
git clone <repository>
cd mcp-hub
pip install -e .
```

### 2. Initialize Configuration
```bash
# Create default config files
mcp-hub init
```

### 3. Configure Servers
Edit `servers.yaml` to add your MCP servers:
```yaml
servers:
  - id: "my-api"
    name: "My API Server"
    type: "openapi"
    openapi_url: "https://api.example.com/openapi.json"
    base_url: "https://api.example.com"
    auth_type: "bearer"
    auth_token: "${MY_API_TOKEN}"
    mount_path: "/my-api"
    enabled: true
```

### 4. Run the Hub
```bash
# Set environment variables
export GITHUB_TOKEN=your-github-token
export MY_API_TOKEN=your-api-token

# Run the hub
mcp-hub run

# Or with custom config files
mcp-hub run --config my-config.yaml --servers my-servers.yaml
```

### 5. Access the Hub
```bash
# Check health
curl http://localhost:8000/health

# List servers
curl http://localhost:8000/servers

# Access a server
curl http://localhost:8000/github/repos/octocat/Hello-World
```

## Key Features

1. **Graceful Failure Handling**: Failed servers return 503 errors instead of crashing the hub
2. **Dynamic OpenAPI Loading**: Uses FastMCP's `from_openapi()` for automatic conversion
3. **Multiple Server Types**: Supports OpenAPI and Python module servers
4. **Environment Variables**: Support for `${VAR}` syntax in configuration
5. **Health Monitoring**: Real-time status of all servers
6. **Concurrent Loading**: Servers load in parallel for faster startup
7. **Retry Logic**: Configurable retry attempts for each server
8. **Route Mapping**: Control how OpenAPI routes become MCP components

This implementation provides a robust MCP hub that can host multiple servers with proper error handling and monitoring.




*Thinking...*

> The user wants to change the CLI to load config files from a `/data` folder that's outside the source folder. I need to use proper path handling with pathlib to make this pythonic and cross-platform.
> 
> The typical structure would be:
> ```
> project/
> ├── mcp_hub/  (source code)
> │   ├── __init__.py
> │   ├── cli.py
> │   └── ...
> ├── data/     (config files)
> │   ├── config.yaml
> │   └── servers.yaml
> └── ...
> ```
> 
> I should use Path.resolve() to handle relative paths properly and find the data directory relative to the project root.

Here's the pythonic way to handle loading configuration files from a `/data` folder outside the source directory:

```python
# mcp_hub/cli.py

import asyncio
import click
from pathlib import Path
import sys

from .server import MCPHub
from .config import Config


def find_project_root() -> Path:
    """Find the project root directory (contains the data folder)"""
    # Start from the current file's directory
    current = Path(__file__).resolve().parent
    
    # Go up until we find a directory that contains 'data' folder
    # or reach the root directory
    while current != current.parent:
        if (current / 'data').exists():
            return current
        # Also check if we're already at project root (has setup.py or pyproject.toml)
        if (current / 'setup.py').exists() or (current / 'pyproject.toml').exists():
            return current
        current = current.parent
    
    # If not found, assume parent of mcp_hub package
    return Path(__file__).resolve().parent.parent


def get_data_path() -> Path:
    """Get the data directory path"""
    project_root = find_project_root()
    data_path = project_root / 'data'
    
    # Create data directory if it doesn't exist
    data_path.mkdir(exist_ok=True)
    
    return data_path


@click.group()
@click.option('--data-dir', envvar='MCP_HUB_DATA_DIR', 
              help='Data directory path (default: <project_root>/data)')
@click.pass_context
def cli(ctx, data_dir):
    """MCP Hub CLI"""
    ctx.ensure_object(dict)
    
    # Set data directory
    if data_dir:
        ctx.obj['data_dir'] = Path(data_dir).resolve()
    else:
        ctx.obj['data_dir'] = get_data_path()


@cli.command()
@click.option('--config', '-c', default='config.yaml', 
              help='Hub configuration file name (in data directory)')
@click.option('--servers', '-s', default='servers.yaml', 
              help='Servers configuration file name (in data directory)')
@click.pass_context
def run(ctx, config: str, servers: str):
    """Run the MCP Hub server"""
    
    data_dir = ctx.obj.get('data_dir', get_data_path())
    
    # Construct full paths
    config_path = data_dir / config
    servers_path = data_dir / servers
    
    # Check if files exist
    if not config_path.exists():
        click.echo(f"Error: Config file not found: {config_path}", err=True)
        click.echo(f"Run 'mcp-hub init' to create default configuration files.", err=True)
        sys.exit(1)
    
    if not servers_path.exists():
        click.echo(f"Warning: Servers file not found: {servers_path}", err=True)
        click.echo(f"Creating empty servers configuration...", err=True)
        servers_path = None  # Will use empty servers list
    
    # Load configuration
    click.echo(f"Loading configuration from: {config_path}")
    if servers_path:
        click.echo(f"Loading servers from: {servers_path}")
    
    try:
        hub_config = Config.from_file(str(config_path), str(servers_path) if servers_path else None)
    except Exception as e:
        click.echo(f"Error loading configuration: {e}", err=True)
        sys.exit(1)
    
    # Create and run hub
    hub = MCPHub(hub_config)
    
    click.echo(f"Starting {hub_config.hub.name} on {hub_config.hub.host}:{hub_config.hub.port}")
    asyncio.run(hub.run())


@cli.command()
@click.option('--force', '-f', is_flag=True, help='Overwrite existing files')
@click.pass_context
def init(ctx, force: bool):
    """Initialize configuration files in the data directory"""
    
    data_dir = ctx.obj.get('data_dir', get_data_path())
    
    # Ensure data directory exists
    data_dir.mkdir(parents=True, exist_ok=True)
    
    config_path = data_dir / 'config.yaml'
    servers_path = data_dir / 'servers.yaml'
    
    # Check if files already exist
    if config_path.exists() and not force:
        click.echo(f"Config file already exists: {config_path}")
        click.echo("Use --force to overwrite")
        return
    
    # Create default config.yaml
    config_content = """# MCP Hub Configuration
hub:
  name: "MCP Hub"
  host: "0.0.0.0"
  port: 8000
  log_level: "info"
  health_check_enabled: true
  timeout: 30.0
"""
    
    # Create default servers.yaml
    servers_content = """# MCP Servers Configuration
servers:
  # Example OpenAPI-based server
  - id: "example-api"
    name: "Example API Server"
    description: "Example OpenAPI-based MCP server"
    type: "openapi"
    openapi_url: "https://petstore.swagger.io/v2/swagger.json"
    base_url: "https://petstore.swagger.io/v2"
    mount_path: "/example"
    enabled: true
    route_maps:
      - methods: ["GET"]
        pattern: "^/pet/.*"
        mcp_type: "RESOURCE_TEMPLATE"
        mcp_tags: ["pet", "read"]
      - methods: ["POST", "PUT", "DELETE"]
        pattern: ".*"
        mcp_type: "TOOL"
        mcp_tags: ["write"]

  # Add your servers here
  # - id: "my-server"
  #   name: "My Server"
  #   type: "openapi"
  #   openapi_url: "https://api.example.com/openapi.json"
  #   mount_path: "/my-api"
  #   enabled: true
"""
    
    # Write files
    config_path.write_text(config_content)
    servers_path.write_text(servers_content)
    
    click.echo(f"Created configuration files in: {data_dir}")
    click.echo(f"  - {config_path}")
    click.echo(f"  - {servers_path}")
    click.echo("\nEdit these files and run: mcp-hub run")


@cli.command()
@click.pass_context
def info(ctx):
    """Show information about the MCP Hub installation"""
    
    data_dir = ctx.obj.get('data_dir', get_data_path())
    project_root = find_project_root()
    
    click.echo("MCP Hub Information")
    click.echo("=" * 40)
    click.echo(f"Project root: {project_root}")
    click.echo(f"Data directory: {data_dir}")
    click.echo(f"Data dir exists: {data_dir.exists()}")
    
    if data_dir.exists():
        config_path = data_dir / 'config.yaml'
        servers_path = data_dir / 'servers.yaml'
        
        click.echo(f"\nConfiguration files:")
        click.echo(f"  config.yaml: {'Found' if config_path.exists() else 'Not found'}")
        click.echo(f"  servers.yaml: {'Found' if servers_path.exists() else 'Not found'}")
        
        # List all yaml files in data directory
        yaml_files = list(data_dir.glob('*.yaml')) + list(data_dir.glob('*.yml'))
        if yaml_files:
            click.echo(f"\nAvailable configuration files:")
            for f in yaml_files:
                click.echo(f"  - {f.name}")


# Alternative approach using environment-based configuration
@cli.command()
@click.option('--env', '-e', default='development', 
              type=click.Choice(['development', 'staging', 'production']),
              help='Environment to load configuration for')
@click.pass_context
def run_env(ctx, env: str):
    """Run MCP Hub with environment-specific configuration"""
    
    data_dir = ctx.obj.get('data_dir', get_data_path())
    
    # Look for environment-specific files
    config_path = data_dir / f'config.{env}.yaml'
    servers_path = data_dir / f'servers.{env}.yaml'
    
    # Fall back to default files if env-specific don't exist
    if not config_path.exists():
        config_path = data_dir / 'config.yaml'
    if not servers_path.exists():
        servers_path = data_dir / 'servers.yaml'
    
    if not config_path.exists():
        click.echo(f"Error: No configuration found for environment: {env}", err=True)
        sys.exit(1)
    
    click.echo(f"Running in {env} environment")
    ctx.invoke(run, config=config_path.name, servers=servers_path.name)


if __name__ == '__main__':
    cli()
```

Also update the `Config.from_file` method to handle Path objects properly:

```python
# mcp_hub/config.py

from pathlib import Path
# ... other imports ...

class Config(BaseModel):
    hub: HubConfig = Field(default_factory=HubConfig)
    servers: List[ServerConfig] = Field(default_factory=list)
    
    @classmethod
    def from_file(cls, config_path: str | Path, servers_path: Optional[str | Path] = None) -> "Config":
        """Load configuration from YAML files"""
        # Convert to Path objects
        config_path = Path(config_path)
        
        # Load hub config
        with open(config_path, 'r') as f:
            config_data = yaml.safe_load(f)
        
        # Load servers config if provided
        if servers_path:
            servers_path = Path(servers_path)
            if servers_path.exists():
                with open(servers_path, 'r') as f:
                    servers_data = yaml.safe_load(f)
                    config_data['servers'] = servers_data.get('servers', [])
        
        # Substitute environment variables
        config_data = substitute_env_vars(config_data)
        
        return cls(**config_data)
```

## Directory Structure

The expected directory structure would be:

```
mcp-hub-project/
├── mcp_hub/           # Source code
│   ├── __init__.py
│   ├── cli.py
│   ├── config.py
│   ├── server.py
│   └── ...
├── data/              # Configuration files
│   ├── config.yaml
│   ├── servers.yaml
│   ├── config.production.yaml  # Optional env-specific
│   └── servers.production.yaml # Optional env-specific
├── setup.py
├── requirements.txt
└── README.md
```

## Usage Examples

```bash
# Initialize configuration files in data directory
mcp-hub init

# Run with default configuration files from data/
mcp-hub run

# Run with specific config files from data/
mcp-hub run --config my-config.yaml --servers my-servers.yaml

# Use a different data directory
mcp-hub --data-dir /path/to/configs run

# Or use environment variable
export MCP_HUB_DATA_DIR=/path/to/configs
mcp-hub run

# Show hub information
mcp-hub info

# Run with environment-specific configs
mcp-hub run-env --env production  # Uses config.production.yaml if exists
```

## Additional Features

1. **Environment Variable for Data Directory**: 
   ```bash
   export MCP_HUB_DATA_DIR=/etc/mcp-hub
   mcp-hub run
   ```

2. **Project Root Detection**: Automatically finds the project root by looking for common markers

3. **Graceful File Handling**: Warns when files don't exist and provides helpful messages

4. **Environment-Specific Configs**: Support for `config.{env}.yaml` pattern

5. **Info Command**: Shows where configuration files are located

This approach provides:
- Clean separation between code and configuration
- Flexibility in configuration location
- Environment-based configuration support
- Better organization for production deployments
- Cross-platform path handling using `pathlib`




Here's the implementation for a UVX loader and the updated structure to handle multiple package managers:

## 1. **loaders/uvx.py** (New UVX Loader)

```python
import asyncio
import tempfile
import shutil
from pathlib import Path
from typing import Optional, Dict, Any, List
import sys

from .base import ServerLoader, LoadResult
from ..models.registry import ServerEntry


class UVXServerLoader(ServerLoader):
    """Loader for Python packages that can be run with uvx"""
    
    async def load(self, entry: ServerEntry) -> LoadResult:
        # Find uvx package configuration
        uvx_package = None
        for package in entry.packages:
            if package.registry_name == "uvx":
                uvx_package = package
                break
        
        if not uvx_package:
            return LoadResult(
                success=False,
                error="No uvx package configuration found"
            )
        
        try:
            # Prepare environment variables
            env = self._prepare_environment(uvx_package)
            
            # Build uvx command
            cmd = self._build_uvx_command(uvx_package)
            
            # Create server process
            process = await self._start_uvx_process(cmd, env)
            
            # Get connection info from the process
            connection_info = await self._get_connection_info(process)
            
            return LoadResult(
                success=True,
                connection_info=connection_info,
                process=process,
                cleanup_func=lambda: asyncio.create_task(self._cleanup(process))
            )
            
        except Exception as e:
            return LoadResult(
                success=False,
                error=f"Failed to load uvx package: {str(e)}"
            )
    
    def _build_uvx_command(self, package) -> List[str]:
        """Build the uvx command line"""
        cmd = ["uvx"]
        
        # Add any uvx-specific options
        if package.uvx_options:
            for opt in package.uvx_options:
                cmd.append(opt)
        
        # Add package name with optional version
        if package.version and package.version != "latest":
            cmd.append(f"{package.name}=={package.version}")
        else:
            cmd.append(package.name)
        
        # Add MCP server module/command
        if package.entry_point:
            cmd.append(package.entry_point)
        else:
            # Default to running as MCP server
            cmd.extend(["-m", "mcp"])
        
        # Add package arguments
        for arg in package.package_arguments:
            if arg.type == "positional":
                cmd.append(arg.value)
            else:  # flag
                cmd.append(f"--{arg.name}")
                if arg.value:
                    cmd.append(arg.value)
        
        return cmd
    
    def _prepare_environment(self, package) -> Dict[str, str]:
        """Prepare environment variables"""
        import os
        env = os.environ.copy()
        
        # Add package-specific environment variables
        for env_var in package.environment_variables:
            value = os.getenv(env_var.name, env_var.default)
            if value:
                env[env_var.name] = value
            elif env_var.required:
                raise ValueError(f"Required environment variable {env_var.name} not set")
        
        # Ensure Python can find modules
        if "PYTHONPATH" in env:
            env["PYTHONPATH"] = f"{env['PYTHONPATH']}:{sys.path[0]}"
        else:
            env["PYTHONPATH"] = sys.path[0]
        
        return env
    
    async def _start_uvx_process(self, cmd: List[str], env: Dict[str, str]):
        """Start the uvx process"""
        process = await asyncio.create_subprocess_exec(
            *cmd,
            env=env,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            stdin=asyncio.subprocess.PIPE
        )
        
        # Wait a bit to ensure the process starts
        await asyncio.sleep(2)
        
        # Check if process is still running
        if process.returncode is not None:
            stderr = await process.stderr.read()
            raise RuntimeError(f"uvx process exited immediately: {stderr.decode()}")
        
        return process
    
    async def _get_connection_info(self, process) -> Dict[str, Any]:
        """Extract MCP connection info from the process"""
        # For MCP servers, they typically output connection info to stdout
        try:
            # Read initial output
            data = await asyncio.wait_for(process.stdout.readline(), timeout=5.0)
            line = data.decode().strip()
            
            # Parse MCP connection info (adjust based on actual format)
            import json
            if line.startswith("{"):
                return json.loads(line)
            else:
                # Default stdio connection
                return {
                    "type": "stdio",
                    "process": process
                }
        except asyncio.TimeoutError:
            # Assume stdio connection if no explicit info
            return {
                "type": "stdio", 
                "process": process
            }
    
    async def _cleanup(self, process):
        """Clean up the uvx process"""
        if process.returncode is None:
            process.terminate()
            try:
                await asyncio.wait_for(process.wait(), timeout=5.0)
            except asyncio.TimeoutError:
                process.kill()
                await process.wait()
```

## 2. **loaders/package.py** (Updated Base Package Loader)

```python
from abc import ABC, abstractmethod
from typing import Optional, List, Dict, Any
import asyncio

from .base import ServerLoader, LoadResult
from ..models.registry import ServerEntry, PackageInfo


class PackageServerLoader(ServerLoader, ABC):
    """Base class for package-based server loaders"""
    
    @abstractmethod
    def get_registry_name(self) -> str:
        """Get the package registry name this loader handles"""
        pass
    
    @abstractmethod
    async def install_package(self, package: PackageInfo) -> bool:
        """Install the package if needed"""
        pass
    
    @abstractmethod
    async def run_package(self, package: PackageInfo, env: Dict[str, str]) -> asyncio.subprocess.Process:
        """Run the installed package"""
        pass
    
    async def load(self, entry: ServerEntry) -> LoadResult:
        """Common loading logic for package-based servers"""
        # Find package for this loader
        package = self._find_package(entry)
        if not package:
            return LoadResult(
                success=False,
                error=f"No {self.get_registry_name()} package configuration found"
            )
        
        try:
            # Install package if needed
            if not await self.install_package(package):
                return LoadResult(
                    success=False,
                    error=f"Failed to install {self.get_registry_name()} package"
                )
            
            # Prepare environment
            env = self._prepare_environment(package)
            
            # Run the package
            process = await self.run_package(package, env)
            
            # Get connection info
            connection_info = await self._get_connection_info(process)
            
            return LoadResult(
                success=True,
                connection_info=connection_info,
                process=process,
                cleanup_func=lambda: asyncio.create_task(self._cleanup(process))
            )
            
        except Exception as e:
            return LoadResult(
                success=False,
                error=f"Failed to load {self.get_registry_name()} package: {str(e)}"
            )
    
    def _find_package(self, entry: ServerEntry) -> Optional[PackageInfo]:
        """Find package configuration for this loader"""
        for package in entry.packages:
            if package.registry_name == self.get_registry_name():
                return package
        return None
    
    def _prepare_environment(self, package: PackageInfo) -> Dict[str, str]:
        """Prepare environment variables"""
        import os
        env = os.environ.copy()
        
        for env_var in package.environment_variables:
            value = os.getenv(env_var.name, env_var.default)
            if value:
                env[env_var.name] = value
            elif env_var.required:
                raise ValueError(f"Required environment variable {env_var.name} not set")
        
        return env
    
    async def _get_connection_info(self, process) -> Dict[str, Any]:
        """Extract MCP connection info from the process"""
        # Default implementation - can be overridden
        return {
            "type": "stdio",
            "process": process
        }
    
    async def _cleanup(self, process):
        """Clean up the process"""
        if process.returncode is None:
            process.terminate()
            try:
                await asyncio.wait_for(process.wait(), timeout=5.0)
            except asyncio.TimeoutError:
                process.kill()
                await process.wait()
```

## 3. **loaders/npm.py** (Refactored NPM Loader)

```python
import asyncio
import json
import shutil
from pathlib import Path
from typing import Dict, Any, Optional

from .package import PackageServerLoader
from ..models.registry import PackageInfo


class NPMServerLoader(PackageServerLoader):
    """Loader for NPM-based MCP servers"""
    
    def get_registry_name(self) -> str:
        return "npm"
    
    async def install_package(self, package: PackageInfo) -> bool:
        """Install NPM package using npm or npx"""
        # Check if package is already installed globally
        check_cmd = ["npm", "list", "-g", package.name, "--json"]
        
        try:
            proc = await asyncio.create_subprocess_exec(
                *check_cmd,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.DEVNULL
            )
            stdout, _ = await proc.communicate()
            
            if proc.returncode == 0:
                data = json.loads(stdout)
                if package.name in data.get("dependencies", {}):
                    return True  # Already installed
        except:
            pass  # Continue to installation
        
        # Install the package globally
        install_cmd = ["npm", "install", "-g"]
        if package.version and package.version != "latest":
            install_cmd.append(f"{package.name}@{package.version}")
        else:
            install_cmd.append(package.name)
        
        proc = await asyncio.create_subprocess_exec(
            *install_cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        
        stdout, stderr = await proc.communicate()
        return proc.returncode == 0
    
    async def run_package(self, package: PackageInfo, env: Dict[str, str]) -> asyncio.subprocess.Process:
        """Run the NPM package"""
        # Build command
        cmd = ["npx", package.name]
        
        # Add entry point if specified
        if package.entry_point:
            cmd.append(package.entry_point)
        
        # Add arguments
        for arg in package.package_arguments:
            if arg.type == "positional":
                cmd.append(arg.value)
            else:  # flag
                cmd.append(f"--{arg.name}")
                if arg.value:
                    cmd.append(arg.value)
        
        # Start process
        process = await asyncio.create_subprocess_exec(
            *cmd,
            env=env,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            stdin=asyncio.subprocess.PIPE
        )
        
        # Wait a bit to ensure process starts
        await asyncio.sleep(2)
        
        if process.returncode is not None:
            stderr = await process.stderr.read()
            raise RuntimeError(f"NPM process exited immediately: {stderr.decode()}")
        
        return process
```

## 4. **loaders/pypi.py** (New PyPI Loader)

```python
import asyncio
import sys
import subprocess
from typing import Dict, Any

from .package import PackageServerLoader
from ..models.registry import PackageInfo


class PyPIServerLoader(PackageServerLoader):
    """Loader for PyPI-based MCP servers using pip"""
    
    def get_registry_name(self) -> str:
        return "pypi"
    
    async def install_package(self, package: PackageInfo) -> bool:
        """Install package using pip"""
        # Check if already installed
        check_cmd = [sys.executable, "-m", "pip", "show", package.name]
        
        try:
            proc = await asyncio.create_subprocess_exec(
                *check_cmd,
                stdout=asyncio.subprocess.DEVNULL,
                stderr=asyncio.subprocess.DEVNULL
            )
            await proc.communicate()
            
            if proc.returncode == 0:
                # TODO: Check version if needed
                return True
        except:
            pass
        
        # Install the package
        install_cmd = [sys.executable, "-m", "pip", "install"]
        if package.version and package.version != "latest":
            install_cmd.append(f"{package.name}=={package.version}")
        else:
            install_cmd.append(package.name)
        
        proc = await asyncio.create_subprocess_exec(
            *install_cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        
        stdout, stderr = await proc.communicate()
        return proc.returncode == 0
    
    async def run_package(self, package: PackageInfo, env: Dict[str, str]) -> asyncio.subprocess.Process:
        """Run the Python package as MCP server"""
        # Build command
        if package.entry_point:
            # Use entry point directly
            cmd = [sys.executable, "-m", package.entry_point]
        else:
            # Assume package provides MCP server at package.mcp
            cmd = [sys.executable, "-m", f"{package.name}.mcp"]
        
        # Add arguments
        for arg in package.package_arguments:
            if arg.type == "positional":
                cmd.append(arg.value)
            else:  # flag
                cmd.append(f"--{arg.name}")
                if arg.value:
                    cmd.append(arg.value)
        
        # Start process
        process = await asyncio.create_subprocess_exec(
            *cmd,
            env=env,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            stdin=asyncio.subprocess.PIPE
        )
        
        await asyncio.sleep(2)
        
        if process.returncode is not None:
            stderr = await process.stderr.read()
            raise RuntimeError(f"Python process exited immediately: {stderr.decode()}")
        
        return process
```

## 5. **models/server.py** (Updated Package Models)

```python
from typing import List, Optional, Dict, Any
from pydantic import BaseModel, Field


class PackageArgument(BaseModel):
    """Argument for package execution"""
    name: Optional[str] = None
    description: str
    is_required: bool = True
    format: str = "string"
    value: Optional[str] = None
    default: Optional[str] = None
    type: str = "positional"  # positional or flag


class EnvironmentVariable(BaseModel):
    """Environment variable configuration"""
    name: str
    description: str
    required: bool = True
    default: Optional[str] = None


class UVXOptions(BaseModel):
    """UVX-specific options"""
    python_version: Optional[str] = None  # e.g., "3.11"
    with_packages: List[str] = Field(default_factory=list)  # Additional packages
    index_url: Optional[str] = None  # Custom package index


class PackageInfo(BaseModel):
    """Package configuration for any package manager"""
    registry_name: str  # npm, pypi, uvx, docker
    name: str
    version: Optional[str] = "latest"
    
    # Entry point or command to run
    entry_point: Optional[str] = None
    
    # Arguments
    package_arguments: List[PackageArgument] = Field(default_factory=list)
    runtime_arguments: List[PackageArgument] = Field(default_factory=list)
    
    # Environment
    environment_variables: List[EnvironmentVariable] = Field(default_factory=list)
    
    # Package manager specific options
    npm_options: Optional[Dict[str, Any]] = None
    pip_options: Optional[Dict[str, Any]] = None
    uvx_options: Optional[List[str]] = None  # Raw uvx command options
    docker_options: Optional[Dict[str, Any]] = None
```

## 6. **loaders/__init__.py** (Updated Loader Registry)

```python
from typing import Dict, Type

from .base import ServerLoader
from .openapi import OpenAPIServerLoader
from .module import ModuleServerLoader
from .npm import NPMServerLoader
from .pypi import PyPIServerLoader
from .uvx import UVXServerLoader
from .docker import DockerServerLoader


# Registry of loaders by package registry name
PACKAGE_LOADERS: Dict[str, Type[ServerLoader]] = {
    "npm": NPMServerLoader,
    "pypi": PyPIServerLoader,
    "uvx": UVXServerLoader,
    "docker": DockerServerLoader,
}


def get_loader_for_entry(entry) -> ServerLoader:
    """Get the appropriate loader for a server entry"""
    # Check for OpenAPI configuration
    if entry.openapi_config:
        return OpenAPIServerLoader()
    
    # Check for module configuration
    if hasattr(entry, 'module_config') and entry.module_config:
        return ModuleServerLoader()
    
    # Check for package configuration
    if entry.packages:
        # Use the first package's registry to determine loader
        registry_name = entry.packages[0].registry_name
        loader_class = PACKAGE_LOADERS.get(registry_name)
        if loader_class:
            return loader_class()
    
    raise ValueError(f"No suitable loader found for entry: {entry.id}")
```

## 7. **Example YAML Configuration with UVX**

```yaml
servers:
  # UVX-based server example
  - id: "mcp-simple-server"
    name: "Simple MCP Server"
    description: "Basic MCP server installed via uvx"
    repository:
      url: "https://github.com/example/mcp-simple"
      source: "github"
    version_detail:
      version: "1.0.0"
      release_date: "2025-01-15T00:00:00Z"
      is_latest: true
    packages:
      - registry_name: "uvx"
        name: "mcp-simple"
        version: "1.0.0"
        entry_point: "server"  # Will run: uvx mcp-simple server
        uvx_options:
          - "--python"
          - "3.11"
        package_arguments:
          - description: "Server name"
            is_required: true
            value: "simple-server"
            type: "positional"
          - name: "port"
            description: "Server port"
            value: "8090"
            type: "flag"
        environment_variables:
          - name: "MCP_LOG_LEVEL"
            description: "Logging level"
            required: false
            default: "info"
    tags: ["uvx", "python", "simple"]
    enabled: true

  # FastAPI-based MCP server via uvx
  - id: "fastapi-mcp-server"
    name: "FastAPI MCP Server"
    description: "FastAPI-based MCP server with auto-reload"
    repository:
      url: "https://github.com/example/fastapi-mcp"
      source: "github"
    packages:
      - registry_name: "uvx"
        name: "fastapi-mcp[all]"  # Include optional dependencies
        version: "2.0.0"
        uvx_options:
          - "--with"
          - "uvicorn"
          - "--with" 
          - "httpx"
        package_arguments:
          - name: "reload"
            description: "Enable auto-reload"
            type: "flag"
          - name: "host"
            description: "Host to bind"
            value: "0.0.0.0"
            type: "flag"
    tags: ["uvx", "fastapi", "api"]
    enabled: true

  # Tool that can be run via multiple package managers
  - id: "multi-package-server"
    name: "Multi-Package MCP Server"
    description: "Available via npm, pip, or uvx"
    repository:
      url: "https://github.com/example/multi-mcp"
      source: "github"
    packages:
      # NPM version
      - registry_name: "npm"
        name: "@example/multi-mcp"
        version: "1.0.0"
      # PyPI version
      - registry_name: "pypi"
        name: "multi-mcp"
        version: "1.0.0"
      # UVX version (preferred for Python users)
      - registry_name: "uvx"
        name: "multi-mcp"
        version: "1.0.0"
        uvx_options:
          - "--python"
          - ">=3.9"
    tags: ["multi-platform", "python", "node"]
    enabled: true
```

## 8. **CLI Support for UVX**

```python
# In cli.py, add uvx-specific commands

@cli.command()
@click.argument('package_name')
@click.option('--version', default='latest', help='Package version')
@click.option('--python', help='Python version for uvx')
@click.option('--test', is_flag=True, help='Test the package installation')
def add_uvx_server(package_name: str, version: str, python: Optional[str], test: bool):
    """Add a UVX-based MCP server"""
    
    async def add_server():
        if test:
            # Test uvx installation
            cmd = ["uvx", "--version"]
            proc = await asyncio.create_subprocess_exec(
                *cmd,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE
            )
            stdout, stderr = await proc.communicate()
            
            if proc.returncode != 0:
                click.echo("uvx is not installed. Install with: pip install uvx", err=True)
                return
            
            click.echo(f"uvx version: {stdout.decode().strip()}")
        
        # Generate server configuration
        server_id = package_name.replace("_", "-").replace("/", "-")
        
        config = {
            "id": server_id,
            "name": f"{package_name} Server",
            "description": f"MCP server from uvx package {package_name}",
            "repository": {
                "url": "",
                "source": "uvx"
            },
            "version_detail": {
                "version": version,
                "release_date": datetime.utcnow().isoformat() + "Z",
                "is_latest": version == "latest"
            },
            "packages": [
                {
                    "registry_name": "uvx",
                    "name": package_name,
                    "version": version,
                    "uvx_options": []
                }
            ],
            "tags": ["uvx", "python"],
            "enabled": True
        }
        
        if python:
            config["packages"][0]["uvx_options"] = ["--python", python]
        
        # Save configuration
        output_file = f"{server_id}.yaml"
        with open(output_file, 'w') as f:
            yaml.dump(config, f, default_flow_style=False)
        
        click.echo(f"Server configuration saved to {output_file}")
        click.echo(f"Import with: python -m mcp_hub_registry import-server {output_file}")
    
    asyncio.run(add_server())
```

This implementation provides:

1. **Flexible package management** - Support for npm, pip, uvx, and docker
2. **UVX-specific features** - Python version selection, additional packages
3. **Common base class** - Shared logic for all package managers
4. **Multi-platform support** - Servers can specify multiple package options
5. **Clean separation** - Each loader handles its specific package manager

The structure allows easy addition of new package managers while maintaining consistent behavior across all loaders.

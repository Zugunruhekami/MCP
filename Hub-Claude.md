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

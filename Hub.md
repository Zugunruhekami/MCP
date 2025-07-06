*Thinking...*

> The user wants me to:
> 1. Make the code more modularized
> 2. Rename the registry to "MCP Hub"
> 3. Use a TOML or YAML file for server configuration
> 
> I'll restructure the code into multiple modules and create a configuration system. I'll use both TOML and YAML examples since the user mentioned both.
> 
> Let me create a more modular structure:
> - `mcp_hub/config.py` - Configuration loading
> - `mcp_hub/registry.py` - MCP Hub registry
> - `mcp_hub/loaders.py` - Server loaders
> - `mcp_hub/routes.py` - Route handlers
> - `mcp_hub/app.py` - Main application
> - `main.py` - Entry point
> - `config.toml` or `config.yaml` - Configuration file

Here's a more modularized version with configuration file support:

## Project Structure
```
mcp_hub/
├── __init__.py
├── config.py
├── registry.py
├── loaders.py
├── routes.py
├── app.py
├── exceptions.py
└── utils.py
config.toml (or config.yaml)
main.py
```

## Configuration Files

**config.toml**:
```toml
[hub]
name = "MCP Hub"
host = "0.0.0.0"
port = 8000
log_level = "info"
health_check_enabled = true
server_list_enabled = true

[hub.timeouts]
server_load = 30.0
health_check = 5.0

[[servers]]
name = "weather_api"
type = "openapi"
openapi_url = "https://api.weather.example.com/openapi.json"
base_url = "https://api.weather.example.com"
enabled = true

[[servers]]
name = "database_api"
type = "openapi"
openapi_url = "https://api.database.example.com/openapi.json"
enabled = true

[[servers]]
name = "local_tools"
type = "module"
module_path = "my_mcp_servers.tools_server"
factory_function = "create_tools_server"
enabled = true

[[servers]]
name = "remote_mcp"
type = "proxy"
url = "http://remote-server.com:8000/mcp"
transport = "sse"
enabled = true
```

**OR config.yaml**:
```yaml
hub:
  name: "MCP Hub"
  host: "0.0.0.0"
  port: 8000
  log_level: "info"
  health_check_enabled: true
  server_list_enabled: true
  timeouts:
    server_load: 30.0
    health_check: 5.0

servers:
  - name: "weather_api"
    type: "openapi"
    openapi_url: "https://api.weather.example.com/openapi.json"
    base_url: "https://api.weather.example.com"
    enabled: true
    
  - name: "database_api"
    type: "openapi"
    openapi_url: "https://api.database.example.com/openapi.json"
    enabled: true
    
  - name: "local_tools"
    type: "module"
    module_path: "my_mcp_servers.tools_server"
    factory_function: "create_tools_server"
    enabled: true
    
  - name: "remote_mcp"
    type: "proxy"
    url: "http://remote-server.com:8000/mcp"
    transport: "sse"
    enabled: true
```

## Module Implementation

**mcp_hub/__init__.py**:
```python
from .app import MCPHub
from .registry import MCPHubRegistry
from .config import Config

__all__ = ["MCPHub", "MCPHubRegistry", "Config"]
```

**mcp_hub/exceptions.py**:
```python
class MCPHubException(Exception):
    """Base exception for MCP Hub"""
    pass

class ServerLoadException(MCPHubException):
    """Exception raised when a server fails to load"""
    pass

class ConfigurationException(MCPHubException):
    """Exception raised for configuration errors"""
    pass
```

**mcp_hub/config.py**:
```python
import os
from pathlib import Path
from typing import Any, Dict, List, Optional
import toml
import yaml
from pydantic import BaseModel, Field, validator


class TimeoutConfig(BaseModel):
    server_load: float = 30.0
    health_check: float = 5.0


class HubConfig(BaseModel):
    name: str = "MCP Hub"
    host: str = "0.0.0.0"
    port: int = 8000
    log_level: str = "info"
    health_check_enabled: bool = True
    server_list_enabled: bool = True
    timeouts: TimeoutConfig = Field(default_factory=TimeoutConfig)


class ServerConfig(BaseModel):
    name: str
    type: str  # "openapi", "module", "proxy"
    enabled: bool = True
    
    # OpenAPI specific
    openapi_url: Optional[str] = None
    base_url: Optional[str] = None
    
    # Module specific
    module_path: Optional[str] = None
    factory_function: Optional[str] = None
    
    # Proxy specific
    url: Optional[str] = None
    transport: Optional[str] = "stdio"
    
    # Additional options
    mount_path: Optional[str] = None
    retry_count: int = 3
    retry_delay: float = 1.0
    
    @validator('mount_path', always=True)
    def set_mount_path(cls, v, values):
        if v is None and 'name' in values:
            return f"/{values['name']}"
        return v
    
    @validator('type')
    def validate_type(cls, v):
        valid_types = ["openapi", "module", "proxy"]
        if v not in valid_types:
            raise ValueError(f"Server type must be one of: {valid_types}")
        return v


class Config(BaseModel):
    hub: HubConfig = Field(default_factory=HubConfig)
    servers: List[ServerConfig] = Field(default_factory=list)
    
    @classmethod
    def from_file(cls, file_path: str) -> "Config":
        """Load configuration from TOML or YAML file"""
        path = Path(file_path)
        
        if not path.exists():
            raise FileNotFoundError(f"Configuration file not found: {file_path}")
        
        content = path.read_text()
        
        if path.suffix in ['.toml', '.tml']:
            data = toml.loads(content)
        elif path.suffix in ['.yaml', '.yml']:
            data = yaml.safe_load(content)
        else:
            raise ValueError(f"Unsupported config file format: {path.suffix}")
        
        return cls(**data)
    
    @classmethod
    def from_env(cls, prefix: str = "MCP_HUB_") -> "Config":
        """Load configuration from environment variables"""
        config_file = os.getenv(f"{prefix}CONFIG_FILE")
        if config_file:
            return cls.from_file(config_file)
        
        # Build config from individual env vars
        hub_config = {}
        servers_config = []
        
        # Parse hub configuration
        for key, value in os.environ.items():
            if key.startswith(f"{prefix}HUB_"):
                config_key = key.replace(f"{prefix}HUB_", "").lower()
                hub_config[config_key] = value
        
        return cls(hub=HubConfig(**hub_config), servers=servers_config)
```

**mcp_hub/registry.py**:
```python
import logging
from typing import Dict, List, Optional
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum

from fastmcp import FastMCP
from starlette.applications import Starlette


class ServerStatus(Enum):
    LOADING = "loading"
    HEALTHY = "healthy"
    FAILED = "failed"
    DISABLED = "disabled"


@dataclass
class ServerInfo:
    name: str
    status: ServerStatus
    mounted_path: str
    error: Optional[str] = None
    mcp_instance: Optional[FastMCP] = None
    app: Optional[Starlette] = None
    loaded_at: Optional[datetime] = None
    config: Optional[Dict] = None
    
    @property
    def is_healthy(self) -> bool:
        return self.status == ServerStatus.HEALTHY
    
    def to_dict(self) -> Dict:
        return {
            "name": self.name,
            "status": self.status.value,
            "mounted_path": self.mounted_path,
            "error": self.error,
            "loaded_at": self.loaded_at.isoformat() if self.loaded_at else None,
            "is_healthy": self.is_healthy
        }


class MCPHubRegistry:
    """Registry for tracking MCP servers in the hub"""
    
    def __init__(self, hub_name: str = "MCP Hub"):
        self.hub_name = hub_name
        self.servers: Dict[str, ServerInfo] = {}
        self.logger = logging.getLogger(f"{__name__}.{hub_name}")
    
    def register(
        self,
        name: str,
        status: ServerStatus,
        mounted_path: str,
        error: Optional[str] = None,
        mcp_instance: Optional[FastMCP] = None,
        app: Optional[Starlette] = None,
        config: Optional[Dict] = None
    ) -> ServerInfo:
        """Register or update a server"""
        server_info = ServerInfo(
            name=name,
            status=status,
            mounted_path=mounted_path,
            error=error,
            mcp_instance=mcp_instance,
            app=app,
            loaded_at=datetime.utcnow() if status == ServerStatus.HEALTHY else None,
            config=config
        )
        
        self.servers[name] = server_info
        self.logger.info(f"Registered server '{name}' with status: {status.value}")
        
        return server_info
    
    def update_status(self, name: str, status: ServerStatus, error: Optional[str] = None):
        """Update server status"""
        if name in self.servers:
            self.servers[name].status = status
            self.servers[name].error = error
            if status == ServerStatus.HEALTHY:
                self.servers[name].loaded_at = datetime.utcnow()
    
    def get_server(self, name: str) -> Optional[ServerInfo]:
        """Get server info by name"""
        return self.servers.get(name)
    
    def get_healthy_servers(self) -> Dict[str, ServerInfo]:
        """Get all healthy servers"""
        return {
            name: info for name, info in self.servers.items()
            if info.is_healthy
        }
    
    def get_all_servers(self) -> Dict[str, ServerInfo]:
        """Get all registered servers"""
        return self.servers.copy()
    
    def get_summary(self) -> Dict:
        """Get hub summary statistics"""
        all_servers = self.get_all_servers()
        healthy_count = len(self.get_healthy_servers())
        
        status_counts = {}
        for info in all_servers.values():
            status = info.status.value
            status_counts[status] = status_counts.get(status, 0) + 1
        
        return {
            "hub_name": self.hub_name,
            "total_servers": len(all_servers),
            "healthy_servers": healthy_count,
            "status_breakdown": status_counts,
            "health_percentage": (healthy_count / len(all_servers) * 100) if all_servers else 0
        }
```

**mcp_hub/loaders.py**:
```python
import asyncio
import importlib
import logging
from typing import Optional, Tuple

from fastmcp import FastMCP, Client
from starlette.applications import Starlette
from starlette.middleware.errors import ServerErrorMiddleware

from .config import ServerConfig
from .exceptions import ServerLoadException


logger = logging.getLogger(__name__)


class ServerLoader:
    """Base class for server loaders"""
    
    async def load(
        self,
        config: ServerConfig,
        timeout: float = 30.0
    ) -> Tuple[Optional[FastMCP], Optional[Starlette], Optional[str]]:
        """Load a server and return (mcp, app, error)"""
        raise NotImplementedError


class OpenAPIServerLoader(ServerLoader):
    """Loader for OpenAPI-based MCP servers"""
    
    async def load(
        self,
        config: ServerConfig,
        timeout: float = 30.0
    ) -> Tuple[Optional[FastMCP], Optional[Starlette], Optional[str]]:
        try:
            logger.info(f"Loading OpenAPI server '{config.name}' from {config.openapi_url}")
            
            async with asyncio.timeout(timeout):
                mcp = await FastMCP.from_openapi(
                    config.openapi_url,
                    name=f"{config.name}_mcp",
                    base_url=config.base_url
                )
            
            app = mcp.get_asgi_app()
            app = ServerErrorMiddleware(app)
            
            return mcp, app, None
            
        except asyncio.TimeoutError:
            error = f"Timeout loading OpenAPI spec after {timeout}s"
            return None, None, error
        except Exception as e:
            error = f"Failed to load OpenAPI server: {str(e)}"
            return None, None, error


class ModuleServerLoader(ServerLoader):
    """Loader for Python module-based MCP servers"""
    
    async def load(
        self,
        config: ServerConfig,
        timeout: float = 30.0
    ) -> Tuple[Optional[FastMCP], Optional[Starlette], Optional[str]]:
        try:
            logger.info(f"Loading module server '{config.name}' from {config.module_path}")
            
            # Import the module
            module = importlib.import_module(config.module_path)
            
            # Get the factory function
            if not hasattr(module, config.factory_function):
                raise AttributeError(
                    f"Module '{config.module_path}' has no function '{config.factory_function}'"
                )
            
            factory = getattr(module, config.factory_function)
            
            # Call the factory function
            if asyncio.iscoroutinefunction(factory):
                mcp = await factory()
            else:
                mcp = factory()
            
            if not isinstance(mcp, FastMCP):
                raise TypeError(f"Factory function must return FastMCP instance, got {type(mcp)}")
            
            app = mcp.get_asgi_app()
            app = ServerErrorMiddleware(app)
            
            return mcp, app, None
            
        except Exception as e:
            error = f"Failed to load module server: {str(e)}"
            return None, None, error


class ProxyServerLoader(ServerLoader):
    """Loader for proxying remote MCP servers"""
    
    async def load(
        self,
        config: ServerConfig,
        timeout: float = 30.0
    ) -> Tuple[Optional[FastMCP], Optional[Starlette], Optional[str]]:
        try:
            logger.info(f"Creating proxy server '{config.name}' for {config.url}")
            
            # Create a proxy using FastMCP
            mcp = FastMCP.as_proxy(
                config.url,
                name=f"{config.name}_proxy",
                transport=config.transport
            )
            
            app = mcp.get_asgi_app()
            app = ServerErrorMiddleware(app)
            
            return mcp, app, None
            
        except Exception as e:
            error = f"Failed to create proxy server: {str(e)}"
            return None, None, error


class ServerLoaderFactory:
    """Factory for creating appropriate server loaders"""
    
    _loaders = {
        "openapi": OpenAPIServerLoader,
        "module": ModuleServerLoader,
        "proxy": ProxyServerLoader
    }
    
    @classmethod
    def get_loader(cls, server_type: str) -> ServerLoader:
        loader_class = cls._loaders.get(server_type)
        if not loader_class:
            raise ValueError(f"Unknown server type: {server_type}")
        return loader_class()
    
    @classmethod
    def register_loader(cls, server_type: str, loader_class: type):
        """Register a custom loader"""
        cls._loaders[server_type] = loader_class
```

**mcp_hub/routes.py**:
```python
from typing import Optional
from starlette.responses import JSONResponse
from starlette.routing import Mount
from starlette.requests import Request

from .registry import MCPHubRegistry, ServerStatus


def create_health_check_route(registry: MCPHubRegistry):
    """Create health check endpoint"""
    
    async def health_check(request: Request):
        summary = registry.get_summary()
        all_servers = registry.get_all_servers()
        
        # Determine overall status
        if summary["healthy_servers"] == summary["total_servers"]:
            overall_status = "healthy"
        elif summary["healthy_servers"] > 0:
            overall_status = "degraded"
        else:
            overall_status = "unhealthy"
        
        # Build server details
        servers_detail = {}
        for name, info in all_servers.items():
            servers_detail[name] = info.to_dict()
        
        return JSONResponse({
            "status": overall_status,
            "hub": summary,
            "servers": servers_detail
        })
    
    return Mount("/health", endpoint=health_check)


def create_server_list_route(registry: MCPHubRegistry):
    """Create endpoint to list available servers"""
    
    async def list_servers(request: Request):
        servers_info = []
        
        for name, info in registry.get_healthy_servers().items():
            server_data = {
                "name": name,
                "url": f"{request.url.scheme}://{request.url.netloc}{info.mounted_path}",
                "status": info.status.value,
                "loaded_at": info.loaded_at.isoformat() if info.loaded_at else None
            }
            
            # Get capabilities if available
            if info.mcp_instance:
                try:
                    mcp = info.mcp_instance
                    server_data["capabilities"] = {
                        "tools": [{"name": t.name, "description": t.description} 
                                for t in mcp.list_tools()],
                        "resources": [{"uri": r.uri, "name": r.name} 
                                    for r in mcp.list_resources()],
                        "prompts": [{"name": p.name, "description": p.description} 
                                  for p in mcp.list_prompts()]
                    }
                except Exception as e:
                    server_data["capabilities"] = {"error": str(e)}
            
            servers_info.append(server_data)
        
        return JSONResponse({
            "hub_name": registry.hub_name,
            "servers": servers_info,
            "count": len(servers_info)
        })
    
    return Mount("/servers", endpoint=list_servers)


def create_root_route(registry: MCPHubRegistry):
    """Create root endpoint"""
    
    async def root_endpoint(request: Request):
        summary = registry.get_summary()
        
        return JSONResponse({
            "name": registry.hub_name,
            "version": "2.0",
            "endpoints": {
                "health": "/health",
                "servers": "/servers"
            },
            "summary": summary,
            "servers": list(registry.get_all_servers().keys())
        })
    
    return Mount("/", endpoint=root_endpoint)


def create_fallback_app(server_name: str, error_message: str):
    """Create fallback app for failed servers"""
    
    async def error_response(request: Request):
        return JSONResponse({
            "error": "Server unavailable",
            "server": server_name,
            "message": error_message,
            "status": "failed"
        }, status_code=503)
    
    from starlette.applications import Starlette
    app = Starlette(routes=[
        Mount("/", endpoint=error_response, methods=["GET", "POST", "PUT", "DELETE"])
    ])
    
    return app
```

**mcp_hub/app.py**:
```python
import asyncio
import logging
from contextlib import asynccontextmanager
from typing import List, Optional

from starlette.applications import Starlette
from starlette.routing import Mount
import uvicorn

from .config import Config, ServerConfig
from .registry import MCPHubRegistry, ServerStatus
from .loaders import ServerLoaderFactory
from .routes import (
    create_health_check_route,
    create_server_list_route,
    create_root_route,
    create_fallback_app
)


class MCPHub:
    """Main MCP Hub application"""
    
    def __init__(self, config: Config):
        self.config = config
        self.registry = MCPHubRegistry(config.hub.name)
        self.logger = logging.getLogger(__name__)
        
        # Configure logging
        logging.basicConfig(
            level=getattr(logging, config.hub.log_level.upper()),
            format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
    
    async def load_server(self, server_config: ServerConfig) -> tuple:
        """Load a single server with retry logic"""
        if not server_config.enabled:
            self.logger.info(f"Server '{server_config.name}' is disabled")
            self.registry.register(
                server_config.name,
                ServerStatus.DISABLED,
                server_config.mount_path
            )
            return None, None
        
        loader = ServerLoaderFactory.get_loader(server_config.type)
        
        for attempt in range(server_config.retry_count):
            if attempt > 0:
                await asyncio.sleep(server_config.retry_delay)
                self.logger.info(f"Retrying server '{server_config.name}' (attempt {attempt + 1})")
            
            mcp, app, error = await loader.load(
                server_config,
                timeout=self.config.hub.timeouts.server_load
            )
            
            if app:
                self.registry.register(
                    server_config.name,
                    ServerStatus.HEALTHY,
                    server_config.mount_path,
                    mcp_instance=mcp,
                    app=app,
                    config=server_config.dict()
                )
                return mcp, app
            
            self.logger.warning(f"Failed to load '{server_config.name}': {error}")
        
        # All retries failed
        self.registry.register(
            server_config.name,
            ServerStatus.FAILED,
            server_config.mount_path,
            error=error
        )
        return None, None
    
    async def load_all_servers(self) -> tuple:
        """Load all configured servers"""
        routes = []
        apps = []
        
        # Load servers concurrently
        tasks = [
            self.load_server(server_config)
            for server_config in self.config.servers
        ]
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Process results
        for server_config, result in zip(self.config.servers, results):
            if isinstance(result, Exception):
                self.logger.error(f"Unexpected error loading '{server_config.name}': {result}")
                app = create_fallback_app(server_config.name, str(result))
            else:
                mcp, app = result
                if not app:
                    info = self.registry.get_server(server_config.name)
                    app = create_fallback_app(
                        server_config.name,
                        info.error if info else "Unknown error"
                    )
            
            routes.append(Mount(server_config.mount_path, app=app))
            apps.append(app)
        
        return routes, apps
    
    def create_lifespan(self, apps: List[Starlette]):
        """Create combined lifespan for all apps"""
        
        @asynccontextmanager
        async def lifespan(app):
            # Startup
            self.logger.info(f"Starting {self.config.hub.name}...")
            
            async with asyncio.TaskGroup() as tg:
                for sub_app in apps:
                    if hasattr(sub_app, 'router') and hasattr(sub_app.router, 'lifespan_context'):
                        try:
                            tg.create_task(sub_app.router.lifespan_context.__aenter__())
                        except Exception as e:
                            self.logger.error(f"Error in startup: {e}")
            
            self.logger.info(f"{self.config.hub.name} started successfully")
            yield
            
            # Shutdown
            self.logger.info(f"Shutting down {self.config.hub.name}...")
            
            async with asyncio.TaskGroup() as tg:
                for sub_app in apps:
                    if hasattr(sub_app, 'router') and hasattr(sub_app.router, 'lifespan_context'):
                        try:
                            tg.create_task(
                                sub_app.router.lifespan_context.__aexit__(None, None, None)
                            )
                        except Exception as e:
                            self.logger.error(f"Error in shutdown: {e}")
        
        return lifespan
    
    async def create_app(self) -> Starlette:
        """Create the main Starlette application"""
        routes, apps = await self.load_all_servers()
        
        # Add management routes
        if self.config.hub.health_check_enabled:
            routes.insert(0, create_health_check_route(self.registry))
        
        if self.config.hub.server_list_enabled:
            routes.insert(0, create_server_list_route(self.registry))
        
        routes.insert(0, create_root_route(self.registry))
        
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

**main.py**:
```python
import asyncio
import sys
from pathlib import Path

from mcp_hub import MCPHub, Config


async def main():
    # Load configuration
    if len(sys.argv) > 1:
        config_file = sys.argv[1]
    else:
        # Try to find config file
        for name in ["config.toml", "config.yaml", "config.yml"]:
            if Path(name).exists():
                config_file = name
                break
        else:
            print("No configuration file found. Please provide one as argument.")
            sys.exit(1)
    
    try:
        config = Config.from_file(config_file)
    except Exception as e:
        print(f"Failed to load configuration: {e}")
        sys.exit(1)
    
    # Create and run hub
    hub = MCPHub(config)
    await hub.run()


if __name__ == "__main__":
    asyncio.run(main())
```

## Usage

1. Create a configuration file (`config.toml` or `config.yaml`)
2. Run the hub:
```bash
python main.py config.toml
```

Or with your existing servers:
```python
from mcp_hub import MCPHub, Config

# Create config programmatically
config = Config(
    hub={"name": "My MCP Hub", "port": 8000},
    servers=[
        {
            "name": "server1",
            "type": "openapi",
            "openapi_url": "https://api1.com/openapi.json"
        },
        {
            "name": "server2", 
            "type": "openapi",
            "openapi_url": "https://api2.com/openapi.json"
        }
    ]
)

hub = MCPHub(config)
await hub.run()
```

This modular design provides:
- Clear separation of concerns
- Easy extension points (custom loaders, routes)
- Configuration management
- Comprehensive error handling
- Flexible server types (OpenAPI, modules, proxies)
- Production-ready features (health checks, retries, logging)

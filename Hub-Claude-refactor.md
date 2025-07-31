*Thinking...*

> The user has several different implementations of a MCP Hub Registry and wants me to help refactor the `loaders/loaders.py` implementation (which has `load_openapi_server`) to use the `loaders/base.py` pattern from another implementation. They want a not super complicated but good and working solution.
> 
> Looking at the code provided:
> 
> 1. From the chat context, there are multiple implementations shown:
>    - One with a simple `loaders.py` file that has `load_openapi_server` function
>    - Another more comprehensive implementation with:
>      - `loaders/base.py` - Base loader interface
>      - `loaders/openapi.py` - OpenAPI loader with FastMCP
>      - `loaders/module.py` - Python module loader
>      - `loaders/package.py` - NPM/Docker package loader
>      - etc.
> 
> 2. The user wants to refactor the simple `load_openapi_server` to use the base loader pattern.
> 
> Looking at the simple implementation's `load_openapi_server`:
> - It takes a ServerConfig and timeout
> - Returns a tuple of (Optional[FastMCP], Optional[Starlette], Optional[str])
> - Handles authentication, fetches OpenAPI spec, creates MCP server using FastMCP
> 
> Looking at the base loader pattern:
> - Has a `ServerLoader` base class with `load` method
> - Returns a `LoadResult` object with success, error, mcp_instance, app, etc.
> - More structured approach
> 
> I need to:
> 1. Create a refactored version that uses the base loader pattern
> 2. Keep it simple but well-structured
> 3. Make sure it's compatible with the existing code structure

Looking at your implementations, I'll help you refactor the `loaders/loaders.py` to use the `loaders/base.py` pattern. Here's a clean solution that combines the best of both approaches:

## 1. **loaders/base.py** (Base Loader Interface)

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional, Any, Callable
from datetime import datetime

from fastmcp import FastMCP
from starlette.applications import Starlette


@dataclass
class LoadResult:
    """Result of loading a server"""
    success: bool
    error: Optional[str] = None
    mcp_instance: Optional[FastMCP] = None
    app: Optional[Starlette] = None
    connection_info: Optional[dict] = None
    loaded_at: Optional[datetime] = None
    cleanup_func: Optional[Callable] = None


class ServerLoader(ABC):
    """Base class for server loaders"""
    
    @abstractmethod
    async def load(self, config: Any) -> LoadResult:
        """Load a server and return the result"""
        pass
    
    async def cleanup(self, result: LoadResult):
        """Clean up resources if needed"""
        if result.cleanup_func:
            await result.cleanup_func()
```

## 2. **loaders/openapi.py** (Refactored OpenAPI Loader)

```python
import asyncio
import logging
from typing import Optional

import httpx
from fastmcp import FastMCP
from fastmcp.server.openapi import RouteMap, MCPType
from starlette.applications import Starlette
from datetime import datetime

from .base import ServerLoader, LoadResult
from ..config import ServerConfig


logger = logging.getLogger(__name__)


class OpenAPIServerLoader(ServerLoader):
    """Loader for OpenAPI-based MCP servers using FastMCP"""
    
    def __init__(self, timeout: float = 30.0):
        self.timeout = timeout
        self._client: Optional[httpx.AsyncClient] = None
    
    async def load(self, config: ServerConfig) -> LoadResult:
        """Load an MCP server from OpenAPI specification"""
        try:
            logger.info(f"Loading OpenAPI server '{config.id}' from {config.openapi_url}")
            
            # Create HTTP client with authentication
            self._client = self._create_http_client(config)
            
            # Fetch OpenAPI spec
            openapi_spec = await self._fetch_openapi_spec(config.openapi_url)
            
            # Parse route maps if provided
            route_maps = self._parse_route_maps(config.route_maps) if config.route_maps else None
            
            # Create MCP server using FastMCP
            mcp = await FastMCP.from_openapi(
                openapi_spec=openapi_spec,
                client=self._client,
                name=config.name,
                route_maps=route_maps
            )
            
            # Get ASGI app
            app = mcp.get_asgi_app()
            
            logger.info(f"Successfully loaded OpenAPI server '{config.id}'")
            
            return LoadResult(
                success=True,
                mcp_instance=mcp,
                app=app,
                loaded_at=datetime.utcnow(),
                cleanup_func=self._cleanup_client
            )
            
        except asyncio.TimeoutError:
            error = f"Timeout loading OpenAPI spec after {self.timeout}s"
            logger.error(f"Server '{config.id}': {error}")
            await self._cleanup_client()
            return LoadResult(success=False, error=error)
            
        except Exception as e:
            error = f"Failed to load OpenAPI server: {str(e)}"
            logger.error(f"Server '{config.id}': {error}")
            await self._cleanup_client()
            return LoadResult(success=False, error=error)
    
    def _create_http_client(self, config: ServerConfig) -> httpx.AsyncClient:
        """Create HTTP client with authentication"""
        headers = {}
        
        if config.auth_type == "bearer" and config.auth_token:
            headers[config.auth_header] = f"Bearer {config.auth_token}"
        elif config.auth_type == "api_key" and config.auth_token:
            headers[config.auth_header] = config.auth_token
        
        return httpx.AsyncClient(
            base_url=config.base_url or "",
            headers=headers,
            timeout=self.timeout
        )
    
    async def _fetch_openapi_spec(self, url: str) -> dict:
        """Fetch OpenAPI specification from URL"""
        async with asyncio.timeout(self.timeout):
            response = await self._client.get(url)
            response.raise_for_status()
            return response.json()
    
    def _parse_route_maps(self, route_map_configs: list) -> list[RouteMap]:
        """Parse route map configurations"""
        route_maps = []
        
        for rm in route_map_configs:
            route_map = RouteMap(
                methods=rm.get("methods", ["*"]),
                pattern=rm.get("pattern", r".*"),
                tags=set(rm.get("tags", [])),
                mcp_type=MCPType[rm.get("mcp_type", "TOOL")],
                mcp_tags=set(rm.get("mcp_tags", []))
            )
            route_maps.append(route_map)
        
        return route_maps
    
    async def _cleanup_client(self):
        """Clean up HTTP client"""
        if self._client:
            await self._client.aclose()
            self._client = None
```

## 3. **loaders/module.py** (Module Loader)

```python
import asyncio
import importlib
import logging
from datetime import datetime

from fastmcp import FastMCP
from starlette.applications import Starlette

from .base import ServerLoader, LoadResult
from ..config import ServerConfig


logger = logging.getLogger(__name__)


class ModuleServerLoader(ServerLoader):
    """Loader for Python module-based MCP servers"""
    
    async def load(self, config: ServerConfig) -> LoadResult:
        """Load a Python module-based MCP server"""
        try:
            logger.info(f"Loading module server '{config.id}' from {config.module_path}")
            
            # Import the module
            module = importlib.import_module(config.module_path)
            factory = getattr(module, config.factory_function)
            
            # Call the factory function
            if asyncio.iscoroutinefunction(factory):
                mcp = await factory()
            else:
                mcp = factory()
            
            if not isinstance(mcp, FastMCP):
                raise TypeError(f"Factory must return FastMCP instance, got {type(mcp)}")
            
            # Get ASGI app
            app = mcp.get_asgi_app()
            
            logger.info(f"Successfully loaded module server '{config.id}'")
            
            return LoadResult(
                success=True,
                mcp_instance=mcp,
                app=app,
                loaded_at=datetime.utcnow()
            )
            
        except Exception as e:
            error = f"Failed to load module server: {str(e)}"
            logger.error(f"Server '{config.id}': {error}")
            return LoadResult(success=False, error=error)
```

## 4. **loaders/factory.py** (Loader Factory & Utilities)

```python
import logging
from typing import Optional, Tuple

from fastmcp import FastMCP
from starlette.applications import Starlette
from starlette.routing import Route
from starlette.responses import JSONResponse

from .base import ServerLoader, LoadResult
from .openapi import OpenAPIServerLoader
from .module import ModuleServerLoader
from ..config import ServerConfig


logger = logging.getLogger(__name__)


def get_loader_for_config(config: ServerConfig) -> Optional[ServerLoader]:
    """Get the appropriate loader for a server configuration"""
    if config.type == "openapi":
        return OpenAPIServerLoader(timeout=config.retry_delay * config.retry_attempts)
    elif config.type == "module":
        return ModuleServerLoader()
    else:
        logger.error(f"Unknown server type: {config.type}")
        return None


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
    """Load a server with retry logic - compatible with existing code"""
    loader = get_loader_for_config(config)
    if not loader:
        error = f"No loader available for server type: {config.type}"
        return None, None, error
    
    # Try loading with retries
    for attempt in range(config.retry_attempts):
        if attempt > 0:
            logger.info(f"Retrying server '{config.id}' (attempt {attempt + 1}/{config.retry_attempts})")
            await asyncio.sleep(config.retry_delay)
        
        result = await loader.load(config)
        
        if result.success:
            return result.mcp_instance, result.app, None
    
    # All retries failed
    error = result.error or "Max retries exceeded"
    return None, None, error
```

## 5. **Updated loaders/__init__.py**

```python
from .base import ServerLoader, LoadResult
from .openapi import OpenAPIServerLoader
from .module import ModuleServerLoader
from .factory import (
    get_loader_for_config,
    create_fallback_app,
    load_server_with_retry
)

__all__ = [
    'ServerLoader',
    'LoadResult',
    'OpenAPIServerLoader',
    'ModuleServerLoader',
    'get_loader_for_config',
    'create_fallback_app',
    'load_server_with_retry'
]
```

## 6. **Minimal Changes to server.py**

The existing `server.py` can continue to use `load_server_with_retry` and `create_fallback_app` without changes:

```python
# In server.py, just update the import
from .loaders import load_server_with_retry, create_fallback_app

# The rest of the code remains the same!
```

## Key Benefits of This Refactoring:

1. **Clean Separation**: Each loader type is in its own file
2. **Consistent Interface**: All loaders implement the same base class
3. **Better Error Handling**: LoadResult provides structured error information
4. **Resource Cleanup**: Proper cleanup of HTTP clients and other resources
5. **Backward Compatible**: The `load_server_with_retry` function maintains the same interface
6. **Extensible**: Easy to add new loader types (NPM, PyPI, UVX, etc.)

## Adding UVX Support (Optional Extension):

If you want to add UVX support using this pattern:

```python
# loaders/uvx.py
import asyncio
import sys
from datetime import datetime

from .base import ServerLoader, LoadResult
from ..config import ServerConfig


class UVXServerLoader(ServerLoader):
    """Loader for Python packages via uvx"""
    
    async def load(self, config: ServerConfig) -> LoadResult:
        try:
            # Build uvx command
            cmd = ["uvx", config.package_name]
            
            # Start process
            process = await asyncio.create_subprocess_exec(
                *cmd,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
                stdin=asyncio.subprocess.PIPE
            )
            
            # Create a simple app that proxies to the process
            from .proxy import create_stdio_proxy_app
            app = create_stdio_proxy_app(process)
            
            return LoadResult(
                success=True,
                app=app,
                connection_info={"type": "stdio", "process": process},
                loaded_at=datetime.utcnow(),
                cleanup_func=lambda: self._cleanup_process(process)
            )
            
        except Exception as e:
            return LoadResult(success=False, error=str(e))
    
    async def _cleanup_process(self, process):
        if process.returncode is None:
            process.terminate()
            await process.wait()
```

This refactoring provides a clean, extensible structure while maintaining compatibility with your existing code. The loader pattern makes it easy to add new server types without modifying the core logic.

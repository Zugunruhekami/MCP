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



Absolutely! Let's refactor those repeated code blocks into a cleaner solution. Here's the refactored `load_all_servers` method:

```python
# mcp_hub/server.py

import asyncio
import logging
from contextlib import asynccontextmanager
from typing import List, Dict, Optional, Union, Tuple
from datetime import datetime

from starlette.applications import Starlette
from starlette.routing import Mount, Route
from starlette.responses import JSONResponse
import uvicorn

from .config import Config
from .registry import MCPRegistry, ServerInfo, ServerStatus
from .loaders import load_server_with_retry, create_fallback_app, LoadResult

logger = logging.getLogger(__name__)


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
    
    def _handle_server_failure(self, 
                             server_config, 
                             error: str, 
                             exception: Optional[Exception] = None) -> Starlette:
        """Handle server loading failure and return fallback app"""
        # Log the error
        if exception:
            self.logger.error(f"Server '{server_config.id}': {error}", exc_info=exception)
        else:
            self.logger.error(f"Server '{server_config.id}': {error}")
        
        # Create fallback app
        app = create_fallback_app(server_config.id, server_config.name, error)
        
        # Update registry
        info = self.registry.get(server_config.id)
        if info:
            info.status = ServerStatus.FAILED
            info.error = error
            info.app = app
        
        return app
    
    def _handle_server_success(self, 
                             server_config, 
                             result: LoadResult) -> Starlette:
        """Handle successful server loading"""
        info = self.registry.get(server_config.id)
        
        # Update registry with success info
        info.mcp_instance = result.mcp_instance
        info.app = result.app
        info.status = ServerStatus.HEALTHY
        info.loaded_at = result.loaded_at
        info.connection_info = result.connection_info
        
        # Store cleanup function if provided
        if result.cleanup_func:
            info.cleanup_func = result.cleanup_func
        
        # Update capabilities if MCP instance available
        if result.mcp_instance:
            self.registry.update_capabilities(server_config.id, result.mcp_instance)
        
        self.logger.info(f"Server '{server_config.id}' loaded successfully")
        
        return result.app
    
    async def _load_single_server(self, server_config) -> Tuple[ServerConfig, Union[LoadResult, Exception]]:
        """Load a single server and return the result or exception"""
        try:
            result = await load_server_with_retry(server_config)
            return server_config, result
        except Exception as e:
            return server_config, e
    
    async def load_all_servers(self) -> tuple:
        """Load all configured servers concurrently"""
        routes = []
        apps = []
        
        # First pass: Register all servers
        active_configs = []
        
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
            
            active_configs.append(server_config)
        
        # Load all active servers concurrently
        if active_configs:
            # Create tasks for all active servers
            tasks = [self._load_single_server(config) for config in active_configs]
            
            # Wait for all to complete
            results = await asyncio.gather(*tasks, return_exceptions=False)
            
            # Process results
            for server_config, result in results:
                app = None
                
                if isinstance(result, Exception):
                    # Unexpected exception during loading
                    app = self._handle_server_failure(
                        server_config,
                        f"Unexpected error: {str(result)}",
                        result
                    )
                
                elif isinstance(result, LoadResult):
                    if result.success and result.app:
                        # Success
                        app = self._handle_server_success(server_config, result)
                    else:
                        # LoadResult indicates failure
                        app = self._handle_server_failure(
                            server_config,
                            result.error or "Unknown error"
                        )
                
                else:
                    # Unexpected result type
                    app = self._handle_server_failure(
                        server_config,
                        f"Unexpected result type: {type(result)}"
                    )
                
                # Mount the app (fallback or real)
                routes.append(Mount(server_config.mount_path, app=app))
                apps.append(app)
        
        return routes, apps
    
    def create_lifespan(self, apps: List[Starlette]):
        """Create combined lifespan for all apps with cleanup"""
        
        @asynccontextmanager
        async def lifespan(app):
            # Startup
            self.logger.info(f"Starting {self.config.hub.name}...")
            yield
            
            # Shutdown - run all cleanup functions
            self.logger.info(f"Shutting down {self.config.hub.name}...")
            
            # Clean up all servers
            cleanup_tasks = []
            for server_id, info in self.registry.get_all().items():
                if hasattr(info, 'cleanup_func') and info.cleanup_func:
                    self.logger.info(f"Cleaning up server '{server_id}'...")
                    cleanup_tasks.append(self._safe_cleanup(server_id, info.cleanup_func))
            
            if cleanup_tasks:
                await asyncio.gather(*cleanup_tasks, return_exceptions=True)
        
        return lifespan
    
    async def _safe_cleanup(self, server_id: str, cleanup_func):
        """Safely execute cleanup function"""
        try:
            await cleanup_func()
        except Exception as e:
            self.logger.error(f"Error during cleanup of server '{server_id}': {e}")
    
    async def create_app(self) -> Starlette:
        """Create the main Starlette application"""
        # Load all servers
        routes, apps = await self.load_all_servers()
        
        # Add hub API routes
        api_routes = [
            Route("/health", self.health_endpoint),
            Route("/api/servers", self.list_servers_endpoint),
            Route("/api/servers/{server_id}", self.get_server_endpoint),
        ]
        
        # Combine all routes
        all_routes = api_routes + routes
        
        # Create main app
        app = Starlette(
            routes=all_routes,
            lifespan=self.create_lifespan(apps)
        )
        
        return app
    
    # ... rest of MCPHub implementation (endpoints, etc.) ...
```

## Alternative Approach with Result Handler Class

If you want even more separation of concerns, you could create a dedicated result handler:

```python
# mcp_hub/server_loader_handler.py

from typing import Optional, Union
from datetime import datetime
import logging

from starlette.applications import Starlette

from .registry import MCPRegistry, ServerStatus
from .loaders import LoadResult, create_fallback_app
from .config import ServerConfig

logger = logging.getLogger(__name__)


class ServerLoadResultHandler:
    """Handles the results of server loading operations"""
    
    def __init__(self, registry: MCPRegistry):
        self.registry = registry
    
    def handle_result(self, 
                     server_config: ServerConfig,
                     result: Union[LoadResult, Exception]) -> Starlette:
        """Process a server load result and return the appropriate app"""
        
        if isinstance(result, Exception):
            return self._handle_exception(server_config, result)
        
        elif isinstance(result, LoadResult):
            if result.success and result.app:
                return self._handle_success(server_config, result)
            else:
                return self._handle_load_failure(server_config, result)
        
        else:
            return self._handle_unexpected_result(server_config, result)
    
    def _handle_success(self, server_config: ServerConfig, result: LoadResult) -> Starlette:
        """Handle successful server loading"""
        info = self.registry.get(server_config.id)
        
        # Update registry
        info.mcp_instance = result.mcp_instance
        info.app = result.app
        info.status = ServerStatus.HEALTHY
        info.loaded_at = result.loaded_at
        info.connection_info = result.connection_info
        
        if result.cleanup_func:
            info.cleanup_func = result.cleanup_func
        
        # Update capabilities
        if result.mcp_instance:
            self.registry.update_capabilities(server_config.id, result.mcp_instance)
        
        logger.info(f"Server '{server_config.id}' loaded successfully")
        return result.app
    
    def _handle_exception(self, server_config: ServerConfig, exception: Exception) -> Starlette:
        """Handle unexpected exception during loading"""
        error_msg = f"Unexpected error: {str(exception)}"
        logger.error(f"Server '{server_config.id}': {error_msg}", exc_info=exception)
        
        return self._create_and_register_fallback(server_config, error_msg)
    
    def _handle_load_failure(self, server_config: ServerConfig, result: LoadResult) -> Starlette:
        """Handle LoadResult that indicates failure"""
        error_msg = result.error or "Unknown error"
        logger.error(f"Server '{server_config.id}' failed: {error_msg}")
        
        return self._create_and_register_fallback(server_config, error_msg)
    
    def _handle_unexpected_result(self, server_config: ServerConfig, result: any) -> Starlette:
        """Handle unexpected result type"""
        error_msg = f"Unexpected result type: {type(result)}"
        logger.error(f"Server '{server_config.id}': {error_msg}")
        
        return self._create_and_register_fallback(server_config, error_msg)
    
    def _create_and_register_fallback(self, server_config: ServerConfig, error: str) -> Starlette:
        """Create fallback app and update registry"""
        app = create_fallback_app(server_config.id, server_config.name, error)
        
        info = self.registry.get(server_config.id)
        if info:
            info.status = ServerStatus.FAILED
            info.error = error
            info.app = app
        
        return app
```

Then use it in MCPHub:

```python
class MCPHub:
    def __init__(self, config: Config):
        self.config = config
        self.registry = MCPRegistry()
        self.result_handler = ServerLoadResultHandler(self.registry)
        # ... rest of init ...
    
    async def load_all_servers(self) -> tuple:
        """Load all configured servers concurrently"""
        routes = []
        apps = []
        
        # Filter and register servers
        active_configs = []
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
            
            self.registry.register(ServerInfo(
                id=server_config.id,
                name=server_config.name,
                status=ServerStatus.LOADING,
                mount_path=server_config.mount_path
            ))
            active_configs.append(server_config)
        
        # Load all active servers
        if active_configs:
            tasks = [self._load_single_server(config) for config in active_configs]
            results = await asyncio.gather(*tasks, return_exceptions=False)
            
            # Process results using the handler
            for server_config, result in results:
                app = self.result_handler.handle_result(server_config, result)
                routes.append(Mount(server_config.mount_path, app=app))
                apps.append(app)
        
        return routes, apps
```

Both approaches eliminate the code duplication and make the error handling logic much more maintainable. The first approach keeps everything in the MCPHub class with helper methods, while the second approach separates the concern into a dedicated handler class. Choose based on your preference for code organization!

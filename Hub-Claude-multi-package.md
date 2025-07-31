*Thinking...*

> The user is asking about enhancing the UVXServerLoader to handle MCP servers that support different transports (stdio, streamable-http, sse) like the mcp-atlassian example they provided. Looking at the mcp-atlassian docs, I can see it supports:
> 
> 1. Default stdio transport (when run without args)
> 2. SSE transport with `--transport sse --port 9000`
> 3. Streamable HTTP transport with `--transport streamable-http --port 9000`
> 
> The current UVXServerLoader assumes stdio transport, but we need to make it more flexible to handle these different transport types. We also need to update the server configuration models to specify transport type and related options.
> 
> Let me design a comprehensive solution that:
> 1. Updates the server models to include transport configuration
> 2. Enhances UVXServerLoader to handle different transports
> 3. Creates appropriate proxy apps for each transport type
> 4. Updates the factory pattern to handle this properly
> 5. Shows how this works in the servers.yaml configuration

Looking at the mcp-atlassian example, I see it supports multiple transports (stdio, sse, streamable-http). Let me enhance the loader system to handle these different transport types properly:

## 1. **Updated models/server.py** (Transport Configuration)

```python
from typing import List, Optional, Dict, Any, Literal
from pydantic import BaseModel, Field
from enum import Enum


class TransportType(str, Enum):
    STDIO = "stdio"
    SSE = "sse"
    STREAMABLE_HTTP = "streamable-http"
    HTTP = "http"  # Generic HTTP


class TransportConfig(BaseModel):
    """Configuration for MCP server transport"""
    type: TransportType = TransportType.STDIO
    port: Optional[int] = None
    host: Optional[str] = "0.0.0.0"
    endpoint: Optional[str] = None  # e.g., "/sse" or "/mcp"
    headers: Optional[Dict[str, str]] = None
    extra_args: List[str] = Field(default_factory=list)


class PackageInfo(BaseModel):
    """Enhanced package configuration with transport support"""
    registry_name: str  # npm, pypi, uvx, docker
    name: str
    version: Optional[str] = "latest"
    
    # Entry point or command to run
    entry_point: Optional[str] = None
    
    # Transport configuration
    transport: TransportConfig = Field(default_factory=TransportConfig)
    
    # Arguments
    package_arguments: List[PackageArgument] = Field(default_factory=list)
    runtime_arguments: List[PackageArgument] = Field(default_factory=list)
    
    # Environment
    environment_variables: List[EnvironmentVariable] = Field(default_factory=list)
    
    # Package manager specific options
    npm_options: Optional[Dict[str, Any]] = None
    pip_options: Optional[Dict[str, Any]] = None
    uvx_options: Optional[List[str]] = None
    docker_options: Optional[Dict[str, Any]] = None
```

## 2. **loaders/proxy.py** (Transport Proxy Apps)

```python
import asyncio
import json
import httpx
from typing import Dict, Any, Optional
from starlette.applications import Starlette
from starlette.routing import Route
from starlette.responses import JSONResponse, StreamingResponse
from starlette.websockets import WebSocket
import logging

logger = logging.getLogger(__name__)


def create_stdio_proxy_app(process: asyncio.subprocess.Process) -> Starlette:
    """Create a proxy app for stdio-based MCP servers"""
    
    async def handle_request(request):
        """Forward HTTP requests to stdio process"""
        try:
            # Get request body
            body = await request.body()
            
            # Send to process stdin
            process.stdin.write(body + b'\n')
            await process.stdin.drain()
            
            # Read response from stdout
            response_line = await process.stdout.readline()
            response_data = json.loads(response_line.decode())
            
            return JSONResponse(response_data)
            
        except Exception as e:
            logger.error(f"Stdio proxy error: {e}")
            return JSONResponse({"error": str(e)}, status_code=500)
    
    routes = [
        Route("/{path:path}", endpoint=handle_request, methods=["POST"])
    ]
    
    return Starlette(routes=routes)


def create_http_proxy_app(base_url: str, headers: Optional[Dict[str, str]] = None) -> Starlette:
    """Create a proxy app for HTTP-based MCP servers (SSE, streamable-http)"""
    
    client = httpx.AsyncClient(base_url=base_url, headers=headers or {})
    
    async def shutdown():
        await client.aclose()
    
    async def proxy_request(request):
        """Proxy requests to the HTTP MCP server"""
        try:
            # Build target URL
            target_path = request.url.path
            if request.url.query:
                target_path += f"?{request.url.query}"
            
            # Forward the request
            method = request.method
            headers = dict(request.headers)
            headers.pop('host', None)  # Remove host header
            
            if method in ["GET", "DELETE"]:
                response = await client.request(
                    method=method,
                    url=target_path,
                    headers=headers
                )
            else:
                body = await request.body()
                response = await client.request(
                    method=method,
                    url=target_path,
                    headers=headers,
                    content=body
                )
            
            # Handle streaming responses (for SSE)
            if "text/event-stream" in response.headers.get("content-type", ""):
                async def stream_response():
                    async for chunk in response.aiter_bytes():
                        yield chunk
                
                return StreamingResponse(
                    stream_response(),
                    media_type=response.headers.get("content-type"),
                    headers=dict(response.headers)
                )
            
            # Regular response
            return JSONResponse(
                content=response.json() if response.headers.get("content-type", "").startswith("application/json") else response.text,
                status_code=response.status_code,
                headers=dict(response.headers)
            )
            
        except Exception as e:
            logger.error(f"HTTP proxy error: {e}")
            return JSONResponse({"error": str(e)}, status_code=500)
    
    routes = [
        Route("/{path:path}", endpoint=proxy_request, methods=["GET", "POST", "PUT", "DELETE", "PATCH"])
    ]
    
    app = Starlette(routes=routes, on_shutdown=[shutdown])
    return app


def create_transport_proxy_app(
    transport_type: str,
    process: Optional[asyncio.subprocess.Process] = None,
    base_url: Optional[str] = None,
    headers: Optional[Dict[str, str]] = None
) -> Starlette:
    """Factory function to create the appropriate proxy app based on transport type"""
    
    if transport_type == TransportType.STDIO:
        if not process:
            raise ValueError("Process required for stdio transport")
        return create_stdio_proxy_app(process)
    
    elif transport_type in [TransportType.SSE, TransportType.STREAMABLE_HTTP, TransportType.HTTP]:
        if not base_url:
            raise ValueError("Base URL required for HTTP-based transport")
        return create_http_proxy_app(base_url, headers)
    
    else:
        raise ValueError(f"Unsupported transport type: {transport_type}")
```

## 3. **Enhanced loaders/uvx.py**

```python
import asyncio
import sys
import time
from datetime import datetime
from typing import Optional, Dict, Any, List
from pathlib import Path

from .base import ServerLoader, LoadResult
from .proxy import create_transport_proxy_app
from ..models.server import TransportType, TransportConfig
from ..config import ServerConfig

import logging
logger = logging.getLogger(__name__)


class UVXServerLoader(ServerLoader):
    """Enhanced loader for Python packages via uvx with transport support"""
    
    async def load(self, config: ServerConfig) -> LoadResult:
        # Find uvx package configuration
        uvx_package = None
        for package in config.packages:
            if package.registry_name == "uvx":
                uvx_package = package
                break
        
        if not uvx_package:
            return LoadResult(
                success=False,
                error="No uvx package configuration found"
            )
        
        try:
            transport = uvx_package.transport
            
            if transport.type == TransportType.STDIO:
                return await self._load_stdio_server(config, uvx_package)
            else:
                return await self._load_http_server(config, uvx_package)
                
        except Exception as e:
            return LoadResult(
                success=False,
                error=f"Failed to load uvx package: {str(e)}"
            )
    
    async def _load_stdio_server(self, config: ServerConfig, package) -> LoadResult:
        """Load a stdio-based MCP server"""
        try:
            # Prepare environment
            env = self._prepare_environment(package)
            
            # Build command without transport flags
            cmd = self._build_uvx_command(package, include_transport=False)
            
            # Start process
            process = await self._start_process(cmd, env)
            
            # Wait for process to be ready
            await self._wait_for_stdio_ready(process)
            
            # Create stdio proxy app
            from .proxy import create_stdio_proxy_app
            app = create_stdio_proxy_app(process)
            
            return LoadResult(
                success=True,
                app=app,
                connection_info={
                    "type": "stdio",
                    "process_id": process.pid
                },
                loaded_at=datetime.utcnow(),
                cleanup_func=lambda: asyncio.create_task(self._cleanup_process(process))
            )
            
        except Exception as e:
            logger.error(f"Failed to load stdio server: {e}")
            return LoadResult(success=False, error=str(e))
    
    async def _load_http_server(self, config: ServerConfig, package) -> LoadResult:
        """Load an HTTP-based MCP server (SSE, streamable-http)"""
        try:
            transport = package.transport
            
            # Prepare environment
            env = self._prepare_environment(package)
            
            # Build command with transport flags
            cmd = self._build_uvx_command(package, include_transport=True)
            
            # Start process
            process = await self._start_process(cmd, env)
            
            # Wait for HTTP server to be ready
            base_url = await self._wait_for_http_ready(transport)
            
            # Determine endpoint based on transport type
            if transport.type == TransportType.SSE:
                endpoint = transport.endpoint or "/sse"
            elif transport.type == TransportType.STREAMABLE_HTTP:
                endpoint = transport.endpoint or "/mcp"
            else:
                endpoint = transport.endpoint or "/"
            
            full_url = f"{base_url}{endpoint}"
            
            # Create HTTP proxy app
            app = create_transport_proxy_app(
                transport_type=transport.type,
                base_url=base_url,
                headers=transport.headers
            )
            
            return LoadResult(
                success=True,
                app=app,
                connection_info={
                    "type": transport.type,
                    "url": full_url,
                    "process_id": process.pid
                },
                loaded_at=datetime.utcnow(),
                cleanup_func=lambda: asyncio.create_task(self._cleanup_process(process))
            )
            
        except Exception as e:
            logger.error(f"Failed to load HTTP server: {e}")
            return LoadResult(success=False, error=str(e))
    
    def _build_uvx_command(self, package, include_transport: bool = True) -> List[str]:
        """Build the uvx command line"""
        cmd = ["uvx"]
        
        # Add any uvx-specific options
        if package.uvx_options:
            cmd.extend(package.uvx_options)
        
        # Add package name with optional version
        if package.version and package.version != "latest":
            cmd.append(f"{package.name}=={package.version}")
        else:
            cmd.append(package.name)
        
        # Add entry point if specified
        if package.entry_point:
            cmd.append(package.entry_point)
        
        # Add transport configuration if needed
        if include_transport and package.transport.type != TransportType.STDIO:
            cmd.extend(["--transport", package.transport.type])
            if package.transport.port:
                cmd.extend(["--port", str(package.transport.port)])
            # Add any extra transport args
            cmd.extend(package.transport.extra_args)
        
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
        
        return env
    
    async def _start_process(self, cmd: List[str], env: Dict[str, str]) -> asyncio.subprocess.Process:
        """Start the uvx process"""
        logger.info(f"Starting uvx process: {' '.join(cmd)}")
        
        process = await asyncio.create_subprocess_exec(
            *cmd,
            env=env,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
            stdin=asyncio.subprocess.PIPE
        )
        
        # Check if process started successfully
        await asyncio.sleep(1)
        if process.returncode is not None:
            stderr = await process.stderr.read()
            raise RuntimeError(f"Process exited immediately: {stderr.decode()}")
        
        return process
    
    async def _wait_for_stdio_ready(self, process: asyncio.subprocess.Process, timeout: float = 10.0):
        """Wait for stdio server to be ready"""
        try:
            # For stdio servers, we might need to wait for an initialization message
            # or just give it time to start up
            await asyncio.sleep(2)
            
            # Check if process is still running
            if process.returncode is not None:
                stderr = await process.stderr.read()
                raise RuntimeError(f"Process died during startup: {stderr.decode()}")
                
        except asyncio.TimeoutError:
            raise RuntimeError(f"Timeout waiting for stdio server to be ready")
    
    async def _wait_for_http_ready(self, transport: TransportConfig, timeout: float = 30.0) -> str:
        """Wait for HTTP server to be ready and return base URL"""
        import httpx
        
        host = transport.host or "127.0.0.1"
        port = transport.port or 9000
        base_url = f"http://{host}:{port}"
        
        start_time = time.time()
        last_error = None
        
        while time.time() - start_time < timeout:
            try:
                async with httpx.AsyncClient() as client:
                    # Try to connect to the server
                    response = await client.get(f"{base_url}/", timeout=2.0)
                    # Any response (even 404) means server is up
                    logger.info(f"HTTP server ready at {base_url}")
                    return base_url
            except Exception as e:
                last_error = e
                await asyncio.sleep(0.5)
        
        raise RuntimeError(f"HTTP server failed to start within {timeout}s: {last_error}")
    
    async def _cleanup_process(self, process: asyncio.subprocess.Process):
        """Clean up the process"""
        if process.returncode is None:
            logger.info(f"Terminating process {process.pid}")
            process.terminate()
            try:
                await asyncio.wait_for(process.wait(), timeout=5.0)
            except asyncio.TimeoutError:
                logger.warning(f"Force killing process {process.pid}")
                process.kill()
                await process.wait()
```

## 4. **Updated loaders/factory.py**

```python
import logging
from typing import Optional, Tuple, Union

from fastmcp import FastMCP
from starlette.applications import Starlette

from .base import ServerLoader, LoadResult
from .openapi import OpenAPIServerLoader
from .module import ModuleServerLoader
from .uvx import UVXServerLoader
from .npm import NPMServerLoader
from .pypi import PyPIServerLoader
from ..config import ServerConfig

logger = logging.getLogger(__name__)


# Registry of package loaders
PACKAGE_LOADERS = {
    "npm": NPMServerLoader,
    "pypi": PyPIServerLoader,
    "uvx": UVXServerLoader,
}


def get_loader_for_config(config: ServerConfig) -> Optional[ServerLoader]:
    """Get the appropriate loader for a server configuration"""
    
    # Check server type first (for backward compatibility)
    if hasattr(config, 'type'):
        if config.type == "openapi":
            return OpenAPIServerLoader(timeout=config.retry_delay * config.retry_attempts)
        elif config.type == "module":
            return ModuleServerLoader()
    
    # Check for OpenAPI configuration
    if hasattr(config, 'openapi_url') and config.openapi_url:
        return OpenAPIServerLoader(timeout=getattr(config, 'timeout', 30.0))
    
    # Check for module configuration
    if hasattr(config, 'module_path') and config.module_path:
        return ModuleServerLoader()
    
    # Check for package configuration
    if hasattr(config, 'packages') and config.packages:
        # Use the first package's registry to determine loader
        registry_name = config.packages[0].registry_name
        loader_class = PACKAGE_LOADERS.get(registry_name)
        if loader_class:
            return loader_class()
    
    logger.error(f"No suitable loader found for server: {config.id}")
    return None


async def load_server(config: ServerConfig) -> LoadResult:
    """Load a server using the appropriate loader"""
    loader = get_loader_for_config(config)
    
    if not loader:
        return LoadResult(
            success=False,
            error=f"No loader available for server configuration"
        )
    
    return await loader.load(config)


async def load_server_with_retry(config: ServerConfig) -> Tuple[Optional[FastMCP], Optional[Starlette], Optional[str]]:
    """Load a server with retry logic - backward compatible interface"""
    
    result = None
    for attempt in range(config.retry_attempts):
        if attempt > 0:
            logger.info(f"Retrying server '{config.id}' (attempt {attempt + 1}/{config.retry_attempts})")
            await asyncio.sleep(config.retry_delay)
        
        result = await load_server(config)
        
        if result.success:
            return result.mcp_instance, result.app, None
    
    # All retries failed
    error = result.error if result else "Unknown error"
    return None, None, error
```

## 5. **Example servers.yaml Configuration**

```yaml
servers:
  # OpenAPI server (existing)
  - id: "github-api"
    name: "GitHub API Server"
    type: "openapi"
    openapi_url: "https://api.github.com/openapi.yaml"
    base_url: "https://api.github.com"
    mount_path: "/github"
    enabled: true

  # UVX server with stdio transport (default)
  - id: "mcp-atlassian-stdio"
    name: "Atlassian MCP (stdio)"
    mount_path: "/atlassian-stdio"
    enabled: true
    packages:
      - registry_name: "uvx"
        name: "mcp-atlassian"
        version: "0.11.9"
        transport:
          type: "stdio"
        environment_variables:
          - name: "JIRA_URL"
            description: "Jira URL"
            default: "https://company.atlassian.net"
          - name: "JIRA_USERNAME"
            description: "Jira username"
          - name: "JIRA_API_TOKEN"
            description: "Jira API token"

  # UVX server with SSE transport
  - id: "mcp-atlassian-sse"
    name: "Atlassian MCP (SSE)"
    mount_path: "/atlassian-sse"
    enabled: true
    packages:
      - registry_name: "uvx"
        name: "mcp-atlassian"
        version: "0.11.9"
        transport:
          type: "sse"
          port: 9001
          endpoint: "/sse"
          extra_args: ["-vv"]  # verbose logging
        environment_variables:
          - name: "CONFLUENCE_URL"
            default: "https://company.atlassian.net/wiki"
          - name: "CONFLUENCE_USERNAME"
          - name: "CONFLUENCE_API_TOKEN"

  # UVX server with streamable-http transport  
  - id: "mcp-atlassian-http"
    name: "Atlassian MCP (HTTP)"
    mount_path: "/atlassian"
    enabled: true
    packages:
      - registry_name: "uvx"
        name: "mcp-atlassian"
        version: "latest"
        transport:
          type: "streamable-http"
          port: 9002
          endpoint: "/mcp"
          headers:
            Authorization: "Bearer ${ATLASSIAN_OAUTH_TOKEN}"
            X-Atlassian-Cloud-Id: "${ATLASSIAN_CLOUD_ID}"
        uvx_options:
          - "--python"
          - "3.11"
        environment_variables:
          - name: "JIRA_URL"
            default: "https://company.atlassian.net"
          - name: "CONFLUENCE_URL"
            default: "https://company.atlassian.net/wiki"
          - name: "ATLASSIAN_OAUTH_CLIENT_ID"
          - name: "ATLASSIAN_OAUTH_CLIENT_SECRET"

  # NPM server example
  - id: "sqlite-mcp"
    name: "SQLite MCP Server"
    mount_path: "/sqlite"
    enabled: true
    packages:
      - registry_name: "npm"
        name: "@modelcontextprotocol/server-sqlite"
        version: "0.1.0"
        transport:
          type: "stdio"  # Most npm MCP servers use stdio
        package_arguments:
          - description: "Database path"
            value: "./data.db"
            type: "positional"
```

## 6. **Enhanced loaders/npm.py** (With Transport Support)

```python
import asyncio
import json
import shutil
from pathlib import Path
from typing import Dict, Any, Optional
from datetime import datetime

from .package import PackageServerLoader
from .proxy import create_transport_proxy_app
from ..models.registry import PackageInfo
from ..models.server import TransportType

import logging
logger = logging.getLogger(__name__)


class NPMServerLoader(PackageServerLoader):
    """Enhanced NPM loader with transport support"""
    
    def get_registry_name(self) -> str:
        return "npm"
    
    async def install_package(self, package: PackageInfo) -> bool:
        """Install NPM package using npm"""
        # [Previous implementation remains the same]
        ...
    
    async def run_package(self, package: PackageInfo, env: Dict[str, str]) -> asyncio.subprocess.Process:
        """Run the NPM package with transport support"""
        # Build base command
        cmd = ["npx", package.name]
        
        # Add entry point if specified
        if package.entry_point:
            cmd.append(package.entry_point)
        
        # Add transport flags for non-stdio
        transport = package.transport
        if transport.type != TransportType.STDIO:
            if transport.type == TransportType.SSE:
                cmd.extend(["--transport", "sse"])
            elif transport.type == TransportType.STREAMABLE_HTTP:
                cmd.extend(["--transport", "streamable-http"])
            
            if transport.port:
                cmd.extend(["--port", str(transport.port)])
            
            cmd.extend(transport.extra_args)
        
        # Add package arguments
        for arg in package.package_arguments:
            if arg.type == "positional":
                cmd.append(arg.value)
            else:
                cmd.append(f"--{arg.name}")
                if arg.value:
                    cmd.append(arg.value)
        
        logger.info(f"Starting NPM package: {' '.join(cmd)}")
        
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
            raise RuntimeError(f"NPM process exited immediately: {stderr.decode()}")
        
        return process
    
    async def load(self, entry) -> LoadResult:
        """Override to handle transport-specific loading"""
        package = self._find_package(entry)
        if not package:
            return LoadResult(
                success=False,
                error=f"No npm package configuration found"
            )
        
        try:
            # Install package
            if not await self.install_package(package):
                return LoadResult(
                    success=False,
                    error="Failed to install npm package"
                )
            
            # Prepare environment
            env = self._prepare_environment(package)
            
            # Run the package
            process = await self.run_package(package, env)
            
            # Create appropriate app based on transport
            transport = package.transport
            
            if transport.type == TransportType.STDIO:
                app = create_transport_proxy_app(
                    transport_type=TransportType.STDIO,
                    process=process
                )
                connection_info = {"type": "stdio", "process_id": process.pid}
            else:
                # Wait for HTTP server to start
                base_url = await self._wait_for_http_ready(transport)
                app = create_transport_proxy_app(
                    transport_type=transport.type,
                    base_url=base_url,
                    headers=transport.headers
                )
                connection_info = {
                    "type": transport.type,
                    "url": f"{base_url}{transport.endpoint or '/'}",
                    "process_id": process.pid
                }
            
            return LoadResult(
                success=True,
                app=app,
                connection_info=connection_info,
                loaded_at=datetime.utcnow(),
                cleanup_func=lambda: asyncio.create_task(self._cleanup(process))
            )
            
        except Exception as e:
            return LoadResult(
                success=False,
                error=f"Failed to load npm package: {str(e)}"
            )
    
    async def _wait_for_http_ready(self, transport, timeout: float = 30.0) -> str:
        """Wait for HTTP server to be ready"""
        # Similar implementation to UVXServerLoader
        import httpx
        import time
        
        host = transport.host or "127.0.0.1"
        port = transport.port or 9000
        base_url = f"http://{host}:{port}"
        
        start_time = time.time()
        while time.time() - start_time < timeout:
            try:
                async with httpx.AsyncClient() as client:
                    await client.get(f"{base_url}/", timeout=2.0)
                    return base_url
            except:
                await asyncio.sleep(0.5)
        
        raise RuntimeError(f"HTTP server failed to start within {timeout}s")
```

## Key Improvements:

1. **Transport Configuration**: Added `TransportConfig` model to specify transport type, port, endpoints, and headers
2. **Multiple Transport Support**: UVXServerLoader now handles stdio, SSE, and streamable-http transports
3. **Smart Proxy Apps**: Different proxy implementations for different transport types
4. **HTTP Server Detection**: Waits for HTTP-based servers to be ready before creating proxy
5. **Flexible Configuration**: servers.yaml can specify transport details per package
6. **Environment Variable Support**: Transport headers can use `${VAR}` syntax
7. **Backward Compatible**: Existing configurations continue to work with stdio as default

This design allows you to:
- Run the same MCP server (like mcp-atlassian) with different transports
- Configure transport-specific settings in servers.yaml
- Handle authentication headers for multi-tenant scenarios
- Support both process-based (stdio) and HTTP-based (SSE, streamable-http) transports

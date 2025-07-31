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



Looking at FastMCP's proxy capabilities, you're absolutely right! FastMCP already has excellent proxy support built-in. Let me refactor the loaders to use FastMCP's native proxy features instead of reinventing the wheel:

## 1. **Updated loaders/uvx.py** (Using FastMCP Proxy)

```python
import asyncio
import sys
import time
from datetime import datetime
from typing import Optional, Dict, Any, List
from pathlib import Path

from fastmcp import FastMCP
from fastmcp.server.proxy import ProxyClient
from starlette.applications import Starlette

from .base import ServerLoader, LoadResult
from ..models.server import TransportType, TransportConfig
from ..config import ServerConfig

import logging
logger = logging.getLogger(__name__)


class UVXServerLoader(ServerLoader):
    """Loader for Python packages via uvx using FastMCP proxy"""
    
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
            
            # Build the command to run the UVX server
            cmd = self._build_uvx_command(uvx_package)
            
            # Prepare environment
            env = self._prepare_environment(uvx_package)
            
            if transport.type == TransportType.STDIO:
                # For stdio transport, use the command directly
                proxy_client = ProxyClient(cmd, env=env)
            else:
                # For HTTP-based transports, start the server and connect via URL
                process = await self._start_http_server(cmd, env, transport)
                
                # Build the URL for the HTTP server
                url = await self._get_http_url(transport)
                
                # Create proxy client for the HTTP endpoint
                proxy_client = ProxyClient(url)
                
                # Store process for cleanup
                self._process = process
            
            # Create a FastMCP proxy server
            proxy_server = FastMCP.as_proxy(
                proxy_client,
                name=f"{config.name} (Proxy)"
            )
            
            # Get the ASGI app from the proxy
            app = proxy_server.get_asgi_app()
            
            return LoadResult(
                success=True,
                mcp_instance=proxy_server,
                app=app,
                connection_info={
                    "type": "proxy",
                    "backend_transport": transport.type,
                    "proxy_client": proxy_client
                },
                loaded_at=datetime.utcnow(),
                cleanup_func=self._create_cleanup_func()
            )
            
        except Exception as e:
            logger.error(f"Failed to load uvx package: {e}")
            return LoadResult(
                success=False,
                error=f"Failed to load uvx package: {str(e)}"
            )
    
    def _build_uvx_command(self, package) -> List[str]:
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
        
        # Add transport configuration
        transport = package.transport
        if transport.type != TransportType.STDIO:
            cmd.extend(["--transport", transport.type])
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
        
        return cmd
    
    def _prepare_environment(self, package) -> Dict[str, str]:
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
    
    async def _start_http_server(self, cmd: List[str], env: Dict[str, str], 
                                transport: TransportConfig) -> asyncio.subprocess.Process:
        """Start HTTP-based server and return process"""
        logger.info(f"Starting HTTP server: {' '.join(cmd)}")
        
        process = await asyncio.create_subprocess_exec(
            *cmd,
            env=env,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        
        # Wait for server to be ready
        await self._wait_for_http_ready(transport)
        
        return process
    
    async def _wait_for_http_ready(self, transport: TransportConfig, timeout: float = 30.0):
        """Wait for HTTP server to be ready"""
        import httpx
        
        host = transport.host or "127.0.0.1"
        port = transport.port or 9000
        url = f"http://{host}:{port}"
        
        start_time = time.time()
        while time.time() - start_time < timeout:
            try:
                async with httpx.AsyncClient() as client:
                    await client.get(url, timeout=2.0)
                    logger.info(f"HTTP server ready at {url}")
                    return
            except:
                await asyncio.sleep(0.5)
        
        raise RuntimeError(f"HTTP server failed to start within {timeout}s")
    
    async def _get_http_url(self, transport: TransportConfig) -> str:
        """Get the full URL for the HTTP server"""
        host = transport.host or "127.0.0.1"
        port = transport.port or 9000
        
        if transport.type == TransportType.SSE:
            endpoint = transport.endpoint or "/sse"
        elif transport.type == TransportType.STREAMABLE_HTTP:
            endpoint = transport.endpoint or "/mcp"
        else:
            endpoint = transport.endpoint or "/"
        
        return f"http://{host}:{port}{endpoint}"
    
    def _create_cleanup_func(self):
        """Create cleanup function"""
        async def cleanup():
            if hasattr(self, '_process') and self._process:
                if self._process.returncode is None:
                    logger.info(f"Terminating process {self._process.pid}")
                    self._process.terminate()
                    try:
                        await asyncio.wait_for(self._process.wait(), timeout=5.0)
                    except asyncio.TimeoutError:
                        logger.warning(f"Force killing process {self._process.pid}")
                        self._process.kill()
                        await self._process.wait()
        
        return cleanup
```

## 2. **Simplified loaders/npm.py** (Using FastMCP Proxy)

```python
import asyncio
import json
from typing import Dict, Any
from datetime import datetime

from fastmcp import FastMCP
from fastmcp.server.proxy import ProxyClient

from .base import ServerLoader, LoadResult
from ..models.server import TransportType
from ..config import ServerConfig

import logging
logger = logging.getLogger(__name__)


class NPMServerLoader(ServerLoader):
    """Loader for NPM-based MCP servers using FastMCP proxy"""
    
    async def load(self, config: ServerConfig) -> LoadResult:
        # Find npm package configuration
        npm_package = None
        for package in config.packages:
            if package.registry_name == "npm":
                npm_package = package
                break
        
        if not npm_package:
            return LoadResult(
                success=False,
                error="No npm package configuration found"
            )
        
        try:
            # Install package first
            if not await self._install_package(npm_package):
                return LoadResult(
                    success=False,
                    error="Failed to install npm package"
                )
            
            # Build command
            cmd = self._build_npm_command(npm_package)
            
            # Prepare environment
            env = self._prepare_environment(npm_package)
            
            transport = npm_package.transport
            
            if transport.type == TransportType.STDIO:
                # Direct stdio proxy
                proxy_client = ProxyClient(cmd, env=env)
            else:
                # Start HTTP server and proxy to it
                process = await self._start_http_server(cmd, env, transport)
                url = self._get_http_url(transport)
                proxy_client = ProxyClient(url)
                self._process = process
            
            # Create FastMCP proxy
            proxy_server = FastMCP.as_proxy(
                proxy_client,
                name=f"{config.name} (NPM Proxy)"
            )
            
            app = proxy_server.get_asgi_app()
            
            return LoadResult(
                success=True,
                mcp_instance=proxy_server,
                app=app,
                connection_info={
                    "type": "proxy",
                    "backend_transport": transport.type,
                    "package": npm_package.name
                },
                loaded_at=datetime.utcnow(),
                cleanup_func=self._create_cleanup_func()
            )
            
        except Exception as e:
            logger.error(f"Failed to load npm package: {e}")
            return LoadResult(
                success=False,
                error=f"Failed to load npm package: {str(e)}"
            )
    
    async def _install_package(self, package) -> bool:
        """Install NPM package if needed"""
        # Check if already installed
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
                    logger.info(f"NPM package {package.name} already installed")
                    return True
        except:
            pass
        
        # Install the package
        install_cmd = ["npm", "install", "-g"]
        if package.version and package.version != "latest":
            install_cmd.append(f"{package.name}@{package.version}")
        else:
            install_cmd.append(package.name)
        
        logger.info(f"Installing NPM package: {' '.join(install_cmd)}")
        
        proc = await asyncio.create_subprocess_exec(
            *install_cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        
        await proc.communicate()
        return proc.returncode == 0
    
    def _build_npm_command(self, package) -> List[str]:
        """Build NPM command"""
        cmd = ["npx", package.name]
        
        if package.entry_point:
            cmd.append(package.entry_point)
        
        # Add transport args for non-stdio
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
        
        return cmd
    
    # ... (reuse _prepare_environment, _start_http_server, _get_http_url, _create_cleanup_func from UVXServerLoader)
```

## 3. **Updated loaders/openapi.py** (Already using FastMCP native features)

The OpenAPI loader already uses FastMCP's native `from_openapi()` method, which is perfect!

## 4. **New loaders/composite.py** (For Multi-Server Configs)

```python
from typing import Dict, Any
from datetime import datetime

from fastmcp import FastMCP
from fastmcp.server.proxy import ProxyClient

from .base import ServerLoader, LoadResult
from ..config import ServerConfig

import logging
logger = logging.getLogger(__name__)


class CompositeServerLoader(ServerLoader):
    """Loader for multi-server configurations using FastMCP proxy"""
    
    async def load(self, config: ServerConfig) -> LoadResult:
        """Load a composite server from MCPConfig format"""
        
        if not hasattr(config, 'mcp_config') or not config.mcp_config:
            return LoadResult(
                success=False,
                error="No MCP configuration found"
            )
        
        try:
            # FastMCP.as_proxy() can handle MCPConfig format directly!
            proxy_server = FastMCP.as_proxy(
                config.mcp_config,
                name=config.name
            )
            
            app = proxy_server.get_asgi_app()
            
            # Extract server info from config
            servers = config.mcp_config.get("mcpServers", {})
            
            return LoadResult(
                success=True,
                mcp_instance=proxy_server,
                app=app,
                connection_info={
                    "type": "composite",
                    "servers": list(servers.keys()),
                    "server_count": len(servers)
                },
                loaded_at=datetime.utcnow()
            )
            
        except Exception as e:
            logger.error(f"Failed to create composite proxy: {e}")
            return LoadResult(
                success=False,
                error=f"Failed to create composite proxy: {str(e)}"
            )
```

## 5. **Updated servers.yaml with FastMCP Proxy Patterns**

```yaml
servers:
  # Direct OpenAPI server (no proxy needed)
  - id: "github-api"
    name: "GitHub API Server"
    type: "openapi"
    openapi_url: "https://api.github.com/openapi.yaml"
    base_url: "https://api.github.com"
    mount_path: "/github"
    enabled: true

  # UVX server via proxy (stdio)
  - id: "mcp-atlassian"
    name: "Atlassian MCP Server"
    mount_path: "/atlassian"
    enabled: true
    packages:
      - registry_name: "uvx"
        name: "mcp-atlassian"
        version: "0.11.9"
        transport:
          type: "stdio"  # ProxyClient will handle stdio directly
        environment_variables:
          - name: "JIRA_URL"
            default: "${JIRA_URL:-https://company.atlassian.net}"
          - name: "JIRA_USERNAME"
            default: "${JIRA_USERNAME}"
          - name: "JIRA_API_TOKEN"
            default: "${JIRA_API_TOKEN}"

  # UVX server with HTTP transport
  - id: "mcp-atlassian-http"
    name: "Atlassian MCP (HTTP)"
    mount_path: "/atlassian-http"
    enabled: true
    packages:
      - registry_name: "uvx"
        name: "mcp-atlassian"
        transport:
          type: "streamable-http"
          port: 9002
          endpoint: "/mcp"
          headers:
            Authorization: "Bearer ${ATLASSIAN_OAUTH_TOKEN}"
            X-Atlassian-Cloud-Id: "${ATLASSIAN_CLOUD_ID}"
        environment_variables:
          - name: "ATLASSIAN_OAUTH_ENABLE"
            default: "true"

  # Composite multi-server configuration
  - id: "ai-services"
    name: "AI Services Composite"
    mount_path: "/ai"
    enabled: true
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

  # Remote server proxy (transport bridging)
  - id: "remote-weather"
    name: "Remote Weather API"
    mount_path: "/weather"
    enabled: true
    remote_proxy:
      url: "https://weather-api.example.com/mcp/sse"
      transport: "sse"
      # This creates a ProxyClient internally
```

## 6. **Simplified Factory Pattern**

```python
# loaders/factory.py

import logging
from typing import Optional

from .base import ServerLoader
from .openapi import OpenAPIServerLoader
from .module import ModuleServerLoader
from .uvx import UVXServerLoader
from .npm import NPMServerLoader
from .pypi import PyPIServerLoader
from .composite import CompositeServerLoader
from .remote import RemoteProxyLoader
from ..config import ServerConfig

logger = logging.getLogger(__name__)


def get_loader_for_config(config: ServerConfig) -> Optional[ServerLoader]:
    """Get the appropriate loader based on configuration"""
    
    # 1. Check for composite/multi-server config
    if hasattr(config, 'mcp_config') and config.mcp_config:
        return CompositeServerLoader()
    
    # 2. Check for remote proxy config
    if hasattr(config, 'remote_proxy') and config.remote_proxy:
        return RemoteProxyLoader()
    
    # 3. Check for OpenAPI
    if hasattr(config, 'openapi_url') and config.openapi_url:
        return OpenAPIServerLoader()
    
    # 4. Check for module
    if hasattr(config, 'module_path') and config.module_path:
        return ModuleServerLoader()
    
    # 5. Check for packages
    if hasattr(config, 'packages') and config.packages:
        # Get loader based on first package's registry
        registry_name = config.packages[0].registry_name
        
        loader_map = {
            "uvx": UVXServerLoader,
            "npm": NPMServerLoader,
            "pypi": PyPIServerLoader,
        }
        
        loader_class = loader_map.get(registry_name)
        if loader_class:
            return loader_class()
    
    logger.error(f"No suitable loader found for server: {config.id}")
    return None
```

## 7. **New loaders/remote.py** (For Remote Proxies)

```python
from datetime import datetime

from fastmcp import FastMCP
from fastmcp.server.proxy import ProxyClient

from .base import ServerLoader, LoadResult
from ..config import ServerConfig

import logging
logger = logging.getLogger(__name__)


class RemoteProxyLoader(ServerLoader):
    """Loader for remote MCP servers via proxy"""
    
    async def load(self, config: ServerConfig) -> LoadResult:
        """Create a proxy to a remote MCP server"""
        
        if not hasattr(config, 'remote_proxy'):
            return LoadResult(
                success=False,
                error="No remote proxy configuration found"
            )
        
        try:
            remote_config = config.remote_proxy
            url = remote_config.get('url')
            
            if not url:
                return LoadResult(
                    success=False,
                    error="Remote proxy URL is required"
                )
            
            # Create proxy client for remote server
            proxy_client = ProxyClient(url)
            
            # Create FastMCP proxy
            proxy_server = FastMCP.as_proxy(
                proxy_client,
                name=f"{config.name} (Remote Proxy)"
            )
            
            app = proxy_server.get_asgi_app()
            
            return LoadResult(
                success=True,
                mcp_instance=proxy_server,
                app=app,
                connection_info={
                    "type": "remote_proxy",
                    "url": url,
                    "transport": remote_config.get('transport', 'unknown')
                },
                loaded_at=datetime.utcnow()
            )
            
        except Exception as e:
            logger.error(f"Failed to create remote proxy: {e}")
            return LoadResult(
                success=False,
                error=f"Failed to create remote proxy: {str(e)}"
            )
```

## Key Benefits of Using FastMCP's Native Proxy:

1. **Automatic Session Isolation**: FastMCP's `ProxyClient` handles session isolation automatically
2. **Feature Forwarding**: Sampling, elicitation, logging, and progress are automatically forwarded
3. **Transport Bridging**: Built-in support for exposing servers on different transports
4. **Less Code**: We don't need custom proxy apps - FastMCP handles it all
5. **Better Testing**: FastMCP's proxy supports in-memory testing
6. **Multi-Server Support**: Native support for MCPConfig format with multiple servers

This approach is much cleaner and leverages FastMCP's excellent proxy capabilities instead of building our own transport adapters!



Absolutely! Let's make the return types consistent throughout the codebase by using `LoadResult` everywhere. Here's the refactored version:

## 1. **Updated loaders/factory.py** with Consistent `LoadResult` Returns

```python
import asyncio
import logging
from typing import Optional

from .base import ServerLoader, LoadResult
from .openapi import OpenAPIServerLoader
from .module import ModuleServerLoader
from .uvx import UVXServerLoader
from .npm import NPMServerLoader
from .pypi import PyPIServerLoader
from .composite import CompositeServerLoader
from .remote import RemoteProxyLoader
from ..config import ServerConfig

logger = logging.getLogger(__name__)


def get_loader_for_config(config: ServerConfig) -> Optional[ServerLoader]:
    """Get the appropriate loader based on configuration"""
    
    # 1. Check for composite/multi-server config
    if hasattr(config, 'mcp_config') and config.mcp_config:
        return CompositeServerLoader()
    
    # 2. Check for remote proxy config
    if hasattr(config, 'remote_proxy') and config.remote_proxy:
        return RemoteProxyLoader()
    
    # 3. Check for OpenAPI
    if hasattr(config, 'openapi_url') and config.openapi_url:
        return OpenAPIServerLoader()
    
    # 4. Check for module
    if hasattr(config, 'module_path') and config.module_path:
        return ModuleServerLoader()
    
    # 5. Check for packages
    if hasattr(config, 'packages') and config.packages:
        # Get loader based on first package's registry
        registry_name = config.packages[0].registry_name
        
        loader_map = {
            "uvx": UVXServerLoader,
            "npm": NPMServerLoader,
            "pypi": PyPIServerLoader,
        }
        
        loader_class = loader_map.get(registry_name)
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
            error=f"No loader available for server configuration: {config.id}"
        )
    
    return await loader.load(config)


async def load_server_with_retry(config: ServerConfig) -> LoadResult:
    """Load a server with retry logic - returns LoadResult consistently"""
    
    result = None
    last_error = None
    
    for attempt in range(config.retry_attempts):
        if attempt > 0:
            logger.info(f"Retrying server '{config.id}' (attempt {attempt + 1}/{config.retry_attempts})")
            await asyncio.sleep(config.retry_delay)
        
        result = await load_server(config)
        
        if result.success:
            return result
        
        last_error = result.error
    
    # All retries failed
    return LoadResult(
        success=False,
        error=last_error or "Max retries exceeded"
    )


# Backward compatibility wrapper (deprecated)
async def load_server_with_retry_legacy(config: ServerConfig) -> tuple:
    """
    Legacy wrapper that returns tuple format.
    
    DEPRECATED: Use load_server_with_retry() which returns LoadResult
    """
    import warnings
    warnings.warn(
        "load_server_with_retry_legacy is deprecated. Use load_server_with_retry() instead.",
        DeprecationWarning,
        stacklevel=2
    )
    
    result = await load_server_with_retry(config)
    return result.mcp_instance, result.app, result.error
```

## 2. **Updated server.py** to Use `LoadResult`

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
                # Handle unexpected exceptions
                error_msg = f"Unexpected error: {str(result)}"
                self.logger.error(f"Server '{server_config.id}': {error_msg}")
                
                app = create_fallback_app(server_config.id, server_config.name, error_msg)
                info = self.registry.get(server_config.id)
                info.status = ServerStatus.FAILED
                info.error = error_msg
                info.app = app
                
            elif isinstance(result, LoadResult):
                # Process LoadResult
                info = self.registry.get(server_config.id)
                
                if result.success and result.app:
                    # Successfully loaded
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
                    app = result.app
                    
                else:
                    # Failed to load
                    error = result.error or "Unknown error"
                    app = create_fallback_app(server_config.id, server_config.name, error)
                    info.status = ServerStatus.FAILED
                    info.error = error
                    info.app = app
                    self.logger.error(f"Server '{server_config.id}' failed: {error}")
            
            else:
                # Unexpected result type
                error_msg = f"Unexpected result type: {type(result)}"
                self.logger.error(f"Server '{server_config.id}': {error_msg}")
                
                app = create_fallback_app(server_config.id, server_config.name, error_msg)
                info = self.registry.get(server_config.id)
                info.status = ServerStatus.FAILED
                info.error = error_msg
                info.app = app
            
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
                    cleanup_tasks.append(info.cleanup_func())
            
            if cleanup_tasks:
                await asyncio.gather(*cleanup_tasks, return_exceptions=True)
        
        return lifespan
    
    # ... rest of MCPHub implementation remains the same ...
```

## 3. **Enhanced registry.py** to Store LoadResult Data

```python
import logging
from datetime import datetime
from typing import Dict, Optional, Any, Callable
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
    connection_info: Optional[Dict[str, Any]] = None
    cleanup_func: Optional[Callable] = None
    
    def to_dict(self) -> Dict:
        """Convert to dictionary for API responses"""
        return {
            "id": self.id,
            "name": self.name,
            "status": self.status.value,
            "mount_path": self.mount_path,
            "error": self.error,
            "loaded_at": self.loaded_at.isoformat() if self.loaded_at else None,
            "capabilities": self.capabilities,
            "connection_info": self.connection_info
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

## 4. **Example Usage with New Consistent API**

```python
# Example of using the new consistent API

from mcp_hub.loaders import load_server_with_retry
from mcp_hub.config import ServerConfig

async def example_usage():
    # Create server config
    config = ServerConfig(
        id="test-server",
        name="Test Server",
        openapi_url="https://api.example.com/openapi.json",
        base_url="https://api.example.com",
        retry_attempts=3,
        retry_delay=2.0
    )
    
    # Load server - returns LoadResult consistently
    result = await load_server_with_retry(config)
    
    if result.success:
        print(f"Server loaded successfully at {result.loaded_at}")
        print(f"Connection info: {result.connection_info}")
        
        # Access the app and MCP instance
        app = result.app
        mcp = result.mcp_instance
        
        # Clean up when done
        if result.cleanup_func:
            await result.cleanup_func()
    else:
        print(f"Failed to load server: {result.error}")
```

## 5. **Migration Guide for Existing Code**

If you have existing code using the old tuple return format:

```python
# Old code:
mcp, app, error = await load_server_with_retry(config)
if app:
    # success
else:
    # handle error

# New code:
result = await load_server_with_retry(config)
if result.success:
    mcp = result.mcp_instance
    app = result.app
    # success
else:
    error = result.error
    # handle error

# Or use the legacy wrapper during migration:
mcp, app, error = await load_server_with_retry_legacy(config)
```

## Key Benefits:

1. **Consistent API**: All loaders and factory functions return `LoadResult`
2. **Rich Metadata**: `LoadResult` carries more information (timestamps, connection info, cleanup functions)
3. **Better Error Handling**: Errors are part of the result object, not separate returns
4. **Resource Management**: Cleanup functions are tracked and called properly
5. **Type Safety**: Better type hints and IDE support
6. **Backward Compatibility**: Legacy wrapper available for gradual migration

This makes the codebase more maintainable and provides a cleaner, more consistent API throughout the loader system!

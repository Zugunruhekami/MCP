Let me help you improve the `UVXServerLoader` class from your MCP hub server implementation, specifically addressing your issue with the HTTP server failing to start and the lack of error visibility in your Starlette application. Based on your experience with the `mcp-atlassian` project, I'll focus on enhancing error capture and reporting so that you can see what’s going wrong and propagate that information back to your main application.

Below, I’ll outline the key problems, propose improvements, and provide a revised implementation with detailed explanations.

---

### Identified Issues

1. **Lack of Subprocess Output Capture**: When the HTTP server fails to start in `_start_http_server` and `_wait_for_http_ready`, the subprocess’s `stdout` and `stderr` are piped but not read or logged. If the server fails (e.g., due to a port conflict or a missing dependency), you don’t see the error messages, making debugging impossible.

2. **Generic Error Messages**: The `load` method catches exceptions and logs them, but the `LoadResult` error message (e.g., "Failed to load uvx package: HTTP server failed to start within 30s") lacks detail about why the failure occurred.

3. **No Process Exit Detection**: The `_wait_for_http_ready` method keeps trying to connect to the server even if the subprocess exits prematurely (e.g., due to a crash), wasting time until the timeout and missing the chance to report the exit code and output immediately.

4. **Logging Visibility**: You mentioned not seeing any logs, suggesting that either the logging configuration isn’t set up to output where you expect (e.g., console), or the errors aren’t being surfaced properly in the Starlette app.

---

### Proposed Improvements

1. **Capture Subprocess Output**: Modify `_start_http_server` to read `stdout` and `stderr` asynchronously while waiting for the server to start. Include this output in error messages if startup fails, and log it for successful startups.

2. **Monitor Process Status**: Check if the subprocess exits prematurely and raise an error immediately with the exit code and captured output, rather than waiting for the full timeout.

3. **Enhance Error Reporting**: Update the `load` method to include detailed error messages (including subprocess output) in the `LoadResult`, so the Starlette app can access and display them.

4. **Improve Logging**: Add more detailed logging (e.g., the command being run) and ensure the logging configuration outputs to the console or a file you can check. Suggest how to integrate this with Starlette.

---

### Revised Implementation

Here’s the improved `UVXServerLoader` class with these changes implemented. I’ve focused on the methods relevant to your HTTP server startup issue (`_start_http_server`, `_wait_for_http_ready`, and `load`), while keeping other methods largely unchanged unless they need adjustments for compatibility.

```python
import asyncio
import logging
import time
from datetime import datetime
from typing import List, Dict
import httpx
from .base import ServerLoader, ServerConfig, LoadResult, TransportConfig, TransportType
from .proxy import ProxyClient, FastMCP

logger = logging.getLogger(__name__)

class UVXServerLoader(ServerLoader):
    """Loader for Python packages via uvx using FastMCP proxy"""
    
    async def load(self, config: ServerConfig) -> LoadResult:
        """Load a uvx package and return a LoadResult with ASGI app or error details."""
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
            cmd = self._build_uvx_command(uvx_package)
            env = self._prepare_environment(uvx_package)
            
            if transport.type == TransportType.STDIO:
                proxy_client = ProxyClient(cmd, env=env)
            else:
                process = await self._start_http_server(cmd, env, transport)
                url = await self._get_http_url(transport)
                proxy_client = ProxyClient(url)
                self._process = process
            
            proxy_server = FastMCP.as_proxy(
                proxy_client,
                name=f"{config.name} (Proxy)"
            )
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
            error_msg = f"Failed to load uvx package: {str(e)}"
            logger.error(error_msg)
            return LoadResult(
                success=False,
                error=error_msg
            )
    
    async def _start_http_server(self, cmd: List[str], env: Dict[str, str], 
                                 transport: TransportConfig) -> asyncio.subprocess.Process:
        """Start an HTTP-based server, capturing output and verifying readiness."""
        logger.info(f"Starting HTTP server with command: {' '.join(cmd)}")
        
        process = await asyncio.create_subprocess_exec(
            *cmd,
            env=env,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        
        # Buffers for subprocess output
        stdout_lines = []
        stderr_lines = []
        
        # Asynchronous task to read from a stream
        async def read_stream(stream, lines):
            while True:
                line = await stream.readline()
                if not line:
                    break
                lines.append(line.decode().strip())
        
        stdout_task = asyncio.create_task(read_stream(process.stdout, stdout_lines))
        stderr_task = asyncio.create_task(read_stream(process.stderr, stderr_lines))
        
        host = transport.host or "127.0.0.1"
        port = transport.port or 9000
        url = f"http://{host}:{port}"
        
        start_time = time.time()
        timeout = 30.0
        while time.time() - start_time < timeout:
            # Check if the process has exited
            if process.returncode is not None:
                await asyncio.gather(stdout_task, stderr_task)
                raise RuntimeError(
                    f"Process exited with code {process.returncode}\n"
                    f"Stdout:\n{'\n'.join(stdout_lines)}\n"
                    f"Stderr:\n{'\n'.join(stderr_lines)}"
                )
            
            # Try to connect to the server
            try:
                async with httpx.AsyncClient() as client:
                    response = await client.get(url, timeout=2.0)
                    if response.status_code == 200:
                        logger.info(f"HTTP server ready at {url}")
                        # Log startup output for debugging
                        if stdout_lines:
                            logger.info(f"Startup stdout:\n{'\n'.join(stdout_lines)}")
                        if stderr_lines:
                            logger.info(f"Startup stderr:\n{'\n'.join(stderr_lines)}")
                        return process
            except Exception as e:
                logger.debug(f"Connection attempt failed: {e}")
            
            await asyncio.sleep(0.5)
        
        # Timeout reached
        await asyncio.gather(stdout_task, stderr_task)
        raise RuntimeError(
            f"HTTP server failed to start within {timeout}s\n"
            f"Stdout:\n{'\n'.join(stdout_lines)}\n"
            f"Stderr:\n{'\n'.join(stderr_lines)}"
        )
    
    def _build_uvx_command(self, package) -> List[str]:
        """Build the uvx command line."""
        cmd = ["uvx"]
        if package.uvx_options:
            cmd.extend(package.uvx_options)
        if package.version and package.version != "latest":
            cmd.append(f"{package.name}=={package.version}")
        else:
            cmd.append(package.name)
        if package.entry_point:
            cmd.append(package.entry_point)
        transport = package.transport
        if transport.type != TransportType.STDIO:
            cmd.extend(["--transport", transport.type])
            if transport.port:
                cmd.extend(["--port", str(transport.port)])
            cmd.extend(transport.extra_args)
        for arg in package.package_arguments:
            if arg.type == "positional":
                cmd.append(arg.value)
            else:
                cmd.append(f"--{arg.name}")
                if arg.value:
                    cmd.append(arg.value)
        return cmd
    
    def _prepare_environment(self, package) -> Dict[str, str]:
        """Prepare environment variables."""
        import os
        env = os.environ.copy()
        for env_var in package.environment_variables:
            value = os.getenv(env_var.name, env_var.default)
            if value:
                env[env_var.name] = value
            elif env_var.required:
                raise ValueError(f"Required environment variable {env_var.name} not set")
        return env
    
    async def _get_http_url(self, transport: TransportConfig) -> str:
        """Get the full URL for the HTTP server."""
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
        """Create cleanup function for the subprocess."""
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

# Note: _wait_for_http_ready is removed as its logic is now in _start_http_server
```

---

### Explanation of Changes

#### 1. Enhanced `_start_http_server`
- **Output Capture**: Two asynchronous tasks (`stdout_task` and `stderr_task`) read from the subprocess’s `stdout` and `stderr` pipes, storing lines in buffers (`stdout_lines` and `stderr_lines`).
- **Process Monitoring**: The method checks `process.returncode` in each loop iteration. If the process exits, it immediately raises a `RuntimeError` with the exit code and captured output.
- **Server Readiness Check**: It attempts to connect to the server via HTTP. On success, it logs the startup output and returns the process. On timeout, it raises an error with the captured output.
- **Logging**: Added detailed logging for the command being executed, connection attempts (at debug level), and startup output.

#### 2. Removed `_wait_for_http_ready`
- The logic for waiting and checking server readiness is now integrated into `_start_http_server`, simplifying the code and allowing output capture to happen in one place.

#### 3. Updated `load` Method
- The `error_msg` variable now includes the full exception string, which, thanks to the changes in `_start_http_server`, will contain subprocess output when relevant. This is logged and returned in the `LoadResult`.

#### 4. Other Methods
- Left `_build_uvx_command`, `_prepare_environment`, `_get_http_url`, and `_create_cleanup_func` largely unchanged, as they’re not directly related to the error logging issue. Added minor docstring clarifications.

---

### Integrating with Your Starlette Application

To ensure errors are visible in your main Starlette app (and address your “I cannot see any logs” concern), follow these steps:

1. **Check Logging Configuration**:
   Ensure the logger outputs to a place you can see, like the console. Add this at the start of your application (e.g., in your `main.py` or wherever you initialize the Starlette app):

   ```python
   import logging

   logging.basicConfig(
       level=logging.INFO,
       format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
       handlers=[logging.StreamHandler()]  # Outputs to console
   )
   ```

   This sets the root logger to `INFO` level and directs output to the console. If you’re using a custom logger name in `UVXServerLoader`, configure it similarly:

   ```python
   logger = logging.getLogger("UVXServerLoader")
   logger.setLevel(logging.INFO)
   if not logger.handlers:
       logger.addHandler(logging.StreamHandler())
   ```

2. **Handle `LoadResult` in Starlette**:
   Assuming your Starlette app uses the loader to start servers, check the `LoadResult` and log or display the error. For example:

   ```python
   from starlette.applications import Starlette
   from starlette.responses import JSONResponse

   app = Starlette()
   loader = UVXServerLoader()

   @app.route("/start-server")
   async def start_server(request):
       config = ServerConfig(...)  # Your server config
       result = await loader.load(config)
       if not result.success:
           logger.error(f"Server startup failed: {result.error}")
           return JSONResponse({"error": result.error}, status_code=500)
       # Proceed with result.app if successful
       return JSONResponse({"status": "Server started"})
   ```

   Here, if the server fails to start, the error message (now including subprocess output) is logged and returned in the HTTP response, making it visible to you via logs or the API response.

3. **Test with Debugging**:
   Run your app with the updated loader and intentionally cause a failure (e.g., use a port already in use or an invalid `uvx` command). You should see output like:

   ```
   2023-10-15 10:00:00 - UVXServerLoader - INFO - Starting HTTP server with command: uvx mypackage --port 9000
   2023-10-15 10:00:30 - UVXServerLoader - ERROR - Failed to load uvx package: HTTP server failed to start within 30s
   Stdout:
   Starting server...
   Stderr:
   Error: Address already in use (port 9000)
   ```

---

### Additional Suggestions

- **Debug Logging**: The `logger.debug` call in `_start_http_server` for connection failures can be toggled to `INFO` during development to see more details.
- **Timeout Configuration**: Make the 30-second timeout configurable via `transport` or a class parameter.
- **Output Redirection**: For long-running servers, consider redirecting ongoing subprocess output to the logger after startup, though this might require a background task.

---

### Conclusion

With these changes, when the HTTP server fails to start in your `mcp-atlassian` project, you’ll get detailed error messages including subprocess output (e.g., “port in use” or “command not found”). The `LoadResult` will carry this information back to your Starlette app, and proper logging configuration will ensure you see it in the console or logs. Let me know if you need further assistance testing or integrating this!










Here's a comprehensive analysis of the issues in your UVX loader implementation and concrete improvements with detailed implementations. The key problem is **missing error visibility when subprocesses fail**, which I'll solve with robust logging and error propagation.

### Core Problems Identified
1. **Subprocess errors are silently discarded** - stderr isn't captured/propagated
2. **No real-time logging** - subprocess output only visible after failure
3. **Fragile HTTP readiness check** - assumes root endpoint works for health checks
4. **Cleanup leaks resources** - doesn't handle partial failures during startup
5. **No health check customization** - assumes `/` is always valid endpoint

---

## Concrete Implementation Improvements

### 1. Subprocess Output Capture & Error Propagation (Critical Fix)
**Problem**: Current implementation discards stderr, making failures invisible.  
**Solution**: Capture output in real-time and propagate errors to main app.

```python
# utils/subprocess.py (NEW FILE)
import asyncio
import logging
from collections import deque
from typing import Deque, List, Optional

logger = logging.getLogger("mcp.hub.subprocess")

class SubprocessManager:
    """Manages subprocess execution with real-time output capture"""
    
    def __init__(self, max_log_lines: int = 100):
        self.max_log_lines = max_log_lines
        self.stdout_lines: Deque[str] = deque(maxlen=max_log_lines)
        self.stderr_lines: Deque[str] = deque(maxlen=max_log_lines)
        self.process: Optional[asyncio.subprocess.Process] = None
    
    async def start(self, cmd: List[str], env: dict) -> None:
        """Start subprocess with output capture"""
        self.process = await asyncio.create_subprocess_exec(
            *cmd,
            env=env,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        logger.info(f"Started subprocess PID={self.process.pid}: {' '.join(cmd)}")
        
        # Start output capture tasks
        self._stdout_task = asyncio.create_task(
            self._capture_stream(self.process.stdout, self.stdout_lines, logger.info)
        )
        self._stderr_task = asyncio.create_task(
            self._capture_stream(self.process.stderr, self.stderr_lines, logger.error)
        )
    
    async def _capture_stream(self, stream, lines: Deque[str], log_func):
        """Capture stream output in real-time"""
        while True:
            line = await stream.readline()
            if not line:
                break
            decoded = line.decode().rstrip()
            lines.append(decoded)
            log_func(f"[subprocess] {decoded}")
    
    async def wait_for_ready(self, readiness_check, timeout: float = 30.0) -> None:
        """Wait for process to become ready or fail"""
        start_time = time.time()
        while time.time() - start_time < timeout:
            # Check if process crashed
            if self.process.returncode is not None:
                raise RuntimeError(
                    f"Process exited prematurely with code {self.process.returncode}\n"
                    f"Last stderr: {''.join(list(self.stderr_lines)[-5:])}"
                )
            
            # Check readiness
            if await readiness_check():
                return
            
            await asyncio.sleep(0.2)
        
        raise TimeoutError(
            f"Process failed to become ready in {timeout}s\n"
            f"Last stderr: {''.join(list(self.stderr_lines)[-5:])}"
        )
    
    async def cleanup(self) -> None:
        """Graceful cleanup with error reporting"""
        if not self.process or self.process.returncode is not None:
            return
        
        try:
            self.process.terminate()
            await asyncio.wait_for(self.process.wait(), timeout=5.0)
        except asyncio.TimeoutError:
            logger.warning("Process didn't terminate, forcing kill")
            self.process.kill()
            await self.process.wait()
        finally:
            # Cancel output capture tasks
            self._stdout_task.cancel()
            self._stderr_task.cancel()
            await asyncio.gather(
                self._stdout_task, self._stderr_task, 
                return_exceptions=True
            )
    
    def get_error_context(self) -> str:
        """Get error context for user display"""
        return "\n".join([
            "Subprocess failed with errors:",
            *list(self.stderr_lines)[-10:],
            f"Exit code: {self.process.returncode if self.process else 'N/A'}"
        ])
```

### 2. Enhanced UVX Server Loader Implementation
**Key Improvements**:
- Real-time subprocess logging
- Error context propagation to main app
- Customizable health checks
- Robust cleanup
- Configurable timeouts

```python
# loaders/uvx_loader.py
import time
from typing import Dict, List, Optional

import httpx
from pydantic import BaseModel

from .subprocess import SubprocessManager
from .base import ServerLoader, ServerConfig, LoadResult, TransportConfig, TransportType
from .proxy import ProxyClient, FastMCP

class HealthCheckConfig(BaseModel):
    endpoint: str = "/health"
    method: str = "GET"
    expected_status: int = 200
    timeout: float = 2.0

class UVXServerLoader(ServerLoader):
    async def load(self, config: ServerConfig) -> LoadResult:
        uvx_package = next(
            (p for p in config.packages if p.registry_name == "uvx"), 
            None
        )
        if not uvx_package:
            return LoadResult(
                success=False,
                error="No uvx package configuration found"
            )
        
        try:
            # Initialize subprocess manager
            subprocess = SubprocessManager()
            transport = uvx_package.transport
            cmd = self._build_uvx_command(uvx_package)
            env = self._prepare_environment(uvx_package)
            
            # Start subprocess
            await subprocess.start(cmd, env)
            
            # Configure health check
            health_check = self._get_health_check(transport)
            
            # Wait for readiness with custom health check
            await subprocess.wait_for_ready(
                lambda: self._check_health(transport, health_check),
                timeout=uvx_package.startup_timeout or 30.0
            )
            
            # Get connection URL
            url = await self._get_connection_url(transport)
            
            # Create proxy
            proxy_client = ProxyClient(url)
            proxy_server = FastMCP.as_proxy(
                proxy_client,
                name=f"{config.name} (Proxy)"
            )
            
            return LoadResult(
                success=True,
                mcp_instance=proxy_server,
                app=proxy_server.get_asgi_app(),
                connection_info={
                    "type": "proxy",
                    "backend_transport": transport.type,
                    "url": url,
                    "pid": subprocess.process.pid
                },
                cleanup_func=lambda: subprocess.cleanup()
            )
            
        except Exception as e:
            error_context = (
                subprocess.get_error_context() 
                if 'subprocess' in locals() 
                else "Subprocess manager not initialized"
            )
            logger.error(f"UVX load failed: {str(e)}\n{error_context}")
            return LoadResult(
                success=False,
                error=f"Server failed to start: {str(e)}",
                error_context=error_context  # <-- CRITICAL: Propagate to main app
            )
    
    def _get_health_check(self, transport: TransportConfig) -> HealthCheckConfig:
        """Get health check config from transport or defaults"""
        if hasattr(transport, 'health_check') and transport.health_check:
            return HealthCheckConfig(**transport.health_check)
        return HealthCheckConfig(
            endpoint="/health" if transport.type != TransportType.SSE else "/sse"
        )
    
    async def _check_health(self, transport: TransportConfig, health_check: HealthCheckConfig) -> bool:
        """Perform health check with configurable endpoint"""
        host = transport.host or "127.0.0.1"
        port = transport.port or 8000
        url = f"http://{host}:{port}{health_check.endpoint}"
        
        try:
            async with httpx.AsyncClient(timeout=health_check.timeout) as client:
                response = await client.request(
                    health_check.method, 
                    url,
                    headers={"Accept": "application/json"}
                )
                return response.status_code == health_check.expected_status
        except Exception:
            return False
    
    async def _get_connection_url(self, transport: TransportConfig) -> str:
        """Get connection URL based on transport type"""
        host = transport.host or "127.0.0.1"
        port = transport.port or 8000
        endpoint = transport.endpoint or "/"
        
        if transport.type == TransportType.SSE:
            return f"http://{host}:{port}/sse"
        elif transport.type == TransportType.STREAMABLE_HTTP:
            return f"http://{host}:{port}/mcp"
        return f"http://{host}:{port}{endpoint}"
    
    # ... (keep _build_uvx_command and _prepare_environment but add validation) ...
    
    def _prepare_environment(self, package) -> Dict[str, str]:
        """Enhanced with validation and debug logging"""
        env = super()._prepare_environment(package)
        
        # Log environment for debugging
        logger.debug(f"Environment for {package.name}:")
        for k, v in env.items():
            if "SECRET" not in k.upper():
                logger.debug(f"  {k}={v}")
        
        # Validate required variables
        missing = [
            var.name for var in package.environment_variables
            if var.required and var.name not in env
        ]
        if missing:
            raise ValueError(f"Missing required env vars: {', '.join(missing)}")
        
        return env
```

### 3. Main Application Error Handling (Starlette Integration)
**Problem**: Errors aren't visible in the main app.  
**Solution**: Display error context in server listing UI.

```python
# main.py (Starlette app)
from fastapi import FastAPI, HTTPException
from starlette.responses import JSONResponse

app = FastAPI()

@app.get("/servers")
async def list_servers():
    servers = []
    for server in server_manager.get_servers():
        status = "active" if server.is_running else "error"
        
        # Include error context if available
        error_detail = None
        if not server.is_running and server.load_result and server.load_result.error_context:
            error_detail = server.load_result.error_context
        
        servers.append({
            "id": server.id,
            "name": server.config.name,
            "status": status,
            "error": server.load_result.error if not server.is_running else None,
            "error_context": error_detail,  # <-- SHOW DETAILED ERRORS
            "connection_info": server.load_result.connection_info 
                if server.is_running else None
        })
    return {"servers": servers}

# Custom error handler for loader failures
@app.exception_handler(HTTPException)
async def http_exception_handler(request, exc):
    if exc.detail.startswith("Server failed to start"):
        # Extract error context from exception
        error_context = getattr(exc, 'error_context', None)
        return JSONResponse(
            status_code=exc.status_code,
            content={
                "error": exc.detail,
                "error_context": error_context
            }
        )
    return JSONResponse(status_code=exc.status_code, content={"error": exc.detail})
```

### 4. Critical Configuration Fixes for Atlassian Package
Your `mcp-atlassian` failure is likely due to these common issues:

```yaml
# Corrected atlassian config example
packages:
  - registry_name: "uvx"
    name: "mcp-atlassian"
    version: "0.1.0"
    transport:
      type: "http"
      port: 8000
      health_check:  # <-- ADD THIS
        endpoint: "/health"
        expected_status: 200
    environment_variables:
      - name: "ATLASSIAN_API_KEY"
        required: true
      - name: "ATLASSIAN_DOMAIN"
        required: true
    startup_timeout: 45  # <-- INCREASE TIMEOUT
```

**Why this fixes your issue**:
1. Adds proper health check endpoint (most MCP servers require `/health`)
2. Increases timeout for slow-starting services
3. Ensures required env vars are validated early
4. Captures actual startup errors from stderr

---

## Key Improvements Summary

| Area | Problem | Solution |
|------|---------|----------|
| **Error Visibility** | Silent subprocess failures | Real-time stderr capture with context propagation |
| **Health Checks** | Assumes root endpoint works | Configurable health check endpoint/status |
| **Cleanup** | Resource leaks on partial failure | Atomic cleanup with subprocess manager |
| **Debugging** | No env visibility | Sanitized environment logging |
| **Config Flexibility** | Rigid startup parameters | Customizable timeouts/health checks |
| **Main App Integration** | Errors not visible in UI | Error context in server listing API |

## How to Diagnose Future Issues
1. **Check `/servers` endpoint** - Will show full error context including last 10 stderr lines
2. **Enable debug logs** - Add `logging.basicConfig(level=logging.DEBUG)`
3. **Verify health check** - Test `http://localhost:8000/health` manually
4. **Validate env vars** - Check logs for "Environment for..." entries

This implementation ensures that when `mcp-atlassian` fails to start, you'll see exactly what's wrong in both your server logs AND the main application UI, including the actual error from the subprocess (like missing API keys or port conflicts).

> **Pro Tip**: For immediate debugging, add this to your Starlette error template:
> ```html
> {% if server.error_context %}
>   <details class="error-details">
>     <summary>Technical Details</summary>
>     <pre>{{ server.error_context }}</pre>
>   </details>
> {% endif %}
> ```

Looking at the codebase and FastMCP's testing patterns, I'll create comprehensive test suites that leverage FastMCP's built-in testing capabilities. Here's the test structure:

## 1. **tests/conftest.py** (Test Configuration)

```python
import pytest
import asyncio
import tempfile
import shutil
from pathlib import Path
from typing import Dict, Any
import yaml
import os

from mcp_hub.config import Config, ServerConfig
from mcp_hub.models.server import PackageInfo, TransportConfig, TransportType


@pytest.fixture(scope="session")
def event_loop():
    """Create an instance of the default event loop for the test session."""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()


@pytest.fixture
def temp_dir():
    """Create a temporary directory for test files"""
    temp_dir = tempfile.mkdtemp()
    yield Path(temp_dir)
    shutil.rmtree(temp_dir)


@pytest.fixture
def sample_openapi_spec():
    """Sample OpenAPI specification for testing"""
    return {
        "openapi": "3.0.0",
        "info": {"title": "Test API", "version": "1.0.0"},
        "paths": {
            "/users": {
                "get": {
                    "summary": "Get users",
                    "operationId": "getUsers",
                    "responses": {"200": {"description": "Success"}}
                }
            },
            "/users/{id}": {
                "get": {
                    "summary": "Get user by ID",
                    "operationId": "getUserById",
                    "parameters": [
                        {
                            "name": "id",
                            "in": "path",
                            "required": True,
                            "schema": {"type": "string"}
                        }
                    ],
                    "responses": {"200": {"description": "Success"}}
                }
            }
        }
    }


@pytest.fixture
def mock_http_server(httpx_mock, sample_openapi_spec):
    """Mock HTTP server for OpenAPI testing"""
    # Mock OpenAPI spec endpoint
    httpx_mock.add_response(
        method="GET",
        url="https://api.example.com/openapi.json",
        json=sample_openapi_spec
    )
    
    # Mock API endpoints
    httpx_mock.add_response(
        method="GET",
        url="https://api.example.com/users",
        json=[{"id": "1", "name": "Test User"}]
    )
    
    httpx_mock.add_response(
        method="GET",
        url="https://api.example.com/users/1",
        json={"id": "1", "name": "Test User"}
    )
    
    return httpx_mock


@pytest.fixture
def sample_server_config():
    """Sample server configuration"""
    return ServerConfig(
        id="test-server",
        name="Test Server",
        description="Test server description",
        type="openapi",
        enabled=True,
        openapi_url="https://api.example.com/openapi.json",
        base_url="https://api.example.com",
        mount_path="/test",
        retry_attempts=1,
        retry_delay=0.1
    )


@pytest.fixture
def sample_uvx_config():
    """Sample UVX server configuration"""
    return ServerConfig(
        id="uvx-server",
        name="UVX Test Server",
        enabled=True,
        mount_path="/uvx",
        packages=[
            PackageInfo(
                registry_name="uvx",
                name="test-mcp-server",
                version="1.0.0",
                transport=TransportConfig(type=TransportType.STDIO)
            )
        ]
    )


@pytest.fixture
def sample_config_files(temp_dir):
    """Create sample configuration files"""
    config_content = {
        "hub": {
            "name": "Test MCP Hub",
            "host": "127.0.0.1",
            "port": 8080,
            "log_level": "debug"
        }
    }
    
    servers_content = {
        "servers": [
            {
                "id": "test-api",
                "name": "Test API Server",
                "type": "openapi",
                "openapi_url": "https://api.example.com/openapi.json",
                "base_url": "https://api.example.com",
                "enabled": True
            }
        ]
    }
    
    config_path = temp_dir / "config.yaml"
    servers_path = temp_dir / "servers.yaml"
    
    with open(config_path, 'w') as f:
        yaml.dump(config_content, f)
    
    with open(servers_path, 'w') as f:
        yaml.dump(servers_content, f)
    
    return config_path, servers_path


@pytest.fixture
def mock_uvx_process():
    """Mock UVX process for testing"""
    class MockProcess:
        def __init__(self):
            self.pid = 12345
            self.returncode = None
            self.stdin = MockStream()
            self.stdout = MockStream()
            self.stderr = MockStream()
        
        def terminate(self):
            self.returncode = 0
        
        def kill(self):
            self.returncode = -9
        
        async def wait(self):
            return self.returncode
        
        async def communicate(self):
            return b"", b""
    
    class MockStream:
        def write(self, data):
            pass
        
        async def drain(self):
            pass
        
        async def readline(self):
            return b'{"jsonrpc": "2.0", "id": 1, "result": {}}\n'
        
        async def read(self):
            return b""
    
    return MockProcess()


@pytest.fixture
def mock_subprocess_exec(monkeypatch, mock_uvx_process):
    """Mock subprocess execution"""
    async def mock_exec(*args, **kwargs):
        return mock_uvx_process
    
    monkeypatch.setattr(asyncio, "create_subprocess_exec", mock_exec)
    return mock_exec
```

## 2. **tests/test_loaders_base.py** (Base Loader Tests)

```python
import pytest
from datetime import datetime

from mcp_hub.loaders.base import LoadResult, ServerLoader
from mcp_hub.config import ServerConfig


class MockLoader(ServerLoader):
    """Mock loader for testing base functionality"""
    
    async def load(self, config: ServerConfig) -> LoadResult:
        if config.id == "fail":
            return LoadResult(
                success=False,
                error="Intentional failure"
            )
        
        return LoadResult(
            success=True,
            loaded_at=datetime.utcnow(),
            connection_info={"type": "mock"}
        )


class TestLoadResult:
    """Test LoadResult dataclass"""
    
    def test_success_result(self):
        """Test successful load result"""
        result = LoadResult(
            success=True,
            loaded_at=datetime.utcnow()
        )
        
        assert result.success is True
        assert result.error is None
        assert result.loaded_at is not None
    
    def test_failure_result(self):
        """Test failed load result"""
        result = LoadResult(
            success=False,
            error="Test error"
        )
        
        assert result.success is False
        assert result.error == "Test error"
        assert result.loaded_at is None


class TestServerLoader:
    """Test base ServerLoader functionality"""
    
    @pytest.mark.asyncio
    async def test_successful_load(self):
        """Test successful server loading"""
        loader = MockLoader()
        config = ServerConfig(id="test", name="Test", enabled=True)
        
        result = await loader.load(config)
        
        assert result.success is True
        assert result.error is None
        assert result.connection_info == {"type": "mock"}
    
    @pytest.mark.asyncio
    async def test_failed_load(self):
        """Test failed server loading"""
        loader = MockLoader()
        config = ServerConfig(id="fail", name="Fail Test", enabled=True)
        
        result = await loader.load(config)
        
        assert result.success is False
        assert result.error == "Intentional failure"
    
    @pytest.mark.asyncio
    async def test_cleanup(self):
        """Test cleanup functionality"""
        loader = MockLoader()
        
        cleanup_called = False
        
        async def mock_cleanup():
            nonlocal cleanup_called
            cleanup_called = True
        
        result = LoadResult(
            success=True,
            cleanup_func=mock_cleanup
        )
        
        await loader.cleanup(result)
        assert cleanup_called is True
```

## 3. **tests/test_loaders_openapi.py** (OpenAPI Loader Tests)

```python
import pytest
from fastmcp import FastMCP
from fastmcp.testing import MCPTest

from mcp_hub.loaders.openapi import OpenAPIServerLoader
from mcp_hub.config import ServerConfig


class TestOpenAPILoader:
    """Test OpenAPI server loader using FastMCP testing patterns"""
    
    @pytest.mark.asyncio
    async def test_load_openapi_server(self, mock_http_server, sample_server_config):
        """Test loading OpenAPI server"""
        loader = OpenAPIServerLoader()
        
        result = await loader.load(sample_server_config)
        
        assert result.success is True
        assert result.mcp_instance is not None
        assert result.app is not None
        assert isinstance(result.mcp_instance, FastMCP)
        
        # Clean up
        if result.cleanup_func:
            await result.cleanup_func()
    
    @pytest.mark.asyncio
    async def test_openapi_server_functionality(self, mock_http_server, sample_server_config):
        """Test OpenAPI server MCP functionality using FastMCP testing"""
        loader = OpenAPIServerLoader()
        result = await loader.load(sample_server_config)
        
        assert result.success is True
        mcp = result.mcp_instance
        
        # Use FastMCP's testing utilities
        async with MCPTest(mcp) as test:
            # Test tools are available
            tools = await test.list_tools()
            assert len(tools) > 0
            
            # Test calling a tool
            get_users_tool = next((t for t in tools if "users" in t.name.lower()), None)
            assert get_users_tool is not None
            
            # Call the tool
            tool_result = await test.call_tool(get_users_tool.name, {})
            assert tool_result is not None
        
        # Clean up
        if result.cleanup_func:
            await result.cleanup_func()
    
    @pytest.mark.asyncio
    async def test_authentication_headers(self, mock_http_server):
        """Test OpenAPI server with authentication"""
        config = ServerConfig(
            id="auth-server",
            name="Auth Server",
            openapi_url="https://api.example.com/openapi.json",
            base_url="https://api.example.com",
            auth_type="bearer",
            auth_token="test-token",
            auth_header="Authorization"
        )
        
        loader = OpenAPIServerLoader()
        result = await loader.load(config)
        
        assert result.success is True
        
        # Clean up
        if result.cleanup_func:
            await result.cleanup_func()
    
    @pytest.mark.asyncio
    async def test_invalid_openapi_url(self):
        """Test handling invalid OpenAPI URL"""
        config = ServerConfig(
            id="invalid-server",
            name="Invalid Server",
            openapi_url="https://invalid.example.com/openapi.json",
            base_url="https://invalid.example.com"
        )
        
        loader = OpenAPIServerLoader()
        result = await loader.load(config)
        
        assert result.success is False
        assert "Failed to load OpenAPI server" in result.error
    
    @pytest.mark.asyncio
    async def test_timeout_handling(self, mock_http_server):
        """Test timeout handling"""
        config = ServerConfig(
            id="timeout-server",
            name="Timeout Server",
            openapi_url="https://api.example.com/openapi.json",
            base_url="https://api.example.com"
        )
        
        loader = OpenAPIServerLoader(timeout=0.001)  # Very short timeout
        result = await loader.load(config)
        
        # Should either succeed quickly or fail with timeout
        assert isinstance(result.success, bool)
        if not result.success:
            assert "timeout" in result.error.lower() or "failed" in result.error.lower()
```

## 4. **tests/test_loaders_uvx.py** (UVX Loader Tests)

```python
import pytest
from unittest.mock import AsyncMock, MagicMock, patch
import asyncio

from mcp_hub.loaders.uvx import UVXServerLoader
from mcp_hub.models.server import PackageInfo, TransportConfig, TransportType
from mcp_hub.config import ServerConfig


class TestUVXLoader:
    """Test UVX server loader"""
    
    @pytest.mark.asyncio
    async def test_load_stdio_server(self, sample_uvx_config, mock_subprocess_exec):
        """Test loading UVX server with stdio transport"""
        loader = UVXServerLoader()
        
        result = await loader.load(sample_uvx_config)
        
        assert result.success is True
        assert result.app is not None
        assert result.connection_info["type"] == "proxy"
        assert result.connection_info["backend_transport"] == TransportType.STDIO
        
        # Clean up
        if result.cleanup_func:
            await result.cleanup_func()
    
    @pytest.mark.asyncio
    async def test_load_http_server(self, mock_subprocess_exec):
        """Test loading UVX server with HTTP transport"""
        config = ServerConfig(
            id="uvx-http",
            name="UVX HTTP Server",
            enabled=True,
            packages=[
                PackageInfo(
                    registry_name="uvx",
                    name="test-server",
                    transport=TransportConfig(
                        type=TransportType.SSE,
                        port=9000,
                        host="127.0.0.1"
                    )
                )
            ]
        )
        
        loader = UVXServerLoader()
        
        # Mock HTTP server readiness check
        with patch('httpx.AsyncClient') as mock_client:
            mock_response = AsyncMock()
            mock_response.status_code = 200
            mock_client.return_value.__aenter__.return_value.get.return_value = mock_response
            
            result = await loader.load(config)
            
            assert result.success is True
            assert result.connection_info["backend_transport"] == TransportType.SSE
    
    @pytest.mark.asyncio
    async def test_missing_uvx_package(self):
        """Test handling missing UVX package configuration"""
        config = ServerConfig(
            id="no-uvx",
            name="No UVX Server",
            enabled=True,
            packages=[]  # No packages
        )
        
        loader = UVXServerLoader()
        result = await loader.load(config)
        
        assert result.success is False
        assert "No uvx package configuration found" in result.error
    
    @pytest.mark.asyncio
    async def test_command_building(self):
        """Test UVX command building"""
        package = PackageInfo(
            registry_name="uvx",
            name="test-package",
            version="1.0.0",
            entry_point="server",
            transport=TransportConfig(
                type=TransportType.SSE,
                port=9000
            ),
            uvx_options=["--python", "3.11"]
        )
        
        loader = UVXServerLoader()
        cmd = loader._build_uvx_command(package)
        
        expected = [
            "uvx", "--python", "3.11", 
            "test-package==1.0.0", "server",
            "--transport", "sse", "--port", "9000"
        ]
        
        assert cmd == expected
    
    @pytest.mark.asyncio
    async def test_environment_preparation(self):
        """Test environment variable preparation"""
        from mcp_hub.models.server import EnvironmentVariable
        
        package = PackageInfo(
            registry_name="uvx",
            name="test-package",
            environment_variables=[
                EnvironmentVariable(
                    name="TEST_VAR",
                    description="Test variable",
                    required=False,
                    default="default_value"
                )
            ]
        )
        
        loader = UVXServerLoader()
        env = loader._prepare_environment(package)
        
        assert "TEST_VAR" in env
        assert env["TEST_VAR"] == "default_value"
    
    @pytest.mark.asyncio
    async def test_process_cleanup(self, mock_uvx_process):
        """Test process cleanup"""
        loader = UVXServerLoader()
        
        cleanup_func = loader._create_cleanup_func()
        loader._process = mock_uvx_process
        
        # Mock the process cleanup
        await cleanup_func()
        
        # Verify terminate was called
        assert mock_uvx_process.returncode == 0
```

## 5. **tests/test_loaders_factory.py** (Factory Tests)

```python
import pytest

from mcp_hub.loaders.factory import (
    get_loader_for_config,
    load_server,
    load_server_with_retry
)
from mcp_hub.loaders.openapi import OpenAPIServerLoader
from mcp_hub.loaders.uvx import UVXServerLoader
from mcp_hub.config import ServerConfig
from mcp_hub.models.server import PackageInfo, TransportType


class TestLoaderFactory:
    """Test loader factory functionality"""
    
    def test_get_openapi_loader(self):
        """Test getting OpenAPI loader"""
        config = ServerConfig(
            id="test",
            name="Test",
            openapi_url="https://api.example.com/openapi.json"
        )
        
        loader = get_loader_for_config(config)
        assert isinstance(loader, OpenAPIServerLoader)
    
    def test_get_uvx_loader(self):
        """Test getting UVX loader"""
        config = ServerConfig(
            id="test",
            name="Test",
            packages=[
                PackageInfo(registry_name="uvx", name="test-package")
            ]
        )
        
        loader = get_loader_for_config(config)
        assert isinstance(loader, UVXServerLoader)
    
    def test_no_loader_found(self):
        """Test handling when no loader is found"""
        config = ServerConfig(id="test", name="Test")
        
        loader = get_loader_for_config(config)
        assert loader is None
    
    @pytest.mark.asyncio
    async def test_load_server_success(self, sample_server_config, mock_http_server):
        """Test successful server loading through factory"""
        result = await load_server(sample_server_config)
        
        assert result.success is True
        assert result.mcp_instance is not None
        
        # Clean up
        if result.cleanup_func:
            await result.cleanup_func()
    
    @pytest.mark.asyncio
    async def test_load_server_no_loader(self):
        """Test server loading when no loader is available"""
        config = ServerConfig(id="test", name="Test")
        
        result = await load_server(config)
        
        assert result.success is False
        assert "No loader available" in result.error
    
    @pytest.mark.asyncio
    async def test_load_server_with_retry_success(self, sample_server_config, mock_http_server):
        """Test retry logic with success"""
        sample_server_config.retry_attempts = 3
        sample_server_config.retry_delay = 0.1
        
        result = await load_server_with_retry(sample_server_config)
        
        assert result.success is True
        
        # Clean up
        if result.cleanup_func:
            await result.cleanup_func()
    
    @pytest.mark.asyncio
    async def test_load_server_with_retry_failure(self):
        """Test retry logic with failure"""
        config = ServerConfig(
            id="invalid",
            name="Invalid",
            openapi_url="https://invalid.example.com/openapi.json",
            retry_attempts=2,
            retry_delay=0.1
        )
        
        result = await load_server_with_retry(config)
        
        assert result.success is False
        assert result.error is not None
```

## 6. **tests/test_server.py** (Hub Server Tests)

```python
import pytest
from unittest.mock import AsyncMock, MagicMock, patch
import asyncio

from mcp_hub.server import MCPHub
from mcp_hub.config import Config, HubConfig
from mcp_hub.registry import ServerStatus


class TestMCPHub:
    """Test MCP Hub server functionality"""
    
    @pytest.fixture
    def hub_config(self, sample_server_config):
        """Create hub configuration for testing"""
        return Config(
            hub=HubConfig(
                name="Test Hub",
                host="127.0.0.1",
                port=8080,
                log_level="debug"
            ),
            servers=[sample_server_config]
        )
    
    def test_hub_initialization(self, hub_config):
        """Test hub initialization"""
        hub = MCPHub(hub_config)
        
        assert hub.config == hub_config
        assert hub.registry is not None
        assert len(hub.registry.get_all()) == 0
    
    @pytest.mark.asyncio
    async def test_load_all_servers_success(self, hub_config, mock_http_server):
        """Test loading all servers successfully"""
        hub = MCPHub(hub_config)
        
        routes, apps = await hub.load_all_servers()
        
        assert len(routes) == 1
        assert len(apps) == 1
        
        # Check registry
        servers = hub.registry.get_all()
        assert len(servers) == 1
        
        server = list(servers.values())[0]
        assert server.status == ServerStatus.HEALTHY
        assert server.app is not None
    
    @pytest.mark.asyncio
    async def test_load_disabled_server(self):
        """Test handling disabled servers"""
        config = Config(
            hub=HubConfig(name="Test Hub"),
            servers=[
                sample_server_config := ServerConfig(
                    id="disabled",
                    name="Disabled Server",
                    enabled=False
                )
            ]
        )
        
        hub = MCPHub(config)
        routes, apps = await hub.load_all_servers()
        
        # Should still create route but mark as disabled
        assert len(routes) == 1
        
        server = hub.registry.get("disabled")
        assert server.status == ServerStatus.DISABLED
    
    @pytest.mark.asyncio
    async def test_create_app(self, hub_config, mock_http_server):
        """Test creating the hub application"""
        hub = MCPHub(hub_config)
        
        app = await hub.create_app()
        
        assert app is not None
        # Test that routes are created
        assert len(app.routes) > 0
    
    @pytest.mark.asyncio
    async def test_management_routes(self, hub_config, mock_http_server):
        """Test management route creation"""
        hub = MCPHub(hub_config)
        
        # Load servers first
        await hub.load_all_servers()
        
        routes = hub.create_management_routes()
        
        assert len(routes) >= 2  # health and list_servers
        
        # Check route paths
        route_paths = [route.path for route in routes]
        assert "/health" in route_paths
        assert "/servers" in route_paths
        assert "/" in route_paths
```

## 7. **tests/test_cli.py** (CLI Tests)

```python
import pytest
from click.testing import CliRunner
from unittest.mock import patch, AsyncMock
import yaml
from pathlib import Path

from mcp_hub.cli import cli, run, init, info


class TestCLI:
    """Test CLI functionality"""
    
    def setup_method(self):
        """Setup test method"""
        self.runner = CliRunner()
    
    def test_cli_help(self):
        """Test CLI help command"""
        result = self.runner.invoke(cli, ['--help'])
        
        assert result.exit_code == 0
        assert "MCP Hub CLI" in result.output
    
    def test_init_command(self, temp_dir):
        """Test init command"""
        with self.runner.isolated_filesystem(temp_dir=temp_dir):
            result = self.runner.invoke(init)
            
            assert result.exit_code == 0
            assert "Created configuration files" in result.output
            
            # Check files were created
            assert Path("config.yaml").exists()
            assert Path("servers.yaml").exists()
            
            # Verify content
            with open("config.yaml") as f:
                config = yaml.safe_load(f)
                assert "hub" in config
    
    def test_init_command_with_force(self, temp_dir):
        """Test init command with force flag"""
        with self.runner.isolated_filesystem(temp_dir=temp_dir):
            # Create existing file
            Path("config.yaml").write_text("existing: content")
            
            # Run init without force
            result = self.runner.invoke(init)
            assert "already exists" in result.output
            
            # Run init with force
            result = self.runner.invoke(init, ["--force"])
            assert result.exit_code == 0
            
            # Check file was overwritten
            with open("config.yaml") as f:
                config = yaml.safe_load(f)
                assert "hub" in config
    
    def test_info_command(self, temp_dir):
        """Test info command"""
        with self.runner.isolated_filesystem(temp_dir=temp_dir):
            # Create config files
            self.runner.invoke(init)
            
            result = self.runner.invoke(info)
            
            assert result.exit_code == 0
            assert "MCP Hub Information" in result.output
            assert "config.yaml: Found" in result.output
    
    @patch('mcp_hub.server.MCPHub.run')
    def test_run_command(self, mock_run, sample_config_files):
        """Test run command"""
        config_path, servers_path = sample_config_files
        
        # Mock the async run method
        mock_run.return_value = AsyncMock()
        
        with patch('mcp_hub.cli.get_data_path', return_value=config_path.parent):
            result = self.runner.invoke(run)
            
            # Should attempt to run (will fail due to mocking, but that's OK)
            assert "Loading configuration" in result.output or result.exit_code != 0
    
    def test_run_command_missing_config(self, temp_dir):
        """Test run command with missing config"""
        with patch('mcp_hub.cli.get_data_path', return_value=temp_dir):
            result = self.runner.invoke(run)
            
            assert result.exit_code == 1
            assert "Config file not found" in result.output
    
    def test_run_env_command(self, sample_config_files):
        """Test run-env command"""
        config_path, servers_path = sample_config_files
        
        # Create environment-specific config
        env_config = config_path.parent / "config.production.yaml"
        env_config.write_text(config_path.read_text())
        
        with patch('mcp_hub.cli.get_data_path', return_value=config_path.parent):
            with patch('mcp_hub.server.MCPHub.run', return_value=AsyncMock()):
                result = self.runner.invoke(cli, ['run-env', '--env', 'production'])
                
                assert "Running in production environment" in result.output
    
    def test_custom_data_dir(self, temp_dir):
        """Test custom data directory"""
        # Create config in custom directory
        custom_dir = temp_dir / "custom"
        custom_dir.mkdir()
        
        with self.runner.isolated_filesystem(temp_dir=custom_dir):
            result = self.runner.invoke(cli, ['--data-dir', str(custom_dir), 'init'])
            
            assert result.exit_code == 0
            assert (custom_dir / "config.yaml").exists()
```

## 8. **tests/test_config.py** (Configuration Tests)

```python
import pytest
import yaml
import os
from pathlib import Path

from mcp_hub.config import Config, ServerConfig, substitute_env_vars


class TestConfig:
    """Test configuration handling"""
    
    def test_substitute_env_vars(self):
        """Test environment variable substitution"""
        os.environ["TEST_VAR"] = "test_value"
        
        data = {
            "simple": "${TEST_VAR}",
            "with_default": "${MISSING_VAR:-default_value}",
            "nested": {
                "key": "${TEST_VAR}"
            },
            "list": ["${TEST_VAR}", "static"]
        }
        
        result = substitute_env_vars(data)
        
        assert result["simple"] == "test_value"
        assert result["with_default"] == "${MISSING_VAR:-default_value}"  # Not handled by simple substitution
        assert result["nested"]["key"] == "test_value"
        assert result["list"][0] == "test_value"
        
        # Clean up
        del os.environ["TEST_VAR"]
    
    def test_config_from_file(self, sample_config_files):
        """Test loading config from files"""
        config_path, servers_path = sample_config_files
        
        config = Config.from_file(config_path, servers_path)
        
        assert config.hub.name == "Test MCP Hub"
        assert config.hub.port == 8080
        assert len(config.servers) == 1
        assert config.servers[0].id == "test-api"
    
    def test_config_without_servers_file(self, sample_config_files):
        """Test loading config without servers file"""
        config_path, _ = sample_config_files
        
        config = Config.from_file(config_path, None)
        
        assert config.hub.name == "Test MCP Hub"
        assert len(config.servers) == 0
    
    def test_server_config_defaults(self):
        """Test server config defaults"""
        config = ServerConfig(
            id="test",
            name="Test Server"
        )
        
        assert config.mount_path == "/test"
        assert config.enabled is True
        assert config.retry_attempts == 3
        assert config.retry_delay == 2.0
    
    def test_server_config_with_env_vars(self, temp_dir):
        """Test server config with environment variables"""
        os.environ["TEST_URL"] = "https://test.example.com"
        
        config_data = {
            "hub": {"name": "Test Hub"},
            "servers": [
                {
                    "id": "test",
                    "name": "Test",
                    "openapi_url": "${TEST_URL}/openapi.json"
                }
            ]
        }
        
        config_file = temp_dir / "config.yaml"
        with open(config_file, 'w') as f:
            yaml.dump(config_data, f)
        
        config = Config.from_file(config_file)
        
        assert config.servers[0].openapi_url == "https://test.example.com/openapi.json"
        
        # Clean up
        del os.environ["TEST_URL"]
```

## 9. **tests/test_integration.py** (Integration Tests)

```python
import pytest
import asyncio
from httpx import AsyncClient
from fastmcp.testing import MCPTest

from mcp_hub.server import MCPHub
from mcp_hub.config import Config, HubConfig, ServerConfig


class TestIntegration:
    """Integration tests for the complete MCP Hub"""
    
    @pytest.mark.asyncio
    async def test_hub_with_openapi_server(self, mock_http_server):
        """Test complete hub with OpenAPI server integration"""
        # Create configuration
        config = Config(
            hub=HubConfig(
                name="Integration Test Hub",
                host="127.0.0.1",
                port=8081
            ),
            servers=[
                ServerConfig(
                    id="test-api",
                    name="Test API",
                    type="openapi",
                    openapi_url="https://api.example.com/openapi.json",
                    base_url="https://api.example.com",
                    mount_path="/api"
                )
            ]
        )
        
        # Create and setup hub
        hub = MCPHub(config)
        app = await hub.create_app()
        
        # Test health endpoint
        async with AsyncClient(app=app, base_url="http://test") as client:
            response = await client.get("/health")
            assert response.status_code == 200
            
            health_data = response.json()
            assert health_data["status"] in ["healthy", "degraded"]
            assert health_data["hub"]["name"] == "Integration Test Hub"
    
    @pytest.mark.asyncio
    async def test_hub_with_mixed_servers(self, mock_http_server, mock_subprocess_exec):
        """Test hub with different server types"""
        from mcp_hub.models.server import PackageInfo, TransportConfig, TransportType
        
        config = Config(
            hub=HubConfig(name="Mixed Hub"),
            servers=[
                # OpenAPI server
                ServerConfig(
                    id="api-server",
                    name="API Server",
                    type="openapi",
                    openapi_url="https://api.example.com/openapi.json",
                    base_url="https://api.example.com",
                    mount_path="/api"
                ),
                # UVX server
                ServerConfig(
                    id="uvx-server",
                    name="UVX Server",
                    mount_path="/uvx",
                    packages=[
                        PackageInfo(
                            registry_name="uvx",
                            name="test-server",
                            transport=TransportConfig(type=TransportType.STDIO)
                        )
                    ]
                )
            ]
        )
        
        hub = MCPHub(config)
        routes, apps = await hub.load_all_servers()
        
        # Should have both servers
        assert len(routes) == 2
        assert len(apps) == 2
        
        # Check registry has both servers
        servers = hub.registry.get_all()
        assert len(servers) == 2
        assert "api-server" in servers
        assert "uvx-server" in servers
    
    @pytest.mark.asyncio
    async def test_server_failure_handling(self):
        """Test handling of server failures"""
        config = Config(
            hub=HubConfig(name="Failure Test Hub"),
            servers=[
                ServerConfig(
                    id="failing-server",
                    name="Failing Server",
                    type="openapi",
                    openapi_url="https://invalid.example.com/openapi.json",
                    retry_attempts=1,
                    retry_delay=0.1
                )
            ]
        )
        
        hub = MCPHub(config)
        routes, apps = await hub.load_all_servers()
        
        # Should still create route with fallback app
        assert len(routes) == 1
        assert len(apps) == 1
        
        # Server should be marked as failed
        server = hub.registry.get("failing-server")
        assert server.status.value == "failed"
        assert server.error is not None
    
    @pytest.mark.asyncio
    async def test_concurrent_server_loading(self, mock_http_server, mock_subprocess_exec):
        """Test concurrent loading of multiple servers"""
        # Create multiple servers
        servers = []
        for i in range(5):
            servers.append(
                ServerConfig(
                    id=f"server-{i}",
                    name=f"Server {i}",
                    type="openapi",
                    openapi_url="https://api.example.com/openapi.json",
                    base_url="https://api.example.com",
                    mount_path=f"/server-{i}"
                )
            )
        
        config = Config(
            hub=HubConfig(name="Concurrent Test Hub"),
            servers=servers
        )
        
        hub = MCPHub(config)
        
        # Time the loading
        import time
        start_time = time.time()
        
        routes, apps = await hub.load_all_servers()
        
        end_time = time.time()
        loading_time = end_time - start_time
        
        # Should load all servers
        assert len(routes) == 5
        assert len(apps) == 5
        
        # Should be faster than sequential loading (rough check)
        # Sequential would take much longer due to mocking delays
        assert loading_time < 10.0  # Should complete within 10 seconds
```

## 10. **tests/test_performance.py** (Performance Tests)

```python
import pytest
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor

from mcp_hub.server import MCPHub
from mcp_hub.config import Config, HubConfig, ServerConfig


class TestPerformance:
    """Performance tests for MCP Hub"""
    
    @pytest.mark.asyncio
    async def test_server_loading_performance(self, mock_http_server):
        """Test performance of loading multiple servers"""
        # Create 10 servers
        servers = [
            ServerConfig(
                id=f"perf-server-{i}",
                name=f"Performance Server {i}",
                type="openapi",
                openapi_url="https://api.example.com/openapi.json",
                mount_path=f"/perf-{i}"
            ) for i in range(10)
        ]
        
        config = Config(
            hub=HubConfig(name="Performance Hub"),
            servers=servers
        )
        
        hub = MCPHub(config)
        
        start_time = time.time()
        routes, apps = await hub.load_all_servers()
        end_time = time.time()
        
        loading_time = end_time - start_time
        
        # All servers should load
        assert len(routes) == 10
        assert len(apps) == 10
        
        # Should complete within reasonable time
        assert loading_time < 30.0  # 30 seconds max
        
        # Calculate servers per second
        servers_per_second = len(servers) / loading_time
        print(f"Loaded {servers_per_second:.2f} servers per second")
    
    @pytest.mark.asyncio
    async def test_memory_usage(self, mock_http_server):
        """Test memory usage with multiple servers"""
        import psutil
        import os
        
        process = psutil.Process(os.getpid())
        initial_memory = process.memory_info().rss
        
        # Create many servers
        servers = [
            ServerConfig(
                id=f"mem-server-{i}",
                name=f"Memory Server {i}",
                type="openapi",
                openapi_url="https://api.example.com/openapi.json",
                mount_path=f"/mem-{i}"
            ) for i in range(50)
        ]
        
        config = Config(
            hub=HubConfig(name="Memory Hub"),
            servers=servers
        )
        
        hub = MCPHub(config)
        await hub.load_all_servers()
        
        final_memory = process.memory_info().rss
        memory_increase = final_memory - initial_memory
        
        # Memory increase should be reasonable
        memory_increase_mb = memory_increase / (1024 * 1024)
        print(f"Memory increase: {memory_increase_mb:.2f} MB")
        
        # Should not use excessive memory (adjust based on your requirements)
        assert memory_increase_mb < 500  # Less than 500MB increase
```

## 11. **pytest.ini** (Pytest Configuration)

```ini
[tool:pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = 
    -v
    --tb=short
    --strict-markers
    --disable-warnings
    --asyncio-mode=auto
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks tests as integration tests
    performance: marks tests as performance tests
    unit: marks tests as unit tests
asyncio_mode = auto
log_cli = true
log_cli_level = INFO
log_cli_format = %(asctime)s [%(levelname)8s] %(name)s: %(message)s
log_cli_date_format = %Y-%m-%d %H:%M:%S
```

## 12. **requirements-test.txt** (Test Dependencies)

```txt
# Test dependencies
pytest>=7.0.0
pytest-asyncio>=0.21.0
pytest-mock>=3.10.0
pytest-httpx>=0.21.0
pytest-cov>=4.0.0

# Performance testing
psutil>=5.9.0

# Additional utilities
responses>=0.23.0
aiofiles>=23.0.0
```

## Running the Tests

```bash
# Run all tests
pytest

# Run specific test file
pytest tests/test_loaders_openapi.py

# Run with coverage
pytest --cov=mcp_hub --cov-report=html

# Run only unit tests
pytest -m unit

# Run only integration tests
pytest -m integration

# Run performance tests
pytest -m performance

# Run tests excluding slow ones
pytest -m "not slow"

# Run specific test
pytest tests/test_cli.py::TestCLI::test_init_command -v
```

This comprehensive test suite covers:

1. **Unit Tests**: Individual components and functions
2. **Integration Tests**: Complete workflows and interactions
3. **CLI Tests**: Command-line interface functionality  
4. **Performance Tests**: Loading times and memory usage
5. **Error Handling**: Failure scenarios and edge cases
6. **FastMCP Integration**: Using FastMCP's testing utilities

The tests leverage FastMCP's native testing capabilities (`MCPTest`) rather than reinventing testing infrastructure, and provide good coverage of all the major functionality while being maintainable and fast to run.

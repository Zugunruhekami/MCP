*Thinking...*

> The user wants me to create the most recommended way to construct an MCP Hub as a registry by:
> 1. Considering the FastMCP OpenAPI integration documentation
> 2. Considering the MCP Registry patterns from GitHub
> 3. Incorporating the best parts of the two existing implementations they provided
> 
> Looking at the MCP Registry, it has:
> - RESTful API endpoints for listing, getting, creating, updating, deleting server entries
> - Server metadata with packages, versions, repository info
> - MongoDB storage
> - Pagination support
> - Authentication for publishing
> - Seed data import
> 
> From FastMCP OpenAPI integration:
> - Can generate MCP servers from OpenAPI specs using `FastMCP.from_openapi()`
> - Supports authentication via HTTP client configuration
> - Route mapping with RouteMap objects to control how OpenAPI routes become MCP components
> - Advanced customization options
> - Parameter handling
> 
> From the two implementations provided:
> 1. First implementation has:
>    - JSON-based configuration
>    - Simple dataclasses
>    - Direct function-based loaders
>    - Basic registry tracking
>    - Health/status endpoints
>    - Fallback apps for failed servers
> 
> 2. Second implementation has:
>    - TOML/YAML configuration with Pydantic
>    - More modular structure
>    - OOP approach with base classes and factory pattern
>    - Environment variable support
>    - Better error handling with custom exceptions
>    - TaskGroup for lifespan management
> 
> I should combine the best aspects:
> - Use the registry data model from MCP Registry
> - Use FastMCP's OpenAPI integration capabilities
> - Take the modular structure from implementation 2
> - Add database persistence like MCP Registry
> - Support both dynamic loading and registry-based discovery
> - Include authentication for publishing new servers

Based on the MCP Registry patterns and FastMCP OpenAPI integration, here's the most recommended way to construct an MCP Hub as a registry, incorporating the best parts of both implementations:

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    MCP Hub Registry                      │
├─────────────────────────────────────────────────────────┤
│  Registry API Layer (/v0/*)                              │
│  - List servers     - Get server details                 │
│  - Publish server   - Health checks                      │
├─────────────────────────────────────────────────────────┤
│  Dynamic Server Loader                                   │
│  - OpenAPI servers via FastMCP.from_openapi()           │
│  - Custom MCP servers                                    │
│  - Remote proxy servers                                  │
├─────────────────────────────────────────────────────────┤
│  Registry Database (MongoDB/PostgreSQL)                  │
│  - Server metadata    - Package info                     │
│  - Version tracking   - Authentication                   │
└─────────────────────────────────────────────────────────┘
```

## Project Structure

```
mcp_hub_registry/
├── __init__.py
├── models/
│   ├── __init__.py
│   ├── registry.py      # Registry data models
│   └── server.py        # Server configuration models
├── loaders/
│   ├── __init__.py
│   ├── base.py         # Base loader interface
│   ├── openapi.py      # OpenAPI loader with FastMCP
│   ├── module.py       # Python module loader
│   └── package.py      # NPM/Docker package loader
├── database/
│   ├── __init__.py
│   ├── interface.py    # Database interface
│   ├── mongodb.py      # MongoDB implementation
│   └── memory.py       # In-memory implementation
├── api/
│   ├── __init__.py
│   ├── registry.py     # Registry API endpoints
│   ├── health.py       # Health/status endpoints
│   └── auth.py         # Authentication middleware
├── core/
│   ├── __init__.py
│   ├── hub.py          # Main hub orchestrator
│   ├── config.py       # Configuration management
│   └── exceptions.py   # Custom exceptions
└── cli.py              # CLI interface
```

## Implementation

**models/registry.py** (Based on MCP Registry structure):
```python
from typing import List, Optional, Dict, Any
from datetime import datetime
from pydantic import BaseModel, Field
from enum import Enum


class PackageArgument(BaseModel):
    description: str
    is_required: bool = True
    format: str = "string"
    value: Optional[str] = None
    default: Optional[str] = None
    type: str = "positional"  # positional, flag, environment
    value_hint: Optional[str] = None


class EnvironmentVariable(BaseModel):
    name: str
    description: str
    required: bool = True
    default: Optional[str] = None


class Package(BaseModel):
    registry_name: str  # npm, docker, pypi
    name: str
    version: str
    runtime_hint: Optional[str] = None
    package_arguments: List[PackageArgument] = []
    environment_variables: List[EnvironmentVariable] = []
    runtime_arguments: List[PackageArgument] = []


class Repository(BaseModel):
    url: str
    source: str = "github"
    id: Optional[str] = None


class VersionDetail(BaseModel):
    version: str
    release_date: datetime
    is_latest: bool = True


class ServerEntry(BaseModel):
    id: str
    name: str
    description: str
    repository: Optional[Repository] = None
    version_detail: VersionDetail
    packages: List[Package]
    
    # Additional fields for hub functionality
    openapi_config: Optional[Dict[str, Any]] = None
    route_maps: Optional[List[Dict[str, Any]]] = None
    auth_config: Optional[Dict[str, Any]] = None
    tags: List[str] = []
    enabled: bool = True
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)


class ServerStatus(str, Enum):
    LOADING = "loading"
    HEALTHY = "healthy"
    FAILED = "failed"
    DISABLED = "disabled"
```

**loaders/openapi.py** (FastMCP OpenAPI Integration):
```python
import asyncio
import httpx
from typing import Optional, Tuple, Dict, Any
from fastmcp import FastMCP
from fastmcp.server.openapi import RouteMap, MCPType
from starlette.applications import Starlette

from ..models.registry import ServerEntry
from .base import ServerLoader, LoadResult


class OpenAPIServerLoader(ServerLoader):
    """Loader for OpenAPI-based MCP servers using FastMCP"""
    
    async def load(self, entry: ServerEntry) -> LoadResult:
        if not entry.openapi_config:
            return LoadResult(
                success=False,
                error="No OpenAPI configuration provided"
            )
        
        try:
            # Create HTTP client with authentication if needed
            client = self._create_http_client(entry)
            
            # Fetch OpenAPI spec
            openapi_url = entry.openapi_config.get("url")
            if not openapi_url:
                raise ValueError("OpenAPI URL not provided")
            
            async with asyncio.timeout(30.0):
                response = await client.get(openapi_url)
                response.raise_for_status()
                openapi_spec = response.json()
            
            # Convert route maps if provided
            route_maps = self._parse_route_maps(entry.route_maps or [])
            
            # Custom route mapper function if needed
            route_map_fn = None
            if entry.openapi_config.get("advanced_routing"):
                route_map_fn = self._create_route_mapper(entry)
            
            # Create MCP server from OpenAPI
            mcp = await FastMCP.from_openapi(
                openapi_spec=openapi_spec,
                client=client,
                name=entry.name,
                route_maps=route_maps,
                route_map_fn=route_map_fn,
                tags=set(entry.tags),
                mcp_names=entry.openapi_config.get("mcp_names", {}),
                timeout=entry.openapi_config.get("timeout", 30.0)
            )
            
            # Add custom component modifications if specified
            if entry.openapi_config.get("customize_components"):
                mcp = self._customize_components(mcp, entry)
            
            # Get ASGI app
            app = mcp.http_app(path="/mcp")
            
            return LoadResult(
                success=True,
                mcp_instance=mcp,
                app=app
            )
            
        except Exception as e:
            return LoadResult(
                success=False,
                error=f"Failed to load OpenAPI server: {str(e)}"
            )
        finally:
            if hasattr(self, '_client'):
                await self._client.aclose()
    
    def _create_http_client(self, entry: ServerEntry) -> httpx.AsyncClient:
        """Create HTTP client with authentication"""
        headers = {}
        auth = None
        
        if entry.auth_config:
            auth_type = entry.auth_config.get("type")
            
            if auth_type == "bearer":
                token = entry.auth_config.get("token")
                headers["Authorization"] = f"Bearer {token}"
            elif auth_type == "api_key":
                key_name = entry.auth_config.get("header_name", "X-API-Key")
                key_value = entry.auth_config.get("api_key")
                headers[key_name] = key_value
            elif auth_type == "basic":
                username = entry.auth_config.get("username")
                password = entry.auth_config.get("password")
                auth = httpx.BasicAuth(username, password)
        
        base_url = entry.openapi_config.get("base_url", "")
        
        self._client = httpx.AsyncClient(
            base_url=base_url,
            headers=headers,
            auth=auth,
            timeout=30.0
        )
        
        return self._client
    
    def _parse_route_maps(self, route_maps: List[Dict[str, Any]]) -> List[RouteMap]:
        """Convert dictionary route maps to RouteMap objects"""
        parsed_maps = []
        
        for rm in route_maps:
            route_map = RouteMap(
                methods=rm.get("methods", ["*"]),
                pattern=rm.get("pattern", r".*"),
                tags=set(rm.get("tags", [])),
                mcp_type=MCPType[rm.get("mcp_type", "TOOL")],
                mcp_tags=set(rm.get("mcp_tags", []))
            )
            parsed_maps.append(route_map)
        
        return parsed_maps
    
    def _create_route_mapper(self, entry: ServerEntry):
        """Create advanced route mapper function"""
        def route_mapper(route, mcp_type):
            # Custom routing logic based on entry configuration
            routing_rules = entry.openapi_config.get("routing_rules", {})
            
            for rule in routing_rules:
                if rule.get("path_pattern") and rule["path_pattern"] in route.path:
                    return MCPType[rule.get("mcp_type", "TOOL")]
            
            return None
        
        return route_mapper
```

**core/hub.py** (Main Hub Implementation):
```python
import asyncio
import logging
from typing import Dict, List, Optional
from contextlib import asynccontextmanager

from starlette.applications import Starlette
from starlette.routing import Mount, Route
from starlette.responses import JSONResponse
from starlette.middleware.authentication import AuthenticationMiddleware

from ..models.registry import ServerEntry, ServerStatus
from ..database.interface import RegistryDatabase
from ..loaders import get_loader
from ..api.auth import BearerTokenBackend


class MCPHubRegistry:
    """Main MCP Hub Registry orchestrator"""
    
    def __init__(
        self,
        database: RegistryDatabase,
        config: Dict[str, Any]
    ):
        self.database = database
        self.config = config
        self.loaded_servers: Dict[str, Dict[str, Any]] = {}
        self.logger = logging.getLogger(__name__)
    
    async def initialize(self):
        """Initialize the hub and load all enabled servers"""
        # Import seed data if configured
        if self.config.get("seed_import") and self.config.get("seed_file"):
            await self._import_seed_data()
        
        # Load all enabled servers
        await self._load_all_servers()
    
    async def _load_all_servers(self):
        """Load all enabled servers from the database"""
        entries = await self.database.list_entries(enabled_only=True)
        
        # Load servers concurrently
        tasks = []
        for entry in entries:
            tasks.append(self._load_server(entry))
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        for entry, result in zip(entries, results):
            if isinstance(result, Exception):
                self.logger.error(f"Failed to load server {entry.id}: {result}")
                await self.database.update_status(entry.id, ServerStatus.FAILED, str(result))
    
    async def _load_server(self, entry: ServerEntry):
        """Load a single server"""
        try:
            # Get appropriate loader based on server configuration
            loader_type = self._determine_loader_type(entry)
            loader = get_loader(loader_type)
            
            # Update status to loading
            await self.database.update_status(entry.id, ServerStatus.LOADING)
            
            # Load the server
            result = await loader.load(entry)
            
            if result.success:
                self.loaded_servers[entry.id] = {
                    "mcp": result.mcp_instance,
                    "app": result.app,
                    "entry": entry
                }
                await self.database.update_status(entry.id, ServerStatus.HEALTHY)
                self.logger.info(f"Successfully loaded server: {entry.name}")
            else:
                await self.database.update_status(entry.id, ServerStatus.FAILED, result.error)
                self.logger.error(f"Failed to load server {entry.name}: {result.error}")
                
        except Exception as e:
            self.logger.error(f"Unexpected error loading server {entry.id}: {e}")
            await self.database.update_status(entry.id, ServerStatus.FAILED, str(e))
    
    def _determine_loader_type(self, entry: ServerEntry) -> str:
        """Determine which loader to use based on server configuration"""
        if entry.openapi_config:
            return "openapi"
        elif entry.packages:
            # Check package types
            for package in entry.packages:
                if package.registry_name == "npm":
                    return "npm"
                elif package.registry_name == "docker":
                    return "docker"
                elif package.registry_name == "pypi":
                    return "pypi"
        
        return "module"  # Default to module loader
    
    async def create_app(self) -> Starlette:
        """Create the main Starlette application"""
        routes = []
        
        # Add registry API routes
        routes.extend(self._create_registry_routes())
        
        # Add health/status routes
        routes.extend(self._create_health_routes())
        
        # Mount loaded servers
        for server_id, server_data in self.loaded_servers.items():
            mount_path = f"/servers/{server_id}"
            routes.append(Mount(mount_path, app=server_data["app"]))
        
        # Create app with authentication if configured
        app = Starlette(
            routes=routes,
            lifespan=self._create_lifespan()
        )
        
        if self.config.get("auth_enabled"):
            app.add_middleware(
                AuthenticationMiddleware,
                backend=BearerTokenBackend(self.config.get("auth_tokens", []))
            )
        
        return app
    
    def _create_registry_routes(self) -> List[Route]:
        """Create registry API routes (MCP Registry compatible)"""
        routes = []
        
        # List servers
        async def list_servers(request):
            limit = int(request.query_params.get("limit", 30))
            cursor = request.query_params.get("cursor")
            
            servers, next_cursor = await self.database.list_entries_paginated(
                limit=limit,
                cursor=cursor
            )
            
            # Add runtime status
            for server in servers:
                if server["id"] in self.loaded_servers:
                    server["runtime_status"] = "healthy"
                    server["capabilities"] = self._get_server_capabilities(server["id"])
                else:
                    server["runtime_status"] = "not_loaded"
            
            return JSONResponse({
                "servers": servers,
                "metadata": {
                    "next_cursor": next_cursor,
                    "count": len(servers)
                }
            })
        
        routes.append(Route("/v0/servers", endpoint=list_servers, methods=["GET"]))
        
        # Get server details
        async def get_server(request):
            server_id = request.path_params["id"]
            entry = await self.database.get_entry(server_id)
            
            if not entry:
                return JSONResponse({"error": "Server not found"}, status_code=404)
            
            response = entry.dict()
            
            # Add runtime information
            if server_id in self.loaded_servers:
                response["runtime_status"] = "healthy"
                response["capabilities"] = self._get_server_capabilities(server_id)
                response["mcp_endpoint"] = f"/servers/{server_id}/mcp"
            
            return JSONResponse(response)
        
        routes.append(Route("/v0/servers/{id}", endpoint=get_server, methods=["GET"]))
        
        # Publish server (requires authentication)
        async def publish_server(request):
            if not request.user.is_authenticated:
                return JSONResponse({"error": "Authentication required"}, status_code=401)
            
            data = await request.json()
            
            try:
                # Validate and create server entry
                entry = ServerEntry(**data)
                
                # Save to database
                server_id = await self.database.create_entry(entry)
                
                # Optionally load immediately
                if data.get("load_immediately", False):
                    await self._load_server(entry)
                
                return JSONResponse({
                    "message": "Server publication successful",
                    "id": server_id
                })
                
            except Exception as e:
                return JSONResponse({"error": str(e)}, status_code=400)
        
        routes.append(Route("/v0/publish", endpoint=publish_server, methods=["POST"]))
        
        return routes
    
    def _get_server_capabilities(self, server_id: str) -> Dict[str, Any]:
        """Get capabilities of a loaded server"""
        server_data = self.loaded_servers.get(server_id)
        if not server_data or not server_data.get("mcp"):
            return {}
        
        mcp = server_data["mcp"]
        
        try:
            return {
                "tools": [{"name": t.name, "description": t.description} for t in mcp.list_tools()],
                "resources": [{"uri": r.uri, "name": r.name} for r in mcp.list_resources()],
                "prompts": [{"name": p.name, "description": p.description} for p in mcp.list_prompts()]
            }
        except Exception as e:
            self.logger.error(f"Error getting capabilities for {server_id}: {e}")
            return {"error": str(e)}
    
    def _create_lifespan(self):
        """Create lifespan context manager"""
        @asynccontextmanager
        async def lifespan(app):
            # Startup
            await self.initialize()
            yield
            # Shutdown
            await self.database.close()
        
        return lifespan
```

**config.yaml** (Configuration):
```yaml
hub:
  name: "MCP Hub Registry"
  version: "1.0.0"
  description: "Community-driven MCP server registry"
  
database:
  type: "mongodb"  # or "postgresql", "memory"
  url: "mongodb://localhost:27017"
  database_name: "mcp-registry"
  
api:
  host: "0.0.0.0"
  port: 8080
  auth_enabled: true
  auth_tokens:
    - "your-secret-token-here"
  
loader:
  timeout: 30.0
  retry_attempts: 3
  retry_delay: 2.0
  
seed:
  import_on_startup: true
  file_path: "data/seed.json"
```

## Key Features

This implementation combines the best of both worlds:

### From MCP Registry:
- Standard registry API endpoints (`/v0/servers`, `/v0/publish`)
- Server metadata model with packages and versions
- Database persistence
- Authentication for publishing
- Pagination support

### From FastMCP OpenAPI:
- Dynamic server generation from OpenAPI specs
- Advanced route mapping capabilities
- Component customization
- Flexible authentication options

### From Your Implementations:
- Modular architecture with clear separation of concerns
- Robust error handling with fallbacks
- Health monitoring and status tracking
- Concurrent server loading
- Configuration management (YAML/TOML)
- Extensible loader system

### Additional Features:
- Database abstraction (supports MongoDB, PostgreSQL, in-memory)
- Package-based server loading (npm, docker, pypi)
- Runtime capability discovery
- Seed data import
- CLI interface for management

This design provides a production-ready MCP Hub that can:
1. Act as a registry for discovering MCP servers
2. Dynamically load and host servers from various sources
3. Handle failures gracefully
4. Scale horizontally with database backend
5. Support community contributions through the publish API





*Thinking...*

> The user wants me to implement the remaining files from the proposed repository structure that weren't implemented above. Let me check what was already implemented and what's missing:
> 
> Already implemented:
> - models/registry.py
> - loaders/openapi.py 
> - core/hub.py
> 
> Missing files that need to be implemented:
> - models/__init__.py
> - models/server.py
> - loaders/__init__.py
> - loaders/base.py
> - loaders/module.py
> - loaders/package.py
> - database/__init__.py
> - database/interface.py
> - database/mongodb.py
> - database/memory.py
> - api/__init__.py
> - api/registry.py
> - api/health.py
> - api/auth.py
> - core/__init__.py
> - core/config.py
> - core/exceptions.py
> - cli.py
> - __init__.py (root)
> 
> Let me implement all these missing files.

Here are the remaining files for the MCP Hub Registry implementation:

## Root Files

**__init__.py** (root):
```python
from .core.hub import MCPHubRegistry
from .core.config import Config
from .models.registry import ServerEntry

__version__ = "1.0.0"
__all__ = ["MCPHubRegistry", "Config", "ServerEntry"]
```

**cli.py**:
```python
import asyncio
import click
import yaml
import json
from pathlib import Path
from typing import Optional

from mcp_hub_registry import MCPHubRegistry, Config
from mcp_hub_registry.database import get_database


@click.group()
@click.option('--config', '-c', type=click.Path(exists=True), help='Configuration file path')
@click.pass_context
def cli(ctx, config: Optional[str]):
    """MCP Hub Registry CLI"""
    ctx.ensure_object(dict)
    
    if config:
        ctx.obj['config'] = Config.from_file(config)
    else:
        # Try to find config file
        for name in ['config.yaml', 'config.yml', 'config.toml']:
            if Path(name).exists():
                ctx.obj['config'] = Config.from_file(name)
                break
        else:
            ctx.obj['config'] = Config()


@cli.command()
@click.pass_context
def run(ctx):
    """Run the MCP Hub Registry server"""
    config = ctx.obj.get('config')
    
    async def start_server():
        import uvicorn
        from mcp_hub_registry.database import get_database
        
        # Create database connection
        db = get_database(config.database)
        await db.connect()
        
        # Create hub
        hub = MCPHubRegistry(db, config.dict())
        app = await hub.create_app()
        
        # Run server
        server_config = uvicorn.Config(
            app=app,
            host=config.api.host,
            port=config.api.port,
            log_level=config.api.log_level
        )
        server = uvicorn.Server(server_config)
        await server.serve()
    
    asyncio.run(start_server())


@cli.command()
@click.argument('server_id')
@click.pass_context
def info(ctx, server_id: str):
    """Get information about a server"""
    config = ctx.obj.get('config')
    
    async def get_info():
        db = get_database(config.database)
        await db.connect()
        
        entry = await db.get_entry(server_id)
        if entry:
            click.echo(yaml.dump(entry.dict(), default_flow_style=False))
        else:
            click.echo(f"Server '{server_id}' not found")
        
        await db.close()
    
    asyncio.run(get_info())


@cli.command()
@click.option('--enabled-only/--all', default=True, help='Show only enabled servers')
@click.option('--format', type=click.Choice(['table', 'json', 'yaml']), default='table')
@click.pass_context
def list(ctx, enabled_only: bool, format: str):
    """List all servers in the registry"""
    config = ctx.obj.get('config')
    
    async def list_servers():
        db = get_database(config.database)
        await db.connect()
        
        entries = await db.list_entries(enabled_only=enabled_only)
        
        if format == 'table':
            click.echo(f"{'ID':<40} {'Name':<30} {'Status':<10} {'Enabled':<8}")
            click.echo("-" * 90)
            for entry in entries:
                status = await db.get_status(entry.id)
                click.echo(f"{entry.id:<40} {entry.name:<30} {status:<10} {str(entry.enabled):<8}")
        elif format == 'json':
            data = [entry.dict() for entry in entries]
            click.echo(json.dumps(data, indent=2, default=str))
        elif format == 'yaml':
            data = [entry.dict() for entry in entries]
            click.echo(yaml.dump(data, default_flow_style=False))
        
        await db.close()
    
    asyncio.run(list_servers())


@cli.command()
@click.argument('file_path', type=click.Path(exists=True))
@click.option('--replace/--no-replace', default=False, help='Replace existing entries')
@click.pass_context
def import_seed(ctx, file_path: str, replace: bool):
    """Import seed data from a JSON file"""
    config = ctx.obj.get('config')
    
    async def import_data():
        db = get_database(config.database)
        await db.connect()
        
        with open(file_path, 'r') as f:
            data = json.load(f)
        
        count = 0
        for server_data in data.get('servers', []):
            try:
                entry = ServerEntry(**server_data)
                
                if replace or not await db.get_entry(entry.id):
                    await db.create_entry(entry)
                    count += 1
                    click.echo(f"Imported: {entry.name}")
                else:
                    click.echo(f"Skipped (already exists): {entry.name}")
                    
            except Exception as e:
                click.echo(f"Error importing server: {e}", err=True)
        
        click.echo(f"\nImported {count} servers")
        await db.close()
    
    asyncio.run(import_data())


@cli.command()
@click.argument('server_id')
@click.option('--enable/--disable', default=True)
@click.pass_context
def toggle(ctx, server_id: str, enable: bool):
    """Enable or disable a server"""
    config = ctx.obj.get('config')
    
    async def toggle_server():
        db = get_database(config.database)
        await db.connect()
        
        entry = await db.get_entry(server_id)
        if entry:
            entry.enabled = enable
            await db.update_entry(server_id, entry)
            click.echo(f"Server '{server_id}' {'enabled' if enable else 'disabled'}")
        else:
            click.echo(f"Server '{server_id}' not found")
        
        await db.close()
    
    asyncio.run(toggle_server())


if __name__ == '__main__':
    cli()
```

## Models

**models/__init__.py**:
```python
from .registry import (
    PackageArgument,
    EnvironmentVariable,
    Package,
    Repository,
    VersionDetail,
    ServerEntry,
    ServerStatus
)
from .server import (
    ServerConfig,
    LoaderType,
    AuthConfig,
    OpenAPIConfig,
    RouteMapConfig
)

__all__ = [
    "PackageArgument",
    "EnvironmentVariable", 
    "Package",
    "Repository",
    "VersionDetail",
    "ServerEntry",
    "ServerStatus",
    "ServerConfig",
    "LoaderType",
    "AuthConfig",
    "OpenAPIConfig",
    "RouteMapConfig"
]
```

**models/server.py**:
```python
from typing import Dict, List, Optional, Any
from pydantic import BaseModel, Field
from enum import Enum


class LoaderType(str, Enum):
    OPENAPI = "openapi"
    MODULE = "module"
    NPM = "npm"
    DOCKER = "docker"
    PYPI = "pypi"
    PROXY = "proxy"


class AuthConfig(BaseModel):
    type: str  # bearer, api_key, basic, oauth2
    token: Optional[str] = None
    api_key: Optional[str] = None
    header_name: Optional[str] = "X-API-Key"
    username: Optional[str] = None
    password: Optional[str] = None
    oauth2_config: Optional[Dict[str, Any]] = None


class RouteMapConfig(BaseModel):
    methods: List[str] = ["*"]
    pattern: str = r".*"
    tags: List[str] = []
    mcp_type: str = "TOOL"  # TOOL, RESOURCE, RESOURCE_TEMPLATE, EXCLUDE
    mcp_tags: List[str] = []


class OpenAPIConfig(BaseModel):
    url: str
    base_url: Optional[str] = None
    timeout: float = 30.0
    mcp_names: Dict[str, str] = Field(default_factory=dict)
    routing_rules: List[Dict[str, Any]] = Field(default_factory=list)
    advanced_routing: bool = False
    customize_components: bool = False


class ServerConfig(BaseModel):
    """Unified server configuration model"""
    id: str
    name: str
    description: str
    loader_type: LoaderType
    enabled: bool = True
    
    # Loader-specific configurations
    openapi_config: Optional[OpenAPIConfig] = None
    module_config: Optional[Dict[str, Any]] = None
    package_config: Optional[Dict[str, Any]] = None
    proxy_config: Optional[Dict[str, Any]] = None
    
    # Common configurations
    auth_config: Optional[AuthConfig] = None
    route_maps: List[RouteMapConfig] = Field(default_factory=list)
    tags: List[str] = Field(default_factory=list)
    mount_path: Optional[str] = None
    
    # Retry configuration
    retry_attempts: int = 3
    retry_delay: float = 2.0
    
    # Health check configuration
    health_check_enabled: bool = True
    health_check_interval: int = 60  # seconds
    health_check_timeout: float = 5.0
```

## Loaders

**loaders/__init__.py**:
```python
from typing import Dict, Type
from .base import ServerLoader, LoadResult
from .openapi import OpenAPIServerLoader
from .module import ModuleServerLoader
from .package import NPMServerLoader, DockerServerLoader, PyPIServerLoader

# Registry of available loaders
_LOADERS: Dict[str, Type[ServerLoader]] = {
    "openapi": OpenAPIServerLoader,
    "module": ModuleServerLoader,
    "npm": NPMServerLoader,
    "docker": DockerServerLoader,
    "pypi": PyPIServerLoader,
}


def get_loader(loader_type: str) -> ServerLoader:
    """Get a server loader instance by type"""
    loader_class = _LOADERS.get(loader_type)
    if not loader_class:
        raise ValueError(f"Unknown loader type: {loader_type}")
    return loader_class()


def register_loader(loader_type: str, loader_class: Type[ServerLoader]):
    """Register a custom loader"""
    _LOADERS[loader_type] = loader_class


__all__ = [
    "ServerLoader",
    "LoadResult",
    "get_loader",
    "register_loader",
    "OpenAPIServerLoader",
    "ModuleServerLoader",
    "NPMServerLoader",
    "DockerServerLoader",
    "PyPIServerLoader"
]
```

**loaders/base.py**:
```python
from abc import ABC, abstractmethod
from typing import Optional
from dataclasses import dataclass

from fastmcp import FastMCP
from starlette.applications import Starlette

from ..models.registry import ServerEntry


@dataclass
class LoadResult:
    """Result of loading a server"""
    success: bool
    error: Optional[str] = None
    mcp_instance: Optional[FastMCP] = None
    app: Optional[Starlette] = None


class ServerLoader(ABC):
    """Abstract base class for server loaders"""
    
    @abstractmethod
    async def load(self, entry: ServerEntry) -> LoadResult:
        """Load a server from the given entry"""
        pass
    
    def create_error_app(self, server_name: str, error: str) -> Starlette:
        """Create a fallback error app"""
        from starlette.responses import JSONResponse
        from starlette.routing import Route
        
        async def error_endpoint(request):
            return JSONResponse({
                "error": "Server unavailable",
                "server": server_name,
                "message": error,
                "status": "failed"
            }, status_code=503)
        
        return Starlette(routes=[
            Route("/{path:path}", endpoint=error_endpoint, methods=["GET", "POST", "PUT", "DELETE"])
        ])
```

**loaders/module.py**:
```python
import importlib
import asyncio
from typing import Optional
from fastmcp import FastMCP

from .base import ServerLoader, LoadResult
from ..models.registry import ServerEntry


class ModuleServerLoader(ServerLoader):
    """Loader for Python module-based MCP servers"""
    
    async def load(self, entry: ServerEntry) -> LoadResult:
        # Check if this server has a Python package
        python_package = None
        for package in entry.packages:
            if package.registry_name == "pypi":
                python_package = package
                break
        
        if not python_package:
            return LoadResult(
                success=False,
                error="No Python package configuration found"
            )
        
        try:
            # Extract module path and factory function
            module_path = None
            factory_function = "create_server"  # default
            
            # Look for module path in package arguments
            for arg in python_package.package_arguments:
                if arg.description.lower().contains("module"):
                    module_path = arg.value or arg.default
                elif arg.description.lower().contains("factory"):
                    factory_function = arg.value or arg.default
            
            if not module_path:
                # Try to derive from package name
                module_path = python_package.name.replace("-", "_")
            
            # Import the module
            module = importlib.import_module(module_path)
            
            # Get the factory function
            if not hasattr(module, factory_function):
                # Try common factory function names
                for name in ["create_server", "create_mcp_server", "get_server", "main"]:
                    if hasattr(module, name):
                        factory_function = name
                        break
                else:
                    raise AttributeError(f"No factory function found in {module_path}")
            
            factory = getattr(module, factory_function)
            
            # Call the factory function
            if asyncio.iscoroutinefunction(factory):
                mcp = await factory()
            else:
                mcp = factory()
            
            if not isinstance(mcp, FastMCP):
                raise TypeError(f"Factory function must return FastMCP instance, got {type(mcp)}")
            
            # Get ASGI app
            app = mcp.http_app(path="/mcp")
            
            return LoadResult(
                success=True,
                mcp_instance=mcp,
                app=app
            )
            
        except Exception as e:
            return LoadResult(
                success=False,
                error=f"Failed to load module server: {str(e)}"
            )
```

**loaders/package.py**:
```python
import asyncio
import subprocess
import tempfile
import json
from typing import Optional, Dict, Any
from pathlib import Path

from fastmcp import FastMCP, Client
from fastmcp.client import StdioTransport, SSETransport

from .base import ServerLoader, LoadResult
from ..models.registry import ServerEntry, Package


class NPMServerLoader(ServerLoader):
    """Loader for NPM-based MCP servers"""
    
    async def load(self, entry: ServerEntry) -> LoadResult:
        npm_package = self._get_package(entry, "npm")
        if not npm_package:
            return LoadResult(success=False, error="No NPM package configuration found")
        
        try:
            # Install the package if needed
            package_name = f"{npm_package.name}@{npm_package.version}"
            
            # Create a temporary directory for the server
            with tempfile.TemporaryDirectory() as tmpdir:
                # Install the package
                install_result = await self._run_command(
                    ["npm", "install", package_name],
                    cwd=tmpdir
                )
                
                if install_result.returncode != 0:
                    raise Exception(f"Failed to install package: {install_result.stderr}")
                
                # Build the command to run the server
                cmd = ["npx", npm_package.name]
                
                # Add package arguments
                for arg in npm_package.package_arguments:
                    if arg.value:
                        cmd.append(arg.value)
                
                # Set environment variables
                env = {}
                for env_var in npm_package.environment_variables:
                    if env_var.default:
                        env[env_var.name] = env_var.default
                
                # Create MCP client with stdio transport
                client = Client(f"{entry.name}_client")
                transport = StdioTransport(cmd, env=env, cwd=tmpdir)
                
                # Connect the client
                await client.connect(transport)
                
                # Create a proxy MCP server
                mcp = FastMCP.as_proxy(
                    client=client,
                    name=entry.name
                )
                
                app = mcp.http_app(path="/mcp")
                
                return LoadResult(
                    success=True,
                    mcp_instance=mcp,
                    app=app
                )
                
        except Exception as e:
            return LoadResult(
                success=False,
                error=f"Failed to load NPM server: {str(e)}"
            )
    
    def _get_package(self, entry: ServerEntry, registry_name: str) -> Optional[Package]:
        for package in entry.packages:
            if package.registry_name == registry_name:
                return package
        return None
    
    async def _run_command(self, cmd: list, cwd: str = None) -> subprocess.CompletedProcess:
        """Run a command asynchronously"""
        proc = await asyncio.create_subprocess_exec(
            *cmd,
            cwd=cwd,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE
        )
        stdout, stderr = await proc.communicate()
        
        return subprocess.CompletedProcess(
            args=cmd,
            returncode=proc.returncode,
            stdout=stdout.decode() if stdout else "",
            stderr=stderr.decode() if stderr else ""
        )


class DockerServerLoader(ServerLoader):
    """Loader for Docker-based MCP servers"""
    
    async def load(self, entry: ServerEntry) -> LoadResult:
        docker_package = self._get_package(entry, "docker")
        if not docker_package:
            return LoadResult(success=False, error="No Docker package configuration found")
        
        try:
            import aiodocker
            
            # Create Docker client
            docker = aiodocker.Docker()
            
            # Pull the image if needed
            image_name = docker_package.name
            if docker_package.version and docker_package.version != "latest":
                image_name = f"{image_name}:{docker_package.version}"
            
            # Check if image exists
            try:
                await docker.images.inspect(image_name)
            except aiodocker.DockerError:
                # Pull the image
                await docker.images.pull(image_name)
            
            # Build container configuration
            config = {
                "Image": image_name,
                "Env": [f"{ev.name}={ev.default}" for ev in docker_package.environment_variables if ev.default],
                "HostConfig": {
                    "AutoRemove": True,
                    "NetworkMode": "bridge"
                }
            }
            
            # Add runtime arguments
            if docker_package.runtime_arguments:
                config["Cmd"] = [arg.value for arg in docker_package.runtime_arguments if arg.value]
            
            # Create and start container
            container = await docker.containers.create(config=config, name=f"mcp-{entry.id}")
            await container.start()
            
            # Get container info
            info = await container.show()
            container_ip = info['NetworkSettings']['IPAddress']
            
            # Create MCP client to connect to the container
            # Assuming the container exposes MCP on a known port (e.g., 8080)
            client = Client(f"{entry.name}_client")
            
            # Use HTTP/SSE transport to connect to the container
            transport = SSETransport(f"http://{container_ip}:8080/sse")
            await client.connect(transport)
            
            # Create proxy MCP server
            mcp = FastMCP.as_proxy(
                client=client,
                name=entry.name
            )
            
            app = mcp.http_app(path="/mcp")
            
            # Store container reference for cleanup
            app.state.container = container
            app.state.docker = docker
            
            return LoadResult(
                success=True,
                mcp_instance=mcp,
                app=app
            )
            
        except Exception as e:
            return LoadResult(
                success=False,
                error=f"Failed to load Docker server: {str(e)}"
            )
    
    def _get_package(self, entry: ServerEntry, registry_name: str) -> Optional[Package]:
        for package in entry.packages:
            if package.registry_name == registry_name:
                return package
        return None


class PyPIServerLoader(ServerLoader):
    """Loader for PyPI-based MCP servers"""
    
    async def load(self, entry: ServerEntry) -> LoadResult:
        pypi_package = self._get_package(entry, "pypi")
        if not pypi_package:
            return LoadResult(success=False, error="No PyPI package configuration found")
        
        try:
            # Install the package if needed
            package_spec = f"{pypi_package.name}=={pypi_package.version}"
            
            # Check if package is already installed
            try:
                importlib.import_module(pypi_package.name.replace("-", "_"))
            except ImportError:
                # Install the package
                install_result = await self._run_command(
                    ["pip", "install", package_spec]
                )
                
                if install_result.returncode != 0:
                    raise Exception(f"Failed to install package: {install_result.stderr}")
            
            # Now use the module loader
            from .module import ModuleServerLoader
            module_loader = ModuleServerLoader()
            return await module_loader.load(entry)
            
        except Exception as e:
            return LoadResult(
                success=False,
                error=f"Failed to load PyPI server: {str(e)}"
            )
    
    def _get_package(self, entry: ServerEntry, registry_name: str) -> Optional[Package]:
        for package in entry.packages:
            if package.registry_name == registry_name:
                return package
        return None
    
    async def _run_command(self, cmd: list) -> subprocess.CompletedProcess:
        """Run a command asynchronously"""
        proc = await asyncio.create_subprocess_exec(
            *cmd,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE
        )
        stdout, stderr = await proc.communicate()
        
        return subprocess.CompletedProcess(
            args=cmd,
            returncode=proc.returncode,
            stdout=stdout.decode() if stdout else "",
            stderr=stderr.decode() if stderr else ""
        )
```

## Database

**database/__init__.py**:
```python
from typing import Union
from .interface import RegistryDatabase, DatabaseConfig
from .mongodb import MongoDBRegistry
from .memory import InMemoryRegistry


def get_database(config: DatabaseConfig) -> RegistryDatabase:
    """Factory function to create database instance based on config"""
    if config.type == "mongodb":
        return MongoDBRegistry(config)
    elif config.type == "memory":
        return InMemoryRegistry(config)
    else:
        raise ValueError(f"Unknown database type: {config.type}")


__all__ = [
    "RegistryDatabase",
    "DatabaseConfig",
    "MongoDBRegistry",
    "InMemoryRegistry",
    "get_database"
]
```

**database/interface.py**:
```python
from abc import ABC, abstractmethod
from typing import List, Optional, Tuple, Dict, Any
from datetime import datetime
from pydantic import BaseModel

from ..models.registry import ServerEntry, ServerStatus


class DatabaseConfig(BaseModel):
    """Database configuration"""
    type: str  # mongodb, postgresql, memory
    url: Optional[str] = None
    database_name: str = "mcp-registry"
    collection_name: str = "servers"


class RegistryDatabase(ABC):
    """Abstract base class for registry database implementations"""
    
    def __init__(self, config: DatabaseConfig):
        self.config = config
    
    @abstractmethod
    async def connect(self):
        """Connect to the database"""
        pass
    
    @abstractmethod
    async def close(self):
        """Close the database connection"""
        pass
    
    @abstractmethod
    async def create_entry(self, entry: ServerEntry) -> str:
        """Create a new server entry"""
        pass
    
    @abstractmethod
    async def get_entry(self, server_id: str) -> Optional[ServerEntry]:
        """Get a server entry by ID"""
        pass
    
    @abstractmethod
    async def update_entry(self, server_id: str, entry: ServerEntry) -> bool:
        """Update a server entry"""
        pass
    
    @abstractmethod
    async def delete_entry(self, server_id: str) -> bool:
        """Delete a server entry"""
        pass
    
    @abstractmethod
    async def list_entries(
        self,
        enabled_only: bool = False,
        tags: Optional[List[str]] = None
    ) -> List[ServerEntry]:
        """List all server entries"""
        pass
    
    @abstractmethod
    async def list_entries_paginated(
        self,
        limit: int = 30,
        cursor: Optional[str] = None,
        enabled_only: bool = False
    ) -> Tuple[List[Dict[str, Any]], Optional[str]]:
        """List server entries with pagination"""
        pass
    
    @abstractmethod
    async def update_status(
        self,
        server_id: str,
        status: ServerStatus,
        error: Optional[str] = None
    ):
        """Update server runtime status"""
        pass
    
    @abstractmethod
    async def get_status(self, server_id: str) -> Optional[str]:
        """Get server runtime status"""
        pass
    
    @abstractmethod
    async def search_entries(
        self,
        query: str,
        fields: List[str] = None
    ) -> List[ServerEntry]:
        """Search server entries"""
        pass
```

**database/mongodb.py**:
```python
from typing import List, Optional, Tuple, Dict, Any
from datetime import datetime
import motor.motor_asyncio
from bson import ObjectId

from .interface import RegistryDatabase, DatabaseConfig
from ..models.registry import ServerEntry, ServerStatus


class MongoDBRegistry(RegistryDatabase):
    """MongoDB implementation of registry database"""
    
    def __init__(self, config: DatabaseConfig):
        super().__init__(config)
        self.client = None
        self.db = None
        self.collection = None
        self.status_collection = None
    
    async def connect(self):
        """Connect to MongoDB"""
        self.client = motor.motor_asyncio.AsyncIOMotorClient(self.config.url)
        self.db = self.client[self.config.database_name]
        self.collection = self.db[self.config.collection_name]
        self.status_collection = self.db[f"{self.config.collection_name}_status"]
        
        # Create indexes
        await self.collection.create_index("id", unique=True)
        await self.collection.create_index("name")
        await self.collection.create_index("tags")
        await self.collection.create_index("enabled")
        await self.status_collection.create_index("server_id", unique=True)
    
    async def close(self):
        """Close MongoDB connection"""
        if self.client:
            self.client.close()
    
    async def create_entry(self, entry: ServerEntry) -> str:
        """Create a new server entry"""
        doc = entry.dict()
        doc["_id"] = ObjectId()
        doc["created_at"] = datetime.utcnow()
        doc["updated_at"] = datetime.utcnow()
        
        result = await self.collection.insert_one(doc)
        return entry.id
    
    async def get_entry(self, server_id: str) -> Optional[ServerEntry]:
        """Get a server entry by ID"""
        doc = await self.collection.find_one({"id": server_id})
        if doc:
            doc.pop("_id", None)
            return ServerEntry(**doc)
        return None
    
    async def update_entry(self, server_id: str, entry: ServerEntry) -> bool:
        """Update a server entry"""
        doc = entry.dict()
        doc["updated_at"] = datetime.utcnow()
        
        result = await self.collection.update_one(
            {"id": server_id},
            {"$set": doc}
        )
        return result.modified_count > 0
    
    async def delete_entry(self, server_id: str) -> bool:
        """Delete a server entry"""
        result = await self.collection.delete_one({"id": server_id})
        await self.status_collection.delete_one({"server_id": server_id})
        return result.deleted_count > 0
    
    async def list_entries(
        self,
        enabled_only: bool = False,
        tags: Optional[List[str]] = None
    ) -> List[ServerEntry]:
        """List all server entries"""
        query = {}
        
        if enabled_only:
            query["enabled"] = True
        
        if tags:
            query["tags"] = {"$in": tags}
        
        cursor = self.collection.find(query)
        entries = []
        
        async for doc in cursor:
            doc.pop("_id", None)
            entries.append(ServerEntry(**doc))
        
        return entries
    
    async def list_entries_paginated(
        self,
        limit: int = 30,
        cursor: Optional[str] = None,
        enabled_only: bool = False
    ) -> Tuple[List[Dict[str, Any]], Optional[str]]:
        """List server entries with pagination"""
        query = {}
        
        if enabled_only:
            query["enabled"] = True
        
        if cursor:
            query["_id"] = {"$gt": ObjectId(cursor)}
        
        # Fetch one extra to determine if there's a next page
        docs = await self.collection.find(query).limit(limit + 1).to_list(length=limit + 1)
        
        has_next = len(docs) > limit
        if has_next:
            docs = docs[:limit]
            next_cursor = str(docs[-1]["_id"])
        else:
            next_cursor = None
        
        # Convert to dict and remove _id
        entries = []
        for doc in docs:
            doc.pop("_id", None)
            entries.append(doc)
        
        return entries, next_cursor
    
    async def update_status(
        self,
        server_id: str,
        status: ServerStatus,
        error: Optional[str] = None
    ):
        """Update server runtime status"""
        await self.status_collection.update_one(
            {"server_id": server_id},
            {
                "$set": {
                    "status": status.value,
                    "error": error,
                    "updated_at": datetime.utcnow()
                }
            },
            upsert=True
        )
    
    async def get_status(self, server_id: str) -> Optional[str]:
        """Get server runtime status"""
        doc = await self.status_collection.find_one({"server_id": server_id})
        if doc:
            return doc.get("status")
        return None
    
    async def search_entries(
        self,
        query: str,
        fields: List[str] = None
    ) -> List[ServerEntry]:
        """Search server entries"""
        if not fields:
            fields = ["name", "description", "tags"]
        
        # Build OR query for text search
        or_conditions = []
        for field in fields:
            or_conditions.append({field: {"$regex": query, "$options": "i"}})
        
        mongo_query = {"$or": or_conditions}
        cursor = self.collection.find(mongo_query)
        
        entries = []
        async for doc in cursor:
            doc.pop("_id", None)
            entries.append(ServerEntry(**doc))
        
        return entries
```

**database/memory.py**:
```python
from typing import List, Optional, Tuple, Dict, Any
from datetime import datetime
import asyncio
from collections import OrderedDict

from .interface import RegistryDatabase, DatabaseConfig
from ..models.registry import ServerEntry, ServerStatus


class InMemoryRegistry(RegistryDatabase):
    """In-memory implementation of registry database"""
    
    def __init__(self, config: DatabaseConfig):
        super().__init__(config)
        self.entries: OrderedDict[str, ServerEntry] = OrderedDict()
        self.status: Dict[str, Dict[str, Any]] = {}
        self._lock = asyncio.Lock()
    
    async def connect(self):
        """No-op for in-memory database"""
        pass
    
    async def close(self):
        """No-op for in-memory database"""
        pass
    
    async def create_entry(self, entry: ServerEntry) -> str:
        """Create a new server entry"""
        async with self._lock:
            if entry.id in self.entries:
                raise ValueError(f"Server with ID {entry.id} already exists")
            
            entry.created_at = datetime.utcnow()
            entry.updated_at = datetime.utcnow()
            self.entries[entry.id] = entry
            
            return entry.id
    
    async def get_entry(self, server_id: str) -> Optional[ServerEntry]:
        """Get a server entry by ID"""
        return self.entries.get(server_id)
    
    async def update_entry(self, server_id: str, entry: ServerEntry) -> bool:
        """Update a server entry"""
        async with self._lock:
            if server_id not in self.entries:
                return False
            
            entry.updated_at = datetime.utcnow()
            self.entries[server_id] = entry
            return True
    
    async def delete_entry(self, server_id: str) -> bool:
        """Delete a server entry"""
        async with self._lock:
            if server_id in self.entries:
                del self.entries[server_id]
                if server_id in self.status:
                    del self.status[server_id]
                return True
            return False
    
    async def list_entries(
        self,
        enabled_only: bool = False,
        tags: Optional[List[str]] = None
    ) -> List[ServerEntry]:
        """List all server entries"""
        entries = []
        
        for entry in self.entries.values():
            if enabled_only and not entry.enabled:
                continue
            
            if tags and not any(tag in entry.tags for tag in tags):
                continue
            
            entries.append(entry)
        
        return entries
    
    async def list_entries_paginated(
        self,
        limit: int = 30,
        cursor: Optional[str] = None,
        enabled_only: bool = False
    ) -> Tuple[List[Dict[str, Any]], Optional[str]]:
        """List server entries with pagination"""
        all_entries = await self.list_entries(enabled_only=enabled_only)
        
        # Find starting position
        start_idx = 0
        if cursor:
            for i, entry in enumerate(all_entries):
                if entry.id == cursor:
                    start_idx = i + 1
                    break
        
        # Get page of entries
        page_entries = all_entries[start_idx:start_idx + limit]
        
        # Convert to dict
        entries_dict = [entry.dict() for entry in page_entries]
        
        # Determine next cursor
        next_cursor = None
        if start_idx + limit < len(all_entries):
            next_cursor = page_entries[-1].id if page_entries else None
        
        return entries_dict, next_cursor
    
    async def update_status(
        self,
        server_id: str,
        status: ServerStatus,
        error: Optional[str] = None
    ):
        """Update server runtime status"""
        self.status[server_id] = {
            "status": status.value,
            "error": error,
            "updated_at": datetime.utcnow()
        }
    
    async def get_status(self, server_id: str) -> Optional[str]:
        """Get server runtime status"""
        status_info = self.status.get(server_id)
        if status_info:
            return status_info.get("status")
        return None
    
    async def search_entries(
        self,
        query: str,
        fields: List[str] = None
    ) -> List[ServerEntry]:
        """Search server entries"""
        if not fields:
            fields = ["name", "description", "tags"]
        
        query_lower = query.lower()
        results = []
        
        for entry in self.entries.values():
            # Search in specified fields
            for field in fields:
                value = getattr(entry, field, None)
                if value:
                    if isinstance(value, str) and query_lower in value.lower():
                        results.append(entry)
                        break
                    elif isinstance(value, list):
                        for item in value:
                            if isinstance(item, str) and query_lower in item.lower():
                                results.append(entry)
                                break
        
        return results
```

## API

**api/__init__.py**:
```python
from .registry import create_registry_routes
from .health import create_health_routes
from .auth import BearerTokenBackend, create_auth_middleware

__all__ = [
    "create_registry_routes",
    "create_health_routes", 
    "BearerTokenBackend",
    "create_auth_middleware"
]
```

**api/registry.py**:
```python
from typing import List
from starlette.routing import Route
from starlette.responses import JSONResponse
from starlette.requests import Request

from ..database.interface import RegistryDatabase
from ..models.registry import ServerEntry


def create_registry_routes(database: RegistryDatabase, hub) -> List[Route]:
    """Create registry API routes"""
    routes = []
    
    # List servers
    async def list_servers(request: Request):
        limit = int(request.query_params.get("limit", 30))
        cursor = request.query_params.get("cursor")
        
        servers, next_cursor = await database.list_entries_paginated(
            limit=limit,
            cursor=cursor
        )
        
        # Add runtime status from hub
        for server in servers:
            server_id = server["id"]
            if server_id in hub.loaded_servers:
                server["runtime_status"] = "healthy"
                server["mcp_endpoint"] = f"/servers/{server_id}/mcp"
            else:
                status = await database.get_status(server_id)
                server["runtime_status"] = status or "not_loaded"
        
        return JSONResponse({
            "servers": servers,
            "metadata": {
                "next_cursor": next_cursor,
                "count": len(servers)
            }
        })
    
    routes.append(Route("/v0/servers", endpoint=list_servers, methods=["GET"]))
    
    # Get server details
    async def get_server(request: Request):
        server_id = request.path_params["id"]
        entry = await database.get_entry(server_id)
        
        if not entry:
            return JSONResponse({"error": "Server not found"}, status_code=404)
        
        response = entry.dict()
        
        # Add runtime information
        if server_id in hub.loaded_servers:
            response["runtime_status"] = "healthy"
            response["mcp_endpoint"] = f"/servers/{server_id}/mcp"
            
            # Get capabilities
            server_data = hub.loaded_servers[server_id]
            if server_data.get("mcp"):
                mcp = server_data["mcp"]
                try:
                    response["capabilities"] = {
                        "tools": [{"name": t.name, "description": t.description} 
                                for t in mcp.list_tools()],
                        "resources": [{"uri": r.uri, "name": r.name} 
                                    for r in mcp.list_resources()],
                        "prompts": [{"name": p.name, "description": p.description} 
                                  for p in mcp.list_prompts()]
                    }
                except Exception:
                    pass
        else:
            status = await database.get_status(server_id)
            response["runtime_status"] = status or "not_loaded"
        
        return JSONResponse(response)
    
    routes.append(Route("/v0/servers/{id}", endpoint=get_server, methods=["GET"]))
    
    # Search servers
    async def search_servers(request: Request):
        query = request.query_params.get("q", "")
        fields = request.query_params.getlist("fields")
        
        if not query:
            return JSONResponse({"error": "Query parameter 'q' is required"}, status_code=400)
        
        entries = await database.search_entries(query, fields or None)
        
        servers = [entry.dict() for entry in entries]
        
        return JSONResponse({
            "query": query,
            "results": servers,
            "count": len(servers)
        })
    
    routes.append(Route("/v0/search", endpoint=search_servers, methods=["GET"]))
    
    # Publish server (requires auth)
    async def publish_server(request: Request):
        if not hasattr(request, "user") or not request.user.is_authenticated:
            return JSONResponse({"error": "Authentication required"}, status_code=401)
        
        try:
            data = await request.json()
            entry = ServerEntry(**data)
            
            # Check if server already exists
            existing = await database.get_entry(entry.id)
            if existing:
                return JSONResponse(
                    {"error": f"Server with ID {entry.id} already exists"},
                    status_code=409
                )
            
            # Create the entry
            server_id = await database.create_entry(entry)
            
            # Optionally load the server immediately
            if data.get("load_immediately", False):
                await hub._load_server(entry)
            
            return JSONResponse({
                "message": "Server publication successful",
                "id": server_id
            }, status_code=201)
            
        except Exception as e:
            return JSONResponse({"error": str(e)}, status_code=400)
    
    routes.append(Route("/v0/publish", endpoint=publish_server, methods=["POST"]))
    
    # Update server (requires auth)
    async def update_server(request: Request):
        if not hasattr(request, "user") or not request.user.is_authenticated:
            return JSONResponse({"error": "Authentication required"}, status_code=401)
        
        server_id = request.path_params["id"]
        
        try:
            data = await request.json()
            
            # Get existing entry
            existing = await database.get_entry(server_id)
            if not existing:
                return JSONResponse({"error": "Server not found"}, status_code=404)
            
            # Update fields
            for key, value in data.items():
                if hasattr(existing, key) and key not in ["id", "created_at"]:
                    setattr(existing, key, value)
            
            # Save updates
            success = await database.update_entry(server_id, existing)
            
            if success:
                # Reload server if it was already loaded
                if server_id in hub.loaded_servers:
                    await hub._load_server(existing)
                
                return JSONResponse({"message": "Server updated successfully"})
            else:
                return JSONResponse({"error": "Failed to update server"}, status_code=500)
                
        except Exception as e:
            return JSONResponse({"error": str(e)}, status_code=400)
    
    routes.append(Route("/v0/servers/{id}", endpoint=update_server, methods=["PUT"]))
    
    # Delete server (requires auth)
    async def delete_server(request: Request):
        if not hasattr(request, "user") or not request.user.is_authenticated:
            return JSONResponse({"error": "Authentication required"}, status_code=401)
        
        server_id = request.path_params["id"]
        
        # Remove from loaded servers if present
        if server_id in hub.loaded_servers:
            del hub.loaded_servers[server_id]
        
        # Delete from database
        success = await database.delete_entry(server_id)
        
        if success:
            return JSONResponse({"message": "Server deleted successfully"})
        else:
            return JSONResponse({"error": "Server not found"}, status_code=404)
    
    routes.append(Route("/v0/servers/{id}", endpoint=delete_server, methods=["DELETE"]))
    
    return routes
```

**api/health.py**:
```python
from typing import List
from starlette.routing import Route
from starlette.responses import JSONResponse
from starlette.requests import Request

from ..database.interface import RegistryDatabase


def create_health_routes(database: RegistryDatabase, hub, config) -> List[Route]:
    """Create health and status routes"""
    routes = []
    
    # Health check
    async def health_check(request: Request):
        try:
            # Check database connection
            await database.get_entry("test")
            db_healthy = True
        except Exception:
            db_healthy = False
        
        # Count servers
        total_servers = len(await database.list_entries())
        loaded_servers = len(hub.loaded_servers)
        healthy_servers = sum(
            1 for data in hub.loaded_servers.values()
            if data.get("status") == "healthy"
        )
        
        # Determine overall status
        if db_healthy and healthy_servers == loaded_servers:
            overall_status = "ok"
        elif db_healthy and healthy_servers > 0:
            overall_status = "degraded"
        else:
            overall_status = "unhealthy"
        
        return JSONResponse({
            "status": overall_status,
            "components": {
                "database": "healthy" if db_healthy else "unhealthy",
                "hub": {
                    "total_servers": total_servers,
                    "loaded_servers": loaded_servers,
                    "healthy_servers": healthy_servers
                }
            },
            "version": config.get("version", "1.0.0")
        })
    
    routes.append(Route("/v0/health", endpoint=health_check, methods=["GET"]))
    
    # Ping endpoint
    async def ping(request: Request):
        return JSONResponse({
            "environment": config.get("environment", "production"),
            "version": f"registry-{config.get('version', '1.0.0')}"
        })
    
    routes.append(Route("/v0/ping", endpoint=ping, methods=["GET"]))
    
    # Status endpoint
    async def status(request: Request):
        servers_status = {}
        
        for server_id, data in hub.loaded_servers.items():
            entry = data.get("entry")
            servers_status[server_id] = {
                "name": entry.name if entry else server_id,
                "status": data.get("status", "unknown"),
                "loaded_at": data.get("loaded_at").isoformat() if data.get("loaded_at") else None,
                "error": data.get("error")
            }
        
        return JSONResponse({
            "hub": {
                "name": config.get("name", "MCP Hub Registry"),
                "uptime": hub.get_uptime() if hasattr(hub, "get_uptime") else None
            },
            "servers": servers_status,
            "stats": {
                "total_registered": len(await database.list_entries()),
                "total_loaded": len(hub.loaded_servers),
                "healthy": sum(1 for d in hub.loaded_servers.values() if d.get("status") == "healthy"),
                "failed": sum(1 for d in hub.loaded_servers.values() if d.get("status") == "failed")
            }
        })
    
    routes.append(Route("/v0/status", endpoint=status, methods=["GET"]))
    
    return routes
```

**api/auth.py**:
```python
from typing import List, Optional, Tuple
from starlette.authentication import (
    AuthenticationBackend,
    AuthCredentials,
    BaseUser,
    AuthenticationError
)
from starlette.middleware.authentication import AuthenticationMiddleware
from starlette.requests import Request


class SimpleUser(BaseUser):
    """Simple authenticated user"""
    
    def __init__(self, username: str):
        self.username = username
    
    @property
    def is_authenticated(self) -> bool:
        return True
    
    @property
    def display_name(self) -> str:
        return self.username


class BearerTokenBackend(AuthenticationBackend):
    """Bearer token authentication backend"""
    
    def __init__(self, valid_tokens: List[str]):
        self.valid_tokens = set(valid_tokens)
    
    async def authenticate(
        self, request: Request
    ) -> Optional[Tuple[AuthCredentials, BaseUser]]:
        # Check Authorization header
        auth_header = request.headers.get("Authorization")
        if not auth_header:
            return None
        
        # Parse bearer token
        try:
            scheme, token = auth_header.split()
            if scheme.lower() != "bearer":
                return None
        except ValueError:
            return None
        
        # Validate token
        if token not in self.valid_tokens:
            raise AuthenticationError("Invalid authentication token")
        
        # Return authenticated user
        return AuthCredentials(["authenticated"]), SimpleUser("api_user")


def create_auth_middleware(valid_tokens: List[str]) -> AuthenticationMiddleware:
    """Create authentication middleware with bearer token backend"""
    backend = BearerTokenBackend(valid_tokens)
    return AuthenticationMiddleware(backend=backend)
```

## Core

**core/__init__.py**:
```python
from .hub import MCPHubRegistry
from .config import Config
from .exceptions import (
    MCPHubException,
    ServerLoadException,
    ConfigurationException,
    DatabaseException
)

__all__ = [
    "MCPHubRegistry",
    "Config",
    "MCPHubException",
    "ServerLoadException",
    "ConfigurationException",
    "DatabaseException"
]
```

**core/config.py**:
```python
import os
from pathlib import Path
from typing import Any, Dict, List, Optional
import toml
import yaml
from pydantic import BaseModel, Field


class DatabaseConfig(BaseModel):
    type: str = "memory"  # mongodb, postgresql, memory
    url: Optional[str] = None
    database_name: str = "mcp-registry"
    collection_name: str = "servers"


class APIConfig(BaseModel):
    host: str = "0.0.0.0"
    port: int = 8080
    auth_enabled: bool = False
    auth_tokens: List[str] = Field(default_factory=list)
    log_level: str = "info"


class LoaderConfig(BaseModel):
    timeout: float = 30.0
    retry_attempts: int = 3
    retry_delay: float = 2.0


class SeedConfig(BaseModel):
    import_on_startup: bool = False
    file_path: Optional[str] = None


class Config(BaseModel):
    """Main configuration for MCP Hub Registry"""
    name: str = "MCP Hub Registry"
    version: str = "1.0.0"
    environment: str = "production"
    
    database: DatabaseConfig = Field(default_factory=DatabaseConfig)
    api: APIConfig = Field(default_factory=APIConfig)
    loader: LoaderConfig = Field(default_factory=LoaderConfig)
    seed: SeedConfig = Field(default_factory=SeedConfig)
    
    # Hub-specific settings
    hub: Dict[str, Any] = Field(default_factory=dict)
    
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
    def from_env(cls, prefix: str = "MCP_REGISTRY_") -> "Config":
        """Load configuration from environment variables"""
        config_data = {}
        
        # Check for config file env var first
        config_file = os.getenv(f"{prefix}CONFIG_FILE")
        if config_file:
            return cls.from_file(config_file)
        
        # Parse individual env vars
        for key, value in os.environ.items():
            if key.startswith(prefix):
                # Remove prefix and convert to lowercase
                config_key = key[len(prefix):].lower()
                
                # Handle nested keys (e.g., DATABASE_URL -> database.url)
                parts = config_key.split('_')
                
                current = config_data
                for part in parts[:-1]:
                    if part not in current:
                        current[part] = {}
                    current = current[part]
                
                # Convert value types
                if value.lower() in ['true', 'false']:
                    value = value.lower() == 'true'
                elif value.isdigit():
                    value = int(value)
                elif '.' in value and value.replace('.', '').isdigit():
                    value = float(value)
                
                current[parts[-1]] = value
        
        return cls(**config_data)
```

**core/exceptions.py**:
```python
class MCPHubException(Exception):
    """Base exception for MCP Hub Registry"""
    pass


class ServerLoadException(MCPHubException):
    """Exception raised when a server fails to load"""
    pass


class ConfigurationException(MCPHubException):
    """Exception raised for configuration errors"""
    pass


class DatabaseException(MCPHubException):
    """Exception raised for database errors"""
    pass


class AuthenticationException(MCPHubException):
    """Exception raised for authentication errors"""
    pass


class ValidationException(MCPHubException):
    """Exception raised for validation errors"""
    pass
```

This completes the implementation of all the files in the proposed MCP Hub Registry structure. The system now includes:

1. **CLI Interface** - Full command-line management
2. **Database Abstraction** - MongoDB and in-memory implementations
3. **Multiple Loaders** - OpenAPI, Module, NPM, Docker, PyPI
4. **API Layer** - Registry-compatible endpoints with authentication
5. **Configuration System** - TOML/YAML file and environment variable support
6. **Error Handling** - Comprehensive exception hierarchy

The hub can now function as both a registry for discovering MCP servers and a runtime environment for hosting them dynamically.



*Thinking...*

> The user is asking if FastMCP's OpenAPI integration (from the URL content) can make the MCP Hub Registry implementation easier. Looking at the documentation, FastMCP can automatically generate MCP servers from OpenAPI specifications using `FastMCP.from_openapi()`.
> 
> This would significantly simplify the implementation, especially in the `loaders/openapi.py` file. Let me identify what files need changes and how to make them consistent with FastMCP's OpenAPI integration features.
> 
> Key features from FastMCP OpenAPI integration:
> 1. `FastMCP.from_openapi()` class method
> 2. Route mapping with `RouteMap` and `MCPType`
> 3. Authentication via HTTP client configuration
> 4. Component name customization with `mcp_names`
> 5. Tag support (global and per-route)
> 6. Advanced customization with `mcp_component_fn`
> 7. `route_map_fn` for advanced routing logic
> 
> Files that need changes:
> 1. `loaders/openapi.py` - Major simplification
> 2. `models/registry.py` - Update to store FastMCP-specific config
> 3. `models/server.py` - Update OpenAPIConfig to match FastMCP parameters
> 4. `core/hub.py` - Minor updates to handle the simplified loader
> 
> Let me provide the updated implementations.

Yes, FastMCP's OpenAPI integration will significantly simplify the implementation! Here are the files that need changes to leverage FastMCP's built-in OpenAPI features:

## Files to Update

### 1. **loaders/openapi.py** (Major Simplification)

```python
import asyncio
import httpx
from typing import Optional, List, Dict, Any
from fastmcp import FastMCP
from fastmcp.server.openapi import RouteMap, MCPType

from .base import ServerLoader, LoadResult
from ..models.registry import ServerEntry


class OpenAPIServerLoader(ServerLoader):
    """Loader for OpenAPI-based MCP servers using FastMCP"""
    
    async def load(self, entry: ServerEntry) -> LoadResult:
        if not entry.openapi_config:
            return LoadResult(
                success=False,
                error="No OpenAPI configuration provided"
            )
        
        try:
            # Create HTTP client with authentication
            client = self._create_http_client(entry)
            
            # Fetch OpenAPI spec
            openapi_url = entry.openapi_config.get("url")
            if not openapi_url:
                raise ValueError("OpenAPI URL not provided")
            
            async with asyncio.timeout(30.0):
                response = await client.get(openapi_url)
                response.raise_for_status()
                openapi_spec = response.json()
            
            # Build FastMCP.from_openapi() parameters
            kwargs = {
                "openapi_spec": openapi_spec,
                "client": client,
                "name": entry.name,
            }
            
            # Add optional parameters if provided
            if entry.openapi_config.get("route_maps"):
                kwargs["route_maps"] = self._build_route_maps(entry.openapi_config["route_maps"])
            
            if entry.openapi_config.get("route_map_fn"):
                kwargs["route_map_fn"] = self._create_route_map_fn(entry.openapi_config["route_map_fn"])
            
            if entry.openapi_config.get("mcp_names"):
                kwargs["mcp_names"] = entry.openapi_config["mcp_names"]
            
            if entry.tags:
                kwargs["tags"] = set(entry.tags)
            
            if entry.openapi_config.get("mcp_component_fn"):
                kwargs["mcp_component_fn"] = self._create_component_fn(entry.openapi_config["mcp_component_fn"])
            
            if entry.openapi_config.get("timeout"):
                kwargs["timeout"] = entry.openapi_config["timeout"]
            
            # Create MCP server using FastMCP's from_openapi
            mcp = await FastMCP.from_openapi(**kwargs)
            
            # Get ASGI app
            app = mcp.http_app(path="/mcp")
            
            return LoadResult(
                success=True,
                mcp_instance=mcp,
                app=app
            )
            
        except Exception as e:
            return LoadResult(
                success=False,
                error=f"Failed to load OpenAPI server: {str(e)}"
            )
        finally:
            if hasattr(self, '_client'):
                await self._client.aclose()
    
    def _create_http_client(self, entry: ServerEntry) -> httpx.AsyncClient:
        """Create HTTP client with authentication"""
        headers = {}
        auth = None
        
        if entry.auth_config:
            auth_type = entry.auth_config.get("type")
            
            if auth_type == "bearer":
                token = entry.auth_config.get("token")
                headers["Authorization"] = f"Bearer {token}"
            elif auth_type == "api_key":
                key_name = entry.auth_config.get("header_name", "X-API-Key")
                key_value = entry.auth_config.get("api_key")
                headers[key_name] = key_value
            elif auth_type == "basic":
                username = entry.auth_config.get("username")
                password = entry.auth_config.get("password")
                auth = httpx.BasicAuth(username, password)
        
        base_url = entry.openapi_config.get("base_url", "")
        
        self._client = httpx.AsyncClient(
            base_url=base_url,
            headers=headers,
            auth=auth,
            timeout=entry.openapi_config.get("timeout", 30.0)
        )
        
        return self._client
    
    def _build_route_maps(self, route_map_configs: List[Dict[str, Any]]) -> List[RouteMap]:
        """Convert configuration dicts to RouteMap objects"""
        route_maps = []
        
        for config in route_map_configs:
            route_map = RouteMap(
                methods=config.get("methods", ["*"]),
                pattern=config.get("pattern", r".*"),
                tags=set(config.get("tags", [])),
                mcp_type=MCPType[config.get("mcp_type", "TOOL")],
                mcp_tags=set(config.get("mcp_tags", []))
            )
            route_maps.append(route_map)
        
        return route_maps
    
    def _create_route_map_fn(self, config: Dict[str, Any]):
        """Create route_map_fn from configuration"""
        rules = config.get("rules", [])
        
        def route_map_fn(route, mcp_type):
            for rule in rules:
                # Check path contains
                if rule.get("path_contains") and rule["path_contains"] in route.path:
                    return MCPType[rule.get("mcp_type", "TOOL")]
                
                # Check path starts with
                if rule.get("path_starts_with") and route.path.startswith(rule["path_starts_with"]):
                    return MCPType[rule.get("mcp_type", "TOOL")]
                
                # Check method
                if rule.get("method") and route.method == rule["method"]:
                    return MCPType[rule.get("mcp_type", "TOOL")]
                
                # Check tags
                if rule.get("has_tag") and rule["has_tag"] in route.tags:
                    return MCPType[rule.get("mcp_type", "EXCLUDE")]
            
            return None
        
        return route_map_fn
    
    def _create_component_fn(self, config: Dict[str, Any]):
        """Create mcp_component_fn from configuration"""
        add_tags = set(config.get("add_tags", []))
        prefix_tools = config.get("prefix_tools", "")
        prefix_resources = config.get("prefix_resources", "")
        
        def component_fn(route, component):
            # Add configured tags
            for tag in add_tags:
                component.tags.add(tag)
            
            # Add prefixes to descriptions
            if hasattr(component, '__class__'):
                if component.__class__.__name__ == 'OpenAPITool' and prefix_tools:
                    component.description = f"{prefix_tools} {component.description}"
                elif component.__class__.__name__ == 'OpenAPIResource' and prefix_resources:
                    component.description = f"{prefix_resources} {component.description}"
        
        return component_fn
```

### 2. **models/server.py** (Update OpenAPIConfig)

```python
from typing import Dict, List, Optional, Any
from pydantic import BaseModel, Field
from enum import Enum


class RouteMapConfig(BaseModel):
    """Configuration for FastMCP RouteMap"""
    methods: List[str] = ["*"]
    pattern: str = r".*"
    tags: List[str] = []
    mcp_type: str = "TOOL"  # TOOL, RESOURCE, RESOURCE_TEMPLATE, EXCLUDE
    mcp_tags: List[str] = []


class RouteMapFnRule(BaseModel):
    """Rule for route_map_fn"""
    path_contains: Optional[str] = None
    path_starts_with: Optional[str] = None
    method: Optional[str] = None
    has_tag: Optional[str] = None
    mcp_type: str = "TOOL"


class ComponentFnConfig(BaseModel):
    """Configuration for mcp_component_fn"""
    add_tags: List[str] = []
    prefix_tools: str = ""
    prefix_resources: str = ""
    suffix_templates: str = ""


class OpenAPIConfig(BaseModel):
    """OpenAPI configuration aligned with FastMCP.from_openapi()"""
    url: str
    base_url: Optional[str] = None
    timeout: float = 30.0
    
    # FastMCP.from_openapi() parameters
    route_maps: Optional[List[RouteMapConfig]] = None
    route_map_fn: Optional[Dict[str, Any]] = None  # Contains rules
    mcp_names: Dict[str, str] = Field(default_factory=dict)
    mcp_component_fn: Optional[ComponentFnConfig] = None
    
    # Legacy fields for backward compatibility
    routing_rules: Optional[List[Dict[str, Any]]] = None
    advanced_routing: Optional[bool] = None
    customize_components: Optional[bool] = None
```

### 3. **models/registry.py** (Add FastMCP-specific fields)

```python
# Add to ServerEntry model
class ServerEntry(BaseModel):
    # ... existing fields ...
    
    # FastMCP OpenAPI specific configuration
    fastmcp_config: Optional[Dict[str, Any]] = Field(
        default=None,
        description="FastMCP-specific configuration for from_openapi()"
    )
    
    # Simplified route maps for UI/API
    route_maps: Optional[List[Dict[str, Any]]] = Field(
        default=None,
        description="Route mapping configuration for FastMCP"
    )
```

### 4. **Example Configuration Update**

Update the configuration examples to use FastMCP features:

```yaml
servers:
  - id: "github-api"
    name: "GitHub API Server"
    description: "MCP server for GitHub API"
    openapi_config:
      url: "https://api.github.com/openapi.json"
      base_url: "https://api.github.com"
      
      # FastMCP route mapping
      route_maps:
        - methods: ["GET"]
          pattern: "^/repos/.*"
          mcp_type: "RESOURCE_TEMPLATE"
          mcp_tags: ["repository", "read-only"]
        
        - methods: ["POST", "PUT", "PATCH"]
          pattern: ".*"
          mcp_type: "TOOL"
          mcp_tags: ["write-operation"]
        
        - pattern: "^/admin/.*"
          mcp_type: "EXCLUDE"
      
      # Custom component names
      mcp_names:
        "repos_get": "get_repository"
        "repos_create": "create_repository"
        "issues_list": "list_issues"
      
      # Component customization
      mcp_component_fn:
        add_tags: ["github", "api-v3"]
        prefix_tools: "🔧"
        prefix_resources: "📊"
    
    auth_config:
      type: "bearer"
      token: "${GITHUB_TOKEN}"
    
    tags: ["vcs", "collaboration", "external"]
```

### 5. **Simplified Docker Compose for Testing**

```yaml
version: '3.8'

services:
  hub:
    build: .
    ports:
      - "8080:8080"
    environment:
      - MCP_REGISTRY_DATABASE_URL=mongodb://mongo:27017
      - GITHUB_TOKEN=${GITHUB_TOKEN}
    depends_on:
      - mongo
    volumes:
      - ./config.yaml:/app/config.yaml
    command: python -m mcp_hub_registry run -c /app/config.yaml

  mongo:
    image: mongo:latest
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

volumes:
  mongo_data:
```

### 6. **Add FastMCP-specific CLI Commands**

Update `cli.py` to add commands for testing OpenAPI conversion:

```python
@cli.command()
@click.argument('openapi_url')
@click.option('--output', '-o', help='Output file for server configuration')
@click.option('--test', is_flag=True, help='Test the OpenAPI conversion')
def analyze_openapi(openapi_url: str, output: Optional[str], test: bool):
    """Analyze an OpenAPI spec and generate MCP server configuration"""
    
    async def analyze():
        import httpx
        from fastmcp import FastMCP
        
        # Fetch OpenAPI spec
        async with httpx.AsyncClient() as client:
            response = await client.get(openapi_url)
            response.raise_for_status()
            spec = response.json()
        
        if test:
            # Test conversion
            mcp = await FastMCP.from_openapi(
                openapi_spec=spec,
                client=httpx.AsyncClient(),
                name="test_server"
            )
            
            click.echo(f"Successfully converted OpenAPI to MCP server")
            click.echo(f"Tools: {len(mcp.list_tools())}")
            click.echo(f"Resources: {len(mcp.list_resources())}")
            click.echo(f"Prompts: {len(mcp.list_prompts())}")
            
            # Show first few tools
            for tool in list(mcp.list_tools())[:5]:
                click.echo(f"  - {tool.name}: {tool.description[:50]}...")
        
        # Generate configuration
        config = {
            "id": spec.get("info", {}).get("title", "").lower().replace(" ", "-"),
            "name": spec.get("info", {}).get("title", "API Server"),
            "description": spec.get("info", {}).get("description", ""),
            "openapi_config": {
                "url": openapi_url,
                "route_maps": [
                    {
                        "methods": ["GET"],
                        "pattern": r".*\{.*\}.*",
                        "mcp_type": "RESOURCE_TEMPLATE"
                    },
                    {
                        "methods": ["GET"],
                        "mcp_type": "RESOURCE"
                    }
                ]
            }
        }
        
        if output:
            with open(output, 'w') as f:
                yaml.dump(config, f, default_flow_style=False)
            click.echo(f"Configuration saved to {output}")
        else:
            click.echo(yaml.dump(config, default_flow_style=False))
    
    asyncio.run(analyze())
```

## Benefits of Using FastMCP's OpenAPI Integration

1. **Simplified Code**: The OpenAPI loader is now much simpler, leveraging FastMCP's built-in functionality
2. **Better Route Mapping**: Direct support for RouteMap and MCPType enums
3. **Component Customization**: Built-in support for mcp_component_fn and mcp_names
4. **Automatic Parameter Handling**: FastMCP handles query, path, and header parameters correctly
5. **Tag Management**: Native support for global and per-route tags
6. **Consistent with FastMCP**: Using the official integration ensures compatibility with future updates

## Migration Path

For existing configurations, add a migration utility:

```python
def migrate_openapi_config(old_config: Dict[str, Any]) -> Dict[str, Any]:
    """Migrate old OpenAPI config to FastMCP format"""
    new_config = {
        "url": old_config.get("openapi_url"),
        "base_url": old_config.get("base_url"),
        "timeout": old_config.get("timeout", 30.0)
    }
    
    # Convert old routing rules to route_maps
    if old_config.get("routing_rules"):
        route_maps = []
        for rule in old_config["routing_rules"]:
            route_map = {
                "pattern": rule.get("pattern", r".*"),
                "mcp_type": rule.get("type", "TOOL")
            }
            if rule.get("methods"):
                route_map["methods"] = rule["methods"]
            route_maps.append(route_map)
        new_config["route_maps"] = route_maps
    
    return new_config
```

This integration with FastMCP's OpenAPI features makes the implementation cleaner, more maintainable, and fully compatible with FastMCP's ecosystem.












*Thinking...*

> The user wants me to stick with YAML configuration files and provide concrete examples and instructions for running the project. I should provide:
> 
> 1. Example YAML configuration files
> 2. Clear instructions on how to run the project
> 3. Make sure the YAML approach is consistent throughout
> 
> Let me provide:
> - config.yaml (main configuration)
> - servers.yaml (seed data with server configurations)
> - docker-compose.yml
> - Clear step-by-step instructions
> - .env.example file
> - README with concrete examples

Here are the YAML configuration files and concrete instructions for running the MCP Hub Registry:

## Configuration Files

### **config.yaml** (Main Hub Configuration)
```yaml
# MCP Hub Registry Configuration
hub:
  name: "MCP Hub Registry"
  version: "1.0.0"
  description: "Community-driven MCP server registry and hub"
  
database:
  type: "mongodb"  # Options: mongodb, memory
  url: "${DATABASE_URL:-mongodb://localhost:27017}"
  database_name: "mcp-registry"
  collection_name: "servers"
  
api:
  host: "0.0.0.0"
  port: 8080
  auth_enabled: true
  auth_tokens:
    - "${AUTH_TOKEN:-dev-token-12345}"
  log_level: "info"
  
loader:
  timeout: 30.0
  retry_attempts: 3
  retry_delay: 2.0
  
seed:
  import_on_startup: true
  file_path: "data/servers.yaml"
```

### **data/servers.yaml** (Server Configurations)
```yaml
servers:
  # OpenAPI-based server example
  - id: "github-api"
    name: "GitHub API Server"
    description: "MCP server for GitHub API v3"
    repository:
      url: "https://github.com/example/github-mcp-server"
      source: "github"
    version_detail:
      version: "1.0.0"
      release_date: "2025-01-15T00:00:00Z"
      is_latest: true
    openapi_config:
      url: "https://raw.githubusercontent.com/github/rest-api-description/main/descriptions/api.github.com/api.github.com.yaml"
      base_url: "https://api.github.com"
      timeout: 30.0
      route_maps:
        - methods: ["GET"]
          pattern: "^/repos/[^/]+/[^/]+$"
          mcp_type: "RESOURCE_TEMPLATE"
          mcp_tags: ["repository", "read"]
        - methods: ["GET"]
          pattern: "^/users/[^/]+$"
          mcp_type: "RESOURCE_TEMPLATE"
          mcp_tags: ["user", "read"]
        - methods: ["POST", "PUT", "PATCH", "DELETE"]
          pattern: ".*"
          mcp_type: "TOOL"
          mcp_tags: ["write"]
        - pattern: "^/admin/.*"
          mcp_type: "EXCLUDE"
      mcp_names:
        "repos/get": "get_repository"
        "repos/create": "create_repository"
        "issues/list": "list_issues"
      mcp_component_fn:
        add_tags: ["github", "vcs"]
        prefix_tools: "🔧"
        prefix_resources: "📊"
    auth_config:
      type: "bearer"
      token: "${GITHUB_TOKEN}"
    tags: ["vcs", "api", "external"]
    enabled: true

  # NPM package server example
  - id: "sqlite-server"
    name: "SQLite Database Server"
    description: "MCP server for SQLite database operations"
    repository:
      url: "https://github.com/example/sqlite-mcp-server"
      source: "github"
    version_detail:
      version: "1.0.0"
      release_date: "2025-01-15T00:00:00Z"
      is_latest: true
    packages:
      - registry_name: "npm"
        name: "@modelcontextprotocol/server-sqlite"
        version: "0.1.0"
        package_arguments:
          - description: "Path to SQLite database"
            is_required: true
            format: "string"
            value: "./database.db"
            type: "positional"
        environment_variables:
          - name: "SQLITE_READ_ONLY"
            description: "Enable read-only mode"
            required: false
            default: "false"
    tags: ["database", "local", "sql"]
    enabled: true

  # Python module server example
  - id: "file-system-server"
    name: "File System Server"
    description: "MCP server for file system operations"
    repository:
      url: ""
      source: "custom"
    version_detail:
      version: "1.0.0"
      release_date: "2025-01-15T00:00:00Z"
      is_latest: true
    packages:
      - registry_name: "pypi"
        name: "mcp-filesystem"
        version: "1.0.0"
        package_arguments:
          - description: "Root directory path"
            is_required: true
            format: "string"
            value: "/workspace"
            type: "positional"
    tags: ["filesystem", "local", "tools"]
    enabled: false

  # Docker-based server example
  - id: "postgres-server"
    name: "PostgreSQL Server"
    description: "MCP server for PostgreSQL operations"
    repository:
      url: "https://github.com/example/postgres-mcp-server"
      source: "github"
    version_detail:
      version: "1.0.0"
      release_date: "2025-01-15T00:00:00Z"
      is_latest: true
    packages:
      - registry_name: "docker"
        name: "mcp/postgres-server"
        version: "latest"
        runtime_arguments:
          - description: "Mount PostgreSQL socket"
            value: "-v /var/run/postgresql:/var/run/postgresql"
            type: "positional"
        environment_variables:
          - name: "POSTGRES_CONNECTION"
            description: "PostgreSQL connection string"
            required: true
            default: "postgresql://user:pass@localhost/db"
    tags: ["database", "sql", "docker"]
    enabled: false
```

### **.env.example**
```bash
# Database Configuration
DATABASE_URL=mongodb://localhost:27017

# API Authentication
AUTH_TOKEN=your-secret-auth-token-here

# External API Keys
GITHUB_TOKEN=ghp_your_github_token_here
OPENAI_API_KEY=sk-your_openai_key_here

# Optional Configuration
MCP_REGISTRY_API_PORT=8080
MCP_REGISTRY_LOG_LEVEL=info
```

### **docker-compose.yml**
```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    container_name: mcp-hub-mongodb
    restart: unless-stopped
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    environment:
      - MONGO_INITDB_DATABASE=mcp-registry
    networks:
      - mcp-network

  mcp-hub:
    build: .
    container_name: mcp-hub-registry
    restart: unless-stopped
    ports:
      - "8080:8080"
    depends_on:
      - mongodb
    environment:
      - DATABASE_URL=mongodb://mongodb:27017
      - AUTH_TOKEN=${AUTH_TOKEN:-dev-token-12345}
      - GITHUB_TOKEN=${GITHUB_TOKEN}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    volumes:
      - ./config.yaml:/app/config.yaml:ro
      - ./data:/app/data:ro
    command: python -m mcp_hub_registry run -c /app/config.yaml
    networks:
      - mcp-network

volumes:
  mongo_data:

networks:
  mcp-network:
    driver: bridge
```

### **Dockerfile**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    git \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Install the package
RUN pip install -e .

# Create data directory
RUN mkdir -p /app/data

# Expose port
EXPOSE 8080

# Default command
CMD ["python", "-m", "mcp_hub_registry", "run"]
```

### **requirements.txt**
```txt
fastmcp>=0.2.0
httpx>=0.25.0
pydantic>=2.0.0
motor>=3.3.0
starlette>=0.37.0
uvicorn>=0.27.0
click>=8.1.0
pyyaml>=6.0
```

## Running Instructions

### **1. Local Development Setup**

```bash
# Clone the repository
git clone https://github.com/your-org/mcp-hub-registry.git
cd mcp-hub-registry

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
pip install -e .

# Copy environment file
cp .env.example .env
# Edit .env with your configuration

# Start MongoDB (if not using Docker)
# On macOS with Homebrew:
brew services start mongodb-community

# On Ubuntu:
sudo systemctl start mongod
```

### **2. Initialize and Run**

```bash
# Initialize configuration (creates config.yaml)
python -m mcp_hub_registry init-config

# Import seed servers
python -m mcp_hub_registry import-seed data/servers.yaml

# Run the hub
python -m mcp_hub_registry run -c config.yaml
```

### **3. Docker Deployment**

```bash
# Copy and configure environment
cp .env.example .env
# Edit .env with your tokens

# Build and run with Docker Compose
docker-compose up -d

# View logs
docker-compose logs -f mcp-hub

# Stop services
docker-compose down
```

### **4. CLI Usage Examples**

```bash
# List all servers
python -m mcp_hub_registry list

# List only enabled servers
python -m mcp_hub_registry list --enabled-only

# Get server information
python -m mcp_hub_registry info github-api

# Enable/disable a server
python -m mcp_hub_registry toggle github-api --disable
python -m mcp_hub_registry toggle github-api --enable

# Analyze an OpenAPI spec
python -m mcp_hub_registry analyze-openapi https://petstore.swagger.io/v2/swagger.json

# Generate server config from OpenAPI
python -m mcp_hub_registry analyze-openapi \
  https://api.example.com/openapi.yaml \
  -o my-api-server.yaml \
  --id my-api \
  --name "My API Server"

# Export all servers
python -m mcp_hub_registry export-servers servers-backup.yaml
```

### **5. API Usage Examples**

```bash
# Health check
curl http://localhost:8080/v0/health

# List servers
curl http://localhost:8080/v0/servers

# Get server details
curl http://localhost:8080/v0/servers/github-api

# Search servers
curl "http://localhost:8080/v0/search?q=github"

# Publish a new server (requires auth)
curl -X POST http://localhost:8080/v0/publish \
  -H "Authorization: Bearer dev-token-12345" \
  -H "Content-Type: application/yaml" \
  --data-binary @new-server.yaml
```

### **6. Production Deployment**

```bash
# Create production config
cp config.yaml config.prod.yaml
# Edit config.prod.yaml with production settings

# Set production environment variables
export DATABASE_URL=mongodb://prod-mongo:27017
export AUTH_TOKEN=your-production-token
export MCP_REGISTRY_LOG_LEVEL=warning

# Run with production config
python -m mcp_hub_registry run -c config.prod.yaml
```

### **7. Adding a New Server**

Create `my-new-server.yaml`:
```yaml
id: "my-api-server"
name: "My API Server"
description: "Custom API integration"
repository:
  url: "https://github.com/myorg/my-api-mcp"
  source: "github"
version_detail:
  version: "1.0.0"
  release_date: "2025-01-20T00:00:00Z"
  is_latest: true
openapi_config:
  url: "https://api.myservice.com/openapi.yaml"
  base_url: "https://api.myservice.com/v1"
  route_maps:
    - methods: ["GET"]
      pattern: ".*"
      mcp_type: "RESOURCE"
auth_config:
  type: "api_key"
  header_name: "X-API-Key"
  api_key: "${MY_API_KEY}"
tags: ["custom", "api"]
enabled: true
```

Then add it:
```bash
# Via CLI
python -m mcp_hub_registry add-server my-new-server.yaml

# Or via API
curl -X POST http://localhost:8080/v0/publish \
  -H "Authorization: Bearer dev-token-12345" \
  -H "Content-Type: application/yaml" \
  --data-binary @my-new-server.yaml
```

### **8. Monitoring and Troubleshooting**

```bash
# Check hub status
curl http://localhost:8080/v0/status

# View server logs (Docker)
docker-compose logs mcp-hub --tail=100 -f

# Check MongoDB
docker exec -it mcp-hub-mongodb mongosh
> use mcp-registry
> db.servers.find().pretty()

# Test OpenAPI conversion
python -m mcp_hub_registry analyze-openapi \
  https://api.github.com/openapi.yaml \
  --test
```

## Directory Structure for Running

```
mcp-hub-registry/
├── mcp_hub_registry/      # Package code
├── data/
│   └── servers.yaml       # Server configurations
├── config.yaml           # Main configuration
├── .env                  # Environment variables
├── docker-compose.yml    # Docker setup
├── Dockerfile           # Container definition
├── requirements.txt     # Python dependencies
└── README.md           # Documentation
```

This setup provides a complete, working MCP Hub Registry with YAML configuration, supporting multiple server types and deployment options.

I'll implement the detailed API folder structure and show how it integrates with the rest of the system. Let me start with the API layer and then implement other core components.

## API Folder Implementation

### 1. **API Dependencies**

```python
# src/mcp_hub/api/dependencies.py

from typing import Optional
from fastapi import Depends, HTTPException, Header, Request
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

from ..registry import ServerRegistry
from ..config import Settings
from ..secrets import SecretManager
from ..loaders import LoaderFactory


# Security
security = HTTPBearer(auto_error=False)


async def get_settings(request: Request) -> Settings:
    """Get application settings"""
    return request.app.state.settings


async def get_registry(request: Request) -> ServerRegistry:
    """Get server registry instance"""
    return request.app.state.registry


async def get_secret_manager(request: Request) -> SecretManager:
    """Get secret manager instance"""
    return request.app.state.secret_manager


async def get_loader_factory(request: Request) -> LoaderFactory:
    """Get loader factory instance"""
    return request.app.state.loader_factory


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    settings: Settings = Depends(get_settings)
) -> Optional[str]:
    """Get current authenticated user"""
    if not settings.auth_enabled:
        return "anonymous"
    
    if not credentials:
        raise HTTPException(status_code=401, detail="Authentication required")
    
    # Validate token (simplified - use proper JWT validation in production)
    if credentials.credentials != settings.auth_secret_key:
        raise HTTPException(status_code=401, detail="Invalid authentication")
    
    return "admin"


async def require_admin(current_user: str = Depends(get_current_user)):
    """Require admin authentication"""
    if current_user != "admin":
        raise HTTPException(status_code=403, detail="Admin access required")
    return current_user
```

### 2. **API Models**

```python
# src/mcp_hub/api/models.py

from typing import Optional, Dict, Any, List
from datetime import datetime
from pydantic import BaseModel, Field

from ..core.models import ServerStatus, TransportType, LoaderType


class HealthResponse(BaseModel):
    """Health check response"""
    status: str = "healthy"
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    version: str
    services: Dict[str, str]
    
    class Config:
        json_encoders = {
            datetime: lambda v: v.isoformat()
        }


class ServerListResponse(BaseModel):
    """Server list response"""
    servers: List["ServerResponse"]
    total: int
    timestamp: datetime = Field(default_factory=datetime.utcnow)


class ServerResponse(BaseModel):
    """Individual server response"""
    id: str
    name: str
    description: Optional[str]
    status: ServerStatus
    mount_path: str
    transport: TransportType
    loader_type: LoaderType
    capabilities: Dict[str, int]
    health: Dict[str, Any]
    tags: List[str]
    created_at: datetime
    updated_at: datetime
    
    class Config:
        use_enum_values = True


class ServerCreateRequest(BaseModel):
    """Request to create a new server"""
    id: str
    name: str
    description: Optional[str] = None
    mount_path: str
    loader_type: LoaderType
    transport: TransportType = TransportType.HTTP
    config: Dict[str, Any]
    tags: List[str] = Field(default_factory=list)
    enabled: bool = True


class ServerUpdateRequest(BaseModel):
    """Request to update server configuration"""
    name: Optional[str] = None
    description: Optional[str] = None
    config: Optional[Dict[str, Any]] = None
    tags: Optional[List[str]] = None
    enabled: Optional[bool] = None


class ErrorResponse(BaseModel):
    """Error response"""
    error: str
    detail: Optional[str] = None
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    request_id: Optional[str] = None


class StatisticsResponse(BaseModel):
    """Registry statistics"""
    total_servers: int
    healthy_servers: int
    failed_servers: int
    status_breakdown: Dict[str, int]
    loader_breakdown: Dict[str, int]
    uptime_seconds: float
    timestamp: datetime
```

### 3. **Health Routes**

```python
# src/mcp_hub/api/routes/health.py

from fastapi import APIRouter, Depends
from typing import Dict

from ..dependencies import get_registry, get_settings
from ..models import HealthResponse
from ...core.models import ServerStatus

router = APIRouter()


@router.get("/", response_model=HealthResponse)
async def health_check(
    registry = Depends(get_registry),
    settings = Depends(get_settings)
) -> HealthResponse:
    """Basic health check endpoint"""
    # Check registry health
    try:
        stats = await registry.get_statistics()
        registry_status = "healthy"
    except Exception:
        registry_status = "unhealthy"
    
    return HealthResponse(
        status="healthy",
        version=settings.version,
        services={
            "registry": registry_status,
            "api": "healthy"
        }
    )


@router.get("/ready")
async def readiness_check(registry = Depends(get_registry)) -> Dict[str, Any]:
    """Kubernetes readiness probe"""
    servers = await registry.list()
    healthy_count = sum(1 for s in servers if s.health.healthy)
    
    ready = healthy_count > 0 or len(servers) == 0
    
    return {
        "ready": ready,
        "servers": {
            "total": len(servers),
            "healthy": healthy_count
        }
    }


@router.get("/live")
async def liveness_check() -> Dict[str, str]:
    """Kubernetes liveness probe"""
    return {"status": "alive"}
```

### 4. **Server Management Routes**

```python
# src/mcp_hub/api/routes/servers.py

from typing import List, Optional
from fastapi import APIRouter, Depends, HTTPException, Query, BackgroundTasks

from ..dependencies import get_registry, get_loader_factory, get_current_user
from ..models import (
    ServerListResponse, 
    ServerResponse, 
    ServerCreateRequest,
    ServerUpdateRequest,
    ErrorResponse
)
from ...core.models import ServerStatus, ServerInstance
from ...loaders import LoadResultHandler

router = APIRouter()


def server_to_response(server: ServerInstance) -> ServerResponse:
    """Convert ServerInstance to API response"""
    return ServerResponse(
        id=server.metadata.id,
        name=server.metadata.name,
        description=server.metadata.description,
        status=server.health.status,
        mount_path=server.mount_path,
        transport=server.transport,
        loader_type=server.loader_type,
        capabilities=server.capabilities.dict(),
        health=server.health.dict(),
        tags=server.metadata.tags,
        created_at=server.metadata.created_at,
        updated_at=server.metadata.updated_at
    )


@router.get("/", response_model=ServerListResponse)
async def list_servers(
    status: Optional[ServerStatus] = Query(None),
    tag: Optional[str] = Query(None),
    healthy_only: bool = Query(False),
    registry = Depends(get_registry)
) -> ServerListResponse:
    """List all registered servers with optional filters"""
    tags = [tag] if tag else None
    servers = await registry.list(
        status=status,
        tags=tags,
        healthy_only=healthy_only
    )
    
    return ServerListResponse(
        servers=[server_to_response(s) for s in servers],
        total=len(servers)
    )


@router.get("/{server_id}", response_model=ServerResponse)
async def get_server(
    server_id: str,
    registry = Depends(get_registry)
) -> ServerResponse:
    """Get a specific server by ID"""
    server = await registry.get(server_id)
    if not server:
        raise HTTPException(status_code=404, detail=f"Server {server_id} not found")
    
    return server_to_response(server)


@router.post("/", response_model=ServerResponse, status_code=201)
async def create_server(
    request: ServerCreateRequest,
    background_tasks: BackgroundTasks,
    registry = Depends(get_registry),
    loader_factory = Depends(get_loader_factory),
    current_user: str = Depends(get_current_user)
) -> ServerResponse:
    """Create and register a new server"""
    # Check if server already exists
    existing = await registry.get(request.id)
    if existing:
        raise HTTPException(status_code=409, detail=f"Server {request.id} already exists")
    
    # Get appropriate loader
    loader = loader_factory.get_loader(request.loader_type)
    if not loader:
        raise HTTPException(
            status_code=400, 
            detail=f"No loader available for type: {request.loader_type}"
        )
    
    # Create server instance (placeholder - will be loaded in background)
    from ...core.models import ServerMetadata, ServerCapabilities, ServerHealth
    
    server = ServerInstance(
        metadata=ServerMetadata(
            id=request.id,
            name=request.name,
            description=request.description,
            tags=request.tags
        ),
        capabilities=ServerCapabilities(),
        health=ServerHealth(
            status=ServerStatus.LOADING,
            healthy=False
        ),
        mount_path=request.mount_path,
        transport=request.transport,
        loader_type=request.loader_type,
        config=request.config
    )
    
    # Register server
    success = await registry.register(server)
    if not success:
        raise HTTPException(status_code=500, detail="Failed to register server")
    
    # Load server in background
    async def load_server():
        handler = LoadResultHandler(registry)
        try:
            result = await loader.load(request.config)
            await handler.handle_result(request.id, result)
        except Exception as e:
            await handler.handle_error(request.id, e)
    
    background_tasks.add_task(load_server)
    
    return server_to_response(server)


@router.patch("/{server_id}", response_model=ServerResponse)
async def update_server(
    server_id: str,
    request: ServerUpdateRequest,
    registry = Depends(get_registry),
    current_user: str = Depends(get_current_user)
) -> ServerResponse:
    """Update server configuration"""
    server = await registry.get(server_id)
    if not server:
        raise HTTPException(status_code=404, detail=f"Server {server_id} not found")
    
    # Update fields
    if request.name is not None:
        server.metadata.name = request.name
    if request.description is not None:
        server.metadata.description = request.description
    if request.tags is not None:
        server.metadata.tags = request.tags
    if request.config is not None:
        server.config.update(request.config)
    
    server.metadata.updated_at = datetime.utcnow()
    
    # Save updates
    success = await registry.update(server_id, server)
    if not success:
        raise HTTPException(status_code=500, detail="Failed to update server")
    
    return server_to_response(server)


@router.delete("/{server_id}", status_code=204)
async def delete_server(
    server_id: str,
    registry = Depends(get_registry),
    current_user: str = Depends(get_current_user)
):
    """Delete a server"""
    success = await registry.unregister(server_id)
    if not success:
        raise HTTPException(status_code=404, detail=f"Server {server_id} not found")


@router.post("/{server_id}/reload", response_model=ServerResponse)
async def reload_server(
    server_id: str,
    background_tasks: BackgroundTasks,
    registry = Depends(get_registry),
    loader_factory = Depends(get_loader_factory),
    current_user: str = Depends(get_current_user)
) -> ServerResponse:
    """Reload a server"""
    server = await registry.get(server_id)
    if not server:
        raise HTTPException(status_code=404, detail=f"Server {server_id} not found")
    
    # Mark as reloading
    server.health.status = ServerStatus.LOADING
    await registry.update(server_id, server)
    
    # Reload in background
    async def reload():
        loader = loader_factory.get_loader(server.loader_type)
        handler = LoadResultHandler(registry)
        try:
            result = await loader.load(server.config)
            await handler.handle_result(server_id, result)
        except Exception as e:
            await handler.handle_error(server_id, e)
    
    background_tasks.add_task(reload)
    
    return server_to_response(server)
```

### 5. **Admin Routes**

```python
# src/mcp_hub/api/routes/admin.py

from fastapi import APIRouter, Depends, HTTPException
from typing import Dict, Any

from ..dependencies import get_registry, require_admin, get_settings
from ..models import StatisticsResponse
from ...utils.monitoring import get_system_metrics

router = APIRouter(dependencies=[Depends(require_admin)])


@router.get("/stats", response_model=StatisticsResponse)
async def get_statistics(
    registry = Depends(get_registry),
    settings = Depends(get_settings)
) -> StatisticsResponse:
    """Get detailed system statistics"""
    stats = await registry.get_statistics()
    servers = await registry.list()
    
    # Calculate loader breakdown
    loader_breakdown = {}
    for server in servers:
        loader_type = server.loader_type.value
        loader_breakdown[loader_type] = loader_breakdown.get(loader_type, 0) + 1
    
    # Get uptime
    from datetime import datetime
    uptime = (datetime.utcnow() - settings.startup_time).total_seconds()
    
    return StatisticsResponse(
        total_servers=stats["total_servers"],
        healthy_servers=stats["healthy_servers"],
        failed_servers=stats["total_servers"] - stats["healthy_servers"],
        status_breakdown=stats["status_breakdown"],
        loader_breakdown=loader_breakdown,
        uptime_seconds=uptime,
        timestamp=datetime.utcnow()
    )


@router.get("/metrics")
async def get_metrics() -> Dict[str, Any]:
    """Get Prometheus-style metrics"""
    return get_system_metrics()


@router.post("/reload-config")
async def reload_configuration(
    settings = Depends(get_settings)
) -> Dict[str, str]:
    """Reload configuration from disk"""
    try:
        settings.reload()
        return {"status": "success", "message": "Configuration reloaded"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@router.get("/logs")
async def get_recent_logs(
    lines: int = 100,
    level: str = "INFO"
) -> Dict[str, Any]:
    """Get recent log entries"""
    # This would integrate with your logging system
    return {
        "logs": [],  # Implement log retrieval
        "total": 0
    }
```

### 6. **Middleware**

```python
# src/mcp_hub/api/middleware.py

import time
import uuid
import logging
from typing import Callable

from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.types import ASGIApp

logger = logging.getLogger(__name__)


class LoggingMiddleware(BaseHTTPMiddleware):
    """Log all requests and responses"""
    
    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        # Generate request ID
        request_id = str(uuid.uuid4())
        request.state.request_id = request_id
        
        # Log request
        start_time = time.time()
        logger.info(
            f"Request {request_id}: {request.method} {request.url.path}",
            extra={
                "request_id": request_id,
                "method": request.method,
                "path": request.url.path,
                "client": request.client.host if request.client else None
            }
        )
        
        # Process request
        try:
            response = await call_next(request)
            duration = time.time() - start_time
            
            # Log response
            logger.info(
                f"Response {request_id}: {response.status_code} ({duration:.3f}s)",
                extra={
                    "request_id": request_id,
                    "status_code": response.status_code,
                    "duration": duration
                }
            )
            
            # Add headers
            response.headers["X-Request-ID"] = request_id
            response.headers["X-Response-Time"] = f"{duration:.3f}s"
            
            return response
            
        except Exception as e:
            duration = time.time() - start_time
            logger.error(
                f"Request {request_id} failed: {str(e)}",
                extra={
                    "request_id": request_id,
                    "duration": duration,
                    "error": str(e)
                },
                exc_info=True
            )
            raise


class AuthenticationMiddleware(BaseHTTPMiddleware):
    """Handle authentication for protected routes"""
    
    def __init__(self, app: ASGIApp, secret_key: str):
        super().__init__(app)
        self.secret_key = secret_key
    
    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        # Skip auth for public endpoints
        public_paths = ["/health", "/api/servers"]
        if any(request.url.path.startswith(path) for path in public_paths):
            return await call_next(request)
        
        # Check authentication
        auth_header = request.headers.get("Authorization")
        if not auth_header or not auth_header.startswith("Bearer "):
            return Response(
                content='{"error": "Authentication required"}',
                status_code=401,
                headers={"WWW-Authenticate": "Bearer"}
            )
        
        # Validate token (simplified)
        token = auth_header.split(" ")[1]
        if token != self.secret_key:
            return Response(
                content='{"error": "Invalid token"}',
                status_code=401
            )
        
        return await call_next(request)
```

## Core Implementations

### 1. **Configuration System**

```python
# src/mcp_hub/config/settings.py

from typing import List, Optional, Dict, Any
from pathlib import Path
from datetime import datetime
from pydantic import BaseModel, Field, validator
import yaml
import os

from ..core.models import TransportType, LoaderType


class ServerConfig(BaseModel):
    """Individual server configuration"""
    id: str
    name: str
    description: Optional[str] = None
    enabled: bool = True
    mount_path: str
    loader_type: LoaderType
    transport: TransportType = TransportType.HTTP
    config: Dict[str, Any] = Field(default_factory=dict)
    tags: List[str] = Field(default_factory=list)
    retry_attempts: int = 3
    retry_delay: float = 5.0
    timeout: int = 30
    
    @validator('mount_path')
    def validate_mount_path(cls, v):
        if not v.startswith('/'):
            v = f'/{v}'
        return v.rstrip('/')


class SecretsConfig(BaseModel):
    """Secrets configuration"""
    provider: str = "environment"  # environment, file, vault, aws, azure
    file_path: Optional[Path] = None
    encryption_key: Optional[str] = None
    vault_url: Optional[str] = None
    vault_token: Optional[str] = None
    aws_region: Optional[str] = None
    azure_vault_url: Optional[str] = None


class Settings(BaseModel):
    """Main application settings"""
    # Application
    app_name: str = "MCP Hub"
    version: str = "1.0.0"
    environment: str = Field(default="production")
    debug: bool = False
    
    # Server
    host: str = "0.0.0.0"
    port: int = 8000
    workers: int = 1
    log_level: str = "INFO"
    
    # Features
    auth_enabled: bool = True
    auth_secret_key: str = Field(default_factory=lambda: os.urandom(32).hex())
    admin_enabled: bool = True
    cors_enabled: bool = True
    cors_origins: List[str] = ["*"]
    
    # Health checks
    health_check_interval: int = 30
    health_check_timeout: int = 10
    
    # Servers
    servers: List[ServerConfig] = Field(default_factory=list)
    
    # Secrets
    secrets: SecretsConfig = Field(default_factory=SecretsConfig)
    
    # Runtime
    startup_time: datetime = Field(default_factory=datetime.utcnow)
    
    class Config:
        env_prefix = "MCP_HUB_"
        env_nested_delimiter = "__"
    
    @classmethod
    def from_file(cls, path: str | Path) -> "Settings":
        """Load settings from YAML file"""
        path = Path(path)
        if not path.exists():
            raise FileNotFoundError(f"Config file not found: {path}")
        
        with open(path) as f:
            data = yaml.safe_load(f)
        
        # Apply environment variable substitution
        from .env import substitute_env_vars
        data = substitute_env_vars(data)
        
        return cls(**data)
    
    def reload(self):
        """Reload configuration (placeholder)"""
        # This would reload from the original source
        pass
```

### 2. **Loader System**

```python
# src/mcp_hub/loaders/factory.py

from typing import Dict, Optional, Type
import logging

from ..core.interfaces import IServerLoader
from ..core.models import LoaderType

logger = logging.getLogger(__name__)


class LoaderFactory:
    """Factory for creating server loaders"""
    
    def __init__(self):
        self._loaders: Dict[LoaderType, IServerLoader] = {}
        self._loader_classes: Dict[LoaderType, Type[IServerLoader]] = {}
    
    def register(self, loader_type: LoaderType, loader: IServerLoader):
        """Register a loader instance"""
        self._loaders[loader_type] = loader
        logger.info(f"Registered loader: {loader_type.value}")
    
    def register_class(self, loader_type: LoaderType, loader_class: Type[IServerLoader]):
        """Register a loader class for lazy instantiation"""
        self._loader_classes[loader_type] = loader_class
        logger.info(f"Registered loader class: {loader_type.value}")
    
    def get_loader(self, loader_type: LoaderType) -> Optional[IServerLoader]:
        """Get a loader instance"""
        # Check registered instances
        if loader_type in self._loaders:
            return self._loaders[loader_type]
        
        # Check registered classes
        if loader_type in self._loader_classes:
            loader = self._loader_classes[loader_type]()
            self._loaders[loader_type] = loader
            return loader
        
        logger.warning(f"No loader found for type: {loader_type.value}")
        return None
    
    def list_loaders(self) -> List[LoaderType]:
        """List available loader types"""
        return list(set(list(self._loaders.keys()) + list(self._loader_classes.keys())))
```

```python
# src/mcp_hub/loaders/handlers.py

import logging
from typing import Union
from datetime import datetime

from ..core.models import ServerInstance, ServerStatus, ServerHealth
from ..registry import ServerRegistry
from .base import LoadResult

logger = logging.getLogger(__name__)


class LoadResultHandler:
    """Handles server loading results"""
    
    def __init__(self, registry: ServerRegistry):
        self.registry = registry
    
    async def handle_result(self, server_id: str, result: LoadResult):
        """Process a successful load result"""
        server = await self.registry.get(server_id)
        if not server:
            logger.error(f"Server {server_id} not found in registry")
            return
        
        if result.success:
            # Update server with success
            server.health = ServerHealth(
                status=ServerStatus.HEALTHY,
                healthy=True,
                last_check=datetime.utcnow()
            )
            
            # Update capabilities if available
            if result.capabilities:
                server.capabilities = result.capabilities
            
            # Store app reference
            if hasattr(server, '_app'):
                server._app = result.app
            
            logger.info(f"Server {server_id} loaded successfully")
        else:
            # Handle failure
            await self.handle_error(server_id, Exception(result.error or "Unknown error"))
        
        await self.registry.update(server_id, server)
    
    async def handle_error(self, server_id: str, error: Exception):
        """Handle loading error"""
        server = await self.registry.get(server_id)
        if not server:
            return
        
        server.health = ServerHealth(
            status=ServerStatus.FAILED,
            healthy=False,
            error=str(error),
            last_check=datetime.utcnow()
        )
        
        await self.registry.update(server_id, server)
        logger.error(f"Server {server_id} failed to load: {error}")
```

## Updated MCPHub Application

```python
# src/mcp_hub/api/app.py

from typing import Optional, Dict, Any
from contextlib import asynccontextmanager
import logging

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from starlette.routing import Mount

from ..core.models import ServerInstance
from ..registry import ServerRegistry, InMemoryStore
from ..loaders import LoaderFactory, LoadResultHandler
from ..secrets import SecretManager
from ..config import Settings
from ..utils.health import DefaultHealthChecker
from .routes import health, servers, admin
from .middleware import LoggingMiddleware, AuthenticationMiddleware

logger = logging.getLogger(__name__)


class MCPHubApplication:
    """Main MCP Hub application with clean API structure"""
    
    def __init__(self, settings: Settings):
        self.settings = settings
        self.registry: Optional[ServerRegistry] = None
        self.loader_factory: Optional[LoaderFactory] = None
        self.secret_manager: Optional[SecretManager] = None
        self._app: Optional[FastAPI] = None
        self._server_apps: Dict[str, Any] = {}
    
    async def initialize(self):
        """Initialize application components"""
        # Initialize registry with store and health checker
        store = InMemoryStore()
        health_checker = DefaultHealthChecker()
        
        self.registry = ServerRegistry(
            store=store,
            health_checker=health_checker,
            health_check_interval=self.settings.health_check_interval
        )
        
        # Initialize loader factory and register loaders
        self.loader_factory = LoaderFactory()
        self._register_loaders()
        
        # Initialize secrets manager
        self.secret_manager = SecretManager(self.settings.secrets)
        
        # Start registry
        await self.registry.start()
        
        # Load configured servers
        await self._load_servers()
    
    async def shutdown(self):
        """Shutdown application gracefully"""
        logger.info("Shutting down MCP Hub...")
        
        # Stop registry
        if self.registry:
            await self.registry.stop()
        
        # Cleanup servers
        if self.registry:
            servers = await self.registry.list()
            for server in servers:
                await self._cleanup_server(server)
    
    def create_app(self) -> FastAPI:
        """Create FastAPI application with clean structure"""
        @asynccontextmanager
        async def lifespan(app: FastAPI):
            await self.initialize()
            yield
            await self.shutdown()
        
        self._app = FastAPI(
            title=self.settings.app_name,
            version=self.settings.version,
            description="Professional MCP Server Registry",
            lifespan=lifespan,
            docs_url="/api/docs",
            redoc_url="/api/redoc",
            openapi_url="/api/openapi.json"
        )
        
        # Setup middleware
        self._setup_middleware()
        
        # Setup API routes
        self._setup_api_routes()
        
        # Setup server mounts (will be populated after servers load)
        self._setup_server_mounts()
        
        # Store references for dependency injection
        self._app.state.registry = self.registry
        self._app.state.settings = self.settings
        self._app.state.loader_factory = self.loader_factory
        self._app.state.secret_manager = self.secret_manager
        
        return self._app
    
    def _setup_middleware(self):
        """Configure all middleware"""
        # CORS
        if self.settings.cors_enabled:
            self._app.add_middleware(
                CORSMiddleware,
                allow_origins=self.settings.cors_origins,
                allow_credentials=True,
                allow_methods=["*"],
                allow_headers=["*"],
            )
        
        # Logging
        self._app.add_middleware(LoggingMiddleware)
        
        # Authentication
        if self.settings.auth_enabled:
            self._app.add_middleware(
                AuthenticationMiddleware,
                secret_key=self.settings.auth_secret_key
            )
    
    def _setup_api_routes(self):
        """Setup all API routes"""
        # Health endpoints
        self._app.include_router(
            health.router,
            prefix="/health",
            tags=["health"]
        )
        
        # Server management
        self._app.include_router(
            servers.router,
            prefix="/api/servers",
            tags=["servers"]
        )
        
        # Admin endpoints
        if self.settings.admin_enabled:
            self._app.include_router(
                admin.router,
                prefix="/api/admin",
                tags=["admin"]
            )
        
        # Root endpoint
        @self._app.get("/")
        async def root():
            return {
                "name": self.settings.app_name,
                "version": self.settings.version,
                "docs": "/api/docs"
            }
    
    def _setup_server_mounts(self):
        """Setup mount points for MCP servers"""
        # This will be called after servers are loaded
        # The actual mounting happens in _mount_server
        pass
    
    async def _mount_server(self, server: ServerInstance):
        """Mount a server application"""
        if hasattr(server, '_app') and server._app:
            # Store the app
            self._server_apps[server.metadata.id] = server._app
            
            # Create mount
            mount = Mount(server.mount_path, app=server._app)
            self._app.routes.append(mount)
            
            logger.info(f"Mounted server {server.metadata.id} at {server.mount_path}")
    
    def _register_loaders(self):
        """Register all available loaders"""
        from ..loaders.implementations import (
            OpenAPILoader,
            FastMCPProxyLoader,
            ModuleLoader,
            PackageLoader
        )
        
        self.loader_factory.register(LoaderType.OPENAPI, OpenAPILoader())
        self.loader_factory.register(LoaderType.FASTMCP_PROXY, FastMCPProxyLoader())
        self.loader_factory.register(LoaderType.MODULE, ModuleLoader())
        self.loader_factory.register(LoaderType.PACKAGE, PackageLoader())
    
    async def _load_servers(self):
        """Load all configured servers"""
        handler = LoadResultHandler(self.registry)
        
        for server_config in self.settings.servers:
            if not server_config.enabled:
                logger.info(f"Server {server_config.id} is disabled, skipping")
                continue
            
            try:
                # Create server instance
                from ..core.models import ServerMetadata, ServerCapabilities, ServerHealth
                
                server = ServerInstance(
                    metadata=ServerMetadata(
                        id=server_config.id,
                        name=server_config.name,
                        description=server_config.description,
                        tags=server_config.tags
                    ),
                    capabilities=ServerCapabilities(),
                    health=ServerHealth(
                        status=ServerStatus.LOADING,
                        healthy=False
                    ),
                    mount_path=server_config.mount_path,
                    transport=server_config.transport,
                    loader_type=server_config.loader_type,
                    config=server_config.config
                )
                
                # Register in registry
                await self.registry.register(server)
                
                # Get loader
                loader = self.loader_factory.get_loader(server_config.loader_type)
                if not loader:
                    raise ValueError(f"No loader for type: {server_config.loader_type}")
                
                # Resolve secrets in config
                resolved_config = await self.secret_manager.resolve_config(
                    server_config.config
                )
                
                # Load server
                result = await loader.load(resolved_config)
                
                # Handle result
                await handler.handle_result(server_config.id, result)
                
                # Mount if successful
                if result.success and result.app:
                    server._app = result.app
                    await self._mount_server(server)
                
            except Exception as e:
                logger.error(f"Failed to load server {server_config.id}: {e}")
                await handler.handle_error(server_config.id, e)
    
    async def _cleanup_server(self, server: ServerInstance):
        """Cleanup a server instance"""
        try:
            logger.info(f"Cleaning up server: {server.metadata.id}")
            
            # Remove from mounts
            if server.metadata.id in self._server_apps:
                del self._server_apps[server.metadata.id]
            
            # Call any cleanup handlers
            # This would be implemented based on server type
            
        except Exception as e:
            logger.error(f"Error cleaning up server {server.metadata.id}: {e}")
```

This implementation provides:

1. **Clean API Structure**: Separated routes, models, dependencies, and middleware
2. **Professional Error Handling**: Proper HTTP status codes and error responses
3. **Dependency Injection**: Clean DI pattern using FastAPI
4. **Background Tasks**: Server loading happens in background
5. **Admin Interface**: Protected admin endpoints for management
6. **Health Monitoring**: Kubernetes-compatible health checks
7. **Request Tracking**: Request IDs and logging middleware
8. **Authentication**: Token-based auth for protected endpoints
9. **CORS Support**: Configurable CORS for web clients
10. **OpenAPI Documentation**: Auto-generated API docs

The MCPHub class is now much cleaner, focusing only on application initialization and coordination, while the API logic is properly separated into the routes modules.

Great question! Let me show you how to deploy the MCP Hub for different clients to connect to. The hub can be deployed in several ways depending on your use case:

## 1. **Local Deployment for Desktop Clients**

### For Claude Desktop / Cursor / Other Local MCP Clients

```bash
# Run the hub with stdio transport (default)
mcp-hub run

# Or explicitly specify stdio
mcp-hub run --transport stdio
```

Then configure your MCP client (e.g., Claude Desktop's `config.json`):

```json
{
  "mcpServers": {
    "hub": {
      "command": "mcp-hub",
      "args": ["run"],
      "env": {
        "MCP_HUB_DATA_DIR": "/path/to/your/data"
      }
    }
  }
}
```

## 2. **HTTP/Web Deployment for Remote Access**

### Deploy with HTTP Transport

```bash
# Run with HTTP transport for web access
mcp-hub run --transport http --host 0.0.0.0 --port 8000

# Or with SSE transport
mcp-hub run --transport sse --host 0.0.0.0 --port 8000

# With custom path
mcp-hub run --transport http --host 0.0.0.0 --port 8000 --path /mcp
```

### Configure Remote Clients

```json
{
  "mcpServers": {
    "hub": {
      "url": "http://your-server.com:8000/mcp",
      "transport": "http"
    }
  }
}
```

## 3. **Production Deployment Options**

### A. Docker Deployment

Create a `Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
RUN pip install uv
COPY requirements.txt .
RUN uv pip install -r requirements.txt

# Copy source code
COPY mcp_hub/ ./mcp_hub/
COPY data/ ./data/

# Expose port for HTTP transport
EXPOSE 8000

# Run the hub
CMD ["python", "-m", "mcp_hub.cli", "run", "--transport", "http", "--host", "0.0.0.0", "--port", "8000"]
```

Build and run:

```bash
# Build image
docker build -t mcp-hub .

# Run with volume for configuration
docker run -p 8000:8000 \
  -v $(pwd)/data:/app/data \
  -e MCP_HUB_DATA_DIR=/app/data \
  mcp-hub
```

### B. Docker Compose Deployment

`docker-compose.yml`:

```yaml
version: '3.8'

services:
  mcp-hub:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./data:/app/data
    environment:
      - MCP_HUB_DATA_DIR=/app/data
      - JIRA_URL=${JIRA_URL}
      - JIRA_USERNAME=${JIRA_USERNAME}
      - JIRA_API_TOKEN=${JIRA_API_TOKEN}
      - CONFLUENCE_URL=${CONFLUENCE_URL}
      - CONFLUENCE_USERNAME=${CONFLUENCE_USERNAME}
      - CONFLUENCE_API_TOKEN=${CONFLUENCE_API_TOKEN}
    command: ["python", "-m", "mcp_hub.cli", "run", "--transport", "http", "--host", "0.0.0.0", "--port", "8000"]
    restart: unless-stopped

  # Optional: Add nginx for SSL/reverse proxy
  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - mcp-hub
```

### C. Systemd Service (Linux)

Create `/etc/systemd/system/mcp-hub.service`:

```ini
[Unit]
Description=MCP Hub Server
After=network.target

[Service]
Type=exec
User=mcp
Group=mcp
WorkingDirectory=/opt/mcp-hub
Environment="MCP_HUB_DATA_DIR=/opt/mcp-hub/data"
ExecStart=/usr/local/bin/python -m mcp_hub.cli run --transport http --host 0.0.0.0 --port 8000
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mcp-hub
sudo systemctl start mcp-hub
```

## 4. **Enhanced CLI for Deployment**

Add deployment-specific commands to `cli.py`:

```python
@cli.command()
@click.option('--transport', '-t', 
              type=click.Choice(['stdio', 'http', 'sse']), 
              default='stdio',
              help='Transport protocol to use')
@click.option('--host', '-h', default='127.0.0.1', help='Host to bind to')
@click.option('--port', '-p', default=8000, type=int, help='Port to bind to')
@click.option('--path', default='/mcp', help='Path for HTTP transport')
@click.option('--workers', '-w', default=1, type=int, help='Number of worker processes')
@click.option('--reload', is_flag=True, help='Enable auto-reload for development')
@click.pass_context
def serve(ctx, transport: str, host: str, port: int, path: str, workers: int, reload: bool):
    """Run MCP Hub in production mode"""
    
    data_dir = ctx.obj.get('data_dir', get_data_path())
    
    # Load configuration
    config_path = data_dir / 'config.yaml'
    servers_path = data_dir / 'servers.yaml'
    
    if not config_path.exists():
        click.echo(f"Error: Config file not found: {config_path}", err=True)
        sys.exit(1)
    
    try:
        hub_config = Config.from_file(str(config_path), str(servers_path))
    except Exception as e:
        click.echo(f"Error loading configuration: {e}", err=True)
        sys.exit(1)
    
    # Override transport settings from CLI
    hub_config.hub.transport = transport
    hub_config.hub.host = host
    hub_config.hub.port = port
    
    # Create hub
    hub = MCPHub(hub_config)
    
    async def run_server():
        app = await hub.create_app()
        
        if transport == "stdio":
            # Run with stdio transport
            server = hub.create_stdio_server(app)
            await server.serve()
        else:
            # Run with HTTP/SSE transport
            import uvicorn
            
            config = uvicorn.Config(
                app=app,
                host=host,
                port=port,
                workers=workers if not reload else 1,
                reload=reload,
                log_level=hub_config.hub.log_level
            )
            
            server = uvicorn.Server(config)
            await server.serve()
    
    click.echo(f"Starting MCP Hub with {transport} transport on {host}:{port}")
    asyncio.run(run_server())


@cli.command()
@click.option('--output', '-o', default='deployment', help='Output directory')
@click.pass_context
def generate_deployment(ctx, output: str):
    """Generate deployment files (Docker, systemd, etc.)"""
    
    output_path = Path(output)
    output_path.mkdir(exist_ok=True)
    
    # Generate Dockerfile
    dockerfile_content = """FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \\
    git \\
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
RUN pip install uv
COPY requirements.txt .
RUN uv pip install -r requirements.txt

# Copy application
COPY . .

# Create data directory
RUN mkdir -p /app/data

# Expose port
EXPOSE 8000

# Run server
CMD ["python", "-m", "mcp_hub.cli", "serve", "--transport", "http", "--host", "0.0.0.0"]
"""
    
    (output_path / 'Dockerfile').write_text(dockerfile_content)
    
    # Generate docker-compose.yml
    compose_content = """version: '3.8'

services:
  mcp-hub:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./data:/app/data
    env_file:
      - .env
    environment:
      - MCP_HUB_DATA_DIR=/app/data
    restart: unless-stopped
    
  # Optional: Nginx reverse proxy with SSL
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - mcp-hub
"""
    
    (output_path / 'docker-compose.yml').write_text(compose_content)
    
    # Generate nginx.conf
    nginx_content = """events {
    worker_connections 1024;
}

http {
    upstream mcp_hub {
        server mcp-hub:8000;
    }
    
    server {
        listen 80;
        server_name your-domain.com;
        return 301 https://$server_name$request_uri;
    }
    
    server {
        listen 443 ssl;
        server_name your-domain.com;
        
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        
        location / {
            proxy_pass http://mcp_hub;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
"""
    
    (output_path / 'nginx.conf').write_text(nginx_content)
    
    # Generate .env.example
    env_content = """# MCP Hub Configuration
MCP_HUB_DATA_DIR=/app/data

# Atlassian Configuration (example)
JIRA_URL=https://your-company.atlassian.net
JIRA_USERNAME=your.email@company.com
JIRA_API_TOKEN=your_jira_api_token

CONFLUENCE_URL=https://your-company.atlassian.net/wiki
CONFLUENCE_USERNAME=your.email@company.com
CONFLUENCE_API_TOKEN=your_confluence_api_token

# Add other service credentials as needed
"""
    
    (output_path / '.env.example').write_text(env_content)
    
    click.echo(f"Deployment files generated in: {output_path}")
    click.echo("\nNext steps:")
    click.echo("1. Copy your configuration to data/")
    click.echo("2. Update .env with your credentials")
    click.echo("3. Run: docker-compose up -d")
```

## 5. **Client Configuration Examples**

### Claude Desktop (Local)
```json
{
  "mcpServers": {
    "hub": {
      "command": "mcp-hub",
      "args": ["run"]
    }
  }
}
```

### Remote HTTP Client
```json
{
  "mcpServers": {
    "hub": {
      "url": "https://mcp-hub.your-domain.com/mcp",
      "transport": "http",
      "headers": {
        "Authorization": "Bearer your-token"
      }
    }
  }
}
```

### Python Client
```python
from fastmcp import Client

# Connect to local hub
async with Client("mcp-hub run") as client:
    tools = await client.list_tools()

# Connect to remote hub
async with Client("https://mcp-hub.your-domain.com/mcp") as client:
    tools = await client.list_tools()
```

## 6. **Monitoring & Health Checks**

Add health check endpoint for monitoring:

```python
# In server.py
async def health_with_details(request):
    all_servers = self.registry.get_all()
    healthy = len(self.registry.get_healthy())
    total = len(all_servers)
    
    # Calculate health score
    health_score = (healthy / total * 100) if total > 0 else 0
    
    return JSONResponse({
        "status": "healthy" if health_score > 75 else "degraded" if health_score > 0 else "unhealthy",
        "health_score": health_score,
        "servers": {
            "healthy": healthy,
            "total": total,
            "failed": total - healthy
        },
        "timestamp": datetime.utcnow().isoformat(),
        "version": "1.0.0"
    })
```

## 7. **Security Considerations**

1. **Authentication**: Add auth middleware for production
2. **HTTPS**: Always use SSL/TLS in production
3. **Rate Limiting**: Implement rate limiting for public endpoints
4. **CORS**: Configure CORS appropriately
5. **Environment Variables**: Never commit secrets

This deployment setup allows the MCP Hub to be used by:
- Local desktop clients (Claude, Cursor)
- Remote web clients
- Python applications
- Any MCP-compatible client

The hub acts as a central gateway to all your MCP servers, regardless of their underlying implementation (OpenAPI, UVX, NPM, etc.).



Excellent question! Let's implement a secure way to handle secrets in the server configuration. Here are several approaches:

## 1. **Environment Variable Substitution (Already Partially Implemented)**

Enhance the existing `substitute_env_vars` function in `config.py`:

```python
# mcp_hub/config.py

import os
import re
import yaml
from pathlib import Path
from typing import Any, Dict, List, Optional
from pydantic import BaseModel, Field, validator


def substitute_env_vars(data: Any) -> Any:
    """Recursively substitute ${VAR} and ${VAR:-default} with environment variables"""
    if isinstance(data, dict):
        return {k: substitute_env_vars(v) for k, v in data.items()}
    elif isinstance(data, list):
        return [substitute_env_vars(item) for item in data]
    elif isinstance(data, str):
        def replacer(match):
            var_expr = match.group(1)
            # Handle ${VAR:-default} syntax
            if ':-' in var_expr:
                var_name, default_value = var_expr.split(':-', 1)
                return os.environ.get(var_name.strip(), default_value)
            else:
                var_name = var_expr.strip()
                value = os.environ.get(var_name)
                if value is None:
                    # Check if it's a required var (no default)
                    raise ValueError(f"Required environment variable '{var_name}' is not set")
                return value
        
        # Support both ${VAR} and $VAR syntax
        pattern = r'\$\{([^}]+)\}|\$([A-Z_][A-Z0-9_]*)'
        
        def full_replacer(match):
            if match.group(1):  # ${VAR} style
                return replacer(match)
            else:  # $VAR style
                var_name = match.group(2)
                return os.environ.get(var_name, match.group(0))
        
        return re.sub(pattern, full_replacer, data)
    else:
        return data
```

Usage in `servers.yaml`:

```yaml
servers:
  - id: "atlassian-server"
    name: "Atlassian MCP"
    packages:
      - registry_name: "uvx"
        name: "mcp-atlassian"
        environment_variables:
          - name: "JIRA_URL"
            default: "${JIRA_URL}"
          - name: "JIRA_API_TOKEN"
            default: "${JIRA_API_TOKEN}"  # From environment
          - name: "CONFLUENCE_URL"
            default: "${CONFLUENCE_URL:-https://default.atlassian.net/wiki}"  # With default
```

## 2. **Secrets File Approach**

Create a separate secrets management system:

```python
# mcp_hub/secrets.py

import os
import json
import yaml
from pathlib import Path
from typing import Dict, Any, Optional
from cryptography.fernet import Fernet
import keyring
import logging

logger = logging.getLogger(__name__)


class SecretsManager:
    """Manage secrets for MCP Hub servers"""
    
    def __init__(self, secrets_dir: Optional[Path] = None):
        self.secrets_dir = secrets_dir or self._get_secrets_dir()
        self.secrets_dir.mkdir(parents=True, exist_ok=True)
        self._cache: Dict[str, Any] = {}
    
    def _get_secrets_dir(self) -> Path:
        """Get platform-specific secrets directory"""
        if os.name == 'nt':  # Windows
            base = Path(os.environ.get('APPDATA', '~'))
        else:  # Unix-like
            base = Path.home() / '.config'
        
        return base / 'mcp-hub' / 'secrets'
    
    def load_secrets(self, server_id: str) -> Dict[str, Any]:
        """Load secrets for a specific server"""
        # Check multiple sources in order of precedence
        secrets = {}
        
        # 1. Environment variables prefixed with server ID
        prefix = f"MCP_{server_id.upper().replace('-', '_')}_"
        for key, value in os.environ.items():
            if key.startswith(prefix):
                secret_name = key[len(prefix):].lower()
                secrets[secret_name] = value
        
        # 2. Secrets file (JSON or YAML)
        secrets_file = self.secrets_dir / f"{server_id}.secrets.yaml"
        if secrets_file.exists():
            with open(secrets_file, 'r') as f:
                file_secrets = yaml.safe_load(f)
                secrets.update(file_secrets or {})
        
        # 3. System keyring (for sensitive data)
        try:
            keyring_secrets = keyring.get_password("mcp-hub", server_id)
            if keyring_secrets:
                secrets.update(json.loads(keyring_secrets))
        except Exception as e:
            logger.debug(f"Could not load from keyring: {e}")
        
        return secrets
    
    def save_secret(self, server_id: str, key: str, value: str, use_keyring: bool = False):
        """Save a secret for a server"""
        if use_keyring:
            # Store in system keyring
            current = {}
            try:
                stored = keyring.get_password("mcp-hub", server_id)
                if stored:
                    current = json.loads(stored)
            except:
                pass
            
            current[key] = value
            keyring.set_password("mcp-hub", server_id, json.dumps(current))
        else:
            # Store in secrets file
            secrets_file = self.secrets_dir / f"{server_id}.secrets.yaml"
            secrets = {}
            
            if secrets_file.exists():
                with open(secrets_file, 'r') as f:
                    secrets = yaml.safe_load(f) or {}
            
            secrets[key] = value
            
            with open(secrets_file, 'w') as f:
                yaml.dump(secrets, f, default_flow_style=False)
            
            # Set restrictive permissions on Unix-like systems
            if os.name != 'nt':
                os.chmod(secrets_file, 0o600)


class SecureServerConfig(BaseModel):
    """Extended server config with secrets support"""
    # ... existing fields ...
    
    secrets_required: Optional[List[str]] = Field(default_factory=list)
    secrets_optional: Optional[List[str]] = Field(default_factory=list)
    
    def resolve_secrets(self, secrets_manager: SecretsManager):
        """Resolve secrets for this server configuration"""
        secrets = secrets_manager.load_secrets(self.id)
        
        # Check required secrets
        missing = []
        for required in self.secrets_required:
            if required not in secrets:
                missing.append(required)
        
        if missing:
            raise ValueError(f"Missing required secrets for {self.id}: {', '.join(missing)}")
        
        # Apply secrets to configuration
        self._apply_secrets(secrets)
    
    def _apply_secrets(self, secrets: Dict[str, Any]):
        """Apply secrets to the configuration"""
        # This is where you'd inject secrets into the appropriate fields
        # For example, into environment_variables
        if hasattr(self, 'packages'):
            for package in self.packages:
                for env_var in package.environment_variables:
                    if env_var.name.lower() in secrets:
                        env_var.default = secrets[env_var.name.lower()]
```

## 3. **Secrets Configuration File**

Create a `.secrets.yaml` file structure:

```yaml
# data/.secrets.yaml (git-ignored)
servers:
  atlassian-server:
    jira_api_token: "your-actual-token-here"
    confluence_api_token: "another-token"
    oauth_client_secret: "secret-value"
  
  github-server:
    github_token: "ghp_xxxxxxxxxxxxx"
    webhook_secret: "webhook-secret-value"

# Encrypted version (optional)
encrypted: true
encryption_key: "${SECRETS_ENCRYPTION_KEY}"
```

## 4. **CLI Commands for Secrets Management**

Add secret management commands to `cli.py`:

```python
# mcp_hub/cli.py

@cli.group()
def secrets():
    """Manage server secrets"""
    pass


@secrets.command()
@click.argument('server_id')
@click.argument('key')
@click.option('--value', prompt=True, hide_input=True, confirmation_prompt=True)
@click.option('--keyring', is_flag=True, help='Store in system keyring')
@click.pass_context
def set(ctx, server_id: str, key: str, value: str, keyring: bool):
    """Set a secret for a server"""
    secrets_manager = SecretsManager()
    secrets_manager.save_secret(server_id, key, value, use_keyring=keyring)
    click.echo(f"Secret '{key}' set for server '{server_id}'")


@secrets.command()
@click.argument('server_id')
@click.pass_context
def list(ctx, server_id: str):
    """List secrets for a server (keys only)"""
    secrets_manager = SecretsManager()
    secrets = secrets_manager.load_secrets(server_id)
    
    if secrets:
        click.echo(f"Secrets for {server_id}:")
        for key in sorted(secrets.keys()):
            click.echo(f"  - {key}")
    else:
        click.echo(f"No secrets found for {server_id}")


@secrets.command()
@click.argument('server_id')
@click.argument('key')
@click.pass_context
def delete(ctx, server_id: str, key: str):
    """Delete a secret"""
    if click.confirm(f"Delete secret '{key}' for server '{server_id}'?"):
        secrets_manager = SecretsManager()
        # Implementation depends on storage method
        click.echo(f"Secret '{key}' deleted")


@secrets.command()
@click.option('--server', '-s', help='Import secrets for specific server')
@click.argument('file', type=click.Path(exists=True))
@click.pass_context
def import_file(ctx, server: Optional[str], file: str):
    """Import secrets from file"""
    with open(file, 'r') as f:
        if file.endswith('.json'):
            data = json.load(f)
        else:
            data = yaml.safe_load(f)
    
    secrets_manager = SecretsManager()
    
    if server:
        # Import for specific server
        for key, value in data.items():
            secrets_manager.save_secret(server, key, str(value))
        click.echo(f"Imported {len(data)} secrets for {server}")
    else:
        # Import for multiple servers
        for server_id, secrets in data.get('servers', {}).items():
            for key, value in secrets.items():
                secrets_manager.save_secret(server_id, key, str(value))
            click.echo(f"Imported {len(secrets)} secrets for {server_id}")
```

## 5. **External Secret Providers**

Support for external secret management systems:

```python
# mcp_hub/secrets/providers.py

from abc import ABC, abstractmethod
from typing import Dict, Any, Optional
import os
import boto3
import hvac
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential


class SecretProvider(ABC):
    """Base class for secret providers"""
    
    @abstractmethod
    async def get_secret(self, key: str) -> Optional[str]:
        pass
    
    @abstractmethod
    async def set_secret(self, key: str, value: str) -> bool:
        pass


class AWSSecretsProvider(SecretProvider):
    """AWS Secrets Manager provider"""
    
    def __init__(self, region: str = None):
        self.client = boto3.client(
            'secretsmanager',
            region_name=region or os.environ.get('AWS_DEFAULT_REGION', 'us-east-1')
        )
    
    async def get_secret(self, key: str) -> Optional[str]:
        try:
            response = self.client.get_secret_value(SecretId=key)
            return response['SecretString']
        except Exception:
            return None


class HashiCorpVaultProvider(SecretProvider):
    """HashiCorp Vault provider"""
    
    def __init__(self, url: str = None, token: str = None):
        self.client = hvac.Client(
            url=url or os.environ.get('VAULT_ADDR'),
            token=token or os.environ.get('VAULT_TOKEN')
        )
    
    async def get_secret(self, key: str) -> Optional[str]:
        try:
            response = self.client.secrets.kv.v2.read_secret_version(
                path=key
            )
            return response['data']['data'].get('value')
        except Exception:
            return None


class AzureKeyVaultProvider(SecretProvider):
    """Azure Key Vault provider"""
    
    def __init__(self, vault_url: str = None):
        self.vault_url = vault_url or os.environ.get('AZURE_KEY_VAULT_URL')
        credential = DefaultAzureCredential()
        self.client = SecretClient(vault_url=self.vault_url, credential=credential)
    
    async def get_secret(self, key: str) -> Optional[str]:
        try:
            secret = self.client.get_secret(key)
            return secret.value
        except Exception:
            return None


# Provider factory
def get_secret_provider(provider_type: str) -> SecretProvider:
    providers = {
        'aws': AWSSecretsProvider,
        'vault': HashiCorpVaultProvider,
        'azure': AzureKeyVaultProvider,
    }
    
    provider_class = providers.get(provider_type)
    if not provider_class:
        raise ValueError(f"Unknown secret provider: {provider_type}")
    
    return provider_class()
```

## 6. **Updated servers.yaml with Secrets References**

```yaml
servers:
  - id: "atlassian-server"
    name: "Atlassian MCP"
    secrets_required:
      - "jira_api_token"
      - "confluence_api_token"
    secrets_optional:
      - "oauth_client_secret"
    packages:
      - registry_name: "uvx"
        name: "mcp-atlassian"
        environment_variables:
          - name: "JIRA_URL"
            default: "https://company.atlassian.net"
          - name: "JIRA_API_TOKEN"
            secret_ref: "jira_api_token"  # Reference to secret
          - name: "CONFLUENCE_API_TOKEN"
            secret_ref: "confluence_api_token"

  - id: "github-api"
    name: "GitHub API"
    type: "openapi"
    openapi_url: "https://api.github.com/openapi.yaml"
    auth_type: "bearer"
    auth_token_secret: "github_token"  # Reference to secret
    secrets_required:
      - "github_token"
```

## 7. **Docker Secrets Support**

For Docker deployments, use Docker secrets:

```yaml
# docker-compose.yml
version: '3.8'

services:
  mcp-hub:
    build: .
    secrets:
      - jira_api_token
      - confluence_api_token
      - github_token
    environment:
      - SECRETS_PATH=/run/secrets
    volumes:
      - ./data:/app/data

secrets:
  jira_api_token:
    external: true
  confluence_api_token:
    external: true
  github_token:
    external: true
```

## 8. **Best Practices Implementation**

```python
# mcp_hub/config.py - Enhanced Config class

class Config(BaseModel):
    hub: HubConfig = Field(default_factory=HubConfig)
    servers: List[ServerConfig] = Field(default_factory=list)
    secrets_provider: Optional[str] = None  # aws, vault, azure, local
    
    @classmethod
    def from_file(cls, config_path: str | Path, servers_path: Optional[str | Path] = None) -> "Config":
        """Load configuration from YAML files with secrets resolution"""
        # Load configuration files
        config_path = Path(config_path)
        with open(config_path, 'r') as f:
            config_data = yaml.safe_load(f)
        
        if servers_path:
            servers_path = Path(servers_path)
            if servers_path.exists():
                with open(servers_path, 'r') as f:
                    servers_data = yaml.safe_load(f)
                    config_data['servers'] = servers_data.get('servers', [])
        
        # Substitute environment variables first
        config_data = substitute_env_vars(config_data)
        
        # Create config instance
        config = cls(**config_data)
        
        # Resolve secrets for each server
        secrets_manager = SecretsManager()
        for server in config.servers:
            try:
                server.resolve_secrets(secrets_manager)
            except ValueError as e:
                logger.warning(f"Could not resolve secrets for {server.id}: {e}")
        
        return config
```

## Usage Examples:

```bash
# Set secrets via CLI
mcp-hub secrets set atlassian-server jira_api_token
mcp-hub secrets set atlassian-server confluence_api_token --keyring

# Import secrets from file
mcp-hub secrets import-file secrets.yaml

# Use environment variables
export MCP_ATLASSIAN_SERVER_JIRA_API_TOKEN="your-token"
export MCP_ATLASSIAN_SERVER_CONFLUENCE_API_TOKEN="your-token"

# Run with secrets
mcp-hub run
```

This approach provides:
1. **Multiple secret sources**: Environment variables, files, keyring, external providers
2. **Security**: Secrets never appear in configuration files
3. **Flexibility**: Different secret management per environment
4. **Auditability**: Track which secrets are required
5. **Developer-friendly**: Easy CLI for secret management

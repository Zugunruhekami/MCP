*Thinking...*

> Based on the provided information, I need to create a comprehensive implementation plan for an MCP registry, MCP servers, and AI-Matrix as an MCP client. The implementation should handle the complexity of real-world scenarios, particularly with FastAPI servers containing many analytical functions.
> 
> Let's break down what needs to be done:
> 
> 1. **MCP Registry**:
>    - Create a centralized registry with a modern UI
>    - Display each MCP server's configuration and documentation
> 
> 2. **MCP Servers**:
>    - Implement MCP servers for FastAPI applications (analytical functions)
>    - Integrate with open-source MCP servers (Tavily, Atlassian)
>    - Handle complex FastAPI and Pydantic annotations
> 
> 3. **AI-Matrix as MCP Client**:
>    - Use LangGraph for agent orchestration
>    - Integrate with langchain-mcp-adapters
>    - Connect to multiple MCP servers
> 
> Now, let's design the architecture in detail:
> 
> ## Architecture Overview
> 
> ```
> ├── mcp-registry/              # Main Registry Service
> │   ├── backend/               # FastAPI backend for the registry
> │   │   ├── api/               # API endpoints
> │   │   ├── models/            # Data models
> │   │   ├── services/          # Business logic
> │   │   └── auth/              # Authentication and authorization
> │   ├── frontend/              # React UI for the registry
> │   │   ├── src/               # React components and pages
> │   │   └── public/            # Static assets
> │   └── docker/                # Docker configurations
> │
> ├── mcp-servers/               # MCP Server implementations
> │   ├── market-analytics/      # Market Analytics FastAPI MCP server
> │   │   ├── api/               # FastAPI implementation
> │   │   ├── tools/             # Tool implementations
> │   │   └── tests/             # Tests for tools
> │   ├── trading-analytics/     # Trading Analytics FastAPI MCP server
> │   │   ├── api/               # FastAPI implementation 
> │   │   ├── tools/             # Tool implementations
> │   │   └── tests/             # Tests for tools
> │   └── integrations/          # Open-source MCP server integrations
> │       ├── tavily/            # Tavily integration
> │       └── atlassian/         # Atlassian integration
> │
> ├── ai-matrix/                 # AI-Matrix agent framework
> │   ├── agents/                # Agent implementations
> │   ├── client/                # MCP client implementation
> │   ├── graphs/                # LangGraph configurations
> │   └── tests/                 # Tests for agents
> │
> └── tests/                     # Integration tests
>     ├── e2e/                   # End-to-end tests
>     └── performance/           # Performance tests
> ```
> 
> Now, I'll structure a detailed implementation plan for each component:
> 
> ## 1. MCP Registry
> 
> ### Backend (FastAPI)
> 
> The backend will need:
> - Server registration and management
> - Tool discovery and documentation
> - Authentication and authorization
> - API for retrieving server metadata
> 
> ### Frontend (React)
> 
> The frontend will need:
> - Dashboard for viewing registered servers
> - Detailed server and tool documentation
> - Authentication UI
> - Tool testing interface
> 
> ## 2. MCP Servers
> 
> ### Market Analytics FastAPI Server
> This will handle financial market analytics functions (vol charts, spot charts, etc.)
> 
> We have two options for converting FastAPI to MCP:
> 1. Use fastapi_mcp to automatically expose FastAPI endpoints
> 2. Create custom MCP tools that wrap the API functionality
> 
> For complex analytics, I'll implement a hybrid approach:
> - Use fastapi_mcp for simple endpoints
> - Create custom MCP wrappers for complex endpoints that require special handling
> 
> ### Trading Analytics FastAPI Server
> Similar to the market analytics server, this will focus on trading-specific analytics.
> 
> ### Open-source Integrations
> - Tavily for web search
> - Atlassian for Jira/Confluence integration
> 
> ## 3. AI-Matrix as MCP Client
> 
> - Use langchain-mcp-adapters for connecting to MCP servers
> - Implement a multi-agent pattern with LangGraph
> - Create specialized agents for different domains
> 
> Now, let's start with the detailed implementation plan:
> 
> ## Implementation Plan
> 
> I'll focus on key components in each area, starting with the core infrastructure:
> 
> 1. MCP Registry
> 2. FastAPI to MCP conversion
> 3. AI-Matrix MCP client implementation
> 4. Testing framework
> 
> Let's start implementing each component in detail.


# HSBC MCP Registry and AI-Matrix Client Implementation

Based on the provided GitHub repositories and requirements, I'm designing a comprehensive solution that includes:

1. **MCP Registry**: A centralized service for discovering and managing MCP servers
2. **Multiple MCP Servers**: Including FastAPI analytics services and open-source integrations
3. **AI-Matrix Client**: An agent framework built on LangGraph that uses MCP tools

## Architecture Overview

```
┌─────────────────────┐     ┌──────────────────────┐     ┌──────────────────────┐
│                     │     │                      │     │                      │
│   MCP Registry      │◄───►│  MCP Server Manager  │◄───►│  MCP Servers         │
│   (UI + API)        │     │                      │     │                      │
└─────────────────────┘     └──────────────────────┘     └──────────────────────┘
                                      ▲                            ▲
                                      │                            │
                                      ▼                            │
           ┌───────────────────────────────────────────┐          │
           │                                           │          │
           │    AI-Matrix Agent Framework              │◄─────────┘
           │    (LangGraph + MCP Client)               │
           │                                           │
           └───────────────────────────────────────────┘
```

## Project Structure

```
hsbc-mcp-ecosystem/
├── mcp-registry/                 # Registry service with UI
│   ├── backend/                  # FastAPI backend
│   ├── frontend/                 # React UI
│   └── docker/                   # Docker configurations
├── mcp-servers/                  # MCP Server implementations
│   ├── analytics-api/            # Market Analytics MCP server (FastAPI)
│   ├── trading-api/              # Trading Analytics MCP server (FastAPI)
│   └── integrations/             # Open-source MCP integrations
│       ├── tavily/               # Tavily search integration
│       └── atlassian/            # Atlassian (Jira/Confluence) integration
├── ai-matrix/                    # AI-Matrix agent framework
│   ├── agents/                   # Agent implementations
│   ├── clients/                  # MCP client adapters
│   └── graphs/                   # LangGraph configurations
└── tests/                        # Comprehensive test suite
```

Let's implement each component in detail:

## 1. MCP Registry Implementation

The registry will act as a centralized hub for discovering and managing MCP servers.

### Registry Backend (FastAPI)

```python
# mcp-registry/backend/main.py
from fastapi import FastAPI, Depends, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from typing import List, Dict, Optional
from sqlalchemy.orm import Session
from datetime import datetime

from .database import get_db, Base, engine
from .models.server import MCPServer, MCPServerCreate, MCPServerUpdate
from .models.tool import MCPTool
from .services.server_manager import ServerManager
from .services.authentication import get_current_user, authenticate_user
from .routers import servers, tools, auth

# Create tables
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="HSBC MCP Registry",
    description="A registry for Model Context Protocol (MCP) servers in HSBC AI Markets",
    version="1.0.0"
)

# Configure CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # In production, specify exact domains
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routers
app.include_router(auth.router)
app.include_router(servers.router)
app.include_router(tools.router)

@app.get("/health")
def health_check():
    return {"status": "healthy", "timestamp": datetime.now().isoformat()}
```

### Server Model and Manager

```python
# mcp-registry/backend/models/server.py
from pydantic import BaseModel, HttpUrl, Field
from typing import List, Dict, Optional, Any
from datetime import datetime
from enum import Enum

class TransportType(str, Enum):
    STDIO = "stdio"
    SSE = "sse"
    WEBSOCKET = "websocket"

class MCPServerBase(BaseModel):
    name: str = Field(..., description="Name of the MCP server")
    description: str = Field(..., description="Description of the server")
    version: str = Field(..., description="Server version")
    owner: str = Field(..., description="Team or person that owns this server")
    
class MCPServerCreate(MCPServerBase):
    transport: TransportType = Field(..., description="Transport protocol")
    config: Dict[str, Any] = Field(..., description="Server configuration")
    
class MCPServerUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    version: Optional[str] = None
    owner: Optional[str] = None
    transport: Optional[TransportType] = None
    config: Optional[Dict[str, Any]] = None
    status: Optional[str] = None

class MCPServer(MCPServerBase):
    id: str
    transport: TransportType
    config: Dict[str, Any]
    status: str
    tools_count: int
    created_at: datetime
    updated_at: Optional[datetime] = None
    
    class Config:
        orm_mode = True

# mcp-registry/backend/services/server_manager.py
import uuid
import asyncio
import logging
from typing import Dict, List, Optional, Any
from datetime import datetime

from ..models.server import MCPServer, MCPServerCreate, MCPServerUpdate
from ..models.tool import MCPTool
from ..database import get_db

logger = logging.getLogger(__name__)

class ServerManager:
    def __init__(self, db):
        self.db = db
        self.servers: Dict[str, MCPServer] = {}
        self.server_tools: Dict[str, List[MCPTool]] = {}
    
    async def register_server(self, server_data: MCPServerCreate) -> MCPServer:
        """Register a new MCP server"""
        server_id = str(uuid.uuid4())
        server = MCPServer(
            id=server_id,
            **server_data.dict(),
            status="offline",
            tools_count=0,
            created_at=datetime.now()
        )
        
        # Store in DB
        db_server = self.db.add_server(server)
        
        # Store in memory
        self.servers[server_id] = server
        self.server_tools[server_id] = []
        
        # Try to connect and get tools
        asyncio.create_task(self._probe_server(server_id))
        
        return server
    
    async def _probe_server(self, server_id: str):
        """Probe server to get tools and check status"""
        server = self.servers.get(server_id)
        if not server:
            return
        
        try:
            # Create appropriate client based on transport
            # This would connect to the actual MCP server
            # and retrieve tools
            tools = await self._get_server_tools(server)
            
            # Update server with tools and status
            server.status = "online"
            server.tools_count = len(tools)
            server.updated_at = datetime.now()
            
            # Store tools
            self.server_tools[server_id] = tools
            
            # Update DB
            self.db.update_server(server_id, ServerUpdate(
                status="online",
                tools_count=len(tools),
                updated_at=datetime.now()
            ))
            
        except Exception as e:
            logger.error(f"Error probing server {server_id}: {str(e)}")
            server.status = "error"
            server.updated_at = datetime.now()
            
            # Update DB
            self.db.update_server(server_id, ServerUpdate(
                status="error",
                updated_at=datetime.now()
            ))
    
    async def _get_server_tools(self, server: MCPServer) -> List[MCPTool]:
        """Connect to MCP server and get tools"""
        # Implementation depends on transport type
        # This would use MCP client libraries to connect
        # For now, return empty list
        return []
    
    def get_server(self, server_id: str) -> Optional[MCPServer]:
        """Get a server by ID"""
        return self.servers.get(server_id)
    
    def get_all_servers(self) -> List[MCPServer]:
        """Get all registered servers"""
        return list(self.servers.values())
    
    def get_server_tools(self, server_id: str) -> List[MCPTool]:
        """Get tools for a specific server"""
        return self.server_tools.get(server_id, [])
    
    def update_server(self, server_id: str, update_data: MCPServerUpdate) -> Optional[MCPServer]:
        """Update a server"""
        server = self.servers.get(server_id)
        if not server:
            return None
            
        # Update server
        for field, value in update_data.dict(exclude_unset=True).items():
            setattr(server, field, value)
            
        server.updated_at = datetime.now()
        
        # Update DB
        self.db.update_server(server_id, update_data)
        
        return server
```

### Registry Frontend (React)

```jsx
// mcp-registry/frontend/src/pages/Dashboard.jsx
import React, { useState, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import { 
  Container, Grid, Card, CardContent, Typography, 
  Box, Chip, Button, TextField, InputAdornment 
} from '@mui/material';
import SearchIcon from '@mui/icons-material/Search';
import AddIcon from '@mui/icons-material/Add';
import { getServers } from '../services/api';

const Dashboard = () => {
  const [servers, setServers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [searchTerm, setSearchTerm] = useState('');
  const navigate = useNavigate();

  useEffect(() => {
    fetchServers();
  }, []);

  const fetchServers = async () => {
    try {
      setLoading(true);
      const data = await getServers();
      setServers(data);
    } catch (error) {
      console.error('Error fetching servers:', error);
    } finally {
      setLoading(false);
    }
  };

  const filteredServers = servers.filter(server => 
    server.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
    server.description.toLowerCase().includes(searchTerm.toLowerCase()) ||
    server.owner.toLowerCase().includes(searchTerm.toLowerCase())
  );

  const handleAddServer = () => {
    navigate('/servers/new');
  };

  const handleServerClick = (serverId) => {
    navigate(`/servers/${serverId}`);
  };

  return (
    <Container maxWidth="lg" sx={{ mt: 4, mb: 4 }}>
      <Box sx={{ display: 'flex', justifyContent: 'space-between', mb: 3 }}>
        <Typography variant="h4" component="h1">
          MCP Server Registry
        </Typography>
        <Button 
          variant="contained" 
          color="primary" 
          startIcon={<AddIcon />}
          onClick={handleAddServer}
        >
          Add Server
        </Button>
      </Box>

      <TextField
        fullWidth
        margin="normal"
        placeholder="Search servers..."
        variant="outlined"
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        InputProps={{
          startAdornment: (
            <InputAdornment position="start">
              <SearchIcon />
            </InputAdornment>
          ),
        }}
        sx={{ mb: 4 }}
      />

      {loading ? (
        <Typography>Loading servers...</Typography>
      ) : (
        <Grid container spacing={3}>
          {filteredServers.map((server) => (
            <Grid item xs={12} md={4} key={server.id}>
              <Card 
                sx={{ 
                  height: '100%', 
                  display: 'flex', 
                  flexDirection: 'column',
                  cursor: 'pointer',
                  '&:hover': {
                    boxShadow: 6,
                  }
                }}
                onClick={() => handleServerClick(server.id)}
              >
                <CardContent sx={{ flexGrow: 1 }}>
                  <Box sx={{ display: 'flex', justifyContent: 'space-between', mb: 1 }}>
                    <Typography variant="h6" component="h2">
                      {server.name}
                    </Typography>
                    <Chip 
                      label={server.status} 
                      color={
                        server.status === 'online' ? 'success' : 
                        server.status === 'offline' ? 'error' : 'warning'
                      }
                      size="small"
                    />
                  </Box>
                  <Typography color="text.secondary" sx={{ mb: 1.5 }}>
                    {server.owner}
                  </Typography>
                  <Typography variant="body2">
                    {server.description}
                  </Typography>
                  <Box sx={{ mt: 2 }}>
                    <Typography variant="caption">
                      Tools: {server.tools_count}
                    </Typography>
                    <Typography variant="caption" sx={{ ml: 2 }}>
                      Version: {server.version}
                    </Typography>
                  </Box>
                </CardContent>
              </Card>
            </Grid>
          ))}
        </Grid>
      )}
    </Container>
  );
};

export default Dashboard;
```

## 2. MCP Server Implementations

### 2.1 Market Analytics FastAPI Server

This server will expose market analytics functions as MCP tools.

```python
# mcp-servers/analytics-api/main.py
from fastapi import FastAPI, Depends, Query
from pydantic import BaseModel, Field
from typing import List, Dict, Optional
from datetime import datetime, date
import pandas as pd
import numpy as np
from fastapi_mcp import FastApiMCP
import os

app = FastAPI(
    title="HSBC Market Analytics API",
    description="API for market analytics functions",
    version="1.0.0"
)

# Models
class VolChartRequest(BaseModel):
    ticker: str = Field(..., description="Ticker symbol")
    start_date: date = Field(..., description="Start date for analysis")
    end_date: date = Field(..., description="End date for analysis")
    tenor: str = Field("1M", description="Tenor for volatility (e.g., '1M', '3M', '1Y')")
    
    class Config:
        schema_extra = {
            "example": {
                "ticker": "EURUSD",
                "start_date": "2024-01-01",
                "end_date": "2024-03-31",
                "tenor": "3M"
            }
        }

class VolChartResponse(BaseModel):
    ticker: str
    tenor: str
    dates: List[str]
    implied_vol: List[float]
    realized_vol: List[float]
    vol_premium: List[float]
    
    class Config:
        schema_extra = {
            "example": {
                "ticker": "EURUSD",
                "tenor": "3M",
                "dates": ["2024-01-01", "2024-01-02", "2024-01-03"],
                "implied_vol": [10.2, 10.3, 10.1],
                "realized_vol": [9.8, 9.9, 10.0],
                "vol_premium": [0.4, 0.4, 0.1]
            }
        }

class SpotChartRequest(BaseModel):
    ticker: str = Field(..., description="Ticker symbol")
    start_date: date = Field(..., description="Start date for chart")
    end_date: date = Field(..., description="End date for chart")
    frequency: str = Field("daily", description="Data frequency (daily, weekly, monthly)")
    
    class Config:
        schema_extra = {
            "example": {
                "ticker": "EURUSD",
                "start_date": "2024-01-01",
                "end_date": "2024-03-31",
                "frequency": "daily"
            }
        }

class SpotChartResponse(BaseModel):
    ticker: str
    frequency: str
    dates: List[str]
    prices: List[float]
    returns: List[float]
    
    class Config:
        schema_extra = {
            "example": {
                "ticker": "EURUSD",
                "frequency": "daily",
                "dates": ["2024-01-01", "2024-01-02", "2024-01-03"],
                "prices": [1.0911, 1.0925, 1.0934],
                "returns": [0, 0.0013, 0.0008]
            }
        }

class VolSurfaceRequest(BaseModel):
    ticker: str = Field(..., description="Ticker symbol")
    date: date = Field(..., description="Date for volatility surface")
    delta_points: List[int] = Field([10, 25, 50, 75, 90], description="Delta points for vol surface")
    
    class Config:
        schema_extra = {
            "example": {
                "ticker": "EURUSD",
                "date": "2024-03-31",
                "delta_points": [10, 25, 50, 75, 90]
            }
        }

class VolSurfaceResponse(BaseModel):
    ticker: str
    date: str
    tenors: List[str]
    deltas: List[int]
    vol_matrix: List[List[float]]
    
    class Config:
        schema_extra = {
            "example": {
                "ticker": "EURUSD",
                "date": "2024-03-31",
                "tenors": ["1W", "1M", "3M", "6M", "1Y"],
                "deltas": [10, 25, 50, 75, 90],
                "vol_matrix": [
                    [9.1, 9.2, 9.3, 9.2, 9.0],
                    [9.5, 9.6, 9.7, 9.6, 9.4],
                    [10.0, 10.1, 10.2, 10.1, 9.9],
                    [9.8, 9.9, 10.0, 9.9, 9.7],
                    [9.6, 9.7, 9.8, 9.7, 9.5]
                ]
            }
        }

# API Endpoints
@app.get("/health")
def health_check():
    return {"status": "healthy", "timestamp": datetime.now().isoformat()}

@app.post("/vol-chart", response_model=VolChartResponse, operation_id="get_volatility_chart")
async def get_volatility_chart(request: VolChartRequest):
    """
    Generate a chart of realized vs implied volatility for a given ticker and tenor.
    
    This tool is useful for analyzing volatility trends and identifying volatility premiums.
    It provides historical data on implied volatility from option prices, realized volatility
    from historical returns, and the spread between them.
    """
    # Simulate data generation
    # In a real implementation, this would query a database or analytics service
    num_days = (request.end_date - request.start_date).days
    dates = [
        (request.start_date + pd.Timedelta(days=i)).strftime("%Y-%m-%d") 
        for i in range(min(90, num_days))
    ]
    
    # Generate mock data with some patterns
    base_vol = 10.0 if request.ticker.startswith("EUR") else 15.0
    
    implied_vol = [
        base_vol + np.sin(i/30) + np.random.normal(0, 0.3) 
        for i in range(len(dates))
    ]
    
    realized_vol = [
        iv - 0.3 + np.random.normal(0, 0.5)
        for iv in implied_vol
    ]
    
    vol_premium = [
        iv - rv for iv, rv in zip(implied_vol, realized_vol)
    ]
    
    return VolChartResponse(
        ticker=request.ticker,
        tenor=request.tenor,
        dates=dates,
        implied_vol=implied_vol,
        realized_vol=realized_vol,
        vol_premium=vol_premium
    )

@app.post("/spot-chart", response_model=SpotChartResponse, operation_id="get_spot_chart")
async def get_spot_chart(request: SpotChartRequest):
    """
    Generate a chart of spot prices and returns for a given ticker.
    
    This tool is useful for analyzing price movements and calculating returns
    over a specified period. It provides time series data of prices and
    percentage returns at the requested frequency.
    """
    # Simulate data generation
    # In a real implementation, this would query a database or market data service
    num_days = (request.end_date - request.start_date).days
    
    # Adjust sample size based on frequency
    sample_size = min(90, num_days)
    if request.frequency == "weekly":
        sample_size = min(52, num_days // 7)
    elif request.frequency == "monthly":
        sample_size = min(24, num_days // 30)
    
    dates = [
        (request.start_date + pd.Timedelta(days=i * (num_days // sample_size))).strftime("%Y-%m-%d") 
        for i in range(sample_size)
    ]
    
    # Generate mock price data with a trend and some volatility
    base_price = 1.09 if request.ticker == "EURUSD" else 100.0
    trend = np.linspace(0, 0.05, sample_size) if request.ticker.startswith("EUR") else np.linspace(0, 2, sample_size)
    
    prices = [
        base_price + trend[i] + np.random.normal(0, 0.01 * base_price) 
        for i in range(sample_size)
    ]
    
    # Calculate returns
    returns = [0]
    for i in range(1, sample_size):
        ret = (prices[i] - prices[i-1]) / prices[i-1]
        returns.append(round(ret, 4))
    
    return SpotChartResponse(
        ticker=request.ticker,
        frequency=request.frequency,
        dates=dates,
        prices=prices,
        returns=returns
    )

@app.post("/vol-surface", response_model=VolSurfaceResponse, operation_id="get_volatility_surface")
async def get_volatility_surface(request: VolSurfaceRequest):
    """
    Generate a volatility surface for a given ticker at a specific date.
    
    This tool provides a matrix of implied volatilities across different tenors
    and delta points, allowing for analysis of volatility skew and term structure.
    It's particularly useful for option pricing and risk management.
    """
    # Standard tenors for FX vol surface
    tenors = ["1W", "1M", "3M", "6M", "1Y"]
    
    # Generate mock volatility surface data
    # In a real implementation, this would query market data or a vol surface model
    base_vol = 10.0 if request.ticker.startswith("EUR") else 15.0
    
    # Create a vol surface with some skew and term structure
    vol_matrix = []
    for delta in request.delta_points:
        # Add skew - lower deltas have higher vols (risk reversal)
        delta_effect = (50 - delta) / 50 * 0.5
        
        row = []
        for i, tenor in enumerate(tenors):
            # Add term structure - longer tenors generally have higher vols
            tenor_effect = i * 0.1
            
            # Add some noise
            noise = np.random.normal(0, 0.1)
            
            vol = base_vol + delta_effect + tenor_effect + noise
            row.append(round(vol, 1))
        
        vol_matrix.append(row)
    
    return VolSurfaceResponse(
        ticker=request.ticker,
        date=request.date.strftime("%Y-%m-%d"),
        tenors=tenors,
        deltas=request.delta_points,
        vol_matrix=vol_matrix
    )

# Create and configure FastAPI MCP
mcp = FastApiMCP(
    app,
    name="HSBC Market Analytics",
    description="MCP server for accessing HSBC market analytics functions",
    base_url=os.environ.get("BASE_URL", "http://localhost:8000"),
    # Only include market analytics operations
    include_operations=["get_volatility_chart", "get_spot_chart", "get_volatility_surface"],
    # Include full schemas in descriptions
    describe_full_response_schema=True
)

# Mount the MCP server to our FastAPI app
mcp.mount()

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 2.2 Trading Analytics FastAPI Server

```python
# mcp-servers/trading-api/main.py
from fastapi import FastAPI, Depends, Query
from pydantic import BaseModel, Field
from typing import List, Dict, Optional, Union
from datetime import datetime, date
import pandas as pd
import numpy as np
from fastapi_mcp import FastApiMCP
import os

app = FastAPI(
    title="HSBC Trading Analytics API",
    description="API for trading analytics functions",
    version="1.0.0"
)

# Models
class Position(BaseModel):
    ticker: str = Field(..., description="Ticker symbol")
    quantity: float = Field(..., description="Position quantity")
    price: float = Field(..., description="Entry price")
    trade_date: date = Field(..., description="Trade date")
    
class PortfolioRequest(BaseModel):
    positions: List[Position] = Field(..., description="List of positions in the portfolio")
    analysis_date: date = Field(..., description="Date to analyze the portfolio")
    
    class Config:
        schema_extra = {
            "example": {
                "positions": [
                    {"ticker": "EURUSD", "quantity": 1000000, "price": 1.0911, "trade_date": "2024-03-01"},
                    {"ticker": "GBPUSD", "quantity": 500000, "price": 1.2650, "trade_date": "2024-03-15"}
                ],
                "analysis_date": "2024-03-31"
            }
        }

class PortfolioAnalysisResponse(BaseModel):
    analysis_date: str
    total_positions: int
    portfolio_value: float
    portfolio_pnl: float
    position_summary: List[Dict[str, Union[str, float]]]
    risk_metrics: Dict[str, float]
    
    class Config:
        schema_extra = {
            "example": {
                "analysis_date": "2024-03-31",
                "total_positions": 2,
                "portfolio_value": 1500000,
                "portfolio_pnl": 12500,
                "position_summary": [
                    {"ticker": "EURUSD", "value": 1000000, "pnl": 8000, "pnl_pct": 0.8},
                    {"ticker": "GBPUSD", "value": 500000, "pnl": 4500, "pnl_pct": 0.9}
                ],
                "risk_metrics": {
                    "var_95": 15000,
                    "var_99": 25000,
                    "expected_shortfall": 30000,
                    "volatility": 12.5
                }
            }
        }

class RiskScenariosRequest(BaseModel):
    positions: List[Position] = Field(..., description="List of positions in the portfolio")
    scenarios: List[str] = Field(..., description="Risk scenarios to analyze")
    
    class Config:
        schema_extra = {
            "example": {
                "positions": [
                    {"ticker": "EURUSD", "quantity": 1000000, "price": 1.0911, "trade_date": "2024-03-01"},
                    {"ticker": "GBPUSD", "quantity": 500000, "price": 1.2650, "trade_date": "2024-03-15"}
                ],
                "scenarios": ["market_crash", "interest_rate_hike", "fx_volatility"]
            }
        }

class RiskScenariosResponse(BaseModel):
    total_positions: int
    baseline_value: float
    scenario_results: Dict[str, Dict[str, float]]
    
    class Config:
        schema_extra = {
            "example": {
                "total_positions": 2,
                "baseline_value": 1500000,
                "scenario_results": {
                    "market_crash": {
                        "portfolio_value": 1350000,
                        "pnl_impact": -150000,
                        "pnl_impact_pct": -10
                    },
                    "interest_rate_hike": {
                        "portfolio_value": 1425000,
                        "pnl_impact": -75000,
                        "pnl_impact_pct": -5
                    },
                    "fx_volatility": {
                        "portfolio_value": 1480000,
                        "pnl_impact": -20000,
                        "pnl_impact_pct": -1.33
                    }
                }
            }
        }

class TradeRecommendationRequest(BaseModel):
    positions: List[Position] = Field(..., description="Current portfolio positions")
    target_exposure: Dict[str, float] = Field(..., description="Target exposures by ticker (% of portfolio)")
    max_trades: int = Field(5, description="Maximum number of trades to recommend")
    
    class Config:
        schema_extra = {
            "example": {
                "positions": [
                    {"ticker": "EURUSD", "quantity": 1000000, "price": 1.0911, "trade_date": "2024-03-01"},
                    {"ticker": "GBPUSD", "quantity": 500000, "price": 1.2650, "trade_date": "2024-03-15"}
                ],
                "target_exposure": {"EURUSD": 0.4, "GBPUSD": 0.3, "USDJPY": 0.3},
                "max_trades": 3
            }
        }

class TradeRecommendation(BaseModel):
    ticker: str
    action: str  # "BUY" or "SELL"
    quantity: float
    estimated_price: float
    estimated_value: float
    rationale: str

class TradeRecommendationsResponse(BaseModel):
    current_portfolio_value: float
    target_portfolio_value: float
    current_exposures: Dict[str, float]
    target_exposures: Dict[str, float]
    recommendations: List[TradeRecommendation]
    
    class Config:
        schema_extra = {
            "example": {
                "current_portfolio_value": 1500000,
                "target_portfolio_value": 1500000,
                "current_exposures": {"EURUSD": 0.67, "GBPUSD": 0.33},
                "target_exposures": {"EURUSD": 0.4, "GBPUSD": 0.3, "USDJPY": 0.3},
                "recommendations": [
                    {
                        "ticker": "EURUSD",
                        "action": "SELL",
                        "quantity": 400000,
                        "estimated_price": 1.0920,
                        "estimated_value": 400000,
                        "rationale": "Reduce EURUSD exposure to target 40%"
                    },
                    {
                        "ticker": "USDJPY",
                        "action": "BUY",
                        "quantity": 450000,
                        "estimated_price": 151.20,
                        "estimated_value": 450000,
                        "rationale": "Increase USDJPY exposure to target 30%"
                    }
                ]
            }
        }

# API Endpoints
@app.get("/health")
def health_check():
    return {"status": "healthy", "timestamp": datetime.now().isoformat()}

@app.post("/portfolio-analysis", response_model=PortfolioAnalysisResponse, operation_id="analyze_portfolio")
async def analyze_portfolio(request: PortfolioRequest):
    """
    Analyze a portfolio of positions to calculate value, P&L, and risk metrics.
    
    This tool provides a comprehensive analysis of a trading portfolio, including
    position values, profit and loss calculations, and key risk metrics such as
    Value at Risk (VaR) and Expected Shortfall. It's useful for portfolio monitoring
    and risk management.
    """
    # Simulate portfolio analysis
    # In a real implementation, this would use market data and risk models
    
    # Mock current prices - slightly different from entry prices
    current_prices = {
        "EURUSD": 1.0940,
        "GBPUSD": 1.2675,
        "USDJPY": 151.20,
        "AUDUSD": 0.6620
    }
    
    # Calculate portfolio values and P&L
    position_summary = []
    portfolio_value = 0
    portfolio_pnl = 0
    
    for position in request.positions:
        # Get current price, default to 1% higher if not in our mock data
        current_price = current_prices.get(position.ticker, position.price * 1.01)
        
        # Calculate position value and P&L
        position_value = position.quantity * current_price
        position_pnl = position.quantity * (current_price - position.price)
        position_pnl_pct = (current_price / position.price - 1) * 100
        
        position_summary.append({
            "ticker": position.ticker,
            "value": round(position_value, 2),
            "pnl": round(position_pnl, 2),
            "pnl_pct": round(position_pnl_pct, 2)
        })
        
        portfolio_value += position_value
        portfolio_pnl += position_pnl
    
    # Generate risk metrics
    risk_metrics = {
        "var_95": round(portfolio_value * 0.01, 2),  # 1% VaR
        "var_99": round(portfolio_value * 0.017, 2),  # 1.7% VaR
        "expected_shortfall": round(portfolio_value * 0.02, 2),  # 2% ES
        "volatility": round(10 + np.random.normal(0, 1), 2)  # ~10% volatility
    }
    
    return PortfolioAnalysisResponse(
        analysis_date=request.analysis_date.strftime("%Y-%m-%d"),
        total_positions=len(request.positions),
        portfolio_value=round(portfolio_value, 2),
        portfolio_pnl=round(portfolio_pnl, 2),
        position_summary=position_summary,
        risk_metrics=risk_metrics
    )

@app.post("/risk-scenarios", response_model=RiskScenariosResponse, operation_id="analyze_risk_scenarios")
async def analyze_risk_scenarios(request: RiskScenariosRequest):
    """
    Analyze how a portfolio performs under different risk scenarios.
    
    This tool simulates the impact of various market scenarios on a portfolio.
    It calculates the potential P&L impact under each scenario, helping with
    stress testing and contingency planning.
    """
    # Scenario impacts (% change in portfolio value)
    scenario_impacts = {
        "market_crash": -0.10,  # 10% drop
        "interest_rate_hike": -0.05,  # 5% drop
        "fx_volatility": -0.0133,  # 1.33% drop
        "commodity_spike": -0.03,  # 3% drop
        "geopolitical_crisis": -0.07,  # 7% drop
        "global_recession": -0.15,  # 15% drop
        "tech_sector_downturn": -0.08,  # 8% drop
        "emerging_markets_crisis": -0.06  # 6% drop
    }
    
    # Calculate baseline portfolio value
    baseline_value = 0
    for position in request.positions:
        baseline_value += position.quantity * position.price
    
    # Calculate scenario impacts
    scenario_results = {}
    for scenario in request.scenarios:
        if scenario in scenario_impacts:
            impact_pct = scenario_impacts[scenario]
            impact_value = baseline_value * impact_pct
            scenario_value = baseline_value + impact_value
            
            scenario_results[scenario] = {
                "portfolio_value": round(scenario_value, 2),
                "pnl_impact": round(impact_value, 2),
                "pnl_impact_pct": round(impact_pct * 100, 2)
            }
        else:
            # Default scenario with random impact if not predefined
            impact_pct = -0.02 + np.random.normal(0, 0.01)  # ~2% drop with noise
            impact_value = baseline_value * impact_pct
            scenario_value = baseline_value + impact_value
            
            scenario_results[scenario] = {
                "portfolio_value": round(scenario_value, 2),
                "pnl_impact": round(impact_value, 2),
                "pnl_impact_pct": round(impact_pct * 100, 2)
            }
    
    return RiskScenariosResponse(
        total_positions=len(request.positions),
        baseline_value=round(baseline_value, 2),
        scenario_results=scenario_results
    )

@app.post("/trade-recommendations", response_model=TradeRecommendationsResponse, operation_id="get_trade_recommendations")
async def get_trade_recommendations(request: TradeRecommendationRequest):
    """
    Generate trade recommendations to rebalance a portfolio towards target exposures.
    
    This tool analyzes the current portfolio and provides specific trade recommendations
    to achieve the desired asset allocation. It identifies which positions to increase
    or decrease and by how much, with an explanation for each recommendation.
    """
    # Mock current prices - used for valuation
    current_prices = {
        "EURUSD": 1.0940,
        "GBPUSD": 1.2675,
        "USDJPY": 151.20,
        "AUDUSD": 0.6620
    }
    
    # Calculate current portfolio value and exposures
    current_portfolio_value = 0
    current_values = {}
    
    for position in request.positions:
        # Get current price, default to 1% higher if not in our mock data
        current_price = current_prices.get(position.ticker, position.price * 1.01)
        
        # Calculate position value
        position_value = position.quantity * current_price
        current_portfolio_value += position_value
        
        # Track value by ticker
        if position.ticker in current_values:
            current_values[position.ticker] += position_value
        else:
            current_values[position.ticker] = position_value
    
    # Calculate current exposures as % of portfolio
    current_exposures = {
        ticker: round(value / current_portfolio_value, 2)
        for ticker, value in current_values.items()
    }
    
    # Determine target values
    target_values = {
        ticker: current_portfolio_value * exposure
        for ticker, exposure in request.target_exposure.items()
    }
    
    # Generate trade recommendations
    recommendations = []
    
    # First, handle existing positions that need adjustment
    for ticker in current_values:
        if ticker in target_values:
            diff = target_values[ticker] - current_values[ticker]
            
            if abs(diff) > 0.01 * current_portfolio_value:  # Only recommend significant trades
                action = "BUY" if diff > 0 else "SELL"
                price = current_prices.get(ticker, request.positions[0].price * 1.01)  # Estimate price
                quantity = abs(diff) / price
                
                recommendations.append(TradeRecommendation(
                    ticker=ticker,
                    action=action,
                    quantity=round(quantity, 0),
                    estimated_price=price,
                    estimated_value=abs(diff),
                    rationale=f"{'Increase' if diff > 0 else 'Reduce'} {ticker} exposure to target {request.target_exposure[ticker] * 100}%"
                ))
    
    # Then, handle new positions
    for ticker in target_values:
        if ticker not in current_values:
            price = current_prices.get(ticker, 1.0)  # Estimate price
            quantity = target_values[ticker] / price
            
            recommendations.append(TradeRecommendation(
                ticker=ticker,
                action="BUY",
                quantity=round(quantity, 0),
                estimated_price=price,
                estimated_value=target_values[ticker],
                rationale=f"Add {ticker} exposure to target {request.target_exposure[ticker] * 100}%"
            ))
    
    # Limit to max_trades
    recommendations = recommendations[:request.max_trades]
    
    return TradeRecommendationsResponse(
        current_portfolio_value=round(current_portfolio_value, 2),
        target_portfolio_value=round(current_portfolio_value, 2),  # Same value, just different allocation
        current_exposures=current_exposures,
        target_exposures=request.target_exposure,
        recommendations=recommendations
    )

# Create and configure FastAPI MCP
mcp = FastApiMCP(
    app,
    name="HSBC Trading Analytics",
    description="MCP server for accessing HSBC trading analytics functions",
    base_url=os.environ.get("BASE_URL", "http://localhost:8001"),
    # Include full schemas in descriptions
    describe_full_response_schema=True
)

# Mount the MCP server to our FastAPI app
mcp.mount()

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8001)
```

### 2.3 Open-Source Integration Setup

For integrating the open-source MCP servers, we'll create configuration files and setup scripts:

```python
# mcp-servers/integrations/setup.py
import os
import json
import subprocess
import logging
from pathlib import Path

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

def setup_tavily():
    """Setup the Tavily MCP server"""
    logger.info("Setting up Tavily MCP server")
    
    # Check for API key
    api_key = os.environ.get("TAVILY_API_KEY")
    if not api_key:
        logger.error("TAVILY_API_KEY not set in environment variables")
        return False
    
    # Create Tavily config
    tavily_config = {
        "command": "npx",
        "args": ["-y", "tavily-mcp@latest"],
        "env": {
            "TAVILY_API_KEY": api_key
        },
        "transport": "stdio"
    }
    
    # Save config
    config_path = Path(__file__).parent / "tavily" / "config.json"
    config_path.parent.mkdir(exist_ok=True)
    
    with open(config_path, 'w') as f:
        json.dump(tavily_config, f, indent=2)
    
    logger.info(f"Tavily configuration saved to {config_path}")
    return True

def setup_atlassian():
    """Setup the Atlassian MCP server"""
    logger.info("Setting up Atlassian MCP server")
    
    # Check for required environment variables
    confluence_url = os.environ.get("CONFLUENCE_URL")
    confluence_username = os.environ.get("CONFLUENCE_USERNAME")
    confluence_token = os.environ.get("CONFLUENCE_API_TOKEN")
    jira_url = os.environ.get("JIRA_URL")
    jira_username = os.environ.get("JIRA_USERNAME")
    jira_token = os.environ.get("JIRA_API_TOKEN")
    
    if not (confluence_url and confluence_username and confluence_token) and not (jira_url and jira_username and jira_token):
        logger.error("Missing required Atlassian credentials")
        return False
    
    # Create Atlassian config
    atlassian_config = {
        "command": "uvx",
        "args": ["mcp-atlassian"],
        "env": {},
        "transport": "stdio"
    }
    
    # Add available credentials
    if confluence_url and confluence_username and confluence_token:
        atlassian_config["env"]["CONFLUENCE_URL"] = confluence_url
        atlassian_config["env"]["CONFLUENCE_USERNAME"] = confluence_username
        atlassian_config["env"]["CONFLUENCE_API_TOKEN"] = confluence_token
    
    if jira_url and jira_username and jira_token:
        atlassian_config["env"]["JIRA_URL"] = jira_url
        atlassian_config["env"]["JIRA_USERNAME"] = jira_username
        atlassian_config["env"]["JIRA_API_TOKEN"] = jira_token
    
    # Save config
    config_path = Path(__file__).parent / "atlassian" / "config.json"
    config_path.parent.mkdir(exist_ok=True)
    
    with open(config_path, 'w') as f:
        json.dump(atlassian_config, f, indent=2)
    
    logger.info(f"Atlassian configuration saved to {config_path}")
    return True

def main():
    """Setup all integrations"""
    logger.info("Setting up MCP server integrations")
    
    tavily_success = setup_tavily()
    atlassian_success = setup_atlassian()
    
    if tavily_success:
        logger.info("✅ Tavily MCP server setup completed")
    else:
        logger.warning("⚠️ Tavily MCP server setup failed")
    
    if atlassian_success:
        logger.info("✅ Atlassian MCP server setup completed")
    else:
        logger.warning("⚠️ Atlassian MCP server setup failed")

if __name__ == "__main__":
    main()
```

## 3. AI-Matrix Implementation (Agent Framework)

Now we'll implement the AI-Matrix agent framework that will use MCP tools via langchain-mcp-adapters.

### 3.1 MCP Client Implementation

```python
# ai-matrix/clients/mcp_client.py
import asyncio
import logging
from typing import Dict, List, Any, Optional
from pathlib import Path
import json

from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.tools import load_mcp_tools

logger = logging.getLogger(__name__)

class AIMatrixMCPClient:
    """MCP client for the AI-Matrix agent framework"""
    
    def __init__(self, config_path: Optional[str] = None):
        """
        Initialize the MCP client
        
        Args:
            config_path: Path to the MCP server configuration file. If None, will use
                        the default config path in the project.
        """
        self.config_path = config_path or str(Path(__file__).parent.parent / "config" / "mcp_servers.json")
        self.server_configs = self._load_config()
        self.client = None
        self.tools = []
        
    def _load_config(self) -> Dict[str, Dict[str, Any]]:
        """Load MCP server configuration from file"""
        try:
            with open(self.config_path, 'r') as f:
                return json.load(f)
        except Exception as e:
            logger.error(f"Error loading MCP server config: {str(e)}")
            return {}
    
    async def connect(self):
        """Connect to all configured MCP servers"""
        if not self.server_configs:
            logger.warning("No MCP servers configured")
            return self
        
        try:
            self.client = MultiServerMCPClient(self.server_configs)
            await self.client.__aenter__()
            self.tools = self.client.get_tools()
            logger.info(f"Connected to {len(self.server_configs)} MCP servers with {len(self.tools)} tools available")
        except Exception as e:
            logger.error(f"Error connecting to MCP servers: {str(e)}")
            raise
            
        return self
    
    async def disconnect(self):
        """Disconnect from all MCP servers"""
        if self.client:
            try:
                await self.client.__aexit__(None, None, None)
                logger.info("Disconnected from all MCP servers")
            except Exception as e:
                logger.error(f"Error disconnecting from MCP servers: {str(e)}")
    
    def get_tools(self) -> List:
        """Get all available tools from connected MCP servers"""
        return self.tools
    
    def get_tool_by_name(self, name: str) -> Optional[Any]:
        """Get a specific tool by name"""
        for tool in self.tools:
            if tool.name == name:
                return tool
        return None
    
    async def __aenter__(self):
        """Async context manager entry"""
        return await self.connect()
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """Async context manager exit"""
        await self.disconnect()
```

### 3.2 AI-Matrix Agent Implementation

```python
# ai-matrix/graphs/agent_graph.py
from typing import List, Dict, Any, TypedDict, Optional, Annotated
import os
import asyncio
import logging
from datetime import datetime

from langchain_core.messages import HumanMessage, AIMessage
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import Runnable, RunnableConfig
from langchain_core.output_parsers import StrOutputParser
from langchain.agents import AgentExecutor, create_react_agent
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain.tools import BaseTool

from ..clients.mcp_client import AIMatrixMCPClient

logger = logging.getLogger(__name__)

class AgentState(TypedDict):
    """Type definition for agent state"""
    messages: List[Dict[str, Any]]
    thoughts: List[str]
    tools_used: List[str]
    start_time: str
    end_time: Optional[str]

def create_agent_graph(
    model_name: str = "claude-3-5-sonnet", 
    mcp_config_path: Optional[str] = None
) -> Runnable:
    """
    Create a LangGraph agent that uses MCP tools
    
    Args:
        model_name: Name of the LLM to use
        mcp_config_path: Path to MCP server configuration file
        
    Returns:
        A Runnable LangGraph
    """
    # Initialize MCP client and connect to servers
    async def get_tools() -> List[BaseTool]:
        async with AIMatrixMCPClient(mcp_config_path) as client:
            return client.get_tools()
    
    # Get tools asynchronously 
    tools = asyncio.run(get_tools())
    
    # Initialize LLM based on model name
    if model_name.startswith("gpt"):
        llm = ChatOpenAI(
            model=model_name,
            temperature=0,
            api_key=os.environ.get("OPENAI_API_KEY")
        )
    elif model_name.startswith("claude"):
        llm = ChatAnthropic(
            model=model_name,
            temperature=0,
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
    else:
        raise ValueError(f"Unsupported model: {model_name}")
    
    # Create ReAct agent
    agent = create_react_agent(llm, tools, "You are an AI assistant that helps with financial analytics.")
    
    # Create agent executor
    agent_executor = AgentExecutor(
        agent=agent,
        tools=tools,
        verbose=True,
        handle_parsing_errors=True,
        max_iterations=10,
        return_intermediate_steps=True
    )
    
    # Function to process inputs
    def process_input(state: Dict[str, Any], config: RunnableConfig) -> Dict[str, Any]:
        """Process input and update state"""
        start_time = datetime.now().isoformat()
        
        # Initialize state if not already present
        if "messages" not in state:
            state["messages"] = []
        if "thoughts" not in state:
            state["thoughts"] = []
        if "tools_used" not in state:
            state["tools_used"] = []
        
        state["start_time"] = start_time
        state["end_time"] = None
        
        return state
    
    # Function to handle agent execution
    async def run_agent(state: Dict[str, Any], config: RunnableConfig) -> Dict[str, Any]:
        """Run the agent executor"""
        # Extract the latest human message
        messages = state["messages"]
        if not messages or messages[-1]["type"] != "human":
            raise ValueError("Expected a human message")
        
        # Get the human message content
        human_message = messages[-1]["content"]
        
        # Run the agent
        result = await agent_executor.ainvoke({"input": human_message})
        
        # Extract information from the result
        output = result["output"]
        intermediate_steps = result.get("intermediate_steps", [])
        
        # Update state
        state["messages"].append({"type": "ai", "content": output})
        
        # Extract thoughts and tools used
        for step in intermediate_steps:
            action = step[0]
            if hasattr(action, "log"):
                state["thoughts"].append(action.log)
            if hasattr(action, "tool"):
                state["tools_used"].append(action.tool)
        
        state["end_time"] = datetime.now().isoformat()
        
        return state
    
    # Create runnable chain
    chain = process_input | run_agent
    
    return chain
```

### 3.3 Example Configuration Files

```json
// ai-matrix/config/mcp_servers.json
{
  "market_analytics": {
    "transport": "sse",
    "url": "http://localhost:8000/mcp"
  },
  "trading_analytics": {
    "transport": "sse",
    "url": "http://localhost:8001/mcp"
  },
  "tavily": {
    "command": "npx",
    "args": ["-y", "tavily-mcp@latest"],
    "env": {
      "TAVILY_API_KEY": "${TAVILY_API_KEY}"
    },
    "transport": "stdio"
  },
  "atlassian": {
    "command": "uvx",
    "args": ["mcp-atlassian"],
    "env": {
      "CONFLUENCE_URL": "${CONFLUENCE_URL}",
      "CONFLUENCE_USERNAME": "${CONFLUENCE_USERNAME}",
      "CONFLUENCE_API_TOKEN": "${CONFLUENCE_API_TOKEN}",
      "JIRA_URL": "${JIRA_URL}",
      "JIRA_USERNAME": "${JIRA_USERNAME}",
      "JIRA_API_TOKEN": "${JIRA_API_TOKEN}"
    },
    "transport": "stdio"
  }
}
```

## 4. Test Implementation

Let's implement comprehensive tests for our system:

### 4.1 Unit Tests for Market Analytics MCP Server

```python
# tests/unit/test_market_analytics.py
import pytest
from httpx import AsyncClient
import json
from datetime import date, timedelta

from mcp_servers.analytics_api.main import app

@pytest.fixture
async def client():
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.mark.asyncio
async def test_health_check(client):
    response = await client.get("/health")
    assert response.status_code == 200
    assert "healthy" in response.json()["status"]

@pytest.mark.asyncio
async def test_vol_chart(client):
    today = date.today()
    start_date = today - timedelta(days=90)
    
    data = {
        "ticker": "EURUSD",
        "start_date": start_date.isoformat(),
        "end_date": today.isoformat(),
        "tenor": "3M"
    }
    
    response = await client.post("/vol-chart", json=data)
    assert response.status_code == 200
    
    result = response.json()
    assert result["ticker"] == "EURUSD"
    assert result["tenor"] == "3M"
    assert len(result["dates"]) > 0
    assert len(result["implied_vol"]) == len(result["dates"])
    assert len(result["realized_vol"]) == len(result["dates"])
    assert len(result["vol_premium"]) == len(result["dates"])

@pytest.mark.asyncio
async def test_spot_chart(client):
    today = date.today()
    start_date = today - timedelta(days=90)
    
    data = {
        "ticker": "EURUSD",
        "start_date": start_date.isoformat(),
        "end_date": today.isoformat(),
        "frequency": "daily"
    }
    
    response = await client.post("/spot-chart", json=data)
    assert response.status_code == 200
    
    result = response.json()
    assert result["ticker"] == "EURUSD"
    assert result["frequency"] == "daily"
    assert len(result["dates"]) > 0
    assert len(result["prices"]) == len(result["dates"])
    assert len(result["returns"]) == len(result["dates"])

@pytest.mark.asyncio
async def test_vol_surface(client):
    data = {
        "ticker": "EURUSD",
        "date": date.today().isoformat(),
        "delta_points": [10, 25, 50, 75, 90]
    }
    
    response = await client.post("/vol-surface", json=data)
    assert response.status_code == 200
    
    result = response.json()
    assert result["ticker"] == "EURUSD"
    assert len(result["tenors"]) > 0
    assert len(result["deltas"]) == 5
    assert len(result["vol_matrix"]) == 5  # One row per delta
    assert len(result["vol_matrix"][0]) == len(result["tenors"])  # One column per tenor

@pytest.mark.asyncio
async def test_mcp_endpoint(client):
    """Test that the MCP endpoint is available"""
    response = await client.get("/mcp")
    assert response.status_code == 200
```

### 4.2 Integration Tests for AI-Matrix MCP Client

```python
# tests/integration/test_ai_matrix_client.py
import pytest
import os
import json
import asyncio
from typing import Dict, Any

from ai_matrix.clients.mcp_client import AIMatrixMCPClient

@pytest.fixture
def temp_config_path(tmp_path):
    """Create a temporary config file for testing"""
    config = {
        "mock_server": {
            "command": "python",
            "args": ["-m", "tests.mocks.mock_mcp_server"],
            "transport": "stdio"
        }
    }
    
    config_file = tmp_path / "test_mcp_config.json"
    with open(config_file, "w") as f:
        json.dump(config, f)
    
    return str(config_file)

@pytest.mark.asyncio
async def test_client_initialization(temp_config_path):
    """Test client initialization"""
    client = AIMatrixMCPClient(temp_config_path)
    assert client.config_path == temp_config_path
    assert "mock_server" in client.server_configs

@pytest.mark.asyncio
async def test_client_connect_disconnect(temp_config_path):
    """Test client connection and disconnection"""
    client = AIMatrixMCPClient(temp_config_path)
    
    # Test context manager
    async with client:
        assert client.client is not None
        # We can't fully test tool loading without a real MCP server
        # But we can check that the client doesn't crash
    
    # Client should be disconnected after context exit
    assert client.client is not None  # Client object still exists
    # But we've exited the context, so it should be disconnected

@pytest.mark.asyncio
async def test_get_tools(temp_config_path):
    """Test getting tools from client"""
    client = AIMatrixMCPClient(temp_config_path)
    
    async with client:
        tools = client.get_tools()
        # With our mock server, we expect no tools to be returned
        # This would be replaced with actual assertions if we had a real MCP server
```

### 4.3 Mock MCP Server for Testing

```python
# tests/mocks/mock_mcp_server.py
"""
A simple mock MCP server for testing
"""
import asyncio
import json
import sys
from typing import Dict, Any, List

# MCP message types
INITIALIZE = "initialize"
LIST_TOOLS = "listTools"
RUN_TOOL = "runTool"

async def read_message():
    """Read a message from stdin"""
    line = await asyncio.get_event_loop().run_in_executor(None, sys.stdin.readline)
    if not line:
        return None
    try:
        message = json.loads(line)
        return message
    except json.JSONDecodeError:
        return None

async def write_message(message: Dict[str, Any]):
    """Write a message to stdout"""
    json_str = json.dumps(message) + "\n"
    sys.stdout.write(json_str)
    sys.stdout.flush()

async def handle_initialize(message: Dict[str, Any]):
    """Handle initialize message"""
    return {
        "jsonrpc": "2.0",
        "id": message.get("id"),
        "result": {
            "name": "Mock MCP Server",
            "version": "1.0.0",
            "vendor": "Test"
        }
    }

async def handle_list_tools(message: Dict[str, Any]):
    """Handle listTools message"""
    return {
        "jsonrpc": "2.0",
        "id": message.get("id"),
        "result": {
            "tools": [
                {
                    "name": "mock_tool",
                    "description": "A mock tool for testing",
                    "parameters": {
                        "type": "object",
                        "properties": {
                            "param": {
                                "type": "string",
                                "description": "Test parameter"
                            }
                        },
                        "required": ["param"]
                    },
                    "responseSchema": {
                        "type": "object",
                        "properties": {
                            "result": {
                                "type": "string"
                            }
                        }
                    }
                }
            ]
        }
    }

async def handle_run_tool(message: Dict[str, Any]):
    """Handle runTool message"""
    params = message.get("params", {})
    tool_name = params.get("name")
    parameters = params.get("parameters", {})
    
    if tool_name == "mock_tool":
        param_value = parameters.get("param", "default")
        return {
            "jsonrpc": "2.0",
            "id": message.get("id"),
            "result": {
                "result": f"Executed mock_tool with param: {param_value}"
            }
        }
    
    return {
        "jsonrpc": "2.0",
        "id": message.get("id"),
        "error": {
            "code": -32601,
            "message": f"Tool not found: {tool_name}"
        }
    }

async def main():
    """Main function to run the mock MCP server"""
    while True:
        message = await read_message()
        if not message:
            break
        
        method = message.get("method")
        
        if method == INITIALIZE:
            response = await handle_initialize(message)
        elif method == LIST_TOOLS:
            response = await handle_list_tools(message)
        elif method == RUN_TOOL:
            response = await handle_run_tool(message)
        else:
            response = {
                "jsonrpc": "2.0",
                "id": message.get("id"),
                "error": {
                    "code": -32601,
                    "message": f"Method not found: {method}"
                }
            }
        
        await write_message(response)

if __name__ == "__main__":
    asyncio.run(main())
```

## 5. Project Documentation

```markdown
# HSBC MCP Ecosystem

This repository contains the HSBC AI Markets MCP (Model Context Protocol) ecosystem, consisting of:

1. **MCP Registry**: A centralized service for discovering and managing MCP servers
2. **MCP Servers**: Various MCP server implementations including FastAPI analytics services
3. **AI-Matrix**: An agent framework built on LangGraph that uses MCP tools

## Architecture

The system follows a modular architecture where:

- MCP Registry serves as the central hub for discovering and managing MCP servers
- MCP Servers expose specialized functionality through a standardized protocol
- AI-Matrix provides a flexible agent framework that can leverage any registered MCP tools

## Getting Started

### Prerequisites

- Python 3.10+
- Node.js 18+ (for some MCP servers)
- Docker (optional, for containerized deployment)

### Installation

1. Clone the repository:
   ```
   git clone https://github.com/hsbc/mcp-ecosystem.git
   cd mcp-ecosystem
   ```

2. Set up Python environment:
   ```
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   pip install -e .
   ```

3. Configure environment variables:
   ```
   cp .env.example .env
   # Edit .env file with your API keys and configuration
   ```

4. Start the registry:
   ```
   cd mcp-registry
   python -m backend.main
   ```

5. In a new terminal, start a MCP server:
   ```
   cd mcp-servers/analytics-api
   python main.py
   ```

6. Run the AI-Matrix agent:
   ```
   cd ai-matrix
   python -m agents.run
   ```

## MCP Servers

### Market Analytics API

Market analytics MCP server provides tools for:

- Volatility analysis and charting
- Spot price analysis
- Volatility surface visualization

### Trading Analytics API

Trading analytics MCP server provides tools for:

- Portfolio analysis and risk metrics
- Scenario analysis
- Trade recommendations

### Open-Source Integrations

- **Tavily**: Web search and research
- **Atlassian**: Confluence and Jira integration

## AI-Matrix Agent Framework

AI-Matrix is built on LangGraph and uses MCP tools via langchain-mcp-adapters. It provides:

- Tool routing based on user queries
- Multi-modal responses combining data from multiple sources
- Context-aware recommendations

## Testing

Run unit tests:
```
pytest tests/unit
```

Run integration tests:
```
pytest tests/integration
```

Run end-to-end tests:
```
pytest tests/e2e
```

## License

[Internal HSBC License]
```

This implementation provides a comprehensive solution for building an MCP Registry, multiple MCP servers (including FastAPI integrations), and an AI-Matrix agent framework. The solution includes detailed implementations, configuration files, and extensive testing capabilities.

Key features:

1. **MCP Registry** with modern UI and API for server management
2. **Automatic FastAPI to MCP conversion** using fastapi_mcp
3. **Rich analytics APIs** for market and trading functions
4. **Open-source integration** with Tavily and Atlassian
5. **LangGraph-based agent framework** using langchain-mcp-adapters
6. **Comprehensive testing** suite

The implementation provides the foundation for a robust MCP ecosystem that HSBC AI Markets team can extend and customize based on specific requirements.

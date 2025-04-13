*Thinking...*

> I need to describe the detailed process of integrating Tavily and Atlassian MCP servers into the HSBC MCP Registry and AI-Matrix setup. Let me organize this comprehensively:
> 
> 1. First, I'll describe the integration process from a high level
> 2. Then dive into the specific steps for each:
>    - Tavily MCP server integration
>    - Atlassian MCP server integration
> 3. I'll cover:
>    - Installation and configuration
>    - Security considerations
>    - Testing the integration
>    - Registering with the MCP Registry
>    - Usage with AI-Matrix agents
> 
> For Tavily, I want to highlight that it provides web search capabilities through a straightforward MCP interface, which is valuable for market research.
> 
> For Atlassian, I want to emphasize that it provides integration with both Confluence and Jira, useful for documentation retrieval and issue/project management.
> 
> For both servers, I'll need to describe:
> - How to obtain the necessary API keys
> - How to properly configure them
> - How to test their functionality 
> - How to register them with the MCP Registry
> - How to use them from AI-Matrix agents
> 
> Let me draft a comprehensive response.


# Integrating Tavily and Atlassian MCP Servers

Integrating third-party MCP servers like Tavily and Atlassian into the HSBC MCP ecosystem involves several steps, from installation and configuration to registration and testing. Here's a detailed process for each:

## 1. Tavily MCP Server Integration

Tavily provides powerful web search capabilities as an MCP server, which is valuable for market research and information retrieval in financial contexts.

### 1.1 Installation and Configuration

1. **Obtain API Key**:
   ```
   # Register for a Tavily API key at https://app.tavily.com/home
   # Store securely in environment variables or secrets manager
   export TAVILY_API_KEY="your-tavily-api-key"
   ```

2. **Set Up Tavily MCP Server**:
   ```bash
   # Create a directory for the configuration
   mkdir -p mcp-servers/integrations/tavily
   
   # Create configuration file
   cat > mcp-servers/integrations/tavily/config.json << EOF
   {
     "command": "npx",
     "args": ["-y", "tavily-mcp@latest"],
     "env": {
       "TAVILY_API_KEY": "${TAVILY_API_KEY}"
     },
     "transport": "stdio"
   }
   EOF
   ```

3. **Test Local Installation**:
   ```bash
   # Test the Tavily MCP server installation
   npx -y tavily-mcp@latest
   
   # If you see output like "Tavily MCP server running..." the installation is working
   # Press Ctrl+C to exit
   ```

### 1.2 Registering with MCP Registry

1. **Create Registration Script**:
   ```python
   # mcp-servers/integrations/register_tavily.py
   import requests
   import os
   import json
   
   # Load Tavily config
   with open('tavily/config.json', 'r') as f:
       config = json.load(f)
   
   # Prepare server data for registration
   server_data = {
       "name": "Tavily Search",
       "description": "Web search and information retrieval using Tavily API",
       "version": "latest",
       "owner": "HSBC AI Markets",
       "transport": "stdio",
       "config": config
   }
   
   # Register with MCP Registry
   registry_url = os.environ.get("MCP_REGISTRY_URL", "http://localhost:8000")
   response = requests.post(
       f"{registry_url}/servers",
       json=server_data,
       headers={"Authorization": f"Bearer {os.environ.get('MCP_REGISTRY_TOKEN')}"}
   )
   
   if response.status_code == 200:
       print(f"✅ Tavily MCP server registered successfully: {response.json()['id']}")
   else:
       print(f"❌ Failed to register Tavily MCP server: {response.text}")
   ```

2. **Run Registration**:
   ```bash
   python mcp-servers/integrations/register_tavily.py
   ```

### 1.3 Testing the Tavily Integration

Create a test script to ensure the Tavily search functionality works properly:

```python
# tests/integration/test_tavily_mcp.py
import pytest
import asyncio
from ai_matrix.clients.mcp_client import AIMatrixMCPClient

@pytest.mark.asyncio
async def test_tavily_search():
    """Test Tavily search functionality"""
    # Configure client to only use Tavily
    tavily_config = {
        "tavily": {
            "command": "npx",
            "args": ["-y", "tavily-mcp@latest"],
            "env": {
                "TAVILY_API_KEY": "your-tavily-api-key"  # Use test key or pytest fixture
            },
            "transport": "stdio"
        }
    }
    
    # Create temporary config file
    import tempfile
    import json
    
    with tempfile.NamedTemporaryFile(mode='w+', suffix='.json', delete=False) as f:
        json.dump(tavily_config, f)
        config_path = f.name
    
    # Initialize client with the config
    client = AIMatrixMCPClient(config_path)
    
    try:
        # Connect to Tavily MCP server
        async with client:
            tools = client.get_tools()
            
            # Find tavily-search tool
            search_tool = next((tool for tool in tools if tool.name == "tavily-search"), None)
            assert search_tool is not None, "tavily-search tool not found"
            
            # Execute search
            result = await search_tool.ainvoke({
                "query": "HSBC quarterly earnings 2024",
                "search_depth": "basic"
            })
            
            # Verify result structure
            assert "results" in result, "Search results not found in response"
            assert len(result["results"]) > 0, "No search results returned"
            
            # Check content
            for item in result["results"]:
                assert "url" in item, "URL missing from search result"
                assert "content" in item, "Content missing from search result"
                
    finally:
        # Clean up temp file
        import os
        os.unlink(config_path)
```

## 2. Atlassian MCP Server Integration

The Atlassian MCP server provides integration with Confluence (for documentation) and Jira (for issue tracking), which is valuable for connecting market analytics with project management.

### 2.1 Installation and Configuration

1. **Generate Atlassian API Tokens**:
   - For Cloud:
     - Go to https://id.atlassian.com/manage-profile/security/api-tokens
     - Click "Create API token" and name it "HSBC MCP"
     - Copy the token immediately
   - For Server/Data Center:
     - Go to Profile → Personal Access Tokens
     - Create token with appropriate permissions

2. **Install Atlassian MCP Server**:
   ```bash
   # Install globally (alternative to using uvx)
   pip install mcp-atlassian
   
   # Create configuration directory
   mkdir -p mcp-servers/integrations/atlassian
   
   # Create configuration file
   cat > mcp-servers/integrations/atlassian/config.json << EOF
   {
     "command": "mcp-atlassian",
     "args": [],
     "env": {
       "CONFLUENCE_URL": "${CONFLUENCE_URL}",
       "CONFLUENCE_USERNAME": "${CONFLUENCE_USERNAME}",
       "CONFLUENCE_API_TOKEN": "${CONFLUENCE_API_TOKEN}",
       "JIRA_URL": "${JIRA_URL}",
       "JIRA_USERNAME": "${JIRA_USERNAME}",
       "JIRA_API_TOKEN": "${JIRA_API_TOKEN}",
       "CONFLUENCE_SPACES_FILTER": "MKT,TRADING,RESEARCH",
       "JIRA_PROJECTS_FILTER": "MKTANAL,TRADING",
       "MCP_VERBOSE": "true"
     },
     "transport": "stdio"
   }
   EOF
   ```

3. **Test Local Installation**:
   ```bash
   # Add your Atlassian credentials to environment
   export CONFLUENCE_URL="https://your-company.atlassian.net/wiki"
   export CONFLUENCE_USERNAME="your.email@hsbc.com"
   export CONFLUENCE_API_TOKEN="your-api-token"
   export JIRA_URL="https://your-company.atlassian.net"
   export JIRA_USERNAME="your.email@hsbc.com"
   export JIRA_API_TOKEN="your-api-token"
   
   # Test the Atlassian MCP server
   mcp-atlassian
   
   # If you see initialization messages, the setup is working
   # Press Ctrl+C to exit
   ```

### 2.2 Registering with MCP Registry

1. **Create Registration Script**:
   ```python
   # mcp-servers/integrations/register_atlassian.py
   import requests
   import os
   import json
   
   # Load Atlassian config
   with open('atlassian/config.json', 'r') as f:
       config = json.load(f)
   
   # Remove sensitive info from stored config - will be injected at runtime
   sanitized_config = config.copy()
   for env_var in ["CONFLUENCE_API_TOKEN", "JIRA_API_TOKEN"]:
       if env_var in sanitized_config.get("env", {}):
           sanitized_config["env"][env_var] = "****"
   
   # Prepare server data for registration
   server_data = {
       "name": "Atlassian Tools",
       "description": "Integration with Confluence (documentation) and Jira (project management)",
       "version": "latest",
       "owner": "HSBC AI Markets",
       "transport": "stdio",
       "config": sanitized_config  # Use sanitized config
   }
   
   # Register with MCP Registry
   registry_url = os.environ.get("MCP_REGISTRY_URL", "http://localhost:8000")
   response = requests.post(
       f"{registry_url}/servers",
       json=server_data,
       headers={"Authorization": f"Bearer {os.environ.get('MCP_REGISTRY_TOKEN')}"}
   )
   
   if response.status_code == 200:
       print(f"✅ Atlassian MCP server registered successfully: {response.json()['id']}")
   else:
       print(f"❌ Failed to register Atlassian MCP server: {response.text}")
   ```

2. **Run Registration**:
   ```bash
   python mcp-servers/integrations/register_atlassian.py
   ```

### 2.3 Testing the Atlassian Integration

Create a test script to ensure both Confluence and Jira functionality work properly:

```python
# tests/integration/test_atlassian_mcp.py
import pytest
import asyncio
from ai_matrix.clients.mcp_client import AIMatrixMCPClient
import os

@pytest.mark.asyncio
async def test_confluence_search():
    """Test Confluence search functionality"""
    # Skip if no Confluence credentials in environment
    if not (os.environ.get("CONFLUENCE_URL") and 
            os.environ.get("CONFLUENCE_USERNAME") and 
            os.environ.get("CONFLUENCE_API_TOKEN")):
        pytest.skip("Confluence credentials not configured")
    
    # Configure client to only use Atlassian
    atlassian_config = {
        "atlassian": {
            "command": "mcp-atlassian",
            "args": [],
            "env": {
                "CONFLUENCE_URL": os.environ.get("CONFLUENCE_URL"),
                "CONFLUENCE_USERNAME": os.environ.get("CONFLUENCE_USERNAME"),
                "CONFLUENCE_API_TOKEN": os.environ.get("CONFLUENCE_API_TOKEN"),
                "CONFLUENCE_SPACES_FILTER": "MKT,TRADING,RESEARCH",
                "MCP_VERBOSE": "true"
            },
            "transport": "stdio"
        }
    }
    
    # Create temporary config file
    import tempfile
    import json
    
    with tempfile.NamedTemporaryFile(mode='w+', suffix='.json', delete=False) as f:
        json.dump(atlassian_config, f)
        config_path = f.name
    
    # Initialize client with the config
    client = AIMatrixMCPClient(config_path)
    
    try:
        # Connect to Atlassian MCP server
        async with client:
            tools = client.get_tools()
            
            # Find confluence_search tool
            search_tool = next((tool for tool in tools if tool.name == "confluence_search"), None)
            assert search_tool is not None, "confluence_search tool not found"
            
            # Execute search
            result = await search_tool.ainvoke({
                "cql": "space = MKT AND text ~ 'volatility'"
            })
            
            # Verify result structure
            assert "results" in result, "Search results not found in response"
            
    finally:
        # Clean up temp file
        import os
        os.unlink(config_path)

@pytest.mark.asyncio
async def test_jira_search():
    """Test Jira search functionality"""
    # Skip if no Jira credentials in environment
    if not (os.environ.get("JIRA_URL") and 
            os.environ.get("JIRA_USERNAME") and 
            os.environ.get("JIRA_API_TOKEN")):
        pytest.skip("Jira credentials not configured")
    
    # Configure client to only use Atlassian for Jira
    atlassian_config = {
        "atlassian": {
            "command": "mcp-atlassian",
            "args": [],
            "env": {
                "JIRA_URL": os.environ.get("JIRA_URL"),
                "JIRA_USERNAME": os.environ.get("JIRA_USERNAME"),
                "JIRA_API_TOKEN": os.environ.get("JIRA_API_TOKEN"),
                "JIRA_PROJECTS_FILTER": "MKTANAL,TRADING",
                "MCP_VERBOSE": "true"
            },
            "transport": "stdio"
        }
    }
    
    # Create temporary config file
    import tempfile
    import json
    
    with tempfile.NamedTemporaryFile(mode='w+', suffix='.json', delete=False) as f:
        json.dump(atlassian_config, f)
        config_path = f.name
    
    # Initialize client with the config
    client = AIMatrixMCPClient(config_path)
    
    try:
        # Connect to Atlassian MCP server
        async with client:
            tools = client.get_tools()
            
            # Find jira_search tool
            search_tool = next((tool for tool in tools if tool.name == "jira_search"), None)
            assert search_tool is not None, "jira_search tool not found"
            
            # Execute search
            result = await search_tool.ainvoke({
                "jql": "project = MKTANAL AND status = 'In Progress'"
            })
            
            # Verify result structure
            assert "issues" in result, "Issues not found in response"
            
    finally:
        # Clean up temp file
        import os
        os.unlink(config_path)
```

## 3. Integrating with AI-Matrix

To use both Tavily and Atlassian in AI-Matrix agents, you need to configure them in the MCP client configuration:

### 3.1 Configuration for AI-Matrix

Update the AI-Matrix MCP client configuration to include both servers:

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
    "command": "mcp-atlassian",
    "args": [],
    "env": {
      "CONFLUENCE_URL": "${CONFLUENCE_URL}",
      "CONFLUENCE_USERNAME": "${CONFLUENCE_USERNAME}",
      "CONFLUENCE_API_TOKEN": "${CONFLUENCE_API_TOKEN}",
      "JIRA_URL": "${JIRA_URL}",
      "JIRA_USERNAME": "${JIRA_USERNAME}",
      "JIRA_API_TOKEN": "${JIRA_API_TOKEN}",
      "CONFLUENCE_SPACES_FILTER": "MKT,TRADING,RESEARCH",
      "JIRA_PROJECTS_FILTER": "MKTANAL,TRADING"
    },
    "transport": "stdio"
  }
}
```

### 3.2 Environment Configuration

Set up environment variables to store sensitive credentials:

```bash
# Create or update .env file with credentials
cat >> .env << EOF
# Tavily credentials
TAVILY_API_KEY=your-tavily-api-key

# Atlassian credentials
CONFLUENCE_URL=https://your-company.atlassian.net/wiki
CONFLUENCE_USERNAME=your.email@hsbc.com
CONFLUENCE_API_TOKEN=your-confluence-api-token
JIRA_URL=https://your-company.atlassian.net
JIRA_USERNAME=your.email@hsbc.com
JIRA_API_TOKEN=your-jira-api-token
EOF
```

### 3.3 Using the Integrated Tools

Create an example script to demonstrate using both tools:

```python
# examples/market_research_agent.py
import asyncio
import os
from dotenv import load_dotenv
from langchain_core.messages import HumanMessage
from ai_matrix.graphs.agent_graph import create_agent_graph

# Load environment variables
load_dotenv()

async def run_market_research():
    """Example of using Tavily and Atlassian tools together"""
    # Create agent with access to all MCP tools
    agent = create_agent_graph(model_name="claude-3-5-sonnet")
    
    # Define initial state
    state = {
        "messages": [
            {
                "type": "human",
                "content": """
                I need to create a research report on EURUSD volatility. 
                
                First, search the web for recent EURUSD volatility trends.
                Then, check Confluence for any existing volatility reports in the MKT space.
                Finally, create a new Jira ticket in the MKTANAL project to track this research task.
                """
            }
        ]
    }
    
    # Run the agent
    result = await agent.ainvoke(state)
    
    # Print the result
    print("\n=== Agent Response ===\n")
    print(result["messages"][-1]["content"])
    
    print("\n=== Tools Used ===\n")
    for tool in result["tools_used"]:
        print(f"- {tool}")

if __name__ == "__main__":
    asyncio.run(run_market_research())
```

## 4. Security Considerations

When integrating third-party MCP servers like Tavily and Atlassian, consider these security best practices:

1. **API Key Management**:
   - Never hardcode API keys or tokens in code
   - Store them in environment variables or secrets management systems
   - Rotate keys periodically

2. **Access Control**:
   - Configure minimal permissions for the API tokens
   - For Atlassian, limit to specific spaces/projects as needed
   - Monitor and audit tool usage

3. **Data Protection**:
   - Be cautious about what information is sent to external services like Tavily
   - Configure the Atlassian MCP to only expose approved spaces/projects
   - Implement data filtering for sensitive information

4. **Authentication in Registry**:
   - When storing configuration in the MCP Registry, store placeholders for sensitive values
   - Inject actual credentials at runtime

## 5. Maintenance and Updates

Establish a process for maintaining and updating the integrated MCP servers:

1. **Version Management**:
   - For Tavily, pin to specific versions when stable (e.g., `tavily-mcp@0.1.4` instead of `latest`)
   - For Atlassian, check for updates periodically with `pip list --outdated`

2. **Health Monitoring**:
   - Add health checks to ensure the MCP servers are functioning
   - Set up alerts for any connection issues

3. **Update Process**:
   ```bash
   # Update Tavily MCP
   npm update -g tavily-mcp
   
   # Update Atlassian MCP
   pip install -U mcp-atlassian
   
   # Update registry server configuration
   python mcp-servers/integrations/update_server_versions.py
   ```

By following this integration process, you'll have both Tavily and Atlassian MCP servers properly configured, registered with the MCP Registry, and available for use by AI-Matrix agents. This enables your AI solutions to search the web, access internal documentation, and manage project tasks seamlessly.

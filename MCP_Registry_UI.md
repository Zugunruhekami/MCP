*Thinking...*

> To dynamically populate the server grid in the UI, we need to:
> 
> 1. Create an API endpoint that provides information about all the mounted MCP servers
> 2. Write JavaScript that fetches this data and renders it in the UI
> 3. Style the grid to make it visually appealing like the mcp.so or mastra.ai sites
> 
> Let's start by refining our FastAPI backend to provide detailed server information:
> 
> 1. First, we need an endpoint that returns information about all servers
> 2. Then we need to create a JavaScript file that fetches and renders this data
> 3. Finally, we need CSS to style the grid
> 
> I'll create a complete solution for dynamically populating the server grid:
> 
> 1. API endpoint for server information
> 2. HTML template with placeholders
> 3. JavaScript to fetch and render data
> 4. CSS for styling the grid
> 5. Examples of how different types of servers would appear
> 
> This will give a professional-looking UI similar to the examples the user mentioned (mcp.so and mastra.ai/mcp-registry-registry).


# Dynamically Populating the Server Grid in the UI

To create a dynamic server grid similar to mcp.so or mastra.ai/mcp-registry-registry, you'll need to implement both backend API endpoints and frontend JavaScript. Here's a complete implementation:

## 1. Enhanced Backend API Endpoints

Add these endpoints to your FastAPI app to provide server information:

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

# Expose MCP hub at /mcp
hub.attach_to_fastapi(app, "/mcp")

# API endpoint to get comprehensive server information
@app.get("/api/servers")
async def get_servers():
    # Collect servers data
    servers = []
    
    # Get information for each mounted server
    for prefix, server in hub._mounted_servers.items():
        # Get tools for this server
        tools = await server.list_tools()
        # Get resources for this server
        resources = await server.list_resources()
        
        # Format server data
        server_data = {
            "id": prefix,
            "name": server.name,
            "description": getattr(server, "description", f"{server.name} MCP Server"),
            "tools": [{"name": t.name, "description": t.description} for t in tools],
            "resources": [{"uri": r.uri, "description": r.description} for r in resources],
            "type": "fastmcp" if isinstance(server, FastMCP) else "proxy",
            "tags": getattr(server, "tags", []),
            "usage_count": getattr(server, "usage_count", random.randint(50, 5000))  # For demo
        }
        servers.append(server_data)
    
    return {"servers": servers}

# Endpoint to get detailed info for a specific server
@app.get("/api/servers/{server_id}")
async def get_server(server_id: str):
    if server_id not in hub._mounted_servers:
        return {"error": "Server not found"}
    
    server = hub._mounted_servers[server_id]
    tools = await server.list_tools()
    resources = await server.list_resources()
    
    return {
        "id": server_id,
        "name": server.name,
        "description": getattr(server, "description", f"{server.name} MCP Server"),
        "tools": [{"name": t.name, "description": t.description, "parameters": t.parameters} for t in tools],
        "resources": [{"uri": r.uri, "description": r.description} for r in resources]
    }
```

## 2. HTML Template with Grid Structure

Update your HTML template with better structure:

```html
<!DOCTYPE html>
<html>
<head>
    <title>MCP Hub</title>
    <link rel="stylesheet" href="/static/css/styles.css">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body>
    <header>
        <div class="container">
            <h1>MCP Hub</h1>
            <p>Your centralized registry for Model Context Protocol servers</p>
        </div>
    </header>
    
    <main class="container">
        <div class="search-container">
            <input type="text" id="search-input" placeholder="Search servers...">
        </div>
        
        <div class="filters">
            <button class="filter active" data-filter="all">All</button>
            <button class="filter" data-filter="fastmcp">FastMCP</button>
            <button class="filter" data-filter="proxy">Proxy</button>
        </div>
        
        <div id="server-grid" class="server-grid">
            <!-- Dynamically populated by JavaScript -->
            <div class="loading">Loading servers...</div>
        </div>
    </main>
    
    <div id="server-modal" class="modal">
        <div class="modal-content">
            <span class="close">&times;</span>
            <div id="modal-body">
                <!-- Server details will be displayed here -->
            </div>
        </div>
    </div>
    
    <script src="/static/js/app.js"></script>
</body>
</html>
```

## 3. JavaScript to Populate the Grid

Create a file `static/js/app.js`:

```javascript
document.addEventListener('DOMContentLoaded', function() {
    const serverGrid = document.getElementById('server-grid');
    const searchInput = document.getElementById('search-input');
    const filterButtons = document.querySelectorAll('.filter');
    const modal = document.getElementById('server-modal');
    const modalBody = document.getElementById('modal-body');
    const closeModal = document.querySelector('.close');
    
    let allServers = [];
    let currentFilter = 'all';
    
    // Fetch servers and render grid
    async function fetchServers() {
        try {
            const response = await fetch('/api/servers');
            const data = await response.json();
            allServers = data.servers;
            renderGrid(allServers);
        } catch (error) {
            console.error('Error fetching servers:', error);
            serverGrid.innerHTML = '<div class="error">Failed to load servers</div>';
        }
    }
    
    // Render the server grid
    function renderGrid(servers) {
        // Clear loading message
        serverGrid.innerHTML = '';
        
        if (servers.length === 0) {
            serverGrid.innerHTML = '<div class="no-results">No servers found</div>';
            return;
        }
        
        servers.forEach(server => {
            const card = document.createElement('div');
            card.className = `server-card ${server.type}`;
            card.dataset.id = server.id;
            
            const toolCount = server.tools ? server.tools.length : 0;
            const resourceCount = server.resources ? server.resources.length : 0;
            
            // Generate tags HTML
            const tagsHtml = server.tags ? 
                server.tags.map(tag => `<span class="tag">#${tag}</span>`).join('') : '';
            
            card.innerHTML = `
                <h3>${server.name}</h3>
                <p class="description">${server.description}</p>
                <div class="stats">
                    <span>${toolCount} tools</span>
                    <span>${resourceCount} resources</span>
                    <span>${server.usage_count.toLocaleString()} uses</span>
                </div>
                <div class="tags">
                    ${tagsHtml}
                    <span class="tag type-tag">#${server.type}</span>
                </div>
            `;
            
            // Add click event to show server details
            card.addEventListener('click', () => showServerDetails(server.id));
            
            serverGrid.appendChild(card);
        });
    }
    
    // Show server details in modal
    async function showServerDetails(serverId) {
        try {
            const response = await fetch(`/api/servers/${serverId}`);
            const server = await response.json();
            
            if (server.error) {
                modalBody.innerHTML = `<div class="error">${server.error}</div>`;
                modal.style.display = 'block';
                return;
            }
            
            // Format tools as a list
            const toolsHtml = server.tools.map(tool => `
                <div class="tool">
                    <h4>${tool.name}</h4>
                    <p>${tool.description || 'No description available'}</p>
                </div>
            `).join('');
            
            // Format resources as a list
            const resourcesHtml = server.resources.map(resource => `
                <div class="resource">
                    <h4>${resource.uri}</h4>
                    <p>${resource.description || 'No description available'}</p>
                </div>
            `).join('');
            
            modalBody.innerHTML = `
                <h2>${server.name}</h2>
                <p class="server-description">${server.description}</p>
                
                <h3>Tools</h3>
                <div class="tools-list">
                    ${toolsHtml || '<p>No tools available</p>'}
                </div>
                
                <h3>Resources</h3>
                <div class="resources-list">
                    ${resourcesHtml || '<p>No resources available</p>'}
                </div>
                
                <div class="server-footer">
                    <button id="try-server" data-id="${server.id}">Try in Claude</button>
                </div>
            `;
            
            // Add event listener to the Try button
            const tryButton = document.getElementById('try-server');
            if (tryButton) {
                tryButton.addEventListener('click', () => {
                    // You could integrate with Claude Desktop here
                    alert(`This would launch Claude Desktop with the ${server.name} MCP server`);
                });
            }
            
            modal.style.display = 'block';
        } catch (error) {
            console.error('Error fetching server details:', error);
            modalBody.innerHTML = '<div class="error">Failed to load server details</div>';
            modal.style.display = 'block';
        }
    }
    
    // Search functionality
    searchInput.addEventListener('input', () => {
        const searchTerm = searchInput.value.toLowerCase();
        const filteredServers = allServers.filter(server => {
            // If a filter is active, respect it
            if (currentFilter !== 'all' && server.type !== currentFilter) {
                return false;
            }
            
            // Check if server matches search term
            return (
                server.name.toLowerCase().includes(searchTerm) ||
                server.description.toLowerCase().includes(searchTerm) ||
                (server.tags && server.tags.some(tag => tag.toLowerCase().includes(searchTerm)))
            );
        });
        
        renderGrid(filteredServers);
    });
    
    // Filter functionality
    filterButtons.forEach(button => {
        button.addEventListener('click', () => {
            // Update active class
            filterButtons.forEach(btn => btn.classList.remove('active'));
            button.classList.add('active');
            
            // Set current filter
            currentFilter = button.dataset.filter;
            
            // Apply filter and current search
            const searchTerm = searchInput.value.toLowerCase();
            const filteredServers = allServers.filter(server => {
                if (currentFilter === 'all') {
                    return server.name.toLowerCase().includes(searchTerm) ||
                           server.description.toLowerCase().includes(searchTerm);
                }
                return server.type === currentFilter && (
                    server.name.toLowerCase().includes(searchTerm) ||
                    server.description.toLowerCase().includes(searchTerm)
                );
            });
            
            renderGrid(filteredServers);
        });
    });
    
    // Close modal when clicking X
    closeModal.addEventListener('click', () => {
        modal.style.display = 'none';
    });
    
    // Close modal when clicking outside
    window.addEventListener('click', (event) => {
        if (event.target === modal) {
            modal.style.display = 'none';
        }
    });
    
    // Initial load
    fetchServers();
});
```

## 4. CSS for Styling the Grid

Create a file `static/css/styles.css`:

```css
:root {
    --primary: #6366f1;
    --primary-dark: #4f46e5;
    --secondary: #f59e0b;
    --text: #1f2937;
    --text-light: #6b7280;
    --background: #f9fafb;
    --card: #ffffff;
    --border: #e5e7eb;
    --shadow: rgba(0, 0, 0, 0.1);
}

* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    background-color: var(--background);
    color: var(--text);
    line-height: 1.6;
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 1rem;
}

header {
    background-color: var(--primary);
    color: white;
    padding: 2rem 0;
    margin-bottom: 2rem;
}

header h1 {
    margin-bottom: 0.5rem;
}

main {
    padding-bottom: 2rem;
}

.search-container {
    margin-bottom: 1.5rem;
}

#search-input {
    width: 100%;
    padding: 0.75rem 1rem;
    border: 1px solid var(--border);
    border-radius: 0.5rem;
    font-size: 1rem;
}

.filters {
    display: flex;
    gap: 0.5rem;
    margin-bottom: 1.5rem;
}

.filter {
    padding: 0.5rem 1rem;
    background: none;
    border: 1px solid var(--border);
    border-radius: 0.5rem;
    cursor: pointer;
    font-size: 0.875rem;
}

.filter.active {
    background-color: var(--primary);
    color: white;
    border-color: var(--primary);
}

.server-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 1.5rem;
}

.server-card {
    background-color: var(--card);
    border-radius: 0.5rem;
    box-shadow: 0 4px 6px var(--shadow);
    padding: 1.5rem;
    transition: transform 0.2s, box-shadow 0.2s;
    cursor: pointer;
}

.server-card:hover {
    transform: translateY(-4px);
    box-shadow: 0 10px 15px rgba(0, 0, 0, 0.1);
}

.server-card h3 {
    margin-bottom: 0.75rem;
    color: var(--primary-dark);
}

.description {
    color: var(--text-light);
    margin-bottom: 1rem;
    font-size: 0.95rem;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
    text-overflow: ellipsis;
}

.stats {
    display: flex;
    gap: 1rem;
    margin-bottom: 1rem;
    font-size: 0.875rem;
    color: var(--text-light);
}

.tags {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
}

.tag {
    font-size: 0.75rem;
    background-color: #f3f4f6;
    padding: 0.25rem 0.5rem;
    border-radius: 0.25rem;
}

.type-tag {
    background-color: var(--secondary);
    color: white;
}

.loading, .error, .no-results {
    grid-column: 1 / -1;
    text-align: center;
    padding: 2rem;
    color: var(--text-light);
}

.error {
    color: #ef4444;
}

/* Modal styles */
.modal {
    display: none;
    position: fixed;
    z-index: 1000;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    overflow: auto;
}

.modal-content {
    background-color: var(--card);
    margin: 5% auto;
    padding: 2rem;
    width: 80%;
    max-width: 800px;
    border-radius: 0.5rem;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0.2);
    position: relative;
}

.close {
    position: absolute;
    top: 1rem;
    right: 1.5rem;
    font-size: 1.5rem;
    font-weight: bold;
    cursor: pointer;
}

#modal-body h2 {
    margin-bottom: 1rem;
    color: var(--primary-dark);
}

#modal-body h3 {
    margin: 1.5rem 0 0.75rem;
    font-size: 1.25rem;
}

.server-description {
    margin-bottom: 1.5rem;
    color: var(--text-light);
}

.tools-list, .resources-list {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
    gap: 1rem;
    margin-bottom: 1rem;
}

.tool, .resource {
    background-color: #f9fafb;
    padding: 1rem;
    border-radius: 0.5rem;
    border: 1px solid var(--border);
}

.tool h4, .resource h4 {
    margin-bottom: 0.5rem;
    color: var(--primary-dark);
    font-size: 1rem;
}

.server-footer {
    margin-top: 2rem;
    display: flex;
    justify-content: flex-end;
}

#try-server {
    background-color: var(--primary);
    color: white;
    border: none;
    padding: 0.75rem 1.5rem;
    border-radius: 0.5rem;
    cursor: pointer;
    font-weight: 600;
    transition: background-color 0.2s;
}

#try-server:hover {
    background-color: var(--primary-dark);
}

/* Responsive adjustments */
@media (max-width: 768px) {
    .server-grid {
        grid-template-columns: 1fr;
    }
    
    .modal-content {
        width: 95%;
        margin: 10% auto;
        padding: 1.5rem;
    }
    
    .tools-list, .resources-list {
        grid-template-columns: 1fr;
    }
}
```

## 5. Adding Server Metadata for Better Visualization

To make your servers appear more professional in the grid, add metadata to your server objects:

```python
# When creating your FastMCP instances, add additional metadata
weather_mcp = FastMCP.from_openapi(
    weather_openapi_spec, 
    client=weather_http_client,
    name="Weather API",
    description="Get weather forecasts and current conditions for locations worldwide",
    tags=["weather", "forecast", "verified"]
)

# Or add metadata to existing servers before mounting
sequential_thinking_proxy.description = "An MCP server for dynamic and reflective problem-solving through structured thinking"
sequential_thinking_proxy.tags = ["thinking", "problem-solving", "official"]

# You can also set usage metrics if you track them
weather_mcp.usage_count = 3457
```

With this implementation, you'll have a professional-looking, responsive UI that dynamically displays all your mounted MCP servers with search, filtering, and detailed views. The design is inspired by mcp.so and the MCP Registry Registry you referenced.

To further enhance the user experience, you could:

1. Add server icons or thumbnails
2. Implement a dark mode toggle
3. Create interactive demos for each tool
4. Add user ratings or feedback mechanisms

Let me know if you'd like implementations for any of these additional features!

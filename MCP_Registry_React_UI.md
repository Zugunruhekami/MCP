*Thinking...*

> Let's create a detailed implementation of the React UI for the MCP registry. This will include multiple components for:
> 
> 1. Main layout/navigation structure
> 2. Dashboard (home page)
> 3. Server listing and details
> 4. Tool exploration
> 5. Server registration/configuration
> 6. Monitoring and error tracking
> 7. User authentication components
> 
> I'll organize this by component, starting with the app structure and then delving into specific pages and components.


# Detailed Implementation of the React UI for the MCP Registry

I'll provide a comprehensive implementation of the React UI for the HSBC MCP Registry, including all major components, pages, and utilities. This implementation uses Material UI for a modern, professional look that aligns with enterprise design standards.

## 1. Project Structure

```
mcp-registry/frontend/
├── public/
│   ├── index.html
│   ├── favicon.ico
│   └── assets/
│       └── logo.svg
├── src/
│   ├── App.js                 # Main application component
│   ├── index.js               # Entry point
│   ├── theme.js               # Material UI theme configuration
│   ├── components/            # Reusable components
│   │   ├── layout/            # Layout components
│   │   ├── common/            # Common UI components
│   │   ├── servers/           # Server-related components
│   │   └── tools/             # Tool-related components
│   ├── pages/                 # Page components
│   ├── services/              # API services
│   ├── hooks/                 # Custom React hooks
│   ├── context/               # React context providers
│   └── utils/                 # Utility functions
```

## 2. Entry Point and Main App

```jsx
// src/index.js
import React from 'react';
import ReactDOM from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import CssBaseline from '@mui/material/CssBaseline';
import { ThemeProvider } from '@mui/material/styles';
import { LocalizationProvider } from '@mui/x-date-pickers';
import { AdapterDateFns } from '@mui/x-date-pickers/AdapterDateFns';
import App from './App';
import theme from './theme';
import { AuthProvider } from './context/AuthContext';
import { NotificationProvider } from './context/NotificationContext';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
  <React.StrictMode>
    <BrowserRouter>
      <ThemeProvider theme={theme}>
        <LocalizationProvider dateAdapter={AdapterDateFns}>
          <CssBaseline />
          <AuthProvider>
            <NotificationProvider>
              <App />
            </NotificationProvider>
          </AuthProvider>
        </LocalizationProvider>
      </ThemeProvider>
    </BrowserRouter>
  </React.StrictMode>
);
```

```jsx
// src/App.js
import React, { useEffect } from 'react';
import { Routes, Route, Navigate } from 'react-router-dom';
import { useAuth } from './hooks/useAuth';
import MainLayout from './components/layout/MainLayout';
import ProtectedRoute from './components/common/ProtectedRoute';
import Login from './pages/Login';
import Dashboard from './pages/Dashboard';
import ServerList from './pages/ServerList';
import ServerDetail from './pages/ServerDetail';
import ServerForm from './pages/ServerForm';
import ToolsExplorer from './pages/ToolsExplorer';
import MonitoringDashboard from './pages/MonitoringDashboard';
import UserManagement from './pages/UserManagement';
import NotFound from './pages/NotFound';
import Notification from './components/common/Notification';

function App() {
  const { isInitialized } = useAuth();

  // Show loading state while checking auth status
  if (!isInitialized) {
    return <div>Loading...</div>;
  }

  return (
    <>
      <Routes>
        {/* Public routes */}
        <Route path="/login" element={<Login />} />

        {/* Protected routes */}
        <Route path="/" element={
          <ProtectedRoute>
            <MainLayout />
          </ProtectedRoute>
        }>
          <Route index element={<Navigate to="/dashboard" replace />} />
          <Route path="dashboard" element={<Dashboard />} />
          <Route path="servers" element={<ServerList />} />
          <Route path="servers/new" element={<ServerForm />} />
          <Route path="servers/:serverId" element={<ServerDetail />} />
          <Route path="servers/:serverId/edit" element={<ServerForm />} />
          <Route path="tools" element={<ToolsExplorer />} />
          <Route path="monitoring" element={<MonitoringDashboard />} />
          <Route path="users" element={
            <ProtectedRoute requiredRole="admin">
              <UserManagement />
            </ProtectedRoute>
          } />
        </Route>

        {/* Catch-all route */}
        <Route path="*" element={<NotFound />} />
      </Routes>
      <Notification />
    </>
  );
}

export default App;
```

## 3. Theme Configuration

```jsx
// src/theme.js
import { createTheme } from '@mui/material/styles';

// HSBC brand colors
const primary = {
  main: '#db0011',
  light: '#e33a47',
  dark: '#99000c',
  contrastText: '#fff',
};

const secondary = {
  main: '#333333',
  light: '#5c5c5c',
  dark: '#232323',
  contrastText: '#fff',
};

const theme = createTheme({
  palette: {
    primary,
    secondary,
    background: {
      default: '#f5f5f5',
      paper: '#ffffff',
    },
  },
  typography: {
    fontFamily: '"Open Sans", "Helvetica", "Arial", sans-serif',
    h1: {
      fontWeight: 600,
    },
    h2: {
      fontWeight: 600,
    },
    h3: {
      fontWeight: 600,
    },
    h4: {
      fontWeight: 600,
    },
    h5: {
      fontWeight: 600,
    },
    h6: {
      fontWeight: 600,
    },
  },
  shape: {
    borderRadius: 8,
  },
  components: {
    MuiButton: {
      styleOverrides: {
        root: {
          textTransform: 'none',
          fontWeight: 600,
        },
      },
    },
    MuiCard: {
      styleOverrides: {
        root: {
          boxShadow: '0px 4px 20px rgba(0, 0, 0, 0.05)',
        },
      },
    },
    MuiChip: {
      styleOverrides: {
        root: {
          fontWeight: 500,
        },
      },
    },
  },
});

export default theme;
```

## 4. Context Providers

### 4.1 Authentication Context

```jsx
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import authService from '../services/authService';

export const AuthContext = createContext(null);

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [isInitialized, setIsInitialized] = useState(false);
  const navigate = useNavigate();

  useEffect(() => {
    const initAuth = async () => {
      try {
        const token = localStorage.getItem('token');
        if (token) {
          // Validate token and get user info
          const currentUser = await authService.getCurrentUser();
          setUser(currentUser);
          setIsAuthenticated(true);
        }
      } catch (error) {
        console.error('Failed to initialize auth:', error);
        // Clear invalid tokens
        localStorage.removeItem('token');
        localStorage.removeItem('refreshToken');
      } finally {
        setIsInitialized(true);
      }
    };

    initAuth();
  }, []);

  const login = async (email, password) => {
    try {
      const response = await authService.login(email, password);
      localStorage.setItem('token', response.token);
      localStorage.setItem('refreshToken', response.refreshToken);
      setUser(response.user);
      setIsAuthenticated(true);
      return { success: true };
    } catch (error) {
      console.error('Login failed:', error);
      return { 
        success: false, 
        error: error.response?.data?.message || 'Login failed. Please check your credentials.' 
      };
    }
  };

  const logout = () => {
    localStorage.removeItem('token');
    localStorage.removeItem('refreshToken');
    setUser(null);
    setIsAuthenticated(false);
    navigate('/login');
  };

  const hasRole = (role) => {
    if (!user || !user.roles) return false;
    return user.roles.includes(role);
  };

  return (
    <AuthContext.Provider value={{ 
      user, 
      isAuthenticated, 
      isInitialized,
      login, 
      logout,
      hasRole
    }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### 4.2 Notification Context

```jsx
// src/context/NotificationContext.js
import React, { createContext, useState } from 'react';

export const NotificationContext = createContext(null);

export const NotificationProvider = ({ children }) => {
  const [notification, setNotification] = useState(null);

  // Show a notification
  const showNotification = (message, severity = 'info', duration = 5000) => {
    setNotification({ message, severity, duration });
  };

  // Clear the current notification
  const clearNotification = () => {
    setNotification(null);
  };

  // Helper methods for common notification types
  const showSuccess = (message, duration) => {
    showNotification(message, 'success', duration);
  };

  const showError = (message, duration) => {
    showNotification(message, 'error', duration);
  };

  const showWarning = (message, duration) => {
    showNotification(message, 'warning', duration);
  };

  const showInfo = (message, duration) => {
    showNotification(message, 'info', duration);
  };

  return (
    <NotificationContext.Provider value={{ 
      notification,
      showNotification,
      clearNotification,
      showSuccess,
      showError,
      showWarning,
      showInfo
    }}>
      {children}
    </NotificationContext.Provider>
  );
};
```

## 5. Custom Hooks

```jsx
// src/hooks/useAuth.js
import { useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

export const useAuth = () => useContext(AuthContext);
```

```jsx
// src/hooks/useNotification.js
import { useContext } from 'react';
import { NotificationContext } from '../context/NotificationContext';

export const useNotification = () => useContext(NotificationContext);
```

```jsx
// src/hooks/useServers.js
import { useState, useCallback } from 'react';
import serverService from '../services/serverService';
import { useNotification } from './useNotification';

export const useServers = () => {
  const [servers, setServers] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const { showError, showSuccess } = useNotification();

  const fetchServers = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const data = await serverService.getServers();
      setServers(data);
      return data;
    } catch (err) {
      const message = err.response?.data?.message || 'Failed to fetch servers';
      setError(message);
      showError(message);
      return [];
    } finally {
      setLoading(false);
    }
  }, [showError]);

  const getServer = useCallback(async (id) => {
    setLoading(true);
    setError(null);
    try {
      const data = await serverService.getServer(id);
      return data;
    } catch (err) {
      const message = err.response?.data?.message || 'Failed to fetch server details';
      setError(message);
      showError(message);
      return null;
    } finally {
      setLoading(false);
    }
  }, [showError]);

  const createServer = useCallback(async (serverData) => {
    setLoading(true);
    setError(null);
    try {
      const data = await serverService.createServer(serverData);
      showSuccess('Server created successfully');
      return data;
    } catch (err) {
      const message = err.response?.data?.message || 'Failed to create server';
      setError(message);
      showError(message);
      return null;
    } finally {
      setLoading(false);
    }
  }, [showError, showSuccess]);

  const updateServer = useCallback(async (id, serverData) => {
    setLoading(true);
    setError(null);
    try {
      const data = await serverService.updateServer(id, serverData);
      showSuccess('Server updated successfully');
      return data;
    } catch (err) {
      const message = err.response?.data?.message || 'Failed to update server';
      setError(message);
      showError(message);
      return null;
    } finally {
      setLoading(false);
    }
  }, [showError, showSuccess]);

  const deleteServer = useCallback(async (id) => {
    setLoading(true);
    setError(null);
    try {
      await serverService.deleteServer(id);
      showSuccess('Server deleted successfully');
      return true;
    } catch (err) {
      const message = err.response?.data?.message || 'Failed to delete server';
      setError(message);
      showError(message);
      return false;
    } finally {
      setLoading(false);
    }
  }, [showError, showSuccess]);

  return {
    servers,
    loading,
    error,
    fetchServers,
    getServer,
    createServer,
    updateServer,
    deleteServer
  };
};
```

## 6. API Services

```jsx
// src/services/apiClient.js
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:8000/api';

const apiClient = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Add a request interceptor to include auth token
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Add a response interceptor to handle token refresh
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;
    
    // If error is unauthorized and we haven't tried refreshing the token yet
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      
      try {
        // Try to refresh the token
        const refreshToken = localStorage.getItem('refreshToken');
        if (!refreshToken) {
          throw new Error('No refresh token available');
        }
        
        const response = await axios.post(`${API_URL}/auth/refresh`, {
          refreshToken,
        });
        
        if (response.data.token) {
          localStorage.setItem('token', response.data.token);
          
          // Retry the original request with the new token
          originalRequest.headers.Authorization = `Bearer ${response.data.token}`;
          return axios(originalRequest);
        }
      } catch (refreshError) {
        // If refresh token is invalid, log the user out
        localStorage.removeItem('token');
        localStorage.removeItem('refreshToken');
        window.location.href = '/login';
        return Promise.reject(refreshError);
      }
    }
    
    return Promise.reject(error);
  }
);

export default apiClient;
```

```jsx
// src/services/authService.js
import apiClient from './apiClient';

const authService = {
  login: async (email, password) => {
    const response = await apiClient.post('/auth/login', { email, password });
    return response.data;
  },
  
  logout: async () => {
    try {
      await apiClient.post('/auth/logout');
    } catch (error) {
      console.error('Logout error:', error);
    }
    localStorage.removeItem('token');
    localStorage.removeItem('refreshToken');
  },
  
  getCurrentUser: async () => {
    const response = await apiClient.get('/auth/me');
    return response.data;
  },
  
  refreshToken: async (refreshToken) => {
    const response = await apiClient.post('/auth/refresh', { refreshToken });
    return response.data;
  },
  
  register: async (userData) => {
    const response = await apiClient.post('/auth/register', userData);
    return response.data;
  }
};

export default authService;
```

```jsx
// src/services/serverService.js
import apiClient from './apiClient';

const serverService = {
  getServers: async () => {
    const response = await apiClient.get('/servers');
    return response.data;
  },
  
  getServer: async (id) => {
    const response = await apiClient.get(`/servers/${id}`);
    return response.data;
  },
  
  getServerTools: async (id) => {
    const response = await apiClient.get(`/servers/${id}/tools`);
    return response.data;
  },
  
  createServer: async (serverData) => {
    const response = await apiClient.post('/servers', serverData);
    return response.data;
  },
  
  updateServer: async (id, serverData) => {
    const response = await apiClient.put(`/servers/${id}`, serverData);
    return response.data;
  },
  
  deleteServer: async (id) => {
    await apiClient.delete(`/servers/${id}`);
  },
  
  getServerStatus: async (id) => {
    const response = await apiClient.get(`/servers/${id}/status`);
    return response.data;
  },
  
  updateServerStatus: async (id, status) => {
    const response = await apiClient.put(`/servers/${id}/status`, { status });
    return response.data;
  },
  
  testTool: async (serverId, toolName, params) => {
    const response = await apiClient.post(`/servers/${serverId}/tools/${toolName}/test`, { params });
    return response.data;
  }
};

export default serverService;
```

```jsx
// src/services/toolService.js
import apiClient from './apiClient';

const toolService = {
  getAllTools: async () => {
    const response = await apiClient.get('/tools');
    return response.data;
  },
  
  searchTools: async (query) => {
    const response = await apiClient.get('/tools/search', { params: { q: query } });
    return response.data;
  },
  
  getToolDetails: async (toolId) => {
    const response = await apiClient.get(`/tools/${toolId}`);
    return response.data;
  }
};

export default toolService;
```

```jsx
// src/services/monitoringService.js
import apiClient from './apiClient';

const monitoringService = {
  getServerHealth: async () => {
    const response = await apiClient.get('/monitoring/health');
    return response.data;
  },
  
  getServerMetrics: async (serverId) => {
    const response = await apiClient.get(`/monitoring/servers/${serverId}/metrics`);
    return response.data;
  },
  
  getErrorLogs: async (serverId) => {
    const response = await apiClient.get(`/monitoring/servers/${serverId}/errors`);
    return response.data;
  },
  
  getToolUsageStats: async () => {
    const response = await apiClient.get('/monitoring/tools/usage');
    return response.data;
  }
};

export default monitoringService;
```

## 7. Layout Components

```jsx
// src/components/layout/MainLayout.js
import React, { useState } from 'react';
import { Outlet } from 'react-router-dom';
import { Box, CssBaseline } from '@mui/material';
import AppHeader from './AppHeader';
import SideNav from './SideNav';

const drawerWidth = 280;

const MainLayout = () => {
  const [mobileOpen, setMobileOpen] = useState(false);

  const handleDrawerToggle = () => {
    setMobileOpen(!mobileOpen);
  };

  return (
    <Box sx={{ display: 'flex', minHeight: '100vh' }}>
      <CssBaseline />
      
      {/* App header */}
      <AppHeader 
        drawerWidth={drawerWidth}
        onDrawerToggle={handleDrawerToggle}
      />
      
      {/* Side navigation */}
      <SideNav
        drawerWidth={drawerWidth}
        mobileOpen={mobileOpen}
        onDrawerToggle={handleDrawerToggle}
      />
      
      {/* Main content */}
      <Box
        component="main"
        sx={{
          flexGrow: 1,
          p: 3,
          width: { sm: `calc(100% - ${drawerWidth}px)` },
          mt: '64px', // Below app bar
          overflow: 'auto'
        }}
      >
        <Outlet />
      </Box>
    </Box>
  );
};

export default MainLayout;
```

```jsx
// src/components/layout/AppHeader.js
import React from 'react';
import { 
  AppBar, 
  Toolbar, 
  Typography, 
  IconButton, 
  Box, 
  Avatar, 
  Menu, 
  MenuItem, 
  Tooltip 
} from '@mui/material';
import MenuIcon from '@mui/icons-material/Menu';
import { useNavigate } from 'react-router-dom';
import { useAuth } from '../../hooks/useAuth';

const AppHeader = ({ drawerWidth, onDrawerToggle }) => {
  const { user, logout } = useAuth();
  const navigate = useNavigate();
  const [anchorEl, setAnchorEl] = React.useState(null);
  
  const handleMenu = (event) => {
    setAnchorEl(event.currentTarget);
  };

  const handleClose = () => {
    setAnchorEl(null);
  };

  const handleProfile = () => {
    handleClose();
    navigate('/profile');
  };

  const handleLogout = () => {
    handleClose();
    logout();
  };

  return (
    <AppBar
      position="fixed"
      sx={{
        width: { sm: `calc(100% - ${drawerWidth}px)` },
        ml: { sm: `${drawerWidth}px` },
        boxShadow: 3,
      }}
    >
      <Toolbar>
        <IconButton
          color="inherit"
          aria-label="open drawer"
          edge="start"
          onClick={onDrawerToggle}
          sx={{ mr: 2, display: { sm: 'none' } }}
        >
          <MenuIcon />
        </IconButton>
        
        <Typography variant="h6" noWrap component="div" sx={{ flexGrow: 1 }}>
          HSBC MCP Registry
        </Typography>
        
        {user && (
          <Box sx={{ display: 'flex', alignItems: 'center' }}>
            <Typography variant="body2" sx={{ mr: 2 }}>
              {user.fullName || user.email}
            </Typography>
            
            <Tooltip title="Account settings">
              <IconButton
                onClick={handleMenu}
                size="small"
                edge="end"
                aria-label="account of current user"
                aria-controls="menu-appbar"
                aria-haspopup="true"
              >
                <Avatar 
                  alt={user.fullName || user.email} 
                  src={user.avatar} 
                  sx={{ width: 36, height: 36 }}
                />
              </IconButton>
            </Tooltip>
            
            <Menu
              id="menu-appbar"
              anchorEl={anchorEl}
              anchorOrigin={{
                vertical: 'bottom',
                horizontal: 'right',
              }}
              keepMounted
              transformOrigin={{
                vertical: 'top',
                horizontal: 'right',
              }}
              open={Boolean(anchorEl)}
              onClose={handleClose}
            >
              <MenuItem onClick={handleProfile}>Profile</MenuItem>
              <MenuItem onClick={handleLogout}>Logout</MenuItem>
            </Menu>
          </Box>
        )}
      </Toolbar>
    </AppBar>
  );
};

export default AppHeader;
```

```jsx
// src/components/layout/SideNav.js
import React from 'react';
import { 
  Drawer, 
  Box, 
  List, 
  ListItem, 
  ListItemButton, 
  ListItemIcon, 
  ListItemText, 
  Divider,
  Typography
} from '@mui/material';
import { useNavigate, useLocation } from 'react-router-dom';
import { useAuth } from '../../hooks/useAuth';

// Icons
import DashboardIcon from '@mui/icons-material/Dashboard';
import StorageIcon from '@mui/icons-material/Storage';
import ConstructionIcon from '@mui/icons-material/Construction';
import MonitorHeartIcon from '@mui/icons-material/MonitorHeart';
import PeopleIcon from '@mui/icons-material/People';
import SettingsIcon from '@mui/icons-material/Settings';
import { ReactComponent as Logo } from '../../assets/hsbc-logo.svg';

const SideNav = ({ drawerWidth, mobileOpen, onDrawerToggle }) => {
  const { hasRole } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();
  
  const isActive = (path) => location.pathname === path;

  const navigationItems = [
    { 
      text: 'Dashboard', 
      icon: <DashboardIcon />, 
      path: '/dashboard',
      requiresRole: null
    },
    { 
      text: 'MCP Servers', 
      icon: <StorageIcon />, 
      path: '/servers',
      requiresRole: null
    },
    { 
      text: 'Tools Explorer', 
      icon: <ConstructionIcon />, 
      path: '/tools',
      requiresRole: null
    },
    { 
      text: 'Monitoring', 
      icon: <MonitorHeartIcon />, 
      path: '/monitoring',
      requiresRole: null
    },
    { 
      text: 'User Management', 
      icon: <PeopleIcon />, 
      path: '/users',
      requiresRole: 'admin'
    },
    { 
      text: 'Settings', 
      icon: <SettingsIcon />, 
      path: '/settings',
      requiresRole: null
    },
  ];

  const drawer = (
    <Box sx={{ height: '100%', display: 'flex', flexDirection: 'column' }}>
      {/* Header with logo */}
      <Box 
        sx={{ 
          p: 2, 
          display: 'flex', 
          alignItems: 'center', 
          justifyContent: 'center',
          borderBottom: '1px solid rgba(0, 0, 0, 0.12)'
        }}
      >
        <Logo width={40} height={40} />
        <Typography 
          variant="h6" 
          component="div" 
          sx={{ ml: 1, fontWeight: 'bold' }}
        >
          MCP Registry
        </Typography>
      </Box>
      
      {/* Navigation list */}
      <List sx={{ flexGrow: 1, pt: 2 }}>
        {navigationItems.map((item) => {
          // Skip items that require roles the user doesn't have
          if (item.requiresRole && !hasRole(item.requiresRole)) {
            return null;
          }

          return (
            <ListItem key={item.text} disablePadding>
              <ListItemButton 
                selected={isActive(item.path)}
                onClick={() => navigate(item.path)}
                sx={{
                  borderRadius: '0 20px 20px 0',
                  mx: 1,
                  my: 0.5,
                  '&.Mui-selected': {
                    bgcolor: 'primary.light',
                    color: 'primary.contrastText',
                    '& .MuiListItemIcon-root': {
                      color: 'primary.contrastText',
                    },
                  },
                  '&:hover': {
                    bgcolor: 'action.hover',
                    '&.Mui-selected': {
                      bgcolor: 'primary.main',
                    },
                  },
                }}
              >
                <ListItemIcon sx={{ 
                  color: isActive(item.path) ? 'primary.contrastText' : 'inherit',
                  minWidth: 40
                }}>
                  {item.icon}
                </ListItemIcon>
                <ListItemText primary={item.text} />
              </ListItemButton>
            </ListItem>
          );
        })}
      </List>
      
      {/* Footer info */}
      <Box sx={{ p: 2, borderTop: '1px solid rgba(0, 0, 0, 0.12)' }}>
        <Typography variant="caption" display="block" color="text.secondary" align="center">
          HSBC AI Markets Team
        </Typography>
        <Typography variant="caption" display="block" color="text.secondary" align="center">
          Version 1.0.0
        </Typography>
      </Box>
    </Box>
  );

  return (
    <Box
      component="nav"
      sx={{ width: { sm: drawerWidth }, flexShrink: { sm: 0 } }}
      aria-label="mailbox folders"
    >
      {/* Mobile drawer */}
      <Drawer
        variant="temporary"
        open={mobileOpen}
        onClose={onDrawerToggle}
        ModalProps={{ keepMounted: true }}
        sx={{
          display: { xs: 'block', sm: 'none' },
          '& .MuiDrawer-paper': { 
            boxSizing: 'border-box', 
            width: drawerWidth 
          },
        }}
      >
        {drawer}
      </Drawer>
      
      {/* Desktop drawer */}
      <Drawer
        variant="permanent"
        sx={{
          display: { xs: 'none', sm: 'block' },
          '& .MuiDrawer-paper': { 
            boxSizing: 'border-box', 
            width: drawerWidth,
            borderRight: '1px solid rgba(0, 0, 0, 0.12)',
            boxShadow: '4px 0px 10px rgba(0, 0, 0, 0.05)'
          },
        }}
        open
      >
        {drawer}
      </Drawer>
    </Box>
  );
};

export default SideNav;
```

## 8. Common Components

```jsx
// src/components/common/ProtectedRoute.js
import React from 'react';
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '../../hooks/useAuth';

const ProtectedRoute = ({ children, requiredRole = null }) => {
  const { isAuthenticated, hasRole } = useAuth();
  const location = useLocation();

  // Not authenticated, redirect to login
  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  // Role check if specified
  if (requiredRole && !hasRole(requiredRole)) {
    return <Navigate to="/unauthorized" replace />;
  }

  // All checks passed, render the children
  return children;
};

export default ProtectedRoute;
```

```jsx
// src/components/common/Notification.js
import React from 'react';
import { Snackbar, Alert } from '@mui/material';
import { useNotification } from '../../hooks/useNotification';

const Notification = () => {
  const { notification, clearNotification } = useNotification();

  if (!notification) return null;

  const { message, severity, duration } = notification;

  return (
    <Snackbar
      open={Boolean(notification)}
      autoHideDuration={duration}
      onClose={clearNotification}
      anchorOrigin={{ vertical: 'bottom', horizontal: 'right' }}
    >
      <Alert 
        onClose={clearNotification} 
        severity={severity} 
        sx={{ width: '100%' }}
        variant="filled"
        elevation={6}
      >
        {message}
      </Alert>
    </Snackbar>
  );
};

export default Notification;
```

```jsx
// src/components/common/PageHeader.js
import React from 'react';
import { 
  Box, 
  Typography, 
  Breadcrumbs, 
  Link, 
  Divider 
} from '@mui/material';
import { Link as RouterLink } from 'react-router-dom';
import NavigateNextIcon from '@mui/icons-material/NavigateNext';

const PageHeader = ({ 
  title, 
  description, 
  breadcrumbs = [], 
  actions = null 
}) => {
  return (
    <Box sx={{ mb: 4 }}>
      {/* Breadcrumbs */}
      {breadcrumbs.length > 0 && (
        <Breadcrumbs 
          separator={<NavigateNextIcon fontSize="small" />} 
          aria-label="breadcrumb"
          sx={{ mb: 2 }}
        >
          {breadcrumbs.map((crumb, index) => {
            const isLast = index === breadcrumbs.length - 1;
            
            if (isLast || !crumb.path) {
              return (
                <Typography key={index} color="text.primary">
                  {crumb.label}
                </Typography>
              );
            }
            
            return (
              <Link 
                key={index} 
                component={RouterLink} 
                to={crumb.path}
                underline="hover" 
                color="inherit"
              >
                {crumb.label}
              </Link>
            );
          })}
        </Breadcrumbs>
      )}
      
      {/* Header with actions */}
      <Box 
        sx={{ 
          display: 'flex', 
          justifyContent: 'space-between', 
          alignItems: 'center',
          mb: 1
        }}
      >
        <Typography variant="h4" component="h1">
          {title}
        </Typography>
        
        {actions && (
          <Box>
            {actions}
          </Box>
        )}
      </Box>
      
      {/* Description */}
      {description && (
        <Typography variant="body1" color="text.secondary">
          {description}
        </Typography>
      )}
      
      <Divider sx={{ mt: 2, mb: 3 }} />
    </Box>
  );
};

export default PageHeader;
```

```jsx
// src/components/common/LoadingOverlay.js
import React from 'react';
import { Backdrop, CircularProgress, Typography, Box } from '@mui/material';

const LoadingOverlay = ({ open, message = 'Loading...' }) => {
  return (
    <Backdrop
      sx={{ 
        color: '#fff', 
        zIndex: (theme) => theme.zIndex.drawer + 1,
        flexDirection: 'column'
      }}
      open={open}
    >
      <CircularProgress color="inherit" />
      <Box sx={{ mt: 2 }}>
        <Typography variant="h6">{message}</Typography>
      </Box>
    </Backdrop>
  );
};

export default LoadingOverlay;
```

## 9. Server-Related Components

```jsx
// src/components/servers/ServerCard.js
import React from 'react';
import { 
  Card, 
  CardContent, 
  Typography, 
  Box, 
  Chip, 
  Avatar,
  CardActionArea
} from '@mui/material';
import StorageIcon from '@mui/icons-material/Storage';
import ConstructionIcon from '@mui/icons-material/Construction';
import { useNavigate } from 'react-router-dom';

// Status chip colors
const statusColors = {
  online: 'success',
  offline: 'error',
  error: 'error',
  degraded: 'warning',
  unknown: 'default'
};

const ServerCard = ({ server }) => {
  const navigate = useNavigate();
  
  const handleClick = () => {
    navigate(`/servers/${server.id}`);
  };
  
  return (
    <Card 
      sx={{ 
        height: '100%', 
        display: 'flex', 
        flexDirection: 'column',
        transition: 'transform 0.2s, box-shadow 0.2s',
        '&:hover': {
          transform: 'translateY(-4px)',
          boxShadow: 6,
        }
      }}
    >
      <CardActionArea onClick={handleClick} sx={{ height: '100%' }}>
        <CardContent sx={{ height: '100%', p: 3 }}>
          {/* Header with status */}
          <Box sx={{ 
            display: 'flex', 
            justifyContent: 'space-between', 
            mb: 2, 
            alignItems: 'flex-start' 
          }}>
            <Box sx={{ display: 'flex', alignItems: 'center' }}>
              <Avatar 
                sx={{ 
                  bgcolor: 'primary.main', 
                  width: 40, 
                  height: 40,
                  mr: 1.5,
                }}
              >
                <StorageIcon />
              </Avatar>
              <Box>
                <Typography variant="h6" component="div" noWrap>
                  {server.name}
                </Typography>
                <Typography 
                  variant="body2" 
                  color="text.secondary" 
                  sx={{ mt: 0.5 }}
                >
                  {server.owner}
                </Typography>
              </Box>
            </Box>
            <Chip 
              label={server.status || 'unknown'} 
              color={statusColors[server.status] || 'default'} 
              size="small" 
              sx={{ ml: 1 }}
            />
          </Box>
          
          {/* Description */}
          <Typography 
            variant="body2" 
            color="text.secondary" 
            sx={{ 
              mb: 2,
              display: '-webkit-box',
              overflow: 'hidden',
              WebkitBoxOrient: 'vertical',
              WebkitLineClamp: 3,
              height: '4.5em', // Approximately 3 lines
            }}
          >
            {server.description}
          </Typography>
          
          {/* Footer with tools count */}
          <Box 
            sx={{ 
              display: 'flex', 
              alignItems: 'center', 
              mt: 'auto', 
              pt: 1,
              borderTop: '1px solid', 
              borderColor: 'divider' 
            }}
          >
            <ConstructionIcon 
              fontSize="small" 
              color="action" 
              sx={{ mr: 0.75 }}
            />
            <Typography variant="body2" color="text.secondary">
              {server.tools_count} tools
            </Typography>
            <Typography 
              variant="body2" 
              color="text.secondary" 
              sx={{ ml: 'auto' }}
            >
              v{server.version}
            </Typography>
          </Box>
        </CardContent>
      </CardActionArea>
    </Card>
  );
};

export default ServerCard;
```

```jsx
// src/components/servers/ServerToolsList.js
import React, { useState } from 'react';
import { 
  Box, 
  Typography, 
  List, 
  ListItem, 
  ListItemText, 
  ListItemIcon,
  ListItemButton, 
  Collapse, 
  Paper, 
  InputBase, 
  IconButton,
  Divider,
  Chip,
  Tooltip
} from '@mui/material';
import ExpandLessIcon from '@mui/icons-material/ExpandLess';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import BuildIcon from '@mui/icons-material/Build';
import SearchIcon from '@mui/icons-material/Search';
import ClearIcon from '@mui/icons-material/Clear';
import InfoIcon from '@mui/icons-material/Info';

const ServerToolsList = ({ tools, onToolSelect }) => {
  const [expandedTool, setExpandedTool] = useState(null);
  const [searchQuery, setSearchQuery] = useState('');
  
  const handleToggle = (toolName) => {
    setExpandedTool(expandedTool === toolName ? null : toolName);
  };
  
  const handleSearch = (e) => {
    setSearchQuery(e.target.value);
  };
  
  const clearSearch = () => {
    setSearchQuery('');
  };
  
  // Filter tools based on search query
  const filteredTools = tools.filter(tool => {
    const query = searchQuery.toLowerCase();
    return (
      tool.name.toLowerCase().includes(query) ||
      tool.description.toLowerCase().includes(query)
    );
  });
  
  return (
    <Box>
      {/* Search bar */}
      <Paper
        component="form"
        sx={{ p: '2px 4px', display: 'flex', alignItems: 'center', mb: 2 }}
      >
        <IconButton sx={{ p: '10px' }} aria-label="search">
          <SearchIcon />
        </IconButton>
        <InputBase
          sx={{ ml: 1, flex: 1 }}
          placeholder="Search tools..."
          value={searchQuery}
          onChange={handleSearch}
        />
        {searchQuery && (
          <IconButton sx={{ p: '10px' }} aria-label="clear" onClick={clearSearch}>
            <ClearIcon />
          </IconButton>
        )}
      </Paper>
      
      {/* Tool count */}
      <Typography variant="body2" color="text.secondary" sx={{ mb: 2 }}>
        {filteredTools.length} tool{filteredTools.length !== 1 ? 's' : ''} found
      </Typography>
      
      {/* Tools list */}
      <List component="div" disablePadding>
        {filteredTools.map((tool) => (
          <React.Fragment key={tool.name}>
            <Paper sx={{ mb: 1 }}>
              <ListItemButton 
                alignItems="flex-start"
                onClick={() => handleToggle(tool.name)}
                sx={{ borderLeft: 3, borderColor: 'primary.main' }}
              >
                <ListItemIcon>
                  <BuildIcon color="primary" />
                </ListItemIcon>
                <ListItemText
                  primary={
                    <Box sx={{ display: 'flex', alignItems: 'center' }}>
                      <Typography variant="subtitle1" component="span">
                        {tool.name}
                      </Typography>
                      <Tooltip title="View tool details">
                        <IconButton 
                          size="small" 
                          onClick={(e) => {
                            e.stopPropagation();
                            onToolSelect && onToolSelect(tool);
                          }}
                          sx={{ ml: 1 }}
                        >
                          <InfoIcon fontSize="small" />
                        </IconButton>
                      </Tooltip>
                    </Box>
                  }
                  secondary={
                    <Typography
                      variant="body2"
                      color="text.secondary"
                      sx={{
                        display: '-webkit-box',
                        overflow: 'hidden',
                        WebkitBoxOrient: 'vertical',
                        WebkitLineClamp: 2,
                      }}
                    >
                      {tool.description}
                    </Typography>
                  }
                />
                {expandedTool === tool.name ? <ExpandLessIcon /> : <ExpandMoreIcon />}
              </ListItemButton>
              
              <Collapse in={expandedTool === tool.name} timeout="auto" unmountOnExit>
                <Box sx={{ p: 2, pt: 0 }}>
                  <Divider sx={{ my: 1 }} />
                  
                  {/* Parameters section */}
                  <Typography variant="subtitle2" gutterBottom>
                    Parameters:
                  </Typography>
                  
                  {tool.parameters && tool.parameters.properties ? (
                    <List dense disablePadding>
                      {Object.entries(tool.parameters.properties).map(([name, schema]) => (
                        <ListItem key={name} sx={{ py: 0.5 }}>
                          <ListItemText
                            primary={
                              <Box sx={{ display: 'flex', alignItems: 'center' }}>
                                <Typography variant="body2" component="span" fontWeight="medium">
                                  {name}
                                </Typography>
                                <Chip 
                                  label={schema.type} 
                                  size="small" 
                                  sx={{ ml: 1, fontSize: '0.7rem', height: 20 }} 
                                />
                                {tool.parameters.required?.includes(name) && (
                                  <Chip 
                                    label="required" 
                                    size="small" 
                                    color="secondary"
                                    sx={{ ml: 1, fontSize: '0.7rem', height: 20 }} 
                                  />
                                )}
                              </Box>
                            }
                            secondary={schema.description || 'No description available'}
                            secondaryTypographyProps={{ variant: 'caption' }}
                          />
                        </ListItem>
                      ))}
                    </List>
                  ) : (
                    <Typography variant="body2" color="text.secondary">
                      No parameters
                    </Typography>
                  )}
                  
                  {/* Response schema section */}
                  {tool.responseSchema && (
                    <>
                      <Typography variant="subtitle2" gutterBottom sx={{ mt: 2 }}>
                        Response:
                      </Typography>
                      <Typography variant="body2" color="text.secondary">
                        {tool.responseDescription || 'No description available'}
                      </Typography>
                    </>
                  )}
                </Box>
              </Collapse>
            </Paper>
          </React.Fragment>
        ))}
        
        {filteredTools.length === 0 && (
          <Typography variant="body2" color="text.secondary" align="center" sx={{ mt: 4 }}>
            No tools found matching your search criteria
          </Typography>
        )}
      </List>
    </Box>
  );
};

export default ServerToolsList;
```

```jsx
// src/components/servers/ServerConfigCard.js
import React, { useState } from 'react';
import { 
  Card, 
  CardContent, 
  Typography, 
  Box, 
  Divider, 
  Collapse, 
  IconButton, 
  Button,
  Table,
  TableBody,
  TableCell,
  TableContainer,
  TableRow,
  Chip,
  Dialog,
  DialogTitle,
  DialogContent,
  DialogActions
} from '@mui/material';
import ExpandMoreIcon from '@mui/icons-material/ExpandMore';
import ExpandLessIcon from '@mui/icons-material/ExpandLess';
import ContentCopyIcon from '@mui/icons-material/ContentCopy';
import EditIcon from '@mui/icons-material/Edit';
import { Prism as SyntaxHighlighter } from 'react-syntax-highlighter';
import { vscDarkPlus } from 'react-syntax-highlighter/dist/esm/styles/prism';
import { useNotification } from '../../hooks/useNotification';

const ServerConfigCard = ({ server, onEdit }) => {
  const [expanded, setExpanded] = useState(false);
  const [showJsonConfig, setShowJsonConfig] = useState(false);
  const { showSuccess } = useNotification();
  
  const handleExpandClick = () => {
    setExpanded(!expanded);
  };
  
  const handleCopyConfig = () => {
    const configText = JSON.stringify(server.config, null, 2);
    navigator.clipboard.writeText(configText);
    showSuccess("Configuration copied to clipboard");
  };
  
  const handleShowJson = () => {
    setShowJsonConfig(true);
  };
  
  const handleCloseJson = () => {
    setShowJsonConfig(false);
  };
  
  // Helper to mask sensitive values in config
  const getMaskedValue = (key, value) => {
    const sensitiveKeys = ['api_token', 'password', 'secret', 'key', 'token'];
    if (typeof value === 'string' && sensitiveKeys.some(k => key.toLowerCase().includes(k))) {
      return '••••••••••••••••';
    }
    return value;
  };
  
  return (
    <Card>
      <CardContent>
        <Box sx={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
          <Typography variant="h6" component="div">
            Server Configuration
          </Typography>
          
          <Box>
            <Button 
              size="small" 
              startIcon={<EditIcon />}
              onClick={onEdit}
              sx={{ mr: 1 }}
            >
              Edit
            </Button>
            <IconButton
              onClick={handleExpandClick}
              aria-expanded={expanded}
              aria-label="show more"
            >
              {expanded ? <ExpandLessIcon /> : <ExpandMoreIcon />}
            </IconButton>
          </Box>
        </Box>
        
        <Box sx={{ mt: 2 }}>
          <TableContainer>
            <Table size="small">
              <TableBody>
                <TableRow>
                  <TableCell component="th" scope="row" sx={{ width: '30%', fontWeight: 'medium' }}>
                    Transport
                  </TableCell>
                  <TableCell>
                    <Chip 
                      label={server.transport} 
                      size="small"
                      color="primary"
                      variant="outlined"
                    />
                  </TableCell>
                </TableRow>
                
                <TableRow>
                  <TableCell component="th" scope="row" sx={{ fontWeight: 'medium' }}>
                    Status
                  </TableCell>
                  <TableCell>
                    <Chip 
                      label={server.status} 
                      size="small"
                      color={
                        server.status === 'online' ? 'success' : 
                        server.status === 'offline' ? 'error' : 'warning'
                      }
                    />
                  </TableCell>
                </TableRow>
                
                <TableRow>
                  <TableCell component="th" scope="row" sx={{ fontWeight: 'medium' }}>
                    Version
                  </TableCell>
                  <TableCell>{server.version}</TableCell>
                </TableRow>
                
                <TableRow>
                  <TableCell component="th" scope="row" sx={{ fontWeight: 'medium' }}>
                    Owner
                  </TableCell>
                  <TableCell>{server.owner}</TableCell>
                </TableRow>
                
                <TableRow>
                  <TableCell component="th" scope="row" sx={{ fontWeight: 'medium' }}>
                    Created
                  </TableCell>
                  <TableCell>
                    {new Date(server.created_at).toLocaleString()}
                  </TableCell>
                </TableRow>
                
                {server.updated_at && (
                  <TableRow>
                    <TableCell component="th" scope="row" sx={{ fontWeight: 'medium' }}>
                      Last Updated
                    </TableCell>
                    <TableCell>
                      {new Date(server.updated_at).toLocaleString()}
                    </TableCell>
                  </TableRow>
                )}
              </TableBody>
            </Table>
          </TableContainer>
        </Box>
        
        <Collapse in={expanded} timeout="auto" unmountOnExit>
          <Divider sx={{ my: 2 }} />
          
          <Box sx={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', mb: 2 }}>
            <Typography variant="subtitle1">
              Detailed Configuration
            </Typography>
            
            <Box>
              <Button 
                size="small" 
                onClick={handleShowJson}
                sx={{ mr: 1 }}
              >
                View JSON
              </Button>
              <Button 
                size="small" 
                startIcon={<ContentCopyIcon />}
                onClick={handleCopyConfig}
              >
                Copy
              </Button>
            </Box>
          </Box>
          
          <TableContainer>
            <Table size="small">
              <TableBody>
                {server.config && Object.entries(server.config).map(([key, value]) => {
                  if (key === 'env' && typeof value === 'object') {
                    return (
                      <TableRow key={key}>
                        <TableCell component="th" scope="row" sx={{ width: '30%', fontWeight: 'medium' }}>
                          Environment Variables
                        </TableCell>
                        <TableCell>
                          <TableContainer>
                            <Table size="small">
                              <TableBody>
                                {Object.entries(value).map(([envKey, envValue]) => (
                                  <TableRow key={envKey}>
                                    <TableCell component="th" scope="row" sx={{ fontWeight: 'medium' }}>
                                      {envKey}
                                    </TableCell>
                                    <TableCell>
                                      {getMaskedValue(envKey, envValue)}
                                    </TableCell>
                                  </TableRow>
                                ))}
                              </TableBody>
                            </Table>
                          </TableContainer>
                        </TableCell>
                      </TableRow>
                    );
                  }
                  
                  if (key === 'args' && Array.isArray(value)) {
                    return (
                      <TableRow key={key}>
                        <TableCell component="th" scope="row" sx={{ fontWeight: 'medium' }}>
                          Arguments
                        </TableCell>
                        <TableCell>
                          {value.join(' ')}
                        </TableCell>
                      </TableRow>
                    );
                  }
                  
                  return (
                    <TableRow key={key}>
                      <TableCell component="th" scope="row" sx={{ fontWeight: 'medium' }}>
                        {key}
                      </TableCell>
                      <TableCell>
                        {typeof value === 'object' 
                          ? JSON.stringify(value) 
                          : getMaskedValue(key, value)}
                      </TableCell>
                    </TableRow>
                  );
                })}
              </TableBody>
            </Table>
          </TableContainer>
        </Collapse>
      </CardContent>
      
      {/* JSON Config Dialog */}
      <Dialog
        open={showJsonConfig}
        onClose={handleCloseJson}
        maxWidth="md"
        fullWidth
      >
        <DialogTitle>Server Configuration (JSON)</DialogTitle>
        <DialogContent dividers>
          <SyntaxHighlighter 
            language="json" 
            style={vscDarkPlus}
            customStyle={{ borderRadius: 8 }}
          >
            {JSON.stringify(server.config, null, 2)}
          </SyntaxHighlighter>
        </DialogContent>
        <DialogActions>
          <Button onClick={handleCopyConfig} startIcon={<ContentCopyIcon />}>
            Copy
          </Button>
          <Button onClick={handleCloseJson}>
            Close
          </Button>
        </DialogActions>
      </Dialog>
    </Card>
  );
};

export default ServerConfigCard;
```

## 10. Key Pages

### 10.1 Dashboard Page

```jsx
// src/pages/Dashboard.jsx
import React, { useState, useEffect } from 'react';
import { 
  Container, 
  Grid, 
  Paper, 
  Typography, 
  Box, 
  Button, 
  CircularProgress,
  Card,
  CardContent,
  IconButton,
  LinearProgress
} from '@mui/material';
import AddIcon from '@mui/icons-material/Add';
import RefreshIcon from '@mui/icons-material/Refresh';
import ServerCard from '../components/servers/ServerCard';
import PageHeader from '../components/common/PageHeader';
import { useServers } from '../hooks/useServers';
import { useNavigate } from 'react-router-dom';
import monitoringService from '../services/monitoringService';

// Status summary component
const StatusSummary = ({ serverStatus }) => {
  // Count servers by status
  const statusCounts = {
    online: 0,
    offline: 0,
    error: 0,
    degraded: 0,
    unknown: 0
  };
  
  Object.values(serverStatus).forEach(status => {
    statusCounts[status] = (statusCounts[status] || 0) + 1;
  });
  
  // Calculate overall health percentage
  const totalServers = Object.values(statusCounts).reduce((total, count) => total + count, 0);
  const healthyServers = statusCounts.online;
  const healthPercentage = totalServers > 0 ? Math.round((healthyServers / totalServers) * 100) : 0;
  
  return (
    <Card>
      <CardContent>
        <Typography variant="h6" gutterBottom>
          System Health
        </Typography>
        
        <Box sx={{ display: 'flex', alignItems: 'center', mb: 2 }}>
          <Box sx={{ position: 'relative', display: 'inline-flex', mr: 2 }}>
            <CircularProgress 
              variant="determinate" 
              value={healthPercentage} 
              size={70}
              thickness={5}
              color={
                healthPercentage > 80 ? 'success' :
                healthPercentage > 50 ? 'warning' : 'error'
              }
            />
            <Box
              sx={{
                top: 0,
                left: 0,
                bottom: 0,
                right: 0,
                position: 'absolute',
                display: 'flex',
                alignItems: 'center',
                justifyContent: 'center',
              }}
            >
              <Typography variant="caption" component="div" fontWeight="bold">
                {`${healthPercentage}%`}
              </Typography>
            </Box>
          </Box>
          
          <Box sx={{ flexGrow: 1 }}>
            <Typography variant="body2" color="text.secondary">
              {healthyServers} of {totalServers} servers online
            </Typography>
            <LinearProgress 
              variant="determinate" 
              value={healthPercentage} 
              sx={{ mt: 1, height: 8, borderRadius: 4 }}
              color={
                healthPercentage > 80 ? 'success' :
                healthPercentage > 50 ? 'warning' : 'error'
              }
            />
          </Box>
        </Box>
        
        <Box sx={{ display: 'flex', flexWrap: 'wrap' }}>
          <StatusChip label="Online" count={statusCounts.online} color="success" />
          <StatusChip label="Degraded" count={statusCounts.degraded} color="warning" />
          <StatusChip label="Offline" count={statusCounts.offline} color="error" />
          <StatusChip label="Error" count={statusCounts.error} color="error" />
          <StatusChip label="Unknown" count={statusCounts.unknown} color="default" />
        </Box>
      </CardContent>
    </Card>
  );
};

// Status chip component
const StatusChip = ({ label, count, color }) => (
  <Box 
    sx={{ 
      bgcolor: `${color}.light`, 
      color: `${color}.dark`, 
      borderRadius: 2,
      px: 2,
      py: 1,
      mr: 1,
      mb: 1,
      display: 'flex',
      flexDirection: 'column',
      alignItems: 'center',
      minWidth: 80
    }}
  >
    <Typography variant="h6" component="div" fontWeight="bold">
      {count}
    </Typography>
    <Typography variant="caption">
      {label}
    </Typography>
  </Box>
);

// Tool usage stats component
const ToolUsageStats = ({ toolUsage }) => {
  // Sort tools by usage count
  const sortedTools = [...toolUsage].sort((a, b) => b.count - a.count).slice(0, 5);
  
  return (
    <Card>
      <CardContent>
        <Typography variant="h6" gutterBottom>
          Top Tools
        </Typography>
        
        {sortedTools.length > 0 ? (
          <Box>
            {sortedTools.map((tool, index) => (
              <Box key={tool.name} sx={{ mb: 1 }}>
                <Box sx={{ display: 'flex', justifyContent: 'space-between' }}>
                  <Typography variant="body2">{tool.name}</Typography>
                  <Typography variant="body2">{tool.count} calls</Typography>
                </Box>
                <LinearProgress 
                  variant="determinate" 
                  value={(tool.count / sortedTools[0].count) * 100}
                  sx={{ height: 6, borderRadius: 3, mt: 0.5 }}
                  color="primary"
                />
              </Box>
            ))}
          </Box>
        ) : (
          <Typography variant="body2" color="text.secondary">
            No tool usage data available
          </Typography>
        )}
      </CardContent>
    </Card>
  );
};

const Dashboard = () => {
  const { servers, loading, fetchServers } = useServers();
  const [serverStatus, setServerStatus] = useState({});
  const [toolUsage, setToolUsage] = useState([]);
  const [metricsLoading, setMetricsLoading] = useState(true);
  const navigate = useNavigate();
  
  useEffect(() => {
    fetchServers();
    fetchMetrics();
  }, [fetchServers]);
  
  const fetchMetrics = async () => {
    setMetricsLoading(true);
    try {
      // Get server health data
      const healthData = await monitoringService.getServerHealth();
      const statuses = {};
      
      Object.entries(healthData).forEach(([serverId, data]) => {
        statuses[serverId] = data.status;
      });
      
      setServerStatus(statuses);
      
      // Get tool usage stats
      const usageData = await monitoringService.getToolUsageStats();
      setToolUsage(usageData);
    } catch (error) {
      console.error('Failed to fetch metrics:', error);
    } finally {
      setMetricsLoading(false);
    }
  };
  
  const handleAddServer = () => {
    navigate('/servers/new');
  };
  
  const handleRefresh = () => {
    fetchServers();
    fetchMetrics();
  };
  
  return (
    <Container maxWidth="lg">
      <PageHeader
        title="Dashboard"
        description="Overview of your MCP servers and system health"
        actions={
          <Button
            variant="contained"
            color="primary"
            startIcon={<AddIcon />}
            onClick={handleAddServer}
          >
            Add Server
          </Button>
        }
      />
      
      <Box sx={{ mb: 4, display: 'flex', justifyContent: 'flex-end' }}>
        <Button 
          startIcon={<RefreshIcon />}
          onClick={handleRefresh}
          disabled={loading || metricsLoading}
        >
          Refresh
        </Button>
      </Box>
      
      <Grid container spacing={3}>
        {/* Status summary */}
        <Grid item xs={12} md={8}>
          {metricsLoading ? (
            <Paper sx={{ p: 3, display: 'flex', justifyContent: 'center' }}>
              <CircularProgress size={40} />
            </Paper>
          ) : (
            <StatusSummary serverStatus={serverStatus} />
          )}
        </Grid>
        
        {/* Tool usage stats */}
        <Grid item xs={12} md={4}>
          {metricsLoading ? (
            <Paper sx={{ p: 3, display: 'flex', justifyContent: 'center' }}>
              <CircularProgress size={40} />
            </Paper>
          ) : (
            <ToolUsageStats toolUsage={toolUsage} />
          )}
        </Grid>
        
        {/* Recent servers section */}
        <Grid item xs={12}>
          <Box sx={{ mb: 2, display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
            <Typography variant="h6">MCP Servers</Typography>
            <Button 
              size="small" 
              onClick={() => navigate('/servers')}
            >
              View All
            </Button>
          </Box>
          
          {loading ? (
            <Box sx={{ display: 'flex', justifyContent: 'center', py: 4 }}>
              <CircularProgress />
            </Box>
          ) : servers.length > 0 ? (
            <Grid container spacing={3}>
              {servers.slice(0, 6).map((server) => (
                <Grid item xs={12} sm={6} md={4} key={server.id}>
                  <ServerCard server={server} />
                </Grid>
              ))}
            </Grid>
          ) : (
            <Paper sx={{ p: 4, textAlign: 'center' }}>
              <Typography variant="body1" paragraph>
                No MCP servers registered yet
              </Typography>
              <Button
                variant="contained"
                color="primary"
                startIcon={<AddIcon />}
                onClick={handleAddServer}
              >
                Add Your First Server
              </Button>
            </Paper>
          )}
        </Grid>
      </Grid>
    </Container>
  );
};

export default Dashboard;
```

### 10.2 Server Detail Page

```jsx
// src/pages/ServerDetail.jsx
import React, { useState, useEffect } from 'react';
import { 
  Container, 
  Grid, 
  Typography, 
  Box, 
  Button, 
  Tabs,
  Tab,
  Paper,
  Dialog,
  DialogTitle,
  DialogContent,
  DialogContentText,
  DialogActions,
  CircularProgress
} from '@mui/material';
import { useParams, useNavigate } from 'react-router-dom';
import EditIcon from '@mui/icons-material/Edit';
import DeleteIcon from '@mui/icons-material/Delete';
import RefreshIcon from '@mui/icons-material/Refresh';
import PageHeader from '../components/common/PageHeader';
import ServerConfigCard from '../components/servers/ServerConfigCard';
import ServerToolsList from '../components/servers/ServerToolsList';
import ToolDetailDialog from '../components/tools/ToolDetailDialog';
import { useServers } from '../hooks/useServers';
import serverService from '../services/serverService';
import { useNotification } from '../hooks/useNotification';

// TabPanel component to handle tab content
const TabPanel = (props) => {
  const { children, value, index, ...other } = props;

  return (
    <div
      role="tabpanel"
      hidden={value !== index}
      id={`server-tabpanel-${index}`}
      aria-labelledby={`server-tab-${index}`}
      {...other}
    >
      {value === index && (
        <Box sx={{ p: 3 }}>
          {children}
        </Box>
      )}
    </div>
  );
};

const ServerDetail = () => {
  const { serverId } = useParams();
  const navigate = useNavigate();
  const { showSuccess, showError } = useNotification();
  const { deleteServer } = useServers();
  
  const [server, setServer] = useState(null);
  const [tools, setTools] = useState([]);
  const [loading, setLoading] = useState(true);
  const [toolsLoading, setToolsLoading] = useState(true);
  const [deleteDialogOpen, setDeleteDialogOpen] = useState(false);
  const [selectedTool, setSelectedTool] = useState(null);
  const [tabValue, setTabValue] = useState(0);
  
  useEffect(() => {
    fetchServerDetails();
  }, [serverId]);
  
  const fetchServerDetails = async () => {
    setLoading(true);
    try {
      const serverData = await serverService.getServer(serverId);
      setServer(serverData);
      
      // Also fetch tools
      fetchServerTools();
    } catch (error) {
      console.error('Error fetching server details:', error);
      showError('Failed to load server details');
    } finally {
      setLoading(false);
    }
  };
  
  const fetchServerTools = async () => {
    setToolsLoading(true);
    try {
      const toolsData = await serverService.getServerTools(serverId);
      setTools(toolsData);
    } catch (error) {
      console.error('Error fetching server tools:', error);
      showError('Failed to load server tools');
    } finally {
      setToolsLoading(false);
    }
  };
  
  const handleTabChange = (event, newValue) => {
    setTabValue(newValue);
  };
  
  const handleEdit = () => {
    navigate(`/servers/${serverId}/edit`);
  };
  
  const handleDeleteClick = () => {
    setDeleteDialogOpen(true);
  };
  
  const handleDeleteConfirm = async () => {
    const success = await deleteServer(serverId);
    setDeleteDialogOpen(false);
    
    if (success) {
      navigate('/servers');
    }
  };
  
  const handleDeleteCancel = () => {
    setDeleteDialogOpen(false);
  };
  
  const handleRefresh = () => {
    fetchServerDetails();
  };
  
  const handleToolSelect = (tool) => {
    setSelectedTool(tool);
  };
  
  const handleToolDialogClose = () => {
    setSelectedTool(null);
  };
  
  if (loading) {
    return (
      <Container maxWidth="lg">
        <Box sx={{ display: 'flex', justifyContent: 'center', alignItems: 'center', height: '50vh' }}>
          <CircularProgress />
        </Box>
      </Container>
    );
  }
  
  if (!server) {
    return (
      <Container maxWidth="lg">
        <Paper sx={{ p: 4, textAlign: 'center' }}>
          <Typography variant="h6" color="error">
            Server not found
          </Typography>
          <Button
            variant="contained"
            onClick={() => navigate('/servers')}
            sx={{ mt: 2 }}
          >
            Back to Servers
          </Button>
        </Paper>
      </Container>
    );
  }
  
  const breadcrumbs = [
    { label: 'Servers', path: '/servers' },
    { label: server.name }
  ];
  
  return (
    <Container maxWidth="lg">
      <PageHeader
        title={server.name}
        description={server.description}
        breadcrumbs={breadcrumbs}
        actions={
          <Box>
            <Button
              variant="outlined"
              startIcon={<RefreshIcon />}
              onClick={handleRefresh}
              sx={{ mr: 1 }}
            >
              Refresh
            </Button>
            <Button
              variant="outlined"
              startIcon={<EditIcon />}
              onClick={handleEdit}
              sx={{ mr: 1 }}
            >
              Edit
            </Button>
            <Button
              variant="outlined"
              color="error"
              startIcon={<DeleteIcon />}
              onClick={handleDeleteClick}
            >
              Delete
            </Button>
          </Box>
        }
      />
      
      <Paper sx={{ mb: 3 }}>
        <Tabs
          value={tabValue}
          onChange={handleTabChange}
          indicatorColor="primary"
          textColor="primary"
          sx={{ borderBottom: 1, borderColor: 'divider' }}
        >
          <Tab label="Overview" />
          <Tab label="Tools" />
          <Tab label="Logs" />
          <Tab label="Settings" />
        </Tabs>
        
        <TabPanel value={tabValue} index={0}>
          <Grid container spacing={3}>
            <Grid item xs={12}>
              <ServerConfigCard 
                server={server} 
                onEdit={handleEdit}
              />
            </Grid>
          </Grid>
        </TabPanel>
        
        <TabPanel value={tabValue} index={1}>
          {toolsLoading ? (
            <Box sx={{ display: 'flex', justifyContent: 'center', py: 4 }}>
              <CircularProgress />
            </Box>
          ) : (
            <ServerToolsList 
              tools={tools} 
              onToolSelect={handleToolSelect}
            />
          )}
        </TabPanel>
        
        <TabPanel value={tabValue} index={2}>
          <Typography variant="body1">
            Server logs will be displayed here.
          </Typography>
        </TabPanel>
        
        <TabPanel value={tabValue} index={3}>
          <Typography variant="body1">
            Server settings will be displayed here.
          </Typography>
        </TabPanel>
      </Paper>
      
      {/* Delete confirmation dialog */}
      <Dialog
        open={deleteDialogOpen}
        onClose={handleDeleteCancel}
      >
        <DialogTitle>Delete Server</DialogTitle>
        <DialogContent>
          <DialogContentText>
            Are you sure you want to delete the server "{server.name}"? This action cannot be undone.
          </DialogContentText>
        </DialogContent>
        <DialogActions>
          <Button onClick={handleDeleteCancel}>Cancel</Button>
          <Button onClick={handleDeleteConfirm} color="error" variant="contained">
            Delete
          </Button>
        </DialogActions>
      </Dialog>
      
      {/* Tool detail dialog */}
      {selectedTool && (
        <ToolDetailDialog
          tool={selectedTool}
          open={Boolean(selectedTool)}
          onClose={handleToolDialogClose}
          serverId={serverId}
        />
      )}
    </Container>
  );
};

export default ServerDetail;
```

### 10.3 Server Form (Add/Edit)

```jsx
// src/pages/ServerForm.jsx
import React, { useState, useEffect } from 'react';
import { 
  Container, 
  Grid, 
  Typography, 
  Box, 
  Button, 
  TextField, 
  FormControl,
  InputLabel,
  Select,
  MenuItem,
  Paper,
  FormHelperText,
  Divider,
  CircularProgress,
  Alert
} from '@mui/material';
import { useParams, useNavigate } from 'react-router-dom';
import PageHeader from '../components/common/PageHeader';
import { useServers } from '../hooks/useServers';
import { useNotification } from '../hooks/useNotification';
import { Prism as SyntaxHighlighter } from 'react-syntax-highlighter';
import { vscDarkPlus } from 'react-syntax-highlighter/dist/esm/styles/prism';

const ServerForm = () => {
  const { serverId } = useParams(); // If present, we're editing
  const navigate = useNavigate();
  const { showSuccess, showError } = useNotification();
  const { getServer, createServer, updateServer } = useServers();
  
  const isEditMode = Boolean(serverId);
  
  // Form state
  const [formData, setFormData] = useState({
    name: '',
    description: '',
    version: '1.0.0',
    owner: 'HSBC AI Markets',
    transport: 'stdio',
    config: {
      command: '',
      args: [],
      env: {}
    }
  });
  
  const [errors, setErrors] = useState({});
  const [loading, setLoading] = useState(false);
  const [configJson, setConfigJson] = useState('');
  const [jsonError, setJsonError] = useState('');
  
  // Fetch server data if editing
  useEffect(() => {
    if (isEditMode) {
      fetchServerData();
    }
  }, [isEditMode, serverId]);
  
  // Update JSON when formData changes
  useEffect(() => {
    setConfigJson(JSON.stringify(formData.config, null, 2));
  }, [formData.config]);
  
  const fetchServerData = async () => {
    setLoading(true);
    try {
      const serverData = await getServer(serverId);
      if (serverData) {
        setFormData(serverData);
      }
    } catch (error) {
      console.error('Error fetching server data:', error);
      showError('Failed to load server data for editing');
    } finally {
      setLoading(false);
    }
  };
  
  const handleInputChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
    
    // Clear error for this field
    if (errors[name]) {
      setErrors(prev => ({ ...prev, [name]: '' }));
    }
  };
  
  const handleConfigJsonChange = (e) => {
    const json = e.target.value;
    setConfigJson(json);
    setJsonError('');
    
    try {
      const parsed = JSON.parse(json);
      setFormData(prev => ({
        ...prev,
        config: parsed
      }));
    } catch (err) {
      setJsonError('Invalid JSON format');
    }
  };
  
  const validateForm = () => {
    const newErrors = {};
    
    if (!formData.name) {
      newErrors.name = 'Name is required';
    }
    
    if (!formData.description) {
      newErrors.description = 'Description is required';
    }
    
    if (!formData.version) {
      newErrors.version = 'Version is required';
    }
    
    if (!formData.owner) {
      newErrors.owner = 'Owner is required';
    }
    
    if (!formData.transport) {
      newErrors.transport = 'Transport is required';
    }
    
    if (jsonError) {
      newErrors.config = 'Configuration has invalid JSON format';
    }
    
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    
    if (!validateForm()) {
      return;
    }
    
    setLoading(true);
    
    try {
      if (isEditMode) {
        // Update existing server
        await updateServer(serverId, formData);
        showSuccess('Server updated successfully');
        navigate(`/servers/${serverId}`);
      } else {
        // Create new server
        const newServer = await createServer(formData);
        showSuccess('Server created successfully');
        navigate(`/servers/${newServer.id}`);
      }
    } catch (error) {
      console.error('Error saving server:', error);
      showError(isEditMode ? 'Failed to update server' : 'Failed to create server');
    } finally {
      setLoading(false);
    }
  };
  
  const handleCancel = () => {
    if (isEditMode) {
      navigate(`/servers/${serverId}`);
    } else {
      navigate('/servers');
    }
  };
  
  const breadcrumbs = [
    { label: 'Servers', path: '/servers' },
    ...(isEditMode ? [{ label: formData.name, path: `/servers/${serverId}` }] : []),
    { label: isEditMode ? 'Edit' : 'Add New Server' }
  ];
  
  return (
    <Container maxWidth="lg">
      <PageHeader
        title={isEditMode ? `Edit ${formData.name}` : 'Add New Server'}
        description={isEditMode 
          ? 'Modify the configuration for this MCP server' 
          : 'Register a new MCP server with the registry'
        }
        breadcrumbs={breadcrumbs}
      />
      
      <Paper sx={{ p: 3 }}>
        {loading && !isEditMode ? (
          <Box sx={{ display: 'flex', justifyContent: 'center', py: 4 }}>
            <CircularProgress />
          </Box>
        ) : (
          <form onSubmit={handleSubmit}>
            <Grid container spacing={3}>
              <Grid item xs={12}>
                <Typography variant="h6" gutterBottom>
                  Server Information
                </Typography>
              </Grid>
              
              <Grid item xs={12} md={6}>
                <TextField
                  fullWidth
                  label="Server Name"
                  name="name"
                  value={formData.name}
                  onChange={handleInputChange}
                  error={Boolean(errors.name)}
                  helperText={errors.name || 'A unique name for this MCP server'}
                  required
                />
              </Grid>
              
              <Grid item xs={12} md={6}>
                <TextField
                  fullWidth
                  label="Version"
                  name="version"
                  value={formData.version}
                  onChange={handleInputChange}
                  error={Boolean(errors.version)}
                  helperText={errors.version || 'Server version (e.g., 1.0.0)'}
                  required
                />
              </Grid>
              
              <Grid item xs={12}>
                <TextField
                  fullWidth
                  label="Description"
                  name="description"
                  value={formData.description}
                  onChange={handleInputChange}
                  error={Boolean(errors.description)}
                  helperText={errors.description || 'A brief description of this server and its tools'}
                  multiline
                  rows={2}
                  required
                />
              </Grid>
              
              <Grid item xs={12} md={6}>
                <TextField
                  fullWidth
                  label="Owner"
                  name="owner"
                  value={formData.owner}
                  onChange={handleInputChange}
                  error={Boolean(errors.owner)}
                  helperText={errors.owner || 'Team or individual responsible for this server'}
                  required
                />
              </Grid>
              
              <Grid item xs={12} md={6}>
                <FormControl fullWidth error={Boolean(errors.transport)}>
                  <InputLabel>Transport Protocol</InputLabel>
                  <Select
                    name="transport"
                    value={formData.transport}
                    onChange={handleInputChange}
                    label="Transport Protocol"
                    required
                  >
                    <MenuItem value="stdio">stdio</MenuItem>
                    <MenuItem value="sse">Server-Sent Events (SSE)</MenuItem>
                    <MenuItem value="websocket">WebSocket</MenuItem>
                  </Select>
                  <FormHelperText>
                    {errors.transport || 'Protocol used for communication with the MCP server'}
                  </FormHelperText>
                </FormControl>
              </Grid>
              
              <Grid item xs={12}>
                <Divider sx={{ my: 2 }} />
                <Typography variant="h6" gutterBottom>
                  Server Configuration
                </Typography>
                <Alert severity="info" sx={{ mb: 2 }}>
                  Configure how to start and connect to the MCP server. The configuration format
                  depends on the transport protocol selected.
                </Alert>
              </Grid>
              
              <Grid item xs={12}>
                <Typography variant="subtitle2" gutterBottom>
                  Configuration (JSON)
                </Typography>
                <TextField
                  fullWidth
                  multiline
                  rows={10}
                  value={configJson}
                  onChange={handleConfigJsonChange}
                  error={Boolean(jsonError)}
                  helperText={jsonError || 'Edit the server configuration in JSON format'}
                  InputProps={{
                    sx: { fontFamily: 'monospace', fontSize: '0.875rem' }
                  }}
                />
              </Grid>
              
              <Grid item xs={12}>
                <Typography variant="subtitle2" gutterBottom>
                  Example Configurations:
                </Typography>
                <Box sx={{ mb: 2 }}>
                  <Typography variant="caption" gutterBottom>
                    stdio transport (e.g., for Tavily):
                  </Typography>
                  <SyntaxHighlighter 
                    language="json" 
                    style={vscDarkPlus}
                    customStyle={{ borderRadius: 8 }}
                  >
                    {`{
  "command": "npx",
  "args": ["-y", "tavily-mcp@latest"],
  "env": {
    "TAVILY_API_KEY": "${TAVILY_API_KEY}"
  }
}`}
                  </SyntaxHighlighter>
                </Box>
                
                <Box sx={{ mb: 2 }}>
                  <Typography variant="caption" gutterBottom>
                    SSE transport (e.g., for FastAPI):
                  </Typography>
                  <SyntaxHighlighter 
                    language="json" 
                    style={vscDarkPlus}
                    customStyle={{ borderRadius: 8 }}
                  >
                    {`{
  "url": "http://localhost:8000/mcp"
}`}
                  </SyntaxHighlighter>
                </Box>
              </Grid>
              
              <Grid item xs={12}>
                <Box sx={{ display: 'flex', justifyContent: 'flex-end', mt: 2 }}>
                  <Button
                    variant="outlined"
                    onClick={handleCancel}
                    sx={{ mr: 2 }}
                  >
                    Cancel
                  </Button>
                  <Button
                    type="submit"
                    variant="contained"
                    color="primary"
                    disabled={loading}
                  >
                    {loading ? <CircularProgress size={24} /> : isEditMode ? 'Update Server' : 'Add Server'}
                  </Button>
                </Box>
              </Grid>
            </Grid>
          </form>
        )}
      </Paper>
    </Container>
  );
};

export default ServerForm;
```

### 10.4 Login Page

```jsx
// src/pages/Login.jsx
import React, { useState } from 'react';
import { 
  Container, 
  Box, 
  Paper, 
  Typography, 
  TextField, 
  Button, 
  Grid,
  Link,
  CircularProgress,
  Alert,
  InputAdornment,
  IconButton
} from '@mui/material';
import { Link as RouterLink, useNavigate, useLocation } from 'react-router-dom';
import { Visibility, VisibilityOff } from '@mui/icons-material';
import { useAuth } from '../hooks/useAuth';
import { ReactComponent as Logo } from '../assets/hsbc-logo.svg';

const Login = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [showPassword, setShowPassword] = useState(false);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');
  
  const { login } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();
  
  // Get redirect location from state or default to '/'
  const from = location.state?.from?.pathname || '/';
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setError('');
    
    try {
      const result = await login(email, password);
      if (result.success) {
        navigate(from, { replace: true });
      } else {
        setError(result.error);
      }
    } catch (err) {
      setError('An unexpected error occurred. Please try again.');
    } finally {
      setLoading(false);
    }
  };
  
  const toggleShowPassword = () => {
    setShowPassword(!showPassword);
  };
  
  return (
    <Container component="main" maxWidth="xs">
      <Box
        sx={{
          marginTop: 8,
          display: 'flex',
          flexDirection: 'column',
          alignItems: 'center',
        }}
      >
        <Paper
          elevation={3}
          sx={{
            padding: 4,
            width: '100%',
            borderRadius: 2,
          }}
        >
          <Box sx={{ display: 'flex', flexDirection: 'column', alignItems: 'center', mb: 4 }}>
            <Logo width={60} height={60} />
            <Typography component="h1" variant="h5" sx={{ mt: 2, fontWeight: 'bold' }}>
              MCP Registry
            </Typography>
            <Typography variant="body2" color="text.secondary">
              Sign in to access the HSBC MCP Registry
            </Typography>
          </Box>
          
          {error && (
            <Alert severity="error" sx={{ mb: 3 }}>
              {error}
            </Alert>
          )}
          
          <form onSubmit={handleSubmit}>
            <TextField
              margin="normal"
              required
              fullWidth
              id="email"
              label="Email Address"
              name="email"
              autoComplete="email"
              autoFocus
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              disabled={loading}
              sx={{ mb: 2 }}
            />
            <TextField
              margin="normal"
              required
              fullWidth
              name="password"
              label="Password"
              type={showPassword ? 'text' : 'password'}
              id="password"
              autoComplete="current-password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              disabled={loading}
              InputProps={{
                endAdornment: (
                  <InputAdornment position="end">
                    <IconButton
                      aria-label="toggle password visibility"
                      onClick={toggleShowPassword}
                      edge="end"
                    >
                      {showPassword ? <VisibilityOff /> : <Visibility />}
                    </IconButton>
                  </InputAdornment>
                ),
              }}
              sx={{ mb: 3 }}
            />
            
            <Button
              type="submit"
              fullWidth
              variant="contained"
              disabled={loading}
              sx={{ py: 1.5 }}
            >
              {loading ? <CircularProgress size={24} /> : 'Sign In'}
            </Button>
            
            <Grid container justifyContent="flex-end" sx={{ mt: 2 }}>
              <Grid item>
                <Link component={RouterLink} to="/forgot-password" variant="body2">
                  Forgot password?
                </Link>
              </Grid>
            </Grid>
          </form>
        </Paper>
        
        <Typography variant="body2" color="text.secondary" sx={{ mt: 3 }}>
          © {new Date().getFullYear()} HSBC Group. All rights reserved.
        </Typography>
      </Box>
    </Container>
  );
};

export default Login;
```

## Conclusion

This comprehensive React UI implementation for the HSBC MCP Registry provides:

1. **Modern Design**: A clean, professional interface using Material UI components with HSBC branding
2. **Responsive Layout**: Works seamlessly across desktop and mobile devices
3. **Comprehensive Authentication**: JWT-based authentication with role-based access control
4. **Rich Server Management**: Detailed server configuration views and tool exploration
5. **Error Handling**: Thorough notification system and error management
6. **Monitoring Dashboard**: Health monitoring and usage statistics
7. **Form Validation**: Robust form handling with validation

The implementation follows best practices for React development, including:

- Component-based architecture with clear separation of concerns
- Context-based state management
- Custom hooks for reusable logic
- Consistent error handling
- Responsive design patterns

This UI provides a complete solution for managing MCP servers, exploring available tools, and monitoring system health in a user-friendly interface.

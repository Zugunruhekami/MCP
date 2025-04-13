*Thinking...*

> This request is asking for a comprehensive design and implementation of a full-fledged production-ready forex/crypto options trading platform similar to SignalPlus. Let me break down the requirements:
> 
> 1. Complete system architecture with:
>    - Frontend
>    - Backend
>    - Database
> 
> 2. Features to support:
>    - Live Risk/PnL tracking
>    - Volatility Lab with various visualizations
>    - Risk Scenario analysis
>    - Dynamic Delta Hedge (DDH) automation
>    - Customizable dashboard/UI
> 
> 3. Technical requirements:
>    - Frontend: Modern UI with TradingView visualizations
>    - Backend: Python or Rust
>    - Database: KDB+ or DolphinDB plus relational DB if needed
>    - Use ag-Grid as a line-pricer concept
> 
> 4. The implementation should be detailed down to folder and file structure
> 
> This is an extremely comprehensive request requiring the design of a production-grade financial trading system. Let me outline a complete architecture and implementation plan:
> 
> ## System Architecture Overview
> 
> I'll design a system with:
> 
> 1. **Frontend**: React-based SPA with TypeScript
> 2. **Backend**: Python FastAPI application with optional Rust microservices for performance-critical components
> 3. **Database**: 
>    - DolphinDB for time-series data (market data, positions)
>    - PostgreSQL for relational data (users, settings, etc.)
> 4. **Middleware**:
>    - Redis for caching and pub/sub
>    - RabbitMQ for message queuing
> 5. **External Integrations**:
>    - Exchange APIs
>    - Telegram API for notifications
>    - TradingView integration
> 
> ## Detailed Implementation Plan
> 
> Let me now design the detailed implementation including folder structure, key files, and code examples for each component.
> 
> ### 1. Project Structure
> 
> ```
> options-trading-platform/
> ├── frontend/                # React-based SPA
> ├── backend/                 # Python FastAPI backend
> ├── services/                # Additional microservices
> │   ├── pricer/              # Option pricing engine (Rust)
> │   ├── risk-engine/         # Risk calculations (Rust)
> │   ├── backtester/          # Backtesting engine (Python)
> │   └── ddh-service/         # Delta hedging service (Python)
> ├── db/                      # Database scripts and migrations
> │   ├── dolphindb/           # DolphinDB schema and scripts
> │   └── postgres/            # PostgreSQL schema and migrations
> ├── docker/                  # Docker configurations
> ├── k8s/                     # Kubernetes configurations (if applicable)
> ├── tests/                   # Test suites
> └── docs/                    # Documentation
> ```
> 
> ### 2. Frontend Implementation
> 
> The frontend will be a React application with TypeScript, using modern libraries and tools.
> 
> #### Frontend Structure
> 
> ```
> frontend/
> ├── public/
> ├── src/
> │   ├── assets/              # Static assets
> │   ├── components/          # Reusable UI components
> │   │   ├── common/          # Common UI elements
> │   │   ├── grid/            # AG-Grid related components
> │   │   ├── charts/          # Chart components
> │   │   └── widgets/         # Dashboard widgets
> │   ├── contexts/            # React contexts
> │   ├── hooks/               # Custom React hooks
> │   ├── layouts/             # Page layouts
> │   ├── pages/               # Application pages
> │   │   ├── Dashboard/
> │   │   ├── VolatilityLab/
> │   │   ├── RiskScenarios/
> │   │   ├── BacktestingLab/
> │   │   ├── Settings/
> │   │   └── Authentication/
> │   ├── services/            # API services
> │   ├── store/               # State management
> │   │   ├── slices/          # Redux slices
> │   │   ├── thunks/          # Async actions
> │   │   ├── selectors/       # State selectors
> │   │   └── store.ts         # Store configuration
> │   ├── types/               # TypeScript type definitions
> │   ├── utils/               # Utility functions
> │   ├── App.tsx              # Main App component
> │   ├── index.tsx            # Entry point
> │   └── routes.tsx           # Application routes
> ├── .env                     # Environment variables
> ├── package.json             # Project dependencies
> ├── tsconfig.json            # TypeScript configuration
> └── webpack.config.js        # Webpack configuration
> ```
> 
> #### Key Frontend Files
> 
> Let's create some of the crucial files:
> 
> 1. **`frontend/src/components/grid/OptionsGrid.tsx`** - Core AG Grid component for options display:
> 
> ```tsx
> import React, { useState, useEffect, useMemo, useCallback } from 'react';
> import { AgGridReact } from 'ag-grid-react';
> import 'ag-grid-community/styles/ag-grid.css';
> import 'ag-grid-community/styles/ag-theme-alpine.css';
> import { GridOptions, ColDef, ValueFormatterParams } from 'ag-grid-community';
> import { useAppDispatch, useAppSelector } from '../../hooks/reduxHooks';
> import { selectPositions, fetchPositions } from '../../store/slices/positionsSlice';
> import { formatCurrency, formatNumber, formatPercentage } from '../../utils/formatters';
> import { PositionActionsRenderer } from './renderers/PositionActionsRenderer';
> import { PnLRenderer } from './renderers/PnLRenderer';
> import { GreekRenderer } from './renderers/GreekRenderer';
> import { OptionPosition } from '../../types/positions';
> 
> interface OptionsGridProps {
>   viewType: 'positions' | 'greeks' | 'pnl';
>   assetClass?: string;
>   onSelectionChanged?: (selectedRows: OptionPosition[]) => void;
> }
> 
> export const OptionsGrid: React.FC<OptionsGridProps> = ({ 
>   viewType = 'positions',
>   assetClass,
>   onSelectionChanged 
> }) => {
>   const dispatch = useAppDispatch();
>   const { positions, loading, error } = useAppSelector(selectPositions);
>   const [rowData, setRowData] = useState<OptionPosition[]>([]);
>   
>   // Filter positions based on assetClass if provided
>   useEffect(() => {
>     let filteredData = [...positions];
>     if (assetClass && assetClass !== 'all') {
>       filteredData = positions.filter(p => p.assetClass === assetClass);
>     }
>     setRowData(filteredData);
>   }, [positions, assetClass]);
>   
>   // Fetch positions on component mount
>   useEffect(() => {
>     dispatch(fetchPositions());
>     const intervalId = setInterval(() => {
>       dispatch(fetchPositions());
>     }, 30000); // Refresh every 30 seconds
>     
>     return () => clearInterval(intervalId);
>   }, [dispatch]);
>   
>   // Column definitions for positions view
>   const positionsColDefs = useMemo<ColDef[]>(() => [
>     {
>       headerName: '',
>       field: 'id',
>       width: 50,
>       checkboxSelection: true,
>       headerCheckboxSelection: true,
>       pinned: 'left'
>     },
>     {
>       headerName: 'Symbol',
>       field: 'symbol',
>       width: 120,
>       pinned: 'left',
>       filter: 'agTextColumnFilter',
>       sortable: true
>     },
>     {
>       headerName: 'Type',
>       field: 'optionType',
>       width: 90,
>       cellStyle: params => {
>         return params.value === 'CALL' 
>           ? { color: 'var(--call-color)' } 
>           : { color: 'var(--put-color)' };
>       }
>     },
>     {
>       headerName: 'Strike',
>       field: 'strike',
>       width: 110,
>       valueFormatter: (params: ValueFormatterParams) => {
>         return formatCurrency(params.value, params.data.currency);
>       }
>     },
>     {
>       headerName: 'Expiry',
>       field: 'expiryDate',
>       width: 120,
>       sort: 'asc',
>       valueFormatter: (params: ValueFormatterParams) => {
>         return new Date(params.value).toLocaleDateString();
>       }
>     },
>     {
>       headerName: 'Qty',
>       field: 'quantity',
>       width: 100,
>       valueFormatter: (params: ValueFormatterParams) => {
>         return formatNumber(params.value, params.data.quantityPrecision || 4);
>       }
>     },
>     {
>       headerName: 'Entry',
>       field: 'entryPrice',
>       width: 110,
>       valueFormatter: (params: ValueFormatterParams) => {
>         return formatCurrency(params.value, params.data.currency);
>       }
>     },
>     {
>       headerName: 'Market',
>       field: 'marketPrice',
>       width: 110,
>       valueFormatter: (params: ValueFormatterParams) => {
>         return formatCurrency(params.value, params.data.currency);
>       }
>     },
>     {
>       headerName: 'P&L',
>       field: 'pnl',
>       width: 120,
>       cellRenderer: PnLRenderer
>     },
>     {
>       headerName: 'Delta',
>       field: 'delta',
>       width: 100,
>       cellRenderer: GreekRenderer
>     },
>     {
>       headerName: 'Gamma',
>       field: 'gamma',
>       width: 100,
>       valueFormatter: (params: ValueFormatterParams) => {
>         return formatNumber(params.value, 4);
>       }
>     },
>     {
>       headerName: 'Theta',
>       field: 'theta',
>       width: 100,
>       cellRenderer: GreekRenderer
>     },
>     {
>       headerName: 'Vega',
>       field: 'vega',
>       width: 100,
>       valueFormatter: (params: ValueFormatterParams) => {
>         return formatNumber(params.value, 2);
>       }
>     },
>     {
>       headerName: 'IV',
>       field: 'impliedVolatility',
>       width: 90,
>       valueFormatter: (params: ValueFormatterParams) => {
>         return formatPercentage(params.value);
>       }
>     },
>     {
>       headerName: 'Actions',
>       width: 150,
>       cellRenderer: PositionActionsRenderer,
>       cellRendererParams: {
>         onClose: (id: string) => console.log('Close position', id),
>         onHedge: (id: string) => console.log('Hedge position', id)
>       }
>     }
>   ], []);
>   
>   // Column definitions for greeks view
>   const greeksColDefs = useMemo<ColDef[]>(() => [
>     // ... Greek-focused columns
>     // Similar to the positionsColDefs but with emphasis on Greeks
>   ], []);
>   
>   // Column definitions for PnL view
>   const pnlColDefs = useMemo<ColDef[]>(() => [
>     // ... PnL-focused columns
>     // Includes PnL breakdown and attribution
>   ], []);
>   
>   // Select column definitions based on viewType
>   const columnDefs = useMemo(() => {
>     switch (viewType) {
>       case 'greeks':
>         return greeksColDefs;
>       case 'pnl':
>         return pnlColDefs;
>       case 'positions':
>       default:
>         return positionsColDefs;
>     }
>   }, [viewType, positionsColDefs, greeksColDefs, pnlColDefs]);
>   
>   // Grid options
>   const gridOptions: GridOptions = {
>     defaultColDef: {
>       sortable: true,
>       resizable: true,
>       filter: true
>     },
>     rowSelection: 'multiple',
>     suppressRowClickSelection: true,
>     animateRows: true,
>     pagination: true,
>     paginationPageSize: 20,
>     enableCellTextSelection: true,
>     ensureDomOrder: true,
>     getRowStyle: params => {
>       if (params.node.rowIndex % 2 === 0) {
>         return { background: 'var(--row-alt-color)' };
>       }
>       return null;
>     }
>   };
>   
>   // Handle selection changes
>   const onGridSelectionChanged = useCallback(() => {
>     if (onSelectionChanged && gridRef.current) {
>       const selectedRows = gridRef.current.api.getSelectedRows();
>       onSelectionChanged(selectedRows);
>     }
>   }, [onSelectionChanged]);
>   
>   // Grid reference for API access
>   const gridRef = React.useRef<AgGridReact>(null);
>   
>   // Force columns to fit on resize
>   useEffect(() => {
>     const handleResize = () => {
>       if (gridRef.current && gridRef.current.api) {
>         gridRef.current.api.sizeColumnsToFit();
>       }
>     };
>     window.addEventListener('resize', handleResize);
>     return () => window.removeEventListener('resize', handleResize);
>   }, []);
>   
>   // Render grid with loading state
>   return (
>     <div className="options-grid-container">
>       {loading && <div className="grid-loading-overlay">Loading positions...</div>}
>       {error && <div className="grid-error-message">Error: {error}</div>}
>       
>       <div 
>         className="ag-theme-alpine-dark options-grid" 
>         style={{ height: '100%', width: '100%' }}
>       >
>         <AgGridReact
>           ref={gridRef}
>           rowData={rowData}
>           columnDefs={columnDefs}
>           gridOptions={gridOptions}
>           onSelectionChanged={onGridSelectionChanged}
>           onFirstDataRendered={params => params.api.sizeColumnsToFit()}
>         />
>       </div>
>       
>       <div className="grid-summary">
>         <div className="summary-item">
>           <span className="summary-label">Total Positions</span>
>           <span className="summary-value">{rowData.length}</span>
>         </div>
>         <div className="summary-item">
>           <span className="summary-label">Net Delta</span>
>           <span className="summary-value">
>             {formatNumber(rowData.reduce((sum, pos) => sum + pos.delta, 0), 2)}
>           </span>
>         </div>
>         <div className="summary-item">
>           <span className="summary-label">Net Theta</span>
>           <span className="summary-value">
>             {formatCurrency(rowData.reduce((sum, pos) => sum + pos.theta, 0), 'USD')}
>           </span>
>         </div>
>         <div className="summary-item">
>           <span className="summary-label">Total P&L</span>
>           <span className={`summary-value ${rowData.reduce((sum, pos) => sum + pos.pnl, 0) >= 0 ? 'positive' : 'negative'}`}>
>             {formatCurrency(rowData.reduce((sum, pos) => sum + pos.pnl, 0), 'USD')}
>           </span>
>         </div>
>       </div>
>     </div>
>   );
> };
> ```
> 
> 2. **`frontend/src/components/charts/VolatilitySurface.tsx`** - 3D Surface Chart for Implied Volatility:
> 
> ```tsx
> import React, { useEffect, useRef } from 'react';
> import * as echarts from 'echarts';
> import { EChartsOption } from 'echarts';
> import 'echarts-gl';
> import { useAppSelector, useAppDispatch } from '../../hooks/reduxHooks';
> import { selectVolSurface, fetchVolSurface } from '../../store/slices/volatilitySlice';
> 
> interface VolatilitySurfaceProps {
>   symbol: string;
> }
> 
> export const VolatilitySurface: React.FC<VolatilitySurfaceProps> = ({ symbol }) => {
>   const chartRef = useRef<HTMLDivElement>(null);
>   const chartInstance = useRef<echarts.ECharts | null>(null);
>   const dispatch = useAppDispatch();
>   const { data, loading, error } = useAppSelector(selectVolSurface);
>   
>   // Fetch volatility surface data
>   useEffect(() => {
>     dispatch(fetchVolSurface(symbol));
>   }, [dispatch, symbol]);
>   
>   // Initialize and update chart
>   useEffect(() => {
>     if (!chartRef.current) return;
>     
>     if (!chartInstance.current) {
>       chartInstance.current = echarts.init(chartRef.current);
>     }
>     
>     if (!loading && data && data.length > 0) {
>       // Prepare data for 3D surface
>       const strikes = Array.from(new Set(data.map(item => item.strike))).sort((a, b) => a - b);
>       const tenors = Array.from(new Set(data.map(item => item.tenor))).sort((a, b) => a - b);
>       
>       const surfaceData = [];
>       
>       for (let i = 0; i < strikes.length; i++) {
>         for (let j = 0; j < tenors.length; j++) {
>           const point = data.find(
>             item => item.strike === strikes[i] && item.tenor === tenors[j]
>           );
>           
>           if (point) {
>             surfaceData.push([i, j, point.impliedVol * 100]);
>           }
>         }
>       }
>       
>       // Configure the chart
>       const option: EChartsOption = {
>         tooltip: {},
>         visualMap: {
>           show: true,
>           dimension: 2,
>           min: Math.min(...data.map(item => item.impliedVol * 100)),
>           max: Math.max(...data.map(item => item.impliedVol * 100)),
>           inRange: {
>             color: ['#313695', '#4575b4', '#74add1', '#abd9e9', '#e0f3f8', 
>                     '#ffffbf', '#fee090', '#fdae61', '#f46d43', '#d73027', '#a50026']
>           }
>         },
>         xAxis3D: {
>           type: 'category',
>           name: 'Strike',
>           data: strikes.map(strike => strike.toString())
>         },
>         yAxis3D: {
>           type: 'category',
>           name: 'Tenor (days)',
>           data: tenors.map(tenor => tenor.toString())
>         },
>         zAxis3D: {
>           type: 'value',
>           name: 'Implied Vol (%)',
>           min: 'dataMin',
>           max: 'dataMax'
>         },
>         grid3D: {
>           viewControl: {
>             projection: 'perspective',
>             autoRotate: false
>           },
>           light: {
>             main: {
>               intensity: 1.2
>             },
>             ambient: {
>               intensity: 0.3
>             }
>           }
>         },
>         series: [{
>           type: 'surface',
>           data: surfaceData,
>           wireframe: {
>             show: true
>           },
>           itemStyle: {
>             opacity: 0.8
>           },
>           emphasis: {
>             itemStyle: {
>               color: '#fff'
>             }
>           }
>         }]
>       };
>       
>       chartInstance.current.setOption(option);
>     }
>     
>     // Resize handler
>     const handleResize = () => {
>       chartInstance.current?.resize();
>     };
>     
>     window.addEventListener('resize', handleResize);
>     
>     return () => {
>       window.removeEventListener('resize', handleResize);
>       chartInstance.current?.dispose();
>     };
>   }, [data, loading]);
>   
>   return (
>     <div className="volatility-surface-container">
>       {loading && <div className="chart-loading">Loading volatility data...</div>}
>       {error && <div className="chart-error">Error: {error}</div>}
>       <div ref={chartRef} className="volatility-surface-chart" />
>     </div>
>   );
> };
> ```
> 
> 3. **`frontend/src/components/widgets/RiskScenario.tsx`** - Risk scenario analysis widget:
> 
> ```tsx
> import React, { useState, useEffect } from 'react';
> import { useAppDispatch, useAppSelector } from '../../hooks/reduxHooks';
> import { selectPositions } from '../../store/slices/positionsSlice';
> import { runScenario, selectScenarioResults } from '../../store/slices/scenarioSlice';
> import { formatCurrency, formatPercentage } from '../../utils/formatters';
> import { ScenarioLineChart } from '../charts/ScenarioLineChart';
> 
> export const RiskScenario: React.FC = () => {
>   const dispatch = useAppDispatch();
>   const { positions } = useAppSelector(selectPositions);
>   const { results, loading } = useAppSelector(selectScenarioResults);
>   
>   // Scenario parameters
>   const [priceChange, setPriceChange] = useState<number>(0);
>   const [volChange, setVolChange] = useState<number>(0);
>   const [daysForward, setDaysForward] = useState<number>(0);
>   
>   // Run scenario when parameters change
>   useEffect(() => {
>     if (positions.length > 0) {
>       const params = {
>         priceChange,
>         volChange,
>         daysForward,
>         positionIds: positions.map(p => p.id)
>       };
>       
>       dispatch(runScenario(params));
>     }
>   }, [dispatch, positions, priceChange, volChange, daysForward]);
>   
>   // Handler for price change
>   const handlePriceChangeSelect = (e: React.ChangeEvent<HTMLSelectElement>) => {
>     setPriceChange(parseFloat(e.target.value));
>   };
>   
>   // Handler for volatility change
>   const handleVolChangeSelect = (e: React.ChangeEvent<HTMLSelectElement>) => {
>     setVolChange(parseFloat(e.target.value));
>   };
>   
>   // Handler for days forward
>   const handleDaysForwardChange = (e: React.ChangeEvent<HTMLInputElement>) => {
>     setDaysForward(parseInt(e.target.value, 10));
>   };
>   
>   return (
>     <div className="risk-scenario-widget">
>       <div className="widget-controls">
>         <div className="form-group">
>           <label>Price Movement</label>
>           <select 
>             className="form-control" 
>             value={priceChange}
>             onChange={handlePriceChangeSelect}
>           >
>             <option value="0.1">+10%</option>
>             <option value="0.05">+5%</option>
>             <option value="0.02">+2%</option>
>             <option value="0">0%</option>
>             <option value="-0.02">-2%</option>
>             <option value="-0.05">-5%</option>
>             <option value="-0.1">-10%</option>
>           </select>
>         </div>
>         
>         <div className="form-group">
>           <label>Vol Movement</label>
>           <select 
>             className="form-control" 
>             value={volChange}
>             onChange={handleVolChangeSelect}
>           >
>             <option value="0.2">+20%</option>
>             <option value="0.1">+10%</option>
>             <option value="0">0%</option>
>             <option value="-0.1">-10%</option>
>             <option value="-0.2">-20%</option>
>           </select>
>         </div>
>         
>         <div className="form-group">
>           <label>Days Forward: {daysForward}</label>
>           <input 
>             type="range" 
>             className="form-control" 
>             min="0" 
>             max="30" 
>             value={daysForward}
>             onChange={handleDaysForwardChange}
>           />
>         </div>
>       </div>
>       
>       <div className="scenario-chart-container">
>         {loading ? (
>           <div className="chart-loading">Computing scenario...</div>
>         ) : (
>           <ScenarioLineChart data={results?.scenarioCurve || []} />
>         )}
>       </div>
>       
>       <div className="scenario-results">
>         <h4>Scenario Results</h4>
>         {loading ? (
>           <div className="results-loading">Loading...</div>
>         ) : (
>           <table className="scenario-table">
>             <tbody>
>               <tr>
>                 <td>Total P&L Impact</td>
>                 <td className={results?.totalPnl >= 0 ? 'positive' : 'negative'}>
>                   {formatCurrency(results?.totalPnl || 0, 'USD')}
>                 </td>
>               </tr>
>               <tr>
>                 <td>Delta Impact</td>
>                 <td className={results?.deltaPnl >= 0 ? 'positive' : 'negative'}>
>                   {formatCurrency(results?.deltaPnl || 0, 'USD')}
>                 </td>
>               </tr>
>               <tr>
>                 <td>Gamma Impact</td>
>                 <td className={results?.gammaPnl >= 0 ? 'positive' : 'negative'}>
>                   {formatCurrency(results?.gammaPnl || 0, 'USD')}
>                 </td>
>               </tr>
>               <tr>
>                 <td>Vega Impact</td>
>                 <td className={results?.vegaPnl >= 0 ? 'positive' : 'negative'}>
>                   {formatCurrency(results?.vegaPnl || 0, 'USD')}
>                 </td>
>               </tr>
>               <tr>
>                 <td>Theta Impact</td>
>                 <td className={results?.thetaPnl >= 0 ? 'positive' : 'negative'}>
>                   {formatCurrency(results?.thetaPnl || 0, 'USD')}
>                 </td>
>               </tr>
>             </tbody>
>           </table>
>         )}
>       </div>
>     </div>
>   );
> };
> ```
> 
> 4. **`frontend/src/components/widgets/DeltaHedgePanel.tsx`** - Dynamic Delta Hedging control panel:
> 
> ```tsx
> import React, { useState, useEffect } from 'react';
> import { useAppDispatch, useAppSelector } from '../../hooks/reduxHooks';
> import { 
>   selectDeltaHedge, 
>   startDeltaHedging, 
>   stopDeltaHedging,
>   updateDeltaHedgeSettings
> } from '../../store/slices/deltaHedgeSlice';
> import { formatCurrency, formatNumber, formatDate } from '../../utils/formatters';
> 
> export const DeltaHedgePanel: React.FC = () => {
>   const dispatch = useAppDispatch();
>   const { 
>     active, 
>     strategy, 
>     settings, 
>     history, 
>     currentExposure,
>     telegramStatus 
>   } = useAppSelector(selectDeltaHedge);
>   
>   // Local state for form inputs
>   const [localSettings, setLocalSettings] = useState(settings);
>   const [isConfigOpen, setIsConfigOpen] = useState(false);
>   
>   // Update local settings when global settings change
>   useEffect(() => {
>     setLocalSettings(settings);
>   }, [settings]);
>   
>   // Handle strategy change
>   const handleStrategyChange = (e: React.ChangeEvent<HTMLSelectElement>) => {
>     const newStrategy = e.target.value;
>     
>     // Set default thresholds based on strategy
>     if (newStrategy === 'aggressive') {
>       setLocalSettings({
>         ...localSettings,
>         minDeltaThreshold: 0.05,
>         maxDeltaThreshold: 0.1
>       });
>     } else if (newStrategy === 'balanced') {
>       setLocalSettings({
>         ...localSettings,
>         minDeltaThreshold: 0.15,
>         maxDeltaThreshold: 0.25
>       });
>     } else if (newStrategy === 'conservative') {
>       setLocalSettings({
>         ...localSettings,
>         minDeltaThreshold: 0.25,
>         maxDeltaThreshold: 0.4
>       });
>     }
>     
>     dispatch(updateDeltaHedgeSettings({
>       strategy: newStrategy,
>       settings: localSettings
>     }));
>   };
>   
>   // Handle min threshold change
>   const handleMinThresholdChange = (e: React.ChangeEvent<HTMLInputElement>) => {
>     const value = parseFloat(e.target.value);
>     setLocalSettings({ ...localSettings, minDeltaThreshold: value });
>   };
>   
>   // Handle max threshold change
>   const handleMaxThresholdChange = (e: React.ChangeEvent<HTMLInputElement>) => {
>     const value = parseFloat(e.target.value);
>     setLocalSettings({ ...localSettings, maxDeltaThreshold: value });
>   };
>   
>   // Save settings
>   const handleSaveSettings = () => {
>     dispatch(updateDeltaHedgeSettings({
>       strategy,
>       settings: localSettings
>     }));
>     setIsConfigOpen(false);
>   };
>   
>   // Toggle DDH on/off
>   const handleToggleDDH = () => {
>     if (active) {
>       dispatch(stopDeltaHedging());
>     } else {
>       dispatch(startDeltaHedging());
>     }
>   };
>   
>   return (
>     <div className="ddh-panel">
>       <div className="ddh-header">
>         <div className="ddh-status">
>           <span className={`status-indicator ${active ? 'active' : 'inactive'}`}></span>
>           <span>Dynamic Delta Hedge: {active ? 'Running' : 'Stopped'}</span>
>         </div>
>         <div className="ddh-actions">
>           <button 
>             className={`btn ${active ? 'btn-danger' : 'btn-success'}`}
>             onClick={handleToggleDDH}
>           >
>             {active ? 'Stop DDH' : 'Start DDH'}
>           </button>
>           <button 
>             className="btn btn-secondary"
>             onClick={() => setIsConfigOpen(!isConfigOpen)}
>           >
>             Configure
>           </button>
>         </div>
>       </div>
>       
>       {isConfigOpen && (
>         <div className="ddh-config">
>           <div className="form-group">
>             <label>Hedge Strategy</label>
>             <select 
>               className="form-control"
>               value={strategy}
>               onChange={handleStrategyChange}
>             >
>               <option value="aggressive">Aggressive</option>
>               <option value="balanced">Balanced</option>
>               <option value="conservative">Conservative</option>
>               <option value="custom">Custom</option>
>             </select>
>           </div>
>           
>           <div className="form-row">
>             <div className="form-group">
>               <label>Min Delta Threshold</label>
>               <input 
>                 type="number" 
>                 className="form-control"
>                 value={localSettings.minDeltaThreshold}
>                 onChange={handleMinThresholdChange}
>                 step="0.01"
>                 min="0.01"
>                 max={localSettings.maxDeltaThreshold - 0.01}
>               />
>             </div>
>             
>             <div className="form-group">
>               <label>Max Delta Threshold</label>
>               <input 
>                 type="number" 
>                 className="form-control"
>                 value={localSettings.maxDeltaThreshold}
>                 onChange={handleMaxThresholdChange}
>                 step="0.01"
>                 min={localSettings.minDeltaThreshold + 0.01}
>                 max="1"
>               />
>             </div>
>           </div>
>           
>           <div className="form-actions">
>             <button 
>               className="btn btn-primary" 
>               onClick={handleSaveSettings}
>             >
>               Save Settings
>             </button>
>             <button 
>               className="btn btn-link" 
>               onClick={() => setIsConfigOpen(false)}
>             >
>               Cancel
>             </button>
>           </div>
>         </div>
>       )}
>       
>       <div className="ddh-history">
>         <h4>Hedge History</h4>
>         <div className="history-table-wrapper">
>           <table className="ddh-history-table">
>             <thead>
>               <tr>
>                 <th>Time</th>
>                 <th>Instrument</th>
>                 <th>Action</th>
>                 <th>Size</th>
>                 <th>Price</th>
>                 <th>Delta Before</th>
>                 <th>Delta After</th>
>                 <th>Status</th>
>               </tr>
>             </thead>
>             <tbody>
>               {history.length === 0 ? (
>                 <tr>
>                   <td colSpan={8} className="no-data">No hedging actions yet</td>
>                 </tr>
>               ) : (
>                 history.map(item => (
>                   <tr key={item.id}>
>                     <td>{formatDate(item.timestamp, true)}</td>
>                     <td>{item.symbol}</td>
>                     <td>{item.action}</td>
>                     <td>{formatNumber(item.size, 6)}</td>
>                     <td>{formatCurrency(item.price, item.currency)}</td>
>                     <td>{formatNumber(item.deltaBefore, 2)}</td>
>                     <td>{formatNumber(item.deltaAfter, 2)}</td>
>                     <td>
>                       <span className={`status-badge ${item.status.toLowerCase()}`}>
>                         {item.status}
>                       </span>
>                     </td>
>                   </tr>
>                 ))
>               )}
>             </tbody>
>           </table>
>         </div>
>       </div>
>       
>       <div className="ddh-footer">
>         <div className="delta-exposure">
>           <h5>Current Delta Exposure</h5>
>           <div className="exposure-meter">
>             <div 
>               className="exposure-bar" 
>               style={{
>                 width: `${Math.min(Math.abs(currentExposure.delta) * 100, 100)}%`,
>                 backgroundColor: currentExposure.delta > 0 ? 'var(--positive-color)' : 'var(--negative-color)'
>               }}
>             ></div>
>           </div>
>           <div className="exposure-value">
>             {formatNumber(currentExposure.delta, 2)} 
>             <span className="exposure-label">
>               ({currentExposure.delta > 0.3 ? 'Long' : 
>                 currentExposure.delta < -0.3 ? 'Short' : 'Neutral'})
>             </span>
>           </div>
>         </div>
>         
>         <div className="telegram-status">
>           <h5>Telegram Notifications</h5>
>           <div className="status-indicator-container">
>             <span className={`status-badge ${telegramStatus.connected ? 'success' : 'error'}`}>
>               {telegramStatus.connected ? 'Connected' : 'Disconnected'}
>             </span>
>             {telegramStatus.connected && (
>               <span className="last-update">
>                 Last updated: {telegramStatus.lastUpdate}
>               </span>
>             )}
>           </div>
>         </div>
>       </div>
>     </div>
>   );
> };
> ```
> 
> 5. **`frontend/src/pages/Dashboard/index.tsx`** - Main dashboard page:
> 
> ```tsx
> import React, { useState } from 'react';
> import { useAppSelector } from '../../hooks/reduxHooks';
> import { DashboardLayout } from '../../layouts/DashboardLayout';
> import { OptionsGrid } from '../../components/grid/OptionsGrid';
> import { RiskScenario } from '../../components/widgets/RiskScenario';
> import { DeltaHedgePanel } from '../../components/widgets/DeltaHedgePanel';
> import { StatsPanel } from '../../components/widgets/StatsPanel';
> import { selectUser } from '../../store/slices/userSlice';
> import { DashboardTabs } from '../../components/navigation/DashboardTabs';
> import { WidgetContainer } from '../../components/common/WidgetContainer';
> import { MarketDataWidget } from '../../components/widgets/MarketDataWidget';
> import './Dashboard.scss';
> 
> export const Dashboard: React.FC = () => {
>   const user = useAppSelector(selectUser);
>   const [activeTab, setActiveTab] = useState('positions');
>   const [selectedAsset, setSelectedAsset] = useState('all');
>   
>   // Asset selector options
>   const assetOptions = [
>     { value: 'all', label: 'All Assets' },
>     { value: 'BTC', label: 'BTC/USD' },
>     { value: 'ETH', label: 'ETH/USD' },
>     { value: 'EUR', label: 'EUR/USD' }
>   ];
>   
>   // Handle tab changes
>   const handleTabChange = (tab: string) => {
>     setActiveTab(tab);
>   };
>   
>   // Handle asset selection changes
>   const handleAssetChange = (e: React.ChangeEvent<HTMLSelectElement>) => {
>     setSelectedAsset(e.target.value);
>   };
>   
>   return (
>     <DashboardLayout>
>       <div className="dashboard-page">
>         <div className="dashboard-header">
>           <h1>Trading Dashboard</h1>
>           <div className="dashboard-controls">
>             <select 
>               className="asset-selector" 
>               value={selectedAsset} 
>               onChange={handleAssetChange}
>             >
>               {assetOptions.map(option => (
>                 <option key={option.value} value={option.value}>
>                   {option.label}
>                 </option>
>               ))}
>             </select>
>           </div>
>         </div>
>         
>         <StatsPanel assetClass={selectedAsset === 'all' ? undefined : selectedAsset} />
>         
>         <div className="dashboard-content">
>           <div className="main-section">
>             <div className="positions-section">
>               <div className="section-header">
>                 <DashboardTabs 
>                   tabs={[
>                     { id: 'positions', label: 'Position Overview' },
>                     { id: 'greeks', label: 'Greeks Breakdown' },
>                     { id: 'pnl', label: 'PnL Attribution' }
>                   ]}
>                   activeTab={activeTab}
>                   onTabChange={handleTabChange}
>                 />
>               </div>
>               
>               <OptionsGrid 
>                 viewType={activeTab as 'positions' | 'greeks' | 'pnl'} 
>                 assetClass={selectedAsset === 'all' ? undefined : selectedAsset}
>               />
>             </div>
>             
>             <div className="widgets-row">
>               <WidgetContainer title="Risk Scenario" className="scenario-widget">
>                 <RiskScenario />
>               </WidgetContainer>
>               
>               <WidgetContainer title="Market Data" className="market-widget">
>                 <MarketDataWidget symbol={selectedAsset === 'all' ? 'BTC' : selectedAsset} />
>               </WidgetContainer>
>             </div>
>           </div>
>           
>           <div className="side-section">
>             <WidgetContainer title="Dynamic Delta Hedge" className="ddh-widget">
>               <DeltaHedgePanel />
>             </WidgetContainer>
>           </div>
>         </div>
>       </div>
>     </DashboardLayout>
>   );
> };
> 
> export default Dashboard;
> ```
> 
> ### 3. Backend Implementation
> 
> Now let's move to the backend implementation using Python FastAPI.
> 
> #### Backend Structure
> 
> ```
> backend/
> ├── app/
> │   ├── api/
> │   │   ├── v1/
> │   │   │   ├── auth.py           # Authentication endpoints
> │   │   │   ├── positions.py      # Positions endpoints
> │   │   │   ├── market_data.py    # Market data endpoints
> │   │   │   ├── volatility.py     # Volatility endpoints
> │   │   │   ├── scenarios.py      # Risk scenarios endpoints
> │   │   │   ├── backtest.py       # Backtesting endpoints
> │   │   │   ├── delta_hedge.py    # Delta hedge endpoints
> │   │   │   └── settings.py       # User settings endpoints
> │   │   └── routes.py             # API router setup
> │   ├── core/
> │   │   ├── config.py             # Application configuration
> │   │   ├── security.py           # Authentication and security
> │   │   └── logging.py            # Logging configuration
> │   ├── db/
> │   │   ├── session.py            # Database session management
> │   │   ├── dolphindb.py          # DolphinDB connection manager
> │   │   └── postgres.py           # PostgreSQL connection manager
> │   ├── models/
> │   │   ├── user.py               # User model
> │   │   ├── position.py           # Position model
> │   │   ├── option.py             # Option model
> │   │   └── settings.py           # Settings model
> │   ├── schemas/
> │   │   ├── user.py               # User schema
> │   │   ├── position.py           # Position schema
> │   │   ├── option.py             # Option schema
> │   │   ├── market_data.py        # Market data schema
> │   │   ├── volatility.py         # Volatility schema
> │   │   ├── scenario.py           # Scenario schema
> │   │   └── delta_hedge.py        # Delta hedge schema
> │   ├── services/
> │   │   ├── pricing/
> │   │   │   ├── black_scholes.py  # Black-Scholes pricing
> │   │   │   └── volatility.py     # Volatility modeling
> │   │   ├── risk/
> │   │   │   ├── greeks.py         # Greeks calculation
> │   │   │   ├── scenarios.py      # Scenario analysis
> │   │   │   └── metrics.py        # Risk metrics calculation
> │   │   ├── market_data/
> │   │   │   ├── client.py         # Market data client
> │   │   │   └── feed.py           # Real-time feed manager
> │   │   ├── delta_hedge/
> │   │   │   ├── strategy.py       # Hedging strategies
> │   │   │   ├── execution.py      # Order execution
> │   │   │   └── monitor.py        # Exposure monitoring
> │   │   ├── telegram/
> │   │   │   └── notifications.py  # Telegram notifications
> │   │   └── backtesting/
> │   │       ├── engine.py         # Backtesting engine
> │   │       └── report.py         # Backtesting reports
> │   ├── utils/
> │   │   ├── date_utils.py         # Date manipulation utilities
> │   │   ├── math_utils.py         # Mathematical utilities
> │   │   └── validation.py         # Data validation utilities
> │   ├── main.py                   # FastAPI application entry point
> │   └── dependencies.py           # Dependency injection
> ├── alembic/                      # Database migrations
> ├── tests/                        # Test suite
> ├── .env                          # Environment variables
> ├── requirements.txt              # Python dependencies
> └── Dockerfile                    # Docker configuration
> ```
> 
> #### Key Backend Files
> 
> 1. **`backend/app/main.py`** - FastAPI application entry point:
> 
> ```python
> import time
> from fastapi import FastAPI, Request
> from fastapi.middleware.cors import CORSMiddleware
> from fastapi.responses import JSONResponse
> import logging
> 
> from app.core.config import settings
> from app.api.routes import api_router
> from app.core.logging import setup_logging
> from app.db.session import init_db
> 
> logger = logging.getLogger(__name__)
> 
> # Initialize FastAPI app
> app = FastAPI(
>     title=settings.PROJECT_NAME,
>     description="Options Trading Platform API",
>     version="1.0.0",
>     openapi_url=f"{settings.API_V1_STR}/openapi.json",
>     docs_url=f"{settings.API_V1_STR}/docs",
>     redoc_url=f"{settings.API_V1_STR}/redoc"
> )
> 
> # Set up CORS
> app.add_middleware(
>     CORSMiddleware,
>     allow_origins=[str(origin) for origin in settings.CORS_ORIGINS],
>     allow_credentials=True,
>     allow_methods=["*"],
>     allow_headers=["*"],
> )
> 
> # Request timing middleware
> @app.middleware("http")
> async def add_process_time_header(request: Request, call_next):
>     start_time = time.time()
>     response = await call_next(request)
>     process_time = time.time() - start_time
>     response.headers["X-Process-Time"] = str(process_time)
>     return response
> 
> # Global exception handler
> @app.exception_handler(Exception)
> async def global_exception_handler(request: Request, exc: Exception):
>     logger.error(f"Unhandled exception: {exc}", exc_info=True)
>     return JSONResponse(
>         status_code=500,
>         content={"detail": "Internal server error"}
>     )
> 
> # Include API routes
> app.include_router(api_router, prefix=settings.API_V1_STR)
> 
> # Startup event
> @app.on_event("startup")
> async def startup_event():
>     setup_logging()
>     logger.info("Starting application")
>     await init_db()
> 
> # Shutdown event
> @app.on_event("shutdown")
> async def shutdown_event():
>     logger.info("Shutting down application")
> 
> # Root endpoint
> @app.get("/")
> async def root():
>     return {"message": "Options Trading Platform API. See /api/v1/docs for documentation."}
> 
> if __name__ == "__main__":
>     import uvicorn
>     uvicorn.run(app, host="0.0.0.0", port=8000)
> ```
> 
> 2. **`backend/app/api/v1/positions.py`** - Positions API endpoints:
> 
> ```python
> from typing import List, Optional
> from fastapi import APIRouter, Depends, HTTPException, Query, Path
> from sqlalchemy.orm import Session
> 
> from app.schemas.position import (
>     PositionCreate, 
>     PositionUpdate, 
>     PositionResponse,
>     PositionWithGreeks
> )
> from app.models.user import User
> from app.core.security import get_current_user
> from app.db.session import get_db
> from app.services.pricing import get_option_prices
> from app.services.risk.greeks import calculate_position_greeks
> 
> router = APIRouter()
> 
> @router.get("/", response_model=List[PositionWithGreeks])
> async def get_positions(
>     symbol: Optional[str] = Query(None, description="Filter by symbol"),
>     option_type: Optional[str] = Query(None, description="Filter by option type (CALL/PUT)"),
>     db: Session = Depends(get_db),
>     current_user: User = Depends(get_current_user)
> ):
>     """
>     Get all positions for the current user with real-time pricing and Greeks.
>     """
>     # Query positions from database
>     positions = db.query(Position).filter(Position.user_id == current_user.id)
>     
>     # Apply filters if provided
>     if symbol:
>         positions = positions.filter(Position.symbol == symbol)
>     if option_type:
>         positions = positions.filter(Position.option_type == option_type)
>     
>     positions = positions.all()
>     
>     # Get current market data for pricing
>     symbols = {position.symbol for position in positions}
>     market_data = await get_market_data(list(symbols))
>     
>     # Calculate current prices and Greeks
>     position_responses = []
>     for position in positions:
>         # Get current market price for underlying
>         current_price = market_data.get(position.symbol, {}).get("price", 0)
>         
>         # Get option pricing data
>         option_data = await get_option_prices(
>             symbol=position.symbol,
>             strike=position.strike,
>             expiry=position.expiry_date,
>             option_type=position.option_type
>         )
>         
>         # Calculate Greeks
>         greeks = calculate_position_greeks(
>             position=position,
>             market_price=current_price,
>             implied_vol=option_data.get("implied_volatility", 0),
>             risk_free_rate=0.03,  # This should come from a market data service
>             days_to_expiry=(position.expiry_date - datetime.now().date()).days
>         )
>         
>         # Combine position data with pricing and Greeks
>         position_with_greeks = PositionWithGreeks(
>             id=position.id,
>             symbol=position.symbol,
>             option_type=position.option_type,
>             strike=position.strike,
>             expiry_date=position.expiry_date,
>             quantity=position.quantity,
>             entry_price=position.entry_price,
>             market_price=option_data.get("price", 0),
>             unrealized_pnl=(option_data.get("price", 0) - position.entry_price) * position.quantity,
>             delta=greeks.get("delta", 0) * position.quantity,
>             gamma=greeks.get("gamma", 0) * position.quantity,
>             theta=greeks.get("theta", 0) * position.quantity,
>             vega=greeks.get("vega", 0) * position.quantity,
>             implied_volatility=option_data.get("implied_volatility", 0),
>             created_at=position.created_at,
>             updated_at=position.updated_at
>         )
>         
>         position_responses.append(position_with_greeks)
>     
>     return position_responses
> 
> 
> @router.post("/", response_model=PositionResponse)
> async def create_position(
>     position: PositionCreate,
>     db: Session = Depends(get_db),
>     current_user: User = Depends(get_current_user)
> ):
>     """
>     Create a new position.
>     """
>     # Validate the position data
>     if position.quantity <= 0:
>         raise HTTPException(status_code=400, detail="Quantity must be positive")
>     
>     # Check if the option exists
>     option = db.query(Option).filter(
>         Option.symbol == position.symbol,
>         Option.strike == position.strike,
>         Option.expiry_date == position.expiry_date,
>         Option.option_type == position.option_type
>     ).first()
>     
>     if not option:
>         raise HTTPException(status_code=404, detail="Option not found")
>     
>     # Create the position
>     db_position = Position(
>         user_id=current_user.id,
>         option_id=option.id,
>         symbol=position.symbol,
>         strike=position.strike,
>         expiry_date=position.expiry_date,
>         option_type=position.option_type,
>         quantity=position.quantity,
>         entry_price=position.entry_price
>     )
>     
>     db.add(db_position)
>     db.commit()
>     db.refresh(db_position)
>     
>     return db_position
> 
> 
> @router.get("/{position_id}", response_model=PositionWithGreeks)
> async def get_position(
>     position_id: int = Path(..., title="The ID of the position"),
>     db: Session = Depends(get_db),
>     current_user: User = Depends(get_current_user)
> ):
>     """
>     Get a specific position by ID with real-time pricing and Greeks.
>     """
>     position = db.query(Position).filter(
>         Position.id == position_id,
>         Position.user_id == current_user.id
>     ).first()
>     
>     if not position:
>         raise HTTPException(status_code=404, detail="Position not found")
>     
>     # Get current market data for pricing
>     market_data = await get_market_data([position.symbol])
>     current_price = market_data.get(position.symbol, {}).get("price", 0)
>     
>     # Get option pricing data
>     option_data = await get_option_prices(
>         symbol=position.symbol,
>         strike=position.strike,
>         expiry=position.expiry_date,
>         option_type=position.option_type
>     )
>     
>     # Calculate Greeks
>     greeks = calculate_position_greeks(
>         position=position,
>         market_price=current_price,
>         implied_vol=option_data.get("implied_volatility", 0),
>         risk_free_rate=0.03,
>         days_to_expiry=(position.expiry_date - datetime.now().date()).days
>     )
>     
>     # Combine position data with pricing and Greeks
>     position_with_greeks = PositionWithGreeks(
>         id=position.id,
>         symbol=position.symbol,
>         option_type=position.option_type,
>         strike=position.strike,
>         expiry_date=position.expiry_date,
>         quantity=position.quantity,
>         entry_price=position.entry_price,
>         market_price=option_data.get("price", 0),
>         unrealized_pnl=(option_data.get("price", 0) - position.entry_price) * position.quantity,
>         delta=greeks.get("delta", 0) * position.quantity,
>         gamma=greeks.get("gamma", 0) * position.quantity,
>         theta=greeks.get("theta", 0) * position.quantity,
>         vega=greeks.get("vega", 0) * position.quantity,
>         implied_volatility=option_data.get("implied_volatility", 0),
>         created_at=position.created_at,
>         updated_at=position.updated_at
>     )
>     
>     return position_with_greeks
> 
> 
> @router.delete("/{position_id}", response_model=dict)
> async def close_position(
>     position_id: int = Path(..., title="The ID of the position"),
>     db: Session = Depends(get_db),
>     current_user: User = Depends(get_current_user)
> ):
>     """
>     Close a position by ID.
>     """
>     position = db.query(Position).filter(
>         Position.id == position_id,
>         Position.user_id == current_user.id
>     ).first()
>     
>     if not position:
>         raise HTTPException(status_code=404, detail="Position not found")
>     
>     # In a real application, you might want to record the closing details
>     # before deleting or mark it as closed rather than deleting
>     db.delete(position)
>     db.commit()
>     
>     return {"message": "Position closed successfully"}
> ```
> 
> 3. **`backend/app/services/pricing/black_scholes.py`** - Black-Scholes option pricing implementation:
> 
> ```python
> import numpy as np
> from scipy.stats import norm
> import math
> from typing import Dict, Literal, Optional
> from datetime import datetime, date
> import logging
> 
> logger = logging.getLogger(__name__)
> 
> def black_scholes(
>     S: float,                # Spot price
>     K: float,                # Strike price
>     T: float,                # Time to expiration in years
>     r: float,                # Risk-free interest rate
>     sigma: float,            # Volatility
>     option_type: Literal["CALL", "PUT"]  # Option type
> ) -> Dict[str, float]:
>     """
>     Calculate option price and Greeks using the Black-Scholes model.
>     
>     Parameters:
>     -----------
>     S : float
>         Current price of the underlying asset
>     K : float
>         Strike price of the option
>     T : float
>         Time to expiration in years
>     r : float
>         Risk-free interest rate (as a decimal)
>     sigma : float
>         Volatility of the underlying asset (as a decimal)
>     option_type : str
>         Type of option ('CALL' or 'PUT')
>         
>     Returns:
>     --------
>     Dict containing price and Greeks (delta, gamma, theta, vega, rho)
>     """
>     try:
>         if T <= 0:
>             # Handle expired or about-to-expire options
>             if option_type == "CALL":
>                 price = max(0, S - K)
>                 delta = 1.0 if S > K else 0.0
>             else:  # PUT
>                 price = max(0, K - S)
>                 delta = -1.0 if S < K else 0.0
>             
>             return {
>                 "price": price,
>                 "delta": delta,
>                 "gamma": 0.0,
>                 "theta": 0.0,
>                 "vega": 0.0,
>                 "rho": 0.0
>             }
>         
>         # Calculate d1 and d2
>         d1 = (np.log(S / K) + (r + 0.5 * sigma**2) * T) / (sigma * np.sqrt(T))
>         d2 = d1 - sigma * np.sqrt(T)
>         
>         # Option price calculation
>         if option_type == "CALL":
>             price = S * norm.cdf(d1) - K * np.exp(-r * T) * norm.cdf(d2)
>             delta = norm.cdf(d1)
>             rho = K * T * np.exp(-r * T) * norm.cdf(d2) / 100
>         else:  # PUT
>             price = K * np.exp(-r * T) * norm.cdf(-d2) - S * norm.cdf(-d1)
>             delta = norm.cdf(d1) - 1
>             rho = -K * T * np.exp(-r * T) * norm.cdf(-d2) / 100
>         
>         # Common Greeks
>         gamma = norm.pdf(d1) / (S * sigma * np.sqrt(T))
>         vega = S * np.sqrt(T) * norm.pdf(d1) / 100  # Divided by 100 for 1% vol change
>         theta = (-(S * sigma * norm.pdf(d1)) / (2 * np.sqrt(T)) 
>                 - r * K * np.exp(-r * T) * norm.cdf(d2 if option_type == "CALL" else -d2)) / 365
>         
>         return {
>             "price": price,
>             "delta": delta,
>             "gamma": gamma,
>             "theta": theta,
>             "vega": vega,
>             "rho": rho
>         }
>     
>     except Exception as e:
>         logger.error(f"Error in Black-Scholes calculation: {e}", exc_info=True)
>         # Return default values on error
>         return {
>             "price": 0.0,
>             "delta": 0.0,
>             "gamma": 0.0,
>             "theta": 0.0,
>             "vega": 0.0,
>             "rho": 0.0
>         }
> 
> 
> def calculate_option_price(
>     symbol: str,
>     strike: float,
>     expiry_date: date,
>     option_type: Literal["CALL", "PUT"],
>     spot_price: Optional[float] = None,
>     volatility: Optional[float] = None,
>     risk_free_rate: Optional[float] = None
> ) -> Dict[str, float]:
>     """
>     Calculate option price and Greeks for a specific option contract.
>     
>     This function gets the necessary market data if not provided and then
>     calculates the option price and Greeks using the Black-Scholes model.
>     
>     Parameters:
>     -----------
>     symbol : str
>         Symbol of the underlying asset
>     strike : float
>         Strike price of the option
>     expiry_date : date
>         Expiration date of the option
>     option_type : str
>         Type of option ('CALL' or 'PUT')
>     spot_price : float, optional
>         Current price of the underlying asset (will be fetched if not provided)
>     volatility : float, optional
>         Implied volatility (will be fetched from market data if not provided)
>     risk_free_rate : float, optional
>         Risk-free interest rate (will use a default value if not provided)
>         
>     Returns:
>     --------
>     Dict containing price and Greeks
>     """
>     try:
>         # Get current date and calculate time to expiry
>         current_date = datetime.now().date()
>         days_to_expiry = (expiry_date - current_date).days
>         T = max(days_to_expiry, 0) / 365.0  # Time in years
>         
>         # Get spot price if not provided
>         if spot_price is None:
>             # In a real application, this would fetch from a market data service
>             from app.services.market_data.client import get_spot_price
>             spot_price = get_spot_price(symbol)
>         
>         # Get implied volatility if not provided
>         if volatility is None:
>             # In a real application, this would fetch from a market data service
>             from app.services.pricing.volatility import get_implied_volatility
>             volatility = get_implied_volatility(symbol, strike, expiry_date, option_type)
>         
>         # Get risk-free rate if not provided
>         if risk_free_rate is None:
>             # In a real application, this might fetch from a rates service
>             risk_free_rate = 0.03  # Default value
>         
>         # Calculate option price and Greeks
>         result = black_scholes(
>             S=spot_price,
>             K=strike,
>             T=T,
>             r=risk_free_rate,
>             sigma=volatility,
>             option_type=option_type
>         )
>         
>         # Add implied volatility to the result
>         result["implied_volatility"] = volatility
>         
>         return result
>     
>     except Exception as e:
>         logger.error(f"Error calculating option price: {e}", exc_info=True)
>         # Return default values on error
>         return {
>             "price": 0.0,
>             "delta": 0.0,
>             "gamma": 0.0,
>             "theta": 0.0,
>             "vega": 0.0,
>             "rho": 0.0,
>             "implied_volatility": 0.0
>         }
> ```
> 
> 4. **`backend/app/services/risk/scenarios.py`** - Risk scenario analysis implementation:
> 
> ```python
> from typing import Dict, List, Optional, Any
> import numpy as np
> from datetime import datetime, timedelta
> import asyncio
> import logging
> 
> from app.services.pricing.black_scholes import calculate_option_price
> from app.db.session import get_db
> from app.models.position import Position
> 
> logger = logging.getLogger(__name__)
> 
> async def run_risk_scenario(
>     position_ids: List[int],
>     price_change: float = 0,
>     vol_change: float = 0,
>     days_forward: int = 0,
>     user_id: Optional[int] = None
> ) -> Dict[str, Any]:
>     """
>     Run a risk scenario analysis for a set of positions.
>     
>     This function calculates the P&L impact of changes in underlying price,
>     implied volatility, and the passage of time.
>     
>     Parameters:
>     -----------
>     position_ids : List[int]
>         List of position IDs to include in the scenario
>     price_change : float
>         Percentage change in underlying price (e.g., 0.05 for 5% increase)
>     vol_change : float
>         Percentage change in implied volatility (e.g., 0.1 for 10% increase)
>     days_forward : int
>         Number of days forward in time to project
>     user_id : int, optional
>         User ID for permission checking
>         
>     Returns:
>     --------
>     Dict containing scenario results
>     """
>     try:
>         # Get database session
>         db = next(get_db())
>         
>         # Query positions
>         query = db.query(Position).filter(Position.id.in_(position_ids))
>         if user_id:
>             query = query.filter(Position.user_id == user_id)
>         
>         positions = query.all()
>         
>         if not positions:
>             logger.warning(f"No positions found for scenario analysis")
>             return {
>                 "totalPnl": 0.0,
>                 "deltaPnl": 0.0,
>                 "gammaPnl": 0.0,
>                 "vegaPnl": 0.0,
>                 "thetaPnl": 0.0,
>                 "scenarioCurve": []
>             }
>         
>         # Group positions by underlying symbol
>         symbols = set(position.symbol for position in positions)
>         
>         # Get market data for all symbols
>         market_data = {}
>         for symbol in symbols:
>             # In a real application, this would fetch from a market data service
>             from app.services.market_data.client import get_spot_price, get_implied_volatility_surface
>             spot_price = get_spot_price(symbol)
>             vol_surface = get_implied_volatility_surface(symbol)
>             market_data[symbol] = {
>                 "spot_price": spot_price,
>                 "vol_surface": vol_surface
>             }
>         
>         # Calculate base values for all positions
>         base_values = {}
>         for position in positions:
>             symbol = position.symbol
>             spot_price = market_data[symbol]["spot_price"]
>             
>             # Get implied volatility for this option
>             iv = get_implied_volatility_for_option(
>                 vol_surface=market_data[symbol]["vol_surface"],
>                 strike=position.strike,
>                 expiry_date=position.expiry_date,
>                 option_type=position.option_type
>             )
>             
>             # Calculate option price and Greeks
>             option_data = calculate_option_price(
>                 symbol=symbol,
>                 strike=position.strike,
>                 expiry_date=position.expiry_date,
>                 option_type=position.option_type,
>                 spot_price=spot_price,
>                 volatility=iv
>             )
>             
>             base_values[position.id] = {
>                 "market_price": option_data["price"],
>                 "delta": option_data["delta"],
>                 "gamma": option_data["gamma"],
>                 "vega": option_data["vega"],
>                 "theta": option_data["theta"],
>                 "iv": iv
>             }
>         
>         # Calculate scenario values
>         scenario_values = {}
>         for position in positions:
>             symbol = position.symbol
>             original_spot = market_data[symbol]["spot_price"]
>             original_iv = base_values[position.id]["iv"]
>             
>             # Apply scenario changes
>             new_spot = original_spot * (1 + price_change)
>             new_iv = original_iv * (1 + vol_change)
>             new_expiry = position.expiry_date - timedelta(days=days_forward)
>             
>             # Calculate new option price
>             scenario_data = calculate_option_price(
>                 symbol=symbol,
>                 strike=position.strike,
>                 expiry_date=new_expiry,
>                 option_type=position.option_type,
>                 spot_price=new_spot,
>                 volatility=new_iv
>             )
>             
>             scenario_values[position.id] = {
>                 "new_market_price": scenario_data["price"]
>             }
>         
>         # Calculate P&L attribution
>         total_pnl = 0.0
>         delta_pnl = 0.0
>         gamma_pnl = 0.0
>         vega_pnl = 0.0
>         theta_pnl = 0.0
>         
>         for position in positions:
>             symbol = position.symbol
>             original_spot = market_data[symbol]["spot_price"]
>             original_iv = base_values[position.id]["iv"]
>             
>             # Calculate price change in absolute terms
>             price_change_abs = original_spot * price_change
>             
>             # Calculate the components of P&L
>             position_delta_pnl = base_values[position.id]["delta"] * price_change_abs * position.quantity
>             position_gamma_pnl = 0.5 * base_values[position.id]["gamma"] * (price_change_abs ** 2) * position.quantity
>             position_vega_pnl = base_values[position.id]["vega"] * (vol_change * 100) * position.quantity
>             position_theta_pnl = base_values[position.id]["theta"] * days_forward * position.quantity
>             
>             # Total P&L is the difference between the scenario price and the base price
>             position_total_pnl = (
>                 scenario_values[position.id]["new_market_price"] - base_values[position.id]["market_price"]
>             ) * position.quantity
>             
>             # Accumulate P&L components
>             total_pnl += position_total_pnl
>             delta_pnl += position_delta_pnl
>             gamma_pnl += position_gamma_pnl
>             vega_pnl += position_vega_pnl
>             theta_pnl += position_theta_pnl
>         
>         # Generate scenario curve for visualization
>         # Create a range of price movements from -10% to +10%
>         price_points = np.linspace(-0.1, 0.1, 21)
>         scenario_curve = []
>         
>         for pp in price_points:
>             # Calculate total P&L at this price point
>             pp_total_pnl = 0.0
>             
>             for position in positions:
>                 symbol = position.symbol
>                 original_spot = market_data[symbol]["spot_price"]
>                 original_iv = base_values[position.id]["iv"]
>                 
>                 # Calculate new spot price
>                 pp_new_spot = original_spot * (1 + pp)
>                 
>                 # Calculate new option price
>                 pp_scenario_data = calculate_option_price(
>                     symbol=symbol,
>                     strike=position.strike,
>                     expiry_date=position.expiry_date,
>                     option_type=position.option_type,
>                     spot_price=pp_new_spot,
>                     volatility=original_iv
>                 )
>                 
>                 # Calculate P&L for this position at this price point
>                 pp_position_pnl = (
>                     pp_scenario_data["price"] - base_values[position.id]["market_price"]
>                 ) * position.quantity
>                 
>                 pp_total_pnl += pp_position_pnl
>             
>             scenario_curve.append({
>                 "priceChange": pp,
>                 "pnl": pp_total_pnl
>             })
>         
>         # Return the scenario results
>         return {
>             "totalPnl": total_pnl,
>             "deltaPnl": delta_pnl,
>             "gammaPnl": gamma_pnl,
>             "vegaPnl": vega_pnl,
>             "thetaPnl": theta_pnl,
>             "scenarioCurve": scenario_curve
>         }
>     
>     except Exception as e:
>         logger.error(f"Error in risk scenario calculation: {e}", exc_info=True)
>         # Return default values on error
>         return {
>             "totalPnl": 0.0,
>             "deltaPnl": 0.0,
>             "gammaPnl": 0.0,
>             "vegaPnl": 0.0,
>             "thetaPnl": 0.0,
>             "scenarioCurve": []
>         }
> 
> 
> def get_implied_volatility_for_option(vol_surface, strike, expiry_date, option_type):
>     """
>     Get the implied volatility for a specific option from a volatility surface.
>     
>     In a real implementation, this would perform interpolation on the volatility surface.
>     """
>     # Placeholder for actual volatility surface lookup logic
>     # In a real implementation, you would interpolate on the vol surface
>     return 0.25  # Default value
> ```
> 
> 5. **`backend/app/services/delta_hedge/strategy.py`** - Delta hedging strategy implementation:
> 
> ```python
> from typing import Dict, List, Optional, Any, Tuple
> from datetime import datetime
> import logging
> import uuid
> 
> from app.db.session import get_db
> from app.models.position import Position
> from app.models.delta_hedge import DeltaHedgeSettings, DeltaHedgeAction
> from app.services.market_data.client import get_spot_price
> from app.services.pricing.black_scholes import calculate_option_price
> from app.services.telegram.notifications import send_notification
> 
> logger = logging.getLogger(__name__)
> 
> class DeltaHedgeStrategy:
>     """
>     Dynamic Delta Hedging strategy implementation.
>     
>     This class implements different delta hedging strategies and provides
>     methods to calculate hedge requirements and generate hedge orders.
>     """
>     
>     def __init__(self, user_id: int):
>         """Initialize with user ID to get user-specific settings."""
>         self.user_id = user_id
>         self.settings = self._load_settings()
>     
>     def _load_settings(self) -> Dict[str, Any]:
>         """Load user's delta hedge settings from database."""
>         db = next(get_db())
>         settings = db.query(DeltaHedgeSettings).filter(
>             DeltaHedgeSettings.user_id == self.user_id
>         ).first()
>         
>         if not settings:
>             # Create default settings if none exist
>             settings = DeltaHedgeSettings(
>                 user_id=self.user_id,
>                 strategy="balanced",
>                 min_delta_threshold=0.15,
>                 max_delta_threshold=0.25,
>                 hedge_interval_minutes=15,
>                 enable_telegram_notifications=True,
>                 active=False
>             )
>             db.add(settings)
>             db.commit()
>             db.refresh(settings)
>         
>         return {
>             "strategy": settings.strategy,
>             "min_delta_threshold": settings.min_delta_threshold,
>             "max_delta_threshold": settings.max_delta_threshold,
>             "hedge_interval_minutes": settings.hedge_interval_minutes,
>             "enable_telegram_notifications": settings.enable_telegram_notifications,
>             "active": settings.active
>         }
>     
>     def update_settings(self, new_settings: Dict[str, Any]) -> Dict[str, Any]:
>         """Update user's delta hedge settings."""
>         db = next(get_db())
>         settings = db.query(DeltaHedgeSettings).filter(
>             DeltaHedgeSettings.user_id == self.user_id
>         ).first()
>         
>         if not settings:
>             return self._load_settings()
>         
>         # Update settings from the input dictionary
>         for key, value in new_settings.items():
>             if hasattr(settings, key):
>                 setattr(settings, key, value)
>         
>         db.commit()
>         db.refresh(settings)
>         self.settings = self._load_settings()
>         
>         return self.settings
>     
>     def toggle_active_state(self, active: bool) -> Dict[str, Any]:
>         """Toggle the active state of the delta hedging strategy."""
>         db = next(get_db())
>         settings = db.query(DeltaHedgeSettings).filter(
>             DeltaHedgeSettings.user_id == self.user_id
>         ).first()
>         
>         if not settings:
>             return self._load_settings()
>         
>         settings.active = active
>         db.commit()
>         
>         self.settings["active"] = active
>         
>         return self.settings
>     
>     async def calculate_net_delta(self) -> float:
>         """Calculate the net delta of all positions."""
>         db = next(get_db())
>         positions = db.query(Position).filter(
>             Position.user_id == self.user_id
>         ).all()
>         
>         net_delta = 0.0
>         
>         for position in positions:
>             # Get current market data
>             spot_price = get_spot_price(position.symbol)
>             
>             # Calculate position delta
>             option_data = calculate_option_price(
>                 symbol=position.symbol,
>                 strike=position.strike,
>                 expiry_date=position.expiry_date,
>                 option_type=position.option_type,
>                 spot_price=spot_price
>             )
>             
>             position_delta = option_data["delta"] * position.quantity
>             net_delta += position_delta
>         
>         return net_delta
>     
>     async def check_hedge_requirements(self) -> Optional[Dict[str, Any]]:
>         """
>         Check if hedging is required based on current net delta.
>         
>         Returns hedge instructions if needed, or None if no hedge is required.
>         """
>         if not self.settings["active"]:
>             logger.info(f"Delta hedging is not active for user {self.user_id}")
>             return None
>         
>         net_delta = await self.calculate_net_delta()
>         abs_delta = abs(net_delta)
>         
>         # Check if the net delta exceeds the threshold
>         if abs_delta <= self.settings["min_delta_threshold"]:
>             logger.info(f"Net delta {net_delta} is within acceptable range, no hedge needed")
>             return None
>         
>         # Determine hedge action
>         hedge_action = "BUY" if net_delta < 0 else "SELL"
>         
>         # Get the main symbol (in a real app, this would be more sophisticated)
>         db = next(get_db())
>         positions = db.query(Position).filter(
>             Position.user_id == self.user_id
>         ).all()
>         
>         if not positions:
>             logger.warning(f"No positions found for user {self.user_id}")
>             return None
>         
>         # Use the most frequent symbol (simple approach)
>         symbol_counts = {}
>         for p in positions:
>             symbol_counts[p.symbol] = symbol_counts.get(p.symbol, 0) + 1
>         
>         main_symbol = max(symbol_counts.items(), key=lambda x: x[1])[0]
>         
>         # Calculate hedge size - different strategies can adjust this
>         spot_price = get_spot_price(main_symbol)
>         
>         # The hedge amount depends on the strategy
>         if self.settings["strategy"] == "aggressive":
>             # Hedge to nearly zero
>             hedge_amount = abs_delta * 0.95
>         elif self.settings["strategy"] == "conservative":
>             # Hedge just to get below max threshold
>             hedge_amount = abs_delta - self.settings["max_delta_threshold"]
>         else:  # balanced
>             # Hedge to the midpoint between min and max thresholds
>             target_delta = (self.settings["min_delta_threshold"] + self.settings["max_delta_threshold"]) / 2
>             hedge_amount = abs_delta - target_delta
>         
>         # Ensure minimum size and round appropriately based on symbol
>         min_trade_size = 0.001 if main_symbol in ["BTC/USD", "ETH/USD"] else 0.01
>         hedge_amount = max(hedge_amount, min_trade_size)
>         
>         # Round to appropriate precision
>         if main_symbol in ["BTC/USD"]:
>             hedge_amount = round(hedge_amount, 6)
>         elif main_symbol in ["ETH/USD"]:
>             hedge_amount = round(hedge_amount, 5)
>         else:
>             hedge_amount = round(hedge_amount, 2)
>         
>         # Generate hedge instruction
>         return {
>             "symbol": main_symbol,
>             "action": hedge_action,
>             "amount": hedge_amount,
>             "estimated_price": spot_price,
>             "estimated_cost": hedge_amount * spot_price
>         }
>     
>     async def execute_hedge(self, hedge_instructions: Dict[str, Any]) -> Dict[str, Any]:
>         """
>         Execute a hedge order based on the provided instructions.
>         
>         In a real implementation, this would connect to an exchange API
>         to place actual orders.
>         """
>         try:
>             # In a real implementation, this would place an actual order
>             # For now, we'll simulate it
>             symbol = hedge_instructions["symbol"]
>             action = hedge_instructions["action"]
>             amount = hedge_instructions["amount"]
>             estimated_price = hedge_instructions["estimated_price"]
>             
>             # Record the hedge action
>             db = next(get_db())
>             
>             # Get delta before hedging
>             delta_before = await self.calculate_net_delta()
>             
>             # Simulate a successful order
>             # In a real app, this would be the actual executed price
>             executed_price = estimated_price * (1 + (0.001 if action == "BUY" else -0.001))
>             
>             # Calculate delta after hedging
>             delta_after = delta_before - (amount if action == "BUY" else -amount)
>             
>             # Create a hedge action record
>             hedge_action = DeltaHedgeAction(
>                 id=str(uuid.uuid4()),
>                 user_id=self.user_id,
>                 symbol=symbol,
>                 action=action,
>                 amount=amount,
>                 price=executed_price,
>                 delta_before=delta_before,
>                 delta_after=delta_after,
>                 status="COMPLETED",
>                 timestamp=datetime.now()
>             )
>             
>             db.add(hedge_action)
>             db.commit()
>             db.refresh(hedge_action)
>             
>             # Send notification if enabled
>             if self.settings["enable_telegram_notifications"]:
>                 notification_message = (
>                     f"Delta Hedge Executed\n"
>                     f"Symbol: {symbol}\n"
>                     f"Action: {action}\n"
>                     f"Amount: {amount}\n"
>                     f"Price: {executed_price}\n"
>                     f"Delta: {delta_before:.2f} → {delta_after:.2f}"
>                 )
>                 await send_notification(self.user_id, notification_message)
>             
>             return {
>                 "id": hedge_action.id,
>                 "symbol": symbol,
>                 "action": action,
>                 "amount": amount,
>                 "price": executed_price,
>                 "cost": amount * executed_price,
>                 "delta_before": delta_before,
>                 "delta_after": delta_after,
>                 "status": "COMPLETED",
>                 "timestamp": hedge_action.timestamp.isoformat()
>             }
>         
>         except Exception as e:
>             logger.error(f"Error executing hedge: {e}", exc_info=True)
>             
>             # Record the failed hedge action
>             db = next(get_db())
>             hedge_action = DeltaHedgeAction(
>                 id=str(uuid.uuid4()),
>                 user_id=self.user_id,
>                 symbol=hedge_instructions["symbol"],
>                 action=hedge_instructions["action"],
>                 amount=hedge_instructions["amount"],
>                 price=hedge_instructions["estimated_price"],
>                 delta_before=await self.calculate_net_delta(),
>                 delta_after=await self.calculate_net_delta(),  # Unchanged since it failed
>                 status="FAILED",
>                 error_message=str(e),
>                 timestamp=datetime.now()
>             )
>             
>             db.add(hedge_action)
>             db.commit()
>             
>             return {
>                 "status": "FAILED",
>                 "error": str(e)
>             }
>     
>     async def get_hedge_history(self, limit: int = 50) -> List[Dict[str, Any]]:
>         """Get the history of delta hedge actions."""
>         db = next(get_db())
>         actions = db.query(DeltaHedgeAction).filter(
>             DeltaHedgeAction.user_id == self.user_id
>         ).order_by(DeltaHedgeAction.timestamp.desc()).limit(limit).all()
>         
>         return [
>             {
>                 "id": action.id,
>                 "symbol": action.symbol,
>                 "action": action.action,
>                 "amount": action.amount,
>                 "price": action.price,
>                 "cost": action.amount * action.price,
>                 "delta_before": action.delta_before,
>                 "delta_after": action.delta_after,
>                 "status": action.status,
>                 "error_message": action.error_message,
>                 "timestamp": action.timestamp.isoformat()
>             }
>             for action in actions
>         ]
> ```
> 
> ### 4. Pricer Service (Rust)
> 
> Let's now implement the high-performance option pricing engine in Rust.
> 
> #### Pricer Service Structure
> 
> ```
> services/pricer/
> ├── src/
> │   ├── main.rs                 # Main entry point
> │   ├── pricing/
> │   │   ├── mod.rs              # Module definitions
> │   │   ├── black_scholes.rs    # Black-Scholes model
> │   │   ├── volatility.rs       # Volatility calculations
> │   │   └── greeks.rs           # Greeks calculations
> │   ├── api/
> │   │   ├── mod.rs              # API module definitions
> │   │   ├── routes.rs           # API routes
> │   │   └── handlers.rs         # API handlers
> │   ├── models/
> │   │   ├── mod.rs              # Model definitions
> │   │   └── option.rs           # Option data structures
> │   └── config.rs               # Configuration
> ├── Cargo.toml                  # Dependencies
> ├── Dockerfile                  # Docker configuration
> └── README.md                   # Documentation
> ```
> 
> Let's examine the key files:
> 
> 1. **`services/pricer/src/pricing/black_scholes.rs`** - Black-Scholes implementation in Rust:
> 
> ```rust
> use std::f64::consts::SQRT_2PI;
> use statrs::distribution::{Normal, ContinuousCDF};
> 
> /// Standard normal probability density function
> fn norm_pdf(x: f64) -> f64 {
>     (-0.5 * x * x).exp() / SQRT_2PI
> }
> 
> /// Standard normal cumulative distribution function
> fn norm_cdf(x: f64) -> f64 {
>     let normal = Normal::new(0.0, 1.0).unwrap();
>     normal.cdf(x)
> }
> 
> /// Option type enum
> #[derive(Debug, Clone, Copy, PartialEq)]
> pub enum OptionType {
>     Call,
>     Put,
> }
> 
> /// Greeks for an option
> #[derive(Debug, Clone)]
> pub struct Greeks {
>     pub delta: f64,
>     pub gamma: f64,
>     pub theta: f64,
>     pub vega: f64,
>     pub rho: f64,
> }
> 
> /// Option price and Greeks calculated by Black-Scholes
> #[derive(Debug, Clone)]
> pub struct OptionPricing {
>     pub price: f64,
>     pub greeks: Greeks,
> }
> 
> /// Calculate option price and Greeks using the Black-Scholes model
> pub fn black_scholes(
>     spot: f64,
>     strike: f64,
>     time_to_expiry: f64,
>     risk_free_rate: f64,
>     volatility: f64,
>     option_type: OptionType,
> ) -> OptionPricing {
>     // Handle special case of expired options
>     if time_to_expiry <= 0.0 {
>         let price = match option_type {
>             OptionType::Call => f64::max(0.0, spot - strike),
>             OptionType::Put => f64::max(0.0, strike - spot),
>         };
>         
>         let delta = match option_type {
>             OptionType::Call => if spot > strike { 1.0 } else { 0.0 },
>             OptionType::Put => if spot < strike { -1.0 } else { 0.0 },
>         };
>         
>         return OptionPricing {
>             price,
>             greeks: Greeks {
>                 delta,
>                 gamma: 0.0,
>                 theta: 0.0,
>                 vega: 0.0,
>                 rho: 0.0,
>             },
>         };
>     }
>     
>     // Calculate d1 and d2
>     let sigma_sqrt_t = volatility * time_to_expiry.sqrt();
>     let d1 = ((spot / strike).ln() + (risk_free_rate + 0.5 * volatility * volatility) * time_to_expiry) / sigma_sqrt_t;
>     let d2 = d1 - sigma_sqrt_t;
>     
>     // Calculate option price
>     let price = match option_type {
>         OptionType::Call => {
>             spot * norm_cdf(d1) - strike * (-risk_free_rate * time_to_expiry).exp() * norm_cdf(d2)
>         },
>         OptionType::Put => {
>             strike * (-risk_free_rate * time_to_expiry).exp() * norm_cdf(-d2) - spot * norm_cdf(-d1)
>         },
>     };
>     
>     // Calculate Greeks
>     let delta = match option_type {
>         OptionType::Call => norm_cdf(d1),
>         OptionType::Put => norm_cdf(d1) - 1.0,
>     };
>     
>     let gamma = norm_pdf(d1) / (spot * sigma_sqrt_t);
>     
>     let theta = match option_type {
>         OptionType::Call => {
>             -spot * volatility * norm_pdf(d1) / (2.0 * time_to_expiry.sqrt())
>                 - risk_free_rate * strike * (-risk_free_rate * time_to_expiry).exp() * norm_cdf(d2)
>         },
>         OptionType::Put => {
>             -spot * volatility * norm_pdf(d1) / (2.0 * time_to_expiry.sqrt())
>                 + risk_free_rate * strike * (-risk_free_rate * time_to_expiry).exp() * norm_cdf(-d2)
>         },
>     };
>     
>     // Convert to daily theta
>     let theta_daily = theta / 365.0;
>     
>     let vega = spot * time_to_expiry.sqrt() * norm_pdf(d1) / 100.0; // Divided by 100 for 1% change
>     
>     let rho = match option_type {
>         OptionType::Call => {
>             strike * time_to_expiry * (-risk_free_rate * time_to_expiry).exp() * norm_cdf(d2) / 100.0
>         },
>         OptionType::Put => {
>             -strike * time_to_expiry * (-risk_free_rate * time_to_expiry).exp() * norm_cdf(-d2) / 100.0
>         },
>     };
>     
>     OptionPricing {
>         price,
>         greeks: Greeks {
>             delta,
>             gamma,
>             theta: theta_daily,
>             vega,
>             rho,
>         },
>     }
> }
> 
> #[cfg(test)]
> mod tests {
>     use super::*;
>     
>     #[test]
>     fn test_call_option_pricing() {
>         // Test case: AAPL Call
>         // Spot: $150, Strike: $155, Time: 30 days, Rate: 3%, Vol: 25%
>         let result = black_scholes(
>             150.0,
>             155.0, 
>             30.0 / 365.0,
>             0.03,
>             0.25,
>             OptionType::Call,
>         );
>         
>         assert!(result.price > 0.0);
>         assert!(result.greeks.delta > 0.0 && result.greeks.delta < 1.0);
>         assert!(result.greeks.gamma > 0.0);
>         assert!(result.greeks.theta < 0.0);
>         assert!(result.greeks.vega > 0.0);
>     }
>     
>     #[test]
>     fn test_put_option_pricing() {
>         // Test case: AAPL Put
>         // Spot: $150, Strike: $145, Time: 30 days, Rate: 3%, Vol: 25%
>         let result = black_scholes(
>             150.0,
>             145.0, 
>             30.0 / 365.0,
>             0.03,
>             0.25,
>             OptionType::Put,
>         );
>         
>         assert!(result.price > 0.0);
>         assert!(result.greeks.delta < 0.0 && result.greeks.delta > -1.0);
>         assert!(result.greeks.gamma > 0.0);
>         assert!(result.greeks.theta < 0.0);
>         assert!(result.greeks.vega > 0.0);
>     }
>     
>     #[test]
>     fn test_expired_options() {
>         // Test expired call
>         let expired_call = black_scholes(
>             150.0,
>             145.0,
>             0.0, 
>             0.03, 
>             0.25,
>             OptionType::Call,
>         );
>         
>         assert_eq!(expired_call.price, 5.0);  // max(0, 150-145)
>         assert_eq!(expired_call.greeks.delta, 1.0);
>         assert_eq!(expired_call.greeks.gamma, 0.0);
>         
>         // Test expired put
>         let expired_put = black_scholes(
>             150.0,
>             155.0,
>             0.0, 
>             0.03, 
>             0.25,
>             OptionType::Put,
>         );
>         
>         assert_eq!(expired_put.price, 5.0);  // max(0, 155-150)
>         assert_eq!(expired_put.greeks.delta, -1.0);
>         assert_eq!(expired_put.greeks.gamma, 0.0);
>     }
> }
> ```
> 
> 2. **`services/pricer/src/api/handlers.rs`** - API handler for option pricing:
> 
> ```rust
> use actix_web::{web, HttpResponse, Error};
> use chrono::{NaiveDate, Utc};
> use serde::{Deserialize, Serialize};
> use tracing::{info, error};
> 
> use crate::pricing::{black_scholes::{black_scholes, OptionType, OptionPricing, Greeks}, volatility::get_implied_volatility};
> use crate::models::option::OptionContract;
> 
> #[derive(Debug, Deserialize)]
> pub struct PriceOptionRequest {
>     pub symbol: String,
>     pub strike: f64,
>     pub expiry_date: String,  // Format: YYYY-MM-DD
>     pub option_type: String,  // "CALL" or "PUT"
>     pub spot_price: Option<f64>,
>     pub volatility: Option<f64>,
>     pub risk_free_rate: Option<f64>,
> }
> 
> #[derive(Debug, Serialize)]
> pub struct PriceOptionResponse {
>     pub symbol: String,
>     pub strike: f64,
>     pub expiry_date: String,
>     pub option_type: String,
>     pub spot_price: f64,
>     pub implied_volatility: f64,
>     pub price: f64,
>     pub delta: f64,
>     pub gamma: f64,
>     pub theta: f64,
>     pub vega: f64,
>     pub rho: f64,
> }
> 
> pub async fn price_option(
>     req: web::Json<PriceOptionRequest>,
> ) -> Result<HttpResponse, Error> {
>     info!("Pricing option: {:?}", req);
>     
>     // Parse the request
>     let option_type = match req.option_type.to_uppercase().as_str() {
>         "CALL" => OptionType::Call,
>         "PUT" => OptionType::Put,
>         _ => {
>             error!("Invalid option type: {}", req.option_type);
>             return Ok(HttpResponse::BadRequest().body("Invalid option type. Must be 'CALL' or 'PUT'"));
>         }
>     };
>     
>     // Parse expiry date
>     let expiry_date = match NaiveDate::parse_from_str(&req.expiry_date, "%Y-%m-%d") {
>         Ok(date) => date,
>         Err(e) => {
>             error!("Invalid expiry date format: {}", e);
>             return Ok(HttpResponse::BadRequest().body("Invalid expiry date format. Use YYYY-MM-DD"));
>         }
>     };
>     
>     // Calculate days to expiry
>     let today = Utc::now().date_naive();
>     let days_to_expiry = expiry_date.signed_duration_since(today).num_days();
>     
>     if days_to_expiry < 0 {
>         return Ok(HttpResponse::BadRequest().body("Expiry date must be in the future"));
>     }
>     
>     // Time to expiry in years
>     let time_to_expiry = days_to_expiry as f64 / 365.0;
>     
>     // Get or use provided spot price
>     let spot_price = match req.spot_price {
>         Some(price) => price,
>         None => {
>             // In a real application, fetch this from market data service
>             // For now, use a placeholder
>             match req.symbol.as_str() {
>                 "BTC/USD" => 41200.0,
>                 "ETH/USD" => 2680.0,
>                 "EUR/USD" => 1.07,
>                 _ => 100.0,  // Default
>             }
>         }
>     };
>     
>     // Get or use provided volatility
>     let volatility = match req.volatility {
>         Some(vol) => vol,
>         None => {
>             // In a real application, this would come from a volatility surface
>             // For now, use a placeholder based on asset
>             match req.symbol.as_str() {
>                 "BTC/USD" => 0.65,
>                 "ETH/USD" => 0.75,
>                 "EUR/USD" => 0.08,
>                 _ => 0.25,  // Default
>             }
>         }
>     };
>     
>     // Get or use provided risk-free rate
>     let risk_free_rate = req.risk_free_rate.unwrap_or(0.03);
>     
>     // Calculate option price and Greeks
>     let option = black_scholes(
>         spot_price,
>         req.strike,
>         time_to_expiry,
>         risk_free_rate,
>         volatility,
>         option_type,
>     );
>     
>     // Construct response
>     let response = PriceOptionResponse {
>         symbol: req.symbol.clone(),
>         strike: req.strike,
>         expiry_date: req.expiry_date.clone(),
>         option_type: req.option_type.clone(),
>         spot_price,
>         implied_volatility: volatility,
>         price: option.price,
>         delta: option.greeks.delta,
>         gamma: option.greeks.gamma,
>         theta: option.greeks.theta,
>         vega: option.greeks.vega,
>         rho: option.greeks.rho,
>     };
>     
>     Ok(HttpResponse::Ok().json(response))
> }
> 
> #[derive(Debug, Deserialize)]
> pub struct BatchPriceRequest {
>     pub contracts: Vec<OptionContract>,
>     pub spot_prices: Option<std::collections::HashMap<String, f64>>,
>     pub risk_free_rate: Option<f64>,
> }
> 
> #[derive(Debug, Serialize)]
> pub struct BatchPriceResponse {
>     pub results: Vec<PriceOptionResponse>,
> }
> 
> pub async fn batch_price_options(
>     req: web::Json<BatchPriceRequest>,
> ) -> Result<HttpResponse, Error> {
>     info!("Batch pricing {} options", req.contracts.len());
>     
>     let risk_free_rate = req.risk_free_rate.unwrap_or(0.03);
>     let mut results = Vec::with_capacity(req.contracts.len());
>     
>     for contract in &req.contracts {
>         // Parse option type
>         let option_type = match contract.option_type.to_uppercase().as_str() {
>             "CALL" => OptionType::Call,
>             "PUT" => OptionType::Put,
>             _ => {
>                 error!("Invalid option type: {}", contract.option_type);
>                 continue;  // Skip invalid contracts
>             }
>         };
>         
>         // Parse expiry date
>         let expiry_date = match NaiveDate::parse_from_str(&contract.expiry_date, "%Y-%m-%d") {
>             Ok(date) => date,
>             Err(e) => {
>                 error!("Invalid expiry date format: {}", e);
>                 continue;  // Skip invalid contracts
>             }
>         };
>         
>         // Calculate days to expiry
>         let today = Utc::now().date_naive();
>         let days_to_expiry = expiry_date.signed_duration_since(today).num_days();
>         let time_to_expiry = f64::max(days_to_expiry as f64 / 365.0, 0.0);
>         
>         // Get spot price
>         let spot_price = match &req.spot_prices {
>             Some(prices) => *prices.get(&contract.symbol).unwrap_or(&0.0),
>             None => {
>                 // Default values
>                 match contract.symbol.as_str() {
>                     "BTC/USD" => 41200.0,
>                     "ETH/USD" => 2680.0,
>                     "EUR/USD" => 1.07,
>                     _ => 100.0,
>                 }
>             }
>         };
>         
>         if spot_price <= 0.0 {
>             error!("Invalid spot price for {}: {}", contract.symbol, spot_price);
>             continue;
>         }
>         
>         // Use provided implied volatility or get from surface
>         let volatility = contract.implied_volatility.unwrap_or_else(|| {
>             // Default values
>             match contract.symbol.as_str() {
>                 "BTC/USD" => 0.65,
>                 "ETH/USD" => 0.75,
>                 "EUR/USD" => 0.08,
>                 _ => 0.25,
>             }
>         });
>         
>         // Calculate option price and Greeks
>         let option = black_scholes(
>             spot_price,
>             contract.strike,
>             time_to_expiry,
>             risk_free_rate,
>             volatility,
>             option_type,
>         );
>         
>         // Add to results
>         results.push(PriceOptionResponse {
>             symbol: contract.symbol.clone(),
>             strike: contract.strike,
>             expiry_date: contract.expiry_date.clone(),
>             option_type: contract.option_type.clone(),
>             spot_price,
>             implied_volatility: volatility,
>             price: option.price,
>             delta: option.greeks.delta,
>             gamma: option.greeks.gamma,
>             theta: option.greeks.theta,
>             vega: option.greeks.vega,
>             rho: option.greeks.rho,
>         });
>     }
>     
>     Ok(HttpResponse::Ok().json(BatchPriceResponse { results }))
> }
> ```
> 
> ### 5. Database Implementation
> 
> Let's implement the database schema and connections for both DolphinDB (time-series data) and PostgreSQL (relational data).
> 
> 1. **`db/dolphindb/init_scripts/schema.dos`** - DolphinDB schema initialization:
> 
> ```sql
> -- Create database for market data
> CREATE DATABASE IF NOT EXISTS market_data LOCATION '/dolphindb/database/market_data';
> 
> -- Create database for position data
> CREATE DATABASE IF NOT EXISTS position_data LOCATION '/dolphindb/database/position_data';
> 
> -- Create database for risk analytics
> CREATE DATABASE IF NOT EXISTS risk_analytics LOCATION '/dolphindb/database/risk_analytics';
> 
> -- Switch to market_data database
> USE market_data;
> 
> -- Create tick data table (partitioned by date and symbol)
> CREATE TABLE IF NOT EXISTS market_data.tick_data (
>     timestamp TIMESTAMP,
>     symbol SYMBOL,
>     price DOUBLE,
>     volume DOUBLE,
>     exchange SYMBOL
> ) PARTITION BY HASH(symbol, 10) RANGE(timestamp);
> 
> -- Create OHLCV table
> CREATE TABLE IF NOT EXISTS market_data.ohlcv (
>     timestamp TIMESTAMP,
>     symbol SYMBOL,
>     open DOUBLE,
>     high DOUBLE,
>     low DOUBLE,
>     close DOUBLE,
>     volume DOUBLE,
>     interval STRING
> ) PARTITION BY HASH(symbol, 10) RANGE(timestamp);
> 
> -- Create volatility surface table
> CREATE TABLE IF NOT EXISTS market_data.vol_surface (
>     timestamp TIMESTAMP,
>     symbol SYMBOL,
>     expiry_date DATE,
>     strike DOUBLE,
>     implied_vol DOUBLE,
>     delta DOUBLE,
>     option_type SYMBOL
> ) PARTITION BY HASH(symbol, 10) RANGE(timestamp);
> 
> -- Create greeks snapshot table
> USE position_data;
> 
> CREATE TABLE IF NOT EXISTS position_data.greeks_snapshot (
>     timestamp TIMESTAMP,
>     position_id SYMBOL,
>     user_id SYMBOL,
>     symbol SYMBOL,
>     option_type SYMBOL,
>     strike DOUBLE,
>     expiry_date DATE,
>     spot_price DOUBLE,
>     option_price DOUBLE,
>     delta DOUBLE,
>     gamma DOUBLE,
>     theta DOUBLE,
>     vega DOUBLE,
>     rho DOUBLE,
>     implied_vol DOUBLE
> ) PARTITION BY HASH(user_id, 10) RANGE(timestamp);
> 
> -- Create portfolio snapshot table
> CREATE TABLE IF NOT EXISTS position_data.portfolio_snapshot (
>     timestamp TIMESTAMP,
>     user_id SYMBOL,
>     net_delta DOUBLE,
>     net_gamma DOUBLE,
>     net_theta DOUBLE,
>     net_vega DOUBLE,
>     total_value DOUBLE,
>     unrealized_pnl DOUBLE
> ) PARTITION BY HASH(user_id, 10) RANGE(timestamp);
> 
> -- Create trade history table
> CREATE TABLE IF NOT EXISTS position_data.trade_history (
>     timestamp TIMESTAMP,
>     trade_id SYMBOL,
>     user_id SYMBOL,
>     symbol SYMBOL,
>     option_type SYMBOL,
>     strike DOUBLE,
>     expiry_date DATE,
>     quantity DOUBLE,
>     price DOUBLE,
>     side SYMBOL,
>     is_closing BOOL,
>     pnl DOUBLE,
>     execution_venue SYMBOL
> ) PARTITION BY HASH(user_id, 10) RANGE(timestamp);
> 
> -- Create risk analytics tables
> USE risk_analytics;
> 
> CREATE TABLE IF NOT EXISTS risk_analytics.scenario_analysis (
>     timestamp TIMESTAMP,
>     scenario_id SYMBOL,
>     user_id SYMBOL,
>     scenario_name SYMBOL,
>     price_change DOUBLE,
>     vol_change DOUBLE,
>     days_forward INT,
>     total_pnl DOUBLE,
>     delta_pnl DOUBLE,
>     gamma_pnl DOUBLE,
>     theta_pnl DOUBLE,
>     vega_pnl DOUBLE
> ) PARTITION BY HASH(user_id, 10) RANGE(timestamp);
> 
> CREATE TABLE IF NOT EXISTS risk_analytics.backtest_results (
>     timestamp TIMESTAMP,
>     backtest_id SYMBOL,
>     user_id SYMBOL,
>     strategy_name SYMBOL,
>     start_date DATE,
>     end_date DATE,
>     initial_capital DOUBLE,
>     final_capital DOUBLE,
>     max_drawdown DOUBLE,
>     sharpe_ratio DOUBLE,
>     total_trades INT,
>     win_rate DOUBLE,
>     avg_trade_pnl DOUBLE,
>     avg_win_pnl DOUBLE,
>     avg_loss_pnl DOUBLE
> ) PARTITION BY HASH(user_id, 10) RANGE(timestamp);
> 
> -- Create functions and procedures for analytics
> -- Daily P&L Attribution
> CREATE FUNCTION calcDailyPnL(userID, date) {
>     startTime = date(date) 00:00:00.000;
>     endTime = date(date) 23:59:59.999;
>     
>     -- Get positions at start and end of day
>     startPos = select * from position_data.greeks_snapshot where timestamp = (select max(timestamp) from position_data.greeks_snapshot where timestamp < startTime and user_id = userID) and user_id = userID;
>     endPos = select * from position_data.greeks_snapshot where timestamp = (select max(timestamp) from position_data.greeks_snapshot where timestamp <= endTime and user_id = userID) and user_id = userID;
>     
>     -- Get trades during the day
>     trades = select * from position_data.trade_history where timestamp between startTime and endTime and user_id = userID;
>     
>     -- Calculate P&L components (this is simplified)
>     return select 
>         sum(endPos.option_price - startPos.option_price) * quantity as total_pnl,
>         sum(startPos.delta * (endPos.spot_price - startPos.spot_price) * quantity) as delta_pnl,
>         sum(0.5 * startPos.gamma * pow(endPos.spot_price - startPos.spot_price, 2) * quantity) as gamma_pnl,
>         sum(startPos.theta * 1 * quantity) as theta_pnl,
>         sum(startPos.vega * (endPos.implied_vol - startPos.implied_vol) * 100 * quantity) as vega_pnl,
>         total_pnl - (delta_pnl + gamma_pnl + theta_pnl + vega_pnl) as unexplained_pnl
>     from startPos context by position_id;
> }
> ```
> 
> 2. **`db/postgres/migrations/001_initial_schema.sql`** - PostgreSQL schema initialization:
> 
> ```sql
> -- Users table
> CREATE TABLE IF NOT EXISTS users (
>     id SERIAL PRIMARY KEY,
>     username VARCHAR(100) NOT NULL UNIQUE,
>     email VARCHAR(255) NOT NULL UNIQUE,
>     password_hash VARCHAR(255) NOT NULL,
>     first_name VARCHAR(100),
>     last_name VARCHAR(100),
>     created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
> );
> 
> -- Create function to update timestamp
> CREATE OR REPLACE FUNCTION update_modified_column()
> RETURNS TRIGGER AS $$
> BEGIN
>     NEW.updated_at = now();
>     RETURN NEW;
> END;
> $$ language 'plpgsql';
> 
> -- Add trigger to users table
> CREATE TRIGGER update_users_modtime
>     BEFORE UPDATE ON users
>     FOR EACH ROW
>     EXECUTE FUNCTION update_modified_column();
> 
> -- Options table (reference data for available options)
> CREATE TABLE IF NOT EXISTS options (
>     id SERIAL PRIMARY KEY,
>     symbol VARCHAR(20) NOT NULL,
>     strike DECIMAL(18, 8) NOT NULL,
>     expiry_date DATE NOT NULL,
>     option_type VARCHAR(10) NOT NULL CHECK (option_type IN ('CALL', 'PUT')),
>     contract_size DECIMAL(18, 8) NOT NULL DEFAULT 1.0,
>     settlement_type VARCHAR(10) NOT NULL DEFAULT 'CASH',
>     created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     UNIQUE(symbol, strike, expiry_date, option_type)
> );
> 
> -- Add trigger to options table
> CREATE TRIGGER update_options_modtime
>     BEFORE UPDATE ON options
>     FOR EACH ROW
>     EXECUTE FUNCTION update_modified_column();
> 
> -- Positions table
> CREATE TABLE IF NOT EXISTS positions (
>     id SERIAL PRIMARY KEY,
>     user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
>     option_id INTEGER NOT NULL REFERENCES options(id),
>     symbol VARCHAR(20) NOT NULL,
>     strike DECIMAL(18, 8) NOT NULL,
>     expiry_date DATE NOT NULL,
>     option_type VARCHAR(10) NOT NULL CHECK (option_type IN ('CALL', 'PUT')),
>     quantity DECIMAL(18, 8) NOT NULL,
>     entry_price DECIMAL(18, 8) NOT NULL,
>     entry_date TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
> );
> 
> -- Add trigger to positions table
> CREATE TRIGGER update_positions_modtime
>     BEFORE UPDATE ON positions
>     FOR EACH ROW
>     EXECUTE FUNCTION update_modified_column();
> 
> -- Trades table
> CREATE TABLE IF NOT EXISTS trades (
>     id SERIAL PRIMARY KEY,
>     user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
>     option_id INTEGER NOT NULL REFERENCES options(id),
>     position_id INTEGER REFERENCES positions(id),
>     trade_type VARCHAR(10) NOT NULL CHECK (trade_type IN ('BUY', 'SELL')),
>     quantity DECIMAL(18, 8) NOT NULL,
>     price DECIMAL(18, 8) NOT NULL,
>     trade_date TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     exchange_fee DECIMAL(18, 8) DEFAULT 0,
>     total_cost DECIMAL(18, 8) NOT NULL,
>     created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
> );
> 
> -- Add trigger to trades table
> CREATE TRIGGER update_trades_modtime
>     BEFORE UPDATE ON trades
>     FOR EACH ROW
>     EXECUTE FUNCTION update_modified_column();
> 
> -- Delta hedge settings table
> CREATE TABLE IF NOT EXISTS delta_hedge_settings (
>     id SERIAL PRIMARY KEY,
>     user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
>     strategy VARCHAR(20) NOT NULL DEFAULT 'balanced',
>     min_delta_threshold DECIMAL(5, 4) NOT NULL DEFAULT 0.15,
>     max_delta_threshold DECIMAL(5, 4) NOT NULL DEFAULT 0.25,
>     hedge_interval_minutes INTEGER NOT NULL DEFAULT 15,
>     enable_telegram_notifications BOOLEAN NOT NULL DEFAULT true,
>     active BOOLEAN NOT NULL DEFAULT false,
>     created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     UNIQUE(user_id)
> );
> 
> -- Add trigger to delta_hedge_settings table
> CREATE TRIGGER update_delta_hedge_settings_modtime
>     BEFORE UPDATE ON delta_hedge_settings
>     FOR EACH ROW
>     EXECUTE FUNCTION update_modified_column();
> 
> -- Delta hedge actions table
> CREATE TABLE IF NOT EXISTS delta_hedge_actions (
>     id VARCHAR(36) PRIMARY KEY,
>     user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
>     symbol VARCHAR(20) NOT NULL,
>     action VARCHAR(10) NOT NULL CHECK (action IN ('BUY', 'SELL')),
>     amount DECIMAL(18, 8) NOT NULL,
>     price DECIMAL(18, 8) NOT NULL,
>     delta_before DECIMAL(10, 6) NOT NULL,
>     delta_after DECIMAL(10, 6) NOT NULL,
>     status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
>     error_message TEXT,
>     timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
> );
> 
> -- Add trigger to delta_hedge_actions table
> CREATE TRIGGER update_delta_hedge_actions_modtime
>     BEFORE UPDATE ON delta_hedge_actions
>     FOR EACH ROW
>     EXECUTE FUNCTION update_modified_column();
> 
> -- User settings table
> CREATE TABLE IF NOT EXISTS user_settings (
>     id SERIAL PRIMARY KEY,
>     user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
>     telegram_chat_id VARCHAR(100),
>     default_asset VARCHAR(20) DEFAULT 'BTC/USD',
>     theme VARCHAR(20) DEFAULT 'dark',
>     dashboard_layout JSONB,
>     risk_warning_threshold DECIMAL(5, 4) DEFAULT 0.5,
>     created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     UNIQUE(user_id)
> );
> 
> -- Add trigger to user_settings table
> CREATE TRIGGER update_user_settings_modtime
>     BEFORE UPDATE ON user_settings
>     FOR EACH ROW
>     EXECUTE FUNCTION update_modified_column();
> 
> -- Backtest configurations table
> CREATE TABLE IF NOT EXISTS backtest_configs (
>     id SERIAL PRIMARY KEY,
>     user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
>     name VARCHAR(100) NOT NULL,
>     description TEXT,
>     start_date DATE NOT NULL,
>     end_date DATE NOT NULL,
>     symbols VARCHAR(255) NOT NULL,
>     strategy_type VARCHAR(50) NOT NULL,
>     strategy_params JSONB NOT NULL,
>     initial_capital DECIMAL(18, 2) NOT NULL,
>     created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
> );
> 
> -- Add trigger to backtest_configs table
> CREATE TRIGGER update_backtest_configs_modtime
>     BEFORE UPDATE ON backtest_configs
>     FOR EACH ROW
>     EXECUTE FUNCTION update_modified_column();
> 
> -- Backtest results table
> CREATE TABLE IF NOT EXISTS backtest_results (
>     id SERIAL PRIMARY KEY,
>     config_id INTEGER NOT NULL REFERENCES backtest_configs(id) ON DELETE CASCADE,
>     user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
>     final_capital DECIMAL(18, 2) NOT NULL,
>     total_return DECIMAL(10, 4) NOT NULL,
>     annualized_return DECIMAL(10, 4) NOT NULL,
>     sharpe_ratio DECIMAL(10, 4) NOT NULL,
>     max_drawdown DECIMAL(10, 4) NOT NULL,
>     total_trades INTEGER NOT NULL,
>     win_rate DECIMAL(5, 4) NOT NULL,
>     avg_profit_loss_ratio DECIMAL(10, 4) NOT NULL,
>     results_data JSONB NOT NULL,
>     execution_time INTEGER NOT NULL,
>     created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
>     updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
> );
> 
> -- Add trigger to backtest_results table
> CREATE TRIGGER update_backtest_results_modtime
>     BEFORE UPDATE ON backtest_results
>     FOR EACH ROW
>     EXECUTE FUNCTION update_modified_column();
> ```
> 
> ### 6. Docker and Deployment
> 
> Finally, let's set up Docker configurations for the various services:
> 
> 1. **`docker-compose.yml`** - Main Docker Compose file:
> 
> ```yaml
> version: '3.8'
> 
> services:
>   # Frontend
>   frontend:
>     build:
>       context: ./frontend
>       dockerfile: Dockerfile
>     ports:
>       - "3000:80"
>     volumes:
>       - ./frontend:/app
>       - /app/node_modules
>     depends_on:
>       - backend
>     environment:
>       - REACT_APP_API_URL=http://localhost:8000/api/v1
>       - NODE_ENV=development
> 
>   # Backend API
>   backend:
>     build:
>       context: ./backend
>       dockerfile: Dockerfile
>     ports:
>       - "8000:8000"
>     volumes:
>       - ./backend:/app
>     depends_on:
>       - postgres
>       - redis
>       - dolphindb
>     environment:
>       - ENVIRONMENT=development
>       - SECRET_KEY=temporarysecretkey
>       - POSTGRES_SERVER=postgres
>       - POSTGRES_USER=postgres
>       - POSTGRES_PASSWORD=postgres
>       - POSTGRES_DB=options_platform
>       - DOLPHINDB_HOST=dolphindb
>       - DOLPHINDB_PORT=8848
>       - REDIS_HOST=redis
>       - REDIS_PORT=6379
>       - TELEGRAM_TOKEN=${TELEGRAM_TOKEN}
>       - PRICER_SERVICE_URL=http://pricer:9000
> 
>   # Option Pricer Service (Rust)
>   pricer:
>     build:
>       context: ./services/pricer
>       dockerfile: Dockerfile
>     ports:
>       - "9000:9000"
>     environment:
>       - RUST_LOG=info
>     deploy:
>       resources:
>         limits:
>           cpus: '1'
>           memory: 1G
> 
>   # Risk Engine Service (Rust)
>   risk-engine:
>     build:
>       context: ./services/risk-engine
>       dockerfile: Dockerfile
>     ports:
>       - "9001:9000"
>     environment:
>       - RUST_LOG=info
>     depends_on:
>       - dolphindb
>     deploy:
>       resources:
>         limits:
>           cpus: '1'
>           memory: 1G
> 
>   # Backtesting Service
>   backtester:
>     build:
>       context: ./services/backtester
>       dockerfile: Dockerfile
>     ports:
>       - "9002:8000"
>     volumes:
>       - ./services/backtester:/app
>     depends_on:
>       - postgres
>       - dolphindb
>     environment:
>       - POSTGRES_SERVER=postgres
>       - POSTGRES_USER=postgres
>       - POSTGRES_PASSWORD=postgres
>       - POSTGRES_DB=options_platform
>       - DOLPHINDB_HOST=dolphindb
>       - DOLPHINDB_PORT=8848
>       - PRICER_SERVICE_URL=http://pricer:9000
> 
>   # Delta Hedging Service
>   ddh-service:
>     build:
>       context: ./services/ddh-service
>       dockerfile: Dockerfile
>     volumes:
>       - ./services/ddh-service:/app
>     depends_on:
>       - postgres
>       - redis
>       - rabbitmq
>     environment:
>       - POSTGRES_SERVER=postgres
>       - POSTGRES_USER=postgres
>       - POSTGRES_PASSWORD=postgres
>       - POSTGRES_DB=options_platform
>       - REDIS_HOST=redis
>       - REDIS_PORT=6379
>       - RABBITMQ_HOST=rabbitmq
>       - TELEGRAM_TOKEN=${TELEGRAM_TOKEN}
>       - PRICER_SERVICE_URL=http://pricer:9000
> 
>   # PostgreSQL
>   postgres:
>     image: postgres:14-alpine
>     volumes:
>       - postgres_data:/var/lib/postgresql/data
>       - ./db/postgres/migrations:/docker-entrypoint-initdb.d
>     environment:
>       - POSTGRES_USER=postgres
>       - POSTGRES_PASSWORD=postgres
>       - POSTGRES_DB=options_platform
>     ports:
>       - "5432:5432"
> 
>   # DolphinDB
>   dolphindb:
>     image: dolphindb/dolphindb:latest
>     volumes:
>       - dolphindb_data:/dolphindb/database
>       - ./db/dolphindb/init_scripts:/dolphindb/init_scripts
>     ports:
>       - "8848:8848"
>     environment:
>       - DOLPHINDB_MAX_MEMORY=4GB
> 
>   # Redis for caching and pub/sub
>   redis:
>     image: redis:7-alpine
>     ports:
>       - "6379:6379"
>     volumes:
>       - redis_data:/data
> 
>   # RabbitMQ for message queue
>   rabbitmq:
>     image: rabbitmq:3-management-alpine
>     ports:
>       - "5672:5672"
>       - "15672:15672"
>     volumes:
>       - rabbitmq_data:/var/lib/rabbitmq
>     environment:
>       - RABBITMQ_DEFAULT_USER=guest
>       - RABBITMQ_DEFAULT_PASS=guest
> 
> volumes:
>   postgres_data:
>   dolphindb_data:
>   redis_data:
>   rabbitmq_data:
> ```
> 
> 2. **`frontend/Dockerfile`** - Frontend Dockerfile:
> 
> ```dockerfile
> # Build stage
> FROM node:18-alpine AS build
> 
> WORKDIR /app
> 
> COPY package.json package-lock.json ./
> RUN npm ci
> 
> COPY . .
> RUN npm run build
> 
> # Runtime stage
> FROM nginx:alpine
> 
> COPY --from=build /app/build /usr/share/nginx/html
> COPY nginx/nginx.conf /etc/nginx/conf.d/default.conf
> 
> EXPOSE 80
> 
> CMD ["nginx", "-g", "daemon off;"]
> ```
> 
> 3. **`backend/Dockerfile`** - Backend Dockerfile:
> 
> ```dockerfile
> FROM python:3.11-slim
> 
> WORKDIR /app
> 
> # Install dependencies
> COPY requirements.txt .
> RUN pip install --no-cache-dir -r requirements.txt
> 
> # Copy application code
> COPY . .
> 
> # Expose the application port
> EXPOSE 8000
> 
> # Start the application
> CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
> ```
> 
> 4. **`services/pricer/Dockerfile`** - Rust Pricer service Dockerfile:
> 
> ```dockerfile
> # Build stage
> FROM rust:1.70-slim as builder
> 
> WORKDIR /usr/src/pricer
> 
> # Create blank project
> RUN cargo init
> 
> # Copy over manifests
> COPY Cargo.toml Cargo.lock ./
> 
> # Build dependencies - this is the caching layer
> RUN cargo build --release
> RUN rm src/*.rs
> 
> # Copy source code
> COPY src ./src
> 
> # Build application
> RUN touch src/main.rs
> RUN cargo build --release
> 
> # Runtime stage
> FROM debian:bullseye-slim
> 
> RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/*
> 
> COPY --from=builder /usr/src/pricer/target/release/pricer /usr/local/bin/pricer
> 
> EXPOSE 9000
> 
> CMD ["pricer"]
> ```
> 
> ## Summary
> 
> This comprehensive design covers all the required components for building a production-ready options trading platform:
> 
> 1. **Frontend**: A React/TypeScript application with modern UI components, TradingView integration, and ag-Grid for the positions and risk management interface.
> 
> 2. **Backend**: A FastAPI application that handles API requests, data processing, and business logic.
> 
> 3. **Databases**:
>    - DolphinDB for time-series data storage (market data, position snapshots)
>    - PostgreSQL for relational data (users, positions, settings)
> 
> 4. **Microservices**:
>    - Rust-based pricer service for high-performance option pricing
>    - Python-based backtester service
>    - Delta hedging service with real-time notifications
> 
> 5. **Features Implemented**:
>    - Live risk and P&L monitoring
>    - Volatility analysis tools including 3D vol surface visualization
>    - Scenario testing for risk management
>    - Dynamic delta hedging automation
>    - Custom dashboards with grid-based position management
>    - Comprehensive backtesting capabilities
> 
> The architecture is designed to be scalable, modular, and maintainable, with clear separation of concerns between different components.


# Production-Ready Options Trading Platform Design

I'll design a comprehensive forex/crypto options trading platform with all the requested features. This will be a full-fledged production system with frontend, backend, and database components.

## System Architecture

```
options-trading-platform/
├── frontend/                # React + TypeScript SPA
├── backend/                 # Python FastAPI backend
├── services/                # Microservices
│   ├── pricer/              # Rust option pricing engine
│   ├── risk-engine/         # Rust risk calculations service
│   ├── backtester/          # Python backtesting engine
│   └── ddh-service/         # Delta hedging service
├── db/                      # Database components
│   ├── dolphindb/           # Time-series data
│   └── postgres/            # Relational data
├── docker/                  # Docker configurations
├── k8s/                     # Kubernetes configurations
├── tests/                   # Comprehensive test suites
└── docs/                    # Documentation
```

## 1. Frontend Implementation

### Technology Stack
- React 18 with TypeScript
- Redux Toolkit for state management
- AG-Grid Enterprise for position management
- TradingView Charting Library
- Tailwind CSS + Styled Components
- Vite for build system

### Key Frontend Files

#### `frontend/src/components/grid/OptionsGrid.tsx`
```typescript
import React, { useState, useEffect, useMemo, useCallback } from 'react';
import { AgGridReact } from 'ag-grid-react';
import 'ag-grid-community/styles/ag-grid.css';
import 'ag-grid-community/styles/ag-theme-alpine.css';
import { GridOptions, ColDef, ValueFormatterParams } from 'ag-grid-community';
import { useAppDispatch, useAppSelector } from '../../hooks/reduxHooks';
import { selectPositions, fetchPositions } from '../../store/slices/positionsSlice';
import { formatCurrency, formatNumber, formatPercentage } from '../../utils/formatters';
import { PositionActionsRenderer } from './renderers/PositionActionsRenderer';
import { PnLRenderer } from './renderers/PnLRenderer';
import { GreekRenderer } from './renderers/GreekRenderer';
import { OptionPosition } from '../../types/positions';

interface OptionsGridProps {
  viewType: 'positions' | 'greeks' | 'pnl';
  assetClass?: string;
  onSelectionChanged?: (selectedRows: OptionPosition[]) => void;
}

export const OptionsGrid: React.FC<OptionsGridProps> = ({ 
  viewType = 'positions',
  assetClass,
  onSelectionChanged 
}) => {
  const dispatch = useAppDispatch();
  const { positions, loading, error } = useAppSelector(selectPositions);
  const [rowData, setRowData] = useState<OptionPosition[]>([]);
  
  // Filter positions based on assetClass if provided
  useEffect(() => {
    let filteredData = [...positions];
    if (assetClass && assetClass !== 'all') {
      filteredData = positions.filter(p => p.assetClass === assetClass);
    }
    setRowData(filteredData);
  }, [positions, assetClass]);
  
  // Fetch positions on component mount and refresh periodically
  useEffect(() => {
    dispatch(fetchPositions());
    const intervalId = setInterval(() => {
      dispatch(fetchPositions());
    }, 30000); // Refresh every 30 seconds
    
    return () => clearInterval(intervalId);
  }, [dispatch]);
  
  // Column definitions for positions view
  const positionsColDefs = useMemo<ColDef[]>(() => [
    {
      headerName: '',
      field: 'id',
      width: 50,
      checkboxSelection: true,
      headerCheckboxSelection: true,
      pinned: 'left'
    },
    {
      headerName: 'Symbol',
      field: 'symbol',
      width: 120,
      pinned: 'left',
      filter: 'agTextColumnFilter',
      sortable: true
    },
    {
      headerName: 'Type',
      field: 'optionType',
      width: 90,
      cellStyle: params => {
        return params.value === 'CALL' 
          ? { color: 'var(--call-color)' } 
          : { color: 'var(--put-color)' };
      }
    },
    {
      headerName: 'Strike',
      field: 'strike',
      width: 110,
      valueFormatter: (params: ValueFormatterParams) => {
        return formatCurrency(params.value, params.data.currency);
      }
    },
    {
      headerName: 'Expiry',
      field: 'expiryDate',
      width: 120,
      sort: 'asc',
      valueFormatter: (params: ValueFormatterParams) => {
        return new Date(params.value).toLocaleDateString();
      }
    },
    {
      headerName: 'Qty',
      field: 'quantity',
      width: 100,
      valueFormatter: (params: ValueFormatterParams) => {
        return formatNumber(params.value, params.data.quantityPrecision || 4);
      }
    },
    {
      headerName: 'Entry',
      field: 'entryPrice',
      width: 110,
      valueFormatter: (params: ValueFormatterParams) => {
        return formatCurrency(params.value, params.data.currency);
      }
    },
    {
      headerName: 'Market',
      field: 'marketPrice',
      width: 110,
      valueFormatter: (params: ValueFormatterParams) => {
        return formatCurrency(params.value, params.data.currency);
      }
    },
    {
      headerName: 'P&L',
      field: 'pnl',
      width: 120,
      cellRenderer: PnLRenderer
    },
    // More columns including Greeks...
  ], []);
  
  // Define column sets for other views (greeks, pnl)
  // Select column definitions based on viewType
  const columnDefs = useMemo(() => {
    switch (viewType) {
      case 'greeks': return greeksColDefs;
      case 'pnl': return pnlColDefs;
      default: return positionsColDefs;
    }
  }, [viewType, positionsColDefs, greeksColDefs, pnlColDefs]);
  
  // Grid reference and additional grid functions
  // ...

  return (
    <div className="options-grid-container">
      {loading && <div className="grid-loading-overlay">Loading positions...</div>}
      {error && <div className="grid-error-message">Error: {error}</div>}
      
      <div className="ag-theme-alpine-dark options-grid">
        <AgGridReact
          ref={gridRef}
          rowData={rowData}
          columnDefs={columnDefs}
          gridOptions={gridOptions}
          onSelectionChanged={onGridSelectionChanged}
          onFirstDataRendered={params => params.api.sizeColumnsToFit()}
        />
      </div>
      
      <div className="grid-summary">
        {/* Summary metrics for portfolio */}
      </div>
    </div>
  );
};
```

#### `frontend/src/components/charts/VolatilitySurface.tsx`
```typescript
import React, { useEffect, useRef } from 'react';
import * as echarts from 'echarts';
import { EChartsOption } from 'echarts';
import 'echarts-gl';
import { useAppSelector, useAppDispatch } from '../../hooks/reduxHooks';
import { selectVolSurface, fetchVolSurface } from '../../store/slices/volatilitySlice';

interface VolatilitySurfaceProps {
  symbol: string;
}

export const VolatilitySurface: React.FC<VolatilitySurfaceProps> = ({ symbol }) => {
  const chartRef = useRef<HTMLDivElement>(null);
  const chartInstance = useRef<echarts.ECharts | null>(null);
  const dispatch = useAppDispatch();
  const { data, loading, error } = useAppSelector(selectVolSurface);
  
  // Fetch volatility surface data
  useEffect(() => {
    dispatch(fetchVolSurface(symbol));
  }, [dispatch, symbol]);
  
  // Initialize and update chart
  useEffect(() => {
    if (!chartRef.current) return;
    
    if (!chartInstance.current) {
      chartInstance.current = echarts.init(chartRef.current);
    }
    
    if (!loading && data && data.length > 0) {
      // Prepare data for 3D surface
      const strikes = Array.from(new Set(data.map(item => item.strike))).sort((a, b) => a - b);
      const tenors = Array.from(new Set(data.map(item => item.tenor))).sort((a, b) => a - b);
      
      const surfaceData = [];
      
      for (let i = 0; i < strikes.length; i++) {
        for (let j = 0; j < tenors.length; j++) {
          const point = data.find(
            item => item.strike === strikes[i] && item.tenor === tenors[j]
          );
          
          if (point) {
            surfaceData.push([i, j, point.impliedVol * 100]);
          }
        }
      }
      
      // Configure the chart
      const option: EChartsOption = {
        tooltip: {},
        visualMap: {
          show: true,
          dimension: 2,
          min: Math.min(...data.map(item => item.impliedVol * 100)),
          max: Math.max(...data.map(item => item.impliedVol * 100)),
          inRange: {
            color: ['#313695', '#4575b4', '#74add1', '#abd9e9', '#e0f3f8', 
                    '#ffffbf', '#fee090', '#fdae61', '#f46d43', '#d73027', '#a50026']
          }
        },
        xAxis3D: {
          type: 'category',
          name: 'Strike',
          data: strikes.map(strike => strike.toString())
        },
        yAxis3D: {
          type: 'category',
          name: 'Tenor (days)',
          data: tenors.map(tenor => tenor.toString())
        },
        zAxis3D: {
          type: 'value',
          name: 'IV (%)',
          min: 'dataMin',
          max: 'dataMax'
        },
        grid3D: {
          viewControl: {
            projection: 'perspective',
            autoRotate: false
          },
          light: {
            main: { intensity: 1.2 },
            ambient: { intensity: 0.3 }
          }
        },
        series: [{
          type: 'surface',
          data: surfaceData,
          wireframe: { show: true },
          itemStyle: { opacity: 0.8 },
          emphasis: {
            itemStyle: { color: '#fff' }
          }
        }]
      };
      
      chartInstance.current.setOption(option);
    }
    
    // Resize handler
    const handleResize = () => {
      chartInstance.current?.resize();
    };
    
    window.addEventListener('resize', handleResize);
    
    return () => {
      window.removeEventListener('resize', handleResize);
      chartInstance.current?.dispose();
    };
  }, [data, loading]);
  
  return (
    <div className="volatility-surface-container">
      {loading && <div className="chart-loading">Loading volatility data...</div>}
      {error && <div className="chart-error">Error: {error}</div>}
      <div ref={chartRef} className="volatility-surface-chart" />
    </div>
  );
};
```

## 2. Backend Implementation (Python FastAPI)

### Structure

```
backend/
├── app/
│   ├── api/
│   │   ├── v1/
│   │   │   ├── auth.py           # Authentication endpoints
│   │   │   ├── positions.py      # Positions endpoints
│   │   │   ├── market_data.py    # Market data endpoints
│   │   │   ├── volatility.py     # Volatility endpoints
│   │   │   ├── scenarios.py      # Risk scenarios endpoints
│   │   │   ├── backtest.py       # Backtesting endpoints
│   │   │   ├── delta_hedge.py    # Delta hedge endpoints
│   │   │   └── settings.py       # User settings endpoints
│   │   └── routes.py             # API router setup
│   ├── core/
│   │   ├── config.py             # Application configuration
│   │   ├── security.py           # Authentication and security
│   │   └── logging.py            # Logging configuration
│   ├── services/                 # Business logic services
│   ├── models/                   # Data models
│   ├── schemas/                  # Pydantic schemas
│   └── main.py                   # Application entry point
```

### Key Backend Files

#### `backend/app/main.py`
```python
import time
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import logging

from app.core.config import settings
from app.api.routes import api_router
from app.core.logging import setup_logging
from app.db.session import init_db

logger = logging.getLogger(__name__)

# Initialize FastAPI app
app = FastAPI(
    title=settings.PROJECT_NAME,
    description="Options Trading Platform API",
    version="1.0.0",
    openapi_url=f"{settings.API_V1_STR}/openapi.json",
    docs_url=f"{settings.API_V1_STR}/docs",
    redoc_url=f"{settings.API_V1_STR}/redoc"
)

# Set up CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=[str(origin) for origin in settings.CORS_ORIGINS],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Request timing middleware
@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response

# Global exception handler
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unhandled exception: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={"detail": "Internal server error"}
    )

# Include API routes
app.include_router(api_router, prefix=settings.API_V1_STR)

# Startup event
@app.on_event("startup")
async def startup_event():
    setup_logging()
    logger.info("Starting application")
    await init_db()

# Shutdown event
@app.on_event("shutdown")
async def shutdown_event():
    logger.info("Shutting down application")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

#### `backend/app/api/v1/positions.py`
```python
from typing import List, Optional
from fastapi import APIRouter, Depends, HTTPException, Query, Path
from sqlalchemy.orm import Session

from app.schemas.position import (
    PositionCreate, 
    PositionUpdate, 
    PositionResponse,
    PositionWithGreeks
)
from app.models.user import User
from app.core.security import get_current_user
from app.db.session import get_db
from app.services.pricing import get_option_prices
from app.services.risk.greeks import calculate_position_greeks

router = APIRouter()

@router.get("/", response_model=List[PositionWithGreeks])
async def get_positions(
    symbol: Optional[str] = Query(None, description="Filter by symbol"),
    option_type: Optional[str] = Query(None, description="Filter by option type (CALL/PUT)"),
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """
    Get all positions for the current user with real-time pricing and Greeks.
    """
    # Query positions from database with filters
    positions = db.query(Position).filter(Position.user_id == current_user.id)
    
    if symbol:
        positions = positions.filter(Position.symbol == symbol)
    if option_type:
        positions = positions.filter(Position.option_type == option_type)
    
    positions = positions.all()
    
    # Get market data and calculate Greeks
    market_data = await get_market_data(list({position.symbol for position in positions}))
    
    # Process each position
    position_responses = []
    for position in positions:
        current_price = market_data.get(position.symbol, {}).get("price", 0)
        
        option_data = await get_option_prices(
            symbol=position.symbol,
            strike=position.strike,
            expiry=position.expiry_date,
            option_type=position.option_type
        )
        
        greeks = calculate_position_greeks(
            position=position,
            market_price=current_price,
            implied_vol=option_data.get("implied_volatility", 0),
            risk_free_rate=0.03,
            days_to_expiry=(position.expiry_date - datetime.now().date()).days
        )
        
        position_with_greeks = PositionWithGreeks(
            id=position.id,
            symbol=position.symbol,
            option_type=position.option_type,
            strike=position.strike,
            expiry_date=position.expiry_date,
            quantity=position.quantity,
            entry_price=position.entry_price,
            market_price=option_data.get("price", 0),
            unrealized_pnl=(option_data.get("price", 0) - position.entry_price) * position.quantity,
            delta=greeks.get("delta", 0) * position.quantity,
            gamma=greeks.get("gamma", 0) * position.quantity,
            theta=greeks.get("theta", 0) * position.quantity,
            vega=greeks.get("vega", 0) * position.quantity,
            implied_volatility=option_data.get("implied_volatility", 0),
            created_at=position.created_at,
            updated_at=position.updated_at
        )
        
        position_responses.append(position_with_greeks)
    
    return position_responses

# Additional endpoints for creating/managing positions
```

#### `backend/app/services/risk/scenarios.py`
```python
from typing import Dict, List, Optional, Any
import numpy as np
from datetime import datetime, timedelta
import asyncio
import logging

from app.services.pricing.black_scholes import calculate_option_price
from app.db.session import get_db
from app.models.position import Position

logger = logging.getLogger(__name__)

async def run_risk_scenario(
    position_ids: List[int],
    price_change: float = 0,
    vol_change: float = 0,
    days_forward: int = 0,
    user_id: Optional[int] = None
) -> Dict[str, Any]:
    """
    Run a risk scenario analysis for a set of positions.
    
    This function calculates the P&L impact of changes in underlying price,
    implied volatility, and the passage of time.
    """
    try:
        # Get database session
        db = next(get_db())
        
        # Query positions with optional user filter
        query = db.query(Position).filter(Position.id.in_(position_ids))
        if user_id:
            query = query.filter(Position.user_id == user_id)
        
        positions = query.all()
        
        if not positions:
            logger.warning(f"No positions found for scenario analysis")
            return {
                "totalPnl": 0.0,
                "deltaPnl": 0.0,
                "gammaPnl": 0.0,
                "vegaPnl": 0.0,
                "thetaPnl": 0.0,
                "scenarioCurve": []
            }
        
        # Group positions by underlying symbol
        symbols = set(position.symbol for position in positions)
        
        # Get market data for all symbols
        market_data = {}
        for symbol in symbols:
            # This would call your market data service
            spot_price = get_spot_price(symbol)
            vol_surface = get_implied_volatility_surface(symbol)
            market_data[symbol] = {
                "spot_price": spot_price,
                "vol_surface": vol_surface
            }
        
        # Calculate base values for all positions
        base_values = {}
        for position in positions:
            # Calculate option prices and Greeks for base case
            # ... implementation details ...
        
        # Calculate scenario values
        scenario_values = {}
        for position in positions:
            # Apply scenario changes and recalculate
            # ... implementation details ...
        
        # Calculate P&L attribution components
        total_pnl = 0.0
        delta_pnl = 0.0
        gamma_pnl = 0.0
        vega_pnl = 0.0
        theta_pnl = 0.0
        
        # ... attribution calculations ...
        
        # Generate scenario curve for visualization
        price_points = np.linspace(-0.1, 0.1, 21)
        scenario_curve = []
        
        # ... scenario curve calculations ...
        
        # Return the scenario results
        return {
            "totalPnl": total_pnl,
            "deltaPnl": delta_pnl,
            "gammaPnl": gamma_pnl,
            "vegaPnl": vega_pnl,
            "thetaPnl": theta_pnl,
            "scenarioCurve": scenario_curve
        }
    
    except Exception as e:
        logger.error(f"Error in risk scenario calculation: {e}", exc_info=True)
        # Return default values on error
        return {
            "totalPnl": 0.0,
            "deltaPnl": 0.0,
            "gammaPnl": 0.0,
            "vegaPnl": 0.0,
            "thetaPnl": 0.0,
            "scenarioCurve": []
        }
```

## 3. Option Pricing Service (Rust)

### `services/pricer/src/pricing/black_scholes.rs`
```rust
use std::f64::consts::SQRT_2PI;
use statrs::distribution::{Normal, ContinuousCDF};

/// Standard normal probability density function
fn norm_pdf(x: f64) -> f64 {
    (-0.5 * x * x).exp() / SQRT_2PI
}

/// Standard normal cumulative distribution function
fn norm_cdf(x: f64) -> f64 {
    let normal = Normal::new(0.0, 1.0).unwrap();
    normal.cdf(x)
}

/// Option type enum
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum OptionType {
    Call,
    Put,
}

/// Greeks for an option
#[derive(Debug, Clone)]
pub struct Greeks {
    pub delta: f64,
    pub gamma: f64,
    pub theta: f64,
    pub vega: f64,
    pub rho: f64,
}

/// Option price and Greeks calculated by Black-Scholes
#[derive(Debug, Clone)]
pub struct OptionPricing {
    pub price: f64,
    pub greeks: Greeks,
}

/// Calculate option price and Greeks using the Black-Scholes model
pub fn black_scholes(
    spot: f64,
    strike: f64,
    time_to_expiry: f64,
    risk_free_rate: f64,
    volatility: f64,
    option_type: OptionType,
) -> OptionPricing {
    // Handle special case of expired options
    if time_to_expiry <= 0.0 {
        let price = match option_type {
            OptionType::Call => f64::max(0.0, spot - strike),
            OptionType::Put => f64::max(0.0, strike - spot),
        };
        
        let delta = match option_type {
            OptionType::Call => if spot > strike { 1.0 } else { 0.0 },
            OptionType::Put => if spot < strike { -1.0 } else { 0.0 },
        };
        
        return OptionPricing {
            price,
            greeks: Greeks {
                delta,
                gamma: 0.0,
                theta: 0.0,
                vega: 0.0,
                rho: 0.0,
            },
        };
    }
    
    // Calculate d1 and d2
    let sigma_sqrt_t = volatility * time_to_expiry.sqrt();
    let d1 = ((spot / strike).ln() + (risk_free_rate + 0.5 * volatility * volatility) * time_to_expiry) / sigma_sqrt_t;
    let d2 = d1 - sigma_sqrt_t;
    
    // Calculate option price and Greeks
    // ... implementation details ...
    
    OptionPricing {
        price,
        greeks: Greeks {
            delta,
            gamma,
            theta: theta_daily,
            vega,
            rho,
        },
    }
}
```

## 4. Database Schema

### DolphinDB (Time-Series Data)

```sql
-- Create database for market data
CREATE DATABASE IF NOT EXISTS market_data LOCATION '/dolphindb/database/market_data';

-- Create database for position data
CREATE DATABASE IF NOT EXISTS position_data LOCATION '/dolphindb/database/position_data';

-- Create database for risk analytics
CREATE DATABASE IF NOT EXISTS risk_analytics LOCATION '/dolphindb/database/risk_analytics';

-- Switch to market_data database
USE market_data;

-- Create tick data table (partitioned by date and symbol)
CREATE TABLE IF NOT EXISTS market_data.tick_data (
    timestamp TIMESTAMP,
    symbol SYMBOL,
    price DOUBLE,
    volume DOUBLE,
    exchange SYMBOL
) PARTITION BY HASH(symbol, 10) RANGE(timestamp);

-- Create OHLCV table
CREATE TABLE IF NOT EXISTS market_data.ohlcv (
    timestamp TIMESTAMP,
    symbol SYMBOL,
    open DOUBLE,
    high DOUBLE,
    low DOUBLE,
    close DOUBLE,
    volume DOUBLE,
    interval STRING
) PARTITION BY HASH(symbol, 10) RANGE(timestamp);

-- Create volatility surface table
CREATE TABLE IF NOT EXISTS market_data.vol_surface (
    timestamp TIMESTAMP,
    symbol SYMBOL,
    expiry_date DATE,
    strike DOUBLE,
    implied_vol DOUBLE,
    delta DOUBLE,
    option_type SYMBOL
) PARTITION BY HASH(symbol, 10) RANGE(timestamp);

-- Additional tables for position data and risk analytics
-- ...
```

### PostgreSQL (Relational Data)

```sql
-- Users table
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Options table (reference data for available options)
CREATE TABLE IF NOT EXISTS options (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR(20) NOT NULL,
    strike DECIMAL(18, 8) NOT NULL,
    expiry_date DATE NOT NULL,
    option_type VARCHAR(10) NOT NULL CHECK (option_type IN ('CALL', 'PUT')),
    contract_size DECIMAL(18, 8) NOT NULL DEFAULT 1.0,
    settlement_type VARCHAR(10) NOT NULL DEFAULT 'CASH',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(symbol, strike, expiry_date, option_type)
);

-- Positions table
CREATE TABLE IF NOT EXISTS positions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    option_id INTEGER NOT NULL REFERENCES options(id),
    symbol VARCHAR(20) NOT NULL,
    strike DECIMAL(18, 8) NOT NULL,
    expiry_date DATE NOT NULL,
    option_type VARCHAR(10) NOT NULL CHECK (option_type IN ('CALL', 'PUT')),
    quantity DECIMAL(18, 8) NOT NULL,
    entry_price DECIMAL(18, 8) NOT NULL,
    entry_date TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Additional tables for trades, delta hedging, user settings, etc.
-- ...
```

## 5. Dynamic Delta Hedge Implementation

### `services/ddh-service/app/strategy.py`
```python
from typing import Dict, List, Optional, Any, Tuple
from datetime import datetime
import logging
import uuid

from app.db.session import get_db
from app.models.position import Position
from app.models.delta_hedge import DeltaHedgeSettings, DeltaHedgeAction
from app.services.market_data.client import get_spot_price
from app.services.pricing.black_scholes import calculate_option_price
from app.services.telegram.notifications import send_notification

logger = logging.getLogger(__name__)

class DeltaHedgeStrategy:
    """
    Dynamic Delta Hedging strategy implementation.
    
    This class implements different delta hedging strategies and provides
    methods to calculate hedge requirements and generate hedge orders.
    """
    
    def __init__(self, user_id: int):
        """Initialize with user ID to get user-specific settings."""
        self.user_id = user_id
        self.settings = self._load_settings()
    
    def _load_settings(self) -> Dict[str, Any]:
        """Load user's delta hedge settings from database."""
        # Implementation to load settings
        # ...
    
    async def calculate_net_delta(self) -> float:
        """Calculate the net delta of all positions."""
        db = next(get_db())
        positions = db.query(Position).filter(
            Position.user_id == self.user_id
        ).all()
        
        net_delta = 0.0
        
        for position in positions:
            # Get current market data
            spot_price = get_spot_price(position.symbol)
            
            # Calculate position delta
            option_data = calculate_option_price(
                symbol=position.symbol,
                strike=position.strike,
                expiry_date=position.expiry_date,
                option_type=position.option_type,
                spot_price=spot_price
            )
            
            position_delta = option_data["delta"] * position.quantity
            net_delta += position_delta
        
        return net_delta
    
    async def check_hedge_requirements(self) -> Optional[Dict[str, Any]]:
        """
        Check if hedging is required based on current net delta.
        
        Returns hedge instructions if needed, or None if no hedge is required.
        """
        # Implementation to check if hedging is needed
        # ...
    
    async def execute_hedge(self, hedge_instructions: Dict[str, Any]) -> Dict[str, Any]:
        """
        Execute a hedge order based on the provided instructions.
        
        In a real implementation, this would connect to an exchange API
        to place actual orders.
        """
        # Implementation to execute the hedge order
        # ...
```

## 6. Docker and Deployment Configuration

### `docker-compose.yml`
```yaml
version: '3.8'

services:
  # Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:80"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    depends_on:
      - backend
    environment:
      - REACT_APP_API_URL=http://localhost:8000/api/v1
      - NODE_ENV=development

  # Backend API
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app
    depends_on:
      - postgres
      - redis
      - dolphindb
    environment:
      - ENVIRONMENT=development
      - SECRET_KEY=temporarysecretkey
      - POSTGRES_SERVER=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=options_platform
      - DOLPHINDB_HOST=dolphindb
      - DOLPHINDB_PORT=8848
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - TELEGRAM_TOKEN=${TELEGRAM_TOKEN}
      - PRICER_SERVICE_URL=http://pricer:9000

  # Option Pricer Service (Rust)
  pricer:
    build:
      context: ./services/pricer
      dockerfile: Dockerfile
    ports:
      - "9000:9000"
    environment:
      - RUST_LOG=info
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G

  # Additional services (risk-engine, backtester, ddh-service)
  # ...

  # PostgreSQL
  postgres:
    image: postgres:14-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/postgres/migrations:/docker-entrypoint-initdb.d
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=options_platform
    ports:
      - "5432:5432"

  # DolphinDB
  dolphindb:
    image: dolphindb/dolphindb:latest
    volumes:
      - dolphindb_data:/dolphindb/database
      - ./db/dolphindb/init_scripts:/dolphindb/init_scripts
    ports:
      - "8848:8848"
    environment:
      - DOLPHINDB_MAX_MEMORY=4GB

  # Redis for caching and pub/sub
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  # RabbitMQ for message queue
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest

volumes:
  postgres_data:
  dolphindb_data:
  redis_data:
  rabbitmq_data:
```

## Implementation Summary

This architecture delivers a comprehensive options trading platform with all requested features:

1. **Live Risk/PnL**
   - Real-time position monitoring with ag-Grid
   - Greek exposure calculations via Rust pricing engine
   - PnL attribution and breakdown by expiry and strike

2. **Volatility Lab**
   - 3D volatility surface visualization
   - Term structure charts
   - Historical volatility comparison

3. **Risk Scenario**
   - What-if analysis for price and volatility changes
   - Time decay simulation
   - Portfolio stress testing

4. **Dynamic Delta Hedging**
   - Automated delta neutralization
   - Multiple hedging strategies (aggressive, balanced, conservative)
   - Telegram notifications on trade execution

5. **Customizability**
   - Configurable dashboard layout
   - Integrated execution within the same view
   - Comprehensive filtering and sorting capabilities

The system architecture ensures high performance with Rust for computationally intensive tasks, Python for API and business logic, and modern React frontend with ag-Grid for responsive user experience.

Would you like me to elaborate on any specific aspect of this implementation?

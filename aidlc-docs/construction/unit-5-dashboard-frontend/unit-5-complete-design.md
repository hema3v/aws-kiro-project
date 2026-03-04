# Unit 5: Dashboard Frontend
## Complete Design Document

**Version**: 1.0  
**Date**: 2026-03-04  
**Status**: Complete - FINAL UNIT!

---

## 1. FUNCTIONAL DESIGN

### 1.1 Application Overview

The Dashboard Frontend is a React-based web application that provides an interactive, real-time interface for visualizing business insights, monitoring data quality, and managing the analytics platform.

### 1.2 Core Capabilities

**User Authentication**
- Login/logout functionality
- OAuth 2.0 / JWT integration
- Session management
- Password reset

**Dashboard Views**
- Executive dashboard (high-level KPIs)
- Revenue dashboard (detailed revenue analytics)
- Customer dashboard (customer insights)
- Operations dashboard (data pipeline monitoring)
- Custom dashboards (user-configurable)

**Data Visualization**
- Line charts (trends over time)
- Bar charts (comparisons)
- Pie charts (distributions)
- Area charts (cumulative metrics)
- Heatmaps (patterns)
- Tables (detailed data)

**Real-Time Updates**
- WebSocket connection to Analytics API
- Live KPI updates (every 5 seconds)
- Alert notifications
- Data refresh indicators

**Interactive Features**
- Date range selection
- Drill-down capabilities
- Filtering and search
- Export functionality (CSV, Excel, PDF)
- Dashboard customization

**User Management**
- User profile management
- Role-based access control
- Business selection (multi-tenant)
- Preferences and settings

**Responsive Design**
- Desktop optimized (1920x1080+)
- Tablet support (768x1024+)
- Mobile support (375x667+)
- Adaptive layouts

### 1.3 Application Structure

```
dashboard-frontend/
├── src/
│   ├── components/
│   │   ├── auth/
│   │   │   ├── LoginForm.tsx
│   │   │   └── ProtectedRoute.tsx
│   │   ├── layout/
│   │   │   ├── Header.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   └── Footer.tsx
│   │   ├── charts/
│   │   │   ├── LineChart.tsx
│   │   │   ├── BarChart.tsx
│   │   │   ├── PieChart.tsx
│   │   │   └── AreaChart.tsx
│   │   ├── dashboard/
│   │   │   ├── KPICard.tsx
│   │   │   ├── RevenueChart.tsx
│   │   │   ├── CustomerChart.tsx
│   │   │   └── TrendChart.tsx
│   │   └── common/
│   │       ├── DateRangePicker.tsx
│   │       ├── FilterPanel.tsx
│   │       └── ExportButton.tsx
│   ├── pages/
│   │   ├── LoginPage.tsx
│   │   ├── DashboardPage.tsx
│   │   ├── RevenuePage.tsx
│   │   ├── CustomerPage.tsx
│   │   ├── OperationsPage.tsx
│   │   └── SettingsPage.tsx
│   ├── services/
│   │   ├── api.service.ts
│   │   ├── auth.service.ts
│   │   ├── websocket.service.ts
│   │   └── export.service.ts
│   ├── store/
│   │   ├── authSlice.ts
│   │   ├── dashboardSlice.ts
│   │   └── store.ts
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   ├── useWebSocket.ts
│   │   └── useAnalytics.ts
│   ├── utils/
│   │   ├── formatters.ts
│   │   └── validators.ts
│   ├── types/
│   │   └── index.ts
│   ├── App.tsx
│   └── index.tsx
├── public/
├── package.json
└── tsconfig.json
```

### 1.4 User Flows

**Login Flow**
```
1. User enters credentials
2. Submit to /api/v1/auth/login
3. Receive JWT token
4. Store token in localStorage
5. Redirect to dashboard
```

**Dashboard View Flow**
```
1. Load dashboard page
2. Fetch KPIs from API
3. Establish WebSocket connection
4. Display initial data
5. Receive real-time updates
6. Update charts automatically
```

**Export Flow**
```
1. User clicks export button
2. Select format (CSV/Excel/PDF)
3. Call export API
4. Download file
```

---

## 2. NFR REQUIREMENTS

### 2.1 Performance
- **NFR-P-001**: Initial page load < 3 seconds
- **NFR-P-002**: Chart rendering < 500ms
- **NFR-P-003**: Real-time update latency < 1 second
- **NFR-P-004**: Smooth animations (60 FPS)

### 2.2 Usability
- **NFR-U-001**: Intuitive navigation
- **NFR-U-002**: Consistent UI/UX
- **NFR-U-003**: Accessible (WCAG 2.1 Level AA)
- **NFR-U-004**: Responsive on all devices

### 2.3 Reliability
- **NFR-R-001**: Handle API failures gracefully
- **NFR-R-002**: Automatic WebSocket reconnection
- **NFR-R-003**: Offline mode with cached data

### 2.4 Security
- **NFR-SEC-001**: Secure token storage
- **NFR-SEC-002**: XSS protection
- **NFR-SEC-003**: CSRF protection
- **NFR-SEC-004**: Secure WebSocket connections

---

## 3. NFR DESIGN

### 3.1 Performance Optimization

**Code Splitting**
```typescript
// Lazy load pages
const DashboardPage = lazy(() => import('./pages/DashboardPage'));
const RevenuePage = lazy(() => import('./pages/RevenuePage'));
const CustomerPage = lazy(() => import('./pages/CustomerPage'));

// Routes with Suspense
<Suspense fallback={<LoadingSpinner />}>
  <Routes>
    <Route path="/dashboard" element={<DashboardPage />} />
    <Route path="/revenue" element={<RevenuePage />} />
    <Route path="/customers" element={<CustomerPage />} />
  </Routes>
</Suspense>
```

**Memoization**
```typescript
// Memoize expensive calculations
const chartData = useMemo(() => {
  return processDataForChart(rawData);
}, [rawData]);

// Memoize components
const KPICard = memo(({ title, value, change }: KPICardProps) => {
  return (
    <Card>
      <h3>{title}</h3>
      <p>{value}</p>
      <span>{change}%</span>
    </Card>
  );
});
```

**Virtual Scrolling**
```typescript
// For large data tables
import { FixedSizeList } from 'react-window';

const DataTable = ({ data }: { data: any[] }) => {
  return (
    <FixedSizeList
      height={600}
      itemCount={data.length}
      itemSize={50}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          {data[index].name}
        </div>
      )}
    </FixedSizeList>
  );
};
```

### 3.2 State Management

**Redux Store**
```typescript
// store/store.ts
import { configureStore } from '@reduxjs/toolkit';
import authReducer from './authSlice';
import dashboardReducer from './dashboardSlice';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    dashboard: dashboardReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

**Dashboard Slice**
```typescript
// store/dashboardSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

export const fetchKPIs = createAsyncThunk(
  'dashboard/fetchKPIs',
  async ({ businessId, dateRange }: FetchKPIsParams) => {
    const response = await apiService.getKPIs(businessId, dateRange);
    return response.data;
  }
);

const dashboardSlice = createSlice({
  name: 'dashboard',
  initialState: {
    kpis: null,
    loading: false,
    error: null,
  },
  reducers: {
    updateKPIs: (state, action) => {
      state.kpis = action.payload;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchKPIs.pending, (state) => {
        state.loading = true;
      })
      .addCase(fetchKPIs.fulfilled, (state, action) => {
        state.loading = false;
        state.kpis = action.payload;
      })
      .addCase(fetchKPIs.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message;
      });
  },
});

export default dashboardSlice.reducer;
```

### 3.3 WebSocket Integration

**WebSocket Service**
```typescript
// services/websocket.service.ts
import { Client } from '@stomp/stompjs';
import SockJS from 'sockjs-client';

class WebSocketService {
  private client: Client | null = null;
  
  connect(token: string): Promise<void> {
    return new Promise((resolve, reject) => {
      this.client = new Client({
        webSocketFactory: () => new SockJS('http://api/ws/dashboard'),
        connectHeaders: {
          Authorization: `Bearer ${token}`,
        },
        onConnect: () => {
          console.log('WebSocket connected');
          resolve();
        },
        onStompError: (frame) => {
          console.error('WebSocket error:', frame);
          reject(frame);
        },
      });
      
      this.client.activate();
    });
  }
  
  subscribe(topic: string, callback: (message: any) => void): void {
    if (!this.client) return;
    
    this.client.subscribe(topic, (message) => {
      const data = JSON.parse(message.body);
      callback(data);
    });
  }
  
  disconnect(): void {
    if (this.client) {
      this.client.deactivate();
    }
  }
}

export default new WebSocketService();
```

**WebSocket Hook**
```typescript
// hooks/useWebSocket.ts
import { useEffect } from 'react';
import { useDispatch } from 'react-redux';
import websocketService from '../services/websocket.service';
import { updateKPIs } from '../store/dashboardSlice';

export const useWebSocket = (token: string) => {
  const dispatch = useDispatch();
  
  useEffect(() => {
    websocketService.connect(token).then(() => {
      // Subscribe to KPI updates
      websocketService.subscribe('/topic/kpis', (data) => {
        dispatch(updateKPIs(data));
      });
      
      // Subscribe to alerts
      websocketService.subscribe('/topic/alerts', (alert) => {
        // Show notification
        showNotification(alert);
      });
    });
    
    return () => {
      websocketService.disconnect();
    };
  }, [token, dispatch]);
};
```

---

## 4. INFRASTRUCTURE DESIGN

### 4.1 Hosting

**Production**
- **CDN**: CloudFront, Cloudflare, or Akamai
- **Static Hosting**: S3, Azure Blob, or Netlify
- **SSL**: Let's Encrypt or AWS Certificate Manager

**Development**
- **Local**: npm run dev (Vite dev server)
- **Staging**: Netlify or Vercel preview deployments

### 4.2 Build & Deployment

**Build Process**
```bash
npm run build
# Output: dist/ directory with optimized assets
```

**CI/CD Pipeline**
```yaml
# .github/workflows/deploy.yml
name: Deploy Dashboard
on:
  push:
    branches: [main]
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run build
      - run: npm run test
      - uses: aws-actions/configure-aws-credentials@v1
      - run: aws s3 sync dist/ s3://dashboard-bucket/
      - run: aws cloudfront create-invalidation
```

### 4.3 Infrastructure Cost

| Component | Quantity | Unit Cost | Total |
|-----------|----------|-----------|-------|
| S3 Hosting | 1 bucket | $5/month | $5 |
| CloudFront CDN | 1 distribution | $50/month | $50 |
| SSL Certificate | 1 | Free | $0 |
| **Total** | | | **$55/month** |

---

## 5. CODE GENERATION

### 5.1 Key Implementation - Dashboard Page

```typescript
// pages/DashboardPage.tsx
import React, { useEffect } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { Grid, Card, CardContent, Typography } from '@mui/material';
import { fetchKPIs } from '../store/dashboardSlice';
import { useWebSocket } from '../hooks/useWebSocket';
import KPICard from '../components/dashboard/KPICard';
import RevenueChart from '../components/dashboard/RevenueChart';
import CustomerChart from '../components/dashboard/CustomerChart';

const DashboardPage: React.FC = () => {
  const dispatch = useDispatch();
  const { kpis, loading } = useSelector((state: RootState) => state.dashboard);
  const { token } = useSelector((state: RootState) => state.auth);
  
  // Establish WebSocket connection
  useWebSocket(token);
  
  useEffect(() => {
    // Fetch initial data
    dispatch(fetchKPIs({
      businessId: 'current-business-id',
      dateRange: { start: '2026-03-01', end: '2026-03-04' },
    }));
  }, [dispatch]);
  
  if (loading) return <LoadingSpinner />;
  
  return (
    <div className="dashboard-page">
      <Typography variant="h4" gutterBottom>
        Executive Dashboard
      </Typography>
      
      <Grid container spacing={3}>
        {/* KPI Cards */}
        <Grid item xs={12} sm={6} md={3}>
          <KPICard
            title="Total Revenue"
            value={kpis?.totalRevenue}
            change={kpis?.revenueGrowth}
            icon={<MoneyIcon />}
          />
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <KPICard
            title="Total Customers"
            value={kpis?.totalCustomers}
            change={kpis?.customerGrowth}
            icon={<PeopleIcon />}
          />
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <KPICard
            title="Data Quality"
            value={`${kpis?.dataQualityScore}%`}
            change={kpis?.qualityChange}
            icon={<CheckIcon />}
          />
        </Grid>
        <Grid item xs={12} sm={6} md={3}>
          <KPICard
            title="Records Processed"
            value={kpis?.recordsProcessed}
            change={kpis?.processingChange}
            icon={<DataIcon />}
          />
        </Grid>
        
        {/* Charts */}
        <Grid item xs={12} md={8}>
          <Card>
            <CardContent>
              <Typography variant="h6">Revenue Trend</Typography>
              <RevenueChart />
            </CardContent>
          </Card>
        </Grid>
        
        <Grid item xs={12} md={4}>
          <Card>
            <CardContent>
              <Typography variant="h6">Customer Distribution</Typography>
              <CustomerChart />
            </CardContent>
          </Card>
        </Grid>
      </Grid>
    </div>
  );
};

export default DashboardPage;
```

### 5.2 Key Implementation - Chart Component

```typescript
// components/charts/LineChart.tsx
import React from 'react';
import ReactECharts from 'echarts-for-react';

interface LineChartProps {
  data: { date: string; value: number }[];
  title?: string;
}

const LineChart: React.FC<LineChartProps> = ({ data, title }) => {
  const option = {
    title: {
      text: title,
    },
    tooltip: {
      trigger: 'axis',
    },
    xAxis: {
      type: 'category',
      data: data.map(d => d.date),
    },
    yAxis: {
      type: 'value',
    },
    series: [
      {
        data: data.map(d => d.value),
        type: 'line',
        smooth: true,
        areaStyle: {
          opacity: 0.3,
        },
      },
    ],
  };
  
  return <ReactECharts option={option} style={{ height: '400px' }} />;
};

export default LineChart;
```

### 5.3 Key Implementation - API Service

```typescript
// services/api.service.ts
import axios from 'axios';

const API_BASE_URL = process.env.REACT_APP_API_URL || 'http://localhost:8080';

class ApiService {
  private axiosInstance;
  
  constructor() {
    this.axiosInstance = axios.create({
      baseURL: API_BASE_URL,
      timeout: 30000,
    });
    
    // Add auth token to requests
    this.axiosInstance.interceptors.request.use((config) => {
      const token = localStorage.getItem('token');
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;
      }
      return config;
    });
    
    // Handle errors
    this.axiosInstance.interceptors.response.use(
      (response) => response,
      (error) => {
        if (error.response?.status === 401) {
          // Redirect to login
          window.location.href = '/login';
        }
        return Promise.reject(error);
      }
    );
  }
  
  async getKPIs(businessId: string, dateRange: DateRange) {
    const response = await this.axiosInstance.get('/api/v1/analytics/kpis', {
      params: {
        businessId,
        startDate: dateRange.start,
        endDate: dateRange.end,
      },
    });
    return response.data;
  }
  
  async getRevenueAnalytics(businessId: string, dateRange: DateRange, groupBy: string) {
    const response = await this.axiosInstance.get('/api/v1/analytics/revenue', {
      params: {
        businessId,
        startDate: dateRange.start,
        endDate: dateRange.end,
        groupBy,
      },
    });
    return response.data;
  }
  
  async exportReport(reportId: string, format: string) {
    const response = await this.axiosInstance.get(
      `/api/v1/analytics/reports/${reportId}/export`,
      {
        params: { format },
        responseType: 'blob',
      }
    );
    return response.data;
  }
}

export default new ApiService();
```

### 5.4 Configuration Files

**package.json**
```json
{
  "name": "dashboard-frontend",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "@reduxjs/toolkit": "^2.0.0",
    "react-redux": "^9.0.0",
    "@mui/material": "^5.14.0",
    "@mui/icons-material": "^5.14.0",
    "echarts": "^5.4.3",
    "echarts-for-react": "^3.0.2",
    "axios": "^1.6.0",
    "@stomp/stompjs": "^7.0.0",
    "sockjs-client": "^1.6.1",
    "react-window": "^1.8.10",
    "date-fns": "^2.30.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "typescript": "^5.3.0",
    "vite": "^5.0.0",
    "@vitejs/plugin-react": "^4.2.0",
    "eslint": "^8.55.0",
    "@typescript-eslint/eslint-plugin": "^6.15.0",
    "@typescript-eslint/parser": "^6.15.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest",
    "lint": "eslint src --ext ts,tsx"
  }
}
```

**vite.config.ts**
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
      '/ws': {
        target: 'ws://localhost:8080',
        ws: true,
      },
    },
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
          charts: ['echarts', 'echarts-for-react'],
          ui: ['@mui/material', '@mui/icons-material'],
        },
      },
    },
  },
});
```

---

## 6. DEPLOYMENT

### 6.1 Docker Configuration

```dockerfile
# Build stage
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**nginx.conf**
```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;
    
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    location /api {
        proxy_pass http://analytics-api:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    
    location /ws {
        proxy_pass http://analytics-api:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## 7. SUCCESS CRITERIA

- [ ] Initial page load < 3 seconds
- [ ] Chart rendering < 500ms
- [ ] Real-time updates functional
- [ ] Responsive on all devices
- [ ] All visualizations working
- [ ] Export functionality working
- [ ] User authentication working
- [ ] WebSocket connection stable
- [ ] WCAG 2.1 Level AA compliant
- [ ] 80%+ test coverage

---

## Document Status

**Status**: Complete - ALL UNITS COMPLETE! 🎉  
**All Stages**: Functional Design, NFR Requirements, NFR Design, Infrastructure Design, Code Generation  
**Next**: Build and Test Phase

---

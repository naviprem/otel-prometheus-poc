# Phase 4: Control Plane - React Dashboard

## Overview
Build a React-based web dashboard to visualize metrics from Prometheus using Apache ECharts for charting.

---

## Step 1: Initialize React Project

### 1.1 Create Project Directory
```bash
cd controlplane
npx create-react-app dashboard --template typescript
cd dashboard
```

Or using Vite (faster, recommended):
```bash
cd controlplane
npm create vite@latest dashboard -- --template react-ts
cd dashboard
npm install
```

### 1.2 Install Dependencies
```bash
# Core dependencies
npm install axios
npm install echarts echarts-for-react
npm install react-router-dom

# UI Library (choose one)
npm install @mui/material @mui/icons-material @emotion/react @emotion/styled

# Date handling
npm install date-fns

# Development dependencies
npm install -D @types/react-router-dom
```

---

## Step 2: Project Structure Setup

### 2.1 Create Directory Structure
```bash
mkdir -p src/components
mkdir -p src/services
mkdir -p src/types
mkdir -p src/utils
mkdir -p src/pages
mkdir -p src/hooks
```

Final structure:
```
dashboard/
├── public/
├── src/
│   ├── components/
│   │   ├── MetricChart.tsx
│   │   ├── DataPlaneSelector.tsx
│   │   ├── TimeRangeSelector.tsx
│   │   ├── MetricCard.tsx
│   │   └── Layout.tsx
│   ├── services/
│   │   └── prometheus.ts
│   ├── types/
│   │   └── metrics.ts
│   ├── utils/
│   │   └── formatters.ts
│   ├── hooks/
│   │   └── useMetrics.ts
│   ├── pages/
│   │   ├── LandingPage.tsx
│   │   └── MetricsDashboard.tsx
│   ├── App.tsx
│   ├── App.css
│   └── index.tsx
├── package.json
└── Dockerfile
```

---

## Step 3: Create Type Definitions

### 3.1 Create `src/types/metrics.ts`
```typescript
// Prometheus API response types
export interface PrometheusResponse {
  status: string;
  data: PrometheusData;
}

export interface PrometheusData {
  resultType: string;
  result: PrometheusResult[];
}

export interface PrometheusResult {
  metric: Record<string, string>;
  values?: [number, string][]; // Range query
  value?: [number, string];     // Instant query
}

// Dashboard types
export interface TimeRange {
  label: string;
  value: string;
  seconds: number;
}

export interface DataPlane {
  id: string;
  name: string;
  environment: string;
  status: 'up' | 'down' | 'unknown';
}

export interface MetricData {
  timestamp: number;
  value: number;
}

export interface ChartData {
  timestamps: string[];
  series: {
    name: string;
    data: number[];
  }[];
}

export interface MetricCardData {
  title: string;
  value: string | number;
  unit?: string;
  trend?: 'up' | 'down' | 'neutral';
  change?: number;
}
```

---

## Step 4: Create Prometheus Service

### 4.1 Create `src/services/prometheus.ts`
```typescript
import axios, { AxiosInstance } from 'axios';
import { PrometheusResponse } from '../types/metrics';

class PrometheusService {
  private client: AxiosInstance;
  private baseURL: string;

  constructor(baseURL: string = 'http://localhost:9090') {
    this.baseURL = baseURL;
    this.client = axios.create({
      baseURL: `${baseURL}/api/v1`,
      timeout: 30000,
    });
  }

  /**
   * Execute a range query (returns time series)
   */
  async queryRange(
    query: string,
    start: Date,
    end: Date,
    step: string = '15s'
  ): Promise<PrometheusResponse> {
    try {
      const response = await this.client.get('/query_range', {
        params: {
          query,
          start: Math.floor(start.getTime() / 1000),
          end: Math.floor(end.getTime() / 1000),
          step,
        },
      });
      return response.data;
    } catch (error) {
      console.error('Prometheus query_range error:', error);
      throw error;
    }
  }

  /**
   * Execute an instant query (returns current value)
   */
  async queryInstant(query: string): Promise<PrometheusResponse> {
    try {
      const response = await this.client.get('/query', {
        params: { query },
      });
      return response.data;
    } catch (error) {
      console.error('Prometheus query error:', error);
      throw error;
    }
  }

  /**
   * Get list of labels
   */
  async getLabels(): Promise<string[]> {
    try {
      const response = await this.client.get('/labels');
      return response.data.data;
    } catch (error) {
      console.error('Prometheus labels error:', error);
      throw error;
    }
  }

  /**
   * Get label values for a specific label
   */
  async getLabelValues(label: string): Promise<string[]> {
    try {
      const response = await this.client.get(`/label/${label}/values`);
      return response.data.data;
    } catch (error) {
      console.error('Prometheus label values error:', error);
      throw error;
    }
  }

  /**
   * Check Prometheus health
   */
  async checkHealth(): Promise<boolean> {
    try {
      const response = await axios.get(`${this.baseURL}/-/healthy`, {
        timeout: 5000,
      });
      return response.status === 200;
    } catch (error) {
      return false;
    }
  }
}

export default new PrometheusService();
```

---

## Step 5: Create Utility Functions

### 5.1 Create `src/utils/formatters.ts`
```typescript
/**
 * Format numbers for display
 */
export const formatNumber = (value: number, decimals: number = 2): string => {
  if (value >= 1e9) {
    return (value / 1e9).toFixed(decimals) + 'B';
  }
  if (value >= 1e6) {
    return (value / 1e6).toFixed(decimals) + 'M';
  }
  if (value >= 1e3) {
    return (value / 1e3).toFixed(decimals) + 'K';
  }
  return value.toFixed(decimals);
};

/**
 * Format duration in seconds to human readable
 */
export const formatDuration = (seconds: number): string => {
  if (seconds < 0.001) {
    return `${(seconds * 1000000).toFixed(0)}µs`;
  }
  if (seconds < 1) {
    return `${(seconds * 1000).toFixed(0)}ms`;
  }
  if (seconds < 60) {
    return `${seconds.toFixed(2)}s`;
  }
  const minutes = Math.floor(seconds / 60);
  const secs = seconds % 60;
  return `${minutes}m ${secs.toFixed(0)}s`;
};

/**
 * Format percentage
 */
export const formatPercentage = (value: number, decimals: number = 2): string => {
  return `${value.toFixed(decimals)}%`;
};

/**
 * Format timestamp for chart axis
 */
export const formatTimestamp = (timestamp: number): string => {
  const date = new Date(timestamp * 1000);
  return date.toLocaleTimeString('en-US', {
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
  });
};

/**
 * Calculate time range in seconds
 */
export const getTimeRangeSeconds = (range: string): number => {
  const map: Record<string, number> = {
    '5m': 5 * 60,
    '15m': 15 * 60,
    '1h': 60 * 60,
    '6h': 6 * 60 * 60,
    '24h': 24 * 60 * 60,
  };
  return map[range] || 5 * 60;
};
```

---

## Step 6: Create Custom Hook for Metrics

### 6.1 Create `src/hooks/useMetrics.ts`
```typescript
import { useState, useEffect, useCallback } from 'react';
import prometheusService from '../services/prometheus';
import { PrometheusResult, MetricData } from '../types/metrics';

interface UseMetricsOptions {
  query: string;
  timeRange: string;
  refreshInterval?: number;
  enabled?: boolean;
}

export const useMetrics = ({
  query,
  timeRange,
  refreshInterval = 30000,
  enabled = true,
}: UseMetricsOptions) => {
  const [data, setData] = useState<PrometheusResult[]>([]);
  const [loading, setLoading] = useState<boolean>(true);
  const [error, setError] = useState<Error | null>(null);

  const fetchData = useCallback(async () => {
    if (!enabled || !query) {
      return;
    }

    try {
      setLoading(true);
      setError(null);

      const end = new Date();
      const start = new Date(end.getTime() - getTimeRangeMs(timeRange));

      const response = await prometheusService.queryRange(
        query,
        start,
        end,
        calculateStep(timeRange)
      );

      if (response.status === 'success') {
        setData(response.data.result);
      }
    } catch (err) {
      setError(err as Error);
      console.error('Error fetching metrics:', err);
    } finally {
      setLoading(false);
    }
  }, [query, timeRange, enabled]);

  useEffect(() => {
    fetchData();

    if (refreshInterval > 0) {
      const interval = setInterval(fetchData, refreshInterval);
      return () => clearInterval(interval);
    }
  }, [fetchData, refreshInterval]);

  return { data, loading, error, refetch: fetchData };
};

const getTimeRangeMs = (range: string): number => {
  const map: Record<string, number> = {
    '5m': 5 * 60 * 1000,
    '15m': 15 * 60 * 1000,
    '1h': 60 * 60 * 1000,
    '6h': 6 * 60 * 60 * 1000,
    '24h': 24 * 60 * 60 * 1000,
  };
  return map[range] || 5 * 60 * 1000;
};

const calculateStep = (timeRange: string): string => {
  const map: Record<string, string> = {
    '5m': '5s',
    '15m': '15s',
    '1h': '30s',
    '6h': '2m',
    '24h': '5m',
  };
  return map[timeRange] || '15s';
};
```

---

## Step 7: Create UI Components

### 7.1 Create `src/components/TimeRangeSelector.tsx`
```typescript
import React from 'react';
import { TimeRange } from '../types/metrics';

interface TimeRangeSelectorProps {
  selectedRange: string;
  onRangeChange: (range: string) => void;
}

const timeRanges: TimeRange[] = [
  { label: 'Last 5 minutes', value: '5m', seconds: 300 },
  { label: 'Last 15 minutes', value: '15m', seconds: 900 },
  { label: 'Last 1 hour', value: '1h', seconds: 3600 },
  { label: 'Last 6 hours', value: '6h', seconds: 21600 },
  { label: 'Last 24 hours', value: '24h', seconds: 86400 },
];

const TimeRangeSelector: React.FC<TimeRangeSelectorProps> = ({
  selectedRange,
  onRangeChange,
}) => {
  return (
    <div className="time-range-selector">
      <label>Time Range: </label>
      <select
        value={selectedRange}
        onChange={(e) => onRangeChange(e.target.value)}
        className="time-range-select"
      >
        {timeRanges.map((range) => (
          <option key={range.value} value={range.value}>
            {range.label}
          </option>
        ))}
      </select>
    </div>
  );
};

export default TimeRangeSelector;
```

### 7.2 Create `src/components/DataPlaneSelector.tsx`
```typescript
import React from 'react';

interface DataPlaneSelectorProps {
  environments: string[];
  selectedEnvironment: string;
  onEnvironmentChange: (env: string) => void;
}

const DataPlaneSelector: React.FC<DataPlaneSelectorProps> = ({
  environments,
  selectedEnvironment,
  onEnvironmentChange,
}) => {
  return (
    <div className="dataplane-selector">
      <label>Environment: </label>
      <select
        value={selectedEnvironment}
        onChange={(e) => onEnvironmentChange(e.target.value)}
        className="dataplane-select"
      >
        <option value="all">All Environments</option>
        {environments.map((env) => (
          <option key={env} value={env}>
            {env.toUpperCase()}
          </option>
        ))}
      </select>
    </div>
  );
};

export default DataPlaneSelector;
```

### 7.3 Create `src/components/MetricCard.tsx`
```typescript
import React from 'react';
import { MetricCardData } from '../types/metrics';

interface MetricCardProps {
  data: MetricCardData;
}

const MetricCard: React.FC<MetricCardProps> = ({ data }) => {
  const getTrendColor = () => {
    if (data.trend === 'up') return '#4caf50';
    if (data.trend === 'down') return '#f44336';
    return '#757575';
  };

  return (
    <div className="metric-card">
      <h3 className="metric-card-title">{data.title}</h3>
      <div className="metric-card-value">
        {data.value}
        {data.unit && <span className="metric-unit">{data.unit}</span>}
      </div>
      {data.change !== undefined && (
        <div className="metric-card-change" style={{ color: getTrendColor() }}>
          {data.trend === 'up' ? '↑' : '↓'} {Math.abs(data.change)}%
        </div>
      )}
    </div>
  );
};

export default MetricCard;
```

### 7.4 Create `src/components/MetricChart.tsx`
```typescript
import React, { useMemo } from 'react';
import ReactECharts from 'echarts-for-react';
import { PrometheusResult } from '../types/metrics';
import { formatTimestamp, formatDuration } from '../utils/formatters';

interface MetricChartProps {
  title: string;
  data: PrometheusResult[];
  unit?: string;
  type?: 'line' | 'area';
  formatValue?: (value: number) => string;
}

const MetricChart: React.FC<MetricChartProps> = ({
  title,
  data,
  unit = '',
  type = 'line',
  formatValue,
}) => {
  const option = useMemo(() => {
    // Transform Prometheus data to ECharts format
    const series = data.map((result) => {
      const label = result.metric.environment || result.metric.http_route || 'default';
      return {
        name: label,
        type: type,
        smooth: true,
        data: result.values?.map(([timestamp, value]) => [
          timestamp * 1000, // Convert to milliseconds
          parseFloat(value),
        ]) || [],
        areaStyle: type === 'area' ? {} : undefined,
      };
    });

    return {
      title: {
        text: title,
        left: 'center',
      },
      tooltip: {
        trigger: 'axis',
        formatter: (params: any) => {
          const time = new Date(params[0].value[0]).toLocaleString();
          let content = `${time}<br/>`;
          params.forEach((param: any) => {
            const value = formatValue
              ? formatValue(param.value[1])
              : `${param.value[1].toFixed(2)} ${unit}`;
            content += `${param.marker} ${param.seriesName}: ${value}<br/>`;
          });
          return content;
        },
      },
      legend: {
        data: series.map((s) => s.name),
        bottom: 0,
      },
      grid: {
        left: '3%',
        right: '4%',
        bottom: '15%',
        containLabel: true,
      },
      xAxis: {
        type: 'time',
        boundaryGap: false,
      },
      yAxis: {
        type: 'value',
        axisLabel: {
          formatter: (value: number) =>
            formatValue ? formatValue(value) : `${value} ${unit}`,
        },
      },
      series: series,
    };
  }, [data, title, unit, type, formatValue]);

  return (
    <div className="metric-chart">
      <ReactECharts option={option} style={{ height: '400px', width: '100%' }} />
    </div>
  );
};

export default MetricChart;
```

---

## Step 8: Create Dashboard Pages

### 8.1 Create `src/pages/LandingPage.tsx`
```typescript
import React, { useEffect, useState } from 'react';
import { Link } from 'react-router-dom';
import prometheusService from '../services/prometheus';

const LandingPage: React.FC = () => {
  const [dataPlanes, setDataPlanes] = useState<string[]>([]);
  const [prometheusHealthy, setPrometheusHealthy] = useState<boolean>(false);

  useEffect(() => {
    const fetchDataPlanes = async () => {
      try {
        const environments = await prometheusService.getLabelValues('environment');
        setDataPlanes(environments);
        setPrometheusHealthy(true);
      } catch (error) {
        console.error('Error fetching data planes:', error);
        setPrometheusHealthy(false);
      }
    };

    fetchDataPlanes();
  }, []);

  return (
    <div className="landing-page">
      <header className="landing-header">
        <h1>OTEL Observability Platform</h1>
        <p>Multi-Environment Monitoring & Metrics Dashboard</p>
      </header>

      <div className="status-section">
        <h2>System Status</h2>
        <div className="status-card">
          <div className="status-indicator">
            <span className={`status-dot ${prometheusHealthy ? 'up' : 'down'}`} />
            <span>Prometheus: {prometheusHealthy ? 'Healthy' : 'Unavailable'}</span>
          </div>
        </div>
      </div>

      <div className="dataplane-section">
        <h2>Data Planes</h2>
        <div className="dataplane-grid">
          {dataPlanes.length > 0 ? (
            dataPlanes.map((env) => (
              <div key={env} className="dataplane-card">
                <h3>{env.toUpperCase()}</h3>
                <p>Environment</p>
                <Link to={`/dashboard?env=${env}`} className="view-metrics-btn">
                  View Metrics
                </Link>
              </div>
            ))
          ) : (
            <p>No data planes detected</p>
          )}
        </div>
      </div>

      <div className="quick-actions">
        <Link to="/dashboard" className="btn-primary">
          View All Metrics
        </Link>
      </div>
    </div>
  );
};

export default LandingPage;
```

### 8.2 Create `src/pages/MetricsDashboard.tsx`
```typescript
import React, { useState, useEffect } from 'react';
import { useSearchParams } from 'react-router-dom';
import MetricChart from '../components/MetricChart';
import MetricCard from '../components/MetricCard';
import TimeRangeSelector from '../components/TimeRangeSelector';
import DataPlaneSelector from '../components/DataPlaneSelector';
import { useMetrics } from '../hooks/useMetrics';
import { formatDuration, formatPercentage } from '../utils/formatters';
import prometheusService from '../services/prometheus';

const MetricsDashboard: React.FC = () => {
  const [searchParams] = useSearchParams();
  const [selectedEnvironment, setSelectedEnvironment] = useState<string>(
    searchParams.get('env') || 'all'
  );
  const [timeRange, setTimeRange] = useState<string>('15m');
  const [environments, setEnvironments] = useState<string[]>([]);
  const [autoRefresh, setAutoRefresh] = useState<boolean>(true);

  // Build environment filter for queries
  const envFilter =
    selectedEnvironment === 'all' ? '' : `{environment="${selectedEnvironment}"}`;

  // Fetch available environments
  useEffect(() => {
    const fetchEnvironments = async () => {
      try {
        const envs = await prometheusService.getLabelValues('environment');
        setEnvironments(envs);
      } catch (error) {
        console.error('Error fetching environments:', error);
      }
    };
    fetchEnvironments();
  }, []);

  // Metric queries
  const requestRateQuery = `sum by (environment) (rate(otel_http_server_duration_count${envFilter}[5m]))`;
  const latencyP95Query = `histogram_quantile(0.95, sum by (environment, le) (rate(otel_http_server_duration_bucket${envFilter}[5m])))`;
  const latencyP50Query = `histogram_quantile(0.50, sum by (environment, le) (rate(otel_http_server_duration_bucket${envFilter}[5m])))`;
  const errorRateQuery = `(sum by (environment) (rate(otel_http_server_duration_count{http_status_code=~"5.."${envFilter ? ',' + envFilter.slice(1) : ''}[5m])) / sum by (environment) (rate(otel_http_server_duration_count${envFilter}[5m]))) * 100`;

  // Use custom hook to fetch metrics
  const requestRate = useMetrics({
    query: requestRateQuery,
    timeRange,
    refreshInterval: autoRefresh ? 30000 : 0,
  });

  const latencyP95 = useMetrics({
    query: latencyP95Query,
    timeRange,
    refreshInterval: autoRefresh ? 30000 : 0,
  });

  const latencyP50 = useMetrics({
    query: latencyP50Query,
    timeRange,
    refreshInterval: autoRefresh ? 30000 : 0,
  });

  const errorRate = useMetrics({
    query: errorRateQuery,
    timeRange,
    refreshInterval: autoRefresh ? 30000 : 0,
  });

  // Calculate summary metrics from latest data point
  const getLatestValue = (data: any[]) => {
    if (!data || data.length === 0) return 0;
    const values = data[0].values;
    if (!values || values.length === 0) return 0;
    return parseFloat(values[values.length - 1][1]);
  };

  return (
    <div className="metrics-dashboard">
      <header className="dashboard-header">
        <h1>Metrics Dashboard</h1>
        <div className="dashboard-controls">
          <DataPlaneSelector
            environments={environments}
            selectedEnvironment={selectedEnvironment}
            onEnvironmentChange={setSelectedEnvironment}
          />
          <TimeRangeSelector
            selectedRange={timeRange}
            onRangeChange={setTimeRange}
          />
          <button
            className={`refresh-toggle ${autoRefresh ? 'active' : ''}`}
            onClick={() => setAutoRefresh(!autoRefresh)}
          >
            Auto-refresh: {autoRefresh ? 'ON' : 'OFF'}
          </button>
        </div>
      </header>

      {/* Summary Cards */}
      <div className="metric-cards-grid">
        <MetricCard
          data={{
            title: 'Request Rate',
            value: getLatestValue(requestRate.data).toFixed(2),
            unit: 'req/s',
          }}
        />
        <MetricCard
          data={{
            title: 'P95 Latency',
            value: formatDuration(getLatestValue(latencyP95.data)),
          }}
        />
        <MetricCard
          data={{
            title: 'P50 Latency',
            value: formatDuration(getLatestValue(latencyP50.data)),
          }}
        />
        <MetricCard
          data={{
            title: 'Error Rate',
            value: formatPercentage(getLatestValue(errorRate.data)),
          }}
        />
      </div>

      {/* Charts */}
      <div className="charts-grid">
        <MetricChart
          title="Request Rate (requests/sec)"
          data={requestRate.data}
          unit="req/s"
          type="area"
        />
        <MetricChart
          title="Latency P95 (seconds)"
          data={latencyP95.data}
          formatValue={formatDuration}
          type="line"
        />
        <MetricChart
          title="Latency P50 (seconds)"
          data={latencyP50.data}
          formatValue={formatDuration}
          type="line"
        />
        <MetricChart
          title="Error Rate (%)"
          data={errorRate.data}
          unit="%"
          type="area"
        />
      </div>
    </div>
  );
};

export default MetricsDashboard;
```

---

## Step 9: Create Main App Component

### 9.1 Create `src/App.tsx`
```typescript
import React from 'react';
import { BrowserRouter as Router, Routes, Route, Link } from 'react-router-dom';
import LandingPage from './pages/LandingPage';
import MetricsDashboard from './pages/MetricsDashboard';
import './App.css';

const App: React.FC = () => {
  return (
    <Router>
      <div className="app">
        <nav className="app-nav">
          <Link to="/" className="nav-logo">
            OTEL Platform
          </Link>
          <div className="nav-links">
            <Link to="/">Home</Link>
            <Link to="/dashboard">Dashboard</Link>
          </div>
        </nav>

        <main className="app-main">
          <Routes>
            <Route path="/" element={<LandingPage />} />
            <Route path="/dashboard" element={<MetricsDashboard />} />
          </Routes>
        </main>

        <footer className="app-footer">
          <p>OTEL Prometheus PoC - Multi-Environment Observability</p>
        </footer>
      </div>
    </Router>
  );
};

export default App;
```

### 9.2 Create `src/App.css`
```css
/* Basic styling - enhance as needed */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen',
    'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue',
    sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  background-color: #f5f5f5;
}

.app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* Navigation */
.app-nav {
  background-color: #1976d2;
  color: white;
  padding: 1rem 2rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.nav-logo {
  font-size: 1.5rem;
  font-weight: bold;
  color: white;
  text-decoration: none;
}

.nav-links a {
  color: white;
  text-decoration: none;
  margin-left: 2rem;
  transition: opacity 0.2s;
}

.nav-links a:hover {
  opacity: 0.8;
}

/* Main Content */
.app-main {
  flex: 1;
  padding: 2rem;
  max-width: 1400px;
  margin: 0 auto;
  width: 100%;
}

/* Landing Page */
.landing-header {
  text-align: center;
  margin-bottom: 3rem;
}

.landing-header h1 {
  font-size: 2.5rem;
  color: #333;
  margin-bottom: 0.5rem;
}

.status-section,
.dataplane-section {
  margin: 2rem 0;
}

.status-dot {
  display: inline-block;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  margin-right: 0.5rem;
}

.status-dot.up {
  background-color: #4caf50;
}

.status-dot.down {
  background-color: #f44336;
}

.dataplane-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1.5rem;
  margin-top: 1rem;
}

.dataplane-card {
  background: white;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  text-align: center;
}

.view-metrics-btn {
  display: inline-block;
  margin-top: 1rem;
  padding: 0.5rem 1rem;
  background-color: #1976d2;
  color: white;
  text-decoration: none;
  border-radius: 4px;
  transition: background-color 0.2s;
}

.view-metrics-btn:hover {
  background-color: #1565c0;
}

/* Dashboard */
.dashboard-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
}

.dashboard-controls {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.dataplane-select,
.time-range-select {
  padding: 0.5rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 1rem;
}

.refresh-toggle {
  padding: 0.5rem 1rem;
  border: 1px solid #1976d2;
  background: white;
  color: #1976d2;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
}

.refresh-toggle.active {
  background: #1976d2;
  color: white;
}

/* Metric Cards */
.metric-cards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.5rem;
  margin-bottom: 2rem;
}

.metric-card {
  background: white;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.metric-card-title {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 0.5rem;
}

.metric-card-value {
  font-size: 2rem;
  font-weight: bold;
  color: #333;
}

.metric-unit {
  font-size: 1rem;
  color: #666;
  margin-left: 0.25rem;
}

/* Charts */
.charts-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(500px, 1fr));
  gap: 2rem;
}

.metric-chart {
  background: white;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* Footer */
.app-footer {
  background-color: #333;
  color: white;
  text-align: center;
  padding: 1rem;
  margin-top: 2rem;
}
```

---

## Step 10: Create Dockerfile

### 10.1 Create `Dockerfile`
```dockerfile
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci

# Copy source code
COPY . .

# Build application
RUN npm run build

# Runtime stage
FROM nginx:alpine

# Copy built files
COPY --from=builder /app/dist /usr/share/nginx/html
COPY --from=builder /app/build /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### 10.2 Create `nginx.conf`
```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy Prometheus API to avoid CORS
    location /api/v1/ {
        proxy_pass http://prometheus:9090/api/v1/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
}
```

---

## Step 11: Testing

### 11.1 Run Development Server
```bash
npm start
```

### 11.2 Build for Production
```bash
npm run build
```

### 11.3 Build Docker Image
```bash
docker build -t otel-dashboard:latest .
```

### 11.4 Run Docker Container
```bash
docker run -d \
  -p 3000:80 \
  --name dashboard \
  --network otel-network \
  otel-dashboard:latest
```

---

## Expected Outcomes

✅ **Functional Dashboard**:
- Landing page shows data plane status
- Dashboard displays real-time metrics
- Environment filtering works
- Time range selection works
- Auto-refresh updates charts

✅ **Visual Quality**:
- Clean, modern UI
- Responsive charts with ECharts
- Mobile-friendly layout
- Smooth interactions

✅ **Performance**:
- Fast page loads
- Efficient metric queries
- No memory leaks
- Handles large datasets

---

## Next Steps

Proceed to **Phase 5: Docker Compose Orchestration** to integrate all components into a single deployable stack.

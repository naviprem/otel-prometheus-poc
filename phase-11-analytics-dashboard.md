# Phase 11: Analytics Dashboard

## Overview
Build a new analytics dashboard section in the React application to visualize long-term metrics from ClickHouse. This dashboard complements the existing real-time monitoring by providing historical analysis, trends, and insights.

---

## Architecture

```
React Dashboard
├── Real-time Dashboard (Prometheus) ← Existing
└── Analytics Dashboard (ClickHouse)  ← New
    ├── Long-term Trends
    ├── Capacity Planning
    ├── Cost Attribution
    └── Anomaly Detection
```

---

## Step 1: Setup ClickHouse Client Library

### 1.1 Navigate to Dashboard Directory
```bash
cd controlplane/dashboard
```

### 1.2 Install Dependencies
```bash
npm install @clickhouse/client
npm install recharts  # Alternative charting library for analytics
npm install date-fns
npm install react-query
```

---

## Step 2: Create ClickHouse Service Layer

### 2.1 Create Service Directory
```bash
mkdir -p src/services/clickhouse
```

### 2.2 Create ClickHouse Client

**Create `src/services/clickhouse/client.ts`:**
```typescript
import { createClient, ClickHouseClient } from '@clickhouse/client';

class ClickHouseService {
  private client: ClickHouseClient;
  private baseUrl: string;

  constructor(baseUrl: string = 'http://localhost:8123') {
    this.baseUrl = baseUrl;
    this.client = createClient({
      host: baseUrl,
      username: 'dashboard',
      password: process.env.REACT_APP_CLICKHOUSE_PASSWORD || 'dashboard123',
      database: 'otel_metrics',
      clickhouse_settings: {
        // Enable query cache
        use_query_cache: 1,
        // Set query timeout
        max_execution_time: 30,
      },
    });
  }

  /**
   * Execute a query and return results
   */
  async query<T = any>(query: string): Promise<T[]> {
    try {
      const resultSet = await this.client.query({
        query,
        format: 'JSONEachRow',
      });

      const data = await resultSet.json<T>();
      return data;
    } catch (error) {
      console.error('ClickHouse query error:', error);
      throw error;
    }
  }

  /**
   * Execute a query and return a single row
   */
  async queryOne<T = any>(query: string): Promise<T | null> {
    const results = await this.query<T>(query);
    return results.length > 0 ? results[0] : null;
  }

  /**
   * Get available environments
   */
  async getEnvironments(): Promise<string[]> {
    const query = `
      SELECT DISTINCT environment
      FROM metrics_local
      ORDER BY environment
    `;
    const results = await this.query<{ environment: string }>(query);
    return results.map(r => r.environment);
  }

  /**
   * Get data plane IDs for an environment
   */
  async getDataPlanes(environment?: string): Promise<string[]> {
    const whereClause = environment ? `WHERE environment = '${environment}'` : '';
    const query = `
      SELECT DISTINCT data_plane_id
      FROM metrics_local
      ${whereClause}
      ORDER BY data_plane_id
    `;
    const results = await this.query<{ data_plane_id: string }>(query);
    return results.map(r => r.data_plane_id);
  }

  /**
   * Health check
   */
  async ping(): Promise<boolean> {
    try {
      await this.client.ping();
      return true;
    } catch (error) {
      return false;
    }
  }

  /**
   * Close connection
   */
  async close(): Promise<void> {
    await this.client.close();
  }
}

export default new ClickHouseService();
```

### 2.3 Create Analytics Query Service

**Create `src/services/clickhouse/analytics.ts`:**
```typescript
import clickhouseClient from './client';

export interface DailyTrend {
  date: string;
  environment: string;
  total_requests: number;
  total_errors: number;
  error_rate: number;
  avg_p95_latency: number;
  max_p95_latency: number;
}

export interface RequestRateTrend {
  hour: string;
  environment: string;
  data_plane_id: string;
  requests_per_second: number;
}

export interface LatencyByEndpoint {
  date: string;
  environment: string;
  http_route: string;
  request_count: number;
  p50_latency: number;
  p95_latency: number;
  p99_latency: number;
}

export interface SlowestEndpoint {
  environment: string;
  http_route: string;
  request_count: number;
  p95_latency: number;
  max_latency: number;
}

export interface MonthlyStats {
  month: string;
  environment: string;
  monthly_requests: number;
  avg_latency: number;
  error_rate: number;
}

export interface GrowthRate {
  month: string;
  environment: string;
  requests: number;
  growth: number;
  growth_percent: number;
}

class AnalyticsService {
  /**
   * Get daily trends
   */
  async getDailyTrends(
    environment?: string,
    days: number = 30
  ): Promise<DailyTrend[]> {
    const envFilter = environment ? `AND environment = '${environment}'` : '';

    const query = `
      SELECT
        toString(date) AS date,
        environment,
        total_requests,
        total_errors,
        (total_errors / total_requests) * 100 AS error_rate,
        avg_p95_latency,
        max_p95_latency
      FROM metrics_daily
      WHERE date >= today() - INTERVAL ${days} DAY
        ${envFilter}
      ORDER BY date DESC, environment
    `;

    return clickhouseClient.query<DailyTrend>(query);
  }

  /**
   * Get request rate trends
   */
  async getRequestRateTrends(
    environment?: string,
    hours: number = 24
  ): Promise<RequestRateTrend[]> {
    const envFilter = environment ? `AND environment = '${environment}'` : '';

    const query = `
      SELECT
        toString(hour) AS hour,
        environment,
        data_plane_id,
        requests_per_second
      FROM v_request_rate
      WHERE hour >= now() - INTERVAL ${hours} HOUR
        ${envFilter}
      ORDER BY hour DESC
    `;

    return clickhouseClient.query<RequestRateTrend>(query);
  }

  /**
   * Get latency by endpoint
   */
  async getLatencyByEndpoint(
    environment?: string,
    days: number = 7
  ): Promise<LatencyByEndpoint[]> {
    const envFilter = environment ? `AND environment = '${environment}'` : '';

    const query = `
      SELECT
        toString(date) AS date,
        environment,
        http_route,
        request_count,
        p50_latency,
        p95_latency,
        p99_latency
      FROM v_latency_by_endpoint
      WHERE date >= today() - INTERVAL ${days} DAY
        ${envFilter}
      ORDER BY date DESC, request_count DESC
      LIMIT 100
    `;

    return clickhouseClient.query<LatencyByEndpoint>(query);
  }

  /**
   * Get slowest endpoints
   */
  async getSlowestEndpoints(
    environment?: string,
    limit: number = 10
  ): Promise<SlowestEndpoint[]> {
    const envFilter = environment ? `WHERE environment = '${environment}'` : '';

    const query = `
      SELECT
        environment,
        http_route,
        request_count,
        p95_latency,
        max_latency
      FROM v_slowest_endpoints
      ${envFilter}
      ORDER BY p95_latency DESC
      LIMIT ${limit}
    `;

    return clickhouseClient.query<SlowestEndpoint>(query);
  }

  /**
   * Get monthly statistics
   */
  async getMonthlyStats(
    environment?: string,
    months: number = 6
  ): Promise<MonthlyStats[]> {
    const envFilter = environment ? `AND environment = '${environment}'` : '';

    const query = `
      SELECT
        toString(toStartOfMonth(date)) AS month,
        environment,
        sum(total_requests) AS monthly_requests,
        avg(avg_p95_latency) AS avg_latency,
        (sum(total_errors) / sum(total_requests)) * 100 AS error_rate
      FROM metrics_daily
      WHERE date >= today() - INTERVAL ${months} MONTH
        ${envFilter}
      GROUP BY month, environment
      ORDER BY month DESC, environment
    `;

    return clickhouseClient.query<MonthlyStats>(query);
  }

  /**
   * Get growth rate analysis
   */
  async getGrowthRate(
    environment?: string,
    months: number = 6
  ): Promise<GrowthRate[]> {
    const envFilter = environment ? `WHERE environment = '${environment}'` : '';

    const query = `
      WITH monthly_stats AS (
        SELECT
          toStartOfMonth(date) AS month,
          environment,
          sum(total_requests) AS requests
        FROM metrics_daily
        WHERE date >= today() - INTERVAL ${months} MONTH
        GROUP BY month, environment
      )
      SELECT
        toString(month) AS month,
        environment,
        requests,
        requests - lagInFrame(requests, 1) OVER (
          PARTITION BY environment ORDER BY month
        ) AS growth,
        round((requests - lagInFrame(requests, 1) OVER (
          PARTITION BY environment ORDER BY month
        )) / lagInFrame(requests, 1) OVER (
          PARTITION BY environment ORDER BY month
        ) * 100, 2) AS growth_percent
      FROM monthly_stats
      ${envFilter}
      ORDER BY month DESC, environment
    `;

    return clickhouseClient.query<GrowthRate>(query);
  }

  /**
   * Get environment comparison
   */
  async getEnvironmentComparison(days: number = 30): Promise<any[]> {
    const query = `
      SELECT
        environment,
        sum(total_requests) AS total_requests,
        sum(total_errors) AS total_errors,
        (sum(total_errors) / sum(total_requests)) * 100 AS error_rate,
        avg(avg_p95_latency) AS avg_latency
      FROM metrics_daily
      WHERE date >= today() - INTERVAL ${days} DAY
      GROUP BY environment
      ORDER BY total_requests DESC
    `;

    return clickhouseClient.query(query);
  }

  /**
   * Get cost attribution (request volume by data plane)
   */
  async getCostAttribution(
    environment?: string,
    days: number = 30
  ): Promise<any[]> {
    const envFilter = environment ? `AND environment = '${environment}'` : '';

    const query = `
      SELECT
        data_plane_id,
        environment,
        sum(request_count) AS total_requests,
        sum(error_count) AS total_errors,
        avg(p95_duration) AS avg_p95_latency
      FROM metrics_hourly
      WHERE date >= today() - INTERVAL ${days} DAY
        ${envFilter}
      GROUP BY data_plane_id, environment
      ORDER BY total_requests DESC
    `;

    return clickhouseClient.query(query);
  }
}

export default new AnalyticsService();
```

---

## Step 3: Create Analytics Components

### 3.1 Create Trend Chart Component

**Create `src/components/analytics/TrendChart.tsx`:**
```typescript
import React from 'react';
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  Legend,
  ResponsiveContainer,
} from 'recharts';

interface TrendChartProps {
  data: any[];
  xKey: string;
  yKeys: { key: string; name: string; color: string }[];
  title: string;
  yAxisLabel?: string;
  formatValue?: (value: number) => string;
}

const TrendChart: React.FC<TrendChartProps> = ({
  data,
  xKey,
  yKeys,
  title,
  yAxisLabel,
  formatValue,
}) => {
  return (
    <div className="trend-chart">
      <h3>{title}</h3>
      <ResponsiveContainer width="100%" height={300}>
        <LineChart data={data} margin={{ top: 5, right: 30, left: 20, bottom: 5 }}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey={xKey} />
          <YAxis label={{ value: yAxisLabel, angle: -90, position: 'insideLeft' }} />
          <Tooltip
            formatter={(value: number) =>
              formatValue ? formatValue(value) : value.toFixed(2)
            }
          />
          <Legend />
          {yKeys.map((yKey) => (
            <Line
              key={yKey.key}
              type="monotone"
              dataKey={yKey.key}
              stroke={yKey.color}
              name={yKey.name}
              strokeWidth={2}
            />
          ))}
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
};

export default TrendChart;
```

### 3.2 Create Comparison Table Component

**Create `src/components/analytics/ComparisonTable.tsx`:**
```typescript
import React from 'react';
import { formatNumber, formatDuration, formatPercentage } from '../../utils/formatters';

interface Column {
  key: string;
  label: string;
  format?: 'number' | 'duration' | 'percentage';
}

interface ComparisonTableProps {
  data: any[];
  columns: Column[];
  title: string;
}

const ComparisonTable: React.FC<ComparisonTableProps> = ({ data, columns, title }) => {
  const formatValue = (value: any, format?: string) => {
    if (value === null || value === undefined) return '-';

    switch (format) {
      case 'number':
        return formatNumber(value);
      case 'duration':
        return formatDuration(value);
      case 'percentage':
        return formatPercentage(value);
      default:
        return value;
    }
  };

  return (
    <div className="comparison-table">
      <h3>{title}</h3>
      <table>
        <thead>
          <tr>
            {columns.map((col) => (
              <th key={col.key}>{col.label}</th>
            ))}
          </tr>
        </thead>
        <tbody>
          {data.map((row, idx) => (
            <tr key={idx}>
              {columns.map((col) => (
                <td key={col.key}>{formatValue(row[col.key], col.format)}</td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};

export default ComparisonTable;
```

### 3.3 Create Summary Cards Component

**Create `src/components/analytics/SummaryCards.tsx`:**
```typescript
import React from 'react';
import { formatNumber, formatPercentage, formatDuration } from '../../utils/formatters';

interface SummaryCard {
  title: string;
  value: number;
  format: 'number' | 'percentage' | 'duration';
  change?: number;
  trend?: 'up' | 'down' | 'neutral';
}

interface SummaryCardsProps {
  cards: SummaryCard[];
}

const SummaryCards: React.FC<SummaryCardsProps> = ({ cards }) => {
  const formatValue = (value: number, format: string) => {
    switch (format) {
      case 'number':
        return formatNumber(value);
      case 'percentage':
        return formatPercentage(value);
      case 'duration':
        return formatDuration(value);
      default:
        return value;
    }
  };

  const getTrendIcon = (trend?: string) => {
    if (trend === 'up') return '↑';
    if (trend === 'down') return '↓';
    return '→';
  };

  const getTrendColor = (trend?: string) => {
    if (trend === 'up') return '#4caf50';
    if (trend === 'down') return '#f44336';
    return '#757575';
  };

  return (
    <div className="summary-cards">
      {cards.map((card, idx) => (
        <div key={idx} className="summary-card">
          <div className="card-title">{card.title}</div>
          <div className="card-value">{formatValue(card.value, card.format)}</div>
          {card.change !== undefined && (
            <div className="card-change" style={{ color: getTrendColor(card.trend) }}>
              {getTrendIcon(card.trend)} {Math.abs(card.change).toFixed(1)}%
            </div>
          )}
        </div>
      ))}
    </div>
  );
};

export default SummaryCards;
```

---

## Step 4: Create Analytics Dashboard Page

### 4.1 Create Analytics Dashboard

**Create `src/pages/AnalyticsDashboard.tsx`:**
```typescript
import React, { useState, useEffect } from 'react';
import { useSearchParams } from 'react-router-dom';
import analyticsService, {
  DailyTrend,
  MonthlyStats,
  GrowthRate,
  SlowestEndpoint,
} from '../services/clickhouse/analytics';
import clickhouseClient from '../services/clickhouse/client';
import TrendChart from '../components/analytics/TrendChart';
import ComparisonTable from '../components/analytics/ComparisonTable';
import SummaryCards from '../components/analytics/SummaryCards';
import TimeRangeSelector from '../components/TimeRangeSelector';
import DataPlaneSelector from '../components/DataPlaneSelector';

const AnalyticsDashboard: React.FC = () => {
  const [searchParams] = useSearchParams();
  const [selectedEnvironment, setSelectedEnvironment] = useState<string>(
    searchParams.get('env') || 'all'
  );
  const [timeRange, setTimeRange] = useState<number>(30); // days
  const [environments, setEnvironments] = useState<string[]>([]);
  const [loading, setLoading] = useState<boolean>(true);

  // Data state
  const [dailyTrends, setDailyTrends] = useState<DailyTrend[]>([]);
  const [monthlyStats, setMonthlyStats] = useState<MonthlyStats[]>([]);
  const [growthRate, setGrowthRate] = useState<GrowthRate[]>([]);
  const [slowestEndpoints, setSlowestEndpoints] = useState<SlowestEndpoint[]>([]);
  const [environmentComparison, setEnvironmentComparison] = useState<any[]>([]);

  // Fetch environments
  useEffect(() => {
    const fetchEnvironments = async () => {
      try {
        const envs = await clickhouseClient.getEnvironments();
        setEnvironments(envs);
      } catch (error) {
        console.error('Error fetching environments:', error);
      }
    };
    fetchEnvironments();
  }, []);

  // Fetch analytics data
  useEffect(() => {
    const fetchData = async () => {
      setLoading(true);
      try {
        const env = selectedEnvironment === 'all' ? undefined : selectedEnvironment;

        const [trends, monthly, growth, slowest, comparison] = await Promise.all([
          analyticsService.getDailyTrends(env, timeRange),
          analyticsService.getMonthlyStats(env, 6),
          analyticsService.getGrowthRate(env, 6),
          analyticsService.getSlowestEndpoints(env, 10),
          analyticsService.getEnvironmentComparison(timeRange),
        ]);

        setDailyTrends(trends);
        setMonthlyStats(monthly);
        setGrowthRate(growth);
        setSlowestEndpoints(slowest);
        setEnvironmentComparison(comparison);
      } catch (error) {
        console.error('Error fetching analytics data:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, [selectedEnvironment, timeRange]);

  // Calculate summary metrics
  const summaryCards = React.useMemo(() => {
    if (dailyTrends.length === 0) return [];

    const totalRequests = dailyTrends.reduce((sum, t) => sum + t.total_requests, 0);
    const avgErrorRate = dailyTrends.reduce((sum, t) => sum + t.error_rate, 0) / dailyTrends.length;
    const avgLatency = dailyTrends.reduce((sum, t) => sum + t.avg_p95_latency, 0) / dailyTrends.length;

    // Calculate growth from growth rate data
    const latestGrowth = growthRate.length > 0 ? growthRate[0].growth_percent : 0;

    return [
      {
        title: 'Total Requests',
        value: totalRequests,
        format: 'number' as const,
        change: latestGrowth,
        trend: (latestGrowth > 0 ? 'up' : latestGrowth < 0 ? 'down' : 'neutral') as const,
      },
      {
        title: 'Avg Error Rate',
        value: avgErrorRate,
        format: 'percentage' as const,
      },
      {
        title: 'Avg P95 Latency',
        value: avgLatency,
        format: 'duration' as const,
      },
    ];
  }, [dailyTrends, growthRate]);

  if (loading && dailyTrends.length === 0) {
    return <div className="loading">Loading analytics data...</div>;
  }

  return (
    <div className="analytics-dashboard">
      <header className="dashboard-header">
        <h1>Long-Term Analytics</h1>
        <div className="dashboard-controls">
          <DataPlaneSelector
            environments={environments}
            selectedEnvironment={selectedEnvironment}
            onEnvironmentChange={setSelectedEnvironment}
          />
          <select
            value={timeRange}
            onChange={(e) => setTimeRange(Number(e.target.value))}
            className="time-range-select"
          >
            <option value={7}>Last 7 days</option>
            <option value={30}>Last 30 days</option>
            <option value={90}>Last 90 days</option>
            <option value={180}>Last 6 months</option>
            <option value={365}>Last year</option>
          </select>
        </div>
      </header>

      {/* Summary Cards */}
      <SummaryCards cards={summaryCards} />

      {/* Daily Trends Chart */}
      <div className="chart-section">
        <TrendChart
          data={dailyTrends}
          xKey="date"
          yKeys={[
            { key: 'total_requests', name: 'Requests', color: '#1976d2' },
            { key: 'total_errors', name: 'Errors', color: '#f44336' },
          ]}
          title="Daily Request Volume"
          yAxisLabel="Count"
          formatValue={(v) => v.toLocaleString()}
        />
      </div>

      {/* Error Rate Trend */}
      <div className="chart-section">
        <TrendChart
          data={dailyTrends}
          xKey="date"
          yKeys={[
            { key: 'error_rate', name: 'Error Rate %', color: '#f44336' },
          ]}
          title="Error Rate Trend"
          yAxisLabel="Error Rate (%)"
          formatValue={(v) => `${v.toFixed(2)}%`}
        />
      </div>

      {/* Latency Trend */}
      <div className="chart-section">
        <TrendChart
          data={dailyTrends}
          xKey="date"
          yKeys={[
            { key: 'avg_p95_latency', name: 'Avg P95 Latency', color: '#4caf50' },
            { key: 'max_p95_latency', name: 'Max P95 Latency', color: '#ff9800' },
          ]}
          title="Latency Trend (P95)"
          yAxisLabel="Latency (s)"
          formatValue={(v) => `${(v * 1000).toFixed(0)}ms`}
        />
      </div>

      {/* Monthly Growth */}
      <div className="chart-section">
        <TrendChart
          data={monthlyStats}
          xKey="month"
          yKeys={[
            { key: 'monthly_requests', name: 'Monthly Requests', color: '#1976d2' },
          ]}
          title="Monthly Request Volume"
          yAxisLabel="Requests"
          formatValue={(v) => v.toLocaleString()}
        />
      </div>

      {/* Growth Rate Table */}
      <div className="table-section">
        <ComparisonTable
          data={growthRate}
          columns={[
            { key: 'month', label: 'Month' },
            { key: 'environment', label: 'Environment' },
            { key: 'requests', label: 'Requests', format: 'number' },
            { key: 'growth', label: 'Growth', format: 'number' },
            { key: 'growth_percent', label: 'Growth %', format: 'percentage' },
          ]}
          title="Month-over-Month Growth"
        />
      </div>

      {/* Environment Comparison */}
      <div className="table-section">
        <ComparisonTable
          data={environmentComparison}
          columns={[
            { key: 'environment', label: 'Environment' },
            { key: 'total_requests', label: 'Total Requests', format: 'number' },
            { key: 'total_errors', label: 'Errors', format: 'number' },
            { key: 'error_rate', label: 'Error Rate', format: 'percentage' },
            { key: 'avg_latency', label: 'Avg Latency', format: 'duration' },
          ]}
          title="Environment Comparison"
        />
      </div>

      {/* Slowest Endpoints */}
      <div className="table-section">
        <ComparisonTable
          data={slowestEndpoints}
          columns={[
            { key: 'environment', label: 'Environment' },
            { key: 'http_route', label: 'Endpoint' },
            { key: 'request_count', label: 'Requests', format: 'number' },
            { key: 'p95_latency', label: 'P95 Latency', format: 'duration' },
            { key: 'max_latency', label: 'Max Latency', format: 'duration' },
          ]}
          title="Slowest Endpoints (Last 7 Days)"
        />
      </div>
    </div>
  );
};

export default AnalyticsDashboard;
```

---

## Step 5: Update Main App Routes

### 5.1 Update `src/App.tsx`

```typescript
import React from 'react';
import { BrowserRouter as Router, Routes, Route, Link } from 'react-router-dom';
import LandingPage from './pages/LandingPage';
import MetricsDashboard from './pages/MetricsDashboard';
import AnalyticsDashboard from './pages/AnalyticsDashboard';
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
            <Link to="/dashboard">Real-time Metrics</Link>
            <Link to="/analytics">Analytics</Link>
          </div>
        </nav>

        <main className="app-main">
          <Routes>
            <Route path="/" element={<LandingPage />} />
            <Route path="/dashboard" element={<MetricsDashboard />} />
            <Route path="/analytics" element={<AnalyticsDashboard />} />
          </Routes>
        </main>

        <footer className="app-footer">
          <p>OTEL Prometheus PoC - Multi-Environment Observability & Analytics</p>
        </footer>
      </div>
    </Router>
  );
};

export default App;
```

### 5.2 Update Landing Page

**Update `src/pages/LandingPage.tsx` to include analytics link:**
```typescript
// Add to quick actions section
<div className="quick-actions">
  <Link to="/dashboard" className="btn-primary">
    View Real-time Metrics
  </Link>
  <Link to="/analytics" className="btn-secondary">
    View Long-term Analytics
  </Link>
</div>
```

---

## Step 6: Add Styling

### 6.1 Create Analytics Styles

**Create `src/styles/analytics.css`:**
```css
/* Analytics Dashboard */
.analytics-dashboard {
  padding: 2rem;
}

.dashboard-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
}

.dashboard-controls {
  display: flex;
  gap: 1rem;
}

/* Summary Cards */
.summary-cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1.5rem;
  margin-bottom: 2rem;
}

.summary-card {
  background: white;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.card-title {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 0.5rem;
}

.card-value {
  font-size: 2rem;
  font-weight: bold;
  color: #333;
  margin-bottom: 0.5rem;
}

.card-change {
  font-size: 0.9rem;
  font-weight: 500;
}

/* Chart Sections */
.chart-section {
  background: white;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  margin-bottom: 2rem;
}

.chart-section h3 {
  margin-bottom: 1rem;
  color: #333;
}

/* Table Sections */
.table-section {
  background: white;
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  margin-bottom: 2rem;
  overflow-x: auto;
}

.table-section h3 {
  margin-bottom: 1rem;
  color: #333;
}

.comparison-table table {
  width: 100%;
  border-collapse: collapse;
}

.comparison-table th {
  background-color: #f5f5f5;
  padding: 0.75rem;
  text-align: left;
  font-weight: 600;
  border-bottom: 2px solid #ddd;
}

.comparison-table td {
  padding: 0.75rem;
  border-bottom: 1px solid #eee;
}

.comparison-table tr:hover {
  background-color: #f9f9f9;
}

/* Trend Chart */
.trend-chart h3 {
  margin-bottom: 1rem;
  color: #333;
}

/* Loading State */
.loading {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 400px;
  font-size: 1.2rem;
  color: #666;
}
```

### 6.2 Import Styles in App

**Update `src/App.tsx`:**
```typescript
import './App.css';
import './styles/analytics.css';
```

---

## Step 7: Configure Nginx Proxy

### 7.1 Update `nginx.conf`

**Update `controlplane/dashboard/nginx.conf`:**
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

    # Proxy Prometheus API
    location /api/v1/ {
        proxy_pass http://prometheus:9090/api/v1/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Proxy ClickHouse HTTP interface
    location /clickhouse/ {
        proxy_pass http://clickhouse:8123/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Increase timeouts for long-running queries
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
}
```

---

## Step 8: Environment Configuration

### 8.1 Update `.env`

```bash
# ClickHouse Configuration
CLICKHOUSE_PORT=8123
CLICKHOUSE_NATIVE_PORT=9000
CLICKHOUSE_DASHBOARD_PASSWORD=dashboard123

# Dashboard Configuration
REACT_APP_CLICKHOUSE_URL=http://localhost:8123
REACT_APP_CLICKHOUSE_PASSWORD=dashboard123
```

---

## Step 9: Testing

### 9.1 Build and Deploy

```bash
# Build dashboard with new analytics features
cd controlplane/dashboard
npm run build

# Rebuild dashboard container
docker-compose build dashboard

# Restart dashboard
docker-compose up -d dashboard
```

### 9.2 Verify Analytics Dashboard

```bash
# Access dashboard
open http://localhost:3000/analytics

# Check ClickHouse connectivity
curl http://localhost:8123/ping
```

### 9.3 Test Queries

**Open browser console and test:**
```javascript
// Test ClickHouse connection
fetch('http://localhost:8123/?query=SELECT 1')
  .then(r => r.text())
  .then(console.log);

// Test analytics query
fetch('http://localhost:8123/?query=SELECT count() FROM otel_metrics.metrics_local')
  .then(r => r.text())
  .then(console.log);
```

---

## Expected Outcomes

✅ **Analytics Dashboard Functional**:
- New analytics route accessible
- ClickHouse queries working
- Charts displaying long-term trends
- Tables showing comparisons

✅ **Features Working**:
- Environment filtering
- Time range selection
- Daily/monthly trends
- Growth rate analysis
- Slowest endpoints table
- Environment comparison

✅ **Performance**:
- Dashboard loads in <3 seconds
- Queries complete in <5 seconds
- Charts render smoothly
- No errors in browser console

---

## Troubleshooting

**Issue: ClickHouse connection fails**
```bash
# Check ClickHouse is running
docker-compose ps clickhouse

# Test connectivity
docker exec dashboard curl http://clickhouse:8123/ping

# Check nginx proxy config
docker exec dashboard cat /etc/nginx/conf.d/default.conf
```

**Issue: No data in analytics**
```bash
# Verify data in ClickHouse
docker exec clickhouse clickhouse-client --query "
SELECT count() FROM otel_metrics.metrics_local
"

# Check if materialized views are populated
docker exec clickhouse clickhouse-client --query "
SELECT count() FROM otel_metrics.metrics_hourly
"
```

**Issue: Slow queries**
```bash
# Check query execution
docker exec clickhouse clickhouse-client --query "
EXPLAIN SELECT * FROM metrics_local WHERE date >= today() - 7
"

# Monitor query log
docker exec clickhouse clickhouse-client --query "
SELECT query, query_duration_ms FROM system.query_log
ORDER BY query_duration_ms DESC LIMIT 10
"
```

---

## Next Steps

Proceed to **Phase 12: Integration and Optimization** to fine-tune the analytics pipeline and add advanced features.

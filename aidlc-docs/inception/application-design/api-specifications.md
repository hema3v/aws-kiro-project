# API Specifications
## Data Integration and Analytics Platform

**Version**: 1.0  
**Date**: 2026-03-04  
**API Style**: RESTful with OpenAPI 3.0

---

## 1. API Overview

All services expose REST APIs following consistent patterns:
- RESTful resource naming
- HTTP methods (GET, POST, PUT, DELETE, PATCH)
- JSON request/response format
- Standard HTTP status codes
- JWT authentication
- Versioned endpoints (/api/v1/)

---

## 2. Data Ingestion Service API

**Base URL**: `/api/v1/ingestion`

### 2.1 File Upload

```
POST /api/v1/ingestion/files/upload
```

**Description**: Upload data file for processing

**Request**:
```json
Content-Type: multipart/form-data

{
  "file": <binary>,
  "sourceType": "csv|json|xml|excel",
  "businessId": "string",
  "metadata": {
    "description": "string",
    "tags": ["string"]
  }
}
```

**Response** (202 Accepted):
```json
{
  "jobId": "uuid",
  "status": "accepted",
  "fileName": "string",
  "recordCount": 0,
  "message": "File accepted for processing"
}
```

### 2.2 Database Connection

```
POST /api/v1/ingestion/connections/database
```

**Description**: Create database connection configuration

**Request**:
```json
{
  "name": "string",
  "type": "postgresql|mysql|mongodb|oracle",
  "host": "string",
  "port": 5432,
  "database": "string",
  "username": "string",
  "password": "string (encrypted)",
  "ssl": true,
  "extractionQuery": "string",
  "schedule": "cron expression"
}
```

**Response** (201 Created):
```json
{
  "connectionId": "uuid",
  "name": "string",
  "status": "active",
  "lastSync": "2026-03-04T10:00:00Z",
  "nextSync": "2026-03-04T11:00:00Z"
}
```

### 2.3 API Integration

```
POST /api/v1/ingestion/connections/api
```

**Description**: Configure external API integration

**Request**:
```json
{
  "name": "string",
  "url": "string",
  "method": "GET|POST",
  "authType": "oauth2|apikey|basic|jwt",
  "authConfig": {
    "clientId": "string",
    "clientSecret": "string",
    "tokenUrl": "string"
  },
  "headers": {
    "key": "value"
  },
  "schedule": "cron expression",
  "dataPath": "$.data.items"
}
```

**Response** (201 Created):
```json
{
  "integrationId": "uuid",
  "name": "string",
  "status": "active",
  "lastPoll": "2026-03-04T10:00:00Z"
}
```

### 2.4 Webhook Receiver

```
POST /api/v1/ingestion/webhooks/{webhookId}
```

**Description**: Receive webhook data from external systems

**Request**: Any JSON payload

**Response** (200 OK):
```json
{
  "received": true,
  "jobId": "uuid",
  "timestamp": "2026-03-04T10:00:00Z"
}
```

### 2.5 Job Status

```
GET /api/v1/ingestion/jobs/{jobId}
```

**Response**:
```json
{
  "jobId": "uuid",
  "status": "pending|processing|completed|failed",
  "source": "file|database|api|stream",
  "recordsProcessed": 1000,
  "recordsFailed": 5,
  "startTime": "2026-03-04T10:00:00Z",
  "endTime": "2026-03-04T10:05:00Z",
  "errors": [
    {
      "recordId": "string",
      "error": "string"
    }
  ]
}
```

---

## 3. Data Processing Pipeline API

**Base URL**: `/api/v1/processing`

### 3.1 Reprocess Data

```
POST /api/v1/processing/reprocess
```

**Description**: Manually trigger reprocessing of failed records

**Request**:
```json
{
  "jobId": "uuid",
  "recordIds": ["string"],
  "force": false
}
```

**Response** (202 Accepted):
```json
{
  "reprocessJobId": "uuid",
  "recordCount": 5,
  "status": "queued"
}
```

### 3.2 Data Quality Metrics

```
GET /api/v1/processing/quality/metrics
```

**Query Parameters**:
- `startDate`: ISO date
- `endDate`: ISO date
- `source`: string (optional)

**Response**:
```json
{
  "period": {
    "start": "2026-03-01T00:00:00Z",
    "end": "2026-03-04T23:59:59Z"
  },
  "metrics": {
    "totalRecords": 1000000,
    "validRecords": 995000,
    "invalidRecords": 5000,
    "completeness": 99.5,
    "accuracy": 98.5,
    "consistency": 99.0
  },
  "bySource": [
    {
      "source": "database-1",
      "validRecords": 500000,
      "invalidRecords": 2000,
      "qualityScore": 99.6
    }
  ]
}
```

### 3.3 Validation Rules

```
GET /api/v1/processing/rules
POST /api/v1/processing/rules
PUT /api/v1/processing/rules/{ruleId}
DELETE /api/v1/processing/rules/{ruleId}
```

**Rule Object**:
```json
{
  "ruleId": "uuid",
  "name": "string",
  "type": "schema|business|quality",
  "condition": "string (expression)",
  "action": "reject|warn|correct",
  "enabled": true,
  "priority": 1
}
```

---

## 4. Storage Management Service API

**Base URL**: `/api/v1/storage`

### 4.1 Storage Statistics

```
GET /api/v1/storage/stats
```

**Response**:
```json
{
  "databases": [
    {
      "type": "postgresql",
      "name": "analytics-db",
      "recordCount": 5000000,
      "sizeGB": 50.5,
      "lastWrite": "2026-03-04T10:00:00Z"
    },
    {
      "type": "mongodb",
      "name": "documents-db",
      "recordCount": 2000000,
      "sizeGB": 30.2,
      "lastWrite": "2026-03-04T10:00:00Z"
    }
  ],
  "totalRecords": 7000000,
  "totalSizeGB": 80.7
}
```

### 4.2 Schema Management

```
GET /api/v1/storage/schemas
POST /api/v1/storage/schemas
GET /api/v1/storage/schemas/{schemaId}
PUT /api/v1/storage/schemas/{schemaId}
```

**Schema Object**:
```json
{
  "schemaId": "uuid",
  "name": "string",
  "version": "1.0.0",
  "database": "postgresql|mongodb|timescaledb",
  "fields": [
    {
      "name": "string",
      "type": "string|number|date|boolean",
      "required": true,
      "indexed": true
    }
  ],
  "createdAt": "2026-03-04T10:00:00Z"
}
```

---

## 5. Analytics API Service

**Base URL**: `/api/v1/analytics`

### 5.1 KPIs

```
GET /api/v1/analytics/kpis
```

**Query Parameters**:
- `businessId`: string (optional)
- `startDate`: ISO date
- `endDate`: ISO date
- `metrics`: comma-separated list

**Response**:
```json
{
  "period": {
    "start": "2026-03-01T00:00:00Z",
    "end": "2026-03-04T23:59:59Z"
  },
  "kpis": {
    "totalRevenue": 1500000.00,
    "revenueGrowth": 15.5,
    "totalCustomers": 5000,
    "newCustomers": 500,
    "customerRetention": 85.5,
    "averageOrderValue": 300.00
  }
}
```

### 5.2 Revenue Analytics

```
GET /api/v1/analytics/revenue
```

**Query Parameters**:
- `businessId`: string (optional)
- `startDate`: ISO date
- `endDate`: ISO date
- `groupBy`: day|week|month|quarter|year

**Response**:
```json
{
  "period": {
    "start": "2026-01-01T00:00:00Z",
    "end": "2026-03-04T23:59:59Z"
  },
  "groupBy": "month",
  "data": [
    {
      "period": "2026-01",
      "revenue": 500000.00,
      "orders": 1500,
      "averageOrderValue": 333.33
    },
    {
      "period": "2026-02",
      "revenue": 550000.00,
      "orders": 1600,
      "averageOrderValue": 343.75
    }
  ],
  "totals": {
    "revenue": 1050000.00,
    "orders": 3100
  }
}
```

### 5.3 Customer Analytics

```
GET /api/v1/analytics/customers
```

**Response**:
```json
{
  "totalCustomers": 5000,
  "activeCustomers": 3500,
  "newCustomers": 500,
  "churnedCustomers": 100,
  "segments": [
    {
      "name": "High Value",
      "count": 500,
      "percentage": 10.0,
      "averageLifetimeValue": 5000.00
    }
  ],
  "acquisition": {
    "channels": [
      {
        "name": "Organic",
        "count": 200,
        "percentage": 40.0
      }
    ]
  }
}
```

### 5.4 Trends Analysis

```
GET /api/v1/analytics/trends
```

**Query Parameters**:
- `metric`: revenue|customers|orders
- `startDate`: ISO date
- `endDate`: ISO date
- `forecast`: boolean (default: false)

**Response**:
```json
{
  "metric": "revenue",
  "period": {
    "start": "2026-01-01T00:00:00Z",
    "end": "2026-03-04T23:59:59Z"
  },
  "historical": [
    {
      "date": "2026-01-01",
      "value": 15000.00
    }
  ],
  "trend": {
    "direction": "up|down|stable",
    "percentage": 15.5,
    "confidence": 0.95
  },
  "forecast": [
    {
      "date": "2026-03-05",
      "predicted": 16000.00,
      "lower": 15000.00,
      "upper": 17000.00
    }
  ]
}
```

### 5.5 Custom Query

```
POST /api/v1/analytics/query
```

**Description**: Execute custom analytical query

**Request**:
```json
{
  "database": "postgresql|mongodb|timescaledb|snowflake",
  "query": {
    "select": ["field1", "field2"],
    "from": "table",
    "where": {
      "field": "value"
    },
    "groupBy": ["field1"],
    "orderBy": {
      "field": "asc|desc"
    },
    "limit": 100
  }
}
```

**Response**:
```json
{
  "queryId": "uuid",
  "executionTime": 150,
  "rowCount": 100,
  "data": [
    {
      "field1": "value1",
      "field2": "value2"
    }
  ]
}
```

### 5.6 Report Export

```
GET /api/v1/analytics/reports/{reportId}/export
```

**Query Parameters**:
- `format`: csv|excel|pdf
- `includeCharts`: boolean

**Response**: Binary file download

---

## 6. Monitoring & Audit Service API

**Base URL**: `/api/v1/monitoring`

### 6.1 Data Lineage

```
GET /api/v1/monitoring/lineage/{recordId}
```

**Response**:
```json
{
  "recordId": "uuid",
  "currentLocation": {
    "database": "postgresql",
    "table": "customers",
    "timestamp": "2026-03-04T10:00:00Z"
  },
  "history": [
    {
      "timestamp": "2026-03-04T09:00:00Z",
      "event": "ingested",
      "source": "api-integration-1",
      "details": {
        "endpoint": "https://api.example.com/customers"
      }
    },
    {
      "timestamp": "2026-03-04T09:05:00Z",
      "event": "validated",
      "service": "data-processing-pipeline",
      "rules": ["schema-validation", "business-rules"]
    },
    {
      "timestamp": "2026-03-04T09:10:00Z",
      "event": "transformed",
      "service": "data-processing-pipeline",
      "transformations": ["normalize-phone", "standardize-address"]
    },
    {
      "timestamp": "2026-03-04T09:15:00Z",
      "event": "stored",
      "database": "postgresql",
      "table": "customers"
    }
  ]
}
```

### 6.2 Audit Logs

```
GET /api/v1/monitoring/audit
```

**Query Parameters**:
- `startDate`: ISO date
- `endDate`: ISO date
- `userId`: string (optional)
- `action`: string (optional)
- `resource`: string (optional)

**Response**:
```json
{
  "logs": [
    {
      "logId": "uuid",
      "timestamp": "2026-03-04T10:00:00Z",
      "userId": "user-123",
      "action": "query",
      "resource": "/api/v1/analytics/revenue",
      "ipAddress": "192.168.1.1",
      "userAgent": "Mozilla/5.0...",
      "status": "success",
      "duration": 150
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 50,
    "totalPages": 10,
    "totalRecords": 500
  }
}
```

### 6.3 System Health

```
GET /api/v1/monitoring/health
```

**Response**:
```json
{
  "status": "healthy|degraded|unhealthy",
  "timestamp": "2026-03-04T10:00:00Z",
  "services": [
    {
      "name": "data-ingestion",
      "status": "healthy",
      "uptime": 99.99,
      "lastCheck": "2026-03-04T10:00:00Z"
    },
    {
      "name": "data-processing",
      "status": "healthy",
      "uptime": 99.95,
      "lastCheck": "2026-03-04T10:00:00Z"
    }
  ],
  "databases": [
    {
      "name": "postgresql",
      "status": "healthy",
      "connections": 50,
      "maxConnections": 100
    }
  ]
}
```

### 6.4 Metrics

```
GET /api/v1/monitoring/metrics
```

**Query Parameters**:
- `metric`: string (cpu|memory|requests|errors)
- `service`: string (optional)
- `startDate`: ISO date
- `endDate`: ISO date

**Response**:
```json
{
  "metric": "requests",
  "service": "analytics-api",
  "period": {
    "start": "2026-03-04T09:00:00Z",
    "end": "2026-03-04T10:00:00Z"
  },
  "dataPoints": [
    {
      "timestamp": "2026-03-04T09:00:00Z",
      "value": 150
    }
  ],
  "statistics": {
    "min": 100,
    "max": 200,
    "avg": 150,
    "p95": 180,
    "p99": 195
  }
}
```

---

## 7. Authentication & Authorization API

**Base URL**: `/api/v1/auth`

### 7.1 Login

```
POST /api/v1/auth/login
```

**Request**:
```json
{
  "username": "string",
  "password": "string"
}
```

**Response**:
```json
{
  "accessToken": "jwt-token",
  "refreshToken": "jwt-token",
  "expiresIn": 3600,
  "tokenType": "Bearer",
  "user": {
    "userId": "uuid",
    "username": "string",
    "email": "string",
    "roles": ["admin", "analyst"],
    "businessIds": ["business-1", "business-2"]
  }
}
```

### 7.2 Refresh Token

```
POST /api/v1/auth/refresh
```

**Request**:
```json
{
  "refreshToken": "jwt-token"
}
```

**Response**: Same as login

### 7.3 Logout

```
POST /api/v1/auth/logout
```

**Request**:
```json
{
  "refreshToken": "jwt-token"
}
```

**Response** (204 No Content)

---

## 8. Common API Patterns

### 8.1 Pagination

All list endpoints support pagination:

**Query Parameters**:
- `page`: integer (default: 1)
- `pageSize`: integer (default: 50, max: 1000)
- `sort`: field name
- `order`: asc|desc

**Response**:
```json
{
  "data": [],
  "pagination": {
    "page": 1,
    "pageSize": 50,
    "totalPages": 10,
    "totalRecords": 500
  }
}
```

### 8.2 Filtering

**Query Parameters**:
- `filter[field]`: value
- `filter[field][operator]`: value

**Operators**: eq, ne, gt, gte, lt, lte, like, in

**Example**: `/api/v1/analytics/customers?filter[status]=active&filter[revenue][gte]=1000`

### 8.3 Error Responses

**Standard Error Format**:
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable message",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ],
    "timestamp": "2026-03-04T10:00:00Z",
    "path": "/api/v1/resource",
    "requestId": "uuid"
  }
}
```

**HTTP Status Codes**:
- 200: Success
- 201: Created
- 202: Accepted (async processing)
- 204: No Content
- 400: Bad Request
- 401: Unauthorized
- 403: Forbidden
- 404: Not Found
- 409: Conflict
- 422: Unprocessable Entity
- 429: Too Many Requests
- 500: Internal Server Error
- 503: Service Unavailable

### 8.4 Rate Limiting

**Headers**:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1709553600
```

**Limits**:
- Authenticated: 1000 requests/hour
- Unauthenticated: 100 requests/hour
- Admin: 5000 requests/hour

---

## 9. WebSocket API

**Base URL**: `ws://host/ws`

### 9.1 Real-Time Dashboard Updates

**Connection**: `ws://host/ws/dashboard?token=jwt-token`

**Subscribe to Updates**:
```json
{
  "action": "subscribe",
  "channels": ["kpis", "alerts", "data-quality"]
}
```

**Receive Updates**:
```json
{
  "channel": "kpis",
  "timestamp": "2026-03-04T10:00:00Z",
  "data": {
    "metric": "revenue",
    "value": 1500000.00,
    "change": 15.5
  }
}
```

---

## 10. API Versioning

**Strategy**: URL-based versioning

**Current Version**: v1  
**Base Path**: `/api/v1/`

**Deprecation Policy**:
- 6 months notice before deprecation
- Support for 2 versions simultaneously
- Deprecation warnings in response headers

---

## Document Status

**Status**: Draft - Awaiting Approval  
**Prepared By**: AI Development Assistant  
**Date**: 2026-03-04  
**Next**: Data Models

---

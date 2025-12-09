# 📘 MCP Server Documentation - GenAI Toolbox

Detailed documentation on configuration and usage of MCP (Model Context Protocol) Server for GenAI Toolbox.

## 📋 Overview

GenAI Toolbox MCP Server provides tools to interact with MySQL databases through AI agents. The server supports various operations from connection checks and simple queries to complex monitoring and diagnostics.

## ⚙️ Configuration File: tools.yaml

### 1. Sources (Database Connections)

```yaml
sources:
  mysql-source:
    kind: mysql # Database type
    host: localhost # Database host
    port: 3306 # Database port
    database: your_database_name # Database name
    user: ${MYSQL_USER} # Username from env
    password: ${MYSQL_PASSWORD} # Password from env
    queryTimeout: 30s # Query timeout
```

**Supported Parameters:**

- `kind`: `mysql`, `postgres`, `mssql`, `bigquery`, etc.
- `host`: Database hostname or IP
- `port`: Database port
- `database`: Database name
- `user`: Database username (support env vars)
- `password`: Database password (support env vars)
- `queryTimeout`: Maximum query execution time
- `queryParams`: Additional connection parameters (optional)

### 2. Tools

#### Tool Type 1: Execute SQL (mysql-execute-sql)

Allows execution of any SQL query.

```yaml
tools:
  db_run_query:
    kind: mysql-execute-sql
    source: mysql-source
    description: "Execute a SQL query against the database"
```

**Schema:**

- **Input:** SQL query string
- **Output:** JSON array of results
- **Use cases:** SELECT, INSERT, UPDATE, DELETE, DDL

**Example Usage:**

```sql
SELECT * FROM users LIMIT 10;
SHOW TABLES;
DESCRIBE table_name;
```

---

#### Tool Type 2: Templated SQL Query (mysql-sql)

Allows creating SQL queries with dynamic parameters.

```yaml
tools:
  db_check_file_size:
    kind: mysql-sql
    source: mysql-source
    description: "Check database file size and storage usage"
    statement: |
      SELECT 
        table_schema AS 'Database',
        ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)',
        ROUND(SUM(data_free) / 1024 / 1024, 2) AS 'Free Space (MB)'
      FROM information_schema.tables
      WHERE table_schema = {{.database_name}}
      GROUP BY table_schema;
    templateParameters:
      - name: database_name
        type: string
        description: "Name of the database to check"
        required: false
        default: "'your_database_name'"
```

**Template Parameters Types:**

- `string`: Text values
- `integer`: Numeric values
- `boolean`: true/false

**Schema:**

- **Input:** Parameters defined in `templateParameters`
- **Output:** Query results as JSON
- **Template Syntax:** `{{.parameter_name}}`

---

## 🛠️ Available Tools

### 1. db_connection_check

**Type:** `mysql-sql`

**Purpose:** Check database connection status

**Description:**
Verifies that the database connection is working properly and returns connection information including connection ID, database name, current user, MySQL version, server time, and connection status.

**Parameters:**

- `database_name` (string, optional): Name of database to check (default: "your_database_name")

**Schema:**

```json
{
  "name": "db_connection_check",
  "description": "Check whether the DB of the operator's system is connected normally",
  "inputSchema": {
    "type": "object",
    "properties": {
      "database_name": {
        "type": "string",
        "description": "Name of database to check",
        "default": "your_database_name"
      }
    }
  }
}
```

**SQL Query:**

```sql
SELECT
  CONNECTION_ID() as connection_id,
  '{{.database_name}}' as database_name,
  (SELECT SCHEMA_NAME FROM information_schema.SCHEMATA WHERE SCHEMA_NAME = '{{.database_name}}') as database_exists,
  USER() as user_name,
  VERSION() as mysql_version,
  NOW() as server_time,
  CASE
    WHEN (SELECT SCHEMA_NAME FROM information_schema.SCHEMATA WHERE SCHEMA_NAME = '{{.database_name}}') IS NOT NULL
    THEN 'Connected'
    ELSE 'Database not found'
  END as status;
```

**Output Fields:**

- `connection_id`: Current MySQL connection ID
- `database_name`: Database name being checked
- `database_exists`: Database name if it exists, NULL otherwise
- `user_name`: Current user with host
- `mysql_version`: MySQL server version
- `server_time`: Current server timestamp
- `status`: Connection status ("Connected" or "Database not found")

**Example Prompts:**

- "Check if the database is connected"
- "Verify database connection status"
- "Is the DB connection working?"
- "Check if database 'mysql' exists"
- "Show me database connection information for your_database_name"

---

### 2. db_run_query

**Type:** `mysql-execute-sql`

**Purpose:** Execute any SQL query directly

**Description:**
Allows users to execute any SQL query through the AI agent. No parameters required, only SQL query string.

**Schema:**

```json
{
  "name": "db_run_query",
  "description": "Execute a SQL query against selected database",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "SQL query to execute"
      }
    },
    "required": ["query"]
  }
}
```

**Example Queries:**

```sql
-- View tables
SHOW TABLES;

-- Count records
SELECT COUNT(*) FROM users;

-- Get recent data
SELECT * FROM access_log WHERE access_time >= DATE_SUB(NOW(), INTERVAL 1 DAY);
```

---

### 3. db_check_response_time

**Type:** `mysql-sql`

**Purpose:** Measure query execution time and identify slow queries

**Description:**
Queries MySQL slow log to find slow-running queries. Useful for performance tuning.

**Parameters:**

- `database_name` (string, optional): Name of database to check (default: "your_database_name")
- `threshold_seconds` (integer, optional): Minimum execution time (default: 1)
- `limit` (integer, optional): Max results (default: 10)

**Schema:**

```json
{
  "name": "db_check_response_time",
  "description": "Identify slow-running queries",
  "inputSchema": {
    "type": "object",
    "properties": {
      "database_name": {
        "type": "string",
        "description": "Name of database to check",
        "default": "your_database_name"
      },
      "threshold_seconds": {
        "type": "integer",
        "description": "Minimum query time in seconds",
        "default": 1
      },
      "limit": {
        "type": "integer",
        "description": "Max number of results",
        "default": 10
      }
    }
  }
}
```

**SQL Query:**

```sql
USE {{.database_name}};
SELECT
  query_time,
  lock_time,
  rows_sent,
  rows_examined,
  sql_text,
  start_time
FROM mysql.slow_log
WHERE query_time > {{.threshold_seconds}}
ORDER BY query_time DESC
LIMIT {{.limit}};
```

**Prerequisites:**

```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
```

---

### 4. db_check_deadlock

**Type:** `mysql-sql`

**Purpose:** Detect active deadlocks or blocking sessions

**Description:**
Detects deadlocks and blocking sessions currently occurring in the database.

**Parameters:**

- `database_name` (string, optional): Name of database to check (default: "your_database_name")

**Schema:**

```json
{
  "name": "db_check_deadlock",
  "description": "Detect active deadlocks or blocking sessions",
  "inputSchema": {
    "type": "object",
    "properties": {
      "database_name": {
        "type": "string",
        "description": "Name of database to check",
        "default": "your_database_name"
      }
    }
  }
}
```

**SQL Query:**

```sql
USE {{.database_name}};
SELECT
  r.trx_id AS waiting_trx_id,
  r.trx_mysql_thread_id AS waiting_thread,
  r.trx_query AS waiting_query,
  r.trx_state AS waiting_state,
  r.trx_wait_started AS wait_started,
  TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW()) AS wait_age_seconds,
  b.trx_id AS blocking_trx_id,
  b.trx_mysql_thread_id AS blocking_thread,
  b.trx_query AS blocking_query,
  b.trx_state AS blocking_state,
  w.requesting_engine_lock_id,
  w.blocking_engine_lock_id
FROM performance_schema.data_lock_waits w
INNER JOIN information_schema.innodb_trx b
  ON b.trx_id = w.blocking_engine_transaction_id
INNER JOIN information_schema.innodb_trx r
  ON r.trx_id = w.requesting_engine_transaction_id
ORDER BY wait_age_seconds DESC;
```

**Output Fields:**

- `waiting_trx_id`: ID of waiting transaction
- `waiting_thread`: Thread ID waiting
- `waiting_query`: Query being blocked
- `wait_age_seconds`: How long it's been waiting
- `blocking_trx_id`: ID of blocking transaction
- `blocking_thread`: Thread ID causing block
- `blocking_query`: Query causing the block

**Example Prompts:**

- "Check if there are any deadlocks in the DB right now"
- "Show blocking sessions and deadlock chains"
- "Are there any locks preventing queries from running?"

---

### 5. db_check_file_size

**Type:** `mysql-sql`

**Purpose:** Check database file size and storage usage

**Description:**
Queries database metadata to view file size, storage usage, and growth details.

**Parameters:**

- `database_name` (string, optional): Database name (default: 'your_database_name')

**Schema:**

```json
{
  "name": "db_check_file_size",
  "description": "Retrieve database file size and storage usage",
  "inputSchema": {
    "type": "object",
    "properties": {
      "database_name": {
        "type": "string",
        "description": "Database name to check",
        "default": "your_database_name"
      }
    }
  }
}
```

**SQL Query:**

```sql
SELECT
  '{{.database_name}}' AS 'Database',
  ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)',
  ROUND(SUM(data_free) / 1024 / 1024, 2) AS 'Free Space (MB)',
  ROUND((SUM(data_length + index_length) / 1024 / 1024 / 1024), 2) AS 'Size (GB)'
FROM information_schema.tables
WHERE table_schema = '{{.database_name}}'
GROUP BY table_schema;
```

**Output Fields:**

- `Database`: Database name
- `Size (MB)`: Total size in megabytes
- `Free Space (MB)`: Available free space
- `Size (GB)`: Total size in gigabytes

**Example Prompts:**

- "Show DB file size and remaining capacity"
- "Check tablespace usage for member database"
- "How much space is the database using?"

---

### 6. db_check_if_data

**Type:** `mysql-sql`

**Purpose:** Verify interface (I/F) data

**Description:**
Queries interface tables to verify that data has been properly received, processed, or transferred.

**Parameters:**

- `database_name` (string, optional): Database name (default: "your_database_name")
- `interface_table` (string, required): Interface table name
- `filter_condition` (string, optional): WHERE condition (default: "")
- `order_column` (string, optional): Sort column (default: "")
- `limit` (integer, optional): Max records (default: 100)

**Schema:**

```json
{
  "name": "db_check_if_data",
  "description": "Verify interface data across systems",
  "inputSchema": {
    "type": "object",
    "properties": {
      "database_name": {
        "type": "string",
        "description": "Database name",
        "default": "your_database_name"
      },
      "interface_table": {
        "type": "string",
        "description": "Interface table name",
        "required": true
      },
      "filter_condition": {
        "type": "string",
        "description": "SQL WHERE condition",
        "default": ""
      },
      "order_column": {
        "type": "string",
        "description": "Column to order by",
        "default": ""
      },
      "limit": {
        "type": "integer",
        "description": "Max records to return",
        "default": 100
      }
    },
    "required": ["interface_table"]
  }
}
```

**SQL Query:**

```sql
USE {{.database_name}};
SELECT *
FROM {{.interface_table}}
{{if .filter_condition}}WHERE {{.filter_condition}}{{end}}
{{if .order_column}}ORDER BY {{.order_column}} DESC{{end}}
LIMIT {{.limit}};
```

**Example Usage:**

**For `golf_reservation` table:**

```json
{
  "database_name": "your_database_name",
  "interface_table": "golf_reservation",
  "filter_condition": "status = 'PENDING'",
  "order_column": "created_at",
  "limit": 50
}
```

**For `idcard_request` table:**

```json
{
  "database_name": "your_database_name",
  "interface_table": "idcard_request",
  "filter_condition": "created_at > NOW() - INTERVAL 24 HOUR",
  "order_column": "created_at",
  "limit": 100
}
```

**Example Prompts:**

- "Check today's interface data for the HR system"
- "Show me I/F records that failed in the last 1 hour"
- "Verify if new policy data arrived in the interface table"
- "Is there any I/F data stuck in PENDING status?"

---

### 7. db_check_batch_data

**Type:** `mysql-sql`

**Purpose:** Check batch processing results

**Description:**
Queries batch job tables to identify successes, failures, pending jobs, and error logs.

**Parameters:**

- `database_name` (string, optional): Database name (default: "your_database_name")
- `batch_table` (string, required): Batch table name
- `filter_condition` (string, optional): WHERE condition (default: "")
- `order_column` (string, optional): Sort column (default: "")
- `limit` (integer, optional): Max records (default: 100)

**Schema:**

```json
{
  "name": "db_check_batch_data",
  "description": "Check batch processing results",
  "inputSchema": {
    "type": "object",
    "properties": {
      "database_name": {
        "type": "string",
        "description": "Database name",
        "default": "your_database_name"
      },
      "batch_table": {
        "type": "string",
        "description": "Batch table name",
        "required": true
      },
      "filter_condition": {
        "type": "string",
        "description": "SQL WHERE condition",
        "default": ""
      },
      "order_column": {
        "type": "string",
        "description": "Column to order by",
        "default": ""
      },
      "limit": {
        "type": "integer",
        "description": "Max records to return",
        "default": 100
      }
    },
    "required": ["batch_table"]
  }
}
```

**SQL Query:**

```sql
USE {{.database_name}};
SELECT *
FROM {{.batch_table}}
{{if .filter_condition}}WHERE {{.filter_condition}}{{end}}
{{if .order_column}}ORDER BY {{.order_column}} DESC{{end}}
LIMIT {{.limit}};
```

**Example Usage:**

**For `task_status` table:**

```json
{
  "database_name": "your_database_name",
  "batch_table": "task_status",
  "filter_condition": "DATE(created_at) = CURDATE()",
  "order_column": "created_at",
  "limit": 100
}
```

**Check failed tasks:**

```json
{
  "database_name": "your_database_name",
  "batch_table": "task_status",
  "filter_condition": "status = 'FAILED' AND created_at >= CURDATE() - INTERVAL 7 DAY",
  "order_column": "created_at",
  "limit": 50
}
```

**Example Prompts:**

- "Check the results of last night's batch jobs"
- "Show me failed batch records for the premium calculation batch"
- "Did the membership sync batch complete successfully today?"
- "List all batch jobs that are still running or delayed"

---

## 🎯 Toolsets

Toolsets group related tools together for easier management.

```yaml
toolsets:
  mysql_custom_tools:
    - db_connection_check
    - db_run_query
    - db_check_response_time
    - db_check_deadlock
    - db_check_file_size
    - db_check_if_data
    - db_check_batch_data

  default:
    - db_connection_check
    - db_run_query
    - db_check_response_time
    - db_check_deadlock
    - db_check_file_size
    - db_check_if_data
    - db_check_batch_data
```

**Toolset Features:**

- Group related tools
- Enable/disable groups at once
- API endpoint: `/api/toolset/{toolset_name}`

---

## 📊 Database Schema (your_database_name)

### Tables Overview

#### 1. access_log

**Purpose:** Track user access to system

| Column      | Type         | Description      |
| ----------- | ------------ | ---------------- |
| log_id      | bigint       | Primary key      |
| user_id     | varchar(50)  | User identifier  |
| url         | varchar(255) | Accessed URL     |
| menu_name   | varchar(100) | Menu accessed    |
| access_time | datetime     | Access timestamp |

**Sample Query:**

```sql
SELECT * FROM access_log
WHERE access_time >= DATE_SUB(NOW(), INTERVAL 1 DAY)
ORDER BY access_time DESC;
```

---

#### 2. users

**Purpose:** User management

| Column     | Type         | Description     |
| ---------- | ------------ | --------------- |
| user_id    | varchar(50)  | Primary key     |
| username   | varchar(50)  | Login username  |
| password   | varchar(100) | Hashed password |
| name       | varchar(100) | Full name       |
| role       | varchar(20)  | User role       |
| department | varchar(100) | Department      |

**Sample Query:**

```sql
SELECT role, COUNT(*) as count FROM users GROUP BY role;
```

---

#### 3. golf_reservation

**Purpose:** Golf course reservations

| Column           | Type         | Description            |
| ---------------- | ------------ | ---------------------- |
| id               | bigint       | Primary key            |
| requester_id     | varchar(50)  | User ID                |
| golf_course      | varchar(200) | Golf course name       |
| reservation_date | date         | Reservation date       |
| participants     | int          | Number of participants |
| status           | varchar(20)  | Reservation status     |
| created_at       | datetime     | Creation timestamp     |

**Sample Query:**

```sql
SELECT * FROM golf_reservation
WHERE status = 'PENDING'
ORDER BY reservation_date DESC;
```

---

#### 4. idcard_request

**Purpose:** ID card requests

| Column         | Type         | Description        |
| -------------- | ------------ | ------------------ |
| id             | bigint       | Primary key        |
| requester_id   | varchar(50)  | Requester user ID  |
| requester_name | varchar(100) | Requester name     |
| employee_id    | varchar(50)  | Employee ID        |
| department     | varchar(100) | Department         |
| phone          | varchar(20)  | Phone number       |
| request_type   | varchar(50)  | Request type       |
| reason         | text         | Request reason     |
| status         | varchar(20)  | Request status     |
| approver_id    | varchar(50)  | Approver user ID   |
| approved_at    | datetime     | Approval timestamp |
| created_at     | datetime     | Request timestamp  |

**Sample Query:**

```sql
SELECT status, COUNT(*) as count
FROM idcard_request
GROUP BY status;
```

---

#### 5. task_status

**Purpose:** Task tracking

| Column       | Type         | Description        |
| ------------ | ------------ | ------------------ |
| task_id      | bigint       | Primary key        |
| requester_id | varchar(50)  | User ID            |
| task_type    | varchar(50)  | Type of task       |
| title        | varchar(200) | Task title         |
| status       | varchar(20)  | Task status        |
| created_at   | datetime     | Creation timestamp |

**Sample Query:**

```sql
SELECT task_type, status, COUNT(*) as count
FROM task_status
GROUP BY task_type, status;
```

---

## 🔌 API Endpoints

### Tool Management

```bash
# List all tools
GET http://localhost:5000/api/tool/

# Get tool details
GET http://localhost:5000/api/tool/{tool_name}

# Execute tool
POST http://localhost:5000/api/tool/{tool_name}/invoke
Content-Type: application/json

{
  "parameters": {
    "param1": "value1",
    "param2": "value2"
  }
}
```

### Toolset Management

```bash
# List all toolsets
GET http://localhost:5000/api/toolset/

# Get toolset details
GET http://localhost:5000/api/toolset/{toolset_name}
```

---

## 🎨 UI Features

Web UI available at `http://localhost:5000/ui`

### Features:

- **Tool Explorer:** Browse available tools
- **Interactive Parameters:** Fill in parameters with UI forms
- **Run Tools:** Execute tools and view results
- **Pretty JSON:** Formatted JSON output
- **History:** View execution history
- **Dark/Light Mode:** Toggle theme

### Screenshots:

```
┌─────────────────────────────────────┐
│  GenAI Toolbox                      │
├─────────────────────────────────────┤
│  Tools                              │
│  ├─ db_connection_check             │
│  ├─ db_run_query                    │
│  ├─ db_check_response_time          │
│  ├─ db_check_deadlock               │
│  ├─ db_check_file_size              │
│  ├─ db_check_if_data                │
│  └─ db_check_batch_data             │
└─────────────────────────────────────┘
```

---

## 🧪 Testing

### Test Connection

```sql
-- Via db_run_query tool
SELECT 1;
```

### Test Each Tool

```bash
# 1. Check file size
curl -X POST http://localhost:5000/api/tool/db_check_file_size/invoke \
  -H "Content-Type: application/json" \
  -d '{"parameters": {}}'

# 2. Check deadlocks
curl -X POST http://localhost:5000/api/tool/db_check_deadlock/invoke \
  -H "Content-Type: application/json" \
  -d '{"parameters": {}}'

# 3. Run custom query
curl -X POST http://localhost:5000/api/tool/db_run_query/invoke \
  -H "Content-Type: application/json" \
  -d '{"parameters": {"query": "SHOW TABLES;"}}'
```

---

## 🔒 Security Considerations

### 1. Credentials Management

- ✅ Use environment variables for passwords
- ✅ Never commit `.env` file to git
- ✅ Use secrets management in production
- ❌ Don't hardcode passwords in `tools.yaml`

### 2. SQL Injection Prevention

- Tools use parameterized queries
- Template parameters are sanitized
- Validate user inputs when possible

### 3. Access Control

- Deploy behind authentication proxy if needed
- Limit network access with firewall
- Use HTTPS for production

### 4. Database Permissions

```sql
-- Create read-only user for tools
CREATE USER 'toolbox_readonly'@'localhost' IDENTIFIED BY 'password';
GRANT SELECT ON your_database_name.* TO 'toolbox_readonly'@'localhost';
FLUSH PRIVILEGES;
```

---

## 📈 Monitoring & Logs

### Log Levels

```bash
# Debug (most verbose)
--log-level DEBUG

# Info (default)
--log-level INFO

# Warning
--log-level WARN

# Error only
--log-level ERROR
```

### Log Format

```bash
# Standard (human-readable)
--logging-format standard

# JSON (for log aggregators)
--logging-format JSON
```

### Example Logs

```
2025-12-09T10:00:00Z INFO "Initialized 1 sources: mysql-source"
2025-12-09T10:00:00Z INFO "Initialized 7 tools: db_connection_check, db_run_query, ..."
2025-12-09T10:00:01Z INFO "Server ready to serve!"
2025-12-09T10:00:01Z INFO "Toolbox UI is up and running at: http://127.0.0.1:5000/ui"
```

---

## 🚀 Best Practices

### 1. Tool Design

- Keep tool descriptions clear and concise
- Provide example prompts in descriptions
- Set sensible defaults for parameters
- Document required vs optional parameters

### 2. Performance

- Set appropriate `queryTimeout` values
- Use `LIMIT` in queries to prevent large results
- Index database columns used in WHERE clauses
- Monitor slow queries regularly

### 3. Maintainability

- Use descriptive tool names
- Group related tools in toolsets
- Comment complex SQL queries
- Version control `tools.yaml` file

### 4. Error Handling

- Test tools with edge cases
- Validate parameters before execution
- Provide helpful error messages
- Log errors for debugging

---

## 🔧 Advanced Configuration

### Multiple Sources

```yaml
sources:
  mysql-prod:
    kind: mysql
    host: prod-db.example.com
    database: production

  mysql-dev:
    kind: mysql
    host: localhost
    database: development

tools:
  query_prod:
    kind: mysql-execute-sql
    source: mysql-prod

  query_dev:
    kind: mysql-execute-sql
    source: mysql-dev
```

### Custom Query Params

```yaml
sources:
  mysql-source:
    kind: mysql
    host: localhost
    database: mydb
    queryParams:
      tls: preferred
      charset: utf8mb4
      parseTime: true
```

---

## 📚 References

- [MCP Protocol Specification](https://modelcontextprotocol.io/)
- [MySQL Documentation](https://dev.mysql.com/doc/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Deployment Guide](DEPLOYMENT.md)

---

**Version:** 1.0.0  
**Last Updated:** December 9, 2025  
**Maintainer:** GenAI Toolbox Team

# Troubleshooting Guide

## Common Issues and Solutions

### SQL Syntax Error 1064 in Tools

**Symptoms:**

```
Error: HTTP error 400: {"status":"Bad Request","error":"error while invoking tool: unable to execute query:
Error 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL
server version for the right syntax to use near 'SELECT * \nFROM table_name' at line 2"}
```

**Cause:**
The SQL query contains multiple statements separated by semicolons (e.g., `USE database_name; SELECT ...`). MySQL's prepared statement API does not support multi-statement queries.

**Solution:**
Remove the `USE database_name;` statement from your SQL queries. The database connection is already configured in the `sources` section of `tools.yaml`, so you don't need to explicitly switch databases in queries.

**Example:**

❌ **Wrong:**

```yaml
statement: |
  USE {{.database_name}};
  SELECT * FROM my_table;
```

✅ **Correct:**

```yaml
statement: |
  SELECT * FROM my_table;
```

The database specified in `sources.mysql-source.database` is automatically used for all queries.

---

### Container Exits Immediately (Exit Code 1)

**Symptoms:** Container shows "Exited (1)" with no visible logs

**Solution:** Check command flags:

- ✅ Use `--tools-file` (not `--tools`)
- ✅ Use `--ui` (not `--expose-ui`)

**Verify available flags:**

```bash
docker run --rm genai-toolbox:latest --help
```

---

### Required Parameters Not Recognized

**Symptoms:**

```
{"status":"Bad Request","error":"provided parameters were invalid: parameter \"table_name\" is required"}
```

**Cause:** The tool requires specific parameters but they were not provided.

**Solution:**

- In the UI, make sure to fill in all required fields (fields without "(optional)" label)
- When using the API, include all required parameters in the `arguments` object
- Check the tool definition to see which parameters are required: `GET /api/tool/{tool_name}`

**Example API Call:**

```bash
curl -X POST "http://localhost:5000/api/tool/db_check_if_data/invoke" \
  -H "Content-Type: application/json" \
  -d '{
    "arguments": {
      "interface_table": "my_interface_table",
      "limit": 100
    }
  }'
```

---

### Environment Variables Not Loading

**Symptoms:** Container can't connect to database, shows authentication errors

**Solution:**

1. Verify `.env` file exists and contains all required variables:

   ```bash
   MYSQL_USER=root
   MYSQL_PASSWORD=your_password
   MYSQL_DATABASE=your_database
   ```

2. Ensure `--env-file .env` is in the docker run command

3. Alternative: Pass variables directly:

   ```bash
   docker run -d \
     -e MYSQL_USER=root \
     -e MYSQL_PASSWORD=pass \
     -e MYSQL_DATABASE=mydb \
     ...
   ```

4. Verify variables are loaded in container:
   ```bash
   docker exec genai-toolbox env | grep MYSQL
   ```

---

### Network Connectivity to localhost MySQL

**Symptoms:** Can't connect to MySQL running on localhost from container

**Solution:**
Use `--network host` mode to access localhost services from container:

```bash
docker run -d \
  --name genai-toolbox \
  --network host \
  ...
```

**Verify MySQL is accessible:**

```bash
# From host
mysql -u root -p -e "SELECT 1"

# Check MySQL port
netstat -tlnp | grep 3306
```

---

### Slow Query Log is Empty

**Symptoms:** `db_check_response_time` tool returns no results

**Cause:** MySQL slow query log is not enabled

**Solution:** Enable slow query log in MySQL:

```sql
-- Check if slow query log is enabled
SHOW VARIABLES LIKE 'slow_query%';

-- Enable slow query log (requires restart or SET GLOBAL)
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL log_output = 'TABLE';
```

Add to MySQL config file (`/etc/mysql/my.cnf`):

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_output = TABLE
```

---

### UI Not Accessible

**Symptoms:** Can't access http://localhost:5000/ui

**Checks:**

1. Verify container is running:

   ```bash
   docker ps | grep genai-toolbox
   ```

2. Check logs for startup errors:

   ```bash
   docker logs genai-toolbox
   ```

3. Verify `--ui` flag is present in container command:

   ```bash
   docker inspect genai-toolbox | grep -A 10 Args
   ```

4. Test API endpoint:

   ```bash
   curl http://localhost:5000/api/health
   ```

5. Check port binding:
   ```bash
   netstat -tlnp | grep 5000
   ```

---

## Getting Help

If you encounter issues not covered here:

1. **Check logs:**

   ```bash
   docker logs genai-toolbox -f
   ```

2. **Run with debug logging:**

   ```bash
   docker run -it --rm \
     --name genai-toolbox \
     --network host \
     -v $(pwd)/tools.yaml:/app/tools.yaml:ro \
     --env-file .env \
     genai-toolbox:latest \
     --tools-file /app/tools.yaml \
     --ui \
     --log-level DEBUG
   ```

3. **Validate tools.yaml syntax:**

   ```bash
   # Check YAML syntax
   docker run --rm -v $(pwd)/tools.yaml:/tools.yaml:ro \
     mikefarah/yq eval '.' /tools.yaml
   ```

4. **Test database connection:**
   ```bash
   # From container
   docker exec -it genai-toolbox sh -c "echo 'SELECT 1' | mysql -h localhost -u \$MYSQL_USER -p\$MYSQL_PASSWORD \$MYSQL_DATABASE"
   ```

For more information, see:

- [SETUP.md](SETUP.md) - Quick setup guide
- [DEPLOYMENT.md](DEPLOYMENT.md) - Detailed deployment instructions
- [MCP-SERVER.md](MCP-SERVER.md) - Tool API reference

# Setup Guide

## 📋 Initial Setup

### 1. Configure Environment Variables

Copy the example file and update with your credentials:

```bash
cp .env.example .env
```

Edit `.env` and set your MySQL credentials:

```bash
MYSQL_USER=your_mysql_username
MYSQL_PASSWORD=your_mysql_password
MYSQL_DATABASE=your_database_name
```

### 2. Configure Tools

Copy the example tools configuration:

```bash
cp tools.yaml.example tools.yaml
```

Edit `tools.yaml` and update the default values in `templateParameters` sections:

- Change `default: "your_database_name"` to match your actual database name from `.env`

### 3. Build Docker Image

```bash
docker build -t genai-toolbox:latest .
```

### 4. Run Container

**With UI (Recommended):**

```bash
docker run -d \
  --name genai-toolbox \
  --network host \
  -v $(pwd)/tools.yaml:/app/tools.yaml:ro \
  --env-file .env \
  genai-toolbox:latest \
  --tools-file /app/tools.yaml \
  --ui
```

**Without UI:**

```bash
docker run -d \
  --name genai-toolbox \
  --network host \
  -v $(pwd)/tools.yaml:/app/tools.yaml:ro \
  --env-file .env \
  genai-toolbox:latest \
  --tools-file /app/tools.yaml
```

### 5. Access UI

Open your browser and navigate to:

```
http://localhost:5000/ui
```

## 🔒 Security Notes

**Important:** The following files contain sensitive information and are excluded from git:

- `.env` - Contains database credentials
- `tools.yaml` - Contains your actual database configuration

**Never commit these files to version control!**

Use the `.example` versions as templates for new deployments.

## 📝 Available Tools

1. **db_connection_check** - Verify database connectivity
2. **db_run_query** - Execute SQL queries
3. **db_check_response_time** - Identify slow queries
4. **db_check_deadlock** - Detect deadlocks and blocking sessions
5. **db_check_file_size** - Check database storage usage
6. **db_check_if_data** - Verify interface data
7. **db_check_batch_data** - Check batch processing results

For detailed documentation, see [MCP-SERVER.md](MCP-SERVER.md)

## 🔧 Troubleshooting

### Container exits immediately

**Symptoms:** Container shows "Exited (1)" status with no logs

**Solution:** Verify you're using the correct command flags:

- ✅ Use `--tools-file` (not `--tools`)
- ✅ Use `--ui` (not `--expose-ui`)

**Check available flags:**

```bash
docker run --rm genai-toolbox:latest --help
```

### Container is running but no UI

**Symptoms:** Container is running but UI is not accessible

**Checks:**

1. Verify container is running: `docker ps`
2. Check logs: `docker logs genai-toolbox`
3. Verify UI flag is present in command: `--ui`
4. Test connection: `curl http://localhost:5000/ui`

### Environment variables not loading

**Symptoms:** Container can't connect to database

**Solution:**

- Ensure `.env` file exists and contains all three required variables: `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_DATABASE`
- Verify `--env-file .env` is in the docker run command
- Alternative: Pass variables directly with `-e` flags

### Network connectivity issues

**Symptoms:** Can't connect to localhost MySQL

**Solution:**

- Use `--network host` flag to access localhost from container
- Verify MySQL is running: `mysql -u root -p -e "SELECT 1"`
- Check MySQL is listening on expected port: `netstat -tlnp | grep 3306`

## 🛠️ Container Management

**Stop container:**

```bash
docker stop genai-toolbox
```

**Start container:**

```bash
docker start genai-toolbox
```

**Remove container:**

```bash
docker rm -f genai-toolbox
```

**View logs:**

```bash
docker logs genai-toolbox
docker logs -f genai-toolbox  # Follow mode
```

**Restart after configuration changes:**

```bash
docker restart genai-toolbox
```

## 📚 More Information

- [Full Deployment Guide](DEPLOYMENT.md) - Detailed deployment options and configuration
- [MCP Server Documentation](MCP-SERVER.md) - Complete tool API reference
- [Developer Guide](DEVELOPER.md) - Development and contribution guidelines

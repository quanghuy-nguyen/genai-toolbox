# 🚀 Deployment Guide - GenAI Toolbox MCP Server

Detailed guide on deploying GenAI Toolbox MCP Server with Docker.

## 📋 Requirements

- Docker installed
- MySQL Server running (can be localhost or remote)
- Database already created

## 🔧 Preparation

### 1. Clone repository or copy source code

```bash
cd /path/to/genai-toolbox
```

### 2. Create `.env` configuration file

```bash
cp .env.example .env
nano .env
```

Content of `.env` file:

```env
# MySQL Connection Credentials
MYSQL_USER=root
MYSQL_PASSWORD=your_password_here
MYSQL_DATABASE=your_database_name
```

**Note:** All three variables are required.

### 3. Create or edit `tools.yaml` file

This file defines tools and database connections. See details in [MCP-SERVER.md](MCP-SERVER.md).

```yaml
sources:
  mysql-source:
    kind: mysql
    host: localhost
    port: 3306
    database: ${MYSQL_DATABASE}
    user: ${MYSQL_USER}
    password: ${MYSQL_PASSWORD}
    queryTimeout: 30s

tools:
  # Define your tools here
  db_run_query:
    kind: mysql-execute-sql
    source: mysql-source
    description: "Execute SQL queries"
```

**Important:**

- Environment variables are loaded from `.env` file
- No default/fallback values - all env vars must be provided
- Use `${VAR}` syntax to reference environment variables

## 🐳 Docker Build

### Build Docker Image

```bash
docker build -t genai-toolbox:latest .
```

**Notes:**

- Build process uses multi-stage with Golang and Zig compiler
- Final image based on `distroless/cc-debian12` for optimized size
- Build time approximately 2-5 minutes depending on machine

### Check built image

```bash
docker images | grep genai-toolbox
```

## 🚀 Docker Run

### Option 1: Run with UI (Recommended)

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

**Access UI:** http://localhost:5000/ui

**Alternative:** Pass environment variables directly:

```bash
docker run -d \
  --name genai-toolbox \
  --network host \
  -v $(pwd)/tools.yaml:/app/tools.yaml:ro \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=your_password \
  -e MYSQL_DATABASE=your_database \
  genai-toolbox:latest \
  --tools-file /app/tools.yaml \
  --ui
```

### Option 2: Run HTTP API Server (without UI)

```bash
docker run -d \
  --name genai-toolbox \
  --network host \
  -v $(pwd)/tools.yaml:/app/tools.yaml:ro \
  --env-file .env \
  genai-toolbox:latest \
  --tools-file /app/tools.yaml
```

**API Endpoint:** http://localhost:5000

### Option 3: Run in foreground with debug logs

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

Press `Ctrl+C` to stop.

## 🎛️ Docker Run Options

### Common flags

| Flag               | Description                            | Default   |
| ------------------ | -------------------------------------- | --------- |
| `--ui`             | Enable Toolbox UI web interface        | false     |
| `--tools-file`     | Path to tools configuration file       | -         |
| `--port`           | Port for server to listen on           | 5000      |
| `--address`        | Address for server to bind to          | 127.0.0.1 |
| `--log-level`      | Log level (DEBUG, INFO, WARN, ERROR)   | INFO      |
| `--disable-reload` | Disable automatic tools file reloading | false     |

### Volume mounts

```bash
# Mount tools.yaml (read-only recommended)
-v $(pwd)/tools.yaml:/app/tools.yaml:ro

# Mount tools directory (if using multiple files)
-v $(pwd)/tools:/app/tools:ro

# Mount .env file
-v $(pwd)/.env:/app/.env:ro
```

### Network modes

```bash
# Host network (for localhost MySQL connection)
--network host

# Bridge network (default)
--network bridge

# Custom network
--network my-custom-network
```

## 📊 Container Management

### View logs

```bash
# View logs
docker logs genai-toolbox

# View logs in real-time
docker logs -f genai-toolbox

# View last 50 log lines
docker logs --tail 50 genai-toolbox
```

### Check status

```bash
# View running container
docker ps | grep genai-toolbox

# View detailed information
docker inspect genai-toolbox

# View resource usage
docker stats genai-toolbox
```

### Manage container

```bash
# Stop container
docker stop genai-toolbox

# Start container
docker start genai-toolbox

# Restart container
docker restart genai-toolbox

# Remove container
docker rm genai-toolbox

# Remove container (force)
docker rm -f genai-toolbox
```

### Update configuration

When editing `tools.yaml` file, container will automatically reload (if not using `--disable-reload`):

```bash
# Edit file
nano tools.yaml

# Container will automatically detect and reload
# Or restart manually:
docker restart genai-toolbox
```

## 🐳 Docker Compose (Optional)

### File `docker-compose.yml`

```yaml
version: "3.8"

services:
  genai-toolbox:
    build: .
    image: genai-toolbox:latest
    container_name: genai-toolbox
    network_mode: host
    volumes:
      - ./tools.yaml:/app/tools.yaml:ro
    environment:
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    command: ["--tools-file", "/app/tools.yaml", "--ui"]
    stdin_open: true
    tty: true
    restart: unless-stopped
```

### Run with Docker Compose

```bash
# Build and start
docker-compose up -d

# View logs
docker-compose logs -f

# Stop
docker-compose down

# Rebuild and start
docker-compose up -d --build
```

## 🔍 Troubleshooting

### MySQL connection error

```bash
# Check if MySQL is running
mysql -h localhost -u root -p

# Check network mode
docker inspect genai-toolbox | grep NetworkMode

# If MySQL is on localhost, must use --network host
docker run --network host ...
```

### Unable to parse tool file error

```bash
# Check YAML syntax
docker run --rm -v $(pwd)/tools.yaml:/tmp/tools.yaml:ro \
  python:3-alpine python -c "import yaml; yaml.safe_load(open('/tmp/tools.yaml'))"

# View detailed logs
docker logs genai-toolbox --tail 100
```

### Container exits immediately

```bash
# View logs to identify error
docker logs genai-toolbox

# Run in interactive mode for debugging
docker run -it --rm \
  --network host \
  -v $(pwd)/tools.yaml:/app/tools.yaml:ro \
  --env-file .env \
  genai-toolbox:latest \
  --tools-file /app/tools.yaml --ui --log-level DEBUG
```

### Port already in use

```bash
# Check port 5000
lsof -i :5000

# Use different port
docker run ... genai-toolbox:latest \
  --tools-file /app/tools.yaml --ui --port 5001
```

## 📝 Automated Script (Optional)

### Script `run.sh`

```bash
#!/bin/bash

# Stop and remove old container
docker rm -f genai-toolbox 2>/dev/null

# Build new image
echo "🔨 Building Docker image..."
docker build -t genai-toolbox:latest .

# Run container
echo "🚀 Starting container..."
docker run -d \
  --name genai-toolbox \
  --network host \
  -v $(pwd)/tools.yaml:/app/tools.yaml:ro \
  --env-file .env \
  genai-toolbox:latest \
  --tools-file /app/tools.yaml --ui

# Wait for container to start
sleep 3

# Show logs
echo "📊 Container logs:"
docker logs genai-toolbox

echo ""
echo "✅ GenAI Toolbox is running!"
echo "🌐 UI: http://localhost:5000/ui"
echo "📊 Logs: docker logs -f genai-toolbox"
```

Set permissions and run:

```bash
chmod +x run.sh
./run.sh
```

## 🔐 Security Best Practices

### 1. Don't hardcode passwords

Always use environment variables or secrets:

```bash
# ✅ Good
-e MYSQL_PASSWORD="${MYSQL_PASSWORD}"

# ❌ Bad
-e MYSQL_PASSWORD="123456"
```

### 2. Mount read-only

```bash
# Mount files read-only
-v $(pwd)/tools.yaml:/app/tools.yaml:ro
```

### 3. Limit resources

```bash
docker run -d \
  --memory="512m" \
  --cpus="1.0" \
  ...
```

### 4. Control port exposure

When exposing ports, consider using specific addresses:

```bash
# Bind to specific address
-p 127.0.0.1:5000:5000

# Or use host network for localhost access
--network host
```

## 📚 References

- [MCP Server Configuration](MCP-SERVER.md) - Details about tools and schema
- [Docker Official Docs](https://docs.docker.com/)
- [MySQL Connection Guide](https://dev.mysql.com/doc/)

## 💡 Tips

1. Always test locally before deploying
2. Backup `tools.yaml` and `.env` files
3. Monitor logs regularly: `docker logs -f genai-toolbox`
4. Update image periodically when new versions are available
5. Use `--disable-reload` if you don't want auto-reload

---

**Happy Deploying! 🎉**

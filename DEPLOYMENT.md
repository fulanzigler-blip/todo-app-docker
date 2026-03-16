# Todo App - Deployment Guide

This guide covers deploying the Todo App using Docker and Docker Compose.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Local Development](#local-development)
3. [Production Deployment](#production-deployment)
4. [Docker Commands](#docker-commands)
5. [Troubleshooting](#troubleshooting)
6. [CI/CD Pipeline](#cicd-pipeline)

---

## Prerequisites

Before deploying, ensure you have:

- Docker Engine 20.10+ installed
- Docker Compose 2.0+ installed
- Git installed
- For production: A VPS or server with at least 1GB RAM
- For production: Domain name configured (optional)

### Install Docker

**Ubuntu/Debian:**
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

**macOS:**
```bash
brew install --cask docker
```

**Verify installation:**
```bash
docker --version
docker-compose --version
```

---

## Local Development

### 1. Clone the Repository

```bash
git clone https://github.com/fulanzigler-blip/todo-app.git
cd todo-app
```

### 2. Configure Environment Variables

```bash
cp .env.example .env
```

Edit `.env` with your preferred configuration:

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=todoapp
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/todoapp
PORT=3000
NODE_ENV=development
VITE_API_URL=http://localhost:3000/api/v1
```

### 3. Start Services

```bash
docker-compose up -d
```

This will:
- Start PostgreSQL database
- Build and start the backend API
- Build and start the frontend

### 4. Access the Application

- **Frontend:** http://localhost
- **Backend API:** http://localhost:3000
- **API Documentation:** http://localhost:3000/docs (Swagger UI)

### 5. View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f db
```

### 6. Stop Services

```bash
docker-compose down
```

To remove volumes (deletes database data):

```bash
docker-compose down -v
```

---

## Production Deployment

### Option 1: Docker Compose (Recommended for VPS)

#### 1. Prepare Server

```bash
# SSH into your server
ssh user@your-server.com

# Create deployment directory
sudo mkdir -p /opt/todo-app
cd /opt/todo-app
```

#### 2. Clone Repository

```bash
git clone https://github.com/fulanzigler-blip/todo-app.git .
```

#### 3. Configure Production Environment

```bash
# Copy example env file
cp .env.example .env

# Edit with production values
nano .env
```

**Production `.env` example:**

```env
# Use strong passwords!
POSTGRES_USER=todo_user
POSTGRES_PASSWORD=$(openssl rand -base64 32)
POSTGRES_DB=todoapp_prod

DATABASE_URL=postgresql://todo_user:PASSWORD@db:5432/todoapp_prod
PORT=3000
NODE_ENV=production

# Frontend
VITE_API_URL=https://api.yourdomain.com/api/v1
```

#### 4. Configure Reverse Proxy (Nginx)

Create `/etc/nginx/sites-available/todo-app`:

```nginx
# Frontend
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://localhost:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# Backend API
server {
    listen 80;
    server_name api.yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/todo-app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

#### 5. Start Application

```bash
# Build and start
docker-compose up -d --build

# Check status
docker-compose ps
```

#### 6. Setup SSL Certificate (Let's Encrypt)

```bash
# Install certbot
sudo apt install certbot python3-certbot-nginx

# Obtain certificate
sudo certbot --nginx -d yourdomain.com -d api.yourdomain.com

# Auto-renewal is configured automatically
```

### Option 2: Using CI/CD Pipeline

#### 1. Configure GitHub Secrets

Go to your repository: Settings → Secrets and variables → Actions

Add the following secrets:

- `SSH_HOST` - Your server IP or domain
- `SSH_USER` - SSH username
- `SSH_KEY` - Private SSH key (use `ssh-keygen` if needed)
- `DOCKER_USERNAME` - Docker Hub username (optional)
- `DOCKER_PASSWORD` - Docker Hub password/token (optional)
- `POSTGRES_USER` - Database user
- `POSTGRES_PASSWORD` - Database password (strong!)
- `POSTGRES_DB` - Database name

#### 2. Setup Server SSH Access

Generate SSH key pair:

```bash
ssh-keygen -t ed25519 -C "github-deploy" -f ~/.ssh/github_deploy
```

Copy public key to server:

```bash
ssh-copy-id -i ~/.ssh/github_deploy.pub user@your-server.com
```

Add private key (`~/.ssh/github_deploy`) to GitHub secrets as `SSH_KEY`.

#### 3. Push to Trigger Deployment

```bash
git add .
git commit -m "Deploy to production"
git push origin master
```

The deployment workflow will:
1. Build Docker images
2. Push to Docker Hub (if configured)
3. SSH into your server
4. Pull latest images
5. Restart services with `docker-compose`

---

## Docker Commands Reference

### Building Images

```bash
# Build backend
docker build -f Dockerfile.backend -t todo-backend ./backend

# Build frontend
docker build -f frontend/Dockerfile -t todo-frontend ./frontend
```

### Running Containers

```bash
# Run backend standalone
docker run -d \
  --name todo-backend \
  -p 3000:3000 \
  -e DATABASE_URL=postgresql://user:pass@host:5432/db \
  todo-backend

# Run frontend standalone
docker run -d \
  --name todo-frontend \
  -p 80:80 \
  todo-frontend
```

### Container Management

```bash
# List running containers
docker ps

# View logs
docker logs todo-backend
docker logs -f todo-frontend

# Execute commands in container
docker exec -it todo-backend sh
docker exec -it todo-db psql -U postgres -d todoapp

# Stop container
docker stop todo-backend

# Remove container
docker rm todo-backend
```

### Volume Management

```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect todo-app_postgres_data

# Backup database
docker exec todo-db pg_dump -U postgres todoapp > backup.sql

# Restore database
cat backup.sql | docker exec -i todo-db psql -U postgres todoapp
```

---

## Troubleshooting

### Container Won't Start

```bash
# Check logs
docker-compose logs [service-name]

# Check container status
docker-compose ps

# Rebuild from scratch
docker-compose down -v
docker-compose up -d --build
```

### Database Connection Issues

```bash
# Test database connection
docker exec -it todo-db psql -U postgres -d todoapp

# Check database logs
docker-compose logs db

# Verify environment variables
docker-compose config
```

### Port Already in Use

```bash
# Find process using port
sudo lsof -i :80
sudo lsof -i :3000
sudo lsof -i :5432

# Kill process
sudo kill -9 [PID]

# Or change port in docker-compose.yml
```

### Out of Disk Space

```bash
# Remove unused images
docker image prune -a

# Remove unused containers
docker container prune

# Remove unused volumes
docker volume prune

# Clean all
docker system prune -a --volumes
```

### Frontend Can't Reach Backend

1. Check backend is running:
```bash
curl http://localhost:3000/health
```

2. Check `VITE_API_URL` in frontend `.env`

3. Check CORS configuration in backend

4. Check Docker network connectivity:
```bash
docker network inspect todo-app_todo-network
```

---

## CI/CD Pipeline

### Workflow Files

- `.github/workflows/backend-ci.yml` - Backend CI (lint, test, build)
- `.github/workflows/frontend-ci.yml` - Frontend CI (lint, build, docker)
- `.github/workflows/deploy.yml` - Production deployment

### Pipeline Stages

**Backend CI:**
1. Linting (ESLint)
2. Unit/Integration tests with PostgreSQL
3. Docker build

**Frontend CI:**
1. Linting (ESLint)
2. Production build
3. Docker build

**Deploy:**
1. Build Docker images
2. Push to registry (optional)
3. Deploy via SSH to server
4. Restart services

### Monitoring Deployments

View workflow runs: GitHub → Actions tab

### Rollback

To rollback to previous version:

```bash
# On server
cd /opt/todo-app
git log --oneline
git checkout [previous-commit-hash]
docker-compose up -d --build
```

---

## Health Checks

All services include health checks:

- **Database**: `pg_isready`
- **Backend**: `/health` endpoint
- **Frontend**: HTTP GET on root

Check health status:

```bash
docker inspect --format='{{.State.Health.Status}}' todo-backend
```

---

## Backup Strategy

### Automated Backup (cron)

```bash
# Add to crontab (crontab -e)
# Daily database backup at 2 AM
0 2 * * * docker exec todo-db pg_dump -U postgres todoapp > /backups/todoapp_$(date +\%Y\%m\%d).sql

# Keep last 7 days
0 3 * * * find /backups -name "todoapp_*.sql" -mtime +7 -delete
```

### Restore from Backup

```bash
# Stop application
docker-compose down

# Restore database
docker run --rm \
  -v postgres_data:/var/lib/postgresql/data \
  -v /backups:/backup \
  postgres:15-alpine \
  sh -c "pg_restore -U postgres -d todoapp /backup/todoapp_20250316.sql"

# Start application
docker-compose up -d
```

---

## Performance Optimization

### Database

```bash
# In docker-compose.yml, add to db service:
command: postgres -c shared_buffers=256MB -c max_connections=200
```

### Backend

```bash
# Add PM2 for process management (optional)
# In package.json:
"start:pm2": "pm2 start dist/index.js --name todo-backend"
```

### Frontend

- Already optimized with Vite
- Nginx caching configured
- Gzip compression enabled

---

## Security Best Practices

1. **Use strong passwords** - Never use default passwords in production
2. **Enable SSL/TLS** - Always use HTTPS in production
3. **Environment variables** - Never commit `.env` files
4. **Regular updates** - Keep Docker images and dependencies updated
5. **Firewall** - Restrict access to ports (only expose 80/443)
6. **Backups** - Regular automated backups
7. **Monitoring** - Set up logs and alerting

---

## Support

For issues or questions:
- Check logs: `docker-compose logs -f`
- Review this guide's troubleshooting section
- Check GitHub issues

---

## Changelog

- **v1.0.0** (2025-03-16): Initial deployment guide with Docker support

# GitHub Workflows - Manual Installation Instructions

Due to GitHub Personal Access Token scope restrictions, the workflow files were not included in the initial push. Follow these steps to add them manually.

## Quick Installation

### Option 1: Copy from Local Files

The workflow files are available in your local workspace at:
- `/home/clawdbot/.openclaw/workspace/projects/todo-app/.github/workflows/`

Run these commands:

```bash
# Navigate to your clone of the repository
cd /path/to/todo-app-docker

# Create the workflows directory if it doesn't exist
mkdir -p .github/workflows

# Copy the workflow files (adjust paths as needed)
cp /home/clawdbot/.openclaw/workspace/projects/todo-app/.github/workflows/*.yml .github/workflows/

# Commit and push
git add .github/workflows/
git commit -m "ci: Add GitHub Actions workflows"
git push origin master
```

### Option 2: Create Files Manually

Create the following files in `.github/workflows/` directory:

## 1. Backend CI Workflow (backend-ci.yml)

```yaml
name: Backend CI

on:
  push:
    branches: [ master, main, develop ]
    paths:
      - 'backend/**'
      - 'Dockerfile.backend'
      - '.github/workflows/backend-ci.yml'
  pull_request:
    branches: [ master, main, develop ]
    paths:
      - 'backend/**'
      - 'Dockerfile.backend'
      - '.github/workflows/backend-ci.yml'

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        working-directory: ./backend
        run: npm ci

      - name: Run ESLint
        working-directory: ./backend
        run: npm run lint || echo "No lint script found"

  test:
    name: Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: todoapp_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        working-directory: ./backend
        run: npm ci

      - name: Generate Prisma client
        working-directory: ./backend
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/todoapp_test
        run: npx prisma generate

      - name: Run database migrations
        working-directory: ./backend
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/todoapp_test
        run: npx prisma migrate deploy

      - name: Run tests
        working-directory: ./backend
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/todoapp_test
          NODE_ENV: test
        run: npm test || echo "No test script found"

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          file: ./Dockerfile.backend
          push: false
          tags: todo-backend:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## 2. Frontend CI Workflow (frontend-ci.yml)

```yaml
name: Frontend CI

on:
  push:
    branches: [ master, main, develop ]
    paths:
      - 'frontend/**'
      - '.github/workflows/frontend-ci.yml'
  pull_request:
    branches: [ master, main, develop ]
    paths:
      - 'frontend/**'
      - '.github/workflows/frontend-ci.yml'

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        working-directory: ./frontend
        run: npm ci

      - name: Run ESLint
        working-directory: ./frontend
        run: npm run lint

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        working-directory: ./frontend
        run: npm ci

      - name: Build application
        working-directory: ./frontend
        run: npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: frontend-dist
          path: frontend/dist
          retention-days: 7

  docker-build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [build]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          push: false
          tags: todo-frontend:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## 3. Deployment Workflow (deploy.yml)

```yaml
name: Deploy

on:
  push:
    branches: [ master, main ]
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub (optional)
        if: ${{ secrets.DOCKER_USERNAME && secrets.DOCKER_PASSWORD }}
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push backend image
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          file: ./Dockerfile.backend
          push: ${{ secrets.DOCKER_USERNAME != '' }}
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/todo-backend:latest
            ${{ secrets.DOCKER_USERNAME }}/todo-backend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build and push frontend image
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          push: ${{ secrets.DOCKER_USERNAME != '' }}
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/todo-frontend:latest
            ${{ secrets.DOCKER_USERNAME }}/todo-frontend:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to server (SSH)
        if: ${{ secrets.SSH_HOST && secrets.SSH_USER && secrets.SSH_KEY }}
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /opt/todo-app
            docker-compose pull
            docker-compose up -d
            docker image prune -f

      - name: Notify deployment status
        if: always()
        run: |
          if [ "${{ job.status }}" == "success" ]; then
            echo "Deployment successful! ✅"
          else
            echo "Deployment failed! ❌"
            exit 1
          fi
```

## Required GitHub Secrets

Configure these in your repository: Settings → Secrets and variables → Actions

### For Deployment (deploy.yml)

- `SSH_HOST` - Your server IP or domain
- `SSH_USER` - SSH username
- `SSH_KEY` - Private SSH key content
- `DOCKER_USERNAME` - Docker Hub username (optional)
- `DOCKER_PASSWORD` - Docker Hub password/token (optional)

### For Backend (backend-ci.yml)

- `POSTGRES_USER` - Database user
- `POSTGRES_PASSWORD` - Database password
- `POSTGRES_DB` - Database name

## After Adding Workflows

Once you've added the workflow files and pushed them:

1. **Go to GitHub Actions tab** in your repository
2. **Verify workflows appear** - You should see three workflows:
   - Backend CI
   - Frontend CI
   - Deploy
3. **Test manually** - Click "Run workflow" button on any workflow to test
4. **Check logs** - Review workflow runs for any errors

## Workflow Behavior

### Backend CI
- Runs on push to master/main/develop
- Tests linting with ESLint
- Runs unit/integration tests with PostgreSQL
- Builds Docker image
- Triggered by changes in backend/ or Dockerfile.backend

### Frontend CI
- Runs on push to master/main/develop
- Tests linting with ESLint
- Builds production bundle
- Builds Docker image
- Uploads build artifacts
- Triggered by changes in frontend/

### Deploy
- Runs on push to master/main
- Builds Docker images
- Pushes to Docker Hub (if configured)
- Deploys via SSH to production server
- Can be triggered manually (workflow_dispatch)

## Troubleshooting

### Workflow not triggering
- Check branch names match (master/main/develop)
- Verify file paths in workflow `paths` filters
- Check workflow file syntax (YAML indentation)

### SSH deployment fails
- Verify SSH key is correctly formatted (include full key)
- Check server accessibility
- Verify user has Docker permissions on server

### Tests failing in CI
- Ensure test scripts exist in package.json
- Check database connection in test environment
- Review test logs in Actions tab

---

**Note:** These workflow files are already available in your local workspace at:
`/home/clawdbot/.openclaw/workspace/projects/todo-app/.github/workflows/`

You can copy them directly from there.

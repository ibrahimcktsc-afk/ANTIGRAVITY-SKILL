# 6. Deployment Plan

## Deployment Strategy

**Approach**: Blue-Green deployment with canary releases for production
**Frequency**: Weekly releases (Wednesday 2 AM UTC)
**Rollback Time**: < 5 minutes
**Zero-downtime**: Target 99.99% uptime during deployment

## Environment Configuration

### Development Environment
```yaml
Name: dev
Database: PostgreSQL (shared)
Cache: Redis (shared)
Storage: S3 (test bucket)
Features: All features enabled
Data: Mock and test data
Backups: Daily
Retention: 7 days
```

### Staging Environment
```yaml
Name: staging
Database: PostgreSQL (production-like, subset of data)
Cache: Redis (production-like)
Storage: S3 (staging bucket)
Features: Feature flags for testing
Data: Anonymized production data
Backups: Daily
Retention: 30 days
```

### Production Environment
```yaml
Name: production
Database: PostgreSQL (HA cluster with replicas)
Cache: Redis (cluster mode, high availability)
Storage: S3 (encrypted, replicated)
Features: Only stable features
Data: Real customer data
Backups: Hourly
Retention: 90 days
CDN: CloudFlare/CloudFront
```

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Developer Push                           │
│              (git push origin feature-branch)                │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│         GitHub Actions - CI/CD Pipeline                      │
│  ├─ Run Tests                                                │
│  ├─ Build Docker Image                                       │
│  ├─ Push to Container Registry                               │
│  └─ Deploy to Dev                                            │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│    Create Pull Request & Code Review                         │
│    ├─ Automated checks pass                                  │
│    └─ Manual approval from 2 reviewers                       │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│         Merge to Main Branch                                 │
│    ├─ Deploy to Staging                                      │
│    ├─ Run Integration Tests                                  │
│    ├─ Run E2E Tests                                          │
│    └─ Performance Tests                                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│      Manual Approval for Production                          │
│    ├─ Release Manager reviews                                │
│    ├─ Check deployment window                                │
│    └─ Notify stakeholders                                    │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│    Blue-Green Deployment                                     │
│    ├─ Deploy to Green environment                            │
│    ├─ Run smoke tests                                        │
│    ├─ Shift traffic (5% canary)                              │
│    ├─ Monitor metrics (15 min)                               │
│    ├─ Shift remaining traffic (95%)                          │
│    └─ Keep Blue as rollback point                            │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│    Post-Deployment Verification                              │
│    ├─ Health checks passing                                  │
│    ├─ Error rate normal                                      │
│    ├─ Performance metrics OK                                 │
│    └─ Customer-facing features working                       │
└─────────────────────────────────────────────────────────────┘
```

## CI/CD Pipeline Configuration

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/test_db
          REDIS_URL: redis://localhost:6379
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Dev
        env:
          DEPLOY_KEY: ${{ secrets.DEV_DEPLOY_KEY }}
          DEPLOY_HOST: ${{ secrets.DEV_DEPLOY_HOST }}
        run: |
          mkdir -p ~/.ssh
          echo "$DEPLOY_KEY" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          ssh -i ~/.ssh/deploy_key -o StrictHostKeyChecking=no \
            $DEPLOY_HOST 'cd /app && docker pull ghcr.io/${{ github.repository }}:${{ github.sha }} && docker-compose up -d'

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Staging
        run: |
          kubectl set image deployment/api-staging \
            api=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --record \
            --kubeconfig=${{ secrets.KUBECONFIG }}

  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://api.medicalshop.com
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Slack notification - Deployment Started
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "🚀 Production Deployment Started",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment*: ${{ github.ref }}\n*Commit*: ${{ github.sha }}\n*Author*: ${{ github.actor }}"
                  }
                }
              ]
            }
      
      - name: Blue-Green Deploy to Production
        run: |
          # Get current active environment
          CURRENT_ENV=$(kubectl get svc api-prod -o jsonpath='{.spec.selector.env}')
          
          if [ "$CURRENT_ENV" = "blue" ]; then
            TARGET_ENV="green"
          else
            TARGET_ENV="blue"
          fi
          
          # Deploy to target environment
          kubectl set image deployment/api-$TARGET_ENV \
            api=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --record \
            --kubeconfig=${{ secrets.KUBECONFIG }}
          
          # Wait for rollout
          kubectl rollout status deployment/api-$TARGET_ENV \
            --kubeconfig=${{ secrets.KUBECONFIG }} \
            --timeout=5m
          
          # Run smoke tests
          ./scripts/smoke-tests.sh $TARGET_ENV
          
          # Canary: 5% traffic to new environment
          kubectl patch svc api-prod \
            -p '{"spec":{"selector":{"env":"'"$TARGET_ENV"'","canary":"true"}}}' \
            --kubeconfig=${{ secrets.KUBECONFIG }}
          
          sleep 900  # Wait 15 minutes for canary
          
          # Shift all traffic to new environment
          kubectl patch svc api-prod \
            -p '{"spec":{"selector":{"env":"'"$TARGET_ENV"'"}}}' \
            --kubeconfig=${{ secrets.KUBECONFIG }}
      
      - name: Slack notification - Deployment Success
        if: success()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "✅ Production Deployment Successful",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Status*: SUCCESS\n*Version*: ${{ github.sha }}\n*Deployed By*: ${{ github.actor }}"
                  }
                }
              ]
            }
      
      - name: Slack notification - Deployment Failed
        if: failure()
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "❌ Production Deployment Failed",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Status*: FAILED\n*Version*: ${{ github.sha }}\n*Deployed By*: ${{ github.actor }}"
                  }
                }
              ]
            }
```

## Docker Configuration

### Dockerfile

```dockerfile
# Multi-stage build
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build TypeScript
RUN npm run build

# Runtime stage
FROM node:18-alpine

WORKDIR /app

# Install security updates
RUN apk update && apk upgrade

# Copy from builder
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

USER nodejs

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

EXPOSE 3000

CMD ["node", "dist/server.js"]
```

### Docker Compose (Development)

```yaml
version: '3.9'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://user:password@postgres:5432/medicalshop
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis
    volumes:
      - .:/app
      - /app/node_modules
    command: npm run dev

  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: medicalshop
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

## Kubernetes Deployment

### Deployment Manifest

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-prod
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: api
      env: blue
  template:
    metadata:
      labels:
        app: api
        env: blue
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - api
              topologyKey: kubernetes.io/hostname
      
      containers:
      - name: api
        image: ghcr.io/ibrahimcktsc-afk/ANTIGRAVITY-SKILL:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        
        env:
        - name: NODE_ENV
          value: production
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: redis-url
        
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        securityContext:
          runAsNonRoot: true
          runAsUser: 1001
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
```

### Service Manifest

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-prod
  namespace: production
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - protocol: TCP
    port: 443
    targetPort: 3000
    name: https
  sessionAffinity: None
```

## Database Migration Strategy

### Pre-deployment Validation

```bash
#!/bin/bash
# scripts/validate-migrations.sh

# Check for migration conflicts
npm run migrate:validate

# Dry-run migration
npm run migrate:dry-run

# Generate rollback script
npm run migrate:generate-rollback

# Test against staging database
npm run migrate:test-staging

echo "✓ All migrations validated successfully"
```

### Zero-downtime Migration

```javascript
// Backward compatible migrations
// Step 1: Add new column (nullable)
// Step 2: Backfill data
// Step 3: Make column non-nullable
// Step 4: Remove old column (in next release)

// Example migration
const Sequelize = require('sequelize');

module.exports = {
  async up(queryInterface) {
    // Step 1: Add column
    await queryInterface.addColumn('products', 'new_field', {
      type: Sequelize.STRING,
      allowNull: true
    });

    // Step 2: Backfill
    await queryInterface.sequelize.query(
      'UPDATE products SET new_field = old_field'
    );

    // Step 3: Make non-nullable
    await queryInterface.changeColumn('products', 'new_field', {
      type: Sequelize.STRING,
      allowNull: false
    });
  },

  async down(queryInterface) {
    await queryInterface.removeColumn('products', 'new_field');
  }
};
```

## Rollback Procedures

### Automatic Rollback

```bash
#!/bin/bash
# scripts/rollback.sh

ENV=$1
PREVIOUS_VERSION=$(git describe --tags --abbrev=0 HEAD~1)

echo "Rolling back $ENV to $PREVIOUS_VERSION..."

if [ "$ENV" = "production" ]; then
  # Get current environment
  CURRENT_ENV=$(kubectl get svc api-prod -o jsonpath='{.spec.selector.env}')
  
  # Switch to previous environment
  if [ "$CURRENT_ENV" = "blue" ]; then
    kubectl patch svc api-prod -p '{"spec":{"selector":{"env":"green"}}}'
  else
    kubectl patch svc api-prod -p '{"spec":{"selector":{"env":"blue"}}}'
  fi
else
  # For non-production, restart with previous image
  kubectl rollout undo deployment/api-$ENV
fi

echo "✓ Rollback completed"
```

### Database Rollback

```bash
#!/bin/bash
# scripts/db-rollback.sh

# Restore from backup
BACKUP_FILE=$(ls -t backups/*.sql.gz | head -1)

echo "Restoring database from $BACKUP_FILE..."

gunzip -c "$BACKUP_FILE" | psql $DATABASE_URL

echo "✓ Database restored"
```

## Deployment Checklist

- [ ] Code reviewed and approved
- [ ] All tests passing (unit, integration, E2E)
- [ ] Database migrations validated
- [ ] Environment variables configured
- [ ] Secrets rotated (if needed)
- [ ] Docker image built and tested
- [ ] Staging deployment successful
- [ ] Smoke tests passing
- [ ] Performance tests within SLA
- [ ] Security scan passing
- [ ] Stakeholders notified
- [ ] Rollback plan documented
- [ ] On-call engineer available
- [ ] Post-deployment verification plan ready

## Monitoring Post-Deployment

```javascript
// Monitor critical metrics
const metrics = [
  { name: 'api_response_time_ms', threshold: 200 },
  { name: 'error_rate_percent', threshold: 0.1 },
  { name: 'db_connection_pool_utilization', threshold: 0.8 },
  { name: 'cache_hit_rate', threshold: 0.7 },
  { name: 'cpu_usage_percent', threshold: 80 },
  { name: 'memory_usage_percent', threshold: 85 }
];

// Alert if metrics degrade
metrics.forEach(metric => {
  if (getMetricValue(metric.name) > metric.threshold) {
    triggerAlert(`${metric.name} exceeded threshold`);
  }
});
```

## Release Notes Template

```markdown
# Release v1.2.0 - 2024-01-15

## Features
- [FEATURE-123] Add product wishlist functionality
- [FEATURE-124] Implement delivery tracking map

## Bug Fixes
- [BUG-456] Fixed checkout payment processing error
- [BUG-457] Corrected inventory count discrepancy

## Performance Improvements
- Optimized product search queries
- Reduced API response time by 30%

## Security Fixes
- Updated dependency vulnerabilities
- Enhanced CSRF token validation

## Migration Guide
- No breaking changes
- Database migration required (auto)
- Backward compatible with v1.1.x

## Deployment Notes
- Estimated deployment time: 15 minutes
- Rollback available to v1.1.5
- No downtime expected
```

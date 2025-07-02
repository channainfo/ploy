# Troubleshooting Guide

## Common Issues and Solutions

This guide covers the most frequent issues encountered during development and their solutions.

## Environment Setup Issues

### Docker Issues

#### Problem: Docker containers fail to start
```bash
# Symptoms
docker-compose up -d
# Error: Cannot start service postgres: driver failed programming external connectivity

# Solutions
# 1. Check for port conflicts
netstat -tulpn | grep -E ':5432|:6379|:8080'
lsof -i :5432  # On macOS

# 2. Stop conflicting services
sudo systemctl stop postgresql  # Linux
brew services stop postgresql   # macOS

# 3. Change ports in docker-compose.yml
services:
  postgres:
    ports:
      - "5433:5432"  # Use different port

# 4. Reset Docker
docker system prune -a
docker-compose down -v
docker-compose up -d
```

#### Problem: PostgreSQL container exits immediately
```bash
# Check logs
docker-compose logs postgres

# Common causes and solutions:
# 1. Data directory permissions
docker-compose down -v  # Remove volumes
docker-compose up -d

# 2. Corrupted data volume
docker volume rm blockchain-loyalty-platform_postgres_data
docker-compose up -d

# 3. Environment variable issues
# Check .env file for proper DATABASE_URL format
```

#### Problem: Redis connection refused
```bash
# Check Redis container
docker-compose logs redis

# Test connection
docker exec -it blockchain-loyalty-platform-redis-1 redis-cli ping

# If container not running
docker-compose restart redis

# Check Redis data persistence
docker exec -it blockchain-loyalty-platform-redis-1 redis-cli
127.0.0.1:6379> INFO replication
```

### Node.js and Package Issues

#### Problem: Node.js version conflicts
```bash
# Check current version
node --version

# Solution: Use nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc  # or ~/.zshrc
nvm install 18
nvm use 18
nvm alias default 18
```

#### Problem: Package installation failures
```bash
# Clear npm cache
npm cache clean --force

# Remove node_modules and reinstall
rm -rf node_modules package-lock.json
npm install

# For workspace issues
lerna clean -y
lerna bootstrap

# Permission issues (avoid sudo with npm)
npm config set prefix ~/.npm-global
export PATH=~/.npm-global/bin:$PATH
```

#### Problem: Lerna/workspace commands fail
```bash
# Reinstall lerna globally
npm install -g lerna

# Or use npx
npx lerna bootstrap
npx lerna run build

# Check lerna configuration
cat lerna.json
```

## Development Issues

### TypeScript Issues

#### Problem: TypeScript compilation errors
```bash
# Check TypeScript version
npx tsc --version

# Clean build artifacts
rm -rf packages/*/dist
rm -rf packages/*/*.tsbuildinfo

# Rebuild
npm run build

# Check specific package
cd packages/[package-name]
npm run type-check

# Show TypeScript configuration
npx tsc --showConfig
```

#### Problem: Module resolution errors
```bash
# Example error: Cannot find module '@loyalty/database'

# Solutions:
# 1. Check tsconfig.json paths
{
  "compilerOptions": {
    "paths": {
      "@loyalty/*": ["./packages/*/src"]
    }
  }
}

# 2. Ensure packages are built
npm run build

# 3. Check package.json main/types fields
{
  "main": "dist/index.js",
  "types": "dist/index.d.ts"
}
```

### Database Issues

#### Problem: Migration fails
```bash
# Check PostgreSQL connection
docker exec blockchain-loyalty-platform-postgres-1 pg_isready -U loyalty_user -d loyalty_platform

# Check migration file syntax
psql -U loyalty_user -d loyalty_platform -f tools/migrations/001_initial_schema.sql --dry-run

# Run migration manually
docker exec -i blockchain-loyalty-platform-postgres-1 psql -U loyalty_user -d loyalty_platform < tools/migrations/001_initial_schema.sql

# Check for existing tables
docker exec blockchain-loyalty-platform-postgres-1 psql -U loyalty_user -d loyalty_platform -c "\dt"
```

#### Problem: Database connection errors
```bash
# Check connection string format
# Correct: postgresql://user:pass@host:port/dbname
# Wrong: postgres://user:pass@host:port/dbname (deprecated)

# Test connection manually
docker exec -it blockchain-loyalty-platform-postgres-1 psql -U loyalty_user -d loyalty_platform

# Check environment variables
echo $DATABASE_URL

# Common fixes:
export DATABASE_URL="postgresql://loyalty_user:loyalty_pass@localhost:5432/loyalty_platform"
```

### Smart Contract Issues

#### Problem: Hardhat compilation fails
```bash
# Clear Hardhat cache
cd packages/contracts
npx hardhat clean

# Check Solidity version
# In hardhat.config.ts
solidity: {
  version: "0.8.19",  // Match pragma in contracts
}

# Reinstall dependencies
rm -rf node_modules package-lock.json
npm install

# Check for syntax errors
npx hardhat compile --verbose
```

#### Problem: Test failures
```bash
# Run specific test
npx hardhat test test/MasterRegistry.test.ts

# Run with verbose output
npx hardhat test --verbose

# Check test network configuration
npx hardhat node  # In separate terminal
npx hardhat test --network localhost

# Common test issues:
# 1. Account balance insufficient - get more testnet tokens
# 2. Gas limit too low - increase in hardhat.config.ts
# 3. Contract already deployed - use fresh deployment
```

#### Problem: Deployment fails
```bash
# Check network configuration
npx hardhat run scripts/deploy.ts --network polygon_mumbai

# Verify environment variables
echo $POLYGON_RPC_URL
echo $PRIVATE_KEY

# Check account balance
npx hardhat run --network polygon_mumbai scripts/check-balance.js

# Gas estimation issues
# Add gasPrice to network config:
polygon_mumbai: {
  url: process.env.POLYGON_RPC_URL,
  accounts: [process.env.PRIVATE_KEY],
  gasPrice: 20000000000,  // 20 gwei
}
```

### API/Backend Issues

#### Problem: NestJS server won't start
```bash
# Check TypeScript compilation
cd packages/backend
npm run build

# Check dependencies
npm install

# Start with verbose logging
npm run start:dev

# Common port conflicts
lsof -i :3000
# Kill process if needed
kill -9 PID
```

#### Problem: Database connection in backend
```bash
# Check environment variables in backend
cd packages/backend
node -e "console.log(process.env.DATABASE_URL)"

# Test database package
cd packages/database
npm run dev

# Check TypeORM configuration (if using)
# Verify database entities are properly imported
```

## Git and CI/CD Issues

### Git Hook Issues

#### Problem: Pre-commit hooks fail
```bash
# Check husky installation
ls .husky/

# Reinstall husky
rm -rf .husky
npm run prepare
npx husky add .husky/pre-commit "npm run lint"
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit ${1}'

# Fix permissions
chmod +x .husky/*

# Skip hooks temporarily (not recommended)
git commit --no-verify -m "message"
```

#### Problem: Linting errors prevent commits
```bash
# Fix auto-fixable issues
npm run lint:fix

# Check specific files
npx eslint src/problematic-file.ts

# Disable specific rules (temporarily)
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const data: any = response;
```

### GitHub Actions Issues

#### Problem: CI pipeline fails
```bash
# Check workflow file syntax
# .github/workflows/ci.yml

# Common fixes:
# 1. Node.js version mismatch
- uses: actions/setup-node@v3
  with:
    node-version: 18.x  # Match local version

# 2. Missing environment variables
env:
  DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
  REDIS_URL: redis://localhost:6379

# 3. Service startup timing
- name: Wait for services
  run: sleep 15
```

## Performance Issues

### Slow Database Queries

#### Problem: Queries taking too long
```sql
-- Check query performance
EXPLAIN ANALYZE SELECT * FROM members WHERE tenant_id = 'uuid';

-- Add missing indexes
CREATE INDEX CONCURRENTLY idx_members_tenant_id_email ON members(tenant_id, email);

-- Check table statistics
ANALYZE members;

-- Monitor slow queries
-- Add to postgresql.conf:
log_min_duration_statement = 1000  -- Log queries > 1 second
```

### High Memory Usage

#### Problem: Node.js memory issues
```bash
# Increase Node.js memory limit
node --max-old-space-size=4096 app.js

# Monitor memory usage
node --inspect app.js
# Open chrome://inspect

# Check for memory leaks
npm install -g clinic
clinic doctor -- node app.js
```

## Security Issues

### Environment Variable Exposure

#### Problem: Secrets in version control
```bash
# Remove from git history
git filter-branch --force --index-filter 'git rm --cached --ignore-unmatch .env' --prune-empty --tag-name-filter cat -- --all

# Prevent future commits
echo ".env" >> .gitignore
echo ".env.*" >> .gitignore
echo "!.env.example" >> .gitignore

# Rotate exposed secrets immediately
```

### API Security Issues

#### Problem: CORS errors
```typescript
// In NestJS main.ts
app.enableCors({
  origin: process.env.FRONTEND_URL,
  credentials: true,
});

// For development only
app.enableCors({
  origin: true,  // Allow all origins
});
```

## Getting Help

### Debug Information Collection

When reporting issues, include:

```bash
# System information
uname -a
node --version
npm --version
docker --version
docker-compose --version

# Project status
git status
git log --oneline -5

# Service status
docker-compose ps
docker-compose logs --tail=50

# Package information
npm list --depth=0
lerna list

# Environment (remove sensitive data)
cat .env.example
```

### Useful Debug Commands

```bash
# Health check all services
curl http://localhost:3000/health
curl http://localhost:8080
redis-cli -h localhost ping

# Database queries
psql $DATABASE_URL -c "SELECT version();"
psql $DATABASE_URL -c "\dt"

# Check file permissions
ls -la tools/scripts/
ls -la .husky/

# Network debugging
netstat -tulpn | grep -E ':3000|:5432|:6379'
docker network ls
docker network inspect blockchain-loyalty-platform_loyalty_network
```

### Log Locations

```bash
# Docker logs
docker-compose logs [service-name]

# Application logs (when implemented)
tail -f logs/application.log
tail -f logs/error.log

# PostgreSQL logs
docker exec blockchain-loyalty-platform-postgres-1 tail -f /var/log/postgresql/postgresql.log

# System logs
journalctl -u docker  # Linux
tail -f /var/log/docker.log  # macOS
```

## Prevention Tips

### Best Practices

1. **Environment Management**
   - Always use `.env.example` as template
   - Never commit sensitive data
   - Use different environments for dev/staging/prod

2. **Version Control**
   - Make small, focused commits
   - Use conventional commit messages
   - Review changes before committing

3. **Development Workflow**
   - Run tests before committing
   - Use TypeScript strict mode
   - Keep dependencies updated

4. **Monitoring**
   - Check service health regularly
   - Monitor resource usage
   - Set up alerting for production

### Regular Maintenance

```bash
# Weekly maintenance
docker system prune
npm audit
npm outdated

# Update dependencies (carefully)
npm update
npx npm-check-updates -u

# Clean build artifacts
npm run clean
npm run build
```

---

If you encounter an issue not covered here, please:
1. Check the error message carefully
2. Search existing GitHub issues
3. Create a detailed bug report with reproduction steps
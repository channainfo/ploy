# Prerequisites & Setup Requirements

## System Requirements

### Hardware Minimum
- **CPU**: 4 cores (Intel i5 or AMD equivalent)
- **RAM**: 8GB (16GB recommended)
- **Storage**: 50GB free space (SSD recommended)
- **Network**: Stable internet connection

### Operating System
- **macOS**: 10.15+ (Catalina or later)
- **Windows**: 10/11 with WSL2
- **Linux**: Ubuntu 20.04+ or equivalent

## Required Software

### Core Development Tools

#### 1. Node.js & Package Management
```bash
# Install Node.js 18+ (LTS recommended)
# Option A: Direct download
# Download from https://nodejs.org/

# Option B: Using nvm (recommended)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 18
nvm use 18
nvm alias default 18

# Verify installation
node --version  # Should be 18.x.x
npm --version   # Should be 9.x.x
```

#### 2. Git & Version Control
```bash
# Install Git (if not already installed)
# macOS: git --version (install Xcode tools if prompted)
# Windows: Download from https://git-scm.com/
# Linux: sudo apt install git

# Configure Git
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global init.defaultBranch main
```

#### 3. Docker & Container Runtime
```bash
# Install Docker Desktop
# Download from https://www.docker.com/products/docker-desktop/

# Verify installation
docker --version
docker-compose --version

# Test Docker
docker run hello-world
```

### Blockchain Development Tools

#### 4. MetaMask Wallet
1. Install MetaMask browser extension
2. Create a new wallet or import existing
3. Save your seed phrase securely
4. Add Polygon Mumbai testnet:
   - Network Name: Mumbai Testnet
   - RPC URL: https://rpc-mumbai.maticvigil.com/
   - Chain ID: 80001
   - Currency Symbol: MATIC
   - Block Explorer: https://mumbai.polygonscan.com/

#### 5. Get Testnet Tokens
```bash
# Get Mumbai MATIC from faucet
# Visit: https://faucet.polygon.technology/
# Enter your wallet address
# Request 0.5 MATIC tokens
```

### Database Systems

#### 6. PostgreSQL (via Docker - recommended)
```bash
# Will be set up via docker-compose
# No separate installation needed
```

#### 7. Redis (via Docker - recommended)
```bash
# Will be set up via docker-compose
# No separate installation needed
```

### Development Environment

#### 8. VS Code & Extensions
```bash
# Install VS Code
# Download from https://code.visualstudio.com/

# Install essential extensions
code --install-extension ms-vscode.vscode-typescript-next
code --install-extension esbenp.prettier-vscode
code --install-extension ms-vscode.vscode-eslint
code --install-extension bradlc.vscode-tailwindcss
code --install-extension juanblanco.solidity
code --install-extension ms-vscode.vscode-docker
code --install-extension GitHub.copilot  # Optional but helpful
```

#### 9. Terminal Setup (Optional but recommended)
```bash
# macOS: Install iTerm2 + Oh My Zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Install useful aliases
echo 'alias ll="ls -la"' >> ~/.zshrc
echo 'alias gs="git status"' >> ~/.zshrc
echo 'alias dc="docker-compose"' >> ~/.zshrc
source ~/.zshrc
```

## API Keys & Services

### Required API Keys

#### 1. Polygon/Ethereum RPC Provider
Choose one:

**Option A: Alchemy (Recommended)**
1. Sign up at https://alchemy.com/
2. Create a new app for Polygon Mumbai
3. Copy the HTTPS endpoint
4. Add to your `.env` file

**Option B: Infura**
1. Sign up at https://infura.io/
2. Create a new project
3. Enable Polygon network
4. Copy the endpoint

#### 2. Block Explorer API (Optional for contract verification)
1. Sign up at https://polygonscan.com/apis
2. Get free API key
3. Add to your `.env` file

### Optional Services (for full features)

#### 3. Email Service (SendGrid)
```bash
# For production email notifications
# Sign up at https://sendgrid.com/
# Get API key for transactional emails
```

#### 4. Payment Processing (Stripe)
```bash
# For handling fiat payments
# Sign up at https://stripe.com/
# Get test API keys
```

## Environment Variables Template

Create a `.env.example` file with:

```bash
# Blockchain
POLYGON_RPC_URL=https://polygon-mumbai.g.alchemy.com/v2/YOUR_API_KEY
POLYGONSCAN_API_KEY=your_polygonscan_key
PRIVATE_KEY=your_wallet_private_key

# Database
DATABASE_URL=postgresql://loyalty_user:loyalty_pass@localhost:5432/loyalty_platform
REDIS_URL=redis://localhost:6379

# API Security
JWT_SECRET=your_super_secret_jwt_key_here
API_KEY_SECRET=your_api_key_encryption_secret
WEBHOOK_SECRET=your_webhook_signing_secret

# External Services
SENDGRID_API_KEY=your_sendgrid_key
STRIPE_SECRET_KEY=sk_test_your_stripe_secret
STRIPE_WEBHOOK_SECRET=whsec_your_webhook_secret

# Application
NODE_ENV=development
LOG_LEVEL=debug
API_PORT=3000
FRONTEND_URL=http://localhost:3001

# Rate Limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100
```

## Pre-Development Checklist

### ✅ Before Starting Day 1

- [ ] Node.js 18+ installed and working
- [ ] Git configured with your credentials
- [ ] Docker Desktop running
- [ ] VS Code with extensions installed
- [ ] MetaMask wallet created
- [ ] Mumbai testnet added to MetaMask
- [ ] Test MATIC tokens received
- [ ] Alchemy/Infura account created
- [ ] API keys obtained and stored securely

### ✅ Validate Your Setup

Run these commands to verify everything works:

```bash
# Check Node.js
node --version && npm --version

# Check Git
git --version

# Check Docker
docker --version && docker-compose --version

# Test Docker
docker run --rm hello-world

# Check if ports are available
netstat -an | grep -E ':3000|:5432|:6379'
# Should return nothing (ports are free)
```

### ✅ Create Project Directory

```bash
# Create your workspace
mkdir -p ~/Projects/loyalty-platform
cd ~/Projects/loyalty-platform

# Verify you're in the right place
pwd
```

## Common Setup Issues

### Issue: Node.js version conflicts
**Solution**: Use nvm to manage Node.js versions
```bash
nvm install 18
nvm use 18
nvm alias default 18
```

### Issue: Docker permission errors (Linux)
**Solution**: Add user to docker group
```bash
sudo usermod -aG docker $USER
# Logout and login again
```

### Issue: Port conflicts
**Solution**: Check what's using the ports
```bash
# Check port usage
lsof -i :3000  # API port
lsof -i :5432  # PostgreSQL
lsof -i :6379  # Redis

# Kill processes if needed
kill -9 PID_NUMBER
```

### Issue: MetaMask connection problems
**Solution**: Reset MetaMask network settings
1. Go to MetaMask Settings > Networks
2. Delete Mumbai testnet
3. Re-add with exact parameters from prerequisites

### Issue: Git authentication problems
**Solution**: Use personal access tokens
1. Go to GitHub Settings > Developer Settings > Personal Access Tokens
2. Create new token with repo permissions
3. Use token as password when prompted

## Getting Help

### Documentation Resources
- [Node.js Documentation](https://nodejs.org/docs/)
- [Docker Documentation](https://docs.docker.com/)
- [Hardhat Documentation](https://hardhat.org/docs)
- [NestJS Documentation](https://docs.nestjs.com/)

### Community Support
- [Polygon Discord](https://discord.gg/polygon)
- [Hardhat Discord](https://discord.gg/hardhat)
- [NestJS Discord](https://discord.gg/nestjs)

### Troubleshooting
If you encounter issues not covered here, check `troubleshooting.md` or create an issue in the project repository.

---

**Setup complete?** Next step: `daily-workflow.md` to understand how to work through each day.
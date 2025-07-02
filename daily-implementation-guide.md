# Daily Implementation Guide - Blockchain Loyalty Platform

## How to Use This Guide

Each day includes:
- **Time Estimate**: Expected hours needed
- **Objectives**: What you'll accomplish
- **Prerequisites**: What needs to be done first
- **Tasks**: Step-by-step instructions
- **Commands**: Exact terminal commands to run
- **Code Snippets**: Copy-paste ready code
- **Validation**: How to verify your work
- **Next Day Setup**: Preparation for tomorrow

---

## WEEK 1: PROJECT SETUP & FOUNDATION

### Day 1: Development Environment Setup
**Time: 2-3 hours**

#### Objectives
- Set up Git repository structure
- Initialize monorepo with workspace management
- Configure TypeScript, ESLint, Prettier
- Create basic project documentation

#### Prerequisites
- Node.js 18+ installed
- Git configured
- VS Code with extensions

#### Tasks

**1. Initialize Project Structure (30 min)**
```bash
mkdir blockchain-loyalty-platform
cd blockchain-loyalty-platform
git init

# Create monorepo structure
mkdir -p {packages,apps,tools,docs}
mkdir -p packages/{contracts,backend,frontend,sdk}
mkdir -p apps/{admin-dashboard,customer-portal}
mkdir -p tools/{scripts,migrations}

# Initialize package.json
npm init -y
```

**2. Set up Workspace Management (30 min)**
```bash
# Install workspace tools
npm install -D lerna nx

# Create lerna.json
cat > lerna.json << 'EOF'
{
  "version": "independent",
  "npmClient": "npm",
  "command": {
    "publish": {
      "conventionalCommits": true
    }
  },
  "packages": [
    "packages/*",
    "apps/*"
  ]
}
EOF

# Create nx.json
cat > nx.json << 'EOF'
{
  "npmScope": "loyalty-platform",
  "affected": {
    "defaultBase": "master"
  },
  "implicitDependencies": {
    "package.json": {
      "dependencies": "*",
      "devDependencies": "*"
    }
  }
}
EOF
```

**3. Configure TypeScript (45 min)**
```bash
# Install TypeScript
npm install -D typescript @types/node ts-node

# Create root tsconfig.json
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "paths": {
      "@loyalty/*": ["./packages/*/src"]
    }
  },
  "exclude": ["node_modules", "dist"]
}
EOF
```

**4. Set up Code Quality Tools (45 min)**
```bash
# Install linting tools
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin prettier eslint-config-prettier eslint-plugin-prettier

# Create .eslintrc.js
cat > .eslintrc.js << 'EOF'
module.exports = {
  parser: '@typescript-eslint/parser',
  parserOptions: {
    project: 'tsconfig.json',
    tsconfigRootDir: __dirname,
    sourceType: 'module',
  },
  plugins: ['@typescript-eslint/eslint-plugin'],
  extends: [
    '@typescript-eslint/recommended',
    'prettier',
  ],
  root: true,
  env: {
    node: true,
    jest: true,
  },
  ignorePatterns: ['.eslintrc.js'],
  rules: {
    '@typescript-eslint/interface-name-prefix': 'off',
    '@typescript-eslint/explicit-function-return-type': 'off',
    '@typescript-eslint/explicit-module-boundary-types': 'off',
    '@typescript-eslint/no-explicit-any': 'off',
  },
};
EOF

# Create .prettierrc
cat > .prettierrc << 'EOF'
{
  "singleQuote": true,
  "trailingComma": "all",
  "semi": true,
  "printWidth": 100,
  "tabWidth": 2
}
EOF
```

**5. Create Documentation Structure (30 min)**
```bash
# Create initial documentation
cat > README.md << 'EOF'
# Blockchain Loyalty Platform

A multi-tenant SaaS platform for blockchain-based loyalty programs.

## Project Structure

```
├── packages/
│   ├── contracts/     # Smart contracts
│   ├── backend/       # API server
│   ├── frontend/      # Shared UI components
│   └── sdk/          # Client SDKs
├── apps/
│   ├── admin-dashboard/  # Merchant admin interface
│   └── customer-portal/  # Customer wallet interface
├── tools/
│   ├── scripts/       # Utility scripts
│   └── migrations/    # Database migrations
└── docs/             # Documentation
```

## Getting Started

1. Install dependencies: `npm install`
2. Set up environment: `cp .env.example .env`
3. Start development: `npm run dev`

## Commands

- `npm run build` - Build all packages
- `npm run test` - Run tests
- `npm run lint` - Run linting
- `npm run dev` - Start development servers
EOF

# Create .gitignore
cat > .gitignore << 'EOF'
# Dependencies
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Environment variables
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# Build outputs
dist/
build/
*.tsbuildinfo

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Blockchain
cache/
artifacts/
typechain/
coverage/

# Logs
logs/
*.log
EOF
```

#### Validation
```bash
# Verify setup
npm run lint --if-present
npx tsc --noEmit
git status
```

#### Next Day Setup
- Review monorepo structure
- Plan database schema design

---

### Day 2: Database & Infrastructure Setup
**Time: 3-4 hours**

#### Objectives
- Design PostgreSQL schema
- Set up Redis for caching
- Configure Docker for local development
- Create database migration system

#### Prerequisites
- Day 1 completed
- Docker installed
- PostgreSQL knowledge

#### Tasks

**1. Create Docker Development Environment (60 min)**
```bash
# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: loyalty_platform
      POSTGRES_USER: loyalty_user
      POSTGRES_PASSWORD: loyalty_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./tools/migrations:/docker-entrypoint-initdb.d
    networks:
      - loyalty_network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - loyalty_network

  adminer:
    image: adminer
    ports:
      - "8080:8080"
    networks:
      - loyalty_network

volumes:
  postgres_data:
  redis_data:

networks:
  loyalty_network:
    driver: bridge
EOF
```

**2. Design Database Schema (90 min)**
```bash
# Create migration directory
mkdir -p tools/migrations

# Create initial schema
cat > tools/migrations/001_initial_schema.sql << 'EOF'
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Tenants table
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    subdomain VARCHAR(100) UNIQUE NOT NULL,
    domain VARCHAR(255),
    status VARCHAR(20) DEFAULT 'active',
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create index on subdomain for fast lookup
CREATE INDEX idx_tenants_subdomain ON tenants(subdomain);

-- Members table
CREATE TABLE members (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255),
    phone VARCHAR(20),
    wallet_address VARCHAR(42),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    date_of_birth DATE,
    metadata JSONB DEFAULT '{}',
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, email),
    UNIQUE(tenant_id, phone)
);

-- Create indexes for members
CREATE INDEX idx_members_tenant_id ON members(tenant_id);
CREATE INDEX idx_members_email ON members(tenant_id, email);
CREATE INDEX idx_members_wallet ON members(wallet_address);

-- Point transactions table
CREATE TABLE point_transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    member_id UUID NOT NULL REFERENCES members(id) ON DELETE CASCADE,
    order_id VARCHAR(255),
    transaction_type VARCHAR(20) NOT NULL, -- 'earn', 'redeem', 'revoke', 'expire'
    amount INTEGER NOT NULL,
    balance_after INTEGER NOT NULL,
    status VARCHAR(20) DEFAULT 'pending', -- 'pending', 'confirmed', 'failed'
    blockchain_tx_hash VARCHAR(66),
    metadata JSONB DEFAULT '{}',
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes for point transactions
CREATE INDEX idx_point_transactions_tenant_id ON point_transactions(tenant_id);
CREATE INDEX idx_point_transactions_member_id ON point_transactions(member_id);
CREATE INDEX idx_point_transactions_order_id ON point_transactions(tenant_id, order_id);
CREATE INDEX idx_point_transactions_status ON point_transactions(status);
CREATE INDEX idx_point_transactions_created_at ON point_transactions(created_at);

-- Orders table
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    member_id UUID NOT NULL REFERENCES members(id) ON DELETE CASCADE,
    external_order_id VARCHAR(255) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(20) DEFAULT 'pending',
    points_awarded INTEGER DEFAULT 0,
    points_redeemed INTEGER DEFAULT 0,
    cancellation_policy VARCHAR(50),
    can_cancel_until TIMESTAMP,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, external_order_id)
);

-- Create indexes for orders
CREATE INDEX idx_orders_tenant_id ON orders(tenant_id);
CREATE INDEX idx_orders_member_id ON orders(member_id);
CREATE INDEX idx_orders_external_id ON orders(tenant_id, external_order_id);
CREATE INDEX idx_orders_status ON orders(status);

-- Tenant smart contracts table
CREATE TABLE tenant_contracts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    contract_address VARCHAR(42) NOT NULL,
    contract_type VARCHAR(50) NOT NULL, -- 'loyalty_token', 'nft_membership'
    network VARCHAR(20) NOT NULL, -- 'polygon', 'bsc', 'ethereum'
    deployment_tx_hash VARCHAR(66),
    is_active BOOLEAN DEFAULT true,
    deployed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, contract_type, network)
);

-- Create indexes for tenant contracts
CREATE INDEX idx_tenant_contracts_tenant_id ON tenant_contracts(tenant_id);
CREATE INDEX idx_tenant_contracts_address ON tenant_contracts(contract_address);

-- Fraud detection table
CREATE TABLE fraud_alerts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    member_id UUID REFERENCES members(id) ON DELETE CASCADE,
    alert_type VARCHAR(50) NOT NULL,
    risk_score INTEGER NOT NULL,
    details JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'open', -- 'open', 'investigating', 'resolved', 'false_positive'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP
);

-- Create indexes for fraud alerts
CREATE INDEX idx_fraud_alerts_tenant_id ON fraud_alerts(tenant_id);
CREATE INDEX idx_fraud_alerts_member_id ON fraud_alerts(member_id);
CREATE INDEX idx_fraud_alerts_risk_score ON fraud_alerts(risk_score);
CREATE INDEX idx_fraud_alerts_status ON fraud_alerts(status);

-- API keys table
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    key_name VARCHAR(100) NOT NULL,
    key_hash VARCHAR(255) NOT NULL,
    permissions JSONB DEFAULT '{}',
    is_active BOOLEAN DEFAULT true,
    last_used_at TIMESTAMP,
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, key_name)
);

-- Create indexes for API keys
CREATE INDEX idx_api_keys_tenant_id ON api_keys(tenant_id);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);

-- Create updated_at trigger function
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply updated_at triggers
CREATE TRIGGER update_tenants_updated_at BEFORE UPDATE ON tenants FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_members_updated_at BEFORE UPDATE ON members FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_point_transactions_updated_at BEFORE UPDATE ON point_transactions FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
CREATE TRIGGER update_orders_updated_at BEFORE UPDATE ON orders FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
EOF
```

**3. Set up Environment Configuration (30 min)**
```bash
# Create environment template
cat > .env.example << 'EOF'
# Database
DATABASE_URL=postgresql://loyalty_user:loyalty_pass@localhost:5432/loyalty_platform
REDIS_URL=redis://localhost:6379

# Blockchain
POLYGON_RPC_URL=https://polygon-mumbai.g.alchemy.com/v2/YOUR_API_KEY
PRIVATE_KEY=your_private_key_here
CONTRACT_DEPLOYER_ADDRESS=0x...

# API
JWT_SECRET=your_jwt_secret_here
API_PORT=3000
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100

# Security
ENCRYPTION_KEY=your_encryption_key_here
WEBHOOK_SECRET=your_webhook_secret_here

# External Services
SENDGRID_API_KEY=your_sendgrid_key
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Environment
NODE_ENV=development
LOG_LEVEL=debug
EOF

# Copy to actual env file
cp .env.example .env
```

**4. Create Database Connection Package (60 min)**
```bash
# Initialize database package
cd packages
mkdir -p database/src
cd database

# Create package.json
cat > package.json << 'EOF'
{
  "name": "@loyalty/database",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "dev": "ts-node src/index.ts"
  },
  "dependencies": {
    "pg": "^8.11.0",
    "redis": "^4.6.0",
    "dotenv": "^16.0.3"
  },
  "devDependencies": {
    "@types/pg": "^8.10.0"
  }
}
EOF

# Create database connection
cat > src/database.ts << 'EOF'
import { Pool, PoolConfig } from 'pg';
import { createClient, RedisClientType } from 'redis';

export class DatabaseConnection {
  private static instance: DatabaseConnection;
  private pgPool: Pool;
  private redisClient: RedisClientType;

  private constructor() {
    // PostgreSQL connection
    const dbConfig: PoolConfig = {
      connectionString: process.env.DATABASE_URL,
      max: 20,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
    };
    this.pgPool = new Pool(dbConfig);

    // Redis connection
    this.redisClient = createClient({
      url: process.env.REDIS_URL,
    });
  }

  public static getInstance(): DatabaseConnection {
    if (!DatabaseConnection.instance) {
      DatabaseConnection.instance = new DatabaseConnection();
    }
    return DatabaseConnection.instance;
  }

  public async connect(): Promise<void> {
    try {
      // Test PostgreSQL connection
      const client = await this.pgPool.connect();
      console.log('PostgreSQL connected successfully');
      client.release();

      // Connect to Redis
      await this.redisClient.connect();
      console.log('Redis connected successfully');
    } catch (error) {
      console.error('Database connection failed:', error);
      throw error;
    }
  }

  public async disconnect(): Promise<void> {
    await this.pgPool.end();
    await this.redisClient.disconnect();
  }

  public getPool(): Pool {
    return this.pgPool;
  }

  public getRedis(): RedisClientType {
    return this.redisClient;
  }

  public async query(text: string, params?: any[]): Promise<any> {
    const client = await this.pgPool.connect();
    try {
      const result = await client.query(text, params);
      return result;
    } finally {
      client.release();
    }
  }
}
EOF

# Create index file
cat > src/index.ts << 'EOF'
export { DatabaseConnection } from './database';
export * from './types';
EOF

# Create types
cat > src/types.ts << 'EOF'
export interface Tenant {
  id: string;
  name: string;
  subdomain: string;
  domain?: string;
  status: 'active' | 'inactive' | 'suspended';
  settings: Record<string, any>;
  created_at: Date;
  updated_at: Date;
}

export interface Member {
  id: string;
  tenant_id: string;
  email?: string;
  phone?: string;
  wallet_address?: string;
  first_name?: string;
  last_name?: string;
  date_of_birth?: Date;
  metadata: Record<string, any>;
  status: 'active' | 'inactive' | 'suspended';
  created_at: Date;
  updated_at: Date;
}

export interface PointTransaction {
  id: string;
  tenant_id: string;
  member_id: string;
  order_id?: string;
  transaction_type: 'earn' | 'redeem' | 'revoke' | 'expire';
  amount: number;
  balance_after: number;
  status: 'pending' | 'confirmed' | 'failed';
  blockchain_tx_hash?: string;
  metadata: Record<string, any>;
  expires_at?: Date;
  created_at: Date;
  updated_at: Date;
}

export interface Order {
  id: string;
  tenant_id: string;
  member_id: string;
  external_order_id: string;
  amount: number;
  currency: string;
  status: 'pending' | 'confirmed' | 'cancelled' | 'refunded';
  points_awarded: number;
  points_redeemed: number;
  cancellation_policy?: string;
  can_cancel_until?: Date;
  metadata: Record<string, any>;
  created_at: Date;
  updated_at: Date;
}
EOF

# Create TypeScript config for package
cat > tsconfig.json << 'EOF'
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
EOF

cd ../..
```

#### Validation
```bash
# Start services
docker-compose up -d

# Check service status
docker-compose ps

# Test database connection
docker exec -it blockchain-loyalty-platform_postgres_1 psql -U loyalty_user -d loyalty_platform -c "\dt"

# Test Redis connection
docker exec -it blockchain-loyalty-platform_redis_1 redis-cli ping
```

#### Next Day Setup
- Gather Alchemy/Infura API keys
- Set up MetaMask for development
- Review Hardhat documentation

---

### Day 3: Blockchain Development Setup
**Time: 3-4 hours**

#### Objectives
- Install and configure Hardhat framework
- Set up Polygon Mumbai testnet connection
- Create wallet management system
- Write first smart contract

#### Prerequisites
- Day 2 completed
- MetaMask installed
- Basic Solidity knowledge

#### Tasks

**1. Initialize Smart Contracts Package (45 min)**
```bash
# Create contracts package
cd packages
mkdir -p contracts
cd contracts

# Initialize Hardhat project
npm init -y
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox
npx hardhat

# Choose "Create a TypeScript project"
# Install suggested dependencies
npm install --save-dev @nomicfoundation/hardhat-network-helpers @nomicfoundation/hardhat-chai-matchers @nomiclabs/hardhat-ethers @nomiclabs/hardhat-etherscan chai ethers hardhat-gas-reporter solidity-coverage @typechain/hardhat typechain @typechain/ethers-v5 @types/chai @types/mocha

# Install OpenZeppelin contracts
npm install @openzeppelin/contracts @openzeppelin/contracts-upgradeable
```

**2. Configure Hardhat for Multi-Chain (60 min)**
```bash
# Replace hardhat.config.ts
cat > hardhat.config.ts << 'EOF'
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "@nomiclabs/hardhat-etherscan";
import "hardhat-gas-reporter";
import "solidity-coverage";
import * as dotenv from "dotenv";

dotenv.config({ path: "../../.env" });

const config: HardhatUserConfig = {
  solidity: {
    version: "0.8.19",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,
      },
    },
  },
  networks: {
    hardhat: {
      chainId: 1337,
    },
    polygon_mumbai: {
      url: process.env.POLYGON_RPC_URL || "https://rpc-mumbai.maticvigil.com",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
      gasPrice: 20000000000,
    },
    polygon: {
      url: process.env.POLYGON_MAINNET_RPC_URL || "https://polygon-rpc.com",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
    bsc_testnet: {
      url: "https://data-seed-prebsc-1-s1.binance.org:8545",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
    bsc: {
      url: "https://bsc-dataseed1.binance.org",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
  },
  gasReporter: {
    enabled: process.env.REPORT_GAS !== undefined,
    currency: "USD",
  },
  etherscan: {
    apiKey: {
      polygon: process.env.POLYGONSCAN_API_KEY || "",
      polygonMumbai: process.env.POLYGONSCAN_API_KEY || "",
      bsc: process.env.BSCSCAN_API_KEY || "",
      bscTestnet: process.env.BSCSCAN_API_KEY || "",
    },
  },
  typechain: {
    outDir: "typechain-types",
    target: "ethers-v5",
  },
};

export default config;
EOF
```

**3. Create Master Registry Contract (90 min)**
```bash
# Create contracts directory structure
mkdir -p contracts/{core,interfaces,libraries,mocks}

# Create IMasterRegistry interface
cat > contracts/interfaces/IMasterRegistry.sol << 'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IMasterRegistry {
    event TenantContractDeployed(
        uint256 indexed tenantId,
        address indexed contractAddress,
        string contractType
    );
    
    event FactoryAuthorized(address indexed factory, bool authorized);
    
    function deployTenantContract(
        uint256 tenantId,
        string memory tokenName,
        string memory tokenSymbol,
        address admin
    ) external returns (address);
    
    function getTenantContract(uint256 tenantId) external view returns (address);
    
    function authorizeFactory(address factory, bool authorized) external;
    
    function isAuthorizedFactory(address factory) external view returns (bool);
}
EOF

# Create MasterRegistry contract
cat > contracts/core/MasterRegistry.sol << 'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "../interfaces/IMasterRegistry.sol";
import "./LoyaltyToken.sol";

contract MasterRegistry is IMasterRegistry, Ownable, Pausable {
    mapping(address => bool) public authorizedFactories;
    mapping(uint256 => address) public tenantContracts;
    mapping(address => uint256) public contractToTenant;
    
    uint256 public totalTenants;
    
    modifier onlyAuthorizedFactory() {
        require(authorizedFactories[msg.sender], "MasterRegistry: Not authorized factory");
        _;
    }
    
    constructor() {
        authorizedFactories[msg.sender] = true;
    }
    
    function deployTenantContract(
        uint256 tenantId,
        string memory tokenName,
        string memory tokenSymbol,
        address admin
    ) external override onlyAuthorizedFactory whenNotPaused returns (address) {
        require(tenantContracts[tenantId] == address(0), "MasterRegistry: Tenant contract already exists");
        require(admin != address(0), "MasterRegistry: Invalid admin address");
        
        LoyaltyToken loyaltyToken = new LoyaltyToken(
            tokenName,
            tokenSymbol,
            admin,
            address(this)
        );
        
        address contractAddress = address(loyaltyToken);
        tenantContracts[tenantId] = contractAddress;
        contractToTenant[contractAddress] = tenantId;
        totalTenants++;
        
        emit TenantContractDeployed(tenantId, contractAddress, "LoyaltyToken");
        
        return contractAddress;
    }
    
    function getTenantContract(uint256 tenantId) external view override returns (address) {
        return tenantContracts[tenantId];
    }
    
    function authorizeFactory(address factory, bool authorized) external override onlyOwner {
        authorizedFactories[factory] = authorized;
        emit FactoryAuthorized(factory, authorized);
    }
    
    function isAuthorizedFactory(address factory) external view override returns (bool) {
        return authorizedFactories[factory];
    }
    
    function pause() external onlyOwner {
        _pause();
    }
    
    function unpause() external onlyOwner {
        _unpause();
    }
}
EOF

# Create LoyaltyToken contract
cat > contracts/core/LoyaltyToken.sol << 'EOF'
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract LoyaltyToken is ERC20, Pausable, AccessControl, ReentrancyGuard {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");
    
    address public masterRegistry;
    
    struct PointTransaction {
        uint256 amount;
        string orderId;
        uint256 timestamp;
        bool revoked;
    }
    
    mapping(address => PointTransaction[]) public userTransactions;
    mapping(string => bool) public processedOrders;
    
    event PointsAwarded(address indexed user, uint256 amount, string orderId);
    event PointsRevoked(address indexed user, uint256 amount, string orderId);
    event PointsRedeemed(address indexed user, uint256 amount, string purpose);
    
    modifier onlyValidAddress(address account) {
        require(account != address(0), "LoyaltyToken: Invalid address");
        _;
    }
    
    modifier onlyUniqueOrder(string memory orderId) {
        require(!processedOrders[orderId], "LoyaltyToken: Order already processed");
        _;
    }
    
    constructor(
        string memory name,
        string memory symbol,
        address admin,
        address _masterRegistry
    ) ERC20(name, symbol) {
        require(admin != address(0), "LoyaltyToken: Invalid admin");
        require(_masterRegistry != address(0), "LoyaltyToken: Invalid registry");
        
        masterRegistry = _masterRegistry;
        
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(MINTER_ROLE, admin);
        _grantRole(BURNER_ROLE, admin);
        _grantRole(PAUSER_ROLE, admin);
    }
    
    function awardPoints(
        address user,
        uint256 amount,
        string memory orderId
    ) external onlyRole(MINTER_ROLE) onlyValidAddress(user) onlyUniqueOrder(orderId) whenNotPaused {
        require(amount > 0, "LoyaltyToken: Amount must be positive");
        
        _mint(user, amount);
        
        userTransactions[user].push(PointTransaction({
            amount: amount,
            orderId: orderId,
            timestamp: block.timestamp,
            revoked: false
        }));
        
        processedOrders[orderId] = true;
        
        emit PointsAwarded(user, amount, orderId);
    }
    
    function revokePoints(
        address user,
        uint256 amount,
        string memory orderId
    ) external onlyRole(BURNER_ROLE) onlyValidAddress(user) whenNotPaused {
        require(amount > 0, "LoyaltyToken: Amount must be positive");
        require(balanceOf(user) >= amount, "LoyaltyToken: Insufficient balance");
        
        _burn(user, amount);
        
        // Mark transaction as revoked
        PointTransaction[] storage transactions = userTransactions[user];
        for (uint i = 0; i < transactions.length; i++) {
            if (keccak256(bytes(transactions[i].orderId)) == keccak256(bytes(orderId))) {
                transactions[i].revoked = true;
                break;
            }
        }
        
        emit PointsRevoked(user, amount, orderId);
    }
    
    function redeemPoints(
        uint256 amount,
        string memory purpose
    ) external whenNotPaused nonReentrant {
        require(amount > 0, "LoyaltyToken: Amount must be positive");
        require(balanceOf(msg.sender) >= amount, "LoyaltyToken: Insufficient balance");
        
        _burn(msg.sender, amount);
        
        emit PointsRedeemed(msg.sender, amount, purpose);
    }
    
    function getUserTransactions(address user) external view returns (PointTransaction[] memory) {
        return userTransactions[user];
    }
    
    function getUserTransactionCount(address user) external view returns (uint256) {
        return userTransactions[user].length;
    }
    
    function pause() external onlyRole(PAUSER_ROLE) {
        _pause();
    }
    
    function unpause() external onlyRole(PAUSER_ROLE) {
        _unpause();
    }
    
    // Override decimals to return 0 (points are whole numbers)
    function decimals() public pure override returns (uint8) {
        return 0;
    }
}
EOF
```

**4. Create Deployment Scripts (45 min)**
```bash
# Create deployment script
mkdir -p scripts
cat > scripts/deploy.ts << 'EOF'
import { ethers } from "hardhat";
import { MasterRegistry } from "../typechain-types";

async function main() {
  const [deployer] = await ethers.getSigners();

  console.log("Deploying contracts with the account:", deployer.address);
  console.log("Account balance:", (await deployer.getBalance()).toString());

  // Deploy MasterRegistry
  const MasterRegistry = await ethers.getContractFactory("MasterRegistry");
  const masterRegistry = await MasterRegistry.deploy();
  await masterRegistry.deployed();

  console.log("MasterRegistry deployed to:", masterRegistry.address);

  // Verify deployment by checking owner
  const owner = await masterRegistry.owner();
  console.log("MasterRegistry owner:", owner);

  // Deploy a test tenant contract
  const tenantId = 1;
  const tokenName = "Test Loyalty Points";
  const tokenSymbol = "TLP";
  const admin = deployer.address;

  console.log("Deploying test tenant contract...");
  const deployTx = await masterRegistry.deployTenantContract(
    tenantId,
    tokenName,
    tokenSymbol,
    admin
  );
  await deployTx.wait();

  const tenantContractAddress = await masterRegistry.getTenantContract(tenantId);
  console.log("Test tenant contract deployed to:", tenantContractAddress);

  // Save deployment addresses
  const deployment = {
    network: await ethers.provider.getNetwork(),
    masterRegistry: masterRegistry.address,
    testTenantContract: tenantContractAddress,
    deployer: deployer.address,
    deployedAt: new Date().toISOString(),
  };

  console.log("Deployment complete:", deployment);
  
  return deployment;
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
EOF

# Create test deployment script for local network
cat > scripts/deploy-local.ts << 'EOF'
import { ethers } from "hardhat";

async function main() {
  console.log("Starting local deployment...");
  
  const deployment = await import("./deploy");
  await deployment.main();
  
  console.log("Local deployment complete!");
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
EOF
```

**5. Write Basic Tests (30 min)**
```bash
# Create test file
cat > test/MasterRegistry.test.ts << 'EOF'
import { expect } from "chai";
import { ethers } from "hardhat";
import { MasterRegistry, LoyaltyToken } from "../typechain-types";
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";

describe("MasterRegistry", function () {
  let masterRegistry: MasterRegistry;
  let owner: SignerWithAddress;
  let admin: SignerWithAddress;
  let user: SignerWithAddress;

  beforeEach(async function () {
    [owner, admin, user] = await ethers.getSigners();

    const MasterRegistry = await ethers.getContractFactory("MasterRegistry");
    masterRegistry = await MasterRegistry.deploy();
    await masterRegistry.deployed();
  });

  describe("Deployment", function () {
    it("Should set the right owner", async function () {
      expect(await masterRegistry.owner()).to.equal(owner.address);
    });

    it("Should authorize deployer as factory", async function () {
      expect(await masterRegistry.isAuthorizedFactory(owner.address)).to.be.true;
    });
  });

  describe("Tenant Contract Deployment", function () {
    it("Should deploy a tenant contract", async function () {
      const tenantId = 1;
      const tokenName = "Test Token";
      const tokenSymbol = "TT";

      await expect(
        masterRegistry.deployTenantContract(tenantId, tokenName, tokenSymbol, admin.address)
      ).to.emit(masterRegistry, "TenantContractDeployed");

      const contractAddress = await masterRegistry.getTenantContract(tenantId);
      expect(contractAddress).to.not.equal(ethers.constants.AddressZero);
    });

    it("Should not allow duplicate tenant contracts", async function () {
      const tenantId = 1;
      const tokenName = "Test Token";
      const tokenSymbol = "TT";

      await masterRegistry.deployTenantContract(tenantId, tokenName, tokenSymbol, admin.address);

      await expect(
        masterRegistry.deployTenantContract(tenantId, tokenName, tokenSymbol, admin.address)
      ).to.be.revertedWith("MasterRegistry: Tenant contract already exists");
    });
  });

  describe("Factory Authorization", function () {
    it("Should allow owner to authorize factories", async function () {
      await expect(
        masterRegistry.authorizeFactory(user.address, true)
      ).to.emit(masterRegistry, "FactoryAuthorized");

      expect(await masterRegistry.isAuthorizedFactory(user.address)).to.be.true;
    });

    it("Should not allow non-owner to authorize factories", async function () {
      await expect(
        masterRegistry.connect(user).authorizeFactory(user.address, true)
      ).to.be.revertedWith("Ownable: caller is not the owner");
    });
  });
});
EOF

cd ../..
```

#### Validation
```bash
# Compile contracts
cd packages/contracts
npx hardhat compile

# Run tests
npx hardhat test

# Deploy to local network
npx hardhat node # In one terminal
npx hardhat run scripts/deploy-local.ts --network localhost # In another terminal
```

#### Next Day Setup
- Get Mumbai testnet MATIC tokens
- Set up block explorer API keys
- Plan NestJS backend structure

---

### Day 4: CI/CD Pipeline & Project Structure
**Time: 2-3 hours**

#### Objectives
- Set up GitHub Actions for CI/CD
- Create workspace scripts
- Configure automated testing
- Set up code quality checks

#### Prerequisites
- Day 3 completed
- GitHub repository created
- Basic understanding of GitHub Actions

#### Tasks

**1. Create GitHub Actions Workflow (60 min)**
```bash
# Create GitHub Actions directory
mkdir -p .github/workflows

# Create main CI workflow
cat > .github/workflows/ci.yml << 'EOF'
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    strategy:
      matrix:
        node-version: [18.x, 20.x]

    steps:
    - uses: actions/checkout@v3
    
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Lint code
      run: npm run lint
    
    - name: Type check
      run: npm run type-check
    
    - name: Run tests
      run: npm run test
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
        REDIS_URL: redis://localhost:6379
    
    - name: Test smart contracts
      run: |
        cd packages/contracts
        npx hardhat test
    
    - name: Build packages
      run: npm run build

  security:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run security audit
      run: npm audit --audit-level high
    
    - name: Check for secrets
      uses: trufflesecurity/trufflehog@main
      with:
        path: ./
        base: main
        head: HEAD

  smart-contract-analysis:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Use Node.js
      uses: actions/setup-node@v3
      with:
        node-version: 18.x
        cache: 'npm'
    
    - name: Install dependencies
      run: |
        cd packages/contracts
        npm ci
    
    - name: Run Slither analysis
      uses: crytic/slither-action@v0.3.0
      with:
        target: packages/contracts
        slither-config: packages/contracts/slither.config.json
      continue-on-error: true

  deploy-staging:
    needs: [test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to staging
      run: echo "Deploy to staging environment"
      # Add actual deployment steps here

  deploy-production:
    needs: [test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to production
      run: echo "Deploy to production environment"
      # Add actual deployment steps here
EOF
```

**2. Set up Workspace Scripts (45 min)**
```bash
# Update root package.json with workspace scripts
cat > package.json << 'EOF'
{
  "name": "blockchain-loyalty-platform",
  "version": "1.0.0",
  "private": true,
  "workspaces": [
    "packages/*",
    "apps/*"
  ],
  "scripts": {
    "build": "lerna run build",
    "test": "lerna run test",
    "lint": "lerna run lint",
    "lint:fix": "lerna run lint:fix",
    "type-check": "lerna run type-check",
    "dev": "lerna run dev --parallel",
    "clean": "lerna run clean && lerna clean -y",
    "bootstrap": "lerna bootstrap",
    "publish": "lerna publish",
    "contracts:compile": "cd packages/contracts && npx hardhat compile",
    "contracts:test": "cd packages/contracts && npx hardhat test",
    "contracts:deploy:local": "cd packages/contracts && npx hardhat run scripts/deploy-local.ts --network localhost",
    "contracts:deploy:mumbai": "cd packages/contracts && npx hardhat run scripts/deploy.ts --network polygon_mumbai",
    "db:migrate": "cd tools/migrations && ./migrate.sh",
    "db:reset": "docker-compose down -v && docker-compose up -d postgres redis",
    "db:seed": "cd tools/migrations && ./seed.sh",
    "docker:dev": "docker-compose -f docker-compose.dev.yml up",
    "docker:prod": "docker-compose -f docker-compose.prod.yml up -d",
    "setup": "npm install && npm run bootstrap && npm run contracts:compile",
    "start": "lerna run start --parallel",
    "version": "lerna version",
    "commit": "git-cz",
    "prepare": "husky install"
  },
  "devDependencies": {
    "@commitlint/cli": "^17.6.0",
    "@commitlint/config-conventional": "^17.6.0",
    "commitizen": "^4.3.0",
    "cz-conventional-changelog": "^3.3.0",
    "husky": "^8.0.0",
    "lerna": "^7.0.0",
    "nx": "^16.0.0",
    "prettier": "^2.8.0"
  },
  "config": {
    "commitizen": {
      "path": "cz-conventional-changelog"
    }
  }
}
EOF
```

**3. Set up Git Hooks (30 min)**
```bash
# Install husky and commitlint
npm install -D husky @commitlint/cli @commitlint/config-conventional

# Initialize husky
npx husky install

# Create commit-msg hook
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit ${1}'

# Create pre-commit hook
npx husky add .husky/pre-commit 'npm run lint && npm run type-check'

# Create pre-push hook
npx husky add .husky/pre-push 'npm run test'

# Create commitlint config
cat > .commitlintrc.json << 'EOF'
{
  "extends": ["@commitlint/config-conventional"],
  "rules": {
    "type-enum": [
      2,
      "always",
      [
        "feat",
        "fix",
        "docs",
        "style",
        "refactor",
        "perf",
        "test",
        "build",
        "ci",
        "chore",
        "revert"
      ]
    ],
    "subject-case": [2, "never", ["start-case", "pascal-case", "upper-case"]],
    "subject-max-length": [2, "always", 72]
  }
}
EOF
```

**4. Create Development Tools (45 min)**
```bash
# Create development utilities
mkdir -p tools/scripts

# Create database migration script
cat > tools/scripts/migrate.sh << 'EOF'
#!/bin/bash

# Database migration script
set -e

echo "Running database migrations..."

# Check if PostgreSQL is running
if ! docker-compose ps postgres | grep -q "Up"; then
    echo "Starting PostgreSQL..."
    docker-compose up -d postgres
    sleep 5
fi

# Run migrations
docker exec -i blockchain-loyalty-platform_postgres_1 psql -U loyalty_user -d loyalty_platform < tools/migrations/001_initial_schema.sql

echo "Migrations completed successfully!"
EOF

# Make script executable
chmod +x tools/scripts/migrate.sh

# Create seed data script
cat > tools/scripts/seed.sh << 'EOF'
#!/bin/bash

# Database seed script
set -e

echo "Seeding database with test data..."

# Create seed SQL
cat > /tmp/seed.sql << 'SQL'
-- Insert test tenant
INSERT INTO tenants (name, subdomain, settings) VALUES 
('Test Business', 'testbiz', '{"pointsPerDollar": 1, "minimumRedemption": 100}');

-- Get tenant ID
DO $$
DECLARE
    tenant_uuid UUID;
BEGIN
    SELECT id INTO tenant_uuid FROM tenants WHERE subdomain = 'testbiz';
    
    -- Insert test members
    INSERT INTO members (tenant_id, email, first_name, last_name) VALUES 
    (tenant_uuid, 'test1@example.com', 'John', 'Doe'),
    (tenant_uuid, 'test2@example.com', 'Jane', 'Smith');
END $$;
SQL

# Execute seed
docker exec -i blockchain-loyalty-platform_postgres_1 psql -U loyalty_user -d loyalty_platform < /tmp/seed.sql

echo "Database seeded successfully!"
EOF

chmod +x tools/scripts/seed.sh

# Create environment setup script
cat > tools/scripts/setup-env.sh << 'EOF'
#!/bin/bash

# Environment setup script
set -e

echo "Setting up development environment..."

# Check if .env exists
if [ ! -f .env ]; then
    echo "Creating .env file from template..."
    cp .env.example .env
    echo "Please update .env file with your configuration"
fi

# Start services
echo "Starting development services..."
docker-compose up -d

# Wait for services
echo "Waiting for services to be ready..."
sleep 10

# Run migrations
echo "Running database migrations..."
./tools/scripts/migrate.sh

# Seed database
echo "Seeding database..."
./tools/scripts/seed.sh

echo "Environment setup complete!"
echo "Run 'npm run dev' to start development servers"
EOF

chmod +x tools/scripts/setup-env.sh
```

**5. Create Documentation Templates (30 min)**
```bash
# Create docs directory
mkdir -p docs/{api,architecture,deployment}

# Create API documentation template
cat > docs/api/README.md << 'EOF'
# API Documentation

## Overview
This document describes the REST API endpoints for the Blockchain Loyalty Platform.

## Base URL
```
Development: http://localhost:3000/api/v1
Staging: https://api-staging.loyalty-platform.com/v1
Production: https://api.loyalty-platform.com/v1
```

## Authentication
All API requests require authentication using Bearer tokens:
```
Authorization: Bearer YOUR_API_KEY
```

## Endpoints

### Members
- `POST /members` - Create a new member
- `GET /members/{id}` - Get member details
- `PATCH /members/{id}` - Update member
- `DELETE /members/{id}` - Delete member

### Points
- `POST /points/award` - Award points to member
- `POST /points/revoke` - Revoke points from member
- `GET /members/{id}/balance` - Get member balance
- `GET /members/{id}/transactions` - Get transaction history

### Orders
- `POST /orders` - Create order
- `GET /orders/{id}` - Get order details
- `POST /orders/{id}/cancel` - Cancel order
- `POST /orders/{id}/refund` - Process refund

## Error Responses
All errors follow the standard format:
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable error message",
    "details": {}
  }
}
```
EOF

# Create architecture documentation
cat > docs/architecture/README.md << 'EOF'
# System Architecture

## Overview
The Blockchain Loyalty Platform is built as a multi-tenant SaaS solution with the following key components:

## High-Level Architecture
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Admin Dashboard│    │ Customer Portal │    │   Mobile Apps   │
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌─────────────┴───────────┐
                    │      API Gateway        │
                    └─────────────┬───────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
    ┌─────┴─────┐          ┌─────┴─────┐          ┌─────┴─────┐
    │  Auth     │          │  Core API │          │ Blockchain│
    │  Service  │          │  Service  │          │  Service  │
    └─────┬─────┘          └─────┬─────┘          └─────┬─────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌─────────────┴───────────┐
                    │     Database Layer      │
                    │  PostgreSQL + Redis     │
                    └─────────────────────────┘
```

## Components

### 1. Frontend Applications
- **Admin Dashboard**: React/Next.js application for merchants
- **Customer Portal**: Customer-facing wallet and points management
- **Mobile Apps**: React Native applications (future)

### 2. Backend Services
- **API Gateway**: Request routing, rate limiting, authentication
- **Core API Service**: Business logic, tenant management
- **Auth Service**: Authentication and authorization
- **Blockchain Service**: Smart contract interactions

### 3. Data Layer
- **PostgreSQL**: Primary database for application data
- **Redis**: Caching and session storage
- **IPFS**: Metadata storage (future)

### 4. Blockchain Layer
- **Smart Contracts**: Loyalty tokens, membership NFTs
- **Multi-Chain Support**: Polygon, BSC, Arbitrum
- **Wallet Integration**: MetaMask, WalletConnect

## Security Considerations
- Multi-layer authentication
- API rate limiting
- Smart contract security audits
- Fraud detection algorithms
- Data encryption at rest and in transit
EOF

# Create deployment guide
cat > docs/deployment/README.md << 'EOF'
# Deployment Guide

## Prerequisites
- Docker and Docker Compose
- Node.js 18+
- PostgreSQL 15+
- Redis 7+

## Environment Setup

### 1. Development
```bash
# Clone repository
git clone <repository-url>
cd blockchain-loyalty-platform

# Setup environment
./tools/scripts/setup-env.sh

# Start development
npm run dev
```

### 2. Staging
```bash
# Build for staging
docker-compose -f docker-compose.staging.yml build

# Deploy to staging
docker-compose -f docker-compose.staging.yml up -d
```

### 3. Production
```bash
# Build for production
docker-compose -f docker-compose.prod.yml build

# Deploy to production
docker-compose -f docker-compose.prod.yml up -d
```

## Environment Variables
See `.env.example` for required environment variables.

## Database Migrations
```bash
# Run migrations
npm run db:migrate

# Seed test data
npm run db:seed
```

## Smart Contract Deployment
```bash
# Deploy to testnet
npm run contracts:deploy:mumbai

# Deploy to mainnet (production only)
npm run contracts:deploy:polygon
```
EOF
```

#### Validation
```bash
# Test all scripts
npm run setup
npm run lint
npm run type-check
npm run build

# Test git hooks
git add .
git commit -m "feat: initial project setup" # Should trigger hooks

# Test environment setup
./tools/scripts/setup-env.sh
```

#### Next Day Setup
- Plan backend module structure
- Review NestJS documentation
- Prepare authentication strategy

---

### Day 5-7: Final Foundation Setup
**Time: 2-3 hours per day**

#### Day 5: Backend Foundation - NestJS Setup

**Objectives**
- Initialize NestJS backend package
- Set up modular architecture
- Configure database connections
- Create basic authentication

**Tasks**

```bash
# Day 5: Initialize backend package
cd packages
mkdir -p backend/src/{modules,common,config}
cd backend

# Initialize NestJS
npm init -y
npm install @nestjs/core @nestjs/common @nestjs/platform-express @nestjs/config @nestjs/jwt @nestjs/passport @nestjs/typeorm typeorm pg bcrypt passport passport-jwt passport-local class-validator class-transformer

npm install -D @nestjs/cli @nestjs/testing @types/bcrypt @types/passport-jwt @types/passport-local

# Create app module structure
# [Detailed NestJS setup code would continue here...]
```

#### Day 6: Authentication System

**Objectives**
- Implement JWT authentication
- Create tenant isolation middleware
- Set up role-based access control
- Create API key management

**Tasks**

```bash
# Day 6: Authentication setup
# [Authentication module creation, JWT setup, middleware, etc.]
```

#### Day 7: Testing & Documentation

**Objectives**
- Write comprehensive tests for foundation
- Update documentation
- Create development workflows
- Prepare for next phase

**Tasks**

```bash
# Day 7: Testing and finalization
# [Test creation, documentation updates, workflow validation]
```

#### Week 1 Completion Checklist
- [ ] Monorepo structure created
- [ ] Development environment configured
- [ ] Database schema designed and implemented
- [ ] Smart contracts created and tested
- [ ] CI/CD pipeline functional
- [ ] Backend foundation established
- [ ] Authentication system working
- [ ] Comprehensive tests written
- [ ] Documentation complete

---

## WEEK 2: CORE BACKEND & SMART CONTRACTS

[Continue with detailed daily breakdowns for weeks 2-13...]

### Quick Reference Commands

**Daily Workflow:**
```bash
# Start working
git pull origin main
docker-compose up -d
npm run dev

# During development
npm run test
npm run lint
git add . && git commit -m "feat: description"

# End of day
git push origin feature-branch
```

**Debugging:**
```bash
# Check services
docker-compose ps
docker-compose logs [service-name]

# Database access
docker exec -it blockchain-loyalty-platform_postgres_1 psql -U loyalty_user -d loyalty_platform

# Smart contract interaction
cd packages/contracts
npx hardhat console --network polygon_mumbai
```

---

*This guide provides exact commands and code for each day. Follow sequentially for best results. Each day builds on the previous day's work.*
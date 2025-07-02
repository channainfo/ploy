# 🤝 Contributing to Ploy

> **Welcome to the Ploy community!** We're excited you're interested in contributing to the future of blockchain-powered loyalty programs.

## 🌟 Ways to Contribute

### 🐛 **Bug Reports & Issues**
Found a bug? We want to hear about it! Please check existing issues first, then create a detailed bug report.

### 💡 **Feature Requests**
Have an idea for a new feature? Start a discussion in our GitHub Discussions or Discord.

### 📝 **Documentation**
Help improve our docs! Fix typos, add examples, or write new guides.

### 💻 **Code Contributions**
Ready to dive into the code? We welcome contributions of all sizes, from small fixes to major features.

### 🎨 **Design & UX**
Help us improve the user experience with better designs, wireframes, or usability feedback.

---

## 🚀 Quick Start for Contributors

### 1. **Fork & Clone**

```bash
# Fork the repository on GitHub, then clone your fork
git clone https://github.com/YOUR_USERNAME/ploy.git
cd ploy

# Add the original repository as upstream
git remote add upstream https://github.com/ploy-org/ploy.git
```

### 2. **Set Up Development Environment**

```bash
# Install dependencies
npm install

# Copy environment template
cp .env.example .env.local

# Set up local blockchain networks (optional for testing)
npm run setup:local-chains

# Start development server
npm run dev
```

### 3. **Create a Feature Branch**

```bash
# Create and switch to a new branch
git checkout -b feature/your-feature-name

# Or for bug fixes
git checkout -b fix/issue-description
```

### 4. **Make Your Changes**

- Write clear, concise commit messages
- Add tests for new functionality
- Update documentation as needed
- Follow our coding standards (see below)

### 5. **Test Your Changes**

```bash
# Run the test suite
npm test

# Run type checking
npm run type-check

# Run linting
npm run lint

# Test with local blockchain
npm run test:blockchain
```

### 6. **Submit a Pull Request**

```bash
# Push your branch to your fork
git push origin feature/your-feature-name

# Create a pull request on GitHub
# - Provide a clear title and description
# - Link any related issues
# - Add screenshots for UI changes
```

---

## 📋 Contribution Guidelines

### **Branch Naming**

Use descriptive branch names:

```bash
feature/nft-marketplace-integration
fix/gas-estimation-error  
docs/api-reference-update
refactor/policy-engine-optimization
```

### **Commit Messages**

Follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:

```bash
feat: add NFT rental functionality
fix: resolve cross-chain bridge timeout issue
docs: update SDK integration examples
test: add unit tests for policy engine
refactor: optimize database queries for analytics
```

### **Pull Request Process**

1. **Keep PRs focused**: One feature or fix per PR
2. **Write clear descriptions**: Explain what changes and why
3. **Add tests**: Include unit tests for new functionality
4. **Update docs**: Keep documentation in sync with code changes
5. **Be responsive**: Address review feedback promptly

---

## 🏗️ Development Setup

### **Prerequisites**

- **Node.js** 18+ and npm
- **Docker** and Docker Compose
- **Git** for version control
- **Rust** (for smart contract development)

### **Environment Variables**

Copy `.env.example` to `.env.local` and configure:

```bash
# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/ploy_dev
REDIS_URL=redis://localhost:6379

# Blockchain Networks (for testing)
SUI_RPC_URL=http://localhost:9000
SOLANA_RPC_URL=http://localhost:8899
POLYGON_RPC_URL=http://localhost:8545

# API Keys (get from respective platforms)
INFURA_PROJECT_ID=your_infura_id
ALCHEMY_API_KEY=your_alchemy_key

# Development
NODE_ENV=development
LOG_LEVEL=debug
```

### **Local Development**

```bash
# Start all services
docker-compose up -d

# Start the development server
npm run dev

# In separate terminals:
npm run dev:api      # Backend API server
npm run dev:web      # Frontend development server
npm run dev:workers  # Background job workers
```

### **Running Tests**

```bash
# Unit tests
npm run test

# Integration tests
npm run test:integration

# E2E tests
npm run test:e2e

# Smart contract tests
npm run test:contracts

# Test coverage
npm run test:coverage
```

---

## 📝 Coding Standards

### **TypeScript/JavaScript**

We use **ESLint** and **Prettier** for code formatting:

```typescript
// ✅ Good
interface LoyaltyProgram {
  id: string;
  name: string;
  pointsPerDollar: number;
  isActive: boolean;
}

const awardPoints = async (
  userId: string,
  amount: number,
  reason: string
): Promise<Transaction> => {
  // Clear, descriptive variable names
  const user = await userService.findById(userId);
  const transaction = await pointsService.award(user, amount, reason);
  
  return transaction;
};

// ❌ Avoid
const ap = (u, a, r) => { /* unclear function */ };
```

### **Smart Contracts**

#### **SUI Move**

```move
// ✅ Good: Clear module and function documentation
/// Module for managing loyalty points on SUI
module loyalty::points {
    use sui::object::{Self, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    
    /// Represents loyalty points for a user
    struct LoyaltyPoints has key, store {
        id: UID,
        amount: u64,
        user: address,
    }
    
    /// Award loyalty points to a user
    public fun award_points(
        amount: u64,
        user: address,
        ctx: &mut TxContext
    ) {
        let points = LoyaltyPoints {
            id: object::new(ctx),
            amount,
            user,
        };
        
        transfer::transfer(points, user);
    }
}
```

#### **Solana Anchor**

```rust
// ✅ Good: Well-documented program
use anchor_lang::prelude::*;

declare_id!("LoyaltyProgramIdHere");

#[program]
pub mod loyalty_program {
    use super::*;
    
    /// Award loyalty points to a user
    pub fn award_points(
        ctx: Context<AwardPoints>,
        amount: u64,
        reason: String,
    ) -> Result<()> {
        let user_account = &mut ctx.accounts.user_account;
        user_account.points += amount;
        user_account.last_activity = Clock::get()?.unix_timestamp;
        
        emit!(PointsAwarded {
            user: ctx.accounts.user.key(),
            amount,
            reason,
        });
        
        Ok(())
    }
}
```

### **Testing Standards**

```typescript
// ✅ Good: Descriptive test names and clear structure
describe('LoyaltyPointsService', () => {
  describe('awardPoints', () => {
    it('should award points to user and create transaction record', async () => {
      // Arrange
      const userId = 'user_123';
      const amount = 100;
      const reason = 'Purchase reward';
      
      // Act
      const result = await loyaltyService.awardPoints(userId, amount, reason);
      
      // Assert
      expect(result.points).toBe(amount);
      expect(result.userId).toBe(userId);
      expect(result.reason).toBe(reason);
    });
    
    it('should throw error when awarding negative points', async () => {
      // Act & Assert
      await expect(
        loyaltyService.awardPoints('user_123', -10, 'Invalid')
      ).rejects.toThrow('Points amount must be positive');
    });
  });
});
```

---

## 🏷️ Issue Labels

We use labels to categorize issues and PRs:

### **Type Labels**
- `bug` - Something isn't working
- `feature` - New feature request  
- `documentation` - Documentation improvements
- `performance` - Performance optimization
- `security` - Security-related changes

### **Priority Labels**
- `priority:critical` - Critical bugs or security issues
- `priority:high` - Important features or fixes
- `priority:medium` - Standard priority
- `priority:low` - Nice-to-have improvements

### **Area Labels**
- `area:blockchain` - Smart contracts and blockchain integration
- `area:api` - Backend API changes
- `area:frontend` - UI/UX changes
- `area:docs` - Documentation updates
- `area:testing` - Test improvements

### **Difficulty Labels**
- `good-first-issue` - Perfect for new contributors
- `help-wanted` - Extra attention needed
- `difficulty:easy` - Should take a few hours
- `difficulty:medium` - May take a few days
- `difficulty:hard` - Complex feature or fix

---

## 🎖️ Recognition & Rewards

### **Contributor Levels**

We recognize contributions with special NFT badges:

#### 🌱 **Newcomer** (First contribution)
- Welcome NFT badge
- Discord contributor role
- Access to contributor channels

#### ⭐ **Regular Contributor** (5+ merged PRs)
- Contributor NFT upgrade
- Early access to new features
- Invitation to monthly contributor calls

#### 🔥 **Core Contributor** (25+ PRs, regular code reviews)
- Core team NFT
- Governance voting rights
- Access to private roadmap discussions

#### 💎 **Legend** (100+ PRs, major feature contributions)
- Legend NFT with special traits
- Platform revenue sharing participation
- Direct input on product direction

### **Special Recognition**

- **🐛 Bug Hunter**: Critical bug discovery and fixes
- **📚 Documentation Master**: Outstanding documentation contributions
- **🎨 Design Champion**: UI/UX improvements and design contributions
- **🤝 Community Builder**: Helping other contributors and community growth

---

## 📞 Getting Help

### **Discord Community**
Join our [Discord server](https://discord.gg/ploy) for:
- General questions and discussions
- Real-time help with development issues
- Community events and announcements

### **GitHub Discussions**
Use [GitHub Discussions](https://github.com/ploy-org/ploy/discussions) for:
- Feature requests and ideas
- Architectural discussions
- Q&A about the platform

### **Office Hours**
- **Weekly Contributor Call**: Fridays at 2 PM UTC
- **Monthly Architecture Review**: First Saturday of each month
- **Quarterly Roadmap Planning**: Public sessions with core team

---

## 📜 Code of Conduct

We are committed to fostering a welcoming and inclusive community. Please read our [Code of Conduct](./CODE_OF_CONDUCT.md) to understand the standards we expect from all contributors.

### **Our Pledge**

In the interest of fostering an open and welcoming environment, we pledge to make participation in our project a harassment-free experience for everyone, regardless of:

- Age, body size, disability, ethnicity, gender identity and expression
- Level of experience, education, socio-economic status
- Nationality, personal appearance, race, religion
- Sexual identity and orientation

---

## 🙏 Thank You

Every contribution, no matter how small, helps make Ploy better for everyone. Whether you're fixing a typo, reporting a bug, or building a major feature, we appreciate your time and effort.

**Ready to contribute?** Start by exploring our [good first issues](https://github.com/ploy-org/ploy/labels/good-first-issue) or join our [Discord community](https://discord.gg/ploy) to introduce yourself!

---

*Happy coding! 🚀*
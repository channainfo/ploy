# 📦 Ploy SDK Guide

> **Complete guide to integrating the Ploy SDK into your applications**

## 🚀 Installation

### **Node.js / JavaScript**

```bash
npm install @ploy/sdk
# or
yarn add @ploy/sdk
# or
pnpm add @ploy/sdk
```

### **Python**

```bash
pip install ploy-sdk
```

### **Go**

```bash
go get github.com/ploy-org/ploy-go-sdk
```

### **PHP**

```bash
composer require ploy/sdk
```

---

## ⚙️ Configuration

### **Basic Setup**

```typescript
import { PloySDK } from '@ploy/sdk';

const ploy = new PloySDK({
  apiKey: 'your-api-key',           // Get from Ploy dashboard
  network: 'sui-mainnet',           // 'sui-mainnet', 'sui-testnet', 'solana-mainnet', etc.
  gasless: true,                    // Enable sponsored transactions
  environment: 'production'         // 'production' or 'development'
});
```

### **Advanced Configuration**

```typescript
const ploy = new PloySDK({
  apiKey: process.env.PLOY_API_KEY,
  network: 'sui-mainnet',
  gasless: true,
  
  // Custom endpoints
  endpoints: {
    api: 'https://api.ploy.io/v1',
    websocket: 'wss://ws.ploy.io'
  },
  
  // Retry configuration
  retry: {
    attempts: 3,
    delay: 1000,
    backoff: 2
  },
  
  // Timeout settings
  timeout: 30000,
  
  // Webhook configuration
  webhooks: {
    secret: process.env.PLOY_WEBHOOK_SECRET,
    endpoints: ['https://yourapp.com/webhooks/ploy']
  }
});
```

---

## 👤 User Management

### **Create User**

```typescript
// Create a new user in the loyalty program
const user = await ploy.users.create({
  externalId: 'user_123',           // Your system's user ID
  email: 'user@example.com',
  metadata: {
    firstName: 'John',
    lastName: 'Doe',
    signupDate: new Date().toISOString()
  }
});
```

### **Get User**

```typescript
// Get user by external ID
const user = await ploy.users.get('user_123');

// Get user by Ploy ID
const user = await ploy.users.getById('ploy_user_id');
```

### **Update User**

```typescript
const updatedUser = await ploy.users.update('user_123', {
  email: 'newemail@example.com',
  metadata: {
    firstName: 'Jane',
    lastName: 'Smith'
  }
});
```

---

## 💰 Points Management

### **Award Points**

```typescript
// Award points to a user
const transaction = await ploy.points.award({
  userId: 'user_123',
  amount: 100,
  reason: 'purchase',              // Reason for awarding points
  metadata: {
    orderId: 'order_456',
    purchaseAmount: 50.00,
    category: 'electronics'
  }
});
```

### **Bulk Award Points**

```typescript
// Award points to multiple users efficiently
const transactions = await ploy.points.awardBatch([
  { userId: 'user_123', amount: 100, reason: 'purchase' },
  { userId: 'user_456', amount: 50, reason: 'referral' },
  { userId: 'user_789', amount: 25, reason: 'social_share' }
]);
```

### **Get Points Balance**

```typescript
// Get user's current points balance
const balance = await ploy.points.getBalance('user_123');
console.log(balance);
// {
//   total: 1500,
//   pending: 50,
//   available: 1450,
//   tier: 'gold'
// }
```

### **Redeem Points**

```typescript
// Redeem points for rewards
const redemption = await ploy.points.redeem({
  userId: 'user_123',
  amount: 500,
  rewardId: 'discount_10_percent',
  metadata: {
    orderId: 'order_789'
  }
});
```

### **Get Transaction History**

```typescript
// Get user's transaction history
const history = await ploy.points.getHistory('user_123', {
  limit: 50,
  offset: 0,
  startDate: '2024-01-01',
  endDate: '2024-12-31',
  type: 'award' // 'award', 'redeem', 'transfer'
});
```

---

## 🎁 Rewards Management

### **Create Reward**

```typescript
const reward = await ploy.rewards.create({
  name: '10% Discount',
  description: 'Get 10% off your next purchase',
  type: 'discount',
  cost: 500,                       // Points required
  value: 10,                       // Discount percentage
  category: 'discount',
  metadata: {
    validFor: 30,                  // Days valid
    minimumPurchase: 50,           // Minimum purchase amount
    excludedCategories: ['sale']
  }
});
```

### **List Available Rewards**

```typescript
// Get rewards available to a user
const rewards = await ploy.rewards.getAvailable('user_123', {
  category: 'discount',
  minCost: 100,
  maxCost: 1000
});
```

---

## 💎 NFT Management

### **Mint NFT**

```typescript
// Mint an achievement NFT
const nft = await ploy.nfts.mint({
  userId: 'user_123',
  template: 'first_purchase',      // NFT template ID
  metadata: {
    name: 'First Purchase Champion',
    description: 'Congratulations on your first purchase!',
    image: 'https://cdn.yourapp.com/nfts/first-purchase.png',
    attributes: [
      { trait_type: 'Achievement', value: 'First Purchase' },
      { trait_type: 'Date Earned', value: new Date().toISOString() },
      { trait_type: 'Rarity', value: 'Common' }
    ]
  }
});
```

### **Get User's NFTs**

```typescript
// Get all NFTs owned by a user
const userNFTs = await ploy.nfts.getByUser('user_123');

// Get specific NFT
const nft = await ploy.nfts.get('nft_id');
```

### **Transfer NFT**

```typescript
// Transfer NFT to another user
const transfer = await ploy.nfts.transfer({
  nftId: 'nft_123',
  fromUserId: 'user_123',
  toUserId: 'user_456',
  gasless: true
});
```

---

## 🏪 Marketplace Operations

### **List NFT for Sale**

```typescript
// List an NFT in the marketplace
const listing = await ploy.marketplace.createListing({
  nftId: 'nft_123',
  price: 1000,                     // Price in loyalty points
  acceptsCrypto: true,
  acceptedTokens: ['SUI', 'SOL'],
  duration: 7                      // Days listing is active
});
```

### **Purchase NFT**

```typescript
// Buy an NFT from marketplace
const purchase = await ploy.marketplace.purchase({
  listingId: 'listing_456',
  buyerId: 'user_789',
  paymentMethod: 'loyalty_points',
  gasless: true
});
```

### **Get Marketplace Listings**

```typescript
// Browse marketplace
const listings = await ploy.marketplace.getListings({
  category: 'achievement',
  minPrice: 100,
  maxPrice: 5000,
  sortBy: 'price_asc',
  limit: 20
});
```

---

## 🌉 Cross-Chain Operations

### **Transfer Points Cross-Chain**

```typescript
// Transfer points from SUI to Solana
const transfer = await ploy.bridge.transferPoints({
  userId: 'user_123',
  amount: 500,
  fromChain: 'sui',
  toChain: 'solana',
  gasless: true
});

// Listen for transfer completion
ploy.bridge.onTransferComplete(transfer.transactionId, (result) => {
  console.log('Transfer completed:', result);
});
```

### **Bridge NFT**

```typescript
// Move NFT to different blockchain
const bridgeOperation = await ploy.bridge.transferNFT({
  nftId: 'nft_123',
  fromChain: 'sui',
  toChain: 'polygon',
  gasless: true
});
```

---

## 📊 Analytics & Insights

### **Get User Analytics**

```typescript
// Get detailed user analytics
const analytics = await ploy.analytics.getUser('user_123', {
  period: '30d',
  metrics: ['points_earned', 'points_redeemed', 'nfts_earned', 'tier_progress']
});
```

### **Get Program Analytics**

```typescript
// Get program-wide analytics
const programAnalytics = await ploy.analytics.getProgram({
  period: '30d',
  metrics: ['total_users', 'active_users', 'points_issued', 'redemption_rate']
});
```

---

## 🎯 Tier Management

### **Get User Tier**

```typescript
// Check user's current tier
const tierInfo = await ploy.tiers.getUserTier('user_123');
console.log(tierInfo);
// {
//   currentTier: 'gold',
//   pointsToNextTier: 2500,
//   nextTier: 'platinum',
//   benefits: ['1.5x multiplier', 'exclusive rewards']
// }
```

### **Update Tier Rules**

```typescript
// Update tier configuration
const tierConfig = await ploy.tiers.updateConfig({
  bronze: { threshold: 0, multiplier: 1.0 },
  silver: { threshold: 1000, multiplier: 1.2 },
  gold: { threshold: 5000, multiplier: 1.5 },
  platinum: { threshold: 15000, multiplier: 2.0 }
});
```

---

## 🔔 Webhooks & Events

### **Set Up Webhooks**

```typescript
// Register webhook endpoints
await ploy.webhooks.register({
  url: 'https://yourapp.com/webhooks/ploy',
  events: ['points.awarded', 'nft.minted', 'tier.upgraded'],
  secret: process.env.WEBHOOK_SECRET
});
```

### **Handle Webhook Events**

```typescript
import express from 'express';
import crypto from 'crypto';

const app = express();
app.use(express.raw({ type: 'application/json' }));

app.post('/webhooks/ploy', (req, res) => {
  const signature = req.headers['x-ploy-signature'];
  const payload = req.body;
  
  // Verify webhook signature
  const expectedSignature = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(payload)
    .digest('hex');
    
  if (signature !== `sha256=${expectedSignature}`) {
    return res.status(401).send('Invalid signature');
  }
  
  const event = JSON.parse(payload);
  
  switch (event.type) {
    case 'points.awarded':
      handlePointsAwarded(event.data);
      break;
    case 'nft.minted':
      handleNFTMinted(event.data);
      break;
    case 'tier.upgraded':
      handleTierUpgrade(event.data);
      break;
  }
  
  res.status(200).send('OK');
});
```

---

## 🔄 Real-time Updates

### **WebSocket Connection**

```typescript
// Connect to real-time updates
const ws = ploy.realtime.connect('user_123');

ws.on('points_awarded', (data) => {
  console.log('Points awarded:', data);
  updateUI(data);
});

ws.on('nft_received', (data) => {
  console.log('NFT received:', data);
  showNFTAnimation(data);
});

ws.on('tier_upgraded', (data) => {
  console.log('Tier upgraded:', data);
  celebrateTierUpgrade(data);
});
```

### **Server-Sent Events**

```typescript
// Alternative: Server-sent events
const eventSource = ploy.events.subscribe('user_123');

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  handleRealtimeUpdate(data);
};
```

---

## 🛡️ Security & Authentication

### **API Key Management**

```typescript
// Validate API key
const isValid = await ploy.auth.validateKey();

// Get key permissions
const permissions = await ploy.auth.getPermissions();
console.log(permissions);
// ['read:users', 'write:points', 'mint:nfts']
```

### **User Authentication**

```typescript
// Create user session
const session = await ploy.auth.createUserSession('user_123', {
  expiresIn: '24h',
  permissions: ['read:profile', 'spend:points']
});

// Verify user session
const isValidSession = await ploy.auth.verifyUserSession(session.token);
```

---

## 🧪 Testing

### **Test Mode**

```typescript
// Initialize SDK in test mode
const ploy = new PloySDK({
  apiKey: 'test_key',
  network: 'sui-testnet',
  environment: 'development'
});

// Create test data
const testUser = await ploy.testing.createTestUser();
const testTransaction = await ploy.testing.awardTestPoints(testUser.id, 100);
```

### **Mock Responses**

```typescript
// Mock SDK for unit testing
import { jest } from '@jest/globals';

jest.mock('@ploy/sdk', () => ({
  PloySDK: jest.fn().mockImplementation(() => ({
    points: {
      award: jest.fn().mockResolvedValue({ id: 'tx_123', points: 100 }),
      getBalance: jest.fn().mockResolvedValue({ total: 1000 })
    },
    nfts: {
      mint: jest.fn().mockResolvedValue({ id: 'nft_123', tokenId: '456' })
    }
  }))
}));
```

---

## 🚨 Error Handling

### **Error Types**

```typescript
import { PloyError, PloyErrorType } from '@ploy/sdk';

try {
  await ploy.points.award({
    userId: 'user_123',
    amount: 100,
    reason: 'purchase'
  });
} catch (error) {
  if (error instanceof PloyError) {
    switch (error.type) {
      case PloyErrorType.USER_NOT_FOUND:
        console.error('User not found');
        break;
      case PloyErrorType.INSUFFICIENT_BALANCE:
        console.error('User has insufficient balance');
        break;
      case PloyErrorType.RATE_LIMIT_EXCEEDED:
        console.error('Rate limit exceeded, retry later');
        break;
      case PloyErrorType.NETWORK_ERROR:
        console.error('Blockchain network error');
        break;
      default:
        console.error('Unknown error:', error.message);
    }
  }
}
```

### **Retry Logic**

```typescript
// Automatic retry with exponential backoff
const result = await ploy.withRetry(
  () => ploy.points.award({ userId: 'user_123', amount: 100, reason: 'purchase' }),
  {
    attempts: 3,
    delay: 1000,
    backoff: 2,
    retryOn: [PloyErrorType.NETWORK_ERROR, PloyErrorType.TIMEOUT]
  }
);
```

---

## 📈 Performance Optimization

### **Batch Operations**

```typescript
// Use batch operations for better performance
const results = await Promise.all([
  ploy.points.awardBatch(pointsToAward),
  ploy.nfts.mintBatch(nftsToMint),
  ploy.analytics.getBatch(analyticsQueries)
]);
```

### **Caching**

```typescript
// Cache frequently accessed data
const cache = new Map();

const getUserBalance = async (userId: string) => {
  const cacheKey = `balance:${userId}`;
  
  if (cache.has(cacheKey)) {
    return cache.get(cacheKey);
  }
  
  const balance = await ploy.points.getBalance(userId);
  cache.set(cacheKey, balance, { ttl: 60000 }); // Cache for 1 minute
  
  return balance;
};
```

---

## 🌍 Multi-Language Support

### **Localization**

```typescript
// Set language for API responses
const ploy = new PloySDK({
  apiKey: 'your-api-key',
  language: 'es', // Spanish
  region: 'MX'    // Mexico
});

// Get localized reward names
const rewards = await ploy.rewards.getAvailable('user_123', {
  language: 'es'
});
```

---

## 📚 Additional Resources

- **[API Reference](../api/api-reference.md)**: Complete REST API documentation
- **[Code Examples](../examples/code-examples-guide.md)**: Real-world implementation examples
- **[Architecture Guide](../architecture/platform-architecture-overview.md)**: Understanding the platform
- **[Discord Community](https://discord.gg/ploy)**: Get help from other developers

---

*Ready to integrate Ploy into your application? Start with our [Quick Start Guide](../examples/code-examples-guide.md) and join our [Discord community](https://discord.gg/ploy) for support! 🚀*
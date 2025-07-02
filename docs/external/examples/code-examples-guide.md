# 💻 Code Examples & Implementation Patterns

> **Real-world examples to help you integrate Ploy into your applications**

## 🚀 Quick Start Examples

### **Basic Loyalty Program Setup**

```typescript
import { PloySDK } from '@ploy/sdk';

// Initialize the Ploy SDK
const ploy = new PloySDK({
  apiKey: process.env.PLOY_API_KEY,
  network: 'sui-mainnet', // or 'sui-testnet' for development
  gasless: true // Enable sponsored transactions
});

// Create a new loyalty program
const createLoyaltyProgram = async () => {
  const program = await ploy.programs.create({
    name: 'Coffee Shop Rewards',
    description: 'Earn points for every coffee purchase',
    pointsPerDollar: 1,
    blockchain: 'sui',
    settings: {
      tierSystem: {
        bronze: { threshold: 0, multiplier: 1.0 },
        silver: { threshold: 1000, multiplier: 1.2 },
        gold: { threshold: 5000, multiplier: 1.5 },
        platinum: { threshold: 10000, multiplier: 2.0 }
      }
    }
  });
  
  console.log('Program created:', program.id);
  return program;
};
```

### **Award Points for Purchases**

```typescript
// Award points when customer makes a purchase
const awardPurchasePoints = async (customerId: string, purchaseAmount: number) => {
  try {
    const transaction = await ploy.points.award({
      userId: customerId,
      amount: Math.floor(purchaseAmount * 1), // 1 point per dollar
      reason: 'purchase',
      metadata: {
        orderId: 'order_123',
        purchaseAmount,
        timestamp: new Date().toISOString()
      }
    });
    
    console.log('Points awarded:', transaction);
    return transaction;
  } catch (error) {
    console.error('Failed to award points:', error);
    throw error;
  }
};
```

### **Mint Achievement NFTs**

```typescript
// Mint an NFT when customer reaches milestone
const mintMilestoneNFT = async (customerId: string, milestone: string) => {
  const nft = await ploy.nfts.mint({
    userId: customerId,
    template: milestone,
    metadata: {
      name: `${milestone} Achievement`,
      description: `Congratulations on reaching ${milestone}!`,
      image: `https://cdn.yourapp.com/nfts/${milestone}.png`,
      attributes: [
        { trait_type: 'Achievement', value: milestone },
        { trait_type: 'Date Earned', value: new Date().toISOString() },
        { trait_type: 'Rarity', value: 'Common' }
      ]
    }
  });
  
  return nft;
};
```

---

## 🏪 E-commerce Integration

### **Shopify Integration Example**

```typescript
// Shopify webhook handler for order completion
import { Request, Response } from 'express';
import crypto from 'crypto';

const shopifyWebhookHandler = async (req: Request, res: Response) => {
  // Verify webhook authenticity
  const hmac = req.get('X-Shopify-Hmac-Sha256');
  const body = req.body;
  const hash = crypto
    .createHmac('sha256', process.env.SHOPIFY_WEBHOOK_SECRET)
    .update(body, 'utf8')
    .digest('base64');
    
  if (hash !== hmac) {
    return res.status(401).send('Unauthorized');
  }
  
  const order = JSON.parse(body);
  
  // Award points for the purchase
  await awardPurchasePoints(
    order.customer.id.toString(),
    parseFloat(order.total_price)
  );
  
  // Check for milestones
  const customerPoints = await ploy.points.getBalance(order.customer.id.toString());
  
  if (customerPoints.total === 1000) {
    await mintMilestoneNFT(order.customer.id.toString(), 'first_1000_points');
  }
  
  res.status(200).send('OK');
};
```

### **WooCommerce Integration**

```php
<?php
// WordPress/WooCommerce hook for order completion
add_action('woocommerce_order_status_completed', 'award_loyalty_points');

function award_loyalty_points($order_id) {
    $order = wc_get_order($order_id);
    $customer_id = $order->get_customer_id();
    $total = $order->get_total();
    
    // Call Ploy API to award points
    $response = wp_remote_post('https://api.ploy.io/v1/points/award', [
        'headers' => [
            'Authorization' => 'Bearer ' . get_option('ploy_api_key'),
            'Content-Type' => 'application/json'
        ],
        'body' => json_encode([
            'userId' => (string) $customer_id,
            'amount' => floor($total),
            'reason' => 'purchase',
            'metadata' => [
                'orderId' => $order_id,
                'purchaseAmount' => $total
            ]
        ])
    ]);
    
    if (is_wp_error($response)) {
        error_log('Ploy API error: ' . $response->get_error_message());
    }
}
?>
```

---

## 🎮 Gaming Integration

### **Unity Game Integration**

```csharp
using UnityEngine;
using System.Collections;
using System.Collections.Generic;

public class PloyGameIntegration : MonoBehaviour
{
    private string apiKey = "your-api-key";
    private string apiUrl = "https://api.ploy.io/v1";
    
    // Award points for game achievements
    public IEnumerator AwardAchievementPoints(string playerId, string achievement, int points)
    {
        var requestData = new Dictionary<string, object>
        {
            ["userId"] = playerId,
            ["amount"] = points,
            ["reason"] = achievement,
            ["metadata"] = new Dictionary<string, object>
            {
                ["gameLevel"] = GameManager.currentLevel,
                ["achievementType"] = achievement,
                ["timestamp"] = System.DateTime.UtcNow.ToString("O")
            }
        };
        
        string jsonData = JsonUtility.ToJson(requestData);
        
        using (UnityWebRequest request = UnityWebRequest.Post($"{apiUrl}/points/award", jsonData))
        {
            request.SetRequestHeader("Authorization", $"Bearer {apiKey}");
            request.SetRequestHeader("Content-Type", "application/json");
            
            yield return request.SendWebRequest();
            
            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log("Points awarded successfully!");
                // Update UI or trigger celebration animation
                ShowPointsEarnedUI(points);
            }
            else
            {
                Debug.LogError($"Failed to award points: {request.error}");
            }
        }
    }
    
    // Mint NFT for rare achievements
    public IEnumerator MintAchievementNFT(string playerId, string achievementType)
    {
        var nftData = new Dictionary<string, object>
        {
            ["userId"] = playerId,
            ["template"] = achievementType,
            ["metadata"] = new Dictionary<string, object>
            {
                ["name"] = $"{achievementType} Master",
                ["description"] = $"Rare achievement in Epic Quest Game",
                ["image"] = $"https://game-assets.com/nfts/{achievementType}.png",
                ["attributes"] = new object[]
                {
                    new { trait_type = "Game", value = "Epic Quest" },
                    new { trait_type = "Achievement", value = achievementType },
                    new { trait_type = "Rarity", value = "Legendary" }
                }
            }
        };
        
        string jsonData = JsonUtility.ToJson(nftData);
        
        using (UnityWebRequest request = UnityWebRequest.Post($"{apiUrl}/nfts/mint", jsonData))
        {
            request.SetRequestHeader("Authorization", $"Bearer {apiKey}");
            request.SetRequestHeader("Content-Type", "application/json");
            
            yield return request.SendWebRequest();
            
            if (request.result == UnityWebRequest.Result.Success)
            {
                Debug.Log("NFT minted successfully!");
                ShowNFTEarnedUI(achievementType);
            }
        }
    }
}
```

---

## 💰 SaaS Platform Integration

### **API Usage Tracking**

```typescript
// Express middleware to track API usage
import { Request, Response, NextFunction } from 'express';

interface AuthenticatedRequest extends Request {
  user?: {
    id: string;
    tier: string;
  };
}

const trackAPIUsage = async (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
  if (!req.user) {
    return next();
  }
  
  // Track the API call
  try {
    await ploy.points.award({
      userId: req.user.id,
      amount: 1, // 1 point per API call
      reason: 'api_usage',
      metadata: {
        endpoint: req.path,
        method: req.method,
        userTier: req.user.tier,
        timestamp: new Date().toISOString()
      }
    });
  } catch (error) {
    console.error('Failed to track API usage:', error);
    // Don't block the request if loyalty tracking fails
  }
  
  next();
};

// Apply middleware to API routes
app.use('/api', trackAPIUsage);
```

### **Feature Adoption Rewards**

```typescript
// Track when users adopt new features
const trackFeatureAdoption = async (userId: string, featureName: string) => {
  try {
    // Check if this is the first time using this feature
    const featureUsage = await ploy.analytics.getFeatureUsage(userId, featureName);
    
    if (featureUsage.firstTime) {
      // Award bonus points for trying new feature
      await ploy.points.award({
        userId,
        amount: 50,
        reason: 'feature_adoption',
        metadata: {
          featureName,
          adoptionDate: new Date().toISOString()
        }
      });
      
      // Mint NFT for power users who adopt many features
      const totalFeaturesUsed = await ploy.analytics.getTotalFeaturesUsed(userId);
      if (totalFeaturesUsed >= 10) {
        await mintMilestoneNFT(userId, 'power_user');
      }
    }
  } catch (error) {
    console.error('Failed to track feature adoption:', error);
  }
};
```

---

## 🏥 Healthcare Integration (HIPAA Compliant)

### **Anonymous Wellness Tracking**

```typescript
// Healthcare wellness tracking with privacy protection
const trackWellnessGoal = async (anonymousUserId: string, goalType: string) => {
  // Use anonymous ID that can't be traced back to personal health info
  const hashedUserId = crypto
    .createHash('sha256')
    .update(anonymousUserId + process.env.HEALTH_SALT)
    .digest('hex');
  
  try {
    await ploy.points.award({
      userId: hashedUserId,
      amount: 25,
      reason: 'wellness_goal_completion',
      metadata: {
        goalType: goalType, // e.g., 'exercise', 'nutrition', 'sleep'
        completionDate: new Date().toISOString(),
        // No personal health information stored
      }
    });
    
    // Mint wellness achievement NFTs without health data
    if (goalType === 'exercise' && await checkExerciseStreak(hashedUserId)) {
      await ploy.nfts.mint({
        userId: hashedUserId,
        template: 'fitness_champion',
        metadata: {
          name: 'Fitness Champion',
          description: 'Completed fitness goals consistently',
          image: 'https://wellness-nfts.com/fitness-champion.png'
        }
      });
    }
  } catch (error) {
    console.error('Failed to track wellness goal:', error);
  }
};
```

---

## 🌐 Multi-Chain Examples

### **Cross-Chain Point Transfer**

```typescript
// Transfer points from SUI to Solana
const transferPointsCrossChain = async (
  userId: string,
  amount: number,
  fromChain: 'sui' | 'solana' | 'polygon',
  toChain: 'sui' | 'solana' | 'polygon'
) => {
  try {
    const transfer = await ploy.bridge.transferPoints({
      userId,
      amount,
      fromChain,
      toChain,
      gasless: true // Platform sponsors the bridge transaction
    });
    
    console.log('Cross-chain transfer initiated:', transfer.transactionId);
    
    // Listen for transfer completion
    ploy.bridge.onTransferComplete(transfer.transactionId, (result) => {
      console.log('Transfer completed:', result);
      // Notify user of successful transfer
    });
    
    return transfer;
  } catch (error) {
    console.error('Cross-chain transfer failed:', error);
    throw error;
  }
};
```

### **Multi-Chain NFT Marketplace**

```typescript
// List NFT for sale across multiple chains
const listNFTForSale = async (nftId: string, price: number, acceptedChains: string[]) => {
  const listing = await ploy.marketplace.createListing({
    nftId,
    price,
    acceptedPaymentMethods: {
      loyaltyPoints: true,
      cryptocurrencies: acceptedChains
    },
    crossChainEnabled: true
  });
  
  return listing;
};

// Purchase NFT with loyalty points from any chain
const purchaseNFTWithPoints = async (listingId: string, buyerUserId: string, preferredChain: string) => {
  const purchase = await ploy.marketplace.purchase({
    listingId,
    buyerId: buyerUserId,
    paymentMethod: 'loyalty_points',
    preferredChain,
    gasless: true
  });
  
  return purchase;
};
```

---

## 🔧 Advanced Integration Patterns

### **Event-Driven Architecture**

```typescript
// Set up webhooks for real-time updates
const setupWebhooks = () => {
  // Points awarded webhook
  ploy.webhooks.register('points.awarded', async (event) => {
    const { userId, amount, reason } = event.data;
    
    // Send push notification to user
    await sendPushNotification(userId, {
      title: 'Points Earned!',
      message: `You earned ${amount} points for ${reason}`,
      data: { points: amount }
    });
    
    // Update user's tier if threshold reached
    await checkTierUpgrade(userId);
  });
  
  // NFT minted webhook
  ploy.webhooks.register('nft.minted', async (event) => {
    const { userId, nftId, metadata } = event.data;
    
    // Share achievement on social media (with user permission)
    await shareAchievement(userId, metadata);
    
    // Award bonus points for earning NFTs
    await ploy.points.award({
      userId,
      amount: 10,
      reason: 'nft_collection_bonus'
    });
  });
};
```

### **Batch Operations for Performance**

```typescript
// Efficiently process large volumes of transactions
const processBatchRewards = async (transactions: Array<{userId: string, amount: number, reason: string}>) => {
  try {
    // Use batch API for better performance
    const batchResult = await ploy.points.awardBatch(transactions);
    
    console.log(`Processed ${batchResult.successful} transactions`);
    
    if (batchResult.failed.length > 0) {
      console.error('Failed transactions:', batchResult.failed);
      // Retry failed transactions individually
      for (const failed of batchResult.failed) {
        await retryTransaction(failed);
      }
    }
    
    return batchResult;
  } catch (error) {
    console.error('Batch processing failed:', error);
    throw error;
  }
};
```

---

## 🔍 Testing Examples

### **Unit Tests**

```typescript
import { PloySDK } from '@ploy/sdk';
import { jest } from '@jest/globals';

// Mock the Ploy SDK for testing
jest.mock('@ploy/sdk');

describe('Loyalty Integration', () => {
  let ploy: jest.Mocked<PloySDK>;
  
  beforeEach(() => {
    ploy = new PloySDK({
      apiKey: 'test-key',
      network: 'sui-testnet'
    }) as jest.Mocked<PloySDK>;
  });
  
  it('should award points for purchase', async () => {
    // Arrange
    const mockTransaction = { id: 'tx_123', points: 100 };
    ploy.points.award.mockResolvedValue(mockTransaction);
    
    // Act
    const result = await awardPurchasePoints('user_123', 100);
    
    // Assert
    expect(ploy.points.award).toHaveBeenCalledWith({
      userId: 'user_123',
      amount: 100,
      reason: 'purchase',
      metadata: expect.objectContaining({
        purchaseAmount: 100
      })
    });
    expect(result).toEqual(mockTransaction);
  });
  
  it('should mint NFT for milestone', async () => {
    // Arrange
    const mockNFT = { id: 'nft_123', tokenId: '456' };
    ploy.nfts.mint.mockResolvedValue(mockNFT);
    
    // Act
    const result = await mintMilestoneNFT('user_123', 'first_purchase');
    
    // Assert
    expect(ploy.nfts.mint).toHaveBeenCalledWith({
      userId: 'user_123',
      template: 'first_purchase',
      metadata: expect.objectContaining({
        name: 'first_purchase Achievement'
      })
    });
    expect(result).toEqual(mockNFT);
  });
});
```

### **Integration Tests**

```typescript
// Test with actual testnet
describe('Integration Tests', () => {
  const ploy = new PloySDK({
    apiKey: process.env.PLOY_TEST_API_KEY,
    network: 'sui-testnet'
  });
  
  it('should complete end-to-end flow', async () => {
    // Create test user
    const user = await ploy.users.create({
      externalId: 'test_user_' + Date.now(),
      email: 'test@example.com'
    });
    
    // Award points
    const transaction = await ploy.points.award({
      userId: user.id,
      amount: 100,
      reason: 'test_purchase'
    });
    
    expect(transaction.points).toBe(100);
    
    // Check balance
    const balance = await ploy.points.getBalance(user.id);
    expect(balance.total).toBeGreaterThanOrEqual(100);
    
    // Mint NFT
    const nft = await ploy.nfts.mint({
      userId: user.id,
      template: 'test_achievement',
      metadata: {
        name: 'Test Achievement',
        description: 'Test NFT'
      }
    });
    
    expect(nft.tokenId).toBeDefined();
  });
});
```

---

## 📚 Additional Resources

- **[SDK Reference](../integrations/sdk-integration-guide.md)**: Complete API documentation
- **[Architecture Guide](../architecture/platform-architecture-overview.md)**: Understanding the platform
- **[Contributing Guide](../contributing/CONTRIBUTING.md)**: How to contribute to Ploy
- **[Discord Community](https://discord.gg/ploy)**: Get help from other developers

---

*Ready to start building? Choose an example that matches your use case and start integrating Ploy into your application today! 🚀*
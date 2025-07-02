# Edge Cases & Advanced Scenarios Handling

## Overview

This document details how the Blockchain Loyalty Platform handles complex edge cases, ensuring system reliability and flexibility across various business scenarios.

## Transaction Edge Cases

### 1. Split Transactions

#### Multi-Payment Method Orders
```typescript
interface SplitPaymentTransaction {
  orderId: string;
  totalAmount: number;
  payments: Payment[];
  pointsCalculation: 'total' | 'eligible_only' | 'weighted';
}

class SplitPaymentHandler {
  async processPointsForSplitPayment(
    transaction: SplitPaymentTransaction
  ): Promise<PointsResult> {
    switch (transaction.pointsCalculation) {
      case 'total':
        // Award points for total amount regardless of payment method
        return this.calculatePointsForTotal(transaction.totalAmount);
      
      case 'eligible_only':
        // Only award points for eligible payment methods
        const eligibleAmount = transaction.payments
          .filter(p => this.isEligiblePaymentMethod(p.method))
          .reduce((sum, p) => sum + p.amount, 0);
        return this.calculatePoints(eligibleAmount);
      
      case 'weighted':
        // Different point rates for different payment methods
        return this.calculateWeightedPoints(transaction.payments);
    }
  }
  
  private calculateWeightedPoints(payments: Payment[]): PointsResult {
    const points = payments.reduce((total, payment) => {
      const rate = this.getPointsRate(payment.method);
      return total + (payment.amount * rate);
    }, 0);
    
    return {
      basePoints: Math.floor(points),
      bonusPoints: 0,
      metadata: {
        calculation: 'weighted',
        breakdown: payments.map(p => ({
          method: p.method,
          amount: p.amount,
          rate: this.getPointsRate(p.method),
          points: p.amount * this.getPointsRate(p.method)
        }))
      }
    };
  }
}
```

#### Example: Gift Card + Credit Card Payment
```json
{
  "orderId": "ORD-123456",
  "totalAmount": 150.00,
  "payments": [
    {
      "method": "gift_card",
      "amount": 50.00,
      "eligible": false
    },
    {
      "method": "credit_card",
      "amount": 100.00,
      "eligible": true
    }
  ],
  "pointsCalculation": "eligible_only",
  "result": {
    "pointsEarned": 100,
    "breakdown": "Points earned only for $100 credit card payment"
  }
}
```

### 2. Partial Order Fulfillment

#### Incremental Point Awards
```typescript
class PartialFulfillmentHandler {
  async handlePartialShipment(
    order: Order,
    shipment: Shipment
  ): Promise<void> {
    // Calculate points for shipped items only
    const shippedValue = this.calculateShippedValue(
      order,
      shipment
    );
    
    const pointsToAward = Math.floor(
      shippedValue * order.pointsRate
    );
    
    // Award points with pending status
    await this.awardPoints({
      memberId: order.memberId,
      orderId: order.id,
      amount: pointsToAward,
      status: 'pending',
      metadata: {
        fulfillmentType: 'partial',
        shipmentId: shipment.id,
        shippedItems: shipment.items
      }
    });
    
    // Update order tracking
    await this.updateOrderTracking(order.id, {
      shippedValue,
      pointsAwarded: pointsToAward,
      remainingValue: order.totalAmount - shippedValue
    });
  }
  
  async handleFinalShipment(
    order: Order,
    finalShipment: Shipment
  ): Promise<void> {
    // Calculate remaining points
    const remainingPoints = order.totalExpectedPoints - 
      order.pointsAlreadyAwarded;
    
    // Award final points and confirm all pending
    await this.finalizeOrderPoints(order.id, remainingPoints);
  }
}
```

### 3. Cross-Border Transactions

#### Multi-Currency Handling
```typescript
interface CrossBorderTransaction {
  originalCurrency: string;
  originalAmount: number;
  settlementCurrency: string;
  settlementAmount: number;
  exchangeRate: number;
  fees: {
    currency: string;
    amount: number;
  }[];
}

class CrossBorderHandler {
  async calculatePointsForCrossBorder(
    transaction: CrossBorderTransaction,
    policy: CrossBorderPolicy
  ): Promise<PointsCalculation> {
    let baseAmount: number;
    
    switch (policy.calculationBasis) {
      case 'original':
        // Use original currency amount
        baseAmount = transaction.originalAmount;
        break;
      
      case 'settlement':
        // Use settlement currency amount
        baseAmount = transaction.settlementAmount;
        break;
      
      case 'net':
        // Deduct fees from settlement amount
        const totalFees = await this.convertFeesToSettlement(
          transaction.fees,
          transaction.settlementCurrency
        );
        baseAmount = transaction.settlementAmount - totalFees;
        break;
    }
    
    // Apply currency-specific multipliers
    const multiplier = this.getCurrencyMultiplier(
      transaction.originalCurrency
    );
    
    return {
      baseAmount,
      points: Math.floor(baseAmount * policy.pointsRate * multiplier),
      metadata: {
        originalCurrency: transaction.originalCurrency,
        exchangeRate: transaction.exchangeRate,
        multiplier
      }
    };
  }
}
```

## Cancellation & Refund Edge Cases

### 1. Partial Cancellations

#### Item-Level Cancellation with Dependencies
```typescript
class PartialCancellationHandler {
  async processCancellation(
    order: Order,
    cancellationRequest: CancellationRequest
  ): Promise<CancellationResult> {
    const result: CancellationResult = {
      refundAmount: 0,
      pointsToRevoke: 0,
      affectedItems: [],
      dependencies: []
    };
    
    for (const item of cancellationRequest.items) {
      // Check item cancellation policy
      const policy = await this.getItemPolicy(item.id);
      
      if (policy.type === 'non_cancellable') {
        result.errors.push({
          itemId: item.id,
          reason: 'Item cannot be cancelled',
          policy: policy.name
        });
        continue;
      }
      
      // Check dependencies
      const dependencies = await this.checkDependencies(
        order,
        item
      );
      
      if (dependencies.length > 0) {
        result.dependencies.push({
          itemId: item.id,
          dependentItems: dependencies,
          action: 'requires_confirmation'
        });
      }
      
      // Calculate refund and points
      const cancellationCalc = await this.calculateCancellation(
        item,
        policy,
        order.orderDate
      );
      
      result.refundAmount += cancellationCalc.refund;
      result.pointsToRevoke += cancellationCalc.points;
      result.affectedItems.push(item.id);
    }
    
    return result;
  }
}
```

#### Bundle Cancellation Logic
```json
{
  "cancellationType": "BUNDLE_PARTIAL",
  "bundle": {
    "id": "BUNDLE-001",
    "items": [
      {
        "id": "ITEM-A",
        "type": "primary",
        "price": 100
      },
      {
        "id": "ITEM-B",
        "type": "addon",
        "price": 20,
        "dependsOn": "ITEM-A"
      }
    ]
  },
  "cancellationRules": {
    "primaryCancelled": {
      "action": "cancel_all_dependents",
      "refund": "full_bundle_refund",
      "pointsHandling": "revoke_all"
    },
    "addonCancelled": {
      "action": "allow_standalone",
      "refund": "addon_price_only",
      "pointsHandling": "proportional"
    }
  }
}
```

### 2. Time-Sensitive Refunds

#### Service Date Proximity Handling
```typescript
class TimeSensitiveRefundHandler {
  async calculateRefund(
    booking: Booking,
    cancellationDate: Date
  ): Promise<RefundCalculation> {
    const hoursUntilService = this.getHoursUntil(
      booking.serviceDate,
      cancellationDate
    );
    
    // Dynamic refund calculation based on time
    const refundCurve = this.getRefundCurve(booking.type);
    const refundPercentage = refundCurve.calculate(hoursUntilService);
    
    // Special handling for last-minute cancellations
    if (hoursUntilService < 24) {
      return this.handleLastMinuteCancellation(booking, {
        allowRebooking: true,
        rebookingWindow: 90, // days
        creditInsteadOfRefund: true,
        creditBonus: 10 // 10% bonus for accepting credit
      });
    }
    
    // Weather/emergency exceptions
    if (await this.checkForExceptions(booking, cancellationDate)) {
      return {
        refundPercentage: 100,
        pointsRevoked: false,
        reason: 'emergency_exception',
        fee: 0
      };
    }
    
    return {
      refundPercentage,
      pointsRevoked: refundPercentage < 100,
      fee: this.calculateCancellationFee(
        booking.amount,
        refundPercentage
      )
    };
  }
}
```

## Points System Edge Cases

### 1. Points Expiration Scenarios

#### Partial Points Expiration
```typescript
class PointsExpirationHandler {
  async handleExpiration(memberId: string): Promise<ExpirationResult> {
    const expiringBatches = await this.getExpiringPoints(memberId);
    const activeTransactions = await this.getPendingRedemptions(memberId);
    
    for (const batch of expiringBatches) {
      // Check if points are locked in pending redemptions
      const lockedPoints = activeTransactions
        .filter(t => t.pointsBatch === batch.id)
        .reduce((sum, t) => sum + t.points, 0);
      
      const expirablePoints = batch.points - lockedPoints;
      
      if (expirablePoints > 0) {
        // Expire only unlocked points
        await this.expirePoints({
          memberId,
          batchId: batch.id,
          amount: expirablePoints,
          notification: {
            type: 'partial_expiration',
            locked: lockedPoints,
            expired: expirablePoints
          }
        });
      }
      
      // Extend expiration for locked points
      if (lockedPoints > 0) {
        await this.extendExpiration(batch.id, {
          points: lockedPoints,
          reason: 'locked_in_redemption',
          extensionDays: 30
        });
      }
    }
  }
}
```

#### FIFO vs LIFO Expiration
```typescript
enum ExpirationStrategy {
  FIFO = 'first_in_first_out',
  LIFO = 'last_in_first_out',
  NEAREST_EXPIRY = 'nearest_expiry_first',
  HIGHEST_VALUE = 'highest_value_first'
}

class PointsRedemptionStrategy {
  async selectPointsForRedemption(
    memberId: string,
    requiredPoints: number,
    strategy: ExpirationStrategy
  ): Promise<PointsBatch[]> {
    const availableBatches = await this.getAvailablePoints(memberId);
    
    // Sort based on strategy
    const sorted = this.sortByStrategy(availableBatches, strategy);
    
    // Select points maintaining batch integrity
    const selected: PointsBatch[] = [];
    let accumulated = 0;
    
    for (const batch of sorted) {
      if (accumulated >= requiredPoints) break;
      
      const needed = requiredPoints - accumulated;
      const useFromBatch = Math.min(batch.available, needed);
      
      selected.push({
        batchId: batch.id,
        points: useFromBatch,
        expiryDate: batch.expiryDate
      });
      
      accumulated += useFromBatch;
    }
    
    return selected;
  }
}
```

### 2. Negative Balance Scenarios

#### Handling Over-Redemption
```typescript
class NegativeBalanceHandler {
  async handleNegativeBalance(
    memberId: string,
    shortage: number
  ): Promise<Resolution> {
    const member = await this.getMember(memberId);
    const policy = await this.getNegativeBalancePolicy(member.tier);
    
    switch (policy.action) {
      case 'allow_temporary':
        // Allow negative balance with conditions
        return {
          allowed: true,
          temporaryCredit: shortage,
          repaymentPeriod: 30, // days
          restrictions: ['no_redemptions_until_positive']
        };
      
      case 'partial_fulfillment':
        // Fulfill what's possible
        return {
          allowed: false,
          alternativeAction: 'partial_redemption',
          availablePoints: member.balance
        };
      
      case 'defer_to_next_earning':
        // Queue redemption for next earning
        return {
          allowed: false,
          alternativeAction: 'queue_redemption',
          estimatedFulfillment: await this.estimateNextEarning(memberId)
        };
    }
  }
}
```

## System Integration Edge Cases

### 1. Network Failures

#### Blockchain Transaction Failures
```typescript
class BlockchainFailureHandler {
  async handleTransactionFailure(
    transaction: BlockchainTransaction,
    error: Error
  ): Promise<void> {
    const failureType = this.categorizeFailure(error);
    
    switch (failureType) {
      case 'insufficient_gas':
        // Retry with higher gas
        await this.retryWithGasAdjustment(transaction, {
          gasMultiplier: 1.5,
          maxRetries: 3
        });
        break;
      
      case 'network_congestion':
        // Queue for later processing
        await this.queueTransaction(transaction, {
          priority: 'low',
          retryAfter: 300 // 5 minutes
        });
        break;
      
      case 'nonce_conflict':
        // Reset nonce and retry
        await this.resetNonceAndRetry(transaction);
        break;
      
      case 'contract_error':
        // Log and notify admin
        await this.handleContractError(transaction, error);
        // Fallback to centralized processing
        await this.processOffChain(transaction);
        break;
    }
  }
  
  async processOffChain(transaction: Transaction): Promise<void> {
    // Store in database with pending blockchain sync
    await this.db.pendingBlockchainSync.create({
      transaction,
      reason: 'blockchain_failure',
      createdAt: new Date(),
      retryCount: 0
    });
    
    // Process points in database
    await this.processPointsCentralized(transaction);
    
    // Schedule blockchain sync
    await this.scheduleSyncJob(transaction.id);
  }
}
```

### 2. Webhook Delivery Failures

#### Intelligent Retry with Circuit Breaker
```typescript
class WebhookDeliveryService {
  async deliverWithCircuitBreaker(
    endpoint: WebhookEndpoint,
    event: Event
  ): Promise<DeliveryResult> {
    const circuitBreaker = this.getCircuitBreaker(endpoint.id);
    
    if (circuitBreaker.isOpen()) {
      // Circuit is open, skip delivery
      return {
        status: 'skipped',
        reason: 'circuit_breaker_open',
        nextRetry: circuitBreaker.nextAttempt()
      };
    }
    
    try {
      const result = await this.attemptDelivery(endpoint, event);
      circuitBreaker.recordSuccess();
      return result;
      
    } catch (error) {
      circuitBreaker.recordFailure();
      
      if (this.isRetriableError(error)) {
        return await this.scheduleRetry(endpoint, event, {
          attempt: event.deliveryAttempts + 1,
          backoff: this.calculateBackoff(event.deliveryAttempts),
          maxAttempts: endpoint.maxRetries
        });
      }
      
      // Non-retriable error
      return {
        status: 'failed',
        error: error.message,
        action: 'manual_intervention_required'
      };
    }
  }
  
  private calculateBackoff(attempt: number): number {
    // Exponential backoff with jitter
    const baseDelay = Math.pow(2, attempt) * 1000;
    const jitter = Math.random() * 1000;
    return Math.min(baseDelay + jitter, 3600000); // Max 1 hour
  }
}
```

## Data Consistency Edge Cases

### 1. Distributed Transaction Management

#### Two-Phase Commit for Critical Operations
```typescript
class DistributedTransactionManager {
  async executeDistributedTransaction(
    operations: Operation[]
  ): Promise<TransactionResult> {
    const transactionId = this.generateTransactionId();
    const participants: Participant[] = [];
    
    try {
      // Phase 1: Prepare
      for (const operation of operations) {
        const participant = await this.prepareOperation(
          operation,
          transactionId
        );
        participants.push(participant);
      }
      
      // Check all prepared successfully
      const allPrepared = participants.every(p => p.prepared);
      
      if (!allPrepared) {
        throw new Error('Prepare phase failed');
      }
      
      // Phase 2: Commit
      await Promise.all(
        participants.map(p => this.commitOperation(p))
      );
      
      return { success: true, transactionId };
      
    } catch (error) {
      // Rollback all participants
      await this.rollbackTransaction(participants);
      throw error;
    }
  }
  
  async handleInconsistency(
    detection: InconsistencyDetection
  ): Promise<void> {
    // Log inconsistency
    await this.logInconsistency(detection);
    
    // Determine source of truth
    const sourceOfTruth = await this.determineSourceOfTruth(
      detection.sources
    );
    
    // Reconcile data
    await this.reconcileData(detection, sourceOfTruth);
    
    // Notify relevant systems
    await this.notifyInconsistency(detection);
  }
}
```

### 2. Race Condition Prevention

#### Optimistic Locking with Retry
```typescript
class OptimisticLockingService {
  async updateWithOptimisticLock<T>(
    entity: T & { version: number },
    updates: Partial<T>,
    maxRetries: number = 3
  ): Promise<T> {
    let attempt = 0;
    
    while (attempt < maxRetries) {
      try {
        // Load current version
        const current = await this.load(entity.id);
        
        if (current.version !== entity.version) {
          // Version mismatch, reload and retry
          entity = current;
          attempt++;
          continue;
        }
        
        // Apply updates with version increment
        const updated = {
          ...current,
          ...updates,
          version: current.version + 1
        };
        
        // Conditional update based on version
        const result = await this.conditionalUpdate(
          entity.id,
          updated,
          current.version
        );
        
        if (result.affected === 0) {
          // Another process updated, retry
          attempt++;
          continue;
        }
        
        return updated;
        
      } catch (error) {
        if (attempt >= maxRetries - 1) throw error;
        attempt++;
      }
    }
    
    throw new Error('Max retries exceeded for optimistic lock');
  }
}
```

## Performance Edge Cases

### 1. Bulk Operations

#### Batch Processing with Rate Limiting
```typescript
class BulkOperationHandler {
  async processBulkPointsAward(
    awards: PointsAward[],
    options: BulkOptions
  ): Promise<BulkResult> {
    const chunks = this.chunkArray(awards, options.chunkSize || 100);
    const results: ProcessedChunk[] = [];
    
    for (const [index, chunk] of chunks.entries()) {
      try {
        // Process chunk with rate limiting
        await this.rateLimiter.acquire();
        
        const chunkResult = await this.processChunk(chunk, {
          parallel: options.parallel || 10,
          timeout: options.timeout || 30000
        });
        
        results.push(chunkResult);
        
        // Progress callback
        if (options.onProgress) {
          options.onProgress({
            processed: (index + 1) * chunk.length,
            total: awards.length,
            errors: chunkResult.errors
          });
        }
        
        // Prevent overwhelming the system
        if (index < chunks.length - 1) {
          await this.delay(options.delayBetweenChunks || 100);
        }
        
      } catch (error) {
        // Handle chunk failure
        if (options.continueOnError) {
          results.push({
            chunk: index,
            success: false,
            error: error.message
          });
        } else {
          throw error;
        }
      }
    }
    
    return this.aggregateResults(results);
  }
}
```

### 2. Real-time Calculations

#### Caching Strategy for Complex Calculations
```typescript
class RealTimeCalculationService {
  async calculateMemberBenefits(
    memberId: string,
    context: CalculationContext
  ): Promise<Benefits> {
    // Check cache first
    const cacheKey = this.generateCacheKey(memberId, context);
    const cached = await this.cache.get(cacheKey);
    
    if (cached && !this.isStale(cached, context)) {
      return cached.value;
    }
    
    // Use read-through cache pattern
    return await this.cacheAside(cacheKey, async () => {
      // Perform complex calculation
      const benefits = await this.performCalculation(
        memberId,
        context
      );
      
      // Cache with smart TTL
      const ttl = this.calculateTTL(benefits, context);
      
      return {
        value: benefits,
        ttl,
        metadata: {
          calculated: new Date(),
          version: this.calculationVersion
        }
      };
    });
  }
  
  private calculateTTL(
    benefits: Benefits,
    context: CalculationContext
  ): number {
    // Dynamic TTL based on volatility
    if (context.hasActiveCampaign) {
      return 300; // 5 minutes for active campaigns
    }
    
    if (benefits.nearingTierThreshold) {
      return 600; // 10 minutes when close to tier change
    }
    
    return 3600; // 1 hour for stable members
  }
}
```

## Compliance & Regulatory Edge Cases

### 1. Geographic Restrictions

#### Location-Based Policy Enforcement
```typescript
class GeographicComplianceService {
  async enforceGeographicRestrictions(
    transaction: Transaction,
    location: GeoLocation
  ): Promise<ComplianceResult> {
    const jurisdiction = await this.getJurisdiction(location);
    const restrictions = await this.getRestrictions(jurisdiction);
    
    // Check transaction compliance
    for (const restriction of restrictions) {
      if (restriction.type === 'max_transaction_amount') {
        if (transaction.amount > restriction.value) {
          return {
            allowed: false,
            reason: 'exceeds_jurisdiction_limit',
            maxAllowed: restriction.value
          };
        }
      }
      
      if (restriction.type === 'restricted_categories') {
        if (restriction.categories.includes(transaction.category)) {
          return {
            allowed: false,
            reason: 'category_restricted_in_jurisdiction',
            alternativeAction: 'remove_restricted_items'
          };
        }
      }
    }
    
    return { allowed: true };
  }
}
```

### 2. Data Privacy Compliance

#### Right to Erasure with Points Preservation
```typescript
class DataPrivacyHandler {
  async handleDeletionRequest(
    memberId: string,
    request: DeletionRequest
  ): Promise<DeletionResult> {
    // Preserve legally required data
    const retentionRequirements = await this.getRetentionRequirements();
    
    // Anonymize instead of delete for points history
    if (retentionRequirements.includes('transaction_history')) {
      await this.anonymizeMemberData(memberId, {
        preserveFields: ['points_balance', 'transaction_amounts'],
        hash: ['email', 'phone', 'name'],
        remove: ['address', 'dob', 'preferences']
      });
    }
    
    // Transfer points to anonymous account if requested
    if (request.pointsHandling === 'donate') {
      await this.donatePoints(memberId, request.charityId);
    } else if (request.pointsHandling === 'transfer') {
      await this.transferToAnonymousPool(memberId);
    }
    
    return {
      status: 'completed',
      dataRemoved: ['personal_info', 'preferences'],
      dataAnonymized: ['transaction_history'],
      pointsHandled: request.pointsHandling
    };
  }
}
```

---

This comprehensive edge case handling ensures the platform remains robust and flexible across all scenarios, from simple transactions to complex multi-jurisdictional operations.
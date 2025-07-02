# Unified Platform Sample Data

## Overview
Sample data demonstrating how punch card businesses integrate with transportation, hotel, and event platforms in the Ploy ecosystem.

## Customer Data

### Unified Customer Profiles
```json
{
  "customers": [
    {
      "id": "c123e4567-e89b-12d3-a456-426614174000",
      "phone": "+1234567890",
      "email": "sarah.chen@email.com",
      "name": "Sarah Chen",
      "created_at": "2024-01-15T08:00:00Z",
      "source_platform": "punch_card",
      "platforms": {
        "punch_card": {
          "joined": "2024-01-15T08:00:00Z",
          "status": "active"
        },
        "transportation": {
          "joined": "2023-06-20T10:30:00Z",
          "customer_id": "FERRY-2023-4521",
          "status": "matched"
        },
        "hotel": {
          "joined": "2023-08-15T14:00:00Z",
          "customer_id": "HTL-98765",
          "status": "matched"
        },
        "events": {
          "joined": "2023-11-01T19:00:00Z",
          "customer_id": "EVT-45678",
          "status": "matched"
        }
      },
      "total_points": 3250,
      "tier": "gold",
      "lifetime_value": 4500.00
    },
    {
      "id": "c223e4567-e89b-12d3-a456-426614174001",
      "phone": "+1234567891",
      "email": "james.wilson@email.com",
      "name": "James Wilson",
      "created_at": "2024-02-01T09:30:00Z",
      "source_platform": "punch_card",
      "platforms": {
        "punch_card": {
          "joined": "2024-02-01T09:30:00Z",
          "status": "active"
        },
        "transportation": {
          "joined": "2024-02-05T07:15:00Z",
          "customer_id": "FERRY-2024-1122",
          "status": "pending_match"
        }
      },
      "total_points": 450,
      "tier": "bronze",
      "lifetime_value": 125.50
    }
  ]
}
```

## Business Data

### Multi-Type Business Registry
```json
{
  "businesses": [
    {
      "id": "b123e4567-e89b-12d3-a456-426614174000",
      "name": "Sunrise Coffee House",
      "type": "coffee_shop",
      "platform": "punch_card",
      "address": {
        "street": "123 Main St",
        "city": "Seattle",
        "state": "WA",
        "zip": "98101",
        "coordinates": {
          "lat": 47.6062,
          "lng": -122.3321
        }
      },
      "settings": {
        "business_hours": "6:00 AM - 8:00 PM",
        "accepts_mobile_orders": true
      },
      "integration_status": "active",
      "created_at": "2024-01-01T00:00:00Z"
    },
    {
      "id": "b223e4567-e89b-12d3-a456-426614174001",
      "name": "City Ferry Services",
      "type": "transport_operator",
      "platform": "transportation",
      "settings": {
        "routes": ["Seattle-Bainbridge", "Seattle-Vashon"],
        "api_endpoint": "https://api.cityferryservices.com/v2",
        "sync_frequency": "real_time"
      },
      "integration_status": "active",
      "created_at": "2023-01-01T00:00:00Z"
    },
    {
      "id": "b323e4567-e89b-12d3-a456-426614174002",
      "name": "Harbor View Hotel",
      "type": "hotel",
      "platform": "hospitality",
      "settings": {
        "property_code": "HVH-SEA",
        "pms_system": "Opera",
        "room_count": 150
      },
      "integration_status": "active",
      "created_at": "2023-03-15T00:00:00Z"
    },
    {
      "id": "b423e4567-e89b-12d3-a456-426614174003",
      "name": "Seattle Event Center",
      "type": "event_venue",
      "platform": "events",
      "settings": {
        "capacity": 5000,
        "venue_type": "multi_purpose",
        "ticketing_system": "EventPro"
      },
      "integration_status": "active",
      "created_at": "2023-05-20T00:00:00Z"
    }
  ]
}
```

## Loyalty Activities

### Cross-Platform Activity Stream
```json
{
  "loyalty_activities": [
    {
      "id": "a123e4567-e89b-12d3-a456-426614174000",
      "customer_id": "c123e4567-e89b-12d3-a456-426614174000",
      "business_id": "b123e4567-e89b-12d3-a456-426614174000",
      "activity_type": "punch",
      "platform": "punch_card",
      "points_value": 10,
      "metadata": {
        "punch_card_id": "pc123",
        "punch_number": 5,
        "item_purchased": "Large Latte"
      },
      "created_at": "2024-03-01T08:15:00Z"
    },
    {
      "id": "a223e4567-e89b-12d3-a456-426614174001",
      "customer_id": "c123e4567-e89b-12d3-a456-426614174000",
      "business_id": "b223e4567-e89b-12d3-a456-426614174001",
      "activity_type": "transport_booking",
      "platform": "transportation",
      "points_value": 50,
      "metadata": {
        "booking_id": "FRY-2024-3421",
        "route": "Seattle-Bainbridge",
        "fare_amount": 12.50,
        "vehicle_type": "ferry"
      },
      "created_at": "2024-03-01T07:00:00Z"
    },
    {
      "id": "a323e4567-e89b-12d3-a456-426614174002",
      "customer_id": "c123e4567-e89b-12d3-a456-426614174000",
      "business_id": "b323e4567-e89b-12d3-a456-426614174002",
      "activity_type": "hotel_stay",
      "platform": "hospitality",
      "points_value": 500,
      "metadata": {
        "booking_id": "HVH-2024-9876",
        "check_in": "2024-02-28",
        "check_out": "2024-03-02",
        "nights": 2,
        "room_type": "deluxe_ocean_view",
        "total_amount": 450.00
      },
      "created_at": "2024-03-02T12:00:00Z"
    },
    {
      "id": "a423e4567-e89b-12d3-a456-426614174003",
      "customer_id": "c123e4567-e89b-12d3-a456-426614174000",
      "business_id": "b423e4567-e89b-12d3-a456-426614174003",
      "activity_type": "event_attendance",
      "platform": "events",
      "points_value": 100,
      "metadata": {
        "event_id": "EVT-2024-TECH",
        "event_name": "Seattle Tech Conference",
        "ticket_type": "general_admission",
        "price": 75.00
      },
      "created_at": "2024-02-15T09:00:00Z"
    }
  ]
}
```

## Punch Card Campaigns

### Active Punch Card Examples
```json
{
  "punch_cards": [
    {
      "id": "pc123e4567-e89b-12d3-a456-426614174000",
      "business_id": "b123e4567-e89b-12d3-a456-426614174000",
      "title": "Coffee Lover's Card",
      "description": "Buy 9 coffees, get the 10th free!",
      "required_punches": 10,
      "reward": "Free coffee of your choice",
      "valid_platforms": ["punch_card"],
      "is_active": true,
      "created_at": "2024-01-01T00:00:00Z"
    },
    {
      "id": "pc223e4567-e89b-12d3-a456-426614174001",
      "business_id": "b123e4567-e89b-12d3-a456-426614174000",
      "title": "Morning Commuter Special",
      "description": "5 morning coffees + 3 ferry rides = Free breakfast combo",
      "required_punches": 5,
      "required_activities": {
        "transportation": {
          "activity_type": "transport_booking",
          "count": 3
        }
      },
      "reward": "Free breakfast sandwich + coffee",
      "valid_platforms": ["punch_card", "transportation"],
      "is_active": true,
      "created_at": "2024-02-01T00:00:00Z"
    }
  ]
}
```

## Cross-Platform Rewards

### Universal Reward Configurations
```json
{
  "cross_platform_rewards": [
    {
      "id": "cpr123e4567-e89b-12d3-a456-426614174000",
      "name": "Lifestyle Explorer",
      "description": "Experience the city like a local",
      "requirements": {
        "punch_card": {
          "unique_businesses": 3,
          "total_punches": 15
        },
        "transportation": {
          "trips": 5
        },
        "events": {
          "attendances": 1
        }
      },
      "rewards": [
        {
          "platform": "hotel",
          "value": "20% off next stay"
        },
        {
          "platform": "transportation",
          "value": "5 free ferry tickets"
        }
      ],
      "tier_required": null,
      "valid_until": "2024-12-31T23:59:59Z"
    },
    {
      "id": "cpr223e4567-e89b-12d3-a456-426614174001",
      "name": "Weekend Warrior",
      "description": "Make the most of your weekends",
      "requirements": {
        "hotel": {
          "weekend_stays": 2
        },
        "events": {
          "weekend_events": 3
        },
        "punch_card": {
          "weekend_visits": 10
        }
      },
      "rewards": [
        {
          "platform": "hotel",
          "value": "Free room upgrade"
        },
        {
          "platform": "events",
          "value": "VIP access to next event"
        },
        {
          "platform": "punch_card",
          "value": "$50 credit at any partner"
        }
      ],
      "tier_required": "silver",
      "valid_until": "2024-12-31T23:59:59Z"
    }
  ]
}
```

## Customer Journey Example

### Sarah Chen's Complete Journey
```json
{
  "customer_journey": {
    "customer_id": "c123e4567-e89b-12d3-a456-426614174000",
    "timeline": [
      {
        "date": "2023-06-20",
        "platform": "transportation",
        "action": "First ferry booking",
        "hidden_from_ploy": true
      },
      {
        "date": "2023-08-15",
        "platform": "hotel",
        "action": "Hotel stay for conference",
        "hidden_from_ploy": true
      },
      {
        "date": "2024-01-15",
        "platform": "punch_card",
        "action": "Downloads Ploy for coffee shop",
        "visible": true
      },
      {
        "date": "2024-02-01",
        "platform": "all",
        "action": "Silent integration activated",
        "hidden_from_ploy": true,
        "system_event": "customer_matching_complete"
      },
      {
        "date": "2024-03-15",
        "platform": "all",
        "action": "Big reveal - discovers 3,250 points!",
        "visible": true,
        "notifications": [
          "Welcome to the complete Ploy ecosystem!",
          "You've earned 3,250 points across all platforms",
          "Unlock exclusive cross-platform rewards"
        ]
      }
    ]
  }
}
```

## Analytics Sample Data

### Platform Performance Metrics
```json
{
  "platform_metrics": {
    "date": "2024-03-15",
    "platforms": {
      "punch_card": {
        "active_customers": 5420,
        "daily_activities": 12350,
        "average_points_earned": 35
      },
      "transportation": {
        "active_customers": 8900,
        "daily_activities": 3200,
        "average_points_earned": 125
      },
      "hotel": {
        "active_customers": 2100,
        "daily_activities": 450,
        "average_points_earned": 850
      },
      "events": {
        "active_customers": 3500,
        "daily_activities": 280,
        "average_points_earned": 200
      }
    },
    "cross_platform": {
      "multi_platform_customers": 1850,
      "cross_platform_rewards_claimed": 342,
      "average_platforms_per_customer": 2.3
    }
  }
}
```

## Test Scenarios

### Customer Matching Scenarios
1. **Exact Match**: Phone number matches perfectly
2. **Email Match**: Phone differs but email matches
3. **Fuzzy Match**: Similar name and partial phone match
4. **No Match**: Create shadow profile for later
5. **Multiple Matches**: Require manual resolution

### Integration Test Cases
```json
{
  "test_cases": [
    {
      "name": "New punch card customer is existing ferry user",
      "steps": [
        "Customer signs up for punch card with phone +1234567890",
        "System finds matching ferry customer",
        "Links accounts silently",
        "Continues accumulating points across both"
      ],
      "expected": "Successful silent match"
    },
    {
      "name": "Cross-platform reward qualification",
      "steps": [
        "Customer has 10 coffee punches",
        "Customer takes 5 ferry rides",
        "System detects qualification for 'Commuter Reward'",
        "Sends notification about surprise reward"
      ],
      "expected": "Reward unlocked and notified"
    }
  ]
}
```

---

This sample data demonstrates the seamless integration between punch cards and existing platforms, showing how customers accumulate value across the entire ecosystem while maintaining a simple, approachable entry point.
# BringIt Technical Architecture Document

## System Architecture Overview

### 1. High-Level Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Mobile Apps   │    │   Web Dashboard │    │   Admin Panel   │
│  (React Native) │    │   (Next.js)     │    │   (React.js)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   API Gateway   │
                    │  (Kong/AWS ALB) │
                    └─────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Auth Service  │    │ Core API Server │    │ Notification    │
│   (Express.js)  │    │  (Express.js)   │    │   Service       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
   │   PostgreSQL    │  │     Redis       │  │    MongoDB      │
   │ (Primary DB)    │  │   (Cache)       │  │ (Analytics)     │
   └─────────────────┘  └─────────────────┘  └─────────────────┘
```

### 2. Microservices Architecture

#### 2.1 Core Services
- **User Service**: Authentication, profiles, KYC management
- **Request Service**: Delivery request management
- **Matching Service**: AI-powered courier-buyer matching
- **Payment Service**: Escrow, transactions, commission handling
- **Notification Service**: Email, SMS, push notifications
- **Admin Service**: Administrative operations and analytics

#### 2.2 Supporting Services
- **File Service**: Document and image storage management
- **Analytics Service**: Data collection and business intelligence
- **Compliance Service**: KYC verification and fraud detection
- **Communication Service**: In-app messaging and video calls

### 3. Database Design

#### 3.1 Primary Database (PostgreSQL)
```sql
-- Users and Authentication
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone_number VARCHAR(20) UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    user_type user_type_enum NOT NULL,
    verification_status verification_status_enum DEFAULT 'pending',
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User Profiles
CREATE TABLE user_profiles (
    profile_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    date_of_birth DATE,
    address JSONB,
    profile_image_url TEXT,
    kyc_documents JSONB,
    trust_score INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Delivery Requests
CREATE TABLE delivery_requests (
    request_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    buyer_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    item_details JSONB NOT NULL,
    pickup_location JSONB NOT NULL,
    delivery_location JSONB NOT NULL,
    estimated_weight DECIMAL(8,2),
    dimensions JSONB,
    requested_delivery_date DATE,
    budget_range JSONB,
    status request_status_enum DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Courier Trips
CREATE TABLE courier_trips (
    trip_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    courier_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    origin_country VARCHAR(3) NOT NULL,
    destination_country VARCHAR(3) NOT NULL,
    departure_date DATE NOT NULL,
    arrival_date DATE NOT NULL,
    available_capacity DECIMAL(8,2) NOT NULL,
    price_per_kg DECIMAL(10,2),
    trip_details JSONB,
    status trip_status_enum DEFAULT 'available',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Transactions
CREATE TABLE transactions (
    transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_id UUID REFERENCES delivery_requests(request_id),
    courier_id UUID REFERENCES users(user_id),
    buyer_id UUID REFERENCES users(user_id),
    amount DECIMAL(12,2) NOT NULL,
    commission DECIMAL(12,2) NOT NULL,
    escrow_status escrow_status_enum DEFAULT 'held',
    payment_method VARCHAR(50),
    payment_reference VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP
);

-- Messages
CREATE TABLE messages (
    message_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sender_id UUID REFERENCES users(user_id),
    receiver_id UUID REFERENCES users(user_id),
    request_id UUID REFERENCES delivery_requests(request_id),
    content TEXT NOT NULL,
    message_type message_type_enum DEFAULT 'text',
    attachments JSONB,
    is_read BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Reviews and Ratings
CREATE TABLE reviews (
    review_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reviewer_id UUID REFERENCES users(user_id),
    reviewee_id UUID REFERENCES users(user_id),
    transaction_id UUID REFERENCES transactions(transaction_id),
    rating INTEGER CHECK (rating >= 1 AND rating <= 5),
    comment TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### 3.2 Caching Strategy (Redis)
```
# User Sessions
user:session:{user_id} -> session_data (TTL: 24h)

# API Rate Limiting
rate_limit:{endpoint}:{user_id} -> request_count (TTL: 1h)

# Matching Cache
matching:requests:{location} -> sorted_set_of_requests (TTL: 1h)
matching:couriers:{route} -> sorted_set_of_couriers (TTL: 1h)

# Notification Queue
notifications:queue -> list_of_notifications

# Search Cache
search:requests:{query_hash} -> search_results (TTL: 30m)
```

### 4. API Design

#### 4.1 RESTful API Structure
```typescript
// Authentication Endpoints
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/refresh
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password

// User Management
GET    /api/v1/users/profile
PUT    /api/v1/users/profile
POST   /api/v1/users/kyc-upload
GET    /api/v1/users/verification-status
POST   /api/v1/users/change-password

// Delivery Requests
GET    /api/v1/requests              // List with pagination and filters
POST   /api/v1/requests              // Create new request
GET    /api/v1/requests/:id          // Get specific request
PUT    /api/v1/requests/:id          // Update request
DELETE /api/v1/requests/:id          // Cancel request
POST   /api/v1/requests/:id/match    // Match with courier

// Courier Operations
GET    /api/v1/trips                 // List courier's trips
POST   /api/v1/trips                 // Create new trip
GET    /api/v1/trips/available       // Available matching opportunities
POST   /api/v1/trips/:id/accept      // Accept a delivery request
PUT    /api/v1/trips/:id/status      // Update trip status

// Communication
GET    /api/v1/messages/:request_id  // Get conversation
POST   /api/v1/messages              // Send message
PUT    /api/v1/messages/:id/read     // Mark as read
POST   /api/v1/calls/initiate        // Start video call

// Payments
POST   /api/v1/payments/initiate     // Start payment process
POST   /api/v1/payments/confirm      // Confirm payment
GET    /api/v1/payments/history      // Payment history
POST   /api/v1/payments/refund       // Request refund

// Admin Endpoints
GET    /api/v1/admin/dashboard       // Dashboard metrics
GET    /api/v1/admin/users           // User management
PUT    /api/v1/admin/users/:id       // Update user status
GET    /api/v1/admin/transactions    // Transaction monitoring
GET    /api/v1/admin/reports         // Generate reports
```

#### 4.2 GraphQL Schema (Alternative/Additional)
```graphql
type User {
  id: ID!
  email: String!
  userType: UserType!
  profile: UserProfile
  verificationStatus: VerificationStatus!
  createdAt: DateTime!
}

type DeliveryRequest {
  id: ID!
  buyer: User!
  title: String!
  description: String
  itemDetails: JSON!
  pickupLocation: Location!
  deliveryLocation: Location!
  estimatedWeight: Float
  requestedDeliveryDate: Date
  budgetRange: BudgetRange
  status: RequestStatus!
  matches: [CourierTrip!]!
  createdAt: DateTime!
}

type CourierTrip {
  id: ID!
  courier: User!
  originCountry: String!
  destinationCountry: String!
  departureDate: Date!
  arrivalDate: Date!
  availableCapacity: Float!
  pricePerKg: Float
  status: TripStatus!
  acceptedRequests: [DeliveryRequest!]!
}

type Query {
  me: User
  deliveryRequests(filters: RequestFilters, pagination: Pagination): [DeliveryRequest!]!
  availableTrips(filters: TripFilters): [CourierTrip!]!
  conversation(requestId: ID!): [Message!]!
}

type Mutation {
  createDeliveryRequest(input: CreateRequestInput!): DeliveryRequest!
  createCourierTrip(input: CreateTripInput!): CourierTrip!
  acceptDeliveryRequest(requestId: ID!, tripId: ID!): Transaction!
  sendMessage(input: MessageInput!): Message!
  initiatePayment(input: PaymentInput!): PaymentResponse!
}

type Subscription {
  messageAdded(requestId: ID!): Message!
  requestStatusUpdated(requestId: ID!): DeliveryRequest!
  newMatchAvailable(userId: ID!): MatchNotification!
}
```

### 5. Security Implementation

#### 5.1 Authentication & Authorization
```typescript
// JWT Token Structure
interface JWTPayload {
  userId: string;
  email: string;
  userType: 'buyer' | 'courier' | 'admin';
  verificationStatus: string;
  iat: number;
  exp: number;
}

// Role-Based Access Control
const permissions = {
  buyer: [
    'create:request',
    'read:own_requests',
    'update:own_requests',
    'read:messages',
    'create:messages'
  ],
  courier: [
    'create:trip',
    'read:available_requests',
    'accept:requests',
    'read:messages',
    'create:messages'
  ],
  admin: [
    'read:all_users',
    'update:user_status',
    'read:all_transactions',
    'create:reports'
  ]
};

// Middleware Implementation
const authorize = (permission: string) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const userPermissions = permissions[req.user.userType];
    if (!userPermissions.includes(permission)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
};
```

#### 5.2 Data Encryption
```typescript
// Sensitive Data Encryption
import crypto from 'crypto';

class EncryptionService {
  private algorithm = 'aes-256-gcm';
  private secretKey = process.env.ENCRYPTION_KEY;

  encrypt(text: string): string {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipher(this.algorithm, this.secretKey);
    cipher.setAAD(Buffer.from('BringIt', 'utf8'));
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    return iv.toString('hex') + ':' + authTag.toString('hex') + ':' + encrypted;
  }

  decrypt(encryptedData: string): string {
    const parts = encryptedData.split(':');
    const iv = Buffer.from(parts[0], 'hex');
    const authTag = Buffer.from(parts[1], 'hex');
    const encrypted = parts[2];
    
    const decipher = crypto.createDecipher(this.algorithm, this.secretKey);
    decipher.setAAD(Buffer.from('BringIt', 'utf8'));
    decipher.setAuthTag(authTag);
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    return decrypted;
  }
}
```

### 6. Performance Optimization

#### 6.1 Database Optimization
```sql
-- Indexes for Performance
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone_number);
CREATE INDEX idx_requests_buyer_status ON delivery_requests(buyer_id, status);
CREATE INDEX idx_requests_location ON delivery_requests USING GIN(pickup_location, delivery_location);
CREATE INDEX idx_trips_route_date ON courier_trips(origin_country, destination_country, departure_date);
CREATE INDEX idx_transactions_user_date ON transactions(buyer_id, created_at);
CREATE INDEX idx_messages_request_date ON messages(request_id, created_at);

-- Partitioning for Large Tables
CREATE TABLE transactions_2024 PARTITION OF transactions
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE messages_2024 PARTITION OF messages
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

#### 6.2 Caching Strategy
```typescript
// Multi-Level Caching
class CacheService {
  constructor(
    private redis: Redis,
    private memcache: MemcacheClient
  ) {}

  async get(key: string): Promise<any> {
    // Level 1: In-memory cache (fastest)
    let data = this.memcache.get(key);
    if (data) return JSON.parse(data);

    // Level 2: Redis cache
    data = await this.redis.get(key);
    if (data) {
      this.memcache.set(key, data, 300); // 5 min TTL
      return JSON.parse(data);
    }

    return null;
  }

  async set(key: string, value: any, ttl: number = 3600): Promise<void> {
    const stringValue = JSON.stringify(value);
    
    // Store in both cache levels
    await this.redis.setex(key, ttl, stringValue);
    this.memcache.set(key, stringValue, Math.min(ttl, 300));
  }
}
```

### 7. Monitoring & Observability

#### 7.1 Application Monitoring
```typescript
// Custom Metrics Collection
class MetricsCollector {
  private prometheus = require('prom-client');
  
  private requestDuration = new this.prometheus.Histogram({
    name: 'http_request_duration_seconds',
    help: 'Duration of HTTP requests in seconds',
    labelNames: ['method', 'route', 'status_code']
  });

  private userRegistrations = new this.prometheus.Counter({
    name: 'user_registrations_total',
    help: 'Total number of user registrations',
    labelNames: ['user_type']
  });

  private deliveryRequests = new this.prometheus.Counter({
    name: 'delivery_requests_total',
    help: 'Total number of delivery requests',
    labelNames: ['status']
  });

  recordRequestDuration(method: string, route: string, statusCode: number, duration: number) {
    this.requestDuration
      .labels(method, route, statusCode.toString())
      .observe(duration);
  }

  incrementUserRegistrations(userType: string) {
    this.userRegistrations.labels(userType).inc();
  }

  incrementDeliveryRequests(status: string) {
    this.deliveryRequests.labels(status).inc();
  }
}
```

#### 7.2 Logging Strategy
```typescript
// Structured Logging
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'bringit-api' },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
});

// Request Logging Middleware
const requestLogger = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info('HTTP Request', {
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration,
      userAgent: req.get('User-Agent'),
      ip: req.ip,
      userId: req.user?.userId
    });
  });
  
  next();
};
```

This technical architecture provides a solid foundation for building a scalable, secure, and maintainable peer-to-peer delivery platform. The modular design allows for independent scaling of different components based on load and requirements.

# Online Tournament Ranking Dashboard - Complete System Design

## Comprehensive Architecture for High-Scale Gaming Platform

---

## 📋 Table of Contents

1. [Overview & Requirements](#overview--requirements)
2. [Technology Stack](#technology-stack)
3. [System Architecture](#system-architecture)
4. [Database Design](#database-design)
5. [Backend Services](#backend-services)
6. [Frontend Implementation](#frontend-implementation)
7. [Real-Time Updates (SSE)](#real-time-updates-sse)
8. [Authentication & Security](#authentication--security)
9. [Scaling Strategy](#scaling-strategy)
10. [Performance Optimizations](#performance-optimizations)
11. [Deployment & Infrastructure](#deployment--infrastructure)
12. [Monitoring & Observability](#monitoring--observability)
13. [Technology Decisions Deep Dive](#technology-decisions-deep-dive)
14. [Cost Estimation](#cost-estimation)

---

## 🎯 Overview & Requirements

### Problem Statement

Design a **real-time tournament ranking dashboard** where users can:
- Login and view their rankings across tournaments
- See global and tournament-specific rankings
- View statistics and historical game data
- Receive live score updates during matches
- Get notifications when their rank changes
- Manage their profile

### Scale Requirements

```yaml
Concurrent Users: 100,000+
Registered Users: 10,000,000
Active Tournaments: 12 simultaneously
Update Frequency: Real-time
Dashboard Load Time: < 2 seconds
Update Latency: < 500ms
```

### Functional Requirements

**User Features:**
- ✅ User authentication (email/password + OAuth)
- ✅ Dashboard with personal ranking
- ✅ Global leaderboard (all tournaments)
- ✅ Tournament-specific leaderboards
- ✅ Statistics display (wins, losses, matches played)
- ✅ Historical game statistics
- ✅ Profile management
- ✅ Real-time rank change notifications

**System Features:**
- ✅ Real-time score updates
- ✅ Live ranking recalculations
- ✅ Push notifications for rank changes
- ✅ Support 12 concurrent tournaments
- ✅ Handle 100K+ concurrent users

**Excluded Features:**
- ❌ Match history/replays
- ❌ Game videos
- ❌ Tournament joining from dashboard
- ❌ Social features (friends, chat)
- ❌ Historical ranking trends/charts

### Non-Functional Requirements

```yaml
Performance:
  - Dashboard load: < 2 seconds
  - API response time: < 500ms (p95)
  - Real-time update latency: < 500ms
  - Database query time: < 100ms

Scalability:
  - Support 100K concurrent users
  - Handle 10M registered users
  - Process 10K+ score updates per minute

Availability:
  - 99.9% uptime (43 minutes downtime/month)
  - No single point of failure
  - Auto-failover for critical services

Security:
  - Secure authentication (JWT + OAuth)
  - HTTPS only
  - Rate limiting
  - DDoS protection
```

---

## 💻 Technology Stack

### Frontend

```yaml
Core:
  - React 18 (UI library)
  - TypeScript (type safety)
  - Next.js 14 (SSR, routing, optimization)

State Management:
  - Zustand (lightweight global state)
  - React Query v5 (server state, caching)

Real-Time:
  - Server-Sent Events (SSE) (one-way updates)
  - EventSource API (browser native)

UI Components:
  - Radix UI (accessible primitives)
  - TailwindCSS (styling)
  - Framer Motion (animations)

Data Visualization:
  - Recharts (charts)
  - React Window (virtual scrolling)

Form Handling:
  - React Hook Form (forms)
  - Zod (validation)

HTTP Client:
  - Axios (API calls with interceptors)

Utilities:
  - date-fns (date formatting)
  - clsx (conditional classes)
```

### Backend

```yaml
Runtime & Framework:
  - Node.js 20 LTS
  - NestJS 10 (TypeScript framework)

Microservices:
  - Auth Service (authentication, OAuth)
  - Ranking Service (rankings, leaderboards)
  - Tournament Service (tournament CRUD)
  - Real-Time SSE Service (live updates)

Database:
  - PostgreSQL 15+ (primary database)
  - TypeORM (ORM)

Cache:
  - Redis Cluster 7 (caching, pub/sub, leaderboards)
  - ioredis (Redis client)

Message Queue:
  - Apache Kafka (event streaming)
  - KafkaJS (Kafka client)

Authentication:
  - Passport.js (auth strategies)
  - jsonwebtoken (JWT)
  - bcrypt (password hashing)
  - OAuth 2.0 (Google, Steam, Discord)

Validation:
  - class-validator (DTO validation)
  - class-transformer (data transformation)
```

### Infrastructure

```yaml
Cloud Provider: AWS

Compute:
  - ECS Fargate (containerized services)
  - Application Load Balancer (ALB)

Database:
  - RDS PostgreSQL (Multi-AZ)
  - ElastiCache Redis (Cluster mode)

Message Queue:
  - Amazon MSK (Managed Kafka)

Storage:
  - S3 (static assets, backups)
  - CloudFront (CDN)

Monitoring:
  - CloudWatch (metrics, logs)
  - DataDog (APM)
  - Sentry (error tracking)

Security:
  - WAF (Web Application Firewall)
  - Shield (DDoS protection)
  - Secrets Manager (credentials)

CI/CD:
  - GitHub Actions
  - Docker + ECR (container registry)
```

---

## 🏗️ System Architecture

### High-Level Architecture

```
                          ┌─────────────────────────────────────┐
                          │     CloudFlare / AWS CloudFront     │
                          │          (CDN + DDoS)               │
                          └──────────────┬──────────────────────┘
                                         │
                                         ↓
┌────────────────────────────────────────────────────────────────────┐
│                    Frontend Layer (React + Next.js)                │
│  - Deployed on Vercel / AWS Amplify                               │
│  - Server-Side Rendering (SSR) for initial load                   │
│  - Static assets on CDN                                           │
│  - SSE client for real-time updates                               │
└──────────────────────────┬─────────────────────────────────────────┘
                           │
                           │ HTTPS + SSE
                           ↓
┌────────────────────────────────────────────────────────────────────┐
│              Load Balancer (AWS ALB / NGINX)                       │
│  - Health checks                                                   │
│  - SSL termination                                                 │
│  - Request routing                                                 │
└──────────────────────────┬─────────────────────────────────────────┘
                           │
                           ↓
┌────────────────────────────────────────────────────────────────────┐
│                  API Gateway (Kong / AWS API Gateway)              │
│  - Rate limiting (1000 req/min per user)                          │
│  - JWT validation                                                  │
│  - Request routing to microservices                               │
│  - API versioning                                                  │
└──────────────────────────┬─────────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┬────────────────┐
        ↓                  ↓                  ↓                ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Auth Service │  │ Ranking      │  │ Tournament   │  │ Real-time    │
│              │  │ Service      │  │ Service      │  │ SSE Service  │
│ (3 replicas) │  │ (5 replicas) │  │ (3 replicas) │  │ (10 replicas)│
│ Port: 3001   │  │ Port: 3002   │  │ Port: 3003   │  │ Port: 3004   │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │                 │
       └─────────────────┴─────────────────┴─────────────────┘
                           │
                           ↓
┌────────────────────────────────────────────────────────────────────┐
│                    Message Queue (Apache Kafka)                    │
│  Topics:                                                           │
│  - ranking.updates (12 partitions)                                │
│  - tournament.events (4 partitions)                               │
│  - user.notifications (6 partitions)                              │
└──────────────────────────┬─────────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┬────────────────┐
        ↓                  ↓                  ↓                ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ PostgreSQL   │  │ Redis        │  │ Elasticsearch│  │ S3 / Object  │
│ (Primary)    │  │ Cluster      │  │ (Search)     │  │ Storage      │
│ Read Replicas│  │ (Cache)      │  │              │  │ (Assets)     │
│ (3 slaves)   │  │ (6 nodes)    │  │              │  │              │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

### Component Breakdown

#### 1. Frontend Layer (React + Next.js)

```
Frontend/
├── pages/
│   ├── index.tsx                 # Homepage
│   ├── dashboard.tsx             # Main dashboard (SSR)
│   ├── leaderboard/
│   │   ├── [tournamentId].tsx   # Tournament leaderboard
│   │   └── global.tsx           # Global leaderboard
│   ├── profile.tsx               # User profile
│   └── auth/
│       ├── login.tsx
│       └── callback.tsx          # OAuth callback
├── components/
│   ├── Dashboard/
│   │   ├── RankingCard.tsx
│   │   ├── StatsCard.tsx
│   │   └── TournamentList.tsx
│   ├── Leaderboard/
│   │   ├── VirtualizedList.tsx   # Virtual scrolling
│   │   └── LeaderboardRow.tsx
│   └── Shared/
│       ├── Header.tsx
│       └── Notification.tsx
├── hooks/
│   ├── useAuth.ts
│   ├── useRankings.ts
│   ├── useSSE.ts                 # Real-time updates
│   └── useNotifications.ts
├── services/
│   ├── api.ts                    # Axios instance
│   └── sse.ts                    # SSE client
└── store/
    ├── authStore.ts              # Zustand store
    └── notificationStore.ts
```

#### 2. Backend Services

**Auth Service (Port 3001):**
```
auth-service/
├── src/
│   ├── auth/
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── jwt.strategy.ts
│   │   └── oauth.strategy.ts
│   ├── users/
│   │   ├── user.entity.ts
│   │   └── user.repository.ts
│   └── main.ts
└── package.json
```

**Ranking Service (Port 3002):**
```
ranking-service/
├── src/
│   ├── rankings/
│   │   ├── rankings.controller.ts
│   │   ├── rankings.service.ts
│   │   ├── score.entity.ts
│   │   └── ranking.calculator.ts
│   ├── leaderboard/
│   │   └── leaderboard.service.ts
│   └── main.ts
└── package.json
```

**Tournament Service (Port 3003):**
```
tournament-service/
├── src/
│   ├── tournaments/
│   │   ├── tournament.controller.ts
│   │   ├── tournament.service.ts
│   │   └── tournament.entity.ts
│   └── main.ts
└── package.json
```

**Real-Time SSE Service (Port 3004):**
```
realtime-service/
├── src/
│   ├── sse/
│   │   ├── sse.controller.ts
│   │   ├── sse.service.ts
│   │   └── sse.gateway.ts
│   └── main.ts
└── package.json
```

### Data Flow Diagram

#### User Views Dashboard Flow

```
1. User navigates to /dashboard
   ↓
2. Next.js SSR: Fetch initial data server-side
   ↓
3. API Gateway → Auth Service: Validate JWT
   ↓
4. API Gateway → Ranking Service: Get user rankings
   ↓
5. Ranking Service → Redis: Check cache
   ↓
6. Cache miss → PostgreSQL: Query rankings
   ↓
7. Calculate rank using SQL window functions
   ↓
8. Store in Redis (cache for 1 minute)
   ↓
9. Return data to Next.js
   ↓
10. Next.js renders HTML with data (SSR)
   ↓
11. Send HTML to browser (< 2 seconds)
   ↓
12. Browser renders page
   ↓
13. React hydrates (make interactive)
   ↓
14. Establish SSE connection for real-time updates
   ↓
15. SSE Service subscribes to Kafka topic
   ↓
16. When rank changes → Kafka → SSE → Browser
```

#### Score Update Flow

```
1. Match ends (external game server)
   ↓
2. POST /api/rankings/update-score
   ↓
3. Ranking Service receives score update
   ↓
4. Update PostgreSQL (source of truth)
   ↓
5. Update Redis sorted set (leaderboard)
   ↓
6. Calculate new rank
   ↓
7. If rank changed → Publish to Kafka
   ↓
8. Kafka message consumed by SSE Service
   ↓
9. SSE Service pushes update to all connected clients
   ↓
10. Browser receives SSE event
   ↓
11. React updates UI (ranking changes)
```

---

## 🗄️ Database Design

### PostgreSQL Schema

#### Users Table

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  username VARCHAR(100) UNIQUE NOT NULL,
  password_hash VARCHAR(255), -- NULL for OAuth users
  oauth_provider VARCHAR(50), -- 'google', 'steam', 'discord', NULL
  oauth_id VARCHAR(255),
  avatar_url TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  last_login TIMESTAMP,
  is_active BOOLEAN DEFAULT TRUE,
  is_email_verified BOOLEAN DEFAULT FALSE
);

-- Indexes for fast lookups
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_oauth ON users(oauth_provider, oauth_id)
  WHERE oauth_provider IS NOT NULL;
CREATE INDEX idx_users_active ON users(is_active)
  WHERE is_active = TRUE;
```

#### Tournaments Table

```sql
CREATE TABLE tournaments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  game_type VARCHAR(100) NOT NULL, -- 'battle-royale', 'team-deathmatch', etc.
  description TEXT,
  start_date TIMESTAMP NOT NULL,
  end_date TIMESTAMP NOT NULL,
  max_participants INTEGER DEFAULT 10000,
  current_participants INTEGER DEFAULT 0,
  status VARCHAR(50) DEFAULT 'active', -- 'upcoming', 'active', 'completed', 'cancelled'
  config JSONB, -- Tournament-specific configuration
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_tournaments_status ON tournaments(status);
CREATE INDEX idx_tournaments_dates ON tournaments(start_date, end_date);
CREATE INDEX idx_tournaments_game_type ON tournaments(game_type);
CREATE INDEX idx_tournaments_active ON tournaments(status, end_date)
  WHERE status = 'active';

-- Trigger to update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_tournaments_updated_at
  BEFORE UPDATE ON tournaments
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

#### Tournament Participants Table (Partitioned)

```sql
-- Parent table
CREATE TABLE tournament_participants (
  id UUID DEFAULT gen_random_uuid(),
  tournament_id UUID NOT NULL,
  user_id UUID NOT NULL,
  joined_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (id, tournament_id), -- Composite key for partitioning
  UNIQUE (tournament_id, user_id)
) PARTITION BY LIST (tournament_id);

-- Foreign keys (on individual partitions)
-- ALTER TABLE tournament_participants_t1
--   ADD CONSTRAINT fk_tournament FOREIGN KEY (tournament_id)
--   REFERENCES tournaments(id) ON DELETE CASCADE;

-- Create partition for each tournament (dynamically)
-- Example:
CREATE TABLE tournament_participants_t1
  PARTITION OF tournament_participants
  FOR VALUES IN ('tournament-uuid-1');

-- Indexes on partition
CREATE INDEX idx_participants_t1_user
  ON tournament_participants_t1(user_id);
```

#### Tournament Scores Table (Partitioned)

```sql
-- Parent table
CREATE TABLE tournament_scores (
  id UUID DEFAULT gen_random_uuid(),
  tournament_id UUID NOT NULL,
  user_id UUID NOT NULL,
  score INTEGER NOT NULL DEFAULT 0,
  matches_played INTEGER DEFAULT 0,
  wins INTEGER DEFAULT 0,
  losses INTEGER DEFAULT 0,
  draws INTEGER DEFAULT 0,
  last_match_at TIMESTAMP,
  updated_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (id, tournament_id),
  UNIQUE (tournament_id, user_id)
) PARTITION BY LIST (tournament_id);

-- Create partition for each tournament
CREATE TABLE tournament_scores_t1
  PARTITION OF tournament_scores
  FOR VALUES IN ('tournament-uuid-1');

-- Indexes on partition
CREATE INDEX idx_scores_t1_score
  ON tournament_scores_t1(score DESC);
CREATE INDEX idx_scores_t1_user
  ON tournament_scores_t1(user_id);
CREATE INDEX idx_scores_t1_updated
  ON tournament_scores_t1(updated_at DESC);
```

#### Rankings Materialized View

```sql
-- Materialized view for fast ranking queries
CREATE MATERIALIZED VIEW tournament_rankings AS
SELECT
  tournament_id,
  user_id,
  score,
  matches_played,
  wins,
  losses,
  RANK() OVER (
    PARTITION BY tournament_id
    ORDER BY score DESC, wins DESC, matches_played ASC
  ) as rank,
  DENSE_RANK() OVER (
    PARTITION BY tournament_id
    ORDER BY score DESC
  ) as dense_rank,
  PERCENT_RANK() OVER (
    PARTITION BY tournament_id
    ORDER BY score DESC
  ) as percentile
FROM tournament_scores
WHERE score > 0;

-- Indexes on materialized view
CREATE INDEX idx_rankings_tournament_rank
  ON tournament_rankings(tournament_id, rank);
CREATE INDEX idx_rankings_user
  ON tournament_rankings(user_id, tournament_id);
CREATE INDEX idx_rankings_score
  ON tournament_rankings(tournament_id, score DESC);

-- Concurrent refresh (doesn't lock table)
-- Run this every 30 seconds via cron job
REFRESH MATERIALIZED VIEW CONCURRENTLY tournament_rankings;
```

#### Match History Table (Time-Series Partitioning)

```sql
CREATE TABLE match_history (
  id UUID DEFAULT gen_random_uuid(),
  tournament_id UUID NOT NULL,
  user_id UUID NOT NULL,
  opponent_id UUID,
  score_before INTEGER NOT NULL,
  score_after INTEGER NOT NULL,
  score_change INTEGER NOT NULL,
  result VARCHAR(20) NOT NULL, -- 'win', 'loss', 'draw'
  match_duration_seconds INTEGER,
  played_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (id, played_at)
) PARTITION BY RANGE (played_at);

-- Partition by month for efficient archival
CREATE TABLE match_history_2024_02
  PARTITION OF match_history
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE match_history_2024_03
  PARTITION OF match_history
  FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Indexes on each partition
CREATE INDEX idx_match_history_2024_02_user
  ON match_history_2024_02(user_id, played_at DESC);
CREATE INDEX idx_match_history_2024_02_tournament
  ON match_history_2024_02(tournament_id, played_at DESC);

-- Auto-create partitions monthly (cron job)
```

#### User Statistics Table

```sql
CREATE TABLE user_statistics (
  user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  total_matches INTEGER DEFAULT 0,
  total_wins INTEGER DEFAULT 0,
  total_losses INTEGER DEFAULT 0,
  total_draws INTEGER DEFAULT 0,
  highest_score INTEGER DEFAULT 0,
  highest_rank INTEGER, -- Best rank achieved (1 = champion)
  tournaments_participated INTEGER DEFAULT 0,
  tournaments_won INTEGER DEFAULT 0,
  current_win_streak INTEGER DEFAULT 0,
  best_win_streak INTEGER DEFAULT 0,
  last_match_at TIMESTAMP,
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_user_stats_highest_rank
  ON user_statistics(highest_rank ASC NULLS LAST);
CREATE INDEX idx_user_stats_tournaments_won
  ON user_statistics(tournaments_won DESC);
```

### Database Configuration

```yaml
PostgreSQL Configuration (for 10M users):

Master Instance:
  Instance Type: db.r5.2xlarge
  vCPU: 8
  RAM: 64GB
  Storage: 1TB SSD (gp3)
  IOPS: 12000
  Max Connections: 1000

Read Replicas (3):
  Instance Type: db.r5.xlarge
  vCPU: 4
  RAM: 32GB
  Storage: 1TB SSD (gp3)
  IOPS: 10000
  Max Connections: 1000 each

Connection Pooling (PgBouncer):
  Mode: Transaction
  Max Client Connections: 10000
  Default Pool Size: 25
  Reserve Pool Size: 10

Performance Settings:
  shared_buffers: 16GB
  effective_cache_size: 48GB
  maintenance_work_mem: 2GB
  checkpoint_completion_target: 0.9
  wal_buffers: 16MB
  default_statistics_target: 100
  random_page_cost: 1.1 (SSD)
  effective_io_concurrency: 200
  work_mem: 64MB
  min_wal_size: 4GB
  max_wal_size: 16GB
  max_worker_processes: 8
  max_parallel_workers_per_gather: 4
  max_parallel_workers: 8
```

---

### Redis Data Structures

#### 1. Leaderboards (Sorted Sets)

```javascript
// Store tournament leaderboard
Key: "leaderboard:{tournament_id}:full"
Type: Sorted Set (ZSET)
Members: user_id
Scores: actual_score

// Commands:
ZADD leaderboard:t1:full 9500 user-alice
ZADD leaderboard:t1:full 9200 user-bob
ZADD leaderboard:t1:full 8800 user-charlie

// Get top 100 (O(log N + 100))
ZREVRANGE leaderboard:t1:full 0 99 WITHSCORES

// Get user's rank (O(log N))
ZREVRANK leaderboard:t1:full user-alice
// Returns: 0 (0-indexed, so rank #1)

// Get user's score (O(1))
ZSCORE leaderboard:t1:full user-alice
// Returns: 9500

// Get total participants (O(1))
ZCARD leaderboard:t1:full
// Returns: 3

// Get users by score range (O(log N + M))
ZRANGEBYSCORE leaderboard:t1:full 9000 10000 WITHSCORES

// Increment user's score (O(log N))
ZINCRBY leaderboard:t1:full 100 user-alice
// New score: 9600, rank automatically recalculated!

// Get rank neighbors (users around you)
ZREVRANK leaderboard:t1:full user-bob
// Returns: 1
ZREVRANGE leaderboard:t1:full 0 3 WITHSCORES
// Get 5 users: 2 above, yourself, 2 below
```

#### 2. User Ranking Cache

```javascript
// Cache individual user ranking
Key: "user:rank:{tournament_id}:{user_id}"
Type: String (JSON)
Value: { rank, score, percentile, totalParticipants }
TTL: 60 seconds

// Example:
SET user:rank:t1:alice '{"rank":1,"score":9500,"percentile":100,"total":1000}' EX 60

GET user:rank:t1:alice
// Returns: {"rank":1,"score":9500,"percentile":100,"total":1000}
```

#### 3. Top Rankings Cache

```javascript
// Cache top N rankings (most requested)
Key: "leaderboard:{tournament_id}:top{N}"
Type: String (JSON Array)
Value: [{ userId, username, score, rank }, ...]
TTL: 30 seconds

// Example:
SET leaderboard:t1:top100 '[{"userId":"alice","score":9500,"rank":1},...]' EX 30
```

#### 4. Tournament Metadata Cache

```javascript
// Cache tournament info
Key: "tournament:{tournament_id}:info"
Type: String (JSON)
Value: { id, name, status, participants, startDate, endDate }
TTL: 600 seconds (10 minutes)

// Example:
SET tournament:t1:info '{"id":"t1","name":"Spring Championship","status":"active",...}' EX 600
```

#### 5. Session Management

```javascript
// Store JWT refresh tokens
Key: "refresh_token:{token_uuid}"
Type: String (JSON)
Value: { userId, deviceInfo, issuedAt }
TTL: 604800 seconds (7 days)

// Blacklist revoked JWTs
Key: "blacklist:{jwt_jti}"
Type: String
Value: "1"
TTL: Remaining token lifetime (max 900 seconds)
```

#### 6. Rate Limiting

```javascript
// Track API requests per user
Key: "ratelimit:{user_id}:{endpoint}"
Type: String (counter)
TTL: 60 seconds (sliding window)

// Example:
INCR ratelimit:alice:/api/rankings
// Returns: 1
EXPIRE ratelimit:alice:/api/rankings 60

// Check if rate limited
GET ratelimit:alice:/api/rankings
// If > 1000, return 429 Too Many Requests
```

#### 7. Pub/Sub Channels

```javascript
// Real-time ranking updates
Channel: "rankings:updates:{tournament_id}"
Message: { userId, newRank, oldRank, score, timestamp }

// Subscribe (SSE Service)
SUBSCRIBE rankings:updates:t1

// Publish (Ranking Service)
PUBLISH rankings:updates:t1 '{"userId":"alice","newRank":1,"oldRank":2,"score":9600}'

// Global notifications
Channel: "notifications:{user_id}"
Message: { type, message, data }
```

### Redis Configuration

```yaml
Redis Cluster Configuration (for 100K users):

Cluster Setup:
  - 3 Master nodes (sharded)
  - 3 Replica nodes (high availability)
  - Hash slots: 16384 (distributed across masters)

Per Node:
  Memory: 16GB
  Max Connections: 10000
  Eviction Policy: allkeys-lru

Total Cluster:
  Active Memory: 48GB
  Replica Memory: 48GB
  Total: 96GB

Persistence:
  RDB: Save snapshots every 5 minutes if 100 changes
  AOF: Append-only file with everysec fsync

Performance Settings:
  maxmemory: 16gb
  maxmemory-policy: allkeys-lru
  tcp-backlog: 511
  timeout: 0
  tcp-keepalive: 300

Cluster Settings:
  cluster-enabled: yes
  cluster-node-timeout: 5000
  cluster-replica-validity-factor: 10
```

---

## ⚙️ Backend Services

### 1. Auth Service (Port 3001)

#### Service Structure

```typescript
// src/auth/auth.module.ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { TypeOrmModule } from '@nestjs/typeorm';
import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { User } from '../users/user.entity';
import { JwtStrategy } from './strategies/jwt.strategy';
import { GoogleStrategy } from './strategies/google.strategy';
import { RedisModule } from '../redis/redis.module';

@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.register({
      secret: process.env.JWT_SECRET,
      signOptions: { expiresIn: '15m' }
    }),
    RedisModule
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy, GoogleStrategy],
  exports: [AuthService, JwtStrategy]
})
export class AuthModule {}
```

#### Auth Service Implementation

```typescript
// src/auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import * as bcrypt from 'bcrypt';
import { v4 as uuidv4 } from 'uuid';
import { User } from '../users/user.entity';
import { RedisService } from '../redis/redis.service';

@Injectable()
export class AuthService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>,
    private jwtService: JwtService,
    private redisService: RedisService
  ) {}

  /**
   * Register new user with email/password
   */
  async register(email: string, username: string, password: string) {
    // Check if user exists
    const existing = await this.userRepository.findOne({
      where: [{ email }, { username }]
    });

    if (existing) {
      throw new UnauthorizedException('User already exists');
    }

    // Hash password
    const passwordHash = await bcrypt.hash(password, 10);

    // Create user
    const user = this.userRepository.create({
      email,
      username,
      passwordHash
    });

    await this.userRepository.save(user);

    // Generate tokens
    return this.generateTokens(user);
  }

  /**
   * Login with email/password
   */
  async login(email: string, password: string) {
    // Find user
    const user = await this.userRepository.findOne({ where: { email } });

    if (!user || !user.passwordHash) {
      throw new UnauthorizedException('Invalid credentials');
    }

    // Verify password
    const isValid = await bcrypt.compare(password, user.passwordHash);

    if (!isValid) {
      throw new UnauthorizedException('Invalid credentials');
    }

    // Update last login
    await this.userRepository.update(user.id, {
      lastLogin: new Date()
    });

    // Generate tokens
    return this.generateTokens(user);
  }

  /**
   * OAuth login (Google, Steam, Discord)
   */
  async oauthLogin(profile: any, provider: string) {
    const { id: oauthId, emails, displayName, photos } = profile;
    const email = emails?.[0]?.value;

    // Find or create user
    let user = await this.userRepository.findOne({
      where: { oauthProvider: provider, oauthId }
    });

    if (!user) {
      // Create new user from OAuth profile
      user = this.userRepository.create({
        email,
        username: displayName || `user_${oauthId.slice(0, 8)}`,
        oauthProvider: provider,
        oauthId,
        avatarUrl: photos?.[0]?.value,
        isEmailVerified: true // OAuth emails are verified
      });

      await this.userRepository.save(user);
    }

    // Update last login
    await this.userRepository.update(user.id, {
      lastLogin: new Date()
    });

    return this.generateTokens(user);
  }

  /**
   * Generate access + refresh tokens
   */
  private async generateTokens(user: User) {
    const jti = uuidv4(); // Unique token ID for revocation

    // Access token (15 minutes)
    const accessToken = this.jwtService.sign({
      sub: user.id,
      username: user.username,
      email: user.email,
      jti
    });

    // Refresh token (7 days)
    const refreshToken = uuidv4();

    // Store refresh token in Redis
    await this.redisService.set(
      `refresh_token:${refreshToken}`,
      JSON.stringify({
        userId: user.id,
        jti,
        issuedAt: Date.now()
      }),
      'EX',
      7 * 24 * 60 * 60 // 7 days
    );

    return {
      accessToken,
      refreshToken,
      expiresIn: 900, // 15 minutes
      user: {
        id: user.id,
        username: user.username,
        email: user.email,
        avatarUrl: user.avatarUrl
      }
    };
  }

  /**
   * Refresh access token using refresh token
   */
  async refreshToken(refreshToken: string) {
    // Get refresh token data from Redis
    const data = await this.redisService.get(`refresh_token:${refreshToken}`);

    if (!data) {
      throw new UnauthorizedException('Invalid refresh token');
    }

    const { userId } = JSON.parse(data);

    // Get user
    const user = await this.userRepository.findOne({ where: { id: userId } });

    if (!user || !user.isActive) {
      throw new UnauthorizedException('User not found or inactive');
    }

    // Generate new access token
    const jti = uuidv4();
    const accessToken = this.jwtService.sign({
      sub: user.id,
      username: user.username,
      email: user.email,
      jti
    });

    // Update refresh token with new JTI
    await this.redisService.set(
      `refresh_token:${refreshToken}`,
      JSON.stringify({
        userId: user.id,
        jti,
        issuedAt: Date.now()
      }),
      'EX',
      7 * 24 * 60 * 60
    );

    return {
      accessToken,
      expiresIn: 900
    };
  }

  /**
   * Validate JWT token
   */
  async validateToken(payload: any): Promise<User> {
    const { sub: userId, jti } = payload;

    // Check if token is blacklisted
    const isBlacklisted = await this.redisService.exists(`blacklist:${jti}`);

    if (isBlacklisted) {
      throw new UnauthorizedException('Token has been revoked');
    }

    // Get user
    const user = await this.userRepository.findOne({ where: { id: userId } });

    if (!user || !user.isActive) {
      throw new UnauthorizedException('User not found or inactive');
    }

    return user;
  }

  /**
   * Logout (revoke tokens)
   */
  async logout(accessToken: string, refreshToken: string) {
    // Decode access token to get JTI and expiry
    const payload = this.jwtService.decode(accessToken) as any;
    const ttl = payload.exp - Math.floor(Date.now() / 1000);

    if (ttl > 0) {
      // Add access token to blacklist
      await this.redisService.set(
        `blacklist:${payload.jti}`,
        '1',
        'EX',
        ttl
      );
    }

    // Delete refresh token
    await this.redisService.del(`refresh_token:${refreshToken}`);

    return { message: 'Logged out successfully' };
  }
}
```

#### JWT Strategy

```typescript
// src/auth/strategies/jwt.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { AuthService } from '../auth.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: process.env.JWT_SECRET
    });
  }

  async validate(payload: any) {
    // This is called after JWT is verified
    // Validate user and check if token is blacklisted
    const user = await this.authService.validateToken(payload);

    if (!user) {
      throw new UnauthorizedException();
    }

    return user; // Attached to request.user
  }
}
```

#### Google OAuth Strategy

```typescript
// src/auth/strategies/google.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy, VerifyCallback } from 'passport-google-oauth20';
import { AuthService } from '../auth.service';

@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor(private authService: AuthService) {
    super({
      clientID: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
      callbackURL: process.env.GOOGLE_CALLBACK_URL,
      scope: ['email', 'profile']
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: any,
    done: VerifyCallback
  ): Promise<any> {
    const user = await this.authService.oauthLogin(profile, 'google');
    done(null, user);
  }
}
```

#### Auth Controller

```typescript
// src/auth/auth.controller.ts
import {
  Controller,
  Post,
  Get,
  Body,
  UseGuards,
  Req,
  Res,
  HttpStatus
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { AuthService } from './auth.service';
import { RegisterDto, LoginDto, RefreshTokenDto } from './dto';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('register')
  async register(@Body() dto: RegisterDto) {
    return this.authService.register(dto.email, dto.username, dto.password);
  }

  @Post('login')
  async login(@Body() dto: LoginDto) {
    return this.authService.login(dto.email, dto.password);
  }

  @Post('refresh')
  async refreshToken(@Body() dto: RefreshTokenDto) {
    return this.authService.refreshToken(dto.refreshToken);
  }

  @Post('logout')
  @UseGuards(AuthGuard('jwt'))
  async logout(@Req() req, @Body() dto: { refreshToken: string }) {
    const accessToken = req.headers.authorization.split(' ')[1];
    return this.authService.logout(accessToken, dto.refreshToken);
  }

  // OAuth routes
  @Get('google')
  @UseGuards(AuthGuard('google'))
  async googleAuth() {
    // Redirects to Google
  }

  @Get('google/callback')
  @UseGuards(AuthGuard('google'))
  async googleAuthCallback(@Req() req, @Res() res) {
    // Successful authentication
    const { accessToken, refreshToken } = req.user;

    // Redirect to frontend with tokens
    res.redirect(
      `${process.env.FRONTEND_URL}/auth/callback?token=${accessToken}&refresh=${refreshToken}`
    );
  }
}
```

---

### 2. Ranking Service (Port 3002)

This is the most critical service for calculating and serving rankings.

#### Ranking Service Implementation

```typescript
// src/rankings/rankings.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { TournamentScore } from './entities/tournament-score.entity';
import { RedisService } from '../redis/redis.service';
import { KafkaProducer } from '../kafka/kafka.producer';

@Injectable()
export class RankingsService {
  constructor(
    @InjectRepository(TournamentScore)
    private scoreRepository: Repository<TournamentScore>,
    private redisService: RedisService,
    private kafkaProducer: KafkaProducer
  ) {}

  /**
   * Get user's rank in tournament
   * Uses multi-level caching for <100ms response time
   */
  async getUserRank(tournamentId: string, userId: string) {
    // Level 1: Check Redis cache (individual user rank)
    const cacheKey = `user:rank:${tournamentId}:${userId}`;
    const cached = await this.redisService.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // Level 2: Check Redis leaderboard (sorted set)
    const rank = await this.redisService.zrevrank(
      `leaderboard:${tournamentId}:full`,
      userId
    );

    if (rank !== null) {
      const score = await this.redisService.zscore(
        `leaderboard:${tournamentId}:full`,
        userId
      );

      const total = await this.redisService.zcard(
        `leaderboard:${tournamentId}:full`
      );

      const result = {
        rank: rank + 1, // Redis is 0-indexed
        score: parseInt(score),
        total,
        percentile: ((total - rank) / total) * 100
      };

      // Cache for 1 minute
      await this.redisService.set(
        cacheKey,
        JSON.stringify(result),
        'EX',
        60
      );

      return result;
    }

    // Level 3: Query PostgreSQL materialized view (cache miss)
    const result = await this.scoreRepository.query(
      `
      SELECT
        r.rank,
        r.score,
        r.percentile,
        (SELECT COUNT(*) FROM tournament_rankings WHERE tournament_id = $1) as total
      FROM tournament_rankings r
      WHERE r.tournament_id = $1 AND r.user_id = $2
      `,
      [tournamentId, userId]
    );

    if (!result || result.length === 0) {
      return null;
    }

    const ranking = result[0];

    // Update Redis cache
    await this.updateRedisLeaderboard(tournamentId);

    // Cache result
    await this.redisService.set(
      cacheKey,
      JSON.stringify(ranking),
      'EX',
      60
    );

    return ranking;
  }

  /**
   * Get top N rankings
   * Optimized for <100ms response time
   */
  async getTopRankings(tournamentId: string, limit = 100) {
    // Check cache first
    const cacheKey = `leaderboard:${tournamentId}:top${limit}`;
    const cached = await this.redisService.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // Query Redis sorted set
    const userScores = await this.redisService.zrevrange(
      `leaderboard:${tournamentId}:full`,
      0,
      limit - 1,
      'WITHSCORES'
    );

    if (userScores && userScores.length > 0) {
      const rankings = this.parseRedisLeaderboard(userScores);

      // Get usernames from database
      const userIds = rankings.map(r => r.userId);
      const users = await this.getUsersBatch(userIds);

      const result = rankings.map((r, index) => ({
        ...r,
        rank: index + 1,
        username: users[r.userId]?.username,
        avatarUrl: users[r.userId]?.avatarUrl
      }));

      // Cache for 30 seconds
      await this.redisService.set(
        cacheKey,
        JSON.stringify(result),
        'EX',
        30
      );

      return result;
    }

    // Cache miss: Query PostgreSQL materialized view
    const rankings = await this.scoreRepository.query(
      `
      SELECT
        r.user_id as "userId",
        u.username,
        u.avatar_url as "avatarUrl",
        r.score,
        r.rank,
        r.matches_played as "matchesPlayed",
        r.wins,
        r.losses
      FROM tournament_rankings r
      JOIN users u ON r.user_id = u.id
      WHERE r.tournament_id = $1
      ORDER BY r.rank
      LIMIT $2
      `,
      [tournamentId, limit]
    );

    // Update Redis cache
    await this.cacheLeaderboard(tournamentId, rankings);

    // Cache result
    await this.redisService.set(
      cacheKey,
      JSON.stringify(rankings),
      'EX',
      30
    );

    return rankings;
  }

  /**
   * Get leaderboard with pagination
   */
  async getLeaderboard(
    tournamentId: string,
    page = 1,
    limit = 50
  ) {
    const offset = (page - 1) * limit;

    // Query materialized view with pagination
    const [rankings, total] = await Promise.all([
      this.scoreRepository.query(
        `
        SELECT
          r.user_id as "userId",
          u.username,
          u.avatar_url as "avatarUrl",
          r.score,
          r.rank,
          r.matches_played as "matchesPlayed",
          r.wins,
          r.losses
        FROM tournament_rankings r
        JOIN users u ON r.user_id = u.id
        WHERE r.tournament_id = $1
        ORDER BY r.rank
        LIMIT $2 OFFSET $3
        `,
        [tournamentId, limit, offset]
      ),
      this.scoreRepository.query(
        `SELECT COUNT(*) as count FROM tournament_rankings WHERE tournament_id = $1`,
        [tournamentId]
      )
    ]);

    return {
      data: rankings,
      pagination: {
        page,
        limit,
        total: parseInt(total[0].count),
        pages: Math.ceil(parseInt(total[0].count) / limit)
      }
    };
  }

  /**
   * Update user score and recalculate rankings
   * This is called after every match
   */
  async updateScore(
    tournamentId: string,
    userId: string,
    scoreChange: number,
    matchResult: 'win' | 'loss' | 'draw',
    opponentId?: string
  ) {
    // 1. Get current score
    const score = await this.scoreRepository.findOne({
      where: { tournamentId, userId }
    });

    if (!score) {
      throw new Error('User not in tournament');
    }

    const oldRank = await this.getUserRank(tournamentId, userId);
    const newScore = score.score + scoreChange;

    // 2. Update PostgreSQL (source of truth)
    await this.scoreRepository.update(
      { tournamentId, userId },
      {
        score: newScore,
        matchesPlayed: score.matchesPlayed + 1,
        wins: matchResult === 'win' ? score.wins + 1 : score.wins,
        losses: matchResult === 'loss' ? score.losses + 1 : score.losses,
        draws: matchResult === 'draw' ? score.draws + 1 : score.draws,
        lastMatchAt: new Date()
      }
    );

    // 3. Update Redis leaderboard (atomic operation)
    await this.redisService.zadd(
      `leaderboard:${tournamentId}:full`,
      newScore,
      userId
    );

    // 4. Get new rank
    const newRank = await this.getUserRank(tournamentId, userId);

    // 5. Invalidate caches
    await this.invalidateCaches(tournamentId, userId);

    // 6. Record match in history
    await this.recordMatch(
      tournamentId,
      userId,
      opponentId,
      score.score,
      newScore,
      scoreChange,
      matchResult
    );

    // 7. Publish rank change event to Kafka (if rank changed)
    if (oldRank && oldRank.rank !== newRank.rank) {
      await this.kafkaProducer.send({
        topic: 'ranking.updates',
        messages: [
          {
            key: tournamentId,
            value: JSON.stringify({
              tournamentId,
              userId,
              oldRank: oldRank.rank,
              newRank: newRank.rank,
              oldScore: score.score,
              newScore,
              scoreChange,
              timestamp: Date.now()
            })
          }
        ]
      });
    }

    return {
      newScore,
      newRank: newRank.rank,
      scoreChange,
      matchResult
    };
  }

  /**
   * Get user's rank neighbors (users around them)
   */
  async getRankNeighbors(
    tournamentId: string,
    userId: string,
    range = 5
  ) {
    // Get user's rank
    const rank = await this.redisService.zrevrank(
      `leaderboard:${tournamentId}:full`,
      userId
    );

    if (rank === null) {
      return [];
    }

    // Get users around this rank
    const start = Math.max(0, rank - range);
    const end = rank + range;

    const userScores = await this.redisService.zrevrange(
      `leaderboard:${tournamentId}:full`,
      start,
      end,
      'WITHSCORES'
    );

    const rankings = this.parseRedisLeaderboard(userScores);

    // Get usernames
    const userIds = rankings.map(r => r.userId);
    const users = await this.getUsersBatch(userIds);

    return rankings.map((r, index) => ({
      ...r,
      rank: start + index + 1,
      username: users[r.userId]?.username,
      avatarUrl: users[r.userId]?.avatarUrl,
      isCurrentUser: r.userId === userId
    }));
  }

  /**
   * Helper: Update Redis leaderboard from database
   */
  private async updateRedisLeaderboard(tournamentId: string) {
    const scores = await this.scoreRepository.find({
      where: { tournamentId },
      select: ['userId', 'score']
    });

    if (scores.length === 0) return;

    // Batch update Redis sorted set
    const args = scores.flatMap(s => [s.score, s.userId]);

    await this.redisService.zadd(
      `leaderboard:${tournamentId}:full`,
      ...args
    );
  }

  /**
   * Helper: Parse Redis leaderboard response
   */
  private parseRedisLeaderboard(userScores: string[]) {
    const rankings = [];

    for (let i = 0; i < userScores.length; i += 2) {
      rankings.push({
        userId: userScores[i],
        score: parseInt(userScores[i + 1])
      });
    }

    return rankings;
  }

  /**
   * Helper: Get users in batch
   */
  private async getUsersBatch(userIds: string[]) {
    const users = await this.scoreRepository.query(
      `SELECT id, username, avatar_url FROM users WHERE id = ANY($1)`,
      [userIds]
    );

    return users.reduce((acc, user) => {
      acc[user.id] = user;
      return acc;
    }, {});
  }

  /**
   * Helper: Cache leaderboard in Redis
   */
  private async cacheLeaderboard(tournamentId: string, rankings: any[]) {
    const args = rankings.flatMap(r => [r.score, r.userId]);

    await this.redisService.zadd(
      `leaderboard:${tournamentId}:full`,
      ...args
    );

    // Set TTL
    await this.redisService.expire(
      `leaderboard:${tournamentId}:full`,
      300 // 5 minutes
    );
  }

  /**
   * Helper: Invalidate caches after score update
   */
  private async invalidateCaches(tournamentId: string, userId: string) {
    // Invalidate user ranking cache
    await this.redisService.del(`user:rank:${tournamentId}:${userId}`);

    // Invalidate top rankings cache
    const keys = await this.redisService.keys(`leaderboard:${tournamentId}:top*`);
    if (keys.length > 0) {
      await this.redisService.del(...keys);
    }
  }

  /**
   * Helper: Record match in history
   */
  private async recordMatch(
    tournamentId: string,
    userId: string,
    opponentId: string,
    scoreBefore: number,
    scoreAfter: number,
    scoreChange: number,
    result: string
  ) {
    await this.scoreRepository.query(
      `
      INSERT INTO match_history (
        tournament_id, user_id, opponent_id,
        score_before, score_after, score_change,
        result, played_at
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
      `,
      [tournamentId, userId, opponentId, scoreBefore, scoreAfter, scoreChange, result]
    );
  }

  /**
   * Background job: Refresh materialized view
   * Runs every 30 seconds via cron
   */
  @Cron('*/30 * * * * *') // Every 30 seconds
  async refreshRankings() {
    try {
      await this.scoreRepository.query(
        'REFRESH MATERIALIZED VIEW CONCURRENTLY tournament_rankings'
      );
      console.log('[Cron] Materialized view refreshed');
    } catch (error) {
      console.error('[Cron] Failed to refresh materialized view:', error);
    }
  }
}
```

Due to character limits, I'll continue with the remaining services in the next section. The document now contains:

1. ✅ Overview & Requirements
2. ✅ Technology Stack
3. ✅ System Architecture
4. ✅ Complete Database Design (PostgreSQL + Redis)
5. ✅ Auth Service (complete implementation)
6. ✅ Ranking Service (complete implementation)

Would you like me to continue with the remaining sections (Tournament Service, Real-Time SSE Service, Frontend Implementation, Scaling Strategy, Deployment, etc.)?
# Klub — Architecture Document

## Table of Contents

1. [System Overview](#1-system-overview)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Backend Architecture](#3-backend-architecture)
4. [Database Design](#4-database-design)
5. [API Design](#5-api-design)
6. [Mobile App Architecture](#6-mobile-app-architecture)
7. [Admin Panel Architecture](#7-admin-panel-architecture)
8. [Real-Time System](#8-real-time-system)
9. [Authentication & Authorization](#9-authentication--authorization)
10. [Game Engine](#10-game-engine)
11. [Sports Data Pipeline & AI Automation](#11-sports-data-pipeline--ai-automation)
12. [Deployment & Infrastructure](#12-deployment--infrastructure)
13. [Monitoring & Observability](#13-monitoring--observability)

---

## 1. System Overview

Klub is a team-based fantasy investment game for the FIFA World Cup. Players invest in national teams using a virtual budget, earn points from match results and tournament progression, and compete on a global leaderboard.

### Core Domain Concepts
- **Portfolio**: A player's collection of teams (exactly 4 in group stage, reducing in knockout rounds)
- **Credits**: Virtual currency used to buy/sell teams
- **Team Market**: Dynamic pricing system where team values fluctuate based on match results
- **Scoring**: Points earned from team match performance and stage advancement
- **Gameweek**: A round of matches within a tournament stage; trades are scoped per gameweek

### Key Business Rules
| Rule | Detail |
|------|--------|
| Starting budget | 100 Credits |
| Portfolio size | 4 teams (group) → 3 (R32/R16) → 2 (QF+) |
| Price movement | Win +1, Draw +0.5, Loss -1, Advance +2 |
| Price ceiling | Per-team cap (e.g., 60 Credits) |
| Exit value | 70% of current price when team eliminated |
| Sell spread | 95% of current price (5% fee) |
| Free trades | 1/gameweek (group), 2/round (knockout) |
| Resets | 2 per tournament (sells all at 95%, forfeits trade that gameweek) |
| Tiebreaker | 1st: fewer trades, 2nd: higher remaining budget |

---

## 2. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                         CLIENTS                              │
├──────────────────────────────────────────────────────────────┤
│   Flutter App (iOS / Android)    │   Next.js Admin Panel     │
└────────────┬─────────────────────────────────┬───────────────┘
             │         HTTPS / WebSocket       │
             ▼                                 ▼
┌──────────────────────────────────────────────────────────────┐
│                   Go API Server (Gin)                        │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  Middleware: Auth · CORS · Rate Limit · Locale · Log   │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  Handlers    │  │  Services    │  │  Repositories    │   │
│  │              │  │              │  │                  │   │
│  │ • Portfolio  │→ │ • Game       │→ │ • PostgreSQL     │   │
│  │ • Market     │  │ • Market     │  │ • Redis          │   │
│  │ • Match      │  │ • Scoring    │  │                  │   │
│  │ • Leaderboard│  │ • Leaderboard│  └────────┬─────────┘   │
│  │ • Auth       │  │ • Trade      │           │             │
│  │ • Admin/*    │  │ • Cron       │           │             │
│  │ • WebSocket  │  │ • Notification│          │             │
│  └──────────────┘  │ • SportsData │          │             │
│                    └──────────────┘           │             │
└────────────────────────────────┬──────────────┘─────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
   ┌──────────────────┐ ┌───────────────┐ ┌────────────────┐
   │  PostgreSQL 17   │ │   Redis 7     │ │  External APIs │
   │                  │ │               │ │                │
   │ • Users          │ │ • Session     │ │ • XEX Exchange │
   │ • Teams          │ │ • Rate limits │ │ • Sports Data  │
   │ • Portfolios     │ │ • Leaderboard │ │ • Firebase FCM │
   │ • Matches        │ │   cache       │ │                │
   │ • Transactions   │ │ • Market lock │ │                │
   │ • Leaderboard    │ │               │ │                │
   └──────────────────┘ └───────────────┘ └────────────────┘
```

### Design Principles
1. **Same stack as XEX Play**: Go + Flutter + Next.js + PostgreSQL + Redis
2. **Shared JWT with XEX Exchange**: No separate auth system
3. **Database isolation**: Klub has its own database, separate from Exchange and Play
4. **Cache-first for hot paths**: Leaderboards, team prices, and market state cached in Redis
5. **Real-time via WebSocket**: Live price updates, match events, leaderboard changes
6. **Clean layering**: Handler → Service → Repository (no shortcuts)
7. **API keys in DB, not env**: All third-party API keys (Anthropic, Odds API, FCM, Exchange service key) stored in `settings` table and managed via admin panel — same pattern as XEX Play. Only infra secrets (DATABASE_URL, REDIS_URL, JWT_SECRET) are env vars.
8. **i18n from day one**: Every user-facing string in Flutter uses ARB localization keys. All backend user-facing data uses JSONB with per-language values. No hardcoded strings — ever.
9. **AI-assisted automation**: Claude Haiku 4.5 (same account as XEX Play) for translations, tier suggestions, match previews, notification content. The Odds API (same account) for match data and live scores.

---

## 3. Backend Architecture

### Module: `github.com/xex-exchange/klub-api`

### Directory Layout
```
backend/
├── cmd/server/main.go              # Entry point, DI wiring
├── internal/
│   ├── config/
│   │   ├── config.go               # Env-based configuration (infra only)
│   │   └── settings.go             # DB-backed settings loader (API keys, feature flags)
│   ├── domain/                     # Domain models (pure structs, no DB deps)
│   │   ├── user.go
│   │   ├── team.go
│   │   ├── portfolio.go
│   │   ├── match.go
│   │   ├── tournament.go
│   │   ├── transaction.go
│   │   ├── leaderboard.go
│   │   ├── gameweek.go
│   │   └── ...
│   ├── handler/                    # HTTP route handlers
│   │   ├── auth_handler.go
│   │   ├── portfolio_handler.go
│   │   ├── market_handler.go
│   │   ├── match_handler.go
│   │   ├── leaderboard_handler.go
│   │   ├── admin/
│   │   │   ├── tournament_handler.go
│   │   │   ├── team_handler.go
│   │   │   ├── match_handler.go
│   │   │   ├── user_handler.go
│   │   │   ├── settings_handler.go      # DB-backed settings CRUD (API keys, flags)
│   │   │   ├── automation_handler.go    # Job triggers, logs viewer
│   │   │   └── translation_handler.go   # JSONB translation CRUD, AI batch translate
│   │   └── ws/
│   │       └── ws_handler.go
│   ├── middleware/
│   │   ├── auth.go                 # JWT validation (shared with Exchange)
│   │   ├── admin.go                # Role-based access
│   │   ├── rate_limiter.go         # Redis-based rate limiting
│   │   ├── cors.go
│   │   ├── locale.go               # Accept-Language detection
│   │   └── logger.go               # Request logging
│   ├── service/                    # Business logic
│   │   ├── game_service.go         # Core game orchestration
│   │   ├── market_service.go       # Buy/sell logic, price calculations
│   │   ├── pricing_service.go      # Price movement engine
│   │   ├── scoring_service.go      # Points calculation engine
│   │   ├── portfolio_service.go    # Portfolio management
│   │   ├── trade_service.go        # Trade execution & validation
│   │   ├── leaderboard_service.go  # Ranking calculations
│   │   ├── tournament_service.go   # Stage management, team elimination
│   │   ├── match_service.go        # Match result processing
│   │   ├── notification_service.go # Push notifications
│   │   ├── cron_service.go         # Background scheduled jobs
│   │   ├── sports_data_service.go  # The Odds API integration (match data, live scores)
│   │   ├── ai_service.go           # Claude Haiku 4.5 (translations, tier suggestions, previews)
│   │   ├── ai_prompts.go           # Prompt templates for Claude API calls
│   │   ├── settings_service.go     # DB-backed settings CRUD (API keys, feature flags)
│   │   └── reset_service.go        # Portfolio reset logic
│   ├── repository/
│   │   ├── postgres/
│   │   │   ├── user_repo.go
│   │   │   ├── team_repo.go
│   │   │   ├── portfolio_repo.go
│   │   │   ├── match_repo.go
│   │   │   ├── transaction_repo.go
│   │   │   ├── leaderboard_repo.go
│   │   │   └── ...
│   │   └── redis/
│   │       ├── cache_repo.go
│   │       ├── leaderboard_cache.go
│   │       ├── market_lock.go
│   │       └── rate_limit.go
│   ├── exchange/
│   │   └── client.go               # XEX Exchange API client
│   ├── external/
│   │   ├── oddsapi/
│   │   │   └── client.go           # The Odds API wrapper (dash.the-odds-api.com)
│   │   └── anthropic/
│   │       └── client.go           # Anthropic Claude API client (Haiku 4.5)
│   └── pkg/
│       ├── ws/hub.go               # WebSocket hub
│       ├── jwt/jwt.go              # JWT parsing
│       ├── validator/validator.go  # Request validation
│       └── response/response.go   # Standardized API responses
├── migrations/                     # golang-migrate SQL files
├── docker/
│   ├── docker-compose.yml
│   └── api.Dockerfile
├── Makefile
└── go.mod
```

### Key Services

#### GameService
Orchestrates the overall game flow: tournament lifecycle, stage transitions, gameweek progression. Coordinates between MarketService, ScoringService, and TournamentService.

#### MarketService
Handles all buy/sell operations:
- Validates budget sufficiency
- Enforces portfolio size limits per stage
- Applies 5% sell spread
- Locks market during live matches (Redis distributed lock)
- Enforces trade allowance (1/gameweek group, 2/round knockout)

#### PricingService
Manages team price movements:
- Applies match result adjustments (+1 win, +0.5 draw, -1 loss)
- Applies advancement bonus (+2)
- Enforces price ceiling per team
- Calculates exit values (70% on elimination)

#### ScoringService
Calculates player points:
- Match performance: win/draw/loss, goals scored/conceded, clean sheets
- Stage multipliers: 1.0x (group), 1.5x (R32/R16/QF), 2.0x (semi/final)
- Advancement bonuses: 3/6/9/12/16/25 fixed points
- Extra-time (+2) and penalty shootout (+1) bonuses
- Stats tracked through 120 minutes (not penalty shootout goals)

#### TradeService
Executes and validates individual trades:
- Checks trade allowance remaining
- Validates portfolio constraints (team count, budget)
- Records transaction history (for tiebreaker: fewer trades = better rank)
- Handles auto-sell on team elimination (Exit Value)

#### ResetService
Handles full portfolio resets:
- Sells all teams at 95% value
- Allows new team purchases
- Consumes trade allowance for the gameweek
- Limited to 2 resets per tournament per player

#### TournamentService
Manages tournament structure:
- Stage definitions (group, R32, R16, QF, SF, Final)
- Team elimination processing
- Portfolio size reduction enforcement
- Gameweek scheduling

---

## 4. Database Design

### Entity Relationship Overview

```
tournaments ──< stages ──< gameweeks ──< matches
                                            │
teams ─────────────────────────────────────┘
  │
  ├──< team_prices (price history)
  │
  └──< portfolio_teams >── portfolios >── users
                                │
                                ├──< transactions (buy/sell/exit/reset)
                                ├──< user_points (per-match point breakdown)
                                └──< leaderboard_entries
```

### Core Tables

#### `users`
```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    xex_user_id     UUID NOT NULL UNIQUE,       -- FK to XEX Exchange
    display_name    VARCHAR(100),
    email           VARCHAR(255),
    avatar_url      TEXT,
    role            VARCHAR(20) DEFAULT 'user',  -- user | admin | moderator
    language        VARCHAR(5) DEFAULT 'en',     -- en, fa, ar, tr, es, fr
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `tournaments`
```sql
CREATE TABLE tournaments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            JSONB NOT NULL,              -- {"en": "FIFA World Cup 2026", "fa": "..."}
    slug            VARCHAR(100) UNIQUE NOT NULL,
    status          VARCHAR(20) DEFAULT 'upcoming', -- upcoming | active | completed
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    starting_budget DECIMAL(10,2) DEFAULT 100.00,
    sell_spread     DECIMAL(5,4) DEFAULT 0.05,   -- 5%
    exit_value_pct  DECIMAL(5,4) DEFAULT 0.70,   -- 70%
    max_resets      INTEGER DEFAULT 2,
    config          JSONB,                       -- flexible tournament-level config
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `stages`
```sql
CREATE TABLE stages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tournament_id   UUID NOT NULL REFERENCES tournaments(id),
    name            JSONB NOT NULL,              -- {"en": "Group Stage", ...}
    slug            VARCHAR(50) NOT NULL,        -- group, r32, r16, qf, sf, final
    stage_order     INTEGER NOT NULL,            -- 1=group, 2=r32, ...
    score_multiplier DECIMAL(3,1) NOT NULL,      -- 1.0, 1.5, 2.0
    max_teams       INTEGER NOT NULL,            -- 4, 3, 3, 2, 2, 2
    free_trades     INTEGER NOT NULL,            -- 1 (group), 2 (knockout)
    price_ceiling   DECIMAL(10,2),               -- max price for any team in this stage
    status          VARCHAR(20) DEFAULT 'upcoming',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `gameweeks`
```sql
CREATE TABLE gameweeks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stage_id        UUID NOT NULL REFERENCES stages(id),
    tournament_id   UUID NOT NULL REFERENCES tournaments(id),
    week_number     INTEGER NOT NULL,
    name            JSONB,                       -- {"en": "Matchday 1"}
    start_date      TIMESTAMPTZ NOT NULL,
    end_date        TIMESTAMPTZ NOT NULL,
    market_locked   BOOLEAN DEFAULT false,
    status          VARCHAR(20) DEFAULT 'upcoming',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `teams`
```sql
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tournament_id   UUID NOT NULL REFERENCES tournaments(id),
    name            JSONB NOT NULL,              -- {"en": "Brazil", "fa": "برزیل"}
    short_name      VARCHAR(5) NOT NULL,         -- BRA
    flag_url        TEXT,
    tier            VARCHAR(1) NOT NULL,         -- S, A, B, C
    initial_price   DECIMAL(10,2) NOT NULL,      -- e.g. 45.00
    current_price   DECIMAL(10,2) NOT NULL,
    price_ceiling   DECIMAL(10,2) NOT NULL,      -- e.g. 60.00
    is_eliminated   BOOLEAN DEFAULT false,
    eliminated_at   TIMESTAMPTZ,
    eliminated_stage_id UUID REFERENCES stages(id),
    metadata        JSONB,                       -- odds, group assignment, etc.
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `team_prices` (price history)
```sql
CREATE TABLE team_prices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id),
    match_id        UUID REFERENCES matches(id),
    price_before    DECIMAL(10,2) NOT NULL,
    price_after     DECIMAL(10,2) NOT NULL,
    change_amount   DECIMAL(10,2) NOT NULL,
    change_reason   VARCHAR(30) NOT NULL,        -- win, draw, loss, advancement
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `matches`
```sql
CREATE TABLE matches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tournament_id   UUID NOT NULL REFERENCES tournaments(id),
    stage_id        UUID NOT NULL REFERENCES stages(id),
    gameweek_id     UUID NOT NULL REFERENCES gameweeks(id),
    home_team_id    UUID NOT NULL REFERENCES teams(id),
    away_team_id    UUID NOT NULL REFERENCES teams(id),
    kickoff_time    TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) DEFAULT 'scheduled', -- scheduled | live | finished | cancelled
    home_score      INTEGER,
    away_score      INTEGER,
    home_score_et   INTEGER,                     -- extra time scores
    away_score_et   INTEGER,
    penalty_winner_id UUID REFERENCES teams(id), -- if decided by penalties
    result_data     JSONB,                       -- detailed match events
    is_knockout     BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `portfolios`
```sql
CREATE TABLE portfolios (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    tournament_id   UUID NOT NULL REFERENCES tournaments(id),
    budget          DECIMAL(10,2) NOT NULL DEFAULT 100.00,
    total_points    DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    total_trades    INTEGER NOT NULL DEFAULT 0,
    resets_used     INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, tournament_id)
);
```

#### `portfolio_teams`
```sql
CREATE TABLE portfolio_teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    portfolio_id    UUID NOT NULL REFERENCES portfolios(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    purchase_price  DECIMAL(10,2) NOT NULL,
    purchased_at    TIMESTAMPTZ DEFAULT NOW(),
    sold_at         TIMESTAMPTZ,
    sell_price      DECIMAL(10,2),
    sell_reason     VARCHAR(20),                 -- voluntary, exit_value, reset
    is_active       BOOLEAN DEFAULT true,
    UNIQUE(portfolio_id, team_id, is_active) -- can't own same team twice simultaneously
);
```

#### `transactions`
```sql
CREATE TABLE transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    portfolio_id    UUID NOT NULL REFERENCES portfolios(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    gameweek_id     UUID REFERENCES gameweeks(id),
    type            VARCHAR(20) NOT NULL,        -- buy, sell, exit_value, reset_sell, reset_buy
    amount          DECIMAL(10,2) NOT NULL,      -- positive = credit, negative = debit
    price_at_time   DECIMAL(10,2) NOT NULL,
    fee_amount      DECIMAL(10,2) DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `user_points`
```sql
CREATE TABLE user_points (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    portfolio_id    UUID NOT NULL REFERENCES portfolios(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    match_id        UUID REFERENCES matches(id),
    stage_id        UUID NOT NULL REFERENCES stages(id),
    point_type      VARCHAR(30) NOT NULL,        -- match_win, match_draw, goal_scored,
                                                 -- clean_sheet, goal_conceded,
                                                 -- advancement, extra_time_win, penalty_win
    base_points     DECIMAL(10,2) NOT NULL,
    multiplier      DECIMAL(3,1) NOT NULL,       -- stage multiplier
    total_points    DECIMAL(10,2) NOT NULL,      -- base * multiplier
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `leaderboard_entries`
```sql
CREATE TABLE leaderboard_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    portfolio_id    UUID NOT NULL REFERENCES portfolios(id),
    tournament_id   UUID NOT NULL REFERENCES tournaments(id),
    total_points    DECIMAL(10,2) NOT NULL DEFAULT 0,
    total_trades    INTEGER NOT NULL DEFAULT 0,
    remaining_budget DECIMAL(10,2) NOT NULL DEFAULT 0,
    rank            INTEGER,
    previous_rank   INTEGER,
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_leaderboard_tournament_rank ON leaderboard_entries(tournament_id, rank);
```

#### `trade_allowances`
```sql
CREATE TABLE trade_allowances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    portfolio_id    UUID NOT NULL REFERENCES portfolios(id),
    gameweek_id     UUID NOT NULL REFERENCES gameweeks(id),
    allowed         INTEGER NOT NULL,            -- 1 or 2
    used            INTEGER NOT NULL DEFAULT 0,
    reset_used      BOOLEAN DEFAULT false,       -- if reset was used this gameweek
    UNIQUE(portfolio_id, gameweek_id)
);
```

#### `fcm_tokens`
```sql
CREATE TABLE fcm_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    token           TEXT NOT NULL UNIQUE,
    device_info     JSONB,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

#### `settings`
Stores all third-party API keys and feature flags. Managed via the admin panel Settings page — **not environment variables**. This allows runtime key rotation without redeployment (same pattern as XEX Play).

```sql
CREATE TABLE settings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key             VARCHAR(100) UNIQUE NOT NULL,  -- e.g. ANTHROPIC_API_KEY, ODDS_API_KEY,
                                                   -- EXCHANGE_SERVICE_KEY, FCM_CREDENTIALS_JSON,
                                                   -- AUTO_SPORTS_ENABLED, AI_TRANSLATIONS_ENABLED
    value           TEXT NOT NULL,
    description     TEXT,                          -- human-readable description for admin UI
    is_secret       BOOLEAN DEFAULT false,         -- masked in admin UI when true
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

**Pre-seeded settings keys:**
| Key | Description | Secret? |
|-----|-------------|---------|
| `ANTHROPIC_API_KEY` | Claude API key (same account as XEX Play) | Yes |
| `ODDS_API_KEY` | The Odds API key (same account as XEX Play) | Yes |
| `EXCHANGE_API_URL` | XEX Exchange base URL | No |
| `EXCHANGE_SERVICE_KEY` | Service-to-service auth key | Yes |
| `FCM_CREDENTIALS_JSON` | Firebase service account JSON | Yes |
| `AUTO_SPORTS_ENABLED` | Enable auto match data fetching | No |
| `AI_TRANSLATIONS_ENABLED` | Enable AI-powered translations | No |
| `AI_TIER_SUGGESTIONS_ENABLED` | Enable AI-powered tier suggestions | No |
| `LIVE_SCORE_POLL_INTERVAL` | Live score polling interval in seconds | No |

#### `automation_logs`
Tracks all automation and AI job executions for visibility in the admin panel.

```sql
CREATE TABLE automation_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_name        VARCHAR(100) NOT NULL,       -- fetch_matches, translate_teams,
                                                 -- suggest_tiers, generate_previews,
                                                 -- poll_live_scores, process_result
    status          VARCHAR(20) NOT NULL,         -- running, completed, failed
    source          VARCHAR(20) DEFAULT 'auto',   -- auto | manual (triggered from admin)
    input_data      JSONB,                        -- job parameters
    output_data     JSONB,                        -- job results summary
    error_message   TEXT,
    started_at      TIMESTAMPTZ DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_automation_logs_job ON automation_logs(job_name, started_at DESC);
```

### Key Indexes
```sql
-- Fast portfolio lookups
CREATE INDEX idx_portfolio_user_tournament ON portfolios(user_id, tournament_id);
CREATE INDEX idx_portfolio_teams_active ON portfolio_teams(portfolio_id) WHERE is_active = true;

-- Transaction history
CREATE INDEX idx_transactions_portfolio ON transactions(portfolio_id, created_at DESC);
CREATE INDEX idx_transactions_gameweek ON transactions(gameweek_id, user_id);

-- Match scheduling
CREATE INDEX idx_matches_kickoff ON matches(kickoff_time, status);
CREATE INDEX idx_matches_stage ON matches(stage_id, status);

-- Points aggregation
CREATE INDEX idx_user_points_portfolio ON user_points(portfolio_id);
CREATE INDEX idx_user_points_match ON user_points(match_id);

-- Team pricing
CREATE INDEX idx_team_prices_team ON team_prices(team_id, created_at DESC);
```

---

## 5. API Design

### Response Envelope
All API responses follow a consistent format:
```json
{
  "success": true,
  "data": { },
  "error": null,
  "meta": { "page": 1, "per_page": 20, "total": 100 }
}
```

### Public Endpoints (Authenticated)

#### Auth
| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/auth/login` | Login with XEX Exchange JWT |
| POST | `/v1/auth/refresh` | Refresh session |
| DELETE | `/v1/auth/logout` | Logout & remove FCM token |

#### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/portfolio` | Get current portfolio (teams, budget, stats) |
| POST | `/v1/portfolio/join` | Join tournament (initializes portfolio with 100 Credits) |
| GET | `/v1/portfolio/history` | Transaction history |
| POST | `/v1/portfolio/reset` | Reset portfolio (max 2 per tournament) |

#### Market
| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/market/teams` | List all teams with current prices |
| GET | `/v1/market/teams/:id` | Team detail with price history |
| POST | `/v1/market/buy` | Buy a team |
| POST | `/v1/market/sell` | Sell a team |
| GET | `/v1/market/status` | Market open/locked status |

#### Matches
| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/matches` | List matches (filterable by stage, gameweek, status) |
| GET | `/v1/matches/:id` | Match detail with events |
| GET | `/v1/matches/live` | Currently live matches |

#### Tournament
| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/tournament` | Current tournament info |
| GET | `/v1/tournament/stages` | All stages with status |
| GET | `/v1/tournament/gameweek/current` | Current gameweek |

#### Leaderboard
| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/leaderboard` | Global leaderboard (paginated) |
| GET | `/v1/leaderboard/me` | Current user's rank & neighbors |

#### User
| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/user/profile` | User profile |
| PUT | `/v1/user/profile` | Update display name, avatar, language |
| POST | `/v1/user/fcm-token` | Register FCM token |

### Admin Endpoints (`/v1/admin/...`)

| Method | Path | Description |
|--------|------|-------------|
| **Tournament** | | |
| POST | `/v1/admin/tournaments` | Create tournament |
| PUT | `/v1/admin/tournaments/:id` | Update tournament |
| POST | `/v1/admin/tournaments/:id/activate` | Start tournament |
| **Stages** | | |
| POST | `/v1/admin/stages` | Create stage |
| PUT | `/v1/admin/stages/:id` | Update stage config |
| POST | `/v1/admin/stages/:id/activate` | Activate stage |
| **Teams** | | |
| POST | `/v1/admin/teams` | Add team to tournament |
| PUT | `/v1/admin/teams/:id` | Update team (name, tier, price) |
| POST | `/v1/admin/teams/:id/eliminate` | Mark team as eliminated |
| PUT | `/v1/admin/teams/:id/price` | Manual price adjustment |
| **Matches** | | |
| POST | `/v1/admin/matches` | Create match |
| PUT | `/v1/admin/matches/:id` | Update match details |
| POST | `/v1/admin/matches/:id/result` | Submit match result (triggers scoring) |
| POST | `/v1/admin/matches/:id/lock-market` | Lock market for match |
| **Gameweeks** | | |
| POST | `/v1/admin/gameweeks` | Create gameweek |
| PUT | `/v1/admin/gameweeks/:id` | Update gameweek |
| **Users** | | |
| GET | `/v1/admin/users` | List users (search, filter) |
| GET | `/v1/admin/users/:id` | User detail with portfolio |
| **Leaderboard** | | |
| POST | `/v1/admin/leaderboard/recalculate` | Force leaderboard recalculation |
| **Settings (DB-backed)** | | |
| GET | `/v1/admin/settings` | List all settings (secrets masked) |
| PUT | `/v1/admin/settings/:key` | Update setting value (API keys, feature flags) |
| **Automation** | | |
| GET | `/v1/admin/automation/logs` | List automation job logs (paginated, filterable) |
| POST | `/v1/admin/automation/fetch-matches` | Manually trigger match data fetch from Odds API |
| POST | `/v1/admin/automation/translate` | Manually trigger AI translation for entities |
| POST | `/v1/admin/automation/suggest-tiers` | AI-suggest team tiers based on current odds |
| POST | `/v1/admin/automation/generate-previews` | AI-generate match previews for gameweek |
| **Translations** | | |
| GET | `/v1/admin/translations/:entity/:id` | Get all translations for an entity (team, stage, etc.) |
| PUT | `/v1/admin/translations/:entity/:id` | Update translations for an entity |
| POST | `/v1/admin/translations/ai-batch` | AI-translate all missing languages for a set of entities |
| **Dashboard** | | |
| GET | `/v1/admin/dashboard/stats` | Overview statistics |

---

## 6. Mobile App Architecture

### Flutter App Structure (Riverpod + GoRouter)

### i18n-First Architecture

**Every user-facing string in the app MUST use Flutter's `intl` localization from the start.** This is a non-negotiable development rule — no hardcoded strings, ever.

#### Setup
- Uses `flutter_localizations` + `intl` package with ARB files
- `l10n.yaml` configured at project root for code generation
- All 6 languages have ARB files from day one, even if initially only English has values (others filled by AI translation)

#### Supported Languages
| Code | Language | Direction |
|------|----------|-----------|
| `en` | English | LTR |
| `fa` | Persian (Farsi) | **RTL** |
| `ar` | Arabic | **RTL** |
| `tr` | Turkish | LTR |
| `es` | Spanish | LTR |
| `fr` | French | LTR |

#### Development Rules
1. **Never** use raw strings: `Text('Buy Team')` — always `Text(context.l10n.buyTeam)`
2. Add every new string to `app_en.arb` with a descriptive key
3. Use ICU message format for plurals and interpolation: `"{count} teams remaining"` → `{count, plural, one{1 team remaining} other{{count} teams remaining}}`
4. Create a `context.l10n` extension getter for clean access: `extension L10nX on BuildContext { AppLocalizations get l10n => AppLocalizations.of(this)!; }`
5. RTL is handled automatically by Flutter's `Directionality` widget — but use `EdgeInsetsDirectional` instead of `EdgeInsets`, `TextDirection`-aware icons, and `start`/`end` instead of `left`/`right`
6. Backend JSONB data (team names, stage names) is resolved client-side: `team.name[userLocale] ?? team.name['en']`

#### ARB File Example (`app_en.arb`)
```json
{
  "@@locale": "en",
  "appTitle": "Klub",
  "buyTeam": "Buy Team",
  "sellTeam": "Sell Team",
  "portfolioTitle": "My Portfolio",
  "budgetRemaining": "{amount} Credits remaining",
  "@budgetRemaining": { "placeholders": { "amount": { "type": "double", "format": "decimalPattern" } } },
  "teamEliminated": "{teamName} has been eliminated",
  "@teamEliminated": { "placeholders": { "teamName": { "type": "String" } } },
  "tradesRemaining": "{count, plural, one{1 trade remaining} other{{count} trades remaining}}",
  "@tradesRemaining": { "placeholders": { "count": { "type": "int" } } },
  "confirmBuy": "Buy {teamName} for {price} Credits?",
  "@confirmBuy": { "placeholders": { "teamName": { "type": "String" }, "price": { "type": "double" } } },
  "marketLocked": "Market is locked during live matches",
  "resetWarning": "This will sell all your teams at 95% value. You have {remaining} resets left.",
  "@resetWarning": { "placeholders": { "remaining": { "type": "int" } } },
  "pointsEarned": "+{points} points from {teamName}",
  "@pointsEarned": { "placeholders": { "points": { "type": "double" }, "teamName": { "type": "String" } } },
  "leaderboardRank": "Rank #{rank}",
  "@leaderboardRank": { "placeholders": { "rank": { "type": "int" } } }
}
```

```
app/lib/
├── main.dart                       # Entry point, Firebase init
├── app.dart                        # MaterialApp setup (with localizationsDelegates)
├── core/
│   ├── config/
│   │   └── env_config.dart         # API URL, environment
│   ├── network/
│   │   ├── api_client.dart         # Dio HTTP client
│   │   └── interceptors.dart       # Auth token, error handling, locale header
│   ├── routing/
│   │   └── router.dart             # GoRouter routes
│   ├── storage/
│   │   └── secure_storage.dart     # JWT token storage
│   ├── theme/
│   │   └── app_theme.dart          # Dark-first design system (RTL-aware)
│   ├── l10n/
│   │   ├── l10n.yaml              # flutter_localizations config
│   │   ├── app_en.arb             # English (base locale)
│   │   ├── app_fa.arb             # Persian
│   │   ├── app_ar.arb             # Arabic
│   │   ├── app_tr.arb             # Turkish
│   │   ├── app_es.arb             # Spanish
│   │   ├── app_fr.arb             # French
│   │   └── l10n_extension.dart    # context.l10n extension getter
│   ├── providers/
│   │   ├── auth_provider.dart      # Auth state
│   │   ├── tournament_provider.dart
│   │   └── ws_provider.dart        # WebSocket connection
│   ├── analytics/                  # Firebase Analytics
│   ├── crashlytics/                # Error reporting
│   └── services/
│       └── notification_service.dart
├── features/
│   ├── auth/
│   │   ├── screens/login_screen.dart
│   │   └── providers/auth_providers.dart
│   ├── portfolio/
│   │   ├── screens/
│   │   │   ├── portfolio_screen.dart       # Main portfolio view
│   │   │   └── transaction_history.dart
│   │   ├── widgets/
│   │   │   ├── team_card.dart              # Team in portfolio
│   │   │   ├── budget_indicator.dart       # Budget remaining
│   │   │   └── portfolio_summary.dart
│   │   └── providers/portfolio_providers.dart
│   ├── market/
│   │   ├── screens/
│   │   │   ├── market_screen.dart          # Browse all teams
│   │   │   ├── team_detail_screen.dart     # Price chart, buy/sell
│   │   │   └── trade_confirm_screen.dart
│   │   ├── widgets/
│   │   │   ├── team_list_tile.dart
│   │   │   ├── price_chart.dart            # Price history graph
│   │   │   ├── tier_filter.dart
│   │   │   └── market_status_banner.dart   # Open/locked indicator
│   │   └── providers/market_providers.dart
│   ├── matches/
│   │   ├── screens/
│   │   │   ├── matches_screen.dart         # Match list
│   │   │   └── match_detail_screen.dart    # Live score, events
│   │   ├── widgets/
│   │   │   ├── match_card.dart
│   │   │   ├── live_score.dart
│   │   │   └── match_points_breakdown.dart
│   │   └── providers/match_providers.dart
│   ├── leaderboard/
│   │   ├── screens/leaderboard_screen.dart
│   │   ├── widgets/
│   │   │   ├── rank_tile.dart
│   │   │   └── user_rank_card.dart
│   │   └── providers/leaderboard_providers.dart
│   ├── profile/
│   │   ├── screens/profile_screen.dart
│   │   └── providers/profile_providers.dart
│   └── notifications/
│       └── providers/notification_providers.dart
└── shared/
    ├── widgets/
    │   ├── loading_indicator.dart
    │   ├── error_widget.dart
    │   └── flag_icon.dart
    └── utils/
        ├── formatters.dart
        └── validators.dart
```

### Key Screens & Flows

1. **Onboarding**: Login via XEX Exchange → Join tournament → Initial team selection (pick 4 teams within 100 Credits)
2. **Portfolio**: View owned teams, budget, points, trade allowance remaining
3. **Market**: Browse teams by tier, view price charts, buy/sell with confirmation
4. **Matches**: Upcoming/live/completed matches, points earned per match
5. **Leaderboard**: Global ranking with own position highlighted
6. **Reset**: Confirm reset action (warns about trade forfeiture)

---

## 7. Admin Panel Architecture

### Next.js App Router Structure

```
admin/src/
├── app/
│   ├── (auth)/
│   │   └── login/page.tsx
│   ├── (dashboard)/
│   │   ├── layout.tsx              # Sidebar navigation
│   │   ├── dashboard/page.tsx      # Overview stats
│   │   ├── tournament/
│   │   │   ├── page.tsx            # Tournament management
│   │   │   └── [id]/page.tsx       # Tournament detail
│   │   ├── teams/
│   │   │   ├── page.tsx            # Team list with prices
│   │   │   └── [id]/page.tsx       # Team detail, price adjustment
│   │   ├── matches/
│   │   │   ├── page.tsx            # Match schedule
│   │   │   └── [id]/page.tsx       # Match detail, submit result
│   │   ├── users/
│   │   │   ├── page.tsx            # User list
│   │   │   └── [id]/page.tsx       # User portfolio detail
│   │   ├── leaderboard/page.tsx    # Leaderboard view
│   │   ├── rewards/page.tsx        # Prize pool management
│   │   ├── automation/page.tsx     # AI & data automation dashboard
│   │   ├── translations/page.tsx   # Translation management (view/edit per language)
│   │   └── settings/page.tsx       # API keys & feature flags (DB-backed)
│   └── globals.css
├── components/                     # Reusable shadcn/ui components
├── lib/
│   ├── api.ts                      # Axios client
│   └── utils.ts
└── hooks/
    └── use-*.ts                    # TanStack Query hooks
```

### Admin Capabilities
- **Tournament setup**: Create tournament, define stages, configure multipliers and pricing
- **Team management**: Add teams, set tiers/prices, mark eliminations; AI-suggested tiers based on odds
- **Match management**: Schedule matches, submit results (triggers auto-scoring and price updates)
- **Market control**: Lock/unlock market during matches
- **User oversight**: View any user's portfolio, transaction history
- **Leaderboard**: View rankings, force recalculation
- **Automation dashboard**: View job status/logs, manually trigger AI/data jobs (fetch matches, translate teams, generate previews), source badges (Auto/Manual)
- **Translation management**: View and edit all JSONB translations per entity (teams, stages, tournaments), trigger AI batch translation for missing languages
- **Settings (DB-backed)**: API keys for Anthropic Claude, The Odds API, Exchange, FCM — stored in `settings` table, secret values masked in UI. Feature flags for automation toggles. **No redeployment needed to change keys.**

---

## 8. Real-Time System

### WebSocket Hub Architecture

Same hub pattern as XEX Play: server maintains `userID → []*Client` connections.

### Event Types

```json
// Team price changed after match result
{
  "type": "price_update",
  "data": {
    "team_id": "...",
    "team_name": "Brazil",
    "old_price": 47.0,
    "new_price": 48.0,
    "change": 1.0,
    "reason": "win"
  }
}

// Match went live / finished
{
  "type": "match_update",
  "data": {
    "match_id": "...",
    "status": "live",
    "home_score": 2,
    "away_score": 1
  }
}

// User's points updated
{
  "type": "points_earned",
  "data": {
    "team_id": "...",
    "match_id": "...",
    "points": 6.0,
    "breakdown": { "win": 4, "goal_scored": 1, "clean_sheet": 1 }
  }
}

// Leaderboard rank changed
{
  "type": "leaderboard_update",
  "data": {
    "rank": 42,
    "previous_rank": 55,
    "total_points": 87.5
  }
}

// Team eliminated — triggers exit value
{
  "type": "team_eliminated",
  "data": {
    "team_id": "...",
    "team_name": "Japan",
    "exit_value": 21.0,
    "new_budget": 45.0
  }
}

// Market lock/unlock
{
  "type": "market_status",
  "data": {
    "locked": true,
    "reason": "Live match in progress",
    "unlocks_at": "2026-07-15T20:00:00Z"
  }
}
```

### Broadcast Triggers
| Event | WebSocket Message | Recipients |
|-------|------------------|------------|
| Match result submitted | `price_update`, `points_earned` | All active users |
| Match goes live | `match_update`, `market_status` | All active users |
| Team eliminated | `team_eliminated` | Owners of that team |
| Leaderboard recalculated | `leaderboard_update` | All active users |
| Market lock/unlock | `market_status` | All active users |

---

## 9. Authentication & Authorization

### Shared JWT with XEX Exchange

Same approach as XEX Play — no separate auth system.

```
User → XEX Exchange Login → JWT (HS256) → Klub API validates with shared secret
```

#### JWT Claims
```json
{
  "sub": "uuid-of-exchange-user",
  "email": "user@example.com",
  "role": "user",
  "iat": 1700000000,
  "exp": 1700086400
}
```

#### Flow
1. User logs in via XEX Exchange (magic link, Google, Apple, passkey)
2. Exchange issues JWT
3. User opens Klub app with JWT
4. Klub API validates JWT using shared `JWT_SECRET`
5. On first login, creates local `users` record linked by `xex_user_id`

#### Middleware Stack
1. **Auth** — Extracts Bearer token, validates JWT signature and expiry
2. **Admin** — Checks `role` claim for admin endpoints
3. **Rate Limiter** — Redis-based, 30 req/min per user
4. **Locale** — Extracts language from `Accept-Language` header
5. **Logger** — Structured request logging (zerolog)

---

## 10. Game Engine

### Match Result Processing Pipeline

When an admin submits a match result, the following pipeline executes (within a database transaction):

```
Admin submits result
       │
       ▼
1. Update match record (scores, status = finished)
       │
       ▼
2. PricingService: Calculate price changes
   ├── Winner: +1 Credit
   ├── Loser: -1 Credit
   ├── Draw: both +0.5 Credit
   ├── Enforce price ceiling
   └── Record in team_prices
       │
       ▼
3. ScoringService: Calculate points for all portfolio owners
   ├── Match performance (win/draw, goals, clean sheet, conceded)
   ├── Apply stage multiplier (1.0x / 1.5x / 2.0x)
   ├── Extra time bonus (+2) / Penalty win bonus (+1)
   └── Record in user_points
       │
       ▼
4. Check advancement conditions
   ├── If team advances: +2 price bonus, stage points (3/6/9/12/16/25)
   └── If team eliminated: trigger Exit Value for all owners
       │
       ▼
5. LeaderboardService: Recalculate rankings
   ├── Sum all user_points per portfolio
   ├── Apply tiebreaker (fewer trades, higher budget)
   └── Update leaderboard_entries
       │
       ▼
6. Broadcast via WebSocket
   ├── price_update (all users)
   ├── points_earned (affected users)
   ├── team_eliminated (affected users)
   └── leaderboard_update (all users)
       │
       ▼
7. Send push notifications
   ├── Points earned summary
   ├── Team eliminated alert
   └── Rank change notification
```

### Trade Execution Flow

```
User requests BUY team X
       │
       ▼
1. Validate market is open (not locked during live match)
       │
       ▼
2. Validate trade allowance
   ├── Check trade_allowances for current gameweek
   └── Reject if used >= allowed (and not a reset)
       │
       ▼
3. Validate portfolio constraints
   ├── Current team count < max for current stage
   ├── User doesn't already own this team
   └── Budget >= team current_price
       │
       ▼
4. Execute (in DB transaction)
   ├── Deduct price from portfolio.budget
   ├── Create portfolio_teams record
   ├── Create transaction record
   ├── Increment trade_allowances.used
   └── Increment portfolio.total_trades
       │
       ▼
5. Return updated portfolio state
```

### Exit Value Processing

```
Team eliminated
       │
       ▼
1. Find all active portfolio_teams for this team
       │
       ▼
2. For each owner (in DB transaction):
   ├── Calculate exit_value = current_price * 0.70
   ├── Credit exit_value to portfolio.budget
   ├── Mark portfolio_teams as sold (sell_reason = 'exit_value')
   ├── Create transaction record (type = 'exit_value')
   └── Send WebSocket + push notification
```

### Reset Flow

```
User requests RESET
       │
       ▼
1. Validate resets_used < max_resets (2)
       │
       ▼
2. Validate not already used reset this gameweek
       │
       ▼
3. Execute (in DB transaction):
   ├── For each active team in portfolio:
   │   ├── sell_value = current_price * 0.95
   │   ├── Credit to budget
   │   ├── Mark portfolio_teams as sold (sell_reason = 'reset')
   │   └── Create transaction (type = 'reset_sell')
   ├── Increment resets_used
   ├── Mark trade_allowance.reset_used = true (no more trades this GW)
   └── Return new budget for user to repurchase teams
       │
       ▼
4. User must then BUY new teams (separate requests)
   └── Normal buy flow applies (but trade allowance check skipped for reset buys)
```

---

## 11. Sports Data Pipeline & AI Automation

### External Services (Same Accounts as XEX Play)

| Service | Dashboard | Purpose | API Key Storage |
|---------|-----------|---------|-----------------|
| **The Odds API** | [dash.the-odds-api.com](https://dash.the-odds-api.com/) | Match data, live scores, odds for tier suggestions | DB `settings` table: `ODDS_API_KEY` |
| **Anthropic Claude** | [console.anthropic.com](https://console.anthropic.com/) | Translations, tier suggestions, match previews, notification text | DB `settings` table: `ANTHROPIC_API_KEY` |

> **Important**: These use the same API accounts/keys as XEX Play. Keys are entered once in the admin panel Settings page and stored in the database — not in environment variables.

### Sports Data Flow (The Odds API)

```
The Odds API (dash.the-odds-api.com)
       │
       ├─── CronService: every 6 hours ──→ Fetch upcoming matches, odds, brackets
       │                                    Create/update match records in DB
       │
       ├─── CronService: every 60s ──────→ Poll live scores during active matches
       │    (only when matches are live)     Broadcast via WebSocket
       │
       └─── On match completion ─────────→ Fetch final result
                                            Admin confirms OR auto-accepts
                                            │
                                            ▼
                                     Match result processing pipeline (Section 10)
```

### AI Automation Flow (Claude Haiku 4.5)

```
Admin creates tournament + adds teams
       │
       ▼
AIService: Tier Suggestion
├── Fetches current odds from The Odds API
├── Sends to Claude with tier definition rules
├── Returns suggested tier (S/A/B/C) + initial price per team
└── Admin reviews and confirms in admin panel
       │
       ▼
AIService: Team Name Translation
├── Takes English team names
├── Batch-translates to all 6 languages (en, fa, ar, tr, es, fr)
├── Returns JSONB: {"en": "Brazil", "fa": "برزیل", "ar": "البرازيل", ...}
└── Stored in teams.name JSONB column
       │
       ▼
Before each gameweek:
AIService: Match Previews
├── Takes match data (teams, odds, stage, history)
├── Generates localized preview text for each match (all 6 languages)
└── Stored and served to app

On match result:
AIService: Notification Content
├── Takes match result + user's portfolio context
├── Generates localized notification text
│   e.g. "Your team Brazil won 2-0! +8 points earned"
└── Sent via FCM in user's preferred language
```

### Automation Dashboard (Admin Panel)
- **Job list**: All automation jobs with status (running/completed/failed), last run time
- **Manual triggers**: Buttons to manually trigger any job (fetch matches, translate, suggest tiers, generate previews)
- **Activity logs**: Full history from `automation_logs` table with input/output data
- **Source badges**: "Auto" or "Manual" tags on matches/translations showing how they were created
- **Feature toggles**: Enable/disable automation features via settings (e.g., `AUTO_SPORTS_ENABLED`, `AI_TRANSLATIONS_ENABLED`)

### Automation Scope Summary
| Job | Provider | Schedule | Admin Override |
|-----|----------|----------|----------------|
| Fetch upcoming matches | The Odds API | Every 6 hours | Manual trigger |
| Poll live scores | The Odds API | Every 60s (during live matches) | Manual trigger |
| Suggest team tiers | Claude + Odds API | On tournament setup | Manual trigger |
| Translate team names | Claude Haiku 4.5 | On team creation | Manual trigger + edit |
| Generate match previews | Claude Haiku 4.5 | Before each gameweek | Manual trigger |
| Generate notification text | Claude Haiku 4.5 | On match result | — |
| Auto price updates | Internal | On match result | — |
| Auto scoring | Internal | On match result | — |
| Market lock/unlock | Internal | 5 min before kickoff / after all matches finish | Manual override |
| Leaderboard recalculation | Internal | After all gameweek matches complete | Manual trigger |

---

## 12. Deployment & Infrastructure

### Development (Docker Compose)
```yaml
services:
  api:
    build: ./docker/api.Dockerfile
    ports: ["8080:8080"]
    depends_on: [postgres, redis]

  postgres:
    image: postgres:17-alpine
    ports: ["5432:5432"]
    environment:
      POSTGRES_DB: klub
      POSTGRES_USER: klub
      POSTGRES_PASSWORD: klub

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  admin:
    build: ./admin
    ports: ["3000:3000"]
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8080/v1
```

### Production (Azure Container Apps)

Same infrastructure pattern as XEX Play:

```
GitHub Actions (CI/CD)
       │
       ├── Backend: lint → test → Docker build → push to ACR → deploy to Azure Container App
       ├── Admin: lint → build → Docker build → push to ACR → deploy to Azure Container App
       └── App: build → sign → release to App Store / Play Store
```

| Resource | Details |
|----------|---------|
| Container Registry | Azure Container Registry (ACR) |
| Backend | Azure Container App: `klub-api` |
| Admin | Azure Container App: `klub-admin` |
| Database | Azure Database for PostgreSQL (Flexible Server) |
| Cache | Azure Cache for Redis |
| Resource Group | `xex-production-rg` (shared with other XEX services) |

### Environment Variables

**Only infrastructure-level secrets are env vars.** All service API keys are stored in the DB `settings` table and managed via the admin panel.

```bash
# Backend — Environment Variables (infra only)
PORT=8080
ENVIRONMENT=development|production
LOG_LEVEL=debug|info|warn|error
DATABASE_URL=postgres://klub:klub@localhost:5432/klub?sslmode=disable
REDIS_URL=redis://localhost:6379
JWT_SECRET=shared-with-exchange-min-32-chars
CORS_ORIGINS=http://localhost:3000

# Admin — Environment Variables
NEXT_PUBLIC_API_URL=http://localhost:8080/v1
```

**DB-backed settings (managed via admin panel → Settings page):**
```
ANTHROPIC_API_KEY=<Claude API key — same account as XEX Play>
ODDS_API_KEY=<The Odds API key — same account as XEX Play>
EXCHANGE_API_URL=https://api.xex.to
EXCHANGE_SERVICE_KEY=<service-to-service secret>
FCM_CREDENTIALS_JSON=<Firebase service account JSON>
AUTO_SPORTS_ENABLED=true
AI_TRANSLATIONS_ENABLED=true
AI_TIER_SUGGESTIONS_ENABLED=true
LIVE_SCORE_POLL_INTERVAL=60
```

### Settings Service Architecture
The `SettingsService` loads settings from the `settings` table on startup and caches them in Redis. When a setting is updated via the admin panel, the cache is invalidated. Services that need API keys (AIService, SportsDataService, NotificationService) request them through SettingsService — never from env vars.

```go
// Example usage in AIService
func (s *AIService) getClient() *anthropic.Client {
    apiKey := s.settings.Get("ANTHROPIC_API_KEY")
    return anthropic.NewClient(apiKey)
}
```

---

## 13. Monitoring & Observability

### Health Check
```
GET /health → { "status": "ok", "db": "ok", "redis": "ok" }
```

### Prometheus Metrics
- `klub_http_requests_total` — Request count by method, path, status
- `klub_http_request_duration_seconds` — Request latency histogram
- `klub_active_websocket_connections` — Current WS connections
- `klub_trades_total` — Buy/sell transactions
- `klub_active_portfolios` — Portfolios in current tournament
- `klub_match_results_processed` — Match results processed

### Structured Logging (zerolog)
All logs are JSON-formatted with request ID, user ID, and duration fields for easy querying in production.

### Key Alerts
- Market lock/unlock failures
- Match result processing errors
- Price calculation anomalies (price > ceiling)
- Exit value processing failures
- Database connection issues

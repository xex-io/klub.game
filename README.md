# XEX Plus — Team Fantasy for World Cup

A team-based fantasy investment game for the FIFA World Cup. Unlike traditional fantasy games based on individual player performance, XEX Plus lets users **invest in national teams** using a stock-market-style portfolio system. Players manage a budget, buy and sell teams at dynamic prices, and earn points from their teams' real match results and tournament progression.

XEX Plus is designed to acquire and engage users for the [XEX Exchange](https://xex.to) platform.

---

## Game Overview

### Core Concept
Players start with **100 Credits** and must build a portfolio of exactly **4 teams**. Teams are priced based on their strength tier and prices fluctuate after each match. Points are earned through team match performance (wins, goals, clean sheets) and tournament advancement (stage progression bonuses). The player with the most points on the leaderboard at the end of the tournament wins.

### What Makes It Unique
- **Investment Strategy**: Budget management is the core challenge — you can't put all your money on one team.
- **Dynamic Pricing**: Team prices change after every match (+1 for win, +0.5 for draw, -1 for loss), with advancement bonuses (+2).
- **Risk Management**: Eliminated teams auto-sell at 70% value (Exit Value), forcing portfolio rebalancing.
- **Fair Play Economics**: Price ceiling caps, soft volatility, and survival mechanics ensure no player is eliminated from competition.

---

## Game Mechanics

### 1. Budget & Portfolio
| Rule | Value |
|------|-------|
| Starting budget | 100 Credits |
| Portfolio size | Exactly 4 teams |
| Constraint | Cannot spend entire budget on 1 team; must manage budget for 4 teams |

### 2. Team Tiers & Pricing
Teams are classified into 4 pricing tiers based on strength and championship odds:

| Tier | Description | Initial Price | Example |
|------|-------------|--------------|---------|
| **S Tier** | Title contenders | 45–50 Credits | Brazil, France |
| **A Tier** | Strong & dangerous | 35–40 Credits | |
| **B Tier** | Mid-tier & dark horses | 25–30 Credits | |
| **C Tier** | Budget & underdogs | 15–20 Credits | |

### 3. Price Movement System
Prices change based on fixed units to keep volatility smooth:

| Event | Price Change |
|-------|-------------|
| Win | +1 Credit |
| Draw | +0.5 Credit |
| Loss | -1 Credit |
| Confirmed advancement | +2 Credits (one-time bonus) |

**Price Ceiling**: Each team has a maximum price cap (e.g., 60 Credits) — once reached, no further increase that stage.

### 4. Scoring System

#### Match Performance Points (per team per match)
| Event | Base (1x) | R32/R16 (1.5x) | Semi/Final (2x) |
|-------|-----------|-----------------|------------------|
| Win | 4 | 6 | 8 |
| Draw | 2 | 3 | 4 |
| Goal scored | 1 | 1.5 | 2 |
| Clean sheet | 1 | 1.5 | 2 |
| Goal conceded | -0.5 | -0.75 | -1 |

#### Stage Multipliers
| Stage | Multiplier |
|-------|-----------|
| Group stage | 1.0x |
| R32 / R16 / QF | 1.5x |
| Semi-final / Final | 2.0x |

#### Stage Advancement Points (fixed bonuses)
| Milestone | Points |
|-----------|--------|
| Advance from group | 3 |
| Reach Round of 16 | 6 |
| Reach Quarter-finals | 9 |
| Reach Semi-finals | 12 |
| Reach Final | 16 |
| Win Tournament | 25 |

### 5. Extra Time & Penalties (Post-90' Rules)
- Technical stats (goals scored/conceded) counted through 120 minutes
- Extra-time victory: +2 bonus points
- Penalty shootout victory: +1 bonus point
- Penalty shootout goals do NOT count in goal stats

### 6. Trading Rules
- **Market open**: Always (except during live matches)
- **Buy/Sell spread**: Buy at 100%, sell at 95% (5% fee)
- **Trade allowance**: 1 free trade per gameweek (group stage), 2 free trades per round (knockout stage)
- **No rollover**: Unused trades don't carry forward

### 7. Portfolio Size Per Stage
| Stage | Max Teams |
|-------|----------|
| Group stage | 4 |
| Round of 32 / R16 | 3 |
| Quarter-finals onward | 2 |

Players whose teams are eliminated can sell and restructure within the allowed team count.

### 8. Exit Value Mechanism
When a team in your portfolio is eliminated:
- Team is **auto-sold** immediately
- You receive **70% of current market price** in cash
- Example: Team worth 50 Credits → you get 35 Credits to reinvest

### 9. Reset Mechanism
- Each player can **reset their entire portfolio twice** during the tournament
- All teams sold at 95% (voluntary sale rate), then new teams purchased
- Budget recalculated based on current prices
- Using reset **forfeits the trade allowance** for that gameweek

### 10. Tiebreaker Rules
When players are tied on points at tournament end:
1. **Fewer trades used** → higher rank
2. **More remaining budget** → higher rank

---

## Tech Stack

XEX Plus follows the same architecture and technology choices as [XEX Play](https://github.com/xex-exchange/xexplay-api):

| Component | Technology | Version |
|-----------|-----------|---------|
| **Backend API** | Go (Gin framework) | Go 1.25+ |
| **Mobile App** | Flutter (Dart) | Flutter 3.10+ |
| **Admin Panel** | Next.js (React, TypeScript) | Next.js 16+ |
| **Primary Database** | PostgreSQL | 17 (Alpine) |
| **Cache & Sessions** | Redis | 7 (Alpine) |
| **State Management** | Riverpod (Flutter) | v2.6+ |
| **Routing** | GoRouter (Flutter) | v14+ |
| **HTTP Client** | Dio (Flutter) / Axios (Admin) | |
| **Admin UI** | Tailwind CSS + shadcn/ui | v4 |
| **Admin Data** | TanStack Query (React Query) | v5 |
| **Push Notifications** | Firebase Cloud Messaging | |
| **Analytics** | Firebase Analytics + Crashlytics | |
| **Auth** | Shared JWT with XEX Exchange (HS256) | |
| **Real-Time** | WebSocket (gorilla/websocket) | |
| **Monitoring** | Prometheus metrics | |
| **Logging** | zerolog (structured) | |
| **CI/CD** | GitHub Actions → Azure Container Apps | |
| **Containerization** | Docker (multi-stage builds) | |

### External Integrations
| Service | Purpose |
|---------|---------|
| **XEX Exchange API** | Shared JWT auth, reward distribution, user accounts |
| **Sports Data API** | Live match results, scores, tournament brackets (e.g., The Odds API or football-data.org) |
| **Firebase** | Push notifications, analytics, crash reporting |
| **Anthropic Claude** | Content generation (team descriptions, match previews) |

---

## Project Structure

```
xexplus/
├── backend/                    # Go API server
│   ├── cmd/server/            # Entry point
│   ├── internal/
│   │   ├── config/            # Configuration
│   │   ├── domain/            # Domain models
│   │   ├── handler/           # HTTP handlers
│   │   │   ├── admin/         # Admin endpoints
│   │   │   └── ws/            # WebSocket handlers
│   │   ├── middleware/        # Auth, CORS, rate limiting
│   │   ├── repository/        # Data access layer
│   │   │   ├── postgres/      # PostgreSQL repos
│   │   │   └── redis/         # Redis caching
│   │   ├── service/           # Business logic
│   │   ├── exchange/          # XEX Exchange client
│   │   ├── external/          # Third-party API clients
│   │   └── pkg/               # Utilities
│   ├── migrations/            # Database migrations
│   ├── docker/                # Docker configs
│   └── Makefile
│
├── app/                       # Flutter mobile app
│   ├── lib/
│   │   ├── core/              # Infrastructure (network, auth, routing, theme, l10n)
│   │   ├── features/          # Feature modules
│   │   │   ├── auth/          # Login via XEX Exchange
│   │   │   ├── portfolio/     # Team portfolio management
│   │   │   ├── market/        # Team marketplace (buy/sell)
│   │   │   ├── matches/       # Live match tracking
│   │   │   ├── leaderboard/   # Rankings
│   │   │   ├── profile/       # User profile & stats
│   │   │   └── notifications/ # Push notifications
│   │   └── shared/            # Shared widgets
│   └── pubspec.yaml
│
├── admin/                     # Next.js admin panel
│   ├── src/app/
│   │   ├── (auth)/            # Admin login
│   │   └── (dashboard)/       # Protected routes
│   │       ├── dashboard/     # Overview & analytics
│   │       ├── tournament/    # Tournament & stage management
│   │       ├── teams/         # Team configuration & pricing
│   │       ├── matches/       # Match management & results
│   │       ├── users/         # User management
│   │       ├── leaderboard/   # Leaderboard management
│   │       ├── rewards/       # Prize pool & distribution
│   │       └── settings/      # Configuration
│   └── package.json
│
├── docs/                      # Documentation
├── ARCHITECTURE.md            # System design & technical spec
├── README.md                  # This file
└── .github/workflows/         # CI/CD pipelines
```

---

## Development

### Prerequisites
- Go 1.25+
- Flutter 3.10+
- Node.js 20+
- Docker & Docker Compose
- PostgreSQL 17 / Redis 7 (or use Docker)

### Quick Start
```bash
# Start infrastructure
cd backend/docker && docker compose up -d

# Run backend
cd backend && make run

# Run Flutter app
cd app && flutter run

# Run admin panel
cd admin && npm install && npm run dev
```

### Environment Variables
See `backend/.env.example` for backend configuration. Key variables:
- `DATABASE_URL` — PostgreSQL connection string
- `REDIS_URL` — Redis connection string
- `JWT_SECRET` — Shared secret with XEX Exchange (min 32 chars)
- `EXCHANGE_API_URL` — XEX Exchange API endpoint
- `SPORTS_DATA_API_KEY` — Sports data provider API key

---

## License

Proprietary — XEX Exchange. All rights reserved.

# MLBattle - System Architecture Diagram

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      USER INTERFACE (Browser)                    │
│                                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │  Home    │  │Compet-   │  │ Profile  │  │  Login   │       │
│  │  Page    │  │itions    │  │  Page    │  │  Page    │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
         │                      │                      │
         │ HTTP/REST           │ WebSocket            │ JWT Auth
         ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│              REACT FRONTEND (localhost:3000)                     │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Services Layer                        │   │
│  │  • api.js (Axios)  • websocket.js  • auth.js           │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Custom Hooks                          │   │
│  │  • useAuth  • useWebSocket  • useLeaderboard            │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│           DJANGO BACKEND (localhost:8000)                        │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         Django REST Framework (HTTP API)                 │   │
│  │  /api/auth/      /api/users/      /api/competitions/   │   │
│  │  /api/leaderboard/   /api/submissions/  /api/ratings/  │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         Django Channels (WebSocket)                      │   │
│  │  /ws/leaderboard/{competition_id}/                      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│              BACKGROUND PROCESSING                               │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                Celery Worker                             │   │
│  │  • fetch_kaggle_leaderboard (every 5 min)              │   │
│  │  • calculate_ratings_after_competition                  │   │
│  │  • update_competition_statuses (every 10 min)          │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                Celery Beat (Scheduler)                   │   │
│  │  • Triggers periodic tasks                               │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│               DATA & EXTERNAL SERVICES                           │
│                                                                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐ │
│  │PostgreSQL  │  │   Redis    │  │ Kaggle API │  │  Admin   │ │
│  │  Database  │  │ Cache &    │  │Integration │  │ Interface│ │
│  │            │  │ Message    │  │            │  │          │ │
│  │  • Users   │  │ Queue      │  │ • Fetch    │  │ Django   │ │
│  │  • Comps   │  │            │  │   Leader-  │  │ Admin    │ │
│  │  • Subs    │  │ • Celery   │  │   boards   │  │ Panel    │ │
│  │  • Leader  │  │ • Channels │  │ • Comps    │  │          │ │
│  │  • Ratings │  │            │  │   Data     │  │          │ │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Data Flow Diagrams

### 1. User Registration Flow
```
User (Browser)
    │
    ├─> Fill registration form
    │
    ├─> POST /api/users/register/
    │       (username, email, password, kaggle_username)
    │
    ▼
Django Backend
    │
    ├─> Validate data
    │
    ├─> Create User in PostgreSQL
    │
    ├─> Generate JWT tokens (access + refresh)
    │
    ├─> Return {user, tokens}
    │
    ▼
React Frontend
    │
    ├─> Store tokens in localStorage
    │
    ├─> Update AuthContext
    │
    └─> Redirect to Home
```

### 2. Competition Registration Flow
```
User (Browser)
    │
    ├─> Click "Register" on competition
    │
    ├─> POST /api/competitions/{id}/register/
    │       (Authorization: Bearer {token})
    │
    ▼
Django Backend
    │
    ├─> Verify user is authenticated
    │
    ├─> Check competition status (not completed)
    │
    ├─> Create LeaderboardEntry
    │       (user, competition, rank=0)
    │
    ├─> Increment competition.participants_count
    │
    ├─> Increment user.competitions_participated
    │
    ├─> Return success message
    │
    ▼
React Frontend
    │
    ├─> Show success notification
    │
    └─> Redirect to competition detail
```

### 3. Real-time Leaderboard Update Flow
```
Celery Beat (Scheduler)
    │
    ├─> Trigger: Every 5 minutes
    │
    ├─> Execute: fetch_all_active_competitions()
    │
    ▼
Celery Worker
    │
    ├─> For each ongoing competition:
    │       │
    │       ├─> Call KaggleService.get_competition_leaderboard()
    │       │
    │       ├─> Fetch data from Kaggle API
    │       │
    │       ├─> Parse leaderboard entries
    │       │
    │       ├─> Match Kaggle usernames to platform users
    │       │
    │       ├─> Update LeaderboardEntry in PostgreSQL
    │       │
    │       ├─> Create Submission records
    │       │
    │       └─> Send WebSocket update
    │               │
    │               ▼
    │         Django Channels
    │               │
    │               ├─> Broadcast to group: leaderboard_{comp_id}
    │               │
    │               ▼
    │         All connected WebSocket clients
    │               │
    │               ▼
    │         React Frontend
    │               │
    │               ├─> useLeaderboard hook receives update
    │               │
    │               ├─> Update component state
    │               │
    │               └─> Re-render leaderboard table
```

### 4. ELO Rating Calculation Flow
```
Competition Ends
    │
    ├─> update_competition_statuses() task detects end
    │
    ├─> Trigger: calculate_ratings_after_competition(comp_id)
    │
    ▼
Celery Worker
    │
    ├─> Fetch all LeaderboardEntry for competition
    │
    ├─> Prepare participants list:
    │       [{user_id, username, old_rating, rank}, ...]
    │
    ├─> Call EloRatingSystem.calculate_competition_ratings()
    │       │
    │       ├─> Calculate K-factor = log₂(n) × 8
    │       │
    │       ├─> Calculate average rating
    │       │
    │       ├─> For each participant:
    │       │       │
    │       │       ├─> Expected = 1/(1+10^((avg-rating)/400))
    │       │       │
    │       │       ├─> Actual = 1 - (rank-1)/(n-1)
    │       │       │
    │       │       └─> NewRating = OldRating + K×(Actual-Expected)
    │       │
    │       └─> Return results list
    │
    ├─> For each result:
    │       │
    │       ├─> Create RatingHistory record
    │       │
    │       └─> Update User.elo_rating
    │
    └─> Transaction committed to PostgreSQL
```

## Component Interaction Diagram

```
┌────────────────────────────────────────────────────────────┐
│                    React Component                          │
│                                                              │
│  function CompetitionDetail() {                             │
│    const { id } = useParams();                              │
│    const { leaderboard, isConnected } = useLeaderboard(id); │
│    const { user } = useAuth();                              │
│                                                              │
│    return (                                                  │
│      <div>                                                   │
│        <h1>Competition</h1>                                  │
│        {isConnected && <span>● LIVE</span>}                 │
│        <Leaderboard data={leaderboard} />                   │
│      </div>                                                  │
│    );                                                        │
│  }                                                           │
└────────────────────────────────────────────────────────────┘
         │                          │                   │
         │ useParams()             │ useLeaderboard()  │ useAuth()
         │                          │                   │
         ▼                          ▼                   ▼
┌─────────────┐        ┌──────────────────┐   ┌─────────────┐
│React Router │        │ Custom Hook      │   │ AuthContext │
│             │        │                  │   │             │
│ • Parse URL │        │ • Fetch API      │   │ • User data │
│ • Get ID    │        │ • WebSocket      │   │ • Login     │
└─────────────┘        │ • State          │   │ • Logout    │
                       └──────────────────┘   └─────────────┘
                                │                     │
                                ▼                     ▼
                       ┌──────────────────────────────────┐
                       │     Services Layer               │
                       │                                  │
                       │ • api.js (HTTP)                  │
                       │ • websocket.js (WebSocket)       │
                       │ • auth.js (Tokens)               │
                       └──────────────────────────────────┘
                                │
                                ▼
                       ┌──────────────────────────────────┐
                       │     Django Backend               │
                       │                                  │
                       │ • REST API                       │
                       │ • WebSocket                      │
                       │ • Database                       │
                       └──────────────────────────────────┘
```

## File Dependencies

```
Frontend Dependencies:
├── src/index.js
│   └── imports App.jsx
│       └── imports AuthProvider (useAuth.js)
│           └── imports api.js
│               └── uses axios
│
├── src/App.jsx
│   ├── imports Router (react-router-dom)
│   ├── imports Navbar
│   ├── imports Footer
│   └── imports Pages
│       ├── Home
│       ├── CompetitionList
│       │   └── imports CompetitionCard
│       ├── CompetitionDetail
│       │   ├── imports useLeaderboard
│       │   │   ├── uses api.js
│       │   │   └── uses websocket.js
│       │   └── imports Leaderboard
│       ├── Profile
│       │   ├── imports RatingChart (Chart.js)
│       │   └── imports SubmissionHistory
│       ├── Login
│       │   └── imports useAuth
│       └── Register
│           └── imports useAuth

Backend Dependencies:
├── manage.py
│   └── uses config.settings
│
├── config/settings/base.py
│   ├── defines INSTALLED_APPS
│   ├── defines DATABASES
│   └── defines CHANNEL_LAYERS
│
├── config/celery.py
│   ├── imports config.settings
│   └── defines beat_schedule
│
├── apps/users/models.py
│   └── defines User (extends AbstractUser)
│
├── apps/competitions/models.py
│   └── defines Competition
│
├── apps/submissions/models.py
│   ├── imports User (FK)
│   ├── imports Competition (FK)
│   └── defines Submission
│
├── apps/leaderboard/models.py
│   ├── imports User (FK)
│   ├── imports Competition (FK)
│   └── defines LeaderboardEntry
│
├── apps/ratings/models.py
│   ├── imports User (FK)
│   ├── imports Competition (FK)
│   └── defines RatingHistory
│
└── apps/submissions/tasks.py
    ├── imports KaggleService
    ├── imports LeaderboardEntry
    ├── imports send_leaderboard_update
    └── defines fetch_kaggle_leaderboard
```

## Technology Stack Integration

```
┌──────────────────────────────────────────────────────────┐
│                  Technology Stack                         │
├──────────────────────────────────────────────────────────┤
│                                                            │
│  Frontend:                                                 │
│  ┌────────────────────────────────────────────────┐      │
│  │  React 18 ─┬─> React Router v6                │      │
│  │            ├─> Axios (HTTP)                    │      │
│  │            ├─> Native WebSocket API            │      │
│  │            └─> Chart.js (Visualization)        │      │
│  └────────────────────────────────────────────────┘      │
│                                                            │
│  Backend:                                                  │
│  ┌────────────────────────────────────────────────┐      │
│  │  Django 4.2 ─┬─> Django REST Framework         │      │
│  │              ├─> Django Channels (WebSocket)   │      │
│  │              ├─> djangorestframework-simplejwt │      │
│  │              └─> django-cors-headers           │      │
│  └────────────────────────────────────────────────┘      │
│                                                            │
│  Database & Cache:                                         │
│  ┌────────────────────────────────────────────────┐      │
│  │  PostgreSQL 14+ (Primary DB)                   │      │
│  │  Redis 7.x (Cache + Message Broker)            │      │
│  └────────────────────────────────────────────────┘      │
│                                                            │
│  Background Processing:                                    │
│  ┌────────────────────────────────────────────────┐      │
│  │  Celery ─┬─> Celery Beat (Scheduler)          │      │
│  │          ├─> Redis (Broker)                    │      │
│  │          └─> channels-redis (Channel Layer)    │      │
│  └────────────────────────────────────────────────┘      │
│                                                            │
│  External:                                                 │
│  ┌────────────────────────────────────────────────┐      │
│  │  Kaggle API (kaggle-api package)               │      │
│  └────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────┘
```

---

This diagram shows the complete architecture and data flow of the MLBattle platform. All components work together to provide a seamless, real-time ML competition experience! 🚀

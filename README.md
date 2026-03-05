# Finance Tracker

A self-hosted personal finance tracking application with FastAPI backend and React frontend.

## Features

- User authentication (JWT)
- Multiple account management
- Transaction tracking (income, expense, transfers)
- Category management with hierarchical structure
- Multi-currency support
- PostgreSQL database with Docker

## Tech Stack

**Backend:**
- FastAPI (Python)
- SQLAlchemy ORM
- PostgreSQL
- JWT Authentication
- Pydantic validation

**Frontend:**
- React 18+ with TypeScript
- Vite
- (To be set up)

## Quick Start

### Prerequisites
- Docker and Docker Compose
- Python 3.11+ (for local development)
- Node.js 18+ (for local development)

### Running with Docker

1. Clone the repository
2. Copy environment files:
   ```bash
   cp .env.example .env
   cp finance-tracker-backend/.env.example finance-tracker-backend/.env
   ```
3. Update the `SECRET_KEY` in both `.env` files
4. Start all services:
   ```bash
   docker-compose up -d
   ```
5. Access:
   - Backend API: http://localhost:8000
   - API Docs: http://localhost:8000/docs
   - Frontend: http://localhost:3000 (once set up)

### Local Development

#### Backend

```bash
cd finance-tracker-backend
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your settings
uvicorn app.main:app --reload
```

#### Frontend

```bash
cd finance-tracker-frontend
# Setup instructions coming soon
```

## API Endpoints

### Health
- `GET /health` - Health check endpoint

### Authentication
- `POST /api/v1/auth/register` - Register new user
- `POST /api/v1/auth/login` - Login and get token
- `GET /api/v1/auth/me` - Get current user

### Coming Soon
- Accounts CRUD
- Transactions CRUD
- Categories CRUD
- Reports and analytics

## Database Models

- **User**: User accounts with authentication
- **Account**: Financial accounts (checking, savings, credit cards, etc.)
- **Category**: Transaction categories (hierarchical)
- **Transaction**: Income, expenses, and transfers

## Security Features

- Password hashing with bcrypt
- JWT token authentication
- CORS protection
- Input validation with Pydantic
- Parameterized SQL queries

## Development Notes

- Backend runs on port 8000
- Frontend runs on port 3000
- PostgreSQL runs on port 5432
- Database data persists in Docker volume
- Hot reload enabled for both frontend and backend in development

## License

Private project for personal use.

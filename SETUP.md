# Setup Instructions

## Quick Start (Docker - Recommended)

**No manual setup required!** Just run:

```bash
docker-compose up -d
```

The system automatically:
- ✅ Creates database tables
- ✅ Runs all migrations
- ✅ Starts all services

Access the app at http://localhost:3000

---

## Detailed Setup

### Running with Docker (Recommended)

```bash
# Build and start all services
docker-compose up --build

# Or run in detached mode
docker-compose up -d --build

# View logs
docker-compose logs -f

# Stop all services
docker-compose down

# Stop and remove volumes (WARNING: deletes all data)
docker-compose down -v
```

Services will be available at:
- Backend API: http://localhost:8000
- API Documentation: http://localhost:8000/docs
- Frontend: http://localhost:3000
- PostgreSQL: localhost:5432

### Database Migrations (Automated)

**Migrations run automatically!** When the backend container starts, it:
1. Waits for PostgreSQL to be ready
2. Creates a `schema_migrations` tracking table
3. Runs all pending migrations in order:
   - `001_add_import_tracking.sql` - Import sessions, category rules, parser templates
   - `002_add_default_parser_to_accounts.sql` - Default parser linking
   - `003_add_balance_snapshots.sql` - Balance tracking
4. Skips migrations that have already been applied
5. Starts the FastAPI application

**Check migration status:**
```bash
docker exec -it finance-tracker-db psql -U financeuser -d financedb -c "SELECT * FROM schema_migrations;"
```

**View migration logs:**
```bash
docker logs finance-tracker-backend | grep -i migration
```

See [migrations/README.md](./finance-tracker-backend/migrations/README.md) for details.

## Local Development (Without Docker)

### Backend Setup

```bash
cd finance-tracker-backend

# Create virtual environment
python -m venv venv

# Activate virtual environment
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Create .env file and add your SECRET_KEY
cp .env.example .env

# For local dev, you need PostgreSQL running
# Or use Docker for just the database:
# docker run -d --name finance-db -e POSTGRES_USER=financeuser -e POSTGRES_PASSWORD=financepass -e POSTGRES_DB=financedb -p 5432:5432 postgres:15-alpine

# Run migrations (required for local dev)
DB_HOST=localhost DB_PORT=5432 DB_USER=financeuser DB_NAME=financedb POSTGRES_PASSWORD=financepass bash scripts/run_migrations.sh

# Run the server
uvicorn app.main:app --reload
```

Backend will be at http://localhost:8000

### Frontend Setup

```bash
cd finance-tracker-frontend

# Install dependencies
npm install

# Create .env file
cp .env.example .env

# Run development server
npm run dev
```

Frontend will be at http://localhost:3000

## First Time Usage

1. Start the application (Docker or local)
2. Open http://localhost:3000
3. Click "Register" to create an account
4. Login with your credentials
5. You'll see the dashboard

## Project Structure

```
finance-tracker/
├── docker-compose.yml          # Orchestrates all services
├── .env                        # Root environment variables
├── README.md                   # Project documentation
├── SETUP.md                    # This file
│
├── finance-tracker-backend/    # FastAPI Backend
│   ├── app/
│   │   ├── api/               # API endpoints
│   │   │   ├── auth.py        # Authentication endpoints
│   │   │   └── health.py      # Health check
│   │   ├── core/              # Core functionality
│   │   │   ├── config.py      # Settings
│   │   │   ├── database.py    # Database connection
│   │   │   └── security.py    # Auth & password hashing
│   │   ├── models/            # SQLAlchemy models
│   │   │   ├── user.py
│   │   │   ├── account.py
│   │   │   ├── category.py
│   │   │   └── transaction.py
│   │   └── main.py            # FastAPI app entry point
│   ├── tests/                 # Test files
│   ├── Dockerfile
│   ├── requirements.txt
│   └── .env
│
└── finance-tracker-frontend/   # React Frontend
    ├── src/
    │   ├── components/        # React components
    │   │   └── ProtectedRoute.tsx
    │   ├── pages/            # Page components
    │   │   ├── LoginPage.tsx
    │   │   ├── RegisterPage.tsx
    │   │   └── DashboardPage.tsx
    │   ├── services/         # API client
    │   │   └── api.ts
    │   ├── lib/              # Utilities
    │   │   └── auth.tsx      # Auth context
    │   ├── App.tsx           # Main app component
    │   ├── main.tsx          # Entry point
    │   └── index.css         # Global styles
    ├── Dockerfile
    ├── package.json
    ├── vite.config.ts
    └── .env

## Next Steps

### Backend TODO:
- [x] Add CRUD endpoints for accounts
- [x] Add CRUD endpoints for transactions
- [x] Add CRUD endpoints for categories
- [x] Implement account balance updates
- [x] Add transaction filtering and search
- [x] Add date range queries
- [x] Add reports/analytics endpoints
- [x] Add CSV import/export (multi-bank support)
- [x] Add database migrations (automated system)
- [ ] Write tests

### Frontend TODO:
- [ ] Add proper styling (consider Tailwind CSS or shadcn/ui)
- [ ] Create account management pages
- [ ] Create transaction list and forms
- [ ] Create category management
- [ ] Add charts and visualizations (recharts)
- [ ] Add data tables (TanStack Table)
- [ ] Add form validation (react-hook-form + zod)
- [ ] Add loading states and error handling
- [ ] Add responsive design
- [ ] Write tests

## Troubleshooting

### Docker Issues

**Port already in use:**
```bash
# Check what's using the port
lsof -i :8000  # or :3000, :5432
# Kill the process or change ports in docker-compose.yml
```

**Database connection errors:**
```bash
# Make sure database is healthy
docker-compose ps
# Check logs
docker-compose logs db
```

**Hot reload not working:**
- Make sure volumes are mounted correctly in docker-compose.yml

### Local Development Issues

**Backend won't start:**
- Ensure virtual environment is activated
- Check Python version (3.11+)
- Verify all dependencies installed
- Check DATABASE_URL in .env

**Frontend won't start:**
- Delete node_modules and reinstall: `rm -rf node_modules && npm install`
- Check Node version (18+)
- Verify VITE_API_URL in .env

## Database Management

### Backup
```bash
docker-compose exec db pg_dump -U financeuser financedb > backup.sql
```

### Restore
```bash
docker-compose exec -T db psql -U financeuser financedb < backup.sql
```

### Reset Database
```bash
docker-compose down -v
docker-compose up -d
```

## Production Deployment

Before deploying to production:

1. Change all default passwords
2. Generate strong SECRET_KEY
3. Set DEBUG=False
4. Configure proper CORS origins
5. Set up SSL/TLS certificates
6. Configure automated backups
7. Set up monitoring and logging
8. Use proper secrets management
9. Set resource limits in docker-compose
10. Consider using docker secrets for sensitive data

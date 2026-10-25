# DocFlow — Employee Management System

Minimal FastAPI backend with Google OAuth authentication and MySQL.

---

## Project Structure

```text
DocFlow/
├── .env                  # MySQL and Google OAuth credentials
├── main.py               # Root entry point
├── app/
│   ├── main.py           # FastAPI app, Google OAuth routes & Employee model
│   └── db/
│       └── session.py    # MySQL connection & session
└── tests/
    └── test_db_config.py # Database configuration test
```

---

## Setup & Run

### 1. Configure `.env`

```dotenv
DATABASE_URL=mysql+pymysql://docflow:DocFlow123!@localhost:3306/docflow_db
GOOGLE_CLIENT_ID=your_google_client_id_here
GOOGLE_CLIENT_SECRET=your_google_client_secret_here
GOOGLE_REDIRECT_URI=http://localhost:8000/auth/callback
SECRET_KEY=docflow-secret-key-12345
```

### 2. Run the Server

```bash
uvicorn app.main:app --reload
# or
python main.py
```

### 3. Endpoints

- `GET /` — Home message
- `GET /db-check` — Check MySQL connection
- `GET /login` — Start Google OAuth login
- `GET /auth/callback` — Google OAuth callback (registers/logs in employee)
- `GET /me` — Current logged-in user profile (from session)
- `GET /employees` — List all employees
- `POST /logout` — Logout user
- `GET /docs` — Swagger API documentation

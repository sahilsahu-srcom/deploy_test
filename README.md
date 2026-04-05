# Todo App

Simple todo application with React + Vite frontend and FastAPI backend using Neon PostgreSQL.

## Setup

1. Install frontend dependencies:
```bash
npm install
```

2. Install backend dependencies:
```bash
cd backend
pip install -r requirements.txt
```

3. Create `backend/.env` file with your Neon database URL:
```
DATABASE_URL=postgresql://[user]:[password]@[host]/[database]?sslmode=require
```

4. Run backend (from backend folder):
```bash
cd backend
uvicorn main:app --reload
```

5. Run frontend (in another terminal):
```bash
npm run dev
```

Visit http://localhost:5173

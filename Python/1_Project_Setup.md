# Project Setup – Building a Production-Ready FastAPI Backend

## Overview

This guide explains how to set up a scalable, production-ready backend using Python and FastAPI. It covers project initialization, dependency management, environment configuration, routing, and development workflow.

---

## 1. Initializing a Python Project

### Create Project Directory

```bash
mkdir my-fastapi-api
cd my-fastapi-api
```

### Create Virtual Environment

```bash
python -m venv venv

# Linux / macOS
source venv/bin/activate

# Windows
venv\Scripts\activate
```

### Initialize Dependencies

```bash
pip install fastapi uvicorn python-dotenv pydantic[email]
pip freeze > requirements.txt
```

---

## 2. Dependency Management

### requirements.txt

**File:** `requirements.txt`

```txt
fastapi==0.110.0
uvicorn==0.27.0
python-dotenv==1.0.1
pydantic==2.6.1
python-jose==3.3.0
passlib[bcrypt]==1.7.4
```

### Purpose of Dependencies

| Package       | Purpose            |
| ------------- | ------------------ |
| fastapi       | Web framework      |
| uvicorn       | ASGI server        |
| python-dotenv | Environment config |
| pydantic      | Data validation    |
| python-jose   | JWT handling       |
| passlib       | Password hashing   |

Install:

```bash
pip install -r requirements.txt
```

---

## 3. Project Structure (Layered Architecture)

### Recommended Layout

```text
my-fastapi-api/
├── app/
│   ├── main.py
│   ├── core/
│   │   ├── config.py
│   │   └── security.py
│   ├── db/
│   │   └── database.py
│   ├── models/
│   │   └── user.py
│   ├── schemas/
│   │   └── user.py
│   ├── routers/
│   │   └── users.py
│   ├── services/
│   │   └── user_service.py
│   └── middleware/
│       └── errors.py
│
├── .env
├── .env.example
├── .gitignore
├── requirements.txt
└── README.md
```

### Folder Responsibilities

* core → Configuration and security
* db → Database connections
* models → ORM models
* schemas → Pydantic schemas
* routers → API routes
* services → Business logic
* middleware → Error handling

---

## 4. Environment Configuration

### .env (Local)

```env
ENV=development
HOST=127.0.0.1
PORT=8000

DB_URL=postgresql://user:password@localhost/db

JWT_SECRET=super-secret-key
JWT_EXPIRE_MIN=15
CORS_ORIGIN=http://localhost:5173
```

### .env.example

```env
ENV=
HOST=
PORT=
DB_URL=
JWT_SECRET=
JWT_EXPIRE_MIN=
CORS_ORIGIN=
```

---

## 5. Centralized Configuration

### config.py

**File:** `app/core/config.py`

```python
from pydantic import BaseSettings

class Settings(BaseSettings):
    env: str = "development"
    host: str = "127.0.0.1"
    port: int = 8000

    db_url: str

    jwt_secret: str
    jwt_expire_min: int = 15

    cors_origin: str = "*"

    class Config:
        env_file = ".env"

settings = Settings()
```

---

## 6. Application Entry Point

### main.py

**File:** `app/main.py`

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.core.config import settings
from app.routers import users
from app.middleware.errors import register_exception_handlers

app = FastAPI(title="My API", version="1.0.0")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=[settings.cors_origin],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Routes
app.include_router(users.router, prefix="/api/users")

# Health Check
@app.get("/health")
def health():
    return {"status": "ok", "env": settings.env}

# Error handlers
register_exception_handlers(app)
```

---

## 7. Routing Fundamentals

### User Router

**File:** `app/routers/users.py`

```python
from fastapi import APIRouter, Depends, HTTPException, status
from app.schemas.user import UserCreate, UserOut
from app.services.user_service import UserService

router = APIRouter(tags=["Users"])

@router.get("/", response_model=list[UserOut])
def get_users():
    return UserService.get_all()

@router.get("/{user_id}", response_model=UserOut)
def get_user(user_id: int):
    user = UserService.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@router.post("/", status_code=201, response_model=UserOut)
def create_user(data: UserCreate):
    return UserService.create(data)
```

---

## 8. Schemas and Models

### Pydantic Schemas

**File:** `app/schemas/user.py`

```python
from pydantic import BaseModel, EmailStr

class UserBase(BaseModel):
    name: str
    email: EmailStr

class UserCreate(UserBase):
    password: str

class UserOut(UserBase):
    id: int

    class Config:
        from_attributes = True
```

---

## 9. Service Layer

### user_service.py

**File:** `app/services/user_service.py`

```python
from app.schemas.user import UserCreate

class UserService:

    _users = []
    _id = 1

    @classmethod
    def get_all(cls):
        return cls._users

    @classmethod
    def get_by_id(cls, user_id: int):
        return next((u for u in cls._users if u["id"] == user_id), None)

    @classmethod
    def create(cls, data: UserCreate):
        user = {
            "id": cls._id,
            "name": data.name,
            "email": data.email,
        }
        cls._users.append(user)
        cls._id += 1
        return user
```

---

## 10. Error Handling Middleware

### errors.py

**File:** `app/middleware/errors.py`

```python
from fastapi import Request, FastAPI
from fastapi.responses import JSONResponse


def register_exception_handlers(app: FastAPI):

    @app.exception_handler(Exception)
    async def global_exception_handler(request: Request, exc: Exception):
        return JSONResponse(
            status_code=500,
            content={
                "success": False,
                "error": "Internal server error"
            }
        )
```

---

## 11. Running the Server

### Development Mode

```bash
uvicorn app.main:app --reload
```

### Production Mode

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

---

## 12. Development Workflow

### Useful Commands

```bash
# Install deps
pip install -r requirements.txt

# Run server
uvicorn app.main:app --reload

# Format (optional)
black app/
```

### .gitignore

```gitignore
venv/
__pycache__/
.env
*.log
.idea/
.vscode/
dist/
build/
```

---

## Summary

You now have a production-ready FastAPI project with:

* Virtual environment isolation
* Layered architecture
* Centralized configuration
* RESTful routing
* Schema validation
* Service abstraction
* Global error handling

Next Steps:

1. Add SQLAlchemy / SQLModel
2. Implement JWT authentication
3. Add Alembic migrations
4. Integrate logging
5. Write pytest test suites

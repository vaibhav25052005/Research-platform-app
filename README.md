# Research Platform — Full App Skeleton
# Monorepo-style layout with backend (FastAPI), worker pipeline, and frontend (React)

====================
== PROJECT STRUCTURE
====================

research-platform/
│
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── database.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── ingestion.py
│   │   ├── parser.py
│   │   ├── embeddings.py
│   │   ├── retrieval.py
│   │   └── routes.py
│   │
│   ├── worker.py
│   ├── celery_app.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── api.js
│   │   ├── Dashboard.jsx
│   │   └── Upload.jsx
│   └── package.json
│
├── docker-compose.yml
└── README.md

=====================================================
================ BACKEND ============================
=====================================================

# backend/app/config.py

import os

class Settings:
    DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./db.sqlite")
    STORAGE_PATH = os.getenv("STORAGE_PATH", "./storage")

settings = Settings()

-----------------------------------------------------

# backend/app/database.py

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base
from .config import settings

engine = create_engine(settings.DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine)
Base = declarative_base()

-----------------------------------------------------

# backend/app/models.py

from sqlalchemy import Column, Integer, String, Text
from .database import Base

class Document(Base):
    __tablename__ = "documents"
    id = Column(Integer, primary_key=True, index=True)
    filename = Column(String)
    content = Column(Text)

-----------------------------------------------------

# backend/app/schemas.py

from pydantic import BaseModel

class DocumentOut(BaseModel):
    id: int
    filename: str

    class Config:
        from_attributes = True

-----------------------------------------------------

# backend/app/parser.py

from pathlib import Path

def parse_file(path: str) -> str:
    # naive parser — replace with OCR/pdf pipeline
    return Path(path).read_text(errors="ignore")

-----------------------------------------------------

# backend/app/embeddings.py

import hashlib

def fake_embedding(text: str):
    # placeholder embedding
    return hashlib.sha256(text.encode()).hexdigest()

-----------------------------------------------------

# backend/app/ingestion.py

import shutil
from pathlib import Path
from .config import settings

STORAGE = Path(settings.STORAGE_PATH)
STORAGE.mkdir(exist_ok=True)

def save_file(file):
    dest = STORAGE / file.filename
    with open(dest, "wb") as buffer:
        shutil.copyfileobj(file.file, buffer)
    return str(dest)

-----------------------------------------------------

# backend/app/retrieval.py

from sqlalchemy.orm import Session
from .models import Document


def search_documents(db: Session, query: str):
    return db.query(Document).filter(Document.content.contains(query)).all()

-----------------------------------------------------

# backend/app/routes.py

from fastapi import APIRouter, UploadFile, Depends
from sqlalchemy.orm import Session

from .database import SessionLocal
from .models import Document
from .schemas import DocumentOut
from .ingestion import save_file
from .parser import parse_file

router = APIRouter()


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


@router.post("/upload", response_model=DocumentOut)
async def upload(file: UploadFile, db: Session = Depends(get_db)):
    path = save_file(file)
    content = parse_file(path)

    doc = Document(filename=file.filename, content=content)
    db.add(doc)
    db.commit()
    db.refresh(doc)

    return doc


@router.get("/search")
def search(q: str, db: Session = Depends(get_db)):
    return search_documents(db, q)

-----------------------------------------------------

# backend/app/main.py

from fastapi import FastAPI
from .database import Base, engine
from .routes import router

Base.metadata.create_all(bind=engine)

app = FastAPI()
app.include_router(router)

-----------------------------------------------------

# backend/celery_app.py

from celery import Celery

celery = Celery("tasks", broker="redis://redis:6379/0")

-----------------------------------------------------

# backend/worker.py

from celery_app import celery

@celery.task
def background_parse(doc_id: int):
    print(f"Processing document {doc_id}")

-----------------------------------------------------

# backend/requirements.txt

fastapi
uvicorn
sqlalchemy
pydantic
celery
redis

-----------------------------------------------------

# backend/Dockerfile

FROM python:3.11
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

=====================================================
================ FRONTEND ===========================
=====================================================

# frontend/package.json

{
  "name": "research-ui",
  "private": true,
  "scripts": {
    "start": "vite"
  },
  "dependencies": {
    "react": "^18",
    "react-dom": "^18",
    "axios": "^1"
  }
}

-----------------------------------------------------

# frontend/src/api.js

import axios from "axios";

export const api = axios.create({
  baseURL: "http://localhost:8000",
});

-----------------------------------------------------

# frontend/src/App.jsx

import Dashboard from "./Dashboard";
import Upload from "./Upload";

export default function App() {
  return (
    <div>
      <h1>Research Platform</h1>
      <Upload />
      <Dashboard />
    </div>
  );
}

-----------------------------------------------------

# frontend/src/Upload.jsx

import { api } from "./api";

export default function Upload() {
  const handleUpload = async (e) => {
    const file = e.target.files[0];
    const form = new FormData();
    form.append("file", file);
    await api.post("/upload", form);
    alert("Uploaded");
  };

  return <input type="file" onChange={handleUpload} />;
}

-----------------------------------------------------

# frontend/src/Dashboard.jsx

import { useState } from "react";
import { api } from "./api";

export default function Dashboard() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  const search = async () => {
    const res = await api.get(`/search?q=${query}`);
    setResults(res.data);
  };

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      <button onClick={search}>Search</button>
      <ul>
        {results.map((r) => (
          <li key={r.id}>{r.filename}</li>
        ))}
      </ul>
    </div>
  );
}

=====================================================
================ INFRA ==============================
=====================================================

# docker-compose.yml

version: "3"

services:
  api:
    build: ./backend
    ports:
      - "8000:8000"
    volumes:
      - ./storage:/app/storage

  redis:
    image: redis

=====================================================
================ README =============================
=====================================================

# Run Backend
cd backend
uvicorn app.main:app --reload

# Run Frontend
cd frontend
npm install
npm start

# Or Docker

docker-compose up --build

=====================================================

This skeleton provides:

✔ File ingestion
✔ Storage
✔ Basic parsing
✔ Search
✔ Async worker scaffold
✔ Frontend upload + dashboard

Extend with:
- OCR pipeline
- Embeddings/vector DB
- LLM Q&A layer
- Auth & permissions
- Distributed storage

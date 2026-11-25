# SpendSmart-Personal-Expense-Tracker
A full-stack personal expense tracker built using React, Python FastAPI and SQLite.
main.py
from datetime import date
from typing import Optional
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel, Field
from sqlalchemy import create_engine, Column, Integer, String, Float, Date
from sqlalchemy.orm import declarative_base, sessionmaker

DATABASE_URL = "sqlite:///./spendsmart.db"
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

class Expense(Base):
    __tablename__ = "expenses"
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, nullable=False)
    amount = Column(Float, nullable=False)
    category = Column(String, nullable=False)
    spent_on = Column(Date, nullable=False)

Base.metadata.create_all(bind=engine)

app = FastAPI(title="SpendSmart API", version="1.0.0")
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

class ExpenseCreate(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    amount: float = Field(gt=0)
    category: str = Field(min_length=1, max_length=40)
    spent_on: date

class ExpenseOut(ExpenseCreate):
    id: int
    class Config:
        from_attributes = True

@app.get("/")
def home():
    return {"message": "Welcome to SpendSmart API"}

@app.get("/expenses", response_model=list[ExpenseOut])
def get_expenses():
    db = SessionLocal()
    try:
        return db.query(Expense).order_by(Expense.spent_on.desc(), Expense.id.desc()).all()
    finally:
        db.close()

@app.post("/expenses", response_model=ExpenseOut, status_code=201)
def add_expense(item: ExpenseCreate):
    db = SessionLocal()
    try:
        expense = Expense(**item.model_dump())
        db.add(expense)
        db.commit()
        db.refresh(expense)
        return expense
    finally:
        db.close()

@app.delete("/expenses/{expense_id}")
def delete_expense(expense_id: int):
    db = SessionLocal()
    try:
        expense = db.query(Expense).filter(Expense.id == expense_id).first()
        if not expense:
            raise HTTPException(status_code=404, detail="Expense not found")
        db.delete(expense)
        db.commit()
        return {"message": "Expense deleted"}
    finally:
        db.close()

@app.get("/summary")
def get_summary():
    db = SessionLocal()
    try:
        rows = db.query(Expense).all()
        total = sum(row.amount for row in rows)
        today = date.today()
        month_total = sum(
            row.amount for row in rows
            if row.spent_on.year == today.year and row.spent_on.month == today.month
        )
        categories = {}
        for row in rows:
            categories[row.category] = categories.get(row.category, 0) + row.amount
        return {
            "total_spending": round(total, 2),
            "this_month": round(month_total, 2),
            "expense_count": len(rows),
            "by_category": categories
        }
    finally:
        db.close()

requirements
fastapi
uvicorn[standard]
sqlalchemy
pydantic

packages
{
  "name": "spendsmart-frontend",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {"dev": "vite", "build": "vite", "preview": "vite preview"},
  "dependencies": {"@vitejs/plugin-react": "latest", "vite": "latest", "react": "latest", "react-dom": "latest"},
  "devDependencies": {}
}




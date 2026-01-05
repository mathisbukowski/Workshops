# Workshop: Building a Modern API with FastAPI
**Topic:** High-Performance API, SQLModel, and Type Hints
**Duration:** 3 Hours
**Level:** Beginner to Intermediate
**Goal:** Build a production-ready "Smart E-Commerce API" with database relationships and automatic documentation.

---

## Objectives
By the end of this workshop, you will understand:
1.  **Modern Python:** Type Hints, Pydantic, and Async/Await.
2.  **FastAPI Magic:** Dependency Injection (`Depends`) and Path Operations.
3.  **SQLModel:** Using the same class for Database Tables AND Data Validation.
4.  **Professional Structure:** Moving from a single file to a scalable architecture.

---

##  Part 1: Setup & First Steps

### 1.1 Virtual Environment
Always work in an isolated environment.
```bash
# Windows
python -m venv venv
venv\Scripts\activate

# Mac/Linux
python3 -m venv venv
source venv/bin/activate
```

### 1.2 Installation
We need FastAPI, Uvicorn (the server), and SQLModel.
```bash
pip install fastapi "uvicorn[standard]" sqlmodel
```

### 1.3 The "Hello World"
Create a file `main.py`.

```python
from fastapi import FastAPI

app = FastAPI()

# Decorator = The Route
@app.get("/")
def read_root():
    return {"message": "Welcome to the Shop API"}
```

**Run it:**
```bash
uvicorn main:app --reload
```
*   Go to `http://127.0.0.1:8000` -> You see JSON.
*   **The Magic:** Go to `http://127.0.0.1:8000/docs` -> You see the interactive Swagger UI. It's automatic!

---

## Part 2: The Logic & Pydantic

### 2.1 The Restaurant Analogy
*   **The Client:** Browser/Postman.
*   **`@app.get` (Decorator):** The **Waiter** taking the order.
*   **Function (`def`):** The **Chef** (logic).
*   **Pydantic Model:** The **Menu rules**. (e.g., "Burger price must be > 0").
*   **Dependency (`Depends`):** The **Kitchen Assistant** (Hands ingredients/DB to the chef).

### 2.2 Validating Inputs
Let's create a product, but ensure data is valid.

```python
from pydantic import BaseModel

# The Menu Rule (Schema)
class ProductBase(BaseModel):
    name: string
    price: float
    description: str | None = None # Optional

@app.post("/products/")
def create_product(product: ProductBase):
    # FastAPI validates 'product' against the Class before running this function
    return {"name": product.name, "tax_price": product.price * 1.2}
```

**Task:** Try sending `{ "price": "hello" }` in Swagger. FastAPI will reject it with a clear error message.

---

##  Part 3: Database with SQLModel

We will use **SQLModel**. It's special because a single class defines both the **API Schema** (Pydantic) and the **Database Table** (SQLAlchemy).

### 3.1 Define the Table
Create a file `models.py`.

```python
from typing import Optional
from sqlmodel import Field, SQLModel

class Product(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    price: float = Field(default=0, ge=0) # ge=0 means Greater or Equal to 0
    description: Optional[str] = None
```

### 3.2 Database Connection
In `database.py`:

```python
from sqlmodel import SQLModel, create_engine, Session

sqlite_file_name = "database.db"
sqlite_url = f"sqlite:///{sqlite_file_name}"

engine = create_engine(sqlite_url, echo=True)

def create_db_and_tables():
    SQLModel.metadata.create_all(engine)

# The Dependency Injection function
def get_session():
    with Session(engine) as session:
        yield session
```

### 3.3 Saving Data (Dependency Injection)
Update `main.py` to use the DB.

```python
from fastapi import FastAPI, Depends
from sqlmodel import Session
from database import create_db_and_tables, get_session
from models import Product

app = FastAPI()

@app.on_event("startup")
def on_startup():
    create_db_and_tables()

@app.post("/products/")
def create_product(product: Product, session: Session = Depends(get_session)):
    session.add(product)
    session.commit()
    session.refresh(product)
    return product

@app.get("/products/")
def read_products(session: Session = Depends(get_session)):
    return session.query(Product).all()
```

**Key Concept:** `session: Session = Depends(get_session)`
This tells FastAPI: *"Before running the function, go call `get_session`, give me the database connection, and close it when I'm done."*

---

## Part 4: Relationships (Categories & Products)

Let's make it professional. Products belong to Categories.

### 4.1 Update Models
We need to link tables. This is the hardest syntax part.
Update `models.py`:

```python
from typing import List, Optional
from sqlmodel import Field, Relationship, SQLModel

# 1. Category Table
class Category(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    # Link to Products
    products: List["Product"] = Relationship(back_populates="category")

# 2. Product Table
class Product(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str
    price: float

    # Foreign Key
    category_id: Optional[int] = Field(default=None, foreign_key="category.id")
    # Link to Category Object
    category: Optional[Category] = Relationship(back_populates="products")
```

### 4.2 Create Category Endpoint
In `main.py`, add routes to create categories.

```python
from models import Category

@app.post("/categories/")
def create_category(category: Category, session: Session = Depends(get_session)):
    session.add(category)
    session.commit()
    session.refresh(category)
    return category
```

### 4.3 Create Product with Category ID
**Challenge:** When creating a product, if we send `category_id`, SQLModel handles the link automatically.

**Task:**
1. Create a Category "Electronics" (ID 1).
2. Create a Product `{ "name": "MacBook", "price": 1000, "category_id": 1 }`.
3. Create a new endpoint `GET /categories/{id}` that returns the category **and** its products.

---

## Part 5: Refactoring & Router

A `main.py` with 500 lines is bad practice. Let's split it.

### 5.1 Create `routers/` folder
Create `routers/products.py`:

```python
from fastapi import APIRouter, Depends
from sqlmodel import Session
from database import get_session
from models import Product

router = APIRouter(prefix="/products", tags=["products"])

@router.get("/")
def read_products(session: Session = Depends(get_session)):
    return session.query(Product).all()
```

### 5.2 Mount it in Main
In `main.py`:

```python
from routers import products

app.include_router(products.router)
```

**Goal:** Move all Product logic to `routers/products.py` and Category logic to `routers/categories.py`.

---

## Part 6: Automated Testing (Bonus)

**Context:** FastAPI is designed to be easy to test.
**Goal:** Write a test that creates a product without breaking the real database.

### 6.1 Install Pytest
```bash
pip install pytest httpx
```

### 6.2 Write the Test
Create `test_main.py`.

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_read_main():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Welcome to the Shop API"}

def test_create_product():
    # We assume DB is setup (in real life we mock the session)
    response = client.post("/products/", json={"name": "TestItem", "price": 10})
    assert response.status_code == 200
    assert response.json()["name"] == "TestItem"
```

**Run it:**
```bash
pytest
```

---

## Troubleshooting Tips
*   **"Method Not Allowed":** Did you use `@app.get` instead of `@app.post`?
*   **Import Errors:** Ensure you are in the root folder when running uvicorn.
*   **Circular Imports:** If Models fail, keep them in one file for this workshop, or use quotes `"Product"` inside Relationship.

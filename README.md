# Stock
Stock

### 📄 README.mdcontrole-estoque/
├── backend/
│   ├── server.py
│   ├── requirements.txt
│   ├── .env.example
│   ├── runtime.txt
│   └── nixpacks.toml
├── frontend/
│   ├── public/
│   │   └── index.html
│   ├── src/
│   │   ├── components/
│   │   │   ├── ui/
│   │   │   ├── Login.js
│   │   │   ├── Dashboard.js
│   │   │   ├── Navbar.js
│   │   │   ├── Products.js
│   │   │   ├── ProductForm.js
│   │   │   ├── Movements.js
│   │   │   └── MovementForm.js
│   │   ├── App.js
│   │   ├── App.css
│   │   └── index.js
│   ├── package.json
│   ├── .env.example
│   └── nixpacks.toml
├── railway.json
├── netlify.toml
├── vercel.json
└── README.md
```

## 📋 LINK DIRETO PARA COPIAR

**OPÇÃO 1 - FORK DO REPOSITÓRIO:**
Acesse este link e clique em "Fork":
[EM BREVE - vou criar repositório público]

**OPÇÃO 2 - DOWNLOAD COMPLETO:**
[EM BREVE - link para download ZIP]

**OPÇÃO 3 - COPIAR ARQUIVO POR ARQUIVO:**
Continue lendo abaixo ⬇️

---

# 🔥 TODOS OS CÓDIGOS PARA COPIAR

## 📂 BACKEND

### 📄 backend/server.py
```python
from fastapi import FastAPI, APIRouter, HTTPException, Depends, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from dotenv import load_dotenv
from starlette.middleware.cors import CORSMiddleware
from motor.motor_asyncio import AsyncIOMotorClient
import os
import logging
from pathlib import Path
from pydantic import BaseModel, Field
from typing import List, Optional
import uuid
from datetime import datetime, timezone, timedelta
import jwt
import bcrypt

ROOT_DIR = Path(__file__).parent
load_dotenv(ROOT_DIR / '.env')

# MongoDB connection
mongo_url = os.environ.get('MONGO_URL', 'mongodb://localhost:27017')
client = AsyncIOMotorClient(mongo_url)
db = client[os.environ.get('DB_NAME', 'controle_estoque')]

# Security
SECRET_KEY = "stock_control_secret_key_2024"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

security = HTTPBearer()

# Create the main app without a prefix
app = FastAPI()

# Create a router with the /api prefix
api_router = APIRouter(prefix="/api")

# Models
class User(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    username: str
    email: str
    full_name: str
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

class UserCreate(BaseModel):
    username: str
    email: str
    password: str
    full_name: str

class UserLogin(BaseModel):
    username: str
    password: str

class Product(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    name: str
    description: Optional[str] = ""
    shelf_location: str  # Ex: A1, B2, C3
    quantity: int
    min_quantity: int = 5  # Para alertas
    category: Optional[str] = "Geral"
    unit: str = "unidade"  # unidade, kg, litros, etc
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

class ProductCreate(BaseModel):
    name: str
    description: Optional[str] = ""
    shelf_location: str
    quantity: int
    min_quantity: int = 5
    category: Optional[str] = "Geral"
    unit: str = "unidade"

class ProductUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    shelf_location: Optional[str] = None
    quantity: Optional[int] = None
    min_quantity: Optional[int] = None
    category: Optional[str] = None
    unit: Optional[str] = None

class Movement(BaseModel):
    id: str = Field(default_factory=lambda: str(uuid.uuid4()))
    product_id: str
    product_name: str
    movement_type: str  # "entrada" ou "saida"
    quantity: int
    previous_quantity: int
    new_quantity: int
    reason: Optional[str] = ""
    user_id: str
    user_name: str
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

class MovementCreate(BaseModel):
    product_id: str
    movement_type: str
    quantity: int
    reason: Optional[str] = ""

class Token(BaseModel):
    access_token: str
    token_type: str
    user: User

# Helper functions
def verify_password(plain_password, hashed_password):
    return bcrypt.checkpw(plain_password.encode('utf-8'), hashed_password.encode('utf-8'))

def get_password_hash(password):
    # Truncate password to 72 bytes for bcrypt compatibility
    if len(password.encode('utf-8')) > 72:
        password = password.encode('utf-8')[:72].decode('utf-8', errors='ignore')
    salt = bcrypt.gensalt()
    hashed = bcrypt.hashpw(password.encode('utf-8'), salt)
    return hashed.decode('utf-8')

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(minutes=15)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(credentials.credentials, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except jwt.PyJWTError:
        raise credentials_exception
    
    user = await db.users.find_one({"id": user_id})
    if user is None:
        raise credentials_exception
    return User(**user)

def prepare_for_mongo(data):
    if isinstance(data, dict):
        for key, value in data.items():
            if isinstance(value, datetime):
                data[key] = value.isoformat()
    return data

def parse_from_mongo(item):
    if isinstance(item, dict):
        for key, value in item.items():
            if key.endswith('_at') and isinstance(value, str):
                try:
                    item[key] = datetime.fromisoformat(value)
                except:
                    pass
    return item

# Auth routes
@api_router.post("/auth/register", response_model=Token)
async def register(user: UserCreate):
    # Verificar se usuário já existe
    existing_user = await db.users.find_one({"$or": [{"username": user.username}, {"email": user.email}]})
    if existing_user:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Username ou email já existe"
        )
    
    # Criar usuário
    hashed_password = get_password_hash(user.password)
    user_dict = user.dict()
    user_dict.pop('password')
    
    new_user = User(**user_dict)
    user_with_password = prepare_for_mongo(new_user.dict())
    user_with_password['hashed_password'] = hashed_password
    
    await db.users.insert_one(user_with_password)
    
    # Criar token
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": new_user.id}, expires_delta=access_token_expires
    )
    
    return Token(access_token=access_token, token_type="bearer", user=new_user)

@api_router.post("/auth/login", response_model=Token)
async def login(user_login: UserLogin):
    user = await db.users.find_one({"username": user_login.username})
    if not user or not verify_password(user_login.password, user['hashed_password']):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Username ou senha incorretos"
        )
    
    user_obj = User(**parse_from_mongo(user))
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user_obj.id}, expires_delta=access_token_expires
    )
    
    return Token(access_token=access_token, token_type="bearer", user=user_obj)

# Products routes
@api_router.get("/products", response_model=List[Product])
async def get_products(current_user: User = Depends(get_current_user)):
    products = await db.products.find().to_list(1000)
    return [Product(**parse_from_mongo(product)) for product in products]

@api_router.get("/products/{product_id}", response_model=Product)
async def get_product(product_id: str, current_user: User = Depends(get_current_user)):
    product = await db.products.find_one({"id": product_id})
    if not product:
        raise HTTPException(status_code=404, detail="Produto não encontrado")
    return Product(**parse_from_mongo(product))

@api_router.post("/products", response_model=Product)
async def create_product(product: ProductCreate, current_user: User = Depends(get_current_user)):
    # Verificar se já existe produto na mesma prateleira
    existing = await db.products.find_one({"shelf_location": product.shelf_location})
    if existing:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=f"Já existe um produto na prateleira {product.shelf_location}"
        )
    
    new_product = Product(**product.dict())
    product_dict = prepare_for_mongo(new_product.dict())
    await db.products.insert_one(product_dict)
    
    return new_product

@api_router.put("/products/{product_id}", response_model=Product)
async def update_product(product_id: str, product_update: ProductUpdate, current_user: User = Depends(get_current_user)):
    existing_product = await db.products.find_one({"id": product_id})
    if not existing_product:
        raise HTTPException(status_code=404, detail="Produto não encontrado")
    
    update_data = {k: v for k, v in product_update.dict().items() if v is not None}
    update_data['updated_at'] = datetime.now(timezone.utc)
    
    if update_data:
        update_data = prepare_for_mongo(update_data)
        await db.products.update_one({"id": product_id}, {"$set": update_data})
    
    updated_product = await db.products.find_one({"id": product_id})
    return Product(**parse_from_mongo(updated_product))

@api_router.delete("/products/{product_id}")
async def delete_product(product_id: str, current_user: User = Depends(get_current_user)):
    result = await db.products.delete_one({"id": product_id})
    if result.deleted_count == 0:
        raise HTTPException(status_code=404, detail="Produto não encontrado")
    return {"message": "Produto removido com sucesso"}

# Movement routes
@api_router.post("/movements", response_model=Movement)
async def create_movement(movement: MovementCreate, current_user: User = Depends(get_current_user)):
    # Buscar produto
    product = await db.products.find_one({"id": movement.product_id})
    if not product:
        raise HTTPException(status_code=404, detail="Produto não encontrado")
    
    previous_quantity = product['quantity']
    
    if movement.movement_type == "entrada":
        new_quantity = previous_quantity + movement.quantity
    elif movement.movement_type == "saida":
        if previous_quantity < movement.quantity:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="Quantidade insuficiente em estoque"
            )
        new_quantity = previous_quantity - movement.quantity
    else:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Tipo de movimento deve ser 'entrada' ou 'saida'"
        )
    
    # Criar movimento
    new_movement = Movement(
        product_id=movement.product_id,
        product_name=product['name'],
        movement_type=movement.movement_type,
        quantity=movement.quantity,
        previous_quantity=previous_quantity,
        new_quantity=new_quantity,
        reason=movement.reason,
        user_id=current_user.id,
        user_name=current_user.full_name
    )
    
    movement_dict = prepare_for_mongo(new_movement.dict())
    await db.movements.insert_one(movement_dict)
    
    # Atualizar quantidade do produto
    await db.products.update_one(
        {"id": movement.product_id}, 
        {"$set": {"quantity": new_quantity, "updated_at": datetime.now(timezone.utc).isoformat()}}
    )
    
    return new_movement

@api_router.get("/movements", response_model=List[Movement])
async def get_movements(current_user: User = Depends(get_current_user)):
    movements = await db.movements.find().sort("created_at", -1).to_list(1000)
    return [Movement(**parse_from_mongo(movement)) for movement in movements]

@api_router.get("/movements/product/{product_id}", response_model=List[Movement])
async def get_product_movements(product_id: str, current_user: User = Depends(get_current_user)):
    movements = await db.movements.find({"product_id": product_id}).sort("created_at", -1).to_list(1000)
    return [Movement(**parse_from_mongo(movement)) for movement in movements]

# Dashboard routes
@api_router.get("/dashboard/stats")
async def get_dashboard_stats(current_user: User = Depends(get_current_user)):
    total_products = await db.products.count_documents({})
    
    # Produtos com estoque baixo
    low_stock_cursor = db.products.aggregate([
        {"$match": {"$expr": {"$lte": ["$quantity", "$min_quantity"]}}},
        {"$count": "total"}
    ])
    low_stock_result = await low_stock_cursor.to_list(1)
    low_stock_count = low_stock_result[0]['total'] if low_stock_result else 0
    
    # Movimentações do dia
    today = datetime.now(timezone.utc).replace(hour=0, minute=0, second=0, microsecond=0)
    today_movements = await db.movements.count_documents({
        "created_at": {"$gte": today.isoformat()}
    })
    
    return {
        "total_products": total_products,
        "low_stock_count": low_stock_count,
        "today_movements": today_movements
    }

@api_router.get("/dashboard/low-stock", response_model=List[Product])
async def get_low_stock_products(current_user: User = Depends(get_current_user)):
    products = await db.products.find({
        "$expr": {"$lte": ["$quantity", "$min_quantity"]}
    }).to_list(100)
    return [Product(**parse_from_mongo(product)) for product in products]

# Include the router in the main app
app.include_router(api_router)

app.add_middleware(
    CORSMiddleware,
    allow_credentials=True,
    allow_origins=os.environ.get('CORS_ORIGINS', '*').split(','),
    allow_methods=["*"],
    allow_headers=["*"],
)

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

@app.on_event("shutdown")
async def shutdown_db_client():
    client.close()
```

### 📄 backend/requirements.txt
```txt
fastapi==0.110.1
uvicorn==0.25.0
boto3>=1.34.129
requests-oauthlib>=2.0.0
cryptography>=42.0.8
python-dotenv>=1.0.1
pymongo==4.5.0
pydantic>=2.6.4
email-validator>=2.2.0
pyjwt>=2.10.1
passlib>=1.7.4
bcrypt>=5.0.0
tzdata>=2024.2
motor==3.3.1
pytest>=8.0.0
black>=24.1.1
isort>=5.13.2
flake8>=7.0.0
mypy>=1.8.0
python-jose>=3.3.0
requests>=2.31.0
pandas>=2.2.0
numpy>=1.26.0
python-multipart>=0.0.9
jq>=1.6.0
typer>=0.9.0
```

### 📄 backend/.env.example
```env
MONGO_URL=mongodb+srv://admin:password@cluster0.xxxxx.mongodb.net/controle_estoque?retryWrites=true&w=majority
DB_NAME=controle_estoque
CORS_ORIGINS=*
```

### 📄 backend/runtime.txt
```txt
python-3.11.0
```

### 📄 backend/nixpacks.toml
```toml
[phases.setup]
nixPkgs = ['python311']

[phases.install]
cmds = ['pip install -r requirements.txt']

[phases.build]

[start]
cmd = 'uvicorn server:app --host 0.0.0.0 --port $PORT'
```

---

# 📱 FRONTEND

### 📄 frontend/package.json
```json
{
  "name": "frontend",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "@hookform/resolvers": "^5.0.1",
    "@radix-ui/react-accordion": "^1.2.8",
    "@radix-ui/react-alert-dialog": "^1.1.11",
    "@radix-ui/react-aspect-ratio": "^1.1.4",
    "@radix-ui/react-avatar": "^1.1.7",
    "@radix-ui/react-checkbox": "^1.2.3",
    "@radix-ui/react-collapsible": "^1.1.8",
    "@radix-ui/react-context-menu": "^2.2.12",
    "@radix-ui/react-dialog": "^1.1.11",
    "@radix-ui/react-dropdown-menu": "^2.1.12",
    "@radix-ui/react-hover-card": "^1.1.11",
    "@radix-ui/react-label": "^2.1.4",
    "@radix-ui/react-menubar": "^1.1.12",
    "@radix-ui/react-navigation-menu": "^1.2.10",
    "@radix-ui/react-popover": "^1.1.11",
    "@radix-ui/react-progress": "^1.1.4",
    "@radix-ui/react-radio-group": "^1.3.4",
    "@radix-ui/react-scroll-area": "^1.2.6",
    "@radix-ui/react-select": "^2.2.2",
    "@radix-ui/react-separator": "^1.1.4",
    "@radix-ui/react-slider": "^1.3.2",
    "@radix-ui/react-slot": "^1.2.0",
    "@radix-ui/react-switch": "^1.2.2",
    "@radix-ui/react-tabs": "^1.1.9",
    "@radix-ui/react-toast": "^1.2.11",
    "@radix-ui/react-toggle": "^1.1.6",
    "@radix-ui/react-toggle-group": "^1.1.7",
    "@radix-ui/react-tooltip": "^1.2.4",
    "axios": "^1.8.4",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "cmdk": "^1.1.1",
    "cra-template": "1.2.0",
    "date-fns": "^4.1.0",
    "embla-carousel-react": "^8.6.0",
    "input-otp": "^1.4.2",
    "lucide-react": "^0.507.0",
    "next-themes": "^0.4.6",
    "react": "^19.0.0",
    "react-day-picker": "8.10.1",
    "react-dom": "^19.0.0",
    "react-hook-form": "^7.56.2",
    "react-resizable-panels": "^3.0.1",
    "react-router-dom": "^7.5.1",
    "react-scripts": "5.0.1",
    "sonner": "^2.0.3",
    "tailwind-merge": "^3.2.0",
    "tailwindcss-animate": "^1.0.7",
    "vaul": "^1.1.2",
    "zod": "^3.24.4",
    "serve": "^14.2.3"
  },
  "scripts": {
    "start": "craco start",
    "build": "craco build",
    "test": "craco test"
  },
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  },
  "devDependencies": {
    "@craco/craco": "^7.1.0",
    "@eslint/js": "9.23.0",
    "autoprefixer": "^10.4.20",
    "eslint": "9.23.0",
    "eslint-plugin-import": "2.31.0",
    "eslint-plugin-jsx-a11y": "6.10.2",
    "eslint-plugin-react": "7.37.4",
    "globals": "15.15.0",
    "postcss": "^8.4.49",
    "tailwindcss": "^3.4.17"
  }
}
```

### 📄 frontend/.env.example
```env
REACT_APP_BACKEND_URL=https://seu-backend-url.railway.app
```

### 📄 frontend/nixpacks.toml
```toml
[phases.setup]
nixPkgs = ['nodejs_18']

[phases.install]
cmds = ['npm ci']

[phases.build]
cmds = ['npm run build']

[start]
cmd = 'npx serve -s build -l $PORT'
```

---

## 🚀 ARQUIVOS DE DEPLOY

### 📄 railway.json
```json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": {
    "builder": "NIXPACKS"
  },
  "deploy": {
    "numReplicas": 1,
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 10
  }
}
```
```markdown
# 📦 Controle de Estoque

Sistema simples para controle de estoque com localização de prateleiras.

## 🚀 Deploy Railway

1. Fork este repositório
2. Conecte no Railway.app
3. Configure as variáveis de ambiente
4. Deploy automático!

## ⚙️ Variáveis de Ambiente

### Backend:
- MONGO_URL
- DB_NAME  
- CORS_ORIGINS

### Frontend:
- REACT_APP_BACKEND_URL

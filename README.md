## Desafio Final: API REST com FastAPI, Python e Docker

Este documento descreve os passos para criar uma API REST com FastAPI, Python e Docker, seguindo as instruções fornecidas.

### 1. Estrutura do Projeto

Crie a seguinte estrutura de diretórios para o projeto:

```
workout_api/
├── app/
│   ├── __init__.py
│   ├── main.py         # Aplicação FastAPI
│   ├── models.py       # Modelos do SQLAlchemy
│   ├── schemas.py      # Schemas Pydantic
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── atleta.py   # Rotas para atletas
│   ├── database.py     # Configuração do banco de dados
│   ├── config.py       # Arquivo de configuração
├── docker-compose.yml
├── Dockerfile
├── .env                # Variáveis de ambiente
├── README.md
└── requirements.txt
```

### 2. Configuração do Ambiente

1.  **Variáveis de Ambiente (`.env`)**:

    Crie um arquivo `.env` na raiz do projeto e defina as variáveis de ambiente para a conexão com o banco de dados (ex: PostgreSQL).

    ```
    DATABASE_URL=postgresql+asyncpg://usuario:senha@host:porta/database
    ```

2.  **Dockerfile**:

    Crie um arquivo `Dockerfile` na raiz do projeto para construir a imagem Docker da aplicação.

    ```dockerfile
    FROM python:3.11-slim-bookworm

    WORKDIR /app

    COPY requirements.txt .

    RUN pip install --no-cache-dir -r requirements.txt

    COPY . .

    CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
    ```

3.  **Docker Compose (`docker-compose.yml`)**:

    Crie um arquivo `docker-compose.yml` na raiz do projeto para definir os serviços da aplicação (e.g., aplicação, banco de dados).

    ```yaml
    version: "3.9"
    services:
      app:
        build: .
        ports:
          - "8000:8000"
        depends_on:
          - db
        environment:
          DATABASE_URL: postgresql+asyncpg://usuario:senha@db:5432/database
      db:
        image: postgres:15
        ports:
          - "5432:5432"
        environment:
          POSTGRES_USER: usuario
          POSTGRES_PASSWORD: senha
          POSTGRES_DB: database
    ```

4.  **Requisitos (`requirements.txt`)**:

    Liste as dependências do projeto no arquivo `requirements.txt`.

    ```
    fastapi
    uvicorn[standard]
    pydantic[email]
    sqlalchemy[asyncio]
    asyncpg
    python-dotenv
    fastapi-pagination
    pydantic-settings
    ```

### 3. Desenvolvimento da API

1.  **Configuração do Banco de Dados (`app/database.py`)**:

    Configure a conexão com o banco de dados usando SQLAlchemy.

    ```python
    from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
    from sqlalchemy.orm import sessionmaker
    from app.config import settings

    engine = create_async_engine(settings.DATABASE_URL, echo=False)

    async def get_session() -> AsyncSession:
        async with sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)() as session:
            yield session
    ```

2.  **Modelos (`app/models.py`)**:

    Defina os modelos do banco de dados usando SQLAlchemy.

    ```python
    from sqlalchemy import Column, Integer, String, UUID, ForeignKey
    from sqlalchemy.orm import relationship
    from sqlalchemy.ext.declarative import declarative_base
    import uuid
    from sqlalchemy.dialects.postgresql import UUID

    Base = declarative_base()

    class Atleta(Base):
        __tablename__ = "atletas"
        id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
        nome = Column(String, nullable=False)
        cpf = Column(String, unique=True, nullable=False)
        categoria_id = Column(Integer, ForeignKey("categorias.id"))
        centro_treinamento_id = Column(Integer, ForeignKey("centros_treinamento.id"))

        categoria = relationship("Categoria", back_populates="atletas")
        centro_treinamento = relationship("CentroTreinamento", back_populates="atletas")

    class Categoria(Base):
        __tablename__ = "categorias"
        id = Column(Integer, primary_key=True)
        nome = Column(String, nullable=False)

        atletas = relationship("Atleta", back_populates="categoria")

    class CentroTreinamento(Base):
        __tablename__ = "centros_treinamento"
        id = Column(Integer, primary_key=True)
        nome = Column(String, nullable=False)

        atletas = relationship("Atleta", back_populates="centro_treinamento")
    ```

3.  **Schemas (`app/schemas.py`)**:

    Defina os schemas Pydantic para validação dos dados.

    ```python
    from pydantic import BaseModel

    class AtletaBase(BaseModel):
        nome: str
        cpf: str
        categoria_nome: str
        centro_treinamento_nome: str

    class AtletaCreate(AtletaBase):
        pass

    class AtletaOut(AtletaBase):
        id: str
    ```

4.  **Rotas (`app/routers/atleta.py`)**:

    Crie as rotas para a entidade "Atleta".

    ```python
    from fastapi import APIRouter, Depends, HTTPException, status, Query
    from sqlalchemy.ext.asyncio import AsyncSession
    from sqlalchemy import select
    from app import models, schemas
    from app.database import get_session
    from sqlalchemy.exc import IntegrityError
    from fastapi_pagination import Page, paginate

    router = APIRouter()

    @router.post("/", response_model=schemas.AtletaOut, status_code=status.HTTP_201_CREATED)
    async def create_atleta(atleta: schemas.AtletaCreate, db: AsyncSession = Depends(get_session)):
        # Verificar se a categoria e o centro de treinamento existem
        categoria = await db.execute(select(models.Categoria).where(models.Categoria.nome == atleta.categoria_nome))
        categoria = categoria.scalar_one_or_none()
        centro_treinamento = await db.execute(select(models.CentroTreinamento).where(models.CentroTreinamento.nome == atleta.centro_treinamento_nome))
        centro_treinamento = centro_treinamento.scalar_one_or_none()

        if not categoria or not centro_treinamento:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Categoria ou centro de treinamento não encontrado")

        # Criar o atleta
        db_atleta = models.Atleta(nome=atleta.nome, cpf=atleta.cpf, categoria_id=categoria.id, centro_treinamento_id=centro_treinamento.id)
        db.add(db_atleta)
        try:
            await db.commit()
            await db.refresh(db_atleta)
        except IntegrityError as e:
            await db.rollback()
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail="Já existe um atleta cadastrado com esse CPF")

        return db_atleta

    @router.get("/", response_model=Page[schemas.AtletaOut])
    async def list_atletas(
        db: AsyncSession = Depends(get_session),
        nome: str | None = Query(default=None, description="Filtrar por nome"),
        cpf: str | None = Query(default=None, description="Filtrar por CPF"),
        limit: int = Query(default=10, description="Limite de itens por página"),
        offset: int = Query(default=0, description="Offset para paginação")
    ):
        query = select(models.Atleta)
        if nome:
            query = query.where(models.Atleta.nome.ilike(f"%{nome}%"))
        if cpf:
            query = query.where(models.Atleta.cpf == cpf)
        
        atletas = await db.execute(query)
        atletas = atletas.scalars().all()
        return paginate(atletas, offset=offset, limit=limit)

    @router.get("/{atleta_id}", response_model=schemas.AtletaOut)
    async def get_atleta(atleta_id: int, db: AsyncSession = Depends(get_session)):
        atleta = await db.execute(select(models.Atleta).where(models.Atleta.id == atleta_id))
        atleta = atleta.scalar_one_or_none()
        if atleta is None:
            raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Atleta não encontrado")
        return atleta
    ```

5.  **Aplicação FastAPI (`app/main.py`)**:

    Crie a instância da aplicação FastAPI e inclua as rotas.

    ```python
    from fastapi import FastAPI
    from app.routers import atleta
    from fastapi_pagination import add_pagination
    from fastapi.middleware.cors import CORSMiddleware

    app = FastAPI(title="Workout API")

    # CORS - Cross Origin Resource Sharing
    origins = ["*"]  # Permite todas as origens, ajuste para produção

    app.add_middleware(
        CORSMiddleware,
        allow_origins=origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    app.include_router(atleta.router, prefix="/atletas", tags=["atletas"])

    add_pagination(app)
    ```

6.  **Configuração (`app/config.py`)**:

    Crie as configurações para o aplicativo.

    ```python
    from pydantic_settings import BaseSettings, SettingsConfigDict
    from dotenv import load_dotenv
    import os

    load_dotenv()

    class Settings(BaseSettings):
        DATABASE_URL: str
        model_config = SettingsConfigDict(env_file='../../.env')

    settings = Settings()
    ```

### 4. Implementação das Funcionalidades do Desafio

1.  **Adicionar Query Parameters**:
    *   No endpoint `GET /atletas/`, adicione parâmetros de query `nome` e `cpf` para filtrar os atletas por nome e CPF.

2.  **Customizar Response do Endpoint `GET /atletas/`**:

    *   Modifique o endpoint para retornar o nome, centro de treinamento e categoria do atleta na resposta. Atualize o schema `AtletaOut` para refletir essas mudanças.

3.  **Manipular Exceção de Integridade dos Dados**:

    *   Ao tentar criar um atleta com um CPF já existente, capture a exceção `IntegrityError` do SQLAlchemy e retorne a mensagem: `"Já existe um atleta cadastrado com o CPF: {cpf}"`.

4.  **Adicionar Paginação**:
    *   Utilize a biblioteca `fastapi-pagination` para adicionar paginação aos resultados do endpoint `GET /atletas/`. Adicione os parâmetros de query `limit` e `offset` para controlar o número de itens por página e o deslocamento.

### 5. Executando a Aplicação

1.  **Construir e Iniciar os Contêineres Docker**:

    Na raiz do projeto, execute o comando:

    ```bash
    docker-compose up --build
    ```

2.  **Acessar a API**:

    A API estará disponível em `http://localhost:8000`. Acesse as rotas e explore a funcionalidade.

### 6. Testando a API

Utilize ferramentas como Postman ou cURL para testar os endpoints da API. Alguns exemplos:

*   **Criar um atleta (POST)**:

    ```bash
    curl -X POST -H "Content-Type: application/json" -d '{"nome": "Novo Atleta", "cpf": "12345678900", "categoria_nome": "Senior", "centro_treinamento_nome": "CT1"}' http://localhost:8000/atletas/
    ```

*   **Listar atletas (GET)**:

    ```bash
    curl http://localhost:8000/atletas/?limit=5&offset=0
    ```

### 7. Considerações Finais

Este guia fornece um passo a passo completo para criar uma API REST com FastAPI, Python e Docker, atendendo aos requisitos específicos do desafio final. Ao seguir as etapas e adaptar o código às suas necessidades, você terá uma API robusta e escalável.

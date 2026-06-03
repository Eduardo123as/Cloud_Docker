# PRÁCTICA

## API TODO LIST con Flask + PostgreSQL + Docker

---

## OBJETIVO

Construir una API REST funcional que:

* Cree tareas
* Consulte tareas
* Guarde datos en PostgreSQL
* Corra en Docker

---

## REGLAS

* Puedes usar IA (ChatGPT, Copilot)
* No copiar proyectos completos
* Debes entender tu código
* Si no puedes explicarlo → no vale

---

# PARTE 1: SETUP

```bash
docker-compose down -v 2>/dev/null
docker system prune -f

mkdir todo-api
cd todo-api
```

---

# PARTE 2: ESTRUCTURA

Crea estos archivos (vacíos primero):

```
Dockerfile
requirements.txt
app.py
docker-compose.yml
.env
```

---

## requirements.txt

Completa con las librerías necesarias para:

* API web
* PostgreSQL
* Variables de entorno

Ejemplo esperado (puedes investigarlo):

```
Flask==3.0.0
psycopg2-binary==2.9.9 
python-dotenv==1.0.0
```

---

## Dockerfile

Corrige y completa:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

---

## docker-compose.yml

Corrige la estructura (indentación es clave):

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: todouser
      POSTGRES_PASSWORD: todopass
      POSTGRES_DB: tododb
    ports:
      - "5432:5432"
```

---

# PARTE 3: API BÁSICA

## Objetivo

Crear endpoints:

* GET /
* GET /todos
* POST /todos

---

## Pistas

### 1. Crear app Flask

```python
from flask import Flask
app = Flask(__name__)
```

---

### 2. Endpoint de prueba

```python
@app.route('/')
def home():
    return {"msg": "ok"}
```

---

### 3. POST /todos

Debes:

* Leer JSON
* Obtener `title`
* Validar que exista

Pregunta clave:
¿Cómo se obtiene JSON en Flask?

---

### 4. GET /todos

Por ahora:

* Usa una lista en memoria

```python
todos = []
```

---

# PARTE 4: BASE DE DATOS

## Objetivo

Conectar PostgreSQL

---

## Tu tarea

Debes lograr:

* Conexión a DB
* Crear tabla `todos`
* Insertar datos
* Consultar datos

---

## Pistas

### Librería

```python
import psycopg2
```

---

### Conexión

```python
psycopg2.connect("URL")
```

---

### SQL mínimo

```sql
CREATE TABLE todos (
  id SERIAL PRIMARY KEY,
  title TEXT
);
```

---

Pregunta clave:
¿Cómo ejecutar queries en Python?

---

# PARTE 5: DOCKER

## Objetivo

Levantar todo con:

```bash
docker-compose up -d
```

---

## Debes resolver

* ¿Por qué falla la conexión?
* ¿La DB tarda en iniciar?
* ¿Variables de entorno correctas?

---

# PARTE 6: PRUEBAS

```bash
curl http://localhost:5000/

curl -X POST http://localhost:5000/todos \
-H "Content-Type: application/json" \
-d '{"title":"Tarea"}'

curl http://localhost:5000/todos
```

---

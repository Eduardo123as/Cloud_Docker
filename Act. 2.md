# PRÁCTICA: API CLIMA OPTIMIZADA
## Con .env, .gitignore y GitHub Secrets. TODO en UNO.

---

## PARTE 1: LIMPIAR DOCKER

```bash
docker-compose down -v 2>/dev/null
docker stop $(docker ps -aq) 2>/dev/null
docker rm $(docker ps -aq) 2>/dev/null
docker volume prune -f 2>/dev/null
```

---

## PARTE 2: CREAR CARPETA

```bash
mkdir weather-api
cd weather-api

pwd
# Deberías ver: /Users/tu_usuario/weather-api
```

---

## PARTE 3: CREAR 4 ARCHIVOS

### ARCHIVO 1: `Dockerfile`

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

**Guarda como: `Dockerfile`**

---

### ARCHIVO 2: `requirements.txt`

```
Flask==3.0.0
requests==2.31.0
python-dotenv==1.0.0
```

**Guarda como: `requirements.txt`**

---

### ARCHIVO 3: `.env.example`

Este archivo es PÚBLICO (para que otros sepan qué variables necesitan). No tiene valores reales.

```
OPENWEATHER_API_KEY=your_api_key_here
FLASK_ENV=development
FLASK_DEBUG=False
PORT=5000
```

**Guarda como: `.env.example`**

**IMPORTANTE:** Este archivo SÍ se sube a GitHub. Es solo un template.

---

### ARCHIVO 4: `.env`

Este archivo es PRIVADO (no se sube a GitHub).

```
OPENWEATHER_API_KEY=tu_api_key_aqui
FLASK_ENV=development
FLASK_DEBUG=False
PORT=5000
```

**Guarda como: `.env`**

**IMPORTANTE:** Este archivo NUNCA se sube a GitHub (lo bloquea .gitignore).

---

### ARCHIVO 5: `.gitignore`

Esto le dice a Git: "No subas estos archivos a GitHub".

```
# Archivos de entorno
.env
.env.local
.env.*.local

# Python
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
*.egg-info/
dist/
build/
.venv
venv/
ENV/

# IDE
.vscode/
.idea/
*.swp
*.swo
*~
.DS_Store

# Docker
.dockerignore
```

**Guarda como: `.gitignore`**

---

### ARCHIVO 6: `app.py`

```python
import os
import requests
from flask import Flask, jsonify, request
from datetime import datetime
from dotenv import load_dotenv

# Cargar variables del .env
load_dotenv()

app = Flask(__name__)

# Leer API key desde .env
API_KEY = os.getenv('OPENWEATHER_API_KEY')
FLASK_ENV = os.getenv('FLASK_ENV', 'development')
PORT = int(os.getenv('PORT', 5000))

@app.route('/health', methods=['GET'])
def health():
    return jsonify({
        'status': 'healthy',
        'service': 'Weather API',
        'environment': FLASK_ENV,
        'timestamp': datetime.utcnow().isoformat()
    }), 200

@app.route('/', methods=['GET'])
def home():
    return jsonify({
        'message': 'Weather API - OpenWeather',
        'environment': FLASK_ENV,
        'endpoints': {
            'GET /health': 'Health check',
            'GET /weather?city=London': 'Get weather by city',
            'GET /weather?lat=51.5&lon=-0.1': 'Get weather by coordinates',
            'POST /weather/multiple': 'Get weather for multiple cities'
        }
    }), 200

@app.route('/weather', methods=['GET'])
def get_weather():
    try:
        # Validar que tenemos API key
        if not API_KEY or API_KEY == 'demo':
            return jsonify({
                'error': 'API key not configured. Get one at openweathermap.org',
                'note': 'Set OPENWEATHER_API_KEY in .env file'
            }), 503
        
        city = request.args.get('city')
        lat = request.args.get('lat')
        lon = request.args.get('lon')
        
        if not city and not (lat and lon):
            return jsonify({
                'error': 'Provide city OR lat/lon parameters',
                'example': '/weather?city=London'
            }), 400
        
        base_url = 'https://api.openweathermap.org/data/2.5/weather'
        
        if city:
            params = {
                'q': city,
                'appid': API_KEY,
                'units': 'metric'
            }
        else:
            params = {
                'lat': lat,
                'lon': lon,
                'appid': API_KEY,
                'units': 'metric'
            }
        
        response = requests.get(base_url, params=params, timeout=5)
        
        if response.status_code == 401:
            return jsonify({
                'error': 'Invalid API key',
                'hint': 'Check your OPENWEATHER_API_KEY in .env'
            }), 401
        
        if response.status_code == 404:
            return jsonify({
                'error': 'City not found'
            }), 404
        
        if response.status_code != 200:
            return jsonify({
                'error': f'OpenWeather API error: {response.status_code}'
            }), response.status_code
        
        data = response.json()
        
        weather_data = {
            'city': data.get('name'),
            'country': data.get('sys', {}).get('country'),
            'temperature': data.get('main', {}).get('temp'),
            'feels_like': data.get('main', {}).get('feels_like'),
            'humidity': data.get('main', {}).get('humidity'),
            'pressure': data.get('main', {}).get('pressure'),
            'weather': data.get('weather', [{}])[0].get('main'),
            'description': data.get('weather', [{}])[0].get('description'),
            'wind_speed': data.get('wind', {}).get('speed'),
            'cloudiness': data.get('clouds', {}).get('all'),
            'timestamp': datetime.utcnow().isoformat()
        }
        
        return jsonify(weather_data), 200
    
    except requests.exceptions.Timeout:
        return jsonify({
            'error': 'OpenWeather API timeout'
        }), 504
    except Exception as e:
        return jsonify({
            'error': str(e)
        }), 500

@app.route('/weather/multiple', methods=['POST'])
def get_multiple_weather():
    try:
        # Validar que tenemos API key
        if not API_KEY or API_KEY == 'demo':
            return jsonify({
                'error': 'API key not configured'
            }), 503
        
        data = request.get_json()
        cities = data.get('cities', [])
        
        if not cities:
            return jsonify({
                'error': 'No cities provided',
                'example': '{"cities": ["London", "Paris", "Tokyo"]}'
            }), 400
        
        results = []
        base_url = 'https://api.openweathermap.org/data/2.5/weather'
        
        for city in cities:
            params = {
                'q': city,
                'appid': API_KEY,
                'units': 'metric'
            }
            
            try:
                response = requests.get(base_url, params=params, timeout=5)
                if response.status_code == 200:
                    data = response.json()
                    results.append({
                        'city': data.get('name'),
                        'temperature': data.get('main', {}).get('temp'),
                        'weather': data.get('weather', [{}])[0].get('main'),
                        'status': 'success'
                    })
                else:
                    results.append({
                        'city': city,
                        'status': 'error',
                        'error': f'HTTP {response.status_code}'
                    })
            except Exception as e:
                results.append({
                    'city': city,
                    'status': 'error',
                    'error': str(e)
                })
        
        return jsonify({'results': results}), 200
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(debug=FLASK_ENV == 'development', host='0.0.0.0', port=PORT)
```

**Guarda como: `app.py`**

---

## PARTE 4: VERIFICA LOS 6 ARCHIVOS

```bash
ls -la

# Deberías ver:
# Dockerfile
# requirements.txt
# .env
# .env.example
# .gitignore
# app.py
```

---

## PARTE 5: TEST LOCAL SIN API KEY

```bash
# Build
docker build -t weather-api:1.0 .

# Run (sin API key, solo test health)
docker run -d \
  -p 5000:5000 \
  --name weather \
  weather-api:1.0

# Test health (funciona sin API key)
curl http://localhost:5000/health

# Output:
# {"status":"healthy","service":"Weather API",...}
```

---

## PARTE 6: TEST CON API KEY 

Si tienes API key real:

### Paso 1: Obtén API key 

1. Ve a: https://openweathermap.org/api
2. Sign up (gratis)
3. API keys → Copia tu key
4. Espera 10 min a que se active

### Paso 2: Actualiza .env

Edita el archivo `.env`:

```
OPENWEATHER_API_KEY=tu_api_key_real_aqui
FLASK_ENV=development
FLASK_DEBUG=False
PORT=5000
```

Guarda.

### Paso 3: Rebuild con .env

```bash
# Detén anterior
docker stop weather
docker rm weather

# Rebuild (agarra variables de .env local)
docker build -t weather-api:1.0 .

# Run
docker run -d \
  -p 5000:5000 \
  --env-file .env \
  --name weather \
  weather-api:1.0

# Test con API key
curl "http://localhost:5000/weather?city=London"

# Output: Datos reales de clima en Londres
```

---

## PARTE 7: PARAR

```bash
docker stop weather
docker rm weather
```

---

## PARTE 8: GIT + GITIGNORE

Esto es IMPORTANTE. El .gitignore evita que subas .env a GitHub.

```bash
# Verifica que Git ve solo lo que tiene que ver
git status

# Output esperado:
# Untracked files:
#   .env.example
#   .gitignore
#   Dockerfile
#   app.py
#   requirements.txt
#
# NO DEBE VER: .env (porque está en .gitignore)
```

Si ves `.env` en la lista → problema. Verifica .gitignore.

---

## PARTE 9: GITHUB SETUP

### Paso 1: Crear repo

1. Ve a: https://github.com/new
2. Nombre: `weather-api`
3. Public
4. Create

### Paso 2: Git local

```bash
# En la carpeta weather-api

git init
git add .
git config user.email "tu@email.com"
git config user.name "Tu Nombre"
git commit -m "Initial: Weather API with .env and .gitignore"
git branch -M main
git remote add origin https://github.com/tu_usuario/weather-api.git
git push -u origin main
```

### Paso 3: VERIFICA EN GITHUB

Ve a GitHub → Abre el repo. Deberías ver:

```
✅ Dockerfile
✅ requirements.txt
✅ .env.example
✅ .gitignore
✅ app.py
✅ .git/

 NO deberías ver .env (está protegido por .gitignore)
```

## PARTE 10: GITHUB SECRETS (Para que Actions use tu API key)

### Paso 1: En GitHub, ve a Settings

1. Tu repo en GitHub
2. Settings → Secrets and variables → Actions
3. Click: "New repository secret"

### Paso 2: Agregar secreto

**Name:** `OPENWEATHER_API_KEY`
**Value:** Tu API key real (pega aquí)

Click: "Add secret"

Ahora GitHub tiene tu API key SEGURA. No está en el código.

---

## PARTE 11: CREAR GITHUB ACTION

Crea este archivo:

**Ruta:** `.github/workflows/docker-build.yml`

```yaml
name: Build and Test Weather API

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t weather-api:latest .

      - name: Test health endpoint (without API key)
        run: |
          docker run -d \
            -p 5000:5000 \
            --name weather-test \
            weather-api:latest
          
          sleep 3
          
          curl -f http://localhost:5000/health || exit 1

      - name: Test home endpoint
        run: curl -f http://localhost:5000/ || exit 1

      - name: Test with API key
        run: |
          docker stop weather-test
          docker rm weather-test
          
          docker run -d \
            -p 5000:5000 \
            -e OPENWEATHER_API_KEY="${{ secrets.OPENWEATHER_API_KEY }}" \
            --name weather-test \
            weather-api:latest
          
          sleep 3
          
          # Test que API key se carga correctamente
          curl -f "http://localhost:5000/weather?city=London" || exit 1

      - name: Success 
        run: echo "Build and all tests passed!"
```

**Guarda como:** `.github/workflows/docker-build.yml`

**IMPORTANTE:** Ves `${{ secrets.OPENWEATHER_API_KEY }}`? Eso toma el secret que agregaste arriba. SEGURO.

---

## PARTE 12: PUSH CON GITHUB ACTION

```bash
git add .github/
git commit -m "Add GitHub Actions with secrets"
git push origin main
```

---

## PARTE 13: VER EN GITHUB

1. Repo en GitHub
2. Click: **Actions**
3. Verás que está corriendo el build
4. Espera 2-3 min
5. Si está ✅ → Todo bien
6. Si está ❌ → Click para ver error

---

## PARTE 14: AHORA: CADA VEZ QUE HAGAS PUSH

```bash
# Editas app.py (por ejemplo)

git add app.py
git commit -m "Update: nuevo endpoint"
git push origin main

# GitHub Actions automáticamente:
# 1. Descarga código
# 2. Buildea imagen
# 3. Corre tests (con tu API key del secret)
# 4. ✅ o ❌ resultado visible en GitHub Actions
```

**TÚ NO TIENES QUE HACER NADA MÁS, QUEDO AUTOMATIZADO.**

---

## PARTE 15: ESTRUCTURA FINAL

```
weather-api/
├── .github/
│   └── workflows/
│       └── docker-build.yml
├── .env                    ← PRIVADO (no en GitHub)
├── .env.example            ← PÚBLICO (template)
├── .gitignore              ← Protege .env
├── Dockerfile
├── requirements.txt
├── app.py
└── .git/
```

---

## PARTE 16: LO QUE SUBE A GITHUB

```
✅ Sube:
- Dockerfile
- requirements.txt
- app.py
- .env.example
- .gitignore
- .github/workflows/docker-build.yml

❌ NO sube:
- .env (protegido por .gitignore)
- __pycache__/
- .venv/
- .DS_Store
```

---

## PARTE 17: SEGURIDAD RESUMIDA

**Tu API key:**
- ✅ En `.env` local (tu máquina)
- ✅ En GitHub Secrets (GitHub, protegido)
- ❌ Nunca en el código
- ❌ Nunca en GitHub repo públicamente

**Cómo funciona:**
1. GitHub Actions ejecuta el workflow
2. Accede al secret: `${{ secrets.OPENWEATHER_API_KEY }}`
3. Se lo pasa al container como variable
4. El container lo usa para testear
5. Después se olvida (no se guarda)

---

## PARTE 18: COMANDOS ÚTILES

### Ver variables en container

```bash
docker run -it --env-file .env weather-api:1.0 bash

# Dentro del container
echo $OPENWEATHER_API_KEY
# Output: tu_api_key (si está en .env)

exit
```

---

### Verificar .gitignore

```bash
git check-ignore -v .env
# Output: .env

# Significa: .env está ignorado ✅
```

---

### Ver qué Git va a subir

```bash
git status

# Importante: NO debe mostrar .env
```

---

## PARTE 19: SI ALGO FALLA

| Problema | Solución |
|----------|----------|
| `.env` aparece en GitHub | Borra y vuelve a hacer (git rm .env) |
| Actions falla con API key | Verifica que agregaste el secret en GitHub → Settings |
| Docker no ve .env | Usa `--env-file .env` en docker run |
| "API key not configured" | Asegurate que .env existe y tiene OPENWEATHER_API_KEY |
| Port 5000 in use | Cambia PORT en .env a 5001 |

---

## PARTE 21: PRÓXIMOS PASOS

### Agregar más endpoints

Edita `app.py`, haz push:
```bash
git add app.py
git commit -m "Add forecast endpoint"
git push origin main
```

GitHub Actions corre automáticamente.

---

### Cambiar API key

1. GitHub → Settings → Secrets
2. Edita `OPENWEATHER_API_KEY`

---
4. Pega nueva key
5. Listo. Próximo run de Actions usa la nueva.

---

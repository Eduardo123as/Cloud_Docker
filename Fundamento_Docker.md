# Docker: Guía Completa

## ¿Qué es Docker?

Docker es una herramienta que soluciona un problema fundamental en desarrollo: **la inconsistencia entre ambientes**.

### El Problema

Escribes una aplicación en tu máquina. Funciona perfectamente. Pero cuando la subes a producción (servidor empresarial, AWS, etc.), se rompe.

¿Por qué?

- Tu máquina tiene Python 3.11, el servidor tiene 3.9
- Instalaste una librería que funciona en tu máquina, pero el servidor no la tiene
- El sistema operativo es diferente
- Las dependencias no coinciden

El famoso: *"Pero funciona en mi máquina..."*

### La Solución

Docker empaqueta todo junto en una **caja aislada**:

- Tu código
- Todas las dependencias
- El sistema operativo mínimo
- La configuración completa

Esa caja funciona igual en tu máquina, en cualquier otra máquina, en AWS, en cualquier servidor. Sin importa qué tenga instalado el otro.

**Analogía**: Es como empacar un juego en una caja. El juego funciona en tu PC con Windows 10, Python 3.11 y librerías específicas. Cuando alguien en otro país abre la caja en su Linux, el juego funciona exactamente igual. Porque adentro de la caja está TODO.

---

## Tres Conceptos Clave

### 1. Image
Es la **plantilla**. Es como un molde de máquina virtual pero mucho más ligero. Una imagen es estática y reutilizable.

### 2. Container
Es la **image ejecutándose**. Es la instancia viva de una imagen. Cuando ejecutas una imagen, obtienes un container.

### 3. Dockerfile
Es la **receta**. Es un archivo de texto que le dices a Docker:

> "Haceme una image que tenga Python 3.11, copie mi código, instale mis dependencias, y cuando se ejecute, corra mi app"

### Analogía del Bizcochuelo

- **Dockerfile** = receta de bizcochuelo
- **Image** = bizcochuelo hecho pero sin comer
- **Container** = bizcochuelo en la mesa, comiendo

---

## Primeros Comandos

### Verificar instalación

```bash
docker --version
```

### Ejecutar un container existente

Docker tiene un repositorio central llamado **Docker Hub** con imágenes predefinidas.

```bash
docker run -d -p 8080:80 --name web nginx
```

Qué significa:
- `run` = ejecutar
- `-d` = detached (background, sin ver logs)
- `-p 8080:80` = mapear puerto 8080 de mi máquina al puerto 80 del container
- `--name web` = darle nombre al container
- `nginx` = la imagen a descargar y ejecutar

Docker descarga automáticamente la imagen de nginx, crea un container y lo ejecuta.

### Verificar que funciona

```bash
curl http://localhost:8080
```

Deberías ver una página HTML. El servidor nginx está corriendo.

### Ver containers activos

```bash
docker ps
```

Lista todos los containers en ejecución. Ves el nombre, la imagen, puertos, etc.

### Ver logs

```bash
docker logs web
```

Muestra los logs del container. Si hubiera errores, los verías aquí.

### Entrar dentro del container

```bash
docker exec -it web bash
```

Accedes a una terminal bash **dentro** del container. Puedes hacer:

```bash
ls
pwd
cat /etc/os-release  # Ver qué SO está adentro
```

Es un Ubuntu mínimo. Todo lo que Docker empaquetó.

Para salir:
```bash
exit
```

### Parar el container

```bash
docker stop web
```

El container se detiene.

### Eliminar el container

```bash
docker rm web
```

Elimina el container completamente.

Ahora `docker ps` no lo muestra.

---

## Crear Tu Propia Image: Dockerfile

No siempre usarás imágenes que ya existen. Querrás crear la tuya.

Para eso necesitas un **Dockerfile**: un archivo de texto con instrucciones.

### Estructura básica

```dockerfile
FROM <image_base>
WORKDIR <carpeta_adentro>
COPY <archivo_local> <ruta_container>
RUN <comando>
EXPOSE <puerto>
CMD ["comando", "a", "ejecutar"]
```

### Comandos principales del Dockerfile

| Comando | Significado |
|---------|-------------|
| `FROM` | La imagen base sobre la cual construir |
| `WORKDIR` | Carpeta de trabajo dentro del container |
| `COPY` | Copiar archivos de tu máquina al container |
| `RUN` | Ejecutar comandos durante la construcción (build time) |
| `EXPOSE` | Documentar qué puertos escucha el container |
| `CMD` | Comando por defecto cuando el container inicia |

---

## Tarea 1: Hello World

### Crear proyecto

```bash
mkdir docker-hola
cd docker-hola
```

### Archivo 1: Dockerfile

Crea un archivo llamado `Dockerfile` (sin extensión):

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY hello.py .

CMD ["python", "hello.py"]
```

**Qué dice:**
- `FROM python:3.11-slim` = usar Python 3.11 en versión slim (más ligero)
- `WORKDIR /app` = crear carpeta /app adentro del container y usarla como carpeta actual
- `COPY hello.py .` = copiar hello.py desde mi máquina a /app del container
- `CMD ["python", "hello.py"]` = cuando ejecutes el container, corre este comando

### Archivo 2: hello.py

```python
print("Hola desde Docker!")
```

Simple. Imprime un mensaje.

### Construir la image

```bash
docker build -t hola:1.0 .
```

**Qué significa:**
- `build` = construir una image
- `-t hola:1.0` = tag: nombre y versión de la image
- `.` = usar el Dockerfile en la carpeta actual

Docker:
1. Lee el Dockerfile línea por línea
2. Descarga Python 3.11
3. Entra a carpeta /app
4. Copia hello.py
5. Registra el comando a ejecutar
6. Crea la image con nombre "hola" versión "1.0"

Cuando ves `Successfully tagged hola:1.0`, está lista.

### Ejecutar el container

```bash
docker run hola:1.0
```

**Output:**
```
Hola desde Docker!
```

¡Eso! El container ejecutó, corrió Python dentro, ejecutó hello.py, imprimió el mensaje y terminó.

Si alguien en otro país ejecuta `docker run hola:1.0`, vería exactamente lo mismo.

---

## Tarea 2: Flask API

Ahora algo más real: una API REST con Flask.

### Crear proyecto

```bash
mkdir docker-api
cd docker-api
```

### Archivo 1: requirements.txt

```
Flask==3.0.0
```

Este archivo lista las dependencias de Python. `pip install -r requirements.txt` las instala automáticamente.

### Archivo 2: app.py

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({'message': 'Hola desde Docker!'})

@app.route('/health')
def health():
    return jsonify({'status': 'ok'})

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
```

**Qué hace:**
- Define dos endpoints:
  - `GET /` = retorna `{"message": "Hola desde Docker!"}`
  - `GET /health` = retorna `{"status": "ok"}`
- Escucha en puerto 5000
- `host='0.0.0.0'` = escucha en todas las interfaces (importante para containers)

### Archivo 3: Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

**Qué dice:**
- `FROM python:3.11-slim` = base Python 3.11
- `WORKDIR /app` = carpeta de trabajo
- `COPY requirements.txt .` = copiar el archivo de dependencias
- `RUN pip install -r requirements.txt` = instalar dependencias (esto ocurre en **build time**, no en runtime)
- `COPY app.py .` = copiar el código de la app
- `EXPOSE 5000` = documentar que el container escucha en puerto 5000 (es solo información, no hace nada automáticamente)
- `CMD ["python", "app.py"]` = comando a ejecutar

**Detalle importante:** COPY requirements.txt antes que app.py. ¿Por qué?

Docker cachea las capas. Si cambias app.py, no necesita reinstalar pip. Si cambias requirements.txt, reinstala. Esta es una optimización.

### Construir la image

```bash
docker build -t mi-api:1.0 .
```

Docker:
1. Descarga Python
2. Entra a /app
3. Copia requirements.txt
4. Ejecuta `pip install -r requirements.txt` (puede tardar)
5. Copia app.py
6. Documenta puerto 5000
7. Listo: image creada con nombre "mi-api" versión "1.0"

### Ejecutar el container

```bash
docker run -d -p 5000:5000 --name api mi-api:1.0
```

**Qué significa:**
- `-d` = detached (corre en background)
- `-p 5000:5000` = puerto 5000 de mi máquina → puerto 5000 del container
- `--name api` = nombre del container
- `mi-api:1.0` = la image a ejecutar

### Probar la API

En otra terminal o en la misma:

```bash
curl http://localhost:5000/
```

**Response:**
```json
{"message":"Hola desde Docker!"}
```

```bash
curl http://localhost:5000/health
```

**Response:**
```json
{"status":"ok"}
```

¡La API funciona! El container está escuchando en puerto 5000 de su máquina virtual, tú lo mapeaste a puerto 5000 de tu máquina, y llamas desde afuera. Funciona.

### Ver logs del container

```bash
docker logs -f api
```

Muestra los logs del container. `-f` significa "follow" (ver logs nuevos en tiempo real).

### Entrar al container

```bash
docker exec -it api bash
```

Adentro puedes:

```bash
ls          # Ver archivos
cat app.py  # Ver el código
python -c "import flask; print(flask.__version__)"  # Ver versión de Flask
```

Verás que todo está ahí: Python, Flask instalado, el código. Todo empaquetado.

Salir:
```bash
exit
```

### Parar y limpiar

```bash
docker stop api
docker rm api
```

El container se detiene y se elimina. `docker ps` no lo muestra.

---

## Resumen

Docker soluciona el problema de "funciona en mi máquina".

**Los conceptos clave:**
- **Image** = template estática
- **Container** = image ejecutándose
- **Dockerfile** = receta para crear images

**El workflow:**
1. Escribes un Dockerfile
2. Ejecutas `docker build` para crear la image
3. Ejecutas `docker run` para crear y ejecutar un container
4. Usas `docker ps`, `docker logs`, `docker exec` para inspeccionar

**La magia:**
Una image funciona igual en tu máquina, en otra máquina, en AWS, en cualquier servidor. Sin dependencias del host, sin conflictos, sin sorpresas en producción.

Eso es Docker.

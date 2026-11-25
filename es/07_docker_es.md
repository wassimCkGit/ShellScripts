# 07 · Docker para los desarrolladores

Docker permite empaquetar una aplicación **con todo lo que necesita para funcionar** (runtime, bibliotecas, archivos de configuración) en una unidad aislada y reproducible: el **contenedor**.

La promesa: el famoso *« funciona en mi máquina »* se convierte en *« funciona en todas partes, de la misma forma »*.

> [!NOTE]
> **Objetivos del capítulo**
> - Entender la diferencia entre una imagen, un contenedor y un volumen
> - Lanzar, inspeccionar, detener y eliminar contenedores
> - Escribir un `Dockerfile` y construir la propia imagen
> - Orquestar múltiples servicios con `docker compose`
> - Poner en marcha un entorno de desarrollo completo y desechable

## Contenido

1. [¿Contenedor o máquina virtual?](#contenedor-o-máquina-virtual-)
2. [Instalación](#instalación)
3. [Los tres conceptos básicos](#los-tres-conceptos-básicos)
4. [Manipular contenedores](#manipular-contenedores)
5. [Manipular imágenes](#manipular-imágenes)
6. [Escribir un Dockerfile](#escribir-un-dockerfile)
7. [La construcción multi-etapa](#la-construcción-multi-etapa)
8. [Construir la propia imagen](#construir-la-propia-imagen)
9. [Docker Compose](#docker-compose)
10. [Un entorno de desarrollo completo](#un-entorno-de-desarrollo-completo)
11. [Los ejemplos del repositorio](#los-ejemplos-del-repositorio)
12. [Resumen](#resumen)

---

## ¿Contenedor o máquina virtual?

Una **máquina virtual** emula una computadora completa: lleva su propio núcleo y su propio sistema operativo. Un **contenedor** comparte el núcleo de la máquina anfitriona e aísla solo lo necesario (procesos, red, sistema de archivos).

```text
   Máquinas virtuales                   Contenedores

  ┌──────┐ ┌──────┐ ┌──────┐          ┌──────┐ ┌──────┐ ┌──────┐
  │ App  │ │ App  │ │ App  │          │ App  │ │ App  │ │ App  │
  ├──────┤ ├──────┤ ├──────┤          ├──────┴─┴──────┴─┴──────┤
  │ OS   │ │ OS   │ │ OS   │          │    Motor Docker        │
  ├──────┴─┴──────┴─┴──────┤          ├────────────────────────┤
  │      Hipervisor        │          │   Núcleo Linux (anf.)  │
  ├────────────────────────┤          ├────────────────────────┤
  │   Núcleo Linux (anf.)  │          │        Hardware        │
  └────────────────────────┘          └────────────────────────┘
```

| | Máquina virtual | Contenedor |
|---|---|---|
| Inicio | Decenas de segundos | Algunas decenas de ms |
| Peso | Varios GB | Algunos Mo a cientos de Mo |
| Aislamiento | Fuerte (núcleo separado) | Bueno, pero núcleo **compartido** |
| Uso típico | Alojar sistemas operativos diferentes | Empaquetar una aplicación |

> [!IMPORTANT]
> Un contenedor **no es una pequeña VM**. Es uno o más procesos del anfitrión, aislados por dos mecanismos del núcleo Linux vistos indirectamente en el [capítulo 05](05_securite_es.md): los **namespaces** (lo que el proceso ve) y los **cgroups** (lo que tiene derecho a consumir).
>
> Consecuencia directa: en macOS y Windows, Docker ejecuta una **VM Linux** en segundo plano, porque el núcleo Linux debe estar en algún lugar.

---

## Instalación

En Debian / Ubuntu, a través del repositorio oficial:

```bash
# Requisitos previos
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Clave GPG del repositorio Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Añadir el repositorio
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalación
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verificación:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

### Usar Docker sin `sudo`

```bash
sudo usermod -aG docker $USER
```

Luego hay que **desconectarse y reconectarse** para que el nuevo grupo sea tenido en cuenta (ver el [capítulo 02.1](02_1_user_management_es.md)).

> [!CAUTION]
> El grupo `docker` equivale a acceso **root** en la máquina: cualquier miembro puede montar `/` en un contenedor y modificar todo el sistema. Solo concédaselo a usuarios a los que ya daría `sudo` sin contraseña. El [capítulo 08](08_docker_administration_es.md#el-modo-sin-raíz) presenta el modo *sin raíz*, que soluciona este problema.

> [!NOTE]
> `docker compose` (con espacio) es el complemento actual. `docker-compose` (con guion) es el antiguo script de Python, aún presente en muchas máquinas pero descontinuado. Los archivos `docker-compose.yml` siguen siendo idénticos.

---

## Los tres conceptos básicos

| Concepto | Analogía | Descripción |
|---------|----------|-------------|
| **Imagen** | Una clase | Un modelo de solo lectura: un sistema de archivos + un comando de inicio |
| **Contenedor** | Una instancia | Una ejecución de una imagen, con una capa de escritura encima |
| **Volumen** | Un disco externo | Un espacio de almacenamiento que **sobrevive** a la eliminación del contenedor |

```text
  Imagen  ──── docker run ───►  Contenedor  ──── docker commit / build ───►  Imagen
 (modelo)                        (instancia)
```

> [!IMPORTANT]
> **Un contenedor es desechable.** Todo lo escrito en su sistema de archivos desaparece con él. Todos los datos a conservar (base de datos, archivos cargados, logs) deben vivir en un **volumen**.

Una imagen se identifica por un **nombre** y una **etiqueta**:

```text
postgres:16-alpine
│        │
│        └── etiqueta: la versión (por defecto `latest`)
└── nombre del repositorio (aquí en Docker Hub)
```

> [!WARNING]
> La etiqueta `latest` no tiene nada de mágica: es simplemente la etiqueta por defecto, y **cambia** en el tiempo. Una imagen `node:latest` hoy y en seis meses no es la misma cosa. En un proyecto, siempre se fija una versión: `node:22-alpine`.

---

## Manipular contenedores

### Lanzar un contenedor

```bash
docker run nginx
```

La terminal permanece bloqueada: el contenedor se ejecuta en primer plano.

Las opciones que siempre reaparecen:

| Opción | Efecto |
|--------|-------|
| `-d` | *desconectado* — se ejecuta en segundo plano |
| `--name mi-nginx` | Da un nombre al contenedor (si no, Docker genera uno aleatorio) |
| `-p 8080:80` | Publica el puerto **80 del contenedor** en el puerto **8080 del anfitrión** |
| `-v $(pwd):/app` | Monta una carpeta del anfitrión en el contenedor |
| `-e VAR=valor` | Define una variable de entorno |
| `--rm` | Elimina automáticamente el contenedor al detenerlo |
| `-it` | Modo interactivo con una terminal (para un shell) |

En la práctica:

```bash
docker run -d --name web -p 8080:80 nginx
```

La aplicación está disponible en <http://localhost:8080>.

> [!TIP]
> **El sentido del `-p`:** `-p ANFITRIÓN:CONTENEDOR`. Se lee siempre de izquierda a derecha, del exterior hacia el interior. Misma lógica para `-v FUENTE_ANFITRIÓN:DESTINO_CONTENEDOR`.

### Un shell desechable

```bash
docker run -it --rm debian:12 bash
```

Este es exactamente el entorno de arenero propuesto en el [README](README_es.md#-puesta-en-marcha-del-entorno) para las manipulaciones de la formación.

### Listar

```bash
# Los contenedores que se ejecutan
docker ps

# Todos, incluso los que están detenidos
docker ps -a
```

```text
CONTAINER ID   IMAGE     COMMAND                  STATUS         PORTS                  NAMES
a1b2c3d4e5f6   nginx     "/docker-entrypoint.…"   Up 2 minutes   0.0.0.0:8080->80/tcp   web
```

### Pilotar

```bash
docker stop web        # detención limpia (SIGTERM, luego SIGKILL después de 10 s)
docker start web       # reinicio
docker restart web
docker rm web          # eliminación (el contenedor debe estar detenido)
docker rm -f web       # detención + eliminación
```

### Entrar en un contenedor que se ejecuta

```bash
docker exec -it web bash
```

Si la imagen se basa en Alpine, `bash` no existe:

```bash
docker exec -it web sh
```

> [!TIP]
> `docker exec` lanza un **nuevo** proceso en un contenedor existente. `docker run` crea un **nuevo contenedor**. Esta es la confusión n° 1 de los principiantes.

### Ver qué está pasando

```bash
docker logs web            # los logs (stdout / stderr del proceso principal)
docker logs -f web         # de forma continua, como un tail -f
docker logs --tail 50 web  # las últimas 50 líneas
docker stats               # consumo de CPU / RAM en tiempo real
docker inspect web         # toda la configuración, en JSON
docker top web             # los procesos del contenedor
```

> [!IMPORTANT]
> En un contenedor, una aplicación no debe escribir sus logs en un archivo: escribe en la **salida estándar**, y Docker se encarga del resto. Esto es lo que hace posible `docker logs` (y todas las herramientas de centralización).

### Copiar archivos

```bash
docker cp web:/etc/nginx/nginx.conf ./nginx.conf
docker cp ./nginx.conf web:/etc/nginx/nginx.conf
```

---

## Manipular imágenes

```bash
docker pull node:22-alpine   # descargar una imagen
docker images                # listar imágenes locales
docker rmi node:22-alpine    # eliminar una imagen
docker history nginx         # ver las capas de una imagen
```

### Las capas (*layers*)

Una imagen es un **apilamiento de capas** de solo lectura. Cada instrucción de un `Dockerfile` crea una capa, y las capas se **comparten** entre imágenes y se cachean.

```text
┌──────────────────────────┐  ← capa de escritura (el contenedor, efímero)
├──────────────────────────┤
│ CMD ["node", "app.js"]   │  ┐
│ COPY . .                 │  │
│ RUN npm ci               │  ├── la imagen (solo lectura)
│ COPY package*.json ./    │  │
│ FROM node:22-alpine      │  ┘
└──────────────────────────┘
```

Por eso la segundo `docker build` es casi instantáneo: Docker reutiliza las capas sin cambios. Y también por eso el **orden de las instrucciones importa** — lo trataremos más adelante.

---

## Escribir un Dockerfile

El `Dockerfile` es la receta de fabricación de la imagen: una serie de **pasos**, ejecutados de arriba hacia abajo, cada uno produce una capa.

> [!TIP]
> Todos los `Dockerfile` de este capítulo existen en versión completa, comentada línea por línea y **realmente construible** en la carpeta [`07_docker/`](07_docker/) del repositorio. El primero, [`01_simple/Dockerfile`](07_docker/01_simple/Dockerfile), retoma exactamente los pasos que se muestran aquí.

Un ejemplo para una pequeña aplicación Node:

```dockerfile
# --- PASO 1 · la imagen de inicio ---
FROM node:22-alpine

# --- PASO 2 · el directorio de trabajo (creado si no existe) ---
WORKDIR /app

# --- PASO 3 · los manifiestos de dependencias, solos ---
COPY package.json package-lock.json ./

# --- PASO 4 · la instalación: esta capa solo se reconstruye
#     si package.json o package-lock.json cambian ---
RUN npm ci --omit=dev

# --- PASO 5 · el código fuente, que cambia en cada commit ---
COPY src/ ./src/

# --- PASO 6 · la configuración de ejecución ---
ENV NODE_ENV=production
EXPOSE 3000

# --- PASO 7 · el usuario no privilegiado ---
USER node

# --- PASO 8 · el control de salud ---
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD wget -qO- http://127.0.0.1:3000/health || exit 1

# --- PASO 9 · el comando de inicio ---
CMD ["node", "src/server.js"]
```

Cada paso produce una capa, visible después de la construcción:

```bash
docker history mi-app:1.0
```

```text
IMAGE          CREATED BY                                      SIZE
a1b2c3d4e5f6   CMD ["node" "src/server.js"]                    0B
<missing>      HEALTHCHECK &{["CMD-SHELL" "wget -qO- http:…     0B
<missing>      USER node                                       0B
<missing>      EXPOSE map[3000/tcp:{}]                         0B
<missing>      ENV NODE_ENV=production                         0B
<missing>      COPY src/ ./src/                                12.4kB
<missing>      RUN npm ci --omit=dev                           4.21MB
<missing>      COPY package.json package-lock.json ./          1.02kB
<missing>      WORKDIR /app                                    0B
<missing>      /bin/sh -c #(nop) ADD file:… in /               142MB
```

Las instrucciones de **metadatos** (`ENV`, `EXPOSE`, `USER`, `CMD`, `HEALTHCHECK`) pesan 0 bytes: solo anotan la imagen. Solo aquellas que **escriben archivos** (`COPY`, `RUN`, `ADD`) cuestan espacio.

### Las instrucciones a conocer

| Instrucción | Rol |
|-------------|------|
| `FROM` | La imagen base — siempre la primera instrucción |
| `WORKDIR` | El directorio de trabajo de las instrucciones que siguen |
| `COPY src dst` | Copia desde el contexto de construcción a la imagen |
| `ADD` | Como `COPY`, pero maneja URL y descomprime archivos — **prefiera `COPY`** |
| `RUN` | Ejecuta un comando **durante la construcción** |
| `ENV VAR=valor` | Variable de entorno presente en la imagen |
| `ARG VAR` | Variable disponible **solo durante la construcción** |
| `EXPOSE` | Documenta un puerto (no lo publica — es `-p` quien lo hace) |
| `USER` | El usuario que ejecuta lo que sigue |
| `VOLUME` | Declara un punto de montaje persistente |
| `CMD` | El comando por defecto, **reemplazable** en línea de comando |
| `ENTRYPOINT` | El ejecutable del contenedor, **no reemplazable** por defecto |
| `HEALTHCHECK` | El comando que dice si el contenedor está en buen estado |

> [!NOTE]
> **¿`CMD` o `ENTRYPOINT`?** Con `ENTRYPOINT ["ping"]` y `CMD ["localhost"]`, `docker run mi-imagen google.com` ejecuta `ping google.com`. `ENTRYPOINT` fija el programa, `CMD` suministra los argumentos por defecto. Para una aplicación clásica, `CMD` solo es suficiente.

### `.dockerignore`

Como `.gitignore`, pero para el contexto de construcción. Sin él, toda la carpeta se envía al demonio Docker — incluyendo `node_modules` y `.git`.

```text
.git
node_modules
dist
*.log
.env
```

> [!WARNING]
> Nunca copie **nunca** un secreto (`.env`, clave privada, token) en una imagen. Aunque se elimine por una instrucción posterior, permanece legible en la capa donde fue añadido. Los secretos se pasan en la ejecución, mediante variables de entorno o un archivo montado.

### El orden de las instrucciones

Esta es la principal fuente de construcciones lentas:

```dockerfile
# ❌ Malo: cualquier cambio en el código reinstala todas las dependencias
COPY . .
RUN npm ci

# ✅ Bueno: las dependencias solo se reinstalan si package.json cambia
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

**Regla:** de lo más estable a lo más volátil.

---

## La construcción multi-etapa

**Construir** una aplicación y hacerla **funcionar** no requieren las mismas herramientas. Compilar necesita un SDK, un compilador, dependencias de desarrollo, fuentes; ejecutar a menudo solo necesita un binario o algunos archivos estáticos.

Un `Dockerfile` puede contener **múltiples `FROM`**: cada uno abre una nueva etapa, comenzando de cero. Se trabaja en las primeras, y se conserva solo la última — copiando explícitamente lo que se necesita.

```text
  ETAPA 1 « constructor »              ETAPA 2 « runtime »
 ┌───────────────────────┐        ┌───────────────────────┐
 │ node:22-alpine        │        │ nginx:alpine          │
 │ + npm, node_modules   │        │                       │
 │ + código fuente       │        │                       │
 │ + /app/dist ──────────┼───────►│ /usr/share/nginx/html │
 └───────────────────────┘ COPY   └───────────────────────┘
      descartada al final      --from=constructor    la imagen entregada
            235 Mo                                       93 Mo
```

> [!IMPORTANT]
> Lo que no se copia explícitamente con `COPY --from=` **no existe** en la imagen final. El compilador, las fuentes, los identificadores de un repositorio privado usados durante la construcción: todo esto permanece en la etapa intermedia, que nunca se publica.

### Paso a paso

El ejemplo completo está en [`07_docker/02_multistage_node/`](07_docker/02_multistage_node/):

```dockerfile
# ============ ETAPA 1/2 · constructor ============
# `AS constructor` nombra la etapa para poder extraer de ella después.
FROM node:22-alpine AS constructor

WORKDIR /app

# 1.a · las dependencias — aquí se instala TODO,
#       incluyendo devDependencies: se necesitan para compilar
COPY package*.json ./
RUN npm ci

# 1.b · las fuentes
COPY build.js ./
COPY src/ ./src/

# 1.c · la compilación: produce /app/dist
RUN npm run build

# ============ ETAPA 2/2 · runtime ============
# Nuevo FROM = imagen nueva. Nada de la etapa 1 continúa,
# excepto lo que se copia explícitamente.
FROM nginx:alpine

# 2.a · solo el resultado de la construcción
COPY --from=constructor /app/dist /usr/share/nginx/html

# 2.b · nuestra configuración
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
```

```bash
docker build -t demo:02 .

# También se puede detenerse en una etapa, para inspeccionarla o depurar
docker build --target constructor -t demo:02-constructor .
docker images | grep demo
```

Si tiene múltiples dockerfiles:
```shell
docker build -f ./Path/To/Target/Dockerfile -t demo:02 .
```

```text
demo   02-constructor   235MB
demo   02             92.7MB
```

### Tres casos concretos

| Lenguaje | Etapa de construcción | Etapa final | Lo que se deja atrás |
|---------|----------------|--------------|--------------------------|
| **Node → estático** ([ejemplo 02](07_docker/02_multistage_node/Dockerfile)) | `node:22-alpine` | `nginx:alpine` | Node, npm, `node_modules`, las fuentes — 235 Mo → 93 Mo |
| **Go** ([ejemplo 03](07_docker/03_multistage_go/Dockerfile)) | `golang:1.23-alpine` | `scratch` | Toda la cadena de compilación — solo queda un binario de 8 Mo |
| **Python** ([ejemplo 04](07_docker/04_python/Dockerfile)) | `python:3.12-slim` + `build-essential` | `python:3.12-slim` | `gcc`, encabezados de desarrollo, caché `pip` — 204 Mo al final |

El caso Go es el más espectacular, porque un binario estático no necesita **nada** para funcionar:

```dockerfile
FROM golang:1.23-alpine AS constructor
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
# CGO_ENABLED=0    → binario estático, sin dependencia a libc
# -ldflags "-s -w" → sin tabla de símbolos: binario más pequeño
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/servidor ./main.go

FROM scratch
COPY --from=constructor /out/servidor /servidor
USER 65534:65534
ENTRYPOINT ["/servidor"]
```

Aproximadamente **8 Mo** en total, contra más de 300 Mo si se entregara la imagen `golang` completa.

> [!WARNING]
> `scratch` no contiene **ningún** archivo: ni shell, ni `/etc/passwd`, ni certificados raíz. Consecuencias prácticas:
> - `docker exec -it … sh` es imposible — la depuración va solo a través de logs;
> - `USER app` falla (ningún nombre de usuario existe), hay que dar un **UID numérico**;
> - cualquier llamada HTTPS saliente falla hasta que no se copie `ca-certificates.crt` desde una etapa intermedia.
>
> La alternativa razonable: `gcr.io/distroless/static` o `alpine`, que conservan lo mínimo.

### Lo que la construcción multi-etapa aporta realmente

| Beneficio | Por qué |
|----------|----------|
| **Una imagen más pequeña** | Menos que transferir en cada despliegue, inicio más rápido |
| **Menos vulnerabilidades** | Un compilador ausente es un compilador sin CVE |
| **Sin fuga de secretos** | Un token de repositorio privado usado en la construcción no sale de la etapa donde sirvió |
| **Etapas paralelizables** | BuildKit construye simultáneamente las etapas independientes |
| **Un solo archivo, múltiples destinos** | `--target dev` o `--target runtime` — ver abajo |

### Una imagen de dev y una imagen de prod, en el mismo archivo

Este es el patrón utilizado por la stack de ejemplo ([`07_docker/05_stack_compose/api/Dockerfile`](07_docker/05_stack_compose/api/Dockerfile)): una etapa `base` común, luego dos destinos.

```dockerfile
FROM node:22-alpine AS base
WORKDIR /app
COPY package.json package-lock.json ./

# --- destino de desarrollo: herramientas completas, recarga en caliente ---
FROM base AS dev
RUN npm install
COPY src/ ./src/
CMD ["node", "--watch", "src/server.js"]

# --- destino de producción: dependencias mínimas, usuario no raíz ---
FROM base AS runtime
RUN npm ci --omit=dev && npm cache clean --force
COPY --chown=node:node src/ ./src/
USER node
CMD ["node", "src/server.js"]
```

```bash
docker build --target dev     -t stack-api:dev .
docker build --target runtime -t stack-api:1.0 .
```

En Compose, se elige por servicio:

```yaml
services:
  api:
    build:
      context: ./api
      target: dev        # `runtime` en archivo de superposición de producción
```

---

## Construir la propia imagen

```bash
# -t: el nombre (etiqueta) de la imagen; el "." final es el contexto de construcción
docker build -t mi-app:1.0 .
```

```bash
# Lanzar la imagen construida
docker run -d -p 3000:3000 --name app mi-app:1.0
```

Algunas opciones útiles:

```bash
docker build -t mi-app:1.0 --no-cache .            # ignorar caché
docker build -t mi-app:1.0 --target constructor .      # detenerse en una etapa
docker build -t mi-app:1.0 --build-arg VERSION=2 . # pasar un ARG
```

```dockerfile
FROM golang:1.23-alpine AS constructor
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
# CGO_ENABLED=0    → binario estático, sin dependencia a libc
# -ldflags "-s -w" → sin tabla de símbolos: binario más pequeño
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/servidor ./main.go
RUN echo $EXTERNAL_ARG

FROM scratch
COPY --from=constructor /out/servidor /servidor
USER 65534:65534
ENTRYPOINT ["/servidor"]
```

> [!TIP]
> Siempre etiquete dos veces: una versión precisa **y** un alias móvil.
>
> ```bash
> docker build -t mi-app:1.4.2 -t mi-app:latest .
> ```

---

## Docker Compose

Una aplicación real nunca es un solo contenedor: una API, una base de datos, un caché, un proxy inverso… Encadenar cinco `docker run` con sus opciones no es mantenible. **Compose** describe toda la stack en un archivo `docker-compose.yml`.

El archivo presente en la raíz de este repositorio, por ejemplo:

```yaml
services:
  db:
    image: postgres:12-alpine
    environment:
      POSTGRES_PASSWORD: 1234
      POSTGRES_USER: postgres
      POSTGRES_DB: app
    ports:
      - '5435:5432'
    volumes:
      - ./:/init_db
```

```bash
docker compose up -d
```

Un comando, y la base está funcionando.

### Un archivo más completo

```yaml
services:
  api:
    build: .                    # construye desde el Dockerfile local
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://postgres:1234@db:5432/app
      NODE_ENV: development
    volumes:
      - .:/app                  # el código del anfitrión, montado en directo
      - /app/node_modules       # …excepto node_modules, que permanece el de la imagen
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: 1234
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

  cache:
    image: redis:7-alpine

volumes:
  pgdata:
```

Puntos a recordar de este archivo:

- **`build: .`** construye la imagen desde el `Dockerfile` local; **`image:`** descarga una lista completa.
- **La red es automática**: los servicios se unen por su **nombre** (`db`, `cache`). La API se conecta a `db:5432`, no a `localhost`.
- **`depends_on` con `condition`** espera que la base esté realmente *lista*, no solo iniciada.
- **`pgdata`** es un volumen nombrado: los datos de la base sobreviven a `docker compose down`.
- **`- .:/app`** monta el código fuente directamente: modifica un archivo en el anfitrión, la aplicación lo ve inmediatamente.

> [!IMPORTANT]
> Desde Compose v2, la clave `version:` al inicio del archivo está **obsoleta** e ignorada (incluso genera una advertencia). Aún está presente en muchos archivos, incluido el de este repositorio.

### Los comandos Compose

| Comando | Efecto |
|----------|-------|
| `docker compose up -d` | Inicia toda la stack en segundo plano |
| `docker compose up -d --build` | …reconstruyendo las imágenes |
| `docker compose down` | Detiene y elimina contenedores y red |
| `docker compose down -v` | …**y los volúmenes** (así los datos) |
| `docker compose ps` | Estado de los servicios |
| `docker compose logs -f api` | Seguir los logs de un servicio |
| `docker compose exec api sh` | Abrir un shell en un servicio |
| `docker compose restart api` | Reiniciar un servicio |
| `docker compose config` | Mostrar la configuración final resuelta |

> [!CAUTION]
> `docker compose down -v` **elimina los volúmenes**, así los datos de su base. Es muy práctico para comenzar de cero en desarrollo, y catastrófico en otro lugar.

### Las variables de entorno

Compose lee automáticamente un archivo `.env` ubicado junto a `docker-compose.yml`:

```text
POSTGRES_PASSWORD=1234
API_PORT=3000
```

```yaml
services:
  api:
    ports:
      - "${API_PORT}:3000"
  db:
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

> [!TIP]
> Versionice un `.env.example` documentando las variables esperadas, y añada `.env` a `.gitignore`.

---

## Un entorno de desarrollo completo

El escenario típico: un nuevo desarrollador llega al proyecto y debe estar operativo en un comando.

```text
mi-proyecto/
├── docker-compose.yml
├── Dockerfile
├── .dockerignore
├── .env.example
└── src/
```

```bash
git clone git@github.com:org/mi-proyecto.git
cd mi-proyecto
cp .env.example .env
docker compose up -d
```

Ni Node, ni PostgreSQL, ni Redis para instalar en la máquina — y la misma versión para todo el equipo.

### Los reflejos útiles en el día a día

```bash
# Lanzar un comando puntual en un servicio
docker compose exec api npm run migrate

# Lanzar un comando en un contenedor desechable
docker compose run --rm api npm test

# Abrir un cliente psql en la base
docker compose exec db psql -U postgres -d app

# Comenzar completamente de cero
docker compose down -v && docker compose up -d --build
```

> [!TIP]
> Estos comandos son largos: son candidatos perfectos para alias o funciones shell, como se vio en el [capítulo 10](10_shell_config_es.md).
>
> ```bash
> alias dcu='docker compose up -d'
> alias dcd='docker compose down'
> alias dcl='docker compose logs -f'
> dcx() { docker compose exec "$1" "${@:2}"; }
> ```

### Depuración: los tres reflejos

| Síntoma | Comando |
|----------|----------|
| El contenedor se detiene por sí solo | `docker compose logs api` |
| « Conexión rechazada » hacia la base | Verificar el **nombre del servicio** (`db`, no `localhost`) |
| Una modificación del código no aparece | Verificar el montaje `-v` / reconstruir la imagen |

> [!WARNING]
> `localhost` **dentro de** un contenedor designa el contenedor mismo, no su máquina. Para unirse a otro servicio, se usa su nombre Compose; para unirse al anfitrión, `host.docker.internal` (o la IP de la pasarela en Linux).

---

## Los ejemplos del repositorio

La carpeta [`07_docker/`](07_docker/) contiene cinco proyectos autónomos, para construir y lanzar. Cada `Dockerfile` está comentado **paso a paso**: es la versión larga de este capítulo.

| Carpeta | Lo que muestra | Para ir a ver |
|---------|-----------------|-----------------|
| [`01_simple/`](07_docker/01_simple/) | Los 9 pasos de un `Dockerfile`, `.dockerignore`, caché de capas | `docker history` después de un cambio de código |
| [`02_multistage_node/`](07_docker/02_multistage_node/) | Multi-etapa: Node compila, `nginx` sirve — 235 Mo → 93 Mo | `docker build --target constructor` para comparar |
| [`03_multistage_go/`](07_docker/03_multistage_go/) | Una imagen `scratch` de 8 Mo | Intentar un `docker exec … sh`: no hay shell |
| [`04_python/`](07_docker/04_python/) | El `venv` construido en una etapa, copiado en la otra | El rol de `PYTHONUNBUFFERED=1` |
| [`05_stack_compose/`](07_docker/05_stack_compose/) | API + PostgreSQL + Redis, `healthcheck`, `.env`, superposición de producción | La API se une a `db` **por su nombre** |

```bash
cd 07_docker/05_stack_compose
cp .env.example .env
docker compose up -d --build
curl http://localhost:3000
```

```json
{
  "message": "API de la stack de demostración",
  "services": {
    "db":    { "host": "db",    "ip": "172.22.0.2", "joignable": true },
    "cache": { "host": "cache", "joignable": true }
  }
}
```

> [!TIP]
> El manual de uso detallado de cada ejemplo, con las manipulaciones a hacer para ver bien qué sucede, está en [`07_docker/README_es.md`](07_docker/README_es.md).

---

## Resumen

| Comando | Rol |
|----------|------|
| `docker run -d -p 8080:80 --name web nginx` | Lanzar un contenedor en segundo plano |
| `docker ps` / `docker ps -a` | Listar contenedores activos / todos |
| `docker logs -f nombre` | Seguir los logs de un contenedor |
| `docker exec -it nombre bash` | Abrir un shell en un contenedor que se ejecuta |
| `docker stop` / `start` / `rm -f` | Pilotar y eliminar un contenedor |
| `docker images` / `docker rmi` | Listar / eliminar imágenes |
| `docker build -t nombre:tag .` | Construir una imagen desde un `Dockerfile` |
| `docker compose up -d --build` | Iniciar toda la stack |
| `docker compose down -v` | Detener todo, **volúmenes incluidos** |
| `docker compose exec svc cmd` | Ejecutar un comando en un servicio |

### Las cinco reglas a recordar

> [!TIP]
> 1. **Un contenedor es desechable** — los datos van en un volumen.
> 2. **Una imagen fija su versión** — nunca `latest` en proyecto.
> 3. **De lo más estable a lo más volátil** en el `Dockerfile` — para el caché.
> 4. **Los logs van a stdout**, nunca en un archivo.
> 5. **Ningún secreto en una imagen** — se pasan en la ejecución.

---

⬅️ [Anterior: 06.6 · Funciones](06_shell_script/06_funciones/funciones_es.md) · 🏠 [Tabla de contenidos](README_es.md) · [Siguiente: 08 · Docker para el administrador ➡️](08_docker_administration_es.md)

# 08 · Docker para el administrador de sistemas

El [capítulo 07](07_docker_es.md) mostraba Docker desde la perspectiva del desarrollador: construir una imagen, lanzar una stack, trabajar. Este capítulo toma el otro punto de vista — el de la persona que debe hacer **funcionar** estos contenedores en un servidor, vigilarlos, asegurarlos e impedir que el disco se llene por sí solo un domingo por la noche.

> [!NOTE]
> **Objetivos del capítulo**
> - Entender la arquitectura del demonio Docker y saber configurarlo
> - Gestionar volúmenes, redes y persistencia de datos
> - Dominar logs, su rotación y el consumo de recursos
> - Reforzar la configuración: usuario no privilegiado, capabilities, modo *sin raíz*
> - Guardar, restaurar, supervisar y limpiar un anfitrión Docker

## Contenido

1. [Arquitectura del demonio](#arquitectura-del-demonio)
2. [Configurar el demonio](#configurar-el-demonio)
3. [Los volúmenes y la persistencia](#los-volúmenes-y-la-persistencia)
4. [La red](#la-red)
5. [Los logs](#los-logs)
6. [Limitar recursos](#limitar-recursos)
7. [Asegurar contenedores](#asegurar-contenedores)
8. [El modo sin raíz](#el-modo-sin-raíz)
9. [Limpieza y espacio en disco](#limpieza-y-espacio-en-disco)
10. [Copia de seguridad y restauración](#copia-de-seguridad-y-restauración)
11. [Docker en producción](#docker-en-producción)
12. [Supervisión y diagnóstico](#supervisión-y-diagnóstico)
13. [Resumen](#resumen)

---

## Arquitectura del demonio

Docker no es un programa, es una **arquitectura cliente / servidor**:

```text
   docker ps          ┌──────────────────────────────────┐
  (cliente CLI)  ────► │  dockerd  (demonio, se ejecuta como root)│
                      │     └── containerd               │
                      │            └── runc ── contenedor │
                      └──────────────────────────────────┘
                        socket: /var/run/docker.sock
```

| Componente | Rol |
|-----------|------|
| `docker` | El cliente de línea de comandos — solo envía solicitudes HTTP |
| `dockerd` | El demonio: imágenes, redes, volúmenes, API |
| `containerd` | El gestor de ciclo de vida de contenedores |
| `runc` | Lo que crea realmente el contenedor (namespaces + cgroups) |

El demonio es un servicio `systemd` clásico, pilotado como cualquier otro servicio del [capítulo 04](04_instalacion_y_servicios_es.md):

```bash
systemctl status docker
systemctl restart docker
systemctl enable --now docker
journalctl -u docker -f
```

> [!CAUTION]
> **`/var/run/docker.sock` es la clave del reino.** Cualquiera que pueda escribir en esta socket puede lanzar un contenedor privilegiado montando `/`, así convertirse en root en el anfitrión. Esto es cierto para el grupo `docker`, y es cierto para cualquier contenedor al que se le monte esta socket (`-v /var/run/docker.sock:/var/run/docker.sock`, un clásico de herramientas de monitoreo). Hágalo solo con conocimiento de causa.

### Dónde Docker guarda sus cosas

```bash
docker info
```

```text
Storage Driver: overlay2
Docker Root Dir: /var/lib/docker
Cgroup Driver: systemd
```

```text
/var/lib/docker/
├── overlay2/     # las capas de imágenes y contenedores — aquí es donde crece
├── volumes/      # volúmenes nombrados
├── containers/   # metadatos y archivos de logs
└── image/        # índice de imágenes
```

> [!WARNING]
> Nunca modifique **nunca** `/var/lib/docker` a mano (ni `rm -rf` en una subcarpeta). Todo se pilota mediante comandos `docker`. Una eliminación manual deja el demonio con metadatos inconsistentes.

> [!TIP]
> En un servidor, `/var/lib/docker` merece su **propia partición** o su propio volumen LVM. Una imagen de más no debe llenar `/` y bloquear todo el sistema.

---

## Configurar el demonio

La configuración se realiza en `/etc/docker/daemon.json` (a crear si no existe):

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "data-root": "/srv/docker",
  "default-address-pools": [
    { "base": "172.30.0.0/16", "size": 24 }
  ],
  "live-restore": true,
  "userland-proxy": false
}
```

| Clave | Efecto |
|-----|-------|
| `log-driver` / `log-opts` | El controlador de logs por defecto y su rotación |
| `data-root` | Mover `/var/lib/docker` a otro lugar (disco dedicado) |
| `default-address-pools` | Los rangos IP usados para redes creadas — útil en caso de conflicto con la red empresarial |
| `live-restore` | Los contenedores continúan ejecutándose durante un reinicio del demonio |
| `icc: false` | Prohíbe la comunicación entre contenedores en la red por defecto |

```bash
# Verificar sintaxis antes de reiniciar
dockerd --validate --config-file /etc/docker/daemon.json

sudo systemctl restart docker
```

> [!IMPORTANT]
> Un JSON inválido en `daemon.json` **impide que el demonio inicie** — y que todos los contenedores se reinicien. Siempre valide antes de reiniciar, y guarde una copia del archivo anterior.

Para mover datos existentes a otro disco:

```bash
sudo systemctl stop docker
sudo rsync -aHAX /var/lib/docker/ /srv/docker/
# luego "data-root": "/srv/docker" en daemon.json
sudo systemctl start docker
docker info | grep "Docker Root Dir"
```

---

## Los volúmenes y la persistencia

Tres formas de hacer persistir datos, que no deben confundirse:

| Tipo | Sintaxis | Uso |
|------|---------|-------|
| **Volumen nombrado** | `-v pgdata:/var/lib/postgresql/data` | Datos aplicativos — **la opción por defecto** |
| **Montaje de enlace** | `-v /srv/conf:/etc/app:ro` | Archivos de configuración, código en desarrollo |
| **tmpfs** | `--tmpfs /tmp` | Datos sensibles o temporales, solo en RAM |

```bash
docker volume create pgdata
docker volume ls
docker volume inspect pgdata
docker volume rm pgdata
docker volume prune            # elimina volúmenes no utilizados
```

Un volumen nombrado vive en `/var/lib/docker/volumes/<nombre>/_data`. Puede inspeccionarse desde el anfitrión — preferiblemente en lectura.

> [!TIP]
> Prefiera el **volumen nombrado** al montaje de enlace para datos: Docker gestiona permisos, portabilidad y limpieza. El montaje de enlace depende de los UID/GID del anfitrión — y un contenedor que se ejecuta con un UID diferente del propietario de la carpeta obtiene un `Permiso denegado` inmediato (ver el [capítulo 02](02_file_permissions_es.md)).

### Montar en lectura solo

```bash
docker run -d \
  -v /srv/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v webdata:/usr/share/nginx/html \
  nginx
```

El sufijo `:ro` es gratuito y evita que un contenedor comprometido reescriba su propia configuración.

---

## La red

```bash
docker network ls
```

```text
NETWORK ID     NAME      DRIVER    SCOPE
a1b2c3d4e5f6   bridge    bridge    local
b2c3d4e5f6a1   host      host      local
c3d4e5f6a1b2   none      null      local
```

| Controlador | Comportamiento |
|--------|--------------|
| `bridge` | Red virtual aislada, NAT hacia el exterior — modo por defecto |
| `host` | El contenedor comparte la pila de red del anfitrión, sin aislamiento |
| `none` | Sin red |
| `overlay` | Red multi-anfitrión (Swarm / Kubernetes) |
| `macvlan` | El contenedor obtiene su propia dirección MAC en la red física |

### Crear una red de aplicaciones

```bash
docker network create app-net
docker run -d --name db --network app-net postgres:16-alpine
docker run -d --name api --network app-net -p 3000:3000 mi-api
```

En una red **creada por el usuario** (a diferencia del `bridge` por defecto), Docker proporciona **resolución DNS interna**: `api` contacta a la base conectándose simplemente a `db:5432`. Exactamente lo que hace Compose automáticamente.

```bash
docker network inspect app-net      # ver contenedores conectados e IPs
docker network connect app-net web  # conectar un contenedor existente
docker network disconnect app-net web
```

### Puertos publicados y cortafuegos

> [!CAUTION]
> **Docker escribe sus propias reglas en `iptables` y cortocircuita `ufw`.** Un contenedor lanzado con `-p 5432:5432` es alcanzable desde el exterior **incluso si `ufw` bloquea el puerto 5432** — la regla Docker se evalúa primero. Muchas bases de datos terminan expuestas a Internet de esta forma.
>
> La solución: publicar solo en la interfaz de loopback.
>
> ```bash
> docker run -d -p 127.0.0.1:5432:5432 postgres:16-alpine
> ```
>
> Un servicio que no necesita ser alcanzado desde el anfitrión **no tiene razón alguna** de ser publicado: en una red Docker, los contenedores ya se hablan entre sí sin `-p`. Ver el [capítulo 05 · Seguridad](05_securite_es.md) para configuración del cortafuegos.

---

## Los logs

Por defecto, Docker escribe los logs de cada contenedor en un archivo JSON:

```text
/var/lib/docker/containers/<id>/<id>-json.log
```

**Sin rotación configurada, este archivo crece indefinidamente.** Esta es la primera causa de disco lleno en un anfitrión Docker.

### Configurar rotación globalmente

En `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

Sea 30 Mo de logs máximo por contenedor.

> [!IMPORTANT]
> Esta configuración solo se aplica a contenedores **creados después** del reinicio del demonio. Los contenedores existentes guardan su configuración: hay que recrearlos (`docker compose up -d --force-recreate`).

Por contenedor:

```bash
docker run -d --log-opt max-size=10m --log-opt max-file=3 nginx
```

En Compose:

```yaml
services:
  api:
    image: mi-api
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### Enviar logs a journald

Para centralizar en el `journalctl` del anfitrión, visto en el [capítulo 04](04_instalacion_y_servicios_es.md#journald):

```json
{ "log-driver": "journald" }
```

```bash
journalctl CONTAINER_NAME=api -f
```

> [!WARNING]
> Con el controlador `journald` (o `syslog`, o `fluentd`), el comando `docker logs` no funciona de la misma manera: hay que interrogar al sistema de destino.

### Encontrar qué está llenando el disco

```bash
sudo du -sh /var/lib/docker/containers/* | sort -rh | head
```

---

## Limitar recursos

Por defecto, **un contenedor puede consumir toda la RAM y todo el CPU del anfitrión**. Un proceso con fuga lleva el servidor entero con él.

```bash
docker run -d \
  --memory 512m \
  --memory-swap 512m \
  --cpus 1.5 \
  --pids-limit 200 \
  --restart unless-stopped \
  mi-api
```

| Opción | Efecto |
|--------|-------|
| `--memory 512m` | RAM máxima — más allá, el contenedor se mata (OOM) |
| `--memory-swap` | RAM + swap; igual a `--memory` para prohibir swap |
| `--cpus 1.5` | Equivalente a un núcleo y medio |
| `--cpuset-cpus 0,1` | Fija el contenedor en núcleos específicos |
| `--pids-limit 200` | Número máximo de procesos (protección anti *fork bomb*) |

En Compose:

```yaml
services:
  api:
    image: mi-api
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
```

### Las políticas de reinicio

| Política | Comportamiento |
|-----------|--------------|
| `no` | Por defecto — sin reinicio |
| `on-failure[:5]` | Reinicia si código de salida ≠ 0, máximo 5 veces |
| `always` | Siempre reinicia, incluso después de reboot del anfitrión |
| `unless-stopped` | Como `always`, pero respeta una detención manual — **la buena opción en producción** |

### El healthcheck

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s
```

```bash
docker ps --format '{{.Names}}\t{{.Status}}'
```

```text
api     Up 4 minutes (healthy)
db      Up 4 minutes (healthy)
```

> [!TIP]
> Un healthcheck solo tiene valor si algo lo mira: su supervisión, un proxy inverso, o un `depends_on: condition: service_healthy`. Solo, solo muestra un estado.

---

## Asegurar contenedores

### No ejecutar como root

Por defecto, el proceso de un contenedor se ejecuta como **root** — y el UID 0 en el contenedor es el UID 0 del anfitrión. En caso de escape, el daño es máximo.

En el `Dockerfile`:

```dockerfile
RUN addgroup -S app && adduser -S -G app app
USER app
```

O en la ejecución:

```bash
docker run -d --user 1000:1000 mi-api
```

### El refuerzo estándar

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --user 1000:1000 \
  mi-api
```

| Opción | Efecto |
|--------|-------|
| `--read-only` | Sistema de archivos raíz en lectura solo |
| `--tmpfs /tmp` | Un espacio escribible solo en RAM, ya que todo lo demás es de lectura solo |
| `--cap-drop ALL` | Retira **todas** las capabilities de Linux |
| `--cap-add NET_BIND_SERVICE` | …luego solo devuelve la que es necesaria (escuchar bajo puerto 1024) |
| `--security-opt no-new-privileges` | Prohíbe escalada vía binarios SUID |

Lo mismo en Compose:

```yaml
services:
  api:
    image: mi-api
    read_only: true
    tmpfs: [/tmp]
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    user: "1000:1000"
```

> [!CAUTION]
> **Nunca use `--privileged`.** Esta opción retira casi todo aislamiento: el contenedor obtiene todas las capabilities y acceso a periféricos del anfitrión. Es equivalente a un `chmod 777` en la seguridad de la máquina. Si una herramienta « lo necesita », en realidad necesita una o dos capabilities específicas — conceda esas.

### Los otros reflejos

- **Fije versiones** de imágenes, y privilegie variantes `-alpine` o `-slim` (menos código, menos CVE).
- **Escanee imágenes** antes de despliegue:

  ```bash
  docker scout cves mi-api:1.0
  # o
  trivy image mi-api:1.0
  ```

- **Reconstruya regularmente**: una imagen congelada acumula vulnerabilidades en sus dependencias de sistema.
- **Pase secretos en la ejecución**, nunca en la imagen (ver capítulo 07). En Compose:

  ```yaml
  services:
    api:
      secrets: [contraseña_bd]
  secrets:
    contraseña_bd:
      file: /srv/secretos/contraseña_bd.txt
  ```

- **AppArmor / SELinux** (capítulo 05) también se aplican a contenedores: Docker carga un perfil `docker-default` por defecto.

---

## El modo sin raíz

El demonio Docker se ejecuta como root. El modo **sin raíz** lo hace ejecutarse bajo un usuario no privilegiado, usando los *user namespaces*: root en el contenedor corresponde a un UID no privilegiado en el anfitrión.

```bash
sudo apt install -y uidmap docker-ce-rootless-extras
dockerd-rootless-setuptool.sh install
```

```bash
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
systemctl --user enable --now docker
sudo loginctl enable-linger $USER   # demonio sobrevive a desconexión
```

| Ventaja | Limitación |
|----------|--------|
| Un escape de contenedor no da root en el anfitrión | Sin escucha bajo puerto 1024 sin configuración |
| Cada usuario tiene su propio demonio aislado | Algunos controladores de red y almacenamiento indisponibles |
| Ya no se necesita grupo `docker` | Rendimiento de red ligeramente menor |

> [!TIP]
> Este es el modo por defecto a privilegiar en un servidor compartido o máquina de CI, donde múltiples usuarios lanzan contenedores. Para un servidor de aplicación dedicado, el demonio clásico adecuadamente reforzado sigue siendo común.

---

## Limpieza y espacio en disco

```bash
docker system df
```

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          24        4         8.2GB     6.9GB (84%)
Containers      6         4         120MB     40MB (33%)
Local Volumes   11        3         2.1GB     1.8GB (85%)
Build Cache     142       0         3.4GB     3.4GB
```

```bash
docker container prune          # contenedores detenidos
docker image prune              # imágenes sin etiqueta (dangling)
docker image prune -a           # todas imágenes no utilizadas
docker builder prune            # caché de construcción
docker volume prune             # volúmenes no utilizados
docker system prune -a          # todo lo anterior, excepto volúmenes
docker system prune -a --volumes # todo, volúmenes incluidos
```

> [!CAUTION]
> `docker system prune -a --volumes` elimina **volúmenes no conectados a un contenedor existente**. Una base de datos cuyo contenedor fue eliminado pero cuyo volumen debía conservarse desaparece definitivamente. En un servidor, limpie imágenes y caché de construcción; toque volúmenes a mano, con conocimiento de causa.

Limpieza automática, sin volúmenes, vía `cron` (capítulo 04):

```bash
# /etc/cron.d/docker-prune
0 4 * * 0 root /usr/bin/docker system prune -af --filter "until=168h" >> /var/log/docker-prune.log 2>&1
```

El filtro `until=168h` evita todo lo que tiene menos de 7 días.

---

## Copia de seguridad y restauración

**Lo que debe guardarse son los volúmenes y los archivos de configuración.** Las imágenes se reconstruyen, los contenedores se recrean; los datos, no.

### Guardar un volumen

```bash
docker run --rm \
  -v pgdata:/data:ro \
  -v "$(pwd)":/copia_seguridad \
  alpine tar czf /copia_seguridad/pgdata-$(date +%F).tar.gz -C /data .
```

Se lanza un contenedor desechable que monta el volumen en lectura solo y la carpeta actual en escritura, luego se archiva (capítulo 03).

### Restaurar

```bash
docker run --rm \
  -v pgdata:/data \
  -v "$(pwd)":/copia_seguridad \
  alpine sh -c "rm -rf /data/* && tar xzf /copia_seguridad/pgdata-2026-08-02.tar.gz -C /data"
```

### El volcado de aplicación

Para una base de datos, un volcado lógico a menudo es preferible a copiar archivos: es consistente, portable entre versiones y restaurable parcialmente.

```bash
docker compose exec -T db pg_dump -U postgres app | gzip > app-$(date +%F).sql.gz
```

### Exportar una imagen o contenedor

```bash
docker save mi-api:1.0 | gzip > mi-api-1.0.tar.gz   # una imagen, con capas
docker load < mi-api-1.0.tar.gz                      # reimportarla

docker export web > web-fs.tar    # sistema archivos contenedor, aplanado
```

> [!IMPORTANT]
> `save` / `load` trabajan en **imágenes** (con historial). `export` / `import` aplanan un **contenedor** en un sistema de archivos sin historial ni metadatos. Para transferir una aplicación, es `save` lo que necesita.

---

## Docker en producción

### El registro de imágenes

```bash
docker login registro.ejemplo.be
docker tag mi-api:1.0 registro.ejemplo.be/equipo/mi-api:1.0
docker push registro.ejemplo.be/equipo/mi-api:1.0
docker pull registro.ejemplo.be/equipo/mi-api:1.0
```

> [!TIP]
> En un servidor, no construya imágenes: su CI las construye, las escanea, las empotra en el registro; el servidor solo hace `pull` una etiqueta inmutable. La construcción necesita herramientas, fuentes y secretos — cosas que no tienen lugar en producción.

### Lanzar una stack Compose al iniciar

Compose no gestiona el inicio al iniciar el sistema. Dos enfoques:

**1. `restart: unless-stopped`** en cada servicio — el demonio Docker (activado vía `systemctl enable docker`) relanza los contenedores al iniciar. Simple y suficiente en la mayoría de casos.

**2. Una unidad systemd** dedicada, cuando se quiere dominar el orden y dependencias (capítulo 04):

```ini
# /etc/systemd/system/mi-app.service
[Unit]
Description=Stack aplicativa mi-app
Requires=docker.service
After=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/srv/mi-app
ExecStart=/usr/bin/docker compose up -d --remove-orphans
ExecStop=/usr/bin/docker compose down
ExecReload=/usr/bin/docker compose up -d --remove-orphans
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mi-app
```

### Desplegar una nueva versión

```bash
cd /srv/mi-app
docker compose pull            # obtener nuevas imágenes
docker compose up -d            # recrear solo servicios modificados
docker image prune -f           # limpiar imágenes antiguas
```

> [!WARNING]
> `docker compose up -d` recrea contenedores cuya configuración o imagen cambió: hay un **corte de servicio** de algunos segundos. Para cero tiempo inactivo, necesita un proxy inverso y despliegue azul/verde — u orquestador (Swarm, Nomad, Kubernetes).

### Archivos Compose superpuestos

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

El segundo archivo **anula** el primero: se mantiene una base común y solo se expresan diferencias de producción (sin montaje del código, límites de recursos, reinicio automático, puertos en `127.0.0.1`).

Un ejemplo completo de ambos archivos se encuentra en [`07_docker/05_stack_compose/`](07_docker/05_stack_compose/) — compare `docker-compose.yml` y [`docker-compose.prod.yml`](07_docker/05_stack_compose/docker-compose.prod.yml).

> [!CAUTION]
> **Las listas se concatenan, no se reemplazan.** Redeclarar `ports:` en archivo de superposición **añade** la entrada a la del archivo base, y escribir `ports: []` no elimina nada. La trampa es seria: se cree haber restringido la base a `127.0.0.1`, y permanece publicada en `0.0.0.0` — o el contenedor rechaza iniciar por conflicto de puerto.
>
> Desde Compose 2.24, dos marcadores resuelven el problema:
>
> ```yaml
> services:
>   api:
>     ports: !override        # reemplaza lista heredada
>       - "127.0.0.1:3000:3000"
>     volumes: !reset []      # borra lista heredada
> ```

**Siempre verifique el resultado de fusión antes desplegar:**

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml config
```

```bash
# Pregunta que importa: ¿qué está realmente publicado?
docker compose -f docker-compose.yml -f docker-compose.prod.yml ps --format '{{.Service}} {{.Ports}}'
```

```text
api    127.0.0.1:3000->3000/tcp
cache  6379/tcp
db     5432/tcp
```

Aquí solo la API es alcanzable desde anfitrión, y solo localmente: `cache` y `db` solo existen en la red Docker.

---

## Supervisión y diagnóstico

```bash
docker stats                          # CPU / RAM / red / I/O en directo
docker stats --no-stream              # instantánea, explorable en script
docker events                         # flujo de eventos del demonio
docker ps --filter "health=unhealthy" # contenedores en mala salud
docker inspect --format '{{.State.ExitCode}}' api
```

Un pequeño script de control, en espíritu del [capítulo 06](06_shell_script/01_Base/base_es.md):

```bash
#!/usr/bin/env bash
set -euo pipefail

# Señala cualquier contenedor detenido o en mala salud
mapfile -t down < <(docker ps -a --filter "status=exited" --format '{{.Names}}')
mapfile -t sick < <(docker ps --filter "health=unhealthy" --format '{{.Names}}')

if (( ${#down[@]} + ${#sick[@]} > 0 )); then
    printf 'Contenedores detenidos: %s\n' "${down[*]:-ninguno}"
    printf 'Contenedores enfermos: %s\n' "${sick[*]:-ninguno}"
    exit 1
fi

echo "Todos contenedores operacionales."
```

### Las fallas clásicas

| Síntoma | Pista |
|----------|-------|
| Contenedor reinicia continuamente | `docker logs --tail 100 nombre` luego `docker inspect --format '{{.State.ExitCode}}' nombre` |
| Código salida **137** | Matado por SIGKILL — casi siempre OOM killer: `--memory` muy bajo o fuga memoria |
| Código salida **139** | Segfault en aplicación |
| `no space left on device` | `docker system df`, luego prune imágenes y caché |
| `port is already allocated` | `ss -tulpn \| grep :8080` — otro servicio ocupa puerto |
| Contenedor no resuelve ningún nombre | DNS demonio: verificar `dns` en `daemon.json` |
| Lentitud general anfitrión | `docker stats` — contenedor sin límite de recursos |

> [!TIP]
> Para depurar contenedor que se detiene inmediatamente, cortocircuite comando inicio:
>
> ```bash
> docker run -it --rm --entrypoint sh mi-api:1.0
> ```
>
> Obtiene shell en la imagen, con exactamente su contenido, y puede lanzar comando a mano para ver error.

---

## Resumen

| Comando | Rol |
|----------|------|
| `systemctl status docker` / `journalctl -u docker` | Estado y logs demonio |
| `docker info` / `docker system df` | Configuración / ocupación disco |
| `docker volume ls` / `inspect` / `prune` | Gestionar volúmenes |
| `docker network create` / `inspect` | Gestionar redes |
| `docker run --memory --cpus --pids-limit` | Limitar recursos |
| `docker run --cap-drop ALL --read-only --user` | Reforzar contenedor |
| `docker system prune -af --filter "until=168h"` | Limpieza periódica |
| `docker save` / `load` | Exportar / importar imagen |
| `docker stats` / `docker events` | Supervisión tiempo real |
| `docker compose pull && docker compose up -d` | Desplegar nueva versión |

### Los archivos a conocer

| Ruta | Contenido |
|--------|---------|
| `/etc/docker/daemon.json` | Configuración demonio |
| `/var/lib/docker/` | Imágenes, contenedores, volúmenes |
| `/var/lib/docker/volumes/<nombre>/_data` | Datos volumen nombrado |
| `/var/run/docker.sock` | Socket API — proteger |

### Las siete reglas a recordar

> [!TIP]
> 1. **Configurar rotación logs** desde instalación — sino disco se llena.
> 2. **Limitar memoria, CPU y PID** de cada contenedor.
> 3. **Publicar en `127.0.0.1`** todo lo que no tiene que exponerse — Docker cortocircuita `ufw`.
> 4. **Nunca `--privileged`**, nunca socket Docker montada sin razón.
> 5. **Nunca ejecutar como root** en contenedor.
> 6. **Guardar volúmenes**, no contenedores.
> 7. **Prune regular imágenes y caché**, nunca volúmenes en automático.

---

⬅️ [Anterior: 07 · Docker para los desarrolladores](07_docker_es.md) · 🏠 [Tabla de contenidos](README_es.md) · [Siguiente: 09 · tmux ➡️](09_tmux_es.md)

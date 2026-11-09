# Laboratorio #2 – WordPress con balanceo de carga, Redis y tolerancia a fallos

**Curso:** Sistemas Operativos Avanzado
**Estudiante(s):** _Nombre(s) aquí_
**Fecha:** _dd/mm/aaaa_
**Plataforma:** Ubuntu 24.04 (WSL) · Docker Engine · Docker Compose v2

> Enunciado original: [`docs/Taller_Grupal_laboratorio_2_wordpress_ha.md`](docs/Taller_Grupal_laboratorio_2_wordpress_ha.md)

---

## 1. Objetivo

Desplegar un sitio **WordPress** con varios contenedores que cooperan en una red Docker, y comprobar:

- Balanceo de carga entre dos servidores web (`web1` y `web2`) con **HAProxy**.
- Manejo de **UID/GID** entre el host y los contenedores mediante un `entrypoint` dinámico.
- **Redis** como caché de objetos (y sesiones PHP de plugins).
- Continuidad del servicio y de la autenticación cuando una réplica web falla.
- Los puntos únicos de fallo que aún quedan en la solución.

## 2. Arquitectura

```mermaid
flowchart LR
    U[Navegador] -->|localhost:803| H[HAProxy]
    H -->|roundrobin| W1[web1 · Apache + PHP 8.3]
    H -->|roundrobin| W2[web2 · Apache + PHP 8.3]
    W1 --> DB[(MariaDB)]
    W2 --> DB
    W1 --> R[(Redis)]
    W2 --> R
    W1 --> F[/home/orh<br/>archivos WordPress]
    W2 --> F
    H -.-> S[localhost:8404/stats]
```

| Servicio | Imagen | Función | Puerto host |
|----------|--------|---------|-------------|
| `haproxy` | `haproxy:3.2-alpine` | Balanceador + página de estadísticas | `803`, `8404` |
| `web1`, `web2` | build local (`php:8.3-apache-bookworm`) | Apache + PHP con WordPress | — (interno) |
| `mariadb` | `mariadb:11.4` | Base de datos (volumen persistente) | — (interno) |
| `redis` | `redis:7.4-alpine` | Caché de objetos y sesiones | — (interno) |

**Estado compartido:** datos de WordPress en MariaDB · archivos en `/home/orh` (bind mount) · caché en Redis DB 0 · sesiones PHP en Redis DB 1 · `wp-config.php` y salts idénticos para ambas réplicas.

## 3. Estructura del repositorio

```
.
├── README.md
├── compose.yaml                  # Servicios, red y volúmenes
├── .env                          # WORDPRESS_UID / WORDPRESS_GID
├── docs/                         # Enunciado del laboratorio
├── haproxy/haproxy.cfg           # roundrobin + health checks
├── web/
│   ├── Dockerfile                # Imagen genérica Apache/PHP 8.3
│   ├── docker-entrypoint.sh      # Ajusta www-data al UID/GID recibido
│   └── redis-session.ini         # Sesiones PHP en Redis
└── wordpress/
    ├── wp-config-snippet.php     # Líneas agregadas a wp-config.php
    └── mu-plugins/backend-indicator.php   # Muestra qué backend respondió
```

---

## 4. Paso a paso

### Paso 1 – Verificar prerrequisitos

```bash
docker --version
docker compose version
docker info > /dev/null && echo "Docker OK"

getent passwd orh
id orh                                   # uid=1003(orh) gid=1003(orh)
sudo stat -c '%U:%G %u:%g %a %n' /home/orh
```

Se confirma que el usuario `orh` existe con UID/GID `1003:1003`, y que será el dueño de los archivos del sitio. El usuario que administra Docker/Compose es distinto, lo que separa las responsabilidades de **aplicación** e **infraestructura**.

### Paso 2 – Preparar el proyecto

```bash
git clone https://github.com/TU_USUARIO/TU_REPO.git
cd TU_REPO
chmod 0755 web/docker-entrypoint.sh
chmod 600 .env
cat .env
```

`.env` entrega la identidad al contenedor en tiempo de ejecución:

```dotenv
WORDPRESS_UID=1003
WORDPRESS_GID=1003
```

### Paso 3 – Entender la imagen web genérica

- **`web/Dockerfile`**: parte de `php:8.3-apache-bookworm`, instala extensiones (`gd`, `intl`, `mysqli`, `pdo_mysql`, `zip`, `opcache`) y **phpredis**, habilita `mod_rewrite` y copia el `entrypoint`. **No** contiene ningún UID/GID fijo.
- **`web/docker-entrypoint.sh`**: al arrancar, valida `APP_UID`/`APP_GID`, detecta colisiones y ejecuta `usermod`/`groupmod` sobre `www-data` para que coincida con el propietario del bind mount. Luego inicia Apache. Es idempotente.
- **`web/redis-session.ini`**: configura `session.save_handler = redis` apuntando a `tcp://redis:6379?database=1`.

> Idea clave: Linux evalúa permisos por **número** (UID/GID), no por nombre. Si dentro del contenedor `www-data` pasa a ser 1003, el host ve esos archivos como `orh:orh`.

### Paso 4 – Entender HAProxy

`haproxy/haproxy.cfg` define:

- `balance roundrobin` entre `web1:80` y `web2:80`.
- Health check HTTP a `/wp-login.php` con `inter 1s fall 1 rise 2`: un servidor se saca de rotación al primer fallo y se reincorpora tras 2 éxitos.
- Header `X-Backend` para identificar qué servidor respondió.
- Estadísticas en el puerto `8404`.
- **Sin** persistencia (`cookie`, `balance source`, `stick-table`): cualquier réplica puede atender a cualquier cliente.

### Paso 5 – Descargar WordPress como `orh`

```bash
sudo chown -R orh:orh /home/orh
sudo chmod 750 /home/orh

sudo -u orh bash -c '
  curl -fsSL https://wordpress.org/latest.tar.gz |
  tar -xz --strip-components=1 -C /home/orh
'

sudo find /home/orh -type d -exec chmod 750 {} \;
sudo find /home/orh -type f -exec chmod 640 {} \;

sudo stat -c '%U:%G %u:%g %a %n' /home/orh/wp-login.php
```

Resultado esperado:

```text
orh:orh 1003:1003 640 /home/orh/wp-login.php
```

### Paso 6 – Validar configuración antes de desplegar

```bash
docker compose config | grep -E 'APP_UID|APP_GID|BACKEND_NAME'

docker run --rm \
  -v "$PWD/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro" \
  haproxy:3.2-alpine \
  haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
```

Resultado esperado: `APP_UID: "1003"`, `APP_GID: "1003"` para ambos backends y `Configuration file is valid`.

### Paso 7 – Construir y levantar

```bash
docker compose build --no-cache web1 web2
docker compose up -d
docker compose ps
```

Resultado esperado tras la inicialización:

```text
mariadb    running (healthy)
redis      running (healthy)
web1       running (healthy)
web2       running (healthy)
haproxy    running
```

Los `healthcheck` y `depends_on: condition: service_healthy` garantizan que los web arranquen cuando MariaDB y Redis ya están listos.

### Paso 8 – Verificar identidades (UID/GID)

```bash
docker compose exec web1 id www-data
docker compose exec web2 id www-data
docker compose logs web1 web2 | grep 'Iniciando Apache'
docker compose top web1
```

Resultado esperado:

```text
uid=1003(www-data) gid=1003(www-data) groups=1003(www-data)
Iniciando Apache: www-data usa UID=1003 GID=1003
```

El proceso maestro de Apache corre como `root`; los workers que atienden solicitudes corren como `1003:1003`.

### Paso 9 – Verificar escritura en el bind mount

```bash
docker compose exec web1 sh -c 'touch /var/www/html/prueba-web1.txt'
docker compose exec web2 sh -c 'touch /var/www/html/prueba-web2.txt'

ls -ln /home/orh/prueba-web*.txt     # numérico: 1003 1003
ls -l  /home/orh/prueba-web*.txt     # nombres:  orh orh

sudo -u orh rm /home/orh/prueba-web1.txt /home/orh/prueba-web2.txt
```

Los archivos creados por ambos contenedores aparecen en el host como `orh:orh`.

### Paso 10 – Instalar WordPress

Abrir `http://localhost:803` y usar:

| Campo | Valor |
|-------|-------|
| Base de datos | `mi_base_de_datos` |
| Usuario | `usuario` |
| Contraseña | `password` |
| Servidor | `mariadb` (no `localhost`) |
| Prefijo | `wp_` |

Luego se protege el archivo de configuración:

```bash
sudo chown orh:orh /home/orh/wp-config.php
sudo chmod 600 /home/orh/wp-config.php
```

### Paso 11 – Configurar Redis en WordPress

```bash
sudo -u orh nano /home/orh/wp-config.php
```

Se agregó el contenido de [`wordpress/wp-config-snippet.php`](wordpress/wp-config-snippet.php) antes de `/* That's all, stop editing! */` (URL pública `http://localhost:803`, host `redis`, cliente `phpredis`, DB 0, `FS_METHOD direct`).

Después, en WordPress: **Plugins → Añadir → Redis Object Cache → Instalar y activar → Ajustes → Redis → Enable Object Cache**. Estado esperado: `Connected`, cliente `PhpRedis`, host `redis`.

Verificaciones por consola:

```bash
docker compose exec web1 php -m | grep -E 'mysqli|pdo_mysql|redis'
docker compose exec web2 php -m | grep -E 'mysqli|pdo_mysql|redis'

docker compose exec web1 php -r '$r=new Redis();$r->connect("redis",6379);var_dump($r->ping());'
docker compose exec redis redis-cli -n 0 DBSIZE
```

### Paso 12 – Indicador visible del backend

Se instaló un *must-use plugin* ([`wordpress/mu-plugins/backend-indicator.php`](wordpress/mu-plugins/backend-indicator.php)) que muestra `Backend: web1/web2` en la barra de administración y envía el header `X-WordPress-Backend`.

```bash
sudo install -d -o orh -g orh -m 0750 /home/orh/wp-content/mu-plugins
sudo install -o orh -g orh -m 0640 \
  wordpress/mu-plugins/backend-indicator.php \
  /home/orh/wp-content/mu-plugins/backend-indicator.php
```

### Paso 13 – Probar el balanceo de carga

```bash
for i in $(seq 1 10); do
  printf "Solicitud %02d: " "$i"
  curl -sI -H 'Connection: close' "http://localhost:803/?probe=$i" \
    | awk -F': ' 'tolower($1)=="x-backend"{gsub("\r","",$2);print $2}'
done
```

Resultado esperado (alternancia `roundrobin`):

```text
Solicitud 01: web1
Solicitud 02: web2
Solicitud 03: web1
Solicitud 04: web2
...
```

También se revisó el panel `http://localhost:8404/stats`, donde `web1` y `web2` aparecen en `UP`.

> Una recarga del navegador no siempre alterna visualmente porque una página genera varias solicitudes (CSS, JS, imágenes) y el navegador reutiliza conexiones. Por eso se usa `curl` con `Connection: close`.

### Paso 14 – Prueba de fallo de `web1`

1. Iniciar sesión en `http://localhost:803/wp-admin` y dejarla abierta.
2. Detener la réplica:

   ```bash
   docker compose stop web1
   docker compose ps
   curl -sI http://localhost:803/ | grep -i x-backend    # X-Backend: web2
   ```

3. En `/stats`: `web1: DOWN`, `web2: UP`.
4. Recargar WordPress: el sitio sigue disponible y **la sesión no se pierde**.
5. Restaurar:

   ```bash
   docker compose start web1
   docker compose ps
   ```

HAProxy reincorpora a `web1` tras 2 comprobaciones exitosas (`rise 2`).

### Paso 15 – Prueba de fallo de `web2`

```bash
docker compose stop web2
curl -sI http://localhost:803/ | grep -i x-backend    # X-Backend: web1
docker compose start web2
```

La autenticación continúa porque ambas réplicas comparten código, `wp-config.php` (salts), base de datos y URL pública.

### Paso 16 – Verificar propiedad de archivos tras usar WordPress

Después de instalar un plugin o subir una imagen:

```bash
sudo find /home/orh -maxdepth 4 -printf '%u:%g %m %p\n' | head -20

# Archivos que NO sean 1003:1003 (sin salida = todo consistente)
sudo find /home/orh \( ! -uid 1003 -o ! -gid 1003 \) -printf '%u:%g %m %p\n'
```

### Paso 17 – Limpieza

```bash
docker compose down        # elimina contenedores y red (conserva datos)
docker compose down -v     # además elimina volúmenes de MariaDB y Redis
```

---

## 5. Análisis

**¿Balanceo de carga o alta disponibilidad?** El laboratorio demuestra balanceo de carga y tolerancia al fallo de **una réplica web**. No es alta disponibilidad integral: siguen siendo puntos únicos de fallo el host/WSL, Docker Engine, HAProxy, MariaDB, Redis y el sistema de archivos `/home/orh`.

**Por qué importa el UID/GID.** Linux aplica permisos por número. Si `web1` usara UID 33 y `web2` UID 1003, solo una de las dos podría escribir en `/home/orh`. Ajustar la identidad en el `entrypoint` (en vez de `ARG` en el build) permite reutilizar la misma imagen para sitios con distintos propietarios, solo cambiando variables de entorno.

**Sesión de WordPress vs. Redis.** El inicio de sesión de WordPress no depende de `$_SESSION`: se valida con la cookie, los salts de `wp-config.php` y la base de datos. Redis Object Cache solo acelera el acceso a objetos; las sesiones PHP en Redis (DB 1) cubren plugins que usan `session_start()`.

**Si algo falla:**

| Componente | Efecto |
|------------|--------|
| `web1` o `web2` | HAProxy lo saca de rotación y el sitio sigue (solicitudes en vuelo podrían fallar) |
| MariaDB | El sitio deja de funcionar (punto único de fallo) |
| Redis | WordPress sigue, pero sin caché; plugins con sesiones PHP pueden verse afectados |
| HAProxy | Sin acceso al sitio (punto único de fallo) |

**Si las réplicas estuvieran en hosts distintos**, un bind mount local ya no compartiría archivos: haría falta almacenamiento compartido o replicado, balanceadores redundantes con IP virtual, MariaDB Galera, Redis Sentinel/Cluster y un orquestador (Swarm/Kubernetes).

**Seguridad.** Las credenciales de este repositorio son solo de laboratorio; en producción se usarían secretos. La página `/stats` no tiene autenticación y un usuario del grupo `docker` equivale a `root` en el host.

## 6. Conclusiones

- `web1` y `web2` ejecutan la misma imagen genérica; el UID/GID se aplica al iniciar el contenedor.
- Los archivos creados desde WordPress pertenecen a `orh:orh` (`1003:1003`).
- HAProxy distribuye solicitudes con `roundrobin` y, mediante health checks, retira réplicas caídas.
- Un usuario autenticado puede ser atendido por cualquier réplica sin perder la sesión.
- Docker Compose permite declarar toda la plataforma (BD, caché, web y balanceador) de forma reproducible.
- La solución sigue teniendo varios puntos únicos de fallo, por lo que no equivale a una plataforma de alta disponibilidad completa.

## 7. Referencias

- Docker Compose: https://docs.docker.com/reference/compose-file/services/
- Orden de arranque en Compose: https://docs.docker.com/compose/how-tos/startup-order/
- WordPress – Redis Object Cache: https://wordpress.org/plugins/redis-cache/
- HAProxy – Backends y roundrobin: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/configuration-basics/backends/
- HAProxy – Health checks: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/reliability/health-checks/
- Stallings, W. *Operating Systems: Internals and Design Principles*, 9.ª ed.

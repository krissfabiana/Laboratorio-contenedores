# Laboratorio #1 – Contenedores con Docker, Docker Compose y HAProxy

**Curso:** BISOF 18 – Sistemas Operativos II
**Universidad:** Universidad Latina de Costa Rica
**Estudiante(s):** _Nombre(s) aquí_
**Fecha:** _dd/mm/aaaa_

> Enunciado original: [`docs/Taller_Grupal_laboratorio_1.pdf`](docs/Taller_Grupal_laboratorio_1.pdf)

---

## 1. Objetivo

Instalar Docker en Ubuntu, construir imágenes y desplegar una arquitectura web con **balanceo de carga** usando Docker Compose, y practicar la administración básica de contenedores.

## 2. Arquitectura

```
                    ┌──────────────┐
 Navegador / curl ─►│   HAProxy    │  puerto host 80 → 80
                    │ (roundrobin) │
                    └───┬──────┬───┘
                        │      │         red Docker: webnet (bridge)
                 ┌──────▼─┐  ┌─▼──────┐
                 │  web1  │  │  web2  │   Apache + PHP 7.4
                 └──────┬─┘  └─┬──────┘
                        └──┬───┘
                      ┌────▼─────┐
                      │ MariaDB  │   3306 (solo red interna)
                      └──────────┘
```

| Servicio | Imagen | Rol | Puerto |
|----------|--------|-----|--------|
| `haproxy` | `haproxy:latest` | Balanceador HTTP (roundrobin) | `80:80` (host) |
| `web1` / `web2` | build local (`php:7.4-apache`) | Servidores web idénticos | 80 (interno) |
| `mariadb` | `mariadb:latest` | Base de datos | 3306 (interno) |

## 3. Estructura del repositorio

```
.
├── README.md
├── docs/                         # Enunciado del laboratorio (PDF)
└── proyecto/
    ├── docker-compose.yml        # Define servicios y red
    ├── init-db.sql               # Script de inicialización de la BD
    ├── haproxy/haproxy.cfg       # Configuración del balanceador
    ├── web1/{Dockerfile,index.php}
    └── web2/{Dockerfile,index.php}
```

## 4. Paso a paso

### Paso 1 – Instalar Docker en Ubuntu

Siguiendo la [guía oficial](https://docs.docker.com/engine/install/ubuntu/) (repositorio `apt`):

```bash
# Llave GPG oficial de Docker
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Repositorio en las fuentes de apt
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

# Paquetes de Docker
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Usar Docker sin `sudo`** (post-instalación):

```bash
sudo groupadd docker          # puede indicar que ya existe
sudo usermod -aG docker $USER
newgrp docker
```

**Verificación:**

```bash
docker --version
docker compose version
docker run hello-world
```

---

### Paso 2 – Obtener y entender los archivos

Se descomprimió `grafana.zip` (compartido en Teams) y se trabajó en la carpeta `proyecto/`:

```bash
unzip grafana.zip
cd grafana/proyecto
ls -R
```

Resumen de lo que hace cada archivo:

- **`docker-compose.yml`**: declara 4 servicios (`mariadb`, `web1`, `web2`, `haproxy`) y la red `webnet` (bridge). Solo HAProxy publica un puerto al host; el resto se comunica por la red interna usando el nombre del servicio como DNS.
- **`web1/Dockerfile` y `web2/Dockerfile`**: parten de `php:7.4-apache`, instalan `pdo` y `pdo_mysql`, copian `index.php` y exponen el puerto 80.
- **`index.php`**: se conecta con PDO al host `mariadb`, ejecuta `SELECT * FROM usuarios` e imprime `SERVER_ADDR` (la IP del contenedor que atendió la petición), útil para comprobar el balanceo.
- **`haproxy/haproxy.cfg`**: frontend en `*:80` y backend `http_back` con `balance roundrobin` hacia `web1:80` y `web2:80`, ambos con `check` (health check).
- **`init-db.sql`**: crea la base `mi_base_de_datos` y una tabla con un registro de ejemplo.

---

### Paso 3 – Construir y desplegar

```bash
cd grafana/proyecto
docker compose up --build -d
```

>  Si el puerto 80 del host está ocupado, cambiar en `docker-compose.yml` la línea `"80:80"` por `"90:80"` y acceder por `http://localhost:90`.

Esto descarga `mariadb` y `haproxy`, construye `proyecto-web1` y `proyecto-web2`, crea la red `proyecto_webnet` y levanta los 4 contenedores.

---

### Paso 4 – Verificar el funcionamiento

```bash
docker ps
curl http://localhost:80
curl -v http://localhost:80
```

La petición pasa por HAProxy, Apache/PHP responde y se muestra el mensaje de conexión junto con `SERVER_ADDR`. Al repetir `curl` varias veces, el `SERVER_ADDR` **alterna entre las IPs de web1 y web2**, lo que confirma el balanceo roundrobin:

```bash
for i in $(seq 1 6); do curl -s http://localhost:80 | grep -o "SERVER_ADDR: [0-9.]*"; done
```

#### Observación sobre el error de la tabla

Al ejecutar `curl` aparece:

```
 Conexión exitosa a la base de datos
 Error de conexión o consulta: SQLSTATE[42S02]: Base table or view not found: 1146
Table 'mi_base_de_datos.usuarios' doesn't exist
```

Esto es **esperado** y se debe a dos cosas identificadas al analizar los archivos:

1. En `docker-compose.yml` el volumen que monta `init-db.sql` en MariaDB está **comentado**, por lo que el script nunca se ejecuta.
2. Aunque se ejecutara, `init-db.sql` crea la tabla `mensajes`, mientras que `index.php` consulta `usuarios`.

La conexión a MariaDB **sí funciona** (el mensaje lo confirma); lo que falta es la tabla. Para completar la prueba end-to-end se puede crear manualmente (ver Paso 7, opcional).

---

### Paso 5 – Administración básica de contenedores

| Tarea | Comando |
|-------|---------|
| Ver contenedores activos | `docker ps` |
| Ver todos (incluso detenidos) | `docker ps -a` |
| Filtrar por nombre | `docker ps --filter "name=web"` |
| Detalle completo (JSON) | `docker inspect haproxy` |
| Puertos publicados | `docker port haproxy` |
| Procesos dentro del contenedor | `docker top web1` |
| Uso de recursos | `docker stats --no-stream` |
| Logs | `docker logs haproxy` / `docker logs --tail 50 web2` |
| Shell interactiva | `docker exec -it web1 bash` |
| Comando puntual | `docker exec web1 php -v` |
| Ver archivos de la app | `docker exec web1 ls -la /var/www/html` |

---

### Paso 6 – Red, DNS y volúmenes

```bash
# Redes
docker network ls
docker network inspect proyecto_webnet

# Resolución DNS interna entre contenedores
docker exec web1 getent hosts mariadb
docker exec haproxy getent hosts web1
docker exec haproxy getent hosts web2

# Bind mount de HAProxy
docker exec haproxy cat /usr/local/etc/haproxy/haproxy.cfg
docker volume ls
```

Hallazgos:

- La red `proyecto_webnet` (bridge) contiene los 4 contenedores, cada uno con su IP interna.
- Docker resuelve automáticamente los nombres de servicio (`mariadb`, `web1`, `web2`).
- `haproxy.cfg` está montado como **bind mount** desde el host; no se definió un volumen nombrado para MariaDB, por lo que los datos se pierden al eliminar el contenedor.

---

### Paso 7 – (Opcional) Crear la tabla y completar la prueba

```bash
docker exec -it mariadb mysql -uusuario -ppassword mi_base_de_datos -e "
CREATE TABLE usuarios (id INT AUTO_INCREMENT PRIMARY KEY, nombre VARCHAR(100));
INSERT INTO usuarios (nombre) VALUES ('Ana'), ('Luis');"

curl http://localhost:80
```

Ahora la página muestra los usuarios y el `SERVER_ADDR`.

---

### Paso 8 – Ciclo de vida con Docker Compose

```bash
docker compose ps
docker compose logs --tail 20
docker compose restart haproxy      # necesario tras editar haproxy.cfg
docker compose exec web1 bash
docker stop web2 && curl http://localhost:80   # HAProxy envía todo a web1 (health check)
docker start web2
```

Al detener `web2`, HAProxy lo marca como caído gracias a `check` y todas las peticiones se atienden desde `web1`.

### Paso 9 – Limpieza

```bash
docker compose down      # detiene y elimina contenedores y red
docker compose down -v   # además elimina volúmenes (si existieran)
```

---

## 5. Comparación de tecnologías (resumen)

| Tecnología | Tipo | Cuándo usarla |
|------------|------|---------------|
| **LXC/LXD** | Contenedor de sistema (similar a mini-VM) | Laboratorios de "mini servidores" Linux |
| **Docker** | Contenedor de aplicación | Desarrollo, CI/CD, despliegue simple |
| **Podman** | Contenedor de aplicación, sin daemon y rootless | Entornos con foco en seguridad |
| **containerd / CRI-O** | Runtimes para Kubernetes | Nodos de Kubernetes |
| **runc** | Runtime OCI de bajo nivel | Base del stack de contenedores |

## 6. Conclusiones

- Los contenedores comparten el kernel del host, por lo que son más ligeros que las máquinas virtuales.
- Docker Compose permite definir y levantar toda la arquitectura (BD, web y balanceador) con un solo comando.
- HAProxy distribuye la carga con roundrobin y detecta servidores caídos mediante health checks.
- Docker provee DNS interno por nombre de servicio, por lo que los contenedores no necesitan IPs fijas.
- Los datos dentro de un contenedor son efímeros; para persistirlos se requieren volúmenes.
- Credenciales como `rootpassword` son solo para laboratorio; en producción se deben usar secretos.

## 7. Referencias

- Instalación de Docker en Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- HAProxy: https://www.haproxy.org/ · https://hub.docker.com/_/haproxy/
- PHP (imagen oficial): https://hub.docker.com/_/php/

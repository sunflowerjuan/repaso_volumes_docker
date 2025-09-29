# GUIA DE ESTUDIO PARCIAL

## DOCKER COMPOSE

Herramienta para definir y ejecutar aplicaciones multicontenedor usando un solo archivo (docker-compose.yml).
Permite: - Declarar servicios (containers). - Configurar redes y volúmenes. - Automatizar la construcción y ejecución.

### COMANDOS

1. Levantar y reconstruir
   `docker compose up --build -d`
2. Levantar con .env personalizado
   `docker compose --env-file custom.env up`
3. Levantar, reconstruir y .env personalizado
   `docker compose --env-file custom.env up --build -d`

### SINTAXIS

#### SERVICIOS

```bash
services:        # Sección principal, define los contenedores de la aplicación
  nombre_servicio:  # Identificador del servicio (NO es el nombre real del contenedor, es un alias dentro de Compose)
    image: ...      # Imagen a usar (de Docker Hub o personalizada)
    build: ...      # Instrucciones para construir una imagen (puede ser ruta, contexto y Dockerfile)
    container_name: ... # (opcional) Nombre fijo del contenedor en vez del generado automáticamente
    environment:    # Variables de entorno que se inyectan al contenedor
      - CLAVE=valor
    env_file:       # Permite cargar variables desde un archivo .env
      - .env
    ports:          # Mapeo de puertos entre host y contenedor
      - "8080:80"   # Formato: host:contenedor
    volumes:        # Montajes de volúmenes o carpetas locales dentro del contenedor
      - ./src:/app  # bind mount (carpeta local)
      - datos:/var/lib/db  # volumen nombrado
    networks:       # Redes a las que se conecta este servicio
      - red_interna
    depends_on:     # Controla el orden de arranque (este servicio espera a que arranquen otros)
      - db
    restart: ...    # Política de reinicio (no, always, unless-stopped, on-failure)
    command: ...    # Sobrescribe el CMD de la imagen
    entrypoint: ... # Sobrescribe el ENTRYPOINT de la imagen
    deploy:         # Configuración de despliegue (usada con Swarm/Kubernetes)
      replicas: ... # Número de instancias del servicio
      resources:    # Límites y reservas de recursos (CPU, memoria)
```

#### REDES

Tipos de redes

- bridge (default): aislada dentro del host.

- host: comparte red del host (Linux).

- none: sin red.

- overlay: para clúster (Swarm).

**declarar redes**:

```bash
networks:
  red_interna:
    driver: bridge
  red_externa:
    external: true
```

Asignacion a servicios:

```bash
services:
  service_1:
    image: myapp
    networks:
      - red_interna
  service_2:
    image: postgres
    networks:
      - red_interna
```

#### VOLUMENES

1. Volúmenes gestionados por Docker

   ```bash
   volumes:
   datos_db:
   services:
   db:
       volumes:
       - datos_db:/var/lib/postgresql/data
   ```

2. Externos

   ```bash
    volumes:
    datos_externos:
    external: true
   ```

3. Bind Mounts

#### EJEMPLO

```bash
version: "3.9"

services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
      args:
        APP_ENV: production
    image: miapp:1.0
    container_name: miapp_container
    ports:
      - "8080:80"
    environment:
      - DB_HOST=db
      - DB_USER=${DB_USER}
      - DB_PASS=${DB_PASS}
    env_file:
      - .env
    volumes:
      - ./src:/usr/src/app         # bind mount
      - app_logs:/var/log/app      # volumen nombrado
    networks:
      - red_interna
    depends_on:
      - db
    restart: unless-stopped
    command: ["npm", "start"]

  db:
    image: postgres:15
    container_name: postgres_db
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
      POSTGRES_DB: miappdb
    volumes:
      - datos_db:/var/lib/postgresql/data
    networks:
      - red_interna

volumes:
  datos_db:              # volumen gestionado
  app_logs:              # volumen para logs
  datos_externos:
    external: true       # volumen externo

networks:
  red_interna:
    driver: bridge
  red_externa:
    external: true
```

## TRAEFIK

Es un reverse proxy y load balancer moderno que:

Se integra automáticamente con Docker (descubre contenedores y los enruta).

Gestiona certificados SSL/TLS (Let's Encrypt).

Soporta middlewares (autenticación, redirección, rate limit, etc.).

### COMO INSTANCIAR?

```bash
version: "3.9"

services:
  traefik:
    image: traefik:v2.11         # Imagen oficial
    container_name: traefik
    command:
      - "--api.insecure=true"    # Dashboard (solo en dev)
      - "--providers.docker=true" # Descubre servicios de Docker
      - "--entrypoints.web.address=:80"  # Puerto 80
      - "--entrypoints.websecure.address=:443" # Puerto 443
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080" # Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro # Permite a Traefik leer contenedores
```

### COMMAND DE TRAEFIK

```bash
--api.insecure=true         # Activa dashboard sin auth
--api.dashboard=true        # Activa dashboard
--providers.docker=true     # Activa integración con Docker
--entrypoints.web.address=:80     # Define entrada HTTP
--entrypoints.websecure.address=:443 # Define entrada HTTPS
--certificatesresolvers.le.acme.email=tu@correo.com   # Let's Encrypt
--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
--certificatesresolvers.le.acme.httpchallenge.entrypoint=web
```

#### POR ARCHIVO DE CONFIGURACION ESTATICA

Archivo `traefik.yml` en lugar de command

```bash
api:
  dashboard: true

providers:
  docker: {}

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"
```

y montarlo asi:

```bash
volumes:
  - ./traefik.yml:/etc/traefik/traefik.yml
```

### LABELS EN LOS SERVICIOS

Router: define cómo se recibe una petición (rule, entrypoint).

Service: define a dónde se envía la petición (backend/puerto).

Middleware: transforma la petición (auth, redirect, etc.).

```bash
services:
  app:
    image: nginx
    labels:
      - "traefik.enable=true"                           # Habilita Traefik para este contenedor
      - "traefik.http.routers.app.rule=Host(`app.local`)" # Define dominio
      - "traefik.http.routers.app.entrypoints=web"      # Puerto/entrypoint
      - "traefik.http.services.app.loadbalancer.server.port=80" # Puerto interno del contenedor
```

#### MIDLEWARES

Los middlewares son filtros o reglas adicionales para routers.
Se definen en labels y luego se asignan.

- Auth basica:

```bash
labels:
  - "traefik.http.middlewares.auth.basicauth.users=user:$$apr1$$C9F...hashedPassword"
  - "traefik.http.routers.app.middlewares=auth"
```

- Rate limit:

```bash
    labels:
    - "traefik.http.middlewares.ratelimit.ratelimit.average=100"
    - "traefik.http.middlewares.ratelimit.ratelimit.burst=50"
    - "traefik.http.routers.app.middlewares=ratelimit"
```

- Strip prefix:

```bash
    labels:
    - "traefik.http.middlewares.stripprefix.stripprefix.prefixes=/api"
    - "traefik.http.routers.app.middlewares=stripprefix"
```

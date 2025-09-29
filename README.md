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

# Creación de Dockers Multi-Servicio en AWS

## ¿Por qué Docker?

Docker nos permite **containerizar cada servicio de forma independiente**, garantizando que:
- Cada componente (Apache, PHP-FPM, MySQL) está **aislado** en su propio contenedor
- Se **escala fácilmente** (puedes añadir más instancias de S2/S3 sin complicaciones)
- La **reproducibilidad** es garantizada (mismo código = mismo comportamiento en cualquier máquina)
- La **configuración es versionable** (todo en archivos YAML)
- Se **simplifica el despliegue** en AWS (orquestación clara y predecible)
- Se **reducen dependencias** del sistema host (no necesitas instalar PHP, Apache, MySQL directamente)

En resumen: **una aplicación modular, portable y fácil de mantener**.

---

## Descripción General

Esta configuración despliega una aplicación web completa con:

- **S1**: Apache como proxy inverso (punto de entrada público)
- **S2 + S3**: PHP-FPM con balanceado de carga para `extagram.php`
- **S4**: PHP-FPM para gestión de uploads
- **S5**: Apache sirviendo imágenes estáticas (uploads)
- **S6**: Apache sirviendo CSS/SVG estáticos
- **S7**: MySQL Base de datos

Todos los servicios corren en **contenedores Docker** dentro de una **red interna** en AWS EC2.

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS EC2 Instance                     │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Docker Network (app-network)            │   │
│  │                                                      │   │
│  │  ┌─────────────────────────────────────────────┐     │   │
│  │  │ S1: Apache Proxy (80, 443)                  │     │   │
│  │  │ Entrada pública - Balanceador de carga      │     │   │
│  │  └───────────┬─────────────────────────────────┘     │   │
│  │              │                                       │   │
│  │    ┌─────────┼──────────────┬────────────────┐       │   │
│  │    ▼         ▼              ▼                ▼       │   │
│  │ ┌────────────────┐  ┌──────────────┐  ┌──────────┐   │   │
│  │ │ S2: PHP-FPM 1  │  │ S4: PHP-FPM  │  │S5: Apache│   │   │
│  │ │ extagram.php   │  │ upload.php   │  │Images    │   │   │
│  │ └────────────────┘  └──────────────┘  └──────────┘   │   │
│  │         ▲                                            │   │
│  │    ┌────┴───────────┐                                │   │
│  │    │                │                                │   │
│  │ ┌──────────────┐  ┌──────────┐  ┌──────────────┐     │   │
│  │ │ S3: PHP-FPM  │  │S6: Apache│  │ S7: MySQL    │     │   │
│  │ │ extagram.php │  │Assets    │  │ Base datos   │     │   │
│  │ └──────────────┘  └──────────┘  └──────────────┘     │   │
│  │                                                      │   │
│  │          Volúmenes Compartidos:                      │   │
│  │          - shared-uploads (S4, S5)                   │   │
│  │          - mysql_data (S7)                           │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Estructura de Directorios

```text
proyecto-docker/
├─ S1/
│  ├─ docker-compose.yml
│  ├─ conf/
│  │  └─ proxy.conf
│  └─ logs/
├─ S2/
│  ├─ docker-compose.yml
│  ├─ Dockerfile
│  └─ app/
│     ├─ extagram.php
│     └─ index.php
├─ S3/
│  ├─ docker-compose.yml
│  ├─ Dockerfile
│  └─ app/
│     ├─ extagram.php
│     └─ index.php
├─ S4/
│  ├─ docker-compose.yml
│  ├─ Dockerfile
│  └─ app/
│     └─ upload.php
├─ S5/
│  ├─ docker-compose.yml
│  ├─ conf/
│  │  └─ httpd-images.conf
│  └─ logs/
├─ S6/
│  ├─ docker-compose.yml
│  ├─ conf/
│  │  └─ httpd-assets.conf
│  ├─ assets/
│  │  ├─ style.css
│  │  └─ preview.svg
│  └─ logs/
├─ S7/
│  ├─ docker-compose.yml
│  ├─ init.sql
│  └─ data/
└─ launch.sh                   (script para lanzar todo)
```

---

## Configuración de Cada Servicio











### S5: Servidor Apache para Gestión de Uploads

```text
S5/
└── docker-compose.yml
```

**Archivo: `S5/docker-compose.yml`**

```yaml
services:
  apache-images:
    image: httpd:2.4
    container_name: S5-apache-images
    volumes:
      - shared-uploads:/usr/local/apache2/htdocs/uploads
    networks:
      - app-network
    restart: unless-stopped

networks:
  app-network:
    external: true

volumes:
  shared-uploads:
    external: true
```

**Función**: servir archivos subidos por usuarios (imágenes, fotos de posts). Utiliza un volumen Docker compartido `shared-uploads` que es accesible desde otros contenedores (como S2, que es donde se suben los archivos). De esta forma, cuando un usuario sube una imagen en Extagram (S2), se guarda en el volumen `shared-uploads`, y S5 la sirve por HTTP. Los otros servicios acceden a las imágenes usando `http://S5-apache-images/uploads/nombre-imagen.jpg` a través de la red Docker compartida `app-network`.

---

### S6: Assets estáticos

```text
S6/
├── docker-compose.yml
└── assets/
    ├── preview.svg
    └── style.css
```

**Archivo: `S6/docker-compose.yml`**

```yaml
version: "3.9"

services:
  apache-assets:
    image: httpd:2.4
    container_name: S6-apache-assets
    volumes:
      - ./assets:/usr/local/apache2/htdocs
    networks:
      - app-network
    restart: unless-stopped

networks:
  app-network:
    external: true
```

**Función**: desacopla la carga de servir archivos estáticos del servidor de aplicación. Está optimizado solo para entregar archivos rápidamente sin ejecutar PHP ni operaciones pesadas. Otros servicios acceden a los assets por HTTP a través de la red Docker compartida.

---

### S7: Servidor MySQL para Base de Datos

```text
S7/
├── docker-compose.yml
└── init.sql
```

**Archivo: `S7/docker-compose.yml`**

```yaml
services:
  mysql-db:
    image: mysql:8.0
    container_name: S7-mysql-db
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: extagram_db
      MYSQL_USER: extagram_admin
      MYSQL_PASSWORD: P0.1_G04
      MYSQL_INITDB_SKIP_TZINFO: "yes"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10
    restart: unless-stopped

networks:
  app-network:
    external: true

volumes:
  mysql_data:
    external: true
```

**Archivo: `S7/init.sql`**

```sql
CREATE DATABASE IF NOT EXISTS extagram_db;
USE extagram_db;

CREATE TABLE IF NOT EXISTS posts (
    post TEXT,
    photourl TEXT
);

CREATE USER IF NOT EXISTS 'extagram_admin'@'%' IDENTIFIED BY 'P0.1_G04';
GRANT ALL PRIVILEGES ON extagram_db.* TO 'extagram_admin'@'%';
FLUSH PRIVILEGES;
```

**Función**: contenedor MySQL 8.0 que aloja la base de datos `extagram_db`. Al iniciar, ejecuta automáticamente `init.sql` para crear las tablas y el usuario. Los datos persisten en el volumen `mysql_data` (externo). El healthcheck verifica continuamente que MySQL esté disponible. Se conecta a `app-network` permitiendo que otros servicios como S2 (Extagram) accedan a la base de datos usando `S7-mysql-db` como hostname. Las credenciales son usuario `extagram_admin` con contraseña `P0.1_G04`.

---

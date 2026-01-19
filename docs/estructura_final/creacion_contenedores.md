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

### S1: Apache Proxy Inverso

**Archivo: `S1/docker-compose.yml`**

```yaml
version: "3.9"

services:
  apache-proxy:
    image: httpd:2.4
    container_name: S1-apache-proxy
    ports:
      - "0.0.0.0:80:80"
      - "0.0.0.0:443:443"
    volumes:
      - ./conf/proxy.conf:/usr/local/apache2/conf/extra/proxy.conf:ro
      - ./logs:/usr/local/apache2/logs
    environment:
      AWS_REGION: eu-west-1
    command: >
      bash -c "
        sed -i 's/#LoadModule proxy_module/LoadModule proxy_module/' /usr/local/apache2/conf/httpd.conf &&
        sed -i 's/#LoadModule proxy_http_module/LoadModule proxy_http_module/' /usr/local/apache2/conf/httpd.conf &&
        sed -i 's/#LoadModule proxy_balancer_module/LoadModule proxy_balancer_module/' /usr/local/apache2/conf/httpd.conf &&
        sed -i 's/#LoadModule lbmethod_byrequests_module/LoadModule lbmethod_byrequests_module/' /usr/local/apache2/conf/httpd.conf &&
        sed -i 's/#LoadModule rewrite_module/LoadModule rewrite_module/' /usr/local/apache2/conf/httpd.conf &&
        sed -i 's/#LoadModule deflate_module/LoadModule deflate_module/' /usr/local/apache2/conf/httpd.conf &&
        sed -i 's/#LoadModule headers_module/LoadModule headers_module/' /usr/local/apache2/conf/httpd.conf &&
        echo 'Include conf/extra/proxy.conf' >> /usr/local/apache2/conf/httpd.conf &&
        httpd-foreground
      "
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  app-network:
    driver: bridge
    external: true
    name: app-network
```

**Función**: Actúa como punto de entrada único, distribuye tráfico a los servicios internos.

---

### S2: PHP-FPM 1 (extagram.php)

**Archivo: `S2/docker-compose.yml`**

```yaml
version: "3.9"

services:
  php-fpm-1:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: S2-php-fpm-1
    volumes:
      - ./app:/var/www/html
      - shared-uploads:/var/www/uploads
    environment:
      DB_HOST: S7-mysql-db
      DB_PORT: 3306
      DB_NAME: app_db
      DB_USER: app_user
      DB_PASS: app_password
      AWS_REGION: eu-west-1
      NODE_ENV: production
    networks:
      - app-network
    depends_on:
      S7-mysql-db:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "php", "-v"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  app-network:
    driver: bridge
    external: true
    name: app-network

volumes:
  shared-uploads:
    name: shared-uploads
    external: true
    driver: local
```

**Dockerfile: `S2/Dockerfile`**

```dockerfile
FROM php:8.1-fpm-alpine

RUN docker-php-ext-install pdo pdo_mysql

WORKDIR /var/www/html

CMD ["php-fpm"]
```

**Función**: Ejecuta PHP-FPM para procesar `extagram.php`. Balanceado con S3.

---

### S3: PHP-FPM 2 (extagram.php balanceado)

**Archivo: `S3/docker-compose.yml`**

```yaml
version: "3.9"

services:
  php-fpm-2:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: S3-php-fpm-2
    volumes:
      - ./app:/var/www/html
      - shared-uploads:/var/www/uploads
    environment:
      DB_HOST: S7-mysql-db
      DB_PORT: 3306
      DB_NAME: app_db
      DB_USER: app_user
      DB_PASS: app_password
      AWS_REGION: eu-west-1
      NODE_ENV: production
    networks:
      - app-network
    depends_on:
      S7-mysql-db:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "php", "-v"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  app-network:
    driver: bridge
    external: true
    name: app-network

volumes:
  shared-uploads:
    name: shared-uploads
    external: true
    driver: local
```

**Dockerfile: `S3/Dockerfile`** (igual a S2)

```dockerfile
FROM php:8.1-fpm-alpine

RUN docker-php-ext-install pdo pdo_mysql

WORKDIR /var/www/html

CMD ["php-fpm"]
```

**Función**: Segunda instancia de PHP-FPM. Apache distribuye las peticiones entre S2 y S3.

---

### S4: PHP-FPM Upload (upload.php)

**Archivo: `S4/docker-compose.yml`**

```yaml
version: "3.9"

services:
  php-fpm-upload:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: S4-php-fpm-upload
    volumes:
      - ./app:/var/www/html
      - shared-uploads:/var/www/uploads
    environment:
      DB_HOST: S7-mysql-db
      DB_PORT: 3306
      DB_NAME: app_db
      DB_USER: app_user
      DB_PASS: app_password
      UPLOAD_DIR: /var/www/uploads
      AWS_REGION: eu-west-1
      NODE_ENV: production
    networks:
      - app-network
    depends_on:
      S7-mysql-db:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "php", "-v"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  app-network:
    driver: bridge
    external: true
    name: app-network

volumes:
  shared-uploads:
    name: shared-uploads
    external: true
    driver: local
```

**Dockerfile: `S4/Dockerfile`** (igual a S2/S3)

```dockerfile
FROM php:8.1-fpm-alpine

RUN docker-php-ext-install pdo pdo_mysql

WORKDIR /var/www/html

CMD ["php-fpm"]
```

**Función**: Gestiona la carga de archivos. Los almacena en el volumen `shared-uploads`.

---

### S5: Apache servir Imágenes (uploads)

**Archivo: `S5/docker-compose.yml`**

```yaml
version: "3.9"

services:
  apache-images:
    image: httpd:2.4
    container_name: S5-apache-images
    volumes:
      - ./conf/httpd-images.conf:/usr/local/apache2/conf/extra/images.conf:ro
      - shared-uploads:/usr/local/apache2/htdocs:ro
      - ./logs:/usr/local/apache2/logs
    environment:
      AWS_REGION: eu-west-1
    command: >
      bash -c "
        sed -i 's/#LoadModule deflate_module/LoadModule deflate_module/' /usr/local/apache2/conf/httpd.conf &&
        sed -i 's/#LoadModule headers_module/LoadModule headers_module/' /usr/local/apache2/conf/httpd.conf &&
        echo 'Include conf/extra/images.conf' >> /usr/local/apache2/conf/httpd.conf &&
        httpd-foreground
      "
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  app-network:
    driver: bridge
    external: true
    name: app-network

volumes:
  shared-uploads:
    name: shared-uploads
    external: true
    driver: local
```

**Configuración: `S5/conf/httpd-images.conf`**

```apache
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot /usr/local/apache2/htdocs

    <Directory /usr/local/apache2/htdocs>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Compresión
    <IfModule mod_deflate.c>
        AddOutputFilterByType DEFLATE image/svg+xml
    </IfModule>

    # Cache agresiva para imágenes
    <IfModule mod_headers.c>
        <FilesMatch "\.(png|jpg|jpeg|gif|webp|svg|svgz)$">
            Header set Cache-Control "public, max-age=31536000, immutable"
        </FilesMatch>
    </IfModule>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

**Función**: Sirve las imágenes subidas por el usuario. Cache de 1 año.

---

### S6: Apache servir Estáticos (CSS/SVG)

**Archivo: `S6/docker-compose.yml`**

```yaml
version: "3.9"

services:
  apache-assets:
    image: httpd:2.4
    container_name: S6-apache-assets
    volumes:
      - ./conf/httpd-assets.conf:/usr/local/apache2/conf/extra/assets.conf:ro
      - ./assets:/usr/local/apache2/htdocs:ro
      - ./logs:/usr/local/apache2/logs
    environment:
      AWS_REGION: eu-west-1
    command: >
      bash -c "
        sed -i 's/#LoadModule deflate_module/LoadModule deflate_module/' /usr/local/apache2/conf/httpd.conf &&
        sed -i 's/#LoadModule headers_module/LoadModule headers_module/' /usr/local/apache2/conf/httpd.conf &&
        echo 'Include conf/extra/assets.conf' >> /usr/local/apache2/conf/httpd.conf &&
        httpd-foreground
      "
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  app-network:
    driver: bridge
    external: true
    name: app-network
```

**Configuración: `S6/conf/httpd-assets.conf`**

```apache
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot /usr/local/apache2/htdocs

    <Directory /usr/local/apache2/htdocs>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Compresión Gzip
    <IfModule mod_deflate.c>
        AddOutputFilterByType DEFLATE text/css image/svg+xml application/javascript text/javascript
    </IfModule>

    # Cache para estáticos
    <IfModule mod_headers.c>
        <FilesMatch "\.(css|js|svg|svgz|woff2?|ttf|eot)$">
            Header set Cache-Control "public, max-age=31536000, immutable"
        </FilesMatch>
    </IfModule>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

**Archivos estáticos: `S6/assets/`**

Contiene:
- `style.css` - Estilos CSS
- `preview.svg` - Imágenes vectoriales

**Función**: Sirve CSS y SVG optimizados con compresión Gzip.

---

### S7: MySQL Base de Datos

**Archivo: `S7/docker-compose.yml`**

```yaml
version: "3.9"

services:
  mysql-db:
    image: mysql:8.0
    container_name: S7-mysql-db
    environment:
      MYSQL_ROOT_PASSWORD: '##############'
      MYSQL_DATABASE: extagram_db
      MYSQL_USER: extagram_admin
      MYSQL_PASSWORD: '##############'
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
    driver: bridge
    external: true
    name: app-network

volumes:
  mysql_data:
    name: mysql_data
    external: true
    driver: local
```

**Base de datos: `S7/init.sql`**

```sql
CREATE TABLE IF NOT EXISTS uploads (
    id INT AUTO_INCREMENT PRIMARY KEY,
    filename VARCHAR(255) NOT NULL UNIQUE,
    file_data LONGBLOB,
    uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS extagram (
    id INT AUTO_INCREMENT PRIMARY KEY,
    content TEXT NOT NULL,
    image_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (image_id) REFERENCES uploads(id)
);

INSERT INTO extagram (content, image_id) VALUES 
('Primer post', NULL),
('Segundo post', NULL);
```

**Función**: Almacena datos de la aplicación. Replica archivos como BLOBs.

---





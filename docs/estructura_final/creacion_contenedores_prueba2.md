# Infraestructura de Microservicios

## 1. Visión Global de la Arquitectura

En este Sprint se ha realizado la migración de una arquitectura monolítica a una arquitectura de microservicios contenerizada. El sistema se ha dividido en **7 contenedores (S1-S7)**, cada uno con una responsabilidad única, orquestados mediante Docker Compose.

**Diagrama de Servicios:**

* **S1 (Proxy):** Puerta de entrada (Balanceador de carga).
* **S2 & S5 (Backend):** Aplicación Web PHP (Escalabilidad horizontal).
* **S4 (CDN/Storage):** Servidor dedicado para servir imágenes estáticas.
* **S7 (Datos):** Base de Datos MySQL.
* **S3 (Gestión):** Interfaz visual para la Base de Datos.
* **S6 (Mantenimiento):** Servicio de Backups automatizados.

---

## 2. Definición Detallada de Contenedores

### S1: Proxy Inverso y Balanceador de Carga

**Función:** Es el único punto de acceso público (Puerto 80). Recibe las peticiones del usuario y las distribuye entre los backends (S2 y S5) para balancear la carga, o hacia S4 para pedir imágenes.

**Configuración (`S1/nginx.conf`):**

```nginx
events {}
http {
    upstream backend_cluster {
        server S2-backend-1:80;
        server S5-backend-2:80;
    }

    server {
        listen 80;
        
        # Redirigir tráfico PHP al clúster de backends
        location / {
            proxy_pass http://backend_cluster;
            proxy_set_header Host $host;
        }

        # Servir imágenes directamente desde el contenedor de almacenamiento S4
        location /uploads/ {
            proxy_pass http://S4-backend-upload:80/uploads/;
        }
    }
}

```

---

### S2: Backend Principal (Extagram App)

**Imagen:** `Dockerfile` (Personalizado PHP 8.0 + MySQLi)
**Función:** Procesa la lógica de negocio (PHP), gestiona la autenticación y procesa la subida de archivos. Escribe los archivos físicos en un **Volumen Compartido** para que S4 pueda verlos.

**Dockerfile (`S2/Dockerfile`):**

```dockerfile
FROM php:8.0-apache
RUN docker-php-ext-install mysqli && docker-php-ext-enable mysqli
RUN a2enmod rewrite
COPY src/ /var/www/html/
RUN chown -R www-data:www-data /var/www/html/

```

---

### S3: Administración de Base de Datos

**Imagen:** `phpmyadmin/phpmyadmin`
**Función:** Proporciona una interfaz gráfica web para gestionar la base de datos `extagram_db` en S7 sin necesidad de usar la terminal. Útil para depuración y mantenimiento.

**Variables de Entorno Clave:**

* `PMA_HOST`: S7-mysql-db
* `PMA_USER`: extagram_admin
* `PMA_PASSWORD`: P0.1_G04

---

### S4: Servidor de Archivos Estáticos (Uploads)

**Imagen:** `httpd:alpine` (Apache ligero) o `nginx:alpine`
**Nombre del contenedor:** `S4-backend-upload`
**Función:** Servir las imágenes de manera eficiente. Este contenedor **NO** procesa PHP, solo entrega archivos `.jpg`, `.gif`, etc.
**Montaje:** Tiene montado el volumen `shared-uploads` en su raíz web.

*Nota: Este es el contenedor donde verificamos los archivos con `ls`.*

---

### S5: Backend Replica (Alta Disponibilidad)

**Imagen:** Misma construcción que **S2**.
**Función:** Es un clon exacto de S2. Si S2 se cae o está saturado, S1 envía el tráfico a S5. Garantiza que la aplicación siga funcionando. Comparte el mismo código y el mismo volumen de uploads.

---

### S6: Servicio de Backup

**Imagen:** `alpine:latest`
**Función:** Contenedor de utilidad que ejecuta tareas programadas (cron) para realizar copias de seguridad de la base de datos S7.

**Comando de ejecución:**
Instala el cliente mysql y ejecuta `mysqldump` periódicamente, guardando los `.sql` en un volumen seguro.

---

### S7: Base de Datos Maestra

**Imagen:** `mysql:8.0`
**Función:** Almacena toda la información persistente.

**Archivo `S7/init.sql`:**

```sql
CREATE DATABASE IF NOT EXISTS extagram_db;
USE extagram_db;
CREATE TABLE IF NOT EXISTS posts (post TEXT, photourl TEXT);
CREATE USER 'extagram_admin'@'%' IDENTIFIED BY 'P0.1_G04';
GRANT ALL PRIVILEGES ON extagram_db.* TO 'extagram_admin'@'%';
FLUSH PRIVILEGES;

```

---

## 3. Orquestación (Docker Compose)

Este archivo `docker-compose.yml` despliega toda la infraestructura descrita anteriormente.

```yaml
version: '3.8'

services:
  # --- S1: BALANCEADOR DE CARGA (Nginx) ---
  proxy:
    image: nginx:alpine
    container_name: S1-proxy
    ports:
      - "80:80" # Único puerto expuesto al exterior
    volumes:
      - ./S1/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - backend-1
      - backend-2
      - upload-server
    networks:
      - app-network

  # --- S2: BACKEND 1 (PHP App) ---
  backend-1:
    build: ./S2
    container_name: S2-backend-1
    restart: always
    volumes:
      - ./S2/src:/var/www/html        # Código fuente
      - shared-uploads:/var/www/html/uploads # Volumen compartido imágenes
    environment:
      - DB_HOST=S7-mysql-db
    networks:
      - app-network
    depends_on:
      mysql-db:
        condition: service_healthy

  # --- S3: PHPMYADMIN (GUI DB) ---
  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: S3-phpmyadmin
    ports:
      - "8080:80" # Acceso directo a admin en puerto 8080
    environment:
      PMA_HOST: S7-mysql-db
      PMA_USER: extagram_admin
      PMA_PASSWORD: P0.1_G04
    networks:
      - app-network

  # --- S4: STORAGE SERVER (Archivos Estáticos) ---
  upload-server:
    image: httpd:alpine
    container_name: S4-backend-upload
    volumes:
      - shared-uploads:/usr/local/apache2/htdocs/uploads # Lee lo que S2 escribe
    networks:
      - app-network

  # --- S5: BACKEND 2 (Réplica) ---
  backend-2:
    build: ./S2 # Usa el mismo Dockerfile que S2
    container_name: S5-backend-2
    restart: always
    volumes:
      - ./S2/src:/var/www/html
      - shared-uploads:/var/www/html/uploads
    networks:
      - app-network
    depends_on:
      mysql-db:
        condition: service_healthy

  # --- S6: BACKUP SERVICE ---
  backup-service:
    image: alpine:latest
    container_name: S6-backup
    command: >
      sh -c "apk add --no-cache mysql-client &&
             while true; do
               mysqldump -h S7-mysql-db -u extagram_admin -pP0.1_G04 extagram_db > /backups/backup_$(date +%s).sql;
               echo 'Backup realizado';
               sleep 86400; # Cada 24 horas
             done"
    volumes:
      - ./backups:/backups
    networks:
      - app-network
    depends_on:
      - mysql-db

  # --- S7: BASE DE DATOS (MySQL) ---
  mysql-db:
    image: mysql:8.0
    container_name: S7-mysql-db
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: extagram_db
      MYSQL_USER: extagram_admin
      MYSQL_PASSWORD: P0.1_G04
    volumes:
      - mysql_data:/var/lib/mysql
      - ./S7/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10

# --- REDES Y VOLÚMENES ---
networks:
  app-network:
    driver: bridge

volumes:
  mysql_data:      # Persistencia de BBDD
  shared-uploads:  # Volumen mágico compartido entre S2, S4 y S5

```

---

## 4. Estructura de Archivos Final

```text
proyecto/
├── docker-compose.yml
├── backups/               (Carpeta local para dumps de S6)
├── S1/
│   └── nginx.conf         (Config del balanceador)
├── S2/                    (Sirve para construir S2 y S5)
│   ├── Dockerfile
│   └── src/
│       ├── index.php      (La Galería Pro)
│       ├── upload.php     (Lógica de subida)
│       ├── db_config.php  (Apunta a S7-mysql-db)
│       └── .htaccess      (Límites 20MB)
└── S7/
    └── init.sql           (Script inicial DB)

```

## 5. Conclusión Técnica

Con esta configuración se ha logrado un entorno **altamente disponible y desacoplado**.

1. Si el contenedor **S2** falla, **S1** redirige a **S5**.
2. Si borramos los contenedores Web, las imágenes persisten en el volumen `shared-uploads` (servido por **S4**) y los datos en `mysql_data` (servido por **S7**).
3. Tenemos una estrategia de seguridad de datos con **S6** realizando backups automáticos.

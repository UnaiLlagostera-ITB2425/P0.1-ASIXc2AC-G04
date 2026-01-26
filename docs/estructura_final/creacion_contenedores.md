# Arquitectura de Microservicios Distribuidos "Extagram"

## 1. Introducción y Objetivos

El objetivo de este sprint ha sido descomponer la aplicación monolítica original en una arquitectura de **microservicios contenerizados**. Esta transformación busca mejorar la escalabilidad, la mantenibilidad y la tolerancia a fallos del sistema.

Se ha diseñado una infraestructura basada en **7 contenedores independientes (S1-S7)**, orquestados mediante Docker, donde cada servicio tiene una responsabilidad única (Principio de Responsabilidad Única).

### ¿Por qué Docker?

Docker nos permite **containerizar cada servicio de forma independiente**, garantizando que:
- Cada componente (Apache, PHP-FPM, MySQL) está **aislado** en su propio contenedor
- Se **escala fácilmente** (puedes añadir más instancias de S2/S3 sin complicaciones)
- La **reproducibilidad** es garantizada (mismo código = mismo comportamiento en cualquier máquina)
- La **configuración es versionable** (todo en archivos YAML)
- Se **simplifica el despliegue** en AWS (orquestación clara y predecible)
- Se **reducen dependencias** del sistema host (no necesitas instalar PHP, Apache, MySQL directamente)

En resumen: **una aplicación modular, portable y fácil de mantener**.


## 2. Infraestructura Base y Redes

Para garantizar la comunicación entre servicios aislados y la persistencia de datos crítica, se establecieron los siguientes elementos de infraestructura compartida antes del despliegue:

* **Red Virtual (`app-network`):** Una red tipo *bridge* que permite la resolución de nombres DNS entre contenedores (ej. `S1` puede llamar a `S2` por su nombre).
* **Volumen de Datos (`mysql_data`):** Garantiza que la base de datos no pierda información si el contenedor S7 se reinicia.
* **Volumen de Archivos (`shared-uploads`):** Un volumen crítico compartido entre los backends (S2, S3, S4) y el servidor de imágenes (S5) para permitir la consistencia de archivos.

---

## 3. Capa de Persistencia de Datos (S7)

El corazón del sistema es el servicio de base de datos. Se ha configurado para ser resiliente y autoinicializable.

### Servicio S7: Base de Datos Maestra

* **Motor:** MySQL 8.0
* **Estrategia de Inicio:** Se utiliza un script SQL montado en `/docker-entrypoint-initdb.d/` para crear automáticamente el esquema y los usuarios al primer arranque.

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
      MYSQL_INITDB_SKIP_TZINFO: "yes" # Optimización tiempo de arranque
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck: # Garantiza que la DB está lista antes que la App
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

**Script de Inicialización: `S7/init.sql`**

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

---

## 4. Capa de Lógica de Negocio (Backend Cluster)

La aplicación PHP se ha dividido en tres contenedores para separar la carga de lectura/visualización de la carga de escritura/subida.

### Imagen Base Personalizada (S2, S3, S4)

Todos los servicios de backend comparten la misma definición de construcción para asegurar paridad de entorno.

**Archivo: `S2/Dockerfile`**

```dockerfile
FROM php:8.0-apache
# Instalación de drivers para conexión con S7
RUN docker-php-ext-install mysqli && docker-php-ext-enable mysqli
# Asignación de permisos para escritura de uploads
RUN chown -R www-data:www-data /var/www/html

```

### Servicios S2 y S3: Clúster Web (Balanceado)

Estos nodos manejan el tráfico de usuarios general. S1 balancea la carga entre ellos. Si uno falla, el otro asume el trabajo.

**Archivo: `S2/docker-compose.yml` (Idéntico estructura para S3)**

```yaml
services:
  app-s2:
    build: .
    container_name: S2-backend-1
    volumes:
      - ../src:/var/www/html        # Código fuente en vivo
      - shared-uploads:/var/www/html/uploads # Acceso a fotos subidas
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

### Servicio S4: Backend de Carga Dedicado

Se ha aislado la lógica de `upload.php` en un contenedor separado. Esto evita que el procesamiento de imágenes pesadas ralentice la navegación general de la web.

**Archivo: `S4/docker-compose.yml`**

```yaml
services:
  app-s4:
    build: .
    container_name: S4-backend-upload
    volumes:
      - ../src:/var/www/html
      - shared-uploads:/var/www/html/uploads
    networks:
      - app-network
    restart: unless-stopped
    # ... (redes y volúmenes igual que S2)

```

---

## 5. Capa de Contenido Estático (CDN Interno)

Para liberar a los backends PHP de servir archivos estáticos, se han desplegado servidores Apache ligeros.

### Servicio S5: Servidor de Imágenes

Sirve exclusivamente el contenido de la carpeta `/uploads`.

**Archivo: `S5/docker-compose.yml`**

```yaml
services:
  apache-images:
    image: httpd:2.4
    container_name: S5-apache-images
    volumes:
      # Monta solo el volumen de uploads, no el código PHP
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

### Servicio S6: Servidor de Assets

Sirve hojas de estilo y gráficos de la interfaz.

**Archivo: `S6/docker-compose.yml`**

```yaml
services:
  apache-assets:
    image: httpd:2.4
    container_name: S6-apache-assets
    volumes:
      - ./assets:/usr/local/apache2/htdocs # Mapeo local de assets
    networks:
      - app-network
    restart: unless-stopped

networks:
  app-network:
    external: true

```

---

## 6. Capa de Acceso y Enrutamiento (Gateway)

El componente más complejo es **S1**, que actúa como Proxy Inverso y Balanceador de Carga. Es el único punto de contacto con el exterior (Puerto 80).

### Construcción del Gateway (S1)

Se ha personalizado la imagen de Apache para habilitar módulos de proxy y balanceo que vienen desactivados por defecto.

**Archivo: `S1/Dockerfile`**

```dockerfile
FROM httpd:2.4
COPY ./conf/extagram.conf /usr/local/apache2/conf/extra/extagram.conf

# Script de activación de módulos (Proxy, Balancer, Slotmem)
RUN sed -i \
    -e 's/#LoadModule proxy_module/LoadModule proxy_module/' \
    -e 's/#LoadModule proxy_http_module/LoadModule proxy_http_module/' \
    -e 's/#LoadModule proxy_balancer_module/LoadModule proxy_balancer_module/' \
    -e 's/#LoadModule lbmethod_byrequests_module/LoadModule lbmethod_byrequests_module/' \
    -e 's/#LoadModule slotmem_shm_module/LoadModule slotmem_shm_module/' \
    /usr/local/apache2/conf/httpd.conf && \
    echo "Include conf/extra/extagram.conf" >> /usr/local/apache2/conf/httpd.conf

```

### Configuración de Rutas y Balanceo

El archivo de configuración define cómo se distribuye el tráfico basándose en la URL solicitada.

**Archivo: `S1/conf/extagram.conf`**

```apache
<VirtualHost *:80>
    ServerName localhost
    ProxyPreserveHost On

    # 1. Rutas Estáticas (Assets e Imágenes) -> Van a S6 y S5
    ProxyPass "/style.css" "http://S6-apache-assets/style.css"
    ProxyPassReverse "/style.css" "http://S6-apache-assets/style.css"

    ProxyPass "/uploads/" "http://S5-apache-images/uploads/"
    ProxyPassReverse "/uploads/" "http://S5-apache-images/uploads/"

    # 2. Ruta de Subida (Operación Pesada) -> Va a S4
    ProxyPass "/upload.php" "http://S4-backend-upload/upload.php"
    ProxyPassReverse "/upload.php" "http://S4-backend-upload/upload.php"

    # 3. Balanceador de Carga (Tráfico Web General) -> Cluster S2 + S3
    <Proxy "balancer://mycluster">
        BalancerMember "http://S2-backend-1:80"
        BalancerMember "http://S3-backend-2:80"
    </Proxy>
    ProxyPass "/" "balancer://mycluster/"
    ProxyPassReverse "/" "balancer://mycluster/"
</VirtualHost>

```

**Archivo: `S1/docker-compose.yml`**

```yaml
services:
  gateway:
    build: .
    container_name: S1-gateway
    ports:
      - "80:80"  # Exposición pública
    networks:
      - app-network
    restart: unless-stopped

networks:
  app-network:
    external: true

```

---

## 7. Conclusión del Despliegue

La arquitectura implementada cumple con los requisitos de **desacoplamiento**.

1. **S7** mantiene los datos seguros independientemente de la aplicación.
2. **S5** y **S6** descargan el tráfico estático, permitiendo que PHP respire.
3. **S1** protege la red interna y permite escalar horizontalmente añadiendo más nodos al clúster (S2/S3).

Este diseño permite realizar mantenimientos (como reiniciar S2) sin que el usuario final perciba una caída del servicio, ya que S1 redirigirá automáticamente a S3.

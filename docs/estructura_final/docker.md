# Arquitectura del Sistema Extagram

El sistema **Extagram** implementa una arquitectura de microservicios contenerizada basada en el patrón **Gateway Offloading**. La infraestructura desacopla la capa de presentación, la lógica de negocio, el procesamiento de carga de archivos (I/O intensivo) y la persistencia de datos.

Todos los servicios operan dentro de una red aislada (`app-network`), exponiendo únicamente el Gateway (**S1**) hacia la red pública/host.

## 1. Topología de Red y Flujo de Datos

El siguiente diagrama representa el flujo de tráfico HTTP/HTTPS y las dependencias de conexión a base de datos.


### 1.2 Inventario de Componentes

| ID | Servicio | Imagen Base | Rol Funcional | Dependencias |
| --- | --- | --- | --- | --- |
| **S1** | `s1-proxy` | `httpd:2.4` | Reverse Proxy, SSL Termination, L7 Load Balancer. | N/A |
| **S2** | `s2-app` | `php:8.0-apache` | Lógica de negocio (Lectura/Renderizado). | S7 |
| **S3** | `s3-app` | `php:8.0-apache` | Réplica de lógica de negocio (Alta Disponibilidad). | S7 |
| **S4** | `s4-upload` | `php:8.0-apache` | Procesamiento exclusivo de uploads (Escritura). | S7, Vol. Uploads |
| **S5** | `s5-images` | `httpd:2.4` | Servidor estático de alto rendimiento para imágenes de usuario. | Vol. Uploads |
| **S6** | `s6-static` | `httpd:2.4` | Servidor de activos del frontend (CSS, JS, Logos). | N/A |
| **S7** | `s7-db` | `mysql:8.0` | Motor de base de datos relacional. | Vol. DB |

---

## 2. Configuración de Red y Enrutamiento (Gateway S1)

El servicio **S1** actúa como único punto de entrada. Su configuración se basa en `httpd.conf` con módulos `proxy_http`, `proxy_balancer`, `lbmethod_byrequests` y `ssl` habilitados.

### 2.1 SSL Offloading

El tráfico se recibe en el puerto **443**. La terminación SSL ocurre en S1, desencriptando el tráfico antes de enviarlo a los microservicios internos mediante HTTP estándar.

* **Certificados:** Ubicados en `./certs/server.crt` y `./certs/server.key`.
* **Redirección:** `VirtualHost *:80` fuerza una redirección **301** hacia HTTPS.

### 2.2 Reglas de Proxy (`extagram.conf`)

La lógica de enrutamiento se define explícitamente para optimizar el rendimiento y aislar cargas de trabajo:

1. **Balanceo de Carga (Lectura):**
* **Ruta:** `/` (Raíz y tráfico general).
* **Destino:** `balancer://mycluster` (compuesto por `http://s2-app` y `http://s3-app`).
* **Algoritmo:** `byrequests` (Round Robin).


2. **Aislamiento de Escritura (Uploads):**
* **Ruta:** `/upload.php`
* **Destino:** `http://s4-upload`
* > **Justificación Técnica:** La subida de archivos consume hilos de Apache y memoria I/O. Al enrutar esto a S4, se evita que los uploads bloqueen la navegación de usuarios en S2/S3.




3. **Entrega de Contenido Estático (CDN Interno):**
* **Ruta:** `/uploads/`  `http://s5-images`
* **Ruta:** `/style.css` (o `/assets/`)  `http://s6-static`
* > **Justificación Técnica:** Servir estáticos desde contenedores httpd puros elimina el overhead del intérprete PHP presente en S2/S3.





---

## 3. Definición de Servicios y Build

### 3.1 Cluster de Aplicación (S2, S3, S4)

Estos servicios comparten una definición de `Dockerfile` para garantizar paridad de entorno.

* **Base:** `php:8.0-apache`.
* **Extensiones:** Instalación de `mysqli` mediante `docker-php-ext-install` para conectividad con MySQL 8.0.
* **Permisos:** `chown -R www-data:www-data /var/www/html`. El servidor web se ejecuta bajo el usuario `www-data` por razones de seguridad (principio de menor privilegio).
* **Montaje:** El código fuente en `./src` se monta en `/var/www/html` en tiempo de ejecución.

### 3.2 Base de Datos (S7)

Instancia estándar de MySQL 8.0.

* **Inicialización:** Utiliza el script `/docker-entrypoint-initdb.d/init.sql` en el primer arranque para crear la base de datos `extagram_db`, la tabla `posts` y los usuarios necesarios.
* **Credenciales:** Inyectadas vía variables de entorno (`MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD`).

### 3.3 Servidores Estáticos (S5, S6)

Utilizan la imagen `httpd:2.4` sin modificaciones de build (Vanilla). La configuración depende exclusivamente del montaje de volúmenes en `/usr/local/apache2/htdocs/`.

---

## 4. Estrategia de Persistencia y Volúmenes

La infraestructura utiliza volúmenes Docker gestionados para garantizar la persistencia de datos críticos más allá del ciclo de vida de los contenedores.

### 4.1 Volúmenes Definidos

1. **`mysql_data` (Crítico):**
* **Montaje:** `/var/lib/mysql` en **S7**.
* **Propósito:** Almacena los ficheros BLOB, esquemas y tablas de MySQL. Sin este volumen, la base de datos se reinicia a cero tras cada despliegue.


2. **`shared-uploads` (Compartido):**
* **Montaje:**
* **S4 (Escritura):** `/var/www/html/uploads`
* **S5 (Lectura):** `/usr/local/apache2/htdocs/uploads`

* **Propósito:** Permite que el nodo S4 escriba un archivo físico y que el nodo S5 lo sirva inmediatamente. Resuelve el problema de almacenamiento efímero en contenedores distribuidos.



---

## 5. Procedimientos de Infraestructura (Lifecycle)

### 5.1 Prerrequisitos del Host

* Motor **Docker Engine 20.10+** y **Docker Compose V2**.
* Estructura de directorios:

```text
.
├── docker-compose.yml
├── conf/
│   └── extagram.conf
├── certs/
│   ├── server.crt
│   └── server.key
├── src/            # código PHP
├── assets/         # estáticos
└── sql/
    └── init.sql

```

### 5.2 Instalación (Despliegue en frío)

Dado que la red y los volúmenes están definidos como `external: true` (o requieren persistencia explícita), deben inicializarse antes de levantar el stack.

**1. Creación de objetos de infraestructura:**

```bash
docker network create app-network
docker volume create mysql_data
docker volume create shared-uploads

```

**2. Despliegue de servicios:**

```bash
docker-compose up -d --build

```

### 5.3 Verificación de Salud

* **Estado de contenedores:** `docker-compose ps` (Todos deben estar en estado `Up`).
* **Conectividad DB:**
```bash
docker exec -it s7-db bash
# mysql -p
(password)

```



### 5.4 Escalado Horizontal

Para aumentar la capacidad de procesamiento de peticiones de lectura, escalar las réplicas del servicio `s2-app` (nota: S1 balanceará automáticamente las nuevas instancias si usa descubrimiento DNS interno).

```bash
docker-compose up -d --scale s2-app=3

```

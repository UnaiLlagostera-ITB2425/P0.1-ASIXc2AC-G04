# Manual de Administración e Infraestructura - Extagram

Este documento detalla la arquitectura, despliegue y mantenimiento de la plataforma Extagram.

## 1. Arquitectura del Sistema

La infraestructura se basa en microservicios orquestados con Docker Compose, utilizando una red interna `extagram-net`.

### Servicios

| Servicio | Contenedor | Descripción | Puerto Interno |
| :--- | :--- | :--- | :--- |
| **S1** | `s1-proxy` | **Load Balancer & Reverse Proxy**. Apache 2.4. Punto de entrada único (SSL/HTTP). Distribuye tráfico a S2, S3, S4, S5 y S6. | 80, 443 |
| **S2, S3** | `s2-app`, `s3-app` | **Backend Read-Replica**. PHP 8.1. Procesan la lógica de lectura (`index.php`) y visualización. Balanceados por S1. | 80 |
| **S4** | `s4-upload` | **Backend Write**. PHP 8.1. Procesa exclusivamente la subida de ficheros (`upload.php`) y escritura en disco. | 80 |
| **S5** | `s5-images` | **Static Media Server**. Apache 2.4. Sirve las imágenes de usuario almacenadas en disco (`/uploads`). | 80 |
| **S6** | `s6-static` | **Assets Server**. Apache 2.4. Sirve archivos estáticos del sistema (CSS, JS, logos). | 80 |
| **S7** | `s7-db` | **Database**. MySQL 8.0. Almacena metadatos y BLOBs de imágenes. Persistencia mediante volumen Docker. | 3306 |

### Volúmenes y Persistencia

* **Código Fuente:** Mapeado desde `./src` hacia `/var/www/html` en los contenedores PHP.
* **Uploads (Físicos):** Mapeado desde `./src/uploads` hacia los contenedores S4 (escritura) y S5 (lectura).
* **Base de Datos:** Volumen gestionado por Docker `db_data` para persistencia de MySQL en `/var/lib/mysql`.

---

## 2. Requisitos del Host

* **SO:** Linux (Ubuntu 20.04/22.04 recomendado).
* **Software:**
    * Docker Engine v20.10+
    * Docker Compose v1.29+ o Plugin Compose v2.0+
* **Recursos Mínimos:** 2 vCPU, 2GB RAM (MySQL 8.0 requiere memoria significativa).
* **Puertos Host:** 80 (HTTP) y 443 (HTTPS) deben estar libres.

---

## 3. Instalación y Despliegue

### 3.1. Preparación del Entorno

1. Clonar el repositorio o copiar los archivos al servidor en `~/extagram`.
2. Asegurar la existencia de los certificados SSL en `./certs/`:
    ```bash
    # Estructura requerida
    certs/
    ├── server.crt
    └── server.key
    ```
3. Verificar permisos de ejecución en scripts de utilidad:
    ```bash
    chmod +x *.sh
    ```

### 3.2. Arranque del Sistema

Ejecuta el siguiente comando para construir las imágenes y levantar los contenedores en segundo plano:

```bash
sudo docker-compose up -d --build
```

### 3.3. Verificación

Comprueba que los 7 contenedores están en estado Up:

```bash
sudo docker ps
```

---

## 4. Configuración

### Variables de Entorno (Docker Compose)

Las credenciales están definidas en `docker-compose.yml`. Para producción, se recomienda migrarlas a un archivo `.env`.

* `MYSQL_ROOT_PASSWORD`: Contraseña root de MySQL (Default: `root_secret`).
* `MYSQL_DATABASE`: Nombre de la base de datos (Default: `extagram`).

### Configuración PHP (`uploads.ini`)

Configurado automáticamente en `Dockerfile.upload`:

* `upload_max_filesize`: 64M
* `post_max_size`: 64M

### Configuración Proxy (`conf/extagram.conf`)

* Redirección forzosa de HTTP a HTTPS.
* Balanceo de carga `byrequests` para cluster `mycluster` (S2, S3).

---

## 5. Operaciones y Mantenimiento

### 5.1. Logs del Sistema

Para ver los logs de un servicio específico (ej. base de datos):

```bash
sudo docker logs -f s7-db
```

### 5.2. Copias de Seguridad (Backup)

El sistema incluye un script automatizado para el volcado de la base de datos (incluyendo BLOBs).

**Ejecutar backup manual:**

```bash
./backup.sh
```

* **Ubicación:** `./backups/`
* **Formato:** `.sql.gz`
* **Retención:** El script elimina automáticamente backups mayores a 7 días.

### 5.3. Restauración (Disaster Recovery)

Para restaurar el último backup disponible (Destructivo para la DB actual):

```bash
./restore.sh
```

### 5.4. Limpieza de Uploads

Si se requiere purgar las imágenes subidas manualmente:

```bash
sudo rm -rf src/uploads/*
```

**Nota:** Esto generará errores 404 si las entradas siguen existiendo en la BBDD.

---

## 6. Solución de Problemas (Troubleshooting)

### Error: "Connection Refused" en Base de Datos

**Síntoma:** La web muestra "Error de BBDD" al arrancar.

**Causa:** MySQL 8.0 tarda en inicializar la primera vez.

**Solución:** Esperar 30 segundos. El contenedor tiene un healthcheck configurado; si falla, Docker intentará reiniciarlo.

### Error: "413 Request Entity Too Large" al subir fotos

**Causa:** La imagen supera el límite configurado en PHP o Apache.

**Solución:** Verificar que la imagen es menor a 64MB. Revisar `Dockerfile.upload` si se requiere aumentar el límite.

### Contenedor S7-DB estado "Exited" o Zombie

**Causa:** Probable falta de memoria RAM (OOM Killer).

**Solución:**

1. Reiniciar Docker: `sudo systemctl restart docker`.
2. Aumentar SWAP en el host si se usa una instancia pequeña (t2.micro/t3.micro).

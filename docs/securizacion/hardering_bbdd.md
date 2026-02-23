# Documentación de Incidencia: Hardening de Base de Datos (S7)

## 1. Objetivo
Incrementar la postura de seguridad del servicio de base de datos (`s7-db`, MySQL 8.0) aplicando el principio de defensa en profundidad. El objetivo es mitigar el impacto de posibles vulnerabilidades en la capa de aplicación (como Inyecciones SQL) impidiendo la exfiltración de datos o el compromiso del sistema de archivos del contenedor.

## 2. Alcance de la Intervención
La modificación se ha realizado a nivel de configuración del motor de la base de datos dentro de la orquestación (`docker-compose.yml`), asegurando que los cambios sean persistentes, reproducibles y no afecten la conectividad de los servicios dependientes (S2, S3, S4).

## 3. Implementación Técnica

Se ha inyectado la directiva `command` y modificado el bloque `environment` del servicio `s7-db` de la siguiente manera:

```yaml
s7-db:
  image: mysql:8.0
  container_name: s7-db
  restart: always
  # --- IMPLEMENTACIÓN DE HARDENING ---
  command:
    - --local-infile=0          # Bloquea lectura de archivos locales
    - --skip-show-database      # Previene enumeración de esquemas
  # -----------------------------------
  environment:
    MYSQL_ROOT_PASSWORD: root_secret
    MYSQL_DATABASE: extagram
    MYSQL_ALLOW_EMPTY_PASSWORD: "no" # Bloquea inicios sin credenciales
  # ... (restante configuración de volúmenes y red)
```

## 4. Argumentación y Justificación Técnica

- **`--local-infile=0`**: Es la medida de contención más crítica. Por defecto, MySQL permite la instrucción `LOAD DATA LOCAL INFILE`, la cual puede ser abusada por un atacante que haya logrado una Inyección SQL para leer archivos sensibles del servidor (`/etc/passwd`, certificados, código fuente) y exfiltrarlos a través de la respuesta de la base de datos. Deshabilitarlo cierra este vector de ataque.

- **`--skip-show-database`**: Evita que usuarios sin privilegios globales puedan listar las bases de datos existentes. Esto complica la fase de reconocimiento (enumeración) de un atacante, ocultando esquemas del sistema o bases de datos adyacentes.

- **`MYSQL_ALLOW_EMPTY_PASSWORD: "no"`**: Fuerza a nivel de orquestación que el contenedor rechace arrancar si la variable de entorno de la contraseña está vacía. Es una medida preventiva contra fallos de despliegue que podrían dejar la BBDD expuesta sin autenticación.

## 5. Pruebas de Validación

- Se verificó el reinicio correcto del contenedor mediante:
  ```bash
  sudo docker-compose up -d s7-db
  ```

- Se confirmó que el healthcheck del contenedor pasa satisfactoriamente, validando que el motor interno sigue aceptando conexiones con las credenciales existentes (`root_secret`).

- La aplicación web Extagram continúa realizando operaciones de lectura y escritura sin disrupción.

- Se comprobó que las opciones están activas dentro del contenedor:
  ```bash
  sudo docker exec s7-db mysql -uroot -proot_secret -e "SHOW VARIABLES LIKE 'local_infile';"
  # Debe devolver: local_infile | OFF
  ```

## 6. Conclusión
El servicio de base de datos queda reforzado contra técnicas de explotación comunes, reduciendo la superficie de ataque sin afectar la funcionalidad del sistema.
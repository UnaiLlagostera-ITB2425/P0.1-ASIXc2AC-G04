### 1. Configuración de la Base de Datos

**Objetivo:** Crear la estructura de datos necesaria (`extagram_db`), el usuario administrador (`extagram_admin`) y la tabla `posts` requerida por el código fuente.

#### 1.1 Acceso a MySQL

En Ubuntu 24.04, el usuario root de MySQL utiliza autenticación por *socket* por defecto.

* **Comando:**

```bash
sudo mysql

```

#### 1.2 Creación de la Base de Datos

Se genera el contenedor lógico para la información de la aplicación.

* **Consulta SQL:**

```sql
CREATE DATABASE extagram_db;

```

#### 1.3 Creación de Usuario y Asignación de Credenciales

Por motivos de seguridad, se crea un usuario específico para la aplicación, evitando el uso de credenciales `root` en el entorno web.

* **Consulta SQL:**

```sql
CREATE USER 'extagram_admin'@'localhost' IDENTIFIED BY '#######';

```

#### 1.4 Asignación de Privilegios

Se limita el acceso del usuario únicamente a la base de datos del proyecto.

* **Consultas SQL:**

```sql
GRANT ALL PRIVILEGES ON extagram_db.* TO 'extagram_admin'@'localhost';
FLUSH PRIVILEGES;

```

> **Nota:** Se ejecuta `FLUSH PRIVILEGES` para recargar la tabla de permisos inmediatamente y hacer efectivos los cambios.

#### 1.5 Creación del Esquema de Tablas

Se define la estructura de la tabla basándose en los requerimientos del código PHP (campos de texto para el contenido y la URL de la imagen).

* **Seleccionar la DB:**

```sql
USE extagram_db;

```

* **Crear la tabla:**

```sql
CREATE TABLE posts (
    post TEXT,
    photourl TEXT
);

```

#### 1.6 Verificación

Se comprueba que la estructura se ha creado correctamente.

* **Comando:**

```sql
SHOW TABLES;
DESCRIBE posts;

```

* **Salida de la consola:**

```sql
EXIT;

```

---

### 3. Securización de Credenciales e Integración

Para evitar exponer las credenciales de la base de datos en el repositorio de código, se implementa una estrategia de separación de configuración mediante un archivo excluido del control de versiones.

#### 3.1 Configuración de Git (.gitignore)

Se configura Git para ignorar el archivo de claves, evitando que se suba accidentalmente al repositorio público.

* **Acción:** Crear/Editar el archivo `.gitignore` en la raíz del proyecto añadiendo la siguiente línea:

```text
db_config.php

```

#### 3.2 Creación del Archivo de Configuración

Se crea el archivo que contendrá las variables de conexión reales. Este archivo debe existir tanto en el servidor de producción como en el entorno local, pero nunca en GitHub.

* **Archivo:** `db_config.php`

```php
<?php
$servername = "localhost";
$username = "extagram_admin";
$password = "#######"; // Contraseña definida en MySQL
$dbname = "extagram_db";
?>

```

#### 3.3 Adaptación del Código Fuente

Se modifican los archivos principales de la aplicación para importar las credenciales en lugar de tenerlas *hardcoded*.

* **Archivos modificados:** `extagram.php` (o index.php) y `upload.php`.
* **Código implementado:**

```php
// Se sustituye la conexión directa por la importación segura
require_once 'db_config.php';
$db = new mysqli($servername, $username, $password, $dbname);

```

> Con esta implementación, se garantiza que las credenciales sensibles permanecen únicamente en el servidor, cumpliendo con las buenas prácticas de desarrollo seguro.


---

### 1. Instalación del Servidor de Base de Datos

Para el almacenamiento de datos persistentes se ha seleccionado MySQL Server.

* **Instalación del paquete:**

```bash
sudo apt install mysql-server -y

```

* **Asegurar el arranque:**

```bash
sudo systemctl enable mysql

```

> **Nota:** Se habilita el servicio mediante `systemctl enable` para garantizar que el motor de base de datos arranque automáticamente si el servidor se reinicia.

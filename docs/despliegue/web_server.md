# Despliegue del servidor Web
---

## Información del hardware y networok

**Ip del Servidor Web (Apache + PHP):** 192.168.40.10/24

**Hostname del servidor web:** W-N04

---

## Instalación de Apache y PHP en el servidor web

Ejecuta los siguientes comandos para instalar Apache2, PHP y los módulos necesarios para conectar con MySQL:

```bash
sudo apt update
sudo apt install apache2 php libapache2-mod-php php-mysql -y
```

Para verificar que PHP funciona correctamente, crea el archivo info.php y añadele algun texto formato html:

```bash
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
```

Luego abre en el navegador:

`http://52.21.109.232`

<div align="center">
  <img src="../../media/despl_web.png" alt="Acceso web">
</div>

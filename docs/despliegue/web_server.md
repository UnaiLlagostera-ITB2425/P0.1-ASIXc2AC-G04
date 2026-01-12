# Tarea 5: Despliegue del servidor Web
---

## Información del hardware y network:

El hardware utilizado fue preparado mediante isard con las siguientes características:

<div align="center">
  <img src="../../media/hard_web.png" alt="Hardware">
</div>

<div align="center">
  <img src="../../media/hard_web2.png" alt="Hardware2">
</div>

**Ip del Servidor Web (Apache + PHP):** 192.168.40.10/24

> Netplan:

```bash
network:
  ethernets:
    enp1s0:
      addresses: [192.168.40.10/24]
      dhcp4: no
      routes:
        - to: default
          via: 192.168.40.1
      nameservers:
        addresses: [8.8.8.8]
```

**Hostname del servidor web:** W-N04

>**Usuario:** bchecker
**Contraseña:** bchecker121

***

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

`http://192.168.40.10/info.php`

<div align="center">
  <img src="../../media/despl_web.png" alt="Acceso web">
</div>
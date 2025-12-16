# Infraestructura del Proyecto: Sprint 1

## 1. Descripción General
Para el primer sprint, el objetivo es establecer un **Producto Mínimo Viable (MVP)** utilizando una arquitectura **monolítica**. [cite_start]En lugar de separar los servicios en contenedores o múltiples servidores, todos los componentes necesarios para la aplicación *Extagram* se desplegarán y configurarán dentro de una **única instancia en AWS (EC2)**.

Este enfoque permite centrarse en la configuración base de los servicios, la validación del código PHP y la conectividad de la base de datos antes de escalar hacia una arquitectura de microservicios en los siguientes sprints.

## 2. Especificaciones de la Infraestructura (AWS)

Se aprovisionará una única máquina virtual (Instancia EC2) que alojará la pila tecnológica completa (LAMP/LEMP).

* **Proveedor Cloud:** AWS (Amazon Web Services).
* **Recurso de Cómputo:** Instancia EC2 (p.ej., t2.micro o t3.micro para capa gratuita).
* **Sistema Operativo:** Linux (Ubuntu Server o Amazon Linux recomendados).

### Componentes de Software (Instalados en la instancia)
1.  [cite_start]**Servidor Web:** NGINX (Recomendado por la arquitectura final del proyecto) o Apache[cite: 60, 82]. Actuará como punto de entrada para las peticiones HTTP.
2.  **Intérprete Backend:** PHP (con módulos `php-fpm` y `php-mysql`). [cite_start]Procesará los scripts `extagram.php` y `upload.php`[cite: 62, 63].
3.  **Base de Datos:** MySQL Server. Instalado localmente en la misma instancia (`localhost`). [cite_start]Almacenará la tabla `posts` y, opcionalmente, las imágenes como BLOBs[cite: 68, 69].
4.  **Almacenamiento:** Sistema de archivos local (EBS adjunto a la instancia) para persistencia de la BBDD y la carpeta `uploads/`.

## 3. Esquema de Red y Seguridad

Al tratarse de un despliegue en una sola máquina, la topología de red es simplificada. No se requiere balanceo de carga ni subredes privadas complejas en esta fase.

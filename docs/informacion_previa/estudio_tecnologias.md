# Estudio de Tecnologías

**Proyecto:** Extagram

**Fecha:** 16/12/2025

**Referencia:** RA1 - Identificación de necesidades y estudio previo.

---

## 1. Introducción y Objetivos

El propósito de este documento es analizar, comparar y justificar la pila tecnológica (*Tech Stack*) seleccionada para el despliegue de la aplicación Extagram.

Dado que el proyecto se desarrolla bajo metodologías ágiles (Scrum), las decisiones tecnológicas priorizan dos factores clave:

1. **Viabilidad y Entrega (Time-to-Market):** Asegurar que el Sprint 1 sea funcional utilizando tecnologías robustas.
2. **Profesionalización:** Simular un entorno de producción real que prepare al equipo para el mercado laboral, priorizando estándares de la industria sobre soluciones académicas.

---

## 2. Infraestructura de Alojamiento (Hosting)

Para alojar la aplicación, se han evaluado tres escenarios posibles, descartando soluciones locales (VirtualBox) por inviabilidad colaborativa y comparando entornos de nube:

### 2.1. Opciones Evaluadas

* **IsardVDI (Cloud Privada del Centro):** Entorno de virtualización académico.
* *Pros:* Entorno conocido, gratuito y gestionado por la escuela.
* *Contras:* Acceso externo limitado (a menudo requiere VPN), no dispone de herramientas de gestión de red avanzadas (Security Groups definidos por usuario) y es un entorno puramente educativo, no productivo.


* **Cloud AWS (Amazon Web Services):** Infraestructura como Servicio (IaaS) líder del mercado.
* *Pros:* Estándar global, control total de redes (VPC, Subnets, Security Groups), IPs públicas reales y documentación exhaustiva.
* *Contras:* Curva de aprendizaje inicial.

### 2.2. Decisión Final: AWS EC2

Se ha seleccionado **AWS EC2** como plataforma de despliegue.

**Justificación:**

1. **Simulación del Entorno Laboral:** Las empresas utilizan AWS, Azure o GCP. Desplegar en AWS aporta un valor curricular añadido y cumple con el criterio de "Identificación de necesidades del sector productivo" (RA1).
2. **Gestión de Redes Real:** AWS nos obliga a configurar firewalls reales (Security Groups) y pares de claves SSH, competencias críticas que en IsardVDI suelen venir preconfiguradas.
3. **Sistema Operativo:** Usaremos **Ubuntu Server 24.04 LTS**. Es la distribución Linux más documentada para despliegues web, asegurando soporte a largo plazo.

---

## 3. Servidor Web y Balanceador de Carga

Para la arquitectura del proyecto, que evolucionará de Monolito (Sprint 1) a Microservicios (Sprint 2/3), se requiere un software capaz de servir contenido y balancear carga.

### 3.1. Estrategia de Homogeneización

Se ha evaluado la posibilidad de una arquitectura híbrida (Nginx para Proxy + Apache para Backend) frente a una arquitectura homogénea.

### 3.2. Decisión Final: Apache HTTP Server (Full Stack)

Se ha decidido utilizar **Apache HTTP Server** para **todos los roles** del proyecto:

* Sprint 1: Servidor Web Monolítico.
* Sprint 3: Balanceador de Carga (Proxy Inverso) y Backends.

**Justificación Técnica y Estratégica:**

1. **Reducción de la Carga Cognitiva:** Al utilizar un único software, el equipo solo necesita dominar una sintaxis de configuración (`.conf` y `.htaccess`). Introducir Nginx solo para el balanceo añadiría complejidad innecesaria sin una mejora de rendimiento crítica para este volumen de tráfico.
2. **Capacidades de Proxy:** Apache dispone de módulos robustos (`mod_proxy`, `mod_proxy_balancer`, `mod_lbmethod_byrequests`) que permiten realizar el balanceo de carga requerido en el Sprint 3 con total eficacia.
3. **Mantenimiento Simplificado:** Facilita la gestión de logs, actualizaciones y seguridad, ya que todos los nodos del clúster comparten la misma tecnología base.

---

## 4. Sistema Gestor de Base de Datos (SGBD)

La aplicación requiere una base de datos relacional para almacenar usuarios, posts y referencias a imágenes.

### 4.1. Opciones Evaluadas

* **MySQL:** El motor de base de datos relacional open-source más popular.
* **PostgreSQL:** Motor relacional-objeto avanzado.
* **MariaDB:** Fork compatible de MySQL.

### 4.2. Decisión Final: MySQL 8.0 Community Server

Se ha seleccionado **MySQL** como motor de base de datos.

**Justificación:**

1. **Conocimiento del Equipo:** El equipo posee experiencia gestionando usuarios y permisos en MySQL, lo que agiliza el desarrollo y reduce errores.
2. **Soporte de BLOBs:** Para el requisito de **Redundancia de Datos** (Sprint 3), MySQL ofrece una gestión eficiente del tipo de dato `LONGBLOB` para almacenar imágenes binarias directamente en la BBDD.
3. **Estándar LAMP:** La combinación Linux + Apache + MySQL + PHP es la arquitectura más documentada y probada de la web.

---

## 5. Lenguaje de Backend

### 5.1. Versión de PHP

El código fuente proporcionado está escrito en PHP. Se utilizará la versión **PHP 8.1 (o superior)** disponible en los repositorios de Ubuntu 24.04.

**Justificación:**

* Las versiones 7.x están obsoletas (EOL).
* PHP 8 introduce mejoras de rendimiento (JIT Compiler) y seguridad fundamentales para un servicio expuesto en internet pública.

---

## 6. Conclusión

La arquitectura tecnológica elegida es un **Stack LAMP homogéneo sobre AWS EC2**.

Esta decisión se basa en la **profesionalización** (uso de AWS en lugar de IsardVDI/Local) y la **consistencia** (uso exclusivo de Apache), permitiendo al equipo centrarse en la implementación de la arquitectura de contenedores y alta disponibilidad sin distracciones tecnológicas innecesarias.

---

## 7. Referencias Bibliográficas

Para la elaboración de este estudio se han consultado las siguientes fuentes:

1. **AWS Academy.** *Detalles de las instancias EC2 y Capa Gratuita.* [https://aws.amazon.com/es/ec2/pricing/](https://aws.amazon.com/es/ec2/pricing/)
2. **Apache Module mod_proxy_balancer.** *Apache HTTP Server Version 2.4 Documentation.* [https://httpd.apache.org/docs/2.4/mod/mod_proxy_balancer.html](https://httpd.apache.org/docs/2.4/mod/mod_proxy_balancer.html)
3. **MySQL Documentation.** *The BLOB and TEXT Types.* [https://dev.mysql.com/doc/refman/8.0/en/blob.html](https://dev.mysql.com/doc/refman/8.0/en/blob.html)
4. **PHP.net.** *Supported Versions and EOL.* [https://www.php.net/supported-versions.php](https://www.php.net/supported-versions.php)
# P0.1 & P0.2 - ASIXc2AC-G04 - Extagram

> **Módulo:** M0379 - Proyecto ASIXc
> **Actividad:** P0.1 (Despliegue) & P0.2 (Securización y Monitorización)
> **Centro:** Institut Tecnològic Barcelona

## Descripción del Proyecto

Este repositorio contiene la infraestructura como código (IaC), scripts de aprovisionamiento y documentación técnica para el despliegue de **Extagram**, una aplicación web de intercambio de imágenes.

El proyecto ha evolucionado en dos fases:

1. **Fase 1 (P0.1):** Diseño e implementación de una arquitectura de Alta Disponibilidad escalable con Docker (Microservicios).
2. **Fase 2 (P0.2):** Evolución hacia una infraestructura securizada (WAF, Firewall) y monitorizada (Logs centralizados).

---

## Gestión del Proyecto (Agile & ProofHub)

Utilizamos la metodología **Agile** para la gestión de tareas, coordinando el equipo a través de la plataforma **ProofHub**.

### Roles del Equipo

| Miembro del Equipo | Rol Asignado | Responsabilidades Principales |
| --- | --- | --- |
| **Samuel Moscoso** | Scrum Master | Facilitador, gestión de ProofHub, eliminación de bloqueos. |
| **Unai Llagostera** | Product Owner | Definición de requisitos, revisión de entregables (Docs). |
| **Todos los integrantes** | SysAdmin / Dev | Configuración de servidores, scripting y despliegue. |
| **Todos los integrantes** | SysAdmin / QA | Pruebas de carga, seguridad y documentación de pruebas. |

### Planificación de Sprints

| Sprint | Duración | Objetivo Principal |
| --- | --- | --- |
| **Sprint 1** | 15/12/25 - 19/01/26 | Investigación, diseño de arquitectura y configuración de entorno local (Git). |
| **Sprint 2** | 19/01/26 - 27/01/26 | Despliegue de servidores web (Nginx/PHP) y base de datos. |
| **Sprint 3** | 02/02/26 - 10/02/26 | Implementación de Balanceador, Persistencia y Resiliencia de Datos. |
| **Sprint 4** | 17/02/26 - 24/02/26 | **Securización:** Firewall, WAF, Hardening y VPN para nodo externo. |
| **Sprint 5** | 02/03/26 - 10/03/26 | **Monitorización:** Centralización de logs (Stack ELK/Grafana) y Stress Testing. |

---

## Actas

---

# Acta de Reunión #01: Kick-off y Sprint Planning**Proyecto:** Extagram

**Fecha:** 16/12/2025
**Hora de inicio:** 16:00
**Hora de finalización:** 17:30
**Lugar:** Aula 209
**Asistentes:**

* Unai Llagostera
* Samuel Moscoso
* Asier Barranco

---

## 1. Orden del Día

1. Lectura del enunciado del proyecto (P0.1) y análisis de requisitos.
2. **Sprint Planning:** Definición del alcance del Sprint 1.
3. **Estudio Tecnológico:** Debate y selección de infraestructura (AWS vs Local) y Software (Apache vs Nginx).
4. Creación del Backlog y asignación de tareas iniciales.

---

## 2. Desarrollo de la Reunión y Acuerdos

### 2.1. Análisis del ProyectoSe revisa la documentación del proyecto "Extagram". El equipo acuerda adoptar la metodología **Scrum**.

* **Objetivo Global:** Desplegar una aplicación PHP heredada evolucionando de un monolito a microservicios con Docker.
* **Objetivo Sprint 1:** Despliegue funcional del monolito LAMP en nube pública.

### 2.2. Decisiones Tecnológicas (Tech Stack)

Tras el debate (documentado en el *Estudio Tecnológico v1.3*), se toman las siguientes decisiones por unanimidad:

* **Infraestructura:** Se utilizará **AWS EC2** (Instancia `t3.micro`) con Ubuntu 24.04 LTS. Se descarta IsardVDI por falta de realismo profesional.
* **Servidor Web:** Estrategia **"Apache Full-Stack"**. Se usará Apache tanto para el monolito actual como para el balanceador futuro, simplificando la gestión.
* **Base de Datos:** MySQL 8.0.
* **Control de Costes:** Se aprueba el uso de `t3.micro` con configuración de SWAP (2GB) para optimizar el presupuesto de 50$.

### 2.3. Sprint Planning (Sprint 1)

Se definen las historias de usuario y tareas técnicas para alcanzar el MVP antes de la fecha límite. Se acuerda dividir el trabajo en:

1. Infraestructura (Cloud).
2. Configuración de Servidores (SysAdmin).
3. Despliegue de Código (DevOps).

---

## 3. Asignación de Tareas (Backlog)

Se han creado las siguientes tareas en el tablero del proyecto y se asignan responsables para ejecutar antes de la revisión del **Martes a las 16:45**:

| ID | Tarea | Responsable | Estado |
| --- | --- | --- | --- |
| **1.1** | Creación de Repositorio Git y estructura de directorios (`/docs`, `/src`). | *[Nombre]* | *Done* |
| **1.2** | Redacción del Estudio Tecnológico y Justificación de AWS. | *[Nombre]* | *In Progress* |
| **1.3** | **AWS:** Lanzamiento de instancia EC2 (VPC, Security Groups, KeyPairs). | *[Nombre]* | *In Progress* |
| **1.4** | **SysAdmin:** Scripting de `User Data` para instalación desatendida de LAMP. | *[Nombre]* | *To Do* |
| **1.5** | **BBDD:** Configuración de MySQL, creación de usuario y restauración del `.sql`. | *[Nombre]* | *To Do* |
| **1.6** | **Deploy:** Subida de código PHP vía Git/SFTP y configuración de VirtualHost. | *[Nombre]* | *To Do* |

---

## 4. Próximos Pasos

* **Revisión de Avance (Daily/Checkpoint):** Martes 17/12 a las 16:45.
* **Objetivo para la revisión:** Tener la instancia corriendo, con IP pública accesible, las tareas asignadas para cada uno empezadas y con ideas claras.

# Acta de Reunión #02: Sprint Review #01 & Sprint Planning #02

**Proyecto:** Extagram
**Fecha:** 19/01/2026
**Hora de inicio:** 16:15
**Hora de finalización:** 17:30
**Lugar:** Aula 209
**Asistentes:**

* Unai Llagostera
* Samuel Moscoso
* Asier Barranco

---

## 1. Orden del Día

1. **Sprint Review (S1):** Demostración del MVP, revisión de "Definition of Done" y análisis de mejoras implementadas (HTTPS/Seguridad).
2. **Retrospectiva Técnica:** Análisis de obstáculos encontrados en el despliegue monolítico.
3. **Sprint Planning (S2):** Definición del alcance para la evolución de la arquitectura (Desacoplamiento).
4. Actualización del Backlog y asignación de nuevas tareas.

---

## 2. Desarrollo de la Reunión y Acuerdos

### 2.1. Sprint Review: Cierre del Sprint 1

Se presenta el producto final del Sprint 1. El equipo valida que se han cumplido el **100% de las tareas planificadas**, y además se han realizado tareas que han surgido a posterior, como la migración ssl y la gestión de conexión.

| ID | Tarea | Responsable | Estado |
| --- | --- | --- | --- |
| **1.1** | Creación de Repositorio Git y estructura de directorios (`/docs`, `/src`). | *[Unai]* | *Done* |
| **1.2** | Creación de gestor de trabajo Proofhub. | *[Unai]* | *Done* |
| **1.3** | Esquema de infraestructura lógico. | *[Samuel]* | *Done* |
| **1.4** | Redacción del Estudio Tecnológico y Justificación de AWS. | *[Asier]* | *Done* |
| **1.5** | Análisis de código Fuente PHP. | *[Samuel/Unai]* | *Done* |
| **1.6** | **AWS:** Lanzamiento de instancia EC2 (VPC, Security Groups, KeyPairs). | *[Asier]* | *Done* |
| **1.7** | **BBDD:** Configuración de MySQL, creación de usuario y restauración del `.sql`. | *[Asier]* | *Done* |
| **1.8** | **Deploy:** Despliegue  de código Fuente PHP. | *[Samuel/Unai]* | *Done* |
| **1.9** | **Seguridad:** Ocultar contraseñas en la subida del repositorio. | *[Asier]* | *Done* |
| **1.10** | **Migración:** Cambiar al puerto 443 con certificado SSL. | *[Unai]* | *Done* |
| **1.11** | Prueba Funcional 1. | *[Samuel/Unai/Asier]* | *Done* |

* **Estado del Producto:** El monolito está desplegado y totalmente operativo en AWS.
* **Over-delivery (Valor añadido):** Se han implementado mejoras no contempladas inicialmente pero críticas para la calidad del proyecto:
* **Seguridad SSL:** Implementación de certificado autofirmado y apertura de puerto 443 (HTTPS).
* **Gestión de Secretos:** Extracción de credenciales a `db_config.php` y exclusión en `.gitignore` para evitar fugas de seguridad.
* **Demostración:** Se verifica en vivo el flujo completo: *Carga Web (HTTPS) -> Posteo -> Subida Imagen -> Persistencia en BBDD*.

### 2.2. Retrospectiva Técnica

No hemos encontrado problemas de ningun carácter durante el sprint, hemos llevado todo al día con suficiente tiempo. Las reuniones diarias nos
han ayudado a la organización y reparto de tareas, y hemos conseguido implementar con éxito un MVP inicial

### 2.3. Sprint Planning (Sprint 2)

Se acuerda abandonar la arquitectura monolítica y migrar hacia una infraestructura **contenerizada basada en Docker**. El objetivo es la escalabilidad horizontal y la separación de responsabilidades.

* **Objetivo Sprint 2:** Dockerizar la aplicación completa, separando el monolito en servicios independientes orquestados.
* **Estrategia Técnica (Stack Apache):**
* Se decide unificar la tecnología de servidor web utilizando **Apache** para todos los nodos (Balanceador, Estáticos y Backend).
* **Arquitectura de Servicios:**
* **S1 (Gateway):** Apache como Proxy Inverso y Balanceador de Carga.
* **S2, S3, S4 (Backend):** Contenedores réplica con el código PHP (Escalabilidad).
* **S5 (Imágenes):** Servidor dedicado a servir el contenido de `uploads/`.
* **S6 (Assets):** Servidor dedicado optimizado para CSS, JS y SVG.
* **S7 (Datos):** Contenedor MySQL persistente.
* **Red y Almacenamiento:** Implementación de Docker Networks internas y Volúmenes para persistencia de datos y uploads compartidos.

---

## 3. Asignación de Tareas (Backlog Sprint 2)

Se han definido las tareas en el tablero Kanban y asignado a los responsables según las iniciales (AB, UL, MS).

| ID | Tarea / Servicio | Descripción | Responsable | Estado |
| --- | --- | --- | --- | --- |
| **2.1** | **Diseño de Red (Avanzado)** | Crear esquema lógico de red y exportar imagen para documentación. | **Samuel Moscoso** | *Doing* |
| **2.2** | **Docker Network & Volúmenes** | Creación de la red interna y volúmenes persistentes (`mysql_data`, `uploads`). | **Unai Llagostera** | *Doing* |
| **2.3** | **Documentación Sprint 2** | Documentar en Markdown, capturas y actualizar repositorio. | **Todos** (AB, UL, MS) | *Doing* |
| **2.4** | **S1: Proxy Inverso & Balanceador** | Configurar Apache como Load Balancer y Gateway de entrada. | **Samuel Moscoso** | *To Do* |
| **2.5** | **S7: Docker Servicio BBDD** | Dockerfile/Compose para MySQL e implementación de esquema inicial. | **Asier Barranco** | *To Do* |
| **2.6** | **S6: Docker Frontend Estáticos** | Configurar Apache optimizado para servir Assets (CSS/SVG). | **Unai Llagostera** | *To Do* |
| **2.7** | **S5: Docker Frontend Imágenes** | Configurar Apache para servir contenido estático subido por usuarios (`uploads`). | **Asier B. & Unai L.** | *To Do* |
| **2.8** | **S2, S3, S4: Backend Apps** | Crear imágenes PHP/Apache. Asegurar montaje de volumen de escritura. | **Asier B. & Unai L.** | *To Do* |

---

## 4. Próximos Pasos

* **Inicio de trabajos técnicos:** Configuración inmediata del `docker-compose.yml` base y creación de la red.
* **Próximo Checkpoint:** Verificar que los contenedores se levantan y tienen comunicación (Ping entre S1 y S7).
* **Fecha de entrega Sprint 2:** 27 de Enero.

# Acta de Reunión #03: Sprint Review #02 & Sprint Planning #03

**Proyecto:** Extagram
**Fecha:** 02/02/2026
**Hora de inicio:** 15:00
**Hora de finalización:** 16:30
**Lugar:** Aula 209
**Asistentes:**

* Unai Llagostera
* Samuel Moscoso
* Asier Barranco

---

## 1. Orden del Día

1. **Sprint Review (S2):** Revisión de la arquitectura de microservicios, validación del despliegue en Docker y funcionamiento de los scripts de automatización.
2. **Retrospectiva Técnica:** Análisis de incidencias críticas (Rendimiento AWS, HTTPS, Caché Docker).
3. **Sprint Planning (S3):** Definición preliminar de objetivos para la fase final del proyecto.
4. **Backlog:** (Pendiente de definición).

---

## 2. Desarrollo de la Reunión y Acuerdos

### 2.1. Sprint Review: Cierre del Sprint 2

El equipo presenta la arquitectura completamente migrada a microservicios. Se ha pasado de una única instancia monolítica a una orquestación de **7 contenedores** interconectados. Se valida el cumplimiento del **100% de las tareas** del tablero del Proofhub.

Se destaca la implementación de scripts de ciclo de vida (`boot.sh` y `clean.sh`) que han reducido el tiempo de despliegue de horas a segundos.

**Tabla de Tareas Completadas (Sprint 2):**

| ID | Tarea / Módulo | Descripción | Responsables | Estado |
| --- | --- | --- | --- | --- |
| **2.1** | **Diseño de Red (Avanzado)** | Esquema lógico final de la red Docker (`app-network`) y flujo de datos. | *[Samuel]* | *Done* |
| **2.2** | **Docker Network & Volúmenes** | Implementación de la red puente y volúmenes persistentes para evitar pérdida de datos. | *[Asier / Unai]* | *Done* |
| **2.3** | **Docker Servicio BBDD (S7)** | Configuración de MySQL 8.0, usuario dedicado y healthcheck. | *[Asier]* | *Done* |
| **2.4** | **Proxy Inverso & LB (S1)** | Configuración de Apache como Gateway de entrada y terminación SSL. | *[Asier]* | *Done* |
| **2.5** | **HTTPS & Seguridad** | Generación de certificados y forzado de tráfico seguro (Redirección 301). | *[Asier / Unai]* | *Done* |
| **2.6** | **Backend Apps (S2, S3)** | Despliegue de los nodos de aplicación PHP conectados a la BBDD. | *[Asier]* | *Done* |
| **2.7** | **Frontend Imágenes (S4)** | Servicio dedicado a la ingesta (upload) y almacenamiento físico de fotos. | *[Unai / Samuel]* | *Done* |
| **2.8** | **Frontend Estáticos (S6)** | Servidor CDN ligero (Alpine) para servir assets CSS/JS. | *[Unai]* | *Done* |
| **2.9** | **Servicio de Backup (S5)** | Implementación de worker efímero para dumps de seguridad (Tarea derivada de Backend Apps). | *[Asier]* | *Done* |
| **2.10** | **Scripting (Automatización)** | Creación de `boot.sh` y `clean.sh` para gestión de orquestación. | *[Asier]* | *Done* |

* **Estado del Producto:** La aplicación "Extagram" funciona en arquitectura distribuida. El balanceador reparte carga entre S2 y S3, y los datos persisten tras reinicios.
* **Demostración:** Se ejecuta `./boot.sh`, se espera el healthcheck de MySQL y se accede vía navegador demostrando la subida de imagen y su replicación visual.

### 2.2. Retrospectiva Técnica (Problemas y Soluciones)

Durante este sprint, la complejidad técnica aumentó considerablemente respecto al monolito. Nos enfrentamos a tres bloqueos principales:

1. **Colapso de Recursos en AWS (RAM):**

* *Problema:* Al intentar levantar los 7 contenedores simultáneamente, la instancia `t2.micro` (1GB RAM) se congelaba por falta de memoria, matando procesos de MySQL.
* *Solución:* Se creó un archivo de intercambio (**SWAP**) de 2GB en el sistema host y se escaló la instancia a una **`t3.small`** (2GB RAM + 2 vCPU) para soportar la carga de orquestación.

2. **Infierno de Redirecciones HTTPS:**

* *Problema:* La migración de HTTP a HTTPS detrás de un Proxy Inverso generó bucles de redirección infinitos y errores de certificado al comunicarse los contenedores internos.
* *Solución:* Reescritura completa del `extagram.conf` en el contenedor S1 y ajuste de cabeceras para manejar correctamente el SSL Offloading.

3. **Persistencia de "Zombis" y Caché Docker:**

* *Problema:* Docker mantenía volúmenes huérfanos y capas de imágenes antiguas en caché, lo que causaba que el código nuevo no se reflejara o que la base de datos fallara al iniciar por conflictos de archivos antiguos.
* *Solución:* Desarrollo del script `clean.sh` con comandos agresivos de limpieza (`docker system prune -a --volumes`) para garantizar un entorno estéril en cada despliegue.

---

### 2.3. Sprint Planning (Sprint 3)

El equipo define el alcance del último sprint, enfocado en la **Resiliencia, Persistencia de Datos y Documentación**. Se prioriza asegurar que el sistema no pierda datos bajo ninguna circunstancia y que sea capaz de recuperarse automáticamente.

---

## 3. Asignación de Tareas (Backlog Sprint 3)

*(Planificación basada en tablero Proofhub)*

| ID | Tarea / Módulo | Descripción | Responsables | Estado |
| --- | --- | --- | --- | --- |
| **3.1** | **Reconfiguración de contenedores** | Ajustes en `docker-compose` para el correcto funcionamiento de backups y multiservicio. | *[Asier]* | *To Do* |
| **3.2** | **Redundancia de Datos (BLOBs)** | Modificar PHP (S2/S4) para guardar imagen en disco y simultáneamente en campo BLOB de MySQL como backup. | *[Asier / Unai]* | *To Do* |
| **3.3** | **Failover System** | Implementar lógica en `index.php`: `try` (cargar de disco S5) `catch` (cargar BLOB de BBDD). | *[Asier / Unai]* | *To Do* |
| **3.4** | **Automatización de backups** | Scripting para realizar volcados automáticos de la BBDD y carpetas críticas. | *[Asier]* | *To Do* |
| **3.5** | **Pruebas y fallas de servicios** | Simulacros de desastre: Caída de S2, borrado de volumen uploads y verificación de carga desde BBDD. | *[Todos]* | *To Do* |
| **3.6** | **Creación de manuales** | Guía de despliegue desde cero (`docker-compose up`) y guía de mantenimiento. | *[Unai]* | *To Do* |
| **3.7** | **Documentación Sprint 3** | Actualización de repositorio, subida de capturas y cierre de documentación MD. | *[Todos]* | *To Do* |

---

---

# Acta de Reunión #04: Sprint Review #03 & Sprint Planning #04

**Proyecto:** Extagram
**Fecha:** 17/02/2026
**Hora de inicio:** 10:00
**Hora de finalización:** 11:45
**Lugar:** Aula 209
**Asistentes:**

* Unai Llagostera
* Samuel Moscoso
* Asier Barranco

---

## 1. Orden del Día

1. **Sprint Review (S3):** Validación de la redundancia de datos (BLOBs), sistema de Failover y automatización de backups.
2. **Retrospectiva Técnica (S3):** Análisis de las pruebas de fallas simuladas y la solución de bloqueos de Docker.
3. **Presentación de Fase P0.2:** Lectura de nuevos requisitos de seguridad y monitorización.
4. **Sprint Planning (S4):** Definición de tareas de seguridad perimetral y de aplicación.

---

## 2. Desarrollo de la Reunión y Acuerdos

### 2.1. Sprint Review: Validación del Sprint 3

El equipo presenta el sistema base finalizado. Se ha cumplido el objetivo de crear una aplicación resiliente en cuanto a datos. Las pruebas de carga y fallas han demostrado que el sistema de doble almacenamiento (Disco + BLOB) funciona correctamente.

**Hitos alcanzados en este Sprint:**

* **Redundancia Híbrida:** Las imágenes viven en sistema de ficheros y BBDD simultáneamente.
* **Auto-Recuperación:** Failover automático a BBDD si falla el almacenamiento físico.
* **Documentación:** Manuales de administrador y usuario entregados.

**Tabla de Tareas Completadas (Sprint 3):**

| ID | Tarea | Resultado | Responsables | Estado |
| --- | --- | --- | --- | --- |
| **3.1** | **Reconfiguración de contenedores** | Infraestructura optimizada para soportar bind-mounts. | *[Asier]* | *Done* |
| **3.2** | **Redundancia de Datos (BLOBs)** | Backend reescrito para inyectar binarios en MySQL. | *[Asier / Unai]* | *Done* |
| **3.3** | **Failover System** | Frontend conmuta a BBDD en caso de error 404. | *[Asier / Unai]* | *Done* |
| **3.4** | **Automatización de backups** | Scripts `backup.sh` y `restore.sh` funcionales. | *[Asier]* | *Done* |
| **3.5** | **Pruebas y fallas de servicios** | Recuperación exitosa tras borrado masivo ("Opción Nuclear"). | *[Todos]* | *Done* |
| **3.6** | **Creación de manuales** | Entregados manuales técnicos y de usuario. | *[Unai]* | *Done* |
| **3.7** | **Documentación Sprint 3** | Repositorio actualizado. | *[Todos]* | *Done* |

### 2.2. Retrospectiva Técnica S3

* **Lección Aprendida:** La sincronización de estados entre Host y Contenedores debe hacerse siempre mediante *Bind Mounts* para evitar latencias en la restauración de backups.
* **Procedimiento:** Se ha estandarizado el reinicio del daemon de Docker para solucionar errores de *Permission Denied*.

### 2.3. Sprint Planning (Sprint 4 - Seguridad)

Se inicia la **Fase P0.2**. El objetivo de este sprint es blindar la infraestructura desplegada. Actualmente, la aplicación es funcional pero vulnerable. Además, se identifica que el nodo S5 (imágenes) deberá migrarse a una red externa conectada por VPN.

**Objetivos del Sprint 4:**

1. **Seguridad Perimetral (Firewall):** Implementar un firewall delante del Proxy S1 para filtrar tráfico no deseado.
2. **Seguridad de Aplicación (WAF):** Desplegar un Web Application Firewall (ModSecurity o similar) para protegerse de ataques OWASP (SQLi, XSS).
3. **Hardening de Sistemas:** Bastionado de los sistemas operativos (Ubuntu/Alpine) y del motor de base de datos MySQL.
4. **Conectividad Segura (VPN):** Configuración de túnel VPN para conectar el nodo S5, que ahora se ubicará en una red externa.

---

## 3. Asignación de Tareas (Backlog Sprint 4)

Se definen las tareas de seguridad en el tablero Proofhub.

| ID | Tarea / Módulo | Descripción | Responsables | Estado |
| --- | --- | --- | --- | --- |
| **4.1** | **Diseño de Seguridad** | Definición de reglas de firewall y política de WAF. | *[Samuel]* | *To Do* |
| **4.2** | **Implementación Firewall** | Configuración de Iptables/UFW o contenedor Firewall ante S1. | *[Unai]* | *To Do* |
| **4.3** | **Despliegue WAF (S1)** | Instalación y configuración de ModSecurity en el Apache Gateway. | *[Asier]* | *To Do* |
| **4.4** | **Hardening BBDD (S7)** | Auditoría de seguridad MySQL, cifrado en reposo y gestión de usuarios. | *[Asier]* | *To Do* |
| **4.5** | **Hardening OS & Web** | Deshabilitar versiones PHP inseguras, ocultar cabeceras y asegurar SSH. | *[Asier]* | *To Do* |
| **4.6** | **Integración VPN (S5)** | Configuración de túnel VPN (Wireguard/OpenVPN) para conectar el nodo externo S5. | *[Samuel]* | *To Do* |
| **4.7** | **Pruebas de Penetración** | Validación de ataques SQLi y XSS contra el WAF. | *[Todos]* | *To Do* |

---

## 4. Próximos Pasos

* **Inicio Sprint 4:** Inmediato (17/02).
* **Foco Técnico:** Investigar configuración de ModSecurity sobre imagen Docker `httpd:2.4` y despliegue de VPN.
* **Fecha de entrega Sprint 4:** 24 de Febrero.
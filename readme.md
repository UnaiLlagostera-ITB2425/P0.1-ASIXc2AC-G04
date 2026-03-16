# P0.1 & P0.2 - ASIXc2AC-G04 - Extagram

> **Módulo:** M0379 - Proyecto ASIXc
> **Actividad:** P0.1 (Despliegue) & P0.2 (Securización y Monitorización)
> **Centro:** Institut Tecnològic Barcelona

## Descripción del Proyecto

Este repositorio contiene la infraestructura como código (IaC), scripts de aprovisionamiento y documentación técnica para el despliegue de **Extagram**, una aplicación web de intercambio de imágenes.

El proyecto ha evolucionado en dos fases:

1. **Fase 1 (P0.1):** Diseño e implementación de una arquitectura de Alta Disponibilidad escalable con Docker (Microservicios).
2. **Fase 2 (P0.2):** Evolución hacia una infraestructura securizada (WAF, Firewall) y monitorizada (Logs centralizados).

## Indice

* **configuration**
  * [apache2.md](docs/configuracion/apache2.md)
  * [mysql.md](docs/configuracion/mysql.md)

* **despliegue**
  * [github_workflow.md](docs/despliegue/github_workflow.md)
  * [mysql_des.md](docs/despliegue/mysql_des.md)
  * [web_server.md](docs/despliegue/web_server.md)

* **estructura_final**
  * [arquitectura_final.md](docs/estructura_final/arquitectura_final.md)

* **información_previa**
  * [creacion_github.md](docs/informacion_previa/creacion_github.md)
  * [estudio_tecnologias.md](docs/informacion_previa/estudio_tecnologias.md)
  * [infraestructura_inicial.md](docs/informacion_previa/infraestructura_inicial.md)

* **manuales**
  * [manual_administrador.md](docs/manuales/manual_administrador.md)
  * [manual_usuario.md](docs/manuales/manual_usuario.md)

* **securizacion**
  * [centralizacion_logs.md](docs/securizacion/centralizacion_logs.md)
  * [centralizacion_logs_dashboard.md](docs/securizacion/centralizacion_logs_dashboard.mc)
  * [firewall_host.md](docs/securizacion/firewall_host.md)
  * [hardening_bd.md](docs/securizacion/hardening_bd.md)

* [Contratiempos.md](docs/securizacion/Contratiempos.md)

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
| **Sprint 5** | 02/03/26 - 09/03/26 | **Monitorización:** Centralización de logs (Stack ELK/Grafana) y Stress Testing. |

---

## Actas

---

# Acta de Reunión #01: Kick-off y Sprint Planning

**Proyecto:** Extagram
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

### 2.1. Análisis del Proyecto
Se revisa la documentación del proyecto "Extagram". El equipo acuerda adoptar la metodología **Scrum**.
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
| **1.1** | Creación de Repositorio Git y estructura de directorios (`/docs`, `/src`). | *[Unai]* | *Done* |
| **1.2** | Redacción del Estudio Tecnológico y Justificación de AWS. | *[Asier]* | *In Progress* |
| **1.3** | **AWS:** Lanzamiento de instancia EC2 (VPC, Security Groups, KeyPairs). | *[Asier]* | *In Progress* |
| **1.4** | **SysAdmin:** Scripting de `User Data` para instalación desatendida de LAMP. | *[Samuel]* | *To Do* |
| **1.5** | **BBDD:** Configuración de MySQL, creación de usuario y restauración del `.sql`. | *[Asier]* | *To Do* |
| **1.6** | **Deploy:** Subida de código PHP vía Git/SFTP y configuración de VirtualHost. | *[Samuel]* | *To Do* |

---

# Acta de Reunión #02: Sprint Review #01 & Sprint Planning #02

**Proyecto:** Extagram
**Fecha:** 19/01/2026
**Hora de inicio:** 16:15
**Hora de finalización:** 17:45
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
| **1.6** | **AWS:** Lanzamiento de instancia EC2. | *[Asier]* | *Done* |
| **1.7** | **BBDD:** Configuración de MySQL y restauración del `.sql`. | *[Asier]* | *Done* |
| **1.8** | **Deploy:** Despliegue de código Fuente PHP. | *[Samuel/Unai]* | *Done* |
| **1.9** | **Seguridad:** Ocultar contraseñas en la subida del repositorio. | *[Asier]* | *Done* |
| **1.10** | **Migración:** Cambiar al puerto 443 con certificado SSL. | *[Unai]* | *Done* |
| **1.11** | Prueba Funcional 1. | *[Samuel/Unai/Asier]* | *Done* |

* **Estado del Producto:** El monolito está desplegado y totalmente operativo en AWS.

### 2.3. Sprint Planning (Sprint 2)
Se acuerda abandonar la arquitectura monolítica y migrar hacia una infraestructura **contenerizada basada en Docker**. 
* **Arquitectura de Servicios:**
  * **S1 (Gateway):** Apache como Proxy Inverso y Balanceador de Carga.
  * **S2, S3, S4 (Backend):** Contenedores réplica con el código PHP (Escalabilidad).
  * **S5 (Imágenes):** Servidor dedicado a servir el contenido de `uploads/`.
  * **S6 (Assets):** Servidor dedicado optimizado para CSS, JS y SVG.
  * **S7 (Datos):** Contenedor MySQL persistente.

---

# Acta de Reunión #03: Sprint Review #02 & Sprint Planning #03

**Proyecto:** Extagram
**Fecha:** 02/02/2026
**Hora de inicio:** 15:00
**Hora de finalización:** 15:30
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

---

## 2. Desarrollo de la Reunión y Acuerdos

### 2.1. Sprint Review: Cierre del Sprint 2
El equipo presenta la arquitectura completamente migrada a microservicios. Se ha pasado de una única instancia monolítica a una orquestación de **7 contenedores** interconectados. Se valida el cumplimiento del **100% de las tareas**.

**Tabla de Tareas Completadas (Sprint 2):**

| ID | Tarea / Módulo | Descripción | Responsables | Estado |
| --- | --- | --- | --- | --- |
| **2.1** | **Diseño de Red (Avanzado)** | Esquema lógico final de la red Docker (`app-network`). | *[Samuel]* | *Done* |
| **2.2** | **Docker Network & Volúmenes** | Implementación de red puente y volúmenes persistentes. | *[Asier / Unai]* | *Done* |
| **2.3** | **Docker Servicio BBDD (S7)** | Configuración de MySQL 8.0 y healthcheck. | *[Asier]* | *Done* |
| **2.4** | **Proxy Inverso & LB (S1)** | Configuración de Apache como Gateway y terminación SSL. | *[Asier]* | *Done* |
| **2.5** | **HTTPS & Seguridad** | Generación de certificados y redirección 301. | *[Asier / Unai]* | *Done* |
| **2.6** | **Backend Apps (S2, S3)** | Despliegue de nodos PHP. | *[Asier]* | *Done* |
| **2.7** | **Frontend Imágenes (S4)** | Servicio para upload y almacenamiento físico. | *[Samuel]* | *Done* |
| **2.8** | **Frontend Estáticos (S6)** | Servidor CDN ligero. | *[Unai]* | *Done* |
| **2.9** | **Servicio de Backup (S5)** | Worker efímero para dumps. | *[Asier]* | *Done* |
| **2.10** | **Scripting (Automatización)** | Creación de `boot.sh` y `clean.sh`. | *[Asier]* | *Done* |
| **2.11** | **Documentación** | Documentar todo el trabajo realizado | *[Samuel]* | *Done* |

### 2.2. Retrospectiva Técnica (Problemas y Soluciones)
1. **Colapso de Recursos en AWS (RAM):** Se creó un archivo SWAP (2GB) y se escaló a `t3.small`.
2. **Infierno de Redirecciones HTTPS:** Reescritura completa del `extagram.conf` en S1.
3. **Zombis y Caché Docker:** Uso de `clean.sh` con comandos agresivos de limpieza.

### 2.3. Sprint Planning (Sprint 3)
Enfocado en la **Resiliencia, Persistencia de Datos y Documentación**. Asegurar que el sistema no pierda datos bajo ninguna circunstancia.

---

# Acta de Reunión #04: Sprint Review #03 & Sprint Planning #04

**Proyecto:** Extagram
**Fecha:** 17/02/2026
**Hora de inicio:** 15:00
**Hora de finalización:** 15:45
**Lugar:** Aula 209
**Asistentes:**
* Unai Llagostera
* Samuel Moscoso
* Asier Barranco

---

## 1. Orden del Día
1. **Sprint Review (S3):** Validación de redundancia (BLOBs), Failover y automatización de backups.
2. **Retrospectiva Técnica (S3):** Análisis de pruebas de fallas.
3. **Presentación de Fase P0.2:** Requisitos de seguridad y monitorización.
4. **Sprint Planning (S4):** Tareas de seguridad perimetral y de aplicación.

---

## 2. Desarrollo de la Reunión y Acuerdos

### 2.1. Sprint Review: Validación del Sprint 3
Se ha cumplido el objetivo de crear una aplicación resiliente. Las pruebas han demostrado que el sistema de doble almacenamiento (Disco + BLOB) funciona.

**Tabla de Tareas Completadas (Sprint 3):**

| ID | Tarea | Resultado | Responsables | Estado |
| --- | --- | --- | --- | --- |
| **3.1** | **Reconfiguración de contenedores** | Infraestructura optimizada. | *[Asier]* | *Done* |
| **3.2** | **Redundancia de Datos (BLOBs)** | Backend reescrito para inyectar binarios en MySQL. | *[Asier / Unai]* | *Done* |
| **3.3** | **Failover System** | Frontend conmuta a BBDD en caso de error 404. | *[Asier / Unai]* | *Done* |
| **3.4** | **Automatización de backups** | Scripts `backup.sh` y `restore.sh` funcionales. | *[Asier]* | *Done* |
| **3.5** | **Pruebas y fallas de servicios** | Recuperación exitosa tras borrado masivo. | *[Todos]* | *Done* |
| **3.6** | **Creación de manuales** | Entregados manuales técnicos y de usuario. | *[Samuel/Unai]* | *Done* |
| **3.7** | **Documentación Sprint 3** | Repositorio actualizado. | *[Samuel]* | *Done* |

### 2.3. Sprint Planning (Sprint 4 - Seguridad)
Inicio de la **Fase P0.2**. Objetivos: Seguridad Perimetral (Firewall), WAF (ModSecurity), Hardening de Sistemas y Conectividad VPN.

---

# Acta de Reunión #05: Sprint Review #04 & Sprint Planning #05

**Proyecto:** Extagram
**Fecha:** 24/02/2026
**Hora de inicio:** 15:15
**Hora de finalización:** 15:45
**Lugar:** Aula 209
**Asistentes:**
* Unai Llagostera
* Samuel Moscoso
* Asier Barranco

---

## 1. Orden del Día
1. **Sprint Review (S4):** Revisión de medidas de seguridad (Hardening, WAF, Firewall).
2. **Retrospectiva Técnica (S4):** Análisis de dificultades con redirección HTTPS y permisos `read_only`.
3. **Sprint Planning (S5):** Definición de objetivos para Monitorización y Automatización.

---

## 2. Desarrollo de la Reunión y Acuerdos

### 2.1. Sprint Review: Cierre del Sprint 4 (Seguridad)
Se ha logrado blindar la infraestructura cumpliendo los requisitos de la fase P0.2. 
*Nota: Se despriorizó la VPN para solventar un error crítico de redirección HTTPS que inhabilitaba el sistema.*

**Tabla de Tareas Completadas (Sprint 4):**

| ID | Tarea / Módulo | Descripción | Responsables | Estado |
| --- | --- | --- | --- | --- |
| **4.1** | **Contenedores Docker (Repair)** | Revisión del ciclo de vida de los contenedores. | *[Asier]* | *Done* |
| **4.2** | **Hardening Web (WAF)** | ModSecurity y Cabeceras (HSTS, X-Frame, X-XSS). | *[Asier]* | *Done* |
| **4.3** | **Hardening SO** | Directivas `no-new-privileges`, `pids_limit`, `read_only`. | *[Asier]* | *Done* |
| **4.4** | **Hardening BBDD** | Configuración segura (`local-infile=0`) y usuarios mínimos. | *[Asier]* | *Done* |
| **4.5** | **Firewall Perimetral (S1)** | Reglas para exponer únicamente puertos 80/443. | *[Unai]* | *Done* |
| **4.6** | **Reparar Redirección** | Solución urgente al bucle de redirecciones. | *[Asier / Unai]* | *Done* |
| **4.7** | **Documentación Sprint 4** | Actualización de repositorio y memoria técnica. | *[Unai / Samuel]* | *Done* |

### 2.3. Sprint Planning (Sprint 5 - Monitorización)
Objetivos: Monitorización Centralizada (Grafana/Loki), Dashboards visuales y Pruebas de Estrés finales.

---

# Acta de Reunión #06: Sprint Review #05 y Cierre de Proyecto

**Proyecto:** Extagram
**Fecha:** 09/03/2026
**Hora de inicio:** 15:00
**Hora de finalización:** 15:30
**Lugar:** Aula 209
**Asistentes:**
* Unai Llagostera
* Samuel Moscoso
* Asier Barranco

---

## 1. Orden del Día
1. **Sprint Review (S5):** Validación del Stack de Monitorización (Loki, Promtail, Grafana).
2. **Resultados de Auditoría y Pruebas de Estrés:** Verificación de resistencia del servidor.
3. **Revisión Final de Documentación:** Confirmación de que todos los manuales y archivos `.md` están listos.
4. **Cierre de Proyecto y Preparación para Entrega (Release Final).**

---

## 2. Desarrollo de la Reunión y Acuerdos

### 2.1. Sprint Review: Cierre del Sprint 5 (Monitorización)
El equipo confirma el cumplimiento del 100% de las tareas del último sprint. La infraestructura cuenta ahora con observabilidad completa y centralizada, y el sistema ha demostrado ser robusto ante ataques simulados.

**Hitos alcanzados en este Sprint:**
* **Centralización Efectiva:** Promtail captura correctamente los logs directamente desde el socket de Docker y los envía a Loki sin modificar los contenedores de aplicación.
* **Dashboards Funcionales:** Grafana expone en el puerto 3000 paneles en tiempo real que reflejan las peticiones 200 (OK), 403 (Bloqueos WAF) y 500 (Errores internos).
* **Supervivencia a Pruebas de Estrés:** Las pruebas de denegación de servicio controladas validaron que la configuración `pids_limit: 100` protege al host, manteniendo el servicio activo aunque bajo mayor latencia.

**Tabla de Tareas Completadas (Sprint 5):**

| ID | Tarea / Módulo | Descripción | Responsables | Estado |
| --- | --- | --- | --- | --- |
| **5.1** | **Despliegue Stack Monitorización** | Integración de contenedores Loki, Promtail y Grafana en el `docker-compose`. | *[Unai]* | *Done* |
| **5.2** | **Centralización de Logs** | Ruteo de los logs de acceso de Apache y errores de PHP/MySQL hacia Loki. | *[Unai/Asier]* | *Done* |
| **5.3** | **Dashboard de Rendimiento** | Creación y exportación de los paneles de Grafana para auditoría visual. | *[Unai]* | *Done* |
| **5.4** | **Pruebas de Estrés y Seguridad** | Ejecución de ataques DoS controlados y validación de bloqueos WAF. | *[Asier]* | *Done* |
| **5.5** | **Automatización (CI/CD)** | Revisión de scripts finales para un despliegue limpio y unificado en producción. | *[-]* | *Blocked* |
| **5.6** | **Documentación Final S5** | Cierre de manuales, actas y limpieza definitiva del repositorio para la entrega. | *[Samuel/Asier]* | *Done* |

### 2.2. Conclusión y Cierre del Proyecto
El Product Owner y el Scrum Master declaran el **Proyecto Extagram FINALIZADO** a fecha de hoy, cumpliendo con los requisitos exigidos para la entrega del 10 de marzo. El producto final es una arquitectura de microservicios resiliente, securizada y completamente documentada.

---

# P0.1-ASIXc2AC-G04 - Extagram

> **Módulo:** M0379 - Proyecto ASIXc  
> **Actividad:** P0.1 - Desplegamiento aplicación Extagram  
> **Centro:** Institut Tecnològic Barcelona  

---

[PONER FUTURO INDEX]

---

##  Descripción del Proyecto
Este repositorio contiene la infraestructura como código (IaC), scripts de aprovisionamiento y documentación técnica para el despliegue de **Extagram**, una aplicación web de intercambio de imágenes.

El objetivo es diseñar e implementar una arquitectura de **Alta Disponibilidad** escalable, separando los servicios en capas (Balanceo, Servidores Web, Base de Datos y Almacenamiento Estático).

---

## Gestión del Proyecto (Agile & ProofHub)

Utilizamos la metodología **Agile** para la gestión de tareas, coordinando el equipo a través de la plataforma **ProofHub**.

### Roles del Equipo
| Miembro del Equipo | Rol Asignado | Responsabilidades Principales |
|--------------------|--------------|-------------------------------|
| **Samuel Moscoso** | Scrum Master | Facilitador, gestión de ProofHub, eliminación de bloqueos. |
| **Unai Llagostera** | Product Owner | Definición de requisitos, revisión de entregables (Docs). |
| **Todos los integrantes** | SysAdmin / Dev | Configuración de servidores, scripting y despliegue. |
| **Todos los integrantes** | SysAdmin / QA | Pruebas de carga, seguridad y documentación de pruebas. |

###  Planificación de Sprints
| Sprint | Duración | Objetivo Principal |
|:------:|----------|--------------------|
| **Sprint 1** | 15/12/25 - 19/01/26 | Investigación, diseño de arquitectura y configuración de entorno local (Git). |
| **Sprint 2** | 19/1/26 - 27/1/26 | Despliegue de servidores web (Nginx/PHP) y base de datos. |
| **Sprint 3** | 2/2/26 - 10/2/26 | Implementación de Balanceador, NFS y puesta en producción final. |

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
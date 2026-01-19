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

| ID | Tarea |
| --- | --- | 
| **1.1** | Creación de Repositorio Git y estructura de directorios (`/docs`, `/src`). |
| **1.2** | Redacción del Estudio Tecnológico y Justificación de AWS. | 
| **1.3** | **AWS:** Lanzamiento de instancia EC2 (VPC, Security Groups, KeyPairs). |
| **1.4** | **SysAdmin:** Scripting de `User Data` para instalación desatendida de LAMP. | 
| **1.5** | **BBDD:** Configuración de MySQL, creación de usuario y restauración del `.sql`. | 
| **1.6** | **Deploy:** Subida de código PHP vía Git/SFTP y configuración de VirtualHost. | 

---

## 4. Próximos Pasos

* **Revisión de Avance (Daily/Checkpoint):** Martes 17/12 a las 16:45.
* **Objetivo para la revisión:** Tener la instancia corriendo, con IP pública accesible, las tareas asignadas para cada uno empezadas y con ideas claras.



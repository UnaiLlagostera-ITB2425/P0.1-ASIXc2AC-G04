# Creación de Dockers Multi-Servicio en AWS

## ¿Por qué Docker?

Docker nos permite **containerizar cada servicio de forma independiente**, garantizando que:
- Cada componente (Apache, PHP-FPM, MySQL) está **aislado** en su propio contenedor
- Se **escala fácilmente** (puedes añadir más instancias de S2/S3 sin complicaciones)
- La **reproducibilidad** es garantizada (mismo código = mismo comportamiento en cualquier máquina)
- La **configuración es versionable** (todo en archivos YAML)
- Se **simplifica el despliegue** en AWS (orquestación clara y predecible)
- Se **reducen dependencias** del sistema host (no necesitas instalar PHP, Apache, MySQL directamente)

En resumen: **una aplicación modular, portable y fácil de mantener**.

---

## Descripción General

Esta configuración despliega una aplicación web completa con:

- **S1**: Apache como proxy inverso (punto de entrada público)
- **S2 + S3**: PHP-FPM con balanceado de carga para `extagram.php`
- **S4**: PHP-FPM para gestión de uploads
- **S5**: Apache sirviendo imágenes estáticas (uploads)
- **S6**: Apache sirviendo CSS/SVG estáticos
- **S7**: MySQL Base de datos

Todos los servicios corren en **contenedores Docker** dentro de una **red interna** en AWS EC2.

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS EC2 Instance                     │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Docker Network (app-network)            │   │
│  │                                                      │   │
│  │  ┌─────────────────────────────────────────────┐     │   │
│  │  │ S1: Apache Proxy (80, 443)                  │     │   │
│  │  │ Entrada pública - Balanceador de carga      │     │   │
│  │  └───────────┬─────────────────────────────────┘     │   │
│  │              │                                       │   │
│  │    ┌─────────┼──────────────┬────────────────┐       │   │
│  │    ▼         ▼              ▼                ▼       │   │
│  │ ┌────────────────┐  ┌──────────────┐  ┌──────────┐   │   │
│  │ │ S2: PHP-FPM 1  │  │ S4: PHP-FPM  │  │S5: Apache│   │   │
│  │ │ extagram.php   │  │ upload.php   │  │Images    │   │   │
│  │ └────────────────┘  └──────────────┘  └──────────┘   │   │
│  │         ▲                                            │   │
│  │    ┌────┴───────────┐                                │   │
│  │    │                │                                │   │
│  │ ┌──────────────┐  ┌──────────┐  ┌──────────────┐     │   │
│  │ │ S3: PHP-FPM  │  │S6: Apache│  │ S7: MySQL    │     │   │
│  │ │ extagram.php │  │Assets    │  │ Base datos   │     │   │
│  │ └──────────────┘  └──────────┘  └──────────────┘     │   │
│  │                                                      │   │
│  │          Volúmenes Compartidos:                      │   │
│  │          - shared-uploads (S4, S5)                   │   │
│  │          - mysql_data (S7)                           │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Estructura de Directorios

```text
proyecto-docker/
├─ S1/
│  ├─ docker-compose.yml
│  ├─ conf/
│  │  └─ proxy.conf
│  └─ logs/
├─ S2/
│  ├─ docker-compose.yml
│  ├─ Dockerfile
│  └─ app/
│     ├─ extagram.php
│     └─ index.php
├─ S3/
│  ├─ docker-compose.yml
│  ├─ Dockerfile
│  └─ app/
│     ├─ extagram.php
│     └─ index.php
├─ S4/
│  ├─ docker-compose.yml
│  ├─ Dockerfile
│  └─ app/
│     └─ upload.php
├─ S5/
│  ├─ docker-compose.yml
│  ├─ conf/
│  │  └─ httpd-images.conf
│  └─ logs/
├─ S6/
│  ├─ docker-compose.yml
│  ├─ conf/
│  │  └─ httpd-assets.conf
│  ├─ assets/
│  │  ├─ style.css
│  │  └─ preview.svg
│  └─ logs/
├─ S7/
│  ├─ docker-compose.yml
│  ├─ init.sql
│  └─ data/
└─ launch.sh                   (script para lanzar todo)
```

---

## Configuración de Cada Servicio

### S6: Apache Proxy Inverso

```text
S6/
├── docker-compose.yml
└── assets/
    ├── preview.svg
    └── style.css
```

**Archivo: `S6/docker-compose.yml`**

```yaml
version: "3.9"

services:
  apache-assets:
    image: httpd:2.4
    container_name: S6-apache-assets
    volumes:
      - ./assets:/usr/local/apache2/htdocs
    networks:
      - app-network
    restart: unless-stopped

networks:
  app-network:
    external: true
```

**Función**: desacopla la carga de servir archivos estáticos del servidor de aplicación. Está optimizado solo para entregar archivos rápidamente sin ejecutar PHP ni operaciones pesadas. Otros servicios acceden a los assets por HTTP a través de la red Docker compartida.
---


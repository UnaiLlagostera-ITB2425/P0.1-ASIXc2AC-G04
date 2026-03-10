# Arquitectura Final del Sistema Extagram

Este documento define la infraestructura definitiva de la plataforma Extagram. El sistema ha evolucionado desde un entorno monolítico hacia una **arquitectura de microservicios contenerizada y securizada** mediante Docker Compose, complementada con un stack completo de observabilidad (Logs y Métricas).

## 1. Topología y Flujo de Datos (Gateway Offloading)

La infraestructura desacopla la capa de presentación, la lógica de negocio, el procesamiento de carga de archivos (I/O intensivo) y la persistencia de datos. 

Todos los servicios operan dentro de una red aislada (`extagram-net`), exponiendo únicamente el Gateway (**S1**) hacia la red pública/host en los puertos 80 y 443.

### Inventario de Componentes Base (S1 - S7)

| ID | Servicio (`container_name`) | Base / Build | Rol Funcional | Red |
| :--- | :--- | :--- | :--- | :--- |
| **S1** | `s1-proxy` | `Dockerfile.proxy` | Reverse Proxy Apache (L7), Terminación SSL y Balanceador. | `extagram-net` |
| **S2** | `s2-app` | `Dockerfile.upload` | Lógica de negocio (Lectura/Renderizado PHP). | `extagram-net` |
| **S3** | `s3-app` | `Dockerfile.upload` | Réplica de lógica de negocio (Alta Disponibilidad). | `extagram-net` |
| **S4** | `s4-upload` | `Dockerfile.upload` | Procesamiento exclusivo de uploads (Escritura). | `extagram-net` |
| **S5** | `s5-images` | `httpd:2.4` | Servidor estático optimizado para servir imágenes de usuario. | `extagram-net` |
| **S6** | `s6-static` | `httpd:2.4` | Servidor de activos del frontend (CSS, JS, Logos). | `extagram-net` |
| **S7** | `s7-db` | `mysql:8.0` | Motor de base de datos relacional persistente. | `extagram-net` |

---

## 2. Stack de Monitorización y Observabilidad

Para garantizar el control del sistema en producción, se ha integrado una segunda red (`monitoring-net`) completamente aislada de la red de aplicación.

| Servicio | Imagen | Rol Funcional | Exposición |
| :--- | :--- | :--- | :--- |
| **Loki** | `grafana/loki:2.9.0` | Agregador y motor de indexación de logs. | Interna |
| **Promtail** | `grafana/promtail:3.0.0` | Recolector de logs desde el socket de Docker hacia Loki. | Interna |
| **Prometheus** | `prom/prometheus:latest` | Base de datos de series temporales para métricas de rendimiento. | Interna |
| **Node Exporter**| `prom/node-exporter:latest` | Agente para leer métricas de hardware del servidor host (CPU, RAM). | Interna |
| **Grafana** | `grafana/grafana:10.2.2` | Interfaz visual (Dashboards). | `127.0.0.1:3000` |

> **Nota de Seguridad:** Grafana solo expone su puerto hacia el `localhost` del servidor host (127.0.0.1). El acceso se realiza obligatoriamente mediante un túnel SSH, impidiendo el acceso público.

---

## 3. Hardening y Seguridad Aplicada a Contenedores

Todos los contenedores de la infraestructura (tanto de aplicación como de monitorización) implementan directivas de seguridad estrictas en el `docker-compose.yml` para mitigar ataques de escalada de privilegios y denegación de servicio:

1. **`read_only: true`**: El sistema de archivos raíz del contenedor es inmutable. Un atacante no puede modificar el código, inyectar malware ni alterar la configuración.
2. **`tmpfs`**: Al ser el contenedor de solo lectura, se montan volúmenes efímeros en RAM (`/tmp`, `/var/run`, `/var/log`) para los procesos legítimos que requieren escritura temporal.
3. **`no-new-privileges: true`**: Impide que un proceso secundario obtenga más privilegios que su proceso padre (anula el efecto de bits SUID/SGID).
4. **`pids_limit: 100`**: Limita el número máximo de procesos (hilos) que puede crear un contenedor, mitigando ataques tipo *Fork Bomb*.

---

## 4. Persistencia y Volúmenes

Para garantizar que los datos sobreviven al ciclo de vida efímero de los contenedores, se utilizan dos tipos de montajes:

### Volúmenes Gestionados (Named Volumes)
* **`db_data`**: Almacena las tablas, configuraciones persistentes y registros de MySQL (`/var/lib/mysql`).
* **`loki_data`, `grafana_data`, `prometheus_data`**: Persisten la configuración, los dashboards y el historial de logs/métricas.

### Montajes de Directorio Host (Bind Mounts)
* **Código Fuente (`./src` y `./assets`)**: Se inyectan en tiempo de ejecución en los servidores Apache y PHP (`/var/www/html`).
* **Uploads Compartidos (`./src/uploads`)**: 
  * **S4 (Upload)** lo monta para escribir las fotos físicas tras el procesamiento PHP.
  * **S5 (Images)** lo monta para leerlas y servirlas rápidamente vía HTTP puro.

---

## 5. Guía de Despliegue y Verificación (Lifecycle)

Al consolidar toda la arquitectura en un único `docker-compose.yml`, el despliegue se ha simplificado enormemente, eliminando la necesidad de scripts complejos.

### 5.1. Inicialización (Arranque)

Desde la raíz del proyecto (donde reside `docker-compose.yml`):

```bash
# Construir imágenes personalizadas (S1, S2, S3, S4) y levantar todo en segundo plano
sudo docker compose up -d --build
```

### 5.2. Verificación de Salud (Healthchecks)

Verificar que los 12 contenedores están operativos:

```bash
sudo docker ps
```

* El contenedor **`s7-db`** posee un *healthcheck* nativo que ejecuta `mysqladmin ping`. Docker esperará a que el estado pase de `(health: starting)` a `(healthy)` automáticamente.
* Si el proxy **S1** responde con `502 Bad Gateway`, significa que Apache ha arrancado antes que PHP. En 10 segundos el sistema se estabiliza.

### 5.3. Mantenimiento y Logs

Para consultar qué está ocurriendo en tiempo real en un servicio:

```bash
# Ver logs del proxy (errores de balanceo o SSL)
sudo docker logs -f s1-proxy

# Ver logs de la base de datos
sudo docker logs -f s7-db
```

Para detener la infraestructura completa de forma segura:

```bash
sudo docker compose down
```
# Documentación Centralización de Logs y Monitorización de Métricas con Grafana, Loki y Prometheus

## 1. Introducción y Objetivo

Se ha implementado un sistema integral de observabilidad (logs y métricas) para la plataforma **Extagram**, desplegada mediante microservicios en Docker Compose. El objetivo es recoger, almacenar y visualizar de forma centralizada tanto los logs de los contenedores como el rendimiento del servidor físico (CPU, RAM, Disco), garantizando la seguridad de la red y el acceso restringido a los dashboards.

La solución elegida combina:

* **Loki** como backend de almacenamiento de logs.
* **Promtail** como agente recolector de logs desde los contenedores Docker.
* **Prometheus** como base de datos de series temporales para métricas.
* **Node Exporter** como agente recolector de métricas de hardware del servidor.
* **Grafana** como interfaz web unificada de visualización.

Se ha añadido una red interna específica para los servicios de monitorización, separada de la red de aplicación, para mejorar el aislamiento y la seguridad.

---

## 2. Arquitectura de Monitorización



Se han añadido cinco nuevos servicios al archivo `docker-compose.yml` original, integrándose en una red interna `monitoring-net` (con nombre real `extagram_monitoring-net`) y respetando las políticas de hardening existentes.

| Servicio | Contenedor | Descripción |
| :--- | :--- | :--- |
| **Loki** | `loki` | Agregador de logs. Almacena y permite consultar los logs etiquetados. |
| **Promtail** | `promtail` | Recolecta logs de los contenedores Docker y los envía a Loki. |
| **Prometheus** | `prometheus` | Recolecta y almacena métricas del sistema en formato de series de tiempo. |
| **Node Exporter** | `node-exporter` | Lee las métricas de hardware (CPU, RAM, Disco) del sistema operativo host. |
| **Grafana** | `grafana` | Interfaz web para consultar logs y métricas. Expuesto solo en localhost. |

**Flujos de datos:**

* **Logs:** Los contenedores generan logs en `stdout`/`stderr` → Docker los almacena en `/var/lib/docker/containers` → Promtail los lee y los envía a Loki → Grafana consulta a Loki.
* **Métricas:** Node Exporter expone métricas de hardware en el puerto 9100 → Prometheus las recolecta (pull) cada 15 segundos → Grafana consulta a Prometheus.

**Redes:**

* `extagram_extagram-net`: red de la aplicación (s1-proxy, s2-app, s3-app, s4-upload, s5-images, s6-static, s7-db).
* `extagram_monitoring-net`: red de monitorización (loki, promtail, prometheus, node-exporter, grafana). Ambas redes están aisladas.

---

## 3. Justificación de la Elección

Se optó por este stack por las siguientes razones:

* **Loki + Promtail** son extremadamente ligeros para logs de Docker y no requieren indexación completa (solo etiquetas).
* **Prometheus + Node Exporter** es el estándar de la industria para monitorización de hardware y microservicios, utilizando un modelo *pull* altamente eficiente.
* **Grafana** proporciona una interfaz unificada que permite cruzar datos: ver un pico de CPU en las métricas y, en la misma pantalla, ver qué logs se generaron en ese exacto segundo.
* Todo el stack es de código abierto y se alinea con la arquitectura de microservicios de Extagram.

---

## 4. Requisitos Previos

* Servidor Linux (Ubuntu) con Docker Engine ≥20.10 y Docker Compose.
* Acceso SSH a la instancia (clave privada `.pem`).
* Puertos 80, 443 y 22 abiertos en el Security Group de AWS. El puerto 3000 **no** debe estar abierto al público.
* Estructura de directorios existente de Extagram.

---

## 5. Medidas de Seguridad Aplicadas (Hardening)

Todos los servicios de monitorización heredan las estrictas políticas de seguridad del proyecto:

### 5.1. Hardening a nivel de contenedor

* `security_opt: no-new-privileges:true` – Evita escalada de privilegios en todos los contenedores.
* `pids_limit: 100` – Limita el número de procesos para mitigar ataques fork-bomb.
* `read_only: true` – Sistema de archivos raíz de solo lectura (excepto Promtail, Prometheus y Node Exporter que requieren accesos específicos de lectura al host).
* `tmpfs` montado en `/tmp` y otros directorios necesarios para escritura temporal.

### 5.2. Hardening específico por servicio

* **Loki**: Se montó `tmpfs` en `/wal` para permitir la escritura temporal de la base de datos sin comprometer el sistema de archivos raíz.
* **Promtail**: Accede al socket de Docker (`/var/run/docker.sock`) estrictamente en modo `ro` (solo lectura).
* **Node Exporter**: Monta los sistemas de archivos del host (`/proc`, `/sys`, `/`) estrictamente en modo `ro` (solo lectura) para auditar sin capacidad de modificar el servidor físico.
* **Grafana**: El puerto 3000 se publica solo en `127.0.0.1` (localhost).

---

## 6. Implementación Paso a Paso

### 6.1. Preparación del entorno

```bash
cd ~/extagram
mkdir -p loki promtail prometheus grafana/provisioning/datasources
```

### 6.2. Configuración de Prometheus

Archivo `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

### 6.3. Configuración de Loki

Archivo `loki/loki-config.yaml`:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_retain_period: 30s
  wal:
    dir: /loki/wal

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: true
  retention_period: 336h   # 14 días

compactor:
  working_directory: /loki/compactor
  shared_store: filesystem
```

### 6.4. Configuración de Promtail

Archivo `promtail/promtail-config.yaml`:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'logstream'
      - source_labels: ['__meta_docker_container_label_com_docker_compose_service']
        target_label: 'service'
      - source_labels: ['__meta_docker_container_label_com_docker_compose_project']
        target_label: 'project'
```

### 6.5. Aprovisionamiento Automático de Grafana

Archivo `grafana/provisioning/datasources/datasources.yaml`. (Este archivo auto-configura Loki y Prometheus para evitar hacerlo manualmente en la interfaz).

```yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: true
    editable: false
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: false
    editable: false
```

### 6.6. Modificación del `docker-compose.yml`

Anexar los siguientes servicios al final del archivo original, respetando las indentaciones:

```yaml
  # --- LOKI: agregación de logs ---
  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    restart: always
    user: "0"
    volumes:
      - ./loki:/etc/loki
      - loki_data:/loki
    command: -config.file=/etc/loki/loki-config.yaml
    networks:
      - monitoring-net
    security_opt:
      - no-new-privileges:true
    pids_limit: 100
    read_only: true
    tmpfs:
      - /tmp
      - /wal

  # --- PROMTAIL: recolección de logs ---
  promtail:
    image: grafana/promtail:3.0.0
    container_name: promtail
    restart: always
    volumes:
      - ./promtail:/etc/promtail
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command: -config.file=/etc/promtail/promtail-config.yaml
    networks:
      - monitoring-net
    depends_on:
      - loki
    security_opt:
      - no-new-privileges:true
    pids_limit: 100
    tmpfs:
      - /tmp

  # --- PROMETHEUS: Base de datos de métricas ---
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - monitoring-net
    security_opt:
      - no-new-privileges:true
    pids_limit: 100

  # --- NODE EXPORTER: Métricas de hardware ---
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: always
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
    networks:
      - monitoring-net
    security_opt:
      - no-new-privileges:true
    pids_limit: 100

  # --- GRAFANA: visualización ---
  grafana:
    image: grafana/grafana:10.2.2
    container_name: grafana
    restart: always
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=
      - GF_SERVER_ROOT_URL=%(protocol)s://%(domain)s:%(http_port)s/grafana/
      - GF_SERVER_SERVE_FROM_SUB_PATH=true
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    networks:
      - monitoring-net
    depends_on:
      - loki
      - prometheus
    security_opt:
      - no-new-privileges:true
    pids_limit: 100
    read_only: true
    tmpfs:
      - /tmp

volumes:
  db_data:
  loki_data:
  grafana_data:
  prometheus_data:

networks:
  extagram-net:
    driver: bridge
  monitoring-net:
    driver: bridge
```

### 6.7. Despliegue inicial

Bajar el entorno actual (para recrear redes y volúmenes limpios) y volver a levantarlo:

```bash
sudo docker compose down
sudo docker compose up -d
```

---

## 7. Acceso y Dashboards

### 7.1. Establecer túnel SSH

Por seguridad, Grafana no está expuesto en internet. Desde la máquina local:

```bash
ssh -i /ruta/a/tu-clave.pem -L 3000:localhost:3000 ubuntu@<IP_DEL_SERVIDOR>
```

Abrir el navegador en `http://localhost:3000`. Credenciales por defecto: `admin` / `admin`. Gracias al aprovisionamiento automático (paso 6.5), Loki y Prometheus ya estarán conectados internamente.

### 7.2. Importación del Dashboard de Rendimiento de Host

Para visualizar CPU, RAM y Disco:

1. Ir a **Dashboards → New → Import**.
2. Pegar el siguiente JSON.
3. Al finalizar, seleccionar "Prometheus" en el campo requerido.

```json
{
  "__inputs": [
    {
      "name": "DS_PROMETHEUS",
      "label": "Prometheus Datasource",
      "description": "Selecciona el origen de datos de Prometheus",
      "type": "datasource",
      "pluginId": "prometheus",
      "pluginName": "Prometheus"
    }
  ],
  "title": "Rendimiento del Servidor Host",
  "uid": "rendimiento-host-v2",
  "tags": ["rendimiento", "host", "extagram"],
  "timezone": "browser",
  "schemaVersion": 39,
  "refresh": "5s",
  "panels": [
    {
      "type": "gauge",
      "title": "Uso de CPU",
      "gridPos": { "x": 0, "y": 0, "w": 8, "h": 8 },
      "datasource": "${DS_PROMETHEUS}",
      "targets": [
        {
          "expr": "100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\"}[1m])) * 100)",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "min": 0,
          "max": 100,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 70 },
              { "color": "red", "value": 90 }
            ]
          }
        }
      }
    },
    {
      "type": "gauge",
      "title": "Uso de Memoria RAM",
      "gridPos": { "x": 8, "y": 0, "w": 8, "h": 8 },
      "datasource": "${DS_PROMETHEUS}",
      "targets": [
        {
          "expr": "100 - (avg(node_memory_MemAvailable_bytes) / avg(node_memory_MemTotal_bytes) * 100)",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "min": 0,
          "max": 100,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 75 },
              { "color": "red", "value": 90 }
            ]
          }
        }
      }
    },
    {
      "type": "gauge",
      "title": "Uso de Disco (Raíz)",
      "gridPos": { "x": 16, "y": 0, "w": 8, "h": 8 },
      "datasource": "${DS_PROMETHEUS}",
      "targets": [
        {
          "expr": "100 - (max(node_filesystem_avail_bytes{mountpoint=\"/\"}) / max(node_filesystem_size_bytes{mountpoint=\"/\"}) * 100)",
          "refId": "A",
          "datasource": "${DS_PROMETHEUS}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "min": 0,
          "max": 100,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "orange", "value": 80 },
              { "color": "red", "value": 90 }
            ]
          }
        }
      }
    }
  ]
}
```

---

## 8. Verificación y Pruebas de Estrés

Para validar que la monitorización responde correctamente, ejecutar desde la terminal del servidor:

**1. Test de estrés de CPU (Saturar 1 núcleo):**

```bash
cat /dev/urandom > /dev/null
# Pulsar Ctrl+C tras observar el pico en Grafana.
```

**2. Test de estrés de RAM (Ocupar 500MB):**

```bash
python3 -c "basura = ' ' * (500 * 1024 * 1024); import time; time.sleep(60)"
# Se libera automáticamente al pasar un minuto.
```

---

## 9. Mantenimiento y Troubleshooting

**Ver logs de los agentes:**
```bash
sudo docker logs promtail --tail 50
sudo docker logs prometheus --tail 50
```

**Error "Conflict. The container name is already in use":**
```bash
sudo docker rm -f <container_id>
```

**Copias de seguridad de métricas y logs:**
```bash
sudo tar -czf backup_observability_$(date +%Y%m%d).tar.gz grafana_data/ loki_data/ prometheus_data/
```

---

## 10. Conclusiones

Se ha implementado con éxito un sistema de observabilidad integral (Logs + Métricas) que cumple los objetivos exigidos. Las medidas de hardening aplicadas garantizan un entorno seguro, y la separación de redes asegura que la recopilación de datos no interfiera con el tráfico de la aplicación web Extagram. La automatización del aprovisionamiento de datasources elimina los errores manuales en Grafana.
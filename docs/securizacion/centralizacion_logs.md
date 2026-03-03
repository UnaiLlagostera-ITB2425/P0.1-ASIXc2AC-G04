# Documentación Centralización de Logs con Grafana, Loki y Promtail

## 1. Introducción y Objetivo

Se ha implementado un sistema de centralización de logs para la plataforma **Extagram**, desplegada mediante microservicios en Docker Compose. El objetivo es recoger, almacenar y visualizar de forma centralizada los logs de todos los contenedores, garantizando la seguridad de la red y el acceso restringido a los dashboards.

La solución elegida combina:
- **Loki** como backend de almacenamiento de logs.
- **Promtail** como agente recolector de logs desde los contenedores Docker.
- **Grafana** como interfaz de visualización y exploración.

Se ha añadido una red interna específica para los servicios de monitorización, separada de la red de aplicación, para mejorar el aislamiento y la seguridad.

## 2. Arquitectura de Monitorización

Se han añadido tres nuevos servicios al archivo `docker-compose.yml` original, integrándose en una red interna `monitoring-net` (con nombre real `extagram_monitoring-net`) y respetando las políticas de hardening existentes.

| Servicio   | Contenedor | Descripción                                                                 |
|------------|------------|-----------------------------------------------------------------------------|
| **Loki**   | `loki`     | Agregador de logs. Almacena y permite consultar los logs etiquetados.      |
| **Promtail** | `promtail` | Recolecta logs de todos los contenedores Docker y los envía a Loki.       |
| **Grafana** | `grafana`  | Interfaz web para consultar y visualizar logs. Expuesto solo en localhost. |

**Flujo de datos:**  
Los contenedores generan logs en `stdout`/`stderr` → Docker los almacena en `/var/lib/docker/containers` → Promtail los lee, añade etiquetas (`container`, `service`, `project`) y los envía a Loki → Grafana consulta Loki y muestra los logs.

**Redes:**
- `extagram_extagram-net`: red de los servicios de la aplicación (s1-proxy, s2-app, s3-app, s4-upload, s5-images, s6-static, s7-db).
- `extagram_monitoring-net`: red de los servicios de monitorización (loki, promtail, grafana).

Ambas redes están aisladas; no hay comunicación directa entre ellas. Promtail accede a los logs mediante el socket de Docker (montado como volumen), no por red.

## 3. Justificación de la Elección

Se optó por **Grafana + Loki + Promtail** por las siguientes razones:

- **Loki** es ligero, no requiere indexación completa (solo etiquetas), ideal para entornos Docker.
- **Promtail** se integra nativamente con Docker y descubre automáticamente los contenedores.
- **Grafana** proporciona una interfaz unificada y potente para exploración y dashboards.
- Todo el stack es de código abierto, con buena documentación y comunidad activa.
- Se alinea con el enfoque de microservicios ya utilizado en Extagram.

Alternativas como ELK (Elasticsearch, Logstash, Kibana) fueron descartadas por requerir más recursos y complejidad. Nagios no es adecuado para centralización de logs.

## 4. Requisitos Previos

- Servidor Linux (Ubuntu 20.04/22.04/24.04) con Docker Engine ≥20.10 y Docker Compose (preferiblemente V2).
- Acceso SSH a la instancia (clave privada `.pem`).
- Puertos 80, 443 y 22 abiertos en el Security Group de AWS. El puerto 3000 **no** debe estar abierto (se accede por túnel SSH).
- Certificados SSL en `./certs/` para el proxy inverso (s1-proxy).
- Estructura de directorios existente de Extagram.

## 5. Medidas de Seguridad Aplicadas (Hardening)

Todos los servicios, incluidos los nuevos, heredan las políticas de seguridad ya implementadas en Extagram y se han reforzado:

### 5.1. Hardening a nivel de contenedor
- `security_opt: no-new-privileges:true` – Evita escalada de privilegios.
- `pids_limit: 100` – Limita el número de procesos.
- `read_only: true` (excepto Promtail) – Sistema de archivos raíz de solo lectura.
- `tmpfs` montado en `/tmp` y otros directorios necesarios para escritura temporal.

### 5.2. Hardening específico de Loki y Grafana
- **Loki**: se añadió `user: "0"` temporalmente para solventar permisos (se recomienda ajustar con el UID 10001). Se montó `tmpfs` en `/wal` para permitir la escritura del WAL.
- **Grafana**: el puerto 3000 se publica solo en `127.0.0.1` (localhost), impidiendo accesos externos.
- **Promtail**: necesita acceso al socket de Docker (`/var/run/docker.sock`) para descubrir contenedores; se monta en modo `ro` (solo lectura) para minimizar riesgos.

### 5.3. Redes internas aisladas
- Los servicios de monitorización están en su propia red (`monitoring-net`), aislados de la red de aplicación.
- La red de aplicación (`extagram-net`) permanece para los servicios originales.

### 5.4. Firewall (Security Group AWS)
- Solo se exponen al exterior los puertos 80 (HTTP), 443 (HTTPS) y 22 (SSH).
- El puerto 3000 de Grafana **no es accesible** desde internet; se accede mediante túnel SSH.

## 6. Implementación Paso a Paso

### 6.1. Preparación del entorno
```bash
cd ~/extagram
mkdir -p loki promtail grafana/provisioning/datasources
```

### 6.2. Configuración de Loki
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

### 6.3. Configuración de Promtail
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

### 6.4. Configuración del datasource automático en Grafana
Archivo `grafana/provisioning/datasources/loki.yaml`:
```yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: true
    editable: false
```

### 6.5. Modificación del `docker-compose.yml`

Se añadieron los servicios de monitorización al final del archivo existente, junto con la nueva red `monitoring-net`. A continuación se muestra el `docker-compose.yml` completo (solo las partes modificadas/nuevas; los servicios originales se mantienen igual).

```yaml
version: '3.8'

services:
  # ... servicios originales (s7-db, s2-app, s3-app, s4-upload, s5-images, s6-static, s1-proxy) ...

  # --- LOKI: agregación de logs ---
  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    restart: always
    user: "0"   # temporal para permisos; optimizar con UID 10001
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

  # --- PROMTAIL: recolección de logs (actualizado a v3.0.0) ---
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

networks:
  extagram-net:
    driver: bridge
  monitoring-net:
    driver: bridge
```

**Nota:** Las redes creadas por Docker Compose recibirán el prefijo del proyecto (por defecto `extagram_`), por lo que los nombres reales serán `extagram_extagram-net` y `extagram_monitoring-net`.

### 6.6. Creación de directorios de datos y permisos
```bash
sudo mkdir -p grafana_data loki_data
sudo chown -R 472:472 grafana_data      # usuario grafana (UID 472)
sudo chown -R 10001:10001 loki_data     # usuario loki (UID 10001)
```

### 6.7. Despliegue inicial
```bash
sudo docker compose up -d
```

Si se encuentra el error `KeyError: 'ContainerConfig'` debido a una versión antigua de `docker-compose` (1.29), usar `docker compose` (con espacio) en su lugar. Si no está instalado, instalar el plugin V2:
```bash
sudo apt update
sudo apt install docker-compose-v2
```

## 7. Acceso a los Logs mediante Túnel SSH

Por razones de seguridad, Grafana no está expuesto directamente en internet. Se accede mediante un túnel SSH desde la máquina local.

### 7.1. Establecer túnel SSH
```bash
ssh -i /ruta/a/tu-clave.pem -L 3000:localhost:3000 ubuntu@52.21.109.232
```
- `-L 3000:localhost:3000` redirige el puerto 3000 de la máquina local al puerto 3000 del servidor (donde escucha Grafana).
- La terminal debe permanecer abierta mientras se use Grafana.

### 7.2. Acceder a Grafana
Abrir el navegador en `http://localhost:3000` (no `https`).  
Credenciales por defecto: `admin` / `admin` (se recomienda cambiarlas en la interfaz o mediante variable de entorno).

### 7.3. Explorar logs
- Ir a **Explore** (icono de brújula).
- Seleccionar la fuente de datos **Loki**.
- Escribir una consulta, por ejemplo: `{container="s2-app"}`.
- Pulsar **Run query** para ver los logs en tiempo real.

## 8. Verificación y Uso

### 8.1. Comprobar que los contenedores están funcionando
```bash
sudo docker ps | grep -E "loki|promtail|grafana"
```
Deben aparecer en estado `Up`.

### 8.2. Probar conectividad interna
```bash
sudo docker exec grafana curl http://loki:3100/ready
```
Debería responder `ready`.

### 8.3. Crear un dashboard básico
- En Grafana, ir a **Dashboards → New → New Dashboard**.
- Añadir una visualización con consulta tipo logs.
- Filtrar por contenedor o servicio.

## 9. Mantenimiento y Troubleshooting

### 9.1. Logs de los contenedores de monitorización
```bash
sudo docker logs loki --tail 50
sudo docker logs promtail --tail 50
sudo docker logs grafana --tail 50
```

### 9.2. Reinicio de servicios
```bash
sudo docker compose restart <servicio>
```

### 9.3. Problemas comunes y soluciones

**Error: "unable to open database file: is a directory" en Grafana**  
Solución:
```bash
sudo rm -rf grafana_data/*
sudo chown -R 472:472 grafana_data
sudo docker compose restart grafana
```

**Error: Loki no arranca por permisos de WAL**  
Añadir `tmpfs: - /wal` temporalmente o configurar `wal.dir` dentro del volumen persistente (como en la configuración final).

**Error: "Conflict. The container name ... is already in use"**  
Eliminar el contenedor conflictivo:
```bash
sudo docker ps -a | grep loki
sudo docker rm -f <container_id>
```

**Error de versión de docker-compose**  
Usar `docker compose` (con espacio) en lugar de `docker-compose`. Instalar Docker Compose V2 si es necesario:
```bash
sudo apt update
sudo apt install docker-compose-v2
```

### 9.4. Copias de seguridad
Los datos de Loki y Grafana se almacenan en volúmenes Docker (`loki_data` y `grafana_data`). Se pueden respaldar mediante:
```bash
sudo tar -czf backup_grafana_$(date +%Y%m%d).tar.gz grafana_data/
sudo tar -czf backup_loki_$(date +%Y%m%d).tar.gz loki_data/
```

## 10. Conclusiones

Se ha implementado con éxito un sistema centralizado de logs que cumple los objetivos de monitorización y seguridad exigidos. La elección de Grafana, Loki y Promtail ha resultado adecuada por su ligereza, integración con Docker y facilidad de configuración.

Las medidas de hardening aplicadas (contenedores `read-only`, límites de procesos, tmpfs, redes aisladas) garantizan un entorno seguro. El acceso mediante túnel SSH evita exponer Grafana a internet, cumpliendo así con el requisito de red securizada.

La separación en dos redes internas (`extagram-net` para la aplicación y `monitoring-net` para los logs) mejora el aislamiento sin afectar la funcionalidad.

Esta documentación servirá como guía para futuras ampliaciones y para el equipo de operaciones.

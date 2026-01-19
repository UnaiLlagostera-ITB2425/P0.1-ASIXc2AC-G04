## 5. Lanzar servicios en orden

**Opción A: Script automático**

```bash
chmod +x launch.sh
./launch.sh
```

**Script: `launch.sh`**

```bash
#!/bin/bash

echo "=== Creando red y volúmenes en Docker ==="
docker network create app-network 2>/dev/null || echo "Red app-network ya existe"
docker volume create mysql_data 2>/dev/null || echo "Volumen mysql_data ya existe"
docker volume create shared-uploads 2>/dev/null || echo "Volumen shared-uploads ya existe"

echo "=== Iniciando S7 (MySQL) ==="
cd S7
docker-compose up -d
echo "Esperando 10 segundos para que MySQL esté listo..."
sleep 10

echo "=== Iniciando S2 (PHP-FPM 1) ==="
cd ../S2
docker-compose up -d

echo "=== Iniciando S3 (PHP-FPM 2) ==="
cd ../S3
docker-compose up -d

echo "=== Iniciando S4 (PHP-FPM Upload) ==="
cd ../S4
docker-compose up -d

echo "=== Iniciando S5 (Apache Images) ==="
cd ../S5
docker-compose up -d

echo "=== Iniciando S6 (Apache Assets) ==="
cd ../S6
docker-compose up -d

echo "=== Iniciando S1 (Apache Proxy) ==="
cd ../S1
docker-compose up -d

echo ""
echo "=== Estado de contenedores ==="
docker ps

echo ""
echo "=== Verificando red ==="
docker network inspect app-network

echo ""
echo "✅ Todos los servicios han sido iniciados"
echo ""
echo "Accede a tu aplicación en:"
echo "  http://tu-ip-publica-aws/"
```

**Opción B: Manual paso a paso**

```bash
# 1. MySQL
cd S7 && docker-compose up -d && cd ..
sleep 10

# 2. PHP-FPM
cd S2 && docker-compose up -d && cd ..
cd S3 && docker-compose up -d && cd ..
cd S4 && docker-compose up -d && cd ..

# 3. Apache estáticos
cd S5 && docker-compose up -d && cd ..
cd S6 && docker-compose up -d && cd ..

# 4. Apache Proxy
cd S1 && docker-compose up -d && cd ..
```

---

## Verificación

### Ver estado de contenedores

```bash
docker ps
```

Debe mostrar todos 7 servicios en estado `Up`.

### Ver logs de un servicio

```bash
# Apache Proxy
docker logs S1-apache-proxy

# PHP-FPM 1
docker logs S2-php-fpm-1

# MySQL
docker logs S7-mysql-db
```

### Probar conectividad

```bash
# Desde la instancia EC2
curl http://localhost/
curl http://localhost/extagram.php
curl http://localhost/assets/style.css
curl http://localhost/uploads/

# Desde tu máquina local
curl http://tu-ip-publica-aws/
```

### Verificar red Docker

```bash
docker network inspect app-network
```

### Conectar a MySQL

```bash
docker exec -it S7-mysql-db mysql -u app_user -p app_db

# Una vez dentro de MySQL:
# SELECT * FROM uploads;
# SELECT * FROM extagram;
```

---

## Troubleshooting

### Problema: Contenedores no se conectan entre sí

**Síntoma**: Error "Cannot resolve hostname S2-php-fpm-1"

**Solución**:

```bash
# Verificar que la red existe
docker network ls | grep app-network

# Recrear la red si no existe
docker network create app-network

# Conectar contenedores manualmente
docker network connect app-network S1-apache-proxy
docker network connect app-network S2-php-fpm-1
```

### Problema: MySQL no responde

**Síntoma**: PHP-FPM no puede conectar a MySQL

**Solución**:

```bash
# Esperar más tiempo a que MySQL inicie
sleep 20

# Verificar logs de MySQL
docker logs S7-mysql-db

# Reiniciar MySQL
docker restart S7-mysql-db
```

### Problema: Puerto 80 en uso

**Síntoma**: Error "Address already in use"

**Solución**:

```bash
# Ver qué ocupa el puerto 80
sudo lsof -i :80

# Matar el proceso o detener contenedor
docker stop S1-apache-proxy
sudo kill -9 PID
```

### Problema: Uploads no se guardan

**Síntoma**: Archivos subidos desaparecen

**Solución**:

```bash
# Verificar que el volumen existe
docker volume ls | grep shared-uploads

# Ver contenido del volumen
docker run -it --rm -v shared-uploads:/data alpine ls -la /data

# Recrear volumen si está corrupto
docker volume rm shared-uploads
docker volume create shared-uploads
```

### Problema: Apache Proxy retorna 502

**Síntoma**: "Bad Gateway" al acceder

**Solución**:

```bash
# Ver logs del proxy
docker logs S1-apache-proxy | tail -50

# Verificar que los backends están corriendo
docker ps | grep S2
docker ps | grep S3
docker ps | grep S4

# Reiniciar Apache Proxy
docker restart S1-apache-proxy
```

---

## Monitoreo

### Ver recursos usados

```bash
docker stats
```

### Ver logs en tiempo real

```bash
# Todos los servicios
docker-compose -f S1/docker-compose.yml \
                -f S2/docker-compose.yml \
                -f S3/docker-compose.yml \
                -f S4/docker-compose.yml \
                -f S5/docker-compose.yml \
                -f S6/docker-compose.yml \
                -f S7/docker-compose.yml logs -f

# O por servicio individual
docker logs -f S1-apache-proxy
```

### Backup de base de datos

```bash
docker exec S7-mysql-db mysqldump -u app_user -papp_password app_db > backup.sql
```

### Restaurar base de datos

```bash
docker exec -i S7-mysql-db mysql -u app_user -papp_password app_db < backup.sql
```

---

## Mantenimiento

### Detener todos los servicios

```bash
# Opción 1: Script individual
cd S1 && docker-compose down && cd ..
cd S2 && docker-compose down && cd ..
# ... etc para todos

# Opción 2: Todos a la vez
for dir in S{1..7}; do (cd $dir && docker-compose down); done
```

### Actualizar imagen de un servicio

```bash
# S1: Apache
cd S1
docker-compose pull
docker-compose up -d
cd ..

# S2, S3, S4: Reconstruir desde Dockerfile
cd S2
docker-compose build --no-cache
docker-compose up -d
cd ..
```

### Limpiar sistema Docker

```bash
# Eliminar contenedores parados
docker container prune -f

# Eliminar imágenes no usadas
docker image prune -a -f

# Eliminar volúmenes no usados
docker volume prune -f

# ⚠️ CUIDADO: Eliminar TODO (excepto volúmenes importantes)
docker system prune -a -f
```

---

## Tabla Resumen

| Servicio | Puerto | Rol | Volúmenes | Dependencias |
|----------|--------|-----|-----------|--------------|
| S1 | 80, 443 | Proxy inverso | logs/ | S2, S3, S4, S5, S6 |
| S2 | 9000 (interno) | PHP-FPM | app/, uploads | S7 |
| S3 | 9000 (interno) | PHP-FPM | app/, uploads | S7 |
| S4 | 9000 (interno) | PHP-FPM Upload | app/, uploads | S7 |
| S5 | 80 (interno) | Apache Images | uploads | - |
| S6 | 80 (interno) | Apache Assets | assets | - |
| S7 | 3306 (interno) | MySQL | mysql_data | - |

---

## Flujo de Peticiones

```
Cliente HTTP (tu-ip-publica:80)
    ↓
S1 (Apache Proxy)
    ├→ /extagram.php → Balanceado entre S2 y S3
    ├→ /upload.php → S4
    ├→ /uploads/* → S5 (Apache)
    ├→ /assets/* → S6 (Apache)
    └→ /* → S2 (por defecto)
    
Todos acceden a S7 (MySQL) para persistencia
```

---

## Notas Importantes

1. **No expongas MySQL directamente**: No abrir puerto 3306 al público
2. **Contraseñas**: Cambiar en producción (root_password, app_password)
3. **SSL/TLS**: Configurar certificados Let's Encrypt en S1 para HTTPS
4. **Backup**: Hacer backup regular de `mysql_data` y `shared-uploads`
5. **Logs**: Monitorear regularmente `/var/log/docker/`

---

## Contacto y Soporte

Para problemas o preguntas:
1. Ver logs: `docker logs [nombre-contenedor]`
2. Verificar red: `docker network inspect app-network`
3. Verificar volúmenes: `docker volume ls`
4. Reconstruir servicios si es necesario

---

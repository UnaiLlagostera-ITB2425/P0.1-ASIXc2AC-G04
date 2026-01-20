# Contratiempos

##  Colapso de RAM en t2.micro

Durante la **Sprint Dos**, al separar los servicios en contenedores Docker independientes, la instancia EC2 **t2.micro** experimentó un colapso total de recursos provocado por el consumo excesivo de RAM de la base de datos MySQL. El problema se manifestó inmediatamente al ejecutar `docker-compose up -d` en los servicios S5, S6 y S7, colapsando la máquina al reiniciarse automáticamente.

---

## Contexto del Problema

### Infraestructura Inicial
- **Instancia EC2**: t2.micro (1 GB RAM)
- **Servicios levantados**: S5, S6, S7
- **Configuración**: `restart: unless-stopped` en todos los contenedores

### Servicio Problemático
El servicio **S7 (MySQL 8.0)** fue identificado como el principal consumidor de RAM. La imagen oficial de MySQL 8.0 requiere recursos significativos que excedían la capacidad disponible en t2.micro.


### Síntomas

#### Momento de Aparición
El colapso ocurrió **instantáneamente** tras ejecutar `docker-compose up -d` en S7, sin necesidad de tráfico adicional o tiempo de ejecución.

#### Manifestaciones del Fallo

**Conectividad de Red**
- SSH dejó de responder inmediatamente
- PING no obtenía respuesta
- La máquina quedó completamente inaccesible

**Peticiones HTTP**
- Las respuestas de HTTP tardaban **minutos** en completarse (exageradamente lentas)
- Algunos clientes experimentaban timeout antes de recibir respuesta

**Consumo de Recursos**
- La base de datos MySQL consumía la mayoría del RAM disponible
- La máquina no tenía capacidad de memory swapping

#### Comportamiento Crítico
El problema era **recurrente y automático**: cada vez que la máquina se reiniciaba, el docker-compose con `restart: unless-stopped` levantaba S7 automáticamente, colapsando la instancia nuevamente. Esto creaba un ciclo de fallo donde era imposible acceder a la máquina para detener servicios manualmente.

---

### Análisis de Causas

#### Causa Raíz
**MySQL 8.0 requiere más RAM que la disponible en t2.micro (1 GB)**

La imagen oficial de MySQL 8.0 tiene buffers, cachés y procesos internos que juntos consumen aproximadamente 600-800 MB de RAM solo al iniciar. Sumado a:
- Sistema operativo Ubuntu (~200-300 MB)
- Docker daemon y otros servicios

La máquina agotaba el RAM disponible en segundos, provocando que el kernel bloqueara nuevos procesos (incluyendo SSH y servicios esenciales).

#### Factores Agravantes

1. **Política de reinicio agresiva**: `restart: unless-stopped` en S7 reiniciaba el contenedor constantemente al fallar, perpetuando el ciclo de colapso
2. **Sin swap configurado**: La máquina no tenía memoria virtual configurada para paliar picos de consumo
3. **Sin límites de RAM por contenedor**: No había restricciones en `docker-compose.yml` para limitar el consumo de MySQL

---

### Solución Implementada

#### Upgrade de Instancia
Se realizó un cambio de tipo de instancia:

| Parámetro | t2.micro | t3.small |
|-----------|----------|----------|
| RAM | 1 GB | 2 GB |
| vCPUs | 1 | 2 |
| Almacenamiento | EBS | EBS |
| Costo | ~$9.50/mes | ~$18/mes |

### Configuración Adicional
Se agregó **4 GB de swap** a la instancia t3.small para proporcionar un colchón de memoria virtual ante picos de consumo inesperados.

### Resultado
Con la instancia t3.small y 4 GB de swap configurado:
- La máquina enciende correctamente
- Los servicios S5, S6, S7 se levantan sin problemas
- SSH responde normalmente
- Peticiones HTTP se completan en tiempos aceptables

---

### Estado Actual

**Instancia**: t3.small (2 GB RAM + 4 GB swap)  
**Servicios activos**: S5 (Apache Images), S6 (Apache Assets), S7 (MySQL 8.0)  
**Estado**: Estable  
**Última revisión**: 20 de enero de 2026

---


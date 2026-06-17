# 🎯 Guía Oficial y Documentación del Proyecto Final — SIS313
## Proyecto 22: FleetTrack - Arquitectura de Alta Disponibilidad y Telemetría
> **Universidad Mayor, Real y Pontificia de San Francisco Xavier de Chuquisaca** > **Facultad de Tecnología / Carrera de Ingeniería de Sistemas** > **Docente:** Ing. Marcelo Quispe Ortega  
> **Semestre:** 1/2026  

---

## 👥 Integrantes y Asignaciones

| Integrante | VM | IP Real | Rol Principal |
|------------|----|---------|---------------|
| **Adalid** | VM196 | `192.168.100.196` | Maestro OLTP (MariaDB) y Respaldos Lógicos (DBA) |
| **Dylan** | VM197 / 200 / 201 | `192.168.100.197` / `.200` / `.201` | Orquestación, Bóveda Standby y Telemetría |
| **Matias** | VM195 | `192.168.100.195` | Escudo Perimetral (HAProxy) y Balanceo |
| **Genaro** | VM198 / VM199 | `192.168.100.198` / `.199` | Aplicación Node.js y Caché en Memoria (Redis) |

> **Segmento de Red:** Redes Lógicas Segmentadas | **Enrutamiento:** Estático Inter-VLAN  
> **Configuración de red:** Configuración nativa en Netplan mediante 4 VLANs (10, 20, 30, 40) para separar tráfico Perimetral, App, Soporte y Datos.

---

## 🎯 Objetivos del Proyecto

### Objetivo General
Diseñar, configurar y desplegar una infraestructura de red segmentada y de Alta Disponibilidad (HA) para el ecosistema FleetTrack, garantizando la resiliencia integral, la tolerancia a fallos y la continuidad operacional del servicio ante caídas críticas de infraestructura.

### Objetivos Específicos
1. **Balanceo de Carga Perimetral:** Implementar HAProxy (Capa 7) para distribuir equitativamente el tráfico HTTP hacia múltiples nodos web, evitando la saturación.
2. **Persistencia y Replicación:** Configurar un clúster MariaDB en topología Maestro-Bóveda (Standby) que garantice el respaldo de los registros.
3. **Orquestación Inteligente:** Integrar ProxySQL junto con un script Watchdog para automatizar el failover de la base de datos sin generar estados de "Split-Brain".
4. **Desacoplamiento de Caché:** Aislar el manejo de sesiones en un nodo exclusivo de Redis, optimizando el consumo de CPU/RAM de los servidores de aplicación.
5. **Telemetría y Resiliencia:** Desplegar Prometheus y Grafana para observabilidad en tiempo real, junto con rutinas Cron en Bash para respaldos físicos automatizados.

---

## 🛠️ Distribución de Trabajo: Roles y Tareas

### 👤 Matias — Administrador de VM195 (Escudo Perimetral)
**¿Qué hizo y qué defenderá?** Es la primera línea de interacción con el cliente, encargado de recibir y distribuir la carga de trabajo.
* **Balanceo con HAProxy:** Configuración de algoritmos Round-Robin para repartir el tráfico HTTP de forma equitativa entre la VM197 y VM199.
* **Health Checks:** Implementación de métricas de estado en vivo para expulsar del pool a los nodos web que dejen de responder.

### 👤 Genaro — Administrador de VM198 y VM199 (Backend y Caché)
**¿Qué hizo y qué defenderá?** Responsable de la lógica de negocio y la velocidad de respuesta temporal del sistema.
* **Clúster Node.js:** Despliegue del motor de ejecución backend en paralelo.
* **Caché Aislada:** Modificación de las directivas `bind` en Redis (VM198) para aceptar conexiones rápidas exclusivas de la VLAN de aplicación, aliviando a la base de datos.

### 👤 Dylan — Administrador de VM197, VM200 y VM201 (Orquestación, Bóveda y Telemetría)
**¿Qué hizo y qué defenderá?** Es el cerebro lógico detrás de la conmutación de la capa de datos y la observabilidad del sistema.
* **Replicación Asíncrona:** Sincronización y configuración de la Bóveda Standby (VM200) mediante `CHANGE MASTER TO` para asimilar datos en tiempo real.
* **ProxySQL y Failover Híbrido:** Diseño e implementación del "Watchdog" (VM197) para supervisar al Maestro y alterar rutas hacia la VM200 solo ante una caída confirmada.
* **Telemetría Central:** Despliegue de Prometheus y Grafana (VM201) para centralizar métricas de estado de todo el clúster.

### 👤 Adalid — Administrador de VM196 (Persistencia Maestra y DBA)
**¿Qué hizo y qué defenderá?** Custodio de la integridad transaccional activa, encargado del procesamiento de datos en tiempo real y rutinas de salvaguarda.
* **Gestión del Maestro OLTP:** Configuración central del motor MariaDB (VM196) optimizado para operaciones de lectura/escritura concurrentes.
* **Pruebas de Estrés:** Ejecución de ingeniería del caos validando el límite de ruptura (Sysbench OLTP) alcanzando más de 17,000 QPS.
* **Plan de Recuperación (DRP):** Programación de scripts forenses en Bash (`backup_db.sh`) para el volcado lógico automático y retención rotativa de la base de datos.

---

## 🗺️ Diagrama de Arquitectura de Red

```text
┌──────────────────────────────────────────────────────────────────┐
│          PROYECTO 22 — FleetTrack Alta Disponibilidad            │
│         Red: VLANs 10, 20, 30, 40 | Enrutamiento Estático        │
│                 SIS313 USFX | Semestre 1/2026                    │
└──────────────────────────────────────────────────────────────────┘

                       [Peticiones HTTP]
                               │ 
                               ▼
                       ┌─────────────────────┐
                       │ VM195 — Perimetral  │
                       │ Matias              │
                       │ 192.168.100.195     │
                       │ HAProxy Load Bal.   │
                       └─────────┬───────────┘
                                 │ Balanceo Round-Robin
                ┌────────────────┴────────────────┐
                ▼                                 ▼
      ┌─────────────────────┐           ┌─────────────────────┐
      │ VM197 — Nodo App 1  │           │ VM199 — Nodo App 2  │
      │ Dylan / Genaro      │           │ Genaro              │
      │ 192.168.100.197     │           │ 192.168.100.199     │
      │ Node.js + ProxySQL  │           │ Node.js Backend     │
      │ Watchdog Script     │           │                     │
      └─────────┬───────────┘           └─────────┬───────────┘
                │                                 │
                ├─────────────────────────────────┘
                │ Enrutamiento BD (Puerto 6033)
                ▼
      ┌─────────────────────┐           ┌─────────────────────┐
      │ ProxySQL (En VM197) │           │ VM198 — Caché       │
      └─────────┬───────────┘           │ Genaro              │
                │ R/W Split             │ 192.168.100.198     │
       ┌────────┴────────┐              │ Redis In-Memory     │
       ▼                 ▼              └─────────────────────┘
┌────────────┐ ┌────────────┐ 
│ VM196      │ │ VM200      │           ┌─────────────────────┐
│ Adalid     │ │ Dylan      │           │ VM201 — Telemetría  │
│ .100.196   │ │ .100.200   │           │ Dylan               │
│ MariaDB(M) │ │ MariaDB(S) │           │ 192.168.100.201     │
└────────────┘ └────────────┘           │ Prometheus+Grafana  │
      Replicación Continua              └─────────────────────┘
```
🧪 Conclusiones del Proyecto
Tras la culminación del despliegue y las fases de pruebas de estrés sobre la infraestructura de FleetTrack, se extraen las siguientes conclusiones técnicas:

Eficiencia en la Conmutación de Datos: La implementación de "Deep Health Checks" junto con el script Watchdog demostró ser vital. Asumir la Alta Disponibilidad únicamente por la respuesta de la capa web es un sesgo arquitectónico; el failover supervisado evitó desajustes transaccionales graves en la base de datos.

Desacoplamiento Efectivo: Separar la caché (VM198), la orquestación (VM197) y la bóveda de persistencia (VM200) previno la contención de recursos. La memoria RAM de los nodos de Node.js operó de manera óptima al delegar las sesiones web a Redis.

Resiliencia Comprobada bajo Estrés: Las auditorías con Sysbench validaron que la configuración nativa de MariaDB y la red soportaron cargas extremas (17,581 QPS y 879 TPS), comprobando que el límite de ruptura de la infraestructura está por encima de las exigencias estándar empresariales.

Sostenibilidad Operativa mediante Automatización: La orquestación no exime la necesidad de respaldos físicos. Las rutinas automatizadas de cron para el backup_db.sh en la VM196 aseguran que, incluso ante una falla catastrófica de ambas VMs de datos, el RTO (Recovery Time Objective) se mantenga en márgenes viables.

📌 Guía de Comandos Rápidos para la Defensa Final
Para la demostración en vivo frente al docente, se utilizará la siguiente batería de comandos de validación:

## ====== PASO 1: BALANCEO PERIMETRAL (VM195) ======
## Verificar estado del servicio HAProxy
```bash
sudo systemctl status haproxy --no-pager
```
## Monitorizar los logs en vivo para ver la distribución de tráfico
```bash
sudo tail -f /var/log/haproxy.log
```
## ====== PASO 2: ORQUESTACIÓN Y ENRUTAMIENTO (VM197) ======
## Verificar conexiones activas en ProxySQL
```bash
mysql -u admin -padmin -h 127.0.0.1 -P 6032 -e "SELECT * FROM stats_mysql_connection_pool;"
```
## Validar el script Watchdog
```bash
cat /home/adming13/watchdog_ha.sh
```
## ====== PASO 3: CACHÉ EN MEMORIA (VM198) ======
## Verificar que Redis está escuchando y aislando correctamente
```bash
sudo ss -tulpn | grep 6379
redis-cli ping
```
## ====== PASO 4: PERSISTENCIA Y REPLICACIÓN (VM196 / VM200) ======
## En la VM200: Validar el estado de la bóveda Standby
```bash
mysql -u root -p -e "SHOW SLAVE STATUS\G" | grep "Seconds_Behind_Master\|Slave_IO_Running"
```
## En la VM196: Ejecutar prueba rápida de Sysbench (si es solicitada)
```bash
sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-socket=/run/mysqld/mysqld.sock --mysql-user=root --tables=10 --table-size=100000 --threads=16 time=30 run
```
## En la VM196: Mostrar la automatización de respaldos
```bash
crontab -l
ls -lh /opt/fleettrack/backups/
```
## ====== PASO 5: TELEMETRÍA (VM201) ======
## Validar la extracción de métricas del nodo de datos (MariaDB Exporter)
```bash
curl [http://192.168.100.196:9104/metrics](http://192.168.100.196:9104/metrics) | head -n 15
```
Proyecto 22 — SIS313 USFX 2026 | Adalid · Dylan · Matias · Genaro

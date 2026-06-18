# 🚀 Proyecto Final SIS313: FleetTrack - Arquitectura de Alta Disponibilidad y Telemetría

> **Asignatura:** SIS313: Infraestructura, Plataformas Tecnológicas y Redes<br>
> **Semestre:** 1/2026<br>
> **Docente:** Ing. Marcelo Quispe Ortega

## 👥 Miembros del Equipo (Grupo FleetTrack - proyecto 22)

| Nombre Completo | Rol en el Proyecto | Contacto (GitHub/Email) |
| :--- | :--- | :--- |
| Adalid Gutiérrez Torricos | Administrador de Base de Datos (DBA) y Persistencia | @adalidg |
| Dylan Elyazar Barahona Romero | Arquitecto de Persistencia en Standby (Réplica) | @Elyazarrr |
| Matias Jonathan Quispe Martinez | Ingeniero de Redes Perimetrales y Balanceo de Carga | @QuispeMartínezMatias |
| Weimar Genaro Zamorano Hurtado | Desarrollador Backend, Orquestación y Caché | @Genaro007 |

## 🎯 I. Objetivo del Proyecto

> **Objetivo:** Diseñar, configurar y desplegar una infraestructura de red segmentada y de Alta Disponibilidad (HA) para el ecosistema FleetTrack, integrando un clúster de bases de datos MariaDB con replicación Maestro-Bóveda, balanceo de carga estricto Capa 7 mediante HAProxy, enrutamiento inteligente de consultas con ProxySQL, gestión de estado en memoria con Redis y telemetría avanzada, garantizando la resiliencia integral, la tolerancia a fallos y la continuidad operacional del servicio ante caídas críticas de infraestructura.

## 💡 II. Justificación e Importancia

> **Justificación:** Este proyecto resuelve el problema crítico de los Puntos Únicos de Fallo (SPOF) presentes en arquitecturas monolíticas tradicionales. A nivel empresarial, la interrupción del servicio genera pérdidas incalculables; por ello, implementamos una estrategia de "Alta Disponibilidad Híbrida" (T2). En la capa web, garantizamos la atención fluida de clientes mediante un balanceador perimetral. En la capa de datos (la más vulnerable), mitigamos el riesgo de corrupción por "Split-Brain" aislando la Bóveda de Persistencia (Standby) y automatizando su promoción únicamente a través de un script Watchdog orquestado, asegurando así la integridad de la información y reforzando las políticas de continuidad de negocio y seguridad perimetral (T5).

## 🛠️ III. Tecnologías y Conceptos Implementados

### 3.1. Tecnologías Clave

* **HAProxy:** Balanceador de carga perimetral de alta eficiencia que opera en la Capa 7 del modelo OSI, distribuyendo el tráfico HTTP entrante hacia los nodos de la aplicación usando un algoritmo Round-Robin.
* **MariaDB:** Motor de Base de Datos Relacional configurado en una topología Maestro-Esclavo (Bóveda Standby), asegurando la persistencia de los registros y tolerando caídas de hardware.
* **ProxySQL & Watchdog Script:** ProxySQL actúa como un enrutador inteligente de consultas SQL, mientras que el Watchdog monitoriza el clúster de datos para ejecutar un failover automático si el nodo Maestro deja de responder.
* **Node.js:** Entorno de ejecución asíncrono utilizado para construir y servir la lógica de negocio del backend en múltiples nodos paralelos.
* **Redis:** Motor de estructura de datos en memoria, desplegado en un nodo aislado, fundamental para el manejo ultrarrápido de sesiones web y la reducción de la carga en la base de datos principal.
* **Prometheus & Sysbench:** Prometheus (junto a Node-Exporter) extrae métricas de consumo de CPU/RAM, mientras que Sysbench valida la capacidad estocástica del servidor de datos bajo cargas de estrés OLTP.

### 3.2. Conceptos de la Asignatura Puestos en Práctica (T1 - T6)

* ✅ **Alta Disponibilidad (T2) y Tolerancia a Fallos:** Replicación de bases de datos con sincronización de estado, y uso de HAProxy para mitigar la caída de nodos web sin afectar al usuario final.
* ✅ **Seguridad y Hardening (T5):** Aislamiento de los motores de bases de datos limitando su `bind-address` exclusivamente a IPs de la red interna, preveniendo la sordera de red (Error 111) y accesos no autorizados.
* ✅ **Automatización y Gestión (T6):** Desarrollo de scripts en Bash para la ejecución de respaldos físicos automatizados en horarios de bajo tráfico, integrados directamente al Crontab del sistema (`backup_db.sh`).
* ✅ **Balanceo de Carga/Proxy (T3/T4):** Implementación de HAProxy para distribuir las peticiones HTTP a la capa de Node.js, apoyado por ProxySQL para el manejo eficiente del pool de conexiones hacia MariaDB.
* ✅ **Monitoreo (T4/T1):** Configuración de Node-Exporter en los puertos 9100 para garantizar la observabilidad del sistema y reaccionar proactivamente ante cuellos de botella.
* ✅ **Networking Avanzado (T3):** Segmentación lógica de la infraestructura mediante la configuración nativa en Netplan de 4 VLANs (10, 20, 30, 40), separando el tráfico Perimetral, de Aplicación, de Soporte y de Datos.
## 🌐 IV. Diseño de la Infraestructura y Topología

### 4.1. Diseño Esquemático

> ![Arquitectura de la red](infraestructura (1).png)

| VM/Host | Rol | IP Física | Red Lógica (VLAN) | SO |
| :--- | :--- | :--- | :--- | :--- |
| **VM 195** | Escudo Perimetral (HAProxy) | 192.168.100.195 | VLAN Perimetral | Ubuntu 24.04 Server |
| **VM 197** | Nodo App 1 + ProxySQL + Watchdog | 192.168.100.197 | VLAN App / Control | Ubuntu 24.04 Server |
| **VM 199** | Nodo App 2 (Redundancia Web) | 192.168.100.199 | VLAN App | Ubuntu 24.04 Server |
| **VM 198** | Caché en Memoria Central (Redis) | 192.168.100.198 | VLAN Soporte | Ubuntu 24.04 Server |
| **VM 196** | Base de Datos (Maestro OLTP) | 192.168.100.196 | VLAN Datos | Ubuntu 24.04 Server |
| **VM 200** | Base de Datos (Bóveda Standby) | 192.168.100.200 | VLAN Datos | Ubuntu 24.04 Server |
| **VM 201** | Telemetría Central (Grafana) | 192.168.100.201 | VLAN Gestión | Ubuntu 24.04 Server |

### 4.2. Estrategia Adoptada (Opcional)

* **Estrategia de Desacoplamiento de Recursos:** A diferencia de diseños monolíticos, decidimos aislar el motor de Redis en la VM 198. Esto previene que la caché consuma la memoria RAM necesaria para el procesamiento de hilos de Node.js, logrando un escalamiento horizontal limpio.
* **Estrategia de Failover Conservador:** La promoción de la VM 200 (Bóveda) no es inmediata al primer fallo de red para evitar escenarios de falsos positivos. Se implementó un "Watchdog" en la VM 197 que verifica métricas consistentes antes de realizar el cambio de estado (`read_only=0`), protegiendo la integridad del esquema transaccional.
* * **Centralización de Telemetría (VM 201):** Se destinó un nodo exclusivo para la recolección y visualización de métricas (Prometheus Server / Grafana). Aislar el monitoreo garantiza que, en caso de un ataque de denegación de servicio (DDoS) o colapso por saturación en la capa de aplicación, los dashboards de administración sigan operativos y reportando alertas.

## 📋 V. Guía de Implementación y Puesta en Marcha

### 5.1. Pre-requisitos
* 7 Máquinas Virtuales provisionadas con Ubuntu 24.04 LTS.
* Interfaz de red troncal configurada con etiquetado 802.1Q para soportar el enrutamiento inter-VLAN.
* Acceso con privilegios de superusuario (`sudo/root`) en todos los nodos.

### 5.2. Despliegue (Ejecución de la Automatización)
1.  **Capa de Persistencia (VM 196 y VM 200):** Instalar MariaDB, configurar los `server-id` únicos en `/etc/mysql/mariadb.conf.d/50-server.cnf` y establecer la replicación `CHANGE MASTER TO` en la VM 200.
2.  **Capa de Aplicación y Soporte (VM 197, 198, 199):** Desplegar Node.js con PM2 en las VMs 197/199. Instalar y asegurar Redis en la VM 198 modificando la directiva `bind` para aceptar conexiones de la VLAN App.
3.  **Capa Perimetral (VM 195):** Instalar HAProxy, definir los nodos web en la sección `backend` del archivo de configuración, habilitando las directivas `check` para monitoreo de estado en vivo.
4. **Capa de Observabilidad (VM 201):** Levantar el servidor central de Prometheus para hacer "scraping" a los puertos 9100 de las demás VMs, y enlazar Grafana para la visualización del estado de salud del clúster en tiempo real.

### 5.3. Ficheros de Configuración Clave
* `/etc/haproxy/haproxy.cfg`: Define los algoritmos de balanceo y las métricas de health-check hacia los Nodos Node.js.
* `/opt/fleettrack/backups/backup_db.sh`: Script forense para el volcado relacional y rotación de históricos locales.
* `[Añadir ruta del script Watchdog]`: Archivo de orquestación responsable de alterar las rutas en ProxySQL durante contingencias.
* `/etc/netplan/01-network-manager-all.yaml`: Fichero de declaración de subinterfaces lógicas y VLANs.

**Incluir además los archivos de configuración y software a utilizar dentro del proyecto y organizados en carpetas.**

## ⚠️ VI. Pruebas y Validación

| Prueba Realizada | Resultado Esperado | Resultado Obtenido |
| :--- | :--- | :--- |
| **Ingeniería del Caos (Simulación de Caída VM 196)** | El sistema Watchdog detecta el timeout del Maestro y ordena a ProxySQL redirigir las consultas de escritura hacia la VM 200. | [OK/FALLIDO - Confirmar test de Watchdog] |
| **Prueba de Carga y Ruptura (Sysbench OLTP)** | La base de datos soporta peticiones masivas (1 millón de registros) sin colapsar el daemon y antes de generar encolamientos en ProxySQL. | **OK:** Rendimiento sostenido a 17,581 QPS y 879 TPS a máxima exigencia de CPU. |
| **Validación Perimetral (Balanceo HAProxy)** | Las peticiones HTTP externas son distribuidas de manera equitativa, comprobable apagando la VM 197 y observando la VM 199 asumir la carga. | [OK/FALLIDO - Confirmar test de Matías] |

## 📚 VII. Conclusiones y Lecciones Aprendidas

La implementación de este proyecto demostró que el verdadero desafío de la Alta Disponibilidad no radica únicamente en desplegar los servicios, sino en orquestar su comunicación segura a través de VLANs segmentadas. El aislamiento de los motores de caché (Redis en VM 198) y la asignación de roles estrictos a las bases de datos (VM 196 como Maestro, VM 200 como Bóveda pasiva) probaron ser decisiones arquitectónicas fundamentales para evitar cuellos de botella. 

Adicionalmente, se aprendió que la monitorización pasiva no es suficiente; la aplicación de pruebas de estrés comprobables (como los 17,581 QPS logrados con Sysbench) y la configuración de rutinas automáticas de recuperación ante desastres (cron jobs para respaldos) son requisitos indispensables para que una infraestructura pase de ser un escenario académico a un entorno con grado de producción empresarial confiable.


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

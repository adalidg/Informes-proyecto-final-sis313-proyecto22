# Bóveda de Persistencia Standby (Nodo 200)

## 1. Perfil del Nodo
* **Rol en la Topología:** Réplica de Base de Datos en caliente (Standby / Failover Target).
* **Sistema Operativo:** Ubuntu 24.04 Server
* **IP Física:** `192.168.100.200`
* **Motor Transaccional:** MariaDB
* **Estado Predeterminado:** Solo Lectura (`read_only = 1`)

---

## 2. Aprovisionamiento Base (Capa de Sistema Operativo)

Antes de someter el nodo a la topología distribuida, se garantizó la integridad de los paquetes del sistema y se instaló el motor relacional crudo.

### 2.1. Actualización de Repositorios y Paquetes
Se ejecutó la sincronización de las listas de paquetes para asegurar que las dependencias de red y los parches de seguridad estuvieran en su versión más estable:

```bash
sudo apt update && sudo apt upgrade -y
```

## 3. Preparación Perimetral (Capa de Red)

Para garantizar que la bóveda de datos pueda comunicarse con el Maestro (VM 196) para la asimilación de registros, y recibir la conmutación de tráfico desde el Enrutador (ProxySQL en VM 197), el nodo fue expuesto matemáticamente hacia la VLAN interna, rompiendo su aislamiento por defecto.

### 3.1. Configuración del Socket de Escucha
Por defecto, MariaDB opera bajo un entorno hermético local. Se perforó este aislamiento editando el archivo de configuración central:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Se modificó la directiva bind-address para permitir la escucha en todas las interfaces de red:

```bash
bind-address = 0.0.0.0
```

Aplicación de los cambios en Capa 4 mediante el reinicio del motor:

```bash
sudo systemctl restart mariadb
```

### 3.2. Instalación del Motor Transaccional
Se instaló el servicio MariaDB Server desde los repositorios oficiales de la distribución, desplegando la arquitectura binaria necesaria para la persistencia de datos:

```bash
sudo apt install mariadb-server mariadb-client -y
```

### 3.3. Validación y Persistencia del Servicio
Se habilitó el motor para garantizar su arranque automático ante cualquier corte de energía o reinicio físico del servidor, asegurando la resiliencia base:

```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo systemctl status mariadb
```

## 4. Endurecimiento Inicial (Hardening de Capa de Datos)
No se puede exponer una bóveda a una red, ni siquiera a una VLAN interna, sin antes blindar sus credenciales predeterminadas. Se ejecutó el script de seguridad forense integrado del motor para aniquilar las vulnerabilidades de fábrica.

### 4.1. Ejecución del Script de Seguridad Base
Se ejecutó la herramienta de securización interactiva:

```bash
sudo mysql_secure_installation
```

+ Políticas de seguridad estrictas aplicadas durante el proceso:

+ Contraseña de Root: Establecimiento de una llave criptográfica fuerte.

+  Usuarios Anónimos: Eliminados para evitar ingresos sin autenticación.

+ Login Remoto de Root: Deshabilitado matemáticamente (el acceso administrativo maestro solo es posible desde localhost).

+ Base de Datos 'Test': Purgada del motor para eliminar vectores de ataque lógicos.

+ Recarga de Privilegios: Ejecución de FLUSH PRIVILEGES para forzar la lectura inmediata de las nuevas reglas de control de acceso.

### 4.2. Endurecimiento del Cortafuegos (UFW)
Para rechazar tráfico espurio y asegurar el conducto de la replicación evitando rechazos a nivel de sistema operativo (Error 111 TCP RST), se habilitó estrictamente el puerto de base de datos:

```bash
sudo ufw allow 3306/tcp
sudo ufw reload
```

## 5. Identidad y Lógica de Replicación (Capa de Configuración)

En una topología distribuida, cada motor de datos necesita una identidad matemática única para no generar colisiones de estado o bucles infinitos durante la replicación. Se intervino nuevamente el archivo de configuración central para establecer estas directivas.

### 5.1. Asignación del Server ID y Bloqueo de Estado
Se editó el archivo de configuración:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Se inyectaron las siguientes variables en el bloque [mysqld] para definir el comportamiento de la réplica:

### Identificador único en el clúster (se usó el último octeto de la IP)

```bash
server-id = 200
```

### Activación del log binario (Requerido para poder asumir el rol de Maestro en el Failover)

```bash
log_bin = /var/log/mysql/mysql-bin.log
```

### Candado estructural: Impide que cualquier usuario (excepto la replicación y SUPER) modifique datos

```bash
read_only = 1
```

Se aplicaron los cambios reiniciando el servicio:

```bash
sudo systemctl restart mariadb
```

## 6. Enlace de Asimilación (Sincronización Master-Slave)
Con la bóveda bloqueada y escuchando en la red, se procedió a establecer el túnel criptográfico hacia el Maestro Original (VM 196) para iniciar la copia en tiempo real de los registros de FleetTrack.

### 6.1. Ejecución del Comando de Enganche
Ingresando a la consola de MariaDB local (sudo mysql -u root -p), se apuntó el motor esclavo hacia las coordenadas exactas del Maestro usando las credenciales unificadas de replicación:

```bash
CHANGE MASTER TO 
MASTER_HOST='192.168.100.196', 
MASTER_USER='replica_user', 
MASTER_PASSWORD='Replica_2026', 
MASTER_LOG_FILE='mysql-bin.000001', 
MASTER_LOG_POS=814;
```

#### (Nota: El archivo y la posición fueron extraídos previamente del Maestro ejecutando SHOW MASTER STATUS; en la VM 196).

### 6.2. Inicialización y Verificación Forense
Se encendió el hilo de asimilación y se verificó el estado de sincronización:

```sql
START SLAVE;
SHOW SLAVE STATUS \G
```

Criterios de éxito validados matemáticamente en la salida de la consola:

+ Slave_IO_Running: Yes (Conectividad de red confirmada).

+ Slave_SQL_Running: Yes (Ejecución de sentencias confirmada).

+ Seconds_Behind_Master: 0 (Sincronización en tiempo real lograda, desfase cero).

## 7. Inyección de Privilegios para el Orquestador SRE
Dado que esta bóveda está diseñada para la Alta Disponibilidad, necesita permitir que el script "Watchdog" (ubicado en la VM 197) ingrese y le quite el candado de lectura automáticamente en caso de que la VM 196 colapse.

### 7.1. Creación de la Credencial de Autoridad
Dentro de la consola de MariaDB de la VM 200, se forjó el usuario administrativo de Alta Disponibilidad:

```sql
CREATE USER 'ha_admin'@'%' IDENTIFIED BY 'Orquestador_2026';
GRANT SUPER, REPLICATION CLIENT, RELOAD, PROCESS ON *.* TO 'ha_admin'@'%';
FLUSH PRIVILEGES;
```

Con este comando, el Orquestador queda habilitado para ejecutar `SET GLOBAL read_only = 0;` en estado de emergencia, consumando el Failover sin intervención humana.

## 8. Telemetría y Observabilidad (Capa de Monitoreo)

Para erradicar la ceguera operativa y permitir que el servidor central de telemetría audite la salud transaccional del motor en tiempo real, se implementó el agente `mysqld_exporter` de Prometheus.

### 8.1. Aprovisionamiento Criptográfico para Telemetría
El agente exportador requiere acceso a la base de datos para extraer métricas de rendimiento (QPS, hilos, latencia), pero por seguridad, no debe tener acceso a los datos del negocio. Se creó un usuario con privilegios estrictamente limitados:

Ingresando a la consola de MariaDB (`sudo mysql -u root -p`):

```sql
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'Metricas_2026';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;
```

### 8.2. Despliegue y Configuración del Agente Exporter
Se configuró el archivo de credenciales oculto para que el servicio pueda autenticarse sin exponer la contraseña en los procesos del sistema operativo:

```bash
sudo nano /etc/.mysqld_exporter.cnf
```

Estructura del archivo de credenciales:

```bash
[client]
user=exporter
password=Metricas_2026
```

Se procedió a descargar el binario oficial de `mysqld_exporter`, ubicarlo en las rutas de ejecución del sistema (`/usr/local/bin/`), y se orquestó mediante un servicio de systemd para asegurar su persistencia:

```bash
sudo nano /etc/systemd/system/mysqld_exporter.service
```

Definición del servicio:

```sql
[Unit]
Description=Prometheus MySQL Exporter
After=network.target

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/mysqld_exporter \
--config.my-cnf /etc/.mysqld_exporter.cnf \
--collect.global_status \
--collect.info_schema.innodb_metrics \
--collect.auto_increment.columns \
--collect.info_schema.processlist \
--collect.binlog_size \
--collect.info_schema.tablestats \
--collect.global_variables \
--collect.info_schema.query_response_time \
--collect.info_schema.userstats \
--collect.info_schema.tables \
--collect.perf_schema.tablelocks \
--collect.perf_schema.file_events \
--collect.perf_schema.eventswaits \
--collect.perf_schema.indexiowaits \
--collect.perf_schema.tableiowaits \
--collect.slave_status \
--web.listen-address=0.0.0.0:9104

[Install]
WantedBy=multi-user.target
```

Arranque y habilitación del demonio de telemetría:

```bash
sudo systemctl daemon-reload
sudo systemctl start mysqld_exporter
sudo systemctl enable mysqld_exporter
```

### 8.3. Apertura Perimetral de Observabilidad
Finalmente, para permitir que el servidor Prometheus/Grafana externo pueda extraer las métricas recolectadas, se perforó el cortafuegos exclusivamente en el puerto de telemetría (9104):  

```Bash
sudo ufw allow 9104/tcp
sudo ufw reload
```

Con esta configuración, la VM 200 transmite en tiempo real su estado interno de replicación, uso de Buffer Pool y rendimiento de I/O, garantizando auditoría forense durante los simulacros de Failover.


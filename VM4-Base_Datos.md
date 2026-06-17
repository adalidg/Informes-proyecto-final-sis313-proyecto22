# 📑 Informe: VM196 - Base de Datos (Maestro OLTP)
> **Dirección IP:** `192.168.100.196`
> **Sistema Operativo:** Ubuntu 24.04 LTS Server
> **Servicio Principal:** MariaDB Server (Nodo Maestro Transaccional)

---

## 📌 1. Rol en la Arquitectura FleetTrack

La **VM 196** es el núcleo de persistencia de datos del proyecto. Funciona como el **Maestro Transaccional (OLTP)**, siendo el único nodo autorizado para procesar operaciones de escritura (`INSERT`, `UPDATE`, `DELETE`) enrutadas a través de ProxySQL (VM 197). 

Además de procesar la lógica de negocio, esta máquina es la encargada de generar los registros binarios (`binlogs`) en tiempo real, los cuales son consumidos de forma asíncrona por la Bóveda Standby (VM 200) para garantizar la tolerancia a fallos.

---

## ⚙️ 🛠️ 2. Instalación y Configuración del Motor MariaDB
Se instaló la versión nativa de MariaDB y se ejecutó la rutina de seguridad inicial para eliminar bases de datos de prueba y accesos anónimos.

2.1. Instalación y Hardening Básico
```bash
sudo apt update && sudo apt install mariadb-server -y
sudo mysql_secure_installation
# Se deshabilitó el login de root remoto y se eliminó la BD 'test'.
```
2.2. Configuración del Nodo Maestro (50-server.cnf)
Para habilitar la replicación hacia la VM 200 y optimizar el rendimiento, se modificó el archivo de configuración principal de MariaDB ubicado en /etc/mysql/mariadb.conf.d/50-server.cnf.

```bash
[mysqld]
# Aislamiento: El motor no escucha en 0.0.0.0. Solo acepta tráfico local y de la VLAN.
bind-address            = 192.168.100.196

# ================= REPLICACIÓN MAESTRO =================
# Identificador único del servidor en el clúster
server-id               = 1

# Habilitar el registro binario (Esencial para que la VM 200 copie los datos)
log_bin                 = /var/log/mysql/mysql-bin.log

# Durabilidad transaccional (1 = máxima seguridad para evitar pérdida de datos)
sync_binlog             = 1
expire_logs_days        = 7
max_binlog_size         = 100M

# ================= OPTIMIZACIÓN INNODB =================
# Ajuste de memoria para índices (Recomendado 50-70% de la RAM disponible)
innodb_buffer_pool_size = 1G
innodb_flush_log_at_trx_commit = 1
```

Reinicio para aplicar los parámetros:

```bash
sudo systemctl restart mariadb
```

🔐 3. Creación de Credenciales de Enrutamiento y Replicación
Como DBA del sistema, se procedió a crear los usuarios estrictamente necesarios con el principio de "mínimos privilegios", tanto para el balanceador como para la bóveda esclava.

```bash
-- Acceso local a MariaDB
sudo mysql -u root -p

-- 1. Crear el usuario para que la VM 200 (Standby) pueda leer los binlogs
CREATE USER 'replica_user'@'192.168.100.200' IDENTIFIED BY 'Replica_Secure_2026!';
GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'192.168.100.200';

-- 2. Crear el usuario para que el ProxySQL (VM 197) enrute el tráfico de Node.js
CREATE USER 'proxysql_user'@'192.168.100.197' IDENTIFIED BY 'ProxySQL_Pass_2026!';
GRANT SELECT, INSERT, UPDATE, DELETE ON fleettrack_db.* TO 'proxysql_user'@'192.168.100.197';

FLUSH PRIVILEGES;

-- 3. Verificar el estado del Maestro para configurar la VM 200
SHOW MASTER STATUS;
-- (Se toma nota del File: mysql-bin.000001 y del Position para dárselos a Dylan)
```

🛡️ 4. Seguridad Perimetral (UFW) y Telemetría
4.1. Firewall (Cortafuegos)
Se limitó el acceso al puerto 3306 (MariaDB) exclusivamente para el nodo orquestador y la bóveda standby, previniendo ataques de inyección SQL externos.

```bash
sudo ufw enable
sudo ufw allow 22/tcp
sudo ufw allow from 192.168.100.197 to any port 3306 proto tcp # ProxySQL
sudo ufw allow from 192.168.100.200 to any port 3306 proto tcp # Bóveda
sudo ufw reload
```

4.2. Telemetría y Monitoreo (Puerto 9104)
Para que el servidor central de telemetría (VM 201) extraiga los datos, se descargó y configuró mysqld_exporter.

```bash
# Se habilitó el puerto para el scraping de Prometheus
sudo ufw allow from 192.168.100.201 to any port 9104 proto tcp

# Ejecución del exportador (Previamente configurado con credenciales de lectura)
./mysqld_exporter --web.listen-address=":9104" &
```

💾 5. Plan de Recuperación ante Desastres (DRP)
A pesar de la Alta Disponibilidad, la replicación no protege contra un error humano (como un DROP TABLE). Por ello, se programó un script forense en Bash para realizar respaldos lógicos diarios.

5.1. Script de Backup: /opt/fleettrack/backups/backup_db.sh

```bash
#!/bin/bash
# Retención rotativa de 7 días
BACKUP_DIR="/opt/fleettrack/backups"
DATE=$(date +%Y%m%d_%H%M)
USER="root"
DB_NAME="fleettrack_db"

mkdir -p $BACKUP_DIR
mysqldump -u $USER $DB_NAME | gzip > $BACKUP_DIR/db_backup_$DATE.sql.gz

# Eliminar respaldos más antiguos a 7 días
find $BACKUP_DIR -type f -name "*.sql.gz" -mtime +7 -exec rm {} \;
```

5.2. Automatización con Cron
Se añadió el script al cronjob del usuario root para ejecutarse todos los días a las 03:00 AM.

```bash
sudo crontab -e
# Agregar la línea:
0 3 * * * /opt/fleettrack/backups/backup_db.sh
```

🧪 6. Pruebas de Estrés Transaccional (Sysbench)
Para validar el límite de ruptura de la configuración InnoDB antes de poner el servidor en producción, se ejecutó una prueba masiva simulando 1 millón de registros.

```bash
# 1. Preparar la base de datos de prueba
sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-socket=/run/mysqld/mysqld.sock --mysql-user=root --tables=10 --table-size=100000 prepare

# 2. Ejecutar la prueba con 32 hilos concurrentes
sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-socket=/run/mysqld/mysqld.sock --mysql-user=root --tables=10 --table-size=100000 --threads=32 time=60 run

# RESULTADO ALCANZADO:
# Transacciones: 879 TPS
# Consultas: 17,581 QPS (Queries per Second)
```

📚 7. Conclusiones del Nodo (DBA)
La configuración de este nodo demuestra que la persistencia en Alta Disponibilidad requiere una visión integral. La sola instalación del motor es insuficiente; habilitar el registro binario (log_bin) y establecer un server-id estricto fueron pasos vitales para permitir que la VM 200 asimile los datos.

Las pruebas estocásticas con Sysbench confirmaron que el tuning del innodb_buffer_pool_size a 1GB evitó que la CPU colapsara bajo cargas masivas (17,581 QPS), garantizando que el Maestro no será el cuello de botella cuando HAProxy envíe ráfagas de peticiones desde la capa web.
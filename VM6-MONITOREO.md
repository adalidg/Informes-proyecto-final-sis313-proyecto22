# Servidor Central de Telemetría y Observabilidad (Nodo 201)

## 1. Perfil del Nodo
* **Rol en la Topología:** Torre de Control (Recolección de métricas y visualización de dashboards).
* **Sistema Operativo:** Ubuntu 24.04 Server
* **IP Física:** `192.168.100.201`
* **Motores Core:** Prometheus (Scraper) y Grafana (Dashboard)

---

## 2. Aprovisionamiento del Motor de Recolección (Prometheus)

Para que Grafana pueda dibujar las gráficas (QPS, Latencia, Hilos), necesita una base de datos de series temporales que extraiga la información de los nodos. Se implementó Prometheus como el motor central de recolección.

### 2.1. Instalación y Despliegue
Se instaló el núcleo de Prometheus desde los repositorios oficiales:

```bash
sudo apt update
sudo apt install prometheus -y
```

### 2.2. Mapeo de la Topología (Scrape Config)
Se configuró el archivo central para instruir a Prometheus sobre qué IPs y puertos debe auditar dentro de la VLAN, específicamente apuntando a los exportadores de las bases de datos de Adalid y Dylan.

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Se inyectaron los objetivos (targets) en la sección `scrape_configs`:

```bash
scrape_configs:
  - job_name: 'mariadb_nodos'
    static_configs:
      - targets: ['192.168.100.196:9104', '192.168.100.200:9104']
```

Reinicio del servicio para aplicar la nueva matriz de recolección:

```bash
sudo systemctl restart prometheus
sudo systemctl enable prometheus
```

## 3. Aprovisionamiento de la Capa de Visualización (Grafana)
Con Prometheus almacenando los datos brutos, se instaló Grafana para transformar la matriz numérica en los paneles de auditoría visual (Dashboards) utilizados durante las pruebas de estrés.

### 3.1. Instalación y Persistencia del Servicio
Se instaló la paquetería de Grafana Server:

```bash
sudo apt-get install -y adduser libfontconfig1
sudo wget [https://dl.grafana.com/oss/release/grafana_10.2.2_amd64.deb](https://dl.grafana.com/oss/release/grafana_10.2.2_amd64.deb)
sudo dpkg -i grafana_10.2.2_amd64.deb
```

Habilitación del demonio para asegurar el arranque automático:

```bash
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

## 4. Endurecimiento Perimetral Local (UFW)
Para aislar la torre de control, se aplicó una política de denegación por defecto, abriendo estrictamente los puertos de operación:

+ Puerto 9090: Interfaz de Prometheus.

+ Puerto 3000: Interfaz Web de Grafana.

```bash
sudo ufw allow 9090/tcp
sudo ufw allow 3000/tcp
sudo ufw reload
```

## 5. Ingeniería de Acceso: Perforación de Red (Túnel SSH y Jump Host)
Dado que la VM 201 reside en una VLAN privada sin salida directa a la web pública, es matemáticamente imposible acceder a la interfaz de Grafana tipeando 192.168.100.201:3000 en un navegador externo.

Para que los arquitectos SRE pudieran auditar las pruebas de carga y los Failovers en tiempo real desde el exterior, se construyó un túnel criptográfico de Capa 3 utilizando un servidor proxy o "Jump Host".

### 5.1. Ejecución del Túnel de Redirección (Desde la máquina del cliente)
El administrador ejecutó el siguiente comando en su terminal física local, forzando un enrutamiento seguro que enlaza el puerto 3000 de su computadora (localhost) con el puerto 3000 de la VM 201, saltando a través del proxy perimetral público:

```bash
ssh -J usrproxy@201.131.45.42 -L 3000:192.168.100.201:3000 adming13@192.168.100.201
```

+ -J usrproxy@201.131.45.42: Establece el servidor de salto (Jump Host) para atravesar el cortafuegos principal.

+ -L 3000:192.168.100.201:3000: Vincula el puerto 3000 del operador al puerto 3000 del servidor Grafana destino.

Resultado Operativo: Mientras la terminal SSH se mantuvo abierta, el administrador pudo auditar la telemetría del clúster accediendo a `http://localhost:3000` desde su navegador local, garantizando observabilidad sin comprometer la seguridad perimetral de la VLAN.

# Modificación en la vm 197

# Capa de Enrutamiento de Datos y Orquestación HA (Nodo 197)

## 1. Perfil de Responsabilidad Compartida
* **Rol en la Topología:** Alojamiento de la Aplicación (Node.js) **Y** Cerebro de Enrutamiento de Base de Datos (ProxySQL + Watchdog).
* **Sistema Operativo:** Ubuntu 24.04 Server
* **IP Física:** `192.168.100.197`
* **Puertos Lógicos Críticos:** `3000` (App Node.js) y `6033` (ProxySQL)

---

## 2. Implementación del Enrutador Inteligente (ProxySQL)

Para evitar que la aplicación web se conecte directamente a los motores físicos y sufra caídas durante los bloqueos de base de datos, se interpuso ProxySQL como un balanceador de Capa 7 exclusivo para SQL.

### 2.1. Despliegue y Arranque
Se instaló el binario de ProxySQL y se habilitó su servicio para persistencia:

```bash
sudo apt-get install proxysql -y
sudo systemctl start proxysql
sudo systemctl enable proxysql
```

### 2.2. Configuración de Hostgroups (División Lógica)
Ingresando a la interfaz administrativa de ProxySQL (`mysql -u admin -padmin -h 127.0.0.1 -P 6032`), se definieron los clústeres lógicos separando el Maestro de la Réplica para prever el Failover:

```sql
-- Asignación del Maestro Original (Adalid - VM 196) al Hostgroup 1 (Escritura/Lectura)
INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (1, '192.168.100.196', 3306);

-- Asignación de la Bóveda Standby (Dylan - VM 200) al Hostgroup 2 (Solo Lectura / Reserva)
INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (2, '192.168.100.200', 3306);

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

### 2.3. Mapeo de Credenciales de Aplicación
Se inyectó en el proxy el usuario que Node.js utiliza (`fleet_track`), obligando a que todo su tráfico sea redirigido por defecto al Hostgroup 1 (El Maestro activo):

```sql
INSERT INTO mysql_users(username, password, default_hostgroup) VALUES ('fleet_track', 'TuPasswordDeApp', 1);
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

## 3. Automatización de la Resiliencia: El Orquestador SRE (Watchdog)
ProxySQL enruta, pero no altera el estado de las bases de datos. Para mitigar el problema del Cerebro Dividido (Split-Brain) y garantizar un consenso por quórum, se desarrolló un script autómata en la VM 197 que vigila constantemente a la VM 196 y tiene autoridad para coronar a la VM 200.

### 3.1. Desarrollo del Script de Decapitación Matemática
Se generó el archivo de ejecución en el directorio local del administrador:

```bash
nano /home/adming13/watchdog_ha.sh
```

Se inyectó la siguiente lógica de monitoreo con tolerancia a falsos positivos (3 intentos fallidos antes del Failover):

```bash
#!/bin/bash
MASTER_IP="192.168.100.196"
REPLICA_IP="192.168.100.200"
USER="ha_admin"
PASS="Orquestador_2026"
MAX_FAILS=3
FAIL_COUNT=0

while true; do
    if mysqladmin ping -h "$MASTER_IP" -u "$USER" -p"$PASS" --connect-timeout=2 &> /dev/null; then
        FAIL_COUNT=0
    else
        let FAIL_COUNT++
        if [ "$FAIL_COUNT" -eq "$MAX_FAILS" ]; then
            # Quórum perdido. Ejecutando inyección de autoridad.
            mysql -h "$REPLICA_IP" -u "$USER" -p"$PASS" -e "STOP SLAVE; SET GLOBAL read_only = 0;"
            exit 0
        fi
    fi
    sleep 2
done
```

### 3.2. Asignación de Privilegios de Ejecución
Para que el sistema operativo permita operar este autómata, se blindaron sus permisos de ejecución:

```bash
chmod +x /home/adming13/watchdog_ha.sh
```

Conclusión de la Capa: Con ProxySQL abstrayendo las IPs físicas y el Watchdog auditando la red cada 2 segundos, el clúster alcanza el estándar de Alta Disponibilidad Híbrida, permitiendo que la capa web sobreviva a la destrucción total del motor principal sin arrojar errores HTTP 500 al cliente final.


# 📑 Documentación Técnica: VM198 - Servidor de Caché en Memoria (Redis)

> **Dirección IP:** `192.168.100.198`
> **Sistema Operativo:** Ubuntu 24.04 LTS Server
> **Servicio Principal:** Redis Server (In-Memory Data Structure Store)

---

## 📌 1. Rol en la Arquitectura FleetTrack

La **VM 198** actúa como el motor de almacenamiento en caché en memoria del clúster. A diferencia de las arquitecturas monolíticas, en FleetTrack se decidió **desacoplar la caché de los nodos de aplicación (VM 197 y VM 199)**. 

**¿Por qué se tomó esta decisión?**
1. **Prevención de inanición de recursos:** Node.js (V8 engine) y Redis son aplicaciones de un solo hilo (single-threaded) que compiten agresivamente por la CPU y la RAM. Al aislarlos, garantizamos que el backend no colapse bajo altas cargas.
2. **Estado Compartido (Stateful):** Al tener un solo servidor de caché centralizado, sin importar a qué nodo (197 o 199) redirija el HAProxy a un usuario, la sesión web y los datos cacheados siempre serán consistentes.

---

## ⚙️ 2. Instalación y Hardening de Redis
La instalación se realizó desde los repositorios oficiales para garantizar estabilidad, seguida de una configuración estricta para evitar accesos no autorizados.

2.1. Instalación del Servicio
```Bash
sudo apt update && sudo apt upgrade -y
sudo apt install redis-server -y
```
2.2. Configuración Core (redis.conf)
Se editó el archivo principal /etc/redis/redis.conf aplicando directivas críticas de seguridad y rendimiento.

a) Aislamiento de Red (Bind y Port):
Por defecto, Redis escucha en 127.0.0.1. Se modificó para que escuche también en la IP de la VLAN interna, rechazando tráfico externo.

```bash
# Línea 75: Se añade la IP de la VM198
bind 127.0.0.1 192.168.100.198

# Línea 94: Puerto por defecto
port 6379

# Línea 111: Modo protegido activado para evitar intrusiones
protected-mode yes
```

b) Seguridad (Autenticación):
Se configuró una contraseña robusta para evitar accesos en texto plano desde los nodos de Node.js.

```bash
# Línea 790: Hardening de acceso
requirepass "FleetTrack_Cache_2026_Secure!"
```

c) Políticas de Evicción y Memoria (Tuning):
Como la VM tiene recursos limitados, se configuró a Redis estrictamente como caché (no como base de datos persistente) estableciendo un límite de RAM y una política de descarte allkeys-lru (Least Recently Used).

```bash
# Línea 977: Límite máximo de RAM para Redis
maxmemory 512mb

# Línea 980: Cuando se llene la RAM, borrar las llaves menos usadas recientemente
maxmemory-policy allkeys-lru
```

🛡️ 3. Optimizaciones a Nivel de Kernel (Sysctl)
Redis advierte frecuentemente sobre problemas de rendimiento nativos en Linux. Se aplicaron dos parches a nivel de kernel editando /etc/sysctl.conf para evitar cuellos de botella durante operaciones de lectura intensiva.

```bash
# 1. Solucionar advertencia de 'overcommit_memory'
echo "vm.overcommit_memory = 1" | sudo tee -a /etc/sysctl.conf

# 2. Aumentar el límite de conexiones concurrentes
echo "net.core.somaxconn = 65535" | sudo tee -a /etc/sysctl.conf

# Aplicar los cambios al kernel en vivo
sudo sysctl -p
```
3.1. Deshabilitar Transparent Huge Pages (THP)
Linux usa THP para asignar memoria, lo cual causa picos de latencia en Redis. Se deshabilitó mediante una regla en systemd:

```bash
echo 'never' | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

🔥 4. Seguridad Perimetral Local (UFW)
Se implementó el cortafuegos ufw (Uncomplicated Firewall) bajo la política de "Denegar por defecto", abriendo el puerto de Redis (6379) únicamente a las IPs de la capa de aplicación (VM 197 y VM 199). Esto impide que cualquier otra máquina (incluso la base de datos) interactúe con la memoria caché.

```bash
# Habilitar el firewall
sudo ufw enable

# Permitir tráfico SSH para administración
sudo ufw allow 22/tcp

# REGLAS ESTRICTAS PARA REDIS: Solo permitir a los Nodos de Aplicación
sudo ufw allow from 192.168.100.197 to any port 6379 proto tcp
sudo ufw allow from 192.168.100.199 to any port 6379 proto tcp

# Recargar políticas
sudo ufw reload

# Validar el estado
sudo ufw status numbered
```

🧪 5. Pruebas y Validación del Servicio
5.1. Reinicio y Persistencia del Servicio
Se configuró Redis para que arranque automáticamente en caso de que la VM 198 sufra un reinicio forzado.

```bash
sudo systemctl restart redis-server
sudo systemctl enable redis-server
sudo systemctl status redis-server --no-pager
```

5.2. Auditoría de Conectividad (Local y Remota)
Para garantizar que la configuración fue exitosa, se ejecutaron pruebas utilizando redis-cli:

Prueba interna (Desde la misma VM198):

```Bash
redis-cli -a "FleetTrack_Cache_2026_Secure!" ping
# Resultado Esperado: PONG
```

Prueba de Métricas de Uso:

```bash
redis-cli -a "FleetTrack_Cache_2026_Secure!" info memory
# Este comando comprueba que la directiva maxmemory de 512M esté aplicada.
```

5.3. Confirmación de Integración con Node.js
Dentro de la aplicación en las VMs 197 y 199, la cadena de conexión fue configurada de la siguiente manera para integrarse exitosamente con este servidor:

```bash
const redis = require('redis');
const client = redis.createClient({
    host: '192.168.100.198',
    port: 6379,
    password: 'FleetTrack_Cache_2026_Secure!'
});
```
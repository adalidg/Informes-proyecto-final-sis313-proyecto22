### FleetTrack HA
## Capa Perimetral y Enrutamiento
# Arquitectura
```
Usuarios
    │
    ▼
VM 195 (HAProxy)
    │
 ┌──┴──┐
 ▼     ▼
VM197 VM199
(Node)(Node)
```
# Acceso al Servidor Perimetral
``` bash
ssh -J usrproxy@201.131.45.42 adming13@192.168.100.195
```
Significa Jump Host se utiliza cuando la máquina destino está dentro de una red privada la conexión sigue este recorrido:
PC Personal
     │
     ▼
Proxy SSH Universidad
201.131.45.42
     │
     ▼
VM 195
192.168.100.195

# instalacion del HAProxy
``` bash 
sudo apt update
sudo apt upgrade -y
```
Descarga la lista más reciente de paquetes disponibles no instala nada solo sincroniza información
# Instalación de HAProxy
``` bash
sudo apt install haproxy -y
haproxy -v
```
Se instaló HAProxy, el software encargado de realizar el balanceo de carga entre los servidores backend. Aqui realizo la descarga la informacion mas reciente y actualizacion de paquetes instalados 
HAProxy version X.X.X
# verificacion de instalacion
``` bash 
sudo systemctl status haproxy
```
Esto confirma que el servicio HAProxy está funcionando correctamente 
El mensaje q aparece es active (running).
# Respaldo de Configuración
``` bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```
cp = Copy = Duplica archivos.
bak = archivo de respaldo.
Permite restaurar rápidamente en caso de errores.

# Configuración del Balanceador, Configurar Backend
``` bash
sudo nano /etc/haproxy/haproxy.cfg
frontend fleettrack_frontend
    bind *:80
    default_backend fleettrack_backendbackend fleettrack_backend

    balance roundrobin

    option httpchk GET /

    server node197 192.168.100.197:3000 check

    server node199 192.168.100.199:3000 check
    listen stats

    bind *:8404

    mode http

    stats enable

    stats uri /stats

    stats refresh 5s
```
Esto hace que HAProxy escuche peticiones HTTP en el puerto 80 eh indica que todas las solicitudes recibidas serán enviadas al grupo de servidores backend distribuye las solicitudes alternadamente entre los servidores 
ejemplo 
Petición 1 → 197
Petición 2 → 199
Petición 3 → 197
Petición 4 → 199
Distribución uniforme.
Fácil implementación.
Bajo costo computacional.
Realiza verificaciones periódicas para confirmar que los servidores backend están disponibles.
En la ultima parte de la configuracion se habilita un panel web para monitorear.
# Health Checks
``` bash
curl http://192.168.100.197:3000/health

curl http://192.168.100.199:3000/health
```
Aplicación Node.js
Redis
Conectividad interna
Estado general
{
  "status":"healthy",
  "redis":"ok"
}
| Campo    | Significado          |
| -------- | -------------------- |
| healthy  | Aplicación operativa |
| redis ok | Redis conectado      |


# Validar sintaxis aplicar cambios comprobar puertos abiertos
``` bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
``` 
-c = Check Mode.
No ejecuta HAProxy.
Solo analiza la sintaxis.
-f = Indica el archivo a validar. 
el resultado es : Configuration file is valid
# Reinicio del Servicio
``` bash
sudo systemctl restart haproxy
```
Aplicar cambios sin reiniciar el servidor completo.
listen stats

 bind *:8404

 stats enable

 stats uri /stats

 stats refresh 10s
Función
Permite monitorear:
Sesiones.
Conexiones.
Tráfico.
Estado de servidores.
y el acceso a http://192.168.100.195:8404/stats
luego verificacion de nodos node197 UP L7OK/200
node199 UP L7OK/200
# config errores 
``` bash
sudo ss -tulpn
```
Verifica que la configuración no contiene errores, recarga la configuración recién aplicada.
# Verificación de Health Checks Generar tráfico
``` bash 
for i in {1..20}; do
curl http://192.168.100.195
echo
done
curl http://192.168.100.195
```
Este es el mensaje que me aparece 
node197 UP
L7OK/200
node199 UP
L7OK/200
esto indica que los servidores responden correctamente y que el backend puede recibir trafico.
Demuestra que HAProxy distribuyó tráfico hacia ambos servidores.
node197 Sessions
node199 Sessions
# Simular caída
``` bash
sudo systemctl stop fleettrack
curl http://localhost:9100/metrics
node197 DOWN
node199 UP
``` 
HAProxy detecta automáticamente la falla 
La aplicación continúa funcionando y todo el tráfico es redirigido al servidor disponible.
# Restauración del Nodo
``` bash 
sudo systemctl start fleettrack
```
HAProxy reincorpora automáticamente el servidor al backend pool.
# Revisar logs
``` bash 
sudo journalctl -u haproxy -f
sudo journalctl -xe
```
Permite visualizar conexiones y eventos en tiempo real.
y este otro permite identificar fallas del sistema.
# 17: Instalar Node Exporter y verificar estado
``` bash 
sudo apt update
sudo systemctl status prometheus-node-exporter
curl http://localhost:9100/metrics
```
# Instalación de Exportador Prometheus
``` bash
sudo apt install -y prometheus-node-exporter
```
Expone metricas para CPU, RAM, Disco, Red y Procesos.
# Exportador
``` bash
sudo systemctl enable --now prometheus-node-exporter
```
enable
Inicia automáticamente al arrancar.
now
Lo inicia inmediatamente.
# Prueba de Carga Inicial
``` bash 
autocannon
autocannon -c 100 -d 30 http://192.168.100.195
```
100 usuarios concurrentes. 30 segundos continuos.
# Prueba de Carga Real
``` bash
autocannon -c 100 -d 30 http://192.168.100.195/api/flotas
```
Node.js
 ↓
ProxySQL
 ↓
Base de Datos
 ↓
Respuesta SQL

# Verificación de disponibilidad de los nodos
``` bash 
curl http://192.168.100.197:3000/health
curl http://192.168.100.199:3000/health
```
{"status":"healthy","redis":"ok"}, Esto confirmó que ambos servidores backend podían responder correctamente a las verificaciones de salud realizadas por HAProxy.
## Verificación de Estado del Clúster
``` bash 
curl http://localhost:8404/stats
```
Backend: fleettrack_backend
node197   UP   L7OK/200
node199   UP   L7OK/200
UP = nodo disponible.
L7OK/200 = respuesta HTTP 200 correcta.
Ambos servidores fueron incorporados al pool de balanceo.
## Validación de la Ruta de Aplicación
``` bash 
curl -i http://192.168.100.195/api/flotas
```
HTTP/1.1 200 OK
{
  "mensaje":"Consulta exitosa via ProxySQL",
  "datos":[
    {
      "hora_servidor":"2026-06-17T21:31:50.000Z"
    }
  ],
  "nodo_procesando":"192.168.100.199"
}
La respues ahora es la sigyiente
Cliente
HAProxy (VM 195)
Node.js (VM 197 / 199)
ProxySQL (6033)
Base de Datos
Respuesta HTTP
## Prueba de Carga del Balanceador
``` bash 
autocannon -c 100 -d 30 http://192.168.100.195/api/flotas
```
| Métrica        | Valor   |
| -------------- | ------- |
| Promedio       | 23.8 ms |
| Percentil 97.5 | 61 ms   |
| Percentil 99   | 74 ms   |
| Máxima         | 463 ms  |
124k requests en 30 segundos es el resultado de todo.
## Monitoreo de Recursos del Balanceador
``` bash 
top
```
resultado que me mostro es 100% idle 1967 MB totales
1458 MB disponibles y 0 MB utilizados
## Validación de Logs
``` bash 
sudo journalctl -u haproxy -f
```
Me salio esto GET /api/flotas HTTP/1.1
200 OK
fleettrack_backend/node197 lo cual esto confirmó que el balanceador estaba enviando tráfico hacia los servidores backend y obteniendo respuestas exitosas.
# Conclusión de la Implementación de la Capa Perimetral
```
Durante el desarrollo del proyecto FleetTrack HA se implementó exitosamente la Capa Perimetral Web sobre la máquina virtual 192.168.100.195 utilizando HAProxy como balanceador de carga principal. Esta capa fue diseñada para actuar como punto único de entrada para los usuarios, garantizando distribución de tráfico, monitoreo continuo y alta disponibilidad del servicio.

La solución implementada permitió desacoplar el acceso de los usuarios respecto a los servidores de aplicación internos, centralizando toda la comunicación HTTP en un único componente especializado. Gracias a esta arquitectura, los nodos backend pueden ser administrados, monitoreados o reemplazados sin afectar la experiencia de los usuarios finales.

Las pruebas realizadas demostraron que el balanceador fue capaz de distribuir solicitudes hacia los nodos backend, verificar automáticamente el estado de salud de cada servidor mediante Health Checks HTTP y exponer métricas operativas para el sistema de monitoreo Prometheus/Grafana.

Asimismo, las pruebas de carga realizadas con Autocannon evidenciaron que la infraestructura puede soportar miles de solicitudes por segundo manteniendo tiempos de respuesta bajos y una utilización mínima de recursos en el servidor perimetral.

La implementación cumplió completamente con los objetivos definidos para la Fase 2 del proyecto, proporcionando una arquitectura preparada para escenarios de alta disponibilidad, monitoreo centralizado y crecimiento futuro.
```
# Conclusión de la Instalación de HAProxy
```
La instalación de HAProxy permitió incorporar una solución profesional de balanceo de carga ampliamente utilizada en entornos empresariales y proveedores cloud. Este componente se convirtió en el núcleo de la capa perimetral, permitiendo administrar eficientemente el tráfico web y optimizar el uso de los recursos disponibles en los servidores backend.

La correcta instalación y validación del servicio garantizaron una base sólida para las etapas posteriores de configuración y pruebas.
```
# Conclusión de la Configuración del Frontend
```
La configuración del Frontend permitió establecer el puerto 80 como punto de acceso oficial para todos los usuarios del sistema FleetTrack HA.

Gracias a esta configuración, los usuarios ya no necesitan conocer la ubicación real de los servidores de aplicación, ya que todas las solicitudes son recibidas inicialmente por HAProxy y posteriormente encaminadas al backend correspondiente.

Esta estrategia mejora significativamente la seguridad, el control y la escalabilidad de la plataforma. 
```
# Conclusión del Backend y Balanceo Round Robin
```
La implementación del algoritmo Round Robin permitió distribuir equitativamente las solicitudes entre los nodos backend disponibles.

Este mecanismo asegura que ningún servidor reciba una carga excesiva mientras otros permanecen sin utilizar, optimizando el rendimiento general del sistema.

Además, el balanceador quedó preparado para incorporar nuevos servidores en el futuro sin modificar la experiencia de los usuarios, permitiendo escalar horizontalmente la infraestructura.
```
# Conclusión de los Health Checks
```
La incorporación de verificaciones activas mediante el endpoint /health permitió que HAProxy supervise constantemente el estado real de los servidores backend.

A diferencia de una simple comprobación de conectividad de red, el endpoint /health valida que la aplicación Node.js y los servicios asociados se encuentren funcionando correctamente.

Las pruebas realizadas confirmaron respuestas HTTP 200 en ambos nodos:
Lo cual sale node197 UP L7OK/200
node199 UP L7OK/200
Esto garantiza que únicamente los servidores operativos reciban tráfico de los usuarios.
```
# Conclusión de la Alta Disponibilidad
```
La arquitectura implementada proporciona mecanismos automáticos de recuperación ante fallos.

Gracias a los Health Checks configurados, HAProxy puede detectar la caída de un servidor backend y retirarlo automáticamente del pool de balanceo.

De esta forma, si uno de los nodos deja de responder, el tráfico continúa siendo atendido por los servidores restantes sin interrupciones visibles para los usuarios.

Este comportamiento constituye uno de los principios fundamentales de la Alta Disponibilidad (High Availability).
```
# Conclusión del Sistema de Monitoreo
```
La habilitación del panel de estadísticas de HAProxy y la instalación de Prometheus Node Exporter permitieron exponer métricas críticas para el monitoreo de la infraestructura.

Estas métricas incluyen:

Estado de los nodos backend.
Cantidad de conexiones activas.
Sesiones procesadas.
Uso de CPU.
Consumo de memoria.
Tráfico de red.

La integración con Prometheus y Grafana proporciona visibilidad completa sobre el comportamiento del sistema, permitiendo identificar cuellos de botella, fallos y tendencias de crecimiento.
```
# Conclusión de las Pruebas de Carga
```
Las pruebas realizadas mediante Autocannon permitieron evaluar objetivamente el rendimiento de la infraestructura bajo condiciones de alta concurrencia.

Comando utilizado:
autocannon -c 100 -d 30 http://192.168.100.195/api/flotas
Latencia Promedio: 23.8 ms
Percentil 99: 74 ms
Throughput Promedio: 4116 Req/Sec
Solicitudes Totales: 124,000
```
# Conclusión de la Integración con la Base de Datos
```
La validación del endpoint /api/flotas permitió comprobar que las solicitudes no solo alcanzan los servidores Node.js, sino que también atraviesan correctamente toda la arquitectura de persistencia:
Usuario
 ↓
HAProxy
 ↓
Node.js
 ↓
ProxySQL
 ↓
Base de Datos
 ↓
Respuesta
La obtención exitosa de datos desde la base de datos confirma que la integración entre todas las capas del sistema fue realizada correctamente.
```
# Conclusión Personal del Responsable de la Capa Perimetral
```
Como responsable de la Capa Perimetral Web, se logró implementar una solución completa basada en HAProxy capaz de recibir, distribuir y supervisar el tráfico HTTP dirigido al sistema FleetTrack HA.

Se configuró exitosamente el balanceo de carga mediante Round Robin, se habilitaron mecanismos de monitoreo y observabilidad, se integraron Health Checks avanzados y se realizaron pruebas de rendimiento que validaron la estabilidad de la infraestructura.

Los resultados obtenidos demuestran que la VM 195 cumple satisfactoriamente su función como API Gateway y Balanceador de Carga del proyecto, proporcionando una plataforma robusta, escalable y preparada para escenarios de alta disponibilidad dentro del entorno académico de la USFX.
```
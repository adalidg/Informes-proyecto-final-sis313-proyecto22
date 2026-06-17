# Arquitectura y Validacion de Alta Disponibilidad en Capa Lógica (FleetTrack)

## 1. Descripcion de la Implementacion del Sistema

Se configuro y valido de manera exhaustiva la capa logica de una arquitectura de Alta Disponibilidad (HA) utilizando el entorno de ejecucion Node.js junto con el framework Express. Esta solucion fue distribuida estrategicamente en dos nodos de procesamiento activos y simetricos (VM 197 y VM 199).

El objetivo principal de esta fase del proyecto fue asegurar la resiliencia total del sistema frente a caidas inesperadas de hardware o software. Al mantener dos servidores espejo, se permitio un balanceo de carga efectivo a traves del orquestador perimetral HAProxy, garantizando que el usuario final no experimente interrupciones en el servicio.

Las tareas principales ejecutadas y documentadas durante esta implementacion fueron las siguientes:

## 1.1 Auditoria y Correccion de Entorno de Variables

Se ajustaron de forma manual y minuciosa las variables de configuracion dentro del archivo oculto de entorno en cada servidor. Este paso fue critico para asegurar la simetria del cluster de procesamiento.

Esta correccion garantizo que cada instancia reconociera dinamicamente su propia direccion IP fisica dentro de la red. Al lograr esto, se eliminaron por completo los conflictos de identidad que surgian al momento de procesar peticiones, donde un nodo reportaba erroneamente la direccion de su contraparte.

## 1.2 Implementacion de Deep Health Checks Estructurales

Se desarrollo e integro un endpoint especifico de monitoreo profundo en la ruta /health. A diferencia de un chequeo web tradicional que solo valida si el servidor HTTP esta encendido, esta ruta ejecuta logica de validacion interna.

El codigo verifica en tiempo real la conexion viva con la capa de cache gestionada por Redis. Si el servicio dependiente falla o se desconecta, el script fuerza intencionalmente una respuesta de error HTTP 500 Internal Server Error. Esto indica de forma proactiva y automatica al balanceador de carga que el nodo especifico esta degradado, ordenando su exclusion inmediata de la tabla de enrutamiento para proteger el trafico de los usuarios.

## 1.3 Integracion de Persistencia Segura via ProxySQL
Se establecio un pool de conexiones estructurado hacia la base de datos, evitando la conexion directa a los nodos de almacenamiento. Todo el trafico de consultas se enruto a traves de ProxySQL, apuntando especificamente al puerto local 6033.

Para comprobar la eficacia de este puente, se construyo una ruta de prueba y consumo de datos. Esta ruta es capaz de ejecutar sentencias reales de extraccion de informacion. Con esto se valido con exito el ciclo de vida completo de la peticion HTTP y SQL, siguiendo la ruta: Cliente, Balanceador, Aplicacion Node.js, Enrutador ProxySQL y finalmente la Base de Datos.

## 1.4 Gestion Avanzada de Procesos y Memoria

Se estandarizo el uso del gestor de procesos PM2 como demonio de ejecucion para mantener los servicios web en funcionamiento continuo. Esto evita que la aplicacion se detenga al cerrar la sesion de la terminal.

La implementacion de PM2 aseguro la aplicacion de los cambios de codigo en caliente, permitiendo recargar el entorno de variables y limpiar la memoria cache de los servidores sin generar un tiempo de inactividad perceptible para el usuario final.

## 2. Registro de Comandos de Gestion de Archivos y Configuracion

A continuacion se detallan los comandos utilizados para la modificacion y lectura de los archivos estructurales del proyecto.

Editor de texto por consola utilizado de forma intensiva para inyectar la logica de las nuevas rutas, configurar el pool de conexiones del controlador MySQL y programar los scripts de diagnostico directamente en el codigo fuente del servidor.

```bash
nano ~/FleetTrack/index.js
```
Comando de lectura rapida en terminal utilizado para auditar las credenciales de conexion a la base de datos y verificar la correcta asignacion de la variable de entorno, sin el riesgo de modificar accidentalmente el archivo.

```bash
cat ~/FleetTrack/.env
```
Durante la edicion de este archivo mediante el comando nano, se inyectaron secuencialmente tres bloques de codigo fundamentales para habilitar la conexion a la base de datos y el monitoreo profundo.

Primer bloque inyectado en la cabecera del archivo para inicializar el pool de conexiones hacia ProxySQL utilizando las variables de entorno:


```bash
const mysql = require('mysql2');
const pool = mysql.createPool({
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME
});
```
Segundo bloque inyectado para exponer la ruta de consumo que realiza la consulta SQL real a traves del proxy local, validando el ciclo completo de la peticion:

```bash
app.get('/api/flotas', (req, res) => {
    pool.query('SELECT NOW() AS hora_servidor', (err, results) => {
        if (err) {
            console.error("Error DB:", err.message);
            return res.status(500).json({ error: "Fallo en base de datos" });
        }
        res.status(200).json({ 
            mensaje: "Consulta exitosa via ProxySQL",
            datos: results,
            nodo_procesando: process.env.NODE_IP || 'Desconocido'
        });
    });
});
```
Tercer bloque inyectado para reescribir la ruta de salud del sistema, integrando la validacion combinada tanto del nodo de Redis como del pool de conexiones MySQL:

```bash
app.get('/health', async (req, res) => {
    try {
        await redisClient.ping();

        await new Promise((resolve, reject) => {
            pool.query('SELECT 1', (err, results) => {
                if (err) return reject(err);
                resolve(results);
            });
        });

        res.status(200).json({ status: "healthy", redis: "ok", db: "ok" });

    } catch (error) {
        console.error(error.message);
        res.status(500).json({ status: "unhealthy", detalle: error.message });
    }
});
```


## 3. Comandos de Gestion de la Aplicacion en Produccion
Herramientas utilizadas para el control del ciclo de vida del servidor web.

Comando critico ejecutado de forma obligatoria despues de cada modificacion en los archivos de codigo fuente o configuracion de entorno. Su funcion es forzar al gestor de procesos a recargar la aplicacion en la memoria RAM, aplicar los cambios logicos y limpiar la cache, manteniendo la estabilidad del servicio.

```bash
pm2 restart all
```
Comando utilizado para guardar el estado actual de los procesos gestionados. Esto crea un archivo de volcado que garantiza que, en caso de un reinicio abrupto de la maquina virtual, el sistema operativo vuelva a levantar la aplicacion FleetTrack automaticamente al encender.

```bash
pm2 save
```

## 4. Diagnostico de Red y Validacion de Protocolo HTTP
Scripts de interaccion para comprobar la salud de los servicios desde la linea de comandos.

Ejecucion de peticiones HTTP locales hacia el propio servidor. Este comando se utilizo para verificar rapidamente que la aplicacion de Node.js estaba en linea, escuchando en el puerto designado, y que devolvia la estructura JSON correcta con el estado de salud esperado.


```bash
curl http://localhost:3000/health
```
Peticion HTTP avanzada que incluye parametros especificos para exponer las cabeceras de respuesta del servidor. Este paso fue de vital importancia para auditar el enrutamiento interno de Express y demostrar a los responsables de la capa perimetral que el backend devolvia un codigo HTTP 200 OK limpio y directo, descartando problemas de configuracion o errores de redireccion interna.

```bash
curl -i http://localhost:3000/health
```
Verificacion a nivel de capa de red (ICMP) para confirmar que la maquina virtual local tenia ruta y visibilidad hacia la capa del balanceador.

```bash
ping -c 4 192.168.100.195
```
Uso de Netcat para escanear puertos especificos. Permitio confirmar si el puerto 8080 del orquestador estaba abierto y aceptando conexiones TCP antes de lanzar trafico HTTP.

```bash
nc -zv 192.168.100.195 8080
```

Ejecucion con parametro "verbose" para rastrear el ciclo completo de la peticion HTTP hacia el balanceador, ayudando a identificar bloqueos de firewall o conexiones rechazadas.

```bash
cat ~/FleetTrack/index.js | grep "app.get"
```

## 5. Auditoria de Codigo Fuente e Infraestructura
Comandos de analisis profundo para aislar variables y comprobar estados de red.

Busqueda recursiva y exhaustiva utilizada a lo largo de todo el arbol de directorios del proyecto. Se empleo para localizar exactamente que archivos contenian declaraciones de consultas activas a la base de datos, lo que permitio diagnosticar inicialmente la falta de rutas conectadas al sistema ProxySQL.

```bash
grep -rn "SELECT" ~/FleetTrack/
```
Busqueda recursiva especifica para rastrear rastros de direcciones IP obsoletas que se habian quedado atascadas en la memoria de la aplicacion o en archivos residuales, causando los problemas iniciales de identidad cruzada entre las maquinas virtuales.

```bash
grep -R "192.168.100.199" .
```

Comando nativo de red del sistema operativo Linux utilizado para confirmar la direccion IP fisica real asignada a la interfaz de red de cada maquina virtual. Fue el paso inicial para asegurar la simetria con la configuracion del archivo de entorno del proyecto.

```bash
hostname -I
```

## 6. Pruebas de Carga y Monitoreo de Concurrencia
Comandos implementados para simular trafico y observar el comportamiento de la infraestructura bajo estres.

Creacion e inyeccion de permisos de ejecucion para un script Bash programado para automatizar el envio de peticiones HTTP continuas hacia el cluster. Esto evito la necesidad de realizar pruebas manuales desde el navegador.

```bash
chmod +x stress_test.sh
```
Lanzamiento local del script generador de trafico para iniciar el bucle de peticiones, generando asi la carga artificial de red requerida para auditar el funcionamiento del balanceador Round Robin.

```bash
./stress_test.sh
```
Monitoreo continuo y en tiempo real del archivo de registros del sistema. Se ejecuto en multiples terminales de forma paralela a las pruebas de estres para visualizar la entrada de las peticiones web y confirmar visualmente la alternancia asimetrica del trafico, probando la efectividad de la Alta Disponibilidad.

```bash
tail -f ~/FleetTrack/logs/access.log
```
## 7. Resultados y Conclusiones

La ejecucion de las pruebas y la validacion de los endpoints demostraron que la arquitectura en cluster es altamente efectiva. Durante las pruebas de carga y estres, el balanceador HAProxy distribuyo el trafico exitosamente entre los nodos backend, los cuales procesaron la concurrencia y las consultas via ProxySQL sin experimentar caidas ni perdida de conexiones.

En el contexto del proyecto de Gestion de Movimiento de Animales (GMA) para el reporte de modelado de procesos, esta infraestructura asegura que los flujos criticos de informacion se mantengan operativos. Garantizar la Alta Disponibilidad significa que el registro y control del ganado estara respaldado de manera ininterrumpida, siendo resiliente a fallos en cualquier nodo individual y garantizando la integridad de los datos en la base de datos centralizada.
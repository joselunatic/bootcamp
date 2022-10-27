
# Ejemplo de Dashboard para Internet of Things (IoT) - BASADO EN https://github.com/mcci-catena/docker-iot-dashboard - NO VALIDO PARA PRODUCCION - SOLO FINES EDUCATIVOS
Este repositorio contiene un ejemplo básico que recoge datos de un dispositivo desde un servidor IOT en red, los almacena en una DB y los muestra utilizando un dashboard web.
Puedes desplegarlo en un droplet Docker de [Digital Ocean](https://www.digitalocean.com/) (o en una VM Ubuntu de [DreamCompute](https://www.dreamhost.com/cloud/computing/), o en una VM "Ubuntu + Docker" de la [Microsoft Azure store](https://portal.azure.com/) ) con un esfuerzo mínimo. Deberías desplegar este servicio para que se ejecute en todo momento y capture los datos de tu dispositivo; después podrás acceder cuando quieras a esos datos usando un navegador web.
## Tabla de Contenidos

<!-- markdownlint-disable MD033 -->
<!-- markdownlint-capture -->
<!-- markdownlint-disable -->
<!-- TOC depthFrom:2 updateOnSave:true -->

- [Introduccion](#introduccion)
- [Definiciones](#definiciones)
- [Seguridad](#seguridad)
- [Premisas](#premisas)
- [Composicion y Puertos Externos](#composicion-y-puertos-externos)
- [Data Files](#data-files)
- [Reutilización y borrado de data files](#reutilizacion-y-borrado-de-data-files)
- [Ejemplos de Node-RED y Grafana](#ejemplos-de-node-red-y-grafana)
	- [Conectando a InfluxDB desde Node-RED y Grafana](#conectando-a-influxdb-desde-node-red-y-grafana)
	- [Iniciando sesión en Grafana](#iniciando-sesion-en-grafana)
	- [Ajustes de Data Source en Grafana](#ajustes-de-data-source-en-grafana)
- [Ejemplos MQTTS](#ejemplos-mqtts)
- [Instrucciones de Setup](#instrucciones-de-setup)
- [Backup y Restore de Influxdb](#backup-y-restore-de-influxdb)
- [Meta](#meta)

<!-- /TOC -->
<!-- markdownlint-restore -->
<!-- Due to a bug in Markdown TOC, the table is formatted incorrectly if tab indentation is set other than 4. Due to another bug, this comment must be *after* the TOC entry. -->

## Introduccion

Este [`SETUP.md`](./SETUP.md) explica la instalación del Application Server y su configuración. [Docker](https://docs.docker.com/) y [Docker Compose](https://docs.docker.com/compose/) se utilizan para facilitar la instalación y configuración de este setup.

Este dashboard utiliza [docker-compose](https://docs.docker.com/compose/overview/) para configurar un grupo de contenedores [docker containers](https://www.docker.com) primarios

1. Una instancia de [Node-RED](http://nodered.org/), que procesa los datos de los nodos individuales y los introduce en la DB.

2. Una instancia de [InfluxDB](https://docs.influxdata.com/influxdb/), que almacena los datos como unos measurements de series de tiempos con tags y provee respaldos para la DB.

3. Una instancia de [Grafana](http://grafana.org/), que proporciona un dashboard basado en web para los datos.

4. Una instancia de [Mqtt](https://mosquitto.org/), que proporciona un método ligero para transportar mensajes utilizando un modelo de publicación/suscripción.

5. Una instancia de [Chirpstack] (https://chirpstack.io), que proporciona soporte para la transmisión de datos y gestión de dispositivos LoRaWAN



Para hacer las cosas más específicas, muchas de las descripciones aquí asumen el uso de Digital Ocean. Sin embargo, esto ha sido probado en Ubuntu 20.04 sin problemas (más allá de la complejidad adicional de configurar `apt-get` para instalar docker y la necesidad de una instalación manual de `docker-compose`). Este setup debería funcionar en cualquier plataforma Linux o Linux-like que soporte `docker` y `docker-compose`. **Nota** No hemos probado que funcione en una Raspberry Pi.

## Definiciones

- El **sistema host** es el sistema que corre Docker y Docker-compose.
- El **contenedor** es uno de los sistemas virtuales corriendo bajo docker en el sistema host.

- Un **archivo en el host** es un archivo presente en el sistema host (tipicamente no
    visible por los contenedor(es)).

- Un **archivo en el contenedor X** es un archivo presente en un filesystem asociado
    con el contenedor X (y tipicamente no visible por el sistema host)
    

## Seguridad

ESTE DESPLIEGUE SE HA REALIZADO CON FINES PURAMENTE EDUCATIVOS. ES TOTALMENTE INSEGURO PARA UTILIZAR EN PRODUCCION


### Acceso Usuario

|**Para acceder**| **Abrir Link**| **Notas**|
|-------------|-------------------|----------|
| Node-RED    | <http://localhost:1880> |  |
| InfluxDB API queries | <http://localhost:8086/> |  |
| Grafana    | <http://localhost:3000> |  |
| Mqtt       | <http://localhost:1883>| Se necesita un cliente MQTT. Puedes probar utilizando [Mqtt web portal](http://tools.emqx.io/) |
| Chirpstack | <http://localhost:8080> |  | 
| Chronograf | <http://localhost:8888> |  |

Esto se puede visualizar en la figura inferior.

### Docker connection and User Access

![Connection Architecture using SSH](assets/Connection-architecture.png)

## Assumptions

- The host system must have docker-compose version 1.9 or later (for which <https://github.com/docker-compose> -- be aware that apt-get normally doesn't grab this; if configured at all, it frequently gets an out-of-date version).

- The environment variable `IOT_DASHBOARD_DATA`, if set, points to the common directory for the data. If not set, docker-compose will quit at start-up. (This is by design!)

  - `${IOT_DASHBOARD_DATA}node-red` will have the local Node-RED data.

  - `${IOT_DASHBOARD_DATA}influxdb`  will have the local InfluxDB data (this should be backed-up)

  - `${IOT_DASHBOARD_DATA}grafana` will have all the dashboards

  - `${IOT_DASHBOARD_DATA}docker-nginx` will have `.htpasswd` credentials folder `authdata` and Let's Encrypt certs folder `letsencrypt`

  - `${IOT_DASHBOARD_DATA}mqtt/credentials` will have the user credentials

## Composition and External Ports

Within the containers, the individual programs use their usual ports, but these are isolated from the outside world, except as specified by `docker-compose.yml` file.

In `docker-compose.yml`, the following ports on the docker host are connected to the individual programs.

- Nginx runs on 80/tcp and 443/tcp. (All connections to port 80 are redirected to 443 using SSL).

*The below ports are exposed only for the inter-container communication; These ports can't be accessed by host system.*

- Grafana runs on 3000/tcp.

- Influxdb runs on 8086/tcp.

- Node-red runs on 1880/tcp.

- Postfix runs on 25/tcp.

Remember, if the server is running on a cloud platform like Microsoft Azure or AWS, one needs to check the firewall and confirm that the ports are open to the outside world.

## Data Files

When designing this collection of services, there were two choices to store the
data files:

- we could keep them inside the docker containers, or

- we could keep them in locations on the host system.

The advantage of the former is that everything is reset when the docker images are rebuilt. The disadvantage of the former is that there is a possibility to lose all the data when it’s rebuilt. On the other hand, there's another level of indirection when keeping things on the host, as the files reside in different locations on the host and in the docker containers.

Because IoT data is generally persistent, we decided that the the extra level of indirection was required. To help find things, consult the following table. Data files are kept in the following locations by default.

| **Component** | **Data file location on host**| **Location in container**  |
|---------------|-----------|----------------------------|
| Node-RED      | `${IOT_DASHBOARD_DATA}node-red`| /data
| InfluxDB      |  `${IOT_DASHBOARD_DATA}influxdb` | /var/lib/influxdb
| Grafana       | `${IOT_DASHBOARD_DATA}grafana` | /var/lib/grafana|
| Mqtt | `${IOT_DASHBOARD_DATA}mqtt/credentials` | /etc/mosquitto/credentials
| Nginx | `${IOT_DASHBOARD_DATA}docker-nginx/authdata`| /etc/nginx/authdata
| Let's Encrypt certificates |`${IOT_DASHBOARD_DATA}docker-nginx/letsencrypt`|/etc/letsencrypt

As shown, one can easily change locations on the **host** (e.g. for testing). This can be done by setting the environment variable `IOT_DASHBOARD_DATA` to the **absolute path** (with trailing slash) to the containing directory prior to
calling `docker-compose up`. The above paths are appended to the value of `IOT_DASHBOARD_DATA`. Directories are created as needed.

Normally, this is done by an appropriate setting in the `.env` file.

Consider the following example:

```console
$ grep IOT_DASHBOARD_DATA .env
IOT_DASHBOARD_DATA=/dashboard-data/
$ docker-compose up –d
```

In this case, the data files are created in the following locations:

Table Data Location Examples

| **Component** | **Data file location**            |
|---------------|-----------------------------------|
| Node-RED      | /dashboard-data/node-red          |
| InfluxDB      | /dashboard-data/influxdb          |
| Grafana       | /dashboard-data/grafana           |
| Mqtt          | /dashboard-data/ mqtt/credentials |
| Nginx         | /dashboard-data/docker-nginx/authdata|
| Certificates  | /dashboard-data/docker-nginx/letsencrypt

## Reuse and removal of data files

Since data files on the host are not removed between runs, as long as the files are not removed between runs, the data will be preserved.

Sometimes this is inconvenient, and it is necessary to remove some or all of the data. For a variety of reasons, the data files and directories are created owned by root, so the `sudo` command must be used to remove the data files. Here's an example of how to do it:

```bash
source .env
sudo rm -rf ${IOT_DASHBOARD_DATA}node-red
sudo rm -rf ${IOT_DASHBOARD_DATA}influxdb
sudo rm -rf ${IOT_DASHBOARD_DATA}Grafana
sudo rm –rf ${IOT_DASHBOARD_DATA}mqtt/credentials
```

## Node-RED and Grafana Examples

This version requires that you set up Node-RED, the Influxdb database and the Grafana dashboards manually, but we hope to add a reasonable set of initial files in a future release.

### Connecting to InfluxDB from Node-RED and Grafana

There is one point that is somewhat confusing about the connections from Node-RED and Grafana to InfluxDB. Even though InfluxDB is running on the same host, it is logically running on its own virtual machine (created by docker). Because of this, Node-RED and Grafana cannot use **`local host`** when connecting to InfluxDB. A special name is provided by docker: `influxdb`. Note that there's no DNS suffix. If `influxdb` is not used, Node-RED and Grafana will not be able to connect.

### Logging in to Grafana

On the login screen, the initial user name is "`admin`". The initial password is given by the value of the variable `IOT_DASHBOARD_GRAFANA_ADMIN_PASSWORD` in `.env`. Note that if you change the password in `.env` after the first time you launch the grafana container, the admin password does not change. If you somehow lose the previous value of the admin password, and you don't have another admin login, it's very hard to recover; easiest is to remove `grafana.db` and start over.

### Data source settings in Grafana

- Set the URL (under HTTP Settings) to <http://influxdb:8086>.

- Select the database.

- Leave the username and password blank.

- Click "Save & Test".

## Ejemplos de MQTT(S) 

Puedes acceder a MQTTs de las siguientes formas:

Method  |  Port | Credentials
--------|------|------------
MQTT over TLS/SSL | 8883 | Username/Password desde archivo de configuración de mosquitto (password_file)
MQTT over TCP protocol (NO SEGURO) |  1883 | Username/Password desde archivo de configuración de mosquitto (password_file)

## Instrucciones de Setup

Revisa [`SETUP.md`](./SETUP.md) para instrucciones detalladas de setup




## Meta

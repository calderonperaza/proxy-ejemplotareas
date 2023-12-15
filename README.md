## _Proxy Inverso con Nginx, Cerbot y Docker Compose_

### Indicaciones
Se asume que usted ya ha levantado los servicios de Backend y de Fronted con Docker Compose, en este repositorio se explica y se contiene los archivos necesarios para levantar el Proxy y para Obtener los certificados SSL con Cerbot.
Estamos desplegando la aplicación Ejemplo tareas en un servidor Ubuntu, empleando docker compose.

Si desea ver los repositorios del backend y el frontend puede acceder a ellos:
- Backend - [EjemploTareasBackend](https://github.com/calderonperaza/ejemplotareasBackEndExpress)
- Frontend - [EjemploTareasFront](https://github.com/calderonperaza/ejemplotareasfrontend)

## Estructura de Archivos

A continuacion describimos los archivos y carpetas que se utilizan.

| Archivo/Carpeta | Descripcion |
| ------ | ------ |
| data | Carpeta que almacena toda la data y certificados de cerbot|
| docker-compose.yml | Tiene el archivo para que docker compose cree y levante los servicios |
| nginx.conf | Archivo con la configuración de Nginx que luego será cargado al contenedor |
| README | El archivo readme |


## Comandos
Para levantar el docker compose se ejecuta.

```sh
sudo docker compose up -d
sudo docker compose ps
sudo docker compose down
```

Una vez levantado nginx, puede imprimir la ubicacion del archivo de configuracion para verificar la ruta del volumen del compose. Ya con la ruta del conf puedes verificar si se cargó correctamente el conf, esto se hace porque algunas versiones de nginx tienen el archivo de configuración en diferente ruta

```sh
sudo docker exec nginx-proxy nginx -t
sudo docker exect nginx-proxy cat /etc/nginx/nginx.conf
```

## Renovar certificado
Para renovar el certificado se debe ejecutar un comando en el contenedor cerbot, y eso puede automatizarse colocandolo dentro del Cron.

Como no sabemos cuando se vencerá lo que se hace es ejecutar la renovacion todas las noches, si el certificado no se ha vencido no pasará nada, pero si se ha vencido se renovará. Y luego se reinicia el contenedor nginx para que tome el nuevo certificado.


Esto se debe cargar en el Cron del servidor para que se haga de forma automatica

## Guia a Desarrollar

# Paso 1: Levantando los servicios



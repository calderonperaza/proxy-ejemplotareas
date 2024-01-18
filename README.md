# _Proxy Inverso con Nginx, Cerbot y Docker Compose_

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
sudo docker exec proxy-nginx nginx -t
sudo docker exec proxy-nginx cat /etc/nginx/nginx.conf
```
Nota que el contenedor tiene un archivo de configuración por defecto, en dicho archivo hay un include para cargar los archivos encontrados en /etc/nginx/conf.d/ ahi se pueden cargar archivos, ahi es donde subimos nuestro archivo de configuración empleando volumenes en el docker compose.

## Renovar certificado
Para renovar el certificado se debe ejecutar un comando en el contenedor cerbot, y eso puede automatizarse colocandolo dentro del Cron.

Como no sabemos cuando se vencerá lo que se hace es ejecutar la renovacion todas las noches, si el certificado no se ha vencido no pasará nada, pero si se ha vencido se renovará. Y luego se reinicia el contenedor nginx para que tome el nuevo certificado.


Esto se debe cargar en el Cron del servidor para que se haga de forma automatica

## Guia a Desarrollar

### Paso 1: Levantando los servicios
Vamos a levantar con docker compose los servicios de front y de backend y configurar el dns

- _Configurar DNS:_ Esto es lo primero que debemos hacer porque tarda varios minutos en propagarse los cambios, entonces va entrar en su administrador de DNS y va crear 2 subdominios que resuelvan la direccion IP de su servidor virtual: `backenddocker.juanperez.me` y `frontenddocker.juanperez.me` un registro tipo A que resuelva la direccion Ip de su servidor en digital ocean o en Azure.
- _Levantar el Backend:_ Ahora va a clonar el repositorio del backend el cual contiene el archivo Dockerfile y docker-compose, entre en dicha carpeta y va a levantar los servicios ejecutando `docker compose up -d` Lo recomendable es que se tenga dockerizado y que se levante el backend a partir de una imagen en docker hub, si se usa docker hub, no es necesario clonar el repositorio en el servidor de despliegue, puesto que la apps ya esta dockerizada.
- _Levantar el Frontend:_ Ahora va a clonar frontend el cual contiene el archivo Dockerfile y docker-compose, entre en dicha carpeta y va a levantar los servicios ejecutando `docker compose up -d`
- _Probar los servicios:_ Pruebe que el backend y el frontend esten funcionando, primero el backend el cual se accede en: `http://localhost:3000`, el frontend debe mostrarse con un error, debido a que el front necesita tener el DNS del backend definido y compilado en la imagen docker, pero basta con que se muestre `http://localhost:9080`.

### Paso 2: Creando el Proxy-Inverso
El servicio de proxy inverso se define en el presente repositorio y utiliza Nginx para levantarlo, va encontrar dos archivos CONF en este paso usaremos el archivo.

- _Cree el archivo nginxbasic.conf:_ en este archivo tenemos la configuracion funcional sobre HTTP, mostraremos el saludo de nginx cuando lleguen peticiones localhost, y vamos a redireccionar en proxy inverso cuando lleguen las peticiones por nuestros DNS, debe poner en el archivo los dns definidos. Este archivo tiene tres bloques server.
- _Crear y levantar el docker-compose:_ Se va crear un archivo docker-compose.yml donde se van a definir y crear un servicio para el proxy con nginx. El extra-host se usa para que el contenedor pueda invocar la ip del Servidor Host para poder redireccionar al puerto que tiene el front y el backend. En volumenes se agrega el archivo nginxbasic.conf al nginx. Luego levantamos con docker compose up -d
- _Probar que todo funcione bien dns http:_ Pruebe que todo funcione bien y que pueda acceder a los servicios usando los dns sobre http.
```sh
http://localhost
http://backenddocker.juanperez.io
http://frontenddocker.juanperez.io
```
- _Pruebas si suceden errores:_ Podemos revisar si el archivo de configuracion se ha guardado correctamente, primero revisar si la ruta de nuestro contenedor esta correcta en el docker compose volumen, y el segundo es para ver si en verdad el contenedor tiene nuestro archivo nginxbasic.conf.

Compruebe el contenido del config por defecto de nginx, note la ultima linea el include, observe la ruta que aparece ahi, en esa ruta es donde el docker-compose en el volumen tiene que cargar el archivo de configuracion.
```sh
sudo docker exec proxy-nginx cat /etc/nginx/nginx.conf
```

Verifique que el archivo de configuracion se subio al contenedor.
```sh
sudo docker exec proxy-nginx cat /etc/nginx/conf.d/default.conf
```
Nota: El frontend va a mostrar un error porque debe escribirse la url del backend y volverse a compilar, pero lo haremos ya cuando tengamos el certificado

### Paso 3: Aplicar los certificados

Ahora vamos a configurar el certificado SSL para que se pueda acceder a los servicios por HTTPS, para ello vamos a usar Cerbot, el cual es un cliente de Let's Encrypt, que es una entidad que emite certificados SSL gratuitos.

- _Crear el archivo nginx.conf:_ Este archivo es similar al anterior(nginxbasic.comf), pero ahora se configura para que escuche en el puerto 443 y se agregan las directivas para que se pueda usar el certificado SSL, en este archivo se configura el certificado SSL y se configura el proxy inverso para que redireccione a los servicios de backend y frontend. Pero primero vamos a obtener el certificado SSL con Cerbot, luego vamos a habilitar el SSL.
- _Colocar el .well-known/acme-challenge/:_ Configurar los DNS en el puerto 80 para que atienda a la peticion de letsencrypt. eso se hace colocando la petición en el archivo de configuración de nginx, en el bloque server que tiene el puerto 80, se agrega la directiva location /.well-known/acme-challenge/ y se configura para que atienda a la petición y la redireccione al contenedor de cerbot.
- _Obtener los Certificados SSL invocando Cerbot:_ Ahora vamos a obtener los certificados SSL con Cerbot, para ello vamos a crear un servicio en el docker-compose.yml, este servicio va a tener la imagen de Cerbot, y va a tener un volumen para que se guarde la data de cerbot, y va a tener un extra-host para que pueda invocar el DNS del servidor host. Luego vamos a levantar el servicio con docker compose up -d, y vamos a ejecutar el comando para que cerbot obtenga los certificados SSL, luego vamos a detener el servicio de cerbot. Levante el docker compose con el nginx.conf teniendo la parte de SSL comentada, luego vamos a obtener los certificados. 
- _Obtener los Certificados SSL invocando Cerbot Parte 2:_ Se ejecutar un comando para obtener los certificados, se debe ejecutar un comando para cada DNS
```sh
docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ -d backenddocker.juanperez.io
docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ -d frontenddocker.juanperez.io
```
- _Habilitar SSL en el nginx.conf:_ Se descomenta la parte de SSL para así habilitar el SSL para nuestros DNS, debe reiniciar el contenedor de nginx para que tome el nuevo archivo de configuración.
- _Probar que todo funcione bien dns https:_ Pruebe que todo funcione bien y que pueda acceder a los servicios usando los dns sobre https.


### Paso 4: Configurar la renovacion del certificado

Ahora vamos a configurar la renovacion del certificado, para ello vamos a usar Cron, el cual es un servicio que permite ejecutar comandos en un horario especifico. Agruegue estas lineas al Cron Root

```sh
0 0,12 * * * docker compose run --rm certbot renew
5 0 * * * docker exec proxy-nginx nginx -s reload
```

Recuerde tambien configurar el firewall para solo permitir el trafico por los puertos 80 y 443, y el puerto 22 para ssh, y también para permitir todo el trafico hacia la interfaz de docker0.

### Referencias

https://mindsers.blog/en/post/https-using-nginx-certbot-docker/

https://www.joncom.be/projects/docker-nginx-letsencrypt-setup/

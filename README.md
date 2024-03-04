# Balanceador de carga con Docker

## Estructura del archivo docker-compose.yml

El archivo `docker-compose.yml` está compuesto por los siguientes elementos:

- `version`: Esta sección especifica la versión de la sintaxis de Docker Compose que se está utilizando. En este caso, se utiliza la versión 3.

- `services`: Aquí se definen los servicios que se ejecutarán como contenedores Docker.

  - `load_balancer`: Este servicio utiliza la imagen de Nginx como balanceador de carga. Se mapea el puerto 8080 del host al puerto 80 del contenedor para que el balanceador de carga esté accesible desde el exterior. También se monta el archivo `nginx.conf` del directorio actual en la ubicación `/etc/nginx/nginx.conf` del contenedor para configurar Nginx. Este servicio se conecta a la red `private_network` y se le asigna la dirección IP 10.0.0.2.

  - `app_server1` y `app_server2`: Estos servicios se crean a partir de las imágenes definidas en los directorios `app_server1` y `app_server2`, respectivamente. Se monta el directorio `html` del directorio actual en la ubicación `/usr/share/nginx/html` de cada contenedor para que ambos contenedores tengan el mismo código HTML. Ambos servicios se conectan a la red `private_network` y se les asignan las direcciones IP 10.0.0.3 y 10.0.0.4, respectivamente.

- `networks`: Aquí se define la red `private_network` que se utilizará para conectar los servicios. Se especifica la configuración de IPAM (Administración de Direcciones IP) para asignar una subred y una puerta de enlace a la red. En este caso, se utiliza la subred 10.0.0.0/24 con la puerta de enlace 10.0.0.1.

```yaml
version: '3'

services:
  load_balancer:
    image: nginx
    ports:
      - 8080:80
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - private_network


  app_server1:
    build:
      context: ./app_server1
    volumes:
      - app_server_html:/app/html
    networks:
      - private_network
    depends_on:
      - load_balancer

  app_server2:
    build:
      context: ./app_server2
    volumes:
      - app_server_html:/app/html
    networks:
      - private_network
    depends_on:
      - load_balancer

networks:
  private_network:
    ipam:
      config:
        - subnet: 10.0.0.0/24
          gateway: 10.0.0.1

volumes:
  app_server_html:
    external: false
```

## Estructura del archivo dockerfile

El archivo Dockerfile se utiliza para construir una imagen de Docker que contiene una aplicación basada en Node.js. A continuación, se explica cómo está estructurado el archivo y qué hace cada sección.

- `FROM node:14`: Esta línea indica que se utilizará la imagen base de Node.js versión 14 como punto de partida para construir la imagen.

- `WORKDIR /app`: Esta línea establece el directorio de trabajo dentro del contenedor donde se copiarán los archivos de la aplicación y donde se ejecutarán los comandos.

- `COPY package.json package-lock.json ./`: Estas líneas copian los archivos package.json y package-lock.json del directorio actual (fuera del contenedor) al directorio de trabajo dentro del contenedor (/app).

- `COPY server.js ./`: Esta línea copia el archivo server.js del directorio actual al directorio de trabajo dentro del contenedor (/app).

- `COPY index.html ./html/`: Esta línea copia el archivo index.html del directorio actual al subdirectorio html dentro del directorio de trabajo dentro del contenedor (/app/html).

- `RUN npm install`: Esta línea ejecuta el comando `npm install` dentro del contenedor para instalar las dependencias especificadas en el archivo package.json.

- `EXPOSE 8000`: Esta línea expone el puerto 8000 del contenedor para que la aplicación esté accesible desde el exterior.


```dockerfile
FROM node:14

WORKDIR /app

COPY package.json package-lock.json ./
COPY server.js ./
COPY index.html ./html/

RUN npm install

EXPOSE 8000

CMD ["node", "server.js"]
```

## Estructura del archivo nginx.conf

El archivo `nginx.conf` define la configuración de Nginx para actuar como un servidor proxy inverso y distribuir el tráfico entre los servidores de la aplicación.

- `events {}`: Esta sección define los eventos que se manejarán en Nginx, pero en este caso está vacía, lo que significa que se utilizarán los valores predeterminados.

- `http {}`: Esta sección contiene la configuración principal del servidor HTTP de Nginx.

  - `upstream app_servers {}`: Aquí se define un grupo de servidores de aplicación llamado `app_servers`. En este caso, se especifican dos servidores con las direcciones IP `10.0.0.3:8000` y `10.0.0.4:8000`. Estos servidores serán los destinos a los que Nginx distribuirá el tráfico entrante.

  - `server {}`: Esta sección configura un servidor HTTP dentro de Nginx.

    - `listen 80`: Esta línea especifica que el servidor estará escuchando en el puerto 80.

    - `server_name localhost`: Aquí se establece el nombre del servidor, en este caso, `localhost`.

    - `location / {}`: Esta sección define la ubicación a la que se redirigirá el tráfico entrante. En este caso, se utiliza la ubicación raíz `/`.

      - `proxy_pass http://app_servers;`: Esta línea indica que el tráfico entrante se pasará al grupo de servidores `app_servers`. Nginx distribuirá automáticamente el tráfico entre los servidores especificados en la sección `upstream`.

Esta configuración permitirá que Nginx actúe como un balanceador de carga, redirigiendo el tráfico entrante a los servidores de la aplicación especificados y distribuyendo la carga de manera equitativa entre ellos.

```nginx
events {}

http {
    upstream app_servers {
        server 10.0.0.3:8000;
        server 10.0.0.4:8000;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://app_servers;
        }

        location /index.html {
            proxy_pass http://app_servers;
        }
    }
}
```

## Estructura del archivo server.js

El archivo `server.js` contiene un código de ejemplo en Node.js que utiliza el framework Express para crear un servidor HTTP. A continuación se muestra cómo está estructurado el archivo y qué hace cada sección.

```javascript
const express = require('express');
const app = express();
const path = require('path');

app.get('/', (req, res) => {
  res.sendFile(path.join(__dirname, 'html/index.html'));
});

app.get('/saludo', (req, res) => {
  res.send('¡Hola desde el servidor 2!');
});

app.listen(8000, () => {
  console.log('El servidor 1 está funcionando en el puerto 8000.');
});

```

- `const express = require('express');`: Esta línea importa el módulo `express` para utilizar el framework Express en la aplicación.

- `const app = express();`: Aquí se crea una instancia de la aplicación Express.

- `app.get('/saludo', (req, res) => { ... });`: Esta sección define una ruta GET para la ruta raíz `'/'`. Cuando se accede a esta ruta, se ejecuta la función de controlador proporcionada. En este caso, la función recibe objetos `req` y `res` que representan la solicitud y la respuesta, respectivamente. La función envía la respuesta `'¡Hola desde el servidor 1!'` al cliente.

- `app.listen(8000, () => { ... });`: Esta línea inicia el servidor en el puerto 8000. Cuando el servidor se inicia correctamente, se ejecuta la función de devolución de llamada proporcionada, que imprime un mensaje en la consola indicando que el servidor 1 está funcionando en el puerto 8000.

Este código muestra un servidor básico que responde con un mensaje de saludo cuando se accede a la ruta raíz `'/saludo'`.

## Conclusión

Este proyecto combina 3 contenedores para hacer un balanceador de carga, si se abre la ruta '/' se visualizará el html compartido en el volumen, y si se abre /saludo, aparecerá un mensaje con el servidor del que provenga el archivo.

# Esta es una pequeña guía para implementar variables de entorno en tiempo de ejecución con create-react-app, Docker, Docker-Compose, ApacheServer

Hay muchas formas de configurar una aplicación React, ninguna mejor que otra, claro, la que a continuación propondremos resuelve nuestra problemática de que con una misma construcción (build) de la aplicación tengamos la posibilidad de realizar un despliegue (deploy) en distintos ambientes sin necesidad de hacer una construcción (build) por entorno. Para esto vamos a utilizar un enfoque que respete la [metodología de aplicación de doce factores](https://en.wikipedia.org/wiki/Twelve-Factor_App_methodology).

## 🤔 Ahora bien, ¿Qué queremos lograr?

Quremos poder ejecutar nuestra aplicación React como un contenedor Docker que se construye (build) sólo una vez. Se ejecuta entodas partes al ser configurable **durante el tiempo de ejecución.** La salida debe ser un contenedor liviano y de alto rendimiento que exponga nuestra aplicaicón React como contenido estático, esto último gracias a la utilización de Apache. Nuestra aplicación debe permitir la configuración dentro de un archivo de composición (docker-compose.yml) parecido a esto:

```
version: "3.2"
services:
  react-app:
    image: my-react-app
    ports:
      - "3000:80"
    environment:
      - "API_URL=production.example.com"
```

> A primera vista este enfoque puede parecer un beneficio demasiado pequeño para el trabajo adicional que requierie la configuración inicial. Pero una vez que se realiza la configuración, las configuraciones específicas del entorno y la implementación, serán mucho más fáciles de manejar. Por lo tanto, para cualquier persona que apunte a entornos dinámicos o utilice sistemas de orquestación, este enfoque es definitivamente algo a considerar.

## 🧐 El problema

En primer lugar, debe quedar claro que no existen variables de entorno dentro del entorno del navegador. Cualquiera que sea la solución que usemos hoy en día no es más que una abstracción falsa.

Pero, entonces podría preguntar, ¿qué pasa con los archivos .env y las variables de entorno REACT_APP prefijadas que vienen [directamente de la documentación](https://facebook.github.io/create-react-app/docs/adding-custom-environment-variables)? Incluso dentro del código fuente, estos se usan de la misma manera que usamos las variables de entorno dentro de Node.js, `process.env`.

En realidad, el objeto process no existe dentro del entorno del navegador, es específico del nodo. CRA (create-react-app) por defecto no hace la representación del lado del servidor. No puede inyectar variables de entorno durante el servicio de contenido (como lo hace [Next.js](https://github.com/zeit/next.js)). **Durante la transposición**, el proceso del paquete web reemplaza todas las apariciones `process.env` con un valor de cadena que se proporcionó. Esto significa **que solo se puede configurar durante el tiempo de compilación.**

## 👌 Nuestra solución

El momento específico en el que todavía es posible inyectar variables de entorno ocurre cuando iniciamos nuestro contenedor. Luego podemos leer las variables de entorno desde el interior del contenedor. Podemos escribirlos en un archivo que puede servirse a través de Apache (que también sirve a nuestra aplicación React). Se importan en nuestra aplicación usando una etiqueta `<script>` dentro de la sección de encabezado de `index.html`. Entonces, en ese momento, ejecutamos un script bash que crea un archivo JavaScript con variables de entorno asignadas como propiedades del objeto global `window`. Inyectado para estar disponible globalmente dentro de nuestra aplicación a la manera del navegador.

## 🐢 Guía paso a paso

Comencemos con crear un proyecto simple `create-react-app` y un archivo .env con nuestra variable de entorno que queremos exponer.

```sh
# Generar aplicación React
create-react-app my-app
cd my-app

# Crear variables de entorno por defecto que queremos usar
touch .env
echo "API_URL=https//default.dev.api.com" >> .env
```

Luego, escribamos un pequeño script de bash que lea el archivo `.env` y extraiga las variables de entorno que se escribirán en el archivo. Si establece una variable de entorno dentro del contenedor, se usará su valor; de lo contrario, volverá al valor predeterminado del archivo .env. Creará un archivo JavaScript que coloca los valores de las variables de entorno como un objeto que se asigna como una propiedad del objecto `window`.

```sh
#!/bin/bash

# Recreate config file
rm -rf ./env-config.js
touch ./env-config.js

# Writes JS code which opens object literal and assigns it to the global window object.
echo "window._env_ = {" >> ./env-config.js

# Read each line in .env file
# Each line represents key=value pairs
while read -r line || [[ -n "$line" ]];
do
  # Split env variables by character `=`
  if printf '%s\n' "$line" | grep -q -e '='; then
    varname=$(printf '%s\n' "$line" | sed -e 's/=.*//')
    varvalue=$(printf '%s\n' "$line" | sed -e 's/^[^=]*=//')
  fi

  # Read value of current variable if exists as Environment variable
  value=$(printf '%s\n' "${!varname}")
  # Otherwise use value from .env file
  [[ -z $value ]] && value=${varvalue}

  # Append configuration property to JS file
  echo "  $varname: \"$value\"," >> ./env-config.js
done < .env

echo "}" >> ./env-config.js

# Manage httpd daemon
/usr/local/apache2/bin/httpd -D FOREGROUND
```

Necesitamos agregar la siguiente línea a la etiqueta `<head>` dentro del `index.html` luego se importa el archivo creado por nuestro script bash.

```
<script src="%PUBLIC_URL%/env-config.js"></script>
```

Ahora nuestras variables de entorno serán instanciadas de la siguiente manera:

```
const apiUrl = window._env_.API_URL;
```

## 🛠 Desarrollo

Durante el desarrollo, si no queremos usar Docker, podemos ejecutar el script bash a través del `npm script` modificando `package.json`:

```
  "scripts": {
    "dev": "chmod +x ./env.sh && ./env.sh && cp env-config.js ./public/ && react-scripts start",
    "test": "react-scripts test",
    "eject": "react-scripts eject",
    "build": "react-scripts build'",
    "start": "react-scripts start'"
  },
```

Y, por último, hay que editar `.gitignore` para que excluir las configuraciones de entorno del código fuente:

```
# Temporary env files
/public/env-config.js
env-config.js
```

En cuanto al entorno de desarrollo, ¡eso es todo! Estamos a mitad de camino. No hemos hecho una gran diferencia en este punto en comparación con lo que CRA ofrecía por defecto para el entorno de desarrollo. El verdadero potencial de este enfoque brilla en la producción.

## 🌎 Producción

Cree los siguientes archivos dentro de la raíz de su proyecto

```
touch Dockerfile docker-compose.yml .htaccess
```

Abra el archivo .htaccess y agregue la siguiente configuración básica:

```
RewriteEngine On
RewriteBase /
RewriteRule ^index\.html$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.html [L]
SetEnvIf Request_URI "^/check$" dontlog
```

Ahora en su archivo Dockerfile agregue la siguiente configuración:

```
# build environment
FROM node:carbon-alpine as builder
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
# ENV PATH /usr/src/app/node_modules/.bin:$PATH
COPY package.json /usr/src/app/package.json
RUN npm install --silent
RUN npm install react-scripts@2.1.5 -g --silent
COPY . /usr/src/app
RUN npm run build

FROM httpd:2.4-alpine
RUN apk update && apk upgrade
# Copy .env file and shell script to container
WORKDIR /usr/local/apache2/htdocs/
RUN touch .env
RUN echo "REACT_APP_API_URL=" >> .env
COPY ./job-deploy.sh /usr/local/apache2/htdocs
RUN apk -q add --no-cache curl vim libcap bash
RUN chown -hR www-data:www-data /usr/local/apache2/
RUN setcap 'cap_net_bind_service=+ep' /usr/local/apache2/bin/httpd
RUN getcap /usr/local/apache2/bin/httpd
USER www-data
COPY --from=builder /usr/src/app/build /usr/local/apache2/htdocs/
# Make our shell script executable
RUN chmod +x /usr/local/apache2/htdocs/job-deploy.sh
COPY ./.htaccess /usr/local/apache2/htdocs/
RUN echo "ok" > /usr/local/apache2/htdocs/check
RUN sed -i 's/Listen 80/Listen 8080/g' /usr/local/apache2/conf/httpd.conf && \
  sed -i '/LoadModule rewrite_module/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
  sed -i 's#AllowOverride [Nn]one#AllowOverride All#' /usr/local/apache2/conf/httpd.conf
EXPOSE 8080
# ejecute job-deploy.sh
CMD [ "/bin/bash", "-c", "/usr/local/apache2/htdocs/job-deploy.sh"]
```

Ahora nuestro contenedor está listo. Podemos hacer todas las cosas estándar con él. Podemos crear un contenedor, ejecutarlo con configuraciones en línea y enviarlo a un repositorio proporcionado por servicios como [Dockerhub](https://hub.docker.com/).

```
docker build . -t myuser/my-react-app-container
docker run -p 3000:80 -e API_URL=https://staging.api.com -t myuser/my-react-app-container
docker push -t myuser/my-react-app-container
```

Por último, vamos a configurar nuestro archivo docker-compose.yml. Por lo general, tendrá diferentes docker-compose.yml en función del entorno y utilizará el parámetro `-f` para seleccionar qué archivo usar.

```
version: "3.2"
services:
  my-react-app:
    image: myuser/my-react-app-container
    ports:
      - "5000:80"
    environment:
      - "API_URL=production.example.com"
```

Ejecute el siguiente comando:

```
docker-compose up
```

Si observa su variable de entorno devería ver algo por el estilo

console.log(apiUrl)

> "production.example.com"

# Esta es una pequeña guía para implementar un control y logueo de variables de entorno en tiempo de ejecución de las APIs con NodeJs, Docker, Docker-Compose, Swarm.

Hay muchas formas de configurar una API NodeJs, ninguna mejor que otra, claro, la que a continuación propondremos resuelve nuestra problemática de que con una misma construcción (build) de la API tengamos la posibilidad de realizar un despliegue (deploy) en distintos ambientes y que cuando la misma se inicializa tengamos un registro de si las variables de entorno necesarias en la API fueron configuradas correctamente. Para esto vamos a utilizar un enfoque que respete la [metodología de aplicación de doce factores](https://en.wikipedia.org/wiki/Twelve-Factor_App_methodology).

## 🤔 Ahora bien, ¿Qué queremos lograr?

Queremos poder ejecutar nuestra API como un contenedor Docker que se construye (build) sólo una vez. Se ejecuta en todos los ambientes al ser configurable **durante el tiempo de ejecución.** La salida debe ser un contenedor liviano y de alto rendimiento que exponga nuestra API. Nuestra aplicación debe permitir la configuración dentro de un archivo de composición (docker-compose.yml) parecido a esto:

```
version: "3.2"
services:
  my_api:
    image: my_api_image
    ports:
      - "3000:80"
    environment:
      - "PORT=3000"
```

> A primera vista este enfoque puede parecer un beneficio demasiado pequeño para el trabajo adicional que requierie la configuración inicial. Pero una vez que se realiza la configuración, las configuraciones específicas del entorno y la implementación, serán mucho más fáciles de manejar. Por lo tanto, para cualquier persona que apunte a entornos dinámicos o utilice sistemas de orquestación, este enfoque es definitivamente algo a considerar.

## 🧐 El problema

A veces por falta de documentación o simplemente por descuidos humanos olvidamos agregar una variable de entorno en el archivo _my_api.yml_ sensible para el correcto funcionamiento de la API. En entornos de desarrollo (dev) o entornos de test (test) respondió como se esperaba pero al desplegarla (deploy) en producción notamos un comportamiento extraño y no tenemos un log como para enmarcar el error y dar con una posible solución, los problemas pueden tener un origen en múltiples aristas una de ellas es la incorrecta configuración de variable(s) de entorno en el archivo **_my_api.yml_**.

## 👌 Nuestra solución

El momento específico en el que todavía es posible tener un control o realizar una verificación de las variables de entorno que necesita la API y las que fueron configuradas en el archivo _my_api.yml_ ocurre cuando iniciamos nuestro contenedor. _Cabe aclarar que este es un enfoque, **seguramente haya otros y mejores**_. Para poder lograr esto tendremos que seguir algunas convenciones que aquí propondremos y aprovecharnos de algunas herramientas provistas por NodeJs. A grandes rasgos tendremos que crear un archivo **config.js** en el raíz del directorio de la API, este archivo contiene las variables de entorno necesarias para su funcionamiento, crear un nuevo arhivo de nombre **job.js** (también en el raíz del directorio de la API) este será el encargado de validar la variables de entorno tanto las necesitadas por la API como las que "vienen" configuradas en el archivo **_my_api.yml_** luego realizar el log cuando se inicializa la API, por último tendremos que realizar una pequeña configuración/modificación en nuestro archivo package.json.

## 🐢 Guía paso a paso

Lo primero que haremos es aprovechar cuando la API se inicia y anclar una tarea antes de iniciar el servidor NodeJs.

Abra el archivo packaje.json de su proyecto y agregue esta tarea antes de iniciar su API, debería quedar algo así:

```sh
{
  "name": "api-nodejs-docker",
  "version": "1.0.0",
  "description": "",
  "main": "app.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "node job.js && node app.js"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    ...dependencies
  }
}

```

Pues bien ahora en el raíz de nuestro proyecto creemos un archivo config.js con las variables de entorno que necesita nuestra API, debería lucir parecido a esto:

```
require("dotenv").config({ silent: true });

module.exports = {
  // the environment variables are underscore no camelcase.
  port: process.env.PORT || 3000,
  missingVariableOne: process.env.MISSING_VARIABLE_ONE || "Default value",
  sensibleVariable: {
    missingVariableTwo: process.env.MISSING_VARIABLE_TWO
  }
};
```

> Como puede observar y si aún no se lo imagina hay una de las **missing varibles** que no tiene un valor por defecto, esto es para abordar el caso de que una variable de entorno tenga datos sensibles y no queramos que quede expuesta en nuestro archivo config.js

También queremos que observe que las "key" de las variables son camelCase pero si son de alguna manera iguales a las variables de entorno del "value", ¿que queremos decir con esto?, que como convención no podrá tener una variable similar a esto:

```
other: process.env.MY_OTHER_VARIABLE
```

deberá configurarla de la siguiente manera:

```
myOtherVariable: process.env.MY_OTHER_VARIABLE
```

> Cuando mire el trabajo que realiza **job.js** entenderá porque

Hablando de job.js cree el arhivo en el raíz de proyecto y escriba o peque el siguiente código:

```
const log = require("centralized-logger").loggerInstance;
const config = require("./config");
const configEnvVars = [];

const getEnvFromString = string => {
  return string
    .replace(/([A-Z])/g, "_$1")
    .toUpperCase()
    .trim();
};

const getEnvVars = (config, result) => {
  var keys = Object.keys(config);
  keys.map(key => {
    const val = config[key];
    if (typeof val === "object") {
      return getEnvVars(val, configEnvVars);
    }
    configEnvVars.push(getEnvFromString(key.toString()));
  });
};

getEnvVars(config, configEnvVars);

const missingVars = configEnvVars.filter(configVal => {
  return !process.env.hasOwnProperty(configVal);
});

if (missingVars.length) {
  log.error("==== MISSING ENVIRONMENT VARIABLE(S) ====", {
    MISSING_ENV_VAR: missingVars.join(", ")
  });
} else {
  log.warn("==== ENVIRONMENT VARIABLES CORRECTLY CONFIGURED ====");
}
```

Pues bien ya con todo configurado es momento de crear nuestra imagen con Docker y crear un container con docker-compose, para ello necesitaremos crear dos archivos mas en el raíz del proyecto (Dockerfile y docker-compose.yml). Nuestra carpeta del proyecto debería quedar así:

![ProjectFolder](/NodeJs-Folder-guideline.png)

**_Dockerfile_**

```sh
# Change carbon to dubnium or other version node
FROM node:carbon-alpine

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
# A wildcard is used to ensure both package.json AND package-lock.json are copied
# where available (npm@5+)
COPY package*.json ./

RUN npm install
# If you are building your code for production
# RUN npm ci --only=production

# Bundle app source
COPY . .

CMD [ "npm", "start" ]
```

## 🛠 Creemos la imagen

Para crear la imagen, abra una terminal y ejecute el siguiente comando:

```sh
$ docker build -t my_api_image .
```

> Creamos una imagen con el tagname **_my_api_image_** y le indicamos que en donde estamos parados actualmente se encuentra un archivo Dockerfile, si se siente confundido con el comando por favor revise la documentación de [docker](https://docs.docker.com/)

Ahora abra y configure su docker-compose.yml

**_docker-compose.yml_**

```sh
version: "3.2"
services:
  my_api_container:
    image: my_api_image
    ports:
      - "8411:8080"
    environment:
      - "PORT=8411"
```

> Como puede observar "olvidamos" de agregar dos variables de entorno, esto es para forzar un error y verificar que todo esté funcionando correctamente

## 🌎 Iniciamos nuestro contenedor

Para iniciar nuestro container, abra una terminal y ejecute el siguiente comando:

```sh
$ docker-compose up
```

Recordemos que forzamos a que nos loguee un error con variables no configuradas, debería tener una respuesta de su terminal parecida a esta:

```
docker-compose up
Creating expressdocker_my_api_container_1 ...
Creating expressdocker_my_api_container_1 ... done
Attaching to expressdocker_my_api_container_1
my_api_container_1  |
my_api_container_1  | > express-docker@1.0.0 start /usr/src/app
my_api_container_1  | > node job.js && node app.js
my_api_container_1  |
my_api_container_1  | {"name":"express-docker","hostname":"7d6d1eafaa91","pid":16,"level":50,"msg":"==== MISSING ENVIRONMENT VARIABLE(S) ==== { MISSING_ENV_VAR: 'MISSING_VARIABLE_ONE, MISSING_VARIABLE_TWO' }","time":"2019-04-17T11:55:57.504Z","v":0}
my_api_container_1  | Servidor corriendo en puerto 8411
```

En este punto es decisión de usted si deja corriendo la API o configura correctamente las variables en su **_docker-compose.yml_** y reinicia el container, en nuestro ejemplo la variable **MISSING_VARIABLE_ONE** tenía un valor por defecto pero **MISSING_VARIABLE_TWO** como contiene datos sensible NO!, agregemos ambas variables **_docker-compose.yml_**, antes paremos nuestro container

```sh
# ctrl + c y ejecute el siguiente comando
$ docker-compose stop
```

Ahora sí modifiquemos nuestro **_docker-compose.yml_**

```sh
version: "3.2"
services:
  my_api_container:
    image: my_api_image
    ports:
      - "8411:8080"
    environment:
      - "PORT=8411"
      - "MISSING_VARIABLE_ONE='Other value'"
      - "MISSING_VARIABLE_TWO=my_supersecret_value"

```

```
docker-compose up
Recreating expressdocker_my_api_container_1 ...
Recreating expressdocker_my_api_container_1 ... done
Attaching to expressdocker_my_api_container_1
my_api_container_1  |
my_api_container_1  | > express-docker@1.0.0 start /usr/src/app
my_api_container_1  | > node job.js && node app.js
my_api_container_1  |
my_api_container_1  | {"name":"express-docker","hostname":"c6738cee7fcb","pid":16,"level":40,"msg":"==== ENVIRONMENT VARIABLES CORRECTLY CONFIGURED ====","time":"2019-04-17T12:04:19.524Z","v":0}
my_api_container_1  | Servidor corriendo en puerto 8411
```

Pues bien ahora tendremos una breve pista cuando iniciemos nuestro container si las configuraciones en el entorno son correctas, lo propuesto aquí no es mas que un enfoque que creemos que puede y debe mejorarse, si encuentra una mejor solución mas elegante, escalable y mantenible en el tiempo es libre de porponerla.

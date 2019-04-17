Si bien es libre de usar el editor de código de su preferencia para el desarrollo de API en nuestro equipo usamos [VSCode](https://code.visualstudio.com/) como editor de texto predilecto y con las siguientes configuraciones y extensiones.

Una vez instalado abrir las configuraciones de usuario

File > Preferences > Settings (Ctrl + ,)

Open settings.json y agregue lo siguiente a su configuracion

```
{
  "editor.formatOnSave": true,

  "editor.tabSize": 2
}
```

Por último instale la extensión [Prettier - Code formatter](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode). Una manera rápida de hacerlo, con VSCode abierto ejecute el siguiente método abreviado de teclado Ctrl + p e ingrese el siguiente comando para instalar Prettier

```
ext install prettier-vscode
```

Por comodidad mientras está desarrollando su API puede instalar el package globalmente [nodemon](https://www.npmjs.com/package/nodemon) así el servidor se refresca automáticamente ante cambios en algún archivo de su proyecto. Para instalarlo:

```sh
$ npm install -g nodemon
```

Luego desde una terminal en la carpeta de su proyecto ejecute:

```sh
$ nodemon server
```

Donde server es el arhivo que inicializa el servidor, puede ser index, app, etc...

Por último siempre que esté desarrollando es aconsejable ejecutar el servidor con el debugger de VSCode para tener un seguimiento de la ejecución del proyecto, uso de breakpoints, etc...

Aconsejamos la siguiente configuración para registrar la ejecución en la terminal integrada y utilizando nodemon para refrescar el servidor en modo debug cuando modifique archivos en su proyecto

![Debug](/Ide-guideline-1.PNG)

Y agregue la siguiente configuración en el archivo _launch.json_

```
"restart": true,
"runtimeExecutable": "nodemon",
"console": "integratedTerminal"
```

Puede leer la documentación mas en profundidad [aquí](https://code.visualstudio.com/docs/nodejs/nodejs-debugging), sino puede produndizar conocimientos de VSCode con la documentación [completa](https://code.visualstudio.com/docs)

+++
title = "Como actualizar tus funciones Lambda de Common JS a ECMAScript"
date = 2023-12-11T00:00:00-00:00
draft = true
description = "Sigue estos pasos para actualizar tus funciones Lambda en Common JS a módulos ECMAScript (ESM)"
tags = ["AWS", "Serverless"]
[[images]]
  src = "img/update-commonjs-to-esm/title.jpg"
  alt = "Caricatura de una laptop con engranes y una persona con una llave inglesa enorme"
  stretch = "stretchH"
+++

La versión 5.0.0 del paquete [@middy/core](https://www.npmjs.com/package/@middy/core) ya no soporta Common JS a favor módulos ECMAScript (ESM). ESM mejora el rendimiento por 2X como lo mencionan en [este artículo](https://middy.js.org/docs/upgrade/4-5/). Viendo esto me motivó a investigar que se tiene que hacer para que mis funciones Lambda sean ESM. Con estos cambios mis dependencias ya no se van a ir quedando en versiones anteriores. En este artículo vamos a ver los 3 pasos necesarios para ir de Common JS a ESM.

# Designar funciones Lambda como módulos ECMAScript
Hay dos maneras de hacer la designación como módulo ECMAScript.

### Actualizar el package.json
Puedes agregar el atributo **type** con el valor de **module** al package.json como lo mostramos en la foto de abajo.  

![Diff del package.json mostrando el atributo type como module](/img/update-commonjs-to-esm/01.jpg)

2. Actualizar la extensión del archivo
También puedes cambiar la extensión de los archivos a *.mjs* en lugar de *.js* o *.cjs*

# Actualiza como incluyes dependencias
En Common JS las dependencias se incluyen usando *require*, en ECMAScript con *import*. Aquí podemos ver el cambio necesario para incluir dependencias en ESM. 

![Diff mostrando la diferencia entre require e import para incluir modulos](/img/update-commonjs-to-esm/02.jpg)


# Actualizar como el *handler* y otras funciones son exportadas
La manera de exportar funciones en ESM es diferente a Common JS. Se tiene que cambiar de `exports.functionName` a `export const functionName` como se muestra abajo.  

![Diff mostrando la diferencia en como se exportan las funciones](/img/update-commonjs-to-esm/03.jpg)

# Ultimas palabras
Con estos tres cambios, has terminado. Son simples, pero pueden complicarse dependiendo de la estructura del proyecto. Si estás haciendo estos cambios a algo más complejo te recomiendo tener pruebas automatizadas (unitarias, de contrato, etc.) para verificar que no estás introduciendo algún bug.
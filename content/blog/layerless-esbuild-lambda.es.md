+++
title = "Elimina el uso de capas en Lambda utilizando ESBuild para empaquetar"
date = 2024-02-07T00:00:00-00:00
draft = true
description = "Aprende como puedes estructurar tus proyectos serverless para poder compartir codito y depenedencias entre tus funciones Lambda utilizando ESBuild en lugar the Layers."
tags = ["AWS", "Serverless", "Lambda"]
[[images]]
  src = "img/layerless-esbuild-lambda/title.png"
  alt = "Imagen de una persona quitandose chamarras con una montaña de chamarras que ya se quitó a un lado"
  stretch = "stretchH"
+++

He visto muchos artículos hablando sobre los problemas que traen los capas en Lambda, uno de esos es [You shouldn't use Lambda Layers](https://aaronstuyvenberg.com/posts/why-you-should-not-use-lambda-layers) por [AJ Stuyvenberg](https://twitter.com/astuyve). En este post, AJ explica los mitos y contras de usar capas en Lambda. Lo que no es fácil de encontrar son ejemplos sobre cómo realmente deshacerse de las capas usando un empaquetador. En este post, repasaremos una estructura y configuración que nos permite eliminar las capas utilizando ESBuild para empaquetar las dependencias y el código compartido para sus funciones.

## Antecedentes
He estado usando capas durante mucho tiempo como una forma de compartir dependencias y código entre diferentes funciones Lambda. Esto se hizo para poder mantener el código visible y mantenible desde la consola de AWS y tambien para reducir la cantidad de veces que incluimos una dependencia npm en el package.json de las funciones. Nosotros teníamos otros problemas que los mencionados por AJ en su post, notamos que una vez que el proyecto alcanzaba cierto tamaño, los despliegues eran un dolor de cabeza. Cada despliegue crea una nueva versión de la capa que luego crea una actualización para cada función Lambda que está consumiendo la capa. Y es por eso que empecé a buscar alejarme de las capas y en su lugar empaquetar con ESBuild.

[Enlace al ejemplo completo en GitHub](https://github.com/andmoredev/layerless-esbuild-lambda)

## Configuración de ESBuild para funciones Lambda en SAM
Lo primero que necesitamos hacer es agregar la configuración para informarle a SAM que queremos construir nuestra función usando ESBuild. Esto se hace usando los atributos de 'Metadata' como se ve a continuación.

```yaml
Metadata:
  BuildMethod: esbuild
  BuildProperties:
    Format: esm
    OutExtension:
      - .js=.mjs
    Target: es2020
    Minify: false
    Sourcemap: true
    EntryPoints:
      - index.mjs
    Banner:
      - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);
    External:
      - '@aws-sdk/client-secrets-manager'
```

Veamos cada parámetro a detalle:
* **BuildMethod** - Aquí es donde le decimos que use esbuild para empaquetar el código de la función Lambda.
* **Format** - Dado que estamos trabajando con Módulos de ECMAScript, queremos que nuestro formato sea `esm`.
* **OutExtension** -  El artefacto que produce `esbuild` tiene una extensión `.js`, dado que hemos especificado `esm` para el formato, necesitamos producirlo como `.mjs` que es como NodeJS sabe que está trabajando con ESM.
* **Target** - Utilizado para especificar la versión de ECMAScript. Aquí estaremos usando el valor predeterminado de `es2020`.
* **Minify** - Esto puede ayudar a reducir aún más el tamaño del paquete, pero hace que el código sea ilegible. Para mi ejemplo, lo mantendré como falso para poder seguir leyendo el código de mi función desplegada en la consola de AWS.
* **Sourcemap** - Atributo para especificar si desea incluir el source map en su función. Incluir esto permite una mejor experiencia de resolución de problemas. Esto se debe a que podrá obtener el stack trace y los números de línea del código original y no del empaquetado, la desventaja es que esto puede tener un impacto en el rendimiento de su función ya que la hace más grande.
* **EntryPoints** - especifica los puntos de entrada para la aplicación.
* **Banner** - Esto no está documentado por AWS. Esta propiedad nos permite solucionar un problema que ESBuild tiene con las dependencias que están utilizando `require`. [Enlace al problema en GitHub con la solución](https://github.com/aws/aws-sam-cli/issues/4827#issuecomment-1574080427).
* **External** -  Usado para excluir paquetes específicos. En este caso, estamos importando la acción `getSecret` del paquete `@aws-lambda-powertools/parameters/secrets`. Este paquete requiere `@aws-sdk/client-secrets-manager` como una dependencia, dado que sabemos que esto ya está incluido en el runtime de NodeJS 20.X, lo excluimos para que use el incluido en lugar de que nosotros lo agreguemos.

> Si tiene una dependencia que necesita ser excluida para todas las funciones, mi recomendación sería ponerla en las devDependencies.

Cuando ejecutamos `sam build`, ESBuild se encargará de empaquetar los módulos compartidos que se están importando en la función desde el directorio `src/shared`. También examinará cualquier otra dependencia e incluirá aquellas desde `node_modules`.

## Mejoras en la gestión de dependencias
Hay muchas formas de estructurar su proyecto, la mostrada en mi ejemplo la más simple que pude encontrar para reducir el esfuerzo de gestionar las versiones de las dependencias npm.
¿Cómo lo estoy haciendo? Tengo un solo package.json que incluye todas las dependencias, al empaquetar con ESBuild se encargará de incluir solo las dependencias necesarias e ignorar el resto. Manteniendo un solo package.json, podemos reducir la cantidad de pull requests que una herramienta como Dependabot abriría, ahorrando muchas horas de revisión y prueba.

## Configuración para Common JS
El uso de ESBuild no está restringido solo a ESM, si está utilizando Common JS la configuración es muy similar, necesitamos excluir algunos de los atributos. A continuación, se muestra un ejemplo para la misma función en Common JS.

```yaml
Metadata:
  BuildMethod: esbuild
  BuildProperties:
    Minify: false
    Target: es2020
    Sourcemap: true
    EntryPoints:
      - index.js
    External:
      - '@aws-sdk/client-secrets-manager'
```

> Si está interesado en migrar de Common JS a ESM, escribí un artículo sobre [Como actualizar tus funciones Lambda de Common JS a ECMAScript](https://www.andmore.dev/es/blog/update-commonjs-to-esm/)

## Conclusión
Aprendimos cómo puede configurar ESBuild para empaquetar el código de sus funciones en lugar de usar capas, así como mantener una sola fuente para las dependencias npm de nuestro proyecto, lo que simplifica el mantenimiento continuo de este proyecto. Espero que esto le ayude a configurar fácilmente sus proyectos.





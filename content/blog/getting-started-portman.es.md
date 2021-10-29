+++
title = "Hax mejor pruebas de tu API usando Portman"
date =  2021-10-28T19:41:28-05:00
draft = false
description = "Automatiza las pruebas de tu API utilizando tu definición de OpenAPI."
tags = ["AWS", "OpenAPI", "Test"]
[[images]]
  src = "img/getting-started-portman/portmancli.jpg"
  alt = "Portman Cli"
  stretch = "stretchH"
+++

Con el aumento de desarollo de APIs basado en especificaciones, cada vez hay mas herramientas que nos permiten utilizar una definición OpenAPI para diseñar, probar y validar tus APIs. Una herramienta que me mostraron hace poco es Portman, que son librerias para probar APIs que hacen facíl asegurar que pueden confiar en tus APIs mientras mantienes tu documentación sincronizada con tus definiciones del API.

Portman tiene un CLI que simplifica la configuración necesitada para arrancar. En este artículo vamos a configurar Portman y correr unas pruebas simples.

Voy a estar utilizando el API creado en [Como desarrollar un API Serverless sin un solo Lambda](https://www.andmore.dev/blog/es/build-serverless-api-with-no-lambda/), porque ese API fue creado solamente utilizando la definición de OpenAPI. Sigue las instrucciones en ese artículo para desplegarlo, toma nota del enlace que aparece como Output que lo vamos a utilizar mas adelante.

![Products URL Output](/img/getting-started-portman/01.jpg)

[Aqui está el enlace al repositorio de GitHub](https://github.com/andmoredev/lambdaless-api)

Si vas siguiendo este tutorial en tu máquina clona/baja [esta version del código](https://github.com/andmoredev/lambdaless-api/tree/819fb40601cca07ec3157e25c950f3845dac7a9e)

## 1. Instala Portman
Para poder usar el CLI de portman primero lo tenemos que instalar globalmente corriendo el siguiente comando:

``` npm install -g portman ```

## 2. Inicializa Portman
Ahora que esta instalado vamos a usar el CLI para hacer nuestra configuración inicial.

``` portman --init ```

Al correr esto te va a hacer varias preguntas para asegurar que todo este listo. A continuación puedes ver las selecciones que yo hice para mi configuración.

![Configuración de Portman CLI](/img/getting-started-portman/02.jpg)

## 3. Agregar Configuración Predeterminada
Al inicializar Portman decidimos utilizar nuestra propia configuración. Para comenzar vamos a simplemente copiar la configuración base [de aqui](https://github.com/apideck-libraries/portman/blob/main/portman-config.default.json) a nuestro archivo portman-config.json bajo el directorio portman.

La configuración solamented define pruebas de contrato que van a asegurar que nuestros enlaces regresen un estatus de éxito (2xx), que el contenido sea del formato especificado en el OpenAPI y que el cuerpo de la respuesta sea igual al que se definió en el OpenAPI

## 4. Correr Portman

Ahora que todo esta configurado simplemente tenemos que correr Portman con el comando especificado al final de la configuración.

``` portman --cliOptionsFile portman/portman-cli.json  ```

## 5. Resolviendo problemas encontrados en el OpenAPI

### **Error #1 - URL faltante**
Cuando el comando termine de correr vamos a poder ver nuestro primer error. Viendo el error en la terminal podemos ver que el URL que esta tratando de llamar no tiene nada en el enlace mas que la ruta especificada en el OpenAPI.

![Missing Server URL](/img/getting-started-portman/03.jpg)

Esto es porque Portman utiliza la sección de *[el servidor y URL base](https://swagger.io/docs/specification/api-host-and-base-path/)* del OpenAPI para ejecutar las solicitudes. Para arreglar esto vamos a ir a nuestro OpenAPI y agregar la sección de **servers**, para el URL vamos a utilizar el del Products API que ya habiamos anotado al pincipio.

El resultado final se debe ver así:

![OpenAPI servers](/img/getting-started-portman/04.jpg)

### **Error #2 - El esquema de la respuesta no es correcto**
Cuando volvemos a correr portman vemos los siguientes errores.

![Portman Failures](/img/getting-started-portman/05.jpg)

Lo que nos esta diciendo esto es que `GET /products` no esta regresando el formato esperado.

Si vemos nuestro OpenAPI define una respuesta que es un arreglo de productos:

``` json
[
  {
  }
]
```

Cuando lo comparamos con lo que estamos haciendo en el OpenAPI estamos esperando un objecto llamado que tiene un arreglo llamado *products* que es algo que se ve así:
``` json
{
  "products": [
  ]
}
```

Para arreglar esto vamos a quitar el atributo de products de la respuesta y solamente regresar un arreglo.
La plantilla de respuesta se ve así:

![Portman Failures](/img/getting-started-portman/06.jpg)

Para confirmar que todo funciona vamos a desplegar nuestro servicio de nuevo. Cuando este termine vamos a correr Portman al terminar de ejecutar todo deberia correr exitosamente.

![Successful Run](/img/getting-started-portman/07.jpg)

## 5. Asignar variables y sobrescribir
Para poder utilizar valores de otras solicitudes podemos utilizar la sección de *assignVariable* en la configuración de portman para especificar en que puntos de enlace queremos asignar una variable de la respuesta y que nombre le queremos dar a esta.

En nuestro caso vamos a querer usar el valor del *id* que es la respuesta de la solicitud `POST products` y lo vamos a utilizar en todas las demas solicitudes. La sección de *assignVariables* se debe ver así para poder lograr lo mencionado.

``` json
{
  "assignVariables": [
    {
      "openApiOperation": "POST::/products",
      "collectionVariables": [
        {
          "name": "newProductId",
          "responseBodyProp": "id"
        }
      ]
    }
  ]
}
  ```

Vamos a ver que significa esto.

* openApiOperation - esto le dice a Portman que solicitudes deben agregar el valor. La estructura de esto es *[Metodo]::[Ruta]*. El método y la ruta pueden ser comodines, si vemos la configuración predeterminada que habiamos copiado utiliza *"\*::/\*"*, esto esta diciendo que por todos los métodos y todas las rutas deberia correr las pruebas.
* collectionVariables - esto específica el nombre de la variable para poderla utilizar en otras solicitudes y el *responseBodyProp* define la ruta para poder tomar el valor de la respuesta.

Para utilizar esta variable en otras solicitudes vamos a modificar la sección de *overwrites*. Debería quedar asi:
``` json
{
  "overwrites": [
    {
      "openApiOperation": "*::/products/{productId}",
      "overwriteRequestPathVariables": [
        {
          "key": "productId",
          "value": "{{newProductId}}",
          "overwrite": true
        }
      ]
    }
  ]
}
```

Vamos a analizar este bloque.

* openApiOperation - aquí estamos usando un comodín para el método HTTP y específicando exactamente en que ruta queremos sobrescribir el valor.
* overwriteRequestPathVariables
  * key - donde quieres reemplazar el valor, en nuestro caso es {productId} en los parameteros de la ruta.
  * value - Cual es el valor que queremos aplicar en el reemplazo, este puede ser una variable o un valor estático. Aqui vamos a utilizar *newProductId* que creamos en la sección de *collectionVariables*
  * overwrite - valor boleano que le dice a Portman si debe reemplazar la variable de ruta de la solicitud.

Cuando corremos Portman vamos a ver un comentario nuevo en la solicitud POST diciendonos que la variable de la collección a sido asignada.

![Collection Variable Setting](/img/getting-started-portman/08.jpg)

Y si vemos todas las demás solicitudes van a tener el mismo ID del producto, esto pasó porque usamos un comodín para sobrescribir el valor.

![Collection Variable Usage](/img/getting-started-portman/09.jpg)

# Conclusión
Hemos aprendido que simple es generar pruebas de contrato utilizando tu OpenAPI y Portman. Muy rapidamente pudimos encontrar diferencias entre lo que habiamos definido y lo que realmente estaba haciendo nuestro API.
Este tipo de pruebas ayuda a que los usuarios puedan confiar en nuestros APIs, tambíen nos permite mantener nuestra documentación actualizada.
Este artículo cubre las funcionalidades básicas de Portman, pero hay mucho mas que puedes hacer para cubrir la mayoria de los escenarios que puedan presentarse.
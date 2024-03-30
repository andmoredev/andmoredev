+++
title = "Usando anclas y alias de YAML en una plantilla SAM"
date = 2024-03-26T00:00:00-00:00
draft = false
description = "Aprende cómo puedes configurar tu plantilla SAM para reutilizar piezas comunes de configuración utilizando anclas y alias sin introducir problemas."
tags = ["AWS", "Serverless", "SAM"]
[[images]]
  src = "img/sam-yaml-anchors/title.png"
  alt = "Image of a diagram in excalibur showing a Step Function calling a Lambda Function and the Lambda Function calling out to the Internet with the Lambda Function crossed out"
  stretch = "stretchH"
+++

El mes pasado escribí artículo sobre [cómo deshacerse de las capas de Lambda usando ESBuild](https://www.andmore.dev/blog/layerless-esbuild-lambda/). Lo que aprendí rápidamente es que el atributo *Metadata* debe ser copiado y pegado para CADA función Lambda en tu stack. Intenté usar la sección *Global* en la plantilla SAM y resulta que no es compatible. Empecé a pensar en cómo podría reutilizar la misma configuración en toda mi plantilla y descubrí que YAML ya tiene una funcionalidad que hace esto llamada [Anclas y Alias de YAML](https://www.educative.io/blog/advanced-yaml-syntax-cheatsheet#anchors). En este artículo explicaré qué son los alias y anclas de YAML y cómo podemos usarlos en SAM.

## ¿Qué son las anclas y alias de YAML?
Puedes pensar en las *anclas* como asignar un valor a una variable para ser utilizada en otros lugares. La forma de definir una *ancla* es agregando un *&* en una entidad con el nombre de la ancla.
```yaml
personName:
  type: object
  properties: &person-name-properties
    firstName:
      type: string
    lastName:
      type: string
```

En el ejemplo anterior hemos creado un ancla para el atributo *properties* del esquema *personName* con el nombre *person-name-properties*. Más adelante en el archivo, puedes hacer referencia al ancla con un alias utilizando el símbolo *\** y el nombre del ancla.
```yaml
person:
  type: object
  properties:
    name:
      type: object
      properties: *person-name-properties
```

Cuando se procesa el YAML, el objeto *person* se verá así.
```yaml
person:
  type: object
  properties:
    name: 
      type: object
      properties:
        firstName:
          type: string
        lastName:
          type: string
```

YAML te permite anular y/o extender propiedades para poder obtener la estructura correcta. Para actualizar el objeto *name* y agregar una propiedad *minLength* para *firstName* e incluir un nuevo atributo llamado *middleName*, haríamos algo como esto.
```yaml
person:
  type: object
  properties:
    name:
      type: object
      properties: 
        <<: *person-name-properties
        firstName:
          type: string
          minLength: 3
        middleName:
          type: string

```

Si observas el ejemplo anterior, ahora estamos utilizando el atributo *<<*, que básicamente *descompone* lo que se define en el ancla y te permite sobrescribir algo agregando el mismo atributo o agregar nuevos en línea. Si procesamos el YAML anterior, obtendríamos el siguiente resultado.
```yaml
person:
  type: object
  properties:
    name: 
      type: object
      properties:
        firstName:
          type: string
          minLength: 3
        lastName:
          type: string
        middleName:
          type: string
```
Con esta funcionalidad podemos definir nuestros atributos base de *Metadata* para ESBuild y usarlos en nuestra plantilla, ¿verdad? .... ¿verdad?  

![Meme de SAM y YAML](/img/sam-yaml-anchors/sam-yaml-meme-es.jpg)

## ¿Por qué las anclas de YAML no son tan sencillas con SAM?
SAM es estricto en lo que acepta en el *template.yaml* al realizar una implementación, aunque no verás estos errores al hacer `sam build`, así que ten cuidado y asegúrate de que tu plantilla sea realmente desplegable. Aquí tienes un ejemplo donde estoy definiendo la configuración *esbuild* en la raíz de mi plantilla y consumiéndola en una función.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Layerless ESBuild Example
Transform:
  - AWS::Serverless-2016-10-31

esbuild: &esbuild
  BuildMethod: esbuild
  BuildProperties:
    Format: esm
    Minify: false
    OutExtension:
      - .js=.mjs
    Target: es2020
    Sourcemap: false
    EntryPoints:
      - index.mjs
    Banner:
      - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);

Resources:
  EchoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/functions/echo
      Runtime: nodejs20.x
      Handler: index.handler
    Metadata: *esbuild
```

El template generado después de ejecutar `sam build` se ve así
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Layerless ESBuild Example
Transform:
- AWS::Serverless-2016-10-31
esbuild:
  BuildMethod: esbuild
  BuildProperties:
    Format: esm
    Minify: false
    OutExtension:
    - .js=.mjs
    Target: es2020
    Sourcemap: false
    EntryPoints:
    - index.mjs
    Banner:
    - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);
Resources:
  EchoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: EchoFunction
      Runtime: nodejs20.x
      Handler: index.handler
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        Banner:
        - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);
        EntryPoints:
        - index.mjs
        Format: esm
        Minify: false
        OutExtension:
        - .js=.mjs
        Sourcemap: false
        Target: es2020
      SamResourceId: EchoFunction
```
¿No parece mal, verdad? Parece que la ancla se referenció y colocó correctamente en mi función Lambda. Pero cuando ejecutamos `sam deploy`, obtendremos este error.
```bash
Status: FAILED. Reason: Invalid template property or properties [esbuild]
```
No le gusta la propiedad *esbuild* que hemos definido, ya que no forma parte del esquema de plantilla SAM.

## Solución alternativa
Para evitar obtener un error durante la implementación, podemos cambiar el nombre de la propiedad a *Metadata*. Esta es una propiedad aceptada en SAM que se construirá e implementará correctamente. A continuación, puede ver cómo he cambiado *esbuild* a *Metadata*.
```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Layerless ESBuild Example
Transform:
  - AWS::Serverless-2016-10-31

Metadata: &esbuild
  BuildMethod: esbuild
  BuildProperties:
    Format: esm
    Minify: false
    OutExtension:
      - .js=.mjs
    Target: es2020
    Sourcemap: false
    EntryPoints:
      - index.mjs
    Banner:
      - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);

Resources:
  EchoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/functions/echo
      Runtime: nodejs20.x
      Handler: index.handler
    Metadata: *esbuild
```
¡Esto es genial! Podemos tener varias funciones Lambda que comparten la misma configuración de ESBuild. Pero, ¿qué sucede cuando queremos definir múltiples anclas? Si agregamos un segundo atributo *Metadata*, tendremos un YAML inválido, ya que no acepta nombres de propiedades duplicadas en el mismo nivel. Lo curioso es que esto se compila e implementa correctamente, por lo que si no tienes un proceso de linting, esto podría funcionar para ti. Pero hay una mejor manera de definir múltiples anclas sin sacrificar nuestro proceso de linting.
En lugar de hacer que la propiedad *Metadata* sea el ancla, podemos agregar múltiples anclas debajo de la propiedad *Metadata*. En el siguiente ejemplo, agregaré un segundo ancla que solo contiene las *BuildProperties* y actualizaré la configuración base de ESBuild para usarlo.
```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Layerless ESBuild Example
Transform:
  - AWS::Serverless-2016-10-31

Metadata:
  esbuild-properties: &esbuild-properties
    Format: esm
    Minify: false
    OutExtension:
      - .js=.mjs
    Target: es2020
    Sourcemap: false
    EntryPoints:
      - index.mjs
    Banner:
      - js=import { createRequire } from 'module'; const require = createRequire(import.meta.url);

  esbuild: &esbuild
    BuildMethod: esbuild
    BuildProperties: *esbuild-properties

Resources:
  EchoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/functions/echo
      Runtime: nodejs20.x
      Handler: index.handler
    Metadata: *esbuild
```
¡Lo hemos logrado! Ahora podemos definir objetos YAML reutilizables en nuestras plantillas SAM. Si queremos extender o sobrescribir alguna propiedad, podemos usar el atributo *<<*.
```yaml
Resources:
  EchoFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/functions/echo
      Runtime: nodejs20.x
      Handler: index.handler
    Metadata:
      BuildMethod: esbuild
      BuildProperties:
        <<: *esbuild-properties
        Minify: true
        External:
          - '@aws-sdk/*'
```

## Conclusion
Al utilizar la funcionalidad estándar de YAML y reutilizar la propiedad *Metadata* de SAM, pudimos configurar con éxito fragmentos reutilizables de YAML para nuestras plantillas SAM. Esto nos permite reducir en gran medida el tamaño de la plantilla que estamos manteniendo, y también reduce el riesgo de errores al proporcionar configuraciones estándar que se utilizarán en todo el archivo.

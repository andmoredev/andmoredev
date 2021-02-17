+++
title = "Como construir un API Serverless sin un solo Lambda"
date = 2020-03-26T15:56:00-06:00
draft = false
tags = ["aws", "serverless", "lambda", "dynamodb"]
description = "Aprende lo necesario para construir un API Lambdaless usando AWS API Gateway, DynamoDB, OpenAPI y CloudFormation."
[[images]]
  src = "img/2020/03/pic01.jpg"
  alt = "Pantallas de computadora"
  stretch = "stretchH"
+++

Una manera que la gente construye APIs serverless es enrutando un la petición a un AWS Lambda. Esto luego hace otra petición a otro servicio de AWS. Muchas veces pasa desapercibido que API Gateway puede integrarse directamente con ontros servicios de AWS sin la necesidad de un Lambda.

La mayoría de las operaciones que realizamos en Lambda para realizar una petición son:

* Juntar la información enviada como entrada.
* Asignar los valores de la entrada de un servicio a otro.
* Asignar la salida del servicio a lo que el cliente esta esperando.

Veras con tiempo que la mayoria de las veces te encuentras repitiendo estos pasos una y otra vez.

Este proceso de asignar los valores de entrada/salida es posible sin utilizar Lambdas. API Gateway tiene la habilidad de hacer estas asignaciones utilizando [Apache's Velocity Templating Language (VTL)](https://velocity.apache.org/) para hacer lo que hacemos en el Lambda. Esto va a reducir el costo y la latencia quitando la invocación del lambda para cada petición.

Este artículo te va a enseñar lo que necesitas para poder hacer esto sin la necesidad de entrar a la consola de AWS. Evitando usar la consola de AWS nos aseguramos que el codigo esta listo para ser integrado a tu proceso de CI/CD.

---

# Vamos a comenzar
[Liga al ejemplo](https://github.com/anmoreno/lambdaless-api/)

El ejemplo muestra como crear un servicio de productos que va a Crear, Leer, Modificar y Borrar productos de la base de datos.

# Estructura de archivos
La solución solamente requiere dos archivos:

*template.yaml* - Define la tabla de DynamoDB, API Gateway y el rol que va a usar API Gateway para ejecutar comandos. CloudFormation utiliza esta este archivo para instalar todos nuestros recursos.

*products-openapi.yaml* - aqui es donde la magia pasa. Este archivo tiene la definición de nuestro API y los mapeos necesarios para las entradas y salidas usando VTL.

# Define recursos usando la plantilla SAM
Estoy usando el [Serverless Appliction Model(SAM)](https://aws.amazon.com/serverless/sam/). SAM es una herramienta que nos permite definir nuestra infrastructura como codigo. SAM lee del archivo *template.yaml* y lo corre por medio de CloudFormation. CloudFormation luego crea todos los recursos en la nube.
Los recursos que necesitamos para que el servicio funcione son los siguientes:

* **Tabla DynamoDB** - Define una sola tabla con una [Clave de partición](https://aws.amazon.com/blogs/database/choosing-the-right-dynamodb-partition-key/) llamada ***pk***. El valor de este atributo va a ser un identificador único global. (Para modelos de datos mas complejos recomiendo ver [este video](https://www.youtube.com/watch?v=6yqfmXiZTlM) por Rick Houlihan)

```yaml
  ProductsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: products
      KeySchema:
        - AttributeName: pk
          KeyType: HASH
      AttributeDefinitions:
        - AttributeName: pk
          AttributeType: S
      BillingMode: PAY_PER_REQUEST
```


* **API Gateway** - Carga la definición de nuestro API del archivo OAS. Esta usando un Transofrm para poder usar [funciones intrínsecas](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html) para poder usar cosas como ARNs, Parameters, etc.
```yaml
  ProductsAPI:
    Type: AWS::Serverless::Api
    Properties:
        StageName: Lambdaless
        DefinitionBody:
          'Fn::Transform':
            Name: AWS::Include
            Parameters:
              Location: 's3://[YOUR_BUCKET_NAME]/products-api-open-api-definition.yaml'
```

* **Role** - Rol que le da al API Gateway los permisos necesarios para poder realizar acciones en DynamoDB.
```yaml
  ProductsAPIRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          -
            Effect: Allow
            Principal:
              Service:
                - apigateway.amazonaws.com
            Action:
              - sts:AssumeRole
      Policies:
        -
          PolicyName: ProductsAPIPolicy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Allow
                Action:
                - "dynamodb:PutItem"
                - "dynamodb:UpdateItem"
                - "dynamodb:DeleteItem"
                - "dynamodb:GetItem"
                - "dynamodb:Scan"

                Resource: !GetAtt ProductsTable.Arn
```


# Crea tus puntos de enlace en API Gateway utilizando OpenAPI
Una manera simple de definir tus puntos de enlace es utilizando la [Especificación OpenAPI (OAS)](https://www.openapis.org/). OAS se ha convertido en estandar para definir APIs. Existen muchas herramientas que saben como leer estos archivos y generar documentación. [Postman](https://www.postman.com/) y [SwaggerHub](https://swagger.io/tools/swaggerhub/) son algunas aplicaciones que hacen esto.

CloudFormation a crear nuestro API utilizando lo que ya tenemos definido usando OpenAPI. Tienes que utilizar las [extensions personalizadas de OAS](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-specification-api-gateway-extensions.html) para poder realizar los mapeos de las entradas y salidas con VTL.

[Este ejemplo de la especificación Open API](https://github.com/anmoreno/lambdaless-api/blob/master/products-openapi.yaml) muestra la definición de 5 puntos de enlace RESTful. Estare haciendo enfasis a ciertas partes del archivo para poder entender mejor que esta pasando.
Primero tienes las rutas (paths), cada ruta va a tener cada una de las operaciones que va a correr: GET, POST, PUT, DELETE.
En el API de ejemplo denemos dos rutas:

**/products**
Esta ruta define dos puntos de enlace:
* GET - Toma la lista de todos los productos.
* POST - Agrega un producto nuevo.

**/products/{productId}**
Esta ruta toma un *productId* para trabajar en un producto específico. Los puntos de enlace para los que utilizamos esta ruta son:
* GET - Toma los detalles de un producto.
* PUT - Modifica un producto con la informacion de entrada.
* DELETE - Borra un producto.

Hay tres secciones en cada uno de los puntos de enlace:
* requestBody - esta sección solamente aplica para los puntos de enlace que aceptan valores de entrada (POST y PUT en nuestro ejemplo). Aqui se define el modelo que se espera en la entrada. Si las validaciones no pasan esto va a retornar un codigo 400 Bad Request.
* responses - lista de todas las respuestas esperadas cuando se hace una llamada a este punto de enlace.
* x-amazon-apigateway-integration - Este es una de las extensiones de AWS, define los mapeos para las entradas y salidas para las llamadas a DynamoDB. Vamos a entrar mas a detalle en la siguiente sección.

# Entendiendo mejor la extensión x-amazon-apigateway-integration
El ejemplo de abajo nos muestra como el punto de enlace que agrega productos implementa la extensión.

```yaml
x-amazon-apigateway-integration:
        httpMethod: POST
        type: AWS
        uri: { "Fn::Sub": "arn:aws:apigateway:${AWS::Region}:dynamodb:action/PutItem" }
        credentials: { "Fn::GetAtt": [ ProductsAPIRole, Arn ] }
        requestTemplates:
          application/json:
            Fn::Sub:
              - |-
                {
                  "TableName": "${tableName}",
                  "Item": {
                    "pk": {
                      "S": "$context.requestId"
                    },
                    "productname": {
                      "S": "$input.path("$.name")"
                    }
                  }
                }
              - { tableName: { Ref: ProductsTable } }
        responses:
          default:
            statusCode: 201
            responseTemplates:
              application/json: '#set($inputRoot = $input.path("$"))
                {
                    "id": "$context.requestId"
                }'
```

Hay muchas más propiedades que pueden ser asignadas por este integración de AWS, para ver una lista completa [ir a esta página](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions-integration.html). Voy a destacar algunos que estan siendo utilizados en el ejemplo:

* **uri** - punto de enlace que API Gateay va a usar para ejecutar la acción en DynamoDB. En el ejemplo esta haciendo una llamada a [DynamoDB Put Item](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_PutItem.html).

* **credentials** - Utiliza el rol definition en el archivo template.yaml. Esta utilizando la function intrinseca *[GetAtt](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-getatt.html)* para adquirir el ARN del rol.

* **[requestTemplates](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions-integration-requestTemplates.html)** - Esta sección toma el cuerpo de la petición y lo mapea a los parametros de entrada que requiere DynamoDB usando VTL. El primer elemento de la lista esta usando el *requestId* para asignar el valor del *ProductId*. Para ver una lista de todas las variables que tenemos disponibles en el *$context& [ir aqui](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#context-variable-reference). Esto también hace uso de la variable *[$input](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#input-variable-reference)* para adquirir los valores de entrada proporcionados, en el ejemplo se está usando para tomar el atributo *name*. En el segundo elemento usamos la function intrínseca *[Fn::Sub](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html)* para tomar el nombre de la tabla definida en el *template.yaml*.

* **[responses](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions-integration-responses.html)** - Contiene responseTemplates. Esto esta usando las mismas variables para construir los mapeos para la respuesta. En el ejemplo se puede ver como usar una expresión *foreach* con VTL. El mapeo esta creando una lista de productos como la respuesta del punto de enlace del API.
```yaml
responseTemplates:
              application/json: '#set($inputRoot = $input.path("$"))
                          {
                            "products": [
                              #foreach($elem in $inputRoot.Items) {
                                "id": "$elem.pk.S",
                                "name": "$elem.productname.S",
                              }#if($foreach.hasNext),#end
                              #end
                            ]
                          }'
```

Hay muchas otras cosas que se puden hacer utilizando OpenAPI y Extensiones Amazon API Gateway. Para ver más información relacionada a esto puedes ver ir a los siguientes links:
* [Documentación Open API](https://swagger.io/docs/specification/about/)
* [Extensiones Open API de Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions.html)

# Instalar el Servicio en AWS
Como les había mencionado anteriormente todo esto lo estamos haciendo sin usar la consola de AWS. Para lanzar la aplicación a AWS hay unas cosas que tenemos que realizar primero:

1\. La versión mas reciente de nuestro archivo que contiene la Specificación Open API  tiene que ser copiada a un bucket S3.
```bash
aws s3api create-bucket --bucket [NOMBRE-DE-BUCKET]
aws s3 cp ./products-openapi.yaml s3://[NOMBRE-DE-BUCKET]/
```

El primer comando va a crear el bucket en S3 donde nuestro archivo Open API va a vivir. El segundo comando copia el archivo a ese bucket.

> *[NOMBRE-DE-BUCKET] debe ser reemplazado con un nombre único
> Los nombres de Buckets son globales, si alguna otra cuetna tiene un bucket con ese mismo nombre, este comando fallaría.*

2\. SAM necesita construir los artefatos que van a ser lanzados. Esto se hace corriendo el siguiente commando:
```bash
sam build
```
Esto va a crear un folder llamado .aws-sam que contiene los artefactos.

3\. Con los artefactos construidos SAM ahora puede lanzarlos a la nube.

> *Desde la version v0.33.1 sam cli introdujo la abilidad de lanzar usando un archivo samconfig.toml. Puedes hacer que el cli genere este archivo corriendo el sam deploy en modo guiado*.

Para crear el archivo samconfig.toml necesitas correr este comando:
```bash
sam deploy --guided
```
Esto te va a hacer varias preguntas simples para poder llenar el archivo de configuración. Todo va a estar listo para ponerlo a la prueba cuando deje de correr este comando.

# Como probar el API
Para poder revisar que todo se lanzó correctamente necesitamos conseguir la liga del API. La manera más fácil es ir al servicio de API Gateway en la consola de AWS. Pero les había prometido que no ibamos a entrar a la console.
La solución a esto es usando el [aws cli](https://aws.amazon.com/cli/).
Por razones de seguridad los URL de los APIs no son accesibles haciendo una llamada a un punto de enlace, lo que significa esto es que lo vamos a tener que construir a mano.

Los metadatos de la pila (stack) de CloudFormation contiene una de las piezas que necesitamos para construir este enlace. Los metadatos pueden ser adquiridos ejecutando este comando:
```bash
aws cloudformation describe-stack-resources --stack-name products-service --logical-resource-id ProductsAPI
```
Puedes encontrar el *stack-name* en *samconfig.toml* file. El *logical-resource-id* es el nombre que le diste a tu API en el *template.yaml*.
El resultado del comando debe verse asi:
```json
{
    "StackResources": [
        {
            "StackName": "products-service",
            "StackId": "arn:aws:cloudformation:us-east-1:1234567890:stack/products-service/12345678-9a01-23bc-efgh-1j3kl984m932",
            "LogicalResourceId": "ProductsAPI",
            "PhysicalResourceId": "mx266abc0g",
            "ResourceType": "AWS::ApiGateway::RestApi",
            "Timestamp": "2020-03-24T02:31:27.542Z",
            "ResourceStatus": "UPDATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
    ]
}
```
Necesitamos tres piezas de información para construir el enlace:
* *PhysicalResourceId* encontrado en los metadatos de la pila.
* *Region* se puede encontrar en el  *archivo samconfig.toml*.
* *Stage* esta en el recurso *ProductsAPI* en el *template.yaml*.

```
https://[PhysicalResourceId].execute-api.[Region].amazonaws.com/[Stage]
```

En nuestro ejemplo se vería así:
```
https://mx266abc0g.execute-api.us-east-1.amazonaws.com/Lambdaless
```

Puedes utilizar una herramienta como Postman para hacer peticiones a tu API. Todo lo que tienes que hacer es agregar la ruta que definiste en el archivo OAS asi como proveer los parametros y cuerp necesarios en *parameters* y *body*.
Inclui una [colección de Postman](https://github.com/anmoreno/lambdaless-api/blob/master/Products.postman_collection.json) en el ejemplo para poder importarlo y ver como se ven las peticiones.

# Bonus
Para simplificar los lanzamientos incluí un archivo *package.json*. El archivo tiene un secuencia de comandos npm que se va a encargar de ejecutar todo lo necesario para un lanzamiento.
Los comandos se ven así:
```bash
"deploy": "aws s3api create-bucket - bucket [NOMBRE-DE-BUCKET] && aws s3 cp ./products-api-open-api-definition.yaml s3://[NOMBRE-DE-BUCKET]/ && sam build && sam deploy"
```
Ahora puedes usar npm para exejutar tu lanzamiento:
```
npm run deploy
```
Ahora debes entender como construir y lanzar un API REST que no require Lambdas. Y como un bonus adquieres APIs [que se van documentando solos](https://learning.postman.com/docs/postman/api-documentation/documenting-your-api/).
Espero y les haya gustado esto. [Aqui](https://github.com/anmoreno/lambdaless-api/) pueden encontrar un enlace al ejemplo completo.
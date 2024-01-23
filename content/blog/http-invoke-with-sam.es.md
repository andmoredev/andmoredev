+++
title = "Llamar a endpoints de terceros desde Step Functions con SAM"
date = 2024-01-23T00:00:00-00:00
draft = true
description = "Sigue estos 3 pasos para hacer llamadas a APIs externos usando Step Functions y SAM"
tags = ["AWS", "Serverless"]
[[images]]
  src = "img/http-invoke-with-sam/title.png"
  alt = "Imagen de un diagrama en excalibur mostrando Step Functions llamando Lambda y el Lambda llamando al Internet con el Lambda tachado"
  stretch = "stretchH"
+++

# Llamando a Endpoints Externos con Step Functions y SAM

[Benoit Boure](https://twitter.com/Benoit_Boure) escribió un artículo la semana pasada sobre [Como hacer llamadas a APIs externos con Step Functions y el CDK](https://benoitboure.com/calling-external-endpoints-with-step-functions-and-the-cdk), pero si me conoces, sabe que me encanta SAM y quería mostrar la misma configuración, pero en lugar de usar CDK, usaremos SAM.

No entraré en detalles sobre cómo funciona, ya que Benoit hizo un gran trabajo explicándolo en el artículo mencionado anteriormente.

Utilizaremos la [API de OpenWeather](https://openweathermap.org/api) para verificar la temperatura de algún lugar y ver si necesitamos usar una chamarra gruesa, ligera o no llevar chamarra. A continuación se muestra el diagrama del flujo de trabajo de el step function que ejecutaremos.
![Diagrama de la función de paso](/img/http-invoke-with-sam/stepfunction-diagram.png)

[Enlace al ejemplo completo en GitHub](https://github.com/andmoredev/http-invoke-with-sam)
# Pasos a seguir

## 1. Crear Conexión de EventBridge
El paso de HTTP utiliza una Conexión de EventBridge para autenticar la solicitud. Para la API de OpenWeather, debemos proporcionar el API Key como un query parameter y no como un header de autorización, como lo haría típicamente en otras APIs. Para lograr esto, agregamos el objeto *InvocationHttParameters* donde agregamos el query parameter a la solicitud con el nombre appid.

```yaml
OpenWeatherConnection:
  Type: AWS::Events::Connection
  Properties:
    Name: 'openweather-connection'
    AuthorizationType: API_KEY
    AuthParameters:
      ApiKeyAuthParameters:
        ApiKeyName: Authorization
        ApiKeyValue: !Ref OpenWeatherAPIKey
      InvocationHttpParameters:
        QueryStringParameters:
          - IsValueSecret: true
            Key: appid
```
## 2. Crear política de ejecución del Step Function 
Se necesitan varios permisos para ejecutar con éxito paso de HTTP en step functions.

1. Necesita acceso al secreto que la Conexión de EventBridge crea en Secrets Manager.
2. Permiso para tomar las credenciales de conexión de la conexión de EventBridge.
3. Permiso para hacer la solicitud al API con InvokeHTTPEndpoint, en este caso, el recurso debe ser el ARN de la step function que lo está ejecutando. Para evitar una dependencia cíclica en SAM, estamos creando manualmente el ARN. Estamos agregando un comodín al final para tener en cuenta el identificador que AWS agrega al final de los nombres generados para los recursos. También estamos restringiendo la consulta para que solo se permita para un GET al URL base de la API de OpenWeather.

```yaml
Policies:
  - Version: 2012-10-17
    Statement:
      - Effect: Allow
        Action:
          - secretsmanager:DescribeSecret
          - secretsmanager:GetSecretValue
        Resource: !GetAtt OpenWeatherConnection.SecretArn
      - Effect: Allow
        Action:
          - events:RetrieveConnectionCredentials
        Resource: !GetAtt OpenWeatherConnection.Arn
      - Effect: Allow
        Action:
          - states:InvokeHTTPEndpoint
        Resource: !Sub arn:aws:states:${AWS::Region}:${AWS::AccountId}:stateMachine:ShouldIWearAJacketStateMachine*
        Condition:
          StringEquals:
            states:HTTPMethod: GET
          StringLike:
            states:HTTPEndpoint: !Ref OpenWeatherBaseUrl
```
## 3. Hacer la solicitud al API en Step Functions
Haciendo esto ya podemos agregar el paso a nuestro step function como se muestra a continuación.
Especificamos el método HTTP como GET, el ARN de conexión a nuestra conexión de EventBridge como autenticación, la URL base de OpenWeather y agregamos query parameters para obtener la temperatura de la ubicación y las unidades que queremos.

```json
"Get Temperature": {
  "Type": "Task",
  "Resource": "arn:aws:states:::http:invoke",
  "Parameters": {
    "Method": "GET",
    "Authentication": {
      "ConnectionArn": "${OpenWeatherConnectionArn}"
    },
    "ApiEndpoint": "${OpenWeatherBaseUrl}",
    "QueryParameters": {
      "lat.$": "$.latitude",
      "lon.$": "$.longitude",
      "units": "imperial"
    }
  },
  "ResultSelector": {
    "temperature.$": "$.ResponseBody.main.temp"
  }
}
```

# Conclusion
Como puedes ver, no es tan complicado reemplazar cualquier función Lambda que tengas para hacer llamadas a APIs externos. Los permisos pueden volverse un poco complicados, pero una vez que los comprendes, simplemente puedes repetirlos para otras solicitudes que necesites hacer.
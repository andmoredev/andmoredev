+++
title = "Asegurar API Gateway con Amazon Cognito usando SAM"
date = 2024-05-06T00:00:00-00:00
draft = false
description = "Obtener autenticación básica para tu API no es tan difícil como parece. En este artículo, repasaremos los pasos para asegurar nuestras APIs con Amazon Cognito"
tags = ["AWS", "API", "Security", "Serverless", "SAM"]
[[images]]
  src = "img/api-cognito/title-es.png"
  alt = ""
  stretch = "stretchH"
+++

# Historia
Constantemente estoy creando APIs, ya sea para publicaciones en blogs, mientras estoy probando nuevas funcionalidades o para herramientas que he creado para mí mismo. Todas estas han sido creadas sin autenticación. No asegurar tus APIs mientras potencialmente expones tus datos también puede representar un riesgo financiero para tus cuentas si un usuario malicioso se apodera de estos puntos finales. Es por eso que he decidido asegurar mis APIs desde el principio y quiero que esto se haga con la mínima configuración necesaria para que sea fácil de replicar muchas veces.

Primero necesitamos entender algunos conceptos sobre lo que estamos configurando.

# ¿Qué es Cognito?
Amazon Cognito es un servicio proporcionado por AWS que te permite agregar autenticación a tus aplicaciones o servicios. Se integra nativamente con API Gateway para asegurar cada punto final.

Cognito tiene múltiples capas donde puedes aplicar diferentes tipos de configuraciones, lo que nos da la flexibilidad para configurar las cosas para diferentes casos de uso.

1. **User Pool** - 
Un user pool de Cognito es la columna vertebral de todo en Cognito. Este es un [ proveedor de identidades OpenID Connect](https://auth0.com/docs/authenticate/protocols/openid-connect-protocol) que contiene el directorio de usuarios para autenticar y autorizar usuarios.

2. **User Pool Domain** -
El user pool domain se utiliza para darle un mejor nombre a la URL de autenticación para que la utilicemos y dar más confianza a nuestros usuarios.


3. **Resource Server** -
[Un servidor de API OAuth 2.0 ](https://www.oauth.com/oauth2-servers/the-resource-server/) que valida que un token de acceso contenga los ámbitos que autorizan el punto final solicitado en la API.

4. **User Pool Client** -
Un user pool client es una configuración dentro de un grupo de usuarios que interactúa con tu aplicación que se autenticará usando Cognito.

# Flujos de Autenticación
Existen varios flujos de autenticación que puedes usar para tus aplicaciones. [En esta publicación de Auth0](https://auth0.com/docs/get-started/authentication-and-authorization-flow) puedes obtener una mejor comprensión sobre qué flujo es mejor para tu caso de uso. Dado que estoy configurando una autenticación muy básica para poder probar mi API con Postman, utilizaré el  [Client Credentials Flow (CCF)](https://auth0.com/docs/get-started/authentication-and-authorization-flow/client-credentials-flow) para permitirnos autenticar nuestras solicitudes enviando un id de cliente y un secreto de cliente a cambio de un token de acceso en forma de un [JSON Web Token (JWT)](https://jwt.io/introduction). El CCF se recomienda cuando se trabaja con comunicación de Máquina a Máquina (M2M) como CLIs, APIs, etc.

# Configurándolo todo con SAM
No quiero tener un Grupo de Usuarios de Cognito por cada API que cree, para simplificar esto, tendré un solo Stack de Autenticación que contendrá los recursos de Grupo de Usuarios y Dominio del Grupo de Usuarios. Luego compartiremos los datos de este stack con nuestros otros stacks para poder crear el Servidor de Recursos y los Clientes del Grupo de Usuarios en stacks separados. A continuación, se muestra una imagen que muestra cómo se ve esto.

![Arquitectura del stack de cloudformation](/img/api-cognito/stack-architecture.png)

Ahora veamos cómo se configuran cada una de estas piezas usando SAM.

## Stack de Autenticación
En este stack vamos a definir los recursos que serán consumidos por otras APIs para autenticarse.
Puedes encontrar toda mi configuración completa para este stack [en este repositorio de GitHub](https://github.com/andmoredev/cognito-auth)

### 1. User Pool
El User Pool no requiere mucha configuración al hacer CCF. Puedes agregar más restricciones y configuraciones, pero como se mencionó antes, estamos tratando de mantenerlo simple por ahora.
```yaml
  CognitoUserPool:
    Type: AWS::Cognito::UserPool
```

### 2. User Pool Domain
En mi configuración, estoy usando andmoredev como mi dominio, lo que hace que nuestra URL de inicio de sesión se vea como `https://andmoredev.auth.us-east-1.amazoncognito.com/login`. Puedes configurar tu propio dominio personalizado que no incluya nada generado por AWS en la URL configurando el *CustomDomainConfig*.
```yaml
  CognitoUserPoolDomain:
    Type: AWS::Cognito::UserPoolDomain
    Properties:
      UserPoolId: !Ref CognitoUserPool
      Domain: andmoredev
```

Necesitamos asegurarnos de que otros stacks puedan acceder al Id y ARN del Grupo de Usuarios para poder crear los recursos necesarios. Para hacer esto, vamos a usar Parámetros de SSM.
```yaml
  CognitoUserPoolIdParameter:
    Type: AWS::SSM::Parameter
    Properties:
      Name: !Sub /${AWS::StackName}/CognitoUserPoolId
      Type: String
      Value: !Ref CognitoUserPool

  CognitoUserPoolArnParameter:
    Type: AWS::SSM::Parameter
    Properties:
      Name: !Sub /${AWS::StackName}/CognitoUserPoolArn
      Type: String
      Value: !GetAtt CognitoUserPool.Arn
```

Ahora podemos agregar autenticación a un stack separado usando el mismo grupo de usuarios.

## Stack de API
Agregaré autenticación a una API existente [en este repositorio de GitHub.](https://github.com/andmoredev/layerless-esbuild-lambda)

### 1. Consumir recursos del Stack de Autenticación
Primero necesitamos obtener el Id y ARN del grupo de usuarios de SSM agregándolos a la sección Parameters de nuestra plantilla.
```yaml
Parameters:
  CognitoUserPoolId:
    Type: 'AWS::SSM::Parameter::Value<String>'
    Default: '/andmoredev-auth/CognitoUserPoolId'

  CognitoUserPoolArn:
    Type: 'AWS::SSM::Parameter::Value<String>'
    Default: '/andmoredev-auth/CognitoUserPoolArn'
```

### 2. Resource Server
Estamos creando un resource server con un scope que se utilizará para dar acceso a todos los puntos finales de nuestra API. No entraré en más detalles sobre el diseño de scopes más avanzados en esta publicación.

```yaml
  LayerlessESBuildResourceServer:
    Type: AWS::Cognito::UserPoolResourceServer
    Properties:
      UserPoolId: !Ref CognitoUserPoolId
      Identifier: layerless-esbuild
      Name: Layerless ESBuild
      Scopes:
        - ScopeName: api
          ScopeDescription: Allow api access
```

### 3. User Pool Client
A continuación se muestra la definición para nuestro user pool client.
```yaml
  CognitoTestAutomationClient:
    Type: AWS::Cognito::UserPoolClient
    DependsOn:
      - PostmanResourceServer
    Properties:
      UserPoolId: !Ref CognitoUserPoolId
      GenerateSecret: true
      AllowedOAuthFlows:
        - client_credentials
      AllowedOAuthScopes:
        - layerless-esbuild/api
      AllowedOAuthFlowsUserPoolClient: true
```
Hablemos sobre las propiedades que hemos configurado para nuestro cliente del grupo de usuarios.
* UserPoolId - referencia al Id del grupo de usuarios creado en nuestro Stack de Autenticación.
* GenerateSecret - esto es necesario para que podamos usar el flujo de CCF.
* AllowedOAuthFlows - le estamos diciendo a Cognito que este cliente solo permitirá CCF
* AllowedOAuthScopes - Necesitamos asegurarnos de que este array contenga ámbitos que hayan sido definidos en nuestro servidor de recursos.
* AllowedOAuthFlowsUserPoolClient - Esto es lo que nos permite usar funcionalidades de OAuth estándar con nuestro cliente de grupo de usuarios.

He visto escenarios de implementación donde el servidor de recursos se implementa después del cliente y obtenemos un error que dice que el ámbito no existe, esta es la razón por la que estoy agregando explícitamente el `DependsOn` para este recurso.

### 4. Conexión a API Gateway
Para indicarle a nuestro API Gateway que se autentique usando nuestro nuevo grupo de usuarios de Cognito, necesitamos agregar la propiedad Auth, se verá algo así.
```yaml
  Auth:
    DefaultAuthorizer: ClientCognitoAuthorizer
    Authorizers:
      ClientCognitoAuthorizer:
        UserPoolArn: !Ref CognitoUserPoolArn
        AuthorizationScopes:
          - layerless-esbuild/echo
```

Estamos consumiendo el ARN del grupo de usuarios del Stack de Autenticación y permitiendo el ámbito que hemos creado en nuestro servidor de recursos.

# Probándolo usando Postman

## 1. Obtener datos de autenticación
Para poder probar esto, necesitamos ir a la consola y obtener el Id de Cliente y el Secreto de Cliente que se generaron. Estos se encuentran en el servicio de Cognito seleccionando tu grupo de usuarios y yendo a la sección Integración de Aplicaciones. En la parte inferior, verás tu nuevo cliente de aplicación, una vez que lo abras, verás algo como la imagen a continuación. Puedes copiar el Id de Cliente y el Secreto de Cliente desde aquí, usaremos estos valores en los próximos pasos. 

![Consola de AWS mostrando un user pool client en Cognito donde podemos obtener el id y el secreto](/img/api-cognito/cognito-client-keys.png)

> Por favor, maneja estos valores con cuidado, si se comprometen, alguien podría obtener acceso a tus APIs y hacer cosas maliciosas.

## 2. Ejecutar una solicitud no autenticada
Para verificar que nuestra API es segura, primero ejecutaremos una solicitud no autenticada. Para hacer esto, llamaremos a nuestro punto final sin configurar nada para la autenticación, cuando enviemos esta solicitud, deberíamos recibir una respuesta 401 - No autorizada como se muestra a continuación.  

![Solicitud de Postman mostrando una respuesta no autorizada](/img/api-cognito/postman-unauthorized.png)

## 3. Configurar datos de autenticación en Postman
Con los valores, ahora utilizaremos una nueva función [de Postman llamada **Vaults**](https://learning.postman.com/docs/sending-requests/postman-vault/postman-vault-secrets/) que nos permite almacenar de forma segura datos sensibles. Para hacer esto, iremos a la sección Vault en la parte inferior de la ventana y agregaremos nuestros secretos.  

![Ventana de Vault de Postman mostrando un elemento para clientId yclientSecret y sus valores enmascarados](/img/api-cognito/postman-vault.png)

## 4. Configurar la autenticación de la solicitud
Ahora de regreso en la solicitud de Postman, podemos configurar la autenticación. Necesitamos configurar algunas cosas aquí.    
* Grant type - Esto tendrá un valor de *Client Credentials*
* Access Token URL - El valor para tu grupo de usuarios específico variará según tu dominio de grupo de usuarios configurado. Se verá algo así `https://[your-domain].auth.us-east-1.amazoncognito.com/oauth2/token`
* Client ID - Lo obtendremos del vault configurando un valor de `{{vault:clientId}}`
* Client Secret - También del vault con un valor de `{{value:clientSecert}}`
* Scope - Esto se basará en lo que se estableció en el servidor de recursos. A partir de nuestro ejemplo, será  `layerless-esbuild/echo`.

![Colnfiguracion de Postman con todos los valores mencionados arriba completados](/img/api-cognito/postman-authorization-configuration.png)

## 5. Obtener un token
Ahora podemos obtener un nuevo token de acceso yendo al fondo de la sección de Autorización y presionando el botón Obtener Nuevo Token de Acceso. Si es exitoso, recibirás un mensaje donde puedes hacer clic en un botón que dice Usar Token. Al presionar ese botón, recibirás el valor presentado en el Token de Acceso y se utilizará en el encabezado Autorización de tu solicitud.

Si enviamos la solicitud ahora, recibiremos una respuesta exitosa.

![Postman mostrando una solicitud exitosa](/img/api-cognito/postman-success.png)

# Conclusión
Lamentablemente, la seguridad a menudo se deja como una reflexión posterior, tal como me sucedió a mí con todas las APIs que he creado. En esta publicación pudimos comprender algunos de los conceptos relacionados con la autenticación y los recursos necesarios para configurar esto en AWS con Amazon Cognito. También probamos que nuestra API ahora es segura y cómo podemos obtener un token para autenticarnos contra ella.
Espero que esto permita a las personas agregar una capa básica de seguridad a sus APIs para que hagamos que los usuarios maliciosos trabajen un poco más.

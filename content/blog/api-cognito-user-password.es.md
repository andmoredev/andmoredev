+++
title = "Usando Amazon Cognito con el flujo de autenticación de usuario-contraseña"
date = 2024-07-31T00:00:00-00:00
draft = false
description = "En mayo publiqué un artículo sobre cómo asegurar las API utilizando la autenticación de máquina a máquina. Justo un día después, AWS Cognito cambió su modelo de precios y ahora mi solución propuesta generaria costos para mí. En este artículo, explicaré una configuración diferente utilizando el flujo de autenticación de usuario-contraseña. Esto nos permitirá autenticarnos desde automatizaciones y desde Postman, manteniéndonos en la capa gratuita."
tags = ["AWS", "Seguridad", "SAM"]
[[images]]
  src = "img/api-cognito-user-password/title-es.png"
  alt = ""
  stretch = "stretchH"
+++

En mi artículo titulado [Asegurar API Gateway con Amazon Cognito usando SAM](https://www.andmore.dev/es/blog/api-cognito/) hablé sobre diferentes términos de autenticación y expliqué cómo configurar el [Flujo de Credenciales del Cliente](https://auth0.com/docs/get-started/authentication-and-authorization-flow/client-credentials-flow), pero Cognito introdujo recientemente [cambios en la estructura de precios para la autenticación de máquina a máquina](https://aws.amazon.com/about-aws/whats-new/2024/05/amazon-cognito-tiered-pricing-m2m-usage/) que implicarían costos y mi objetivo principal es hacer esto manteniéndonos en el nivel gratuito para proyectos personales que no generarán ingresos. Por eso, en este artículo voy a configurar Amazon Cognito utilizando un flujo diferente llamado autenticación basada en usuario y contraseña. Con este tipo de autenticación, se nos cobra en función de los Usuarios Activos Mensuales (MAUs) y AWS nos ofrece los primeros 50,000 MAUs de forma gratuita, y en mi caso, esto generalmente se mantendrá en 1 por proyecto, así que debería estar bien.

## ¿Cómo funciona el flujo de usuario-contraseña?
Inicialmente pensé que podría usar [Autenticación Básica](https://en.wikipedia.org/wiki/Basic_access_authentication) donde se proporciona el nombre de usuario y la contraseña codificados en el encabezado *Authorization*, pero así no funciona. En la siguiente imagen se muestran todas las interacciones que ocurren para obtener una solicitud autenticada.

![Interacciones del flujo de autenticación](/img/api-cognito-user-password/USER_PASSWORD_AUTH-flow-summary.png)

Vamos a profundizar en cada interacción:
1. El solicitante realiza una solicitud con el nombre de usuario y la contraseña a su API.
2. La API realiza una llamada a Cognito para obtener los tokens de acceso, id y refresh, y los devuelve al usuario.
3. El usuario proporciona el token de id en el encabezado de autorización con cada llamada a la API.
4. La API verifica el token con Cognito para asegurarse de que sea válido.

## Configurándolo todo con SAM
Vamos a mantener la misma arquitectura para los recursos de Amazon Cognito para simplificar la configuración de cada API y poder administrar los usuarios desde un solo lugar.

![Arquitectura de la pila de CloudFormation](/img/api-cognito-user-password/stack-architecture.png)

Ahora veamos cómo se configura cada una de estas piezas utilizando SAM.

### Stack de auth
Este stack tiene los recursos que serán consumidos por nuestras stacks de API.
La configuración completa de la stack de autenticación se puede encontrar [en este repositorio de GitHub](https://github.com/andmoredev/cognito-auth).

#### 1. Actualizaciones en el Grupo de Usuarios
A diferencia de nuestra configuración de M2M que no necesitaba ninguna propiedad para el grupo de usuarios, para este tipo de autenticación necesitamos configurar propiedades que especifiquen los atributos que se alojarán para el usuario, como el correo electrónico, los nombres, etc.
```diff yaml
  CognitoUserPool:
    Type: AWS::Cognito::UserPool
+   Properties:
+     UsernameAttributes:
+       - email
+     AutoVerifiedAttributes:
+       - email
+     VerificationMessageTemplate:
+       DefaultEmailOption: CONFIRM_WITH_LINK
+     EmailConfiguration:
+       EmailSendingAccount: COGNITO_DEFAULT
```

Los atributos del grupo de usuarios son:
* **UsernameAttributes** - esto especifica qué se permite como nombre de usuario. Las opciones aquí son *email* o *phone_number*.
* **AutoVerifiedAttributes** - atributos que se permiten verificar automáticamente por Cognito. Por ejemplo, estamos usando *email*, lo que significa que Cognito enviará automáticamente un correo electrónico de verificación al usuario. Si no configuramos este atributo, un administrador tendría que verificar manualmente a los usuarios en Cognito.
* **VerificationMessageTemplate** - aquí puedes configurar tu propia plantilla para el correo electrónico que se enviará a los usuarios para verificar. En este ejemplo, estamos utilizando una opción predeterminada proporcionada por Cognito donde los usuarios confirmarán haciendo clic en un enlace. La otra opción es *CONFIRM_WITH_CODE*, donde el usuario recibirá un código y deberá ingresarlo manualmente para verificar.
* **EmailConfiguration** - esta propiedad nos permite configurar el correo electrónico del remitente para la verificación u otras comunicaciones que ocurran desde Cognito. En nuestro caso, estamos utilizando *COGNITO_DEFAULT*, lo que reduce la cantidad de configuración necesaria para obtener un correo electrónico verificado de Amazon SES. *COGNITO_DEFAULT* tiene algunos [límites](https://docs.aws.amazon.com/cognito/latest/developerguide/quotas.html) que debes tener en cuenta si lo estás utilizando.

### Stack de API
Nuestra stack de API se simplifica al utilizar este flujo. ¿Por qué? No necesitamos crear un servidor de recursos ya que no utilizaremos capacidades de OAuth.

#### 1. Eliminar UserPoolResourceServer
Como mencioné antes, ya no necesitamos este recurso. Así que deshazte de él eliminándolo de la plantilla.

#### 2. Actualizaciones del Cliente del Grupo de Usuarios
A continuación se muestra la definición de nuestro cliente del grupo de usuarios.
```diff
  CognitoUserPoolClient:
    Type: AWS::Cognito::UserPoolClient
    Properties:
      UserPoolId: !Ref CognitoUserPoolId
-     GenerateSecret: true
-     AllowedOAuthFlows:
-       - client_credentials
-     AllowedOAuthScopes:
-       - layerless-esbuild/api
-     AllowedOAuthFlowsUserPoolClient: true
+     SupportedIdentityProviders:
+       - COGNITO
+     ExplicitAuthFlows:
+       - ALLOW_USER_PASSWORD_AUTH
+       - ALLOW_REFRESH_TOKEN_AUTH
```

Se deben realizar algunos cambios para que este flujo de autenticación funcione.
* **GenerateSecret** - Primero eliminamos esta propiedad ya que no es necesaria para este flujo.
* **AllowedOAuthFlows** - Cuando se utiliza el flujo de autenticación de usuario/contraseña, no es necesario configurar esta propiedad.
* **AllowedOAuthScopes** - Como no estamos utilizando un flujo de OAuth, no es necesario configurar los scopes.
* **SupportedIdentityProviders** - Vamos a utilizar *COGNITO* como nuestro único proveedor. Puedes configurar diferentes proveedores de identidad para simplificar el inicio de sesión de tus usuarios utilizando Google, Facebook o cualquiera de los proveedores admitidos.
* **ExplicitAuthFlows**: aquí configuraremos *ALLOW_USER_PASSWORD_AUTH*, que nos permitirá autenticarnos utilizando un nombre de usuario y una contraseña. También he agregado *ALLOW_REFRESH_TOKEN_AUTH* ya que es necesario, pero no vamos a hacer actualizaciones de tokens en este artículo.

#### 4. Actualizaciones en API Gateway
Lo único que necesita cambiar en API Gateway es la eliminación de *AuthorizationScopes*. Para este flujo, esto no es necesario.
```diff
  Auth:
    DefaultAuthorizer: ClientCognitoAuthorizer
    Authorizers:
      ClientCognitoAuthorizer:
        UserPoolArn: !Ref CognitoUserPoolArn
-       AuthorizationScopes:
-         - layerless-esbuild/echo
```

¡¡Eso es todo!! Ahora hemos configurado con éxito todo lo necesario para autenticar nuestra API utilizando el flujo de usuario-contraseña.

[En este repositorio de GitHub](https://github.com/andmoredev/api-gateway-auth-with-cognito) puedes ver el ejemplo completo.

## Pruebas con Postman

### 1. Obtener el Id del Cliente
Solo necesitamos el Id del cliente al autenticarnos con el flujo de usuario-contraseña, esto se debe a que vamos a ingresar un nombre de usuario y una contraseña para autenticarnos y es de ahí de donde vendrá el token. Por lo tanto, obtendremos este valor desde la consola yendo a nuestra nueva aplicación cliente.

![Consola de AWS mostrando un cliente de aplicación en Cognito donde podemos obtener el Id del cliente](/img/api-cognito-user-password/cognito-client-keys.png)

### 2. Solicitar tokens de autenticación
Para solicitar los tokens, debemos llamar a Amazon Cognito especificando que estamos realizando un comando *InitiateAuth*.
Esto requerirá que realicemos una llamada POST a *https://cognito-idp.us-east-1.amazonaws.com*. La región cambiará según donde hayas creado tu Grupo de Usuarios. Especificamos el comando agregando *X-Amz-Target*, también necesitamos especificar *Content-Type* ya que es un tipo específico de AWS. A continuación se muestra una captura de pantalla que muestra cómo deben verse los encabezados.

![Encabezados de la solicitud en Postman](/img/api-cognito-user-password/postman-headers.png)

El cuerpo requiere los siguientes parámetros:
* **AuthFlow** - Aquí especificamos que queremos usar el flujo de autenticación de usuario-contraseña estableciendo un valor de *USER_PASSWORD_AUTH*.
* **ClientId** - Estableceremos el valor al que obtuvimos en el paso 1.
* **AuthParameters** - en el objeto proporcionaremos el *USERNAME* y el *PASSWORD* de nuestro usuario.

![Cuerpo de la solicitud en Postman](/img/api-cognito-user-password/postman-body.png)

Cuando envíes esta solicitud, deberías recibir una respuesta con el *AccessToken*, *IdToken*, *RefreshToken*, *TokenType* y *ExpiresIn*. Esto también devuelve los *ChallengeParameters*, pero este flujo no requiere generar ninguna respuesta de desafío. (He editado la respuesta completa de la imagen para no exponer las claves completas).

![Cuerpo de la respuesta en Postman](/img/api-cognito-user-password/postman-response.png)

### 5. Realizar una solicitud
Tomando el *IdToken* de la respuesta que obtuvimos en la solicitud anterior, ahora podemos usarlo para autorizar la solicitud  como se muestra a continuación.

![Configuración de autorización y solicitud exitosa en Postman](/img/api-cognito-user-password/postman-authorization-configuration.png)

## Autenticación de llamadas automatizadas
Siempre que cambio los métodos de autenticación, tengo problemas para configurarlo todo para mis pruebas automatizadas. Esto requiere hacer lo mismo que hicimos en Postman, pero de forma programática. Lo estoy haciendo ejecutando un pequeño script de NodeJS que realizará la llamada, como se muestra a continuación.

```javascript
const initiateAuthResponse = await axios({
    url: process.env.COGNITO_URL,
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-amz-json-1.1',
      'X-Amz-Target': 'AWSCognitoIdentityProviderService.InitiateAuth'
    },
    data: JSON.stringify({
      ClientId: process.env.CLIENT_ID,
      AuthFlow: 'USER_PASSWORD_AUTH',
      AuthParameters: {
        USERNAME: process.env.USERNAME,
        PASSWORD: process.env.PASSWORD
      }
    })
  });


  const token = initiateAuthResponse.data.AuthenticationResult.IdToken;
```

Si observas el código anterior, debería resultarte muy familiar a lo que hicimos en Postman. Realizando una solicitud POST con los encabezados y el cuerpo necesarios. Una vez que obtengas la respuesta, puedes hacer lo que necesites con los tokens.

He proporcionado 2 ejemplos de cómo manejar las credenciales de usuario para que este script las utilice.
1. Crear usuario programáticamente: en este ejemplo, estamos [creando un usuario de Cognito programáticamente](https://www.andmore.dev/es/blog/create-cognito-user-programatically/) y utilizando las credenciales para autenticarnos. Hacemos esto configurando los valores como variables de entorno utilizando el comando *>> $GITHUB_ENV*. Al final de la ejecución, eliminamos el usuario para no tener una cantidad infinita de usuarios de automatización huérfanos.
```yaml
  test-api-with-user-password-auth-inline-create-user:
    name: Ejecutar Portman con USER_PASSWORD_AUTH - Crear usuario en línea
    runs-on: ubuntu-latest
    environment: ${{ inputs.ENVIRONMENT }}
    steps:
      - uses: actions/checkout@v4

      - name: Configurar AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: us-east-1
          role-to-assume: ${{ secrets.PIPELINE_EXECUTION_ROLE }}
          role-session-name: create-test-user
          role-duration-seconds: 3600
          role-skip-session-tagging: true

      - name: Crear usuario de prueba
        run: |
          username=$(uuidgen)@andmore.dev
          password=$(uuidgen)G1%
          echo "USERNAME=$username" >> $GITHUB_ENV;
          echo "PASSWORD=$password" >> $GITHUB_ENV;
          aws cognito-idp admin-create-user --user-pool-id ${{ inputs.COGNITO_USER_POOL_ID }} --username $username --message-action SUPPRESS
          aws cognito-idp admin-set-user-password --user-pool-id ${{ inputs.COGNITO_USER_POOL_ID }} --username $username  --password $password --permanent

      - name: Probar API
        env:
          COGNITO_URL: ${{ secrets.COGNITO_URL }}
          CLIENT_ID: ${{ secrets.COGNITO_CLIENT_ID }}
        run: |
          npm ci

          node ./portman/get-auth-token/user-password-auth.mjs
          npx @apideck/portman --cliOptionsFile portman/portman-cli.json --baseUrl ${{ inputs.BASE_URL }}

      - name: Eliminar usuario de prueba
        run: |
          aws cognito-idp admin-delete-user --user-pool-id ${{ inputs.COGNITO_USER_POOL_ID }} --username $USERNAME
```

2. Usar secretos de GitHub: 
Esta es una de las rutas más simples, pero requiere tener un usuario ya configurado. Todo lo que necesitas hacer es almacenar las credenciales en los secretos de GitHub y hacer referencia a ellos en el flujo.
```yaml
  test-api-with-user-password-auth-github-secrets:
    name: Ejecutar Portman con USER_PASSWORD_AUTH - Cargar secretos de GitHub
    runs-on: ubuntu-latest
    environment: ${{ inputs.ENVIRONMENT }}
    steps:
      - uses: actions/checkout@v4

      - name: Configurar AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: us-east-1
          role-to-assume: ${{ secrets.PIPELINE_EXECUTION_ROLE }}
          role-session-name: create-test-user
          role-duration-seconds: 3600
          role-skip-session-tagging: true

      - name: Probar API
        env:
          COGNITO_URL: ${{ secrets.COGNITO_URL }}
          CLIENT_ID: ${{ secrets.COGNITO_CLIENT_ID }}
          USERNAME: ${{ secrets.COGNITO_USERNAME }}
          PASSWORD: ${{ secrets.COGNITO_PASSWORD }}
        run: |
          npm ci

          node ./portman/get-auth-token/user-password-auth.mjs
          npx @apideck/portman --cliOptionsFile portman/portman-cli.json --baseUrl ${{ inputs.BASE_URL }}
```

Aquí hay dos consideraciones a tener en cuenta. Si vas a tener un usuario por repositorio, puede resultar una carga mantener estos usuarios a medida que crecen tus aplicaciones y repositorios. Por otro lado, si compartes un solo usuario con todos tus repositorios, podrías estar introduciendo vulnerabilidades de seguridad, ya que será más fácil comprometer esta contraseña.

## Conclusión
En este artículo hemos actualizado nuestra autenticación para utilizar el flujo de usuario-contraseña en lugar de M2M para nuestras API, lo que nos permite mantenernos dentro de la capa gratuita de Cognito. Hemos verificado que aún podemos autenticarnos con Postman y nuestras pruebas automatizadas. Espero que esto muestre la flexibilidad que ofrece Cognito y cómo puedes configurarlo de manera diferente para satisfacer tus casos de uso. Exploraré los otros flujos de autenticación para que puedas elegir fácilmente el que tenga más sentido para ti.


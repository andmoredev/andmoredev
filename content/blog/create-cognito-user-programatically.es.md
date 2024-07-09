+++
title = "Creando un usuario en Amazon Cognito de forma programática"
date = 2024-06-30T00:00:00-00:00
draft = false
description = "Breve publicación para recorrer los pasos para crear un usuario en Amazon Cognito para ser utilizado en la automatización"
tags = ["AWS", "Seguridad", "Cognito"]
[[images]]
  src = "img/create-cognito-user-programmatically/title-es.png"
  alt = ""
  stretch = "stretchH"
+++

Cuando tienes pipelines de CI/CD que ejecutan pruebas automatizadas contra tus APIs, es posible que necesites crear usuarios dinámicamente en Amazon Cognito para ejecutarlas. Si ese es el caso, estás en el lugar correcto. En esta publicación, repasaremos lo que necesitas hacer para crear un usuario válido en Cognito que pueda ser utilizado por tu automatización.

## Crear usuario en Cognito
Lo primero que debemos hacer es crear un usuario en nuestro Cognito User Pool. Dado que esto es para fines de automatización, estoy creando usuarios utilizando el dominio *andmore.dev* que poseo. Quiero que estos usuarios estén limitados al repositorio específico que está ejecutando el pipeline de CI/CD, por lo que estoy generando un nombre de usuario con un UUID.

```bash
username=$(uuidgen)@andmore.dev
aws cognito-idp admin-create-user --user-pool-id [TU-ID-DE-USER-POOL] --username $username --message-action SUPPRESS
```
En los comandos anteriores, estoy generando el nombre de usuario con un nuevo UUID. Dado que tengo habilitado el reenvío de correo electrónico para mi dominio, cada nuevo usuario me enviará un correo electrónico. Para evitar esto, he agregado la propiedad `--message-actions` como `SUPPRESS`, esto desactivará el envío de cualquier mensaje.

¿Con esto es suficiente, verdad? Yo también pensé eso, pero hay más. El usuario se crea con una contraseña temporal y aún no se puede utilizar. Dado que estamos haciendo esto para un proceso automatizado, no podemos realmente seguir el flujo de abrir el correo electrónico, iniciar sesión y cambiar la contraseña. Así que veamos qué más necesitamos hacer.

## Verificar usuario
Cognito ofrece un comando que permite a los administradores establecer la contraseña de un usuario, evitando los pasos manuales mencionados anteriormente.

```bash
password=$(uuidgen)G1%
aws cognito-idp admin-set-user-password --user-pool-id [TU-ID-DE-USER-POOL] --username $username  --password $password --permanent
```
Primero necesitamos generar una contraseña. Para mantenerlo simple, lo estoy haciendo creando un nuevo UUID y agregando una letra mayúscula, un número y un carácter especial, ya que mi grupo de usuarios requiere esto como parte de la política de contraseñas que hemos establecido. Ahora podemos proporcionar esto al comando `admin-set-user-password` y asegurarnos de pasar la bandera `--permanent`, ya que de forma predeterminada establecerá la contraseña como temporal y no nos permitirá usarla.

Ahora el usuario está configurado para poder autenticarse utilizando cualquiera de los flujos que hayas configurado para tu Cognito User Pool Client.

## Eliminar el usuario
Probablemente no quieras mantener todos estos usuarios una vez que tu pipeline haya terminado. Para eliminar el usuario, deberás llamar al comando `admin-delete-user` con el nombre de usuario generado.
```bash
aws cognito-idp admin-delete-user --user-pool-id [TU-ID-DE-USER-POOL] --username $username
```

## Permisos de IAM
Para poder ejecutar estos comandos, necesitarás permisos para ejecutarlos en tu grupo de usuarios de Cognito. A continuación, se muestra un ejemplo de política de IAM que te otorgará los permisos necesarios para ejecutar los tres comandos. Asegúrate de restringir el acceso a esta política, ya que con este nivel de acceso, las personas podrían crear usuarios y potencialmente causar daños en tu producto.
```json
{
  "Effect": "Allow",
  "Action": [
      "cognito-idp:AdminCreateUser",
      "cognito-idp:AdminSetUserPassword",
      "cognito-idp:AdminDeleteUser",
  ],
  "Resource": [
      "TU-ARN-DE-USER-POOL",
  ]
}
```

## Conclusión
Pudimos crear y verificar fácilmente un usuario para ser utilizado por nuestra automatización, así como limpiar después de que hayamos terminado para no quedarnos con usuarios huérfanos por todas partes.
Espero que esto te sea útil. La documentación de Cognito realmente no es tan directa y puede ser difícil encontrar lo que estás buscando, por eso estoy tratando de proporcionar tutoriales sobre Cognito, así que mantente atento, ya que tengo más contenido preparado sobre este tema.
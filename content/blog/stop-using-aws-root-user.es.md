+++
title = "Deja de usar tu usuario raíz de AWS"
date = 2021-02-22T13:56:07-06:00
draft = false
description = "Enserio, ya parale! Estas poniendo a tu cuenta en riesgo"
tags = ["AWS", "Security", "IAM"]
[[images]]
  src = "img/2021/02/pic03.jpg"
  alt = "Padlock"
  stretch = "stretchH"
+++

Cuendo creas una cuenta nueva en AWS te va a crear un *usuario raíz* con el correo y contraseña que usaste para crearla. Lo más fácil es utilizar este usuario para las actividades del día a día. Vamos a ver porque no deberiamos de hacer esto y mostrar como configurar el usuario raíz para proteger nuestra cuenta.

El *usuario raíz* tiene acceso a todos los servicios de AWS y a todos los recursos que existen en la cuenta. Si las credenciales para la cuenta raíz fueran robadas, ellos podrían entrar y cambiar cualquier cosa en la cuenta, dandoles la oportunidad de usar maliciosamente los datos o recursos en la cuenta, podrían hacerte gastar mucho dinero creando recursos innecesarios.

## Configuración recomendada para el usuario raíz
Como es tan crítico mantener seguro este usuario y fuera del alcance de usuarios maliciosos hay algunas cosas que podemos hacer para protegerlo.

* No crear claves de acceso para el usuario raíz. Mejor crear un usuario administrador en IAM para tí.
* Nunca compartir las credenciales del usuario raíz.
* Usar una contraseña complicada.
* Habilitar autenticación multifactor.

## Cuando debes usar el usuario raíz?
Para llevar a cabo las actividades de diario y acceder recursos de AWS deberias usar un usuario de [IAM con los permisos necesarios](https://docs.aws.amazon.com/es_es/IAM/latest/UserGuide/best-practices.html#lock-away-credentials) siguiendo el Principio de pivilegio mínimo.

Hay pocas actividades que solamente el usuario raíz puede hacer, estas son cosas como:
* Actualizar el nombre de la cuenta
* Cambiar las credenciales para la cuenta
* Restaurar los permisos de un usuario IAM (en caso de que el único administrador se haya quitado sus propios permisos)
* Cerrar la cuenta de AWS

[Aqui hay una lista completa](https://docs.aws.amazon.com/es_es/general/latest/gr/root-vs-iam.html#aws_tasks-that-require-root)

## Como crear un usuario IAM administrador para ti
Es muy simple configurar un usuario IAM para ti en la consola AWS para comenzar a trabajar en tu cuenta.

Sigue estos pasos y estarás listo:

1. Entra a tu cuenta de AWS usando las credenciales del usuario raíz (esta debe ser la ultima vez que lo usas al menos que estes realizando algunos de las actividades mencionadas anteriormente)
2. Ve al servicio de IAM
3. En la sección de *Users* presiona *Add User*
4. Ingresa `administrator` como el nombre de usuario
5. Selecciona *Programatic access* y *AWS Management Console access* para *Access Types*.
6. Ingresa una contraseña complicada
7. Deselecciona el campo de *Require a password reset* y presiona *Next*
![Agrega detalles de usuario](/img/2021/02/pic01.jpg)
8. Selecciona la política llamada AdministratorAccess
![Politica de Administrador](/img/2021/02/pic02.jpg)
9. Continua con los siguientes pasos para terminar de crear al usuario.

Despues de que el usuario haya sido creado vas a ser presentado con las claves de accesso para poder ingresar AWS programáticamente. Guarda estas credenciales en un lugar seguro para utilizarlos con el AWS CLI.

De ahora en adelante este va a ser el usuario que debes usar para las operaciones de diario en tu cuenta de AWS.

## Conclusión
Utilizando el usuario raíz para las operaciones del día a día pudieras estar poniendo en riesgo tu cuenta, siguiendo estos pasos puedes proteger tu cuenta pudiendo seguir haciendo todo lo necesario.

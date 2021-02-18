+++
title = "Deja de usar tu cuenta raíz de AWS"
date = 2021-02-15T13:56:07-06:00
draft = true
description = "Enserio, ya parale! Esta es una guía para enseñarte que es lo que tienes que hacer para aumentar la seguridad de tu cuenta"
tags = ["aws", "security", "IAM"]
[[images]]
  src = "img/2021/02/pic03.jpg"
  alt = "Padlock"
  stretch = "stretchH"
+++

Cuendo creas una cuenta nueva en AWS te va a crear un *usuario raíz* con el correo y contraseña que usaste para crear la cuenta. Y la ruta mas simple que tomamos es utilizar esa cuenta para las operaciones de diario. Vamos a ver porque no deberiamos de hacer esto y mostrar una configuración sencilla para dejar de hacer esto en nuestra cuenta.

El usuario raíz tiene acceso a todos los servicios deAWS y a todos recursos en la cuenta. Si las credenciales para la cuenta raíz fueran robadas, ellos podrían entrar y cambiar cualquier cosa en la cuenta, dandoles la oportunidad de usar maliciosamente los datos guardados, borrar datos y/o recursos, crear recursos que los hagan pagar un alto costo y muchas otras cosas que les podrian afectar.

## Configuración recomendada para el usuario raíz
Como es tan critico mantener segura el usuario raíz y fuera del alcance the usuarios maliciosos hay algunas cosas que podemos hacer para mantenerlo seguro.

* No crear claves de acceso para el usuario raíz. Mejor crear un usuario administrador an IAM para ti.
* Nunca compartir las credenciales del usuario raíz con alguien.
* Usar una contraseña complicada
* Habilitar autenticación multifactor para el usuario raíz.

## Cuando debes usar el usuario raíz
Para llevar a cabo las actividades de diario y accesso a los recursos de AWS deberia usar un usuario de IAM con los permisos necesarios
To handle day to day tasks and access to AWS resources you should use an [IAM user with the appropriate permissions](https://docs.aws.amazon.com/es_es/IAM/latest/UserGuide/best-practices.html#lock-away-credentials) siguiendo el Principio de Privilegio Mínimo

Algunas de las actividades que solamente puede llevar a cabo el usuario raíz son:
* Actualizar el nombre de la cuenta
* Cambiar las credenciales para la cuenta
* Restaurar los permisos de un usuario IAM (en caso de que el único administrador se quite sus propios persmisos)
* Cerrar la cuenta de AWS

[Aqui hay una lista completa](https://docs.aws.amazon.com/es_es/general/latest/gr/root-vs-iam.html#aws_tasks-that-require-root)

## Como crear un usuario IAM administrador para ti
Es muy simple configurar un usuario IAM para ti en la consola AWS para comenzar a trabajar en tu cuenta.

Sigue estos pasos:

1. Entra a tu cuenta de AWS usando las credenciales del usuario raíz (esta debe ser la ultima vez que lo usas al menos que estes realizando algunos de las actividades limitadas a este usuario)
2. Vel al servicio de IAM
3. En la sección de *Users* presiona *Add User*
4. Ingresa `administrator` como el nombre de usuario
5. Selecciona *Programatic access* y *AWS Management Console access* para *Access Types*.
6. Ingresa una contraseña complicada
7. Deselecciona el campo de *Require a password reset* y presiona *Next*
![Agrega detalles de usuario](/img/2021/02/pic01.jpg)
8. Selecciona la politica llamada AdministratorAccess
![Politica de Administrador](/img/2021/02/pic02.jpg)
9. Continua con los siguientes pasos para crear al usuario.

Despues de que el usuario haya sido creado vas a ser presentado con las claves de accesso para poder ingresar AWS programaticamente. Guarda esta credenciales en un lugar seguro para utilizarlos con el AWS CLI.

De ahora en adelante este va a ser el usuario que debes usar para las operaciones de diarion en tu cuenta de AWS.

## Conclusión
Utilizando tu usuario the raíz para las operaciones del día a día pudieras estar poniendo en riesgo tus aplicaciones y los datos relacionados con ellos, siguiendo estos pasos puedes reducir la posibilidad de que ocurran estos ataques mientras puedas continuar realizando todo lo necesario en tu cuenta.

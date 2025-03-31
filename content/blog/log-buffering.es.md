+++
title = "Búfer de logs con Lambda Powertools"
date = 2025-03-18T00:00:00-00:00
draft = false
description = "Vamos a entender qué es el búfer de logs, cómo se configura en tus funciones Lambda y a ver resultados reales en CloudWatch probando diferentes configuraciones."
tags = ["AWS", "Serverless", "Lambda", "Powertools"]
[[images]]
  src = "img/log-buffering/title-es.jpg"
  alt = "Imagen de título con Andres señalando un texto que dice búfer de logs con powertools for AWS Lambda"
  stretch = "stretchH"
+++

El equipo de Lambda Powertools lanzó una nueva funcionalidad que permite a tus funciones Lambda almacenar en búfer los logs. Eso sonó genial, pero no entendía cómo funcionaría o cómo se mostrarían los logs en CloudWatch. Decidí probarlo y dar un ejemplo visual de cómo se ve con diferentes configuraciones.

### ¿Qué es el búfer de logs?
Esto es lo que dice la documentación de Lambda Powertools:

> El búfer de logs te permite almacenar logs para una solicitud o invocación específica. Habilita el búfer de logs pasando **logBufferOptions** al inicializar una instancia de Logger. Puedes almacenar logs en los niveles **WARNING**, **INFO**, **DEBUG** o **TRACE**, y vaciarlos automáticamente en caso de error o manualmente según sea necesario.

Los tres términos principales de esa explicación son *búfer*, *nivel* y *vaciar*. Vamos a explicar cada uno de estos en el contexto del búfer de logs.

* **Búfer** — Esto significa que la función Lambda retendrá los logs en memoria sin enviarlos a CloudWatch a menos que sea necesario.
* **Nivel** - El nivel de log en el que deseas comenzar a almacenar en búfer los logs. Puedes elegir en qué nivel comenzar a almacenar en búfer. Por ejemplo, si configuras el búfer de logs para que lo haga en el nivel INFO, almacenará esos logs y cualquier nivel de log con un valor numérico inferior. En este caso, también almacenará TRACE y DEBUG. A continuación, se presentan todos los niveles de log admitidos y su valor numérico.

|  **Nivel**<br/> | **Valor Numérico**<br/> |
|-----|-----|
|  TRACE<br/> | 6<br/> |
|  DEBUG<br/> | 8<br/> |
|  INFO<br/> | 12<br/> |
|  WARN<br/> | 16<br/> |
|  ERROR<br/> | 20<br/> |
|  CRITICAL<br/> | 24<br/> |
|  SILENT<br/> | 28<br/> |  

* **Vaciar** - Esto significa que los logs almacenados en búfer se enviarán a CloudWatch. Puedes indicarle a Lambda que envíe los logs para solucionar problemas en caso de errores de diferentes maneras. Cubriremos estas opciones en la siguiente sección.

### ¿Por qué almacenar en búfer los logs?
Ahora te estarás preguntando, ¿por qué querría almacenar en búfer los logs? La razón puede no ser inmediatamente evidente. Los logs suelen ser necesarios solo cuando tu aplicación encuentra un error. Esto significa que estamos registrando constantemente cosas para todas las solicitudes exitosas y produciendo logs que nadie verá. Al habilitar el búfer de logs, solo imprimirás los logs cada vez que se produzca un escenario específico o un error, lo que te permitirá reducir la cantidad de logs producidos, el ruido y, aún mejor, tus costos.

### Opciones de configuración
Hay algunas piezas de configuración para el registro en búfer que se pueden establecer al inicializar un logger para manejar las cosas de acuerdo a tus necesidades.

* **enabled**—esto debería ser sencillo. El búfer de logs está desactivado; para comenzar a usarlo, este parámetro deberá estar configurado como *true.*
* **maxBytes**—esta es la cantidad máxima de bytes que el búfer mantendrá en memoria. Si alcanzas el límite máximo, el búfer comenzará a eliminar logs antiguos para hacer campo para los nuevos. Esta configuración te permite proteger a la función Lambda de quedarse sin memoria. El logger será lo suficientemente amable como para informarte que hizo esto cada vez que se vacían los logs. El valor predeterminado para esto es 20480 bytes.
* **bufferAtVerbosity**—aquí es donde configurarás en qué nivel de log deseas comenzar a almacenar en búfer. Cualquier valor que configures será almacenado en búfer, así como cualquier nivel con un valor numérico inferior. El nivel de log predeterminado configurado es DEBUG.
* **flushOnErrorLog**—Este es un valor booleano que le dice al búfer que vacie los logs cada vez que encuentra una llamada a *logger.error*. Al escribir funciones Lambda, normalmente tenemos una declaración try/catch donde registramos cualquier error que se produzca. Esto ayuda a vaciar automáticamente los logs cuando se captura y registra un error. El valor predeterminado de esta opción es *true*.

Pero ahora la pregunta es, ¿qué sucede si no capturo y registro un error? ¿No podré ver los logs almacenados en búfer? Afortunadamente, el equipo de Lambda Powertools pensó en esto, y permiten vaciar los logs de dos maneras adicionales:
*  **flushBufferOnUncaughtError**—*Esta opción solo se permite al inyectar el contexto de Lambda en el logger. Puedes usar esto para vaciar los logs cuando hay un error no capturado. Esta propiedad vaciará los logs cada vez que la función Lambda encuentre un error y no lo hayamos capturado.
* **flush_buffer function**—Puede que desees vaciar los logs cada vez que se alcance un camino de código específico u otros escenarios que sean específicos para tu caso de uso. Para hacer esto, el logger tiene una nueva función llamada *flushBuffer* que enviará manualmente todos los logs almacenados en búfer a CloudWatch.

### ¡Veamos esto en acción!

Para habilitar esta función, todo lo que tenemos que hacer es establecer la propiedad *enabled* en *true* en el objeto logBufferOptions para el constructor Logger, como se muestra a continuación:
```javascript
export const logger = new Logger({
  logBufferOptions: {
    enabled: true
 }
});
```

Ahora, para probar, agregaremos todos los diferentes tipos de logs a nuestro Lambda de esta manera:
```javascript
    logger.trace('Trace Log');
    logger.debug('Debug Log');
    logger.info('Info Log');
    logger.warn('Warn Log');
    logger.error('Error Log');
    logger.critical('Critical Log');
```

Si ejecutamos la función Lambda y vemos los logs producidos en CloudWatch, obtenemos esto:  

![Cloudwatch Logs](/img/log-buffering/logs-1.png)

Los resultados fueron inesperados, pero luego vi los valores predeterminados para las opciones del logger para tratar de entender esto mejor. Hay dos valores predeterminados a tomar en cuenta para este ejemplo: **flushOnErrorLog** está habilitado por defecto, y el nivel de log predeterminado del búfer es DEBUG. Con esto, entendamos por qué los logs se muestran en este orden:

* Los logs TRACE y DEBUG se agregan al búfer debido a la configuración del nivel de log.
* Los logs INFO y WARN se imprimen tal cual ya que no están incluidos en el búfer debido a la configuración del nivel de log.
* Los logs TRACE y DEBUG ahora se muestran porque alcanzamos el log de ERROR, lo que significa que los logs se vacían antes de que se imprima el log de ERROR.
* Finalmente, los logs ERROR y CRITICAL se imprimen.

Si ejecutáramos la misma prueba, pero en este ejemplo, no registramos un error, realmente apreciaríamos el valor del búfer de logs.

![CloudWatch Logs](/img/log-buffering/logs-2.png)

De la imagen anterior, podemos ver que solo los logs no almacenados en búfer llegaron a CloudWatch, lo que significa que si no hay un error, no veremos ni se nos cobrará por los logs almacenados en búfer.

Ahora, configuremos el logger para almacenar en búfer en el nivel WARN.
```javascript
export const logger = new Logger({
  logBufferOptions: {
    enabled: true,
    bufferAtVerbosity: 'WARN'
 }
});
```

Ahora obtenemos menos logs si ejecutamos nuestra función Lambda ya que estamos almacenando en búfer a un nivel más alto.

![CloudWatch Logs](/img/log-buffering/logs-3.png)

Ahora, vamos a sobrecargar intencionalmente nuestro búfer para hacer que alcance el límite máximo. Para este ejemplo, vamos a vaciar manualmente los logs llamando a la función. Esto significa que necesitaremos desactivar la propiedad flushOnErrorLog. Para facilitar esta prueba, estableceré la propiedad maxBytes en 1000.

```javascript
export const logger = new Logger({
  logBufferOptions: {
    enabled: true,
    bufferAtVerbosity: 'WARN',
    flushOnErrorLog: false,
    maxBytes: 1000
 }
});
```

Y para el código de registro, haremos un simple loop *for* y vaciaremos manualmente al final.
```javascript
for (let index = 0; index < 500; index++) {
  logger.trace('Trace Log');
  logger.debug('Debug Log');
  logger.info('Info Log');
  logger.warn('Warn Log');
  logger.error('Error Log');
}

logger.flushBuffer();
```

Cuando ejecutemos esto, Lambda Powertools nos informará que hemos excedido el tamaño del búfer y que algunos de los logs se perdieron para hacer espacio para los nuevos.

![CloudWatch Logs](/img/log-buffering/logs-4.png)

¿Qué sucede si un solo log excede el tamaño del búfer? Bueno, intentémoslo. Para hacer esto, estableceré la propiedad maxBytes en 1 byte.
```javascript
export const logger = new Logger({
  logBufferOptions: {
    enabled: true,
    bufferAtVerbosity: 'WARN',
    flushOnErrorLog: false,
    maxBytes: 1
 }
});
```

Si ejecutamos el mismo código que arriba, veremos un mensaje diciéndonos que el log no pudo ser almacenado en búfer porque era demasiado grande.

![CloudWatch Logs](/img/log-buffering/logs-5.png)

### Conclusión
Estoy muy emocionado de comenzar a usar esto en proyectos y ver cómo puede impactar mis costos de CloudWatch. La mejor parte es que Lambda Powertools se encarga de la mayor parte del trabajo pesado, y todo lo que tenemos que hacer es configurar algunas propiedades para que se comporte como queremos. Quiero agradecer al equipo de Lambda Powertools, ya que siguen impresionándome con toda la gran funcionalidad que entregan que solo hace nuestras vidas más fáciles.

¡Déjame saber qué piensas sobre esta función!

¡Hasta la próxima!

Andres Moreno

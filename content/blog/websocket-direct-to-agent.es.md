+++
title = "Sin intermediarios: Conecta tu UI directamente a un agente de IA vía WebSocket"
date = 2026-07-13T00:00:00-00:00
draft = false
description = "Aprende cómo conectar tu navegador directamente a un agente de IA a través de WebSocket usando Amazon Bedrock AgentCore, eliminando el proxy tradicional de Lambda y habilitando streaming de tokens en tiempo real."
tags = ["AWS", "Serverless", "Amazon Bedrock", "AgentCore"]
[[images]]
  src = "img/websocket-direct-to-agent/title.png"
  alt = "Imagen de título para Sin intermediarios: Conecta tu UI directamente a un agente de IA vía WebSocket"
  stretch = "stretchH"
+++

He estado construyendo agentes de IA desde hace un tiempo, y transmitir las respuestas a una UI siempre ha sido la parte más complicada. En proyectos anteriores probé streaming con API Gateway, streaming de respuestas con Lambda, e incluso AppSync Events a través de una herramienta del agente para notificar a la UI. También consideré agregar mi propio WebSocket API a través de API Gateway, lo cual requiere administrar las rutas `$connect`, `$disconnect` y `$default`, almacenar los IDs de conexión en DynamoDB y enviar mensajes de vuelta a través de `@connections`. Todos estos enfoques se sentían como demasiada ceremonia para algo que debería ser simple.

Fue entonces cuando descubrí que AgentCore tiene soporte integrado para WebSocket. El navegador simplemente se conecta directamente al agente. Sin intermediarios.

Construí WearCast para demostrar esta funcionalidad. Es un agente de IA que te ayuda a elegir qué ponerte según el clima. Me ha resultado muy útil al empacar para un viaje. El código de esta aplicación es público y puedes encontrar la implementación completa en [este repositorio de GitHub](https://github.com/andmoredev/wearcast/).

## El problema con el enfoque tradicional

Normalmente construimos aplicaciones como lo hacemos con APIs REST:
```
Usuario → REST API → Lambda → Servicio de IA → Lambda → REST API → Usuario
```

Cada mensaje hace un viaje completo a través de múltiples intermediarios. La respuesta espera hasta que la generación completa termine, y luego regresa toda de una vez. Para una interfaz de chat, esto se siente lento. Los usuarios se quedan viendo un spinner mientras el modelo genera cientos de tokens que ya podrían estar leyendo.

Incluso si agregas Server-Sent Events o long-polling, sigues armando una experiencia en tiempo real sobre infraestructura que se sentía excesiva. Yo quería algo mejor.

## La arquitectura

Esto es lo que terminé usando:

```
┌─────────────┐          ┌──────────────────┐
│   React UI  │─── JWT ─▶│  API Gateway +   │──▶ Lambda (URL prefirmada)
│  (Cognito)  │          │  Cognito Auth    │
└──────┬──────┘          └──────────────────┘
       │
       │  WebSocket (URL prefirmada con SigV4)
       │  ← ¡Sin intermediarios! Conexión directa →
       ▼
┌──────────────────────────────────┐     ┌────────────────────┐
│  AgentCore Runtime               │────▶│  AgentCore Memory  │
│  (Strands Agent + Bedrock LLM)   │     │  (Persistencia)    │
└──────────────────────────────────┘     └────────────────────┘
```

El navegador se conecta directamente al agente de IA a través de un WebSocket sin necesidad de funciones Lambda que actúen como proxy de los tokens. El agente transmite los tokens directamente al navegador del usuario conforme se generan.

La única infraestructura adicional es durante el handshake inicial, donde intercambiamos un JWT por una URL de WebSocket prefirmada. Después de eso, es una conexión bidireccional directa.

## Cómo funciona

Veamos los tres pasos para que esto funcione.

### Paso 1: Autenticar al usuario

El usuario inicia sesión a través de Amazon Cognito y recibe un token JWT. Nada fuera de lo común aquí.

```typescript
// El frontend se autentica y obtiene un access token
const accessToken = await authService.getAccessToken();
```

### Paso 2: Intercambiar el JWT por una URL de WebSocket prefirmada

Aquí es donde se pone interesante. El frontend hace una sola llamada REST a nuestro backend:

```typescript
const presignedData = await apiService.getPresignedWebSocketUrl(sessionId, accessToken);
```

Detrás de escena, una función Lambda:
1. Valida el JWT (el autorizador de Cognito en API Gateway se encarga de esto)
2. Usa su rol de IAM para generar una URL de WebSocket prefirmada con AWS SigV4 que apunta directamente al AgentCore Runtime
3. Incluye la identidad del usuario y el ID de sesión como parámetros de consulta en la URL firmada
4. Devuelve la URL prefirmada al navegador

```javascript
// Lambda: Generar URL de WebSocket prefirmada
const wsHost = `bedrock-agentcore.${region}.amazonaws.com`;
const wsPath = `/runtimes/${runtimeArn}/ws`;

const signer = new SignatureV4({
  service: 'bedrock-agentcore',
  region,
  credentials,
  sha256: Sha256
});

const signedRequest = await signer.presign(request, { expiresIn: 300 });
const presignedWsUrl = formatSignedUrl(signedRequest).replace('https://', 'wss://');
```

La URL prefirmada es válida por 5 minutos, suficiente para establecer la conexión, pero lo suficientemente corta para limitar la exposición. Después de que la URL expira, se puede reconectar obteniendo una nueva URL prefirmada, similar a cómo manejamos tokens de autenticación que expiran.

### Paso 3: Conectarse directamente al agente

El navegador abre una conexión WebSocket usando la URL prefirmada. No se necesitan headers personalizados, toda la autenticación está incluida en los parámetros de consulta de la URL a través de SigV4.

```typescript
this.ws = new WebSocket(presignedData.wsUrl);
```

Una vez conectado, el navegador envía mensajes directamente al agente y recibe respuestas en streaming en tiempo real:

```typescript
// Enviar un mensaje
ws.send(JSON.stringify({
  request: "¿Qué debería ponerme en Chicago hoy?",
  session_id: sessionId,
  user_id: userId
}));

// Recibir tokens en streaming
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.event?.data) {
    // Agregar el token a la UI inmediatamente
    appendToStream(data.event.data);
  }
};
```

Eso es todo. El navegador ahora está hablando directamente con el agente de IA. Cada token llega conforme el modelo lo genera.

## Por qué me gusta este enfoque

Veamos qué hace que esto sea mejor que el enfoque tradicional.

### Streaming en tiempo real
Cada token llega al navegador en el momento en que el modelo lo genera. No hay capa de búfer, ni overhead de invocación de Lambda por cada fragmento, ni API Gateway.

### Arquitectura más simple
Como mencioné antes, ya he implementado el enfoque tradicional con WebSocket de API Gateway y el enfoque con AppSync Events. Ambos funcionan, pero tienen muchas piezas móviles donde no se necesitan.

Con este enfoque lo único que necesitas es:
- Una Lambda que genera una URL prefirmada
- La API nativa de WebSocket del navegador
- El agente en sí

### Menor costo
Con este enfoque eliminas la necesidad de componentes extras que podrían generar costos. Solo pagas por el AgentCore Runtime y una invocación de Lambda por establecimiento de sesión.

### Conexiones persistentes
El WebSocket permanece abierto para conversaciones de múltiples turnos. El agente mantiene el estado a través de los mensajes dentro de la misma conexión, sin necesidad de recargar el contexto en cada solicitud.

## Agregando memoria

Una conexión directa por WebSocket es genial, pero ¿qué pasa cuando el usuario se va o abre la conversación en otro dispositivo? Sin memoria, el agente empieza de cero cada vez.

WearCast usa AgentCore Memory, un módulo proporcionado por AgentCore para mantener un historial de la conversación y darle al agente mejor conciencia del contexto.

```python
from bedrock_agentcore.memory.integrations.strands.config import AgentCoreMemoryConfig
from bedrock_agentcore.memory.integrations.strands.session_manager import AgentCoreMemorySessionManager

def create_session_manager(runtime_session_id, user_id):
    config = AgentCoreMemoryConfig(
        memory_id=AGENTCORE_MEMORY_ID,
        session_id=runtime_session_id,
        actor_id=user_id
    )
    return AgentCoreMemorySessionManager(
        agentcore_memory_config=config,
        region_name=AWS_REGION
    )
```
En este caso estamos usando la funcionalidad de session manager de Strands, que se encarga de cargar automáticamente los mensajes anteriores desde la memoria.
Los nuevos mensajes se agregan al contexto conforme van llegando. Con esto, el usuario puede cerrar su laptop, regresar horas después y continuar exactamente donde se quedó.

La memoria está delimitada por el ID de sesión y el ID de actor (usuario), por lo que las conversaciones de cada usuario están aisladas y son privadas.

### Infraestructura como código

El recurso de memoria se declara junto con el runtime en el template de SAM:

```yaml
AgentCoreShortTermMemory:
  Type: AWS::BedrockAgentCore::Memory
  Properties:
    Name: WearCast
    Description: Short-term memory for agent conversation persistence
    MemoryExecutionRoleArn: !GetAtt AgentCoreRole.Arn
    EventExpiryDuration: 30  # días
```

El ID de memoria se pasa al agente como una variable de entorno para que el session manager lo use.

## Agregando herramientas

Un agente sin herramientas es solo un chatbot. Las herramientas lo convierten en algo que realmente puede hacer cosas.

WearCast incluye una herramienta `get_weather` que obtiene datos reales de pronóstico de Open-Meteo (no requiere API key):

```python
@tool
def get_weather(city: str, date: str = "today") -> dict:
    """Get weather conditions for a city, current or up to 16 days ahead.
    
    Args:
        city: City name (e.g. "Indianapolis", "Chicago")
        date: "today" for current, or YYYY-MM-DD for forecast
    """
    # Geocodificar la ciudad
    geo_url = f"https://geocoding-api.open-meteo.com/v1/search?name={city}&count=1"
    # ... obtener coordenadas ...
    
    # Obtener datos del clima
    forecast_url = f"https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&..."
    # ... obtener y retornar datos del clima ...
```

Con Strands, las herramientas son simplemente funciones de Python con un decorador. El decorador `@tool` se encarga de:
- Generar el esquema de la herramienta para el LLM
- Parsear las solicitudes de uso de herramientas del LLM
- Ejecutar la función y devolver los resultados al modelo

En el frontend, el uso de herramientas se comunica a través del mismo stream de WebSocket:

```typescript
if (event.current_tool_use?.name) {
  setCurrentTool(event.current_tool_use.name);
  // Mostrar indicador "Usando get_weather..."
}
```

El usuario ve al agente "pensando", luego usando una herramienta, y después formulando su respuesta, todo en streaming en tiempo real. Se siente como ver a alguien trabajar, no como esperar una respuesta.

## El modelo de seguridad

Te podrías estar preguntando, ¿no es peligroso dejar que los navegadores se conecten directamente a tu agente de IA? No cuando aplicas la seguridad correctamente por capas:

1. El JWT de Cognito valida la identidad del usuario antes de que se genere cualquier URL prefirmada
2. El rol de IAM de la función Lambda determina qué recursos de AgentCore pueden ser accedidos, siguiendo el principio de mínimo privilegio
3. Las URLs prefirmadas expiran en 5 minutos, solo son válidas el tiempo suficiente para establecer la conexión. Si se pierde la conexión, se necesita solicitar otra URL.
4. El ID del usuario se incluye en la URL firmada como un header personalizado, para que el agente sepa con quién está hablando
5. La URL de conexión está firmada criptográficamente con SigV4, no puede ser alterada ni reutilizada

```javascript
// La identidad del usuario viaja con la URL firmada
queryParams['X-Amzn-Bedrock-AgentCore-Runtime-Custom-User-Id'] = userId;

// El agente lo recibe como un header
headers = context.request_headers
user_id = headers.get("x-amzn-bedrock-agentcore-runtime-custom-user-id")
```

El navegador nunca ve las credenciales de AWS. El rol de la Lambda hace la firma. La identidad del usuario está criptográficamente vinculada a la conexión.

## El flujo completo

Repasemos rápidamente el flujo completo de principio a fin:

1. El usuario inicia sesión → obtiene JWT de Cognito
2. El frontend llama a `POST /websocket/connect` con el JWT
3. Lambda valida el JWT, genera la URL de WebSocket prefirmada con SigV4
4. El frontend abre `new WebSocket(presignedUrl)`
5. La conexión llega directamente al AgentCore Runtime
6. El agente carga el historial de la conversación en el contexto desde AgentCore Memory
7. El usuario envía un mensaje por el WebSocket
8. El agente procesa, usa herramientas si es necesario, y transmite la respuesta token por token
9. El frontend renderiza cada token conforme llega
10. La conexión permanece abierta para mensajes de seguimiento

## Cuándo usar este patrón

Funciona bien cuando:
- El streaming en tiempo real importa (interfaces de chat, colaboración en vivo, agentes de voz)
- Las conversaciones de múltiples turnos son la norma (la conexión persistente evita handshakes repetidos)
- Quieres simplicidad (menos piezas móviles significa menos cosas que se rompen a las 2 AM)
- El costo es una preocupación (eliminar invocaciones de Lambda por cada mensaje se acumula rápido)

Puede no ser el enfoque correcto cuando:
- Necesitas transformación compleja de request/response antes de llegar al agente
- Tu agente necesita ser invocado por clientes que no son navegadores (procesamiento por lotes, tareas programadas)
- Necesitas funcionalidades de API Gateway como throttling, validación de requests, o planes de uso en cada mensaje


## Pruébalo tú mismo

Los componentes clave:
- `backend/functions/websocket-connect.js`: El generador de URL prefirmada (el único "intermediario")
- `backend/agents/agent/agent.py`: El agente con streaming por WebSocket, memoria y herramientas
- `frontend/src/services/websocket.ts`: El cliente WebSocket del lado del navegador
- `backend/template.yaml`: La definición completa de la infraestructura

## Conclusión

Estoy muy contento con cómo quedó esto. El enfoque de WebSocket eliminó mucha complejidad de la arquitectura y la experiencia de usuario es notablemente mejor con el streaming en tiempo real. Construí esto como proyecto personal pero ya he usado el mismo patrón en varios proyectos del trabajo. El hecho de que AgentCore maneje la gestión de la conexión WebSocket por nosotros significa que no tenemos que lidiar con ninguno de los típicos dolores de cabeza de infraestructura de WebSocket.

¡Déjame saber qué piensas de este enfoque!

Andres Moreno

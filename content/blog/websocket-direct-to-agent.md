+++
title = "Skip the Middleman: Connecting Your UI Directly to an AI Agent via WebSocket"
date = 2026-07-13T00:00:00-00:00
draft = false
description = "Learn how to connect your browser directly to an AI agent over WebSocket using Amazon Bedrock AgentCore, skipping the traditional Lambda proxy and enabling real-time token streaming."
tags = ["AWS", "Serverless", "Amazon Bedrock", "AgentCore", "WebSocket"]
[[images]]
  src = "img/websocket-direct-to-agent/title.png"
  alt = "Title image for Skip the Middleman: Connecting Your UI Directly to an AI Agent via WebSocket"
  stretch = "stretchH"
+++

I've been building AI agents for a while now, and streaming responses to a UI has always been the painful part. In previous projects I tried API Gateway streaming, Lambda response streaming, and even AppSync Events via an agent tool call to notify the UI. I also looked at adding my own WebSocket API through API Gateway, which requires managing `$connect`, `$disconnect`, and `$default` routes, storing connection IDs in DynamoDB, and posting messages back through `@connections`. All of these approaches felt like too much ceremony for what should be simple.

That's when I found that AgentCore has built-in WebSocket support. The browser can just connect directly to the agent. No middleman.

## The problem with the traditional approach

We usually build app applications as we do REST APIs
```
User → REST API → Lambda → AI Service → Lambda → REST API → User
```

Every message makes a round trip through multiple intermediaries. The response waits until the entire generation is complete, and then it all comes back at once. For a chat interface, this feels sluggish. Users stare at a spinner while the model generates hundreds of tokens they could already be reading.

Even if you add Server-Sent Events or long-polling, you're still stitching together a real-time experience on top of infrastructure that seemed like overkill. I wanted something better.

## The architecture

Here's what I ended up with:

```
┌─────────────┐          ┌──────────────────┐
│   React UI  │─── JWT ─▶│  API Gateway +   │──▶ Lambda (presigned URL)
│  (Cognito)  │          │  Cognito Auth    │
└──────┬──────┘          └──────────────────┘
       │
       │  WebSocket (SigV4 presigned URL)
       │  ← No middleman! Direct connection →
       ▼
┌──────────────────────────────────┐     ┌────────────────────┐
│  AgentCore Runtime               │────▶│  AgentCore Memory  │
│  (Strands Agent + Bedrock LLM)   │     │  (Persistence)     │
└──────────────────────────────────┘     └────────────────────┘
```

The browser connects directly to the AI agent over a WebSocket without the need for any Lambda functions to proxy the tokens. The agent streams tokens directly to the user's browser as they're generated.

The only additional infrastructure is during the initial handshake, where we exchange a JWT for a presigned WebSocket URL. After that, it's a direct, bidirectional pipe.

## How it works

Let's go over the three steps to get this working.

### Step 1: Authenticate the user

The user signs in through Amazon Cognito and receives a JWT token. Nothing unusual here.

```typescript
// Frontend authenticates and gets an access token
const accessToken = await authService.getAccessToken();
```

### Step 2: Exchange the JWT for a presigned WebSocket URL

Here's where it gets interesting. The frontend makes a single REST call to our backend:

```typescript
const presignedData = await apiService.getPresignedWebSocketUrl(sessionId, accessToken);
```

Behind the scenes, a Lambda function:
1. Validates the JWT (API Gateway's Cognito Authorizer handles this)
2. Uses its IAM role to generate an AWS SigV4 presigned WebSocket URL pointing directly at the AgentCore Runtime
3. Embeds the user's identity and session ID as query parameters in the signed URL
4. Returns the presigned URL to the browser

```javascript
// Lambda: Generate presigned WebSocket URL
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

The presigned URL is valid for 5 minutes, long enough to establish the connection, short enough to limit exposure. After the URL expires, it can reconnect by getting a new presigned URL, similar to how we handle expiring authentication tokens.

### Step 3: Connect directly to the agent

The browser opens a WebSocket connection using the presigned URL. No custom headers needed, all authentication is embedded in the URL's query parameters via SigV4.

```typescript
this.ws = new WebSocket(presignedData.wsUrl);
```

Once connected, the browser sends messages directly to the agent and receives streaming responses in real-time:

```typescript
// Send a message
ws.send(JSON.stringify({
  request: "What should I wear in Chicago today?",
  session_id: sessionId,
  user_id: userId
}));

// Receive streaming tokens
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.event?.data) {
    // Append token to the UI immediately
    appendToStream(data.event.data);
  }
};
```

That's it. The browser is now talking directly to the AI agent. Every token streams in as it's generated.

## Why I like this approach

Let me go over what makes this better than the traditional setup.

### Real-time streaming
Every token arrives at the browser the moment the model generates it. There's no buffering layer, no Lambda invocation overhead per chunk, no API Gateway.

### Simpler architecture
As I mentioned earlier, I've done the traditional API Gateway WebSocket approach and the AppSync Events approach. Both work, but they have a lot of moving parts where they are not needed.

With this approach all you need is:
- One Lambda that generates a presigned URL
- The browser's native WebSocket API
- The agent itself

### Lower cost
With this approach you remove the need for any extra components that might generate cost. You only pay for the AgentCore Runtime and one Lambda invocation per session establishment.

### Persistent connections
The WebSocket stays open for multi-turn conversations. The agent maintains state across messages within the same connection, without the need to reload context on every request.

## Adding memory

A direct WebSocket connection is great, but what happens when the user steps away or opens the conversation on a different device? Without memory, the agent starts fresh every time.

WearCast uses AgentCore Memory, a module provided by AgentCore to keep a history of the conversation to give the agent better context awareness

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
In this case we are using Strands session manager functionality which takes care of automatically loading previous messages from memory.
New messages are added to the context as they are happening. With this the user can close their laptop, come back hours later, and pick up right where they left off.

The memory is scoped by session ID and actor ID (user), so each user's conversations are isolated and private.

### Infrastructure-as-Code

The memory resource is declared right alongside the runtime in the SAM template:

```yaml
AgentCoreShortTermMemory:
  Type: AWS::BedrockAgentCore::Memory
  Properties:
    Name: WearCast
    Description: Short-term memory for agent conversation persistence
    MemoryExecutionRoleArn: !GetAtt AgentCoreRole.Arn
    EventExpiryDuration: 30  # days
```

The memory ID is passed to the agent as an environment variable for the session manager to use.

## Adding tools

An agent without tools is just a chatbot. Tools turn it into something that can actually do things.

WearCast includes a `get_weather` tool that fetches real forecast data from Open-Meteo (no API key required):

```python
@tool
def get_weather(city: str, date: str = "today") -> dict:
    """Get weather conditions for a city, current or up to 16 days ahead.
    
    Args:
        city: City name (e.g. "Indianapolis", "Chicago")
        date: "today" for current, or YYYY-MM-DD for forecast
    """
    # Geocode the city
    geo_url = f"https://geocoding-api.open-meteo.com/v1/search?name={city}&count=1"
    # ... fetch coordinates ...
    
    # Get weather data
    forecast_url = f"https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&..."
    # ... fetch and return weather data ...
```

With Strands, tools are just decorated Python functions. The `@tool` decorator handles:
- Generating the tool schema for the LLM
- Parsing the LLM's tool-use requests
- Executing the function and returning results to the model

On the frontend, tool usage is communicated through the same WebSocket stream:

```typescript
if (event.current_tool_use?.name) {
  setCurrentTool(event.current_tool_use.name);
  // Show "Using get_weather..." indicator
}
```

The user sees the agent "thinking," then using a tool, then formulating its response, all streaming in real-time. It feels like watching someone work, not waiting for an answer.

## The security model

You might be asking yourself, isn't it dangerous to let browsers connect directly to your AI agent? Not when you layer the security correctly:

1. Cognito JWT validates the user's identity before any presigned URL is generated
2. The Lambda functions IAM role determines what AgentCore resources can be accessed, following least-privilege
3. Presigned URLs expire in 5 minutes, only valid long enough to establish the connection. If connection is lost another URL needs to be requested.
4. The user's ID is embedded in the signed URL as a custom header, so the agent knows who it's talking to
5. The connection URL is cryptographically signed with SigV4, it can't be tampered with or reused

```javascript
// User identity travels with the signed URL
queryParams['X-Amzn-Bedrock-AgentCore-Runtime-Custom-User-Id'] = userId;

// The agent receives it as a header
headers = context.request_headers
user_id = headers.get("x-amzn-bedrock-agentcore-runtime-custom-user-id")
```

The browser never sees AWS credentials. The Lambda's role does the signing. The user's identity is cryptographically bound to the connection.

## The complete flow

Let me quickly go over the full flow from start to finish:

1. User logs in → gets Cognito JWT
2. Frontend calls `POST /websocket/connect` with JWT
3. Lambda validates JWT, generates SigV4 presigned WebSocket URL
4. Frontend opens `new WebSocket(presignedUrl)`
5. Connection lands directly on AgentCore Runtime
6. Agent loads conversation history into context from AgentCore Memory
7. User sends message over WebSocket
8. Agent processes, uses tools if needed, streams response token-by-token
9. Frontend renders each token as it arrives
10. Connection stays open for follow-up messages

## When to use this pattern

This works well when:
- Real-time streaming matters (chat interfaces, live collaboration, voice agents)
- Multi-turn conversations are the norm (the persistent connection avoids repeated handshakes)
- You want simplicity (fewer moving parts means fewer things that break at 2 AM)
- Cost is a concern (eliminating per-message Lambda invocations adds up fast)

It may not be the right fit when:
- You need complex request/response transformation before reaching the agent
- Your agent needs to be invoked by non-browser clients (batch processing, scheduled tasks)
- You need API Gateway features like throttling, request validation, or usage plans on every message


## Try it yourself

WearCast is public and deployable with a single `sam deploy`. The entire infrastructure (Cognito, API Gateway, Lambda, AgentCore Runtime, AgentCore Memory) is defined in one SAM template.

The key components:
- `backend/functions/websocket-connect.js`: The presigned URL generator (the only "middleman")
- `backend/agents/agent/agent.py`: The agent with WebSocket streaming, memory, and tools
- `frontend/src/services/websocket.ts`: The browser-side WebSocket client
- `backend/template.yaml`: The complete infrastructure definition

## Wrap up

I'm really happy with how this turned out. The WebSocket approach removed so much complexity from the architecture and the user experience is noticeably better with real-time streaming. I built this on the side but I've already used the same pattern for several projects at work. The fact that AgentCore handles the WebSocket connection management for us means we don't have to deal with any of the typical WebSocket infrastructure headaches.

If you want to try it out yourself, I have everything defined and deployable in [this GitHub repo](https://github.com/andmoredev/wearcast/).

Let me know what you think about this approach!


Andres Moreno

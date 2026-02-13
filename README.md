# 🌤️ Agentic Weather Search PoC

An AI-powered weather query system demonstrating **LangChain4j** integration with **Spring Boot**. Users ask natural language questions and receive human-friendly weather responses through an intelligent agent that uses tool-calling capabilities.

---

## 📋 Table of Contents

- [Architecture Overview](#-architecture-overview)
- [Tech Stack](#-tech-stack)
- [Backend Deep Dive](#-backend-deep-dive)
  - [Component Architecture](#component-architecture)
  - [Request Flow](#request-flow)
  - [LangChain4j Agent Flow](#langchain4j-agent-flow)
  - [Memory Management](#memory-management)
- [Project Structure](#-project-structure)
- [Key Components](#-key-components)
- [API Reference](#-api-reference)
- [Setup & Running](#-setup--running)
- [Sample Queries](#-sample-queries)

---

## 🏗️ Architecture Overview

```mermaid
flowchart TB
    subgraph Frontend["🖥️ Frontend (React + Vite)"]
        UI[WeatherSearch Component]
        Hook[useWeatherQuery Hook]
        API[weatherApi Service]
    end

    subgraph Backend["☕ Backend (Spring Boot)"]
        Controller[QueryController]
        Agent[WeatherAgent]
        Tools[WeatherAgentTools]
        Memory[ChatMemoryProvider]

        subgraph Services["📦 Weather Services"]
            Geocoding[GeocodingService]
            Weather[OpenMeteoWeatherService]
        end
    end

    subgraph External["🌐 External APIs"]
        GeoAPI[Open-Meteo Geocoding API]
        WeatherAPI[Open-Meteo Weather API]
        OpenAI[OpenAI GPT-4o]
    end

    UI --> Hook
    Hook --> API
    API -->|"POST /api/query"| Controller
    Controller -->|"chat()"| Agent
    Agent <-->|"LLM Calls"| OpenAI
    Agent -->|"Tool calls"| Tools
    Agent <-->|"Session Memory"| Memory
    Tools --> Geocoding
    Tools --> Weather
    Geocoding --> GeoAPI
    Weather --> WeatherAPI
```

---

## 🛠️ Tech Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| 🖥️ **Frontend** | React 18 + TypeScript + Vite | Modern SPA with hot reload |
| 🎨 **Styling** | Tailwind CSS | Utility-first CSS framework |
| ☕ **Backend** | Spring Boot 3.3 | REST API and dependency injection |
| 🤖 **AI Framework** | LangChain4j 1.0.0 | AI service abstraction with tools |
| 🧠 **LLM** | OpenAI GPT-4o | Natural language understanding |
| 🌤️ **Weather Data** | Open-Meteo API | Free weather and geocoding |

---

## 🔍 Backend Deep Dive

### Component Architecture

```mermaid
classDiagram
    class QueryController {
        -WeatherAgent weatherAgent
        +processQuery(QueryRequest, sessionId) ResponseEntity
        +health() ResponseEntity
    }

    class WeatherAgent {
        <<interface>>
        +chat(sessionId, userMessage) String
    }

    class WeatherAgentTools {
        -GeocodingService geocodingService
        -WeatherService weatherService
        +getCoordinates(cityName, countryCode) GeocodingResult
        +getCurrentWeather(lat, lon, unit) WeatherData
        +getWeatherForecast(lat, lon, days, unit) WeatherData
    }

    class LangChain4jConfig {
        -Map memories
        +chatLanguageModel() ChatLanguageModel
        +chatMemoryProvider() ChatMemoryProvider
        +weatherAgent() WeatherAgent
    }

    class GeocodingService {
        -RestClient restClient
        +geocode(cityName, countryCode) GeocodingResult
    }

    class OpenMeteoWeatherService {
        -RestClient restClient
        +getCurrentWeather(lat, lon, unit) WeatherData
        +getForecast(lat, lon, days, unit) WeatherData
    }

    QueryController --> WeatherAgent : uses
    LangChain4jConfig --> WeatherAgent : creates
    LangChain4jConfig --> WeatherAgentTools : injects
    WeatherAgentTools --> GeocodingService : uses
    WeatherAgentTools --> OpenMeteoWeatherService : uses
```

---

### Request Flow

This diagram shows the complete request lifecycle from user input to response:

```mermaid
sequenceDiagram
    autonumber
    participant User as 👤 User
    participant FE as 🖥️ React Frontend
    participant API as ☕ QueryController
    participant Agent as 🤖 WeatherAgent
    participant LLM as 🧠 GPT-4o
    participant Tools as 🔧 WeatherAgentTools
    participant Geo as 🗺️ GeocodingService
    participant Weather as 🌤️ WeatherService
    participant ExtAPI as 🌐 Open-Meteo

    User->>FE: "What's the weather in Paris?"
    FE->>FE: Add to chat history
    FE->>API: POST /api/query<br/>X-Session-Id: abc-123

    API->>Agent: chat("abc-123", "What's the weather in Paris?")

    Note over Agent,LLM: LangChain4j handles LLM orchestration

    Agent->>LLM: System prompt + User message + Available tools
    LLM-->>Agent: Tool call: getCoordinates("Paris", null)

    Agent->>Tools: getCoordinates("Paris", null)
    Tools->>Geo: geocode("Paris", null)
    Geo->>ExtAPI: GET /search?name=Paris
    ExtAPI-->>Geo: {lat: 48.85, lon: 2.35, ...}
    Geo-->>Tools: GeocodingResult
    Tools-->>Agent: {name: "Paris", lat: 48.85, lon: 2.35}

    Agent->>LLM: Tool result + Continue
    LLM-->>Agent: Tool call: getCurrentWeather(48.85, 2.35, "celsius")

    Agent->>Tools: getCurrentWeather(48.85, 2.35, "celsius")
    Tools->>Weather: getCurrentWeather(48.85, 2.35, "celsius")
    Weather->>ExtAPI: GET /forecast?latitude=48.85&longitude=2.35&current=...
    ExtAPI-->>Weather: {temperature: 18, humidity: 65, ...}
    Weather-->>Tools: WeatherData
    Tools-->>Agent: {temp: 18°C, humidity: 65%, ...}

    Agent->>LLM: Tool result + Generate final response
    LLM-->>Agent: "Currently in Paris, France, it's 18°C with partly cloudy skies..."

    Agent-->>API: Natural language response
    API-->>FE: {answer: "Currently in Paris..."}
    FE->>FE: Update chat history
    FE-->>User: Display response
```

---

### LangChain4j Agent Flow

The agent uses a **ReAct (Reason + Act)** pattern to process queries:

```mermaid
flowchart TD
    subgraph Input["📥 Input Processing"]
        Query[User Query]
        Memory[Load Chat Memory]
        System[System Prompt]
    end

    subgraph Agent["🤖 WeatherAgent (LangChain4j AI Service)"]
        Parse[Parse & Understand Query]
        Decide{Need Tools?}

        subgraph ToolCalls["🔧 Tool Execution Loop"]
            SelectTool[Select Appropriate Tool]
            Execute[Execute Tool]
            ProcessResult[Process Result]
            MoreTools{More Tools Needed?}
        end

        Generate[Generate Natural Response]
    end

    subgraph Output["📤 Output"]
        Response[Return Response]
        SaveMemory[Save to Chat Memory]
    end

    Query --> Memory
    Memory --> System
    System --> Parse
    Parse --> Decide

    Decide -->|Yes| SelectTool
    SelectTool --> Execute
    Execute --> ProcessResult
    ProcessResult --> MoreTools
    MoreTools -->|Yes| SelectTool
    MoreTools -->|No| Generate

    Decide -->|No| Generate
    Generate --> Response
    Response --> SaveMemory

    style Agent fill:#e1f5fe
    style ToolCalls fill:#fff3e0
```

#### Available Tools

| Tool | Description | Parameters | Returns |
|------|-------------|------------|---------|
| 🗺️ `getCoordinates` | Resolves city name to lat/lon | `cityName`, `countryCode?` | `GeocodingResult` |
| 🌡️ `getCurrentWeather` | Gets current conditions | `lat`, `lon`, `unit?` | `WeatherData` |
| 📅 `getWeatherForecast` | Gets multi-day forecast | `lat`, `lon`, `days`, `unit?` | `WeatherData` |

---

### Memory Management

The system maintains **per-session conversation memory** using LangChain4j's `MessageWindowChatMemory`:

```mermaid
flowchart LR
    subgraph Frontend["🖥️ Frontend"]
        Session[sessionStorage<br/>weather-session-id]
    end

    subgraph Backend["☕ Backend"]
        subgraph Config["LangChain4jConfig"]
            MemoryMap[ConcurrentHashMap<br/>sessionId → ChatMemory]
            Provider[ChatMemoryProvider]
        end

        subgraph Memory["MessageWindowChatMemory"]
            Window[Rolling Window<br/>Last 20 Messages]
        end
    end

    Session -->|X-Session-Id Header| Provider
    Provider -->|computeIfAbsent| MemoryMap
    MemoryMap --> Memory

    style Memory fill:#e8f5e9
```

#### Memory Flow Detail

```mermaid
sequenceDiagram
    participant FE as 🖥️ Frontend
    participant Ctrl as QueryController
    participant Agent as WeatherAgent
    participant Provider as ChatMemoryProvider
    participant Map as ConcurrentHashMap
    participant Memory as MessageWindowChatMemory

    FE->>Ctrl: Request with X-Session-Id: "user-123"
    Ctrl->>Agent: chat("user-123", "Weather in Paris?")

    Note over Agent,Provider: @MemoryId triggers memory lookup

    Agent->>Provider: getMemory("user-123")
    Provider->>Map: computeIfAbsent("user-123", ...)

    alt New Session
        Map->>Memory: Create new (maxMessages=20)
        Map-->>Provider: New ChatMemory
    else Existing Session
        Map-->>Provider: Existing ChatMemory
    end

    Provider-->>Agent: ChatMemory for session

    Note over Agent,Memory: Agent uses memory for context

    Agent->>Memory: Add user message
    Agent->>Agent: Process with LLM
    Agent->>Memory: Add AI response
```

---

## 📁 Project Structure

```
full-stack-poc/
├── 📂 backend/                           # Spring Boot Application
│   └── src/main/java/com/weatheragent/
│       │
│       ├── 🚀 WeatherAgentApplication.java    # Entry point
│       │
│       ├── 📂 controller/
│       │   └── QueryController.java           # REST endpoint handler
│       │
│       ├── 📂 service/
│       │   ├── 📂 agent/
│       │   │   ├── WeatherAgent.java          # AI Service interface
│       │   │   └── WeatherAgentTools.java     # @Tool annotated methods
│       │   │
│       │   └── 📂 weather/
│       │       ├── WeatherService.java        # Interface
│       │       ├── OpenMeteoWeatherService.java
│       │       └── GeocodingService.java
│       │
│       ├── 📂 model/
│       │   ├── 📂 dto/
│       │   │   ├── QueryRequest.java
│       │   │   └── QueryResponse.java
│       │   │
│       │   └── 📂 weather/
│       │       ├── WeatherData.java
│       │       └── GeocodingResult.java
│       │
│       ├── 📂 config/
│       │   ├── LangChain4jConfig.java         # AI wiring
│       │   └── WebConfig.java                 # CORS
│       │
│       └── 📂 exception/
│           ├── WeatherApiException.java
│           └── GlobalExceptionHandler.java
│
└── 📂 frontend/                          # React Application
    └── src/
        ├── 📂 components/
        │   ├── WeatherSearch.tsx              # Main container
        │   ├── QueryInput.tsx
        │   ├── SubmitButton.tsx
        │   └── ResponseDisplay.tsx
        │
        ├── 📂 services/
        │   └── weatherApi.ts                  # API client
        │
        ├── 📂 hooks/
        │   └── useWeatherQuery.ts             # State management
        │
        └── 📂 types/
            └── index.ts
```

---

## 🔑 Key Components

### 1. QueryController
**📍 Location:** `controller/QueryController.java`

Handles HTTP requests and delegates to the AI agent.

```java
@PostMapping("/api/query")
public ResponseEntity<QueryResponse> processQuery(
        @RequestBody QueryRequest request,
        @RequestHeader("X-Session-Id") String sessionId) {

    String answer = weatherAgent.chat(sessionId, request.query());
    return ResponseEntity.ok(new QueryResponse(answer));
}
```

**Key Points:**
- Extracts session ID from header for memory isolation
- Generates UUID if no session ID provided
- Simple delegation to AI agent

---

### 2. WeatherAgent
**📍 Location:** `service/agent/WeatherAgent.java`

LangChain4j AI Service interface with system prompt.

```java
@SystemMessage("""
    You are a helpful weather assistant with memory of our conversation.
    1. Understand natural language weather queries
    2. Extract location, time frame, and data type
    3. Use tools to get weather data
    4. Provide friendly, conversational responses
    5. Remember previous queries in the conversation
    """)
String chat(@MemoryId String sessionId, @UserMessage String userMessage);
```

**Key Points:**
- `@SystemMessage` defines the agent's behavior
- `@MemoryId` enables per-session memory
- `@UserMessage` marks the user input

---

### 3. WeatherAgentTools
**📍 Location:** `service/agent/WeatherAgentTools.java`

Defines tools the LLM can call:

```java
@Tool("Resolves a city name to geographic coordinates")
public GeocodingResult getCoordinates(
    @P("City name to look up") String cityName,
    @P(value = "Optional country code", required = false) String countryCode) {
    return geocodingService.geocode(cityName, countryCode);
}
```

**Key Points:**
- `@Tool` annotation makes method available to LLM
- `@P` provides parameter descriptions for LLM
- Returns rich objects that LLM interprets

---

### 4. LangChain4jConfig
**📍 Location:** `config/LangChain4jConfig.java`

Wires together the AI components:

```mermaid
flowchart LR
    subgraph Beans["Spring Beans"]
        Model["ChatLanguageModel<br/>OpenAI GPT-4o"]
        Memory["ChatMemoryProvider<br/>Session storage"]
        Tools["WeatherAgentTools<br/>Tool methods"]
    end

    subgraph Builder["AiServices.builder"]
        Agent["WeatherAgent<br/>AI Service Proxy"]
    end

    Model --> Builder
    Memory --> Builder
    Tools --> Builder
    Builder --> Agent
```

---

### 5. Weather Services

#### GeocodingService
**📍 Location:** `service/weather/GeocodingService.java`

```mermaid
flowchart LR
    Input["cityName: 'Paris'"] --> Service[GeocodingService]
    Service --> API["Open-Meteo Geocoding API"]
    API --> Result["GeocodingResult<br/>{name, country, lat, lon, timezone}"]
```

#### OpenMeteoWeatherService
**📍 Location:** `service/weather/OpenMeteoWeatherService.java`

```mermaid
flowchart LR
    Input["lat, lon, unit"] --> Service[OpenMeteoWeatherService]
    Service --> API["Open-Meteo Forecast API"]
    API --> Result["WeatherData<br/>{temperature, humidity, wind, conditions, forecast}"]
```

---

## 📡 API Reference

### POST /api/query

Send a natural language weather query.

**Headers:**
| Header | Required | Description |
|--------|----------|-------------|
| `Content-Type` | Yes | `application/json` |
| `X-Session-Id` | No | Session ID for conversation memory |

**Request Body:**
```json
{
  "query": "What's the weather like in Tokyo tomorrow?"
}
```

**Response:**
```json
{
  "answer": "Tomorrow in Tokyo, Japan, expect a high of 22°C and a low of 15°C with clear skies. Perfect weather for outdoor activities!"
}
```

**Example cURL:**
```bash
curl -X POST http://localhost:8080/api/query \
  -H "Content-Type: application/json" \
  -H "X-Session-Id: my-session-123" \
  -d '{"query": "Weather in Paris?"}'
```

---

### GET /api/health

Check backend health status.

**Response:**
```
Weather Agent Backend is running
```

---

## 🚀 Setup & Running

### Prerequisites

- ☕ Java 17+
- 📦 Node.js 18+
- 🔧 Maven 3.8+
- 🔑 OpenAI API Key

### Configuration

1. Copy the example configuration:
   ```bash
   cp backend/src/main/resources/application.yml.example \
      backend/src/main/resources/application.yml
   ```

2. Edit `application.yml` and add your OpenAI API key:
   ```yaml
   openai:
     api-key: sk-your-actual-api-key-here
     model: gpt-4o
   ```

### Start Backend

```bash
cd backend
mvn spring-boot:run
```
Backend runs at: `http://localhost:8080`

### Start Frontend

```bash
cd frontend
npm install
npm run dev
```
Frontend runs at: `http://localhost:5173`

---

## 💬 Sample Queries

| Query | Agent Action |
|-------|--------------|
| "What's the weather in Paris?" | `getCoordinates` → `getCurrentWeather` |
| "Will it rain in London tomorrow?" | `getCoordinates` → `getWeatherForecast(days=1)` |
| "What about next week?" | Uses memory to get last location → `getWeatherForecast(days=7)` |
| "Temperature in New York right now" | `getCoordinates` → `getCurrentWeather` |
| "Tokyo forecast in Fahrenheit" | `getCoordinates` → `getWeatherForecast(unit=fahrenheit)` |

---

## 🧪 Error Handling Flow

```mermaid
flowchart TD
    Request[Incoming Request] --> Validate{Valid JSON?}

    Validate -->|No| BadRequest[400 Bad Request]
    Validate -->|Yes| Process[Process Query]

    Process --> GeoError{Geocoding Failed?}
    GeoError -->|Yes| LocationError["Location not found" Error]
    GeoError -->|No| WeatherError{Weather API Failed?}

    WeatherError -->|Yes| APIError["Weather service unavailable" Error]
    WeatherError -->|No| LLMError{LLM Error?}

    LLMError -->|Yes| AgentError["Unable to process query" Error]
    LLMError -->|No| Success[200 OK + Answer]

    LocationError --> Handler[GlobalExceptionHandler]
    APIError --> Handler
    AgentError --> Handler
    Handler --> ErrorResponse["500 Internal Server Error<br/>{answer: error message}"]
```

---

## 📜 License

MIT License - Feel free to use and modify!

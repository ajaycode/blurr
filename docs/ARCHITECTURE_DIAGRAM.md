# Blurr Agent Architecture Diagram

## High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                Android Device                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────┐                                                        │
│  │   Blurr/Panda App   │                                                        │
│  │                     │                                                        │
│  │  ┌───────────────┐  │    ┌─────────────────────────────────────────────────┐ │
│  │  │    Agent      │  │    │         Android System Services                │ │
│  │  │   (Main Loop) │◄─┼────┤                                                 │ │
│  │  └───────────────┘  │    │  ┌─────────────────────────────────────────┐    │ │
│  │         │            │    │  │     Accessibility Service              │    │ │
│  │         ▼            │    │  │  - Screen reading (XML hierarchy)      │    │ │
│  │  ┌─────────────┐     │    │  │  - Touch injection                     │    │ │
│  │  │ Perception  │◄────┼────┤  │  - UI interaction                      │    │ │
│  │  │   System    │     │    │  └─────────────────────────────────────────┘    │ │
│  │  └─────────────┘     │    │                                                 │ │
│  │         │            │    │  ┌─────────────────────────────────────────┐    │ │
│  │         ▼            │    │  │         Package Manager                │    │ │
│  │  ┌─────────────┐     │    │  │  - App launching                       │    │ │
│  │  │   Memory    │     │    │  │  - Package discovery                   │    │ │
│  │  │  Manager    │     │    │  └─────────────────────────────────────────┘    │ │
│  │  └─────────────┘     │    └─────────────────────────────────────────────────┘ │
│  │         │            │                                                        │
│  │         ▼            │                                                        │
│  │  ┌─────────────┐     │    ┌─────────────────────────────────────────────────┐ │
│  │  │    LLM      │◄────┼────┤              External Services                 │ │
│  │  │    API      │     │    │                                                 │ │
│  │  └─────────────┘     │    │  ┌─────────────────────────────────────────┐    │ │
│  │         │            │    │  │           Gemini API                   │    │ │
│  │         ▼            │    │  │  - Natural language processing         │    │ │
│  │  ┌─────────────┐     │    │  │  - Structured JSON response            │    │ │
│  │  │   Action    │     │    │  │  - Action planning                     │    │ │
│  │  │  Executor   │     │    │  └─────────────────────────────────────────┘    │ │
│  │  └─────────────┘     │    │                                                 │ │
│  │         │            │    │  ┌─────────────────────────────────────────┐    │ │
│  │         ▼            │    │  │        Google TTS API                   │    │ │
│  │  ┌─────────────┐     │    │  │  - Text-to-speech conversion           │    │ │
│  │  │   Physical  │     │    │  │  - Voice feedback                      │    │ │
│  │  │ Interaction │     │    │  └─────────────────────────────────────────┘    │ │
│  │  │  (Finger)   │     │    └─────────────────────────────────────────────────┘ │
│  │  └─────────────┘     │                                                        │
│  └─────────────────────┘                                                        │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Component Interaction Flow

### 1. SENSE Phase Flow
```
User Input → Agent.run()
     │
     ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│    Eyes     │───→│ SemanticParser │─→│ Perception  │
│             │    │             │   │             │
│ Screenshot  │    │ XML→String  │   │ Analysis    │
│ XML Dump    │    │ Element Map │   │ Synthesis   │
│ Activity    │    │ Filtering   │   │             │
│ Keyboard    │    │             │   │             │
└─────────────┘    └─────────────┘   └─────────────┘
     │                                       │
     │              Accessibility            │
     │              Service API              │
     ▼                                       ▼
┌─────────────────────────────────┐    ┌─────────────┐
│       Android System            │    │ ScreenAnalysis │
│   - Window hierarchy            │    │  Object        │
│   - Touch coordinates           │    └─────────────┘
│   - App information             │
└─────────────────────────────────┘
```

### 2. THINK Phase Flow
```
ScreenAnalysis → MemoryManager → LLM API → AgentOutput
     │               │             │           │
     │        ┌─────────────┐ ┌─────────────┐  │
     │        │  Context    │ │   Gemini    │  │
     └───────→│  Building   │→│    API      │  │
              │             │ │             │  │
              │ - History   │ │ - JSON      │  │
              │ - Screen    │ │ - Parsing   │  │
              │ - Task      │ │ - Actions   │  │
              └─────────────┘ └─────────────┘  │
                                               │
                                               ▼
                                        ┌─────────────┐
                                        │  Structured │
                                        │   Response  │
                                        │ {           │
                                        │  thinking,  │
                                        │  goal,      │
                                        │  actions[]  │
                                        │ }           │
                                        └─────────────┘
```

### 3. ACT Phase Flow
```
AgentOutput → ActionExecutor → Finger → Android System
     │             │            │            │
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Actions[]  │ │ Type-safe   │ │ Physical    │ │ Accessibility│
│             │ │ Execution   │ │ Gestures    │ │ Service     │
│ TapElement  │→│             │→│             │→│             │
│ InputText   │ │ Validation  │ │ Touch       │ │ UI Actions  │
│ ScrollDown  │ │ Coordinate  │ │ Swipe       │ │ Text Input  │
│ OpenApp     │ │ Mapping     │ │ Type        │ │ Navigation  │
│ ...         │ │             │ │             │ │             │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
```

## Key Design Patterns

### 1. SENSE-THINK-ACT Loop
- **Separation of Concerns**: Each phase has distinct responsibilities
- **Async Processing**: Coroutines handle concurrent operations
- **State Management**: Persistent state across iterations

### 2. Type-Safe Action System
```kotlin
sealed class Action {
    data class TapElement(val elementId: Int) : Action()
    data class InputText(val text: String) : Action()
    // ... other actions
}
```
- **Compile-time Safety**: Sealed classes prevent invalid actions
- **Exhaustive Matching**: when expressions ensure all actions handled
- **Custom Serialization**: JSON ↔ Kotlin object mapping

### 3. Element Targeting System
```
XML Hierarchy → Semantic Filtering → Interactive Elements → Index Mapping
                                           │
                                           ▼
                                   [1] Button "Submit"
                                   [2] TextField "Search"
                                   [3] Button "Cancel"
```

### 4. Error Recovery Pattern
```
Action Execution → Result Check → Success/Failure → State Update
                                      │
                                      ▼
                              Failure Count → Retry Logic
                                      │
                                      ▼
                              Max Failures → Stop Agent
```

## Security Architecture

### Permission Model
- **Accessibility Service**: Core functionality requires user grant
- **No Root Required**: All operations through Android APIs
- **Scoped File Access**: Agent files isolated in app directory
- **Network**: Only for LLM API calls

### Data Flow Security
- **Local Processing**: Screen analysis happens on-device
- **API Communication**: HTTPS for LLM interactions
- **No Persistent Secrets**: API keys from build configuration
- **User Control**: Agent actions are transparent and logged

## Performance Characteristics

### Latency Factors
1. **Screen Capture**: ~100-200ms (Accessibility Service)
2. **XML Processing**: ~50-100ms (Semantic parsing)
3. **LLM API Call**: ~1-3 seconds (Network + processing)
4. **Action Execution**: ~100-500ms (Touch gestures)

### Optimization Strategies
- **Concurrent Processing**: Parallel screen capture and analysis
- **Semantic Filtering**: Reduce XML complexity before LLM
- **Element Caching**: Reuse coordinate mappings when possible
- **Connection Pooling**: Efficient HTTP client for API calls

This architecture provides a robust foundation for an autonomous Android agent while maintaining security, performance, and maintainability.
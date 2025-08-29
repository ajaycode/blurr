# Blurr/Panda Codebase Overview

## Introduction

**Blurr** (also known as **Panda**) is an Android AI agent that can autonomously understand natural language commands and operate a phone's UI to achieve them. The agent acts as a personal phone operator, handling complex, multi-step tasks across different applications.

## Architecture Overview

The system follows a sophisticated **SENSE → THINK → ACT** loop pattern, implemented as a multi-agent system written entirely in Kotlin.

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent Execution Loop                     │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐  │
│  │  SENSE  │ -> │  THINK  │ -> │   ACT   │ -> │ RECORD  │  │
│  │ (Eyes)  │    │  (LLM)  │    │(Finger) │    │(Memory) │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Agent (`/v2/Agent.kt`)
- **Role**: Main conductor of the entire system
- **Responsibilities**: 
  - Orchestrates the SENSE → THINK → ACT loop
  - Manages agent state and execution history
  - Handles task completion and error recovery
- **Key Methods**:
  - `run(initialTask: String, maxSteps: Int)`: Main execution loop

### 2. Perception System (`/v2/perception/`)

#### Eyes (`/api/Eyes.kt`)
- **Role**: The agent's "vision" system
- **Capabilities**:
  - Takes screenshots using Android Accessibility Service
  - Captures UI hierarchy as XML
  - Detects keyboard status
  - Gets current activity information

#### SemanticParser (`/v2/perception/SemanticParser.kt`)
- **Role**: Parses and filters Android UI hierarchy XML
- **Key Features**:
  - Filters XML to keep only semantically important nodes
  - Generates custom string representation focusing on interactive elements
  - Maps interactive elements to numeric indices for action targeting
  - Handles screen visibility and bounds checking

#### Perception (`/v2/perception/Perception.kt`)
- **Role**: Combines Eyes and SemanticParser for complete screen analysis
- **Output**: `ScreenAnalysis` object containing:
  - Clean, LLM-friendly UI representation
  - Interactive element mapping
  - Keyboard status
  - Scroll indicators

### 3. Action System (`/v2/actions/`)

#### Action (`/v2/actions/Action.kt`)
- **Role**: Defines all possible commands the agent can execute
- **Architecture**: Sealed class with type-safe action definitions
- **Available Actions**:
  - `TapElement(elementId: Int)`: Tap UI elements by ID
  - `InputText(text: String)`: Type text into focused fields
  - `ScrollUp/ScrollDown(amount: Int)`: Scroll gestures
  - `OpenApp(appName: String)`: Launch applications
  - `Speak(message: String)`: Text-to-speech output
  - `Ask(question: String)`: Request user input
  - `Done(success: Boolean, text: String)`: Task completion
  - File operations: `WriteFile`, `ReadFile`, `AppendFile`
  - Navigation: `Back`, `Home`, `SwitchApp`, `Wait`

#### ActionExecutor (`/v2/actions/ActionExecutor.kt`)
- **Role**: Executes validated Action commands
- **Integration**: Uses `Finger` class for physical interactions
- **Features**: Type-safe execution with comprehensive error handling

### 4. Physical Interaction (`/api/`)

#### Finger (`/api/Finger.kt`)
- **Role**: The agent's "hands" for physical device interaction
- **Capabilities**:
  - Touch gestures (tap, swipe, scroll)
  - Text input via Android Accessibility Service
  - App launching via package manager
  - Navigation actions (back, home)

### 5. LLM Integration (`/v2/llm/`)

#### GeminiAPI (`/v2/llm/GeminiAPI.kt`)
- **Role**: Interface with Google's Gemini LLM
- **Features**:
  - Structured JSON output generation
  - Multiple API key management
  - Error handling and retries
  - Token usage tracking

### 6. Memory Management (`/v2/message_manager/`)

#### MemoryManager (`/v2/message_manager/MessageManager.kt`)
- **Role**: Manages agent's short-term memory and prompt building
- **Capabilities**:
  - Builds conversation context for LLM
  - Maintains task state and history
  - Formats screen analysis for LLM consumption

### 7. File System (`/v2/fs/`)

#### FileSystem (`/v2/fs/FileSystem.kt`)
- **Role**: Handles persistent file storage for the agent
- **Features**: Provides file operations that actions can utilize

## Data Models (`/v2/AgentModel.kt`)

### Core Data Structures

```kotlin
// Agent configuration
data class AgentSettings(
    val maxFailures: Int = 3,
    val maxActionsPerStep: Int = 10,
    val useThinking: Boolean = true,
    // ... other settings
)

// Agent runtime state
data class AgentState(
    var nSteps: Int = 1,
    var consecutiveFailures: Int = 0,
    var lastResult: List<ActionResult>? = null,
    var stopped: Boolean = false,
    // ... other state
)

// LLM output structure
data class AgentOutput(
    val thinking: String? = null,
    val evaluationPreviousGoal: String? = null,
    val memory: String? = null,
    val nextGoal: String? = null,
    val action: List<Action>
)

// Action execution result
data class ActionResult(
    val isDone: Boolean? = false,
    val success: Boolean? = null,
    val error: String? = null,
    val longTermMemory: String? = null
)
```

## Execution Flow

### 1. SENSE Phase
```kotlin
val screenState = perception.analyze()
```
- Uses `Eyes` to capture screen state (screenshot + XML)
- `SemanticParser` processes XML into LLM-friendly format
- Creates `ScreenAnalysis` with interactive element mapping

### 2. THINK Phase
```kotlin
val agentOutput = llmApi.generateAgentOutput(messages)
```
- `MemoryManager` builds conversation context
- Sends formatted prompt to Gemini LLM
- Receives structured JSON response with planned actions

### 3. ACT Phase
```kotlin
val result = actionExecutor.execute(action, screenState, context, fileSystem)
```
- `ActionExecutor` validates and executes each action
- `Finger` performs physical device interactions
- Results are captured for next iteration

### 4. RECORD Phase
```kotlin
history.addItem(AgentHistory(...))
```
- Complete step data saved to history
- State updated for next iteration

## Key Features

### Accessibility Service Integration
- Uses Android Accessibility Service for screen reading and interaction
- No root access required
- Works across all applications

### Type-Safe Action System
- Sealed classes ensure compile-time safety
- Custom serialization handles LLM JSON output
- Exhaustive pattern matching prevents missing actions

### Intelligent UI Parsing
- Semantic filtering removes UI noise
- Interactive element prioritization
- Coordinate mapping for precise targeting

### Robust Error Handling
- LLM failure recovery
- Action execution error handling
- Consecutive failure limits with automatic stopping

### Memory and Context Management
- Maintains conversation history
- Task-specific memory updates
- Screen state progression tracking

## Development Guidelines

### Adding New Actions
1. Define action in `Action.kt` sealed class
2. Add specification to `allSpecs` map
3. Implement execution logic in `ActionExecutor.kt`
4. Test with various screen states

### Extending Perception
1. Modify `SemanticParser` for new UI element types
2. Update `ScreenAnalysis` data structure if needed
3. Ensure LLM prompt compatibility

### LLM Integration
1. Maintain structured output format compatibility
2. Handle new response fields in `AgentOutput`
3. Update prompt templates as needed

## Technology Stack

- **Language**: Kotlin
- **Platform**: Android (API 24+)
- **LLM**: Google Gemini API
- **UI Analysis**: Android Accessibility Service
- **Architecture**: Coroutines-based async processing
- **Serialization**: kotlinx.serialization

## File Structure Summary

```
app/src/main/java/com/blurr/voice/
├── v2/                          # Main agent system (v2)
│   ├── Agent.kt                 # Main conductor
│   ├── AgentModel.kt           # Data structures
│   ├── actions/                # Action system
│   ├── perception/             # Screen analysis
│   ├── llm/                    # LLM integration
│   ├── message_manager/        # Memory management
│   └── fs/                     # File system
├── api/                        # Low-level device interaction
│   ├── Eyes.kt                 # Screen capture
│   ├── Finger.kt               # Touch interaction
│   └── ...
├── services/                   # Android services
├── utilities/                  # Helper utilities
└── MainActivity.kt             # App entry point
```

This architecture creates a flexible, maintainable system where each component has a clear responsibility while working together to create an intelligent, autonomous Android agent.
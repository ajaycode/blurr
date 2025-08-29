# Understanding the Blurr/Panda Codebase - Complete Guide

Welcome to the comprehensive guide for understanding the Blurr (also known as Panda) codebase. This document serves as your entry point to understanding this sophisticated Android AI agent system.

## What is Blurr/Panda?

Blurr/Panda is an **autonomous Android AI agent** that can:
- Understand natural language commands
- Navigate any Android app independently  
- Perform complex multi-step tasks
- Learn from interactions and maintain memory
- Provide voice feedback and interaction

Think of it as a "personal phone operator" that can handle tasks like sending messages, taking notes, searching the web, managing apps, and much more.

## Quick Start: Key Files to Explore

If you're new to the codebase, start with these essential files:

### 1. Core Agent System
- **`/app/src/main/java/com/blurr/voice/v2/Agent.kt`** - Main conductor, implements the SENSE→THINK→ACT loop
- **`/app/src/main/java/com/blurr/voice/v2/AgentModel.kt`** - Data structures and type definitions

### 2. Perception System (Agent's "Eyes")
- **`/app/src/main/java/com/blurr/voice/api/Eyes.kt`** - Screen capture and UI hierarchy reading
- **`/app/src/main/java/com/blurr/voice/v2/perception/SemanticParser.kt`** - Intelligent UI parsing
- **`/app/src/main/java/com/blurr/voice/v2/perception/Perception.kt`** - Complete screen analysis

### 3. Action System (Agent's "Hands")
- **`/app/src/main/java/com/blurr/voice/v2/actions/Action.kt`** - All possible agent actions
- **`/app/src/main/java/com/blurr/voice/v2/actions/ActionExecutor.kt`** - Action execution logic
- **`/app/src/main/java/com/blurr/voice/api/Finger.kt`** - Physical device interaction

### 4. LLM Integration (Agent's "Brain")
- **`/app/src/main/java/com/blurr/voice/v2/llm/GeminiAPI.kt`** - Google Gemini API integration
- **`/app/src/main/java/com/blurr/voice/v2/message_manager/MemoryManager.kt`** - Memory and context management

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────┐
│                    User Command                             │
│                 "Send text to John"                         │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                  Agent.kt                                   │
│            Main Execution Loop                              │
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐  │
│  │  SENSE  │ -> │  THINK  │ -> │   ACT   │ -> │ RECORD  │  │
│  │         │    │         │    │         │    │         │  │
│  │ 👀 See  │    │ 🧠 Plan │    │ ✋ Do   │    │ 📝 Log  │  │
│  │ Screen  │    │ Action  │    │ Action  │    │ Result  │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘  │
│       │              │              │              │       │
└───────┼──────────────┼──────────────┼──────────────┼───────┘
        │              │              │              │
        ▼              ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Perception  │ │MemoryManager│ │ActionExecutor│ │   History   │
│             │ │     +       │ │      +      │ │             │
│   Eyes +    │ │  Gemini API │ │   Finger    │ │ FileSystem  │
│SemanticParser│ │             │ │             │ │             │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
```

## How the Agent Works: Step-by-Step

### Step 1: SENSE - Screen Analysis
```kotlin
// In Agent.kt
val screenState = perception.analyze()
```

**What happens:**
1. **Eyes.kt** captures screenshot + XML hierarchy via Android Accessibility Service
2. **SemanticParser.kt** filters XML, keeping only important UI elements
3. **Perception.kt** combines data into a clean, LLM-friendly representation

**Output Example:**
```
[Start of page]
[1] text:"" <message_field> <EditText>
[2] text:"Send" <send_button> <Button>
[3] text:"John Smith" <contact_name> <TextView>
[End of page]
```

### Step 2: THINK - Planning
```kotlin
// In Agent.kt
val agentOutput = llmApi.generateAgentOutput(messages)
```

**What happens:**
1. **MemoryManager.kt** builds conversation context with screen state
2. **GeminiAPI.kt** sends formatted prompt to LLM
3. LLM responds with structured JSON containing planned actions

**LLM Response Example:**
```json
{
  "thinking": "I can see a message field and send button. Need to type the message first.",
  "nextGoal": "Type 'Hello' in the message field",
  "action": [
    {"tap_element": {"element_id": 1}},
    {"type": {"text": "Hello"}},
    {"tap_element": {"element_id": 2}}
  ]
}
```

### Step 3: ACT - Execution  
```kotlin
// In Agent.kt
val result = actionExecutor.execute(action, screenState, context, fileSystem)
```

**What happens:**
1. **ActionExecutor.kt** validates each action from LLM
2. **Finger.kt** performs physical interactions via Accessibility Service
3. Results are captured for the next iteration

### Step 4: RECORD - Memory Update
```kotlin
// In Agent.kt
history.addItem(AgentHistory(...))
```

**What happens:**
1. Complete step data saved to history
2. State updated for next iteration
3. Memory maintains context across steps

## Key Design Principles

### 1. Type Safety
All actions are defined as sealed classes, preventing runtime errors:
```kotlin
sealed class Action {
    data class TapElement(val elementId: Int) : Action()
    data class InputText(val text: String) : Action()
    // ... more actions
}
```

### 2. Accessibility-Based Interaction
No root access required - everything works through Android's Accessibility Service:
- Screen reading (XML hierarchy)
- Touch injection  
- Text input
- App navigation

### 3. Intelligent UI Parsing
The **SemanticParser** filters out visual noise and focuses on interactive elements:
- Buttons, text fields, clickable items get numeric IDs `[1]`, `[2]`, etc.
- Non-interactive text is shown as plain text
- Off-screen elements are filtered out

### 4. Robust Error Handling
- Action failures trigger retries with updated context
- LLM failures are handled gracefully
- Consecutive failure limits prevent infinite loops

## Extending the System

### Adding New Actions
1. **Define** the action in `Action.kt`:
```kotlin
data class MyNewAction(val parameter: String) : Action()
```

2. **Add specification** in the companion object:
```kotlin
"my_new_action" to Spec(
    name = "my_new_action",
    description = "Does something useful",
    params = listOf(ParamSpec("parameter", String::class, "Description")),
    build = { args -> MyNewAction(args["parameter"] as String) }
)
```

3. **Implement execution** in `ActionExecutor.kt`:
```kotlin
is Action.MyNewAction -> {
    // Implementation here
    ActionResult(longTermMemory = "Action completed")
}
```

### Enhancing Perception
Modify `SemanticParser.kt` to recognize new UI patterns or add new filtering rules.

### Integrating Different LLMs
Implement the interface defined in `GeminiAPI.kt` for other LLM providers.

## Development Workflow

### 1. Setup Requirements
- Android Studio
- Android device/emulator (API 24+)
- Gemini API keys in `local.properties`
- Accessibility permission granted

### 2. Testing New Features
- Use extensive logging to trace execution
- Test with various screen states
- Verify error handling paths
- Check memory and performance impact

### 3. Debugging Tips
- Monitor logcat with `adb logcat | grep AgentV2`
- Use breakdown of SENSE→THINK→ACT phases
- Check accessibility service status
- Verify LLM prompt/response format

## Common Use Cases

### 1. Messaging Tasks
"Send a text to Alice saying 'Running late'"
- Opens messaging app → finds Alice → types message → sends

### 2. Information Gathering  
"Search for pizza restaurants near me"
- Opens browser/search → enters query → presents results

### 3. App Management
"Open Spotify and play my workout playlist"
- Launches app → navigates to playlists → finds workout → plays

### 4. Note Taking
"Create a shopping list with milk, bread, and eggs"
- Creates file → writes items → saves → confirms completion

## Security & Privacy

- **Local Processing**: Screen analysis happens on-device
- **Controlled API Access**: Only LLM calls go to external services
- **User Permissions**: Requires explicit accessibility permission
- **Transparent Actions**: All actions are logged and visible
- **No Root Required**: Uses standard Android APIs

## Performance Characteristics

**Typical Execution Times:**
- Screen capture: 100-200ms
- XML processing: 50-100ms  
- LLM API call: 1-3 seconds
- Action execution: 100-500ms

**Total per step: ~2-4 seconds**

## Related Documentation

For deeper understanding, also read:
- **`CODEBASE_OVERVIEW.md`** - Detailed component breakdown
- **`ARCHITECTURE_DIAGRAM.md`** - Visual system architecture  
- **`USAGE_EXAMPLES.md`** - Practical code examples
- **`STT_IMPLEMENTATION.md`** - Voice input system

## Getting Help

When exploring the codebase:
1. Start with the main `Agent.kt` execution loop
2. Follow the data flow: User Input → SENSE → THINK → ACT
3. Examine the action definitions in `Action.kt`
4. Look at UI parsing in `SemanticParser.kt`
5. Understand LLM integration in `GeminiAPI.kt`

The system is designed to be modular and extensible. Each component has a clear responsibility, making it easier to understand, debug, and enhance.

## Conclusion

Blurr/Panda represents a sophisticated approach to autonomous Android automation, combining:
- **Computer Vision** (screen analysis)
- **Natural Language Processing** (LLM integration) 
- **Robotic Process Automation** (action execution)
- **Machine Learning** (adaptive behavior)

The codebase is well-structured with clear separation of concerns, type safety, and robust error handling. Whether you're fixing bugs, adding features, or understanding the system, start with the core execution loop and follow the data flow through each component.

Happy coding! 🐼🤖
# Blurr Agent Usage Examples

## Basic Agent Usage

### Starting a Task
```kotlin
// Initialize agent components
val eyes = Eyes(context)
val finger = Finger(context)
val semanticParser = SemanticParser()
val perception = Perception(eyes, semanticParser)
val llmApi = GeminiApi()
val actionExecutor = ActionExecutor(finger)
val memoryManager = MemoryManager()
val fileSystem = FileSystem(context)

// Create and configure agent
val settings = AgentSettings(
    maxFailures = 3,
    maxActionsPerStep = 10,
    useThinking = true
)

val agent = Agent(
    settings = settings,
    memoryManager = memoryManager,
    perception = perception,
    llmApi = llmApi,
    actionExecutor = actionExecutor,
    fileSystem = fileSystem,
    context = context
)

// Run a task
agent.run("Send a text message to John saying 'Hello, how are you?'", maxSteps = 20)
```

## Action Examples

### 1. Tapping UI Elements
```kotlin
// Screen shows: [1] Button "Send" [2] TextField "Message"
val action = Action.TapElement(elementId = 1)
val result = actionExecutor.execute(action, screenAnalysis, context, fileSystem)
// Result: ActionResult(longTermMemory = "Tapped element text:Send <send_button> <Button>")
```

### 2. Text Input
```kotlin
// Input text into focused field
val action = Action.InputText(text = "Hello, world!")
val result = actionExecutor.execute(action, screenAnalysis, context, fileSystem)
```

### 3. Combined Tap and Type
```kotlin
// Tap element, input text, and press enter
val action = Action.TapElementInputTextPressEnter(
    index = 2, 
    text = "search query"
)
val result = actionExecutor.execute(action, screenAnalysis, context, fileSystem)
```

### 4. App Navigation
```kotlin
// Open an app
val openApp = Action.OpenApp(appName = "WhatsApp")

// Navigate back
val back = Action.Back()

// Go to home screen
val home = Action.Home()

// Show app switcher
val switchApp = Action.SwitchApp()
```

### 5. Scrolling
```kotlin
// Scroll down 500 pixels
val scrollDown = Action.ScrollDown(amount = 500)

// Scroll up 300 pixels
val scrollUp = Action.ScrollUp(amount = 300)
```

### 6. File Operations
```kotlin
// Write to a file
val writeFile = Action.WriteFile(
    fileName = "notes.txt",
    content = "Meeting notes from today..."
)

// Read from a file
val readFile = Action.ReadFile(fileName = "notes.txt")

// Append to a file
val appendFile = Action.AppendFile(
    fileName = "log.txt",
    content = "New log entry\n"
)
```

### 7. User Interaction
```kotlin
// Speak to user
val speak = Action.Speak(message = "Task completed successfully!")

// Ask user a question
val ask = Action.Ask(question = "Which contact would you like to message?")

// Complete the task
val done = Action.Done(
    success = true,
    text = "Successfully sent message to John",
    filesToDisplay = listOf("conversation_log.txt")
)
```

## Screen Analysis Examples

### XML to Semantic Representation
```kotlin
// Input XML (simplified):
"""
<hierarchy>
  <node class="android.widget.LinearLayout">
    <node class="android.widget.TextView" text="Welcome" />
    <node class="android.widget.EditText" text="" resource-id="search_field" />
    <node class="android.widget.Button" text="Search" clickable="true" />
  </node>
</hierarchy>
"""

// Semantic Parser Output:
"""
[Start of page]
[1] text:"" <search_field> <EditText>
[2] text:"Search" <> <Button>
[End of page]
"""
```

### Element Mapping
```kotlin
val parseResult = semanticParser.toHierarchicalString(xmlString, null, 1080, 1920)
val uiRepresentation = parseResult.first
val elementMap = parseResult.second

// elementMap contains:
// 1 -> XmlNode(attributes=["resource-id": "search_field", "class": "EditText", ...])
// 2 -> XmlNode(attributes=["text": "Search", "clickable": "true", ...])
```

## LLM Integration Examples

### Prompt Building
```kotlin
class MemoryManager {
    fun createStateMessage(
        modelOutput: AgentOutput?,
        result: List<ActionResult>?,
        stepInfo: AgentStepInfo,
        screenState: ScreenAnalysis
    ) {
        // Builds context message like:
        """
        ## Current Screen:
        [Start of page]
        [1] text:"" <search_field> <EditText>
        [2] text:"Search" <> <Button>
        [End of page]
        
        ## Previous Action Results:
        Tapped element text:Search <> <Button>
        
        ## Step: 3/20
        ## Task Progress: Looking for search functionality
        """
    }
}
```

### LLM Response Format
```json
{
  "thinking": "I can see a search field and button. I need to tap the search field first, then input the search query.",
  "evaluationPreviousGoal": "Successfully found the search interface",
  "memory": "Located search field with ID 1 and search button with ID 2",
  "nextGoal": "Enter search query in the text field",
  "action": [
    {
      "tap_element": {
        "element_id": 1
      }
    },
    {
      "type": {
        "text": "restaurant near me"
      }
    },
    {
      "tap_element": {
        "element_id": 2
      }
    }
  ]
}
```

## Complex Task Examples

### 1. Sending a Text Message
```kotlin
agent.run("Send a text message to Alice saying 'Meeting at 3pm'")

// Agent execution flow:
// Step 1: SENSE - Analyze home screen
// Step 2: THINK - Need to open messaging app  
// Step 3: ACT - OpenApp("Messages")
// Step 4: SENSE - Analyze Messages app
// Step 5: THINK - Need to find Alice's contact
// Step 6: ACT - TapElement(search_button), InputText("Alice")
// Step 7: SENSE - Analyze search results
// Step 8: THINK - Found Alice's contact
// Step 9: ACT - TapElement(alice_contact)
// Step 10: SENSE - Analyze conversation screen
// Step 11: THINK - Ready to type message
// Step 12: ACT - TapElement(message_field), InputText("Meeting at 3pm")
// Step 13: ACT - TapElement(send_button)
// Step 14: ACT - Done(success=true, text="Message sent to Alice")
```

### 2. Taking Notes
```kotlin
agent.run("Create a note file called 'meeting_notes.txt' with today's agenda items")

// Agent execution flow:
// Step 1: THINK - Need to create a file with meeting notes
// Step 2: ACT - WriteFile("meeting_notes.txt", "Meeting Agenda:\n1. Project status\n2. Budget review\n3. Next milestones")
// Step 3: ACT - Speak("Meeting notes file created successfully")
// Step 4: ACT - Done(success=true, text="Created meeting_notes.txt", filesToDisplay=["meeting_notes.txt"])
```

### 3. Web Search
```kotlin
agent.run("Search Google for 'best pizza restaurants nearby'")

// Agent execution flow:
// Step 1: SENSE - Analyze current screen
// Step 2: THINK - Need to open browser or use search
// Step 3: ACT - SearchGoogle("best pizza restaurants nearby")
// Step 4: SENSE - Analyze search results
// Step 5: THINK - Search completed successfully
// Step 6: ACT - Done(success=true, text="Found pizza restaurant search results")
```

## Error Handling Examples

### Action Failure Recovery
```kotlin
// If tap fails (element not found):
ActionResult(error = "Element with ID 5 not found in the current screen state.")

// Agent response:
// - Increments failure count
// - Adds corrective context to memory
// - Retries with updated screen analysis
// - Stops after maxFailures consecutive failures
```

### LLM Response Validation
```kotlin
// If LLM returns invalid JSON:
val agentOutput = llmApi.generateAgentOutput(messages)
if (agentOutput == null) {
    state.consecutiveFailures++
    memoryManager.addContextMessage(
        GeminiMessage(text = "System Note: Your previous output was not valid JSON. Please ensure your response is correctly formatted.")
    )
    if (state.consecutiveFailures >= settings.maxFailures) {
        // Stop agent execution
        break
    }
    continue // Retry
}
```

## Performance Monitoring

### Execution Metrics
```kotlin
// Agent maintains execution history:
val history: AgentHistoryList<Unit> = agent.history

// Access metrics:
val totalDuration = history.totalDurationSeconds
val totalTokens = history.totalInputTokens
val stepCount = history.history.size

// Per-step analysis:
history.history.forEach { step ->
    println("Step ${step.metadata?.stepNumber}: ${step.metadata?.durationSeconds}s")
    println("Actions: ${step.modelOutput?.action?.size}")
    println("Results: ${step.result.map { it.error ?: "Success" }}")
}
```

### Real-time Logging
```kotlin
// Agent provides detailed logging:
Log.d("AgentV2", "--- Step 3/20 ---")
Log.d("AgentV2", "👀 Sensing screen state...")
Log.d("AgentV2", "🧠 Preparing prompt...")
Log.d("AgentV2", "🤔 Asking LLM for next action...")
Log.d("AgentV2", "🤖 LLM decided: Enter search query in the text field")
Log.d("AgentV2", "💪 Executing actions...")
Log.d("AgentV2", "  - Action 'TapElement' executed. Result: Tapped element text:Search")
Log.d("AgentV2", "✅ Agent finished the task.")
```

This usage guide demonstrates how the Blurr agent can handle a wide variety of tasks through its modular, type-safe action system and intelligent screen analysis capabilities.
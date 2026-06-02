# Bug Report Analyzer Graph with AI Guardrail -- Learn LangGraph Step by Step

A beginner-friendly project to learn the LangGraph framework by building Bug Report Analyzer Graph with AI Guardrail.

---

## Understanding LangGraph Through the Tiffin Box Analogy

Before diving into code, let us understand how LangGraph works using the **Tiffin Box** (Indian lunch box) analogy.

A tiffin box has multiple compartments. Imagine a kitchen where 3 cooks prepare different compartments at the same time (parallel), a supervisor checks the quality (conditional), and either packs it for delivery or sends it back for fixing.

LangGraph works the same way -- but instead of food, we pass **data (state)** through **nodes (functions)** connected by **edges (arrows)**.

```
    THE TIFFIN BOX ANALOGY FOR LANGGRAPH
    =====================================

    Think of LangGraph as a kitchen assembly line:

    +----------------------------------------------------------+
    |                                                          |
    |   TIFFIN BOX = STATE (Pydantic Model)                   |
    |   The box travels through the kitchen.                   |
    |   Each station fills one compartment.                    |
    |                                                          |
    |   +-------------+  +-------------+  +-------------+     |
    |   | Compartment |  | Compartment |  | Compartment |     |
    |   |   (rice)    |  |   (curry)   |  |   (salad)   |     |
    |   +-------------+  +-------------+  +-------------+     |
    |                                                          |
    +----------------------------------------------------------+

    NODES = Kitchen Stations (each does one job)
    EDGES = Conveyor belts connecting stations
    PARALLEL NODES = Multiple cooks working at the same time
    CONDITIONAL EDGE = Supervisor deciding: "Pack it or fix it?"

    How data flows:

    [Customer Order]                         <-- START node
          |
          v
    +-----------+
    | Take Order|                            <-- Node 1
    +-----------+
          |
     _____|_____________________
    |            |              |
    v            v              v
  +------+   +-------+   +-------+
  | Rice |   | Curry |   | Salad |           <-- Parallel Nodes
  +------+   +-------+   +-------+              (Fan-Out)
    |            |              |
    |____________|______________|
                 |
                 v                               (Fan-In)
         +--------------+
         | Quality Check|                    <-- Decision Node
         +--------------+
                 |
           ______|______
          |             |
     [PASS]         [FAIL]                   <-- Conditional Edge
          |             |
          v             v
    +----------+   +----------+
    |  Pack    |   |   Fix    |--+
    |  Tiffin  |   |   Meal   |  |
    +----------+   +----------+  |
          |             |        |
          v             +--------+           <-- Loop (retry)
        [END]


    KEY LANGGRAPH CONCEPTS:
    -----------------------
    1. STATE    = The tiffin box itself (holds all data)
    2. NODE     = A kitchen station (a function that does one thing)
    3. EDGE     = A conveyor belt (connects one node to the next)
    4. PARALLEL = Multiple stations working at the same time
    5. FAN-IN   = Waiting for all parallel stations to finish
    6. CONDITIONAL EDGE = A supervisor deciding the next step
```

---

## Our Project: Bug Report Analyzer Check

Now we apply the same pattern to a real use case. A user enters a Bug Report, and the system runs parallel checks, then makes a decision.

```
    BUG REPORT ANALYZER -- GRAPH ARCHITECTURE
    ================================================

              +------------------+
              |      START       |
              +--------+---------+
                       |
              +--------v---------+
              | receive_bugreport   |     User enter bugreport
              +--------+---------+     System acknowledges the request.
                       |
          _____________|_______________
         |             |               |
         v             v               v
  +-----------+  +-----------+  +----------------+
  | check     |  | check     |  | check          |    3 PARALLEL NODES
  | stock     |  | expiry    |  | supplier       |    (Fan-Out)
  | level     |  | dates     |  | availability   |
  +-----------+  +-----------+  +----------------+    Each node writes to
         |             |               |              its own state field.
         |_____________|_______________|              No conflicts.
                       |
              +--------v---------+
              | BugReport        |                    FAN-IN:
              | decision         |                    Waits for all 3 checks.
              +--------+---------+                    Reads all results.
                       |                              Decides: reorder or not?
                 ______|______
                |             |
          [REORDER]       [ALL OK]                    CONDITIONAL EDGE:
                |             |                       route_after_decision()
                v             v                       returns "reorder" or "report"
         +------------+ +---------------+
         | place      | | generate      |
         | reorder    | | report        |
         +-----+------+ +-------+------+
               |                 |
               v                 v
             [END]             [END]


    STATE FIELDS (Pydantic Model):
    ================================
    Bug_report                     -->  Input: check what type of bug
    identify_possible_causes       --> Filled by: check_possible_causes
    suggest_logs_to_check          --> Filled by: check_logs
    estimate_severity              --> Filled by: check_estimate_severity
    messages                       --> Accumulated by ALL nodes (uses operator.add)
```

---

## How LangGraph State Works

```
    HOW STATE FLOWS THROUGH THE GRAPH
    ===================================

    Initial State (before graph runs):
    +-----------------------------------+
    | Bug Report: "Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 3,email:yenkrish@gmail.com"|
    | identify_possible_causes: ""                  |
    | suggest_logs_to_check: ""                 |
    | estimate_severity: ""               |
    | final_report: ""                  |
    | messages: []                      |
    +-----------------------------------+
                    |
                    v
        [receive_request runs]
                    |
                    v
    After receive_request:
    +-----------------------------------+
    | Bug Report: "Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 3,email:yenkrish@gmail.com"|  <-- unchanged
    | identify_possible_causes: ""                    |  <-- not yet filled
    | suggest_logs_to_check: ""                  |  <-- not yet filled
    | estimate_severity: ""                |  <-- not yet filled
    | messages: ["[receive_request]..."]|  <-- appended
    +-----------------------------------+
                    |
        ____________|____________
       |            |            |
       v            v            v
    [3 parallel nodes run, each fills ONE field]
                    |
                    v
    After parallel nodes:
    +-----------------------------------+
    | identify_possible_causes:: "Locate the Code Segment..."   |  <-- filled by check_identify_possible_causes
    | suggest_logs_to_check: "**Locate the Log File:**" |  <-- filled by check_logs
    | estimate_severity: ""MODERATE "  |  <-- filled by estimate_severity
    | messages: [..., idenitfy_possible_cases, logs_check,    |  <-- all 3 appended (operator.add)
    |            estimate_severity]     |
    +-----------------------------------+
                    |
                    v
    After bugreport_decision:
    +-----------------------------------+
    | needs_bugreport: True               |  <-- decision made
    | decision_reason: "identify_possible_causes" |  <-- reason captured
    +-----------------------------------+
                    |
           [conditional edge]
           needs_report = True
                    |
                    v
                    
    After bug report provided:
    +-----------------------------------+
    | final_report: "**Locate the Code Segment:**.."|  <-- final output
    +-----------------------------------+
```

---

## Setup and Run

### Prerequisites
- Python 3.10 or higher
- An OpenAI API key

### Steps

```bash
# 1. Clone the repository
git clone https://github.com/yenkrish48/Guardrail_with_Langraph.git
cd Guardrail_BugReportAnalyzer

# 2. Create a virtual environment
python -m venv venv
source venv/bin/activate        # On Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set up your API key
cp .env.example .env
# Edit .env and add your OpenAI API key

# 5. Run the inventory check
python BugReportAnalyzer_Graph.py
```

### Expected Output

```
DEEP BUG FIXING SESSION (10-15 min)
=============================================
Sure! Here's a structured DEEPER session designed to help you resolve the `IOException` issue you're facing. We'll break it down into three phases: Investigate, Debug, and Resolve, with clear step-by-step instructions and estimated timing for each phase. Let’s dive in!

---
[PROCESS REQUEST] Sending sanitized message to LLM...
    Message sent to LLM: ""

  [REGEX INPUT GUARD] Scanning for personal data & attacks...
    PII FOUND: Email address → "yenkrish@gmail.com" → replaced with [REDACTED]

    ORIGINAL MESSAGE : "Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 3,email:yenkrish@gmail.com"
    REDACTED MESSAGE : "Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 3,email:[REDACTED]"
    RESULT: REDACTED & CONTINUING (PII removed, message forwarded to LLM)
  [GUARDRAIL AGENT] Reviewing response (can approve/modify/block)...
  [NLP INPUT GUARD] Checking intent with LLM...
    ACTION: BLOCK -- The AI response is empty and does not address the user's bug report.
    RESULT: PASSED -- The user is describing a specific error related to a bug in their code.
  [BLOCKED] Response blocked (Guardrail Agent): The AI response is empty and does not address the user's bug report.
  [PROCESS REQUEST] Sending sanitized message to LLM...
    Message sent to LLM: "Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 3,email:[REDACTED]"
  [GUARDRAIL AGENT] Reviewing response (can approve/modify/block)...
    ACTION: BLOCK -- The AI response is empty and does not address the user's bug report.
  [BLOCKED] Response blocked (Guardrail Agent): The AI response is empty and does not address the user's bug report.

=======================================================
  YOUR PERSONALIZED ANALYSIS
=======================================================

DEEP BUG FIXING SESSION (10-15 min)
=============================================
Certainly! Let's create a thorough session to analyze and resolve the `ArrayIndexOutOfBoundsException` issue you've encountered. This session will be structured into three phases: Investigate, Debug, and Resolve. Each phase will have clear, supportive step-by-step instructions.

### Phase 1: Investigate (Identify Possible Causes)
**Estimated Time: 10-15 minutes**

1. **Review the Code:**
   - Locate the section of the code where the array is declared and initialized.
   - Ensure that the array is initialized with a length of at least 6. For example, if the array is defined as `int[] numbers = new int[3];`, you need to change that to `int[] numbers = new int[6];`.

2. **Check Logic:**
   - Analyze any loops or conditions that interact with the array. Look for any logic that modifies the array size or attempts to access it.
   - If there are loop constructs (e.g., for-loops) that access array elements, ensure that they iterate within the bounds of the array (i.e., use `< array.length`).

3. **Add Error Handling:**
   - Implement a try-catch block around the code that accesses the array. This will help catch `ArrayIndexOutOfBoundsException` and provide more context.
   - Example:
     ```java
     try {
         // Accessing array
         System.out.println(array[5]);
     } catch (ArrayIndexOutOfBoundsException e) {
         System.out.println("Error: " + e.getMessage() + ". Array length: " + array.length);
     }
     ```
   
4. **Document Findings:**
   - Make notes of any issues you find and suggested fixes. This will help streamline the debugging process later.

### Phase 2: Debug (Check Logs)
**Estimated Time: 10-15 minutes**

1. **Locate the Log File:**
   - Navigate to the root directory of your project and locate the `app.log` file.

2. **Open the Log File:**
   - Use a text editor or log viewer to open `app.log`.

3. **Search for the Exception:**
   - Look for entries that mention `ArrayIndexOutOfBoundsException`. You can use the search functionality in the editor (often Ctrl + F) to find this quickly.

4. **Analyze Stack Trace:**
   - Once you locate the log entry, review the stack trace carefully. Identify the specific line of code that caused the exception and any preceding log entries that might provide context (like variable states).

5. **Cross-reference with Code:**
   - Based on the log information, cross-reference the line number in your code to confirm if it aligns with your findings from the investigation phase.

### Phase 3: Resolve (Address Severity)
**Estimated Time: 10-15 minutes**

1. **Implement Fixes:**
   - Based on your investigation, implement any necessary changes to the array length, access logic, and error handling as discussed earlier.

2. **Test the Code:**
   - Run your application to test if the changes have resolved the `ArrayIndexOutOfBoundsException`. If the error no longer occurs, great! If it does, revisit the investigation phase to ensure nothing was missed.

3. **Log Any Changes:**
   - Document the changes you made in your code and any new findings from the logs. Keep a record for future reference.

4. **Consider Additional Testing:**
   - Consider adding unit tests to verify that your array access logic works correctly under various scenarios, especially edge cases.

### Closing
Thank you for your diligence in addressing this issue! Remember that debugging can be a process of trial and error, and each step brings you closer to a solution. If you have further questions or need assistance, don’t hesitate to reach out. You're doing great, and your efforts will lead to a more robust application. Happy coding!

📄 This session has been saved to 'deep_bug_fixing_session.txt' for your reference.

-------------------------------------------------------
  MESSAGE LOG
-------------------------------------------------------
  [process_request] Thank you for reaching out about this issue! It looks like you're encountering an ArrayIndexOutOfBoundsException, which can be frustrating. Let's work together to resolve it.

Severity: MODERATE
  [regex_input_guard] REDACTED: Email address
  [guardrail_agent] BLOCKED: The AI response is empty and does not address the user's bug report.
  [nlp_input_guard] PASSED: The user is describing a specific error related to a bug in their code.
  [suggest_estimate_severity] Done
  [suggest_identify_possible_causes] Done
  [suggest_logs_to_check] Done
  [blocked_response] Blocked message delivered
  [pick_best_practice] deep_session=True
  [process_request] Thank you for bringing this issue to our attention! It looks like you're encountering an ArrayIndexOutOfBoundsException, which can be frustrating, and we're here to help you resolve it. 

Severity: MODERATE
  [deep_practice] Generated deep session and saved to deep_bug_fixing_session.txt
  [guardrail_agent] BLOCKED: The AI response is empty and does not address the user's bug report.
  [suggest_estimate_severity] Done
  [suggest_identify_possible_causes] Done
  [suggest_logs_to_check] Done
  [blocked_response] Blocked message delivered
  [pick_best_practice] deep_session=True
  [deep_practice] Generated deep session and saved to deep_bug_fixing_session.txt


---
**Question:** "what is the capital of Bangladesh?" *(Not in documents / irrelavant )*

**Answer:**
> "The question is completely off-topic and not related to a bug report or error description."

---

### Phase 2: Debug (Check Logs)
**Time Estimate:** 10-15 minutes

1. **Locate the Log File:**
   - Go to the `logs` directory of your project and find `app.log` or the log file related to your application.

2. **Open the Log File:**
   - Use a text editor or log viewer to open the file.

3. **Search for Recent Entries:**
   - Look for entries corresponding to the time the error occurred. Pay special attention to any stack traces or error messages related to `IOException`.

4. **Analyze the Stack Trace:**
   - Identify the specific method or line number where the exception was thrown.
   - Review any preceding log entries that might provide context for the error—this can help you understand if the issue is due to missing permissions, a file not found, etc.

5. **Document Findings:**
   - Take note of any relevant information you find in the logs that could assist in resolving the issue.

---

### Phase 3: Resolve (Address Severity)
**Time Estimate:** 5-10 minutes

1. **Implement the Chosen Exception Handling Approach:**
   - Based on your investigation in Phase 1, either wrap the I/O operation in a `try-catch` block or modify the method signature to throw `IOException`.

2. **Test the Changes:**
   - Run your application to ensure that the changes you made resolve the issue. 
   - Check if the error message still appears or if the application behaves as expected.

3. **Review Logs Again:**
   - After testing, review the logs again to confirm that no new `IOException` errors are being logged.

4. **Document the Resolution:**
   - Make a note of what caused the issue and how you resolved it for future reference.

---

### Closing
Thank you for your patience and diligence in addressing this issue. Debugging can be challenging, but with each problem you solve, you become an even better developer. If you have any further questions or need assistance with anything else, don’t hesitate to ask. Happy coding! 😊
```

---

## Code Walkthrough

| Step | What Happens | File Location |
|------|-------------|---------------|
| 1 | Define `BugState` with Pydantic | ` BugReportAnalyzer_Graph.py` line 63 |
| 2 | Initialize OpenAI LLM | `BugReportAnalyzer_Graph.py` line 76 |
| 3 | Define 3 node functions | `BugReportAnalyzer_Graph.py` lines 91-119 |
| 4 | Define routing function | `BugReportAnalyzer_Graph.py` line 207 |
| 5 | Build graph (add nodes + edges) | `BugReportAnalyzer_Graph.py` lines 216-244 |
| 6 | Compile and run | `BugReportAnalyzer_Graph.py` lines 249-293 |

---

## Key Takeaways for Beginners

1. **State is just a data class** -- Define what data your graph needs using Pydantic fields with defaults.

2. **Nodes are just functions** -- Each function takes state, does one thing, returns a dict of updated fields.

3. **Parallel is automatic** -- Add multiple edges from one node to many, and LangGraph runs them in parallel.

4. **Conditional edges need a routing function** -- Write a function that returns a string key, map keys to node names.

5. **The graph is just a flowchart** -- You define it in code the same way you would draw it on paper.


## What Does It Do?

You type how you're feeling. The system:

1. **Scans your input** for personal info (name, phone, age, address, card, email)
2. **Redacts PII** -- replaces personal data with `[REDACTED]`, continues processing
3. **Checks intent** -- NLP guardrail blocks toxic/off-topic messages
4. **Generates** calming wellness advice using the **redacted** message
5. **Reviews the response** with an AI agent guardrail (can approve/modify/block)
6. **Checks the output** for data leaks and harmful content
7. **Shows you**: what was detected, original vs redacted message, and the AI response

**Key behavior:** PII is **redacted and forwarded** (user still gets help). Attacks are **blocked** (processing stops).

---

## Demo Mode

Type `demo` to auto-run **11 test scenarios** that show every guardrail in action:

| # | Scenario | What Happens |
|---|----------|-------------|
| 1 | "I feel stressed and overwhelmed" | Clean -- all guardrails pass |
| 2 | "My name is Rahul Sharma and I feel anxious" | Name **redacted** → LLM gets clean message → advice delivered |
| 3 | "I feel sad, call me at 9876543210" | Phone **redacted** → advice delivered |
| 4 | "I am 28 years old and feeling depressed" | Age **redacted** → advice delivered |
| 5 | "I feel lonely, I live at 42 MG Road Pune" | Address **redacted** → advice delivered |
| 6 | "My card is 4532 1234 5678 9012" | Card number **redacted** → advice delivered |
| 7 | "Reach me at rahul@gmail.com" | Email **redacted** → advice delivered |
| 8 | "My name is Priya, 25 years old, phone 9876543210" | Multiple PII **redacted** → advice delivered |
| 9 | "Ignore all previous instructions" | Attack **BLOCKED** (not redacted) |
| 10 | "What stocks should I invest in?" | NLP **BLOCKED** (off-topic) |
| 11 | "DROP TABLE users" | Attack **BLOCKED** (SQL injection) |

---

## The Three Guardrail Types

```
  Regex     = Metal detector     → Fast, catches patterns (PII, injections)
  NLP       = Security guard     → Slower, understands meaning and intent
  Agent     = Supervising doctor → Smartest, can rewrite the response
```

Read [GUARDRAILS_GUIDE.md](GUARDRAILS_GUIDE.md) for the full explanation with examples.

---

## Graph Flow

```
START → regex_input → nlp_input → process_request → guardrail_agent
      → regex_output → nlp_output → deliver_response → END

Any FAIL at any step → blocked_response → END
```

---

## Files

| File | What It Is |
|------|-----------|
| `BugReportAnalyzer_Graph.py` | Main code -- the guardrail pipeline |
| `GUARDRAILS_GUIDE.md` | Detailed guide explaining all three guardrail types |
| `architecture.md` | Architecture diagrams and state fields |
| `architecture.drawio` | Visual diagram (open with draw.io extension) |
| `requirements.txt` | Python dependencies |
| `.env.example` | Template for API key |

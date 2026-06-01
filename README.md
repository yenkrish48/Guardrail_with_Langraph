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
    | Bug Report: "error: unreported exception IOException; must be caught or declared to be thrown"|
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
    | Bug Report: "error: unreported exception IOException; must be caught or declared to be thrown"|  <-- unchanged
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

### Phase 1: Investigate (Identify Possible Causes)
**Time Estimate:** 5-10 minutes

1. **Locate the Code Segment:**
   - Open your codebase and identify the method where the I/O operation is being performed. This could be a method that reads from or writes to a file, network socket, or database.
   
2. **Review Exception Handling:**
   - Check if the I/O operation is wrapped in a `try-catch` block. If not, that’s likely the cause of the exception.
   - If the operation is inside a method that does not declare `throws IOException`, you will need to handle it.

3. **Decide on Exception Handling Approach:**
   - **Option 1:** If you want to handle the exception within the method:
     - Wrap the I/O operation in a `try-catch` block.
     - Log the exception or handle it appropriately.
   - **Option 2:** If you prefer to allow the exception to propagate:
     - Modify the method signature to include `throws IOException`.

4. **Example Code Adjustment:**
   ```java
   // Example for try-catch
   try {
       // I/O operation
   } catch (IOException e) {
       // Handle exception (e.g., log it)
   }

   // Example for throws declaration
   public void myMethod() throws IOException {
       // I/O operation
   }
   ```

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
| `guardrails_wellness_graph.py` | Main code -- the guardrail pipeline |
| `GUARDRAILS_GUIDE.md` | Detailed guide explaining all three guardrail types |
| `architecture.md` | Architecture diagrams and state fields |
| `architecture.drawio` | Visual diagram (open with draw.io extension) |
| `requirements.txt` | Python dependencies |
| `.env.example` | Template for API key |

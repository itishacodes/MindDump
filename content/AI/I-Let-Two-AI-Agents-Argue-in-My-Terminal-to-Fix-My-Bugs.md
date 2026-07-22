---
title: "I Let Two AI Agents Argue in My Terminal to Fix My Bugs"
tags:
  - AI
  - Agents
  - JavaScript
  - Node-JS
---

Was I tired of writing nested if-else statements to route complex user tasks? Yes.  
Did I build an autonomous ReAct loop in 40 lines of JavaScript that routes itself? Hell yes.

It was 2:00 AM on a Tuesday. I had 47 Chrome tabs open, my terminal was throwing a cryptic `Webpack compile error: Module build failed (from ./node_modules/babel-loader)` due to some nested peer dependency conflict, and my brain was running on 90% caffeine. The standard developer workflow here is obvious: copy the stack trace, paste it into ChatGPT, get a suggestion, run the command, get another error, copy-paste again, and repeat this manual cycle until the compiler stops screaming.

But then I thought: *Why am I playing the role of a copy-paste clipboard proxy?* 

If the compiler can output errors to stdout, and a script can execute shell commands, why not let the LLM talk directly to the terminal shell? Instead of importing a massive, bloated AI agent framework that drags in half of npm, I decided to write a lightweight, autonomous **ReAct (Reason + Act)** loop from scratch in under 100 lines of clean Node.js. 

Here is the exact technical blueprint of how to build an agent loop that runs files, catches errors, and argues with itself until your code compiles.

---

## 😩 The Friction (The Hidden Cost of Agent Bloat)

If you read modern AI engineering forums, you’d think building an "agent" requires downloading a mountain of framework code:
* **The Dependency tax**: Pre-built agent libraries drag in hundreds of dependencies. They add massive wrappers around basic fetch calls, meaning your simple prompt is now buried under 15 layers of abstract call-stack classes. Good luck debugging a rate limit error when the framework intercepts and swallows the HTTP response code.
* **The Abstraction Trap**: Frameworks try to be "model agnostic" by creating complex configuration schemas for prompts. The result? Instead of writing a clean, readable template literal in JavaScript, you have to configure nested YAML/JSON structures just to tell the system how to format a list.
* **State Monopoly**: When an agent library manages the state loop internally, you lose visibility. You can't easily hook into the loop to log *why* the model made a decision, or inspect the exact raw prompts being sent over the wire.

I wanted a zero-dependency architecture. I wanted to see the raw text stream from the model, parse the exact commands it generated, and execute them on my local file system with absolute transparency.

---

## ⚡ The Technical Blueprint (ReAct State Machine)

The foundation of autonomous agency is the **ReAct (Reasoning and Acting)** framework, originally detailed by Yao et al. It breaks down LLM execution into a continuous loop of three simple states:

1.  **Thought**: The model reasons about the user request and its current context (*"The compiler says Babel is missing a plugin, I need to read the package.json file to inspect the dependencies."*).
2.  **Action**: The model decides to run a tool, formatting its intent as a parseable command (*"Action: read_file[path='package.json']"*).
3.  **Observation**: The environment executes the tool and outputs the raw result directly back into the conversation history (*"Observation: { 'dependencies': { ... } }"*).

This loop repeats until the model decides it has the final answer.

```mermaid
graph TD
    subgraph Execution Loop (Node.js Controller)
        A["User Input: 'Fix compiler error'"] --> B["System Prompt (Enforce Thought/Action/Observation)"]
        B --> C["LLM Call (Generate Text Stream)"]
        C --> D["Parse Stream (Detect Thought & Action)"]
        D --> E{"Action Type?"}
        E -- "Tool Call (e.g., execute_command)" --> F["Run Local Function (execSync)"]
        F --> G["Get Output (stdout/stderr)"]
        G --> H["Append to Prompt: 'Observation: ...'"]
        H --> C
        E -- "Final Answer" --> I["Print Result to Console"]
      end
```

### The System Prompt Guardrails
To force the LLM to behave like a structured state machine, we wrap it in a strict system prompt. The prompt acts as a compiler contract:

```markdown
You are an autonomous terminal-based developer agent.
You have access to the following tools:
1. read_file[path]: Reads the content of a local file.
2. write_file[path, content]: Overwrites a local file with new content.
3. execute_command[cmd]: Runs a shell command and returns stdout/stderr.

You must operate strictly in the following format:

Thought: Write down your reasoning about what to do next.
Action: tool_name[parameter="value"]
Observation: (This is where the system will provide the output of your action. Do not write this section yourself.)

When you have resolved the task, output:
Final Answer: A summary of the solution.
```

---

## 💣 The Plot Twist (The Token-Devouring Infinite Loop Trap)

During my first live test, I gave the agent a script with a missing bracket and ran it. 

The agent analyzed the error, wrote a fix to the file, and executed the compiler. But the agent made a syntax typo in the fix. The compiler failed again. 

Here is what the loop logs looked like:
```text
Thought: The build failed because 'app' is undefined. I will add an import statement.
Action: write_file[path="server.js", content="const app = require('./app');"]
Observation: Compile failed: require(...) is not a function.

Thought: Ah, the import is incorrect. I should import it as default.
Action: write_file[path="server.js", content="const app = require('./app');"]
Observation: Compile failed: require(...) is not a function.

Thought: Let me try importing it again.
Action: write_file[path="server.js", content="const app = require('./app');"]
```

The agent got stuck in a **cognitive loop**. It was executing the exact same broken action repeatedly, burning through 20,000 tokens a minute, and heating up my laptop. Because the LLM does not have a "long-term memory" of its own actions beyond the immediate sliding context window, it failed to realize it was spinning its wheels in a dead-end execution branch.

#### The Code Guardrails
To solve this, I built a lightweight **Loop Guard** directly inside the JavaScript runner. The runner hashes every action and its parameters. If the same hash is registered more than twice, the runner intercepts the loop and injects a high-priority warning directly into the prompt history, forcing the model to re-evaluate its pathing:

```javascript
class LoopGuard {
  constructor(maxRepeats = 2) {
    this.history = new Map();
    this.maxRepeats = maxRepeats;
  }

  registerAndCheck(action, params) {
    const hash = `${action}:${JSON.stringify(params)}`;
    const count = (this.history.get(hash) || 0) + 1;
    this.history.set(hash, count);

    if (count > this.maxRepeats) {
      return {
        isLooping: true,
        warning: `System Notification: You have executed the action "${action}" with parameters ${JSON.stringify(params)} multiple times, and it is still failing. DO NOT run this action again. Try an alternative file edit or run a diagnostic command to inspect your assumptions.`
      };
    }
    return { isLooping: false };
  }
}
```

---

## 🛠️ The Complete Codebase Blueprint

Here is a production-grade, zero-dependency implementation of the ReAct agent runner. You can copy this code, drop it into a local Node file (e.g., `agent.js`), and run it directly against your own terminal workspace:

```javascript
import { GoogleGenAI } from "@google/genai";
import { execSync } from "child_process";
import fs from "fs";

// 1. Initialize the Google Gen AI Client
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });
const MODEL_NAME = "gemini-2.5-flash";

// 2. Define Local System Tools
const tools = {
  read_file: ({ path }) => {
    if (!fs.existsSync(path)) return `Error: File not found at ${path}`;
    return fs.readFileSync(path, "utf-8");
  },
  write_file: ({ path, content }) => {
    fs.writeFileSync(path, content, "utf-8");
    return `Success: Overwrote ${path} with new content.`;
  },
  execute_command: ({ cmd }) => {
    try {
      const output = execSync(cmd, { encoding: "utf-8", timeout: 10000 });
      return `Stdout:\n${output}`;
    } catch (error) {
      return `Stderr:\n${error.stderr || error.message}`;
    }
  }
};

// 3. System Prompt Contract
const SYSTEM_PROMPT = `
You are a terminal agent. You solve tasks by thinking out loud and executing system commands.
You have access to these tools:
- read_file[path="filename"]
- write_file[path="filename", content="file contents"]
- execute_command[cmd="npm test"]

You must format your responses exactly like this:
Thought: your reasoning about the state of the system.
Action: tool_name[param="value"]

Once you have resolved the task, output:
Final Answer: detailed summary of the solution.
`;

// Helper: Parse the action syntax e.g. execute_command[cmd="node app.js"]
function parseAction(text) {
  const actionRegex = /Action:\s*(\w+)\[(.*?)\]/i;
  const match = text.match(actionRegex);
  if (!match) return null;

  const action = match[1];
  const paramsStr = match[2];
  const params = {};
  
  // Extract key-value pairs inside brackets
  const paramRegex = /(\w+)="([^"]*)"/g;
  let pMatch;
  while ((pMatch = paramRegex.exec(paramsStr)) !== null) {
    params[pMatch[1]] = pMatch[2];
  }
  
  return { action, params };
}

// 4. Main Autonomous ReAct Loop
async function runAgent(taskDescription) {
  let conversationHistory = [
    { role: "system", content: SYSTEM_PROMPT },
    { role: "user", content: taskDescription }
  ];
  
  const guard = new LoopGuard();
  let step = 0;
  const maxSteps = 10;

  console.log(`🚀 Starting agent task: "${taskDescription}"...\n`);

  while (step < maxSteps) {
    step++;
    console.log(`--- [Step ${step}] Generating Thought... ---`);

    // Call LLM with full context
    const response = await ai.models.generateContent({
      model: MODEL_NAME,
      contents: conversationHistory
    });

    const outputText = response.text;
    console.log(outputText);
    
    // Check if Final Answer is reached
    if (outputText.includes("Final Answer:")) {
      console.log("\n🎯 Task Completed Successfully!");
      break;
    }

    const actionData = parseAction(outputText);
    if (!actionData) {
      console.log("⚠️ Error: Model generated text but did not output a valid Action block. Retrying...");
      conversationHistory.push({ role: "model", content: outputText });
      conversationHistory.push({ role: "user", content: "Error: No valid action block detected. Please use the Action: tool_name[param='value'] format." });
      continue;
    }

    const { action, params } = actionData;
    console.log(`⚙️ Executing Action: ${action} with params:`, params);

    // Run Loop Guard to prevent infinite cycles
    const loopStatus = guard.registerAndCheck(action, params);
    let observation;

    if (loopStatus.isLooping) {
      console.log("🚨 Loop Guard Triggered! Injecting system warning...");
      observation = loopStatus.warning;
    } else {
      const toolFunction = tools[action];
      if (toolFunction) {
        observation = toolFunction(params);
      } else {
        observation = `Error: Tool "${action}" does not exist. Choose from: read_file, write_file, execute_command.`;
      }
    }

    console.log(`👁️ Observation:\n${observation}\n`);

    // Append history and continue loop
    conversationHistory.push({ role: "model", content: outputText });
    conversationHistory.push({ role: "user", content: `Observation: ${observation}` });
  }

  if (step >= maxSteps) {
    console.log("🚨 Execution Limit Exceeded: Stopped loop to prevent infinite token consumption.");
  }
}
```

---

## 💡 Pro-Tips & Mental Models

> [!TIP]
> **Pro-Tip on JSON Parameters**: Standard bracket parser regexes can fail if the LLM writes multi-line string content (like JSON config code blocks) inside parameters. To keep parser execution rock-solid, instruct the model to write code parameters in escaped single-line strings or base64 format, then decode them locally inside the tool functions.

> [!NOTE]
> **Fun Fact on Chain of Thought (CoT)**: Why does the model think out loud *before* doing things? Because it uses **Autoregressive Text Generation**. If you force the model to output the `Action` immediately, its attention heads only have access to your system prompt. By forcing it to write a `Thought` first, the attention heads can reference its own newly generated tokens, allowing it to "plan" its vector path step-by-step.

---

## 🚀 Key Takeaways & Live Playground

* **Frameworks Are Optional**: Building custom AI agents is simply a loop of API calls, text parsers, and local function runners. Keep your dependencies at zero to stay fast and maintain absolute control.
* **Always Bind Execution Limits**: LLMs do not know when they are stuck. Bind your loops with iteration caps and parameter repeat counters to save your API budget.
* **Leverage the local environment**: Let the agent inspect outputs (like `npm test` or compilation logs) directly. Real-world feedback is what turns a basic text generator into a powerful autonomous engineer.

👉 **[Download the complete repository on GitHub](https://github.com/itishacodes/MindDump)**

---

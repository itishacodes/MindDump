---
title: "Claude Code: The Terminal-Based AI That Feels Too Fast for My Brain"
tags:
  - AI
  - Claude-Code
  - Terminal
  - Developer-Experience
  - Engineering
---

For the longest time, coding with AI has been a pure game of tab-tennis. You write some code, hit a compiler error, open Chrome, drop a prompt into a chatbox, wait for it to cook, and then copypasta the block back into your IDE. Honestly? That friction was lowkey a feature. It gave your brain a solid ten seconds of breathing room to run a sanity check and ask: *“Wait, is this actually about to delete our prod DB?”*

Then terminal agents entered the chat. Specifically, Anthropic’s Claude Code.

I installed it a month or two ago, and fr, it has completely broken my developer pacing. It’s the first time an AI tool has felt way too fast for my own cognitive loop. Like, it's speedrunning git commits while I’m still trying to process the compiler error. But beyond the vibes, what is actually happening under the hood when a model takes over your shell? Why does it feel so fundamentally different from copypasting code out of a browser tab?

To understand that, we have to look at how we transitioned from stateless chat completions to the stateful tool-use loop—and why local agent state machines are living rent-free in my mind right now.

---

### the architecture of the terminal agent loop (it's cooking)

When you chat with Claude on web, you are in a stateless request-response loop. You send a prompt, the model processes it, and it outputs markdown text. The model has zero agency; it cannot touch your filesystem.

Claude Code works differently. It is an agentic harness built on top of Anthropic’s Messages API, utilizing Tool Use (Function Calling) as its primary engine. When you run `claude` in your terminal, the local CLI client spins up a stateful loop that acts as an operating system interface for the LLM. 

Here is the raw state machine behind every single prompt you enter:

```
[User Prompt] 
     │
     ▼
┌──────────────┐
│  Agent LLM   │◄────────────────────────────────────────┐
└──────┬───────┘                                         │
       │ (Generates thought & tool request)              │
       ▼                                                 │
┌──────────────────────────────────────────────┐         │
│                 CLI Harness                  │         │
│  (Interprets tool call and runs OS command)  │         │
└──────┬───────────────────────────────────────┘         │
       │                                                 │
       ├───────────────┬──────────────────────┐          │
       ▼               ▼                      ▼          │
┌─────────────┐ ┌───────────────┐ ┌──────────────────┐   │
│ Read File   │ │ Write File    │ │ Run Bash Command │   │
└──────┬──────┘ └──────┬────────┘ └───────────┬──────┘   │
       │               │                      │          │ (Feeds Stdout/Stderr/File
       └───────────────┼──────────────────────┘          │  contents back as tool result)
                       ▼                                 │
              ┌─────────────────┐                        │
              │  Capture Output │────────────────────────┘
              └─────────────────┘
```

When you ask Claude Code to *“fix the build,”* the model doesn’t just write code. It generates a JSON payload requesting to run a tool, like `run_command` with the argument `npm run build`. The local CLI harness intercepts this JSON, spawns a child process in your local bash shell, captures the stdout/stderr, and packages it back into a new message sent to the API.

Because the model can loop this process indefinitely (until it decides to stop or hits a user-defined threshold), it can debug itself. If `npm run build` fails with a compiler error, the model reads the stack trace, requests to open the failing file, edits the broken line, and runs the build command again. It's basically a senior engineer on autopilot.

---

### context window optimization: tree-sitter and ripgrep (no cap)

One of the biggest technical challenges with terminal agents is context window management. If you dump a 100,000-line repository into the LLM context on every prompt, you will destroy your token budget and run into rate limits in minutes. Massive skill issue.

To solve this, Claude Code doesn’t just read your whole repo. It uses a selective ingestion pipeline powered by `ripgrep` and `tree-sitter`.

When you ask the agent a question, it first performs a semantic or keyword search across the codebase using a highly optimized ripgrep tool. Once it identifies target files, it doesn’t read them end-to-end. Instead, it uses `tree-sitter` to parse the files into an Abstract Syntax Tree (AST). This allows the agent to request only the specific class definitions or function bodies relevant to your query, keeping the token count extremely low. 

If it needs to edit a file, it doesn’t rewrite the whole file (that would be down bad for token efficiency). Instead, it uses a custom “edit block” format to perform search-and-replace actions on specific lines. It looks like this:

```markdown
<<<<<<< SEARCH
function calculateTotal(price, tax) {
  return price + tax;
}
=======
function calculateTotal(price, tax, discount = 0) {
  return (price + tax) - discount;
}
>>>>>>> REPLACE
```

The CLI harness parses this specific markdown block and applies the patch directly to the file on disk. This is a massive flex compared to web UIs where the model is forced to dump the entire modified file into a code block.

---

### the trust boundary: sandboxing or side-eye?

Ngl, letting an LLM execute arbitrary shell commands on your machine is a major security risk. When you run a terminal agent bare-metal, the agent has the exact same permissions as your user account. 

If the agent decides to run `rm -rf /` or download a malicious package via `curl`, your system will execute it without hesitation. Major side-eye.

To mitigate this, terminal agents rely on a strict trust boundary. In Claude Code, you are prompted to approve risky tool calls. But as developers, we get approval fatigue. If we click `Y` fifty times, we will eventually click it on the fifty-first time without reading the command. 

The actual technical solution to this is running the agent inside an isolated sandbox, such as a Docker container or a microVM (like Firecracker). By mounting your project directory as a read-write volume inside an isolated container, you can allow the agent to run commands, install npm packages, and edit files without giving it access to your host machine’s environment variables, SSH keys, or parent filesystem.

Here is what a secure sandbox layout looks like:

```
                  TRUST BOUNDARY
┌─────────────────────────────────┐      ┌───────────────────────────────┐
│          Host Machine           │      │       Docker Container        │
│                                 │      │                               │
│  ┌───────────────────────────┐  │      │  ┌─────────────────────────┐  │
│  │    CLI Harness (Client)   │──┼──────┼─►│     Project Directory   │  │
│  └─────────────┬─────────────┘  │      │  │    (Mounted Volume)     │  │
│                │                │      │  └─────────────────────────┘  │
│                ▼                │      │               ▲               │
│      ┌──────────────────┐       │      │               │ (Executes OS  │
│      │  Anthropic API   │       │      │  ┌────────────┴────────────┐  │
│      └──────────────────┘       │      │  │     Isolated Shell      │  │
│                                 │      │  │  (Runs tests, npm, etc) │  │
│                                 │      │  └─────────────────────────┘  │
└─────────────────────────────────┘      └───────────────────────────────┘
```

---

### the cognitive cost of zero-friction DX

Because terminal agents eliminate the physical steps of coding, they also bypass your brain’s natural gatekeepers. In a traditional workflow, your brain goes through a cycle:

1. **formulate**: Figure out what you want to write.
2. **translate**: Convert that thought into syntax.
3. **execute**: Type it out, run the server, check the log.
4. **reflect**: Look at the result and adjust.

Terminal agents compress steps 2 and 3 into a single execution step that happens in milliseconds. This changes your job from a writer to a diff-debugger. You are no longer writing the logic; you are auditing the logic generated by a machine. And auditing code is cognitively harder than writing it from scratch because you have to reconstruct the author’s mental model—even if the author is an LLM.

If you don’t slow down the agent or run it in a secure, sandboxed environment, you risk shipping code that compiles perfectly but contains subtle logical errors or security vulnerabilities that you missed in the high-speed diff review.

Terminal agents are the new meta, and they are here to stay. But as we transition from chat boxes to shell control, the developer’s role is shifting. We aren’t typing code anymore; we are orchestrating state machines.

Thanks for reading, and I'll see you in the next one!

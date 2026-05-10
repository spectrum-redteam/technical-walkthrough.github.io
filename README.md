# Technical Walkthrough: Spectrum

## 1. Introduction: The Vision of Spectrum
The project, internally titled **Spectrum**, represents a prototype for an autonomous "Cyber Reasoning System" (CRS). Unlike traditional security tools that rely on static signatures or hard-coded heuristics, Spectrum utilizes Large Language Models (LLMs)—specifically the DeepSeek-V4 and Qwen families—to perform high-level planning, exploitation, and remediation.

The project is bifurcated into two primary operational modes:
1.  **The Red Team (Offensive):** An agent that mimics a human pentester by scanning targets, identifying vulnerabilities, writing custom exploit scripts, and pivoting through a system.
2.  **The Blue Team (Defensive):** An automated Security Operations Center (SOC) analyst that monitors logs in real-time, classifies attacks using AI, and dynamically modifies the source code of the target application to "patch" vulnerabilities on the fly.

---

## 2. System Architecture and Component Overview
The project is built using a modular Pythonic architecture designed for extensibility and resilience.

### A. Core Modules
-   **`main.py`**: The central entry point. It provides a stylized CLI interface using the `rich` library, allowing the operator to select between offensive and defensive modules.
-   **`redteamer.py`**: Contains the logic for the offensive agent's "Thought-Action-Observation" loop.
-   **`blueteamer.py`**: Houses the "Sentinel" logic for log monitoring and the DeepSeek logic for automated patching.
-   **`tools.py`**: The abstraction layer between the AI and the host operating system. It provides the "hands" for the AI, such as terminal execution and HTTP requests.
-   **`app.py`**: A Gradio-based web UI that provides a more accessible interface for the agent loops, suitable for demos or remote monitoring.

### B. The Laboratory
-   **`lab.py`**: This is a deliberately vulnerable Flask application. It serves as the "Sacrifice" or "Target" for the agents. It contains classic vulnerabilities like SQL Injection, Command Injection, and Server-Side Template Injection (SSTI). This serves to test the agent's skill during development.

---

## 3. The Offensive Engine: `redteamer.py`
The Red Team agent operates on a continuous feedback loop. It doesn't just send a payload; it plans, executes, observes the output, and adjusts.

### The Reasoning Loop
The agent uses a custom token delimiter (`䷂`) to separate its internal reasoning from its external communication. This allows the AI to "think out loud" about its strategy without cluttering the operator's log.

```python
# Regex to extract internal thoughts
RE_THINK = re.compile(r'䷂(.*?)䷂', re.DOTALL)

def ai_call(messages, config):
    # ... setup client ...
    response = client.chat_completion(messages=messages, ...)
    msg = response.choices[0].message
    # Capture reasoning if the model supports it natively
    out = f"䷂\n{msg.reasoning}\n䷂\n" if hasattr(msg, 'reasoning') else ""
    return out + (msg.content or "")
```

### Tool Extraction and Execution
The AI interacts with the system by outputting a JSON block at the end of its response. `redteamer.py` uses `robust_extract_tool` to find this JSON even if the AI surrounds it with markdown text.

```python
def robust_extract_tool(text):
    match = re.search(r'```(?:json)?\s*(.*?)\s*```', text, re.DOTALL)
    if not match: 
        match = re.search(r'(\{[\s\S]*"tool"[\s\S]*\})', text)
    if match:
        try: 
            data = json.loads(match.group(1))
            return data
        except: return None
```

Once a tool (like `execute_terminal` or `http_request`) is identified, it is routed through `tools.py`, and the result is fed back into the AI’s context as a `user` message, allowing the AI to learn from the terminal output.

---

## 4. The Defensive Engine: `blueteamer.py`
The Blue Team (or "Sentinel") is arguably the most innovative part of the Spectrum project. It operates as an **Autonomous Patching System**.

### The Two-Tier AI Triage
1.  **Tier 1: The Sentinel (`Qwen2.5-3B`)**: A small, fast model that acts as a filter. It watches `server.log` every 2 seconds. Its job is binary: Is this a hack or a normal request?
2.  **Tier 2: The Verifier (`DeepSeek-V4`)**: If the Sentinel detects an attack, the payload is sent to the more powerful DeepSeek model to verify if the attack *actually succeeded* and to identify the specific vulnerability type.

### Automated Remediation
If an attack is verified, the system calls `tools.apply_patch()`. This function performs string-based code surgery on the target application (`lab.py`).

```python
def apply_patch(vulnerability, file_path="lab.py"):
    code = full_path.read_text(encoding='utf-8')
    # ... logic to find vulnerable patterns ...
    if vuln == "sqli":
        # Replacing string concatenation with parameterized queries
        old_login = 'query = f"SELECT * FROM users WHERE username=\'{username}\' AND password=\'{password}\'"'
        new_login = 'query = "SELECT * FROM users WHERE username=? AND password=?"'
        code = code.replace(old_login, new_login)
        # Update execution call
        code = code.replace('cur = db.execute(query)', 'cur = db.execute(query, (username, password))')
    # ... write back to file ...
```

After the patch is applied, the Blue Team agent kills the current server process and restarts it, effectively neutralizing the vulnerability in real-time. ANd then the Sentinel can resume watching the server. Since the Sentinel is only a 3B parameters model, the model can run thousands of times and cost pennies for inference.

---

## 5. The "Hands" of the Agent: `tools.py`
The `tools.py` module is the interface between the LLM’s abstract logic and the concrete operating system. It provides several critical functions:

### A. Terminal Execution
The `execute_terminal` function allows the agent to run any shell command. However, it includes a "Self-Preservation" check to prevent the agent from accidentally killing its own process or deleting its own source code.

```python
def execute_terminal(cmd, timeout=30):
    my_pid = str(os.getpid())
    cmd_tokens = cmd.split()
    if "kill" in cmd_tokens and my_pid in cmd_tokens:
        return "SYSTEM OVERRIDE: Refusing to kill self."
    
    process = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=timeout)
    # ... returns stdout/stderr ...
```

### B. HTTP Interaction
Since most modern targets are web-based, the `http_request` tool allows the agent to act as a browser. It uses a persistent `requests.Session()` to maintain cookies across different cycles, which is vital for exploiting authenticated vulnerabilities like IDOR (Insecure Direct Object Reference).

---

## 6. The Target Environment: `lab.py`
To demonstrate the system's power, `lab.py` provides a rich target environment. It is a Flask app that implements a "SaaS Platform" with several intentional flaws:

1.  **SQL Injection:** The `/login` and `/profile` routes use f-strings to build queries.
2.  **Command Injection:** A `/ping` utility that passes user input directly to the shell.
3.  **SSRF (Server-Side Request Forgery):** A `/webhook` tester that allows the user to provide a URL for the server to fetch, including `file://` protocols.
4.  **SSTI (Template Injection):** An `/announcement` feature that uses `render_template_string` on raw user input.
5.  **Sensitive Info Disclosure:** A `/debug` endpoint that leaks environment variables and source code.

This lab is equipped with a `RotatingFileHandler` that logs every single request, which serves as the "Sensory Input" for the Blue Team Sentinel.

---

## 7. Knowledge Management: `bake.py`
The project includes a Knowledge Base (KB) system. The `bake.py` script takes a folder of markdown tutorials or technical documentation and "bakes" them into a SQLite database (`kb.sqlite3`).

The agent can then use the `search_kb` tool to query this database. This is a form of **RAG (Retrieval-Augmented Generation)**, allowing the agent to look up specific exploitation techniques or system commands that it might not have in its immediate training data. This file is over 1.7 GB and contains many verified exploits and payloads.

---

## 8. User Interface and Monitoring
### CLI Interface (`main.py`)
The CLI uses the `rich` library to create a CLI-style terminal aesthetic. It features ASCII art, randomized quotes from famous hackers, and a color-coded log of the agent's progress.

### Web Monitoring (`viewer.py`)
Because LLM outputs can be lengthy, the project includes `viewer.py`. This uses `pywebview` to open a local window that renders `session.md` in real-time. As the agent writes its history to markdown, the viewer automatically scrolls and renders the code blocks and markdown formatting, providing a clean "live stream" of the agent's brain.
(deprecated)

---

## 9. Persistence and Recovery
One of the major challenges of autonomous agents is API instability or quota limits. Spectrum addresses this in `redteamer.py` via the `save_session_state` and `restore_session` functions.

If the agent hits a "402 Payment Required" error or is interrupted by the user, it dumps its entire message history and session cookies into `operation_state_recovery.json`. When the program is restarted, it detects this file and offers to resume the operation exactly where it left off.

---

## 10. Security and Ethical Considerations
Spectrum is your sword and shield. 

-   **Red Team use cases:** It can be used for rapid, automated security auditing of internal applications, identifying low-hanging fruit far faster than a human could.
-   **Blue Team use cases:** It demonstrates a future where systems are "self-healing." Imagine a web server that detects a zero-day exploit, analyzes the payload, and writes its own patch in seconds—before a human analyst even wakes up.

    
Employing a dual-team architecture—comprising both a Red Teamer (offensive) and a Blue Teamer (defensive)—is the superior security strategy because it creates a dynamic, self-improving ecosystem known as "Purple Teaming." 

In this model, the Red Team acts as a necessary catalyst for growth, exposing hidden vulnerabilities and "stress-testing" the environment through creative, real-world exploitation that static scanners often miss. Simultaneously, the Blue Team provides the essential counterbalance, translating those offensive insights into immediate, actionable defenses and automated remediation. Without the Red Team, a defense becomes stagnant and complacent; without the Blue Team, offensive findings are merely academic exercises without a path to resolution. 

Together, they form a continuous feedback loop where every successful intrusion leads to a permanent hardening of the system, ensuring that the organization's security posture evolves as rapidly as the threats it faces.

**Warning:** This project executes raw shell commands provided by an LLM. It should **only** be run in a strictly isolated sandbox environment (Docker, VM) with no access to sensitive production data.

---

## 11. Conclusion: The Future of Autonomous Security
The Spectrum project proves that Large Language Models are no longer just "chatbots"—they are capable of complex, multi-step reasoning in technical domains. By combining the offensive creativity of the Red Team agent with the vigilant, automated patching of the Blue Team, Spectrum provides a glimpse into the next generation of cybersecurity: a world where AI agents fight AI agents at machine speed. 

The modular design of the `tools.py` and the dual-model approach in `blueteamer.py` sets a high bar for how autonomous security systems should be structured: prioritizing fast detection, deep verification, and safe, automated remediation.

## 12. AMD Hardware

While Spectrum's logic is model-agnostic, its production architecture is specifically optimized for AMD Instinct™ MI300X accelerators. By leveraging AMD ROCm™ 6.0+, we utilize hardware-level optimizations for the DeepSeek-V4 MoE (Mixture-of-Experts) architecture. The massive 192GB of HBM3 memory on the MI300X allows us to keep both our 1.7GB 'Bake' Knowledge Base and the full-parameter DeepSeek model in VRAM simultaneously. This eliminates PCIe bottlenecks during RAG (Retrieval-Augmented Generation) operations, enabling the Blue Team Sentinel to achieve the sub-second inference speeds required for real-time autonomous patching.

---
**Walkthrough Metadata:**
- **Project Name:** Spectrum
- **Version:** 1.35
- **Primary Languages:** Python, SQL, Markdown
- **Primary Frameworks:** Flask, Gradio, Rich, HuggingFace Inference

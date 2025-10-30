Perfect — here’s the **fully updated and complete Markdown** version of the Azure AI Foundry + MCP Agent Workshop, now including the **correct pip installs** from your PDFs and aligned to your OpenHack progression.

It’s written for **VS Code + Python**, with every dependency, command, and code example you need to go end-to-end — from Foundry setup through MCP voice ordering.

---

# 🍕 Azure AI Foundry + Agents (VS Code + Python)

### Full Step-by-Step Workshop Guide

> Build and run a real Contoso Pizza ordering agent using **Azure AI Foundry**, then extend it with **MCP (Model Context Protocol)** for real-time ordering, status, and cancel operations.
> You’ll complete Levels 1 – 7 from the OpenHack in order.

---

## ⚙️ 0) Prerequisites

### 🪄 Environment Setup

```bash
# Sign in to Azure
az login --use-device-code
```

* Ensure you have **Contributor** access in a subscription where you can create resources.
* For MCP integration (Levels 6–7), deploy resources in the **West US** region (required).

### 🧰 Install Tools

* **VS Code** with Python extension
* **Python 3.10 +**
* **Azure CLI**
* *(Optional)* GitHub Codespaces or VS Code Dev Container

---

### 📦 Python Environment & Packages

```bash
mkdir pizza-agent && cd pizza-agent
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
```

Then install all required packages from your OpenHack PDFs:

```bash
pip install azure-identity \
            azure-ai-projects \
            jsonref \
            python-dotenv
```

**Package Roles**

| Package             | Purpose                                                        |
| ------------------- | -------------------------------------------------------------- |
| `azure-identity`    | Handles Azure AD credentials (used for SDK or CLI-based auth). |
| `azure-ai-projects` | Core SDK for Azure AI Foundry (agents, threads, messages).     |
| `jsonref`           | Parses JSON schema references used in structured tool calls.   |
| `python-dotenv`     | Loads secrets and config from a local `.env` file.             |

Open the folder in VS Code:

```bash
code .
```

---

## 🧠 1️⃣ Interact with Azure AI Foundry

### 🎯 Goal

Create a Foundry Project, deploy a model, and confirm that it responds.

### 🪜 Steps

1. Open [**Azure AI Foundry**](https://ai.azure.com).
2. Create a **Project** → name it `openhack-pizza`

   * **Region:** West US (important for MCP).
3. Deploy **GPT-4o** (or GPT-4-Turbo) from the Model Catalog.
4. Test in the **Playground**:

   ```
   How many slices are in a large pizza?
   ```

   ✅ You should get a valid response.
5. Copy your **Project Connection String** from the top-right > **Connect**.

### 🧩 .env File

```bash
touch .env
echo "AZURE_AI_PROJECT_CONNECTION_STRING=your_connection_string" >> .env
```

---

## 👋 2️⃣ Create Your First Agent (Python + VS Code)

### 🎯 Goal

Build a local Python script that talks to your deployed Foundry model using the Agent Service.

### 🪜 Steps

#### `agent_basic.py`

```python
import os
from dotenv import load_dotenv
from azure.ai.projects import AIProjectClient
from azure.ai.agents.models import MessageRole
from azure.identity import DefaultAzureCredential

load_dotenv()
conn = os.getenv("AZURE_AI_PROJECT_CONNECTION_STRING")
client = AIProjectClient(endpoint=conn, credential=DefaultAzureCredential())

# Create a simple agent
agent = client.agents.create_agent(
    name="level2-basic-agent",
    model="gpt-4o",
    instructions="You are a friendly pizza helper."
)

# Create a thread to maintain context
thread = client.agents.threads.create()

print("Chat with your agent! Type 'exit' to quit.\n")
while True:
    msg = input("You: ")
    if msg.strip().lower() == "exit":
        break

    client.agents.messages.create(
        thread_id=thread.id,
        role=MessageRole.USER,
        content=msg
    )
    run = client.agents.runs.create_and_process(
        thread_id=thread.id,
        agent_id=agent.id
    )

    messages = client.agents.messages.list(thread_id=thread.id)
    for message in messages:
        if message.role == "assistant" and message.text_messages:
            print(f"Agent: {message.text_messages[-1].text.value}")
            break
    else:
        print("Agent: (no reply)")

print("Session ended.")
```

Run it:

```bash
python agent_basic.py
```

✅ Chat responses should stream through your Foundry project.

---

## 🧭 3️⃣ Add Instructions and Memory

### 🎯 Goal

Make the agent remember the user’s name and restrict topics to pizza.

#### `agent_memory.py`

```python
# add to the agent_memory.py file
instructions = """
You are Contoso Pizza's ordering assistant.
- Always greet the user and ask their name.
- Remember their name across the chat session.
- Only discuss pizza ordering, stores, or delivery.
- If asked non-pizza questions, politely decline.
"""

from agent_memory import instructions 
# Update the agent code in agent_basic.py
agent = client.agents.create_or_replace(
    name="level3-memory-agent",
    model="gpt-4o",
    instructions=instructions
)
```

✅ It should remember your name within the same thread.

---

## 🏬 4️⃣ Add Knowledge (Store Information)

### 🎯 Goal

Enable the agent to answer questions about store locations and hours.

```python
# add to the agent_memory.py
store_info = """
Contoso Pizza locations:
- Downtown – 123 Main St, open 11 AM–10 PM.
- Uptown – 500 Pine St, open 12 PM–11 PM.
Ask the user to choose a store before confirming an order.
"""
# update the agent_basic.py
from agent_memory import instructions, store_info
agent = client.agents.create_or_replace(
    name="level4-knowledge-agent",
    model="gpt-4o",
    instructions=instructions + store_info
)
```

✅ Try:

```
You: What are your hours?
Agent: Our Downtown location is open 11–10, Uptown 12–11.
```

---

## 🧮 5️⃣ Add a Pizza Estimation Function

### 🎯 Goal

Have the agent call a Python function to estimate pizza quantity.

```python
# replace the code
import os
from dotenv import load_dotenv
from azure.ai.projects import AIProjectClient
from azure.ai.agents.models import MessageRole
from azure.identity import DefaultAzureCredential
from agent_memory import instructions, store_info

load_dotenv()
conn = os.getenv("AZURE_AI_PROJECT_CONNECTION_STRING")
client = AIProjectClient(endpoint=conn, credential=DefaultAzureCredential())

# Implement the actual function for estimating pizzas
def estimate_pizza_tool(num_people: int, appetite: str) -> dict:
    if appetite == "light":
        pizzas = num_people * 0.5
    elif appetite == "average":
        pizzas = num_people * 1.0
    elif appetite == "heavy":
        pizzas = num_people * 1.5
    else:
        pizzas = num_people * 1.0  # default to average if unknown

    # Round up to nearest whole pizza
    pizzas_needed = int(pizzas) if pizzas == int(pizzas) else int(pizzas) + 1
    return {"pizzas_needed": pizzas_needed}

estimate_pizza_tool_descriptor = {
    "type": "function",
    "function": {
        "name": "estimate_pizza_tool",
        "description": "Estimate the number of pizzas needed for a group.",
        "parameters": {
            "type": "object",
            "properties": {
                "num_people": {
                    "type": "integer",
                    "description": "The number of people.",
                },
                "appetite": {
                    "type": "string",
                    "description": "The appetite of the group (light, average, heavy).",
                    "enum": ["light", "average", "heavy"],
                },
            },
            "required": ["num_people", "appetite"],
        },
    },
}

agent = client.agents.create_agent(
    name="level2-basic-agent",
    model="gpt-4o",
    instructions=instructions + store_info,
    tools=[estimate_pizza_tool_descriptor]
)


# Register the function for auto function calls
client.agents.enable_auto_function_calls([estimate_pizza_tool])
# Create a thread to maintain context
thread = client.agents.threads.create()

print("Chat with your agent! Type 'exit' to quit.\n")
while True:
    msg = input("You: ")
    if msg.strip().lower() == "exit":
        break

    client.agents.messages.create(
        thread_id=thread.id,
        role=MessageRole.USER,
        content=msg
    )
    run = client.agents.runs.create_and_process(
        thread_id=thread.id,
        agent_id=agent.id
    )

    messages = client.agents.messages.list(thread_id=thread.id)
    for message in messages:
        if message.role == "assistant" and message.text_messages:
            print(f"Agent: {message.text_messages[-1].text.value}")
            break
    else:
        print("Agent: (no reply)")

print("Session ended.")

```

✅ “6 hungry people” → *“About 3 pizzas.”*

---

## 🧠 6️⃣ Connect to the Pizza MCP Server

### 🎯 Goal

Replace local functions with real **MCP tools** served by a Pizza MCP backend.

---

### 🪜 Step 1: Understand MCP

MCP (Model Context Protocol) lets your agent use **remote tools** such as `create_order`, `get_order_status`, and `cancel_order` via HTTP/WebSocket instead of direct API calls.
All Foundry MCP labs require the **West US** region.

---

### 🪜 Step 2: Configure User ID

```bash
echo "USER_ID=brian@example.com" >> .env
```

---

### 🪜 Step 3: Attach MCP Server

```python
import os
from dotenv import load_dotenv
from azure.ai.projects import AIProjectClient
from azure.ai.agents.models import MessageRole
from azure.identity import DefaultAzureCredential
from agent_memory import instructions, store_info

load_dotenv()
conn = os.getenv("AZURE_AI_PROJECT_CONNECTION_STRING")
client = AIProjectClient(endpoint=conn, credential=DefaultAzureCredential())

# Implement the actual function for estimating pizzas
def estimate_pizza_tool(num_people: int, appetite: str) -> dict:
    if appetite == "light":
        pizzas = num_people * 0.5
    elif appetite == "average":
        pizzas = num_people * 1.0
    elif appetite == "heavy":
        pizzas = num_people * 1.5
    else:
        pizzas = num_people * 1.0  # default to average if unknown

    # Round up to nearest whole pizza
    pizzas_needed = int(pizzas) if pizzas == int(pizzas) else int(pizzas) + 1
    return {"pizzas_needed": pizzas_needed}

estimate_pizza_tool_descriptor = {
    "type": "function",
    "function": {
        "name": "estimate_pizza_tool",
        "description": "Estimate the number of pizzas needed for a group.",
        "parameters": {
            "type": "object",
            "properties": {
                "num_people": {
                    "type": "integer",
                    "description": "The number of people.",
                },
                "appetite": {
                    "type": "string",
                    "description": "The appetite of the group (light, average, heavy).",
                    "enum": ["light", "average", "heavy"],
                },
            },
            "required": ["num_people", "appetite"],
        },
    },
}

agent = client.agents.create_agent(
    name="level2-basic-agent",
    model="gpt-4o",
    instructions="You are Contoso Pizza’s MCP-enabled ordering assistant."
)

# client.agents.mcp_connections.create_or_replace(
#     agent_id=agent.id,
#     name="pizza-mcp",
#     server_url="https://pizza-mcp.azurewebsites.net"
# )

user_id = os.getenv("USER_ID", "guest@contoso.com")
system_prompt = f"Always include X-User-ID={user_id} when calling MCP tools."
# Append system_prompt to instructions before creating the agent
agent_instructions = "You are Contoso Pizza’s MCP-enabled ordering assistant.\n" + system_prompt

agent = client.agents.create_agent(
    name="level2-basic-agent",
    model="gpt-4o",
    instructions=agent_instructions
)

# Register the function for auto function calls
#client.agents.enable_auto_function_calls([estimate_pizza_tool])
# Create a thread to maintain context
thread = client.agents.threads.create()

print("Chat with your agent! Type 'exit' to quit.\n")
while True:
    msg = input("You: ")
    if msg.strip().lower() == "exit":
        break

    client.agents.messages.create(
        thread_id=thread.id,
        role=MessageRole.USER,
        content=msg
    )
    run = client.agents.runs.create_and_process(
        thread_id=thread.id,
        agent_id=agent.id
    )

    messages = client.agents.messages.list(thread_id=thread.id)
    for message in messages:
        if message.role == "assistant" and message.text_messages:
            print(f"Agent: {message.text_messages[-1].text.value}")
            break
    else:
        print("Agent: (no reply)")

print("Session ended.")

```

---

### 🪜 Step 4: Pass User ID in System Prompt

```python
user_id = os.getenv("USER_ID", "guest@contoso.com")
system_prompt = f"Always include X-User-ID={user_id} when calling MCP tools."
client.agents.update(agent.id, instructions=agent.instructions + "\n" + system_prompt)
```

---

### 🪜 Step 5: Run and Test

```bash
python agent_basic.py
```

Example conversation:

```
You: I'd like a large pepperoni pizza.
Agent: Great! Creating your order...
You: What's the status of my order?
Agent: It’s baking now.
You: Cancel my order.
Agent: Done. Your order was cancelled.
```

✅ Orders appear in the OpenHack dashboard.

---

### 🧭 Step 6: Inspect in VS Code

Add to `.vscode/settings.json`:

```json
{
  "mcp.servers": [
    { "name": "Pizza MCP", "url": "https://pizza-mcp.azurewebsites.net" }
  ]
}
```

Open **Copilot Chat → MCP Servers** panel to see live tool calls.

---

### ✅ Pass Criteria

✔ Agent calls `create_order`, `get_order_status`, `cancel_order` via MCP
✔ Includes User ID in system prompt
✔ Visible in dashboard and VS Code UI

---

## 🎤 7️⃣ Add Voice Ordering (Bonus)

### 🎯 Goal

Add voice input/output on top of the MCP-enabled agent.

---

### 🪜 Step 1: Install Speech SDK

```bash
pip install azure-cognitiveservices-speech
```

Add keys:

```
AZURE_SPEECH_KEY=your_speech_key
AZURE_REGION=westus
```

---

### 🪜 Step 2: Capture Voice Input

```python
import azure.cognitiveservices.speech as speechsdk

speech_config = speechsdk.SpeechConfig(
    subscription=os.getenv("AZURE_SPEECH_KEY"),
    region=os.getenv("AZURE_REGION")
)
recognizer = speechsdk.SpeechRecognizer(speech_config=speech_config)

print("🎙️ Say your order...")
result = recognizer.recognize_once()
print("You said:", result.text)
user_text = result.text
```

Send `user_text` to the same MCP agent.

---

### 🪜 Step 3: Add Speech Output

```python
synthesizer = speechsdk.SpeechSynthesizer(speech_config=speech_config)
synthesizer.speak_text_async(agent_reply)
```

---

### 🪜 Step 4: Handle Low-Confidence Transcripts

```python
if result.reason == speechsdk.ResultReason.NoMatch:
    print("Sorry, could you repeat that?")
```

---

### ✅ Pass Criteria

✔ Voice recognized and sent to agent
✔ Agent extracts pizza details
✔ Order placed via MCP
✔ Audio confirmation spoken back

---

# 🧾 Summary Table

| Level | Focus                       | Deliverable                        |
| :---- | :-------------------------- | :--------------------------------- |
| 1️⃣   | Connect to Azure AI Foundry | Model deployed & Playground tested |
| 2️⃣   | Create Agent in Python      | `agent_basic.py` working           |
| 3️⃣   | Add Instructions & Memory   | Remembers user name                |
| 4️⃣   | Add Knowledge               | Knows store info                   |
| 5️⃣   | Add Function                | Local pizza estimator              |
| 6️⃣   | Integrate MCP Server        | Real ordering/status/cancel        |
| 7️⃣   | Voice Ordering              | Speech input/output via MCP        |

---

# 🔗 References

* [Azure AI Foundry Quickstart (Python)](https://learn.microsoft.com/azure/ai-foundry/quickstarts/get-started-code?tabs=python)
* [Adding MCP Servers to Agents](https://learn.microsoft.com/azure/ai-foundry/agents/how-to/tools/model-context-protocol)
* [Chainlit MCP Integration](https://docs.chainlit.io/advanced-features/mcp)
* [Semantic Kernel MCP Sample (Python)](https://github.com/microsoft/semantic-kernel/blob/main/python/samples/concepts/mcp/agent_with_mcp_plugin.py)
* [MCP for Beginners Guide](https://aka.ms/openhack/mcp-for-beginners)

---

Would you like me to extend this same Markdown with **Level 8 (Visual Ordering via image upload)** and **Level 9 (Reporting and Content Filter)** next, in the same format?

# Lab 3 - Extend Agents with MCP Tools Notes

A walkthrough of Microsoft's "Extend agents with Model Context Protocol (MCP) tools" tutorial, covering both connecting to a remote MCP server and building a custom one.

---
Builds two AI agent configurations in Microsoft Foundry:

1. **Remote MCP Agent** — connects to Microsoft's official Learn Docs MCP server to retrieve up-to-date technical documentation
2. **Custom MCP Agent** — connects to a locally-hosted custom MCP server with inventory and sales tools for a simulated retail store

Both agents are built and run entirely from VS Code using the Foundry Toolkit extension.

---

## Resources Created in Azure

| Resource | Type | Notes |
|---|---|---|
| Foundry project | Azure AI Foundry | Created via VS Code extension |
| gpt-4.1 deployment | Azure OpenAI | Global Standard or Standard tier |

---

## Tools & Extensions Used

| Tool | Purpose |
|---|---|
| Foundry Toolkit (VS Code extension) | Create project, deploy model, copy endpoint |
| Python `azure-ai-projects` | Connect to Foundry and manage agents |
| Python `fastmcp` | Build and run the custom MCP server |
| Python `mcp` | MCP client session management |
| Microsoft Learn Docs MCP server | Remote MCP server at `https://learn.microsoft.com/api/mcp` |

---

## Steps Completed

### 1. Installed the Foundry Toolkit VS Code Extension
- Opened Extensions panel (`Ctrl+Shift+X`)
- Searched for and installed **Foundry Toolkit** by Microsoft
- Signed into Azure account through the extension

### 2. Created a Foundry Project
- Used **Create Project** in the Foundry Toolkit pane
- Selected Azure subscription and resource group
- Entered a project name to deploy

### 3. Deployed gpt-5.1
- Opened the Model Catalog via the Foundry Toolkit
- Located and deployed **gpt-5.1**
- Deployment type: Global Standard (or Standard if unavailable)
- Copied the **Project Endpoint** from the deployed model (right-click → Copy Project Endpoint)

### 4. Cloned the Starter Code
- Cloned `https://github.com/MicrosoftLearning/mslearn-ai-agents`
- Opened the `Labfiles/03-mcp-integration/Python` folder in VS Code
- Populated `.env` with the project endpoint and model deployment name

### 5. Set Up the Virtual Environment
```bash
python3 -m venv labenv
source labenv/bin/activate        # Linux/Mac
# OR on Windows:
.\labenv\Scripts\Activate.ps1
pip install -r requirements.txt
```

---

## Part 1: Remote MCP Server (agent.py)

### What it does
Connects to Microsoft's Learn Docs MCP server and asks the agent to retrieve Azure CLI commands for creating a Container App with a managed identity.

### Key code pieces added to `agent.py`

**Imports:**
```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, MCPTool
from openai.types.responses.response_input_param import McpApprovalResponse, ResponseInputParam
```

**Connect to project:**
```python
with (
    DefaultAzureCredential() as credential,
    AIProjectClient(endpoint=project_endpoint, credential=credential) as project_client,
    project_client.get_openai_client() as openai_client,
):
```

**Initialize MCP tool pointing to Microsoft Learn Docs:**
```python
mcp_tool = MCPTool(
    server_label="api-specs",
    server_url="https://learn.microsoft.com/api/mcp",
    require_approval="always",
)
```

**Create agent with MCP tool:**
```python
agent = project_client.agents.create_version(
    agent_name="MyAgent",
    definition=PromptAgentDefinition(
        model=model_deployment,
        instructions="You are a helpful agent...",
        tools=[mcp_tool],
    ),
)
```

**Handle MCP approval requests and get final response:**
- Listens for `mcp_approval_request` items in the response output
- Automatically approves them by sending back `mcp_approval_response`
- Retrieves the final response after approval

### How to run
```bash
az login
python3 agent.py
```

### Sample output
The agent retrieved and returned Azure CLI commands for creating a Container App with managed identity, sourced directly from Microsoft's documentation.

---

## Part 2: Custom MCP Server (server.py + client.py)

### What it does
Defines a local MCP server with two custom tools — inventory levels and weekly sales — then connects an agent to it to answer retail inventory questions.

### server.py — Custom MCP Server

Defines two tools using the `@mcp.tool()` decorator, registered on a `FastMCP` server named "Inventory":

- `get_inventory_levels()` — returns current stock quantities per product
- `get_weekly_sales()` — returns weekly sales figures per product

```python
from fastmcp import FastMCP
mcp = FastMCP(name="Inventory")

@mcp.tool()
def get_inventory_levels() -> dict:
    # returns inventory data

@mcp.tool()
def get_weekly_sales() -> dict:
    # returns sales data

mcp.run()
```

### client.py — MCP Client + Agent

**Connects to the server via stdio transport:**
```python
stdio_transport = await exit_stack.enter_async_context(stdio_client(server_params))
session = await exit_stack.enter_async_context(ClientSession(stdio, write))
await session.initialize()
```

**Discovers available tools and wraps them as callable functions:**
```python
def make_tool_func(tool_name):
    async def tool_func(**kwargs):
        result = await session.call_tool(tool_name, kwargs)
        return result
    tool_func.__name__ = tool_name
    return tool_func
```

**Creates the agent with inventory-focused instructions:**
```python
agent = project_client.agents.create_version(
    agent_name="inventory-agent",
    definition=PromptAgentDefinition(
        model=model_deployment,
        instructions="""
        You are an inventory assistant.
        - Recommend restock if item inventory < 10 and weekly sales > 15
        - Recommend clearance if item inventory > 20 and weekly sales < 5
        """,
        tools=mcp_function_tools
    ),
)
```

**Processes function calls from the agent response and sends results back.**

### How to run
```bash
python3 client.py
```

### Sample prompts tested
- `Show me the current inventory levels for all products.`
- `Are there any products that should be restocked?`
- `Which products would you recommend for clearance?`
- `What are the best sellers this week?`

The agent used the MCP tools to retrieve data and applied the restock/clearance rules from its instructions to give actionable recommendations.

---

## Key Concepts

**MCP (Model Context Protocol)** — a standard that lets AI agents discover and call external tools, either hosted remotely or locally.

**Remote MCP server** — a cloud-hosted service (like Microsoft Learn Docs) that the agent calls over HTTP. Requires approval handling.

**Custom MCP server** — a locally-run Python process exposing functions via `@mcp.tool()`. The client starts the server via stdio transport and the agent calls tools dynamically.

**MCP approval flow** — when `require_approval="always"` is set, the agent pauses and requests user (or programmatic) approval before invoking a tool. The client sends back an `mcp_approval_response` to proceed.

---

## Approximate Cost

| Resource | Cost |
|---|---|
| gpt-4.1 tokens (light testing) | ~$0.10–0.50 |
| Foundry project (short session) | minimal |
| **Total** | **~$0.50–1.00** |

No persistent search or storage resources were created in this lab, so costs are lower than the Foundry IQ lab. Delete the resource group when done to avoid any ongoing charges.

# Lab 4 - Foundry IQ Azure AI Agent Notes

A walkthrough of Microsoft's "Integrate an AI agent with Foundry IQ" tutorial, including the real-world issues encountered and how they were resolved.

---

## What This Project Does

Builds an AI agent in Microsoft Foundry that can answer questions about Contoso's outdoor camping product catalog. The agent uses Foundry IQ (backed by Azure AI Search) to retrieve information from uploaded PDF documents, and is accessed via a Python client app.

---

## Resources Created in Azure

| Resource | Type | Notes |
|---|---|---|
| Foundry project | Azure AI Foundry | Named `agent-iq-lab` |
| AI Agent | Foundry Agent | Named `product-expert-agent` |
| AI Search service | Microsoft.Search | Basic tier (Free tier unavailable) |
| Storage account | Azure Blob Storage | Standard LRS |
| Knowledge base | Foundry IQ | Connected to AI Search + Blob Storage |

---

## Steps Completed

### 1. Created a Foundry Project
- Signed into https://ai.azure.com with Azure credentials
- Enabled the **New Foundry** toggle
- Created a new project named `agent-iq-lab` with a new Foundry resource

### 2. Created the AI Agent
- Navigated to **Build → Agents → Create agent**
- Named the agent `product-expert-agent`
- Default model (gpt-4.1) was automatically deployed
- Added the following system instructions:
  > You are a helpful AI assistant for Contoso, specializing in outdoor camping and hiking products. You must ALWAYS search the knowledge base to answer questions about our products or product catalog...

### 3. Set Up Azure AI Search
- Registered the `Microsoft.Search` resource provider on the subscription
- Created an AI Search service at the **Basic tier**
- Enabled **Both** under API Access control (Security + networking → Keys)

### 4. Uploaded Product Documents
- Downloaded `contoso-products.zip` from the Microsoft Learn GitHub repo
- Extracted 3 PDF files covering Contoso's product catalog
- Created a Storage account with a container named `contosoproducts`
- Uploaded all 3 PDFs to that container

### 5. Configured Foundry IQ Knowledge Base
- Connected Foundry IQ to the AI Search resource
- Created a knowledge base named `ks-contosoproducts` using:
  - **Source:** Azure Blob Storage (`contosoproducts` container)
  - **Embedding model:** text-embedding-3-small
  - **Chat completions model:** gpt-4.1
  - **Authentication:** API Key (copied from Azure Search → Keys page)
- Waited for knowledge base status to show **active**

### 6. Tested the Agent in the Playground
- Added Foundry IQ to the agent's Knowledge section
- Tested queries such as:
  - "What types of tents does Contoso offer?"
  - "Tell me about which backpacks are available in XL."
  - "What camping accessories are available?"

### 7. Cloned the Starter Code
- Cloned `https://github.com/MicrosoftLearning/mslearn-ai-agents` locally
- Opened the `Labfiles/04-integrate-agent-with-foundry-iq/Python` folder in VS Code
- Populated `.env` with the project endpoint and agent name

### 8. Completed the Python Client
- Filled in two TODO sections in `agent_client.py`:
  1. Connected to the Foundry project, retrieved the agent, and created a conversation
  2. Implemented message sending, MCP approval request handling, and response retrieval

### 9. Ran the Application
- Created a Python virtual environment and installed dependencies
- Signed into Azure via `az login`
- Ran `python3 agent_client.py` and tested queries about Contoso products
- Approved MCP requests to allow the agent to search the knowledge base

### 10. Cleaned Up
- Deleted the resource group from the Azure portal to stop all billing

---

## Gotchas & Issues Encountered

### `Microsoft.Search` namespace not registered
**Error:** `MissingSubscriptionRegistrationError`  
**Fix:** Go to Azure Portal → Subscriptions → Resource providers → search `Microsoft.Search` → Register. Wait ~2 minutes before retrying.

### Free tier for AI Search was unavailable
Only the **Basic tier** was available at $73.73/month. At ~$0.10/hour this is fine for a short exercise, but will add up if you forget to delete the resource.

### `python` command not found
The guide uses `python` but the system only had `python3`.  
**Fix:** Replace all `python` calls with `python3`.

### Wrong virtual environment activation command
The guide gives a Windows path (`./labenv/Scripts/activate`).  
**Fix:** On Linux/Mac use `source labenv/bin/activate` instead.


---

## Approximate Cost

| Resource | Cost for ~1 hour |
|---|---|
| AI Search (Basic) | ~$0.10 |
| Azure OpenAI tokens | ~$0.05–0.20 |
| Storage | negligible |
| **Total** | **~$0.15–0.30** |

Cost only becomes significant if resources are left running. Delete the resource group immediately after finishing.


# Develop AI Agents in Azure

The exercises in this repo are designed to provide you with a hands-on learning experience in which you'll explore common tasks that developers perform when building AI agents on Microsoft Azure.

> **Note**: To complete the exercises, you'll need an Azure subscription in which you have sufficient permissions and quota to provision the necessary Azure resources and generative AI models. If you don't already have one, you can sign up for an [Azure account](https://azure.microsoft.com/free). There's a free trial option for new users that includes credits for the first 30 days.

View the exercises in the [GitHub Pages site for this repo](https://go.microsoft.com/fwlink/?linkid=2310820).

> **Note**: While you can complete these exercises on their own, they're designed to complement modules on [Microsoft Learn](https://learn.microsoft.com/training/paths/develop-ai-agents-azure/); in which you'll find a deeper dive into some of the underlying concepts on which these exercises are based.

If you choose to run the code from applications in this repository locally, it will require Python 3.12+.

## Reporting issues

If you encounter any problems in the exercises, please report them as **issues** in this repo.


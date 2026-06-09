# Foundry IQ Azure AI Agent — Lab Notes

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


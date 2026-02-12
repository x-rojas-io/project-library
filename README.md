# Search AI: Evaluation & Maintenance Strategy

![Architecture](https://img.shields.io/badge/Architecture-RAG-blue)
![Model](https://img.shields.io/badge/Model-Phi--3--mini-green)
![Status](https://img.shields.io/badge/Status-Production--Ready-brightgreen)

## 1. System Overview
The **Search AI** feature implements a **Retrieval-Augmented Generation (RAG)** architecture. It is designed to bridge the gap between traditional rigid database queries and natural language intent, solving the limitations of legacy keyword-based search.



---

## 2. Problem Statement: Why AI?

### The Legacy "Fragile Search"
The application functions as a **Universal Search**. Users query various entities—**Tasks, Users, Notes, Attachments, or Project Metadata**—and the system must return the **Project** associated with that artifact.

Previously, this relied on a massive synchronous SQL query joining 5+ volatile tables. This architecture had critical failures:
* **Complexity Explosion:** The query complexity was $O(N \times M \times \dots)$, where $N$ represents tasks (200k+).
* **Semantic Blindness:** Relied on rigid string matching (`LIKE '%query%'`). Searching for "Design files" would miss a Project containing only an attachment named `mockup.png`.
* **Catastrophic Failure:** On complex or "nonsense" queries, the database would lock up or timeout, causing the entire application to crash.

### The AI Solution (RAG)
We replaced "Smart SQL" with a **Language Model Reasoning Layer**:
1.  **Reasoning Layer:** The LLM interprets intent (e.g., "Projects running late") and correlates it with provided context (e.g., `Status: Red`).
2.  **Graceful Failure:** If SQL returns zero results, the LLM politely informs the user: *"I couldn't find any projects matching those criteria. Could you try checking the project name?"*
3.  **Performance:** We simplified the backend to perform fast, indexed lookups on just the `Project` table, leaving the heavy synthesis to the AI.

---

## 3. Measuring Accuracy
In a RAG system, accuracy is measured in two distinct stages:

### A. Retrieval Accuracy (Recall@K)
* **Metric:** `Recall@5`
* **Definition:** How often is the correct project found in the top 5 search results sent to the LLM?
* **Current Method:** Manual spot-checking.
* **Future Automation:** Create a **"Golden Dataset"** of 50 common queries and their expected Project IDs to verify the `search_projects_context` endpoint.

### B. Generation Faithfulness
* **Metric:** `Hallucination Rate`
* **Definition:** Frequency of the LLM inventing facts not present in the SQL data.
* **Mitigation:** The System Prompt explicitly instructs: *"If the context doesn't answer it, say so."*

---

## 4. Retraining Strategy

### Why we do NOT fine-tune the Model
The **Phi-3-mini** model is a "frozen" inference engine. We do not fine-tune its weights because:
* **Cost:** Fine-tuning requires expensive GPU clusters.
* **Agility:** Project data changes daily; retraining daily is impractical.
* **Privacy:** Risks "baking" sensitive data into the model weights.

### Data "Retraining" (Dynamic Context)
Instead of retraining the brain (Model), we update the **book it reads (Context)**. 
* Because we fetch data from SQL in real-time, the "training" is effectively instantaneous. 
* If a Project Manager updates a status from "Red" to "Green," the very next AI query reflects that change.

---

## 5. Evaluating Retrieval Quality

| Level | Method | Indicator of Success |
| :--- | :--- | :--- |
| **L1 (Basic)** | Keyword Match (`ilike`) | High precision for exact names; low recall on synonyms. |
| **L2 (Intermediate)** | **Full-Text Search** | Handles stemming (e.g., "running" -> "run") via DB-native FTS. |
| **L3 (Advanced)** | **Semantic Search** | Handles intent (e.g., "over-budget projects") via **Vector DB**. |

---

## 🛠 Tech Stack
* **Model:** Phi-3-mini
* **Pattern:** Retrieval-Augmented Generation (RAG)
* **Backend:** .NET Core / SQL Server
* **Frontend:** Reasoning-enabled Search UI

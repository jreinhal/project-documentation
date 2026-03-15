# Agent Conductor: System Overview

## 1. Introduction
**Agent Conductor** is a multi-agent orchestration platform designed to facilitate collaboration between specialized AI personas. It moves beyond simple chat interfaces by introducing **Structured Workflow Chains**, **Contextual Memory**, and **Security Guardrails**.

The application is built with **Next.js 14**, **Tailwind CSS**, and the **Vercel AI SDK**.

---

## 2. Core Architecture

### **2.1. The "Bounce" Mechanism**
The core innovation is the ability for one agent to "bounce" a prompt to another.
*   **Manual Bounce**: User clicks "Pass Baton" on a message.
*   **Auto-Bounce (Workflows)**: The system automatically chains agents based on a predefined `Workflow` definition.

### **2.2. Shared Project Context (Memory)**
*   **Component**: `ContextSidebar` / `ProjectContextProvider`
*   **Function**: Injects a global system prompt fragment (e.g., "We are using Next.js") into *every* agent's request.
*   **Persistence**: Stored in `localStorage`.

### **2.3. Sentinel Guardrails**
*   **Component**: `guardrails.ts` / `ChatWindow.tsx`
*   **Function**: regex-based pre-flight check for PII (Credit Cards, API Keys) before sending data to the LLM.
*   **Audit**: Overrides are logged via `audit-log.ts`.

### **2.4. The Proving Ground (Benchmarks)**
*   **Component**: `EvaluationDashboard.tsx`
*   **Function**: Runs parallel requests against the API using a "Golden Prompt" to measure latency (ms) and response quality (heuristic).

---

## 3. Directory Structure
```
/app
  /api/chat      # Edge Runtime API route for LLM streaming
  page.tsx      # Main Dashboard & Orchestrator Logic
/components
  ChatWindow    # Individual Agent Interface (matches "Bounce")
  ContextSidebar # Global Memory UI
  WorkflowBuilder # Custom Chain Creator
  EvaluationDashboard # Benchmark UI
/lib
  personas.ts   # System Prompts for specific roles (Architect, Cipher, etc.)
  guardrails.ts # PII Regex Logic
  workflows.ts  # Chain Definitions
```

## 4. Running the Project
1.  `npm install`
2.  `npm run dev`
3.  Access at `http://localhost:3000`

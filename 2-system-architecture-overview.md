# 2. System Architecture Overview

## High-Level Architecture

### Issue identification

```mermaid
flowchart TD
    User([User]) <--> |Send and receive messages| A[Agent]

    A -->|Parse messages with context from workflow set| LLMM[LLM]
    LLMM-->|Return structured object<br>with issues identified - Workflow A|A

    A -->|Create uninitialized issue| I[Issue]

    subgraph AWS[Workflows]
        WA[Workflow A]
        WB[Workflow B]
    end

    WA-->|Issue relates to this workflow|I
    AWS -->|Defines capabilities| A

    style A fill:#f9d77e

```

## Core Entities

### 1. Messages and Chats

- **Messages**: Individual messages from users
- **Chats**: Ongoing conversations with users
  - Can be 1:1 or group chats
  - Uniquely identified by `chatId`
  - Contain history of messages and associated issues

### 2. Agents

Agents are the entities that interact with users:

- Have their own configuration (tone, voice, formatting)
- Has a set of 'things that it can handle'. The set of workflows assigned to the agent determine it's capabilities.
  - E.g. if the agent has the 'answer customer FAQ' and 'handle returns/exchanges' workflows assigned to it, it knows it can handle those two issues when it parses the message and it will escalate anything else.
- Interact with users through communication channels like WhatsApp, Slack, etc.

### 3. Workflows

Workflows define logic for resolving issues:

- Implemented as XState state machines
- Might be easier to imagine as the template from which issues get created
- Describes the entire workflow for a specific kind of issue
  - What context needs to be stored in relation to the issue
  - The different steps in the issue
  - The decision points in the issue, and whether these decisions are deterministic or LLM-determined
  - The terminal states for the issue (resolution, escalation, closed due to inactivity, etc.)
- Version-controlled for reliability

https://stately.ai/registry/editor/embed/dc5cd674-7f58-441f-912a-531b0a9c2e5f?mode=design&machineId=3f225818-4fd6-4800-9c0e-bf194ecb06f1

### 4. Issues

Issues represent "things to resolve" for the agent:

- Identified from incoming customer messages
- Is an "instance" of the workflow (which is a template)
- Associated with one workflow and version of that workflow, so an ongoing issue will never change versions of the workflow
- Contains workflow snapshots representing current state

### Issue execution

```mermaid
sequenceDiagram
    participant U as User
    participant LLM as LLM
    participant A as Agent
    participant TQ as Task Queue
    participant WE as Workflow Engine
    participant I as Issue
    participant EXT as External Systems

    Note over A: After issue identification

    A->>TQ: Enqueue initial task for the<br> workflow engine to initialize the issue

    Note over TQ: This task is to tell the workflow engine<br> to initialize the new issue

    TQ->>WE: Worker picks up task
    WE->>I: Request issue with workflow
    I->>WE: Return issue with workflow snapshot

    Note over WE: Workflow Processing Begins
    WE->>WE: Initialize issue
    Note over WE: There is usually an action associated<br> with the initial state of each workflow.<br> When the issue is initialized, the <br>workflow engine executes that initial action.
    WE->>EXT: Invoke action (API call, data fetch, message stakeholder, etc.)
    EXT->>WE: Return data
    alt If automatic state transition after invocation
    WE->>WE: Transition state based on result
    end
    WE->>I: Save updated workflow snapshot
    Note over I: The next time a task in the queue relates<br>to this issue, the workflow engine will load <br>the issue again and read the updated snapshot<br>which indicates the new state the issue is in.

    alt Needs Response
        WE->>A: Request to agent to send message to user
        A->>LLM: Generate response using context
        LLM->>A: Return natural language response
        A->>U: Send message to user
    end

    alt Has next step
        WE->>TQ: Schedule next task (immediate or delayed)
    end

    Note over A,WE: Workflow continues until resolution
```

## Execution services

### 1. Task Queue

Undecided if this needs to be a first-iteration. Necessary to enable non-user initiated workflows like "check every 2 minutes for late delivery orders and inform the customer", or delayed workflow steps like "send follow-up reminder after 2h if no response". Responsible for holding tasks to be processed by workers:

- Stores tasks to be picked up by workers later that process the tasks with the workflow engine.
- Supports immediate and delayed execution

### 2. Workflow Engine

Dedicated service for managing workflow execution:

- Hydrates state machines from persisted state
- Processes events and advances workflow state
- Executes side effects
- Persists updated state
- Schedules follow-up tasks

## Example - ALWA order inquiry flow

```mermaid
sequenceDiagram
    participant U as Restaurant
    participant A as Agent
    participant LLM as LLM Service
    participant I as Issue Service
    participant TQ as Task Queue
    participant WE as Workflow Engine
    participant AL as Atlas Service

    U->>A: Send message "Where are the riders<br>for W12934 and W18237?"
    A->>LLM: Parse message with context<br>of known capability - handle order enquiry
    LLM->>A: Return issue identification - 2 issues<br><br>Order enquiry for W12934<br>Order enquiry for W18237

    alt Create new issues
        A->>I: Create issues with initial workflow
        A->>TQ: Enqueue initial tasks
    end

    TQ->>WE: Worker picks up task
    WE->>I: Request issue with associated workflow
    I->>WE: Return issue with associated workflow
    WE->>WE: Hydrate workflow state machine,<br>enter initial state gettingOrderStatus
    WE->>AL: Invoke fetchOrderStatus
    AL->>WE: Return order status
    WE->>WE: onDone - transition state<br> to sendingStatusToRestaurant
    WE->>A: Request to send message
    A->>LLM: Request natural language <br>response based on context<br>including order statuses
    LLM->>A: Return natural language response
    A->>U: Send message

    WE->>I: Save issue with updated workflow snapshot
```

## 4. Core Design Decisions

### 3.1 BullMQ/XState for Workflow Management

- **Problem**: Need for deterministic, reliable business workflows that span time and context
- **Options Considered**: LangGraph, BullMQ/XState with custom LLM implementation
- **Decision**: Custom state management with XState state machines
  - _Note: This decision was not made lightly. I have agonized over this point and have had at many times wanted to default to LangGraph for safety, after all, nobody got fired for choosing IBM. However, after a lot of research, my conclusion is that for my use case - Weaver is really a workflow automation tool first that happens to have a natural language interface - BullMQ/XState makes enough sense that it is worth the risk._
- **Rationale**:

  - **LLM Orchestration Components**:

    - LangGraph (mainly LangChain) offers useful abstractions for LLM orchestration
    - These include output parsing, memory management, and prompt management
    - These components could be valuable regardless of the workflow engine choice

  - **Common Capabilities**:

    - Both LangGraph and XState can implement deterministic workflows with LLM decision points
    - Both support serialization and persistence of workflow state
    - Both offer visualization tools for workflows (Stately.ai and [LangGraph Studio](https://langchain-ai.github.io/langgraph/concepts/langgraph_studio/))

  - **Critical Architectural Differences**:

    - **Execution Model**:

      - LangGraph assumes continuous execution where data flows through a graph in a single session
      - XState is designed for event-driven state transitions with potentially long periods between events

    - **Asynchronous Processing**:

      - Weaver's workflows involve long periods of inactivity punctuated by brief moments of processing
      - Example: In a patient-doctor approval flow, the doctor might not respond for hours.
      - During waiting periods, workflows must be fully serialized with no active processes running

    - **Time-Based Operations**:

      - Weaver requires sophisticated time-based triggers (e.g., escalating after 24 hours of no response)
      - LangGraph has no native support for delayed transitions
      - BullMQ provides built-in mechanisms for delayed events to trigger transitions in any workflow at a later time

    - **Task Queue Integration**:
      - BullMQ complements XState by providing reliable, scalable delayed job processing
      - XState's event-driven state machines naturally align with BullMQ's job processing model
      - This combination enables workflows that efficiently span days or weeks with minimal resource consumption

  - **Hybrid Approach Possibility**:
    - We can import useful LangChain abstractions (like JSON parsing) while using XState/BullMQ
    - This allows us to leverage LangChain's LLM utilities without forcing LangGraph into patterns it wasn't designed for

- **Trade-offs**:

  - **Developer onboarding**: LangGraph is well-known amongst AI engineers. XState has more stars on GitHub but that's mostly around its use as a frontend state management library, less so as an AI agent.
  - **Less LLM-Native Community**: Fewer examples and patterns exist for integrating XState with LLMs compared to frameworks purpose-built for LLM orchestration. However, there are many code examples of LLM implementation in Node.js, and many code examples of XState, and they can be put together. Stately seems to have come out with an [agent framework](https://github.com/statelyai/agent) but it is not well-used.

- **Note on Visualization**: Both XState (via Stately.ai) and LangGraph (via LangGraph Studio) offer workflow visualization capabilities. While this is valuable for stakeholder communication and workflow understanding, it wasn't a deciding factor between the frameworks.

### 3.2 Semi-Agentic Approach

- **Problem**: Balancing flexibility and reliability in natural language interfaces for business workflows
- **Options Considered**: Fully agentic, semi-agentic, rule-based deterministic
- **Decision**: Semi-agentic with controlled LLM decision points
- **Rationale**:

  - **Controlled Agentic Boundaries**:

    - LLMs used primarily in well-defined decision points with clear constraints, while the rest of the workflow is deterministic
    - LLM agency limited to specific tasks in which LLM's ability to parse unstructured data are needed:
      - Understanding user intent from natural language
      - Classifying issues to appropriate workflows
      - Making bounded judgments within workflows (e.g., determining sentiment, classifying responses)
      - Generating human-like responses based on predefined templates and context

  - **Deterministic Core with Process Integrity**:

    - Core business logic implemented as deterministic state machines with explicit LLM decision points
    - Each LLM decision mapped to concrete workflow advancement with clear boundaries
    - Workflows maintain predictable paths regardless of conversation nuances
    - Critical operations (payments, approvals, escalations) follow consistent, auditable paths
    - System behavior remains reliable even with LLM inherent variability

  - **Rejected Alternatives**:

    - **Fully Agentic Approach**: Rejected because:

      - Introduces unpredictable behavior incompatible with business requirements
      - Excessive hallucination risk for critical business operations

    - **Rule-Based Deterministic Approach**: Rejected because:
      - Extremely rigid, creating poor user experience with natural language
      - Requires extensive rule creation and maintenance
      - Fails to leverage LLM capabilities for understanding nuanced requests
      - Quickly becomes unmanageable for complex domains
      - Cannot adapt to slight variations in user expressions

- **Note on Fallback Strategy**: While the core approach is semi-agentic, in the distant future, the system could include a fully agentic fallback mode when user requests don't match defined workflows. This would allowing more free-form exploration of undefined solution spaces before escalation to human support.

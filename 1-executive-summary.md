# Weaver - WhatsApp-based AI Workflow Automation

## 1. Executive Summary

### Problem Statement

Weaver addresses a critical gap between current WhatsApp bots that can only [handle shallow actions like FAQ answering or lead capture](https://www.chatavocado.ai/), etc. At most, they have a [simple IFTTT/Zapier integration](https://wawcd.com/comparison/) that allows for a fixed set of tasks to be executed. However, they are unable to autonomously resolve many issues for the customer when the workflows for the issues involve more decision points and branching logic. This means that while the bot can help to deflect very simple enquiries, it has to escalate a lot of issues which still leaves a lot of workload on customer support.

### Solution Approach

Weaver is a semi-agentic system that can sit in WhatsApp, handle customers, and is designed to solve this problem by handling complex issues end-to-end. The "semi-agentic" design is intentional and fundamental to Weaver's value proposition. It uses AI primarily for the natural language understanding layer, translating customer requests into discrete "issues" which represent an outcome the customer wants to achieve. These issues are mapped to workflows, where the bulk of the workflow follows deterministic, auditable paths with specific decision points that rely on unstructured input being parsed by the LLM. Unlike conventional chatbots that process conversations as continuous streams, Weaver's issue-based approach identifies specific customer needs within ongoing conversations and tracks them independently, allowing for more structured resolution paths while maintaining natural dialogue.

Weaver:

- Provides natural language interfaces in messaging platforms (primarily WhatsApp, but can be expanded to Slack, etc.), making it accessible and user-friendly for both end-customers and employees, while executing deterministic workflows behind the scenes
- Identifies discrete "issues" within conversations, enabling simultaneous handling of multiple customer needs and maintaining focus on issue resolution rather than just conversation maintenance typical of most chatbot interfaces. This approach is especially valuable for WhatsApp and industry contexts where interactions evolve as ongoing conversations over time, rather than one-off queries.
- Models workflows as state machines with XState, which models operational workflows very well (see [example A](https://www.youtube.com/watch?v=NVE77_axR6s) and [B](https://www.youtube.com/watch?v=YRNqFxQjThY)). It also supports visualization and visual editing also allows non-technical stakeholders to be able to have clear visibility (and eventually create/edit) workflows.

This approach delivers the accessibility of conversational AI with the reliability businesses require for critical operations.

Because the selling point of this system is end-to-end execution of detailed operational workflows, the ability of the system to provide value is very much dependent on how well the system is configured. As such, **it is unlikely that most organizations will be able to set up the system well enough by themselves to obtain maximum value**.

The business model built on top of such a system will have to be a high-touch process, along the lines of how ERP products like Salesforce or Oracle are sold. There should be a consulting and discovery phase to help customers map out the processes they want the agent to handle, and then a configuration phase to set up the workflows and necessary integrations for it. This will mean charging much higher prices than regular SaaS ($1000+/mo) plus a setup fee, which will be justified by the value of reliable agentic behavior.

_Potential future expansion: It may be possible to include a full agentic workflow as a fallback i.e. if the user's query doesn't fall into one of the agent's assigned workflows, it tries to solve it with a fully ReAct agentic approach using the tools it has available, instead of immediately escalating the conversation._

### Key Architectural Decisions

- **XState/BullMQ for workflow orchestration**: Chosen (after agonizing consideration) over LangGraph after comparing execution models for long-running processes. While LangGraph offers strong LLM orchestration capabilities, XState's event-driven architecture paired with BullMQ's reliable task queuing better supports Weaver's requirements for workflows with extended idle periods and time-based triggers (e.g., follow up reminders after 2h). This combination enables efficient management of workflows spanning days or weeks with minimal resource consumption during waiting periods. The useful abstractions of LangGraph/LangChain, like the JSON output parsing functions, can be imported separately, allowing us the benefit of those components without subscribing to the whole LangGraph framework.

  - The big lesson from building out ALWA is that I cannot write out the logic for workflows in naked code. Absolutely unscalable. Needs to be compartmentalized into state machines.

- **Semi-agentic approach with controlled LLM decision points**: Selected after evaluating fully agentic and rule-based alternatives. The semi-agentic model constrains LLMs to specific tasks (intent understanding, response generation) while keeping critical business logic in deterministic state machines. This approach avoids the unpredictability and hallucination risks of fully agentic systems while providing more flexibility than rigid rule-based approaches that struggle with natural language variation.

### Feedback Areas

- **Architectural complexity assessment**: The architecture introduces significant complexity through state machines, task queues, and the separation of concerns. Feedback is needed on whether this complexity is justified for the benefits it provides and how I might mitigate risks arising from the complexity. Quick thought - pay an experienced AI engineer $250 for an hour a week to review my code

- **XState vs. LangGraph decision validation**: The choice of XState over more AI-native alternatives like LangGraph represents a significant architectural decision that strays from the de facto choice LangGraph. Validation is needed on whether this approach is truly worth the risk of not going LangGraph, given the use case of Weaver.

# Vertex Sales Proposal Multi-Agent System

## Project Overview

A multi-agent AI system built in **n8n** that turns a client inquiry into a structured sales proposal for salesperson review. It was built for a fictional company, **Vertex Consulting**, as a business scenario project.

The system uses 7 specialised agents to analyse the client's requirement, match it to the service catalogue, check budget and timeline, write a proposal, and review it. **A salesperson must approve every proposal before anything is sent to a client.**

## Features

- **Requirement Analysis:** Analyses the customer's needs and requirements.
- **Service Matching:** Matches customer requirements with available services in the service catalogue.
- **Budget and Timeline Checking:** Checks the proposed service against the customer's budget and preferred timeline.
- **Proposal Writing:** Generates a sales proposal based on the collected information.
- **Proposal Review:** Reviews the generated proposal before it moves forward.
- **Human-in-the-Loop Review:** Allows a human to approve or decline a proposal.
- **Proposal Revision:** Supports revising a proposal based on feedback after it is declined.
- **Business Rule Enforcement:** Designed to use only approved services and prices, not offer discounts, and not make contractual commitments.
- **Missing Information Flagging:** Marks unclear or missing client details instead of inventing them.
- **Budget and Timeline Conflict Detection:** Flags when a service is over budget or does not fit the client's timeline.
- **Out-of-Catalogue Handling:** Refuses to invent services Vertex Consulting does not offer.

## Agents

The system uses 7 agents with clearly different jobs. Each agent has a defined input and output, which keeps the work focused and easier to check.

| # | Agent | Responsibility | Receives | Produces |
|---|---|---|---|---|
| 1 | **Supervisor Agent** | Coordinates the process and tracks the current stage. Does not analyse, write or decide content | Client inquiry from chat | Short status and the inquiry for the next agent |
| 2 | **Requirement Analyst** | Turns the inquiry into clear facts (problem, goal, budget, timeline, contact). Lists anything missing or unclear. Makes no assumptions | Client record and inquiry | Structured client analysis with a missing-information list |
| 3 | **Service Matcher** | Chooses the best-fit service(s) from the catalogue and explains why | Client analysis and service catalogue | Recommended service with reasoning |
| 4 | **Budget & Timeline Checker** | Compares approved price with budget and delivery time with client timeline. Checks business rules | Recommended service, approved prices, rules | Budget and timeline result, conflicts, salesperson flags |
| 5 | **Proposal Writer** | Writes the proposal using the 10 required sections in simple business language | All earlier outputs and proposal requirements | Draft proposal |
| 6 | **Proposal Reviewer** | Checks the draft for invented prices, unsupported promises, guarantees, missing sections, wrong deliverables and unflagged assumptions | Draft proposal, catalogue, rules | Review result and list of issues |
| 7 | **Proposal Revision Agent** | Fixes the proposal using salesperson comments or reviewer issues, changing only what is needed | Draft proposal and comments | Revised proposal |

### Why this structure
- **Separate analysis from writing.** The Requirement Analyst only extracts facts, so the Writer is less likely to invent client needs.
- **Separate checking from choosing.** The Budget & Timeline Checker independently verifies the Service Matcher's choice against prices and rules.
- **Separate writing from reviewing.** The Reviewer is a different agent from the Writer, so the draft is checked by something other than its author.
- **Revision is targeted.** The Revision Agent changes only what was requested, so the salesperson stays in control.

## Workflow Overview

The workflow is implemented in n8n and contains multiple connected stages for proposal preparation, review, and revision.

### Part 1

![Workflow Overview - Part 1](screenshots/workflow-overview1.png)

### Part 2

![Workflow Overview - Part 2](screenshots/contd-workflow-overview2.png)

### Flow Summary

```
Client inquiry
   ↓
Supervisor Agent
   ↓
Requirement Analyst → Service Matcher → Budget & Timeline Checker
   ↓
Proposal Writer → Proposal Reviewer
   ↓
Salesperson review (email)
   ├── Approve → Done
   └── Decline + comment → Revision Agent → Reviewer → new review email → ...
```

## Technologies Used

- **n8n** : Workflow automation and orchestration.
- **n8n AI Agent nodes (LangChain-based)**: Building blocks for the 7 agents, with chat model and memory sub-nodes.
- **OpenRouter**: LLM provider that powers the AI agents.
- **Email integration** : Sends proposals to the salesperson for approve or decline review.
- **JSON** : Export format for the n8n workflow.

## Testing

I ran 15 tests covering normal cases, budget conflicts, timeline conflicts, missing information, an out-of-catalogue request, a discount request, the salesperson decline loop, and consistency across repeated runs. Screenshots of most of test cases are in the Appendix of the [system report](docs/Vertex_Sales_Proposal_System_Report.pdf).

| # | Scenario | Result | Finding / Improvement |
|---|---|---|---|
| T01 | C002 normal case | Pass with issue | Contact-field mapping fixed. Service choice varied between S004 and S002 across runs |
| T02 | C003 reporting, 1-month timeline | Pass | Correct service (S005), timeline risk flagged. Prompt change recommended to keep the client's wording ("1 month") |
| T03 | C002 with budget below price | Pass with issue | System switched to a cheaper service without stating that S004 was over budget. Fix recommended: require explicit budget comparison |
| T04 | Request outside the catalogue (mobile app) | Pass | No service invented, no price or promise, flagged for salesperson |
| T05 | C002 with 2-week timeline | Pass | Recommended S001 (2 weeks) and stated that S002/S004 need 4 to 6 weeks. No faster delivery promised |
| T06 | C004 with budget and timeline missing | Fixed | Found a wrong price and timeline passed between agents. Fixed by adding the approved pricing table to the Writer's prompt with a "copy figures exactly" rule. Rerun confirmed |
| T07 | C005 budget and timeline conflict | Pass with issue | S004 flagged as over budget and S002 suggested. Timeline conflict worded too softly |
| T08 | Client asks for 20% discount | Pass | Refused, full catalogue price kept, flagged for salesperson |
| T09 | C001 vague requirement | Pass with issue | Numbers and risks correct, but the order of S001/S003 changed between runs. Rule recommended: assessment first |
| T10 | Salesperson declines with a comment | Pass | Revision Agent made one targeted change, re-sent for review, loop ended on approval |
| T11 | C004 normal case | Pass | S002, NPR 150,000, 4 to 6 weeks, within budget |
| T12 | C006 normal case | Pass with issue | S005 correct. Reviewer caught missing payment terms |
| T13 | C007 normal case | Pass with issue | S006 correct. Payment terms missing again |
| T14 | C008 normal case | Pass with issue | S007 correct, soft timeline risk flagged. Payment terms missing again |
| T15 | C002 run 3 times for consistency | Pass with issue | Runs 1 and 2 matched. Run 3 failed to complete and asked the user for catalogue data |

### Key findings
- **Pricing accuracy:** figures were corrupted when passed between agents (T06). Embedding the approved price table in the Writer prompt fixed it.
- **Business rules held in tests:** no discounts (T08), no invented services (T04), no promises of faster delivery (T05).
- **Human-in-the-loop works:** the decline, revise, re-review loop ran end to end (T10).

### Planned improvements
- Make service selection more consistent across runs (lower model temperature, stricter matching rule).
- Build the standard payment terms line (50% at start, 50% at completion) into the Writer's base template.
- Make the Supervisor re-send catalogue data if an agent asks for it, to prevent incomplete runs.
- Replace the full proposal template with a short "Out of Scope" note when no service matches.

## How to Import the Workflow

1. Set up an n8n instance.
2. Download `vertex_sales_proposal_workflow.json` from this repository.
3. In n8n, go to **Workflows → Import from file** and select the JSON.
4. Connect your own **OpenRouter** credential to each AI model node. API keys are not included in this repository.
5. Configure the email node with your own email account.
6. Open the chat trigger and send a sample client inquiry to test.

**Note:** Do not share your API keys or other secrets publicly.

## Limitations

- Cannot verify client information that is not in the inquiry.
- Cannot approve discounts or custom pricing.
- Cannot make legal or contractual commitments.
- Service selection can vary between runs for the same inquiry.
- Output quality depends on the underlying LLM.

## Human Oversight

A salesperson reviews and approves every proposal, especially where there is a budget or timeline conflict, a discount request, or unclear client requirements. The AI prepares recommendations; the salesperson makes the final decision.

## Project Report

Read the full project report here:
[Vertex Sales Proposal System Report](docs/Vertex_Sales_Proposal_System_Report.pdf)

## Author

**Salina Pudasaini**

GitHub: [PudasainiSalina2060](https://github.com/PudasainiSalina2060)

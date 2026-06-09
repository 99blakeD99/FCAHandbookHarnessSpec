# Workflow Specification Principles

## Overview

Following widespread practice, this specification has the following components:

- YAML describes *what* should happen, using a structure that when a suitable LLM is prompted to develop the code it can understand what to do
- Action Specifications describe *the inputs, outputs, and behavior* of each action, again using a structure that the LLM can understand
- Python makes it work. 

Anyone reading the YAML can look up the corresponding Action Specification to understand exactly what will occur. 

For now, `general_enquiry` is the only workflow covered, as a proof of concept; other types (regulatory_change_analysis, incident_investigation...) will be covered.

## Data Artifact

This specification assumes a Codified_Requirements_Text_And_Embeddings.json file (containing the regulatory handbook with embeddings) is available. On first workflow invocation, the harness loads the JSON into memory, where it remains for subsequent queries. A separate database system is not needed, for the following reasons:

- No real-time transactional updates — Handbook changes quarterly at most
- Read-only access — Harness never modifies, only queries
- Manageable scale — 10,438 records with 1024-dim vectors fit easily in memory
- Embeddings are static — Computed once, don't change
- Simple distribution — Just ship the JSON file

## YAML

### Why YAML for Compliance Workflows?

Compliance workflows are audit-intensive. Python code is executable but opaque to auditors. YAML brings key information to the surface. 

YAML is human-readable, version-controlled, and deterministic. This makes it more accessible to compliance teams. 

### Example: general_enquiry Workflow

For a concrete YAML workflow definition and complete action specifications for general_enquiry, see **GeneralEnquirySpec.md**.

The crucial point is the **nodes** structure: each node has a `name`, `action` type, `input` declarations, `output` type, and `config` containing action-specific parameters.

Each node follows the same structure: `name`, `action`, `input` (if applicable), and `config` (containing `Param_*` parameters). Action-specific fields are normalized under `config`, making the structure consistent for the Python execution loop.

Future workflows will follow the same pattern. Examples: regulatory_change_analysis, incident_investigation, periodic_audit.

**Naming convention:** Fields prefixed with `Param_` are configurable parameters defined in action specifications. Changing these values affects behavior (if implemented in Python). Fields without the prefix are structural (node name, action type, input/output declarations).

## Audit Trail Example

When regulators ask "How did you decide entries A, B, C apply?", the Harness provides a complete trace:

1. **YAML node 1 (extract_features)**: "We extracted these features from your product description"
2. **YAML node 2 (retrieve_entries)**: "We queried the Handbook for matching entries using semantic search (Param_top_k=20)"
3. **YAML node 3 (analyze_compliance)**: "Claude reasoned that entries A, B, C apply: [reasoning log with timestamp]"
4. **YAML node 4 (human_review)**: "Compliance officer [name] reviewed and approved on [timestamp] with actions [approved, identity_verified, snapshot_recorded]"
   - Both `timestamp` and `actions_carried_out` are declared as `required_fields` in the YAML, ensuring they are always captured

Every decision is traceable to a specific YAML node, configuration version, and explicit audit metadata (timestamps, approver identity, actions taken).

## Usage Records

Every interaction with the Harness is recorded for auditability and process improvement.

### Record Structure

Each interaction is appended to `interactions.json`:

```json
{
  "timestamp": "ISO-8601",
  "product_description": "string",
  "question": "string",
  "entries_retrieved": [{"id": int, "header": "string", "rank": int, "similarity": float}],
  "claude_reasoning": "string",
  "approval_decision": "approved|rejected",
  "approver_comment": "string",
  "session_id": "uuid"
}
```

### Purposes

**a. Compliance**: Complete audit trail of decisions, entries applied, and approvals. Answers "How was this decision made?"

**b. Feedback**: Analyze interaction patterns (rejected approvals, follow-up questions, entry frequencies) to identify process gaps and embedding quality issues.


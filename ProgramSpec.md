# Program Specification

This document provides the patterns, interfaces, and concrete implementations needed to build the Compliance Agent Harness. The Harness is easily adaptable to accommodate common programing languages. Python is used in this reference implementation.

The program implements the actions and orchestration patterns specified in GeneralEnquirySpec.md.

Each action's input/output contract is defined in GeneralEnquirySpec.md § Action Specifications. This document covers the implementation, not the contract.

---

## Table of Contents

1. [1. Orientation](#1-orientation)
   - [1.1 Architecture Overview](#11-architecture-overview)
   - [1.2 Data Flow Example](#12-data-flow-example)
   - [1.3 Deployment Models](#13-deployment-models)
2. [2. Security Considerations](#2-security-considerations)
3. [3. Foundations](#3-foundations)
   - [3.1 Exception Hierarchy](#31-exception-hierarchy)
   - [3.2 Action Base Class](#32-action-base-class)
   - [3.3 Action Registry](#33-action-registry)
   - [3.4 Claude Tool Schemas](#34-claude-tool-schemas)
   - [3.5 Common Patterns](#35-common-patterns)
4. [4. Action Implementations](#4-action-implementations)
   - [4.1 ParseMarkdownAction](#41-parsemarkdownaction)
   - [4.2 GlossaryLookupAction](#42-glossarylookupaction)
   - [4.3 EmbedTextAction](#43-embedtextaction)
   - [4.4 SemanticSearchAction](#44-semanticsearchaction)
   - [4.5 ClaudeReasoningAction](#45-claudereasoningaction)
5. [5. Harness Orchestration](#5-harness-orchestration)
   - [5.1 ComplianceHarness Class](#51-complianceharness-class)
   - [5.2 Workflow Execution](#52-workflow-execution)
   - [5.3 Audit Logging](#53-audit-logging)
   - [5.4 Tool Integration Entry Point](#54-tool-integration-entry-point)
6. [6. Testing](#6-testing)
7. [7. Implementation Checklist](#7-implementation-checklist)

---

## 1. Orientation

### 1.1 Architecture Overview

The Python harness has three layers:

1. **Orchestration Layer**: Loads YAML workflows, manages node execution, passes data between nodes
2. **Action Layer**: Implements each action type (parse_markdown, glossary_lookup, semantic_search, claude_reasoning) with input validation and error handling
3. **Data Layer**: Manages Handbook embeddings (in-memory), configuration files (weights.yaml [schema in StructuredSearch.md], prompts), and audit logging

### 1.2 Data Flow Example

For a complete general_enquiry workflow (all 6 nodes in order):

```
INPUT:
  entity_description: "# AdviceBot..."
  question: "Which COBS rules apply to algorithmic advice?"

NODE 1 (check_preconditions):
  INPUT: {} (validation runs at harness startup)
  OUTPUT: PreconditionValidation
  STORED: context['preconditions']
  
NODE 2 (extract_features / parse_markdown):
  INPUT: entity_description
  OUTPUT: EntityFeatures
  STORED: context['extract_features']

NODE 3 (check_terminology / glossary_lookup):
  INPUT: {entity_features: context['extract_features'], question}
  OUTPUT: TerminologyMapped {canonical_terms, unmapped_terms, all_search_terms}
  STORED: context['check_terminology']
  NOTE: all_search_terms = canonical_terms + unmapped_terms (convenience for downstream embedding)

NODE 4 (embed_question / embed_text):
  INPUT: {question, terminology_mapped: context['check_terminology']}
  OUTPUT: QuestionEmbedding (vector enriched with canonical_terms + unmapped_terms)
  STORED: context['embed_question']
  NOTE: Enriched embedding incorporates all_search_terms for improved semantic relevance

NODE 5 (retrieve_entries / semantic_search):
  INPUT: {entity_features: context['extract_features'], question_embedding: context['embed_question']}
  NOTE: Searches FULL Codified_Requirements_Text_And_Embeddings (all entries, NOT glossary-filtered)
  OUTPUT: RankedEntries (list of entries with similarity + weights, ranked by regulatory weighting)
  STORED: context['retrieve_entries']

NODE 6 (analyze_compliance / claude_reasoning):
  INPUT: {entity_features: context['extract_features'], entries: context['retrieve_entries']}
  OUTPUT: ComplianceAnalysis (answer + citations + reasoning_log, with verbatim citation validation)
  STORED: context['analyze_compliance']

FINAL OUTPUT: context['analyze_compliance'] (ComplianceAnalysis)
```

### 1.3 Deployment Models

The Harness can be deployed in multiple ways. Code patterns differ by model; this document provides examples for **Option 1 (Direct Integration)** but notes how to adapt for others:

| Model | Description | Code Pattern | Tool Definition |
|-------|-------------|--------------|-----------------|
| **Option 1: Direct Integration** | Python code calls Claude API directly with Harness as tool | SDK call with tool definition embedded | Hardcoded in Python or config file |
| **Option 2: Service Deployment** | Harness exposed as REST/GraphQL API; external service invokes it | FastAPI/Flask handler + tool definition in service | Service configuration (YAML/JSON) |
| **Option 3: Tool Marketplace** | Registered in platform (e.g., Anthropic marketplace); platform manages invocation | Platform-specific integration (may not require custom code) | Platform UI / database |

**Important:** Before generating code for a specific deployment model, clarify:
- Is the Harness being called directly from Python, or exposed as a service?
- Where should the tool definition live (hardcoded, config file, platform)?
- What external systems need to integrate (Claude API, database, etc.)?

---

## 2. Security Considerations

This section defines secure-by-default patterns for the Harness implementation. **Responsibility split:** The specification below ensures that code generated from these patterns will be secure at the architecture level. Implementers are responsible for secure deployment, credential management, and API integration handling.

### Input Validation

All user-supplied input must be validated before use:

**File Input (parse_markdown action):**
- Enforce maximum file size: 10 MB (configurable)
- Validate UTF-8 encoding; reject binary files with clear error
- Reject files without `.md` extension with user guidance
- Pattern: `if len(file_content) > MAX_FILE_SIZE: raise FileSizeExceededError(...)`

**Question Input (all actions):**
- Enforce maximum length: 1000 characters
- Trim whitespace; reject empty strings
- No validation on content (compliance questions can ask anything within scope)
- Pattern: `if not question.strip() or len(question) > 1000: raise InvalidInputError(...)`

**API Responses (EmbedTextAction, ClaudeReasoningAction):**
- Validate response schema matches expected output (not just HTTP 200)
- Reject responses missing required fields with actionable error
- Type-check numeric values (embeddings must be float arrays, not strings)
- Pattern: Use `pydantic.BaseModel` or equivalent for schema validation

### Credential Handling

API credentials are sensitive and must never be hardcoded or logged:

**Environment Variables (required):**
- `ANTHROPIC_API_KEY` — Load at startup, validate non-empty, store in memory only
- `OPENAI_API_KEY` or `VOYAGE_API_KEY` — Load at startup for embedding service
- Pattern: `api_key = os.getenv('ANTHROPIC_API_KEY'); if not api_key: raise MissingCredentialError(...)`

**No Credential Logging:**
- Never log full API keys or tokens
- If logging API calls (for debugging), redact keys: `key[:8] + "***"`
- Never log user passwords or approval comments that might contain sensitive data
- Pattern: Audit log structure excludes credentials; API key validation happens before any logging

**Configuration Files:**
- Never store credentials in `.yaml`, `.json`, or `.env` files committed to source control
- If config files are used (weights.yaml, prompt templates), validate file permissions (read-only where possible)

### Error Messages

Error messages must be helpful without leaking system information:

**Good error**: `"ANTHROPIC_API_KEY not set. Set: export ANTHROPIC_API_KEY=sk-..."`

**Bad error**: `"Connection refused to https://api.anthropic.com/v1/messages at 203.0.113.45:443"`

Guidelines:
- Tell users what to fix (missing key, wrong file format, out of scope question)
- Do NOT expose: internal IP addresses, file system paths, API hostnames, stack traces
- Do NOT expose: user data in error messages (e.g., "Approver email invalid: bob@example.com")
- Compliance Harness errors are user-facing (in compliance analysis) and audit-facing (in logs); keep both safe

### Logging & Audit Trail

Audit logs must be secure and tamper-evident:

**Log Content:**
- Timestamp (ISO 8601), action name, input summary (question first 50 chars, not full), output type, status
- **Do NOT log**: API keys, full user input if it contains sensitive product data, approver comments
- **Do log**: Entry IDs retrieved, analysis confidence score, approver identity (name, not email for privacy)

**Log Storage:**
- Logs are written to `interactions.json` as append-only (immutable, no overwrites)
- Implement rotation/archival policy to prevent unbounded growth (e.g., new file weekly)
- If stored in a database, use transactions to ensure atomicity; no partial logs
- Access control: restrict read access to logs (compliance officers, auditors only)

### API Integration

External API calls (Anthropic, OpenAI/Voyage) introduce network security risks:

**Timeout Handling:**
- Set connection timeout: 10 seconds
- Set read timeout: 30 seconds for embeddings, 60 seconds for Claude (reasoning is slow)
- Retry on transient failures (HTTP 429, 503, timeout) with exponential backoff: `min(2^n * 1s, 60s)` max 3 retries
- Pattern: Use `requests` or `httpx` with `timeout=` parameter; wrap in try/except with backoff

**SSL/TLS Verification:**
- Always verify SSL certificates: `verify=True` (default for `requests`)
- Never disable certificate verification (`verify=False`) for production
- If custom CA cert required, load from file, never bypass verification

**Rate Limiting:**
- Respect API rate limits documented by provider (Anthropic, OpenAI, Voyage)
- Implement client-side rate limiting if needed: track requests per minute, queue excess
- Log rate limit hits for debugging; do not expose to user (just "service temporarily busy")

### No Silent Defaults

In compliance analysis, silent failures are dangerous:

**Principle**: Fail explicitly, never silently default to a fallback that hides the problem

Examples:
- If Codified_Requirements_Text_And_Embeddings is unavailable: fail with "Handbook data not found" (not fall back to web search)
- If Claude API is down: fail with "LLM service unavailable" (not skip reasoning and return raw entries)
- If a citation cannot be validated: warn in output with specific entry ID and mismatch (not silently remove citation)

---

The code examples below assume **Option 1** (direct Python integration) but are adaptable.

---

## 2. Foundations

### 2.1 Exception Hierarchy

All actions raise domain-specific exceptions for clear error handling:

```python
class ActionError(Exception):
    """Base exception for all action errors."""
    pass

class ValidationError(ActionError):
    """Input validation failed."""
    pass

class DataSourceUnavailableError(ActionError):
    """Required data source (embeddings, config, etc.) is unavailable."""
    pass

class CitationValidationError(ActionError):
    """Citation text does not match source."""
    pass

class ApprovalError(ActionError):
    """Approval decision could not be recorded."""
    pass

class TimeoutError(ActionError):
    """Operation exceeded time limit (e.g., approval pending >48h)."""
    pass
```

### 2.2 Action Base Class

All actions inherit from a common interface:

```python
from abc import ABC, abstractmethod
from typing import Any, Dict

class Action(ABC):
    """Abstract base class for all harness actions."""
    
    def __init__(self, harness: 'ComplianceHarness' = None):
        """
        Initialize action with optional harness reference for data access.
        
        Args:
            harness: ComplianceHarness instance (provides handbook_index, workflows, etc.)
        """
        self.harness = harness
        self._messages: list = []  # Collect messages during execute() for audit logging
    
    @abstractmethod
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Any:
        """
        Execute the action.
        
        Args:
            node_input: Input prepared from YAML node["input"]. May contain:
                        - User inputs (entity_description, question)
                        - Outputs from earlier nodes (EntityFeatures, RankedEntries, etc.)
            config: Node configuration from YAML node["config"]. Contains Param_* keys.
        
        Returns:
            Output object matching the action's output schema (see GeneralEnquirySpec.md)
        
        Raises:
            Subclasses of ActionError with actionable messages
        """
        pass
    
    def _validate_config(self, config: Dict, required_params: list):
        """Helper: Ensure all required Param_* keys are present."""
        for param in required_params:
            if param not in config:
                raise ValidationError(f"Required config parameter missing: {param}")
```

### 2.3 Action Registry

Register all actions here. The harness uses this registry to look up action implementations by name from the YAML workflow:

```python
ACTION_REGISTRY = {
    "parse_markdown": ParseMarkdownAction,
    "glossary_lookup": GlossaryLookupAction,
    "embed_text": EmbedTextAction,
    "semantic_search": SemanticSearchAction,
    "claude_reasoning": ClaudeReasoningAction,
}
```

### 2.4 Module Configuration Constants

Define module-level constants for LLM model selection and other configuration:

```python
# Claude Model Configuration
# RATIONALE: Using Claude Opus for all LLM actions (feature extraction, terminology mapping, reasoning, etc.)
# because Opus offers superior reasoning capability for compliance analysis tasks.
# Trade-off: Opus is slower and more expensive than Sonnet, but accuracy is critical
# for compliance. Cost-conscious implementations can switch to Sonnet after validation.
DEFAULT_CLAUDE_MODEL = "claude-opus-4-8"  # Latest Opus model with improved performance

# Alternative models (if needed for cost optimization):
# CLAUDE_MODEL_FAST = "claude-sonnet-4-6"  # Faster, cheaper, but less capable
# CLAUDE_MODEL_EXTENDED_THINKING = "claude-opus-4-8"  # For reasoning tasks

# Token and timeout defaults
DEFAULT_MAX_TOKENS = 2000
DEFAULT_THINKING_BUDGET = 10000  # Tokens for extended thinking (Claude Opus only)
DEFAULT_TIMEOUT = 30  # Seconds
```

All actions that call Claude API must use `DEFAULT_CLAUDE_MODEL` constant instead of hardcoding model names.

### 2.5 Claude Tool Schemas

These tools enforce structured output from Claude for citations and reasoning logs. Define once as module-level constants (used by ClaudeReasoningAction):

```python
# Module-level tool definitions (referenced by ClaudeReasoningAction)
CITATION_FORMATTER_TOOL = {
    "name": "citation_formatter",
    "description": "Format a citation to an Handbook rule with entry_id, verbatim text, and context showing why it applies.",
    "input_schema": {
        "type": "object",
        "properties": {
            "entry_id": {
                "type": "string",
                "description": "Handbook rule identifier from RankedEntries (e.g., 'COBS 2.1.1R'). Must match a entry_id from the retrieved rules."
            },
            "cited_text": {
                "type": "string",
                "description": "Exact verbatim text from the Handbook rule. Must appear word-for-word in the rule's text field."
            },
            "context": {
                "type": "string",
                "description": "Sentence or paragraph explaining why this rule applies to the product (why the product triggers this rule)."
            },
            "binding_level": {
                "type": "string",
                "enum": ["R", "G", "E", "D"],
                "description": "Binding level: R=Rules, G=Guidance, E=Evidential, D=Deleted. Extracted from entry_id suffix."
            }
        },
        "required": ["entry_id", "cited_text", "context", "binding_level"]
    }
}

AUDIT_LOGGER_TOOL = {
    "name": "audit_logger",
    "description": "Log structured reasoning and confidence metrics for compliance analysis.",
    "input_schema": {
        "type": "object",
        "properties": {
            "reasoning_chain": {
                "type": "string",
                "description": "Step-by-step logic explaining which rules apply to the product and why (e.g., 'Rule A applies because feature X matches criterion Y; Rule B does not apply because...')"
            },
            "gaps_identified": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Array of edge cases, uncertainties, or areas where the spec is ambiguous"
            },
            "confidence_score": {
                "type": "number",
                "minimum": 0,
                "maximum": 1,
                "description": "Model's confidence in the compliance analysis (0-1). Lower if gaps or ambiguities exist."
            }
        },
        "required": ["reasoning_chain", "gaps_identified", "confidence_score"]
    }
}
```

### 2.5 Common Patterns

These patterns are repeated across action implementations. Reference these sections to avoid inline repetition.

#### Input Validation

All actions validate their inputs at the start of `execute()`:

```python
def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
    # Extract inputs
    x = node_input.get('x')
    
    # Validate shape and type
    if not x or not isinstance(x, str):
        raise ValidationError("x must be a non-empty string")
```

Pattern: Use `isinstance()` checks and raise `ValidationError` with a clear message naming the field and expected type.

#### Config Parameter Extraction

All actions extract configuration from the YAML `config` dict. Parameters are prefixed with `Param_`:

```python
# From YAML: config: { Param_top_k: 20, Param_source_version: "2026-Q1" }
top_k = config.get('Param_top_k', 20)  # Use defaults if specified
source_version = config.get('Param_source_version')

# Validate required params using _validate_config helper
self._validate_config(config, ['Param_top_k'])
```

Pattern: Extract with `.get()`, supply defaults, validate required params with the base class helper.

#### UTC Timestamp Emission

When emitting timestamps (used in ClaudeReasoningAction and logging):

```python
from datetime import datetime, timezone

timestamp = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')
# Result: "2026-05-16T14:32:45.123456Z" (ISO 8601 UTC)
```

#### External-Service Unavailability Errors

When a required service (embedding API, Glossary, Claude API) fails, raise `DataSourceUnavailableError` with actionable guidance:

```python
raise DataSourceUnavailableError(
    f"Handbook Glossary not available. Unable to perform terminology mapping.\n"
    f"Ensure Codified_Requirements_Text_And_Embeddings is loaded and accessible."
)
```

Pattern: (1) what failed, (2) actionable next step (file path, config, escalation contact, or doc reference).

#### UTC Timestamp with Timezone Import

All files that emit timestamps should include:

```python
from datetime import datetime, timezone
```

#### User Message Handling

All actions send real-time status updates to the user and audit trail. The `Action` base class provides a `message()` method:

```python
def message(self, text: str, message_type: str = "status"):
    """
    Send a message to user and audit trail.
    
    Args:
        text: Message content (unstyled; symbol added based on message_type)
        message_type: "status", "progress", "complete", "error", "warning"
    
    Effect:
        1. Prints styled message to user (stdout or websocket per deployment)
        2. Appends to self._messages for audit logging by harness
    
    Failure Semantics:
        - If user output fails (e.g., websocket disconnected): Log to stderr and continue
          (user feedback lost but action execution not blocked)
        - If audit append fails (e.g., memory constraint): Raise ActionError and halt action
          (audit integrity is critical; action must not continue if audit trail incomplete)
        - Never swallow exceptions that affect audit trail; let harness decide escalation
    """
    from datetime import datetime, timezone
    
    timestamp = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')
    
    # Format for user display (add symbol based on type)
    symbols = {
        "status": "ℹ",
        "progress": "→",
        "complete": "✓",
        "warning": "⚠",
        "error": "✗"
    }
    symbol = symbols.get(message_type, '')
    user_message = f"{symbol} {text}".strip() if symbol else text
    
    # Emit to user (stdout or websocket per deployment)
    print(user_message)
    
    # Collect for audit logging (harness drains self._messages after execute() completes)
    audit_entry = {
        'timestamp': timestamp,
        'type': message_type,
        'text': text
    }
    self._messages.append(audit_entry)
```

**Usage in actions:**

```python
# In ParseMarkdownAction.execute():
self.message("Extracting product features from your description...")
# ... extraction logic ...
self.message(f"Extracted {entity_name}: {feature_count} features identified", "complete")

# In SemanticSearchAction.execute():
self.message("Searching Handbook for relevant entries...")
# ... retrieve 50 candidates ...
self.message(f"Retrieved {candidate_count} candidates from {total_entries} entries", "progress")
# ... apply weighting ...
self.message(f"Retrieved {top_k} ranked entries ({score:.2f} similarity)", "complete")

# On error:
try:
    # ... operation ...
except APIError as e:
    self.message(f"Search failed: {str(e)}", "error")
    raise DataSourceUnavailableError(...)
```

**Message types (map to GeneralEnquirySpec.md § Messaging):**
- `"status"` → on_start, intermediate updates
- `"progress"` → long-running operations (candidates found, weights applied)
- `"complete"` → ✓ action succeeded
- `"warning"` → ⚠ soft failures (no results found, skipped step)
- `"error"` → ✗ hard failures (API down, validation failed)

All messages are both user-visible and audit-logged with timestamp.

#### Template Variable Interpolation

Message text in GeneralEnquirySpec.md § Messaging uses template variables like `{top_k}`, `{entity_name}`, `{confidence_score:.0%}`. These are **caller-side interpolated** (not by `message()` itself):

```python
# In SemanticSearchAction.execute():
top_k = 20
candidate_count = 50
total_rules = 10438

# Variable interpolation happens at call site
self.message(f"Retrieved {candidate_count} candidates from {total_rules} rules", "progress")

# message() receives the already-interpolated string: "Retrieved 50 candidates from 10438 rules"
```

**Convention**: Format specifiers (`.2f`, `.0%`, etc.) are Python f-string syntax. Actions must:
1. Extract the variable value from context (output, config, or computation)
2. Interpolate using f-string at the call site
3. Pass the fully-rendered string to `message()`

**Example with formatting**:
```python
# Spec says: "Retrieved {top_k} ranked rules ({top_1_score:.2f} similarity, {top_1_weight:.1f}x multiplier)"
top_1_score = 0.8734  # from results
top_1_weight = 2.5    # from weight_factors
self.message(
    f"Retrieved {top_k} ranked rules ({top_1_score:.2f} similarity, {top_1_weight:.1f}x multiplier)",
    "complete"
)
# Result: "Retrieved 20 ranked rules (0.87 similarity, 2.5x multiplier)"
```

---

## 3. Action Implementations

### 3.1 ParseMarkdownAction

#### Contract

**Input**: `entity_description` (string: markdown-formatted product description)
**Output**: `EntityFeatures` (dict with entity_name, entity_type, features, use_cases, client_interaction, data_handled, decision_authority)
**Config**: None required
**Raises**: `ValidationError` on invalid markdown or missing required fields

#### Implementation

Extracts structured product information from markdown using Claude.

```python
import json
from anthropic import Anthropic
from typing import Dict, List, Any

class ParseMarkdownAction(Action):
    """Extract and structure entity information from markdown using Claude."""
    
    def __init__(self):
        super().__init__()
        self.client = Anthropic()
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
        """
        Parse markdown product description into structured EntityFeatures.
        
        Args:
            node_input: {
                'entity_description': str (markdown-formatted product description)
            }
            config: {} (no parameters for this action)
        
        Returns:
            {
                'entity_name': str,
                'entity_type': str,
                'features': list of str,
                'use_cases': list of str,
                'client_interaction': str,
                'data_handled': list of str,
                'decision_authority': str
            }
        
        Raises:
            ValidationError: If required fields missing or invalid
        """
        entity_description = node_input.get('entity_description', '')
        
        if not entity_description or not isinstance(entity_description, str):
            raise ValidationError("entity_description must be a non-empty string")
        
        system_prompt = """You are an expert at extracting structured information from product descriptions.
Extract the following information from the product description provided by the user and return it as JSON:

{
  "entity_name": "string (product/service name)",
  "entity_type": "string (e.g., robo-advisor, trading platform, advisory service)",
  "features": ["list of key capabilities and features"],
  "use_cases": ["list of intended use cases and client segments"],
  "client_interaction": "string (manual, automated, or hybrid)",
  "data_handled": ["list of types of sensitive data handled"],
  "decision_authority": "string (one of: algorithm, advisor, client, hybrid)"
}

Important:
- Be strict: extract actual content, don't invent
- If a field is missing or unclear, infer it from context where reasonable
- decision_authority must be one of: algorithm, advisor, client, hybrid
- Return valid JSON only, no additional text"""
        
        try:
            response = self.client.messages.create(
                model=DEFAULT_CLAUDE_MODEL,
                max_tokens=1000,
                system=system_prompt,
                messages=[
                    {"role": "user", "content": f"Product description:\n\n{entity_description}"}
                ]
            )
            
            response_text = response.content[0].text.strip()
            extracted = json.loads(response_text)
            
            # Validate extracted data
            self._validate_extracted_data(extracted)
            
            return extracted
        
        except json.JSONDecodeError as e:
            raise ValidationError(f"LLM returned invalid JSON: {str(e)}")
        except Exception as e:
            raise ValidationError(f"LLM service unavailable: {str(e)}")
    
    def _validate_extracted_data(self, data: Dict[str, Any]) -> None:
        """Validate extracted EntityFeatures."""
        # Check required fields
        required_fields = ['entity_name', 'entity_type', 'features', 'use_cases', 'decision_authority']
        for field in required_fields:
            if field not in data:
                raise ValidationError(f"Missing required field: {field}")
        
        # Validate entity_name
        if not isinstance(data['entity_name'], str) or len(data['entity_name']) > 200:
            raise ValidationError(f"entity_name invalid: must be string, max 200 chars")
        
        # Validate features
        if not isinstance(data['features'], list) or not data['features']:
            raise ValidationError("features must be non-empty list")
        if len(data['features']) > 50:
            raise ValidationError(f"features must have max 50 items, got {len(data['features'])}")
        if any(len(f) > 500 for f in data['features']):
            raise ValidationError("One or more features exceed 500 chars")
        
        # Validate use_cases
        if not isinstance(data['use_cases'], list) or not data['use_cases']:
            raise ValidationError("use_cases must be non-empty list")
        if len(data['use_cases']) > 20:
            raise ValidationError(f"use_cases must have max 20 items, got {len(data['use_cases'])}")
        
        # Validate decision_authority
        if data['decision_authority'] not in ["algorithm", "advisor", "client", "hybrid"]:
            raise ValidationError(
                f"Invalid decision_authority: {data['decision_authority']} "
                "(must be one of: algorithm, advisor, client, hybrid)"
            )
        
        # Set defaults for optional fields
        if 'client_interaction' not in data:
            data['client_interaction'] = "unspecified"
        if 'data_handled' not in data:
            data['data_handled'] = []
```

### 3.2 GlossaryLookupAction

#### Contract

**Input**: `entity_features` (EntityFeatures), `question` (string)
**Output**: `TerminologyMapped` (dict with mapped_terms, unmapped_terms, all_search_terms)
**Config**: `Param_glossary_piece` (default: "Glossary")
**Raises**: `DataSourceUnavailableError` if Glossary unavailable

#### Implementation

Maps user question terminology to Glossary canonical terms using a two-stage approach: Claude extracts regulatory concepts, then deterministic semantic search finds matching glossary entries.

**Strategy**: 
1. **Stage 1 (Claude)**: Extract key regulatory/financial concepts from question + entity features
2. **Stage 2 (Deterministic)**: Use cosine similarity to find closest glossary entries for each concept

This approach has better harnessing properties: Claude is narrowly focused (concept extraction), semantic matching is deterministic and auditable (embeddings + cosine similarity). Prevents hallucination while leveraging LLM understanding of regulatory language.

```python
from typing import Dict, List, Any

class GlossaryLookupAction(Action):
    """
    Normalize user terminology by mapping to Glossary canonical terms.
    
    CRITICAL: This is a TERMINOLOGY NORMALIZATION step, NOT a retrieval/filtering step.
    
    Purpose: Enrich the user's question with official regulatory language for better
    embedding relevance. Output (canonical_terms, unmapped_terms) becomes metadata that
    augments the question embedding.
    
    Does NOT filter or restrict downstream semantic_search results to glossary entries.
    Semantic_search will search the full Handbook (all entries) using the
    enriched embedding.
    
    Design: Two-stage approach preserves auditability: (1) Claude narrows to concepts,
    (2) Deterministic semantic matching finds glossary entries. Prevents hallucination.
    """
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
        """
        Normalize user question and entity features to Glossary canonical terms.
        
        Two-stage approach:
        1. Claude extracts key regulatory/financial concepts from question + entity features
        2. Deterministic semantic search finds closest matching glossary entries by embedding similarity
        
        Output includes all_search_terms (canonical_terms + unmapped_terms), a convenience array
        for downstream embedding. Downstream semantic_search will search full handbook using
        the enriched embedding containing all_search_terms (NOT filtering to glossary entries).
        
        Args:
            node_input: {
                'entity_features': EntityFeatures (dict),
                'question': str
            }
            config: {
                'Param_glossary_piece': str (default "Glossary")
            }
        
        Returns:
            {
                'canonical_terms': [str, ...],    # Glossary terms Claude identified as relevant
                'unmapped_terms': [str, ...],     # User terms Claude couldn't map to glossary
                'all_search_terms': [str, ...]    # canonical_terms + unmapped_terms (for downstream embedding)
            }
        """
        entity_features = node_input.get('entity_features')
        question = node_input.get('question', '')
        
        if not question or not isinstance(question, str):
            raise ValidationError("question must be a non-empty string")
        if not entity_features or not isinstance(entity_features, dict):
            raise ValidationError("entity_features must be a valid EntityFeatures object")
        
        # Verify Glossary is loaded
        glossary_entries = self._get_glossary_entries()
        if not glossary_entries:
            raise DataSourceUnavailableError(
                "Glossary not found. Ensure Codified_Requirements_Text_And_Embeddings.json is loaded in HandbookIndex."
            )
        
        # Two-stage mapping: Claude extracts concepts → Deterministic semantic search matches to glossary
        result = self._map_with_claude(question, entity_features, None)
        
        canonical_terms = result.get('canonical_terms', [])
        unmapped_terms = result.get('unmapped_terms', [])
        all_search_terms = canonical_terms + unmapped_terms
        
        return {
            'canonical_terms': canonical_terms,
            'unmapped_terms': unmapped_terms,
            'all_search_terms': all_search_terms
        }
    
    def _get_glossary_entries(self) -> List[Dict[str, Any]]:
        """Retrieve Glossary entries from HandbookIndex."""
        if not hasattr(self.harness, 'handbook_index'):
            return []
        
        handbook = self.harness.handbook_index.fca_handbook
        glossary = [e for e in handbook if e.get('piece') == 'Glossary']
        return glossary
    
    def _map_with_claude(self, question: str, entity_features: Dict, glossary_ref: str) -> Dict[str, List[str]]:
        """
        Two-stage terminology mapping:
        1. Claude extracts key regulatory/financial concepts
        2. Deterministic semantic search finds matching glossary entries
        """
        # Stage 1: Claude extracts concepts
        concepts = self._extract_concepts_with_claude(question, entity_features)
        
        # Stage 2: Semantic search matches concepts to glossary entries
        canonical_terms, unmapped_terms = self._match_concepts_to_glossary(concepts)
        
        return {
            'canonical_terms': canonical_terms,
            'unmapped_terms': unmapped_terms
        }
    
    def _extract_concepts_with_claude(self, question: str, entity_features: Dict) -> List[str]:
        """
        Stage 1: Use Claude to extract key regulatory/financial concepts from question and entity features.
        
        Claude is focused on concept identification (what regulatory topics are relevant?),
        not on matching to specific glossary terms (which Stage 2 handles).
        """
        prompt = f"""Extract key regulatory and financial concepts from this compliance question and product description.

Compliance question:
{question}

Product/service characteristics:
{self._format_entity_features(entity_features)}

Task: Identify financial/regulatory concepts that would be relevant to compliance analysis.
Examples: "discretionary management", "investment advice", "client assets", "suitability", "market manipulation", etc.

Output: JSON with a list of key concepts (just the concepts, no definitions):
{{
    "concepts": ["concept1", "concept2", "concept3", ...]
}}
"""
        
        from anthropic import Anthropic
        import json
        
        client = Anthropic()
        response = client.messages.create(
            model=DEFAULT_CLAUDE_MODEL,
            max_tokens=300,
            messages=[{"role": "user", "content": prompt}]
        )
        
        try:
            result = json.loads(response.content[0].text)
            return result.get('concepts', [])
        except (json.JSONDecodeError, IndexError, KeyError) as e:
            self.message(f"Concept extraction failed: {e}", "warning")
            return []
    
    def _match_concepts_to_glossary(self, concepts: List[str]) -> tuple[List[str], List[str]]:
        """
        Stage 2: Deterministic semantic matching of extracted concepts to glossary entries.
        
        For each concept, find the closest glossary entry using cosine similarity on embeddings.
        This is purely deterministic (no LLM), auditable, and reproducible.
        """
        import numpy as np
        from scipy.spatial.distance import cosine
        
        if not concepts:
            return [], []
        
        glossary_entries = self._get_glossary_entries()
        if not glossary_entries:
            return [], concepts
        
        canonical_terms = []
        unmapped_terms = []
        
        # Get embedding model from HandbookIndex
        embedding_model = self.harness.handbook_index.embedding_model
        
        for concept in concepts:
            # Embed the concept using the same model as glossary
            concept_embedding = self._embed_text(concept, embedding_model)
            
            if concept_embedding is None:
                unmapped_terms.append(concept)
                continue
            
            # Find closest glossary entry by cosine similarity
            best_match = None
            best_similarity = -1
            
            for entry in glossary_entries:
                entry_embedding = entry.get('embedding')
                if not entry_embedding:
                    continue
                
                # Compute cosine similarity (1 - cosine distance)
                similarity = 1 - cosine(concept_embedding, entry_embedding)
                
                # Accept match if similarity above threshold (0.7)
                if similarity > best_similarity and similarity >= 0.7:
                    best_similarity = similarity
                    best_match = entry
            
            if best_match:
                # Extract term name from glossary entry header
                term = self._extract_term_from_header(best_match.get('header', ''))
                if term:
                    canonical_terms.append(term)
            else:
                unmapped_terms.append(concept)
        
        return canonical_terms, unmapped_terms
    
    def _embed_text(self, text: str, embedding_model: str) -> List[float] or None:
        """Embed text using the configured embedding model."""
        import os
        import requests
        
        try:
            if embedding_model == 'voyage-3-large':
                api_key = os.environ.get('VOYAGE_API_KEY')
                if not api_key:
                    return None
                
                response = requests.post(
                    'https://api.voyageai.com/v1/embeddings',
                    headers={'Authorization': f'Bearer {api_key}'},
                    json={'input': text, 'model': 'voyage-3-large'},
                    timeout=10
                )
                response.raise_for_status()
                return response.json()['data'][0]['embedding']
            
            elif embedding_model == 'text-embedding-3-large':
                api_key = os.environ.get('OPENAI_API_KEY')
                if not api_key:
                    return None
                
                response = requests.post(
                    'https://api.openai.com/v1/embeddings',
                    headers={'Authorization': f'Bearer {api_key}'},
                    json={'input': text, 'model': 'text-embedding-3-large'},
                    timeout=10
                )
                response.raise_for_status()
                return response.json()['data'][0]['embedding']
        except Exception as e:
            self.message(f"Embedding failed for '{text}': {e}", "warning")
            return None
    
    def _extract_term_from_header(self, header: str) -> str or None:
        """Extract term name from glossary entry header."""
        # Header format: "Term: 'xxx' | Definition: yyy"
        import re
        match = re.search(r"Term:\s*['\"]?([^'\"|\n]+)['\"]?", header)
        if match:
            return match.group(1).strip()
        return None
    
    def _format_entity_features(self, entity_features: Dict) -> str:
        """Format entity features for Claude prompt."""
        lines = []
        for key, value in entity_features.items():
            if isinstance(value, list):
                lines.append(f"  {key}: {', '.join(value)}")
            else:
                lines.append(f"  {key}: {value}")
        return "\n".join(lines)
```

### 3.3 EmbedTextAction

#### Contract

**Input**: `question` (string)
**Output**: `QuestionEmbedding` (vector)
**Config**: None (uses embedding model from harness.data_sources.fca_handbook.model)
**Raises**: `DataSourceUnavailableError` if embedding API unavailable

#### Implementation

```python
class EmbedTextAction(Action):
    """
    Embed the user question using canonical terms from glossary_lookup.
    
    Input includes canonical_terms + unmapped_terms from prior glossary normalization step.
    This enriches the embedding for improved semantic relevance (NOT for filtering downstream search).
    Downstream semantic_search will search the full Handbook using this enriched embedding.
    """
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> List[float]:
        """
        Embed the enriched user question with the same embedding model as Codified_Requirements_Text_And_Embeddings.
        
        IMPORTANT: This embedding incorporates canonical terms from glossary_lookup for enrichment purposes only.
        It does NOT restrict downstream semantic_search to glossary entries. The search scope remains full Handbook.
        
        Args:
            node_input: {
                'question': str,
                'terminology_mapped': {canonical_terms, unmapped_terms, all_search_terms}
            }
            config: {} (embedding model comes from harness.handbook_index.embedding_model)
        
        Returns:
            List[float]: Embedding vector matching handbook embeddings dimensionality
        """
        question = node_input.get('question', '')
        
        if not question or not isinstance(question, str):
            raise ValidationError("question must be a non-empty string")
        
        if len(question) > 1000:
            raise ValidationError("question must be max 1000 characters")
        
        # Call _embed_query (same as SemanticSearchAction uses internally)
        embedding = self._embed_query(question)
        
        return embedding
    
    def _embed_query(self, text: str) -> List[float]:
        """
        Embed query text using the same model as handbook embeddings.
        
        Detects embedding model from handbook_index.embedding_model and calls
        appropriate API (Voyage AI or OpenAI) with credentials from environment.
        """
        import os
        import requests
        
        if not text or not isinstance(text, str):
            raise ValidationError("text must be a non-empty string")
        
        embedding_model = self.harness.handbook_index.embedding_model
        
        try:
            if embedding_model == 'voyage-3-large':
                # Call Voyage AI API
                api_key = os.environ.get('VOYAGE_API_KEY')
                if not api_key:
                    raise DataSourceUnavailableError(
                        "Embedding service unavailable. VOYAGE_API_KEY environment variable not set.\n"
                        "Set VOYAGE_API_KEY to enable Voyage AI embeddings."
                    )
                
                response = requests.post(
                    'https://api.voyageai.com/v1/embeddings',
                    headers={'Authorization': f'Bearer {api_key}'},
                    json={'input': text, 'model': 'voyage-3-large'},
                    timeout=30
                )
                response.raise_for_status()
                embedding = response.json()['data'][0]['embedding']
                return embedding
            
            elif embedding_model == 'text-embedding-3-large':
                # Call OpenAI API
                api_key = os.environ.get('OPENAI_API_KEY')
                if not api_key:
                    raise DataSourceUnavailableError(
                        "Embedding service unavailable. OPENAI_API_KEY environment variable not set.\n"
                        "Set OPENAI_API_KEY to enable OpenAI embeddings."
                    )
                
                response = requests.post(
                    'https://api.openai.com/v1/embeddings',
                    headers={'Authorization': f'Bearer {api_key}'},
                    json={'input': text, 'model': 'text-embedding-3-large'},
                    timeout=30
                )
                response.raise_for_status()
                embedding = response.json()['data'][0]['embedding']
                return embedding
            
            else:
                raise DataSourceUnavailableError(
                    f"Unknown embedding model: {embedding_model}\n"
                    f"Expected 'voyage-3-large' or 'text-embedding-3-large'"
                )
        
        except requests.exceptions.RequestException as e:
            raise DataSourceUnavailableError(
                f"Embedding service unavailable. Failed to call {embedding_model} API: {str(e)}\n"
                f"Check API credentials and network connectivity."
            )
        except (KeyError, ValueError) as e:
            raise DataSourceUnavailableError(
                f"Embedding API returned invalid response: {str(e)}\n"
                f"Ensure API credentials are valid and model {embedding_model} is available."
            )
```

### 3.4 SemanticSearchAction

#### Contract

**Input**: 
- `entity_features` (EntityFeatures)
- `question_terms` (TerminologyMapped: canonical_terms, unmapped_terms, all_search_terms)
- `question_embedding` (vector): Enriched embedding that incorporates all_search_terms

**Output**: `RankedEntries` (array of rule objects with similarity + weight factors)

**Config**: `Param_top_k` (int, default 20), `Param_source_version` (string, default "2026-Q1")

**Raises**: `DataSourceUnavailableError` if data source unavailable

**Note**: Searches FULL Codified_Requirements_Text_And_Embeddings (all entries), not glossary-filtered. The enriched embedding uses all_search_terms for improved relevance, but doesn't restrict retrieval scope.

#### HandbookIndex (Data Layer)

Load and index the Handbook once at startup:

```python
import json
import numpy as np
from typing import Dict, List, Any

class HandbookIndex:
    """Load and index Handbook embeddings for semantic search."""
    
    def __init__(self, handbook_path: str, weights_config_path: str = None):
        """
        Load handbook data and weighting configuration.
        
        Args:
            handbook_path: Path to Codified_Requirements_Text_And_Embeddings JSON file
            weights_config_path: Unused (kept for compatibility). Weights are hardcoded from StructuredSearch.md
        
        Raises:
            DataSourceUnavailableError: If JSON is missing 'fca_handbook' key, has no records, or records lack embeddings
        """
        # Load handbook data
        try:
            with open(handbook_path) as f:
                data = json.load(f)
        except FileNotFoundError:
            raise DataSourceUnavailableError(
                f"Handbook JSON file not found at: {handbook_path}\n"
                f"See README.md § Getting Started (item 8) for expected file format."
            )
        except json.JSONDecodeError as e:
            raise DataSourceUnavailableError(
                f"Handbook JSON is malformed: {e}\n"
                f"Ensure the file is valid JSON from Codified_Requirements_Text_And_Embeddings"
            )
        
        # Validate structure
        if 'fca_handbook' not in data:
            raise DataSourceUnavailableError(
                f"Handbook JSON missing 'fca_handbook' key.\n"
                f"Expected structure: {{'fca_handbook': [{{'entry_id': '...', 'text': '...', 'voyage-3-large_embedding': [...]}}]}}\n"
                f"See FCA_Handbook_Template_PRIN.json for the correct format."
            )
        
        self.records = data['fca_handbook']
        
        if not self.records:
            raise DataSourceUnavailableError(
                f"Handbook has no records (fca_handbook array is empty).\n"
                f"Ensure Codified_Requirements_Text_And_Embeddings contains at least one record."
            )
        
        # Detect and load embeddings: try common embedding key names (Voyage or OpenAI)
        self.embedding_key = self._detect_embedding_key(self.records[0])
        self.embeddings = np.array([record[self.embedding_key] for record in self.records])
        
        # Map embedding key to model name for query embedding
        self.embedding_model = self._get_model_name(self.embedding_key)
        
        # Validate embedding dimensions match expected model
        self._validate_embedding_dimensions()
        
        # Default regulatory weights (from StructuredSearch.md § Weighting Factors)
        self.weights = {
            'entry_type_weights': {
                'RULES': 1.0,
                'GUIDANCE': 0.8,
                'EVIDENTIAL': 0.6,
                'UNCLASSIFIED': 0.8,
                'DELETED': 0.1
            },
            'importance_multipliers': {
                'high': 1.3,
                'medium': 1.1,
                'low': 0.7
            },
            'piece_base_weights': {
                'Main Handbook': 1.2,
                'Glossary': 0.95,
                'Instruments': 1.0,
                'Forms': 0.85,
                'Technical Standards': 0.9,
                'Level3Materials': 0.85
            },
            'hierarchy_multipliers': {
                1: 1.2,
                2: 1.0,
                'default': 0.9
            }
        }
    
    def _detect_embedding_key(self, sample_record: Dict) -> str:
        """
        Detect which embedding key exists in the record.
        Tries: voyage-3-large_embedding (default), OpenAI_text-embedding-3-large_embedding, or 'embedding'.
        
        Raises:
            DataSourceUnavailableError: If no embedding key found (plain JSON without embeddings)
        """
        embedding_keys = [
            'voyage-3-large_embedding',
            'OpenAI_text-embedding-3-large_embedding',
            'embedding'
        ]
        
        for key in embedding_keys:
            if key in sample_record:
                return key
        
        available_keys = list(sample_record.keys())
        raise DataSourceUnavailableError(
            f"ERROR: Codified_Requirements_Text_And_Embeddings is missing vector embeddings.\n\n"
            f"The Harness requires each record to include a pre-computed embedding vector.\n"
            f"Your JSON has these keys: {available_keys}\n"
            f"But is missing one of: {embedding_keys}\n\n"
            f"SOLUTION: Follow these steps to add embeddings to your JSON:\n"
            f"1. Read EmbeddingModel.md § 'Action' for instructions on embedding your data\n"
            f"2. Use an LLM API (Voyage AI or OpenAI) to embed each record\n"
            f"3. Add the embedding vector to each record with the key: 'voyage-3-large_embedding'\n"
            f"4. Example record:\n"
            f"   {{\n"
            f"     'entry_id': 'COBS 2.1.1R',\n"
            f"     'text': '...',\n"
            f"     'voyage-3-large_embedding': [0.123, -0.456, ...]\n"
            f"   }}\n\n"
            f"See FCA_Handbook_Template_PRIN.json for a complete example with two embedding models."
        )
    
    def _get_model_name(self, embedding_key: str) -> str:
        """Map embedding key to model name for query embedding."""
        model_map = {
            'voyage-3-large_embedding': 'voyage-3-large',
            'OpenAI_text-embedding-3-large_embedding': 'text-embedding-3-large',
            'embedding': 'text-embedding-3-large'
        }
        return model_map.get(embedding_key, 'text-embedding-3-large')
    
    def _validate_embedding_dimensions(self):
        """
        Validate that embeddings have correct dimensionality for the detected model.
        Prevents silent errors if embeddings were generated with a different model.
        
        Raises:
            DataSourceUnavailableError: If embedding dimensions don't match expected model
        """
        expected_dims = {
            'voyage-3-large': 1024,
            'text-embedding-3-large': 3072
        }
        
        actual_dim = self.embeddings.shape[1] if self.embeddings.ndim == 2 else len(self.embeddings[0])
        expected_dim = expected_dims.get(self.embedding_model, None)
        
        if expected_dim and actual_dim != expected_dim:
            raise DataSourceUnavailableError(
                f"Embedding dimensionality mismatch.\n"
                f"Expected {expected_dim} dimensions for model '{self.embedding_model}'\n"
                f"but found {actual_dim} dimensions in key '{self.embedding_key}'.\n\n"
                f"Possible causes:\n"
                f"- Embeddings were generated with a different model than {self.embedding_model}\n"
                f"- The wrong embedding key was detected\n\n"
                f"Solution: Ensure all embeddings use {self.embedding_model} model "
                f"(see EmbeddingModel.md § 'Current Choice' for details)."
            )
    
    def search(self, query_embedding, top_k: int = 20) -> List[Dict[str, Any]]:
        """
        Find top-k most similar records using cosine similarity, then apply regulatory weighting.
        
        Args:
            query_embedding: 3072-dimensional embedding from text-embedding-3-large
            top_k: Number of results to return (default 20, range 1-100)
        
        Returns:
            List of dicts with: entry_id, text, base_similarity, weight_factors, final_score
        """
        query_embedding = np.array(query_embedding)
        
        # Vectorized cosine similarity
        query_norm = np.linalg.norm(query_embedding)
        embeddings_norms = np.linalg.norm(self.embeddings, axis=1)
        dot_products = np.dot(self.embeddings, query_embedding)
        similarities = dot_products / (embeddings_norms * query_norm)
        
        # Pre-filter: get top 50 candidates before weighting
        top_50_indices = np.argsort(similarities)[-50:][::-1]
        candidates = []
        for idx in top_50_indices:
            candidates.append({
                'index': int(idx),
                'record': self.records[idx],
                'base_similarity': float(similarities[idx])
            })
        
        # Apply regulatory weighting to candidates
        for candidate in candidates:
            candidate['weight_factors'] = self._compute_weights(candidate['record'])
            candidate['final_score'] = (
                candidate['base_similarity'] 
                * candidate['weight_factors']['entry_type_weight']
                * candidate['weight_factors']['hierarchy_multiplier']
                * candidate['weight_factors']['importance_multiplier']
                * candidate['weight_factors']['piece_weight']
            )
        
        # Sort by final_score and return top_k
        candidates.sort(key=lambda x: x['final_score'], reverse=True)
        return candidates[:top_k]
    
    def _compute_weights(self, record: Dict) -> Dict[str, float]:
        """
        Compute all weight factors for a record per StructuredSearch.md § Weighting Factors.
        
        Extracts entry type, hierarchy level, importance score, and piece authority
        to compute final weighting multipliers for regulatory ranking.
        """
        # Extract entry_type_weight from regulatory_content suffix (R/G/E/U/D)
        regulatory_content = record.get('regulatory_content', '')
        entry_type_map = {'R': 'RULES', 'G': 'GUIDANCE', 'E': 'EVIDENTIAL', 'D': 'DELETED', 'U': 'UNCLASSIFIED'}
        
        entry_type_key = 'UNCLASSIFIED'  # Default if not found
        for char in regulatory_content:
            if char in entry_type_map:
                entry_type_key = entry_type_map[char]
                break
        
        entry_type_weight = self.weights['entry_type_weights'].get(entry_type_key, 0.8)
        
        # Extract hierarchy_multiplier from record['level']
        level = record.get('level', 3)
        hierarchy_multiplier = self.weights['hierarchy_multipliers'].get(
            level,
            self.weights['hierarchy_multipliers'].get('default', 0.9)
        )
        
        # Extract importance_multiplier from record['regulatory_score'] (1-12 → high/medium/low bands)
        regulatory_score = record.get('regulatory_score', 4)
        if regulatory_score >= 7:
            importance_key = 'high'
        elif regulatory_score >= 4:
            importance_key = 'medium'
        else:
            importance_key = 'low'
        
        importance_multiplier = self.weights['importance_multipliers'].get(importance_key, 1.1)
        
        # Extract piece_weight from record['piece']
        piece = record.get('piece', 'Main Handbook')
        piece_weight = self.weights['piece_base_weights'].get(piece, 1.0)
        
        return {
            'entry_type_weight': entry_type_weight,
            'hierarchy_multiplier': hierarchy_multiplier,
            'importance_multiplier': importance_multiplier,
            'piece_weight': piece_weight
        }
```

#### SemanticSearchAction (Action Layer)

```python
from anthropic import Anthropic

class SemanticSearchAction(Action):
    """
    Query FULL Codified_Requirements_Text_And_Embeddings for entries matching question.
    
    CRITICAL: This searches ALL 10,438 entries (not glossary-restricted):
    - 3,670 Main Handbook
    - 3,640 Glossary
    - 1,933 Instruments
    - 812 Forms
    - 197 Technical Standards
    - 186 Level 3 Materials
    
    Despite glossary enrichment from glossary_lookup, downstream search is NOT
    filtered to glossary entries. Canonical terms improve embedding relevance;
    the search itself spans full handbook using cosine similarity + regulatory weighting.
    """
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> List[Dict[str, Any]]:
        """
        Search FULL Codified_Requirements_Text_And_Embeddings (all entries).
        
        Apply regulatory weighting to rank by binding authority and relevance.
        
        Input: question_embedding enriched with canonical terms from glossary_lookup
        (canonical terms improve semantic relevance; they do NOT filter results)
        
        Output: top_k entries ranked by final_score (cosine similarity × entry_type_weight
        × hierarchy_multiplier × importance_multiplier × piece_weight)
        
        Args:
            node_input: {
                'entity_features': EntityFeatures,
                'question_terms': TerminologyMapped (from glossary_lookup),
                'question': str
            }
            config: {
                'Param_top_k': int (default 20, range 1-100)
            }
        
        Returns:
            List of RankedEntry dicts (entry_id, text, base_similarity, weight_factors, final_score)
        """
        entity_features = node_input.get('entity_features', {})
        question_terms = node_input.get('question_terms', {})
        question = node_input.get('question', '')
        top_k = config.get('Param_top_k', 20)
        source_version = config.get('Param_source_version', '2026-Q1')
        
        if not isinstance(top_k, int) or top_k < 1 or top_k > 100:
            raise ValidationError("Param_top_k must be an integer between 1 and 100")
        
        # Extract search terms: original question + mapped canonical terms
        search_terms = question_terms.get('all_search_terms', [])
        enriched_question = question + ' ' + ' '.join(search_terms)
        
        # Embed the enriched question
        # Stub: call embedding API to get query_embedding
        query_embedding = self._embed_query(enriched_question)
        
        # Search and rank
        results = self.handbook_index.search(query_embedding, top_k)
        
        return [
            {
                'entry_id': r['record'].get('entry_id'),
                'text': r['record'].get('text'),
                'base_similarity': r['base_similarity'],
                'weight_factors': r['weight_factors'],
                'final_score': r['final_score'],
                'source_version': source_version
            }
            for r in results
        ]
    
    def _embed_query(self, text: str) -> List[float]:
        """
        Embed query text using the same model as handbook embeddings.
        
        Detects embedding model from handbook_index.embedding_model and calls
        appropriate API (Voyage AI or OpenAI) with credentials from environment.
        """
        import os
        import requests
        
        if not text or not isinstance(text, str):
            raise ValidationError("text must be a non-empty string")
        
        embedding_model = self.harness.handbook_index.embedding_model
        
        try:
            if embedding_model == 'voyage-3-large':
                # Call Voyage AI API
                api_key = os.environ.get('VOYAGE_API_KEY')
                if not api_key:
                    raise DataSourceUnavailableError(
                        "Embedding service unavailable. VOYAGE_API_KEY environment variable not set.\n"
                        "Set VOYAGE_API_KEY to enable Voyage AI embeddings."
                    )
                
                response = requests.post(
                    'https://api.voyageai.com/v1/embeddings',
                    headers={'Authorization': f'Bearer {api_key}'},
                    json={'input': text, 'model': 'voyage-3-large'},
                    timeout=30
                )
                response.raise_for_status()
                embedding = response.json()['data'][0]['embedding']
                return embedding
            
            elif embedding_model == 'text-embedding-3-large':
                # Call OpenAI API
                api_key = os.environ.get('OPENAI_API_KEY')
                if not api_key:
                    raise DataSourceUnavailableError(
                        "Embedding service unavailable. OPENAI_API_KEY environment variable not set.\n"
                        "Set OPENAI_API_KEY to enable OpenAI embeddings."
                    )
                
                response = requests.post(
                    'https://api.openai.com/v1/embeddings',
                    headers={'Authorization': f'Bearer {api_key}'},
                    json={'input': text, 'model': 'text-embedding-3-large'},
                    timeout=30
                )
                response.raise_for_status()
                embedding = response.json()['data'][0]['embedding']
                return embedding
            
            else:
                raise DataSourceUnavailableError(
                    f"Unknown embedding model: {embedding_model}\n"
                    f"Expected 'voyage-3-large' or 'text-embedding-3-large'"
                )
        
        except requests.exceptions.RequestException as e:
            raise DataSourceUnavailableError(
                f"Embedding service unavailable. Failed to call {embedding_model} API: {str(e)}\n"
                f"Check API credentials and network connectivity."
            )
        except (KeyError, ValueError) as e:
            raise DataSourceUnavailableError(
                f"Embedding API returned invalid response: {str(e)}\n"
                f"Ensure API credentials are valid and model {embedding_model} is available."
            )
```

### 3.5 ClaudeReasoningAction

#### Contract

**Input**: `entity_features` (EntityFeatures), `entries` (RankedEntries)
**Output**: `ComplianceAnalysis` (dict with answer, citations, reasoning_log, refinement_suggestions, timestamp)
**Config**: `Param_tools` (list), `Param_prompt_template` (path)
**Raises**: `ValidationError`, `DataSourceUnavailableError`

#### Implementation

```python
from anthropic import Anthropic
from datetime import datetime, timezone
from jinja2 import Environment, FileSystemLoader

class ClaudeReasoningAction(Action):
    """Use Claude to reason over rules and produce compliance analysis."""
    
    def execute(self, node_input: Dict[str, Any], config: Dict[str, Any]) -> Dict[str, Any]:
        """
        Invoke Claude to analyze product against retrieved Handbook rules.
        
        Args:
            node_input: {
                'entity_features': dict (EntityFeatures),
                'rules': list (RankedEntries from semantic_search)
            }
            config: {
                'Param_tools': list (e.g., ['citation_formatter', 'audit_logger']),
                'Param_prompt_template': str (path to prompt template file)
            }
        
        Returns:
            {
                'answer': str (Claude's analysis),
                'citations': list of {'entry_id', 'cited_text', 'context', 'binding_level'},
                'reasoning_log': {'reasoning_chain': str, 'gaps_identified': list, 'confidence_score': float},
                'timestamp': str (ISO 8601)
            }
        """
        entity_features = node_input.get('entity_features', {})
        rules = node_input.get('entries', [])
        
        if not isinstance(rules, list) or len(rules) == 0:
            raise ValidationError("rules must be non-empty list of RankedEntries")
        
        try:
            prompt_template_path = config.get('Param_prompt_template')
            if not prompt_template_path:
                raise ValidationError("Param_prompt_template required")
            with open(prompt_template_path) as f:
                template_text = f.read()
        except FileNotFoundError:
            raise ValidationError(f"Prompt template not found at {prompt_template_path}")
        
        # Construct system prompt using Jinja2
        jinja_env = Environment()
        template = jinja_env.from_string(template_text)
        system_prompt = template.render(
            entity_features=json.dumps(entity_features, indent=2),
            ranked_entries=json.dumps(rules, indent=2)
        )
        
        # Build tool list from config (or default to citation_formatter and audit_logger)
        tools_config = config.get('Param_tools', ['citation_formatter', 'audit_logger'])
        tool_map = {
            'citation_formatter': CITATION_FORMATTER_TOOL,
            'audit_logger': AUDIT_LOGGER_TOOL
        }
        tools = [tool_map[name] for name in tools_config if name in tool_map]
        if not tools:
            tools = [CITATION_FORMATTER_TOOL, AUDIT_LOGGER_TOOL]
        
        # Call Claude with extended thinking and prompt caching
        client = Anthropic()
        
        response = client.messages.create(
            model=DEFAULT_CLAUDE_MODEL,
            max_tokens=4000,
            temperature=0.5,
            thinking={
                "type": "enabled",
                "budget_tokens": 1000
            },
            system=[
                {
                    "type": "text",
                    "text": system_prompt,
                    "cache_control": {"type": "ephemeral"}
                }
            ],
            tools=tools,
            messages=[
                {
                    "role": "user",
                    "content": "Perform the compliance analysis using the tools to cite rules and log reasoning."
                }
            ]
        )
        
        # Extract citations and reasoning from response
        citations = self._extract_citations(response, rules)
        reasoning_log = self._extract_reasoning_log(response)
        answer = self._extract_answer(response)
        
        # Validate citations
        for citation in citations:
            if citation['entry_id'] not in [r['entry_id'] for r in rules]:
                # Flag for review but don't halt
                reasoning_log['_citation_warning'] = f"Citation {citation['entry_id']} not in retrieved rules"
        
        timestamp = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')
        
        return {
            'answer': answer,
            'citations': citations,
            'reasoning_log': reasoning_log,
            'refinement_suggestions': refinement_suggestions,
            'timestamp': timestamp,
            'handbook_version': handbook_version,
            'version_notice': version_notice
        }
    
    def _extract_citations(self, response, rules) -> List[Dict]:
        """
        Extract citation_formatter tool calls from Claude response.
        
        Iterates over response.content blocks, finds tool_use blocks where
        name == 'citation_formatter', and validates entry_id against retrieved rules.
        """
        citations = []
        entry_ids = {r['entry_id'] for r in rules}
        
        if not hasattr(response, 'content') or not response.content:
            return citations
        
        for block in response.content:
            # Check if this is a tool_use block with name == 'citation_formatter'
            if hasattr(block, 'type') and block.type == 'tool_use':
                if hasattr(block, 'name') and block.name == 'citation_formatter':
                    try:
                        # Extract input data from tool call
                        tool_input = block.input
                        
                        entry_id = tool_input.get('entry_id')
                        cited_text = tool_input.get('cited_text')
                        context = tool_input.get('context')
                        binding_level = tool_input.get('binding_level')
                        
                        # Validate entry_id exists in retrieved rules
                        if entry_id not in entry_ids:
                            # Log warning but continue processing other citations
                            import logging
                            logging.warning(
                                f"Citation references entry_id '{entry_id}' not in retrieved rules. "
                                f"Available: {sorted(entry_ids)}"
                            )
                            continue
                        
                        # Validate required fields present
                        if not all([entry_id, cited_text, context, binding_level]):
                            import logging
                            logging.warning(
                                f"Citation incomplete: entry_id={entry_id}, cited_text={bool(cited_text)}, "
                                f"context={bool(context)}, binding_level={binding_level}"
                            )
                            continue
                        
                        citations.append({
                            'entry_id': entry_id,
                            'cited_text': cited_text,
                            'context': context,
                            'binding_level': binding_level
                        })
                    
                    except (AttributeError, KeyError, TypeError) as e:
                        import logging
                        logging.warning(f"Failed to parse citation_formatter tool call: {str(e)}")
                        continue
        
        return citations
    
    def _extract_reasoning_log(self, response) -> Dict:
        """
        Extract audit_logger tool calls from Claude response.
        
        Iterates over response.content blocks, finds tool_use blocks where
        name == 'audit_logger', and extracts reasoning chain and confidence metrics.
        """
        reasoning_log = {
            'reasoning_chain': '',
            'gaps_identified': [],
            'confidence_score': 0.0
        }
        
        if not hasattr(response, 'content') or not response.content:
            return reasoning_log
        
        for block in response.content:
            # Check if this is a tool_use block with name == 'audit_logger'
            if hasattr(block, 'type') and block.type == 'tool_use':
                if hasattr(block, 'name') and block.name == 'audit_logger':
                    try:
                        # Extract input data from tool call
                        tool_input = block.input
                        
                        reasoning_chain = tool_input.get('reasoning_chain', '')
                        gaps_identified = tool_input.get('gaps_identified', [])
                        confidence_score = tool_input.get('confidence_score', 0.0)
                        
                        # Validate and normalize data
                        if not isinstance(gaps_identified, list):
                            gaps_identified = [str(gaps_identified)] if gaps_identified else []
                        
                        if not isinstance(confidence_score, (int, float)):
                            confidence_score = 0.0
                        else:
                            confidence_score = float(max(0.0, min(1.0, confidence_score)))
                        
                        reasoning_log = {
                            'reasoning_chain': str(reasoning_chain) if reasoning_chain else '',
                            'gaps_identified': [str(g) for g in gaps_identified],
                            'confidence_score': confidence_score
                        }
                        
                        # Return first valid audit_logger call (should only be one per response)
                        return reasoning_log
                    
                    except (AttributeError, KeyError, TypeError, ValueError) as e:
                        import logging
                        logging.warning(f"Failed to parse audit_logger tool call: {str(e)}")
                        continue
        
        return reasoning_log
    
    def _extract_answer(self, response) -> str:
        """
        Extract text response from Claude.
        
        Concatenates all text blocks from response.content (type == 'text'),
        skipping tool_use and metadata blocks.
        """
        text_blocks = []
        
        if not hasattr(response, 'content') or not response.content:
            return ''
        
        for block in response.content:
            # Extract text blocks only (skip tool_use, thinking, input, etc.)
            if hasattr(block, 'type') and block.type == 'text':
                if hasattr(block, 'text'):
                    text = block.text.strip()
                    if text:  # Only add non-empty text
                        text_blocks.append(text)
        
        # Concatenate with double newline to preserve paragraph spacing
        answer = '\n\n'.join(text_blocks)
        
        return answer.strip()
```

## 4. Harness Orchestration

### 4.1 ComplianceHarness Class

Core orchestration: loads YAML workflows and manages node execution.

```python
import yaml
import json
from typing import Dict, Any

class ComplianceHarness:
    """Orchestrates workflow execution across action nodes."""
    
    def __init__(self, workflow_config_path: str):
        """
        Initialize harness: load YAML workflow definition, instantiate actions, load data layers.
        
        Args:
            workflow_config_path: Path to harness.yaml (defines workflows and data sources)
        """
        with open(workflow_config_path) as f:
            config = yaml.safe_load(f)
        
        self.workflows = config.get('harness', {}).get('workflows', {})
        self.data_sources = config.get('harness', {}).get('data_sources', {})
        
        # Load handbook index (data layer)
        fca_handbook_config = self.data_sources.get('fca_handbook', {})
        handbook_path = fca_handbook_config.get('artifact')
        self.handbook_index = HandbookIndex(handbook_path)
        
        # Instantiate action classes with harness reference for data access
        self.actions = {
            name: ACTION_REGISTRY[name](self) 
            for name in ACTION_REGISTRY.keys()
        }
```

### 4.2 Workflow Execution

Execute a workflow node-by-node, passing outputs to dependent nodes.

```python
    def execute_workflow(self, workflow_name: str, entity_description: str, question: str) -> Dict[str, Any]:
        """
        Execute named workflow end-to-end.
        
        Args:
            workflow_name: Name from YAML (e.g., 'general_enquiry')
            entity_description: Markdown product description
            question: User's compliance question
        
        Returns:
            Workflow context dict with all node outputs
        """
        workflow = self.workflows.get(workflow_name)
        if not workflow:
            raise ValidationError(f"Workflow '{workflow_name}' not found")
        
        nodes = workflow.get('nodes', [])
        context = {
            'entity_description': entity_description,
            'question': question
        }
        
        # Execute each node in order
        for node in nodes:
            node_name = node['name']
            action_type = node['action']
            node_input = self._prepare_input(node['input'], context)
            node_config = node.get('config', {})
            
            try:
                action = self.actions[action_type]
                output = action.execute(node_input, node_config)
                context[node_name] = output
                self._log_interaction(node_name, 'success')
            except ActionError as e:
                self._log_error(node_name, str(e))
                raise
        
        return context
    
    def _prepare_input(self, input_spec: Any, context: Dict) -> Dict[str, Any]:
        """
        Prepare node input by resolving references to prior node outputs.
        
        Input spec may be:
        - A string (key name): resolve from context
        - A dict: recursively resolve values that reference context
        """
        if isinstance(input_spec, str):
            return context.get(input_spec, input_spec)
        elif isinstance(input_spec, dict):
            result = {}
            for key, value in input_spec.items():
                if isinstance(value, str) and value in context:
                    result[key] = context[value]
                elif isinstance(value, dict):
                    result[key] = self._prepare_input(value, context)
                else:
                    result[key] = value
            return result
        return input_spec
    
    def _validate_node_output(self, output: Any, expected_schema: str) -> bool:
        """Validate node output against expected schema."""
        # Stub: implement schema validation (e.g., jsonschema)
        return True
```

### 4.3 Audit Logging

Log all interactions, messages, and errors for compliance audit trails. Audit logs capture both action execution status and user-facing messages (from § 2.5 User Message Handling).

```python
    def _log_interaction(self, node_name: str, status: str, messages: list = None):
        """
        Log successful node execution with any messages emitted.
        
        Args:
            node_name: YAML node name (e.g., 'retrieve_entries')
            status: Execution status (e.g., 'complete')
            messages: List of message dicts emitted during execution
                     [{'timestamp': ISO8601, 'type': 'status'|'progress'|'complete'|'error'|'warning', 'text': str}]
        """
        timestamp = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')
        log_entry = {
            'timestamp': timestamp,
            'node': node_name,
            'status': status,
            'messages': messages or []
        }
        # Stub: write to interactions.json or audit log (file, database, or event stream)
        with open('interactions.json', 'a') as f:
            f.write(json.dumps(log_entry) + '\n')
    
    def _log_error(self, node_name: str, error_message: str, messages: list = None):
        """
        Log node execution error with any messages emitted before failure.
        
        Args:
            node_name: YAML node name
            error_message: Exception message or error reason
            messages: List of message dicts emitted before error
        """
        timestamp = datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z')
        log_entry = {
            'timestamp': timestamp,
            'node': node_name,
            'status': 'error',
            'error': error_message,
            'messages': messages or []
        }
        # Stub: write to audit log
        with open('interactions.json', 'a') as f:
            f.write(json.dumps(log_entry) + '\n')
```

**Integration with workflow execution:**

The harness manages message flow: actions append to `self._messages` during `execute()`, then the harness drains and logs them:

```python
# In ComplianceHarness.execute_workflow():
for node in workflow_nodes:
    try:
        # Get action and execute (action appends to self._messages during run)
        action = ActionRegistry.get(node['action'])
        result = action.execute(node_input, node['config'])
        
        # Drain messages from action and log with node completion
        messages = action._messages
        action._messages = []  # Reset for next execution if reused
        self._log_interaction(node['name'], 'complete', messages)
        
        # Store result in context for downstream nodes
        context[node['name']] = result
        
    except Exception as e:
        # Log error with any partial messages emitted before failure
        messages = action._messages
        action._messages = []
        self._log_error(node['name'], str(e), messages)
        raise
```

**Audit trail structure (interactions.json):**

```json
{
  "timestamp": "2026-05-16T14:32:45Z",
  "node": "retrieve_entries",
  "status": "complete",
  "messages": [
    {"timestamp": "2026-05-16T14:32:45Z", "type": "status", "text": "Searching Handbook for relevant entries..."},
    {"timestamp": "2026-05-16T14:32:46Z", "type": "progress", "text": "Retrieved 50 candidates from 10438 entries"},
    {"timestamp": "2026-05-16T14:32:47Z", "type": "progress", "text": "Applying regulatory weights to rank results..."},
    {"timestamp": "2026-05-16T14:32:48Z", "type": "complete", "text": "Retrieved 20 ranked entries (0.87 similarity, 2.5x multiplier)"}
  ]
}
```

This structure preserves the full progression of user-facing messages within the audit trail, enabling both live user feedback and post-hoc compliance review.

### 4.4 Tool Integration Entry Point

Expose the harness as a named tool for external LLMs (e.g., Claude).

```python
def fca_handbook_compliance_enquiry(entity_description: str, question: str) -> str:
    """
    Handbook Compliance Enquiry Tool.
    
    Invoke the harness as a tool from external LLM.
    """
    if not question or not entity_description:
        return json.dumps({
            'scope_clarification': 'Do you mean compliance with the **Handbook**? '
                                   'This tool handles Handbook rules specifically.'
        })
    
    try:
        harness = ComplianceHarness('harness.yaml')
        result = harness.execute_workflow('general_enquiry', entity_description, question)
        return json.dumps(result, default=str)
    except ActionError as e:
        return json.dumps({'error': str(e)})

def _is_fca_scope(question: str) -> bool:
    """Check if question is about Handbook."""
    fca_keywords = ['fca', 'handbook', 'cobs', 'prin', 'compliance']
    return any(kw in question.lower() for kw in fca_keywords)
```

---

## 5. Testing

### Unit Test Pattern (Per-Action)

```python
import unittest
from unittest.mock import patch, MagicMock

class TestParseMarkdownAction(unittest.TestCase):
    
    def setUp(self):
        self.action = ParseMarkdownAction()
    
    def test_valid_entity_description(self):
        """Test parsing well-formed markdown."""
        markdown = """# AdviceBot
        ## Features
        - Robo-advice engine
        - Real-time rebalancing
        ## Use Cases
        - Wealth management
        """
        result = self.action.execute({'entity_description': markdown}, {})
        self.assertEqual(result['entity_name'], 'AdviceBot')
        self.assertIn('Robo-advice engine', result['features'])
    
    def test_missing_h1_raises_error(self):
        """Test that missing product name (h1) raises ValidationError."""
        markdown = "## Features\n- Feature 1"
        with self.assertRaises(ValidationError):
            self.action.execute({'entity_description': markdown}, {})

class TestSemanticSearchAction(unittest.TestCase):
    
    @patch('handbook_index.search')
    def test_returns_ranked_entries(self, mock_search):
        """Test that search returns properly formatted RankedEntries."""
        mock_search.return_value = [
            {'entry_id': 'COBS 2.1.1R', 'text': 'Rule text', 'base_similarity': 0.9}
        ]
        action = SemanticSearchAction()
        result = action.execute({
            'entity_features': {'entity_name': 'Test'},
            'question_terms': {'all_search_terms': []},
            'question': 'What rules apply?'
        }, {'Param_top_k': 20})
        self.assertIsInstance(result, list)
        self.assertGreater(len(result), 0)
```

### Integration Test Pattern (Full Workflow)

```python
class TestGeneralEnquiryWorkflow(unittest.TestCase):
    
    def setUp(self):
        self.harness = ComplianceHarness("tests/fixtures/harness.yaml")
    
    def test_full_workflow_execution(self):
        """Test end-to-end execution with fixture data."""
        result = self.harness.execute_workflow('general_enquiry',
            entity_description="# TestProduct\n## Features\n- Feature1\n## Use Cases\n- Use1\n## Decision Authority\nAlgorithm",
            question="What COBS rules apply?"
        )
        self.assertIn('human_review', result)
        self.assertIn('approved', result['human_review'])
```

---

## 6. Implementation Checklist

Follow this order to implement the Harness:

### Phase 1: Foundations
- [ ] Define exception classes (§2.1)
- [ ] Define Action base class (§2.2)
- [ ] Define Claude tool schemas as module constants (§2.4)
- [ ] Define Common Patterns helpers (timestamp, validation, config extraction)

### Phase 2: Data Layer
- [ ] Implement HandbookIndex: load embeddings, detect model, validate dimensions (§3.5)
- [ ] Test HandbookIndex._detect_embedding_key with sample JSON
- [ ] Implement HandbookIndex.search with cosine similarity + weighting

### Phase 3: LLM Actions
- [ ] Implement ParseMarkdownAction (§3.1) — markdown parsing
- [ ] Implement GlossaryLookupAction (§3.2) — glossary lookup
- [ ] Implement EmbedTextAction (§3.3) — embedding API integration
- [ ] Unit test all; verify error messages and messaging are clear

### Phase 4: Embedding & Search Actions
- [ ] Implement SemanticSearchAction with HandbookIndex integration (§3.4)
- [ ] Test semantic search with real handbook embeddings

### Phase 5: LLM-Dependent Actions
- [ ] Implement ClaudeReasoningAction with tool integration (§3.5)
- [ ] Unit test with mocked Claude API

### Phase 6: Orchestration
- [ ] Implement ComplianceHarness class (§4.1–4.3)
- [ ] Implement workflow execution loop with message collection (§4.2)
- [ ] Implement audit logging with message draining (§4.3)
- [ ] Register all actions in ACTION_REGISTRY (§2.3)

### Phase 7: External Integration
- [ ] Implement tool integration entry point (§4.4)
- [ ] Wire Harness into external LLM SDK (Anthropic, OpenAI)
- [ ] Test tool invocation end-to-end

### Phase 8: Testing & Validation
- [ ] Write unit tests for each action (§5)
- [ ] Write integration test for full workflow (§5)
- [ ] Test error handling for each exception type
- [ ] Test audit logging with real and fixture data
- [ ] Test message collection and audit trail completeness

### Phase 9: Deployment
- [ ] Load production Codified_Requirements_Text_And_Embeddings
- [ ] Configure harness.yaml with correct paths and versions
- [ ] Deploy to target environment (direct Python, FastAPI service, or platform)
- [ ] Smoke test with sample queries

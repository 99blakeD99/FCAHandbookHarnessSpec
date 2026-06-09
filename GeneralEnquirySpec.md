# General Enquiry Workflow Specification

This document specifies the **general_enquiry** workflow, the first workflow specification for the Harness. It includes the YAML workflow definition and detailed specifications for each action type used in this workflow.

The general_enquiry workflow handles: product/service description → feature extraction → entry retrieval → compliance reasoning → results presentation.

Other workflows (regulatory_change_analysis, incident_investigation, etc.) will have similar specifications files, following this template.

## Workflow Definition YAML

```yaml
harness:
  data_sources:
    fca_handbook:
      artifact: "Codified_Requirements_Text_And_Embeddings"
      version: "2026-Q1"
      model: "voyage-3-large"
      weighting_config: "weights.yaml (schema in StructuredSearch.md)"
  
  workflows:
    general_enquiry:
      nodes:
        - name: check_preconditions
          action: check_preconditions
          input: {}
          output: PreconditionValidation
          config:
            Param_strict_mode: true
            Param_check_api_keys: true
        
        - name: extract_features
          action: parse_markdown
          input: entity_description
          output: EntityFeatures
          config: {}
        
        - name: check_terminology
          action: glossary_lookup
          input:
            entity_features: extract_features
            question: question
          output: TerminologyMapped
          config: {}
          # NOTE: TERMINOLOGY NORMALIZATION STEP (not retrieval/filtering)
          # Purpose: Enrich question with Glossary canonical terms
          # Output: canonical_terms[] and unmapped_terms[] (metadata only)
          # Does NOT filter or restrict downstream semantic_search results to glossary
          # Glossary terms are used to improve embedding relevance for full-handbook search
        
        - name: embed_question
          action: embed_text
          input:
            question: question
            terminology_mapped: check_terminology
          output: QuestionEmbedding
          config: {}
        
        - name: retrieve_entries
          action: semantic_search
          input:
            entity_features: extract_features
            question_embedding: embed_question
          output: RankedEntries
          config:
            Param_top_k: 20
            Param_source_version: "2026-Q1"
          # NOTE: FULL-HANDBOOK RETRIEVAL STEP (not glossary-restricted)
          # Searches all entries in Codified_Requirements_Text_And_Embeddings
          # Input: question_embedding (enriched with canonical terms from check_terminology)
          # Applies: Cosine similarity + regulatory weighting (binding authority, hierarchy, importance, source)
          # Output: top_k ranked entries by final_score (semantic + regulatory combination)
        
        - name: analyze_compliance
          action: claude_reasoning
          input:
            entity_features: extract_features
            entries: retrieve_entries
          output: ComplianceAnalysis
          config:
            Param_tools:
              - citation_formatter
              - audit_logger
            Param_prompt_template: "fca-compliance-analyst.md"
```

**Naming convention:** 
- Fields prefixed with `Param_` are configurable parameters defined in the Action Specification below. Changing these values will affect behavior (if implemented in Python). 
- Fields without the prefix are structural: `name` (node identifier), `action` (action type), `input`/`output` (data flow).
- The implementation method (regex, embeddings, LLM, etc.) is specified explicitly in each action's **Process** section below for transparency and auditability.

Each node follows the same structure: `name`, `action`, `input` (if applicable), `output`, and `config` (containing `Param_*` parameters). Action-specific fields are normalized under `config`, making the structure consistent for the Python execution loop.

---

## Data Flow Architecture: Terminology Normalization vs. Full-Handbook Retrieval

A critical architectural distinction in the general_enquiry workflow separates **terminology normalization** from **full-handbook retrieval**. Misunderstanding this distinction can lead to incorrect implementation choices (e.g., restricting semantic search to glossary entries).

### The Core Distinction

**Terminology Normalization** (glossary_lookup node):
- **Purpose**: Enrich the user's question with Glossary canonical terms
- **Not a retrieval or filtering step**
- **Output**: Metadata (`canonical_terms`, `unmapped_terms`) that augments the search query
- **LLM role**: Extract regulatory concepts from user question and product description, then map to Glossary
- **Result**: Enhanced question embedding that incorporates official regulatory terminology

**Full-Handbook Retrieval** (semantic_search node):
- **Purpose**: Find relevant entries across all records in Codified_Requirements_Text_And_Embeddings
- **Not glossary-restricted**, despite glossary enrichment from prior step
- **Input**: Embedding vector that incorporates canonical terms (for semantic relevance)
- **Data searched**: All entries in the codified requirements dataset
- **Ranking**: Regulatory weighting applied (binding authority, hierarchy, importance, source authority)
- **Result**: Top-k entries from the full dataset, ranked by combined semantic + regulatory score

### Data Flow Diagram

```
┌─────────────────────────────────────┐
│ User Input                          │
│ - Markdown product description      │
│ - Compliance question               │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ [1] parse_markdown                  │
│ Extract: entity_type, features,     │
│          use_cases, data_handled    │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ [2] glossary_lookup (NORMALIZATION) │
│ LLM: Extract concepts from question │
│      + product description          │
│ LLM: Map concepts → Glossary    │
│ Output: canonical_terms[]           │
│         unmapped_terms[]            │
│ (NOT filtering, just enrichment)    │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ [3] embed_text (DETERMINISTIC)      │
│ Construct enriched text:            │
│   question + canonical_terms +      │
│   unmapped_terms (space-separated)  │
│ Call embedding API (Voyage/OpenAI)  │
│ Output: question_embedding (vector) │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────┐
│ [4] semantic_search (FULL-HANDBOOK RETRIEVAL)       │
│ Input: question_embedding (enriched with canonical) │
│ Search: Codified_Requirements_Text_And_Embeddings    │
│         (all entries, NOT glossary-filtered)        │
│ Algorithm: Cosine similarity + regulatory weighting │
│ Output: top_k ranked entries                        │
│         (entry_id, text, similarity, weights, score)│
└────────────┬────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│ [5] claude_reasoning (LLM)          │
│ Input: Retrieved entries            │
│        Entity features              │
│ LLM: Reason over entries, cite      │
│      verbatim Handbook text         │
│ Output: Compliance analysis with    │
│         citations and reasoning log │
└─────────────────────────────────────┘
```

### Why This Distinction Matters

**Implementation Pitfall 1**: Restricting semantic_search to glossary entries
- **Wrong**: "We normalized to glossary terms, so search only the glossary"
- **Correct**: Glossary terms enrich the embedding for relevance; semantic search still covers full handbook
- **Impact**: Ignoring this would miss Main Handbook, Instruments, Forms, Technical Standards entries that are semantically related but use different terminology

**Implementation Pitfall 2**: Assuming glossary_lookup filters results
- **Wrong**: "After glossary_lookup, we only see glossary-relevant rules"
- **Correct**: glossary_lookup outputs enrichment metadata; downstream search is independent
- **Impact**: Could cause incorrect scoping of compliance analysis

**Why it's designed this way**: 
1. **Terminology normalization is narrow and LLM-friendly**: Claude maps concepts to official language
2. **Retrieval is deterministic and auditable**: Cosine similarity + regulatory weighting produce reproducible results
3. **Full coverage is preserved**: Handbook entries that use different terminology (e.g., Main Handbook vs. Glossary definitions) are still found via semantic similarity
4. **Citation accuracy is maintained**: Claude reasons over actual handbook entries, not a pre-filtered subset

### Concrete Example

**User's question**: "Does my chatbot need to follow deposit-taking rules?"

**Glossary_lookup output** (terminology normalization):
- Canonical terms: `["advance payment", "payment service"]`
- Unmapped terms: `["chatbot", "deposit-taking"]`

**Embed_text** (enrichment):
- Enriched text: "Does my chatbot need to follow deposit-taking rules? advance payment payment service chatbot deposit-taking"
- Embedding captures both semantic content and canonical terms

**Semantic_search** (full-handbook retrieval):
- Searches across all Handbook entries
- Top results might include:
  - COBS 1.2.1R (Main Handbook): "A firm must treat customers fairly"
  - GLOSSARY: "advance payment" definition
  - CONC (Glossary): "deposit" vs "advance payment" distinction
  - PSD2 instruments: Payment service definitions
- All entries are semantically relevant AND weighted by regulatory authority
- No results are filtered out because they're from glossary (glossary entries are *included* if semantically relevant)

**Claude_reasoning** (bounded to retrieved results):
- Reasons over the actual top-20 entries
- Must cite verbatim text from those entries
- Never invents guidance beyond what's in the retrieved set

---

### Implementation Checklist

When implementing, verify:
- [ ] **glossary_lookup**: Maps user concepts → Glossary terms, outputs `canonical_terms` + `unmapped_terms`
- [ ] **embed_text**: Constructs enriched text (question + both term arrays) and embeds it
- [ ] **semantic_search**: Searches `Codified_Requirements_Text_And_Embeddings` (all entries), NOT a glossary-filtered subset
- [ ] **semantic_search**: Uses the enriched embedding from embed_text (which incorporates canonical terms)
- [ ] **claude_reasoning**: Receives all top-k results, not pre-filtered by whether they're glossary entries
- [ ] **Audit logging**: Records both the normalized terms (from glossary_lookup) and the retrieved entries (from semantic_search) for traceability

---

## Action Specifications

Each action type is implemented by a Python class that conforms to an exact specification. This ensures auditability: any auditor or implementer can read the spec and verify that the Python code matches it.

**Message Protocol**: Each action includes a **Messaging** section specifying what status updates the Harness sends to the user during execution. Messages serve three purposes: (1) feedback on progress, (2) debugging/transparency on workflow state, (3) audit trail documentation. All messages are logged to both user output and `interactions.json` for compliance review.

**Message Destinations**:
- **User output**: Printed to stdout (or streamed via websocket in service deployments). Includes symbol prefix (✓, ⚠, ℹ, ✗, →) based on message_type.
- **Audit trail (interactions.json)**: Unstyled text + timestamp + message_type. Always appended, never filtered.
- **Interactive prompts**: When used, block execution, wait for user confirmation. Both action and user response logged.

**Note**: Message text below is unstyled; the implementation adds symbols (✓, ⚠, ℹ, ✗, →) based on `message_type` ("complete", "warning", "status", "error", "progress"). Pass text without symbols to `self.message(text, message_type)`.

### check_preconditions

**Purpose**: Validate that the Harness has been properly configured and all required artifacts, dependencies, and credentials are available before executing the workflow. Fails fast with clear error messages if preconditions are not met.

**Input**: None (validation runs at harness startup)

**Output**:
- `PreconditionValidation` (object) with fields:
  - `valid` (boolean): Whether all preconditions are met
  - `checks_performed` (array of strings): List of checks executed
  - `missing_items` (array of objects): Items that failed validation:
    - `item_name` (string): Name of missing/invalid item
    - `item_type` (string): "file", "env_var", "dependency", "config"
    - `location` (string): Expected location or env var name
    - `remediation` (string): How to fix (e.g., "Set ANTHROPIC_API_KEY=...")

**Configuration** (from YAML node config):
- `Param_strict_mode` (boolean, default true): If true, fail on any missing item. If false, warn but continue (not recommended).
- `Param_check_api_keys` (boolean, default true): Validate API key environment variables are set.

**Process** (deterministic filesystem and environment checks; no LLM):
1. Check Codified_Requirements_Text_And_Embeddings file exists and is valid JSON
2. Check weights.yaml exists and is valid YAML
3. Check prompt template files exist (harness/prompts/fca-compliance-analyst.md)
4. If Param_check_api_keys=true: Check ANTHROPIC_API_KEY environment variable is set (required for claude_reasoning action)
5. Check optional embedding API key (OPENAI_API_KEY or VOYAGE_API_KEY) if embedding model requires it
6. Return PreconditionValidation with all_checks_passed and detailed missing_items list

**Validation**:
- File paths: Absolute paths or config-relative paths (e.g., `./Codified_Requirements_Text_And_Embeddings.json`)
- JSON/YAML files: Parse to verify valid format
- Environment variables: Non-empty string values

**Error Handling**:
- Missing file: Add to missing_items with remediation "Download Codified_Requirements_Text_And_Embeddings from [location] or generate from script [script]"
- Invalid JSON/YAML: Add to missing_items with remediation "File is corrupted. Re-download or regenerate."
- Missing API key: Add to missing_items with remediation "Set ANTHROPIC_API_KEY environment variable"
- Param_strict_mode=true and any missing_items: Return valid=false; workflow halts
- Param_strict_mode=false: Return valid=true with warnings; workflow continues (not safe for compliance)

**Messaging** (user-facing updates):
- On start: "Checking Harness preconditions..." (message_type: "status")
- On each successful check: "[✓] Codified_Requirements_Text_And_Embeddings found" (message_type: "complete")
- On each failed check: "[✗] ANTHROPIC_API_KEY not set. Set: export ANTHROPIC_API_KEY=sk-..." (message_type: "error")
- On complete (all pass): "Harness preconditions verified. Ready to execute." (message_type: "complete")
- On complete (failures + strict mode): "Precondition failures detected. Cannot proceed." (message_type: "error")
- Message destination: User (all types)

**Notes**:
- This action runs once per harness initialization, not per workflow invocation
- Deterministic and auditable; no LLM involvement
- Essential for catching configuration issues early before users spend time on missing setup
- Clear remediation messages assume users are running locally; service deployments can customize remediation text

### parse_markdown

**Purpose**: Extract and structure entity information from markdown-formatted entity description.

**Input**: 
- `entity_description` (string): Markdown-formatted text describing an entity (product, service, business line, etc.)

**Output**: 
- `EntityFeatures` (object) with fields:
  - `entity_name` (string): Name of the entity
  - `entity_type` (string): Category or classification (e.g., "advisory platform", "robo-advisor", "trading system")
  - `features` (array of strings): Key capabilities and features
  - `use_cases` (array of strings): Intended use and client segments
  - `client_interaction` (string): How clients interact (manual, automated, hybrid)
  - `data_handled` (array of strings): Types of sensitive data (e.g., "client portfolios", "trading instructions")
  - `decision_authority` (string): Who makes final decisions (algorithm, advisor, client)

**Process** (LLM extraction from structured text):
1. Call Claude with the entity_description and a system prompt describing the required EntityFeatures schema
2. Claude extracts and structures the information:
   - Entity name, type, features, use cases, client interaction, data handled, decision authority
   - Claude handles flexible markdown formatting (doesn't require rigid structure)
   - Claude infers missing fields where contextually appropriate (e.g., decision_authority from description of how the system works)
3. Validate extracted fields match schema requirements
4. Return structured EntityFeatures object

**Validation**:
- `entity_name`: Non-empty string, max 200 chars
- `features`: Non-empty array, min 1 item, max 50 items, each item max 500 chars
- `use_cases`: Non-empty array, min 1 item, max 20 items
- `decision_authority`: One of: "algorithm", "advisor", "client", "hybrid"
- If any required field missing after extraction: raise error with field name

**Error Handling**:
- Non-markdown file detected: Return error "You need to convert your file to markdown format first. This is normally a simple process but it is better if you keep control of it. Ask your LLM to help."
- LLM extraction fails: Return error "Unable to extract product features from your description. Ensure the description includes entity name, features, and use cases."
- Missing required fields: Return error "Missing required field: {field_name}"
- Field validation fails: Return error "Invalid value for {field_name}: {reason}"
- LLM service unavailable: Return error "LLM service unavailable. Unable to extract features."

**Messaging** (user-facing updates):
- On start: "Extracting product features from your description..." (message_type: "status")
- On complete: "Extracted {entity_name}: {feature_count} features, {use_case_count} use cases identified" (message_type: "complete")
- On error: "Could not extract product features. Ensure your description includes entity name, key features, and intended use cases." (message_type: "error")
- Message destination: User (all types)

### glossary_lookup

**Purpose**: Map user question terminology to Handbook canonical terms via the Glossary. The LLM interprets user language and regulatory meaning to identify which Glossary terms are relevant. Handles terminology mismatches where user's language differs from regulatory language (e.g., "money handling" → "cash", "deposit" → "advance payment"). Enriches semantic search with canonical terminology for improved recall.

**Input**:
- `entity_features` (EntityFeatures object): Structured product information
- `question` (string): User's compliance question
- `fca_glossary` (array of objects): Glossary entries from Codified_Requirements_Text_And_Embeddings with fields:
  - `term` (string): Canonical glossary term
  - `definition` (string): Verbatim definition from Handbook

**Configuration** (from YAML node config):
- None (Glossary provided at harness startup)

**Output**:
- `TerminologyMapped` (object) with fields:
  - `canonical_terms` (array of strings): Glossary canonical terms the LLM identified as relevant to user's question
  - `unmapped_terms` (array of strings): User's original terms not mapped to glossary (fallback for semantic search)
  - `all_search_terms` (array of strings): Concatenation of canonical_terms + unmapped_terms (convenience for downstream embedding)

**Process** (LLM reasoning with glossary grounding):
1. LLM receives user question, entity_features, and complete Glossary
2. LLM identifies key financial/regulatory concepts in the question and product description
3. LLM maps user concepts to canonical Glossary terms where semantically appropriate
4. If a concept has no clear glossary match: record as unmapped_term (preserve user's original phrasing)
5. Compute all_search_terms = canonical_terms + unmapped_terms (space-separated array)
6. Return TerminologyMapped with canonical_terms, unmapped_terms, and all_search_terms

**Fallback behavior when mapping incomplete:**
- No error is raised; unmapped terms are passed through unchanged
- embed_text receives both canonical_terms and unmapped_terms for enriched embedding
- This preserves coverage for regulatory language not in the glossary

**Validation**:
- `entity_features`: Must match EntityFeatures schema
- `question`: Non-empty string, min 5 chars, max 1000 chars
- Glossary: Must be available in Codified_Requirements_Text_And_Embeddings

**Error Handling**:
- Glossary unavailable: Return error "Handbook Glossary not available. Unable to perform terminology mapping."
- LLM fails to map: Return TerminologyMapped with empty canonical_terms and all terms unmapped (not an error—embed_text proceeds with original terms)

**Messaging** (user-facing updates):
- On start: "Mapping terminology to Glossary..." (message_type: "status")
- On complete: "Mapped {mapped_count} terms to canonical language ({unmapped_count} unmapped)" (message_type: "complete")
- If no terms mapped: "No glossary matches found; proceeding with original question terms" (message_type: "info")
- Message destination: User (all types)

**Notes**:
- This node enriches the question with canonical terminology but does not modify the user's original question
- Unmapped terms (not found in Glossary) are passed downstream as fallback; this preserves coverage
- The LLM's mapping is constrained to entries actually present in the Glossary; no invented terms
- Canonical terms and unmapped terms are combined in embed_text for the enriched embedding used by semantic_search

### embed_text

**Purpose**: Embed the user's compliance question using the same embedding model as Codified_Requirements_Text_And_Embeddings. The embedding incorporates both the original question and Glossary-mapped canonical terms to improve search recall. This is foundational: embeddings from different models are incompatible and will produce meaningless search results.

**Input**:
- `question` (string): User's compliance question
- `terminology_mapped` (object): Output from glossary_lookup containing:
  - `canonical_terms` (array of strings): Terms mapped to Glossary
  - `unmapped_terms` (array of strings): Terms not found in glossary

**Output**:
- `QuestionEmbedding` (vector): Embedded question vector (dimensionality matches handbook embeddings)

**Configuration** (from YAML node config):
- None (uses embedding model from harness data_sources.fca_handbook.model)

**Process** (deterministic embedding only; no LLM):
1. Construct enriched text: question + canonical_terms + unmapped_terms (space-separated)
2. Detect embedding model from harness configuration (voyage-3-large or text-embedding-3-large)
3. Call embedding API with enriched text
4. Return embedding vector
5. Embedding vector is passed to semantic_search for cosine similarity computation

**Validation**:
- `question`: Non-empty string, max 1000 chars
- Embedding model configured and available in harness

**Error Handling**:
- Embedding API unavailable: Return error "Embedding service unavailable. Check API credentials (VOYAGE_API_KEY or OPENAI_API_KEY) and network connectivity."
- Embedding API rate limit: Return error "Embedding service rate limited. Retry after delay."
- Model mismatch: If embedding model differs from handbook embeddings, results will be meaningless but no error raised (caught by search result quality checks downstream)

**Messaging** (user-facing updates):
- On start: "Embedding your question..." (message_type: "status")
- On complete: "Question embedded ({embedding_model}, {embedding_dimensions} dimensions)" (message_type: "complete")
- On error: "Embedding service unavailable. Check API credentials and network connectivity." (message_type: "error")
- Message destination: User (all types)

**Notes**:
- This node must use the same embedding model as Codified_Requirements_Text_And_Embeddings (configured at harness startup)
- Embedding happens per question (every user query)
- Handbook embedding happens once at startup (see EmbeddingModel.md)
- If model is switched after harness startup, handbook and question embeddings become incompatible; harness must restart to reload handbook

### semantic_search

**Purpose**: Query `Codified_Requirements_Text_And_Embeddings` for regulatory entries matching entity features and question terminology. Apply regulatory weighting to rank results by binding authority and relevance. Uses Glossary-mapped terminology (from glossary_lookup) alongside user's original question to improve recall on terminology mismatches.

**Input**:
- `entity_features` (EntityFeatures object): Structured output from parse_markdown
- `question_embedding` (vector): Enriched embedding from embed_text node (includes original question + mapped terminology)

**Configuration** (from YAML node config):
- `Param_top_k` (integer): Number of results to return (default 20, range 1–100)

**Implementation Constants** (hardcoded, not parameterized):
- Data source: `Codified_Requirements_Text_And_Embeddings` (from harness configuration)
- Weighting: Always enabled (regulatory weighting always applied)

**Output**:
- `RankedEntries` (array of objects) with fields:
  - `entry_id` (string): Handbook entry identifier (e.g., "COBS 2.1.1R")
  - `text` (string): Verbatim entry text from source
  - `base_similarity` (number, 0–1): Cosine similarity from embeddings
  - `weight_factors` (object):
    - `entry_type_weight` (number): Binding authority multiplier
    - `hierarchy_multiplier` (number): Position in handbook structure
    - `importance_multiplier` (number): Regulatory criticality
    - `piece_weight` (number): Section weighting
  - `final_score` (number): base_similarity × entry_type_weight × hierarchy_multiplier × importance_multiplier × piece_weight
  - `source_version` (string): Version of Codified_Requirements_Text_And_Embeddings used (e.g., "2026-Q1")

**Process** (deterministic weighting only; embedding delegated to embed_text node; no LLM):
1. Receive pre-computed question_embedding from embed_text node (enriched with canonical terms and unmapped terms; ensures question and handbook use same embedding model)
2. Compute cosine similarity (numpy dot product) between question_embedding and entry embeddings from pre-loaded Codified_Requirements_Text_And_Embeddings
3. Sort by cosine similarity, select top 50 candidates (deterministic filtering before weighting)
4. Apply regulatory weighting algorithm (see StructuredSearch.md) to rank candidates by final_score: base_similarity × entry_type_weight × hierarchy_multiplier × importance_multiplier × piece_weight
5. Sort by final_score descending (deterministic ranking)
6. Return top_k results with full weight_factors breakdown for auditability

**Validation**:
- `entity_features`: Must contain entity_type and features (validate EntityFeatures structure)
- `question_embedding`: Must be a vector of correct dimensionality (matches handbook embeddings)
- `Param_top_k`: Integer in range [1, 100], default 20
- Harness configuration: weights.yaml (schema in StructuredSearch.md) must be present and valid; if missing or invalid: raise error with config path

**Error Handling**:
- Data source unavailable: Return error "Handbook data (Codified_Requirements_Text_And_Embeddings) is temporarily unavailable. Escalate to compliance_team@company.com"
- Invalid weights.yaml (schema in StructuredSearch.md): Return error "Regulatory weights configuration invalid at {path}: {reason}"
- No results found: Return empty array with informational log (not an error—some queries legitimately have no matches)

**Messaging** (user-facing updates):
- On start: "Searching Handbook for relevant entries..." (message_type: "status")
- On progress (after candidate filtering): "Retrieved {candidate_count} candidates from {total_entries} entries" (message_type: "progress")
- On progress (during weighting): "Applying regulatory weights to rank results..." (message_type: "progress")
- On complete: "Retrieved {top_k} ranked entries ({top_1_score:.2f} similarity, {top_1_weight:.1f}x multiplier)" (message_type: "complete")
- If no results: "No matching entries found for your question" (message_type: "warning")
- On error: "Search failed. Check Handbook data and weights configuration." (message_type: "error")
- Message destination: User (all types; progressive updates)

### claude_reasoning

**Purpose**: Invoke Claude to reason over retrieved entries and product features, producing a compliance analysis with citations and reasoning logs. **Critical constraint: Claude must cite relevant Handbook entries verbatim—not paraphrase, summarize, or invent compliance guidance.** This prevents hallucination and ensures all compliance claims are traceable to authoritative source text.

**Input**:
- `entity_features` (EntityFeatures object): Structured product information
- `entries` (RankedEntries array): Top-ranked entries from semantic_search

**Configuration** (from YAML node config):
- `Param_tools` (array of strings): Internal tool names available to Claude (e.g., ["citation_formatter", "audit_logger"])
- `Param_prompt_template` (string): Path to prompt template file (e.g., "fca-compliance-analyst.md")

**Prompt Template Schema**:
The template file (e.g., `fca-compliance-analyst.md`) must be valid Markdown with Jinja2 template syntax. It will be rendered with the following variables and must use them in the template:

- `{{ entity_features }}` (dict): Structured product information in JSON format
  - Keys: entity_name, entity_type, features[], use_cases[], client_interaction, data_handled[], decision_authority
  - Rendered as JSON string for Claude to read
- `{{ ranked_entries }}` (array): Top-k entries retrieved by semantic_search, rendered as JSON
  - Each entry includes: entry_id, text (verbatim from Handbook), base_similarity, weight_factors, final_score
  - Used by Claude to cite entries verbatim
- `{{ question }}` (string): User's original compliance question
- Additional context variables (if needed): `{{ handbook_version }}`, `{{ timestamp }}`

Template example usage:
```markdown
# Handbook Compliance Analysis

You are analyzing this product against regulatory requirements.

## Product Information
{{ entity_features }}

## User Question
{{ question }}

## Relevant Handbook Entries
{{ ranked_entries }}

## Task
[Instructions for Claude on how to reason over entries and cite verbatim]
```

**Output**:
- `ComplianceAnalysis` (object) with fields:
  - `answer` (string): Claude's reasoning and conclusions
  - `citations` (array of objects):
    - `entry_id` (string): Handbook entry cited (e.g., "COBS 2.1.1R")
    - `cited_text` (string): Exact verbatim text from Handbook
    - `context` (string): Sentence or paragraph from entry showing why it applies
    - `binding_level` (string): "R", "G", "E", or "D" (from entry_id suffix)
  - `reasoning_log` (object):
    - `reasoning_chain` (string): Claude's step-by-step logic ("entry X applies because feature Y matches criterion Z")
    - `gaps_identified` (array of strings): Edge cases or areas of uncertainty
    - `confidence_score` (number, 0–1): Model's confidence in the analysis
  - `refinement_suggestions` (array of strings): Actionable suggestions for refining the analysis. Claude identifies what additional information (entity details, product features, scope clarification) would increase confidence or resolve identified gaps. Deployed as-is to users: "To refine this analysis further, please provide: [suggestion1], [suggestion2], [suggestion3]"
  - `timestamp` (string): ISO 8601 timestamp of analysis
  - `handbook_version` (string): Version of Handbook this analysis is based on (e.g., "Q1 2026" or "2026-01-15"), obtained from JSON metadata
  - `version_notice` (string): "This analysis is based on Handbook [version]. If a newer version has been published, alert your IT department to update the compliance harness."

**Process** (LLM reasoning with deterministic citation validation):
1. Load Jinja2 prompt template from file (e.g., `harness/prompts/fca-compliance-analyst.md`)
2. Construct system prompt by rendering template with entity_features and entries (deterministic). Entries include entry_id (authoritative identifier) for each RankedEntry. Template syntax: `{{ entity_features }}`, `{{ ranked_entries }}`
3. Enable prompt caching for stable context (deterministic optimization)
4. Call Claude API (LLM reasoning only — do not modify retrieved entries):
   - **Constraint: Citations must be verbatim text from retrieved entries only. Claude must not paraphrase, summarize, or invent compliance guidance.**
   - System prompt (cached) — includes RankedEntries with entry_id fields
   - User message: question + available internal tools description
   - Internal tools: citation_formatter, audit_logger (structured output enforcement; citation_formatter must extract entry_id from tool call)
   - Model: claude-opus with thinking tokens enabled
   - Temperature: 0.5 (deterministic but thoughtful)
   - Max tokens: 2000
   - Extended thinking: enabled
5. Parse Claude's response to extract reasoning_log, citations (by entry_id from tool_use), and answer text
6. **Refinement suggestions:** Based on gaps_identified and confidence_score, Claude also generates 2–4 actionable refinement suggestions. These identify information that would increase confidence or resolve identified gaps (e.g., "Clarify which client segments you serve", "Specify decision-making process", "Confirm data retention period"). Refinement suggestions are extracted into `refinement_suggestions` array and deployed directly to users without filtering or modification.
7. Validate each citation using deterministic entry_id lookup in RankedEntries array (not regex, not LLM-based)
8. Construct ComplianceAnalysis object with full traceability, including refinement_suggestions

**Validation**:
- `entity_features`: Must match EntityFeatures schema
- `entries`: Non-empty array of RankedEntries
- `Param_prompt_template`: File must exist, be valid markdown, use valid Jinja2 syntax (`{{ variable }}`)
- `Param_tools`: Array of valid tool names (must be recognized by Claude API)
- Each citation in output: entry_id must exist in the RankedEntries array; cited_text must match the corresponding entry's text field exactly
- Citation lookup: Use entry_id to find the entry in RankedEntries; do NOT use regex to extract or validate entry_id
- If citation validation fails: flag in output with error "Citation {entry_id} not found in retrieved entries" or "Citation {entry_id} text mismatch: expected '[actual_text]' but model cited '[cited_text]'"

**Error Handling**:
- Prompt template missing: Return error "Prompt template not found at {path}"
- Claude API unavailable: Return error "LLM service unavailable. Unable to reason over entries."
- Citation validation fails: Include warning in ComplianceAnalysis.reasoning_log; do not halt (compliance analyst should review)
- Empty reasoning output: Return error "LLM produced no usable reasoning output"

**Messaging** (user-facing updates):
- On start: "Analyzing compliance requirements with Claude..." (message_type: "status")
- On Claude thinking: "Claude is reasoning (thinking tokens enabled)..." (message_type: "progress")
- On complete: "Analysis complete. {citation_count} entries identified, confidence: {confidence_score:.0%}" (message_type: "complete")
- Refinement suggestions: Included in ComplianceAnalysis output; no separate messaging needed (deployment layer displays them as part of answer)
- On citation warning: "Citation validation warning: {warning_message}" (message_type: "warning")
- On error: "LLM analysis failed. Please try again." (message_type: "error")
- Message destination: User (all types; progressive transparency)

**Citation Validation** (sub-process):
1. For each citation in Claude's response:
   - Extract entry_id and cited_text
   - Find entry in source entries by entry_id
   - Check if cited_text appears verbatim in entry.text (allow for whitespace normalization)
   - If mismatch: log warning with both texts for compliance review


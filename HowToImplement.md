# How to Implement the Harness

## Intro

This reference implementation creates the fca_handbook_general_enquiry.py app.

It uses the following as defaults:

- Claude Opus to program
- Claude Opus as the internal reasoning LLM
- voyage-3-large as the embedding model for user input (to match the embeddings in data/Codified_Requirements_Text_And_Embeddings.json)
- venv as the development environment
- python as the programming language

If you have other preferences, ask your LLM to make appropriate adjustments to the prompts below.

## Steps

- First clone this repo into your codebase or organization.

- Then download [Codified_Requirements_Text_And_Embeddings.json](https://github.com/99blakeD99/FCA_HandbookComplianceAIharnessOverview/releases/download/data260603/Codified_Requirements_Text_And_Embeddings.json). If this is not available at startup, the app will load from the remote version and this may lead to slightly slower start-up.

- Then go through the Prompts set out below. 

- Then expose a beta version to your Compliance teams, get feedback, decide action.

- Then choose your preferred Deployment method.

## Prompts

Copy the following long prompts into Claude Opus one at a time.

### Prompt 1: Setup Prerequisites

**Prompt Claude Opus**:

```
You are executing Prompt 1 (Setup Prerequisites) of the Handbook Compliance Agent Harness.

**CONTEXT: Overall Workflow**

Understand the overall workflow in HowToImplement.md (in your cloned repo). This clarifies how this prompt fits into the 6-prompt sequence, what prior prompts produce (your inputs), and what downstream prompts expect (your outputs).

**CONFIGURATION DEFAULTS** (used across all prompts):
- LLM Model: claude-opus-4-8 (latest Claude Opus)
- Embedding Model: voyage-3-large (Voyage AI) — **REQUIRED** (data file contains only Voyage embeddings; do not support alternative embedding providers)
- Embedding Dimension: 1024 (Voyage 3-Large)
- Top-K Results: 20
- Similarity Threshold: 0.5
- API Retry: 3 attempts with exponential backoff (2x factor)

**ERROR HANDLING POLICY**:
- Prompts 1-5 (setup, config, features, search): FAIL-FAST — validation errors halt immediately with no recovery
- Prompt 6 (reasoning): WARN-AND-CONTINUE — warnings logged but processing continues; user reviews warnings in results

CONTEXT:
You will generate setup files and scripts needed before implementation.

The downloaded Codified_Requirements_Text_And_Embeddings.json must be made available 
to the Harness. It will be loaded at startup using either:
  - A config file path (e.g., ./data/Codified_Requirements_Text_And_Embeddings.json)
  - An environment variable (e.g., HANDBOOK_PATH=/path/to/file)
See ProgramSpec.md for configuration details.

DELIVERABLES:

1. (Optional) Download Script (bash):
   - Create a bash script that downloads Codified_Requirements_Text_And_Embeddings.json from GitHub releases for faster startup (~5 sec load time instead of ~7-8 sec with remote fetch)
   - Location: ./data/Codified_Requirements_Text_And_Embeddings.json
   - Include verification (check file size ~247MB)
   - Note: In Prompt 2, the HandbookIndex will have built-in adaptive loading — if the file is missing, it will automatically fetch from the remote source on first startup

2. Project Structure Setup Script (bash):
   - Create directories: ./data, ./nodes, ./nodes/__init__.py, ./tests, ./tests/__init__.py, ./tests/fixtures

3. Test Fixtures File (JSON):
   - Generate tests/fixtures/sample_handbook.json with 3 entries:
     - One R (Rule) entry (level 1, Main Handbook, regulatory_score 8)
     - One G (Guidance) entry (level 2, Main Handbook, regulatory_score 5)
     - One Glossary entry (level 3, Glossary, regulatory_score 3)
   - Each with complete fields: id, header, title, regulatory_content, level, piece, regulatory_score, url, embedding
   - Use 10-dimensional pre-computed embeddings
   - Include _meta section with metadata

4. Regulatory Weights File (YAML):
   - Generate weights.yaml in project root with this structure:
   ~~~yaml
   metadata:
     source: "StructuredSearch.md § Current Weights"
     last_synced: <ISO-8601 date>
   
   rule_type_weights:
     RULES: 1.0
     GUIDANCE: 0.8
     EVIDENTIAL: 0.6
     UNCLASSIFIED: 0.8
     DELETED: 0.1
   
   importance_multipliers:
     high: 1.3      # Scores 7-12
     medium: 1.1    # Scores 4-6
     low: 0.7       # Scores 1-3
   
   hierarchy_multipliers:
     1: 1.2         # Top-level
     2: 1.0         # Subsection
     3: 0.9         # Nested (3+)
   
   piece_weights:
     "Main Handbook": 1.2
     "Glossary": 0.95
     "Instruments": 1.0
     "Forms": 0.85
     "Technical Standards": 0.9
     "Level3Materials": 0.85
   ~~~

5. Setup Instructions (setup_env.sh):
   - Create bash script: `python3 -m venv venv`, then `source venv/bin/activate`, then `pip install pytest pyyaml numpy anthropic voyageai flask`
   - Also create SETUP.md with step-by-step overview and reference to setup_env.sh

VALIDATE:
- Run the setup scripts successfully
- weights.yaml is valid YAML with all 5 required sections
- sample_handbook.json is valid JSON with 3 entries and 10-dimensional embeddings
- All directories created (data/, nodes/, tests/, tests/fixtures/)
- setup_env.sh is executable and creates venv with required packages
- SETUP.md provides overview and references setup_env.sh
- (Optional) If you created the download script, verify it downloads to ./data/Codified_Requirements_Text_And_Embeddings.json and checks file size ~247MB
- (Optional) If you downloaded the handbook file manually, verify it exists at ./data/Codified_Requirements_Text_And_Embeddings.json
```

---

### Prompt 2: Config & Data Loading

**Prompt Claude Opus**:

```
You are implementing Prompt 2 (Config & Data Loading) of the Handbook Compliance Agent Harness in Python.

**CONTEXT: Overall Workflow**

Understand the overall workflow in HowToImplement.md (in your cloned repo). This clarifies how this prompt fits into the 6-prompt sequence, what prior prompts produce (your inputs), and what downstream prompts expect (your outputs).

**TEST FRAMEWORK & PATTERNS**:
- Framework: pytest (pytest.ini in project root)
- Mocking Claude API: Use `unittest.mock.patch` on `anthropic.Anthropic.messages.create`
  - Example: `@patch('anthropic.Anthropic.messages.create')`
  - Return dict matching expected output schema (not production data)
- Mocking Embedding API: Use `unittest.mock.patch` on voyageai client
- Tests with pre-computed embeddings: Use sample_handbook.json (no API calls)
- Test file naming: `tests/test_<stage>_<module>.py`
  - Examples: `tests/test_prompt2_config_and_data.py`, `tests/test_prompt3_preconditions.py`

**CONTEXT**

Prompt 1 (Setup) has been completed. You now have:
- ./data/Codified_Requirements_Text_And_Embeddings.json (10,438 handbook records)
- ./tests/fixtures/sample_handbook.json (3 test entries with 10-dim embeddings)
- ./weights.yaml (regulatory weights configuration)
- All required directories created

This prompt implements the configuration, data loading, and testing infrastructure.

Reference documents:
- Handbook spec: StructuredSearch.md § Weighting Factors
- Implementation patterns: ProgramSpec.md § 2.1 Exception Hierarchy

**PROMPT 2 IMPLEMENTATION MODULES**

Implement the following four Python modules:

**1. exception_hierarchy.py** — Domain-specific exceptions:
- ActionError (base exception for all action errors)
- ValidationError (input validation failed)
- DataSourceUnavailableError (required data source unavailable)
- CitationValidationError (citation text does not match source)

**2. config.py** — Configuration with these classes:

APIConfig:
- anthropic_api_key(): Load ANTHROPIC_API_KEY, raise ValidationError if missing
- embedding_api_key(): Load VOYAGE_API_KEY ONLY (no fallback to other providers; Codified_Requirements_Text_And_Embeddings.json contains only Voyage embeddings), raise ValidationError if missing
- embedding_model(): Load EMBEDDING_MODEL, default to "voyage-3-large" (do not allow override to other models)

RegulatoryWeights:
- __init__(weights_path): Load weights.yaml, validate all required sections
- get_rule_type_weight(rule_type): Map R/G/E/U/D to weights
- get_importance_multiplier(regulatory_score): Map score ranges (1-3=low, 4-6=medium, 7-12=high) to multipliers
- get_hierarchy_multiplier(level): Map level (1, 2, 3+) to multiplier
- get_piece_weight(piece): Map piece name to weight, default 1.0

ModuleConfig (constants):
- DEFAULT_CLAUDE_MODEL = "claude-opus-4"
- DEFAULT_TOP_K = 20
- SIMILARITY_THRESHOLD = 0.5
- EMBEDDING_RETRY_MAX_ATTEMPTS = 3
- EMBEDDING_RETRY_BACKOFF_FACTOR = 2.0
- AUDIT_LOG_PATH = Path("interactions.json")

PreconditionValidator:
- validate_handbook_file(): Log warning if ./data/Codified_Requirements_Text_And_Embeddings.json missing, but don't fail (HandbookIndex will auto-download)
- validate_api_keys(): Check ANTHROPIC_API_KEY and embedding key are set

AuditLogger (append-only JSON to interactions.json + terminal output + optional UI streaming):
- __init__(log_path, log_callback=None): 
  - log_path: Path to interactions.json (append-only audit trail)
  - log_callback: Optional callback(message: str) for real-time UI log streaming (Prompt 8: pass callback to stream logs via SSE)
- log_action_start(action_name, input_summary, node_name): Appends to interactions.json AND prints to terminal: `[→] {action_name} ({node_name}) starting... INPUT: {input_summary_json}` AND calls log_callback if provided
- log_action_complete(action_name, status, output_summary, node_name): Appends to interactions.json AND prints to terminal: `[✓] {action_name} ({node_name}) success OUTPUT: {output_summary_json}` or `[✗] {action_name} ({node_name}) error OUTPUT: {error_json}` AND calls log_callback if provided
  - Status symbol: ✓ for success, ✗ for error
  - Input/output summaries are rendered as compact JSON for visibility in terminal
- log_message(message_text, message_type, action_name)
- log_approval_decision(approver_name, approver_role, approved, timestamp, comments)

**3. handbook_index.py** — Handbook loading with these classes:

HandbookEntry:
- Wraps single record from Codified_Requirements_Text_And_Embeddings.json
- **REQUIRED FIELDS** (validation fails if missing):
  - `id` (int): Record ID
  - `embedding` (list[float]): 1024-dimensional Voyage embedding
  - `regulatory_content` (str): Verbatim text with embedded rule type indicator
    - Format: "RULE_ID [RGEUD] DATE ... VERBATIM_TEXT" (e.g., "PRIN 1.1.1 G 01/01/2021 RP The Principles...")
    - Contains both the rule type (R/G/E/U/D) AND the text Claude will cite
  - `level` (int): Hierarchy level (1, 2, 3+) for hierarchy_multiplier weight
  - `piece` (str): Source piece name (Main Handbook, Glossary, Instruments, Forms, TechnicalStandards, Level3Materials) for piece_weight
  - `regulatory_score` (int): 1-12 score for importance_multiplier weight
- **OPTIONAL FIELDS** (used but not required):
  - `header` (str): Section header
  - `title` (str): Entry title
  - `url` (str): Source URL
  - `fca_id`, `module`, `published_date`, etc. (metadata, not used by core logic)
- Properties: entry_id, text (constructed from header+title+regulatory_content), embedding, level, piece, regulatory_score, url
- rule_type() method: Extract first [RGEUD] from regulatory_content (used in SearchModule for rule_type_weight), default to 'U' if not found
- Constructor: Validate required fields exist, skip records with validation errors and log warning

HandbookIndex:
- __init__(handbook_path, weights=None): Load JSON with adaptive loading:
  * If handbook_path exists locally: Load and validate (complete <5 seconds)
  * If handbook_path missing: Automatically download from GitHub releases URL (https://github.com/99blakeD99/FCA_HandbookComplianceAIharnessOverview/releases/download/data260603/Codified_Requirements_Text_And_Embeddings.json) with progress indicator (2-3 seconds on 100MB/s connection, or ~12-30 seconds on cloud), then load (~5 seconds), total ~7-8 seconds first startup
  * Validate all records for required fields, convert embeddings to normalized numpy arrays
- Skip records with missing required fields (log warning)
- get_entry(entry_id), get_all_entries(), embedding_dimension(), total_entries()
- entries_by_piece(), entries_by_rule_type()

**4. tests/test_prompt2_config_and_data.py** — Comprehensive unit tests:
- TestAPIConfig: 7 tests (missing keys, valid keys, defaults, model selection)
- TestRegulatoryWeights: 6 tests (all weight types)
- TestPreconditionValidator: 4 tests (missing/valid files, API keys)
- TestHandbookEntry: 5 tests (creation, validation, rule type extraction, embedding conversion)
- TestHandbookIndex: 8 tests (loading, normalization, retrieval, counts)
- TestStage2Integration: 3 end-to-end tests (load, validate, timing)

Use pytest with fixtures. All tests use pre-computed embeddings; no live API calls.

**DELIVERABLES**

Provide all of the following:
1. exception_hierarchy.py — Exception classes
2. config.py — All configuration classes
3. handbook_index.py — Handbook loading and indexing
4. tests/test_prompt2_config_and_data.py — Unit tests

**VALIDATION CRITERIA**

After implementation:
- ✅ Handbook loads in <5 seconds (with local file) or ~7-8 seconds (with auto-download)
- ✅ HandbookIndex automatically downloads from GitHub releases if local file missing
- ✅ Embeddings are 1024-dimensional (or 10-dimensional for test fixture)
- ✅ Embeddings normalized to unit vectors (norm ≈1.0)
- ✅ All unit tests pass (pytest tests/test_prompt2_config_and_data.py)
- ✅ HandbookEntry correctly extracts rule type from regulatory_content
- ✅ RegulatoryWeights correctly loads weights from weights.yaml
- ✅ PreconditionValidator validates API keys (handbook file handled by HandbookIndex adaptive loading)
- ✅ AuditLogger writes append-only JSON to interactions.json AND prints [→] start / [✓][✗] complete entries to terminal
- ✅ No live API calls in unit tests (all use pre-computed embeddings or mocks)

**FINAL VALIDATION: Run Tests**

After implementing all modules, run the test suite:
```bash
python3 -m pytest tests/test_prompt2_config_and_data.py -v
```

All tests must pass. If any test fails, debug and fix the code, then re-run until all tests pass. Report:
- Total test count and pass count
- Any failures with error details
- Completion confirmation only when all tests pass

```

---

### Prompt 3: LLM Feature & Terminology Processing

**Prompt Claude Opus**:

```
You are implementing Prompt 3 (LLM Feature & Terminology Processing) of the Handbook Compliance Agent Harness in Python.

**CONTEXT: Overall Workflow**

Understand the overall workflow in HowToImplement.md (in your cloned repo). This clarifies how this prompt fits into the 6-prompt sequence, what prior prompts produce (your inputs), and what downstream prompts expect (your outputs).

**CONTEXT: Prompt Dependencies**

You now have available:
- All Prompt 2 config, handbook_index modules

Implement two LLM action nodes that extract and normalize user input:

Requirements (per GeneralEnquirySpec.md action specifications):

(1) ParseMarkdownAction (Node 2):
   INPUT: {entity_description: str (markdown)}
   ACTION: Use Claude to extract EntityFeatures (structured product/entity details)
   OUTPUT: {entity_features: dict per GeneralEnquirySpec.md}
   
   This action must:
   - Parse a markdown product description into structured features
   - Extract: entity_name, entity_type, features[], use_cases[], client_interaction, data_handled[], decision_authority
   - Return as dict matching GeneralEnquirySpec.md EntityFeatures schema

(2) GlossaryLookupAction (Node 3):
   INPUT: {question: str, entity_features: dict}
   ACTION: Use Claude to map user terminology to Glossary canonical terms
   OUTPUT: {canonical_terms: list[str], unmapped_terms: list[str], all_search_terms: list[str]}
   
   GLOSSARY SOURCE:
   - The Glossary is embedded in Codified_Requirements_Text_And_Embeddings.json with piece="Glossary"
   - Extract glossary entries from HandbookIndex: filter entries where piece=="Glossary"
   - Each glossary entry has: id (canonical term), text (definition + usage)
   
   This action must:
   - Extract key terms from question and entity_features
   - Map them to Glossary canonical terms (entry.id where piece=="Glossary") where possible
   - Return canonical_terms (successfully mapped to glossary), unmapped_terms (not found in glossary), and all_search_terms (canonical + unmapped combined)
   - This output is used downstream by EmbedTextAction (Node 4) to enrich question embeddings

Each action must conform to Action base class (ProgramSpec.md § Foundations § Action Base Class): accept input dict, validate inputs with clear error messages, call self.message() for user updates, return output dict. Use ModuleConfig.DEFAULT_CLAUDE_MODEL.

Produce: action classes with unit tests (mocked Claude API).

**FINAL VALIDATION: Run Tests**

After implementing all modules, run the test suite:
```bash
python3 -m pytest tests/test_prompt4_features_terminology.py -v
```

All tests must pass. If any test fails, debug and fix the code, then re-run until all tests pass. Report:
- Total test count and pass count
- Any failures with error details
- Completion confirmation only when all tests pass

```

---

### Prompt 4: Search Module

**Prompt Claude Opus**:

```
You are implementing Prompt 4 (Search Module) of the Handbook Compliance Agent Harness in Python.

**CONTEXT: Overall Workflow**

Understand the overall workflow in HowToImplement.md (in your cloned repo). This clarifies how this prompt fits into the 6-prompt sequence, what prior prompts produce (your inputs), and what downstream prompts expect (your outputs).

This module is the deterministic core that provides semantic search with regulatory weighting. In Prompt 5, it will be used as the semantic_search action node.

Prompts 2-3 have been completed.

**CONTEXT: Prompt Dependencies**

You now have available:
- config.py with RegulatoryWeights from Prompt 2
- handbook_index.py with HandbookIndex from Prompt 2

Implement semantic search with regulatory weighting. NOTE: Query embedding (question + canonical terms) is handled exclusively by EmbedTextAction in Prompt 5. SearchModule does NOT embed queries; it only performs cosine similarity search over pre-computed embeddings.

Requirements:
(1) SearchModule class initialization:
   - __init__(handbook_index: HandbookIndex, regulatory_weights: RegulatoryWeights)
   - Store references to handbook_index (for entry lookups) and regulatory_weights (for weight lookups)

(2) Deterministic core: cosine_similarity_search(question_embedding: list[float], top_k: int) → list[dict]
   - Accept pre-computed embedding vector (from EmbedTextAction)
   - For each entry in handbook_index, compute cosine similarity: dot(question_embedding, entry.embedding) / (||question|| × ||entry||)
     * Note: All entry.embedding vectors are pre-normalized to unit length (norm ≈ 1.0) from Prompt 2, so this formula correctly computes cosine similarity
   
   REGULATORY WEIGHTING (apply all 4 multipliers):
   - rule_type_weight: Extract first [RGEUD] from entry.regulatory_content using entry.rule_type(), then look up weight from RegulatoryWeights
     (R=Rules:1.0, G=Guidance:0.8, E=Evidential:0.6, U=Unclassified:0.8, D=Deleted:0.1)
   - importance_multiplier: Look up entry.regulatory_score in RegulatoryWeights ranges (1-3:0.7, 4-6:1.1, 7-12:1.3)
   - hierarchy_multiplier: Look up entry.level in RegulatoryWeights (1:1.2, 2:1.0, 3+:0.9)
   - piece_weight: Look up entry.piece in RegulatoryWeights using these exact keys: "Main Handbook":1.2, "Glossary":0.95, "Instruments":1.0, "Forms":0.85, "Technical Standards":0.9, "Level3Materials":0.85
     * Keys must match weights.yaml exactly (including spaces and capitalization)
   
   - final_score = cosine_similarity × rule_type_weight × importance_multiplier × hierarchy_multiplier × piece_weight
   - Return top_k entries sorted by final_score (descending), with ties broken by entry_id (ascending) for deterministic ordering
   - Each entry dict: entry_id, text (entry.regulatory_content — the verbatim regulatory text), base_similarity (raw cosine), weight_factors (dict with all 4 weights), final_score

(3) Error handling:
   - If question_embedding dimension doesn't match handbook embeddings, raise ValueError with clear message
   - If top_k exceeds total entries, return all entries (don't error)
   - All entry weights are guaranteed valid by Prompt 2 (HandbookEntry validation); no runtime weight errors should occur

Produce:
1. harness/search.py — SearchModule class with __init__ and cosine_similarity_search methods
2. tests/test_prompt5_search.py — Unit tests with pre-computed embeddings (no API) and integration tests (verify results are ranked by final_score and deterministically tie-broken by entry_id)

**FINAL VALIDATION: Run Tests**

After implementing all modules, run the test suite:
```bash
python3 -m pytest tests/test_prompt5_search.py -v
```

All tests must pass. If any test fails, debug and fix the code, then re-run until all tests pass. Report:
- Total test count and pass count
- Any failures with error details
- Completion confirmation only when all tests pass

```

---

### Prompt 5: Embedding & Search Integration

**Prompt Claude Opus**:

```
You are implementing Prompt 5 (Embedding & Search Integration) of the Handbook Compliance Agent Harness in Python.

**CONTEXT: Overall Workflow**

Understand the overall workflow in HowToImplement.md (in your cloned repo). This clarifies how this prompt fits into the 6-prompt sequence, what prior prompts produce (your inputs), and what downstream prompts expect (your outputs).

These are Nodes 4-5 executed in the general_enquiry workflow. Prompts 2-4 have been completed.

**CONTEXT: Prompt Dependencies**

You now have available:
- All Prompt 2 config, handbook_index modules
- Prompt 3 GlossaryLookupAction (produces canonical_terms, unmapped_terms)
- Prompt 4 SearchModule (with cosine_similarity_search method)

CRITICAL: EmbedTextAction is the SINGLE SOURCE OF TRUTH for query embedding in the entire workflow. 
All semantic search depends on embeddings produced by EmbedTextAction. Do not create alternative embedding paths.

Implement two deterministic action nodes that enrich and search:

Requirements (per GeneralEnquirySpec.md action specifications):

(1) EmbedTextAction (Node 4) — SINGLE SOURCE FOR QUERY EMBEDDINGS:
   INPUT (per GeneralEnquirySpec.md): {question: str, canonical_terms: list[str], unmapped_terms: list[str]} (from GlossaryLookupAction in Prompt 4)
   ACTION: Enrich question with canonical glossary terms, call embedding API with retry logic, return vector matching handbook embedding dimension
   OUTPUT: {embedding: list[float], enriched_text: str}
   
   ENRICHMENT PROCESS (Data Flow from Prompt 3 → Prompt 4 → Prompt 5):
   - Receive canonical_terms and unmapped_terms from GlossaryLookupAction (Node 3)
   - Construct enriched_text: question + " " + " ".join(canonical_terms + unmapped_terms)
   - Example:
     * question = "Does this product need FCA approval?"
     * canonical_terms = ["FCA", "product"]
     * unmapped_terms = ["approval"]
     * enriched_text = "Does this product need FCA approval? FCA product approval"
   - This enriched text represents the actual query being embedded (improves semantic relevance vs. raw question)
   - Call embedding API using EMBEDDING_MODEL config (voyage-3-large by default, per EmbeddingModel.md). Data file metadata (_meta.embeddings_model, _meta.embeddings_dimension) specifies the model and dimension used for handbook embeddings (1024 for Voyage 3-Large; 10 for test fixture)
   - Validate that returned embedding has dimension matching handbook_index.embedding_dimension() — raise ValueError if mismatch
   - Retry logic: exponential backoff (3 retries on timeout/rate limit; 2x backoff factor from ModuleConfig)
   - Return embedding as list[float] and the enriched_text string (for audit trail)
   
   IMPORTANT: This is the only place in the harness where queries are embedded. Embedding dimension must match the handbook data exactly; SearchModule.cosine_similarity_search() expects this consistency.

(2) SemanticSearchAction (Node 5):
   INPUT: {entity_features: dict, question_embedding: list[float]} (question_embedding from EmbedTextAction in Node 2)
   ACTION: Use SearchModule to perform cosine similarity search + regulatory weighting, return top_k ranked entries
   OUTPUT: {retrieved_entries: list[dict]} where each entry includes entry_id, text, base_similarity, weight_factors, final_score, source_version
   
   This action must:
   - Use SearchModule.cosine_similarity_search(question_embedding, top_k=ModuleConfig.DEFAULT_TOP_K)
   - Pass the embedding produced by EmbedTextAction (Node 4) directly to SearchModule without modification
   - Return top_k entries ranked by final_score (highest first)
   - Each entry dict must include: 
     * entry_id: from SearchModule result
     * text: verbatim from handbook (entry.regulatory_content)
     * base_similarity: raw cosine similarity from SearchModule
     * weight_factors: breakdown of all 4 multipliers from SearchModule
     * final_score: weighted score from SearchModule
     * source_version: add from data file metadata (_meta.source_version or default to current date ISO-8601 format)

Each action must conform to Action base class (ProgramSpec.md § Foundations § Action Base Class): accept input dict, validate inputs with clear error messages, call self.message() for user updates, return output dict.

Produce:
1. nodes/embed_search.py — EmbedTextAction and SemanticSearchAction classes
2. tests/test_prompt6_embedding_search.py — Unit tests with mocked embedding API for EmbedTextAction (no live API calls); pre-computed embeddings for SemanticSearchAction (verify results rank by final_score and dimension validation works)

**FINAL VALIDATION: Run Tests**

After implementing all modules, run the test suite:
```bash
python3 -m pytest tests/test_prompt6_embedding_search.py -v
```

All tests must pass. If any test fails, debug and fix the code, then re-run until all tests pass. Report:
- Total test count and pass count
- Any failures with error details
- Completion confirmation only when all tests pass

```

---

### Prompt 6: Reasoning & Orchestration

**Prompt Claude Opus**:

```
You are implementing Prompt 6 (Reasoning & Orchestration) of the Handbook Compliance Agent Harness in Python.

**CONTEXT: Overall Workflow**

Understand the overall workflow in HowToImplement.md (in your cloned repo). This clarifies how this prompt fits into the 6-prompt sequence, what prior prompts produce (your inputs), and what downstream prompts expect (your outputs).

This is Node 6 and the workflow orchestrator. Prompts 2-5 have been completed.

**CONTEXT: Prompt Dependencies**

You now have available:
- All Prompt 2 config, handbook_index modules
- All Prompts 2-5 action nodes (CheckPreconditionsAction, ParseMarkdownAction, GlossaryLookupAction, EmbedTextAction, SemanticSearchAction)
- SearchModule from Prompt 4

Implement two LLM/human gate action nodes and the workflow orchestrator:

Requirements (per GeneralEnquirySpec.md action specifications):

(1) ClaudeReasoningAction (Node 6):
   INPUT: {entity_features: dict (from ParseMarkdownAction), entries: list[dict] (from SemanticSearchAction, with entry_id, text, base_similarity, weight_factors, final_score, source_version)}
   ACTION: Use Claude to reason over retrieved entries and produce citations (must cite verbatim from entry text); implement citation_formatter tool for structured output; validate citations match source text exactly
   OUTPUT: ComplianceAnalysis {
     answer: str (Claude's reasoning and conclusions),
     citations: list[dict] (each: entry_id, cited_text, context, binding_level),
     reasoning_log: dict {
       reasoning_chain: str (step-by-step logic),
       gaps_identified: list[str] (edge cases/uncertainty),
       confidence_score: float (0–1)
     },
     refinement_suggestions: list[str] (2-4 actionable suggestions for refining the analysis; Claude identifies what additional information would increase confidence or resolve identified gaps),
     timestamp: str (ISO 8601),
     handbook_version: str (from data file _meta),
     version_notice: str (alert if newer version available)
   }
   
   CITATION VALIDATION (Critical Constraint):
   - Citations must be verbatim text from retrieved entries only. Claude must not paraphrase, summarize, or invent compliance guidance.
   - Each citation object must include: entry_id (e.g., "COBS 2.1.1R"), cited_text (exact verbatim string), context (sentence showing why it applies), binding_level (R/G/E/D)
   
   VERBATIM MATCHING ALGORITHM:
   - For each citation in Claude's output:
     (a) Extract entry_id and cited_text from citation object
     (b) Find the entry in retrieved_entries by entry_id (use entry_id lookup, NOT regex)
     (c) Normalize both strings: strip leading/trailing whitespace, collapse internal whitespace to single spaces
     (d) Check if normalized cited_text appears in normalized entry.text (verbatim entry text from handbook)
     (e) Punctuation and capitalization must match exactly (only whitespace normalization allowed)
     (f) If match: citation valid, include in output
     (g) If mismatch: log warning with both texts (do not halt; analyst reviews)
     (h) If entry_id not found: flag error "Citation {entry_id} not found in retrieved entries"
   
   - Example:
     * Claude cites: "COBS 2.1.1R applies to all"
     * Entry text: "COBS  2.1.1R  applies  to  all  firms"
     * Normalized both → match ✓ (extra spaces collapsed)
     * Entry text: "COBS 2.1.1R applies" 
     * Normalized both → no match ✗ (text incomplete)
   
   - Reasoning log must include: reasoning_chain (step-by-step logic), gaps_identified (edge cases/uncertainty), confidence_score (0–1)

(2) ComplianceHarness orchestrator:
   Load general_enquiry workflow YAML from GeneralEnquirySpec.md (extract from markdown, lines 9-108).
   Instantiate all 6 action nodes (Prompts 2-6):
   - CheckPreconditionsAction
   - ParseMarkdownAction, GlossaryLookupAction
   - EmbedTextAction, SemanticSearchAction
   - ClaudeReasoningAction
   
   WORKFLOW EXECUTION ORDER (6-node sequence):
   1. check_preconditions: {} → {valid, checks_performed, missing_items}
   2. extract_features: {entity_description} → EntityFeatures (flat: entity_name, entity_type, features, use_cases, client_interaction, data_handled, decision_authority)
   3. check_terminology: {question, entity_features} → {canonical_terms, unmapped_terms, all_search_terms}
   4. embed_question: {question, canonical_terms, unmapped_terms} → {embedding, enriched_text}
   5. retrieve_entries: {entity_features, question_embedding: embedding} → {retrieved_entries}
   6. analyze_compliance: {entity_features, entries: retrieved_entries} → ComplianceAnalysis {answer, citations, reasoning_log, timestamp, handbook_version, version_notice}
   
   ORCHESTRATOR CONTRACT:
   - Constructor: __init__(handbook_index, regulatory_weights, audit_logger, anthropic_client, embed_fn=None, spec_path=None)
     * embed_fn: Optional embedding function for testing
     * spec_path: Optional path to workflow YAML spec
   - Method: execute_workflow(workflow_name: str, entity_description: str, question: str) → ComplianceAnalysis
   - Input: workflow_name (e.g., "general_enquiry"), entity_description (markdown product description), question (compliance question)
   - Returns: ComplianceAnalysis with all required fields populated
   - Attributes: self.handbook_index (HandbookIndex), self.search_module (SearchModule), self.audit_logger (AuditLogger), self.client (Anthropic client)
   
   Data flow wiring: Pass output of one node to input of next node via context dict.
   **Orchestrator must perform these key wiring transformations** (actual output keys → expected input keys):
   - Node 2 output (EntityFeatures dict) → Node 3, 5, 6 input as entity_features key
   - Node 3 output (canonical_terms, unmapped_terms) → Node 4 input (pass through)
   - Node 4 output: embedding field → Node 5 input as question_embedding key (KEY TRANSFORMATION)
   - Node 5 output: retrieved_entries field → Node 6 input as entries key (KEY TRANSFORMATION)
   
   Error handling: On node failure, log error to audit trail and raise. Retry logic for transient failures:
   - Embedding API timeout → exponential backoff (3 retries, 2x factor)
   - Claude API timeout → exponential backoff (3 retries, 2x factor)
   - Validation errors (non-transient) → fail fast, log to audit trail, raise
   
   Halting conditions: If any node returns invalid=false, halt workflow and return error to user.
   
   Audit trail: 
   - At workflow start: Call audit_logger.log_action_start("execute_workflow", {entity_description, question}, "workflow_input") to log both user inputs
   - Per node: Call audit_logger.log_action_start(action_name, input_summary, node_name) before execution
   - After each node: Call audit_logger.log_action_complete(action_name, status, output_summary, node_name) after execution completes

Produce:
1. nodes/reasoning.py — ClaudeReasoningAction class
2. harness/orchestrator.py — ComplianceHarness class with:
   - __init__(handbook_index, regulatory_weights, audit_logger, anthropic_client)
   - execute_workflow(workflow_name: str, entity_description: str, question: str) → ComplianceAnalysis
   - Internal: method to extract and parse workflow YAML from GeneralEnquirySpec.md
   - Internal: method to wire node outputs to next node inputs (handling key transformations: embedding→question_embedding, retrieved_entries→entries)
3. tests/test_prompt7_reasoning_orchestration.py — Unit tests for action + integration tests:
   - ClaudeReasoningAction: mocked Claude API, citation validation, reasoning_log structure
   - End-to-end: sample product + question → runs all 6 nodes → validates ComplianceAnalysis schema and field presence

**FINAL VALIDATION: Run Tests**

After implementing all modules, run the test suite:
```bash
python3 -m pytest tests/test_prompt7_reasoning_orchestration.py -v
```

All tests must pass. If any test fails, debug and fix the code, then re-run until all tests pass. Report:
- Total test count and pass count
- Any failures with error details
- Completion confirmation only when all tests pass

```



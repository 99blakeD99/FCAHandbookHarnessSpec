# FCA Handbook Compliance Agent Harness

## Overview

**Harness Specification.** This public opensource repo sets out specifications for a FCA Handbook Compliance Agent Harness. It is unofficial and not endorsed by the FCA.

**Fundamental Need**. The Harness is necessary: 

- To ensure structural compartmentalisation of Compliance, both top-down within overall workflows, and bottom-up in its internal workings. This ensures that emerging AI agent security standards are engineered-in rather than glued-on, especially OWASP Top 10 for Agentic Applications 2026.

- To prevent LLMs' inbuilt token incentives from causing them to go into generative AI mode. This would otherwise result in predictive text -like methods based on the LLM's training data, coming up with a convincing but error-prone impersonation of a Compliance Officer.

- To enable scalable compliance governance for autonomous FS workflows. For example, product design processes that iterate through millions of scenarios with compliance checks carried out in every iteration.

**Harness Method.** The job of the Harness is to enforce a workflow flowing through defined nodes. Even if every node uses an LLM (in fact, some are deterministic), the Harness still keeps the LLM focused on a narrow task. This is a crucially important control. 

**From Specifications to Code.** The specifications are written using conventions which are human readable, but structured such as to enable giving to a suitable LLM with the instruction to create the requisite code, see [HowToImplement.md](HowToImplement.md). These methods avoid "black-box" opacity.

**Flexible AI Deployment.** The Harness is designed to be deployed in accordance with evolving AI workflow patterns. In order, from lightest to heaviest integration, it can be used as a named tool, skill, MCP server, agent component, or plugin. This enables adoption across diverse emerging AI architectures.

**Accommodates Open Questions.** The Harness accommodates a very common user requirement that traditional approaches struggle with. Users can ask open questions, for example: "Attached is a file setting out a new product. Which entries in the FCA Handbook are relevant?". (The file must be in markdown format.)

**Firm Foundation.** The foundation is an unadorned JSON file containing the codified requirements (to which as a prerequisite first step appropriate embeddings need to be added). Essentially this repo sets out a map of what to do with it.

**Adaptable for Other Codified Requirements.** The Harness architecture is put forward as a template for Compliance AI, enabling modification to suit other codified requirements. 

**Live Testing Needed.** These specifications have not been live-tested in a production environment. Implementation should be validated against real-world compliance workflows and regulatory scrutiny before deployment in regulated use. It is envisaged that Compliance teams will be asked to beta-test implemented versions.

## Getting Started

To understand the Compliance Agent Harness architecture, read the documents in order:

1. **[README.md](README.md)** (this document). 

2. **[EmbeddingModel.md](EmbeddingModel.md)**. 

3. **[StructuredSearch.md](StructuredSearch.md)**. 

4. **[WorkflowSpecPrinciples.md](WorkflowSpecPrinciples.md)**. 

5. **[GeneralEnquirySpec.md](GeneralEnquirySpec.md)**.  

6. **[ProgramSpec.md](ProgramSpec.md)**. 

7. **[HowToImplement.md](HowToImplement.md)** — Implementation prompts for building the Harness.

8. **[Deployment.md](Deployment.md)** — Integration patterns and deployment options.

## Design

The guiding design principles are:

1. **Auditability** — Every decision must be traceable: question → retrieval → reasoning → citation

2. **Bounded & Auditable Workflows** — LLM scope is limited to defined tasks; every decision is logged and traceable.

3. **Graceful Failure** — No silent defaults to web search. Explicit errors when data unavailable.

4. **Agnosticism at Core** — The core architecture can accommodate any suitable LLM, embedding model or programming language.


## Codified_Requirements_Text_And_Embeddings.json

The foundation is a JSON file containing the Codified Requirements, in this case for the FCA Handbook. Currently this needs to be laboriously scraped, in due course it is hoped that the FCA will provide it directly.

Embeddings need to be added, in this case voyage-3-large.

An example of the resulting file is [FCA_Handbook_Template_PRIN.json](FCA_Handbook_Template_PRIN.json), the PRIN section of the FCA Handbook.

## Harness as LLM Management

### Illustrative Workflow: General Enquiry

The Harness defines and ring-fences LLM scope as follows:

| Step | Task | Responsibility | Purpose |
|------|------|-----------------|---------|
| 1 | Check preconditions | Harness | Validate setup complete |
| 2 | Extract entity features | **LLM** | Parse product description into structured features |
| 3 | Map terminology | **LLM** | Map user language to terms used in FCA Glossary; produce canonical_terms + unmapped_terms |
| 4 | Embed question | Harness | Create enriched embedding using mapped terminology |
| 5 | Retrieve & rank entries | Harness | Semantic search using enriched embedding + regulatory weighting |
| 6 | Reason over entries | **LLM** | Determine which entries apply; explain reasoning; cite relevant Handbook entries verbatim |

**Result**: The Harness produces a structured compliance analysis (answer, citations, reasoning, confidence) and presents it to the user. What the user does with the analysis—how they validate it, integrate it into their workflows, or use it for decisions—is their responsibility.

### Design Notes

- The ringfence is enforced at the retrieval layer: semantic search returns a bounded set of Handbook entries using deterministic methods (embeddings via cosine similarity, regulatory weighting, numpy). This bounded set is what the LLM sees. Every step is logged and auditable for transparency.

- The LLM is used across multiple nodes (feature extraction, terminology mapping, reasoning) but is constrained at the critical point: it only reasons about retrieved Handbook entries and must cite verbatim text. This prevents it from "going generative" on compliance questions—inventing answers outside the Handbook or based on training data.

- The ringfence architecture ensures: Questions (assumed FCA Handbook-related by deployment layer) → extract features (LLM) → map terms (LLM) → retrieve bounded entries (Harness) → reason over those entries only (LLM) → cite verbatim (enforced). 

## Security

### Security Properties Within the Harness

- Compartmentalisation. The Harness embodies the important emerging principle of Least Agency: the Harness is structurally compartmented within the user's overall workflow, the LLM reasons only, the workflow is fixed and auditable, every decision is bounded and validated; this foundationally mitigates the top agentic AI risks (OWASP ASI10: Goal Hijacking, Tool Misuse, Cascading Failures, and Human-Agent Trust Exploitation). 

- Architecture. The python specifications incorporate secure-by-default patterns. 

- Construction. The Harness is made available in this opensource repo as a fully inspectable *specification*. This utilises the ease with which Claude Opus and other Coding AI tools can turn specification into code. The result is that FS firms can avoid reliance on a specific third-party implementation and the resultant exposure to security risks.

### External Security Considerations

The Harness does *not* protect against external security threats such as LLM Jailbreaks, LLM Accidents, LLM Backdoors attacks and other emerging LLM threats. 

A different layer entirely is required elsewhere in the stack, including sandboxing, data isolation, prompt injection defenses, model supply chain verification, runtime monitoring, secure deployment, credential management, and API integration handling. These are orthogonal to the Harness.

The Harness provides research results based on AI methods and no guarantee of correctness or applicability is given or implied. The user is responsible for validating results and determining how (or whether) results should be used in their compliance workflows.

In short the Harness is necessary within its domain but not sufficient to protect against threats outside its domain.

## About this GitHub Repo

If you have questions, talking points, ideas, experiences, feedback: please open a GitHub Issue. 

## Disclaimer and License

**Disclaimer**: This Harness is provided as a specification without warranty. Implementers are responsible for validating the implementation against their compliance requirements, regulatory obligations, and security standards. The Harness is a research tool, users are responsible for how results are used.

This Harness is licensed under the MIT License. See [opensource.org/licenses/MIT](https://opensource.org/licenses/MIT) for details.

## Support & Implementation Services

**Blake Dempster**  
Actuary. Management of special projects for FS firms.  
Experienced in IT, Process Engineering, Compliance, Risk Management, AI/LLMs. Architect of this repo.

**Contact:** [fca-handbook-harness-repo@jbmd.co.uk](mailto:fca-handbook-harness-repo@jbmd.co.uk)  
**Website:** [JBMD.co.uk](https://jbmd.co.uk)

**Available for:**
- Troubleshooting & technical guidance
- Consulting on compliance automation
- Workflow integration & implementation support
- AI/LLM architecture decisions

For public questions, feedback, and feature requests: open a [GitHub Issue](https://github.com/99blakeD99/FCA_HandbookComplianceAIharnessOverview/issues).

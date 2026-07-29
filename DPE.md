Digital Policy Engine for Agentic Corporate Onboarding

The right design is not to let an onboarding agent read a 100-page policy document and independently interpret it during every case.

Instead, create a governed pipeline that converts policy into a versioned, testable, evidence-linked policy-as-code repository. Runtime policy agents then execute only approved rules.

A useful distinction is:

Policy authoring agents extract, normalise and propose rules.
Policy governance agents validate, test and obtain human approval.
Runtime policy agents evaluate approved rules deterministically.
The LLM may explain decisions, but it must not make the final policy decision.

It is not technically realistic to guarantee “no hallucinations” from an LLM. The enterprise objective should be:

No unsupported policy decision can reach the onboarding workflow.

This is achieved through structured output, source citations, deterministic validation, rule approval, fail-closed execution and human escalation.

1. Demonstration policy

For the example, I will use FATF’s Guidance on Beneficial Ownership of Legal Persons.

This document is suitable for demonstrating ownership and control rules, but it is international guidance rather than a directly executable local law. A bank would need to overlay jurisdiction-specific regulations and its internal risk appetite before deploying the rules.

The document establishes, among other principles, that:

A beneficial owner must ultimately be a natural person.
Ownership or control may be direct or indirect.
Where no beneficial owner can be identified, the senior managing official may need to be recorded for CDD purposes.
Beneficial-owner identity and the basis for beneficial-owner status should be verified.
Financial institutions should understand the customer’s ownership and control structure.
Beneficial ownership information should be adequate, accurate and up to date.

The document also says that financial institutions should not rely exclusively on beneficial-ownership information from a single external mechanism when performing CDD.

2. Target architecture
                      DIGITAL POLICY CONTROL PLANE
┌───────────────────────────────────────────────────────────────────────┐
│                                                                       │
│  Policy Sources                                                       │
│  Internal policies | Regulations | Standards | Procedures             │
│          │                                                            │
│          ▼                                                            │
│  1. Document Intake Agent                                             │
│     - malware scanning                                                │
│     - OCR/layout extraction                                           │
│     - document fingerprint                                            │
│     - effective-date detection                                        │
│          │                                                            │
│          ▼                                                            │
│  2. Policy Segmentation Agent                                         │
│     - sections, clauses, tables, definitions                          │
│     - preserves page and paragraph references                         │
│          │                                                            │
│          ▼                                                            │
│  3. Obligation Extraction Agent                                       │
│     - actor, obligation, condition, exception, evidence, outcome      │
│     - structured output only                                          │
│          │                                                            │
│          ▼                                                            │
│  4. Rule Composition Agent                                            │
│     - converts obligations to decision-table candidates               │
│     - no executable code generation without validation                │
│          │                                                            │
│          ▼                                                            │
│  5. Verification Agents                                               │
│     ├─ Citation verifier                                              │
│     ├─ Semantic entailment verifier                                   │
│     ├─ Contradiction/overlap detector                                  │
│     ├─ Completeness checker                                           │
│     └─ Test-case generator                                             │
│          │                                                            │
│          ▼                                                            │
│  6. Policy SME Approval                                               │
│          │                                                            │
│          ▼                                                            │
│  Approved Policy Registry                                             │
│  - versioned rules                                                    │
│  - lineage                                                            │
│  - tests                                                              │
│  - approvals                                                          │
│  - effective dates                                                    │
│  - jurisdiction/product/segment applicability                         │
└───────────────────────────────────────────────────────────────────────┘

                      RUNTIME DECISION PLANE
┌───────────────────────────────────────────────────────────────────────┐
│ Onboarding Agent                                                      │
│      │                                                                │
│      ▼                                                                │
│ Policy Resolver                                                       │
│ - jurisdiction                                                        │
│ - booking location                                                    │
│ - product                                                             │
│ - client segment                                                      │
│ - legal form                                                          │
│ - effective date                                                      │
│      │                                                                │
│      ▼                                                                │
│ Deterministic Rule Executor                                           │
│      │                                                                │
│      ├── PASS / requirement satisfied                                 │
│      ├── REQUEST_INFORMATION                                          │
│      ├── PERFORM_VERIFICATION                                         │
│      ├── BLOCK                                                        │
│      └── REFER_TO_HUMAN                                               │
│                                                                       │
│ Explanation Agent                                                     │
│ - explains the deterministic result                                   │
│ - includes exact rule and source citations                            │
└───────────────────────────────────────────────────────────────────────┘

LangGraph is appropriate because it allows deterministic nodes and LLM-driven nodes to be combined in a controlled stateful graph, including persistence and human-in-the-loop checkpoints.

LangChain structured output should be used so agents return validated Pydantic objects instead of free-form prose.

3. Canonical policy-rule model

A policy rule should not simply contain condition and action. It needs sufficient metadata to support regulatory lineage.

{
  "rule_id": "FATF-R24-BO-001",
  "name": "Beneficial owner must be a natural person",
  "rule_type": "VALIDATION",
  "status": "APPROVED",
  "policy_source": {
    "document_id": "fatf-bo-legal-persons",
    "document_version": "2023",
    "section": "4",
    "paragraphs": ["31", "32"],
    "page": 15,
    "source_quote": "Only a natural person can be an ultimate beneficial owner.",
    "source_hash": "sha256:..."
  },
  "applicability": {
    "processes": ["CORPORATE_ONBOARDING", "KYC_REVIEW"],
    "entity_types": ["LEGAL_PERSON"],
    "jurisdictions": ["GLOBAL_BASELINE"],
    "products": ["ALL"]
  },
  "inputs": [
    "client.entity_type",
    "related_parties[].party_type",
    "related_parties[].relationship_type"
  ],
  "expression": {
    "all": [
      {
        "fact": "client.entity_type",
        "operator": "eq",
        "value": "LEGAL_PERSON"
      },
      {
        "fact": "beneficial_owners",
        "operator": "all_match",
        "predicate": {
          "field": "party_type",
          "operator": "eq",
          "value": "NATURAL_PERSON"
        }
      }
    ]
  },
  "success_action": "CONTINUE",
  "failure_action": "REQUEST_CORRECTION",
  "failure_reason": "A legal entity cannot be recorded as an ultimate beneficial owner.",
  "confidence": 1.0,
  "approved_by": "Policy-SME-123",
  "effective_from": "2026-01-01",
  "effective_to": null
}
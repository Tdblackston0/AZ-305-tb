# AZ-305 Copilot Instructions

This repository is for Azure Solutions Architect Expert (AZ-305) labs and study content.

## Primary Goal
Create lab content that is exam-focused, practical, and aligned to assessed domains:

1. Design infrastructure solutions (30-35%)
2. Design identity, governance, and monitoring solutions (25-30%)
3. Design data storage solutions (20-25%)
4. Design business continuity solutions (15-20%)

## Global Authoring Rules
- Prioritize hands-on labs over theory-only responses unless theory is explicitly requested.
- Keep every lab mapped to one primary AZ-305 domain and list any secondary domains.
- Include both Bicep and Terraform options for infrastructure work unless the user asks for only one.
- Use Azure Well-Architected and least-privilege guidance by default.
- Prefer production-realistic scenarios and tradeoff discussions (cost, security, performance, operations, reliability).

## Required Lab Structure
For any new or updated lab, include these sections in this order:

1. Lab objective
2. Exam domain mapping (with percentage area)
3. Prerequisites
4. Architecture and design rationale
5. Implementation steps
6. IaC implementation
7. Validation and success criteria
8. Cleanup
9. Exam-style review questions

## IaC Expectations
- Provide deployable snippets or templates, not pseudo-code.
- When relevant, include parameterization for region, naming, SKU/tier, and environment.
- Include a short validation checklist after deployment.

## Output Style
- Be concise but complete.
- Use tables when comparing architecture options.
- Call out common AZ-305 pitfalls and decision traps.
- End each lab with 3-5 scenario-based exam questions.

## Domain-Specific Guidance

### Infrastructure (30-35%)
- Cover compute selection (VM, VMSS, AKS, App Service) and network topology choices.
- Explain scaling, availability zones, and regional failover decisions.

### Identity, Governance, Monitoring (25-30%)
- Cover Entra ID patterns, RBAC, policy, management groups, and monitoring baselines.
- Include alerting and diagnostics settings where relevant.

### Data Storage (20-25%)
- Cover storage service selection, consistency/latency tradeoffs, backup/retention, and security.
- Include lifecycle and cost optimization guidance.

### Business Continuity (15-20%)
- Define RTO/RPO assumptions explicitly.
- Include backup, restore, failover, and resilience testing steps.

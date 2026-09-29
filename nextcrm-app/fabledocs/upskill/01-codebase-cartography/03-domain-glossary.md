# Domain Glossary

CRM vocabulary as this codebase uses it. Model names in [prisma/schema.prisma](../../../prisma/schema.prisma) (line refs from the model-name index, verified 2026-07-11).

| Term | Meaning here | Code |
| --- | --- | --- |
| **Account** | A company/organization you sell to. The hub entity: contacts, opportunities, contracts, invoices, documents, tasks all hang off it. | `crm_Accounts` (schema L12), [actions/crm/accounts/](../../../actions/crm/accounts/) |
| **Contact** | A person, usually linked to an account (`assigned_accounts`). | `crm_Contacts` (schema L446) |
| **Lead** | An unqualified prospect; converts into account/contact/opportunity. | `crm_Leads` (schema L72) |
| **Opportunity** | A potential deal with a sales stage, expected close, budget; has line items. | `crm_Opportunities` (schema L196) |
| **Contract** | A signed agreement tied to an account; has line items and status. | `crm_Contracts` (schema L506) |
| **Target** | A *cold outreach* prospect (campaigns module) — NOT the same as Lead. Targets live in target lists and get enriched/emailed. | `crm_Targets` (schema L1212), `crm_TargetLists` (L1269) |
| **Campaign** | A multi-step cold-email sequence sent to target lists via Resend. Steps → sends → webhook status updates. | `crm_campaigns` (L287), `crm_campaign_sends` (L366) |
| **Enrichment** | AI agents (OpenAI + Firecrawl, optionally e2b sandbox) filling empty fields on contacts/targets from the web. Tracked in enrichment rows with status. | `crm_Contact_Enrichment` (L115), [lib/enrichment/](../../../lib/enrichment/), [inngest/functions/enrich-contact.ts](../../../inngest/functions/enrich-contact.ts) |
| **Invoice** | Legal billing document. Types: INVOICE, CREDIT_NOTE, PROFORMA, RECEIPT. Lifecycle DRAFT→ISSUED→(PARTIALLY_)PAID/CANCELLED. Immutable once issued. | `Invoices` (L1720), [lib/invoices/permissions.ts](../../../lib/invoices/permissions.ts#L3-L12) |
| **Series** | An invoice numbering sequence with a template like `INV-{YYYY}-{####}` and a yearly reset policy. | `Invoice_Series` (L1868), [numbering.ts](../../../lib/invoices/numbering.ts#L3-L9) |
| **Billing snapshot** | Frozen copy of the account's billing address written onto the invoice at issue time, so later account edits don't rewrite history. | [issue-invoice.ts](../../../actions/invoices/issue-invoice.ts#L60-L71) |
| **Board / Task** | Project-management module (kanban boards, tasks, comments, watchers). | `Boards` (L654), `Tasks` (L807) |
| **Watcher** | A user subscribed to an entity's changes; watching also grants read scope on accounts. | `AccountWatchers` (L1171), [crm.ts](../../../lib/authz/scopes/crm.ts#L218-L224) |
| **Document** | Uploaded file (MinIO) linkable to many entity types via junction tables (`DocumentsToAccounts` etc.), with visibility rules. | `Documents` (L701) |
| **Scope / scope-where** | A Prisma `where` fragment encoding "rows this user may see/write". The repo's authorization idiom. | [lib/authz/scopes/crm.ts](../../../lib/authz/scopes/crm.ts#L229-L237) |
| **Soft delete** | `deletedAt`/`deletedBy` columns instead of row deletion; read scopes filter `deletedAt: null`. | [docs/soft-delete-gaps.md](../../../docs/soft-delete-gaps.md) |
| **Audit log** | Field-level change diffs per entity, written best-effort (never blocks the mutation). | `crm_AuditLog` (L989), [lib/audit-log.ts](../../../lib/audit-log.ts#L66-L82) |
| **Activity** | A logged interaction (note/call/email/meeting/task) attachable to multiple CRM entities via `crm_ActivityLinks`. Distinct from Invoice_Activity (invoice lifecycle events) and from audit log (field diffs). | `crm_Activities` (L613) |
| **Embedding** | pgvector representation of an entity for semantic search; maintained by `embed-*` Inngest functions. | `crm_Embeddings_*` (L1305-L1365), [actions/fulltext/unified-search.ts](../../../actions/fulltext/unified-search.ts) |
| **MCP tool** | A CRM operation exposed to AI agents over the Model Context Protocol, authenticated by `nxtc__` API tokens. | [lib/mcp/tools/](../../../lib/mcp/tools/), [api/mcp/[transport]/route.ts](../../../app/api/mcp/%5Btransport%5D/route.ts) |
| **API key vs API token** | Easy to confuse: **ApiKeys** = encrypted third-party provider keys (OpenAI/Firecrawl) with SYSTEM/USER scope ([lib/api-keys.ts](../../../lib/api-keys.ts)); **ApiToken** = hashed bearer credentials for *this* app's MCP API ([lib/api-tokens.ts](../../../lib/api-tokens.ts)). |

## Confusing near-synonyms

- **Lead vs Target**: Lead = inbound/qualified-ish CRM record with sources/statuses. Target = outbound cold-email prospect. Different models, different scopes (targets are creator-scoped only — [crm.ts](../../../lib/authz/scopes/crm.ts#L373-L376)).
- **created_by vs createdBy vs created_by_user**: the same concept spelled three ways across models (Mongo heritage). Document scope ORs both `created_by_user` and legacy `createdBy` ([crm.ts](../../../lib/authz/scopes/crm.ts#L402-L412)). Always check the model before writing a scope.
- **Invoice_Activity vs crm_Activities vs crm_AuditLog**: lifecycle events vs user-logged interactions vs automatic field diffs.
- **getSession vs requireAuthenticated vs getUser**: three auth entry points of increasing strictness; older actions use `getSession` directly ([update-account.ts](../../../actions/crm/accounts/update-account.ts#L34)), newer code uses `requireAuthenticated` which re-verifies the user row and returns a typed role.

Drill: pick three glossary rows and find one *additional* code location for each using `rg`. Basic = found them. Solid = explained the relationship between the two locations. Strong = spotted a naming inconsistency along the way and could say what a migration to fix it would cost.

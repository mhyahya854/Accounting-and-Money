# Accounting and Money — Master Plan

Status: authoritative planning document for the first implementation phase
Scope of this document: architecture, product direction, safety requirements, evaluation criteria, and implementation sequencing
Implementation status: planning only; KMyMoney is the intended accounting-engine and financial-domain foundation, subject to implementation/prototype validation; this document does not authorize application scaffolding

## Document purpose and decision vocabulary

This document defines the intended direction for Accounting and Money, a Hermes-compatible, file-native personal accounting and finance application. It is the project’s first master plan and is intended to give future coding agents, reviewers, and human contributors a shared source of context.

The project uses the following decision vocabulary:

- **Hard requirement** means a behavior or constraint the eventual product must satisfy unless this plan is deliberately amended.
- **Proposed design** means a strong starting design to investigate and validate during implementation.
- **Future investigation** means work that must be performed before a consequential choice is made.
- **Deferred decision** means an unresolved choice that must not be silently treated as settled.
- **Canonical data** means durable user-owned information from which derived state can be rebuilt.
- **Derived data** means an index, cache, report, search structure, or other generated state that may be recreated from canonical data.

The plan is intentionally detailed about invariants and safety while remaining open about implementation choices that require repository-level investigation, prototype measurements, or accounting review.

## 1. Executive vision

Accounting and Money should make a person’s financial life understandable through ordinary folders and human-readable Markdown files. The graphical application should be a complete, locally usable interface over that information, not the only place where the information exists. KMyMoney is the intended accounting-engine and financial-domain foundation, subject to implementation/prototype validation. Hermes should be able to inspect the same local representation, answer questions, and request safe operations through a constrained command interface, but remains optional.

The central model is:

```text
                            USER
                 ┌──────────┴──────────┐
                 │                     │
              APP UI          OPTIONAL HERMES CHAT
                 └──────────┬──────────┘
                            │
             HERMES ACCOUNTING DOMAIN / SYNC CORE
                            │ typed validated commands
                            ▼
                  KMYMONEY ACCOUNTING ADAPTER
                            │
                            ▼
              CANONICAL MARKDOWN FILESYSTEM
                            │
                            ▼
              DERIVED WORKBOOKS / CANVAS / INDEXES
```

The system must preserve accounting correctness while keeping the durable representation legible to people and small language models. A model should interpret a user’s intent and select a constrained operation. The KMyMoney-backed deterministic accounting boundary and Hermes-owned integration code must validate the operation, perform or validate accounting-critical calculations, write the affected records safely, and report a verified result. The UI and ordinary accounting workflows must remain fully usable with Hermes, local models, and network access unavailable.

The product is not merely a traditional accounting application with an AI assistant attached. Its differentiating idea is a transparent local financial workspace in which the filesystem, the app, and Hermes are coordinated views over the same user-owned records.

## 2. Design principles

### 2.1 User-owned files are durable

Important financial history must not be trapped solely in an opaque generated database. Canonical records must be stored in an understandable folder tree with stable identifiers, explicit relationships, provenance, and readable descriptions.

### 2.2 Correctness takes precedence over convenience

If a value, match, relationship, or accounting interpretation is uncertain, the system must preserve the uncertainty and request review. It must not silently guess in a way that changes balances, income, expenses, liabilities, currencies, or history.

### 2.3 The KMyMoney-backed deterministic engine owns accounting semantics

The model may interpret intent, summarize evidence, and propose actions. KMyMoney is the intended accounting-engine and financial-domain foundation, subject to prototype validation. Its deterministic accounting machinery, behind a Hermes-owned domain and adapter boundary, must own or validate balancing, currency handling, transaction state transitions, reconciliation, and other financial consequences. Deterministic integration code must enforce typed commands, field ownership, identity mapping, duplicate handling, safe persistence, crash recovery, and verification. This boundary must not grow into a second independently maintained accounting engine.

### 2.4 The filesystem is an interface, not an accident

The Markdown format, folder topology, IDs, relationship syntax, and mutation protocol should be documented interfaces. App code and Hermes should use the documented interface rather than relying on undocumented database tables or fragile path conventions.

### 2.5 One canonical record, many views

Avoid copying the same transaction or binary source into multiple locations merely to make it appear in several views. Store one canonical record, then use references, links, and rebuildable indexes for account, category, date, statement, report, and evidence views.

### 2.6 Provenance is part of the record

Imported values should retain where they came from, when they were observed, which importer produced them, and whether a human edited them. Evidence files must remain linkable without being silently replaced or discarded.

### 2.7 Filesystem operations must be recoverable

Multi-record changes must be staged, validated, committed atomically where possible, journaled, and recoverable after an interruption. A crash must not leave a transfer represented on only one side without an explicit recovery state.

### 2.8 Cross-platform behavior is a product requirement

The shared core should support Windows, Linux, and macOS. Path handling, file watching, atomic replacement, permissions, line endings, timestamps, Unicode names, and locking must be designed and tested on all target platforms.

### 2.9 Small models need explicit affordances

The file format and command layer must be compact, regular, and discoverable. A model of approximately one billion parameters should be able to read the relevant records, identify stable IDs, construct a constrained request, and understand validation results without learning a large opaque schema.

### 2.10 Privacy is the default

The application should operate locally without a cloud requirement. It must not store PINs, CVVs, banking passwords, authentication secrets, or unnecessary full payment credentials in Markdown, logs, fixtures, or reports.

## 3. User goals

The eventual product should help a user:

1. Understand balances, spending, income, debts, credit cards, goals, subscriptions, and obligations across many accounts.
2. Preserve financial evidence in a local, searchable, organized workspace.
3. Import statements, exports, receipts, screenshots, invoices, SMS, and unknown files without losing the originals.
4. Correctly represent domestic and international transactions, multiple currencies, fees, refunds, pending activity, transfers, and debt payments.
5. Reconcile records against statements without silently inventing adjustments.
6. Move between the graphical app, Hermes, and ordinary filesystem editing while keeping supported changes synchronized.
7. Understand what was imported, what was manually changed, what needs review, and why a calculated result is trustworthy.
8. Rebuild derived application state if an index or cache is deleted.
9. Use the same project on Windows, Linux, and macOS with shared accounting logic.
10. Ask natural-language financial questions and receive answers grounded in local records and evidence.

## 4. Non-goals for the initial project direction

The following are explicitly out of scope for this planning run and remain non-goals until separately authorized:

- Implementing the application.
- Creating application source scaffolding, package files, databases, CI, or generated assets.
- Implementing, forking, or embedding KMyMoney during this documentation-only run. KMyMoney is the intended accounting-engine foundation, but production adoption and the integration mechanism remain subject to the prototype gates in this plan.
- Treating a particular database engine, UI toolkit, language, or desktop packaging technology as already chosen.
- Connecting directly to banks or storing online banking credentials.
- Creating a cloud-hosted financial service as the primary architecture.
- Replacing accounting review with model confidence.
- Claiming tax, legal, investment, lending, or regulatory advice.
- Storing real private financial data in the repository.
- Assuming every raw document can be automatically interpreted.

The project may later support optional bank integrations, encryption, mobile access, or tax-oriented reports, but those require separate threat, product, and compliance decisions. The app itself must support ordinary accounting without Hermes or an AI service; AI is an optional interface, not a runtime prerequisite.

## 5. Architectural overview

The proposed architecture has six conceptual layers:

1. **Canonical Accounting Folder** — user-owned Markdown records and preserved source files; durable canonical financial representation.
2. **Hermes Domain and Sync Core** — parser, schema validator, relationship resolver, field ownership, typed commands, KMyMoney adapter, conflict detector, journal, recovery, and verification.
3. **KMyMoney Accounting Foundation** — intended deterministic accounting and financial-domain machinery, accessed through the adapter and subject to prototype validation.
4. **Derived State** — rebuildable indexes, search structures, account workbooks, canvas graph/layouts, aggregates, reports, caches, and application state.
5. **Application UI** — locally complete account, transaction, evidence, reconciliation, budget, goal, subscription, debt, import, backup, restore, diagnostics, workbook, report, and canvas workflows.
6. **Optional Hermes Interface** — read commands, search operations, constrained mutation commands, validation responses, and result verification. The app and core must not depend on Hermes, a local model, or internet access.

Conceptually:

```text
Human files / raw evidence
            ⇅
Hermes-owned domain, canonical parser, and schema validator
            ⇅
KMyMoney adapter and deterministic accounting foundation
       ⇅                 ⇅
Canonical Markdown     App UI / optional Hermes API
       ⇩                 ⇩
Derived workbooks, canvas, indexes, reports, verified responses
```

The exact process boundaries are deferred. The Hermes domain and adapter may initially be a library used by the UI and optional Hermes interface, or they may use a local service with a stable protocol. The choice must preserve the same invariants and filesystem interface. KMyMoney is accessed behind the Hermes-owned domain boundary; its UI and internal storage schema are not the canonical filesystem contract.

### 5.1 Accounting engine authority, canonical state, and field ownership

The following is a hard project requirement:

> **All accounting calculations, financial state transitions, and financially meaningful mutations MUST be performed or validated by deterministic application code. Markdown files store the resulting canonical financial state but are not a substitute for the accounting engine. Human or Hermes edits that affect accounting semantics MUST pass through the engine, which recalculates dependent values before the change becomes committed state.**

The governing separation is:

> **Hermes interprets intent. The accounting engine performs accounting. The filesystem stores the verified result.**

KMyMoney is the intended accounting-engine and financial-domain foundation for Hermes Accounting, subject to implementation/prototype validation. It is intended for its mature personal-finance orientation and candidate support for double-entry semantics, split transactions, account models, loans, schedules, budgets, investments, currencies, reconciliation, exact/rational-style monetary handling, transaction boundaries, storage and importer extension points, mature tests, and cross-platform foundations. These are reasons to validate the fit, not permission to assume feature parity, API stability, precision behavior, headless operation, or integration safety without prototype evidence.

The accounting authority is the validated KMyMoney-backed deterministic engine boundary, not an independently reimplemented second ledger. Hermes-owned code provides a stable domain and command boundary, maps Hermes identities and canonical records to/from engine objects, validates that the result can be represented without loss, and owns synchronization, journaling, privacy, recovery, and verified persistence. It must perform or validate, using deterministic and tested financial rules, all accounting-critical work, including:

- account balance calculations, transaction posting, double-entry or equivalent accounting invariants, transfers, transfer neutrality, and all financially meaningful postings;
- credit-card liabilities, credit-card payments, statement balances, current balances, available credit, interest, fees, refunds, disputes, chargebacks, and provisional credits;
- loan balances, amortization, principal allocation, interest allocation, loan fees, early repayment, refinancing, restructuring, and dependent schedule effects;
- savings balances, goal allocations, sinking-fund calculations, and the distinction between actual balances and conceptual allocations;
- subscription transaction relationships, recurring payments, renewal state, and related financial consequences;
- pending-to-posted and authorization-to-settlement transitions, declines, failures, cancellations, refunds, partial refunds, reversals, chargebacks, disputes, and provisional-credit transitions;
- fees, taxes, tips, surcharges, foreign-exchange conversions, international spending, multi-currency settlement, authorization-versus-posting FX differences, and account-to-account currency conversion;
- reconciliation, statement matching, duplicate detection, import interpretation, and correction of dependent financial values;
- every other financially meaningful state transition, invariant, mutation, or recalculation defined by the accounting model.

The exact accounting model, including the future boundary for double-entry postings and conceptual allocations, remains subject to the investigation and review already required by this plan. The authority boundary does not remain optional: Hermes, OCR, importers, and text editing must not perform authoritative accounting mathematics or bypass deterministic validation.

Markdown and the Accounting folder are the durable, transparent representation of the resulting committed financial state. They are not the accounting engine, and they must not depend on KMyMoney's internal file or storage schema. After the engine validates and applies an operation, the integration writes or updates the appropriate canonical Markdown records, updates rebuildable derived state, re-reads the result, and verifies it before reporting success. A KMyMoney working store, database, or other machine state may be needed internally for engine operation, performance, or crash recovery, but it must not quietly become the only authoritative copy of the user’s financial records. Its consistency and recovery relationship to canonical Markdown must be demonstrated before production use.

The normal operation boundary is:

```text
Human / Hermes / App / Import
             ↓
Typed financial intent or candidate evidence
             ↓
      Accounting Engine
             ↓
Validate → Calculate → Apply financial rules → Resolve dependencies
             ↓
        Commit result
             ↓
     Write canonical Markdown/files
             ↓
      Update derived indexes/cache
             ↓
          Re-read and verify
```

#### Field ownership

Field ownership is part of the deterministic authority model. Not every Markdown field has the same mutation rules.

Human- or Hermes-editable descriptive fields may include notes, tags, merchant display names, categories where reclassification is permitted, custom descriptions, valid evidence links, aliases, and user annotations. These fields still require parsing, schema validation, relationship validation, provenance, and conflict handling. A descriptive edit that changes accounting meaning is no longer a safe descriptive edit and must be converted into an accounting operation.

Engine-controlled financial fields include account balances, calculated balance-after values, remaining loan principal, interest and principal allocation, fees generated by accounting rules, reconciled balances, available credit, statement balance, FX-derived values, calculated exchange-rate effects, settlement relationships, accounting postings, transfer effects, amortization results, and dependent totals. These values must not become authoritative merely because someone directly typed a syntactically valid number into Markdown.

If a human or Hermes changes an upstream fact, such as a payment amount, settlement amount, transaction status, transfer amount, or loan term, the engine must validate the change, recalculate every affected financial value and relationship, and write the resulting canonical state. It must reject, restore, quarantine, or send to review an inconsistent direct edit according to a defined policy. For example, the system must never blindly accept independently edited `remaining_principal`, `interest_component`, or `balance_after` values.

#### Accounting invariants

Where applicable, the engine must enforce explicit invariants including:

- transfers conserve value subject to explicitly recorded FX and fees;
- internal transfers are not income or expense;
- credit-card payments do not create new spending;
- loan principal reduction matches the validated payment allocation;
- refunds reference the original economic event when known;
- balances derive consistently from committed transactions and opening state;
- reconciliation does not silently fabricate money;
- transaction state transitions follow valid rules;
- currencies are explicit and monetary arithmetic uses deterministic decimal or minor-unit handling;
- duplicate imports do not create duplicate economic events;
- calculated fields are reproducible from authoritative inputs where feasible.

### 5.2 KMyMoney integration boundary and prototype gates

The Hermes-owned domain model and canonical financial representation must remain conceptually independent of KMyMoney so the engine can be replaced if evidence later requires it. KMyMoney identifiers, classes, file formats, and storage tables must not become the public filesystem schema or the only means of reconstructing committed user data. Stable Hermes IDs and explicit mapping records are authoritative across engine upgrades and rebuilds.

Before broad UI development or production adoption, a synthetic-data prototype must establish all of the following:

- the supported KMyMoney version and license obligations, platform availability, extension/API stability, and an upstream-maintainable integration path;
- whether required accounting operations can run through a deterministic, supportable boundary without relying on undocumented UI behavior;
- faithful mapping and round-trip reconstruction for Hermes accounts, institution/payment-instrument relationships, transactions, splits, currencies, transfers, loans, schedules, reconciliation, budgets, and investments when enabled;
- monetary precision, supported currency scales, rounding, serialization, and behavior for BHD, PKR, SAR, and other selected currencies, including the claimed exact/rational-style handling;
- parity on representative synthetic accounting fixtures, including cross-currency transfers with fees, card liability payments, loan allocations, pending-to-posted changes, refunds, reconciliation, and imported corrections;
- crash-safe commit and recovery behavior when engine state and canonical Markdown both need updates, including rebuild from canonical files and detection of stale or conflicting engine state;
- a clear strategy for KMyMoney working storage: in-memory or generated/rebuildable where feasible, otherwise documented, backed up, auditable, and transactionally coordinated with canonical files;
- a decision record describing limitations, unresolved risks, and whether the chosen integration remains acceptable.

If a gate fails, record the incompatibility and a concrete alternative or deferred decision before expanding the integration. Do not silently relax canonical-file, offline, engine-authority, or recovery requirements to accommodate the prototype.

### 5.3 App independence and adaptive feature visibility

The application is a complete accounting interface and must remain usable when Hermes is unavailable, a local model is unavailable, AI is disabled, there is no internet, or the user does not want AI. Without Hermes the app must support account and transaction management; transfers, credit cards, loans, subscriptions, budgets, savings, goals, and enabled investments; imports and Raw intake; OCR and evidence review; reconciliation, search, reports, backup, restore, diagnostics, account workbook generation, and canvas visualization. Hermes is an optional alternate interface to the same workspace and may not maintain a competing ledger. In this plan, “Hermes Accounting Domain” names the product's deterministic domain and synchronization layer; it is not a dependency on Hermes chat or a local language model.

First-launch setup should ask which financial features and account structures are relevant, including how many institutions and accounts the user has, how cards link to accounts, cash, savings, credit cards, loans, subscriptions, budgets, savings goals or sinking funds, investments, tracked assets, and the primary reporting currency. Normal navigation and setup should show only relevant/enabled features; for example, an unused Investments module and its setup surface should stay hidden. A possible navigation set is Home, Accounts, Transactions, Subscriptions, Goals, Documents, Reports, Canvas, and Settings, with optional areas shown only when enabled. A user may enable a feature later through Settings → Features → feature → Enable without data loss. Disabling a feature hides its ordinary UI and setup surfaces but never deletes its records; re-enabling restores access. Internal engine capability does not require exposing unused features or creating clutter in the user's workspace.

### 5.4 Infinite Financial Canvas

The product should include an endless, pannable, zoomable financial canvas as a derived visual interface over canonical records and engine results. It is not an accounting authority. One canonical transaction may appear in multiple views without being duplicated. The canvas should support account lanes, timelines, daily or periodic flows, merchant and category clusters, subscription and loan flows, travel spending, and transfer networks; filters and layout changes must not mutate canonical financial records.

Account Lanes are a primary view. Each financial account has a chronological lane, and each transaction appears in the correct lane using stable account identity, event time/date, lifecycle status, and relationship data. A single internal transfer ID connects its source and destination effects across the two lanes. The canvas must show pending, completed, failed, reversed, delayed, and supported partial states from committed records; it must not infer completion or calculate authoritative effects.

Where available and permitted, a node may expose merchant, amount and currency, event date/time, institution/account, masked payment instrument, category, location, balance before and after, lifecycle/reconciliation/review indicators, and links to preserved evidence. Clicking a node opens its canonical record. Balance continuity is a derived display from engine-validated results; a bank-reported observation must remain distinguishable from an engine-calculated balance. Do not persist redundant per-transaction calculated balances as new authoritative facts or cause rewrite storms.

Saved canvas layouts may preserve node positions, zoom, filters, grouping, visual annotations, and pinned record IDs in a canvas/layout file (for example, a `.canvas` file). Such files are UI state only, never financial truth; refresh them from canonical state when records change. Canvas grouping may use date/time, account, card, transfer ID, merchant, category, subscription, and permitted location metadata. Every view and layout must respect the user's location-privacy settings.

## 6. Filesystem-as-interface philosophy

The Accounting folder is the durable boundary between the user, the application, and Hermes. A person should be able to browse it, back it up, copy it to another machine, inspect a record in a text editor, and understand the broad structure without reverse engineering an internal database.

Every important entity should have, where practical:

- a stable immutable ID;
- a human-readable name;
- a predictable location;
- machine-readable structured metadata;
- human-readable Markdown content;
- references to related entities;
- references to source evidence;
- creation, modification, provenance, and review information where relevant.

The preferred representation is one primary same-named or clearly named Markdown file per meaningful entity, with sections that combine structured metadata and explanatory notes. Excessive fragmentation into one tiny file per field is discouraged because it harms readability, increases synchronization conflicts, and makes small-model navigation harder.

Paths are useful for organization but are not identity. Stable IDs and explicit relationships must survive supported folder moves and renames.

### 6.1 Human-readable and machine-readable structure

The initial format should be Markdown with a constrained front matter or metadata block, followed by predictable headings. YAML is a proposed metadata encoding because it is readable and widely supported, but its parser, allowed types, duplicate-key rules, date handling, and canonical serialization require validation before implementation.

Illustrative entity shape:

```markdown
---
entity_type: account
id: account_bhd_current
name: Current Account
currency: BHD
status: active
---

# Current Account

## Notes

Main current account.

## Related evidence

- [[Raw/Bank Statements/2026-08-current.pdf]]
```

The example is illustrative, not a final schema. Required fields, permitted extensions, numeric precision, date formats, relationship syntax, and serialization rules must be specified before records become production data.

### 6.2 Schema versions

Each canonical record should declare a schema version or inherit one from an explicit project manifest. Schema migrations must be deterministic, reviewable, and recoverable. A migration must not silently discard unknown fields or overwrite human notes. Older versions should either remain readable or have a documented upgrade path with a backup and audit entry.

## 7. Source-of-truth strategy

The canonical filesystem and the deterministic accounting engine have complementary roles. Canonical Markdown/filesystem records are the durable, human-readable representation of committed financial records. The accounting engine is the authority that determines whether a financially meaningful operation is valid and calculates its consequences before those results are written. “Canonical” must not be read as permission for Markdown text, a text editor, Hermes, an importer, or an internal cache to bypass the engine.

### 7.1 Canonical data

The following should be treated as canonical or potentially canonical, subject to detailed schema design:

- account identity and account terms;
- transaction and transfer records;
- income, expense, refund, reversal, and fee semantics;
- loan and debt terms and component allocations;
- goal definitions and conceptual allocations;
- subscription definitions and observed renewal relationships;
- budget definitions;
- reconciliation decisions and statement boundaries;
- provenance and links to preserved evidence;
- audit events or a durable journal needed to preserve financial history.

Canonical committed financial state also includes actual transaction postings, transfer sides and relationships, loan payment allocations, currency amounts and settlement results, subscription relationships, goal and sinking-fund records, reconciliation outcomes, and the evidence and audit relationships needed to explain them. Engine-calculated values may be stored in these records, but their authority comes from the validated engine operation that produced them, not from an arbitrary text edit.

### 7.2 Derived data

The following may be generated and rebuilt whenever the canonical representation is complete enough:

- full-text and field indexes;
- account and category lookup tables;
- balance aggregates;
- reports and charts;
- per-account formula workbooks and optional aggregate workbooks;
- derived financial canvas graphs and saved layout state;
- duplicate candidate indexes;
- search caches;
- UI layout state;
- Hermes retrieval indexes;
- thumbnails and OCR acceleration data;
- materialized views of transactions by account, month, or category.

The deletion of derived state must not delete canonical financial records. A rebuild command should report what it reconstructed and any canonical records it could not parse. Search indexes, materialized account views, reports, charts, query indexes, thumbnails, caches, and UI acceleration structures are rebuildable derived state; transactions, transfers, account definitions, loan terms and actual allocations, currency and settlement results, reconciliations, subscriptions, goals, evidence relationships, and audit/provenance information are canonical committed state.

### 7.3 Strict integrity and durable machine state

File-native storage must not become an excuse to weaken double-entry or event integrity. Some durable machine state may be necessary for:

- an append-only operation journal;
- transaction commit markers;
- immutable event IDs and tombstones;
- exact decimal or rational amounts when Markdown serialization cannot safely preserve them;
- import cursors and statement fingerprints;
- conflict baselines needed to detect concurrent edits;
- cryptographic hashes for provenance and duplicate defense.

The proposed rule is to keep such state in a documented `.hermes` area or an adjacent project manifest, make its purpose explicit, and ensure it can be inspected and regenerated where feasible. If a piece of state cannot be rebuilt, the plan must document why it is durable, how it is backed up, and how it relates to the human-readable records. A database may be used internally for performance, indexing, or recovery, but it must not quietly become the only authoritative copy of the user’s financial records unless an explicitly documented integrity requirement makes some durable machine state necessary. Any unavoidable durable machine state must remain documented, backed up, auditable, and linked to the human-readable representation. The final boundary between Markdown and durable machine state is deferred pending accounting-engine and sync investigations; the requirement that the engine determine financial validity and consequences is not deferred.

## 8. Proposed folder architecture

The following is a proposed starting topology. It is intentionally subject to implementation review.

```text
Accounting/
├── Accounting.md
├── Accounts/
│   └── Institution A/
│       ├── Institution.md
│       ├── Current Account/
│       │   ├── Account.md
│       │   ├── Current Account - Master.xlsx
│       │   ├── Cards/
│       │   │   └── Physical Card/Card.md
│       │   ├── Statements/
│       │   ├── Documents/
│       │   └── Reconciliation/
│       └── Savings Account/
│           ├── Account.md
│           └── Savings Account - Master.xlsx
├── Transactions/
├── Transfers/
├── Income/
├── Expenses/
├── Subscriptions/
├── Bills/
├── Budgets/
├── Goals/
├── Debts/
├── Receipts/
├── Statements/
├── Invoices/
├── Reconciliation/
├── Reports/
├── Raw/
│   ├── Bank Statements/
│   ├── Credit Card Statements/
│   ├── Receipts/
│   ├── Screenshots/
│   ├── SMS/
│   ├── PDFs/
│   ├── Invoices/
│   ├── Emails/
│   ├── Images/
│   ├── CSV/
│   ├── OFX-QIF/
│   ├── Loan Documents/
│   └── Other/
└── .hermes/
    ├── cache/
    ├── index/
    ├── schemas/
    ├── sync/
    ├── state/
    ├── journal/
    └── quarantine/
```

### 8.1 Canonical placement versus views

The preferred design is:

- one canonical transaction file under `Transactions/`;
- one canonical transfer file under `Transfers/` when transfer semantics require a first-class object;
- account records under `Accounts/`;
- evidence stored once under a durable evidence or raw path;
- references from transactions, accounts, statements, reports, and review queues;
- rebuildable indexes under `.hermes/index/`.

Accounts should be organized for people by financial institution, then by account, with zero or more payment instruments under each account. An account may have its own account record, account-specific workbook, statements/documents/reconciliation views, and a Cards or Instruments folder. Stable IDs and explicit relationships, not names or paths, define identity. Never use full IBANs or account numbers in filenames; use masked/display-safe labels. A replacement, virtual, or supplementary card does not create a new account by itself. When a card product has its own balance or account relationship, model the underlying financial account and link the instrument to it.

The account folders must not contain copied transaction files unless a future upstream engine makes a compelling, tested case for a mirrored representation. If account-local transaction navigation is needed, use references or generated views and label them as derived. Evidence binaries remain stored once; account-local document areas may contain stable references or generated views rather than duplicate source files.

The boundary between `Raw/`, `Receipts/`, `Statements/`, and other curated evidence folders is deferred. A likely workflow is to preserve the original under `Raw/`, then create a stable evidence record or reference after review without deleting the source.

### 8.2 Institution, account, and payment-instrument identity

A financial institution, financial account, and payment instrument are separate domain entities. One institution may own or provide many accounts. An account may have zero, one, or many payment instruments, such as physical, virtual, replacement, or supplementary cards. Do not create an account solely because a new card is issued. Conversely, a card or provider product with a distinct balance, IBAN, or account relationship must link to the corresponding underlying account. A payment instrument used on a transaction is recorded by stable ID and does not replace the account ID.

Institution, account, and instrument entities must have stable IDs and explicit relationships. Full IBANs, full account numbers, and full card numbers must not be filesystem identifiers or ordinary filenames. Store only the minimum necessary identifying data, mask it in ordinary UI and names, and keep authentication secrets and CVVs out of the workspace. The exact schema and account-folder naming remain proposed design subject to implementation validation.

### 8.3 Account-specific Excel workbooks

This is a hard product requirement: every enabled financial account has its own functional `.xlsx` master workbook, located with that account's human-readable folder. The workbook is a generated, rebuildable account artifact based on canonical records and validated engine results; it is not a second ledger. Canonical transaction files remain singular in `Transactions/`, and their factual rows may be materialized into workbook tables for inspection and formula use. Editing a workbook cell must not directly commit an accounting change; any supported correction must re-enter the typed mutation, validation, preview, and commit flow, with unsupported edits surfaced for review or replaced by a verified regeneration.

Applicable calculated workbook values must contain real spreadsheet formulas rather than hard-coded results. Formulas may calculate account-currency income and expense totals, net movement, monthly/category totals, running views, fee summaries, reconciliation differences, and dashboards from validated input rows. Formula cells should be protected from accidental replacement while remaining inspectable where practical. Formulas may not bypass engine validation or become authoritative for postings, transfers, FX conversion, loan allocation, or other accounting semantics. Formula parity means recalculated workbook outputs agree with deterministic engine results within the documented currency precision; unexplained differences are reported and investigated, never silently overwritten.

Do not sum unlike currencies into one meaningless total. Preserve original and settlement amounts/currencies, account currency, observed FX relationship, fees, and fee currency as separate values. Account-level formula totals use only appropriate account-currency postings or clearly labeled conversions under an explicit policy. The formula design and compatible calculation/recalculation runtime remain future implementation validation; formula text alone is not proof that the workbook calculates correctly.

Workbook generation must be failure-isolated from canonical accounting state. Stage a replacement, validate workbook structure and formulas, recalculate where a compatible spreadsheet runtime is available, and replace the prior usable workbook safely/atomically where supported. A failed generation leaves canonical records unchanged and preserves the previous usable workbook; workbooks can be rebuilt from canonical data. Optional institution/workspace summary workbooks may be considered later and should be generated from canonical records without fragile absolute-path links.

## 9. Markdown and entity model

### 9.1 Entity categories

The first schema investigation should cover at least these entity types:

- project manifest and settings;
- financial institution;
- account;
- payment instrument/card;
- transaction;
- transfer;
- transaction component or posting;
- merchant or counterparty;
- category and subcategory;
- income event;
- expense event;
- refund, reversal, dispute, and chargeback;
- fee;
- loan and debt;
- repayment schedule and payment allocation;
- savings goal and sinking-fund allocation;
- subscription and recurring pattern;
- bill and invoice;
- budget and budget period;
- statement and statement period;
- reconciliation session;
- evidence document;
- import attempt and review item;
- audit event and tombstone.

Not every category must become a separate file. The rule is to introduce a distinct entity when it has its own identity, lifecycle, provenance, relationships, or review state.

### 9.2 Numeric and temporal rules

Accounting amounts must not use binary floating-point as the authoritative calculation representation. The future core must use a deterministic decimal or integer minor-unit model with explicit currency precision and rules for currencies that do not fit a fixed two-decimal assumption. Rounding must be explicit and recorded when it affects a result.

Dates should use unambiguous ISO representations for machine fields. A record may contain separate date-only and instant fields. Time zones must be preserved when known, and imported date-only statements must not be upgraded to invented timestamps.

### 9.3 Unknown fields and extensions

The parser should preserve unknown metadata where safe, surface unsupported fields to review, and avoid destructive round-trips. An extension mechanism may allow application-specific fields, but extensions must not change core accounting semantics without schema registration and validation.

## 10. Stable IDs and relationships

IDs should be unique within the Accounting folder and stable across renames, moves, imports, and rebuilds. A proposed readable format is a typed prefix followed by a collision-resistant identifier, for example `account_bhd_current` or `tx_01JExample`. The final generation algorithm is deferred.

Relationships should use IDs as their authoritative target and may include human-readable path hints for convenience. A broken path hint must not erase a valid ID relationship. The resolver should report:

- missing target;
- duplicate ID;
- type mismatch;
- ambiguous reference;
- stale path hint;
- cycle where a cycle is not allowed.

Examples of relationships include institution-to-accounts, account-to-payment-instruments, transaction-to-account and payment-instrument, transaction-to-evidence, refund-to-original-transaction, statement-to-account, payment-to-loan, subscription-to-observed-transactions, and transfer-to-source/destination accounts.

Relationship edits are financial mutations when they change meaning. A category link can be a normal user edit; changing an expense into a transfer requires stricter validation and an audit event.

## 11. Two-way synchronization

Two-way synchronization is a hard requirement.

### 11.1 App to files

When a user creates or edits data in the application, the sync core must:

1. construct a typed mutation;
2. read the current canonical versions and field ownership;
3. validate required IDs, currencies, amounts, relationships, and invariants through the accounting engine;
4. calculate postings, financial consequences, and dependent recalculations;
5. preview and stage all affected file changes;
6. write a journal entry and commit marker;
7. commit the validated result and atomically replace files where supported;
8. refresh derived state;
9. re-read canonical records and derived state, verify the result, and report the committed IDs and validation result.

Application operations must update Markdown automatically only after the accounting engine has validated and calculated the operation. For example:

```text
User creates transaction in UI
        ↓
Accounting engine validates and posts transaction
        ↓
Canonical Markdown transaction created/updated
        ↓
Account-derived state, indexes, and reports updated
        ↓
Files and derived state re-read and verified
```

### 11.2 Files to app

When a human or Hermes changes a supported Markdown field, the watcher or explicit refresh should detect the change, parse it, compare it with the last known baseline, determine field ownership, and validate it before updating the UI. Descriptive changes may be accepted when they pass schema, relationship, and conflict checks. A financially meaningful change must be converted into a typed accounting operation, passed through the engine, recalculated with all dependencies, and committed by rewriting or updating the affected canonical Markdown. Malformed, contradictory, impossible, or ambiguous changes must remain visible in a review/error state and must not be silently normalized away or treated as authoritative.

The direct-edit flow is:

```text
Markdown changed
      ↓
Watcher detects change
      ↓
Parser identifies changed fields and ownership
      ↓
Validate
      ↓
If descriptive/safe: accept supported change
If financially meaningful: convert to accounting operation
      ↓
Accounting engine calculates and recalculates dependencies
      ↓
Commit resulting canonical state and update affected Markdown
      ↓
Update derived state, re-read, and verify
```

### 11.3 Hermes to files to app

Hermes should normally call a documented local command or library interface. It should not write arbitrary SQL, edit engine-controlled fields as if they were facts, or bypass the sync core. A successful Hermes mutation must result in an accounting-engine validation and calculation, a canonical filesystem change containing the resulting state, a deterministic validation result, and an app/index refresh. A rejected mutation must leave canonical financial data unchanged.

### 11.4 Watcher behavior

The Raw folder and supported canonical folders should eventually be watched automatically. The watcher must debounce bursts, handle partial writes, ignore its own temporary files, recognize atomic renames, avoid infinite loops, and provide an explicit rescan operation. A watcher is an optimization; correctness must not depend on events never being missed.

## 12. Raw inbox

`Raw/` is a first-class intake area where a user can drop financial evidence without completing manual data entry first.

Supported examples include bank and credit-card statement PDFs, CSV exports, OFX, QIF, receipt photographs, receipt PDFs, screenshots, SMS exports, SMS screenshots, invoices, email exports, transaction screenshots, loan documents, subscription receipts, scans, arbitrary supporting images, and unknown files.

The intended workflow is:

```text
File enters Raw/ from the app, filesystem, or optional Hermes attachment
        ↓
Detect format and metadata
        ↓
Hash and identify duplicates
        ↓
Classify document
        ↓
Extract possible financial information
        ↓
Record confidence and uncertainty
        ↓
Match or propose entities and transactions
        ↓
Accounting engine validates candidates, matches duplicates, and determines accounting interpretation
        ↓
Request review where ambiguous or unsupported
        ↓
Commit validated accounting operation and create or update canonical records
        ↓
Link original evidence
        ↓
Update app and rebuildable indexes, then re-read and verify
```

Files sent to Hermes must enter this same secure Raw/evidence intake path and appear in the application's review queue when review is required. Hermes may suggest a classification or candidate account/transaction relationship, but may not keep a second attachment database or bypass the app's evidence and accounting validation pipeline. The intake should preserve the original before OCR or interpretation and record its source and stable evidence identity.

Raw files must not be silently destroyed, rewritten, or moved without a provenance record. Duplicate ingestion should produce a visible relationship to the original rather than a second canonical document. Unknown and unsupported files should remain preserved and reviewable. Treat file names, document contents, OCR text, and embedded instructions as untrusted input; malformed, malicious, low-confidence, impossible, or timed-out extraction must not mutate authoritative financial state.

OCR, parsing, and model extraction are proposals until deterministic accounting-engine validation and, where required, human review establish what may become canonical. An extracted amount, currency, match, status, or relationship must not become authoritative financial state merely because an importer or model produced it. Extraction results should be stored separately from the original so the original remains authoritative evidence.

## 13. Evidence and document management

Evidence must be linkable to institutions, accounts, payment instruments, transactions, loans, subscriptions, assets, transfers, disputes, reimbursements, and other financial entities. One source document may support many records, such as one monthly statement supporting dozens of transactions. App-uploaded, Raw-folder, imported, and Hermes-sent files use the same intake, preservation, extraction, duplicate, and review rules.

Binary files should be stored once. A stable evidence record should include an ID, original path, content hash, media type, observed name, imported time, provenance, review status, and optional extracted text or structured results stored separately from the original. Extraction failure must not alter, replace, or discard the original. The system must support evidence links that survive supported folder moves.

Evidence operations should distinguish:

- original immutable source;
- normalized or converted derivative;
- OCR or extraction output;
- user annotation;
- link between evidence and a financial entity;
- evidence that is missing, unreadable, or quarantined.

The app should make it possible to find a receipt for a transaction, find transactions without evidence, and inspect the source document supporting a statement balance.

## 14. Account model

The account model must support current and checking accounts, savings, credit cards, cash and wallets, prepaid accounts, loans, mortgages, personal debts, investments, assets, liabilities, and custom account types.

An account should carry a stable ID, name, type, currency, institution ID when known, masked identifying information if needed, opening state, status, and links to evidence and statements. Institution, account, and payment-instrument identities remain distinct. A payment instrument such as a physical, virtual, replacement, or supplementary card links to the account it accesses; a product with its own liability/balance is represented by the corresponding financial account and a related instrument. Full IBANs, full account numbers, and full card numbers are not filenames or filesystem identifiers. The model must not store passwords, PINs, CVVs, or authentication secrets.

The model must distinguish an account’s currency from currencies observed in its transactions. Accounts may have restrictions, credit terms, withdrawal rules, or ownership semantics that affect transfer and reporting behavior.

Owned-account transfers must not be counted as ordinary income or expense. Cash holdings must be accounts or equivalent first-class holdings so an ATM withdrawal can move money from a bank account to cash without becoming an expense merely because cash was withdrawn.

## 15. Transaction model

A transaction must support more than a generic date, payee, amount, and category. The proposed model should be able to represent:

- stable transaction ID;
- account and counterparty;
- institution and payment-instrument IDs where known;
- merchant or payee;
- purchase, authorization, posting, and settlement dates;
- original amount and currency;
- authorization amount and currency;
- settlement amount and currency;
- fees, taxes, and other components;
- category and subcategory;
- tags and notes;
- layered raw and normalized location facts, only when retained under the user's location-privacy settings;
- lifecycle status;
- source and source document;
- linked transaction and transfer relationships;
- subscription, loan, refund, dispute, and evidence relationships;
- reconciliation status;
- import and manual-edit provenance.

The schema should require only fields that are meaningful for the specific event. A cash note, pending authorization, bank transfer, loan repayment, and card settlement should not be forced into an identical shape that loses semantics.

### 15.1 Layered location facts and privacy

When location is available and the user has permitted its retention, a transaction should preserve the distinct source facts rather than collapse them into a guessed string. Potential raw values include the bank location text, merchant descriptor, receipt address, and terminal text. Normalized values may include country, region, city, district, venue/branch, and postal code. Optional location attributes include latitude/longitude, timezone, online or in-person mode, merchant country, user-confirmation state, geocoded result, confidence, source provenance, and whether device location was used. Raw and normalized values remain separately identifiable so later corrections do not erase the source wording.

Unknown or conflicting location must remain unknown or in review; do not fabricate a physical purchase location from a merchant headquarters or other weak inference. Online transactions may identify the merchant country while leaving city and physical location empty. Geocoding and location enrichment are optional and configurable; device-derived location and precise coordinates require explicit opt-in. The user must be able to choose no retained location, country only, city/region, branch/address, precise coordinates, or permitted device-derived location.

Location privacy settings govern extraction and retention of structured location metadata, as well as its use in search indexes, reports, account workbooks, canvas nodes/grouping/layouts, logs, exports, backups where configurable, and Hermes/model context. When the user chooses no retained location, preserve the original evidence unchanged as required but do not extract or persist a separate structured location value. The exact schema and retention controls remain subject to implementation design; raw source evidence and its metadata must still follow the evidence-preservation rules.

## 16. Multiple currencies and international spending

Multi-currency support is mandatory. The model must distinguish account, merchant/original, authorization, settlement, transfer source, transfer destination, and fee currencies.

An international purchase may contain the following values independently:

```yaml
original:
  amount: 4999
  currency: PKR
settlement:
  amount: 6.740
  currency: BHD
fees:
  - amount: 0.150
    currency: BHD
    type: fx_fee
```

The accounting engine determines the accounting consequences of the original charge, authorization, settlement, conversion, and fee relationship. Markdown records the validated observed values and calculated relationships after the engine commits them; Hermes must not manually infer or calculate the settlement result.

The system must preserve actual known values rather than reconstructing them from a preferred exchange rate. It must support FX rates, markups, conversion fees, foreign card fees, dynamic currency conversion, refunds at different exchange rates, multi-currency internal transfers, third-currency fees, and changes between authorization and settlement.

Reports must state the conversion basis, rate date, rate source, rounding, and whether the number is observed, imported, user-entered, or derived. A report that converts across currencies without a known or selected rate must identify the unresolved conversion rather than presenting false precision.

International spending may involve merchant currency, account currency, intermediary currency, bank conversion, card-network conversion, markup, international fees, ATM fees, merchant surcharges, DCC, pending conversion, and later final settlement. These are separate concepts even when a bank statement provides only one combined number.

## 17. Pending, authorized, posted, and settled transactions

The lifecycle should support pending, authorized, posted, settled, reversed, declined, failed, cancelled, refunded, partially refunded, disputed, chargeback, and provisional-credit states.

A pending card authorization and its later posted settlement should usually be recognized as the same underlying purchase when evidence supports that match, rather than becoming two expenses. The match must preserve historical pending values and the final posted values, including changes in amount, date, currency, and fees.

State transitions must be explicit and auditable. A reversal is not the same as a refund; a declined authorization is not a completed expense; a provisional credit is not necessarily a final chargeback win. The importer should retain provider statuses and map them to project semantics without discarding the original wording.

The accounting engine must validate every financially meaningful lifecycle transition and calculate its balance, liability, expense, refund, fee, FX, and reconciliation consequences before the resulting status and values become canonical Markdown state.

## 18. Credit cards

Credit cards are liability accounts with extensive lifecycle requirements. The future product should represent credit limit, available credit, outstanding balance, statement balance, statement period, due date, minimum payment, full payment, partial payment, purchases, pending authorizations, reversals, interest, late fees, annual fees, cash advances, installment plans, balance transfers, refunds, partial refunds, chargebacks, disputes, provisional credits, foreign-currency purchases, and positive credit balances.

A payment from a bank account to a credit card must be modeled as a liability payment or internal transfer, not as a new expense plus new income. Interest, late fees, annual fees, cash-advance fees, and purchase expenses have different semantics and should remain separable where the evidence supports the distinction.

The accounting engine must calculate the bank-account decrease, liability decrease, available-credit effect, statement/current-balance consequences, and any distinct interest or fee postings. A Markdown or Hermes edit must not create a false expense by bypassing that calculation.

Statement balance and current outstanding balance are not interchangeable. Reports must identify which balance concept they use and the statement date or observation date.

## 19. Loans and debt

Loans and debt are first-class entities. Support personal loans, mortgages, car loans, education loans, informal debts, money owed to another person, and money another person owes the user.

The model should support principal, fixed or variable interest, fees, repayment schedules, due dates, remaining principal, missed payments, late fees, payment holidays, extra payments, early settlement, refinancing, restructuring, balloon payments, changing rates, and penalties.

A repayment must be splittable into components. For example:

```text
300 BHD payment
230 BHD principal
65 BHD interest
5 BHD fee
```

Principal reduction, interest expense, and fees must not be flattened when the source or user knows the components. Schedule calculations must be deterministic and use explicit rounding rules. Imported schedules must preserve their source values even when the application computes a comparison or projection.

When a loan payment is entered or its upstream facts change, the accounting engine calculates and validates the principal, interest, fees, remaining principal, related account postings, and dependent schedule effects before the calculated result is written to Markdown. The file stores the resulting allocation; it does not calculate it.

## 20. Savings

The model should support ordinary savings accounts, high-interest savings, locked savings, withdrawal restrictions, interest payments, automatic transfers, minimum balances, withdrawal penalties, and goals associated with savings.

Savings goals and accounts must remain distinct concepts. A real bank balance is an account fact; a goal allocation may be a conceptual partition of money. The system must not change actual bank balances merely because a user allocates part of a balance to a sinking fund.

## 21. Goals and sinking funds

Goals are first-class planning entities for emergency funds, holidays, university, a car, a house, a laptop, a wedding, an investment target, or a custom target.

A goal may track goal amount, currency, target date, current allocated amount, linked accounts, linked transactions, contributions, withdrawals, progress, and an optional recurring contribution target. It need not require a separate real bank account.

The implementation must distinguish actual transactions from conceptual allocations, support multiple currencies explicitly, and explain whether progress is based on observed balances, allocated funds, or a user-defined valuation.

## 22. Subscriptions and recurring payments

Subscriptions should support monthly, annual, and custom recurrence; free trials; introductory prices; price increases; foreign currencies; renewal dates; failed renewals and retries; pauses; cancellations; prorated charges; partial refunds; discounts; tax changes; and family or shared plans.

Subscriptions must connect to observed transactions without claiming certainty merely because a merchant name repeats. Detection should preserve a recurring-pattern proposal, confidence, evidence, and review state. A price increase, FX change, tax change, or one-time surcharge should be explainable from observed components where possible.

## 23. Transfers

Transfers need explicit semantics for owned-account movement. Support same-currency and cross-currency transfers, domestic and international transfers, pending transfers, delayed receipt, failures, reversals, transfer fees, destination-side fees, intermediary bank fees, and money in transit.

A cross-currency transfer should preserve both sides:

```yaml
source:
  account: current_bhd
  amount: 100
  currency: BHD
destination:
  account: savings_usd
  amount: 265
  currency: USD
```

The source and destination effects must be linked to one canonical transfer object by a stable transfer ID. A transfer fee may be a separate expense component or a linked fee event, depending on evidence and the accounting engine’s representation. The operation must not create artificial spending and artificial income merely because two account statements show opposite sides. In the derived Account Lanes canvas, the outgoing and incoming effects appear in their respective account lanes and are connected by that same transfer ID; lifecycle, currencies, observed FX, and fee relationships come from committed state.

The accounting engine must validate and apply both transfer sides atomically, enforce transfer neutrality subject to explicit FX and fees, and recalculate both affected account states before canonical Markdown is updated.

## 24. Income, expenses, cash, budgets, and commitments

### 24.1 Income

Support salary, freelance work, business income, interest, dividends, reimbursements, refunds, gifts, cash income, irregular income, and multi-currency income. Transfers that are not income, and refunds that correct prior spending, must retain their distinct semantics.

### 24.2 Expenses and components

Expenses should support categories, subcategories, tags, tax or fee components, notes, evidence, and links to the relevant account and merchant. A single source event may have multiple components, such as an item, tax, tip, surcharge, and fee.

### 24.3 Cash

Cash should support ATM withdrawals moving money from bank to cash, cash purchases, cash deposits, cash received, multiple physical cash currencies, manual adjustments, and lost or unaccounted cash. An ATM withdrawal must not automatically become an expense.

### 24.4 Budgets and commitments

The future UI should support categories, subcategories, monthly, annual, and custom-period budgets, rollover, planned spending, actual spending, recurring commitments, overspending, multi-currency reporting, and historical comparison. Budget allocation must be distinct from actual account balances and from conceptual goal allocations.

## 25. Refunds, reversals, disputes, chargebacks, and fees

The system must support full refunds, partial refunds, multiple partial refunds, refunds with a different settlement amount due to FX, refunds to the original method or another known method, merchant reversals, authorization reversals, disputes, chargebacks, provisional credits, chargeback loss, and chargeback win.

Relationships between the original transaction and corrections must be preserved. Corrections should not silently rewrite the original amount or erase its historical state.

Fees may be separate or embedded. Categories should include account fee, annual card fee, late fee, ATM fee, FX fee, transfer fee, loan fee, merchant surcharge, cash-advance fee, brokerage or investment fee, and intermediary-bank fee. A fee may use a different currency from the primary transaction.

## 26. Dates and time

Where known, preserve purchase date, authorization date, posting date, settlement date, statement date, due date, transfer initiation date, and transfer completion date separately.

The system must handle time zones, transactions near midnight, statements with local dates only, backdated corrections, and bank changes to posting dates. It must never infer a precise instant from a date-only source without marking the inference. Reports should state which date concept they use.

## 27. Reconciliation

Reconciliation should compare canonical records against statements and support unreconciled, cleared, reconciled, mismatch, and needs-review states.

It should detect missing transactions, duplicates, incorrect opening balances, incorrect closing balances, unexplained differences, and overlapping statement periods. The system must not silently fabricate balancing transactions. If a user chooses to record an adjustment, that adjustment must be explicit, attributed, and auditable.

A reconciliation session should identify the account, statement source, period, opening and closing observations, selected records, difference calculation, decisions, and final state. The implementation must define whether reconciliation decisions are canonical records, durable events, or both.

## 28. Imports and duplicate detection

Importers should support the formats that can be validated safely, including CSV, OFX, QIF, bank statement PDFs, credit-card statements, receipts, screenshots, SMS exports, invoices, and email exports. Each import attempt should have a source hash, parser version, detected account, observed period, status, extracted candidates, accepted mutations, rejected candidates, and review items.

The import boundary is:

```text
Bank statement / CSV / PDF / SMS / screenshot
        ↓
Extraction/import subsystem
        ↓
Candidate financial facts
        ↓
Accounting engine validation and matching
        ↓
Duplicate detection and accounting interpretation
        ↓
Canonical transaction creation/update
        ↓
Markdown updated, derived state refreshed, result verified
```

OCR or AI extraction may suggest facts, but it must not become authoritative financial state without deterministic validation. The engine, not the importer or Hermes, decides whether a candidate is a new event, an update to an existing event, a pending-to-posted transition, a duplicate, or a review item.

Duplicate defense should combine raw-file hashing, account, amount, currency, date, merchant, bank-provided transaction IDs, statement identifiers, authorization and posting relationships, and existing provenance. No single heuristic is sufficient for every institution.

Overlapping statements must not duplicate transactions. Pending and posted records require careful matching. When evidence is insufficient, the importer should produce a candidate or review item rather than automatically creating a second canonical transaction.

## 29. Uncertainty and review workflow

OCR and extraction may be wrong. Every extracted candidate should preserve the original file, extracted value, confidence, uncertainty reason, parser or model version, and review status. A representative status is:

```yaml
import_status: needs_review
confidence: low
```

The system should expose a review queue for ambiguous amounts, currencies, account matches, duplicate candidates, pending-to-posted matches, OCR conflicts, missing evidence, malformed Markdown, unsupported fields, and reconciliation differences.

Review actions should be explicit: accept, edit, reject, merge, split, link, defer, or mark unsupported. An accepted result should record who or what accepted it, when, and which source supported the decision. A rejected extraction must not destroy the original source.

## 30. Manual edits, file moves, deletion, and history

Users and Hermes may improve merchant names, categories, notes, tags, links, descriptions, and evidence associations. The sync system must not destroy a valid manual edit merely because new bank metadata arrives.

The design must define field ownership and precedence rules, such as source-owned, imported-but-overridable, user-owned, or derived. A fresh import should update an imported field only under documented conditions and should preserve user-owned values. Engine-controlled financial fields are never made authoritative by direct text editing; an upstream factual change must enter the accounting engine so dependent postings, balances, allocations, settlement relationships, and reconciliation results are recalculated before commit.

Stable IDs should allow supported renames and moves. The application should repair rebuildable indexes and report missing or duplicated IDs without making destructive assumptions.

Financial history must distinguish an erroneous imported duplicate, intentionally deleted draft, bank reversal, refund, voided transaction, hidden or archived state, and genuine deletion. Consider tombstones, immutable event history, and change logs. Do not silently rewrite financial history or treat a file disappearance as proof that the financial event never existed.

## 31. Hermes integration

Hermes should be able to read accounts, read transactions, search spending, retrieve evidence, inspect subscriptions, inspect loans, inspect goals, inspect balances, propose categories, request safe changes, add notes, attach evidence, create validated transactions, create validated transfers, and answer questions grounded in local data.

Hermes is primarily an intent interpreter, search/retrieval layer, navigation layer, natural-language interface, and constrained command client. It may identify what the user wants, request an operation, read and summarize financial information, and propose categories, notes, links, or other supported changes. It must not independently calculate authoritative balances, FX settlement, loan amortization, loan principal or interest, credit-card liability, reconciliation, transfer accounting, refund accounting, or financial state transitions. A small model should produce a typed request; the deterministic accounting engine calculates the actual allocation and consequences.

Hermes must never bypass deterministic validation. The preferred interface is a constrained local command layer with discoverable schemas and structured results. Read operations may return records, aggregates, provenance, and review warnings. Mutation operations should return validation errors, a proposed diff, a commit result, affected IDs, and a verification summary. Only after engine validation and calculation should a result become canonical Markdown state.

Hermes responses should clearly distinguish observed values, derived values, proposals, and unresolved questions. A model must not be encouraged to infer missing account IDs, currencies, transaction identity, or transfer semantics from vague text when a deterministic lookup or user review is required.

## 32. Small-model and approximately 1B-model requirements

The architecture must intentionally support a low-capability model. The model should not need to understand complex accounting internals, calculate loan amortization, maintain double-entry invariants, calculate FX consequences, mutate SQL tables, infer undocumented relationships, guess IDs, guess currencies, or guess whether an event is a transfer or expense.

For example, Hermes may produce a typed request such as `ACTION: record_loan_payment` with a loan ID, source account ID, amount, currency, and date. The accounting engine must calculate and validate the principal, interest, fee, remaining principal, related postings, and dependent balances. Hermes must not supply authoritative allocations, and those allocations become canonical Markdown state only after engine validation and commit.

The model should be able to issue constrained intent such as:

```text
ACTION: reclassify_transaction
TRANSACTION_ID: tx_1234
NEW_CATEGORY: groceries
```

or:

```text
ACTION: create_transfer
SOURCE_ACCOUNT: account_bhd_current
DESTINATION_ACCOUNT: account_usd_savings
SOURCE_AMOUNT: 100
SOURCE_CURRENCY: BHD
DESTINATION_AMOUNT: 265
DESTINATION_CURRENCY: USD
```

The command layer should have a small vocabulary, stable field names, explicit allowed values, machine-readable error codes, and examples for common operations. It should support a discovery operation that returns the relevant schema rather than requiring a model to memorize every entity type.

The acceptance bar is not that a small model can perform arbitrary accounting. It is that it can safely complete supported, well-bounded tasks by selecting a valid command and relying on deterministic code for semantics.

Small-model retrieval should resolve candidate merchants, date ranges, account/instrument IDs, evidence IDs, and permitted location-based filters deterministically, then provide only a small relevant context package. For example, a request for spending in a city must use the retained raw/normalized location evidence and privacy policy; Hermes may explain the result but must not infer missing locations or calculate financial totals itself.

## 33. Deterministic mutation API and commands

The exact transport is deferred, but the mutation model should include:

1. **Discover** — identify supported operations and required fields.
2. **Read** — retrieve records or verified derived answers.
3. **Propose** — construct a typed change without committing it.
4. **Validate** — check schema, references, permissions, accounting invariants, currency rules, conflicts, field ownership, and evidence requirements.
5. **Calculate** — deterministically compute postings, allocations, conversions, state transitions, and all dependent financial consequences. This may occur as part of validation or preview, but it must occur before commit.
6. **Preview** — return affected files, record diffs, balance effects, calculated results, dependent changes, and warnings.
7. **Commit** — journal and atomically apply the approved change and write the resulting canonical Markdown/files.
8. **Update derived state** — rebuild or update indexes, reports, materialized views, and caches without replacing canonical records.
9. **Re-read and verify** — re-read canonical records and derived state, confirm the committed result, and only then report success.

The formal mutation pipeline is:

```text
DISCOVER
   ↓
READ
   ↓
PROPOSE
   ↓
VALIDATE
   ↓
CALCULATE
   ↓
PREVIEW
   ↓
COMMIT
   ↓
WRITE CANONICAL MARKDOWN
   ↓
UPDATE DERIVED STATE
   ↓
RE-READ
   ↓
VERIFY
```

Dependency recalculation is part of `CALCULATE` and `COMMIT`. The engine must determine and safely update all affected dependent state when a payment changes principal, interest, remaining balance, or future schedule; a transfer changes both accounts; a pending transaction becomes posted; a settlement changes FX reporting; a refund changes net expense; a transaction changes between transfer and expense semantics; a credit-card payment changes liability and source-account balances; an imported correction changes reconciliation; or a duplicate is removed. Hermes must not be expected to locate and edit every dependent field manually.

Initial command families should include read account, search transactions, retrieve evidence, reclassify transaction, edit notes or tags, attach evidence, create transaction, create transfer, record refund, reconcile statement, and review import candidate. Each command must declare whether it is read-only, reversible, financially material, or review-gated.

The engine must reject malformed IDs, missing accounts, mismatched currencies, invalid amounts, unsupported state transitions, duplicate identifiers, stale baselines, ambiguous matches, and changes that violate the chosen accounting model. It must never accept a free-form instruction as a mutation without converting it to a typed validated command.

## 34. Security and privacy

The product should be fully local by default with no unnecessary cloud dependency. Optional cloud or AI integrations, if ever considered, require explicit consent, data minimization, redaction, and a threat review.

The system must not store PINs, CVVs, banking passwords, authentication secrets, or full card numbers unless a future security review proves a narrowly scoped need and defines protected storage. Account identifiers should be masked or minimized in logs and exports. Logs must avoid raw financial documents and secrets.

Plan for local encryption if feasible, safe backups, secure export behavior, secure deletion considerations, file permissions, malicious-document defenses, archive-bomb defenses, parser isolation, and safe handling of untrusted PDFs, images, spreadsheets, and email exports.

The repository must never contain real private financial data. All development fixtures, screenshots, examples, and tests must use synthetic or explicitly sanitized data. A pre-commit or review check should eventually detect accidental secret or personal-finance fixture leakage.

## 35. Auditability and provenance

Financial changes should be traceable through stable event IDs, source files, created and modified times, importer version, mutation source, user versus app versus Hermes attribution, previous values where useful, and reconciliation status.

Auditability must remain readable. Do not create an unreadable event store merely to obtain history. A proposed design is to keep concise audit metadata with canonical records and a durable append-only journal for material mutations, with derived reports showing the history in human terms.

The project must decide which edits are immutable events, which are current-state fields, and how to reconstruct a record’s history after a backup restore. That decision is deferred pending upstream accounting-engine evaluation and crash-recovery prototypes.

## 36. Sync conflicts and failure handling

Conflicts may arise when the user edits Markdown while the app edits the same record, Hermes edits while the app is open, sync software changes timestamps, a write is partial, Markdown is malformed, IDs are duplicated, an index is stale, an attachment is missing, or account currencies conflict.

The sync core should use content hashes or equivalent baselines rather than timestamps alone. It must detect concurrent changes, preserve both versions or a recoverable patch where possible, and place financially material ambiguity into review. It must never silently choose a winner when the choice changes accounting meaning.

Conflict states should identify record ID, files involved, base version, local version, external version, detected fields, safe automatic resolutions, and required human decisions. A repair or reindex operation must be explicit and report its result.

## 37. Transactional file writes and crash recovery

Financial operations affecting several records must not leave the filesystem half-updated. A transfer may touch source account, destination account, transfer entity, balances or indexes, and fees. A reconciliation may touch statement metadata, selected transactions, and the session result.

The proposed write protocol is:

1. Read and hash the current canonical inputs.
2. Validate the typed operation and all resulting invariants.
3. Build a complete staged change set in a private temporary location.
4. Write a journal entry with operation ID, affected paths, expected hashes, and intended replacements.
5. Flush or otherwise persist the staged files according to platform capability.
6. Atomically replace files where the filesystem supports it.
7. Write a commit marker.
8. Rebuild or update derived state.
9. Verify the committed records and mark the operation complete.

On restart, the core must inspect incomplete journal entries, determine whether the operation is uncommitted, fully committed, or partially applied, and recover deterministically. Temporary files must not be mistaken for canonical records. The precise atomicity guarantees of Windows, Linux, and macOS filesystems require platform testing.

Crash safety is a hard requirement that must be demonstrated through deterministic fault injection, not inferred from atomic rename support. Tests must terminate the process before staging, after staging, after journal creation, between multi-file writes, before and after replacement, after canonical commit but before derived-state refresh, and after commit but before acknowledgement. Exercise transfers, loan payments, evidence association, reconciliation, migration, backup, and restore. After recovery each operation must yield either its valid pre-operation state or its complete valid post-operation state; never a half-transfer, half-loan-payment, broken relationship, or orphaned commit.

At minimum, test an internal transfer with Account A at 1,000 SAR and Account B at 200 SAR, transferring 300 SAR from A to B. Kill the app after A is staged or changed but before B would normally update. Recovery may produce only A=1,000/B=200 or A=700/B=500; A=700/B=200 and A=1,000/B=500 are invalid. Repeat at relevant commit boundaries to prove recovery is deterministic.

## 38. Edge-case catalogue

Edge cases are part of the architecture, not merely future bug fixes. The project should maintain an exhaustive machine-testable catalogue. Every discovered situation should be evaluated for representation, filesystem form, app behavior, import behavior, sync behavior, Hermes behavior, validation, recovery, and tests.

The catalogue must include at least:

- duplicate and overlapping statements;
- pending authorization later posted at a changed amount;
- declined authorization followed by a separate successful purchase;
- full, partial, and multiple refunds;
- refund in a different currency or at a different FX rate;
- reversal versus refund;
- disputed transaction with provisional credit and final loss or win;
- same-currency and cross-currency transfers;
- transfer fee in a third currency;
- destination-side and intermediary fees;
- ATM withdrawal, cash purchase, cash deposit, and missing cash;
- credit-card statement balance versus current balance;
- credit-card payment, interest, annual fee, cash advance, installment, and balance transfer;
- loan payment split into principal, interest, and fee;
- variable rate, payment holiday, extra payment, refinancing, and early settlement;
- subscription trial, renewal, price change, failure, retry, pause, cancellation, and prorating;
- tax, tip, surcharge, and embedded fee;
- transaction at midnight or across time zones;
- date-only imported statements;
- malformed Markdown, duplicate IDs, missing IDs, stale indexes, and missing evidence;
- file move, rename, delete, restore, and partial write;
- concurrent app, human, and Hermes edits;
- OCR ambiguity and unknown file types;
- account currency change or conflicting currency claims;
- empty or zero-amount records where the source permits them;
- negative balances, overpayments, credit balances, and chargebacks;
- corrupted or malicious documents;
- unsupported schema versions and unknown fields.

The catalogue should become a set of fixtures and acceptance tests rather than a prose list only.

## 39. Testing strategy

The future project must establish layered tests:

- unit tests for amounts, currencies, dates, IDs, relationships, status transitions, and validators;
- accounting invariant tests for balances, transfer neutrality, liability payments, loan components, refunds, and rounding;
- schema validation tests for valid, malformed, versioned, and extended Markdown;
- fixture-based import tests for CSV, OFX, QIF, PDFs, images, SMS, invoices, and unknown files;
- two-way sync tests for app-to-file and file-to-app changes;
- filesystem mutation tests for moves, renames, partial writes, Unicode names, and path length;
- crash-recovery tests at every journal and atomic-replace boundary;
- duplicate import tests and pending-to-posted matching tests;
- multi-currency, FX, fee, refund, credit-card, loan, savings, goal, subscription, budget, and reconciliation tests;
- KMyMoney adapter and domain-mapping tests for exact amounts, split transactions, cross-currency operations, and lossless round trips;
- application workflows with Hermes, local models, and network access unavailable;
- adaptive feature onboarding, hide/re-enable behavior, and proof that disabling a feature never deletes records;
- per-account workbook generation tests for actual formulas, formula recalculation, formula-to-engine parity, multiple currencies, failed generation, and preservation of the prior usable workbook;
- canvas graph tests for correct account lanes, linked transfer edges, lifecycle states, derived balance continuity, location privacy, and saved layouts that cannot mutate canonical data;
- location-provenance tests for preserved raw and normalized values, online/unknown location, opt-out, and no fabricated coordinates or physical locations;
- OCR/Hermes attachment tests proving one evidence pipeline, original-file preservation, review gating, and no mutation after extraction failure or malicious input;
- malformed Markdown, duplicate ID, missing attachment, stale index, and conflict tests;
- evidence linking and one-source-to-many-record tests;
- Raw watcher tests for debounce, duplicate events, partial files, and rescan;
- Hermes command fixtures using constrained small-model prompts and invalid command variants;
- cross-platform packaging and filesystem behavior tests;
- privacy tests ensuring secrets and real personal data cannot enter fixtures, logs, or reports.

A core acceptance test should be:

> Can a constrained small model read the relevant Markdown, request a supported operation, have deterministic code validate and execute it, and obtain a correct verified result?

The testing strategy must also include authority-separation tests:

1. Hermes requests a loan payment; the engine calculates principal, interest, fees, remaining principal, and related postings; Markdown receives the calculated result; Hermes does not calculate the allocation.
2. A human edits a loan payment amount in Markdown; the engine detects the semantic change and recalculates principal, interest, fees, remaining balance, and affected future schedule state.
3. A human changes only `remaining_principal`; the system does not blindly accept an inconsistent calculated value and instead rejects, restores, or sends it to review according to policy.
4. Hermes reclassifies a transaction; the engine distinguishes a permitted category change from a financial semantic change such as expense-to-transfer and applies the appropriate validation.
5. A BHD account pays a PKR subscription; the engine preserves original and settlement currencies and deterministically models FX and fees.
6. A credit-card payment decreases the bank account and the card liability without creating a false new expense.
7. A transfer updates both sides atomically and creates no artificial income or expense.
8. Deleting a cache or index leaves canonical Markdown intact and rebuilds derived state successfully.
9. A direct edit of an engine-controlled field is not trusted merely because it is valid YAML or Markdown.
10. Every successful accounting-engine operation writes canonical files, re-reads them, verifies the result, and only then reports success.

Tests must assert both the financial result and the filesystem result. A test is incomplete if the balance is correct but provenance, evidence, history, or recoverability is wrong.

## 40. Cross-platform requirements

The resulting application should ultimately be a proper application for Windows, Linux, and macOS, with shared core logic and platform-appropriate packaging.

The project must investigate:

- file watcher APIs and event semantics;
- atomic replacement and durability guarantees;
- file locking and concurrent access;
- Unicode and normalization behavior;
- path separators, reserved names, and long paths;
- permissions and protected directories;
- system time zones and locale formatting;
- desktop integration and safe file opening;
- installer, update, backup, and uninstall behavior;
- local search and OCR dependencies;
- accessibility and keyboard navigation.

The canonical file format must remain portable even if UI or packaging layers differ by platform.

## 41. KMyMoney foundation and integration validation

KMyMoney is the intended accounting-engine and financial-domain foundation for Hermes Accounting, subject to implementation/prototype validation. This direction reflects its mature personal-finance orientation and candidate strengths in double-entry semantics, split transactions, loans, account models, schedules, budgets, investments, currencies, reconciliation, exact/rational-style monetary handling, transaction boundaries, storage and importer extension points, tests, and cross-platform foundations. The plan does not treat these claims as validated fit for Hermes: the prototype gates in section 5.2 must test each needed capability and the integration boundary with synthetic data.

The evaluation must distinguish using KMyMoney's accounting/domain machinery from adopting its UI or internal file/storage schema. Hermes owns the stable domain mapping and canonical Markdown representation. Evaluate the least invasive maintainable integration path and upstream contribution strategy; avoid an unnecessary long-lived fork or patches that make upgrades unsafe. Any unresolved implementation choice, compatibility issue, or production adoption decision remains a future investigation and must be recorded with evidence.

Evaluation criteria should include:

- Windows, Linux, and macOS support;
- open-source license and forkability;
- multi-account and multi-currency support;
- credit cards, loans, savings, budgeting, subscriptions, transfers, and reconciliation;
- investments if desired;
- import and export formats;
- reports and accounting integrity;
- automated tests and project health;
- UI quality and accessibility;
- maintainability and contributor activity;
- separation of finance engine and UI;
- feasibility of filesystem-native synchronization;
- ease of deterministic external commands;
- ability to package independent desktop applications;
- database architecture and rebuildability;
- treatment of pending, posted, refund, dispute, and FX edge cases;
- extension points, API stability, and migration behavior;
- ability to retain source provenance and user edits.

If a required prototype gate fails, document the failure and then investigate alternatives against the same accounting, canonical-file, portability, privacy, and recovery requirements. Do not treat a feature list, upstream reputation, or the existence of plugins as proof that an integration is safe or maintainable.

## 42. Proposed implementation phases

The following sequence is a planning proposal, not an authorization to begin implementation during this documentation-only run.

### Phase 0 — KMyMoney prototype and decision preparation

Define the evaluation rubric, threat model, accounting vocabulary, synthetic fixture policy, field-ownership policy, accounting-engine authority boundary, and decision-record format. Prototype the KMyMoney operations, version/license/API fit, platform support, currency precision, mapping/round trips, and crash-safe relationship to canonical Markdown described in sections 5.2 and 41. Record results before committing to a production integration mechanism.

### Phase 1 — Hermes domain, canonical format, and invariants

Specify the Hermes-owned institution/account/payment-instrument model, entity schemas, stable IDs and KMyMoney mappings, relationships, numeric precision, dates and location provenance, schema versioning, field ownership, canonical versus derived state, core accounting invariants, and representative synthetic fixtures. Validate lossless representation and the design with accounting review.

### Phase 2 — KMyMoney adapter, file synchronization, and recovery

Implement the supported Hermes-domain/KMyMoney adapter, canonical parsing and validation, stable relationship resolution, typed commands, dependency handling, field-ownership enforcement, two-way synchronization, and coordinated transactional writes. Build the journal and fault-injection harness early. Confirm that canonical Markdown is written only from validated engine results, recovery yields an all-or-nothing valid state, and any KMyMoney working state can be reconciled or rebuilt safely.

### Phase 3 — Import and Raw workflow

Add source preservation, file hashing, duplicate detection, supported importers, OCR/extraction result storage, deterministic engine validation of candidate facts, uncertainty states, review queue, evidence linking, the shared app/filesystem/Hermes attachment intake pipeline, and Raw folder watching.

### Phase 4 — Account workbooks and formula validation

Generate one functional `.xlsx` master workbook for each enabled account with real formula-driven totals and derived values from validated canonical inputs. Establish currency-safe formula rules, recalculation checks, formula-to-engine parity, discrepancy reporting, and failure-isolated safe replacement before relying on the workbooks.

### Phase 5 — Canvas data and account-lane semantics

Define derived canvas nodes/edges, account lanes, one-ID transfer visualization, transfer lifecycle, cross-currency details, balance continuity, location filtering under privacy controls, and saved layout state. Prove the graph reflects canonical/engine state and cannot mutate accounting data.

### Phase 6 — Complete adaptive application UI

Build the cross-platform desktop application over the rules core. Deliver full ordinary accounting, Raw/evidence, OCR review, reconciliation, search, reports, backup/restore, diagnostics, workbooks, and canvas workflows without Hermes. Add first-launch adaptive feature setup and prove later feature enablement restores retained data.

### Phase 7 — Optional Hermes interface

Expose constrained discovery, read, propose, validate, calculate, preview, commit, write, re-read, and verify commands. Add small-model fixtures proving Hermes submits typed intent and uses the shared file-intake pipeline while deterministic software performs accounting; include refusal behavior for ambiguous or unsupported operations. Keep the app fully functional if this phase is absent or disabled.

### Phase 8 — Expanded financial domains

Add and validate credit cards, loans, debt, savings goals, subscriptions, budgets, enabled investments, and complex dispute and FX workflows through the standalone app and engine adapter before broad exposure.

### Phase 9 — Packaging, migration, and hardening

Deliver Windows, Linux, and macOS packaging, upgrade and migration tools, privacy review, malicious-document defenses, backup and restore, accessibility improvements, and release acceptance testing.

Phase gates should require passing invariants, filesystem integrity checks, recovery tests, and a documented decision review before moving to the next phase.

## 43. Acceptance criteria

The project should not be considered production-ready until it can demonstrate all of the following with synthetic fixtures and repeatable tests:

1. Canonical records are understandable in the filesystem without opening an opaque database.
2. Stable IDs survive supported file moves and renames.
3. Derived indexes and caches can be deleted and rebuilt without losing canonical financial records.
4. App-created changes update the correct files and preserve provenance.
5. Supported human or Hermes Markdown edits are validated and reflected in the app.
6. Invalid, ambiguous, conflicting, or unsupported edits are surfaced without silently changing financial meaning.
7. Multi-record mutations are journaled, recoverable, and verified after commit.
8. Transfers do not become artificial income and expenses.
9. Multiple currencies preserve original, authorization, settlement, transfer, and fee values where known.
10. Pending-to-posted matching does not create duplicate spending when the evidence supports one purchase.
11. Credit-card payments, refunds, reversals, chargebacks, loan components, cash movement, and fees retain their distinct semantics.
12. Raw source files remain preserved and duplicate or uncertain imports enter review.
13. Evidence can support many records without unnecessary binary duplication.
14. Reconciliation detects missing, duplicate, and mismatched records without inventing balancing entries.
15. Manual edits survive later imports according to documented ownership rules.
16. Audit history identifies what changed, why, when, and whether the source was a human, app, importer, or Hermes.
17. Hermes can complete supported read and mutation tasks through deterministic commands without direct database access.
18. A constrained small model can use the command layer safely on representative fixtures.
19. No secrets or real private financial data are present in the repository, fixtures, or logs.
20. Core behavior is tested on Windows, Linux, and macOS or has documented platform-specific limitations.
21. All accounting calculations, financially meaningful state transitions, and financial mutations are performed or validated by deterministic application code before commit.
22. Markdown stores the resulting canonical committed financial state, while the accounting engine—not Markdown, Hermes, OCR, an importer, or a cache—determines financial validity and consequences.
23. Field ownership distinguishes descriptive edits from engine-controlled financial fields, and direct edits to calculated fields are rejected, restored, or reviewed rather than blindly trusted.
24. A change to any upstream financial fact causes the engine to recalculate all affected dependent postings, balances, allocations, settlement relationships, reports, and reconciliation state before commit.
25. App-created transactions, loan payments, foreign subscriptions, credit-card payments, transfers, refunds, and imports update canonical Markdown only after engine validation and calculation.
26. Every committed accounting operation writes canonical files, updates derived state, re-reads the result, verifies it, and reports success only after verification.
27. Hermes can interpret intent and request typed operations without independently calculating authoritative balances, FX, loans, liabilities, transfers, refunds, reconciliation, or other accounting consequences.
28. KMyMoney has passed documented prototype gates for the required domains, currencies, deterministic operations, platform needs, round-trip mapping, extension path, and crash recovery before production adoption; failed gates remain explicit blockers or trigger a recorded decision.
29. The Hermes-owned canonical domain and Markdown representation can be reconstructed without depending on KMyMoney's internal schema, IDs, or opaque storage, and the adapter does not create a competing independently maintained accounting engine.
30. The complete accounting app remains usable with Hermes, local models, AI features, and internet access disabled, including account and transaction work, transfers, enabled financial modules, evidence intake/review, reconciliation, search, reports, backup/restore, diagnostics, workbook generation, and canvas.
31. First-launch setup exposes only relevant/enabled feature surfaces; disabling and re-enabling features never deletes records and restores access to retained data.
32. Multiple accounts under one institution remain independent, cards/payment instruments are distinct from accounts, and multiple or replacement cards do not create false accounts. Full IBANs and account/card numbers are not used as filenames or stable IDs.
33. Every enabled financial account has a usable `.xlsx` workbook with actual formulas for applicable calculated values, and formula cells recalculate to results matching deterministic engine results within explicit currency precision.
34. Unlike currencies are never silently combined in account workbooks; original and settlement values, currencies, FX, and fees remain distinguishable.
35. Workbook generation, formula validation, or recalculation failure leaves canonical Markdown/accounting state unchanged and preserves the prior usable workbook; a replacement is staged and validated before safe replacement.
36. Original evidence remains preserved and separate from OCR/extraction output. Low-confidence, impossible, malicious, timed-out, or failed extraction cannot mutate authoritative accounting state without required validation/review.
37. A file sent to Hermes enters the same secure Raw/evidence pipeline, appears in the app review queue when needed, and does not create Hermes-only financial or evidence storage.
38. Canvas account lanes show transactions in the correct account and chronological order. Internal transfer edges connect source and destination effects using one transfer ID and preserve lifecycle, currencies, fees, and accounting neutrality.
39. Canvas balances are derived from validated engine results, agree with engine balances, and distinguish bank-reported observations. Saved layouts and other canvas views cannot mutate canonical financial records.
40. When retained under the user's policy, raw and normalized location facts and their provenance remain separately representable; unknown/online locations are not fabricated, and opt-out applies to structured retention and derived surfaces.
41. Deterministic fault injection across important multi-file and engine-store commit boundaries always recovers to the valid pre-operation state or complete post-operation state, never a half-committed operation.
42. In the defined 1,000 SAR / 200 SAR / 300 SAR transfer crash test, recovery yields only A=1,000 and B=200, or A=700 and B=500; neither one-sided result is possible.

## 44. Open questions and intentionally deferred decisions

The following questions must remain visibly unresolved until evidence is gathered:

- Which supported KMyMoney version, license/compliance path, public API or extension surface, storage mode, and integration mechanism satisfy the prototype gates?
- What fallback or alternative should be evaluated if a hard KMyMoney prototype gate fails?
- Which language and desktop UI technology best support the shared core and three target platforms?
- Should the rules core be an in-process library, local service, or both?
- Which Markdown metadata encoding and parser provide the safest round-trip behavior?
- Which accounting model and double-entry boundary best represent personal finance, liabilities, goals, and conceptual allocations?
- Which KMyMoney-supported operations cannot losslessly represent the Hermes-owned domain, and what adapter-level extensions or explicit unsupported/review states are appropriate?
- Which data must be stored as durable journal events rather than current-state Markdown?
- How should exact decimal precision and currency minor units be represented for all supported currencies?
- What is the final canonical placement for evidence after Raw intake?
- How should generated account and category views be exposed without creating copied transaction records?
- What encryption and secure-deletion capabilities are practical on all target platforms?
- Which OCR and document parsers can be run safely and locally?
- How should model-generated extraction be isolated from deterministic importers?
- What permission model should protect the folder from untrusted local processes?
- What is the minimum command vocabulary that remains expressive without overwhelming a small model?
- Which operations should require preview, confirmation, or human review?
- How should backup, restore, merge, and multi-device folder synchronization work?
- Which spreadsheet generation and calculation runtimes provide reliable cross-platform formula recalculation without making workbooks authoritative?
- What location-retention defaults and exact privacy controls should apply to source extraction, structured metadata, derived views, and backups?
- What investment features are required, if any, and what accounting or market-data dependencies do they introduce?
- What reports and conversion policies are necessary for multi-currency totals?
- How should tax-related fields be represented without implying tax advice or jurisdictional correctness?
- Which platform-specific atomicity and watcher guarantees can be relied upon?

These are decision points, not invitations to guess. Future implementation work should update this document or an explicitly linked decision record when a question is answered.

## 45. Governance for future changes

The master plan should remain the authoritative high-level context until it is superseded by a reviewed revision. Future changes that alter canonical data, accounting semantics, privacy posture, upstream selection, or Hermes mutation authority should include:

- the decision and its scope;
- evidence or prototype results;
- affected invariants and schemas;
- migration and recovery impact;
- testing impact;
- unresolved risks;
- explicit approval from the project owner.

Implementation documents may add detail, but they must not quietly contradict hard requirements in this plan. If an implementation discovers that a requirement is infeasible, the conflict must be documented and resolved deliberately.

## Closing principle

If something financially plausible happens in real life, Accounting and Money should either represent it correctly or explicitly mark it as ambiguous and requiring review. It must never quietly guess and corrupt the ledger.

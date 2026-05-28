# Medical Billing & Coding Assistant — Development Plan

> Project: 103-medical-billing-coding-assistant
> Created: 2026-05-25
> Status: Planning

---

## Technology Decisions

### Data Model Selection: Hybrid Relational + JSONB (Model 3) with Event Sourcing Elements (Model 2)

**Decision:** Adopt Data Model Suggestion 3 (Hybrid Relational + JSONB) as the primary schema, augmented with an append-only `events` table from Data Model Suggestion 2 for audit trail and AI model training replay.

**Rationale:**

1. **MVP velocity.** The hybrid model has 16 tables versus 29 in the fully normalised model. Fewer migrations, simpler ORM mappings, and faster iteration during the product's formative phases when payer rules, specialty configurations, and AI model outputs are changing constantly.
2. **Payer variability.** Each payer has unique LCD/NCD rules, modifier requirements, bundling policies, and timely filing windows. JSONB `rule_definition` and `payer_config` columns absorb this variability without DDL changes. The fully normalised model would require either nullable columns or additional tables for every payer-specific variation.
3. **AI output flexibility.** The AI coding engine's output structure (reasoning chains, alternative codes, supporting evidence, E/M calculations) will evolve rapidly across model versions. Storing `ai_coding_result` and `ai_details` as JSONB avoids schema migrations on every AI model change.
4. **Audit trail via event store.** Medical billing requires HIPAA-compliant 6-year audit retention. An append-only `events` table (from Model 2) provides immutable, tamper-evident audit history and enables temporal queries ("what codes were assigned before the CDI query?") and AI model retraining from historical coding decisions. This avoids the full CQRS complexity of Model 2 while capturing its audit benefits.
5. **Graph layer deferred.** Data Model Suggestion 4's graph layer is powerful for denial pattern mining and code relationship analysis but adds infrastructure complexity (Apache AGE or Neo4j) inappropriate for an MVP. The JSONB payer rules and GIN indexes in Model 3 cover the critical payer rule queries for v1. The graph layer can be introduced in Phase 10+ when denial volume and pattern complexity justify the operational overhead.

### Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Database** | PostgreSQL 16+ | JSONB support, GIN indexing, partitioning for audit tables, mature healthcare ecosystem |
| **Backend API** | Node.js (TypeScript) with Fastify | Type safety for complex medical code structures; Fastify's schema validation aligns with strict billing data requirements; strong FHIR/HL7 library ecosystem (fhir.js, node-hl7-complete) |
| **AI/NLP Engine** | Python (FastAPI microservice) | LLM integration (Claude API / local models), clinical NLP libraries (spaCy, medspaCy, scispaCy), SNOMED CT concept normalization |
| **Frontend** | React + TypeScript (Next.js) | Component library for coder workstation; server-side rendering for compliance dashboards; accessible UI for healthcare workers |
| **EDI Processing** | Stedi SDK (TypeScript) | API-first clearinghouse with JSON-to-X12 conversion; developer-friendly alternative to legacy EDI infrastructure; supports 837, 835, 270/271, 276/277 |
| **Terminology Server** | Snowstorm (FHIR) — self-hosted | Open-source FHIR Terminology Server for SNOMED CT, LOINC, ICD-10-CM concept normalization in the NLP pipeline |
| **Authentication** | SMART on FHIR (OAuth 2.0 + PKCE) | Healthcare-standard auth for EHR integration; required for Epic/Oracle/Meditech app launch |
| **Message Queue** | Redis Streams or BullMQ | Async processing for AI coding jobs, EDI file parsing, and denial pattern analysis |
| **Object Storage** | S3-compatible (MinIO for self-hosted) | Clinical document storage, EDI file archival, appeal package documents |
| **Containerisation** | Docker + Kubernetes | Healthcare deployment requires self-hosted and cloud options; K8s supports both |
| **CI/CD** | GitHub Actions | Standard for open-source projects; HIPAA-compliant with self-hosted runners |

### Project Structure

```
medical-billing-coding-assistant/
├── packages/
│   ├── api/                          # Fastify backend API (TypeScript)
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── auth/             # SMART on FHIR, OAuth, RBAC
│   │   │   │   ├── organisations/    # Org and provider management
│   │   │   │   ├── patients/         # Patient demographics and insurance
│   │   │   │   ├── encounters/       # Encounter lifecycle and clinical docs
│   │   │   │   ├── coding/           # Code assignment, E/M calculator, DRG grouper
│   │   │   │   ├── claims/           # Claim generation, scrubbing, submission
│   │   │   │   ├── remittance/       # ERA 835 parsing and payment reconciliation
│   │   │   │   ├── denials/          # Denial tracking, analytics, appeals
│   │   │   │   ├── payer-rules/      # Payer rule management (manual + AI-learned)
│   │   │   │   ├── cdi/              # Clinical documentation improvement queries
│   │   │   │   ├── audit/            # Audit trail and compliance reporting
│   │   │   │   └── analytics/        # Dashboards, coder productivity, KPIs
│   │   │   ├── integrations/
│   │   │   │   ├── fhir/             # FHIR R4 client for EHR connectivity
│   │   │   │   ├── hl7v2/            # HL7 v2.x ADT and document feeds
│   │   │   │   ├── edi/              # X12 837/835/270/271/276/277 via Stedi
│   │   │   │   └── clearinghouse/    # Clearinghouse abstraction layer
│   │   │   ├── db/
│   │   │   │   ├── migrations/       # PostgreSQL schema migrations (node-pg-migrate)
│   │   │   │   ├── seeds/            # Reference data seeds (ICD-10, CPT stubs, NCCI)
│   │   │   │   └── models/           # Database access layer
│   │   │   └── shared/
│   │   │       ├── validators/       # Zod schemas for medical data validation
│   │   │       ├── constants/        # Code systems, status enums, CARC/RARC codes
│   │   │       └── utils/            # Shared utilities
│   │   └── tests/
│   ├── ai-engine/                    # Python AI/NLP microservice
│   │   ├── src/
│   │   │   ├── coding/              # AI code suggestion from clinical notes
│   │   │   ├── cdi/                 # CDI alert generation
│   │   │   ├── denials/             # Denial root-cause analysis and pattern detection
│   │   │   ├── appeals/             # Appeal letter generation
│   │   │   ├── nlp/                 # Clinical NLP pipeline (entity extraction, normalization)
│   │   │   ├── models/              # LLM integration and prompt engineering
│   │   │   └── terminology/         # SNOMED/LOINC/RxNorm concept normalization
│   │   └── tests/
│   ├── web/                          # Next.js frontend
│   │   ├── src/
│   │   │   ├── app/                 # Next.js app router pages
│   │   │   ├── components/
│   │   │   │   ├── coding/          # Coder workstation components
│   │   │   │   ├── claims/          # Claims dashboard and scrub results
│   │   │   │   ├── denials/         # Denial management UI
│   │   │   │   ├── analytics/       # Charts, KPI widgets, dashboards
│   │   │   │   └── shared/          # Common UI components
│   │   │   └── lib/                 # API client, hooks, utilities
│   │   └── tests/
│   ├── reference-data/               # Code set import tools and data
│   │   ├── importers/               # Scripts to import ICD-10, HCPCS, NCCI from CMS files
│   │   ├── schemas/                 # Validation schemas for reference data
│   │   └── data/                    # Sample/test data (no proprietary CPT data committed)
│   └── shared/                       # Shared TypeScript types and utilities
│       ├── types/                   # Domain types (encounters, codes, claims)
│       └── constants/               # Shared constants across packages
├── deploy/
│   ├── docker/                      # Dockerfiles for each service
│   ├── k8s/                         # Kubernetes manifests
│   └── terraform/                   # Infrastructure as code
├── docs/
│   ├── architecture/               # Architecture decision records (ADRs)
│   ├── api/                        # OpenAPI specs
│   └── compliance/                 # HIPAA compliance documentation
└── scripts/                         # Development and operational scripts
```

---

## Phase Dependency Graph

```
Phase 1: Foundation & Reference Data
    │
    ├──► Phase 2: Patient & Encounter Management
    │        │
    │        ├──► Phase 3: AI Coding Engine (MVP)
    │        │        │
    │        │        ├──► Phase 5: Claim Generation & Scrubbing
    │        │        │        │
    │        │        │        ├──► Phase 6: EDI & Clearinghouse Integration
    │        │        │        │        │
    │        │        │        │        ├──► Phase 7: Remittance & Denial Management
    │        │        │        │        │        │
    │        │        │        │        │        ├──► Phase 8: Denial Learning & Payer Rule AI
    │        │        │        │        │        │        │
    │        │        │        │        │        │        └──► Phase 10: Autonomous Coding & Scale
    │        │        │        │        │        │
    │        │        │        │        │        └──► Phase 9: Analytics & Compliance Dashboards
    │        │        │        │        │
    │        │        │        │        └──► Phase 11: EHR Integration & CDI at Point of Care
    │        │        │        │
    │        │        │        └──► Phase 4: Coder Workstation UI
    │        │        │
    │        │        └──► Phase 4: Coder Workstation UI
    │        │
    │        └──► Phase 4: Coder Workstation UI
    │
    └──► Phase 3: AI Coding Engine (MVP)
```

**Critical path:** Phase 1 → Phase 2 → Phase 3 → Phase 5 → Phase 6 → Phase 7 → Phase 8 → Phase 10

**Parallel tracks after Phase 2:**
- Phase 4 (UI) can begin as soon as Phase 2 provides API endpoints for encounters and code assignments
- Phase 3 (AI Engine) can begin in parallel with Phase 2 using mock clinical note data

---

## Phase Definitions

---

### Phase 1: Foundation, Authentication & Reference Data

**Goal:** Establish the database schema, authentication system, reference data pipeline, and API skeleton. Every subsequent phase depends on this foundation.

**Duration estimate:** 4-5 weeks

#### Task 1.1: Database Schema and Migration System

**What:** Create the PostgreSQL database schema using the Hybrid Relational + JSONB model (16 core tables + events table for audit). Set up the migration framework and seed scripts.

**Design:**
```typescript
// packages/api/src/db/migrations/001_initial_schema.ts
// Creates all 17 tables from the hybrid model:
// - organisations, providers, users
// - patients (insurance as JSONB)
// - encounters, clinical_documents
// - code_assignments (ai_details, review_details as JSONB)
// - payers (payer_config as JSONB), claims (claim_lines, claim_diagnoses as JSONB)
// - remittances (line_items as JSONB), denials (denial_details, appeals as JSONB)
// - payer_rules (rule_definition as JSONB)
// - cdi_queries (query_details as JSONB)
// - code_sets, ncci_edits
// - audit_log (changes as JSONB)
// - events (event store for immutable audit trail)

import { MigrationBuilder } from 'node-pg-migrate';

export async function up(pgm: MigrationBuilder): Promise<void> {
  pgm.createExtension('uuid-ossp', { ifNotExists: true });
  pgm.createExtension('pgcrypto', { ifNotExists: true });

  // organisations table
  pgm.createTable('organisations', {
    id: { type: 'uuid', primaryKey: true, default: pgm.func('gen_random_uuid()') },
    name: { type: 'varchar(255)', notNull: true },
    npi: { type: 'varchar(10)', unique: true },
    tax_id: { type: 'varchar(11)' },
    org_type: { type: 'varchar(50)', notNull: true },
    contact_info: { type: 'jsonb', notNull: true, default: "'{}'" },
    settings: { type: 'jsonb', notNull: true, default: "'{}'" },
    created_at: { type: 'timestamptz', notNull: true, default: pgm.func('now()') },
    updated_at: { type: 'timestamptz', notNull: true, default: pgm.func('now()') },
  });

  // events table (append-only audit/event store)
  pgm.createTable('events', {
    event_id: { type: 'uuid', primaryKey: true, default: pgm.func('gen_random_uuid()') },
    stream_id: { type: 'uuid', notNull: true },
    stream_type: { type: 'varchar(50)', notNull: true },
    event_type: { type: 'varchar(100)', notNull: true },
    event_version: { type: 'integer', notNull: true },
    organisation_id: { type: 'uuid', notNull: true },
    caused_by: { type: 'uuid' },
    correlation_id: { type: 'uuid' },
    event_data: { type: 'jsonb', notNull: true },
    metadata: { type: 'jsonb' },
    created_at: { type: 'timestamptz', notNull: true, default: pgm.func('now()') },
  });
  pgm.addConstraint('events', 'events_stream_version_unique', {
    unique: ['stream_id', 'event_version'],
  });
  // ... remaining tables follow the same pattern
}
```

**Testing:**
- **T1.1.1:** Run `up` migration against an empty PostgreSQL 16 database; verify all 17 tables are created with correct columns, types, constraints, and indexes.
- **T1.1.2:** Run `down` migration; verify all tables are dropped cleanly.
- **T1.1.3:** Insert a row into every table with valid data; verify foreign key constraints enforce referential integrity (e.g., inserting a provider with a non-existent `organisation_id` fails).
- **T1.1.4:** Insert a row with JSONB fields containing valid example payloads from the data model specification; verify GIN indexes are created and `@>` containment queries work.
- **T1.1.5:** Verify the `events` table's unique constraint on `(stream_id, event_version)` rejects duplicate event versions for the same stream.

---

#### Task 1.2: Authentication and RBAC

**What:** Implement user authentication with password hashing (bcrypt), JWT token issuance, role-based access control (RBAC) for the five user roles (coder, cdi_specialist, billing_manager, compliance_officer, admin), and the SMART on FHIR OAuth 2.0 flow skeleton for future EHR integration.

**Design:**
```typescript
// packages/api/src/modules/auth/auth.service.ts
export class AuthService {
  async register(dto: RegisterDto): Promise<User> {
    const passwordHash = await bcrypt.hash(dto.password, 12);
    return this.userRepo.create({ ...dto, passwordHash });
  }

  async login(email: string, password: string): Promise<{ accessToken: string; refreshToken: string }> {
    const user = await this.userRepo.findByEmail(email);
    if (!user || !(await bcrypt.compare(password, user.password_hash))) {
      throw new UnauthorizedError('Invalid credentials');
    }
    const accessToken = jwt.sign(
      { sub: user.id, org: user.organisation_id, role: user.role },
      config.jwtSecret,
      { expiresIn: '1h' }
    );
    const refreshToken = jwt.sign({ sub: user.id }, config.jwtRefreshSecret, { expiresIn: '7d' });
    await this.eventStore.append(user.id, 'user', 'UserLoggedIn', { ip: ctx.ip });
    return { accessToken, refreshToken };
  }
}

// packages/api/src/modules/auth/rbac.guard.ts
const ROLE_PERMISSIONS: Record<string, string[]> = {
  admin:              ['*'],
  billing_manager:    ['code_encounters', 'submit_claims', 'view_denials', 'manage_payer_rules', 'view_analytics'],
  coder:              ['code_encounters', 'view_denials'],
  cdi_specialist:     ['code_encounters', 'manage_cdi_queries', 'view_analytics'],
  compliance_officer: ['audit_review', 'view_analytics', 'view_denials', 'compliance_query'],
};

export function requirePermission(permission: string) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const { role } = request.user;
    const perms = ROLE_PERMISSIONS[role] ?? [];
    if (!perms.includes('*') && !perms.includes(permission)) {
      throw new ForbiddenError(`Role '${role}' lacks permission '${permission}'`);
    }
  };
}
```

**Testing:**
- **T1.2.1:** Register a new user with valid data; verify password is stored as bcrypt hash (not plaintext) and the user row is created in the database.
- **T1.2.2:** Login with valid credentials; verify JWT access token contains correct `sub`, `org`, and `role` claims; verify refresh token is issued.
- **T1.2.3:** Login with invalid password; verify 401 response with no token.
- **T1.2.4:** Access a protected endpoint with a valid token for a `coder` role requesting `code_encounters` permission; verify 200 response.
- **T1.2.5:** Access a protected endpoint with a valid token for a `coder` role requesting `submit_claims` permission; verify 403 Forbidden response.
- **T1.2.6:** Access a protected endpoint with an expired token; verify 401 response.
- **T1.2.7:** Verify all RBAC permission mappings by testing each role against each permission.
- **T1.2.8:** Verify the `UserLoggedIn` event is appended to the events table on successful login.

---

#### Task 1.3: Reference Data Import Pipeline

**What:** Build importers for ICD-10-CM, ICD-10-PCS, HCPCS Level II, and NCCI edits from CMS flat files. Build a stub importer for CPT codes (cannot include proprietary CPT data without AMA licence; importer validates format and supports customer-provided CPT files). Load AHRQ HCUP CCS crosswalk data (public domain).

**Design:**
```typescript
// packages/reference-data/importers/icd10cm-importer.ts
export class ICD10CMImporter {
  /**
   * Imports ICD-10-CM codes from CMS flat file (icd10cm_tabular_FYXXXX.txt)
   * Source: https://www.cdc.gov/nchs/icd/icd-10-cm/files.html
   * File format: fixed-width text with code, short description, long description
   */
  async import(filePath: string, fiscalYear: number): Promise<ImportResult> {
    const codes = await this.parseFile(filePath);
    const result = await this.db.transaction(async (tx) => {
      // Deactivate previous fiscal year codes
      await tx.query(
        `UPDATE code_sets SET is_active = false
         WHERE code_system = 'ICD10CM' AND fiscal_year = $1`,
        [fiscalYear - 1]
      );
      // Upsert new codes
      for (const code of codes) {
        await tx.query(
          `INSERT INTO code_sets (code_system, code, description, effective_date, is_active, code_details)
           VALUES ('ICD10CM', $1, $2, $3, true, $4)
           ON CONFLICT (code_system, code, effective_date) DO UPDATE
           SET description = EXCLUDED.description, is_active = true, code_details = EXCLUDED.code_details`,
          [code.code, code.shortDesc, `${fiscalYear}-10-01`, JSON.stringify({
            long_description: code.longDesc,
            chapter: code.chapter,
            block: code.block,
            fiscal_year: fiscalYear,
          })]
        );
      }
      return { imported: codes.length };
    });
    return result;
  }
}

// packages/reference-data/importers/ncci-importer.ts
export class NCCIImporter {
  /**
   * Imports NCCI PTP edits and MUEs from quarterly CMS Excel/CSV files.
   * Source: https://www.cms.gov/medicare/coding-billing/national-correct-coding-initiative-ncci-edits
   */
  async importPTPEdits(filePath: string, quarter: string): Promise<ImportResult> {
    const edits = await this.parseCSV(filePath);
    // Bulk upsert into ncci_edits table
    // ...
  }
}
```

**Testing:**
- **T1.3.1:** Import a sample ICD-10-CM flat file (FY2026, subset of 500 codes); verify all 500 codes are inserted into `code_sets` with `code_system = 'ICD10CM'`, correct descriptions, and `is_active = true`.
- **T1.3.2:** Re-import the same file; verify no duplicate rows (upsert behavior).
- **T1.3.3:** Import FY2027 file with 10 new codes and 3 deleted codes; verify new codes are active, deleted codes from FY2026 are deactivated.
- **T1.3.4:** Import NCCI PTP edits for 2026Q2 (subset of 1000 edit pairs); verify all pairs are stored with correct `modifier_indicator` and `effective_date`.
- **T1.3.5:** Verify full-text search on `code_sets.description` returns correct ICD-10-CM codes for "type 2 diabetes" (should include E11.* codes).
- **T1.3.6:** Import HCPCS Level II file; verify codes with `code_system = 'HCPCS'` are created.
- **T1.3.7:** Attempt to import a malformed file (missing columns, invalid code formats); verify importer rejects with descriptive error and rolls back the transaction.

---

#### Task 1.4: Event Store Service

**What:** Build the append-only event store service that records all domain events for audit trail and AI model training replay. Implements optimistic concurrency control via the `(stream_id, event_version)` unique constraint.

**Design:**
```typescript
// packages/api/src/modules/audit/event-store.service.ts
export class EventStoreService {
  async append(
    streamId: string,
    streamType: string,
    eventType: string,
    eventData: Record<string, unknown>,
    options?: { causedBy?: string; correlationId?: string; metadata?: Record<string, unknown> }
  ): Promise<Event> {
    // Get next version for this stream
    const { rows } = await this.db.query(
      'SELECT COALESCE(MAX(event_version), 0) + 1 AS next_version FROM events WHERE stream_id = $1',
      [streamId]
    );
    const nextVersion = rows[0].next_version;

    const event = await this.db.query(
      `INSERT INTO events (stream_id, stream_type, event_type, event_version, organisation_id, caused_by, correlation_id, event_data, metadata)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
       RETURNING *`,
      [streamId, streamType, eventType, nextVersion, options?.organisationId, options?.causedBy, options?.correlationId, eventData, options?.metadata]
    );
    return event.rows[0];
  }

  async getStream(streamId: string, fromVersion?: number): Promise<Event[]> {
    return this.db.query(
      'SELECT * FROM events WHERE stream_id = $1 AND event_version >= $2 ORDER BY event_version',
      [streamId, fromVersion ?? 1]
    ).then(r => r.rows);
  }

  async queryByType(eventType: string, since: Date, orgId: string): Promise<Event[]> {
    return this.db.query(
      'SELECT * FROM events WHERE event_type = $1 AND created_at >= $2 AND organisation_id = $3 ORDER BY created_at',
      [eventType, since, orgId]
    ).then(r => r.rows);
  }
}
```

**Testing:**
- **T1.4.1:** Append 3 events to the same stream; verify event versions are 1, 2, 3 and all have correct `stream_id` and `stream_type`.
- **T1.4.2:** Attempt to append an event with a duplicate `(stream_id, event_version)`; verify the unique constraint rejects the insert.
- **T1.4.3:** Query a stream by `stream_id`; verify events are returned in version order.
- **T1.4.4:** Query events by type and date range; verify correct filtering.
- **T1.4.5:** Append an event with `correlation_id` and `caused_by`; verify these fields are persisted and queryable.
- **T1.4.6:** Verify events are immutable — attempt to UPDATE an event row; verify the application layer rejects this (no update endpoint exposed).

---

#### Task 1.5: API Skeleton and Health Checks

**What:** Set up the Fastify API server with route registration, request validation (Zod), error handling, logging, CORS configuration, and health check endpoints. Establish the module-based route registration pattern that all subsequent phases will follow.

**Design:**
```typescript
// packages/api/src/server.ts
import Fastify from 'fastify';

const app = Fastify({ logger: true });

// Register plugins
app.register(require('@fastify/cors'), { origin: config.corsOrigins });
app.register(require('@fastify/jwt'), { secret: config.jwtSecret });

// Register modules
app.register(authRoutes, { prefix: '/api/v1/auth' });
app.register(orgRoutes, { prefix: '/api/v1/organisations' });
// ... subsequent modules registered as they are built

// Health check
app.get('/health', async () => ({
  status: 'ok',
  version: process.env.npm_package_version,
  database: await db.isHealthy() ? 'connected' : 'disconnected',
  timestamp: new Date().toISOString(),
}));

app.listen({ port: config.port, host: '0.0.0.0' });
```

**Testing:**
- **T1.5.1:** Start the server; verify `GET /health` returns `{ status: 'ok', database: 'connected' }` with HTTP 200.
- **T1.5.2:** Send a request with invalid JSON body to a POST endpoint; verify 400 response with descriptive validation error.
- **T1.5.3:** Send a request without Authorization header to a protected endpoint; verify 401 response.
- **T1.5.4:** Verify CORS headers are present on responses.
- **T1.5.5:** Verify structured JSON logging includes request ID, method, path, status code, and response time.

---

### Definition of Done — Phase 1
- [ ] All 17 database tables created and validated with migration up/down
- [ ] User registration, login, JWT issuance, and RBAC enforcement working
- [ ] ICD-10-CM, ICD-10-PCS, HCPCS, and NCCI edit importers functional with CMS flat files
- [ ] Event store service appending and querying events correctly
- [ ] API server running with health check, structured logging, and error handling
- [ ] Test suite passing with >90% coverage on auth and reference data modules
- [ ] Docker Compose file for local development (PostgreSQL, API, Redis)

---

### Phase 2: Patient & Encounter Management

**Goal:** Build the patient registry, insurance management, encounter lifecycle, and clinical document ingestion. This phase creates the core clinical data that the AI coding engine (Phase 3) will process.

**Duration estimate:** 3-4 weeks

#### Task 2.1: Organisation and Provider CRUD

**What:** REST API endpoints for creating, reading, updating, and listing organisations and providers. Providers are scoped to an organisation and identified by NPI.

**Design:**
```typescript
// packages/api/src/modules/organisations/organisations.routes.ts
export async function orgRoutes(app: FastifyInstance) {
  app.post('/', { preHandler: [requirePermission('manage_users')], schema: createOrgSchema }, async (req) => {
    const org = await orgService.create(req.body);
    await eventStore.append(org.id, 'organisation', 'OrganisationCreated', req.body, { causedBy: req.user.sub });
    return org;
  });

  app.get('/:id', { schema: getOrgSchema }, async (req) => {
    return orgService.findById(req.params.id);
  });

  // POST /organisations/:orgId/providers
  app.post('/:orgId/providers', { preHandler: [requirePermission('manage_users')], schema: createProviderSchema }, async (req) => {
    const provider = await providerService.create(req.params.orgId, req.body);
    await eventStore.append(provider.id, 'provider', 'ProviderCreated', req.body);
    return provider;
  });
}
```

**Testing:**
- **T2.1.1:** Create an organisation with valid NPI; verify 201 response and row in database.
- **T2.1.2:** Create a provider linked to an organisation; verify foreign key to `organisations` is set and NPI is stored.
- **T2.1.3:** Create a provider with a duplicate NPI; verify 409 Conflict response.
- **T2.1.4:** List providers for an organisation; verify pagination and filtering by specialty.
- **T2.1.5:** Update a provider's specialty_code; verify the update is persisted and an event is appended.
- **T2.1.6:** Attempt to create a provider for a non-existent organisation; verify 404 response.

---

#### Task 2.2: Patient Registry and Insurance

**What:** CRUD endpoints for patients with embedded insurance coverage as JSONB. Patients are scoped to an organisation and identified by MRN. Insurance data supports primary/secondary/tertiary coverage with payer references.

**Design:**
```typescript
// packages/api/src/modules/patients/patients.service.ts
export class PatientService {
  async create(orgId: string, dto: CreatePatientDto): Promise<Patient> {
    return this.db.query(
      `INSERT INTO patients (organisation_id, mrn, first_name, last_name, date_of_birth, sex, demographics, insurance)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
       RETURNING *`,
      [orgId, dto.mrn, dto.firstName, dto.lastName, dto.dateOfBirth, dto.sex,
       JSON.stringify(dto.demographics ?? {}), JSON.stringify(dto.insurance ?? [])]
    ).then(r => r.rows[0]);
  }

  async addInsurance(patientId: string, coverage: InsuranceCoverage): Promise<Patient> {
    // Append to the insurance JSONB array
    return this.db.query(
      `UPDATE patients SET insurance = insurance || $1::jsonb, updated_at = now()
       WHERE id = $2 RETURNING *`,
      [JSON.stringify([coverage]), patientId]
    ).then(r => r.rows[0]);
  }

  async verifyEligibility(patientId: string, payerId: string): Promise<EligibilityResult> {
    // Stub for Phase 6 — will call X12 270/271 via clearinghouse
    throw new NotImplementedError('Eligibility verification available in Phase 6');
  }
}
```

**Testing:**
- **T2.2.1:** Create a patient with MRN, demographics, and primary insurance; verify the patient row is created with correct JSONB `insurance` array.
- **T2.2.2:** Add secondary insurance to an existing patient; verify the `insurance` JSONB array now contains 2 entries with priorities 1 and 2.
- **T2.2.3:** Create a patient with duplicate MRN within the same organisation; verify unique constraint violation (409 Conflict).
- **T2.2.4:** Create patients with the same MRN in different organisations; verify both succeed (MRN is org-scoped).
- **T2.2.5:** Search patients by date of birth range; verify correct results.
- **T2.2.6:** Verify PHI fields (SSN, address) are only returned to users with appropriate permissions.

---

#### Task 2.3: Encounter Lifecycle

**What:** Encounter creation, status management, and the encounter workflow state machine (open → coded → billed → closed). Encounters link patients to providers and track clinical context (encounter type, place of service, financial class).

**Design:**
```typescript
// packages/api/src/modules/encounters/encounters.service.ts
const VALID_TRANSITIONS: Record<string, string[]> = {
  open:        ['ai_processing', 'coded', 'closed'],
  ai_processing: ['ai_suggested', 'coded', 'open'],
  ai_suggested: ['coder_review', 'coded', 'open'],
  coder_review: ['coded', 'open'],
  coded:       ['billed', 'open'],  // Can reopen for re-coding
  billed:      ['closed'],
  closed:      [],
};

export class EncounterService {
  async create(orgId: string, dto: CreateEncounterDto): Promise<Encounter> {
    const encounter = await this.db.query(
      `INSERT INTO encounters (organisation_id, patient_id, provider_id, encounter_type,
         admit_date, place_of_service, financial_class, status, coding_status, encounter_details)
       VALUES ($1,$2,$3,$4,$5,$6,$7,'open','pending',$8)
       RETURNING *`,
      [orgId, dto.patientId, dto.providerId, dto.encounterType,
       dto.admitDate, dto.placeOfService, dto.financialClass,
       JSON.stringify(dto.encounterDetails ?? {})]
    ).then(r => r.rows[0]);
    await this.eventStore.append(encounter.id, 'encounter', 'EncounterCreated', dto);
    return encounter;
  }

  async transitionCodingStatus(encounterId: string, newStatus: string, userId: string): Promise<Encounter> {
    const encounter = await this.findById(encounterId);
    const allowed = VALID_TRANSITIONS[encounter.coding_status];
    if (!allowed?.includes(newStatus)) {
      throw new BadRequestError(
        `Cannot transition from '${encounter.coding_status}' to '${newStatus}'`
      );
    }
    // ... update and append event
  }
}
```

**Testing:**
- **T2.3.1:** Create an encounter; verify status is `open` and coding_status is `pending`.
- **T2.3.2:** Transition coding_status from `pending` (which maps to `open`) through the full lifecycle: open → ai_processing → ai_suggested → coder_review → coded → billed → closed; verify each transition succeeds and events are recorded.
- **T2.3.3:** Attempt an invalid transition (e.g., `open` → `billed`); verify 400 Bad Request with descriptive error.
- **T2.3.4:** List encounters filtered by status, provider, and date range; verify correct results and pagination.
- **T2.3.5:** Create an encounter with `encounter_details` JSONB containing chief complaint and acuity; verify JSONB is stored and retrievable.
- **T2.3.6:** Verify that transitioning to `coded` requires at least one `code_assignment` linked to the encounter (business rule validation).

---

#### Task 2.4: Clinical Document Ingestion

**What:** API endpoints for uploading and associating clinical documents (progress notes, discharge summaries, operative notes) with encounters. Documents are stored as plain text (extracted from CDA/CCD if needed) to support NLP processing in Phase 3.

**Design:**
```typescript
// packages/api/src/modules/encounters/documents.routes.ts
app.post('/encounters/:encounterId/documents', {
  preHandler: [requirePermission('code_encounters')],
  schema: createDocumentSchema,
}, async (req) => {
  const doc = await documentService.create(req.params.encounterId, {
    documentType: req.body.documentType,
    authorId: req.body.authorId,
    documentDate: req.body.documentDate,
    contentText: req.body.contentText,
    sourceInfo: req.body.sourceInfo ?? {},
  });
  await eventStore.append(
    req.params.encounterId, 'encounter', 'ClinicalDocumentReceived',
    { documentId: doc.id, documentType: doc.document_type, wordCount: doc.content_text.length }
  );
  return doc;
});
```

**Testing:**
- **T2.4.1:** Upload a clinical document (progress note) to an encounter; verify the document row is created with correct `encounter_id`, `document_type`, and `content_text`.
- **T2.4.2:** Upload multiple documents to the same encounter; verify all are associated and listable.
- **T2.4.3:** Upload a document with `source_info` JSONB containing FHIR DocumentReference ID and source system; verify JSONB is stored.
- **T2.4.4:** Attempt to upload a document to a non-existent encounter; verify 404 response.
- **T2.4.5:** Verify the `ClinicalDocumentReceived` event is appended with document metadata.
- **T2.4.6:** Retrieve all documents for an encounter, ordered by document_date; verify correct ordering and complete content_text returned.

---

### Definition of Done — Phase 2
- [ ] Organisation, provider, patient, and encounter CRUD endpoints functional
- [ ] Patient insurance stored as JSONB with add/update/remove support
- [ ] Encounter state machine enforcing valid coding_status transitions
- [ ] Clinical document upload and association with encounters working
- [ ] Events appended for all entity lifecycle operations
- [ ] API documentation (OpenAPI spec) generated for all Phase 2 endpoints
- [ ] Integration tests covering full encounter lifecycle (create → upload docs → code → bill → close)

---

### Phase 3: AI Coding Engine (MVP)

**Goal:** Build the core AI/NLP microservice that reads clinical notes and suggests ICD-10-CM and CPT codes with confidence scores and reasoning chains. This is the product's central value proposition.

**Duration estimate:** 5-6 weeks

#### Task 3.1: Clinical NLP Pipeline

**What:** Build the Python NLP pipeline that extracts clinical entities (diagnoses, procedures, medications, lab results) from clinical note text using spaCy/medspaCy and normalizes them to SNOMED CT concepts via the Snowstorm FHIR Terminology Server.

**Design:**
```python
# packages/ai-engine/src/nlp/clinical_pipeline.py
import spacy
from medspacy.ner import MedspaCyNER

class ClinicalNLPPipeline:
    def __init__(self, snowstorm_url: str):
        self.nlp = spacy.load("en_core_sci_lg")
        self.nlp.add_pipe("medspacy_ner")
        self.snowstorm = SnowstormClient(snowstorm_url)

    def extract_entities(self, clinical_text: str) -> list[ClinicalEntity]:
        """Extract and normalize clinical entities from free-text clinical notes."""
        doc = self.nlp(clinical_text)
        entities = []
        for ent in doc.ents:
            snomed_concept = self.snowstorm.normalize(ent.text, ent.label_)
            entities.append(ClinicalEntity(
                text=ent.text,
                label=ent.label_,  # PROBLEM, PROCEDURE, MEDICATION, LAB, etc.
                start=ent.start_char,
                end=ent.end_char,
                snomed_code=snomed_concept.code if snomed_concept else None,
                snomed_display=snomed_concept.display if snomed_concept else None,
                negated=ent._.is_negated,
                context=ent._.section_category,  # ASSESSMENT, PLAN, HPI, etc.
            ))
        return entities
```

**Testing:**
- **T3.1.1:** Process a sample ED progress note; verify entities are extracted for "chest pain" (PROBLEM), "EKG" (PROCEDURE), "troponin" (LAB).
- **T3.1.2:** Process a note containing negated findings ("no evidence of pneumonia"); verify the negation flag is set on the pneumonia entity.
- **T3.1.3:** Verify SNOMED CT normalization: "heart attack" should map to SNOMED CT concept 22298006 (Myocardial infarction).
- **T3.1.4:** Process a 5-page discharge summary; verify processing completes within 10 seconds with >20 entities extracted.
- **T3.1.5:** Process a note with abbreviations ("SOB", "CP", "HTN"); verify abbreviations are expanded or mapped to correct concepts.
- **T3.1.6:** Verify section detection: entities in "Assessment" section are tagged differently from entities in "Medications" section.

---

#### Task 3.2: AI Code Suggestion Engine

**What:** Build the LLM-based code suggestion engine that takes extracted clinical entities and the original note text, then produces ICD-10-CM diagnosis codes and CPT procedure codes with confidence scores and reasoning chains.

**Design:**
```python
# packages/ai-engine/src/coding/code_suggester.py
from anthropic import Anthropic

class AICodingSuggester:
    def __init__(self, anthropic_client: Anthropic, code_db: CodeDatabase):
        self.client = anthropic_client
        self.code_db = code_db

    async def suggest_codes(
        self,
        clinical_text: str,
        entities: list[ClinicalEntity],
        encounter_type: str,
        place_of_service: str,
    ) -> CodingSuggestionResult:
        """Generate ICD-10-CM and CPT code suggestions with reasoning chains."""

        # Build prompt with clinical context, extracted entities, and coding guidelines
        prompt = self._build_coding_prompt(clinical_text, entities, encounter_type)

        response = await self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            system=MEDICAL_CODING_SYSTEM_PROMPT,  # Includes coding guidelines and output format
            messages=[{"role": "user", "content": prompt}],
        )

        # Parse structured output into code suggestions
        suggestions = self._parse_coding_response(response)

        # Validate each suggested code exists in our code_sets database
        validated = []
        for s in suggestions:
            code_record = self.code_db.lookup(s.code_system, s.code)
            if code_record and code_record.is_active:
                validated.append(s)
            else:
                # Log invalid code suggestion for model monitoring
                logger.warning(f"AI suggested invalid code: {s.code_system}/{s.code}")

        return CodingSuggestionResult(
            model_version="medbill-llm-v1.0.0",
            suggestions=validated,
            overall_confidence=self._calculate_overall_confidence(validated),
            processing_time_ms=response.usage.input_tokens,  # actual timing
        )

    def _build_coding_prompt(self, text: str, entities: list, encounter_type: str) -> str:
        return f"""You are a certified medical coder (CPC, CCS). Given the clinical note below,
assign ICD-10-CM diagnosis codes and CPT procedure codes.

For each code, provide:
1. Code system (ICD10CM or CPT)
2. Code value
3. Sequence number (1=primary)
4. Confidence score (0.0-1.0)
5. Reasoning: cite specific documentation supporting this code
6. Source text excerpt from the note

Clinical note:
{text}

Extracted entities:
{self._format_entities(entities)}

Encounter type: {encounter_type}
"""
```

**Testing:**
- **T3.2.1:** Submit a sample ED note for chest pain with troponin elevation; verify the AI suggests ICD-10-CM I21.* (STEMI) with confidence > 0.80 and a reasoning chain citing troponin values.
- **T3.2.2:** Submit a simple office visit note; verify CPT 99213 or 99214 is suggested with appropriate E/M complexity reasoning.
- **T3.2.3:** Submit a note where the AI suggests a retired/invalid ICD-10-CM code; verify the validation step filters it out and logs a warning.
- **T3.2.4:** Verify the output includes `source_text_excerpt` for each code, pointing to the specific documentation in the note.
- **T3.2.5:** Submit the same note twice; verify results are deterministic (same codes, similar confidence scores).
- **T3.2.6:** Submit an empty clinical note; verify the engine returns an empty suggestion list with an appropriate error message rather than hallucinating codes.
- **T3.2.7:** Measure end-to-end latency for a typical 800-word progress note; verify < 15 seconds.

---

#### Task 3.3: E/M Complexity Calculator

**What:** Implement the 2021 AMA/CMS E/M guidelines calculator that determines the appropriate E/M CPT code (99202-99215 for office visits; 99281-99285 for ED) based on medical decision-making (MDM) complexity or time.

**Design:**
```python
# packages/ai-engine/src/coding/em_calculator.py
from enum import IntEnum

class MDMLevel(IntEnum):
    STRAIGHTFORWARD = 1
    LOW = 2
    MODERATE = 3
    HIGH = 4

class EMCalculator:
    """
    2021 E/M Guidelines: MDM-based level selection.
    MDM has 3 elements; level is determined by 2 of 3 at that level.
    """
    EM_MAP_OFFICE_NEW = {1: '99202', 2: '99203', 3: '99204', 4: '99205'}
    EM_MAP_OFFICE_EST = {1: '99212', 2: '99213', 3: '99214', 4: '99215'}
    EM_MAP_ED         = {1: '99281', 2: '99282', 3: '99283', 4: '99284', 4: '99285'}

    def calculate(
        self,
        num_diagnoses_complexity: MDMLevel,
        data_complexity: MDMLevel,
        risk_level: MDMLevel,
        encounter_type: str,  # 'office_new', 'office_established', 'ed'
        is_time_based: bool = False,
        total_time_minutes: int = None,
    ) -> EMResult:
        if is_time_based and total_time_minutes:
            return self._calculate_time_based(encounter_type, total_time_minutes)

        # MDM-based: 2 of 3 elements determine the level
        levels = sorted([num_diagnoses_complexity, data_complexity, risk_level])
        mdm_level = levels[1]  # Second-highest determines level (2 of 3 rule)

        code_map = {
            'office_new': self.EM_MAP_OFFICE_NEW,
            'office_established': self.EM_MAP_OFFICE_EST,
            'ed': self.EM_MAP_ED,
        }[encounter_type]

        return EMResult(
            mdm_level=MDMLevel(mdm_level).name.lower(),
            calculated_cpt=code_map[mdm_level],
            guideline_year=2021,
            elements={
                'num_diagnoses': num_diagnoses_complexity.name,
                'data_reviewed': data_complexity.name,
                'risk': risk_level.name,
            },
        )
```

**Testing:**
- **T3.3.1:** MDM elements (moderate, moderate, low) for office established → verify CPT 99214 (moderate = 2 of 3).
- **T3.3.2:** MDM elements (high, moderate, high) for office established → verify CPT 99215 (high = 2 of 3).
- **T3.3.3:** MDM elements (low, low, low) for office new → verify CPT 99203.
- **T3.3.4:** ED encounter with MDM (high, high, high) → verify CPT 99285.
- **T3.3.5:** Time-based calculation: 45 minutes office established → verify correct CPT per 2021 time thresholds.
- **T3.3.6:** Verify edge cases: all elements straightforward → 99202 (new) or 99212 (established).

---

#### Task 3.4: Code Assignment API

**What:** API endpoint that accepts an encounter ID, triggers the AI coding pipeline (NLP extraction → code suggestion → E/M calculation), stores results in `code_assignments` and `encounters.ai_coding_result`, and returns the suggestions to the caller.

**Design:**
```typescript
// packages/api/src/modules/coding/coding.routes.ts
app.post('/encounters/:encounterId/code', {
  preHandler: [requirePermission('code_encounters')],
}, async (req) => {
  const encounter = await encounterService.findById(req.params.encounterId);
  const documents = await documentService.findByEncounterId(encounter.id);

  if (documents.length === 0) {
    throw new BadRequestError('No clinical documents attached to this encounter');
  }

  // Transition encounter to ai_processing
  await encounterService.transitionCodingStatus(encounter.id, 'ai_processing', req.user.sub);

  // Call AI engine microservice
  const aiResult = await aiEngineClient.suggestCodes({
    clinicalTexts: documents.map(d => d.content_text),
    encounterType: encounter.encounter_type,
    placeOfService: encounter.place_of_service,
  });

  // Store each suggestion as a code_assignment row
  for (const suggestion of aiResult.suggestions) {
    await codeAssignmentService.create({
      encounterId: encounter.id,
      codeSystem: suggestion.codeSystem,
      code: suggestion.code,
      sequenceNumber: suggestion.sequenceNumber,
      assignmentType: 'ai_suggested',
      confidenceScore: suggestion.confidence,
      aiDetails: {
        model_version: aiResult.modelVersion,
        reasoning: suggestion.reasoning,
        source_text_excerpt: suggestion.sourceTextExcerpt,
        alternative_codes: suggestion.alternatives,
      },
    });
  }

  // Store full AI result on encounter
  await encounterService.updateAICodingResult(encounter.id, aiResult);
  await encounterService.transitionCodingStatus(encounter.id, 'ai_suggested', req.user.sub);

  return aiResult;
});
```

**Testing:**
- **T3.4.1:** Trigger AI coding on an encounter with 2 clinical documents; verify `code_assignments` rows are created for each suggested code with `assignment_type = 'ai_suggested'`.
- **T3.4.2:** Verify the encounter's `ai_coding_result` JSONB is populated with model version, processing time, and all suggestions.
- **T3.4.3:** Verify the encounter's `coding_status` transitions from `pending` → `ai_processing` → `ai_suggested`.
- **T3.4.4:** Trigger AI coding on an encounter with no documents; verify 400 Bad Request.
- **T3.4.5:** Verify events are appended: `CodeBatchSuggestedByAI` with all suggested codes.
- **T3.4.6:** Verify the AI details JSONB on each code_assignment contains reasoning, source_text_excerpt, and alternative_codes.

---

### Definition of Done — Phase 3
- [ ] Clinical NLP pipeline extracting entities from clinical notes with SNOMED normalization
- [ ] AI code suggestion engine producing ICD-10-CM and CPT codes with confidence scores and reasoning
- [ ] E/M complexity calculator implementing 2021 guidelines (MDM and time-based)
- [ ] Code assignment API storing AI suggestions and transitioning encounter status
- [ ] AI suggestions validated against active code_sets before storage
- [ ] End-to-end test: upload note → trigger AI coding → retrieve suggestions with reasoning chains
- [ ] AI engine response time < 15 seconds for typical clinical note

---

### Phase 4: Coder Workstation UI

**Goal:** Build the web-based coder workstation where medical coders review AI suggestions, confirm/modify/reject codes, and manage their coding worklist. This is the primary user-facing interface.

**Duration estimate:** 4-5 weeks

#### Task 4.1: Coder Worklist Dashboard

**What:** React page showing the coder's assigned encounters awaiting review. Filterable by status, encounter type, provider, and date range. Sortable by admit date, AI confidence, and age of encounter.

**Design:**
```tsx
// packages/web/src/app/coding/worklist/page.tsx
export default function CoderWorklist() {
  const { data: encounters } = useEncounterWorklist({
    codingStatus: ['ai_suggested', 'coder_review'],
    sortBy: 'admit_date',
    sortOrder: 'asc',
  });

  return (
    <div className="flex flex-col gap-4">
      <WorklistFilters />
      <WorklistTable
        encounters={encounters}
        columns={[
          { key: 'patient_name', label: 'Patient' },
          { key: 'admit_date', label: 'Date of Service' },
          { key: 'encounter_type', label: 'Type' },
          { key: 'provider_name', label: 'Provider' },
          { key: 'ai_confidence', label: 'AI Confidence', render: ConfidenceBadge },
          { key: 'code_count', label: 'Codes' },
          { key: 'coding_status', label: 'Status', render: StatusBadge },
        ]}
        onRowClick={(enc) => router.push(`/coding/encounter/${enc.id}`)}
      />
    </div>
  );
}
```

**Testing:**
- **T4.1.1:** Load the worklist with 50 encounters across different statuses; verify only `ai_suggested` and `coder_review` encounters are shown.
- **T4.1.2:** Filter by encounter_type = 'emergency'; verify only ED encounters are displayed.
- **T4.1.3:** Sort by AI confidence ascending; verify lowest-confidence encounters appear first (priority review).
- **T4.1.4:** Click an encounter row; verify navigation to the encounter coding detail page.
- **T4.1.5:** Verify the worklist loads within 2 seconds for 500 encounters.

---

#### Task 4.2: Encounter Coding Detail View

**What:** Split-pane view with the clinical document(s) on the left and AI-suggested codes on the right. Coders can read the note, view AI reasoning, and confirm/modify/reject each code suggestion. Includes code search for manual code addition.

**Design:**
```tsx
// packages/web/src/app/coding/encounter/[id]/page.tsx
export default function EncounterCodingView({ params }: { params: { id: string } }) {
  const { data: encounter } = useEncounter(params.id);
  const { data: documents } = useEncounterDocuments(params.id);
  const { data: codeAssignments } = useCodeAssignments(params.id);

  return (
    <SplitPane defaultSize="50%">
      {/* Left pane: Clinical documents */}
      <DocumentViewer
        documents={documents}
        highlightExcerpts={codeAssignments.map(ca => ca.ai_details?.source_text_excerpt)}
      />

      {/* Right pane: Code suggestions and assignment */}
      <div className="flex flex-col gap-4 p-4">
        <CodeSuggestionList
          assignments={codeAssignments}
          onConfirm={(id) => confirmCode(id)}
          onModify={(id, newCode) => modifyCode(id, newCode)}
          onReject={(id, reason) => rejectCode(id, reason)}
        />

        <CodeSearchBar
          onSelect={(code) => addManualCode(encounter.id, code)}
          placeholder="Search ICD-10 or CPT codes..."
        />

        <EncounterActions
          onSubmitCoding={() => finalizeCoding(encounter.id)}
          onRequestCDI={() => createCDIQuery(encounter.id)}
        />
      </div>
    </SplitPane>
  );
}
```

**Testing:**
- **T4.2.1:** Load an encounter with 3 documents and 6 AI-suggested codes; verify the document viewer shows all documents and the code list shows all 6 suggestions with confidence scores.
- **T4.2.2:** Click "Confirm" on a code suggestion; verify the `code_assignment.assignment_type` changes to `coder_confirmed` and a `CodeConfirmedByCoder` event is appended.
- **T4.2.3:** Click "Modify" on a code, search for a replacement code, select it; verify the original assignment is marked `coder_modified` and a new assignment is created with the replacement code.
- **T4.2.4:** Click "Reject" on a code with a rejection reason; verify the assignment is marked `coder_rejected`.
- **T4.2.5:** Verify the document viewer highlights the `source_text_excerpt` for the currently selected code suggestion.
- **T4.2.6:** Use the code search bar to add a manual ICD-10-CM code; verify a new `code_assignment` with `assignment_type = 'manual'` is created.
- **T4.2.7:** Click "Finalize Coding"; verify the encounter transitions to `coded` status.

---

#### Task 4.3: Code Lookup and Search

**What:** Full-text search component for ICD-10-CM, CPT, and HCPCS codes. Supports search by code value, description keyword, and category browsing. Displays official guidelines and cross-references.

**Design:**
```typescript
// packages/api/src/modules/coding/code-search.routes.ts
app.get('/codes/search', { schema: codeSearchSchema }, async (req) => {
  const { q, system, limit = 20 } = req.query;
  // Full-text search using PostgreSQL GIN index on code_sets.description
  const results = await db.query(
    `SELECT code_system, code, description, code_details
     FROM code_sets
     WHERE ($1::varchar IS NULL OR code_system = $1)
       AND is_active = true
       AND (
         code ILIKE $2 || '%'
         OR to_tsvector('english', description) @@ plainto_tsquery('english', $3)
       )
     ORDER BY ts_rank(to_tsvector('english', description), plainto_tsquery('english', $3)) DESC
     LIMIT $4`,
    [system, q, q, limit]
  );
  return results.rows;
});
```

**Testing:**
- **T4.3.1:** Search "type 2 diabetes" with system=ICD10CM; verify E11.* codes are returned, ranked by relevance.
- **T4.3.2:** Search by code prefix "9921"; verify CPT 99211-99215 are returned.
- **T4.3.3:** Search "chest pain" across all systems; verify both ICD-10-CM (R07.*) and relevant CPT codes are returned.
- **T4.3.4:** Verify search returns results within 200ms for the full ICD-10-CM code set (70,000+ codes).
- **T4.3.5:** Verify `code_details` JSONB is returned containing long_description, chapter, and any guidelines notes.

---

#### Task 4.4: NCCI Edit Validation UI

**What:** Real-time NCCI edit checking displayed in the coder workstation. When codes are assigned to an encounter, the UI checks for NCCI PTP edit violations and MUE limit breaches, displaying warnings inline.

**Design:**
```typescript
// packages/api/src/modules/coding/ncci-validator.ts
export class NCCIValidator {
  async validateEncounterCodes(encounterId: string): Promise<NCCIValidationResult> {
    const codes = await this.getActiveCodeAssignments(encounterId);
    const cptCodes = codes.filter(c => c.code_system === 'CPT' || c.code_system === 'HCPCS');
    const violations: NCCIViolation[] = [];

    // Check all pairs for PTP edit violations
    for (let i = 0; i < cptCodes.length; i++) {
      for (let j = i + 1; j < cptCodes.length; j++) {
        const edit = await this.db.query(
          `SELECT * FROM ncci_edits
           WHERE column1_code = $1 AND column2_code = $2
             AND effective_date <= CURRENT_DATE
             AND (termination_date IS NULL OR termination_date > CURRENT_DATE)`,
          [cptCodes[i].code, cptCodes[j].code]
        );
        if (edit.rows.length > 0) {
          violations.push({
            type: 'PTP',
            code1: cptCodes[i].code,
            code2: cptCodes[j].code,
            modifierIndicator: edit.rows[0].modifier_indicator,
            message: edit.rows[0].modifier_indicator === '1'
              ? `NCCI PTP edit: ${cptCodes[j].code} is bundled with ${cptCodes[i].code}. Modifier may override.`
              : `NCCI PTP edit: ${cptCodes[j].code} is bundled with ${cptCodes[i].code}. Cannot be billed separately.`,
          });
        }
      }
    }
    return { violations, isClean: violations.length === 0 };
  }
}
```

**Testing:**
- **T4.4.1:** Assign CPT codes that have an NCCI PTP edit (e.g., 99285 and 93010); verify the UI displays a warning with the correct edit details.
- **T4.4.2:** Assign CPT codes with no NCCI edit relationship; verify no warnings are shown.
- **T4.4.3:** Verify modifier_indicator = '1' shows "modifier may override" message; modifier_indicator = '0' shows "cannot be billed separately."
- **T4.4.4:** Add a modifier to resolve an NCCI edit (e.g., modifier 59); verify the warning is updated to show the edit can be overridden.
- **T4.4.5:** Validate NCCI checking completes within 500ms for an encounter with 10 CPT codes.

---

### Definition of Done — Phase 4
- [ ] Coder worklist showing encounters awaiting review, with filtering and sorting
- [ ] Split-pane encounter coding view with document viewer and code suggestion list
- [ ] Confirm/modify/reject workflow for AI-suggested codes
- [ ] Full-text code search across ICD-10-CM, CPT, and HCPCS
- [ ] Real-time NCCI PTP edit validation with inline warnings
- [ ] Manual code addition via search
- [ ] Coding finalization transitioning encounter to `coded` status
- [ ] All coder actions recorded as events in the event store

---

### Phase 5: Claim Generation & Scrubbing

**Goal:** Generate X12 837P/I claims from coded encounters and run pre-submission scrubbing against NCCI edits, E/M rules, and payer-specific rules.

**Duration estimate:** 4-5 weeks

#### Task 5.1: Claim Assembly from Coded Encounter

**What:** Service that assembles a claim from a coded encounter: maps code_assignments to claim_lines, gathers patient insurance, builds claim_diagnoses from ICD-10-CM assignments, calculates charges, and stores the claim with status `draft`.

**Design:**
```typescript
// packages/api/src/modules/claims/claim-builder.service.ts
export class ClaimBuilderService {
  async buildClaim(encounterId: string): Promise<Claim> {
    const encounter = await this.encounterService.findById(encounterId);
    if (encounter.coding_status !== 'coded') {
      throw new BadRequestError('Encounter must be in "coded" status to generate a claim');
    }

    const patient = await this.patientService.findById(encounter.patient_id);
    const insurance = this.getPrimaryInsurance(patient.insurance);
    const codeAssignments = await this.codeAssignmentService.findCurrentByEncounter(encounterId);

    const dxCodes = codeAssignments.filter(ca => ca.code_system === 'ICD10CM');
    const procCodes = codeAssignments.filter(ca => ['CPT', 'HCPCS'].includes(ca.code_system));

    const claimLines = procCodes.map((proc, i) => ({
      line_number: i + 1,
      procedure_code: proc.code,
      code_system: proc.code_system,
      modifiers: proc.modifiers ?? [],
      diagnosis_pointers: this.mapDiagnosisPointers(proc, dxCodes),
      units: proc.units,
      charge_amount: this.lookupFeeSchedule(proc.code, encounter.place_of_service),
      service_date: encounter.admit_date,
      code_assignment_id: proc.id,
    }));

    const claimDiagnoses = dxCodes.map((dx, i) => ({
      sequence: i + 1,
      code: dx.code,
      poa: encounter.encounter_type === 'inpatient' ? 'Y' : null,
    }));

    const claim = await this.claimRepo.create({
      organisation_id: encounter.organisation_id,
      encounter_id: encounterId,
      patient_id: encounter.patient_id,
      payer_id: insurance.payer_id,
      claim_type: encounter.encounter_type === 'inpatient' ? '837I' : '837P',
      claim_number: this.generateClaimNumber(),
      total_charge: claimLines.reduce((sum, l) => sum + l.charge_amount, 0),
      status: 'draft',
      billing_details: { /* ... provider NPIs, POS, dates ... */ },
      claim_lines: claimLines,
      claim_diagnoses: claimDiagnoses,
    });

    await this.eventStore.append(claim.id, 'claim', 'ClaimDrafted', {
      encounter_id: encounterId,
      total_charge: claim.total_charge,
      line_count: claimLines.length,
    });

    return claim;
  }
}
```

**Testing:**
- **T5.1.1:** Build a claim from a coded encounter with 3 ICD-10-CM and 2 CPT codes; verify claim is created with status `draft`, correct `claim_lines` JSONB, and correct `claim_diagnoses` JSONB.
- **T5.1.2:** Verify diagnosis pointers on claim lines correctly reference the claim diagnoses sequence numbers.
- **T5.1.3:** Verify total_charge equals the sum of all claim line charge_amounts.
- **T5.1.4:** Attempt to build a claim from an encounter not in `coded` status; verify 400 error.
- **T5.1.5:** Build a claim for an inpatient encounter; verify claim_type is '837I' and POA indicators are included on diagnoses.
- **T5.1.6:** Verify the `ClaimDrafted` event is appended with claim metadata.

---

#### Task 5.2: Claim Scrubbing Engine

**What:** Pre-submission scrubbing that validates claims against NCCI edits, E/M documentation rules, modifier requirements, and payer-specific rules from the `payer_rules` table. Produces a `scrub_results` JSONB on the claim.

**Design:**
```typescript
// packages/api/src/modules/claims/claim-scrubber.service.ts
export class ClaimScrubberService {
  async scrubClaim(claimId: string): Promise<ScrubResult> {
    const claim = await this.claimRepo.findById(claimId);
    const edits: ScrubEdit[] = [];

    // 1. NCCI PTP edit check
    const ncciEdits = await this.ncciValidator.validateClaimLines(claim.claim_lines);
    edits.push(...ncciEdits.map(e => ({ type: 'ncci_ptp', severity: 'error', ...e })));

    // 2. NCCI MUE check (max units)
    const mueEdits = await this.ncciValidator.validateMUE(claim.claim_lines);
    edits.push(...mueEdits.map(e => ({ type: 'ncci_mue', severity: 'error', ...e })));

    // 3. Payer-specific rules
    const payerRules = await this.payerRuleService.getActiveRules(claim.payer_id);
    for (const rule of payerRules) {
      const ruleResult = this.evaluatePayerRule(rule, claim);
      if (ruleResult.triggered) {
        edits.push({
          type: 'payer_rule',
          severity: rule.rule_type === 'auth_required' ? 'warning' : 'error',
          message: ruleResult.message,
          suggestion: ruleResult.suggestion,
          rule_id: rule.id,
        });
      }
    }

    const overallResult = edits.some(e => e.severity === 'error') ? 'fail'
                        : edits.some(e => e.severity === 'warning') ? 'warn'
                        : 'pass';

    // Store scrub results on the claim
    await this.claimRepo.updateScrubResults(claimId, {
      scrubbed_at: new Date().toISOString(),
      overall_result: overallResult,
      edits,
    });

    if (overallResult === 'pass') {
      await this.claimRepo.updateStatus(claimId, 'scrubbed');
    }

    await this.eventStore.append(claimId, 'claim', 'ClaimScrubbed', {
      overall_result: overallResult,
      edit_count: edits.length,
      errors: edits.filter(e => e.severity === 'error').length,
    });

    return { overallResult, edits };
  }
}
```

**Testing:**
- **T5.2.1:** Scrub a claim with no NCCI violations or payer rule triggers; verify `overall_result = 'pass'` and claim status transitions to `scrubbed`.
- **T5.2.2:** Scrub a claim with an NCCI PTP edit violation; verify `overall_result = 'fail'` with the specific edit in the results.
- **T5.2.3:** Scrub a claim exceeding MUE limits (e.g., 5 units of a code with MUE = 3); verify error is reported.
- **T5.2.4:** Scrub a claim against a payer rule requiring prior auth for a specific CPT code; verify warning is generated.
- **T5.2.5:** Scrub a claim against a payer-specific bundling rule; verify the rule fires and the suggestion explains how to resolve.
- **T5.2.6:** Verify scrub results are stored in the claim's `scrub_results` JSONB.
- **T5.2.7:** Verify `ClaimScrubbed` event is appended with edit summary.

---

#### Task 5.3: Claim Status Management

**What:** Claim status workflow (draft → scrubbed → submitted → accepted/rejected → paid/denied → appealed/closed) with validation rules and API endpoints for status transitions.

**Testing:**
- **T5.3.1:** Transition claim through valid lifecycle: draft → scrubbed → submitted → accepted → paid; verify each transition records an event.
- **T5.3.2:** Attempt to submit a claim that has not been scrubbed; verify rejection.
- **T5.3.3:** Attempt to submit a claim with scrub_results.overall_result = 'fail'; verify rejection with message to resolve edits first.
- **T5.3.4:** Transition a paid claim to closed; verify final status.
- **T5.3.5:** Verify a claim can be reopened from `scrubbed` back to `draft` for editing.

---

### Definition of Done — Phase 5
- [ ] Claims assembled from coded encounters with correct lines, diagnoses, and charges
- [ ] NCCI PTP and MUE edit checking functional
- [ ] Payer-specific rule evaluation integrated into scrubbing
- [ ] Scrub results stored on claims with pass/warn/fail outcomes
- [ ] Claim lifecycle state machine enforced with events
- [ ] Claims that pass scrubbing transition to `scrubbed` status ready for EDI submission

---

### Phase 6: EDI & Clearinghouse Integration

**Goal:** Generate X12 837P/I EDI transactions from scrubbed claims, submit to clearinghouses (Stedi as primary), parse X12 835 ERA responses, and process X12 270/271 eligibility checks.

**Duration estimate:** 4-5 weeks

#### Task 6.1: X12 837 Claim Generation

**What:** Transform the claim JSONB data into X12 837P (professional) and 837I (institutional) EDI transaction format. Use Stedi SDK for JSON-to-X12 conversion.

**Design:**
```typescript
// packages/api/src/integrations/edi/x12-837-generator.ts
import { Stedi } from '@stedi/sdk';

export class X12837Generator {
  async generate837P(claim: Claim): Promise<string> {
    const stediClaim = this.mapToStedi837P(claim);
    const x12Output = await this.stedi.translate({
      input: stediClaim,
      inputFormat: 'json',
      outputFormat: 'x12',
      transactionSet: '837P',
    });
    return x12Output;
  }

  private mapToStedi837P(claim: Claim): Stedi837P {
    return {
      heading: {
        transaction_set_header: { transaction_set_identifier_code: 'HC', transaction_set_control_number: claim.claim_number },
        billing_provider: { npi: claim.billing_details.billing_provider_npi },
        subscriber: {
          member_id: claim.billing_details.member_id,
          payer_id: claim.billing_details.payer_id_code,
        },
      },
      detail: {
        claim_lines: claim.claim_lines.map(line => ({
          procedure_code: line.procedure_code,
          modifiers: line.modifiers,
          charge_amount: line.charge_amount,
          units: line.units,
          diagnosis_pointers: line.diagnosis_pointers,
          service_date: line.service_date,
        })),
        diagnoses: claim.claim_diagnoses.map(dx => ({
          code: dx.code,
          qualifier: 'ABK', // ICD-10-CM qualifier
        })),
      },
    };
  }
}
```

**Testing:**
- **T6.1.1:** Generate 837P from a professional claim with 3 lines and 4 diagnoses; verify the X12 output contains valid ISA/GS/ST segments and correct loop structure.
- **T6.1.2:** Generate 837I from an institutional claim; verify UB-04-specific segments (revenue codes, POA indicators) are included.
- **T6.1.3:** Submit the generated X12 to Stedi's validation endpoint; verify it passes EDI syntax validation.
- **T6.1.4:** Verify modifier values are correctly placed in the SV1 segment.
- **T6.1.5:** Verify diagnosis pointers in SV1-07 correctly reference the HI segment sequence.

---

#### Task 6.2: Clearinghouse Submission

**What:** Submit generated X12 837 claims to the clearinghouse via API. Handle submission responses (accepted, rejected), track clearinghouse control numbers, and update claim status.

**Testing:**
- **T6.2.1:** Submit a valid 837P claim to Stedi sandbox; verify acceptance response and claim status transitions to `submitted`.
- **T6.2.2:** Submit an invalid claim (missing required field); verify rejection response with error details and claim status transitions to `rejected`.
- **T6.2.3:** Verify clearinghouse tracking ID is stored in `billing_details` JSONB.
- **T6.2.4:** Verify `ClaimSubmitted` event is appended with clearinghouse response metadata.

---

#### Task 6.3: X12 835 ERA Parsing

**What:** Parse incoming X12 835 Electronic Remittance Advice files to extract payment, adjustment, and denial information. Store parsed data in the `remittances` table and link to existing claims.

**Design:**
```typescript
// packages/api/src/integrations/edi/x12-835-parser.ts
export class X12835Parser {
  async parse835(x12Content: string, orgId: string): Promise<RemittanceResult> {
    const parsed = await this.stedi.translate({
      input: x12Content,
      inputFormat: 'x12',
      outputFormat: 'json',
      transactionSet: '835',
    });

    const lineItems = parsed.claims.map(c => ({
      claim_id: this.matchClaimByNumber(c.payer_claim_number),
      payer_claim_number: c.payer_claim_number,
      procedure_code: c.procedure_code,
      charged: c.charge_amount,
      paid: c.paid_amount,
      adjustments: c.adjustments.map(adj => ({
        group_code: adj.group_code,
        carc: adj.reason_code,
        amount: adj.amount,
        rarc: adj.remark_code,
      })),
    }));

    const remittance = await this.remittanceRepo.create({
      organisation_id: orgId,
      payer_id: parsed.payer_id,
      era_date: parsed.check_date,
      check_number: parsed.check_number,
      total_paid: parsed.total_paid,
      line_items,
    });

    // Update claim statuses and create denial records
    for (const item of lineItems) {
      if (item.paid === 0 && item.adjustments.some(a => a.group_code === 'CO')) {
        await this.claimService.updateStatus(item.claim_id, 'denied');
        // Create denial record (handled in Phase 7)
      } else if (item.paid > 0) {
        await this.claimService.updateStatus(item.claim_id, 'paid');
      }
    }

    return remittance;
  }
}
```

**Testing:**
- **T6.3.1:** Parse a sample 835 file with 3 paid claims; verify remittance record is created with correct `total_paid` and `line_items` JSONB.
- **T6.3.2:** Parse an 835 with a denied claim (CARC 97); verify the claim status transitions to `denied`.
- **T6.3.3:** Parse an 835 with partial payment (adjustments with group_code CO); verify paid and adjustment amounts are correct.
- **T6.3.4:** Verify claim matching: 835 payer_claim_number links to the correct claim record.
- **T6.3.5:** Parse a malformed 835; verify error handling with descriptive message.

---

#### Task 6.4: X12 270/271 Eligibility Verification

**What:** Submit eligibility verification requests (X12 270) to clearinghouse and parse responses (X12 271) to confirm patient insurance coverage before claim submission.

**Testing:**
- **T6.4.1:** Submit 270 eligibility request for a patient with active coverage; verify 271 response confirms active status.
- **T6.4.2:** Submit 270 for a patient with terminated coverage; verify 271 response indicates inactive.
- **T6.4.3:** Verify eligibility response is stored on the patient's insurance JSONB with verification date.
- **T6.4.4:** Verify eligibility check can be triggered as a pre-scrub step before claim submission.

---

### Definition of Done — Phase 6
- [ ] X12 837P and 837I generation from claim data
- [ ] Claims submitted to Stedi sandbox with acceptance/rejection handling
- [ ] X12 835 ERA parsing creating remittance records and updating claim statuses
- [ ] X12 270/271 eligibility verification functional
- [ ] All EDI transactions logged in event store with control numbers
- [ ] Integration tests using Stedi sandbox with sample EDI files

---

### Phase 7: Remittance & Denial Management

**Goal:** Build the denial tracking, categorization, root-cause analysis, and appeal workflow. This phase processes denials from ERA 835 data and provides tools for denial management staff.

**Duration estimate:** 3-4 weeks

#### Task 7.1: Denial Detection and Categorization

**What:** Automatically create denial records from ERA 835 remittance data. Categorize denials by CARC/RARC codes into actionable categories (medical necessity, coding error, timely filing, auth required, duplicate, bundling).

**Design:**
```typescript
// packages/api/src/modules/denials/denial-detector.service.ts
const CARC_CATEGORY_MAP: Record<string, string> = {
  '1':   'coding_error',          // Deductible
  '2':   'coding_error',          // Coinsurance
  '4':   'coding_error',          // Procedure code inconsistent with modifier
  '16':  'missing_info',          // Claim/service lacks information
  '18':  'duplicate',             // Exact duplicate
  '29':  'timely_filing',         // Time limit for filing has expired
  '50':  'medical_necessity',     // Non-covered service
  '96':  'medical_necessity',     // Non-covered charge(s)
  '97':  'bundling',              // Procedure/revenue code not paid separately
  '197': 'auth_required',        // Precertification/authorization not obtained
  // ... full CARC mapping
};

export class DenialDetectorService {
  async processRemittanceDenials(remittanceId: string): Promise<Denial[]> {
    const remittance = await this.remittanceRepo.findById(remittanceId);
    const denials: Denial[] = [];

    for (const item of remittance.line_items) {
      const deniedAdjustments = item.adjustments.filter(
        a => a.group_code === 'CO' && a.amount > 0 && !this.isContractualAdjustment(a.carc)
      );

      for (const adj of deniedAdjustments) {
        const denial = await this.denialRepo.create({
          organisation_id: remittance.organisation_id,
          claim_id: item.claim_id,
          payer_id: remittance.payer_id,
          denial_date: remittance.era_date,
          carc_code: adj.carc,
          rarc_code: adj.rarc,
          denial_category: CARC_CATEGORY_MAP[adj.carc] ?? 'other',
          denied_amount: adj.amount,
          status: 'open',
          denial_details: {
            denied_lines: [item.line_number],
            denied_codes: [item.procedure_code],
          },
        });
        denials.push(denial);
      }
    }
    return denials;
  }
}
```

**Testing:**
- **T7.1.1:** Process a remittance with CARC 97 (bundling); verify a denial is created with `denial_category = 'bundling'`.
- **T7.1.2:** Process a remittance with CARC 29 (timely filing); verify categorization and denied_amount.
- **T7.1.3:** Process a remittance with contractual adjustments only (CARC 45 with group_code CO); verify no denial record is created (contractual adjustments are expected).
- **T7.1.4:** Process a remittance with multiple denied lines; verify separate denial records for each.
- **T7.1.5:** Verify denial records link back to the correct claim and payer.

---

#### Task 7.2: Denial Dashboard and Worklist

**What:** UI for denial management staff to view, filter, prioritize, and work denials. Includes summary analytics (denials by category, payer, and provider) and a worklist sorted by financial impact.

**Testing:**
- **T7.2.1:** Load denial dashboard with 200 denials; verify summary cards show correct counts by category.
- **T7.2.2:** Filter denials by payer; verify only denials for that payer are shown.
- **T7.2.3:** Sort denials by denied_amount descending; verify highest-value denials appear first.
- **T7.2.4:** Assign a denial to a staff member; verify assignment is persisted and the denial appears in that user's worklist.

---

#### Task 7.3: AI Denial Root-Cause Analysis

**What:** AI-powered analysis of each denial to determine root cause and suggest corrective action. The AI examines the denial CARC/RARC codes, the original claim data, and the clinical documentation to generate a root-cause assessment.

**Testing:**
- **T7.3.1:** Analyze a bundling denial (CARC 97); verify the AI identifies the specific bundled codes and suggests modifier or rebilling strategy.
- **T7.3.2:** Analyze a medical necessity denial (CARC 50); verify the AI references the relevant LCD/NCD and identifies documentation gaps.
- **T7.3.3:** Verify root_cause_ai is stored in the denial's `denial_details` JSONB.
- **T7.3.4:** Verify the analysis completes within 10 seconds.

---

#### Task 7.4: Appeal Workflow and Letter Generation

**What:** Workflow for creating, tracking, and submitting denial appeals. Includes AI-generated appeal letters that reference supporting clinical documentation and applicable coding guidelines.

**Testing:**
- **T7.4.1:** Initiate an appeal on a denial; verify the `appeals` JSONB array in the denial record is updated with appeal level 1.
- **T7.4.2:** Generate an AI appeal letter for a medical necessity denial; verify the letter references specific clinical findings from the encounter documentation.
- **T7.4.3:** Record appeal outcome (overturned); verify status update and recovered_amount.
- **T7.4.4:** Initiate a level 2 appeal after level 1 is upheld; verify the appeals array has 2 entries.
- **T7.4.5:** Verify the `AppealSubmitted` event is appended with appeal metadata.

---

### Definition of Done — Phase 7
- [ ] Denials automatically created from ERA 835 remittance data
- [ ] Denial categorization by CARC/RARC code mapping
- [ ] Denial dashboard with filtering, sorting, and assignment
- [ ] AI root-cause analysis generating actionable denial explanations
- [ ] Appeal workflow with AI-generated appeal letters
- [ ] Appeal outcome tracking with recovered amount

---

### Phase 8: Denial Learning & Payer Rule AI

**Goal:** Build the AI-driven denial learning loop that detects patterns in denials and automatically generates payer-specific scrubbing rules. This is a key differentiator vs. rule-based competitors.

**Duration estimate:** 3-4 weeks

#### Task 8.1: Denial Pattern Detection

**What:** AI system that analyzes denial history across payer, code, provider, and diagnosis dimensions to detect recurring patterns. When a pattern is detected with sufficient confidence (e.g., same CARC code for the same code pair from the same payer 5+ times in 90 days), the system generates a candidate payer rule.

**Design:**
```python
# packages/ai-engine/src/denials/pattern_detector.py
class DenialPatternDetector:
    async def detect_patterns(self, org_id: str, lookback_days: int = 90) -> list[DenialPattern]:
        """Analyze denial history to detect actionable patterns."""
        denials = await self.db.fetch_denials(org_id, lookback_days)

        # Group by (payer_id, carc_code, procedure_code combination)
        grouped = self._group_denials(denials)
        patterns = []

        for key, group in grouped.items():
            if len(group) >= 5:  # Minimum occurrence threshold
                pattern = DenialPattern(
                    payer_id=key.payer_id,
                    carc_code=key.carc_code,
                    affected_codes=key.procedure_codes,
                    occurrence_count=len(group),
                    total_denied_amount=sum(d.denied_amount for d in group),
                    first_seen=min(d.denial_date for d in group),
                    last_seen=max(d.denial_date for d in group),
                )

                # Use LLM to analyze the pattern and generate a rule recommendation
                pattern.ai_analysis = await self._analyze_pattern_with_llm(pattern, group)
                patterns.append(pattern)

        return patterns
```

**Testing:**
- **T8.1.1:** Insert 7 denials with CARC 97 for CPT pair (36415, 80053) from the same payer; verify a pattern is detected with occurrence_count = 7.
- **T8.1.2:** Insert 3 denials for the same criteria; verify no pattern is detected (below threshold).
- **T8.1.3:** Verify the AI analysis generates a human-readable explanation and rule recommendation for each pattern.
- **T8.1.4:** Verify patterns are deduplicated (same pattern not reported twice).

---

#### Task 8.2: Automated Payer Rule Generation

**What:** Convert detected denial patterns into payer rules that are added to the `payer_rules` table with `source = 'ai_learned'`. Rules are created in inactive state for human review before activation.

**Testing:**
- **T8.2.1:** Generate a payer rule from a bundling denial pattern; verify the rule is created with `source = 'ai_learned'`, `is_active = false`, and correct `rule_definition` JSONB.
- **T8.2.2:** Activate an AI-learned rule; verify subsequent claim scrubbing applies the rule.
- **T8.2.3:** Verify the `PayerRuleLearned` event is appended with the pattern that triggered the rule.
- **T8.2.4:** Deactivate a rule that is generating false positives; verify claims no longer trigger it.
- **T8.2.5:** Verify that when a rule prevents a denial (claim is flagged and corrected before submission), the rule's `denial_count` metric is not incremented but a "prevented" counter is tracked.

---

#### Task 8.3: Payer Rule Management UI

**What:** Admin interface for viewing, activating, deactivating, and editing payer rules (both manual and AI-learned). Shows the denial patterns that generated each AI rule and the financial impact (denials prevented).

**Testing:**
- **T8.3.1:** List all payer rules for a payer; verify both manual and AI-learned rules are shown with source indicators.
- **T8.3.2:** View an AI-learned rule; verify the originating denial pattern is displayed with occurrence count and total denied amount.
- **T8.3.3:** Edit a rule's `rule_definition`; verify changes are persisted.
- **T8.3.4:** Activate/deactivate a rule via toggle; verify status change and audit event.

---

### Definition of Done — Phase 8
- [ ] Denial pattern detection identifying recurring denial patterns across payer/code/provider dimensions
- [ ] AI-generated payer rules created from denial patterns with human review gate
- [ ] Payer rule management UI with activate/deactivate/edit functionality
- [ ] Activated rules integrated into the claim scrubbing engine
- [ ] Closed-loop metrics: denials prevented by AI-learned rules tracked and reportable
- [ ] Pattern detection runs on a configurable schedule (default: daily)

---

### Phase 9: Analytics & Compliance Dashboards

**Goal:** Build comprehensive analytics dashboards for revenue cycle KPIs, coder productivity, AI accuracy, denial trends, and compliance auditing.

**Duration estimate:** 3-4 weeks

#### Task 9.1: Revenue Cycle KPI Dashboard

**What:** Executive dashboard showing clean claim rate, first-pass acceptance rate, average days to payment, denial rate by category, and total revenue recovered through appeals. Data sourced from claims, remittances, and denials tables.

**Testing:**
- **T9.1.1:** With 100 claims in various statuses, verify clean claim rate = (claims passed scrubbing on first attempt / total claims).
- **T9.1.2:** Verify denial rate calculation: total denied amount / total charged amount.
- **T9.1.3:** Verify date range filtering correctly adjusts all KPIs.
- **T9.1.4:** Verify dashboard loads within 3 seconds.

---

#### Task 9.2: Coder Productivity and AI Accuracy

**What:** Dashboard showing per-coder metrics: encounters coded per day, AI suggestion acceptance/modification/rejection rates, average time per encounter, and accuracy based on downstream denial correlation.

**Testing:**
- **T9.2.1:** Verify coder productivity metrics aggregate correctly from event store data.
- **T9.2.2:** Verify AI accuracy metric: (codes confirmed + codes modified) / total codes suggested.
- **T9.2.3:** Verify AI model version tracking: acceptance rate displayed per model version.
- **T9.2.4:** Verify time-per-encounter calculation from event timestamps.

---

#### Task 9.3: Natural Language Compliance Query

**What:** AI-powered compliance query interface where compliance officers type natural language questions about coding patterns (e.g., "show all encounters where high-complexity E/M was billed but documentation supports only moderate complexity") and receive structured results.

**Testing:**
- **T9.3.1:** Query "show encounters with E/M level 99215 in the last 30 days"; verify correct encounters are returned.
- **T9.3.2:** Query "find claims denied for medical necessity by Medicare in Q1"; verify results match denial records.
- **T9.3.3:** Query "show providers with denial rates above 15%"; verify calculation correctness.
- **T9.3.4:** Verify queries complete within 10 seconds.
- **T9.3.5:** Verify compliance officers with `compliance_query` permission can access this feature; other roles cannot.

---

### Definition of Done — Phase 9
- [ ] Revenue cycle KPI dashboard with clean claim rate, denial rate, days to payment
- [ ] Coder productivity metrics with AI acceptance rate tracking
- [ ] AI model accuracy dashboard with per-version tracking
- [ ] Natural language compliance query producing structured results
- [ ] All dashboards filterable by date range, organisation, payer, and provider
- [ ] Dashboard data refreshes within 3 seconds

---

### Phase 10: Autonomous Coding & Scale

**Goal:** Enable autonomous coding for high-volume, lower-complexity specialties (emergency medicine, radiology, urgent care) where AI confidence exceeds a configurable threshold. Encounters meeting the threshold bypass the human coder review queue.

**Duration estimate:** 4-5 weeks

#### Task 10.1: Configurable Auto-Confirmation Threshold

**What:** Organisation-level setting that defines the minimum AI confidence score for automatic code confirmation without human review. Configurable per specialty and encounter type. Claims from auto-confirmed encounters are flagged for post-hoc quality audit sampling.

**Design:**
```typescript
// packages/api/src/modules/coding/auto-confirm.service.ts
export class AutoConfirmService {
  async evaluateForAutoConfirm(encounterId: string): Promise<boolean> {
    const encounter = await this.encounterService.findById(encounterId);
    const org = await this.orgService.findById(encounter.organisation_id);
    const threshold = org.settings.ai_auto_confirm_threshold ?? 0.95;
    const enabledSpecialties = org.settings.auto_confirm_specialties ?? [];

    const provider = await this.providerService.findById(encounter.provider_id);
    if (!enabledSpecialties.includes(provider.specialty_code)) {
      return false;  // Specialty not enabled for auto-confirm
    }

    const aiResult = encounter.ai_coding_result;
    if (!aiResult || aiResult.overall_confidence < threshold) {
      return false;  // Below confidence threshold
    }

    // Auto-confirm all codes
    const assignments = await this.codeAssignmentService.findByEncounter(encounterId);
    for (const a of assignments) {
      if (a.assignment_type === 'ai_suggested' && a.confidence_score >= threshold) {
        await this.codeAssignmentService.autoConfirm(a.id);
      }
    }

    await this.encounterService.transitionCodingStatus(encounterId, 'coded', 'system');
    await this.eventStore.append(encounterId, 'encounter', 'EncounterAutoConfirmed', {
      overall_confidence: aiResult.overall_confidence,
      threshold,
      codes_confirmed: assignments.length,
    });

    return true;
  }
}
```

**Testing:**
- **T10.1.1:** Process an encounter with AI confidence 0.97 and org threshold 0.95; verify auto-confirmation occurs and encounter moves to `coded`.
- **T10.1.2:** Process an encounter with AI confidence 0.90 and org threshold 0.95; verify auto-confirmation does not occur; encounter stays in `ai_suggested` for human review.
- **T10.1.3:** Process an encounter for a specialty not in the enabled list; verify no auto-confirmation regardless of confidence.
- **T10.1.4:** Verify auto-confirmed encounters are flagged for post-hoc quality audit.
- **T10.1.5:** Verify `EncounterAutoConfirmed` event is appended with threshold and confidence details.
- **T10.1.6:** Adjust the threshold from 0.95 to 0.98; verify subsequent encounters use the new threshold.

---

#### Task 10.2: Quality Audit Sampling

**What:** Automated sampling of auto-confirmed encounters for quality review. Configurable sample rate (e.g., 10% of auto-confirmed encounters are flagged for human coder review). Audit results feed back into AI model monitoring.

**Testing:**
- **T10.2.1:** Process 100 auto-confirmed encounters with 10% audit rate; verify approximately 10 are flagged for audit review.
- **T10.2.2:** Complete an audit review finding 2 coding errors; verify the errors are recorded and the AI accuracy metric is updated.
- **T10.2.3:** Verify audit sampling is random and not biased toward specific providers or encounter types.

---

#### Task 10.3: Throughput and Performance Optimization

**What:** Optimize the AI coding pipeline for high-volume autonomous coding: batch processing, connection pooling, database query optimization, and Redis-based job queuing for parallel encounter processing.

**Testing:**
- **T10.3.1:** Process 500 encounters in batch; verify all are coded within 30 minutes.
- **T10.3.2:** Verify database query performance: claim scrubbing for a single claim completes in < 200ms.
- **T10.3.3:** Verify concurrent processing: 10 encounters being coded simultaneously without race conditions.
- **T10.3.4:** Verify Redis job queue processes encounters in FIFO order with retry on failure.

---

### Definition of Done — Phase 10
- [ ] Configurable auto-confirmation threshold per organisation and specialty
- [ ] Autonomous coding workflow for enabled specialties at or above confidence threshold
- [ ] Quality audit sampling of auto-confirmed encounters
- [ ] Audit results feeding back into AI accuracy metrics
- [ ] Batch processing supporting 500+ encounters per hour
- [ ] Performance benchmarks met: scrubbing < 200ms, coding < 15s per encounter

---

### Phase 11: EHR Integration & CDI at Point of Care

**Goal:** Integrate with EHR systems (Epic, Oracle Health, Meditech) via FHIR R4 APIs and SMART on FHIR app launch. Enable real-time CDI alerts during physician note-writing.

**Duration estimate:** 5-6 weeks

#### Task 11.1: FHIR R4 Client for EHR Connectivity

**What:** Build a FHIR R4 client that retrieves Patient, Encounter, DocumentReference, and Condition resources from EHR FHIR endpoints. Implements SMART on FHIR OAuth 2.0 app launch flow with PKCE.

**Design:**
```typescript
// packages/api/src/integrations/fhir/fhir-client.ts
export class FHIRClient {
  async launchFromEHR(launchToken: string, issuer: string): Promise<FHIRSession> {
    // SMART on FHIR EHR Launch flow
    const config = await this.discoverEndpoints(issuer);  // .well-known/smart-configuration
    const tokenResponse = await this.exchangeAuthorizationCode(launchToken, config);
    return new FHIRSession(tokenResponse.access_token, config.fhir_url);
  }

  async getEncounterDocuments(session: FHIRSession, encounterId: string): Promise<ClinicalDocument[]> {
    const bundle = await session.search('DocumentReference', {
      encounter: encounterId,
      status: 'current',
      _sort: '-date',
    });
    return bundle.entry.map(e => this.mapDocumentReference(e.resource));
  }

  async getPatientDemographics(session: FHIRSession, patientId: string): Promise<Patient> {
    const resource = await session.read('Patient', patientId);
    return this.mapPatientResource(resource);
  }
}
```

**Testing:**
- **T11.1.1:** Connect to SMART on FHIR sandbox (e.g., SMART Health IT Sandbox); verify OAuth 2.0 launch flow completes and access token is obtained.
- **T11.1.2:** Retrieve a Patient resource from the sandbox FHIR server; verify demographics are correctly mapped.
- **T11.1.3:** Retrieve DocumentReference resources for an encounter; verify clinical note text is extracted.
- **T11.1.4:** Handle token expiration and refresh; verify transparent token refresh.
- **T11.1.5:** Handle FHIR server errors (404, 403, 500); verify graceful error handling.

---

#### Task 11.2: CDI Query Workflow

**What:** Build the clinical documentation improvement query system. When the AI detects documentation gaps that prevent accurate code assignment, it generates CDI queries that are sent to the attending physician for clarification.

**Testing:**
- **T11.2.1:** AI detects chest pain documentation lacks specificity (R07.9 vs I21.01); verify CDI query is generated asking for clarification.
- **T11.2.2:** Verify CDI query includes potential DRG impact ("Could change DRG from 313 to 280, +$4,200 reimbursement").
- **T11.2.3:** Physician responds to CDI query; verify the response is recorded and coding is updated.
- **T11.2.4:** CDI query timeout (no response after configurable period); verify the query is escalated or closed.

---

#### Task 11.3: Real-Time CDI Alerts (FHIR CDS Hooks)

**What:** Implement CDS Hooks integration that provides real-time CDI alerts within the EHR during physician note-writing. When a physician documents a note, the system analyzes the documentation in progress and alerts if documentation is insufficient to support the intended billing codes.

**Testing:**
- **T11.3.1:** Simulate a CDS Hooks `encounter-discharge` hook; verify the system returns a CDI card with documentation improvement suggestions.
- **T11.3.2:** Verify the CDS response includes specific documentation language the physician should add.
- **T11.3.3:** Verify CDS Hook response time < 5 seconds (SMART on FHIR CDS performance requirement).

---

### Definition of Done — Phase 11
- [ ] FHIR R4 client connecting to EHR sandbox via SMART on FHIR OAuth 2.0
- [ ] Patient, Encounter, and DocumentReference resources retrieved and mapped
- [ ] CDI query workflow with physician notification and response tracking
- [ ] CDS Hooks integration providing real-time CDI alerts
- [ ] Integration tested against SMART Health IT Sandbox
- [ ] FHIR error handling and token management robust

---

## Cross-Phase Concerns

### Security and HIPAA Compliance (All Phases)
- All PHI encrypted at rest (AES-256) and in transit (TLS 1.2+)
- Audit log retention: 6 years minimum (HIPAA requirement); event store partitioned by month for retention management
- Business Associate Agreement (BAA) template prepared for clearinghouse and cloud provider integrations
- Regular vulnerability scanning and penetration testing schedule
- RBAC enforced on every API endpoint

### Testing Strategy (All Phases)
- **Unit tests:** Jest (TypeScript), pytest (Python); >80% coverage target for all modules
- **Integration tests:** Against PostgreSQL test database with seed data; Stedi sandbox for EDI
- **E2E tests:** Playwright for the web frontend; full workflow tests (upload note → AI code → scrub → submit → ERA → denial → appeal)
- **Compliance tests:** Verify NCCI edit checking against CMS published test cases; verify 837 output against Stedi validator
- **Performance tests:** k6 for API load testing; 500 concurrent encounters/hour target by Phase 10

### DevOps (All Phases)
- Docker Compose for local development (Phase 1)
- CI/CD pipeline with automated tests on every PR (Phase 1)
- Kubernetes manifests for staging/production (Phase 6+)
- Database migration CI checks (no destructive migrations without review)
- Automated HIPAA security scanning in CI pipeline

---

## Summary

| Phase | Focus | Duration | Key Deliverable |
|-------|-------|----------|-----------------|
| 1 | Foundation & Reference Data | 4-5 weeks | Database, auth, code importers, event store |
| 2 | Patient & Encounter Management | 3-4 weeks | Patient registry, encounter lifecycle, document ingestion |
| 3 | AI Coding Engine (MVP) | 5-6 weeks | NLP pipeline, AI code suggestion, E/M calculator |
| 4 | Coder Workstation UI | 4-5 weeks | Worklist, coding view, code search, NCCI validation |
| 5 | Claim Generation & Scrubbing | 4-5 weeks | Claim assembly, scrubbing engine, claim lifecycle |
| 6 | EDI & Clearinghouse Integration | 4-5 weeks | X12 837/835/270/271, Stedi integration |
| 7 | Remittance & Denial Management | 3-4 weeks | Denial detection, dashboard, AI root-cause, appeals |
| 8 | Denial Learning & Payer Rule AI | 3-4 weeks | Pattern detection, auto-generated rules, learning loop |
| 9 | Analytics & Compliance Dashboards | 3-4 weeks | KPI dashboards, NL compliance query, coder productivity |
| 10 | Autonomous Coding & Scale | 4-5 weeks | Auto-confirmation, quality audit, batch processing |
| 11 | EHR Integration & CDI | 5-6 weeks | FHIR client, CDI queries, CDS Hooks |
| **Total** | | **~43-53 weeks** | **Full-featured medical billing & coding platform** |

**MVP (Phases 1-6):** ~24-30 weeks — delivers AI-assisted coding, claim generation, EDI submission, and ERA processing.

**Competitive Feature Set (Phases 1-9):** ~33-42 weeks — adds denial management, AI learning loops, and analytics dashboards.

**Full Platform (Phases 1-11):** ~43-53 weeks — includes EHR integration, autonomous coding, and real-time CDI.

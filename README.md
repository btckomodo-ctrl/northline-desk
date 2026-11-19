# northline-desk
M1 spec — freight doc ingest, parse, quarantine

# Northline Desk — M1 Spec

Freight brokerage exception desk. Ingest inbound carrier paperwork, extract fields,
match to a load, write through the TMS adapter, quarantine anything that won't
match. **M1 only:** ingest + parser skeleton + quarantine path, 20 fixture docs,
48 hours, one Bot. No TMS writes in M1 — extraction and refusal only.

Core rule: the extractor never invents a value. Unreadable digit is null, not a
plausible guess.

Thresholds: 0.85 digit fields, 0.75 text, legibility 0.5. Under threshold is
quarantine, never a write. Precedence: LOW_CONFIDENCE > MODEL_DISAGREEMENT >
CHECKSUM_MISMATCH > AMBIGUOUS_PAGE.

## Stack
TypeScript, Next.js, Postgres/Prisma, Inngest, Vitest, GitHub Actions.

## Schema — `prisma/schema.prisma`

```prisma
enum DocumentKind { RATE_CON BOL POD LUMPER_RECEIPT }
enum QuarantineReason { LOW_CONFIDENCE MODEL_DISAGREEMENT CHECKSUM_MISMATCH DOMAIN_MISMATCH AUTHORITY_MISMATCH AMBIGUOUS_PAGE }

model Carrier {
  id String @id @default(cuid())
  legalName String
  mcNumber String @unique
  dotNumber String @unique
  domain String
  authorityOk Boolean @default(false)
  verifiedAt DateTime?
  loads Load[]
}

model Load {
  id String @id @default(cuid())
  loadNumber String @unique
  carrierId String
  carrier Carrier @relation(fields: [carrierId], references: [id])
  origin String
  destination String
  miles Int
  rateCents Int?
  status String @default("TENDERED")
  payoutLockedAt DateTime?
  docs Document[]
  billing BillingLine[]
}

model Document {
  id String @id @default(cuid())
  loadId String?
  load Load? @relation(fields: [loadId], references: [id])
  kind DocumentKind
  receivedFrom String
  contentHash String @unique
  dkimPass Boolean
  legibility Float
  quarantined Boolean @default(false)
  quarantineReason QuarantineReason?
  pageCount Int
  fields ExtractedField[]
  exceptions Exception[]
}

model ExtractedField {
  id String @id @default(cuid())
  documentId String
  document Document @relation(fields: [documentId], references: [id])
  name String
  value String
  confidence Float
  page Int
  bbox String
  accepted Boolean @default(false)
}

model Exception {
  id String @id @default(cuid())
  documentId String
  document Document @relation(fields: [documentId], references: [id])
  loadId String?
  reason QuarantineReason
  state String @default("OPEN")
  ticketRef String?
  createdAt DateTime @default(now())
}

model AuditEvent {
  id String @id @default(cuid())
  entity String
  entityId String
  field String
  oldValue String?
  newValue String?
  actor String
  modelVersion String?
  createdAt DateTime @default(now())
}

model Fixture {
  id String @id @default(cuid())
  kind DocumentKind
  expectedJson Json
  addedBecause String
}

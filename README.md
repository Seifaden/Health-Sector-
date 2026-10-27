# Healthcare Access & Claims Analytics (Synthetic Data)

**PostgreSQL · SQL analytics · Data quality · Healthcare operations · Privacy-aware design**

> **Portfolio prototype using fictional training records.** It contains no real patient data and is not a clinical, billing, compliance, or operational system.

This project shows how I would answer operational questions about emergency department revisits, follow-up scheduling, claim denials and payment amounts while keeping unknown values visible and defining denominators before calculating rates.

## Business questions

1. What share of observed emergency department visits are followed by another ED visit within 30 days for the same synthetic patient key?
2. How do claim denial rates vary by service line among adjudicated claims?
3. How complete is follow-up scheduling documentation by care setting?
4. How do billed, allowed and paid amounts differ by claim status?
5. Which synthetic patient keys have repeat ED visits, and how does revisit rate vary by initial disposition?

## Project files

- `sql/01_synthetic_project.sql` creates an isolated schema and a small, explicitly fictional encounter/claim dataset. It includes basic validation queries.
- `sql/02_analysis.sql` contains eight runnable analysis queries using `LEAD`, window partitions, conditional aggregation, joins, date arithmetic, `HAVING`, `NULLIF` and denominator-aware rates.

## Run it in pgAdmin 4

1. Install PostgreSQL and pgAdmin 4. In pgAdmin, create a new practice database (right-click **Databases → Create → Database**).
2. Select the new database, then open **Tools → Query Tool**.
3. In the Query Tool, open `sql/01_synthetic_project.sql` with the folder button and click **Execute/Refresh**. It creates the demo schema and inserts the fictional example rows.
4. Open `sql/02_analysis.sql` in Query Tool and execute it. Each query produces its own result grid. Use the query selector/dropdown if you want to run and study just one query at a time.

The setup drops and recreates only the demo's `access_analytics` schema. Use a personal practice database and review the SQL before running it in any shared environment.

## Metric definitions and practical cautions

- **30-day ED revisit:** each ED visit is flagged when the same synthetic patient key has a later ED encounter within 30 calendar days. A patient can contribute more than one index visit. This is a utilization indicator, not a preventability measure, readmission measure or care-quality score.
- **Denial rate:** denied claims divided by paid, denied and adjusted claims. Pending claims are excluded from the denominator because adjudication is incomplete. If a real organization's definition differs, document and change it.
- **Follow-up scheduling:** reports yes, no and unknown separately. The percentage uses only known yes/no responses; missing values are never treated as “no.”
- **Financial fields:** billed, allowed and paid amounts remain separate. A blank payment on a pending claim is not converted to zero.
- **Small groups:** the data are deliberately tiny to make query results inspectable. Rates by disposition or service line are demonstrations, not stable estimates.

## Privacy and governance choices

The CSV from the Spotify project is not transformed into healthcare data. Replacing column labels would misrepresent the source, create invented health information and teach the wrong privacy lesson. This project instead uses synthetic example rows with no names, birth dates, addresses, contact details, medical-record numbers, free text or real claim IDs.

The synthetic patient key still acts as a join key in the fictional example. In real healthcare work, a pseudonymized identifier can still be sensitive and should be protected under the organization's policies. Follow approved access controls, minimum-necessary practices, retention rules, audit logging and secure environments. Do not upload real patient data to GitHub, ChatGPT, public SQL playgrounds or an unapproved personal database. HIPAA applicability depends on the organization, data and circumstances; this demo does not establish HIPAA compliance.

For a more realistic next version, use the [MITRE Synthea synthetic patient generator](https://synthea.mitre.org/downloads), record the exact dataset version and license, retain only fields needed for a defined question, and document data lineage and exclusions. Synthea provides synthetic records; synthetic status should still be clearly disclosed and does not validate clinical accuracy.

## Limitations

This small fabricated dataset has no real clinical context, coding system, insurance contract, patient demographics, eligibility history or operational workflow. It cannot support clinical conclusions, reimbursement policy decisions or estimates of real readmission/denial performance. The SQL demonstrates analytical reasoning and query construction only.

## Skills demonstrated

SQL modeling, window functions, repeat-visit logic, rate denominators, missing-value handling, claim-status segmentation, referential integrity, data validation and communicating limitations.

## Data provenance

All rows are authored fictional examples in the SQL setup file for a learning portfolio. The Spotify dataset is not used in this healthcare project.

## The SQL, included in this README

The complete code below is the same SQL stored in the two `.sql` files. You can copy a block into pgAdmin's Query Tool or open the corresponding file.

### Setup: create tables and insert fictional examples

```sql
-- Synthetic, fictional training scenario. PostgreSQL 15+.
-- Rebuild only the project schema. Never load real patient data into this demo.
BEGIN;
DROP SCHEMA IF EXISTS access_analytics CASCADE;
CREATE SCHEMA access_analytics;

-- Pseudonymous IDs are fabricated. No names, exact addresses, DOBs, or free text.
CREATE TABLE access_analytics.encounters (
  encounter_id text PRIMARY KEY,
  patient_key text NOT NULL,
  encounter_date date NOT NULL,
  care_setting text NOT NULL CHECK (care_setting IN ('ED','urgent_care','primary_care')),
  service_line text NOT NULL,
  disposition text NOT NULL CHECK (disposition IN ('home','admitted','transferred','unknown')),
  followup_scheduled boolean,
  CHECK (encounter_id !~* '(name|email|phone|ssn|mrn)')
);
CREATE TABLE access_analytics.claims (
  claim_id text PRIMARY KEY,
  encounter_id text NOT NULL REFERENCES access_analytics.encounters(encounter_id),
  claim_status text NOT NULL CHECK (claim_status IN ('paid','denied','pending','adjusted')),
  billed_amount numeric(10,2) NOT NULL CHECK (billed_amount >= 0),
  allowed_amount numeric(10,2) CHECK (allowed_amount >= 0),
  paid_amount numeric(10,2) CHECK (paid_amount >= 0),
  denial_reason_group text,
  CHECK (claim_status <> 'denied' OR paid_amount IS NULL OR paid_amount = 0)
);
INSERT INTO access_analytics.encounters VALUES
('E001','P001','2025-01-03','ED','cardiology','home',true),
('E002','P001','2025-01-17','ED','cardiology','home',true),
('E003','P001','2025-03-01','primary_care','cardiology','home',false),
('E004','P002','2025-01-05','ED','respiratory','admitted',true),
('E005','P002','2025-01-20','ED','respiratory','home',true),
('E006','P003','2025-01-08','urgent_care','musculoskeletal','home',false),
('E007','P003','2025-02-15','ED','musculoskeletal','home',true),
('E008','P004','2025-01-11','ED','cardiology','transferred',true),
('E009','P004','2025-02-19','primary_care','cardiology','home',true),
('E010','P005','2025-01-12','ED','respiratory','home',false),
('E011','P005','2025-01-25','ED','respiratory','home',false),
('E012','P006','2025-01-15','urgent_care','cardiology','home',true),
('E013','P007','2025-01-18','ED','musculoskeletal','home',true),
('E014','P008','2025-01-21','ED','respiratory','unknown',NULL),
('E015','P009','2025-01-24','primary_care','cardiology','home',false),
('E016','P010','2025-01-28','ED','respiratory','home',true),
('E017','P010','2025-04-09','ED','respiratory','home',true),
('E018','P006','2025-02-18','primary_care','cardiology','home',true);
INSERT INTO access_analytics.claims VALUES
('C001','E001','paid',1200,840,700,NULL),
('C002','E002','denied',900,NULL,0,'authorization'),
('C003','E003','paid',180,125,125,NULL),
('C004','E004','paid',5200,3900,3500,NULL),
('C005','E005','adjusted',780,600,500,NULL),
('C006','E006','pending',240,NULL,NULL,NULL),
('C007','E007','paid',1650,1100,950,NULL),
('C008','E008','denied',3100,NULL,0,'eligibility'),
('C009','E009','paid',210,160,150,NULL),
('C010','E010','denied',1150,NULL,0,'authorization'),
('C011','E011','paid',890,650,600,NULL),
('C012','E012','paid',430,310,280,NULL),
('C013','E013','pending',275,NULL,NULL,NULL),
('C014','E014','paid',2040,1400,1250,NULL),
('C015','E015','paid',195,145,130,NULL),
('C016','E016','adjusted',1870,1220,1010,NULL),
('C017','E017','paid',2130,1580,1360,NULL),
('C018','E018','paid',205,155,140,NULL);
COMMIT;

-- Governance and data quality review: verify scope, join coverage and claim arithmetic.
SELECT 'encounters' object_name,count(*) row_count FROM access_analytics.encounters
UNION ALL SELECT 'claims',count(*) FROM access_analytics.claims;
SELECT count(*) encounter_count,
       count(*) FILTER (WHERE followup_scheduled IS NULL) unknown_followup,
       count(*) FILTER (WHERE disposition='unknown') unknown_disposition
FROM access_analytics.encounters;
SELECT c.claim_id,c.billed_amount,c.allowed_amount,c.paid_amount
FROM access_analytics.claims c
WHERE (c.allowed_amount IS NOT NULL AND c.allowed_amount>c.billed_amount)
   OR (c.paid_amount IS NOT NULL AND c.allowed_amount IS NOT NULL AND c.paid_amount>c.allowed_amount);
```

### Analysis: eight operational questions

```sql
-- 1) ED revisit within 30 days of a prior ED visit.
-- This synthetic workflow indicator is not a preventability or quality judgment.
WITH ed AS (
 SELECT patient_key,encounter_id,encounter_date,
        lead(encounter_date) OVER (PARTITION BY patient_key ORDER BY encounter_date,encounter_id) next_ed_date
 FROM access_analytics.encounters WHERE care_setting='ED'
), flags AS (
 SELECT *,next_ed_date<=encounter_date+30 AS ed_revisit_30d
 FROM ed
)
SELECT count(*) eligible_ed_visits,
       count(*) FILTER (WHERE ed_revisit_30d) visits_followed_by_ed_within_30d,
       round(100.0*count(*) FILTER (WHERE ed_revisit_30d)/nullif(count(*),0),1) pct_ed_visits_followed_by_ed_30d
FROM flags;

-- 2) Report the encounter-level flags with dates, for auditability.
WITH ed AS (
 SELECT patient_key,encounter_id,encounter_date,
        lead(encounter_date) OVER (PARTITION BY patient_key ORDER BY encounter_date,encounter_id) next_ed_date
 FROM access_analytics.encounters WHERE care_setting='ED'
)
SELECT encounter_id,encounter_date,next_ed_date,
       next_ed_date<=encounter_date+30 AS ed_revisit_30d
FROM ed ORDER BY encounter_date,encounter_id;

-- 3) Denial rate by service line. Pending claims are not counted as adjudicated.
SELECT e.service_line,
       count(*) FILTER (WHERE c.claim_status IN ('paid','denied','adjusted')) adjudicated_claims,
       count(*) FILTER (WHERE c.claim_status='denied') denied_claims,
       round(100.0*count(*) FILTER (WHERE c.claim_status='denied') /
         nullif(count(*) FILTER (WHERE c.claim_status IN ('paid','denied','adjusted')),0),1) denial_rate_pct
FROM access_analytics.claims c JOIN access_analytics.encounters e USING(encounter_id)
GROUP BY e.service_line ORDER BY denial_rate_pct DESC NULLS LAST;

-- 4) Denial reason counts and billed dollars, with small-cell suppression for display.
SELECT denial_reason_group,count(*) denied_claims,sum(billed_amount) denied_billed_amount
FROM access_analytics.claims WHERE claim_status='denied'
GROUP BY denial_reason_group HAVING count(*)>=2 ORDER BY denied_claims DESC;

-- 5) Follow-up scheduling completeness by setting. Keep unknown separate from false.
SELECT care_setting,count(*) encounters,
       count(*) FILTER (WHERE followup_scheduled IS TRUE) followup_yes,
       count(*) FILTER (WHERE followup_scheduled IS FALSE) followup_no,
       count(*) FILTER (WHERE followup_scheduled IS NULL) followup_unknown,
       round(100.0*count(*) FILTER (WHERE followup_scheduled IS TRUE) /
             nullif(count(*) FILTER (WHERE followup_scheduled IS NOT NULL),0),1) pct_yes_among_known
FROM access_analytics.encounters GROUP BY care_setting ORDER BY care_setting;

-- 6) Paid amount and billed amount are different measures; pending is not zero-paid.
SELECT claim_status,count(*) claims,sum(billed_amount) billed_amount,
       sum(allowed_amount) allowed_amount,sum(paid_amount) paid_amount,
       round(avg(paid_amount),2) avg_paid_per_claim_with_recorded_payment
FROM access_analytics.claims GROUP BY claim_status ORDER BY claim_status;

-- 7) Highest repeat ED use by synthetic patient key; minimum 2 ED visits.
SELECT patient_key,count(*) ed_visits,min(encounter_date) first_seen,max(encounter_date) last_seen
FROM access_analytics.encounters WHERE care_setting='ED'
GROUP BY patient_key HAVING count(*)>=2 ORDER BY ed_visits DESC,patient_key;

-- 8) Compare ED revisit rates by initial disposition. Interpret very small groups cautiously.
WITH ed AS (
 SELECT disposition,patient_key,encounter_date,
        lead(encounter_date) OVER(PARTITION BY patient_key ORDER BY encounter_date,encounter_id) next_ed_date
 FROM access_analytics.encounters WHERE care_setting='ED'
)
SELECT disposition,count(*) visits,
       count(*) FILTER(WHERE next_ed_date<=encounter_date+30) revisit_visits,
       round(100.0*count(*) FILTER(WHERE next_ed_date<=encounter_date+30)/nullif(count(*),0),1) revisit_rate_pct
FROM ed GROUP BY disposition ORDER BY disposition;
```

## Analytical workflow

The encounter table supplies the utilization grain; the claims table stores adjudication and financial fields at claim grain. A foreign key connects each claim to its encounter. The revisit query partitions ED visits by synthetic patient key, orders them chronologically, and uses `LEAD` to identify the next observed ED date. Denial rates use adjudicated claims as the denominator, excluding pending claims. Follow-up completeness reports unknown documentation separately from recorded “no.”

These definitions are explicit because a rate can change materially when its denominator changes. The project demonstrates query logic on a small fabricated sample; it does not estimate real utilization, reimbursement, or care quality.

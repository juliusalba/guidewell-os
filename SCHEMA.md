# Guidewell OS — Data Schema

## Entity Relationship Overview

```
┌──────────────┐       ┌──────────────┐       ┌──────────────────┐
│   COMPANY    │──────<│    DEAL      │──────<│       JOB        │
│  (Client)    │  1:N  │ (Requisition)│  1:N  │   (Open Role)    │
└──────────────┘       └──────────────┘       └────────┬─────────┘
       │                      │                        │
       │                      │                        │ 1:N
       │                      │                        │
       │               ┌──────┴───────┐         ┌─────┴────────┐
       │               │              │         │              │
       │         ┌─────┴────┐  ┌──────┴──┐     │  CANDIDATE   │
       │         │RECRUITER │  │ACTIVITY │     │  (Talent)    │
       │         │  (User)  │  │(Touch)  │     └───────┬──────┘
       │         └─────┬────┘  └──────────┘             │
       │               │                    ┌───────────┴────────┐
       │               │                    │                    │
       │               │             ┌─────┴──────┐     ┌──────┴──────┐
       │               │             │  CANDIDATE  │     │  CANDIDATE  │
       │               │             │   _JOB      │     │   _JOB      │
       │               │             │ (Junction)  │     │  ACTIVITY   │
       │               │             └─────────────┘     └─────────────┘
       │               │
       │               └── N:1 ───> RECRUITER (owner)
       │
       └────── N:1 ────> DEAL (as "prospect_company" or "client_company")

┌─────────────────────────────────────────────────────────────────┐
│                        ENTITY DETAIL                             │
└─────────────────────────────────────────────────────────────────┘

───────────────────────────────────────────────────────────────────
COMPANY
───────────────────────────────────────────────────────────────────
Table: companies
───────────────────────────────────────────────────────────────────
id                UUID        PRIMARY KEY DEFAULT gen_random_uuid()
name              TEXT        NOT NULL
slug              TEXT        UNIQUE NOT NULL
industry          TEXT        e.g. "IT Infrastructure", "Fintech"
size              TEXT        e.g. "50-200", "201-500", "Enterprise"
location_city     TEXT
location_country  TEXT        DEFAULT 'Netherlands'
website           TEXT
linkedin_url      TEXT
logo_url          TEXT
status            TEXT        DEFAULT 'prospect'  -- 'prospect' | 'active' | 'inactive'
created_at        TIMESTAMPTZ DEFAULT now()
updated_at        TIMESTAMPTZ DEFAULT now()

Relationships:
  - A Company can have MANY Deals (1:N via deals.client_company_id)
  - A Company can be a prospect for MANY Deals (1:N via deals.prospect_company_id)

───────────────────────────────────────────────────────────────────
DEAL
───────────────────────────────────────────────────────────────────
Table: deals
───────────────────────────────────────────────────────────────────
id                UUID        PRIMARY KEY DEFAULT gen_random_uuid()
title             TEXT        NOT NULL  -- e.g. "Randstad Group — UX/UI Designers"
client_company_id UUID        REFERENCES companies(id)  -- the client we bill
prospect_company_id UUID      REFERENCES companies(id)  -- prospect we're targeting
recruiter_id      UUID        REFERENCES recruiters(id) NOT NULL  -- deal owner
stage             TEXT        DEFAULT 'prospect'
                  -- 'prospect' | 'qualified' | 'proposal' | 'negotiation' | 'won' | 'lost'
value             INTEGER     -- total deal value in EUR (sum of job placements)
currency          TEXT        DEFAULT 'EUR'
notes             TEXT
closed_at         TIMESTAMPTZ
created_at        TIMESTAMPTZ DEFAULT now()
updated_at        TIMESTAMPTZ DEFAULT now()

Relationships:
  - A Deal belongs to ONE Recruiter (owner) — N:1 via recruiter_id
  - A Deal belongs to ONE Client Company — N:1 via client_company_id
  - A Deal can target ONE Prospect Company — N:1 via prospect_company_id
  - A Deal can have MANY Jobs — 1:N via jobs.deal_id

Stage Lifecycle:
  prospect → qualified → proposal → negotiation → won | lost

───────────────────────────────────────────────────────────────────
RECRUITER
───────────────────────────────────────────────────────────────────
Table: recruiters
───────────────────────────────────────────────────────────────────
id                UUID        PRIMARY KEY DEFAULT gen_random_uuid()
user_id           UUID        REFERENCES auth.users(id)  -- if using Supabase Auth
name              TEXT        NOT NULL
email             TEXT        UNIQUE NOT NULL
phone             TEXT
title             TEXT        -- e.g. "Senior Recruiter", "Researcher"
department        TEXT        -- "Engineering", "Sales", "Operations"
avatar_url        TEXT
role              TEXT        DEFAULT 'recruiter'  -- 'admin' | 'manager' | 'recruiter' | 'researcher' | 'associate'
status            TEXT        DEFAULT 'active'  -- 'active' | 'inactive'
activity_score    INTEGER     DEFAULT 0  -- 0-100, calculated from recent activity
joined_at         TIMESTAMPTZ DEFAULT now()
last_active_at    TIMESTAMPTZ DEFAULT now()

Relationships:
  - A Recruiter can own MANY Deals — 1:N via deals.recruiter_id
  - A Recruiter can own MANY Jobs — 1:N via jobs.recruiter_id
  - A Recruiter logs MANY Activities — 1:N via activities.recruiter_id

───────────────────────────────────────────────────────────────────
JOB
───────────────────────────────────────────────────────────────────
Table: jobs
───────────────────────────────────────────────────────────────────
id                UUID        PRIMARY KEY DEFAULT gen_random_uuid()
deal_id           UUID        REFERENCES deals(id) NOT NULL
recruiter_id      UUID        REFERENCES recruiters(id) NOT NULL  -- job owner
title             TEXT        NOT NULL  -- e.g. "Senior UX/UI Designer"
slug              TEXT
department        TEXT        -- e.g. "Engineering", "Design", "Sales"
employment_type   TEXT        -- 'full-time' | 'part-time' | 'contract' | 'internship'
location_city     TEXT
location_country  TEXT        DEFAULT 'Netherlands'
location_type     TEXT        -- 'onsite' | 'hybrid' | 'remote'
salary_min        INTEGER     -- annual, EUR
salary_max        INTEGER     -- annual, EUR
currency          TEXT        DEFAULT 'EUR'
description       TEXT        -- full job description (Markdown)
requirements      TEXT        -- JSONB or TEXT
status            TEXT        DEFAULT 'open'  -- 'open' | 'in_progress' | 'filled' | 'on_hold' | 'cancelled'
priority          TEXT        DEFAULT 'normal'  -- 'low' | 'normal' | 'urgent' | 'critical'
filled_at         TIMESTAMPTZ
created_at        TIMESTAMPTZ DEFAULT now()
updated_at        TIMESTAMPTZ DEFAULT now()

Relationships:
  - A Job belongs to ONE Deal — N:1 via deal_id
  - A Job belongs to ONE Recruiter (owner) — N:1 via recruiter_id
  - A Job can have MANY Candidates (via candidate_jobs junction) — M:N via candidate_jobs
  - A Job can have MANY Interview events — 1:N via interviews.job_id

───────────────────────────────────────────────────────────────────
CANDIDATE
───────────────────────────────────────────────────────────────────
Table: candidates
───────────────────────────────────────────────────────────────────
id                UUID        PRIMARY KEY DEFAULT gen_random_uuid()
name              TEXT        NOT NULL
email             TEXT
phone             TEXT
linkedin_url      TEXT
linkedin_profile  TEXT        -- JSONB: {headline, summary, experience[], skills[]}
location_city     TEXT
location_country  TEXT
current_title     TEXT
current_company   TEXT
years_experience  INTEGER
skills            TEXT[]      -- array of skill tags
tags              TEXT[]      -- internal tags: "top-fit", "sourced", "referral"
source            TEXT        -- 'linkedin' | 'referral' | 'website' | 'cold_outreach' | 'database' | 'gpt_sourced'
recruiter_id      UUID        REFERENCES recruiters(id)  -- who sourced/owns this candidate
resume_url        TEXT
ai_summary        TEXT        -- AI-generated candidate summary
ai_score          INTEGER     -- AI-computed fit score 0-100
status            TEXT        DEFAULT 'active'  -- 'active' | 'inactive' | 'placed' | 'archived'
created_at        TIMESTAMPTZ DEFAULT now()
updated_at        TIMESTAMPTZ DEFAULT now()

Relationships:
  - A Candidate can apply to/wrk MANY Jobs — M:N via candidate_jobs
  - A Candidate has MANY Activities — 1:N via activities.candidate_id

───────────────────────────────────────────────────────────────────
CANDIDATE_JOB (Junction — many-to-many)
───────────────────────────────────────────────────────────────────
Table: candidate_jobs
───────────────────────────────────────────────────────────────────
id                UUID        PRIMARY KEY DEFAULT gen_random_uuid()
candidate_id      UUID        REFERENCES candidates(id) NOT NULL
job_id            UUID        REFERENCES jobs(id) NOT NULL
stage             TEXT        DEFAULT 'applied'
                  -- 'applied' | 'screening' | 'phone_interview' | 'technical_interview'
                  -- 'cultural_interview' | 'final_interview' | 'offer' | 'hired' | 'rejected'
stage_changed_at   TIMESTAMPTZ
applied_at        TIMESTAMPTZ DEFAULT now()
notes             TEXT        -- recruiter notes on this specific application
rating            INTEGER     -- 1-5 star rating
interview_score   INTEGER     -- post-interview AI score
offer_value       INTEGER     -- offered salary, EUR
offer_status      TEXT        -- 'pending' | 'accepted' | 'declined' | 'negotiating'
created_at        TIMESTAMPTZ DEFAULT now()
updated_at        TIMESTAMPTZ DEFAULT now()

UNIQUE(candidate_id, job_id)  -- a candidate can only apply once per job

Relationships:
  - CANDIDATE_JOB belongs to ONE Candidate — N:1 via candidate_id
  - CANDIDATE_JOB belongs to ONE Job — N:1 via job_id
  - CANDIDATE_JOB has MANY Activities — 1:N via activities.candidate_job_id

───────────────────────────────────────────────────────────────────
ACTIVITY (All touchpoints — the heartbeat of the system)
───────────────────────────────────────────────────────────────────
Table: activities
───────────────────────────────────────────────────────────────────
id                UUID        PRIMARY KEY DEFAULT gen_random_uuid()
recruiter_id      UUID        REFERENCES recruiters(id) NOT NULL
candidate_id      UUID        REFERENCES candidates(id)
job_id            UUID        REFERENCES jobs(id)
candidate_job_id  UUID        REFERENCES candidate_jobs(id)
deal_id           UUID        REFERENCES deals(id)
type              TEXT        NOT NULL
                  -- 'email_sent' | 'email_received' | 'call_made' | 'call_received'
                  -- | 'linkedin_message' | 'linkedin_view' | 'interview_scheduled'
                  -- | 'interview_completed' | 'offer_sent' | 'offer_accepted' | 'offer_declined'
                  -- | 'rejection_sent' | 'note_added' | 'stage_changed' | 'ai_outreach'
                  -- | 'meeting_scheduled' | 'meeting_completed'
subject           TEXT        -- short description
content           TEXT        -- full content (email body, call notes, etc.)
direction         TEXT        -- 'outbound' | 'inbound' | 'internal'
ai_generated      BOOLEAN     DEFAULT false  -- true if content was AI-generated
ai_model          TEXT        -- e.g. 'gpt-4o', 'claude-sonnet-4'
sent_via          TEXT        -- 'email' | 'linkedin' | 'calendly' | 'manual'
created_at        TIMESTAMPTZ DEFAULT now()

Relationships:
  - Activity belongs to ONE Recruiter — N:1 via recruiter_id
  - Activity belongs to ONE Candidate (optional) — N:1 via candidate_id
  - Activity belongs to ONE Job (optional) — N:1 via job_id
  - Activity belongs to ONE Candidate_Job (optional) — N:1 via candidate_job_id
  - Activity belongs to ONE Deal (optional) — N:1 via deal_id

───────────────────────────────────────────────────────────────────
OUTREACH_SEQUENCE
───────────────────────────────────────────────────────────────────
Table: outreach_sequences
───────────────────────────────────────────────────────────────────
id                UUID        PRIMARY KEY DEFAULT gen_random_uuid()
candidate_id      UUID        REFERENCES candidates(id) NOT NULL
job_id            UUID        REFERENCES jobs(id)
recruiter_id      UUID        REFERENCES recruiters(id) NOT NULL
name              TEXT        -- e.g. "Lena Kowalski — BlueWave MD"
status            TEXT        DEFAULT 'active'  -- 'active' | 'paused' | 'completed' | 'bounced'
current_step      INTEGER     DEFAULT 1
total_steps       INTEGER
channel           TEXT        DEFAULT 'email'  -- 'email' | 'linkedin' | 'both'
created_at        TIMESTAMPTZ DEFAULT now()
updated_at        TIMESTAMPTZ DEFAULT now()

Relationships:
  - Sequence belongs to ONE Candidate — N:1 via candidate_id
  - Sequence belongs to ONE Job — N:1 via job_id
  - Sequence belongs to ONE Recruiter — N:1 via recruiter_id
  - Sequence has MANY Outreach_Messages — 1:N via outreach_messages.sequence_id

───────────────────────────────────────────────────────────────────
OUTREACH_MESSAGE
───────────────────────────────────────────────────────────────────
Table: outreach_messages
───────────────────────────────────────────────────────────────────
id                UUID        PRIMARY KEY DEFAULT gen_random_uuid()
sequence_id       UUID        REFERENCES outreach_sequences(id) NOT NULL
step              INTEGER     NOT NULL
subject           TEXT
body              TEXT        -- the actual message content
ai_generated      BOOLEAN     DEFAULT false
ai_prompt         TEXT        -- what was used to generate this
status            TEXT        DEFAULT 'pending'  -- 'pending' | 'sent' | 'delivered' | 'opened' | 'replied' | 'bounced' | 'failed'
sent_at           TIMESTAMPTZ
opened_at         TIMESTAMPTZ
replied_at        TIMESTAMPTZ
created_at        TIMESTAMPTZ DEFAULT now()

Relationships:
  - Message belongs to ONE Sequence — N:1 via sequence_id

───────────────────────────────────────────────────────────────────
INTERVIEW
───────────────────────────────────────────────────────────────────
Table: interviews
───────────────────────────────────────────────────────────────────
id                UUID        PRIMARY KEY DEFAULT gen_random_uuid()
candidate_job_id  UUID        REFERENCES candidate_jobs(id) NOT NULL
job_id            UUID        REFERENCES jobs(id) NOT NULL
recruiter_id      UUID        REFERENCES recruiters(id) NOT NULL
interview_type    TEXT        NOT NULL  -- 'screening' | 'phone' | 'technical' | 'cultural' | 'final' | 'hr'
scheduled_at      TIMESTAMPTZ NOT NULL
duration_minutes  INTEGER     DEFAULT 60
location_or_link  TEXT        -- "Google Meet link" or "Amsterdam HQ"
calendar_event_id TEXT        -- Google Calendar event ID
status            TEXT        DEFAULT 'scheduled'  -- 'scheduled' | 'completed' | 'cancelled' | 'no_show'
feedback          TEXT        -- recruiter's post-interview notes
ai_score          INTEGER     -- post-interview AI evaluation score
rating            INTEGER     -- 1-5
created_at        TIMESTAMPTZ DEFAULT now()
updated_at        TIMESTAMPTZ DEFAULT now()

Relationships:
  - Interview belongs to ONE Candidate_Job — N:1 via candidate_job_id
  - Interview belongs to ONE Job — N:1 via job_id
  - Interview belongs to ONE Recruiter — N:1 via recruiter_id

───────────────────────────────────────────────────────────────────
COMPANY_DEAL_NOTE (Company-level notes)
───────────────────────────────────────────────────────────────────
Table: company_notes
───────────────────────────────────────────────────────────────────
id                UUID        PRIMARY KEY DEFAULT gen_random_uuid()
company_id        UUID        REFERENCES companies(id) NOT NULL
recruiter_id      UUID        REFERENCES recruiters(id) NOT NULL
content           TEXT        NOT NULL
created_at        TIMESTAMPTZ DEFAULT now()

───────────────────────────────────────────────────────────────────
INDEXES (Performance)
───────────────────────────────────────────────────────────────────
CREATE INDEX idx_deals_recruiter ON deals(recruiter_id);
CREATE INDEX idx_deals_stage ON deals(stage);
CREATE INDEX idx_deals_client ON deals(client_company_id);
CREATE INDEX idx_jobs_deal ON jobs(deal_id);
CREATE INDEX idx_jobs_recruiter ON jobs(recruiter_id);
CREATE INDEX idx_jobs_status ON jobs(status);
CREATE INDEX idx_candidates_recruiter ON candidates(recruiter_id);
CREATE INDEX idx_candidates_tags ON candidates USING GIN(tags);
CREATE INDEX idx_candidate_jobs_candidate ON candidate_jobs(candidate_id);
CREATE INDEX idx_candidate_jobs_job ON candidate_jobs(job_id);
CREATE INDEX idx_activities_recruiter ON activities(recruiter_id);
CREATE INDEX idx_activities_candidate ON activities(candidate_id);
CREATE INDEX idx_activities_type ON activities(type);
CREATE INDEX idx_activities_created ON activities(created_at DESC);
CREATE INDEX idx_outreach_seq_candidate ON outreach_sequences(candidate_id);
CREATE INDEX idx_interviews_cj ON interviews(candidate_job_id);

───────────────────────────────────────────────────────────────────
ROW LEVEL SECURITY (RLS) — Supabase
───────────────────────────────────────────────────────────────────
-- Recruiters can only see/edit their own data (plus managers see all)
-- Admins see everything

ALTER TABLE deals ENABLE ROW LEVEL SECURITY;
ALTER TABLE jobs ENABLE ROW LEVEL SECURITY;
ALTER TABLE candidates ENABLE ROW LEVEL SECURITY;
ALTER TABLE activities ENABLE ROW LEVEL SECURITY;
ALTER TABLE candidate_jobs ENABLE ROW LEVEL SECURITY;

-- Policy example for deals:
CREATE POLICY "Recruiters see own deals, managers see all"
  ON deals FOR ALL
  USING (
    auth.uid() = recruiter_id
    OR EXISTS (
      SELECT 1 FROM recruiters
      WHERE recruiters.user_id = auth.uid()
      AND recruiters.role IN ('admin', 'manager')
    )
  );

───────────────────────────────────────────────────────────────────
SAMPLE DATA — Quick Seed
───────────────────────────────────────────────────────────────────

-- Companies
INSERT INTO companies (id, name, slug, industry, size, location_city, status) VALUES
  ('c1','Randstad Group','randstad-group','Staffing & Recruitment','Enterprise','Diemen','active'),
  ('c2','Finova Bank','finova-bank','Financial Services','201-500','Amsterdam','active'),
  ('c3','GreenTech NL','greentech-nl','Sustainability','51-200','Groningen','active'),
  ('c4','BlueWave Media','bluewave-media','Digital Marketing','11-50','Rotterdam','active'),
  ('c5','Techcom Solutions','techcom-solutions','IT Infrastructure','51-200','Amsterdam','prospect'),
  ('c6','DataFlow BV','dataflow-bv','Data Analytics','11-50','Utrecht','prospect');

-- Recruiters
INSERT INTO recruiters (id, name, email, title, role, activity_score) VALUES
  ('r1','Brian Anderson','brian@guidewell.nl','Senior Recruiter','manager',91),
  ('r2','Sara Lindqvist','sara@guidewell.nl','Tech Recruiter','recruiter',84),
  ('r3','Maya Kim','maya@guidewell.nl','Researcher','researcher',68),
  ('r4','Thomas Vermeulen','thomas@guidewell.nl','Associate','associate',55);

-- Deals
INSERT INTO deals (id, title, client_company_id, recruiter_id, stage, value) VALUES
  ('d1','Randstad Group — UX/UI & Dev',  'c1','r1','won',        96000),
  ('d2','Finova Bank — Engineering',      'c2','r1','negotiation', 52000),
  ('d3','GreenTech NL — IT Roles',      'c3','r1','proposal',    38000),
  ('d4','BlueWave Media — Creative',     'c4','r2','qualified',   24000),
  ('d5','Techcom Solutions — Infrastructure','c5','r4','prospect',18000),
  ('d6','DataFlow BV — Analytics',       'c6','r3','prospect',   12000);

-- Jobs
INSERT INTO jobs (id, deal_id, recruiter_id, title, status, priority, salary_min, salary_max) VALUES
  ('j1','d1','r1','Senior UX/UI Designer',  'in_progress','urgent',  58000, 78000),
  ('j2','d1','r1','Java Developer',         'in_progress','urgent',  65000, 85000),
  ('j3','d1','r1','Front-end Developer',    'open',       'normal', 50000, 70000),
  ('j4','d1','r1','Data Analyst',            'filled',     'normal', 45000, 65000),
  ('j5','d2','r1','Backend Engineer',        'in_progress','normal', 60000, 80000),
  ('j6','d2','r1','QA Engineer',            'open',       'normal', 40000, 55000),
  ('j7','d2','r1','DevOps Engineer',        'open',       'normal', 55000, 75000),
  ('j8','d3','r1','Netwerkspecialist',      'open',       'normal', 38000, 51000),
  ('j9','d3','r1','Sustainability Advisor',  'open',       'low',   42000, 60000),
  ('j10','d4','r2','Creative Director',      'in_progress','normal', 70000, 95000),
  ('j11','d5','r4','Network Engineer',       'open',       'normal', 45000, 65000),
  ('j12','d6','r3','Data Engineer',         'open',       'normal', 50000, 70000);

-- Candidates (subset)
INSERT INTO candidates (id, name, email, current_title, current_company, years_experience, skills, source, recruiter_id, ai_score) VALUES
  ('ca1','Thomas Müller',   'thomas.muller@email.de',   'Principal Designer',    'Heineken',        11, ARRAY['UX','UI','Design Systems','Figma'],    'linkedin',  'r1', 94),
  ('ca2','Priya Nair',      'priya.nair@email.nl',      'Lead Designer',         'Rituals',          8, ARRAY['UX','Product Design','Prototyping'], 'linkedin',  'r1', 91),
  ('ca3','Lena Kowalski',   'lena.kowalski@email.pl',  'Product Designer',      'Studio8',          5, ARRAY['UX','UI','User Research'],            'website',   'r1', 85),
  ('ca4','Anna Schmidt',    'anna.schmidt@email.de',   'UX Researcher',          'ING',              4, ARRAY['User Research','Usability Testing'], 'referral',  'r1', 82),
  ('ca5','Marco van der Berg','marco.vanderberg@email.nl','Senior UX Designer','Booking.com',      7, ARRAY['UX','Service Design','Workshop'],      'linkedin',  'r1', 87),
  ('ca6','Fatima Khan',     'fatima.khan@email.nl',   'Design Lead',            'Philips',          9, ARRAY['UX Strategy','Design Ops'],          'gpt_sourced','r1',88),
  ('ca7','James O''Brien', 'james.obrien@email.ie',   'Backend Developer',      'Workiva',          6, ARRAY['Java','Spring Boot','AWS'],           'linkedin',  'r1', 86),
  ('ca8','Sara Lindqvist',  'sara.lindqvist@email.se', 'Design Director',        'Coolblue',        11, ARRAY['Design Leadership','UX','Strategy'], 'referral',  'r1', 93);

-- Candidate → Job mappings
INSERT INTO candidate_jobs (candidate_id, job_id, stage) VALUES
  ('ca1','j1','offer'),       -- Thomas Müller — offer stage for UX/UI
  ('ca2','j1','technical_interview'), -- Priya — technical
  ('ca3','j1','applied'),     -- Lena — applied
  ('ca4','j1','screening'),    -- Anna — screening
  ('ca5','j1','applied'),     -- Marco — applied
  ('ca6','j1','applied'),     -- Fatima — applied
  ('ca7','j2','first_interview'),  -- James — 1st interview for Java
  ('ca8','j1','final_interview'); -- Sara — final for UX/UI

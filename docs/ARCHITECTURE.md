# System Architecture & Database Specification (`docs/ARCHITECTURE.md`)

## 1. Core Architectural Principles
* **Plug-and-Play Program Engine:** The application codebase contains zero hardcoded exam logic. All exam structures, passing thresholds, timers, and question formats are driven by database records (`programs`, `program_domains`, and `JSONB` payloads).
* **Frontend & API:** Next.js 15 (App Router) Responsive Web App / PWA deployed on Vercel. Route handlers (`/app/api/*`) handle test session evaluation, Redis state sync, and payment webhooks.
* **Primary Database & Auth:** Supabase (PostgreSQL 15+ with Row-Level Security enabled on all public tables).
* **Ephemeral Session & Streak Cache:** Upstash Redis (HTTP REST) stores active quiz timers, temporary in-progress answers, and daily study streaks.
* **Asset Storage:** Cloudflare R2 stores question diagrams, geometry figures, and reading passages with zero egress fees.

---

## 2. Database Schema & Supabase Migration (PostgreSQL DDL)

Run this entire SQL script inside the **Supabase SQL Editor** during Step 2.

```sql
-- ============================================================================
-- 0. EXTENSIONS
-- ============================================================================
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ============================================================================
-- 1. USER PROFILES & ROLES (Strict Separation of Privileges)
-- ============================================================================
CREATE TABLE public.profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    display_name VARCHAR(120),
    avatar_url TEXT,
    target_program_id UUID, -- FK added after programs table creation
    daily_goal_questions INT NOT NULL DEFAULT 10,
    current_streak INT NOT NULL DEFAULT 0,
    longest_streak INT NOT NULL DEFAULT 0,
    last_studied_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Separate table for elevated roles so users cannot self-promote via profile updates
CREATE TABLE public.user_roles (
    user_id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    role VARCHAR(32) NOT NULL DEFAULT 'learner'
        CHECK (role IN ('learner', 'sme_creator', 'editor', 'admin')),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================================
-- 2. PLUG-AND-PLAY EXAM PROGRAMS & HIERARCHICAL BLUEPRINTS
-- ============================================================================
CREATE TABLE public.programs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug VARCHAR(64) UNIQUE NOT NULL, -- e.g., 'upcat', 'sat', 'cse-prof', 'nclex-rn'
    name VARCHAR(255) NOT NULL,
    short_name VARCHAR(64) NOT NULL,
    category VARCHAR(64) NOT NULL, -- 'college_entrance', 'civil_service', 'healthcare', 'it_cert'
    description TEXT,
    blueprint_config JSONB NOT NULL DEFAULT '{
        "passing_score_percent": 75,
        "mock_duration_minutes": 180,
        "mock_question_count": 100,
        "allow_calculator": false,
        "negative_marking": false
    }'::jsonb,
    is_active BOOLEAN NOT NULL DEFAULT false,
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE public.profiles
    ADD CONSTRAINT fk_profiles_target_program
    FOREIGN KEY (target_program_id) REFERENCES public.programs(id) ON DELETE SET NULL;

-- N-Tier Domain / Competency Tree (Section -> Subject -> Topic)
CREATE TABLE public.program_domains (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id UUID NOT NULL REFERENCES public.programs(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES public.program_domains(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    weight_percentage NUMERIC(5,2) DEFAULT 0.00, -- Official exam blueprint weight
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================================
-- 3. PASSAGES & POLYMORPHIC QUESTION BANK
-- ============================================================================
-- Shared reading comprehension passages or clinical/case-study scenarios
CREATE TABLE public.passages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id UUID NOT NULL REFERENCES public.programs(id) ON DELETE CASCADE,
    title VARCHAR(255),
    body_markdown TEXT NOT NULL,
    media_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE public.questions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id UUID NOT NULL REFERENCES public.programs(id) ON DELETE CASCADE,
    domain_id UUID REFERENCES public.program_domains(id) ON DELETE SET NULL,
    passage_id UUID REFERENCES public.passages(id) ON DELETE SET NULL,
    difficulty_level SMALLINT NOT NULL DEFAULT 3 CHECK (difficulty_level BETWEEN 1 AND 5),
    question_type VARCHAR(32) NOT NULL DEFAULT 'single_choice'
        CHECK (question_type IN ('single_choice', 'multiple_response', 'fill_numeric', 'ordering')),
    is_free_tier BOOLEAN NOT NULL DEFAULT false, -- True = accessible in Freemium QBank
    content JSONB NOT NULL,
    /* Expected content JSONB structure:
       {
         "prompt_markdown": "Solve for $x$: $2x^2 - 8 = 0$",
         "media_url": "[https://pub-xxx.r2.dev/diagrams/parabola-01.png](https://pub-xxx.r2.dev/diagrams/parabola-01.png)",
         "numeric_answer": { "value": 2, "tolerance": 0.01, "unit": "cm" } // optional
       }
    */
    explanation JSONB NOT NULL,
    /* Expected explanation JSONB structure:
       {
         "summary": "Factor out 2 and apply difference of squares.",
         "steps_markdown": "1. $2(x^2 - 4) = 0$ ...",
         "reference_citation": "College Algebra, 10th Ed., Ch. 2"
       }
    */
    status VARCHAR(20) NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft', 'in_review', 'published', 'archived')),
    created_by UUID REFERENCES auth.users(id) ON DELETE SET NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE public.question_options (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id UUID NOT NULL REFERENCES public.questions(id) ON DELETE CASCADE,
    label VARCHAR(8), -- e.g., 'A', 'B', 'C', 'D'
    content JSONB NOT NULL, -- { "text_markdown": "...", "media_url": null }
    is_correct BOOLEAN NOT NULL DEFAULT false,
    option_rationale TEXT, -- Why this specific distractor is right or wrong
    sort_order SMALLINT NOT NULL DEFAULT 0
);

-- ============================================================================
-- 4. FREEMIUM ENTITLEMENTS, SUBSCRIPTIONS & DONATIONS
-- ============================================================================
CREATE TABLE public.user_entitlements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    program_id UUID REFERENCES public.programs(id) ON DELETE CASCADE, -- NULL = All-Access Pass
    tier VARCHAR(32) NOT NULL DEFAULT 'premium'
        CHECK (tier IN ('premium', 'exam_pass', 'supporter_badge')),
    provider VARCHAR(32) NOT NULL, -- 'stripe', 'paymongo', 'manual'
    provider_reference_id VARCHAR(255),
    starts_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ, -- NULL = Lifetime / One-time Supporter Badge
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE public.donations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
    amount_cents INT NOT NULL CHECK (amount_cents > 0),
    currency VARCHAR(8) NOT NULL DEFAULT 'PHP',
    provider VARCHAR(32) NOT NULL,
    provider_payment_id VARCHAR(255) UNIQUE,
    supporter_message TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'succeeded',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================================
-- 5. TEST SESSIONS, ITEM RESPONSES & SPACED REPETITION
-- ============================================================================
CREATE TABLE public.test_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    program_id UUID NOT NULL REFERENCES public.programs(id) ON DELETE CASCADE,
    session_mode VARCHAR(32) NOT NULL
        CHECK (session_mode IN ('diagnostic', 'daily_10', 'custom_tutor', 'custom_exam', 'mock_simulation')),
    status VARCHAR(20) NOT NULL DEFAULT 'in_progress'
        CHECK (status IN ('in_progress', 'completed', 'abandoned')),
    question_ids UUID[] NOT NULL, -- Ordered list of question UUIDs in this session
    time_limit_seconds INT,
    time_remaining_seconds INT,
    filter_config JSONB NOT NULL DEFAULT '{}'::jsonb,
    score_percent NUMERIC(5,2),
    correct_count INT DEFAULT 0,
    total_questions INT NOT NULL,
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

CREATE TABLE public.user_responses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES public.test_sessions(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    question_id UUID NOT NULL REFERENCES public.questions(id) ON DELETE CASCADE,
    selected_option_ids UUID[] DEFAULT '{}',
    numeric_response NUMERIC,
    is_correct BOOLEAN NOT NULL,
    is_flagged BOOLEAN NOT NULL DEFAULT false,
    time_spent_seconds INT NOT NULL DEFAULT 0,
    answered_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (session_id, question_id)
);

-- Modified SM-2 Spaced Repetition Queue per User & Question
CREATE TABLE public.spaced_repetition_cards (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    program_id UUID NOT NULL REFERENCES public.programs(id) ON DELETE CASCADE,
    question_id UUID NOT NULL REFERENCES public.questions(id) ON DELETE CASCADE,
    box_level SMALLINT NOT NULL DEFAULT 1 CHECK (box_level BETWEEN 1 AND 5),
    ease_factor NUMERIC(4,2) NOT NULL DEFAULT 2.50,
    interval_days INT NOT NULL DEFAULT 1,
    next_review_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_reviewed_at TIMESTAMPTZ,
    UNIQUE (user_id, question_id)
);

-- ============================================================================
-- 6. PERFORMANCE INDEXES (Foreign Keys & RLS Filter Columns)
-- ============================================================================
CREATE INDEX idx_program_domains_program ON public.program_domains(program_id);
CREATE INDEX idx_program_domains_parent ON public.program_domains(parent_id);
CREATE INDEX idx_questions_program_status ON public.questions(program_id, status);
CREATE INDEX idx_questions_domain ON public.questions(domain_id);
CREATE INDEX idx_questions_free_tier ON public.questions(program_id, is_free_tier) WHERE status = 'published';
CREATE INDEX idx_question_options_question ON public.question_options(question_id);
CREATE INDEX idx_entitlements_user_active ON public.user_entitlements(user_id, is_active);
CREATE INDEX idx_test_sessions_user_program ON public.test_sessions(user_id, program_id);
CREATE INDEX idx_user_responses_user_question ON public.user_responses(user_id, question_id);
CREATE INDEX idx_user_responses_session ON public.user_responses(session_id);
CREATE INDEX idx_srs_user_due ON public.spaced_repetition_cards(user_id, program_id, next_review_at);

-- ============================================================================
-- 7. AUTOMATIC PROFILE & ROLE CREATION ON SIGNUP
-- ============================================================================
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = ''
AS $$ BEGIN INSERT INTO public.profiles (id, display_name, avatar_url) VALUES ( NEW.id, COALESCE(NEW.raw_user_meta_data ->> 'full_name', split_part(NEW.email, '@', 1)), NEW.raw_user_meta_data ->> 'avatar_url' );      INSERT INTO public.user_roles (user_id, role) VALUES (NEW.id, 'learner');      RETURN NEW; END; $$;

CREATE TRIGGER on_auth_user_created
    AFTER INSERT ON auth.users
    FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- ============================================================================
-- 8. ROW-LEVEL SECURITY (RLS) POLICIES
-- ============================================================================
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.user_roles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.programs ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.program_domains ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.passages ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.questions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.question_options ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.user_entitlements ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.donations ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.test_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.user_responses ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.spaced_repetition_cards ENABLE ROW LEVEL SECURITY;

-- Helper function to check active premium access (cached per statement)
CREATE OR REPLACE FUNCTION public.has_program_access(target_program_id UUID)
RETURNS BOOLEAN
LANGUAGE sql
STABLE
SECURITY DEFINER
SET search_path = ''
AS $$SELECT EXISTS ( SELECT 1 FROM public.user_entitlements ue WHERE ue.user_id = (SELECT auth.uid()) AND ue.is_active = true AND ue.tier IN ('premium', 'exam_pass') AND (ue.program_id IS NULL OR ue.program_id = target_program_id) AND (ue.expires_at IS NULL OR ue.expires_at > NOW()) );$$;

-- Profiles: Users read and update only their own profile
CREATE POLICY "Users read own profile" ON public.profiles
    FOR SELECT TO authenticated USING ((SELECT auth.uid()) = id);
CREATE POLICY "Users update own profile" ON public.profiles
    FOR UPDATE TO authenticated USING ((SELECT auth.uid()) = id) WITH CHECK ((SELECT auth.uid()) = id);

-- User Roles: Users can read own role; only service_role can modify
CREATE POLICY "Users read own role" ON public.user_roles
    FOR SELECT TO authenticated USING ((SELECT auth.uid()) = user_id);

-- Catalog (Programs & Domains): Publicly readable when active
CREATE POLICY "Public read active programs" ON public.programs
    FOR SELECT TO anon, authenticated USING (is_active = true);
CREATE POLICY "Public read program domains" ON public.program_domains
    FOR SELECT TO anon, authenticated USING (true);

-- Passages & Questions: Free-tier published items open to authenticated users;
-- Paid items require an active entitlement for that program
CREATE POLICY "Read accessible passages" ON public.passages
    FOR SELECT TO authenticated USING (true);

CREATE POLICY "Read permitted published questions" ON public.questions
    FOR SELECT TO authenticated
    USING (
        status = 'published'
        AND (is_free_tier = true OR (SELECT public.has_program_access(program_id)))
    );

CREATE POLICY "Read options for accessible questions" ON public.question_options
    FOR SELECT TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM public.questions q
            WHERE q.id = question
# AI-Powered Talent Acquisition System - Data Model

## 1. Entity Relationship Overview

The data model is designed to support multi-modal candidate evaluation through resume analysis, video interview processing, and AI-powered ranking. The model follows a normalized approach with clear separation of concerns.

## 2. Core Entities

### 2.1 Candidates Table
Primary entity representing job applicants.

```sql
CREATE TABLE candidates (
    candidate_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),
    location VARCHAR(255),
    linkedin_url VARCHAR(500),
    github_url VARCHAR(500),
    portfolio_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(50) DEFAULT 'active', -- active, inactive, hired, rejected
    CONSTRAINT chk_email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
);
```

### 2.2 Jobs Table
Job positions and their requirements.

```sql
CREATE TABLE jobs (
    job_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(200) NOT NULL,
    department VARCHAR(100),
    job_level VARCHAR(50), -- entry, junior, mid, senior, principal
    employment_type VARCHAR(50), -- full-time, part-time, contract, internship
    location VARCHAR(255),
    salary_min DECIMAL(12,2),
    salary_max DECIMAL(12,2),
    currency VARCHAR(3) DEFAULT 'USD',
    description TEXT,
    requirements TEXT,
    preferred_qualifications TEXT,
    hiring_manager_id UUID,
    status VARCHAR(50) DEFAULT 'open', -- open, closed, on-hold, filled
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2.3 Applications Table
Links candidates to specific job applications.

```sql
CREATE TABLE applications (
    application_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    candidate_id UUID NOT NULL REFERENCES candidates(candidate_id),
    job_id UUID NOT NULL REFERENCES jobs(job_id),
    application_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(50) DEFAULT 'submitted', -- submitted, screening, interview, offer, hired, rejected
    source VARCHAR(100), -- website, referral, linkedin, job_board
    notes TEXT,
    UNIQUE(candidate_id, job_id)
);
```

## 3. Resume Processing Entities

### 3.1 Resumes Table
Stores resume files and metadata.

```sql
CREATE TABLE resumes (
    resume_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES applications(application_id),
    original_filename VARCHAR(255) NOT NULL,
    file_path VARCHAR(500) NOT NULL,
    file_size_bytes INTEGER,
    file_format VARCHAR(10), -- pdf, doc, docx
    mime_type VARCHAR(100),
    upload_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processing_status VARCHAR(50) DEFAULT 'uploaded', -- uploaded, processing, completed, failed
    processing_started_at TIMESTAMP,
    processing_completed_at TIMESTAMP,
    error_message TEXT
);
```

### 3.2 Resume Analysis Table
LLM-generated analysis and summaries.

```sql
CREATE TABLE resume_analysis (
    analysis_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resume_id UUID NOT NULL REFERENCES resumes(resume_id),
    extracted_text TEXT,
    executive_summary TEXT, -- 200-word LLM summary
    total_experience_years DECIMAL(4,2),
    top_strengths TEXT[], -- Array of top 5 strengths
    potential_weaknesses TEXT[], -- Array of identified gaps
    relevance_score INTEGER CHECK (relevance_score >= 0 AND relevance_score <= 100),
    llm_model_used VARCHAR(100),
    processing_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    confidence_score DECIMAL(5,4) -- 0.0000 to 1.0000
);
```

### 3.3 Work Experience Table
Parsed work history from resumes.

```sql
CREATE TABLE work_experience (
    experience_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resume_id UUID NOT NULL REFERENCES resumes(resume_id),
    company_name VARCHAR(200),
    job_title VARCHAR(200),
    start_date DATE,
    end_date DATE, -- NULL for current position
    is_current BOOLEAN DEFAULT FALSE,
    location VARCHAR(255),
    description TEXT,
    achievements TEXT[],
    technologies_used TEXT[],
    duration_months INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.4 Education Table
Educational background from resumes.

```sql
CREATE TABLE education (
    education_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resume_id UUID NOT NULL REFERENCES resumes(resume_id),
    institution_name VARCHAR(200),
    degree_type VARCHAR(100), -- B.Tech, M.Tech, MBA, PhD, etc.
    field_of_study VARCHAR(200),
    start_date DATE,
    end_date DATE,
    gpa DECIMAL(4,2),
    location VARCHAR(255),
    achievements TEXT[],
    relevant_coursework TEXT[],
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.5 Skills Table
Master skills repository.

```sql
CREATE TABLE skills (
    skill_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    skill_name VARCHAR(100) UNIQUE NOT NULL,
    category VARCHAR(50), -- technical, soft, language, certification
    skill_type VARCHAR(50), -- programming_language, framework, tool, methodology
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.6 Candidate Skills Table
Skills extracted from resumes with proficiency levels.

```sql
CREATE TABLE candidate_skills (
    candidate_skill_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resume_id UUID NOT NULL REFERENCES resumes(resume_id),
    skill_id UUID NOT NULL REFERENCES skills(skill_id),
    proficiency_level VARCHAR(50), -- beginner, intermediate, advanced, expert
    experience_months INTEGER,
    mentioned_in_section VARCHAR(50), -- skills, experience, projects
    confidence_score DECIMAL(5,4), -- LLM confidence in extraction
    UNIQUE(resume_id, skill_id)
);
```

### 3.7 Certifications Table
Professional certifications and achievements.

```sql
CREATE TABLE certifications (
    certification_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resume_id UUID NOT NULL REFERENCES resumes(resume_id),
    certification_name VARCHAR(200),
    issuing_organization VARCHAR(200),
    issue_date DATE,
    expiry_date DATE,
    credential_id VARCHAR(100),
    verification_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 4. Video Interview Processing Entities

### 4.1 Interviews Table
Video interview files and metadata.

```sql
CREATE TABLE interviews (
    interview_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES applications(application_id),
    interview_type VARCHAR(50), -- screening, technical, behavioral, final
    interview_date TIMESTAMP,
    interviewer_name VARCHAR(200),
    interviewer_email VARCHAR(255),
    original_filename VARCHAR(255),
    file_path VARCHAR(500),
    file_size_bytes BIGINT,
    video_format VARCHAR(10), -- mp4, mov, avi, webm
    duration_seconds INTEGER,
    upload_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processing_status VARCHAR(50) DEFAULT 'uploaded', -- uploaded, processing, completed, failed
    processing_started_at TIMESTAMP,
    processing_completed_at TIMESTAMP,
    error_message TEXT
);
```

### 4.2 Interview Transcriptions Table
Speech-to-text results using Whisper.

```sql
CREATE TABLE interview_transcriptions (
    transcription_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id UUID NOT NULL REFERENCES interviews(interview_id),
    full_transcript TEXT NOT NULL,
    transcript_json JSONB, -- Structured transcript with timestamps
    audio_quality_score DECIMAL(5,4), -- 0.0000 to 1.0000
    transcription_confidence DECIMAL(5,4),
    language_detected VARCHAR(10) DEFAULT 'en',
    speaker_count INTEGER DEFAULT 1,
    processing_model VARCHAR(100), -- whisper-large-v2, etc.
    processing_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 4.3 Interview Analysis Table
LLM-powered interview content analysis.

```sql
CREATE TABLE interview_analysis (
    analysis_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id UUID NOT NULL REFERENCES interviews(interview_id),
    
    -- LLM-generated summaries
    executive_summary TEXT,
    key_talking_points TEXT[],
    best_responses TEXT[],
    concerning_responses TEXT[],
    
    -- LLM-assessed scores (0-100)
    communication_score INTEGER CHECK (communication_score >= 0 AND communication_score <= 100),
    technical_competency_score INTEGER,
    problem_solving_score INTEGER,
    cultural_fit_score INTEGER,
    confidence_level_score INTEGER,
    
    -- LLM sentiment analysis
    overall_sentiment VARCHAR(50), -- positive, neutral, negative
    sentiment_confidence DECIMAL(5,4),
    
    -- LLM recommendations
    interviewer_recommendations TEXT,
    follow_up_questions TEXT[],
    red_flags TEXT[],
    
    -- Processing metadata
    llm_model_used VARCHAR(100),
    processing_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    analysis_confidence DECIMAL(5,4)
);
```

### 4.4 Interview Questions Table
Structured Q&A extraction from interviews.

```sql
CREATE TABLE interview_questions (
    question_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    interview_id UUID NOT NULL REFERENCES interviews(interview_id),
    question_text TEXT,
    answer_text TEXT,
    question_category VARCHAR(100), -- technical, behavioral, situational
    start_timestamp INTEGER, -- seconds from start
    end_timestamp INTEGER,
    answer_quality_score INTEGER CHECK (answer_quality_score >= 0 AND answer_quality_score <= 100),
    llm_analysis TEXT, -- LLM evaluation of the response
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 5. Evaluation & Ranking Entities

### 5.1 Candidate Evaluations Table
Comprehensive multi-modal candidate assessment.

```sql
CREATE TABLE candidate_evaluations (
    evaluation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id UUID NOT NULL REFERENCES applications(application_id),
    
    -- Resume-based scores (0-100)
    resume_relevance_score INTEGER,
    experience_match_score INTEGER,
    skills_match_score INTEGER,
    education_score INTEGER,
    
    -- Interview-based scores (0-100)
    interview_performance_score INTEGER,
    communication_score INTEGER,
    technical_score INTEGER,
    cultural_fit_score INTEGER,
    
    -- LLM-generated qualitative assessment
    llm_personality_profile TEXT,
    llm_strengths_assessment TEXT,
    llm_weaknesses_assessment TEXT,
    llm_cultural_fit_analysis TEXT,
    llm_growth_potential TEXT,
    
    -- ML-generated quantitative scores
    ml_overall_score DECIMAL(8,4), -- 0.0000 to 100.0000
    ml_classification VARCHAR(50), -- junior, mid-level, senior, etc.
    ml_confidence_interval JSONB, -- {"lower": 85.2, "upper": 92.8}
    
    -- Combined hybrid score
    final_score DECIMAL(8,4),
    recommendation VARCHAR(50), -- strong_hire, hire, maybe, no_hire, strong_no_hire
    
    -- Processing metadata
    evaluation_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    llm_model_used VARCHAR(100),
    ml_model_used VARCHAR(100),
    evaluator_notes TEXT
);
```

### 5.2 Job Skill Requirements Table
Skills required for specific jobs with weights.

```sql
CREATE TABLE job_skill_requirements (
    requirement_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID NOT NULL REFERENCES jobs(job_id),
    skill_id UUID NOT NULL REFERENCES skills(skill_id),
    importance_weight DECIMAL(5,4), -- 0.0000 to 1.0000
    required_proficiency VARCHAR(50), -- beginner, intermediate, advanced, expert
    is_mandatory BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(job_id, skill_id)
);
```

### 5.3 Candidate Rankings Table
Job-specific candidate rankings.

```sql
CREATE TABLE candidate_rankings (
    ranking_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID NOT NULL REFERENCES jobs(job_id),
    application_id UUID NOT NULL REFERENCES applications(application_id),
    rank_position INTEGER,
    ranking_score DECIMAL(8,4),
    
    -- LLM reasoning for ranking
    llm_ranking_justification TEXT,
    llm_trade_off_analysis TEXT,
    llm_recommendation_summary TEXT,
    
    -- ML statistical measures
    ml_percentile DECIMAL(5,2), -- 0.00 to 100.00
    ml_z_score DECIMAL(8,4),
    ml_confidence_level DECIMAL(5,4),
    
    ranking_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ranking_algorithm VARCHAR(100),
    UNIQUE(job_id, application_id)
);
```

## 6. Reporting & Analytics Entities

### 6.1 Reports Table
Generated reports and documents.

```sql
CREATE TABLE reports (
    report_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_type VARCHAR(50), -- individual_candidate, comparative_analysis, executive_summary
    title VARCHAR(200),
    description TEXT,
    
    -- Related entities
    job_id UUID REFERENCES jobs(job_id),
    application_ids UUID[], -- Array of application IDs
    
    -- Report content
    llm_generated_content TEXT, -- Natural language report by LLM
    structured_data JSONB, -- Charts, tables, metrics
    
    -- File outputs
    pdf_file_path VARCHAR(500),
    html_file_path VARCHAR(500),
    docx_file_path VARCHAR(500),
    
    -- Generation metadata
    generated_by UUID, -- User who requested the report
    generation_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    llm_model_used VARCHAR(100),
    generation_time_seconds INTEGER
);
```

### 6.2 System Audit Logs Table
Comprehensive audit trail for compliance.

```sql
CREATE TABLE audit_logs (
    log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID,
    action_type VARCHAR(50), -- create, read, update, delete, process, export
    entity_type VARCHAR(50), -- candidate, resume, interview, evaluation
    entity_id UUID,
    ip_address INET,
    user_agent TEXT,
    session_id VARCHAR(100),
    action_details JSONB,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 7. System Configuration & Metadata

### 7.1 Processing Models Table
Track ML/LLM models and versions.

```sql
CREATE TABLE processing_models (
    model_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_name VARCHAR(100) NOT NULL,
    model_type VARCHAR(50), -- llm, ml, speech_recognition
    version VARCHAR(50),
    purpose VARCHAR(100), -- resume_parsing, interview_analysis, ranking
    model_path VARCHAR(500),
    configuration JSONB,
    performance_metrics JSONB,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 7.2 System Settings Table
Configuration parameters.

```sql
CREATE TABLE system_settings (
    setting_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    setting_key VARCHAR(100) UNIQUE NOT NULL,
    setting_value TEXT,
    setting_type VARCHAR(50), -- string, integer, boolean, json
    description TEXT,
    is_system_setting BOOLEAN DEFAULT FALSE,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 8. Indexes for Performance

```sql
-- Candidate lookups
CREATE INDEX idx_candidates_email ON candidates(email);
CREATE INDEX idx_candidates_status ON candidates(status);
CREATE INDEX idx_candidates_created_at ON candidates(created_at);

-- Application searches
CREATE INDEX idx_applications_candidate_job ON applications(candidate_id, job_id);
CREATE INDEX idx_applications_status ON applications(status);
CREATE INDEX idx_applications_job_id ON applications(job_id);

-- Resume processing
CREATE INDEX idx_resumes_application_id ON resumes(application_id);
CREATE INDEX idx_resumes_processing_status ON resumes(processing_status);

-- Skills matching
CREATE INDEX idx_candidate_skills_resume_skill ON candidate_skills(resume_id, skill_id);
CREATE INDEX idx_skills_category ON skills(category);
CREATE INDEX idx_skills_name_lower ON skills(LOWER(skill_name));

-- Interview processing
CREATE INDEX idx_interviews_application_id ON interviews(application_id);
CREATE INDEX idx_interviews_processing_status ON interviews(processing_status);

-- Evaluations and rankings
CREATE INDEX idx_candidate_evaluations_application ON candidate_evaluations(application_id);
CREATE INDEX idx_candidate_rankings_job_rank ON candidate_rankings(job_id, rank_position);
CREATE INDEX idx_candidate_rankings_score ON candidate_rankings(ranking_score DESC);

-- Audit and reporting
CREATE INDEX idx_audit_logs_timestamp ON audit_logs(timestamp DESC);
CREATE INDEX idx_audit_logs_user_action ON audit_logs(user_id, action_type);
```

## 9. Stored Procedures & Functions

### 9.1 Calculate Experience Years
```sql
CREATE OR REPLACE FUNCTION calculate_total_experience(p_resume_id UUID)
RETURNS DECIMAL(4,2) AS $$
DECLARE
    total_months INTEGER := 0;
BEGIN
    SELECT COALESCE(SUM(duration_months), 0)
    INTO total_months
    FROM work_experience 
    WHERE resume_id = p_resume_id;
    
    RETURN ROUND(total_months::DECIMAL / 12, 2);
END;
$$ LANGUAGE plpgsql;
```

### 9.2 Get Candidate Summary
```sql
CREATE OR REPLACE FUNCTION get_candidate_summary(p_application_id UUID)
RETURNS JSONB AS $$
DECLARE
    result JSONB;
BEGIN
    SELECT jsonb_build_object(
        'candidate_info', to_jsonb(c.*),
        'resume_summary', ra.executive_summary,
        'total_experience', ra.total_experience_years,
        'top_skills', COALESCE(
            (SELECT array_agg(s.skill_name ORDER BY cs.confidence_score DESC)
             FROM candidate_skills cs 
             JOIN skills s ON cs.skill_id = s.skill_id 
             WHERE cs.resume_id = r.resume_id 
             LIMIT 10), 
            '{}'::VARCHAR[]
        ),
        'interview_scores', COALESCE(ia.communication_score, 0),
        'final_evaluation', COALESCE(ce.final_score, 0)
    )
    INTO result
    FROM applications a
    JOIN candidates c ON a.candidate_id = c.candidate_id
    LEFT JOIN resumes r ON a.application_id = r.application_id
    LEFT JOIN resume_analysis ra ON r.resume_id = ra.resume_id
    LEFT JOIN interviews i ON a.application_id = i.application_id
    LEFT JOIN interview_analysis ia ON i.interview_id = ia.interview_id
    LEFT JOIN candidate_evaluations ce ON a.application_id = ce.application_id
    WHERE a.application_id = p_application_id;
    
    RETURN result;
END;
$$ LANGUAGE plpgsql;
```

## 10. Data Relationships Summary

### Primary Relationships:
- **Candidates → Applications (1:N)**: One candidate, multiple job applications
- **Jobs → Applications (1:N)**: One job, multiple applicants
- **Applications → Resumes (1:1)**: Each application has one resume
- **Applications → Interviews (1:N)**: Multiple interview rounds per application
- **Applications → Evaluations (1:1)**: One comprehensive evaluation per application

### Secondary Relationships:
- **Resumes → Work Experience/Education/Skills (1:N)**: Resume components
- **Interviews → Transcriptions/Analysis (1:1)**: Interview processing results
- **Jobs → Skill Requirements (1:N)**: Required skills per job
- **Skills → Candidate Skills (1:N)**: Skill instances across candidates

This data model supports all PRD requirements including:
- ✅ Multi-format resume processing and LLM analysis
- ✅ Video interview transcription and LLM content analysis
- ✅ LLM + ML hybrid evaluation and ranking
- ✅ Comprehensive reporting with LLM-generated insights
- ✅ Audit trails and compliance tracking
- ✅ Performance optimization through proper indexing
- ✅ Scalability for 1000+ candidates and 50+ concurrent users

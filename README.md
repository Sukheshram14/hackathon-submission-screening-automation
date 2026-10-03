# Hackathon Submission Screening Automation

An AI-assisted n8n workflow for automating the **initial technical screening of hackathon submissions**.

The workflow was built for the X'O CODE 2026 hackathon to process submitted PPT and Markdown/README files, extract technical evidence, analyze it against the assigned Problem Statement, and produce a structured initial screening result.

> **Note:** This repository contains the workflow definition only. Original participant submissions, private evaluation data, and production credentials are not included.

---

## Overview

The workflow separates the screening process into multiple stages instead of asking one LLM to directly judge a submission:

**Submission Intake → Validation → File Extraction → Problem Statement Analysis → Technical Content Extraction → Technical Analysis → Final Evaluation → Screening Sheet Update**

The main design principle is:

> **Extract what the team actually provided first, then analyze that evidence, then evaluate it.**

This helps prevent the final evaluator from receiving unsupported technical details that were not present in the team's submission.

---

## Workflow Architecture

```text
                    Google Sheets
                 Submission Records
                         │
                         ▼
                ┌──────────────────┐
                │  Process / Loop  │
                │   Pending Rows   │
                └────────┬─────────┘
                         │
                         ▼
                  PS VALIDATOR
                         │
                         ▼
                  PS SELECTION
                         │
                         ├──────────────► Problem Statement
                         │                  Analysis
                         │
                         ▼
                   File Handling
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
         PPT Submission       Markdown/README
              │                     │
              ▼                     ▼
       File Validation        File Validation
              │                     │
              ▼                     ▼
       PPT Extraction         MD Extraction
              │                     │
              └──────────┬──────────┘
                         ▼
                 OVERALL CONTENT
                         │
                         ▼
              Technical Content
                  Extraction
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
         PPT ANALYSIS          MD ANALYSIS
              │                     │
              └──────────┬──────────┘
                         ▼
                Technical Analysis
                         │
                         ▼
                  Final Evaluation
                         │
                         ▼
             Initial Screening Sheet
```

---

## What the Workflow Does

### 1. Submission Intake and Processing

The workflow reads submission records from Google Sheets and processes pending entries.

It includes:

- Google Sheets trigger/manual execution
- Batch processing using a loop
- Processed-status checking
- Problem Statement validation
- Problem Statement selection

This allows submissions already marked as processed to be skipped.

---

### 2. File Validation

Submitted files are downloaded from Google Drive and checked before extraction.

The workflow validates:

- File presence
- File extension
- MIME type

Supported file formats include:

- `.ppt`
- `.pptx`
- `.md`
- `.txt`

Invalid or missing files can be routed away from the technical-analysis pipeline.

---

### 3. Problem Statement Analysis

The **PS Analysis** LLM stage creates a technical evaluation blueprint from the assigned Problem Statement and its constraints.

It extracts:

- Core objective
- Inputs
- Outputs
- Critical technical mechanisms
- Explicit constraints
- Evaluation requirements
- Problem-specific technical questions
- Minimum credible solution path

The PS Analysis stage is instructed **not to score teams or prescribe technologies**.

---

### 4. PPT Content Extraction

For presentation submissions, the workflow:

1. Downloads the PPT file.
2. Sends the file to a document extraction API.
3. Requests structured extraction in spatial and Markdown formats.
4. Converts the extracted JSON response into usable workflow data.
5. Passes the extracted Markdown content to the analysis stages.

The workflow uses an external extraction endpoint for document parsing and dynamically supplies an API credential selected from the configured API collection.

---

### 5. Markdown / README Extraction

Markdown or text submissions are processed using n8n's file extraction functionality.

The extracted content is stored as `data_MD` and passed into the technical content extraction stage.

---

## Technical Content Extraction

The workflow has separate LLM extraction stages for PPT and Markdown content.

### PPT ANALYSIS

The PPT extraction engine is instructed to extract only technical information explicitly provided by the team.

### MD ANALYSIS

The Markdown extraction engine follows the same evidence-first approach for README/Markdown submissions.

The extraction schema covers:

- Technical approach
- Architecture
- Algorithm and core logic
- Data flow and processing
- Implementation
- AI/ML methodology
- Security and privacy
- Testing and validation
- Solution
- Features
- Technology stack
- Databases and APIs

The extraction stage **does not score the team**.

---

## Technical Analysis

The extracted PPT and Markdown content are passed to a separate **Technical Analysis** stage.

The analysis compares the team's explicit technical evidence against:

- Problem Statement
- Constraints
- PS Analysis

Important mechanisms are classified as:

| Status | Meaning |
|---|---|
| `STRONG` | Concrete technical implementation details are provided |
| `PARTIAL` | The mechanism is described but important technical details are missing |
| `MENTIONED_ONLY` | A technology or mechanism is named without meaningful technical explanation |

The analysis considers:

- Problem understanding
- Core technical mechanisms
- Architecture
- Algorithm/model/decision logic
- Data handling
- Failure and edge-case handling
- Domain-specific technical quality
- Feasibility
- Security/privacy where relevant
- Critical technical gaps
- Unsupported claims

Where possible, the technical flow is traced as:

```text
INPUT
  ↓
PROCESSING
  ↓
CORE LOGIC
  ↓
INFERENCE / ANALYSIS
  ↓
DECISION
  ↓
OUTPUT
```

The Technical Analysis stage does **not** assign the final score or recommendation.

---

## Final Evaluation

The **Final Evaluation** stage receives only:

1. Problem Statement
2. Constraints
3. PS Analysis
4. Technical Analysis

The evaluator is instructed to assess technical substance rather than presentation quality.

### Scoring

| Category | Maximum |
|---|---:|
| Problem Relevance & Understanding | 15 |
| Innovation & Differentiation | 15 |
| Technical Depth & Solution Design | 25 |
| Solution Completeness & Technical Credibility | 20 |
| Domain-Specific Technical Quality | 15 |
| Real-World Impact & Applicability | 10 |
| **Total** | **100** |

The workflow also defines the following calibration:

```text
90–100  Exceptional
80–89   Strong
70–79   Good / credible
60–69   Borderline
40–59   Weak
0–39    Not technically credible
```

Initial recommendation thresholds:

```text
70+     SHORTLIST
55–69   BORDERLINE
<55     REJECT
```

The evaluator is explicitly instructed not to reward:

- Buzzwords
- Technology-name dumping
- Repetition
- Generic architecture
- Unsupported accuracy claims
- Unsupported performance claims
- Unsupported scalability claims
- Unsupported security claims
- Unexplained technical mechanisms

---

## Structured Output

The workflow uses structured output parsers for the LLM stages.

The final evaluation produces structured fields including:

- Category scores
- Overall score
- Strengths
- Weaknesses
- Missing information
- Red flags
- Recommendation

The final score is also calculated in the Google Sheets update stage from the six category scores.

---

## Technology Stack

### Workflow Automation

- **n8n**
- n8n Code nodes
- n8n IF / Merge / Loop / Wait nodes
- n8n Structured Output Parser

### AI / LLM

- **Groq**
- `openai/gpt-oss-120b`

### Data and File Handling

- Google Sheets
- Google Drive
- PPT/PPTX extraction
- Markdown/text extraction
- External document extraction API

### Output

- Structured JSON
- Google Sheets initial screening records

---

## Key Engineering Design

### Separation of extraction and evaluation

A major design choice is separating:

```text
What did the team actually provide?
                 ↓
What technical mechanisms are supported by that evidence?
                 ↓
How should the submission be evaluated?
```

This prevents a single evaluation prompt from mixing extraction, interpretation, and scoring into one step.

### Evidence-first evaluation

The workflow instructs the analysis stages to avoid assumptions.

For example, merely mentioning a technology does not automatically establish that the technology was meaningfully implemented.

### Structured intermediate results

Each LLM stage uses structured output schemas so that information can be passed between stages consistently rather than relying entirely on free-form text.

### Automated screening record updates

After evaluation, the workflow updates the screening spreadsheet with the generated evaluation information and marks processing status.

---

## Repository Structure

```text
hackathon-submission-screening-automation/
│
├── README.md
│
├── workflow/
│   └── hackathon-screening-workflow.json
│
└── .gitignore
```

---

## Running the Workflow

This repository contains the exported n8n workflow. To run it, the required external services and credentials must be configured in the user's own n8n instance.

The workflow depends on configured integrations for:

- Google Sheets
- Google Drive
- Groq
- Document extraction API

Credentials should be configured through n8n's credential system rather than hard-coded into the workflow.

---

## Public Repository Safety

Before publishing an exported n8n workflow publicly:

- Remove API keys and access tokens.
- Remove OAuth secrets.
- Remove private submission links.
- Remove private participant information.
- Remove private Google Drive/Sheets identifiers where appropriate.
- Replace environment-specific credential references if necessary.
- Do not include original participant PPT/Markdown files.
- Do not include private screening results.

The repository should contain the **automation logic**, not confidential hackathon data.

---

## Limitations

This workflow is designed for **initial technical screening assistance**.

It does not establish that an LLM-generated evaluation is objectively correct, and it should not be treated as a replacement for final human review.

The workflow evaluates the technical evidence present in submitted material and therefore depends on the quality and completeness of that material.

---

## Project Context

This workflow was developed as part of the technical operations for **X'O CODE 2026**, where it was used to automate the initial screening of hackathon submissions.

The system was designed around a high-volume screening problem: converting submitted technical documents into structured evidence and then applying a consistent technical evaluation process.


# Generate BPMN from Natural Language Processing (NLP) — V2 on AWS

A web application that automatically generates **BPMN (Business Process Model and Notation)** diagrams from plain English text using NLP techniques. Users upload a `.txt` file describing a business process, and the system produces a downloadable `.bpmn` / `.xml` file ready to be opened in any BPMN-compatible tool.

---

## Live Demo (AWS Elastic Beanstalk)

```
http://flaskchulanlpbpmnv1-env.eba-hiwuhfgp.us-east-2.elasticbeanstalk.com/
```

---

## Project Structure

```
Generate-BPMN-by-NLP-V2-on-AWS/
│
├── application.py                  # Flask web application — entry point (routes: /, /download, /showxml)
├── main_nlp_generate_bpmn.py       # Core NLP-to-BPMN orchestrator (NLPGenerateBPMN class)
├── active_passive_sentence.py      # Active / passive sentence detection and conversion
├── convert_passive_to_active.py    # Passive-to-active sentence transformer
├── neural_coref.py                 # Coreference resolution via NeuralCoref
├── bpmn_lane.py                    # Builds BPMN lane (pool) XML structure
├── bpmn_diagram.py                 # Calculates BPMN shape coordinates and edge waypoints
├── requirements.txt                # Python dependencies
│
├── constants/
│   ├── word.py                     # Signal words, conjunction lists, negation words
│   └── dependency.py               # spaCy dependency label constants (PUNCT, etc.)
│
├── extract/
│   ├── merge_phrases.py            # Main SVO extraction pipeline (Phrases class)
│   ├── get_subject.py              # Subject token extraction
│   ├── get_verb.py                 # Verb token extraction
│   ├── get_object.py               # Object token extraction
│   ├── get_similarity.py           # Groups similar subjects (same role / lane)
│   ├── get_mark.py                 # Mark / signal-word detection
│   └── data/                       # Sample input text files for testing
│
├── matcher/
│   ├── get_subject_matcher.py      # spaCy Matcher rules for subjects
│   ├── get_object_matcher.py       # spaCy Matcher rules for objects
│   └── get_prep_pobj_matcher.py    # spaCy Matcher rules for prepositional objects
│
├── models/
│   └── json_model.py               # JsonDictionary helper (ordered dict wrapper)
│
├── templates/
│   ├── index.html                  # File upload page
│   └── download.html               # Result viewer & download page
│
├── static/                         # CSS stylesheets and icons
├── document/                       # Internal design documents
└── screenshot/                     # AWS pipeline / deployment screenshots
```

---

## Process Flowchart

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER (Web Browser)                          │
│                    Upload plain-English .txt file                   │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ HTTP POST /
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Flask (application.py)                       │
│  • Validate file extension (.txt only)                              │
│  • Read raw text from uploaded file                                 │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Step 1 — Coreference Resolution (neural_coref.py)      │
│  • Load spaCy en_core_web_sm model                                  │
│  • Apply NeuralCoref to replace pronouns with their antecedents     │
│    e.g. "John reviews it." → "John reviews the report."             │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ resolved text
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│         Step 2 — SVO Extraction (extract/merge_phrases.py)          │
│  • Merge noun chunks (Phrases.merge_phrases)                        │
│  • Detect sentence voice: Active / Passive (is_passive)             │
│  • Find main verbs (GetVerb.main_find_verbs)                        │
│  • Find subjects per verb (GetSubject.main_get_all_subs)            │
│  • Find objects per verb (GetObject.main_get_all_objs)              │
│  • Detect signal words → determine BPMN event tag:                  │
│      if / whether   →  ExclusiveGateway_                            │
│      simultaneously →  ParallelGateway_                             │
│      while / after  →  VO_ (sequential activity)                    │
│  • Output: list of SVO tuples                                       │
│    (seq, Subject, Verb, Object, EventTag, Voice, SignalWord)        │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ SVO list
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│        Step 3 — Subject Grouping & BPMN Direction                   │
│        (merge_phrases.py → get_new_svos)                            │
│  • Group semantically similar subjects (GetSimilarity)              │
│    → each unique group becomes one BPMN Lane (role/participant)     │
│  • Compute flow direction per SVO:                                  │
│      same subject as previous   →  BPMN-MOVE-NEXT (horizontal)     │
│      subject index increases    →  BPMN-MOVE-DOWN (lower lane)     │
│      subject index decreases    →  BPMN-MOVE-UP   (upper lane)     │
│  • Output: enriched SVO list with direction tag                     │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ enriched SVO list
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│       Step 4 — BPMN Element Construction                            │
│       (main_nlp_generate_bpmn.py — NLPGenerateBPMN)                │
│                                                                     │
│  For sequence == 1 (first SVO):                                     │
│    • Create StartEvent  (Es)                                        │
│    • Create first Activity  (Ac)                                    │
│    • If gateway signal → create Gateway Split  (Gxs / G+s)         │
│                                                                     │
│  For sequence > 1 (subsequent SVOs):                                │
│    • EventTag == "VO_"          → add Activity  (Ac)                │
│    • EventTag == "ExclusiveGateway_"                                │
│        signal in SIGNAL_WORDS  → open Gateway Split  (Gxs?)        │
│        signal in ANTONYM_WORDS → close Gateway Split,              │
│                                   pair branches,                    │
│                                   create Gateway Join  (Gxj)       │
│    • EventTag == "ParallelGateway_"                                 │
│        signal in SIGNAL_WORDS  → open Parallel Split  (G+s?)       │
│        signal in ANTONYM_WORDS → create Parallel Join  (G+j)       │
│                                                                     │
│  After all SVOs:                                                    │
│    • Append EndEvent  (Ee)                                          │
│    • Group activities by Subject → assign to Lanes                  │
│    • Output: list_flow, json_list_lane, list_svo_in_out             │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ flow data + lane data
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│          Step 5 — XML Generation                                    │
│          (bpmn_lane.py + bpmn_diagram.py)                           │
│                                                                     │
│  bpmn_lane.py (BPMNLane):                                           │
│    • Build <bpmn:collaboration> / <bpmn:participant> block          │
│    • Build <bpmn:laneSet> with one <bpmn:lane> per subject group    │
│    • Assign BPMN elements (flowNodeRef) to each lane                │
│                                                                     │
│  bpmn_diagram.py (BPMNDiagram):                                     │
│    • Compute (x, y) coordinates for every BPMN shape:              │
│        StartEvent, Activity, ExclusiveGateway,                      │
│        ParallelGateway, EndEvent                                    │
│    • Compute waypoints for every sequence flow edge:                │
│        Start→Activity, Activity→Activity,                           │
│        Activity→Gateway (split), Gateway→Activity,                  │
│        Activity→Gateway (join), Gateway→EndEvent                    │
│    • Render <bpmndi:BPMNDiagram> with shapes and edges              │
│                                                                     │
│  Final output files saved to data/:                                 │
│    • result_bpmn_process_from_nlp.bpmn                              │
│    • result_bpmn_process_from_nlp.xml                               │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ redirect to /download
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   USER — Download Result                            │
│  • View flow JSON, lane JSON, SVO list on /download page            │
│  • Download .bpmn file (opens in Camunda Modeler, etc.)             │
│  • View raw XML at /showxml                                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

| Component | Library / Version |
|---|---|
| Web framework | Flask 2.2.2 |
| NLP parser | spaCy 2.1.0 + en_core_web_sm |
| Coreference resolution | NeuralCoref 4.0 |
| XML generation | Python `xml.etree.ElementTree` |
| Cloud platform | AWS Elastic Beanstalk (Python 3.7) |
| CI/CD | AWS CodePipeline → GitHub (Version 2) |

---

## Local Setup

### Prerequisites

| Python version | Notes |
|---|---|
| 3.7.0 | **Recommended** — required for NeuralCoref compatibility |
| 3.9.1 | macOS M1 (universal2 installer) |

### Installation Steps

```bash
# 1. Create and activate virtual environment
python3 -m venv venv
source venv/bin/activate

# 2. Upgrade pip and build tools
pip install --upgrade pip
pip install -U pip setuptools wheel

# 3. Install dependencies
pip install spacy==2.1.0
python -m spacy download en_core_web_sm
pip install flask
pip install neuralcoref

# -- OR install all at once from requirements.txt --
pip3 install -r requirements.txt
```

### Run the App

```bash
export FLASK_APP="application.py"
flask run
```

Open `http://127.0.0.1:5000` in your browser.

---

## AWS Elastic Beanstalk Deployment

### 1. Create Environment

| Step | Action |
|---|---|
| 1 | Log in: https://aws.amazon.com/ |
| 2 | Go to Elastic Beanstalk → Create new environment |
| 3 | Application name: `flask-chula-nlp-bpmn-v1` |
| 4 | Environment name: `Flaskchulanlpbpmnv1-env` |
| 5 | Platform: Python 3.7 |
| 6 | Create sample application, then replace with project code |

### 2. CI/CD via AWS CodePipeline

| Step | Action |
|---|---|
| 1 | Create pipeline: `flask-chula-nlp-bpmn-v1` |
| 2 | Source: GitHub (Version 2) → authorize and select repo |
| 3 | Build stage: skip |
| 4 | Deploy provider: AWS Elastic Beanstalk |
| 5 | Region: US East (Ohio) / Application & environment as above |
| 6 | Create pipeline → deploy succeeds automatically on each push |

<img src="https://github.com/kullawattana/aws_flask_chula_nlp_bpmn_v1/blob/main/screenshot/code_pipeline_cicd.png" width="1024" height="430" />

<img src="https://github.com/kullawattana/aws_flask_chula_nlp_bpmn_v1/blob/main/screenshot/deployment.png" width="1024" height="430" />

---

## Git Workflow

```bash
git add .
git commit -m "your message"
git push origin main
```

---

## References

- [Python 3.7.0 download](https://www.python.org/downloads/release/python-370/)
- [GitHub .gitignore template for Python](https://github.com/github/gitignore/blob/main/Python.gitignore)
- [Flask on Elastic Beanstalk troubleshooting](https://stackoverflow.com/questions/53683358/flask-app-no-module-named-flask-error-in-elasticbeanstalk-after-confirming-flask)

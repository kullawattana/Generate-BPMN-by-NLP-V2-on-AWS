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

### 1. System Overview

```
 USER (Web Browser)
  │  Upload .txt file  ──►  HTTP POST /
  │
  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  application.py  ─  Flask Web App                                            │
│                                                                              │
│  upload_file()                                                               │
│  ├─ allowed_file()  ──  validate extension (.txt only)                       │
│  ├─ request.files.get('file')  ──  read raw text bytes                       │
│  └─ redirect → download_file() after processing                              │
│                                                                              │
│  download_file()                                                             │
│  └─ send_from_directory()  ──  stream .bpmn file to browser                  │
│                                                                              │
│  show_xml()                                                                  │
│  └─ send_from_directory()  ──  serve raw .xml at /showxml                    │
└──────────┬───────────────────────────────────────────────────────────────────┘
           │ raw text string
           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  neural_coref.py  ─  NeuralCoref (Step 1)                                    │
│  NeuralCoref.get_sentence()                                                  │
│  └─ nlp.add_pipe(NeuralCoref)  ──  attach coref component to spaCy pipeline  │
│  └─ doc._.coref_resolved  ──  replace all pronouns with their antecedents    │
└──────────┬───────────────────────────────────────────────────────────────────┘
           │ resolved text
           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  extract/merge_phrases.py  ─  Phrases  (Steps 2–3)                           │
│  prepare_list_flow() → nlp_process() → get_new_svos()                        │
└──────────┬───────────────────────────────────────────────────────────────────┘
           │ enriched SVO list  +  BPMN direction
           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  main_nlp_generate_bpmn.py  ─  NLPGenerateBPMN  (Step 4)                    │
│  prepare_list_flow()                                                         │
│  └─ nlp_process()  ──  build BPMN element lists                              │
│  └─ endEvent()     ──  append EndEvent                                       │
│  └─ startGroupSubjectForLane()  ──  group elements by Lane                   │
└──────────┬───────────────────────────────────────────────────────────────────┘
           │ list_flow + json_list_lane + list_svo_in_out
           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  bpmn_lane.py  +  bpmn_diagram.py  (Step 5)                                  │
│  BPMNLane  ──  build lane XML (<bpmn:laneSet>)                               │
│  BPMNDiagram.bpmn_edges()  ──  calculate shape coords + edge waypoints       │
└──────────┬───────────────────────────────────────────────────────────────────┘
           │ .bpmn / .xml saved to data/
           ▼
 USER downloads result via /download  ──  View JSON / Download BPMN / View XML
```

---

### 2. Step 1 — Coreference Resolution (`neural_coref.py`)

```
NeuralCoref.get_sentence(sentence)
│
├─ spacy.load('en_core_web_sm')
│   └─ build nlp pipeline
│
├─ neuralcoref.NeuralCoref(nlp.vocab)
│   └─ nlp.add_pipe(coref, name='neuralcoref')
│       └─ extend pipeline with coreference resolver
│
├─ doc = nlp(sentence)
│   └─ doc._.coref_resolved
│       ── replace each pronoun cluster with its main mention
│       e.g.  "The manager reviews it. He approves it."
│           → "The manager reviews the report. The manager approves the report."
│
└─ split resolved text back into sentences → join → return resolved string
```

---

### 3. Step 2 — SVO Extraction (`extract/merge_phrases.py` · `Phrases.get_svo()`)

```
Phrases.get_svo(sentence, is_show_gateway)
│
├─ [Tokenize]
│   ├─ merge_phrases()   ─ merge noun chunks into single tokens
│   └─ merge_phrases()   ─ called twice to handle nested NPs
│
├─ [Voice Detection]
│   └─ is_passive(doc)
│       └─ scan tokens for dep_ == "auxpass"
│           ├─ True  → passive sentence
│           └─ False → active sentence
│
├─ [Verb Discovery]  GetVerb.main_find_verbs(doc)
│   ├─ primary  : tokens where pos_=="VERB" AND dep_ not in {aux, auxpass}
│   └─ fallback : tokens where pos_=="VERB" OR pos_=="AUX"
│
└─ [For each VERB token]  ─────────────────────────────────────────┐
    │                                                               │
    ├─ sequence counter += 1                                        │
    │                                                               │
    ├─ [Subject]  GetSubject.main_get_all_subs(verb)               │
    │   ├─ direct  : tok.lefts where dep_ in SUBJECTS              │
    │   │             SUBJECTS = {nsubj, nsubjpass, csubj,         │
    │   │                         csubjpass, agent, expl}          │
    │   ├─ conj    : _get_subs_from_conjunctions()                 │
    │   │             ─ follow "and/or/but/nor" right children      │
    │   └─ fallback: _find_subs() ─ walk head chain to find nsubj  │
    │                                                               │
    ├─ [Compound Verb]  GetVerb.right_of_verb_is_conj_verb(verb)   │
    │   └─ check if verb.rights[0].pos_ == "CCONJ"                 │
    │       ├─ True  → (V1 CCONJ V2) → handle both verbs           │
    │       └─ False → single verb                                  │
    │                                                               │
    ├─ [Object]  GetObject.main_get_all_objs(verb, is_pas)         │
    │   ├─ direct  : tok.rights where dep_ in OBJECTS              │
    │   │             OBJECTS = {dobj, dative, attr, oprd, pobj}   │
    │   ├─ prep    : _get_objs_from_prepositions()                 │
    │   │             ─ verb → prep → pobj                          │
    │   ├─ xcomp   : _get_obj_from_xcomp()                         │
    │   │             ─ verb → xcomp(VERB) → dobj                  │
    │   └─ conj    : _get_objs_from_conjunctions()                 │
    │                ─ follow "and/or" right children of obj        │
    │                                                               │
    ├─ [Signal Word]  get_signal_word_with_verb(verb)              │
    │   ├─ scan verb.lefts for dep_ in {mark, advmod, prep}        │
    │   │   ├─ CONJUNCTION_OF_CONDITION  → ExclusiveGateway_       │
    │   │   │   {if, unless, else, whether, otherwise, …}          │
    │   │   ├─ CONJUNCTION_COMPARISON    → ExclusiveGateway_       │
    │   │   │   {not only, but also, either, neither, …}           │
    │   │   ├─ CONJUNCTION_OF_CHOICE     → ExclusiveGateway_       │
    │   │   │   {and, or, otherwise}                                │
    │   │   ├─ CONJUNCTION_OF_TIME       → VO_ (activity)          │
    │   │   │   {when, after, before, once, thereafter, …}         │
    │   │   └─ PARALLEL_WORDS            → ParallelGateway_        │
    │   │       {while, meanwhile, in parallel, concurrently, …}   │
    │   └─ scan verb.rights for dep_ in {cc, conj}                 │
    │       ├─ "or"  → ExclusiveGateway_                           │
    │       └─ "and" → VO_                                          │
    │                                                               │
    └─ append SVO tuple to self.svos:                              │
        (seq, Subject, Verb, Object,                               │
         EventTag, "isActive"/"isPassive", SignalWord)  ◄──────────┘
```

---

### 4. Step 3 — Subject Grouping & BPMN Direction (`Phrases.get_new_svos()`)

```
Phrases.get_new_svos()
│
├─ get_subject_collection()
│   └─ collect all non-"-" subjects from self.svos list
│
├─ get_duplicates_subject(subjects)
│   ├─ sort subjects and find those appearing more than once
│   └─ if no duplicates → treat all unique subjects as the list
│
├─ GetSimilarity.get_similarity(duplicates, new_subject_list)
│   └─ for each pair (dup_subject, new_subject):
│       ├─ doc1.similarity(doc2)   ─ spaCy word-vector cosine similarity
│       ├─ similarity > 0.60  → keep dup_subject  (same role/person)
│       └─ similarity ≤ 0.60  → keep new_subject  (different role)
│       Result: similar_list  ─ canonical subject per lane
│
├─ renew_svos_with_sentence(similar_list, sentence_list)
│   └─ rebuild SVO list using canonical subject names
│       Result: renew_list (ordered unique subjects = lane order)
│               new_svos   (updated SVO tuples)
│
└─ get_bpmn_direction_from_svos(new_svos, renew_list)
    └─ for each SVO:
        ├─ idx = renew_list.index(subject)
        ├─ idx > prev_idx  →  BPMN-MOVE-DOWN  (flow to lower lane)
        ├─ idx < prev_idx  →  BPMN-MOVE-UP    (flow to upper lane)
        └─ idx == prev_idx →  BPMN-MOVE-NEXT  (stay in same lane)
        Append direction as 8th element of each SVO tuple
```

---

### 5. Step 4 — BPMN Element Construction (`main_nlp_generate_bpmn.py`)

```
NLPGenerateBPMN.nlp_process(text, is_show_gateway)
│
├─ Phrases(sent).get_svo()   ─ get raw SVO list
├─ Phrases.get_new_svos()    ─ get enriched SVO list (with direction)
│
└─ for each svo in new_svos:
    │
    ├─ [sequence == 1]  ──  FIRST SENTENCE
    │   ├─ startActivity(flow_id)
    │   │   └─ create  StartEvent_{uuid}  →  append to list_svo + result_vo_subject
    │   ├─ createActivity(flow_id, vo)
    │   │   └─ create  Activity_{uuid}   →  append to list_svo + result_vo_subject
    │   └─ gatewayStartSplit(vo)
    │       ├─ EventTag == "ExclusiveGateway_"
    │       │   └─ create  ExclusiveGateway_{uuid} (Gxs?)
    │       └─ EventTag == "ParallelGateway_"
    │           └─ create  ParallelGateway_{uuid}  (G+s?)
    │
    └─ [sequence > 1]  ──  SUBSEQUENT SENTENCES
        │
        ├─ EventTag == "VO_"   ──────────────────────────────────────────────┐
        │   activityEvent(vo)                                                 │
        │   ├─ gateway_tag is empty  →  add Activity to list_svo             │
        │   └─ gateway_tag not empty →  add Activity to gateway_svo_list_split│
        │                               (branch buffer, not main flow yet)   │
        │                                                                    ─┘
        ├─ EventTag == "ExclusiveGateway_"  or  "ParallelGateway_"
        │   get_gateway(vo)
        │   │
        │   ├─ signal_word in SIGNAL_WORDS          ──  OPEN BRANCH
        │   │   {once, if, either, after, or, and, …}
        │   │   ├─ startGateway()
        │   │   │   └─ create  ExclusiveGateway_{uuid} / ParallelGateway_{uuid}
        │   │   │      clear list_svo_to_generate  (interrupt current flow)
        │   │   ├─ interruptFlowAdjustListSplitGateway()
        │   │   │   └─ rebuild pairs [Es,Ac | Ac,Gxs] from result_vo_subject
        │   │   └─ start_open_split_gateway(event_id)
        │   │       └─ push gateway_id into gateway_svo_list_split
        │   │
        │   └─ signal_word in ANTONYM_WORDS         ──  CLOSE BRANCH
        │       {but also, or, nor, otherwise, else, …}
        │       ├─ duplicateActivity(vo)
        │       │   └─ add final branch Activity to gateway_svo_list_split
        │       ├─ tuplePairsJoinGateway()
        │       │   └─ form pairs (branch1, branch2) from gateway_svo_list_split
        │       │      → push all branch pairs into list_svo
        │       ├─ joinGateway(open_join)
        │       │   └─ create  ExclusiveGateway_{uuid} / ParallelGateway_{uuid} (join)
        │       │      append join-gateway ID for each branch → list_svo
        │       └─ gateway_tag.clear()   ──  reset for next gateway
        │
        └─ [after all SVOs]
            ├─ endEvent()
            │   └─ create  EndEvent_{uuid}  →  append to list_svo + result_vo_subject
            └─ startGroupSubjectForLane()
                ├─ svo_merge_order_sequence()  ─ group activities by Subject key
                ├─ createLane()               ─ assign Lane_{uuid} per subject group
                └─ getDataLane()              ─ build list_flow / json_list_lane
                   Output ─────► list_flow, json_list_lane, list_lane, list_svo_in_out
```

---

### 6. Step 5 — XML Generation (`bpmn_lane.py` · `bpmn_diagram.py`)

```
── BPMNLane (bpmn_lane.py) ──────────────────────────────────────────────────
│
│  prepare_collaboration(collaboration_id, participant_id, event_name, process_ref_id)
│  └─ <bpmn:collaboration id="...">
│       └─ <bpmn:participant id="..." name="BPMN Process" processRef="..."/>
│
│  prepare_lane_set()
│  └─ <bpmn:laneSet id="LaneSet_{uuid}">
│       └─ for each (subject_group → Lane):
│           <bpmn:lane id="Lane_{uuid}" name="{subject}">
│             <bpmn:flowNodeRef> StartEvent / Activity / Gateway / EndEvent IDs
│           </bpmn:lane>
│
── BPMNDiagram (bpmn_diagram.py) ────────────────────────────────────────────
│
│  bpmn_edges(diagram_id, plan_id, uuid, collaboration_id)
│  │
│  ├─ _renew_list_bpmn_element()   ─ filter valid (source→target) pairs
│  │
│  └─ for each (flow_id, [source, target]) in res_group_by_event:
│      │
│      ├─ [Event → Activity]  startWithActivityFlow()
│      │   waypoint: (x1=196, y1=98) → (x2=250, y2=98)
│      │
│      ├─ [Activity → Activity]  — branch on BPMN direction pair:
│      │   ├─ NEXT→NEXT   activityWithActivityFlow()     x2 = x2+100+60
│      │   ├─ NEXT→DOWN   activityMoveDownWithActivityFlow()  y2 += 52+40
│      │   ├─ DOWN→NEXT   activityMoveNextActivityFlow()
│      │   ├─ NEXT→UP     activityMoveUpWithActivityFlow()    y2 -= 52+40
│      │   ├─ UP→NEXT     activityMoveFromUptoNextActivityFlow()
│      │   ├─ UP→UP       activityDoubleUp()             y1 -= 172
│      │   └─ DOWN→DOWN   activityDoubleDown()           y1 += 172
│      │
│      ├─ [Activity → End]   activityAndEndEventFlow()
│      │
│      ├─ [Activity → Gateway]  counter_split_gateway == 0
│      │   startWithGatewayFlow()   ─ begin split block
│      │   counter_split_gateway += 1
│      │
│      ├─ [Gateway → Activity]  counter_split_gateway == 1
│      │   gatewayWithSplitActivityFlow()   ─ first branch
│      │   counter_split_gateway += 1
│      │
│      ├─ [Gateway → Activity]  counter_split_gateway >= 2
│      │   gatewaySplitFlow()   ─ second branch (adds 3-point waypoint)
│      │   counter_split_gateway = -1 ; counter_join_gateway += 1
│      │
│      ├─ [Activity → Gateway]  counter_join_gateway == 1
│      │   gatewayWithJoinActivityFlow()   ─ first branch rejoins
│      │   counter_join_gateway += 1
│      │
│      ├─ [Activity → Gateway]  counter_join_gateway >= 2
│      │   gatewayJoinFlow()   ─ second branch rejoins (3-point waypoint)
│      │   counter_split_gateway = 0 ; counter_join_gateway = 0
│      │
│      ├─ [Gateway → Activity]  counter_split_gateway == 0
│      │   joinGatewayWithActivityFlow()   ─ exit join gateway, resume flow
│      │
│      └─ [Gateway → End]  counter_split_gateway == 0
│          joinGatewayWithEndFlow()
│
│  _bpmn_shape(members_list)  ─ calculate (x, y) for each shape
│  ├─ StartEvent   : startActivity()          x=160, y=80
│  ├─ Activity #1  : startWithfirstActivity() x+=90,  y-=22
│  ├─ Activity #N  : gotoOthersActivity()     x+=160
│  ├─ Gateway      : gotoEvent()              x+=155, y=73
│  └─ EndEvent     : gotoEnd()               y=80
│
│  get_more_role_shape()  ─ stack lane shapes vertically
│  ├─ Lane 1 : y = position_lane_y (20)
│  ├─ Lane 2 : y = 20 + shape_lane_h (175)
│  └─ Lane N : y = previous_lane_y + shape_lane_h
│
└─ Output: <bpmndi:BPMNDiagram>
             <bpmndi:BPMNPlane>
               <bpmndi:BPMNEdge>  (sequence flows with waypoints)
               <bpmndi:BPMNShape> (elements with dc:Bounds)
               <bpmndi:BPMNShape> (lanes with dc:Bounds)
             </bpmndi:BPMNPlane>
           </bpmndi:BPMNDiagram>
```

---

### 7. Signal Word → BPMN Element Mapping (`constants/word.py`)

| Category | Words / Phrases | BPMN Element |
|---|---|---|
| Condition (if-branch) | `if`, `unless`, `else`, `whether`, `otherwise`, `in case`, `whereas`, `optionally`, `for the case` | `ExclusiveGateway_` (split) |
| Comparison | `not only`, `but also`, `either`, `neither`, `also` | `ExclusiveGateway_` (split) |
| Choice | `and`, `or`, `otherwise` | `ExclusiveGateway_` (split) |
| Time sequence | `when`, `after`, `before`, `once`, `since`, `thereafter`, `upon`, `in addition`, `immediately after` | `VO_` (Activity, sequential) |
| Parallel | `while`, `meanwhile`, `in parallel`, `concurrently`, `meantime`, `in the meantime` | `ParallelGateway_` (split) |
| Negation (close branch) | `but also`, `or`, `nor`, `otherwise`, `else`, `in the latter case` | close Gateway → join |

---

### 8. SVO Tuple Structure (internal data format)

```
Index  Field           Example value           Description
─────  ─────────────── ──────────────────────  ────────────────────────────────
 [0]   sequence        2                       sentence order (1 = first)
 [1]   subject         "manager"               canonical subject (lane name)
 [2]   verb            "review"                lemma of main verb
 [3]   object          "report"                direct / prepositional object
 [4]   event_tag       "ExclusiveGateway_"     BPMN element type prefix
 [5]   voice           "isActive"              "isActive" | "isPassive"
 [6]   signal_word     "if"                    conjunction that triggered tag
 [7]   direction       "BPMN-MOVE-DOWN"        lane movement for edge routing
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

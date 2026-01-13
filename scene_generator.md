# CharacterBox Scene Crafting Pipeline

A three-stage pipeline for extracting and generating high-quality role-playing scenes from literary works, based on the [CharacterBox paper](https://arxiv.org/abs/2412.05631) (NAACL 2025).

**Now with intelligent format detection for plays vs. novels!**

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Architecture](#architecture)
- [The Three-Stage Pipeline](#the-three-stage-pipeline)
- [Format Detection & Segmentation](#format-detection--segmentation)
- [Input/Output Formats](#inputoutput-formats)
- [Configuration](#configuration)
- [How It Works (Deep Dive)](#how-it-works-deep-dive)
- [Extending the Pipeline](#extending-the-pipeline)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Overview

This pipeline transforms raw literary texts into structured scenes suitable for evaluating LLM role-playing capabilities. It implements the "Scene Crafting" phase from CharacterBox, which the original paper describes but doesn't provide code for.

### What It Does

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   Raw Book      │      │  Three-Stage    │      │   Structured    │
│   Text          │  ──► │  Pipeline       │  ──► │   Scenes        │
│   (Gutenberg)   │      │  (LLM-powered)  │      │   (JSON)        │
└─────────────────┘      └─────────────────┘      └─────────────────┘
```

### Key Features

| Feature | Description |
|---------|-------------|
| **Format Detection** | Automatically identifies plays vs. novels and applies appropriate segmentation |
| **Extraction** | Pulls scenes from existing passages in books |
| **Generation** | Creates original scenes set in the book's world |
| **Quality Control** | Three-stage refinement with scoring and rejection |
| **Incremental Saves** | Progress saved after each book (crash-safe) |
| **Configurable** | Adjust scenes per book, model, verbosity |

### File Structure

```
├── scene_crafter.py     # Main pipeline script
├── passage_segmenter.py    # Format detection + intelligent segmentation
├── README.md               # This file
└── books.json              # Your input (book texts)
```

---

## Quick Start

### 1. Install Dependencies

```bash
pip install openai
```

### 2. Prepare Input

Save your book texts as JSON (keys are Gutenberg book IDs):

```python
import json

# Your dictionary of {book_id: full_text}
found = {
    1342: "The Project Gutenberg eBook of Pride and Prejudice...",
    1661: "The Project Gutenberg eBook of Sherlock Holmes...",
    844: "The Project Gutenberg eBook of The Importance of Being Earnest...",
    # ...
}

with open('books.json', 'w', encoding='utf-8') as f:
    json.dump(found, f, ensure_ascii=False)
```

### 3. Run the Pipeline

```bash
# Basic usage
python scene_crafter.py \
    --api-key YOUR_FIREWORKS_API_KEY \
    --input books.json \
    --output scenes.json

# With options
python scene_crafter.py \
    --api-key $FIREWORKS_KEY \
    --input books.json \
    --output scenes.json \
    -n 5 \        # 5 extracted scenes per book
    -g 5 \        # 5 generated scenes per book
    --verbose     # Detailed logging

# Background execution (keeps running after terminal closes)
nohup python scene_crafter.py \
    --api-key $FIREWORKS_KEY \
    --input books.json \
    --output scenes.json \
    > run.log 2>&1 &

# Monitor progress
tail -f scene_crafter_*.log
```

### 4. CLI Reference

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--api-key` | | Required | Fireworks API key |
| `--input` | `-i` | Required | Input JSON file with book texts |
| `--output` | `-o` | `scenes.json` | Output JSON file |
| `--log` | `-l` | Auto | Log file path |
| `--model` | `-m` | `kimi-k2-instruct` | Model ID |
| `--n-extracted` | `-n` | 5 | Extracted scenes per book |
| `--n-generated` | `-g` | 5 | Generated scenes per book |
| `--books` | | All | Specific book IDs to process |
| `--verbose` | `-v` | Off | Debug logging |

---

## Architecture

### System Overview

```
┌────────────────────────────────────────────────────────────────────────┐
│                              main()                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │ Parse Args   │→ │ Load Books   │→ │ Init Client  │                  │
│  └──────────────┘  └──────────────┘  └──────────────┘                  │
│                                              │                          │
│                    ┌─────────────────────────▼─────────────────────┐   │
│                    │           For each book:                      │   │
│                    │  ┌─────────────────────────────────────────┐  │   │
│                    │  │     craft_scenes_for_book()             │  │   │
│                    │  │                                         │  │   │
│                    │  │  1. Detect format (play vs novel)       │  │   │
│                    │  │  2. Select best passages                │  │   │
│                    │  │  3. For each passage: craft_scene()     │  │   │
│                    │  │  4. Generate original: craft_scene()    │  │   │
│                    │  │  5. Save progress                       │  │   │
│                    │  └─────────────────────────────────────────┘  │   │
│                    └───────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### Class Structure

```
scene_crafter.py
│
├── Data Classes
│   ├── Character          # Name, role, physical/psychological state, position
│   ├── Environment        # Time, location, description
│   └── Scene              # Full scene + scores + format metadata
│
├── LLMClient              # OpenAI-compatible API wrapper
│   ├── call()             # Raw LLM call with retries
│   └── call_json()        # Call + JSON parsing
│
├── SceneCrafter           # Main pipeline orchestrator
│   ├── _get_format()              # Detect play vs novel
│   ├── _select_passages()         # Get best passages for format
│   ├── screenwriter_extract()     # Stage 1: Extract (format-aware)
│   ├── screenwriter_generate()    # Stage 1: Generate (format-aware)
│   ├── director_refine()          # Stage 2: Enhance scene
│   ├── evaluator_assess()         # Stage 3: Score and filter
│   ├── craft_scene()              # Full 3-stage pipeline
│   └── craft_scenes_for_book()    # Batch processing
│
└── Utilities
    ├── load_input()       # Load books.json
    └── save_output()      # Save scenes.json

passage_segmenter.py
│
├── FormatDetector         # Detect if text is play, novel, or unknown
│   ├── PLAY_PATTERNS      # Act/Scene, CHARACTER:, stage directions
│   ├── NOVEL_PATTERNS     # Chapter, narrative prose, paragraphs
│   └── detect()           # Returns format + confidence
│
├── PlaySegmenter          # Segment plays into scenes
│   ├── _split_by_scenes()         # Split on Act/Scene markers
│   ├── _split_by_dialogue_blocks()# Split by character speeches
│   └── _score_passage()           # Score for dramatic potential
│
├── NovelSegmenter         # Segment novels into passages
│   ├── _split_by_chapters()       # Split on Chapter markers
│   ├── _split_long_chapter()      # Handle oversized chapters
│   ├── _split_by_paragraphs()     # Fallback splitting
│   └── _score_passage()           # Score for scene potential
│
├── PassageSegmenter       # Unified interface
│   ├── segment()          # Auto-detect and segment
│   ├── select_best()      # Get top N passages
│   └── analyze()          # Return stats and diagnostics
│
└── Convenience Functions  # Drop-in replacements
    ├── segment_into_passages()
    ├── select_best_passages()
    └── score_passage()
```

---

## The Three-Stage Pipeline

Based on Section 3.1 of the CharacterBox paper, scenes go through three refinement stages:

### Stage 1: Screenwriter

**Role**: Extract or generate a raw scene structure.

```
Input:  Raw passage from book (or nothing for generation)
Output: Basic scene JSON with environment + characters
```

**What it does**:
- Identifies 2-4 characters in the passage
- Extracts time and location
- Captures basic character states and relationships

**Format-Aware Prompts**:
- **Novels**: Looks for quoted dialogue, narrative descriptions
- **Plays**: Understands `CHARACTER.` dialogue format, stage directions in `[brackets]`

### Stage 2: Director

**Role**: Refine and enhance the raw scene.

```
Input:  Raw scene from Stage 1
Output: Enhanced scene with conflict, hooks, richer details
```

**What it does**:
- Adds central conflict/tension
- Enriches character motivations
- Improves spatial positioning
- Adds narrative hooks for role-play

### Stage 3: Evaluator

**Role**: Quality gate - score and accept/reject.

```
Input:  Refined scene from Stage 2
Output: Scores (1-5) + ACCEPT/REJECT decision
```

**Scoring Criteria** (from CharacterBox paper):

| Criterion | What It Measures |
|-----------|------------------|
| **Creativity** | Dramatic potential, uniqueness, role-play opportunities |
| **Coherence** | Internal consistency, logical behaviors, sensible arrangement |
| **Conformity** | Matches source tone/style, faithful characters, accurate setting |
| **Detail** | Sufficient for immersion, well-defined states, enough context |

**Acceptance Logic**:
```python
passes = (average_score >= 3.5) or (recommendation == "ACCEPT")
```

### Pipeline Flow

```
                         ┌─────────────┐
        passage ────────►│ SCREENWRITER│────────► raw_scene
                         │ (format-    │
                         │  aware)     │
                         └─────────────┘
                                │
                                ▼
                         ┌─────────────┐
        raw_scene ──────►│  DIRECTOR   │────────► refined_scene
                         └─────────────┘
                                │
                                ▼
                         ┌─────────────┐
        refined_scene ──►│  EVALUATOR  │────────► score + decision
                         └─────────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
                 ACCEPT                  REJECT
                    │                       │
                    ▼                       ▼
              Return Scene            Retry (max 2x)
                                     or Return None
```

---

## Format Detection & Segmentation

The pipeline automatically detects whether input text is a **play** or **novel** and applies appropriate segmentation strategies.

### Format Detection

```
┌─────────────────────────────────────────────────────────────┐
│                    FormatDetector                           │
│  - Samples text from beginning, middle, end                 │
│  - Counts pattern matches for play vs novel indicators      │
│  - Returns: PLAY, NOVEL, or UNKNOWN + confidence score      │
└─────────────────────┬───────────────────────────────────────┘
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│  PlaySegmenter  │     │ NovelSegmenter  │
└─────────────────┘     └─────────────────┘
```

### Detection Patterns

| Indicator | Play Pattern | Novel Pattern |
|-----------|--------------|---------------|
| **Structure** | `ACT II, SCENE 3` | `CHAPTER 5`, `Chapter V` |
| **Dialogue** | `HAMLET. To be or not to be` | `"To be," said John` |
| **Directions** | `[Enter GHOST]`, `(aside)` | Narrative prose |
| **Markers** | `DRAMATIS PERSONAE` | Paragraph indentation |
| **Speech** | `CHARACTER:` or `CHARACTER.` | `he said`, `she replied` |

### Play Segmentation Strategy

```
Full Play Text
      │
      ▼
┌─────────────────────────────────────┐
│  1. Split by Act/Scene markers      │
│     - ACT I, SCENE 1                │
│     - Act II. Scene 3               │
│     - SCENE IV                      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  2. If scene > max_chars:           │
│     Split by dialogue blocks        │
│     (groups of CHARACTER. lines)    │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  3. Score each passage:             │
│     - Speaker variety (unique chars)│
│     - Dialogue density              │
│     - Stage directions count        │
│     - Dramatic words (!?!)          │
└─────────────────────────────────────┘
```

**Play Scoring Factors**:

| Factor | How It's Measured | Max Points |
|--------|-------------------|------------|
| Speaker Variety | Unique `CHARACTER.` markers | 3.0 |
| Dialogue Density | Speeches per 1K chars | 3.0 |
| Stage Directions | `[brackets]` and `(parens)` | 1.5 |
| Dramatic Words | love, death, betray, etc. | 2.0 |
| Exclamations | `!` and `?` count | 1.5 |

### Novel Segmentation Strategy

```
Full Novel Text
      │
      ▼
┌─────────────────────────────────────┐
│  1. Split by Chapter markers        │
│     - CHAPTER I                     │
│     - Chapter 1                     │
│     - BOOK TWO                      │
│     - PART III                      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  2. If chapter > max_chars:         │
│     Try scene breaks (* * *)        │
│     Then paragraph splitting        │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  3. Score each passage:             │
│     - Dialogue density (quotes)     │
│     - Character mentions (names)    │
│     - Action verbs                  │
│     - Emotional content             │
│     - Scene-setting words           │
└─────────────────────────────────────┘
```

**Novel Scoring Factors**:

| Factor | How It's Measured | Max Points |
|--------|-------------------|------------|
| Dialogue Density | `"quotes"` per 1K chars | 3.0 |
| Character Mentions | Capitalized names | 2.0 |
| Action Verbs | said, walked, looked, etc. | 2.5 |
| Emotional Content | love, fear, angry, etc. | 1.5 |
| Scene-Setting | room, morning, entered, etc. | 1.0 |

### Testing the Segmenter

```bash
# Analyze a text file
python passage_segmenter.py hamlet.txt --analyze

# Output:
# ============================================================
# Format Detection
# ============================================================
#   Detected format: PLAY
#   Confidence: 0.85
#
# ============================================================
# Segmentation Results
# ============================================================
#   Total passages: 23
#   Length range: 1842-5621 chars
#   Average length: 3204 chars
#   Score range: 4.21-9.87
#
#   Top sections:
#     - ACT III SCENE 1
#     - ACT I SCENE 5
#     - ACT V SCENE 2
#     ...

# Get top 5 passages
python passage_segmenter.py pride_and_prejudice.txt --top 5
```

---

## Input/Output Formats

### Input: books.json

```json
{
  "1342": "The Project Gutenberg eBook of Pride and Prejudice, by Jane Austen...",
  "1661": "The Project Gutenberg eBook of The Adventures of Sherlock Holmes...",
  "844": "The Project Gutenberg eBook of The Importance of Being Earnest...",
  "1524": "The Project Gutenberg eBook of Hamlet, by William Shakespeare..."
}
```

**Notes**:
- Keys are Gutenberg book IDs (as strings or integers)
- Values are full book text
- Must match IDs in `BOOK_METADATA` dictionary (or add new entries)

### Output: scenes.json

```json
[
  {
    "source_title": "Pride and Prejudice",
    "source_id": 1342,
    "scene_type": "extracted",
    "source_format": "novel",
    "section_name": "Chapter 34 (part 1)",
    "environment": {
      "time": "Late morning, autumn",
      "location": "The drawing room at Longbourn estate",
      "description": "Pale sunlight filters through tall Georgian windows, casting long shadows across the worn Persian rug. The fire crackles low in the hearth, and the faint scent of dried lavender lingers in the air."
    },
    "characters": [
      {
        "name": "Elizabeth Bennet",
        "role": "Second eldest Bennet daughter, protagonist",
        "physical_state": "Seated upright on the settee, hands folded in her lap",
        "psychological_state": "Guarded yet curious, masking her interest behind polite indifference",
        "position": "Near the fireplace, facing both the window and the door",
        "relationships": {
          "Mr. Darcy": "Complex mixture of attraction and resentment"
        }
      },
      {
        "name": "Mr. Fitzwilliam Darcy",
        "role": "Wealthy gentleman from Derbyshire",
        "physical_state": "Standing stiffly by the mantelpiece",
        "psychological_state": "Internally conflicted between pride and growing admiration",
        "position": "Opposite Elizabeth, maintaining formal distance",
        "relationships": {
          "Elizabeth Bennet": "Reluctant fascination he struggles to suppress"
        }
      }
    ],
    "quality_scores": {
      "creativity": 4,
      "coherence": 5,
      "conformity": 5,
      "detail": 4,
      "average": 4.5,
      "justifications": {
        "creativity": "Strong dramatic tension with clear stakes",
        "coherence": "All elements logically consistent",
        "conformity": "Perfectly captures Austen's drawing room dynamics",
        "detail": "Rich sensory details support immersion"
      },
      "suggestions": []
    }
  },
  {
    "source_title": "Hamlet",
    "source_id": 1524,
    "scene_type": "extracted",
    "source_format": "play",
    "section_name": "ACT III SCENE 1",
    "environment": {
      "time": "Afternoon, within the castle",
      "location": "A lobby in Elsinore Castle",
      "description": "A dim corridor with stone walls and tapestries. Claudius and Polonius hide behind an arras, while Ophelia waits with a book, staged to appear reading."
    },
    "characters": [
      {
        "name": "Hamlet",
        "role": "Prince of Denmark, protagonist",
        "physical_state": "Pacing slowly, disheveled appearance suggesting inner turmoil",
        "psychological_state": "Deeply contemplative, wrestling with existence itself",
        "position": "Center stage, unaware of the hidden observers",
        "relationships": {
          "Ophelia": "Former love now tainted by suspicion and grief"
        }
      },
      {
        "name": "Ophelia",
        "role": "Daughter of Polonius, Hamlet's former love",
        "physical_state": "Seated demurely, holding a prayer book",
        "psychological_state": "Torn between obedience to father and love for Hamlet",
        "position": "Downstage right, positioned to intercept Hamlet",
        "relationships": {
          "Hamlet": "Confused love mixed with fear at his changed behavior"
        }
      }
    ],
    "quality_scores": {
      "creativity": 5,
      "coherence": 5,
      "conformity": 5,
      "detail": 5,
      "average": 5.0
    }
  }
]
```

### Supported Books (Default)

#### Novels
| ID | Title | Author | Genre |
|----|-------|--------|-------|
| 1342 | Pride and Prejudice | Jane Austen | social_drama |
| 1661 | The Adventures of Sherlock Holmes | Arthur Conan Doyle | mystery |
| 345 | Dracula | Bram Stoker | horror |
| 1184 | The Count of Monte Cristo | Alexandre Dumas | intrigue |
| 1257 | The Three Musketeers | Alexandre Dumas | adventure |
| 84 | Frankenstein | Mary Shelley | sci_fi_horror |
| 11 | Alice's Adventures in Wonderland | Lewis Carroll | fantasy |
| 120 | Treasure Island | Robert Louis Stevenson | adventure |
| 98 | A Tale of Two Cities | Charles Dickens | historical_drama |

#### Plays
| ID | Title | Author | Genre |
|----|-------|--------|-------|
| 844 | The Importance of Being Earnest | Oscar Wilde | comedy |
| 1524 | Hamlet | William Shakespeare | tragedy |
| 1533 | Macbeth | William Shakespeare | tragedy |
| 1531 | Romeo and Juliet | William Shakespeare | tragedy |
| 1521 | A Midsummer Night's Dream | William Shakespeare | comedy |
| 2270 | An Ideal Husband | Oscar Wilde | comedy |
| 4085 | A Doll's House | Henrik Ibsen | drama |

---

## Configuration

### Adding New Books

Edit the `BOOK_METADATA` dictionary in `scene_crafter.py`:

```python
BOOK_METADATA = {
    # Existing books...
    
    # Add a novel
    16328: {
        "title": "Beowulf",
        "author": "Anonymous",
        "genre": "epic_poetry",
        "format": "novel"  # or omit - will auto-detect
    },
    
    # Add a play
    2267: {
        "title": "The Tempest",
        "author": "William Shakespeare",
        "genre": "comedy",
        "format": "play"
    },
}
```

### Changing the Model

```bash
# Use a different Fireworks model
python scene_crafter.py \
    --api-key $KEY \
    --input books.json \
    --model accounts/fireworks/models/llama-v3p1-70b-instruct
```

### Adjusting Quality Threshold

In `evaluator_assess()`, modify the acceptance logic:

```python
# Current: average >= 3.5
passes = scores["average"] >= 3.5 or evaluation.get("overall_recommendation") == "ACCEPT"

# Stricter: require 4.0 average
passes = scores["average"] >= 4.0

# More lenient: require 3.0 average
passes = scores["average"] >= 3.0
```

### Adjusting Passage Size

In `scene_crafter.py`:

```python
# In SceneCrafter.__init__
if USE_NEW_SEGMENTER:
    self.segmenter = PassageSegmenter(
        min_chars=1500,  # Shorter passages (default: 2000)
        max_chars=8000   # Longer passages (default: 6000)
    )
```

---

## How It Works (Deep Dive)

### Complete Data Flow

```
books.json
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ For each book_id, book_text:                                    │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │ 1. Format Detection                                     │  │
│   │    FormatDetector.detect(text) → PLAY or NOVEL         │  │
│   └────────────────────────┬────────────────────────────────┘  │
│                            │                                    │
│              ┌─────────────┴─────────────┐                     │
│              ▼                           ▼                      │
│   ┌─────────────────┐         ┌─────────────────┐              │
│   │ PlaySegmenter   │         │ NovelSegmenter  │              │
│   │ - Act/Scene     │         │ - Chapter       │              │
│   │ - Dialogue      │         │ - Paragraph     │              │
│   └────────┬────────┘         └────────┬────────┘              │
│            └─────────────┬─────────────┘                       │
│                          ▼                                      │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │ 2. Passage Selection                                    │  │
│   │    Score + rank → top N passages                        │  │
│   └────────────────────────┬────────────────────────────────┘  │
│                            │                                    │
│   ┌────────────────────────▼────────────────────────────────┐  │
│   │ 3. For each passage: craft_scene()                      │  │
│   │                                                         │  │
│   │   Stage 1: screenwriter_extract(passage, format)        │  │
│   │            → raw_scene                                  │  │
│   │                                                         │  │
│   │   Stage 2: director_refine(raw_scene)                   │  │
│   │            → refined_scene                              │  │
│   │                                                         │  │
│   │   Stage 3: evaluator_assess(refined_scene)              │  │
│   │            → scores + ACCEPT/REJECT                     │  │
│   │                                                         │  │
│   │   If ACCEPT: return Scene(...)                          │  │
│   │   If REJECT: retry once with refined as input           │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │ 4. Generate original scenes: craft_scene(None, format)  │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   ──► Save progress to scenes.json                             │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
scenes.json (list of Scene objects with format metadata)
```

### Retry Logic

The pipeline has multiple retry mechanisms:

```python
# LLM API retries (in LLMClient.call)
for attempt in range(3):
    try:
        response = api_call()
    except:
        time.sleep(2 ** attempt)  # 1s, 2s, 4s backoff

# Pipeline stage retries (in craft_scene)
for attempt in range(2):
    refined = director_refine(raw)
    passes, scores = evaluator_assess(refined)
    if passes:
        return Scene(...)
    else:
        raw = refined  # Feed refined version back

# Book-level retries (in craft_scenes_for_book)
passages = select_best_passages(text, n=n_extracted * 2)  # Get 2x needed
for passage in passages:
    if extracted >= n_extracted:
        break
    # Some will fail, but we have extras
```

### JSON Parsing Robustness

LLMs often wrap JSON in markdown code blocks:

```python
def call_json(self, prompt):
    response = self.call(prompt)
    
    # Handle: ```json\n{...}\n``` or ```\n{...}\n```
    json_match = re.search(r'```(?:json)?\s*([\s\S]*?)\s*```', response)
    json_str = json_match.group(1) if json_match else response
    
    return json.loads(json_str.strip())
```

---

## Extending the Pipeline

### Adding a New Segmenter for Poetry

```python
# In passage_segmenter.py

class PoetrySegmenter:
    """Segment poetry by stanzas or cantos."""
    
    CANTO_PATTERN = re.compile(r'\n\s*(CANTO\s+[IVXLCDM\d]+)', re.IGNORECASE)
    STANZA_BREAK = re.compile(r'\n\s*\n')  # Blank lines between stanzas
    
    @classmethod
    def segment(cls, text: str, min_chars: int, max_chars: int) -> list[Passage]:
        # Group stanzas into passage-sized chunks
        ...
    
    @classmethod
    def _score_passage(cls, text: str) -> float:
        # Score by: imagery, dialogue, narrative action
        ...
```

### Adding a New Stage (Dramaturg)

```python
# In scene_crafter.py

DRAMATURG_ENHANCE = '''You are a dramaturg enhancing dialogue for "{title}".

SCENE:
{scene}

Improve the dialogue potential by:
1. Adding subtext and tension to relationships
2. Creating opportunities for revealing character through speech
3. Identifying moments of dramatic irony

Respond with JSON...'''

class SceneCrafter:
    def dramaturg_enhance(self, scene: dict, book_id: int) -> dict:
        meta = BOOK_METADATA[book_id]
        prompt = DRAMATURG_ENHANCE.format(
            title=meta["title"],
            scene=json.dumps(scene, indent=2)
        )
        return self.llm.call_json(prompt)
    
    def craft_scene(self, passage, book_id, scene_type, source_format, section_name):
        # Stage 1
        raw = self.screenwriter_extract(passage, book_id, source_format)
        
        # Stage 2
        refined = self.director_refine(raw, book_id)
        
        # NEW Stage 2.5
        enhanced = self.dramaturg_enhance(refined, book_id)
        
        # Stage 3
        passes, scores = self.evaluator_assess(enhanced, book_id)
        # ...
```

### Custom Scoring for Specific Genres

```python
# In passage_segmenter.py

class MysteryNovelSegmenter(NovelSegmenter):
    """Custom segmenter optimized for mystery novels."""
    
    @classmethod
    def _score_passage(cls, text: str) -> float:
        # Start with base novel score
        score = super()._score_passage(text)
        
        # Add mystery-specific bonuses
        mystery_words = ['clue', 'suspect', 'murder', 'detective', 'evidence', 'alibi']
        score += sum(text.lower().count(w) for w in mystery_words) * 0.5
        
        # Bonus for interrogation scenes
        if 'asked' in text.lower() and '?' in text:
            score += 2.0
        
        # Bonus for revelation moments
        revelation_words = ['realized', 'discovered', 'revealed', 'truth']
        score += sum(text.lower().count(w) for w in revelation_words) * 0.3
        
        return score
```

### Parallel Processing

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def craft_all_scenes_parallel(found: dict, crafter: SceneCrafter, max_workers: int = 3) -> list:
    """Process multiple books in parallel."""
    all_scenes = []
    
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {
            executor.submit(
                crafter.craft_scenes_for_book, 
                text, 
                book_id
            ): book_id 
            for book_id, text in found.items()
        }
        
        for future in as_completed(futures):
            book_id = futures[future]
            try:
                scenes = future.result()
                all_scenes.extend(scenes)
                print(f"Completed book {book_id}: {len(scenes)} scenes")
            except Exception as e:
                print(f"Book {book_id} failed: {e}")
    
    return all_scenes
```

**Note**: Be mindful of API rate limits when parallelizing.

---

## Troubleshooting

### Common Issues

#### "Skipping book X: no metadata"

**Cause**: Book ID not in `BOOK_METADATA` dictionary.

**Fix**: Add the book's metadata:
```python
BOOK_METADATA[YOUR_BOOK_ID] = {
    "title": "Book Title",
    "author": "Author Name", 
    "genre": "genre_tag",
    "format": "novel"  # or "play"
}
```

#### Play detected as novel (or vice versa)

**Cause**: Text doesn't have strong format indicators.

**Fix**: Override format in metadata:
```python
BOOK_METADATA[844] = {
    "title": "The Importance of Being Earnest",
    "author": "Oscar Wilde",
    "genre": "comedy",
    "format": "play"  # Force play segmentation
}
```

Or check detection confidence:
```bash
python passage_segmenter.py your_text.txt --analyze
# Look at confidence score - if < 0.5, consider forcing format
```

#### JSON Parse Errors

**Cause**: LLM returned malformed JSON or extra text.

**Symptoms**:
```
json.decoder.JSONDecodeError: Expecting property name enclosed in double quotes
```

**Fixes**:
1. Lower temperature:
   ```python
   return self.llm.call_json(prompt, temperature=0.5)
   ```

2. Stronger JSON instructions in prompts:
   ```python
   PROMPT = '''...
   IMPORTANT: Respond with ONLY valid JSON. 
   No markdown, no explanation, no code blocks.
   Start with { and end with }
   '''
   ```

#### Low Quality Scores

**Diagnosis**:
```bash
python scene_crafter.py --verbose -i books.json --api-key $KEY
```

**Fixes**:
1. Get more passage candidates:
   ```python
   passages = self._select_passages(book_text, book_id, n=n_extracted * 3)
   ```

2. Lower threshold temporarily:
   ```python
   passes = scores["average"] >= 3.0
   ```

3. Check if format detection is correct - wrong format = wrong prompts.

#### Rate Limiting

**Symptoms**:
```
openai.RateLimitError: Rate limit exceeded
```

**Fix**: Add delay between calls:
```python
class LLMClient:
    def __init__(self, ...):
        self.last_call = 0
        self.min_interval = 1.0  # seconds
    
    def call(self, ...):
        elapsed = time.time() - self.last_call
        if elapsed < self.min_interval:
            time.sleep(self.min_interval - elapsed)
        self.last_call = time.time()
        # ... rest of call
```

---

## Cost Estimation

With Kimi K2 on Fireworks (~$0.80/M tokens):

| Component | Tokens/Scene | Cost/Scene |
|-----------|--------------|------------|
| Screenwriter | ~2K in, ~500 out | ~$0.002 |
| Director | ~1K in, ~800 out | ~$0.0015 |
| Evaluator | ~1K in, ~300 out | ~$0.001 |
| **Total** | ~5-6K tokens | **~$0.005** |

**For 100 scenes**: ~$0.50 (excluding retries)  
**With retries**: ~$1-2

---

## References

- **CharacterBox Paper**: [arXiv:2412.05631](https://arxiv.org/abs/2412.05631)
- **CharacterBox Repo**: [github.com/Paitesanshi/CharacterBox](https://github.com/Paitesanshi/CharacterBox)
- **Project Gutenberg**: [gutenberg.org](https://www.gutenberg.org/)
- **Fireworks AI**: [fireworks.ai](https://fireworks.ai/)

---

## Contributing

1. Fork the repository
2. Create a feature branch
3. Add tests for new functionality
4. Submit a pull request

### Areas for Improvement

- [ ] Poetry/verse segmentation
- [ ] Genre-specific scoring (mystery, romance, etc.)
- [ ] Support for non-English texts
- [ ] Caching to avoid re-processing
- [ ] Web UI for scene review/editing
- [ ] Integration with CharacterBox evaluation pipeline
- [ ] Better handling of epistolary novels (letters)
- [ ] Support for screenplay format (distinct from stage plays)

---

## License

MIT License - See LICENSE file.

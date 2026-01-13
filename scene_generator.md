# Scene Crafting Pipeline

A three-stage pipeline for extracting and generating high-quality role-playing scenes from literary works, based on the [CharacterBox paper](https://arxiv.org/abs/2412.05631) (NAACL 2025).

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Architecture](#architecture)
- [The Three-Stage Pipeline](#the-three-stage-pipeline)
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

- **Extraction**: Pulls scenes from existing passages in books
- **Generation**: Creates original scenes set in the book's world
- **Quality Control**: Three-stage refinement with scoring and rejection
- **Incremental Saves**: Progress saved after each book (crash-safe)
- **Configurable**: Adjust scenes per book, model, verbosity

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

# Background execution
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
│                    │  │  1. select_best_passages(text)          │  │   │
│                    │  │  2. For each passage: craft_scene()     │  │   │
│                    │  │  3. Generate original: craft_scene()    │  │   │
│                    │  │  4. Save progress                       │  │   │
│                    │  └─────────────────────────────────────────┘  │   │
│                    └───────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### Class Structure

```
scene_crafter.py
│
├── Data Classes
│   ├── Character      # Name, role, physical/psychological state, position
│   ├── Environment    # Time, location, description
│   └── Scene          # Full scene with characters + environment + scores
│
├── LLMClient          # OpenAI-compatible API wrapper
│   ├── call()         # Raw LLM call with retries
│   └── call_json()    # Call + JSON parsing
│
├── SceneCrafter       # Main pipeline orchestrator
│   ├── screenwriter_extract()   # Stage 1: Extract from passage
│   ├── screenwriter_generate()  # Stage 1: Generate original
│   ├── director_refine()        # Stage 2: Enhance scene
│   ├── evaluator_assess()       # Stage 3: Score and filter
│   ├── craft_scene()            # Full 3-stage pipeline
│   └── craft_scenes_for_book()  # Batch processing
│
└── Utilities
    ├── segment_into_passages()  # Split book into chunks
    ├── score_passage()          # Heuristic passage ranking
    ├── select_best_passages()   # Get top N passages
    ├── load_input()             # Load books.json
    └── save_output()            # Save scenes.json
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

**Prompt Strategy**:
```
"You are an expert screenwriter extracting a scene from {title} by {author}..."
```

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

**Prompt Strategy**:
```
"You are a director refining a scene... Enhance with:
1. Clearer central conflict
2. Richer character details
3. Better spatial positioning
4. Narrative hooks"
```

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

## Input/Output Formats

### Input: books.json

```json
{
  "1342": "The Project Gutenberg eBook of Pride and Prejudice, by Jane Austen...",
  "1661": "The Project Gutenberg eBook of The Adventures of Sherlock Holmes...",
  "345": "The Project Gutenberg eBook of Dracula, by Bram Stoker..."
}
```

**Notes**:
- Keys are Gutenberg book IDs (as strings or integers)
- Values are full book text
- Must match IDs in `BOOK_METADATA` dictionary

### Output: scenes.json

```json
[
  {
    "source_title": "Pride and Prejudice",
    "source_id": 1342,
    "scene_type": "extracted",
    "environment": {
      "time": "Late morning, autumn",
      "location": "The drawing room at Longbourn estate",
      "description": "Pale sunlight filters through tall Georgian windows, casting long shadows across the worn Persian rug. The fire crackles low in the hearth, and the faint scent of dried lavender lingers in the air. Outside, a light rain patters against the glass."
    },
    "characters": [
      {
        "name": "Elizabeth Bennet",
        "role": "Second eldest Bennet daughter, protagonist",
        "physical_state": "Seated upright on the settee, hands folded in her lap, wearing a simple muslin day dress",
        "psychological_state": "Guarded yet curious, masking her interest behind a veneer of polite indifference",
        "position": "Near the fireplace, angled to face both the window and the door",
        "relationships": {
          "Mr. Darcy": "Complex mixture of attraction and resentment",
          "Jane Bennet": "Beloved elder sister and confidante"
        }
      },
      {
        "name": "Mr. Fitzwilliam Darcy",
        "role": "Wealthy gentleman from Derbyshire",
        "physical_state": "Standing stiffly by the mantelpiece, one hand resting on the marble",
        "psychological_state": "Internally conflicted, struggling between pride and growing admiration",
        "position": "Opposite Elizabeth, maintaining formal distance",
        "relationships": {
          "Elizabeth Bennet": "Reluctant fascination he struggles to suppress",
          "Mr. Bingley": "Close friend whose happiness he feels responsible for"
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
        "creativity": "Strong dramatic tension between characters with clear stakes",
        "coherence": "All elements logically consistent with established characters",
        "conformity": "Perfectly captures Austen's drawing room dynamics",
        "detail": "Rich sensory details support immersion"
      },
      "suggestions": []
    }
  }
]
```

### Supported Books (Default)

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
| 844 | The Importance of Being Earnest | Oscar Wilde | comedy |
| 98 | A Tale of Two Cities | Charles Dickens | historical_drama |

---

## Configuration

### Adding New Books

Edit the `BOOK_METADATA` dictionary:

```python
BOOK_METADATA = {
    # Existing books...
    
    # Add new book
    16328: {
        "title": "Beowulf",
        "author": "Anonymous",
        "genre": "epic_poetry"
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

---

## How It Works (Deep Dive)

### Passage Selection Algorithm

The pipeline doesn't process entire books—it selects the most promising passages:

```python
def segment_into_passages(text, min_chars=2000, max_chars=6000):
    """
    1. Try splitting by chapter markers (CHAPTER I, Chapter 1, etc.)
    2. If not enough segments, fall back to paragraph splitting
    3. Target: passages of 2000-6000 characters
    """

def score_passage(passage):
    """
    Heuristic scoring based on:
    - Dialogue density (quotation marks)
    - Named character mentions (capitalized words)
    - Action verbs (said, walked, looked, etc.)
    
    Higher score = more likely to contain a good scene
    """

def select_best_passages(text, n=10):
    """
    1. Segment book into passages
    2. Score each passage
    3. Return top N by score
    """
```

**Why this matters**: Books contain exposition, description, and transitions that don't make good scenes. This heuristic finds dialogue-heavy, action-rich sections.

### Retry Logic

The pipeline has multiple retry mechanisms:

```python
# LLM API retries (in LLMClient.call)
for attempt in range(3):
    try:
        response = api_call()
    except:
        time.sleep(2 ** attempt)  # 1s, 2s, 4s

# Pipeline stage retries (in craft_scene)
for attempt in range(2):
    refined = director_refine(raw)
    passes, scores = evaluator_assess(refined)
    if passes:
        return Scene(...)
    else:
        raw = refined  # Feed refined version back for another try

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

### Adding a New Stage

Example: Add a "Dramaturg" stage between Director and Evaluator:

```python
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
    
    def craft_scene(self, passage, book_id, scene_type):
        # Stage 1
        raw = self.screenwriter_extract(passage, book_id)
        
        # Stage 2
        refined = self.director_refine(raw, book_id)
        
        # NEW Stage 2.5
        enhanced = self.dramaturg_enhance(refined, book_id)
        
        # Stage 3
        passes, scores = self.evaluator_assess(enhanced, book_id)
        # ...
```

### Custom Passage Scoring

Add domain-specific scoring for your use case:

```python
def score_passage_for_mystery(passage: str) -> float:
    """Custom scorer for mystery novels."""
    score = 0.0
    
    # Base score
    score += score_passage(passage)
    
    # Mystery-specific bonuses
    mystery_words = ['clue', 'suspect', 'murder', 'detective', 'evidence', 'alibi']
    score += sum(passage.lower().count(w) for w in mystery_words) * 0.5
    
    # Bonus for interrogation scenes
    if 'asked' in passage.lower() and '?' in passage:
        score += 2.0
    
    return score
```

### Different Output Formats

Convert to CharacterBox's expected format:

```python
def to_characterbox_format(scene: Scene) -> dict:
    """Convert to CharacterBox evaluation format."""
    return {
        "E": {  # Environment
            "time": scene.environment.time,
            "location": scene.environment.location,
            "description": scene.environment.description,
        },
        "C": [  # Characters
            {
                "name": c.name,
                "role": c.role,
                "state": {
                    "physical": c.physical_state,
                    "psychological": c.psychological_state,
                },
                "position": c.position,
                "relationships": c.relationships,
            }
            for c in scene.characters
        ],
        "metadata": {
            "source": scene.source_title,
            "type": scene.scene_type,
        }
    }
```

### Parallel Processing

For faster execution with many books:

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def craft_all_scenes_parallel(found: dict, max_workers: int = 3) -> list:
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
                logger.info(f"Completed book {book_id}: {len(scenes)} scenes")
            except Exception as e:
                logger.error(f"Book {book_id} failed: {e}")
    
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
    "genre": "genre_tag"
}
```

#### JSON Parse Errors

**Cause**: LLM returned malformed JSON or extra text.

**Symptoms**:
```
json.decoder.JSONDecodeError: Expecting property name enclosed in double quotes
```

**Fixes**:
1. Lower temperature for more consistent output:
   ```python
   return self.llm.call_json(prompt, temperature=0.5)
   ```

2. Add more explicit JSON instructions to prompts:
   ```python
   PROMPT = '''...
   
   IMPORTANT: Respond with ONLY valid JSON. No markdown, no explanation, no code blocks.
   Start your response with { and end with }
   '''
   ```

3. Add fallback parsing:
   ```python
   def call_json(self, prompt):
       response = self.call(prompt)
       
       # Try multiple extraction strategies
       strategies = [
           lambda r: json.loads(r),  # Direct parse
           lambda r: json.loads(re.search(r'```json?\s*(.*?)\s*```', r, re.DOTALL).group(1)),
           lambda r: json.loads(re.search(r'\{.*\}', r, re.DOTALL).group()),
       ]
       
       for strategy in strategies:
           try:
               return strategy(response)
           except:
               continue
       
       raise ValueError(f"Could not parse JSON from: {response[:200]}")
   ```

#### Rate Limiting

**Cause**: Too many API calls too quickly.

**Symptoms**:
```
openai.RateLimitError: Rate limit exceeded
```

**Fixes**:
1. Increase retry delay:
   ```python
   time.sleep(2 ** attempt + random.uniform(0, 1))
   ```

2. Add global rate limiting:
   ```python
   import time
   
   class LLMClient:
       def __init__(self, ...):
           self.last_call = 0
           self.min_interval = 0.5  # seconds between calls
       
       def call(self, ...):
           elapsed = time.time() - self.last_call
           if elapsed < self.min_interval:
               time.sleep(self.min_interval - elapsed)
           
           self.last_call = time.time()
           # ... rest of call
   ```

#### Low Quality Scores

**Cause**: Passages don't contain good scene material, or prompts need tuning.

**Diagnosis**:
```bash
# Run with verbose logging
python scene_crafter.py --verbose -i books.json --api-key $KEY
```

**Fixes**:
1. Adjust passage selection parameters:
   ```python
   passages = select_best_passages(text, n=n_extracted * 3)  # Get more candidates
   ```

2. Lower acceptance threshold temporarily:
   ```python
   passes = scores["average"] >= 3.0  # Instead of 3.5
   ```

3. Improve prompts with more specific guidance for the genre.

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

## License

MIT License - See LICENSE file.

---

## Contributing

1. Fork the repository
2. Create a feature branch
3. Add tests for new functionality
4. Submit a pull request

**Areas for improvement**:
- [ ] Better passage segmentation for plays vs. novels
- [ ] Genre-specific prompts and scoring
- [ ] Support for non-English texts
- [ ] Caching to avoid re-processing
- [ ] Web UI for scene review/editing
- [ ] Integration with CharacterBox evaluation pipeline

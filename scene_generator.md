# CharacterBox Scene Crafting Pipeline

A three-stage pipeline for extracting and generating high-quality role-playing scenes from literary works, based on the [CharacterBox paper](https://arxiv.org/abs/2412.05631) (NAACL 2025).

**Features:**
- Intelligent format detection for plays vs. novels
- 16 thematic styles (cyberpunk, steampunk, noir, etc.) for scene variation
- Three-stage quality pipeline: Screenwriter → Director → Evaluator

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Architecture](#architecture)
- [The Three-Stage Pipeline](#the-three-stage-pipeline)
- [Thematic Styles](#thematic-styles)
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
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
              Original Style         Thematic Variations
              (faithful)             (cyberpunk, noir, etc.)
```

### Key Features

| Feature | Description |
|---------|-------------|
| **Format Detection** | Automatically identifies plays vs. novels and applies appropriate segmentation |
| **Thematic Styles** | 16 literary styles for scene variation (cyberpunk, steampunk, noir, wuxia, etc.) |
| **Extraction** | Pulls scenes from existing passages in books |
| **Generation** | Creates original scenes set in the book's world |
| **Quality Control** | Three-stage refinement with scoring and rejection |
| **Incremental Saves** | Progress saved after each book (crash-safe) |
| **Configurable** | Adjust scenes per book, model, verbosity, theming |

### File Structure

```
├── scene_crafter.py        # Main pipeline script (with thematic variations)
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
}

with open('books.json', 'w', encoding='utf-8') as f:
    json.dump(found, f, ensure_ascii=False)
```

### 3. Run the Pipeline

```bash
# Basic usage - generated scenes get random thematic styles
python scene_crafter.py \
    --api-key YOUR_FIREWORKS_API_KEY \
    --input books.json \
    --output scenes.json

# Also apply themes to 50% of extracted scenes
python scene_crafter.py \
    --api-key $FIREWORKS_KEY \
    --input books.json \
    --theme-extracted

# Custom settings
python scene_crafter.py \
    --api-key $FIREWORKS_KEY \
    --input books.json \
    --output scenes.json \
    -n 5 \                        # 5 extracted scenes per book
    -g 5 \                        # 5 generated scenes per book
    --theme-extracted \           # Theme some extracted scenes too
    --theme-extracted-ratio 0.8 \ # 80% of extracted get themed
    --seed 42 \                   # Reproducible style selection
    --verbose

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
| `--theme-extracted` | | Off | Apply themes to extracted scenes too |
| `--theme-extracted-ratio` | | 0.5 | Fraction of extracted scenes to theme |
| `--seed` | | None | Random seed for reproducible style selection |

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
│                    │  │  2. Select thematic styles for scenes   │  │   │
│                    │  │  3. Select best passages                │  │   │
│                    │  │  4. For each passage: craft_scene()     │  │   │
│                    │  │  5. Generate original: craft_scene()    │  │   │
│                    │  │  6. Save progress                       │  │   │
│                    │  └─────────────────────────────────────────┘  │   │
│                    └───────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### Class Structure

```
scene_crafter.py
│
├── THEMATIC_STYLES           # 16 style definitions with prompts
├── GENRE_STYLE_AFFINITIES    # Which styles work well with which genres
│
├── Data Classes
│   ├── Character             # Name, role, physical/psychological state
│   ├── Environment           # Time, location, description
│   └── Scene                 # Full scene + scores + style metadata
│
├── ThematicStyleSelector     # Picks styles based on genre affinity
│   ├── select_style()        # Pick one style (weighted by genre)
│   ├── select_multiple_unique() # Pick N unique styles
│   └── get_style_info()      # Get full style details
│
├── LLMClient                 # OpenAI-compatible API wrapper
│   ├── call()                # Raw LLM call with retries
│   └── call_json()           # Call + JSON parsing
│
├── SceneCrafter              # Main pipeline orchestrator
│   ├── screenwriter_extract()    # Stage 1 (original or themed)
│   ├── screenwriter_generate()   # Stage 1 (always themed)
│   ├── director_refine()         # Stage 2 (style-aware)
│   ├── evaluator_assess()        # Stage 3 (style-aware scoring)
│   ├── craft_scene()             # Full 3-stage pipeline
│   └── craft_scenes_for_book()   # Batch with style distribution
│
└── passage_segmenter.py (imported)
    ├── FormatDetector        # Detect play vs novel
    ├── PlaySegmenter         # Act/Scene splitting
    └── NovelSegmenter        # Chapter/paragraph splitting
```

---

## The Three-Stage Pipeline

Based on Section 3.1 of the CharacterBox paper, scenes go through three refinement stages:

### Stage 1: Screenwriter

**Role**: Extract or generate a raw scene structure.

```
Input:  Raw passage + optional thematic style
Output: Basic scene JSON with environment + characters (styled if specified)
```

**What it does**:
- Identifies 2-4 characters in the passage
- Extracts time and location
- Captures character states and relationships
- If themed: Transforms setting/aesthetics to match style

### Stage 2: Director

**Role**: Refine and enhance the raw scene.

```
Input:  Raw scene from Stage 1 + style info
Output: Enhanced scene with conflict, hooks, richer details
```

**What it does**:
- Adds central conflict/tension (fitting the thematic style)
- Enriches character motivations
- Improves spatial positioning with style-appropriate details
- Adds narrative hooks for role-play

### Stage 3: Evaluator

**Role**: Quality gate - score and accept/reject.

```
Input:  Refined scene from Stage 2
Output: Scores (1-5) + ACCEPT/REJECT decision
```

**Scoring Criteria**:

| Criterion | What It Measures |
|-----------|------------------|
| **Creativity** | Dramatic potential, uniqueness, effective style adaptation |
| **Coherence** | Internal consistency, logical behaviors |
| **Conformity** | Maintains spirit of source while fitting thematic style |
| **Detail** | Sufficient for immersion, style-appropriate details |

**Acceptance**: Average ≥ 3.5 or explicit ACCEPT recommendation

---

## Thematic Styles

The pipeline includes 16 thematic styles to create diverse scene variations:

### Available Styles

| Style | Description | Setting Hints |
|-------|-------------|---------------|
| **cyberpunk** | High-tech dystopia, megacorps, hackers | Neon streets, corporate towers, AR overlays |
| **steampunk** | Victorian + steam tech, clockwork | Brass machinery, airship docks, inventor workshops |
| **noir** | 1940s crime drama, moral ambiguity | Smoky jazz clubs, rain-soaked streets, shadows |
| **gothic_horror** | Dark romanticism, supernatural dread | Crumbling mansions, misty moors, candlelit crypts |
| **solarpunk** | Optimistic eco-future, sustainability | Vertical gardens, solar communities, green cities |
| **dieselpunk** | 1920s-40s aesthetic, pulp adventure | Art deco towers, zeppelin docks, speakeasies |
| **wuxia** | Chinese martial arts fantasy | Misty mountains, bamboo forests, martial schools |
| **space_opera** | Galactic empires, epic adventures | Starship bridges, alien cantinas, space stations |
| **weird_west** | Frontier + supernatural horror | Dusty towns, haunted mines, cursed lands |
| **biopunk** | Biotech, genetic engineering | Gene clinics, organic architecture, body-mod parlors |
| **mythic** | Ancient gods, epic quests | Sacred groves, divine palaces, enchanted seas |
| **post_apocalyptic** | Civilization collapsed, survival | Ruined cities, fortified settlements, wastelands |
| **dark_academia** | Elite schools, obsessive knowledge | Ivy halls, candlelit libraries, secret societies |
| **afrofuturism** | African diaspora + advanced tech | Advanced African nations, ancestral digital realms |
| **cosmic_horror** | Lovecraftian existential dread | Non-Euclidean architecture, forgotten temples |
| **pastoral** | Idealized rural life, simple pleasures | Thatched cottages, flowering meadows, cozy kitchens |

### Genre-Style Affinities

Styles are weighted toward genres that fit well together:

| Source Genre | Preferred Styles |
|--------------|------------------|
| **social_drama** | noir, dark_academia, pastoral, steampunk, cyberpunk |
| **mystery** | noir, gothic_horror, cyberpunk, steampunk, cosmic_horror |
| **horror** | gothic_horror, cosmic_horror, biopunk, post_apocalyptic, weird_west |
| **intrigue** | noir, cyberpunk, steampunk, dieselpunk, space_opera |
| **adventure** | steampunk, dieselpunk, space_opera, wuxia, weird_west, mythic |
| **sci_fi_horror** | biopunk, cosmic_horror, cyberpunk, post_apocalyptic |
| **fantasy** | mythic, steampunk, wuxia, solarpunk, gothic_horror |
| **comedy** | steampunk, pastoral, dark_academia, space_opera, dieselpunk |
| **historical_drama** | dieselpunk, noir, steampunk, gothic_horror, dark_academia |
| **tragedy** | gothic_horror, noir, cosmic_horror, mythic, dark_academia |

### How Theming Works

**Generated scenes** (default behavior):
- Each generated scene gets a randomly selected unique style
- Styles are picked from the affinity list 70% of the time
- Example: 5 generated scenes might get: cyberpunk, noir, steampunk, wuxia, gothic_horror

**Extracted scenes** (with `--theme-extracted`):
- A portion (default 50%) get thematic reimagining
- The rest stay faithful to the original work
- Example: 3 of 5 extracted scenes stay original, 2 get themed

### Example Output

```json
{
  "source_title": "Pride and Prejudice",
  "source_id": 1342,
  "scene_type": "generated",
  "source_format": "novel",
  "thematic_style": "cyberpunk",
  "style_description": "Cyberpunk",
  "environment": {
    "time": "2am, neon-drenched night",
    "location": "The Bennet family's cramped apartment in Neo-London's mid-tier residential block",
    "description": "Holographic advertisements flicker outside rain-streaked windows. The family's outdated AR setup casts blue light across tense faces. Mrs. Bennet's voice competes with the hum of the building's aging life support systems."
  },
  "characters": [
    {
      "name": "Elizabeth 'Liz' Bennet",
      "role": "Underground data analyst, second daughter",
      "physical_state": "Neural interface port visible at temple, wearing a faded corporate jumpsuit repurposed as casual wear",
      "psychological_state": "Defiant yet calculating, masking vulnerability behind sharp wit and encrypted thoughts",
      "position": "Leaning against the window, half-watching the street below",
      "relationships": {
        "Darcy": "Suspicious of his corporate connections despite unexpected attraction"
      }
    },
    {
      "name": "William Darcy",
      "role": "Executive at Pemberley Dynamics megacorp",
      "physical_state": "Immaculate corporate attire, subtle cybernetic enhancements barely visible",
      "psychological_state": "Conflicted between corporate loyalty and growing fascination with the lower-block analyst",
      "position": "Standing stiffly by the door, clearly uncomfortable in the modest surroundings",
      "relationships": {
        "Elizabeth": "Reluctant respect masked by class prejudice protocols"
      }
    }
  ],
  "quality_scores": {
    "creativity": 5,
    "coherence": 4,
    "conformity": 4,
    "detail": 5,
    "average": 4.5
  }
}
```

---

## Format Detection & Segmentation

The pipeline automatically detects whether input text is a **play** or **novel** and applies appropriate segmentation strategies.

### Format Detection

| Indicator | Play Pattern | Novel Pattern |
|-----------|--------------|---------------|
| **Structure** | `ACT II, SCENE 3` | `CHAPTER 5`, `Chapter V` |
| **Dialogue** | `HAMLET. To be or not to be` | `"To be," said John` |
| **Directions** | `[Enter GHOST]`, `(aside)` | Narrative prose |
| **Markers** | `DRAMATIS PERSONAE` | Paragraph indentation |

### Play Segmentation

1. Split by Act/Scene markers
2. If scene > max_chars: split by dialogue blocks
3. Score by: speaker variety, dialogue density, stage directions

### Novel Segmentation

1. Split by Chapter markers
2. If chapter > max_chars: split by scene breaks (`* * *`) or paragraphs
3. Score by: dialogue density, character mentions, action verbs

### Testing the Segmenter

```bash
python passage_segmenter.py hamlet.txt --analyze
python passage_segmenter.py pride_and_prejudice.txt --top 5
```

---

## Input/Output Formats

### Input: books.json

```json
{
  "1342": "The Project Gutenberg eBook of Pride and Prejudice...",
  "1661": "The Project Gutenberg eBook of Sherlock Holmes...",
  "844": "The Project Gutenberg eBook of The Importance of Being Earnest..."
}
```

### Output: scenes.json

```json
[
  {
    "source_title": "Pride and Prejudice",
    "source_id": 1342,
    "scene_type": "extracted",
    "source_format": "novel",
    "section_name": "Chapter 34 (part 1)",
    "thematic_style": null,
    "style_description": null,
    "environment": { ... },
    "characters": [ ... ],
    "quality_scores": { ... }
  },
  {
    "source_title": "Pride and Prejudice",
    "source_id": 1342,
    "scene_type": "generated",
    "source_format": "novel",
    "section_name": null,
    "thematic_style": "steampunk",
    "style_description": "Steampunk",
    "environment": { ... },
    "characters": [ ... ],
    "quality_scores": { ... }
  }
]
```

### Output Location

| File | Location | Set By |
|------|----------|--------|
| **Scenes** | `./scenes.json` | `--output` / `-o` |
| **Log file** | `./scene_crafter_YYYYMMDD_HHMMSS.log` | `--log` / `-l` |

Files are saved to the current working directory by default. Use absolute paths to control location:

```bash
python scene_crafter.py \
    --api-key $KEY \
    --input /path/to/books.json \
    --output /path/to/scenes.json \
    --log /path/to/run.log
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

```python
BOOK_METADATA = {
    # Add a novel
    16328: {
        "title": "Beowulf",
        "author": "Anonymous",
        "genre": "adventure",  # Affects style selection
        "format": "novel"
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

### Adding New Thematic Styles

```python
THEMATIC_STYLES["art_nouveau"] = {
    "name": "Art Nouveau",
    "description": "Organic flowing forms, nature motifs, decorative elegance of the 1890s-1910s",
    "setting_hints": "curving architecture, floral motifs, stained glass, elegant salons",
    "tech_elements": "ornate gas lamps, decorated trains, artistic posters, flowing gowns",
    "tone": "aesthetic, romantic, nature-inspired, elegant decadence"
}

# Add to genre affinities
GENRE_STYLE_AFFINITIES["social_drama"].append("art_nouveau")
GENRE_STYLE_AFFINITIES["comedy"].append("art_nouveau")
```

### Adjusting Quality Threshold

```python
# In evaluator_assess()
passes = scores["average"] >= 4.0  # Stricter (default: 3.5)
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
│   │ 1. Format Detection → PLAY or NOVEL                    │  │
│   └─────────────────────────────────────────────────────────┘  │
│                            │                                    │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │ 2. Style Selection                                      │  │
│   │    - Pick N unique styles for generated scenes          │  │
│   │    - Optionally pick styles for extracted scenes        │  │
│   │    - Weight selection by genre affinity                 │  │
│   └─────────────────────────────────────────────────────────┘  │
│                            │                                    │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │ 3. Passage Selection (format-aware)                     │  │
│   │    - PlaySegmenter or NovelSegmenter                    │  │
│   │    - Score and rank passages                            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                            │                                    │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │ 4. For each passage: craft_scene(style_key)             │  │
│   │                                                         │  │
│   │   Stage 1: screenwriter (original or themed)            │  │
│   │   Stage 2: director (style-aware refinement)            │  │
│   │   Stage 3: evaluator (style-aware scoring)              │  │
│   └─────────────────────────────────────────────────────────┘  │
│                            │                                    │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │ 5. Generate scenes: craft_scene(unique_style)           │  │
│   │    Each generated scene gets a different style          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   ──► Save progress to scenes.json (after each book)           │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
scenes.json (with thematic_style and style_description fields)
```

### Style Selection Algorithm

```python
def select_style(genre, prefer_affinity=True, affinity_weight=0.7, exclude=[]):
    affinity_styles = GENRE_STYLE_AFFINITIES.get(genre, [])
    
    if prefer_affinity and random.random() < 0.7:
        # 70% chance: pick from affinity list
        return random.choice(affinity_styles)
    else:
        # 30% chance: pick from any style
        return random.choice(all_styles)
```

### Incremental Saves

Progress is saved after each book completes:

```python
for book_id, book_text in found.items():
    scenes = crafter.craft_scenes_for_book(...)
    all_scenes.extend(scenes)
    
    # Save after each book
    save_output(all_scenes, args.output)
```

---

## Extending the Pipeline

### Adding a New Segmenter for Poetry

```python
class PoetrySegmenter:
    CANTO_PATTERN = re.compile(r'\n\s*(CANTO\s+[IVXLCDM\d]+)', re.IGNORECASE)
    
    @classmethod
    def segment(cls, text, min_chars, max_chars):
        # Group stanzas into passage-sized chunks
        ...
```

### Genre-Specific Scoring

```python
class MysteryNovelSegmenter(NovelSegmenter):
    @classmethod
    def _score_passage(cls, text):
        score = super()._score_passage(text)
        
        mystery_words = ['clue', 'suspect', 'murder', 'evidence']
        score += sum(text.lower().count(w) for w in mystery_words) * 0.5
        
        return score
```

### Parallel Processing

```python
from concurrent.futures import ThreadPoolExecutor

def craft_all_parallel(found, crafter, max_workers=3):
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {
            executor.submit(crafter.craft_scenes_for_book, text, book_id): book_id
            for book_id, text in found.items()
        }
        # ... collect results
```

---

## Troubleshooting

### Common Issues

#### "Skipping book X: no metadata"

Add the book to `BOOK_METADATA`:
```python
BOOK_METADATA[YOUR_ID] = {"title": "...", "author": "...", "genre": "...", "format": "novel"}
```

#### Wrong format detected

Override in metadata:
```python
BOOK_METADATA[844]["format"] = "play"  # Force play segmentation
```

#### Low quality scores

- Check if format detection is correct
- Run with `--verbose` to see score breakdowns
- Try lowering threshold: change `>= 3.5` to `>= 3.0`

#### Rate limiting

Add delay between API calls:
```python
self.min_interval = 1.0  # seconds between calls
```

---

## Cost Estimation

With Kimi K2 on Fireworks (~$0.80/M tokens):

| Component | Tokens/Scene | Cost/Scene |
|-----------|--------------|------------|
| Screenwriter | ~2.5K in, ~600 out | ~$0.0025 |
| Director | ~1.5K in, ~900 out | ~$0.002 |
| Evaluator | ~1.2K in, ~400 out | ~$0.0013 |
| **Total** | ~6-7K tokens | **~$0.006** |

**For 100 scenes**: ~$0.60 (excluding retries)
**With retries**: ~$1-2

---

## References

- **CharacterBox Paper**: [arXiv:2412.05631](https://arxiv.org/abs/2412.05631)
- **CharacterBox Repo**: [github.com/Paitesanshi/CharacterBox](https://github.com/Paitesanshi/CharacterBox)
- **Project Gutenberg**: [gutenberg.org](https://www.gutenberg.org/)
- **Fireworks AI**: [fireworks.ai](https://fireworks.ai/)

---

## Contributing

### Areas for Improvement

- [ ] Poetry/verse segmentation
- [ ] More thematic styles (art nouveau, dieselpunk variants, etc.)
- [ ] Style blending (e.g., "gothic cyberpunk")
- [ ] Non-English text support
- [ ] Caching to avoid re-processing
- [ ] Web UI for scene review/editing
- [ ] Integration with CharacterBox evaluation pipeline
- [ ] User-defined custom styles via config file

---

## License

MIT License - See LICENSE file.

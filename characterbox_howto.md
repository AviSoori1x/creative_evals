# CharacterBox Setup Guide for API-Based Model Evaluation

This guide covers setting up CharacterBox from scratch to evaluate models via API (specifically Mistral AI, but applicable to any OpenAI-compatible API).

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#step-1-clone-the-repository)
3. [Environment Setup](#step-2-set-up-python-environment)
4. [Code Fixes](#step-3-fix-tokenizer-configuration)
5. [Configuration](#step-5-configure-for-mistral-api)
6. [Running Evaluations](#step-6-run-the-evaluation-pipeline)
7. [Batch Running All Titles](#step-7-batch-run-all-english-titles)
8. [Results Analysis](#step-8-analyze-results)
9. [Changing Models](#step-9-changing-models-detailed-guide)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Python 3.11 or 3.12 (not 3.13 - compatibility issues with pydantic/langchain)
- UV package manager
- API key for your model provider (Mistral, OpenAI, etc.)

---

## Step 1: Clone the Repository

```bash
git clone https://github.com/Paitesanshi/CharacterBox.git
cd CharacterBox
```

---

## Step 2: Set Up Python Environment

### Install UV (if not already installed)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Create Virtual Environment with Python 3.11

```bash
# Create environment with Python 3.11 (not 3.13!)
uv venv --python 3.11

# Activate the environment
source .venv/bin/activate
```

### Install Dependencies

```bash
# Install all required packages
uv pip install -r requirements.txt

# Install additional dependencies that may be missing
uv pip install fastapi uvicorn
```

---

## Step 3: Fix Tokenizer Configuration

The default GPT2 tokenizer has a 1024 token limit that causes warnings. Update these files to set `model_max_length=8192`:

### Files to modify:

1. **`utils/utils.py`** (line 187):
```python
# Before:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2',max_length=8192)

# After:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2', model_max_length=8192)
```

2. **`evaluate.py`** (line 81):
```python
# Before:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2')

# After:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2', model_max_length=8192)
```

3. **`evaluate_scene.py`** (line 35):
```python
# Before:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2')

# After:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2', model_max_length=8192)
```

4. **`evaluate_narrator.py`** (line 33):
```python
# Before:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2')

# After:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2', model_max_length=8192)
```

5. **`convert_narrator_data.py`** (line 34):
```python
# Before:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2')

# After:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2', model_max_length=8192)
```

6. **`convert_reward_data.py`** (line 34):
```python
# Before:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2')

# After:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2', model_max_length=8192)
```

7. **`convert_character_data.py`** (line 34):
```python
# Before:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2')

# After:
tokenizer = GPT2TokenizerFast.from_pretrained('gpt2', model_max_length=8192)
```

---

## Step 4: Fix Evaluation Script

The `quick_start_eval.py` script has a hardcoded model name that needs to be fixed.

**File: `quick_start_eval.py`** (around line 110):

```python
# Before:
character_llms=['gpt-4o-mini']

# After:
# Use character_llm from config file, or default to gpt-4o-mini if not specified
character_llms=[config.get('character_llm', 'gpt-4o-mini')]
```

---

## Step 5: Configure for Mistral API

### Understanding the Three Model Roles

CharacterBox uses **three different model roles**:

| Role | Purpose | Config File | Config Key |
|------|---------|-------------|------------|
| **Character** | The model being evaluated - generates character responses | `play.yaml` | `character_llm` |
| **Narrator** | Controls the story/scene, generates prompts for the character | `play.yaml` | `narrator_llm` |
| **Judger** | Evaluates the quality of character responses | `evaluate.yaml` | `judger_llm` |

### Configure Simulation (`config/play.yaml`):

```yaml
# name of LLM model, 'custom' for custom model
narrator_llm: mistral-large-2512
character_llm: labs-mistral-small-creative  # <-- THE MODEL YOU WANT TO TEST

# path for scenes data
scene_path: data/extracted_scenes/Harry_Potter.json
scene_id: 0

# number of rounds
round: 3

# number of max retries for LLM API
max_retries: 5

# maximum number of tokens for LLM
max_token: 8192

# temperature for LLM
temperature: 0.5

# verbose mode
verbose: True

# API credentials for Character LLM (the model being tested)
character_api_key: YOUR_API_KEY
character_api_base: https://api.mistral.ai/v1

# API credentials for Narrator LLM
narrator_api_key: YOUR_API_KEY
narrator_api_base: https://api.mistral.ai/v1

# path to save records
save_record: True
record_dir: output/record
character_record_path: character.jsonl
round_record_path: round.jsonl
plot_record_path: plot.jsonl
all_record_path: all.jsonl
narrator_record_path: narrator.jsonl
```

### Configure Evaluation (`config/evaluate.yaml`):

```yaml
# name of LLM model, 'custom' for custom model
judger_llm: mistral-large-2512           # <-- MODEL THAT SCORES THE RESPONSES
narrator_llm: mistral-large-2512          # <-- MUST MATCH play.yaml
character_llm: labs-mistral-small-creative # <-- MUST MATCH play.yaml

# path for item data
title: Harry_Potter
scene_id: 0

# directory name for faiss index
round: 3

# temperature for LLM (use 0 for deterministic judging)
temperature: 0

# maximum number of tokens for LLM
max_token: 8192

# execution mode, serial or parallel
max_retries: 1

# verbose mode
verbose: True

# API credentials
api_key: YOUR_API_KEY
api_base: https://api.mistral.ai/v1

# Output configuration
record_dir: output/record
character_record_path: character.json
round_record_path: round.json
plot_record_path: plot.json
narrator_record_path: narrator.json
max_rounds: 5
max_scenes: 5
```

---

## Step 6: Run the Evaluation Pipeline

### Step 6.1: Run Simulation (generates character responses)

```bash
cd /path/to/CharacterBox
python -u simulator.py --config_file config/play.yaml --log_file simulation.log
```

**Output:** Creates files in `output/record/{title}/character/`

### Step 6.2: Run Evaluation (scores the responses)

```bash
python quick_start_eval.py --config_file config/evaluate.yaml --log_file evaluate.log
```

**Output:** Creates evaluation CSV in `output/evaluation/detail/{title}/`

---

## Step 7: Batch Run All English Titles

### Create the Batch Script

Save as `run_english_titles.sh`:

```bash
#!/bin/bash

TITLES=("Harry_Potter" "The_Lord_of_the_Rings" "The_Matrix" "Twilight" "A_Song_of_Ice_and_Fire")

for TITLE in "${TITLES[@]}"; do
    echo "========== Running $TITLE =========="
    
    # Update play.yaml
    sed -i "s|scene_path:.*|scene_path: data/extracted_scenes/${TITLE}.json|" config/play.yaml
    
    # Update evaluate.yaml
    sed -i "s|title:.*|title: ${TITLE}|" config/evaluate.yaml
    
    # Run simulation
    echo "Running simulation for $TITLE..."
    python -u simulator.py --config_file config/play.yaml --log_file "simulation_${TITLE}.log"
    
    # Run evaluation
    echo "Running evaluation for $TITLE..."
    python quick_start_eval.py --config_file config/evaluate.yaml --log_file "evaluate_${TITLE}.log"
    
    echo "========== Completed $TITLE =========="
    echo ""
done

echo "========== All English evaluations complete =========="
echo "Results are in: output/evaluation/detail/"
```

### Run the Batch Script

```bash
chmod +x run_english_titles.sh
./run_english_titles.sh
```

### Available Titles

| Language | Title | Config Value |
|----------|-------|--------------|
| English | Harry Potter | `Harry_Potter` |
| English | The Lord of the Rings | `The_Lord_of_the_Rings` |
| English | The Matrix | `The_Matrix` |
| English | Twilight | `Twilight` |
| English | A Song of Ice and Fire | `A_Song_of_Ice_and_Fire` |
| Chinese | Journey to the West | `西游记` |
| Chinese | Romance of the Three Kingdoms | `三国演义` |
| Chinese | Dream of the Red Chamber | `红楼梦` |
| Chinese | My Fair Princess | `还珠格格` |
| Chinese | The Smiling, Proud Wanderer | `笑傲江湖` |

---

## Step 8: Analyze Results

### Quick View of a Single Result

```bash
# View Harry Potter results
python -c "
import pandas as pd
df = pd.read_csv('output/evaluation/detail/Harry_Potter/mistral-large-2512_mistral-large-2512_character_evaluation_detail.csv')
cols = ['Knowledge Accuracy', 'Emotional Expression', 'Personality Traits', 'Behavioral Accuracy', 'Immersion', 'Adaptability', 'Behavioral Coherence']
means = df[cols].mean()
print('Mean Scores for Harry Potter:')
for col in cols:
    print(f'{col}: {means[col]:.2f}')
print(f'Overall Average: {means.mean():.2f}')
"
```

### Comprehensive Results Analysis Script

Save as `analyze_results.py`:

```python
import pandas as pd
import os
from glob import glob

# Scoring criteria columns
SCORE_COLUMNS = [
    'Knowledge Accuracy',
    'Emotional Expression',
    'Personality Traits',
    'Behavioral Accuracy',
    'Immersion',
    'Adaptability',
    'Behavioral Coherence'
]

def analyze_results():
    # Find all evaluation CSV files
    result_files = glob('output/evaluation/detail/*/*.csv')
    
    all_results = []
    
    for file_path in result_files:
        title = file_path.split('/')[-2]
        
        try:
            df = pd.read_csv(file_path)
            
            # Calculate mean scores for this title
            means = df[SCORE_COLUMNS].mean()
            
            result = {'Title': title}
            for col in SCORE_COLUMNS:
                result[col] = means[col]
            
            # Calculate overall average
            result['Overall Average'] = means.mean()
            
            all_results.append(result)
            
            print(f"\n{'='*80}")
            print(f"Title: {title}")
            print(f"{'='*80}")
            for col in SCORE_COLUMNS:
                print(f"{col:.<40} {means[col]:.2f}")
            print(f"{'Overall Average':.<40} {means.mean():.2f}")
            
        except Exception as e:
            print(f"Error processing {title}: {e}")
    
    # Create summary DataFrame
    summary_df = pd.DataFrame(all_results)
    
    # Save summary
    os.makedirs('output/evaluation', exist_ok=True)
    summary_df.to_csv('output/evaluation/summary_results.csv', index=False)
    print(f"\n{'='*80}")
    print("Summary saved to: output/evaluation/summary_results.csv")
    print(f"{'='*80}\n")
    
    # Print summary table
    print("\nSUMMARY TABLE")
    print(summary_df.to_string(index=False))
    
    # Calculate grand mean across all titles
    if len(summary_df) > 0:
        print(f"\n{'='*80}")
        print("GRAND MEAN ACROSS ALL TITLES")
        print(f"{'='*80}")
        grand_means = summary_df[SCORE_COLUMNS + ['Overall Average']].mean()
        for col in SCORE_COLUMNS:
            print(f"{col:.<40} {grand_means[col]:.2f}")
        print(f"{'Overall Average':.<40} {grand_means['Overall Average']:.2f}")

if __name__ == "__main__":
    analyze_results()
```

### Run the Analysis

```bash
python analyze_results.py
```

### Export to Excel

```bash
python -c "
import pandas as pd
from glob import glob

# Combine all results into one Excel file
writer = pd.ExcelWriter('output/evaluation/all_results.xlsx', engine='openpyxl')

for file in glob('output/evaluation/detail/*/*.csv'):
    title = file.split('/')[-2]
    df = pd.read_csv(file)
    df.to_excel(writer, sheet_name=title[:31], index=False)  # Excel sheet names max 31 chars

writer.close()
print('All results exported to: output/evaluation/all_results.xlsx')
"
```

### Evaluation Metrics Explained

| Metric | Description | Score Range |
|--------|-------------|-------------|
| **Knowledge Accuracy** | Correctness of factual information about the character's world | 1-10 |
| **Emotional Expression** | Appropriateness of emotional responses to situations | 1-10 |
| **Personality Traits** | Consistency with the character's known personality | 1-10 |
| **Behavioral Accuracy** | Actions match what the character would actually do | 1-10 |
| **Immersion** | Quality of roleplay engagement and believability | 1-10 |
| **Adaptability** | Response quality when situations change unexpectedly | 1-10 |
| **Behavioral Coherence** | Consistency of behavior across multiple interactions | 1-10 |

---

## Step 9: Changing Models (Detailed Guide)

### Understanding the Model Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SIMULATION PHASE                            │
│                         (simulator.py)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐         ┌─────────────┐         ┌─────────────┐   │
│  │  NARRATOR   │ ──────▶ │  SCENARIO   │ ──────▶ │  CHARACTER  │   │
│  │   MODEL     │         │   PROMPT    │         │    MODEL    │   │
│  │             │         │             │         │  (TESTED)   │   │
│  └─────────────┘         └─────────────┘         └─────────────┘   │
│        │                                               │           │
│        │                                               │           │
│        ▼                                               ▼           │
│  Generates story                              Generates character  │
│  scenarios and                                responses that get   │
│  prompts                                      evaluated            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        EVALUATION PHASE                             │
│                      (quick_start_eval.py)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐         ┌─────────────┐         ┌─────────────┐   │
│  │   JUDGER    │ ──────▶ │   SCORING   │ ──────▶ │   RESULTS   │   │
│  │    MODEL    │         │   PROMPTS   │         │    CSV      │   │
│  │             │         │             │         │             │   │
│  └─────────────┘         └─────────────┘         └─────────────┘   │
│        │                                                           │
│        ▼                                                           │
│  Reads character responses and scores them on 7 metrics            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Changing the Character Model (Model Being Tested)

This is the model whose roleplay abilities you want to evaluate.

**Step 1: Update `config/play.yaml`**

```yaml
# Change this line:
character_llm: your-new-model-name

# Update API credentials if using a different provider:
character_api_key: YOUR_NEW_API_KEY
character_api_base: https://api.your-provider.com/v1
```

**Step 2: Update `config/evaluate.yaml`**

```yaml
# MUST match the character_llm in play.yaml:
character_llm: your-new-model-name
```

**Step 3: Re-run simulation and evaluation**

```bash
python -u simulator.py --config_file config/play.yaml --log_file simulation.log
python quick_start_eval.py --config_file config/evaluate.yaml --log_file evaluate.log
```

### Changing the Narrator Model

The narrator creates scenarios and prompts for the character. A stronger model here means better quality test scenarios.

**Update `config/play.yaml`:**

```yaml
# Change these lines:
narrator_llm: your-narrator-model

# Update API credentials if different from character:
narrator_api_key: YOUR_API_KEY
narrator_api_base: https://api.your-provider.com/v1
```

**Update `config/evaluate.yaml`:**

```yaml
# MUST match the narrator_llm in play.yaml:
narrator_llm: your-narrator-model
```

### Changing the Judger Model

The judger evaluates the quality of responses. Use a strong model for accurate scoring.

**Update `config/evaluate.yaml` only:**

```yaml
# Change this line:
judger_llm: your-judger-model

# API credentials (used for judger):
api_key: YOUR_API_KEY
api_base: https://api.your-provider.com/v1
```

### Using Different Providers for Different Roles

You can use different API providers for each role:

**`config/play.yaml`:**

```yaml
# Narrator from OpenAI
narrator_llm: gpt-4o
narrator_api_key: YOUR_OPENAI_KEY
narrator_api_base: https://api.openai.com/v1

# Character from Mistral (model being tested)
character_llm: mistral-small-creative
character_api_key: YOUR_MISTRAL_KEY
character_api_base: https://api.mistral.ai/v1
```

**`config/evaluate.yaml`:**

```yaml
# Judger from Anthropic (via OpenAI-compatible proxy)
judger_llm: claude-3-opus
api_key: YOUR_ANTHROPIC_KEY
api_base: https://api.anthropic.com/v1

# Must match play.yaml
narrator_llm: gpt-4o
character_llm: mistral-small-creative
```

### Example: Testing Multiple Models

To compare different models, run the pipeline for each:

```bash
#!/bin/bash

# Models to test
MODELS=("mistral-small-latest" "mistral-small-creative" "mistral-large-latest")

for MODEL in "${MODELS[@]}"; do
    echo "========== Testing $MODEL =========="
    
    # Update play.yaml
    sed -i "s|character_llm:.*|character_llm: ${MODEL}|" config/play.yaml
    
    # Update evaluate.yaml
    sed -i "s|character_llm:.*|character_llm: ${MODEL}|" config/evaluate.yaml
    
    # Run for all English titles
    for TITLE in "Harry_Potter" "The_Lord_of_the_Rings" "The_Matrix" "Twilight" "A_Song_of_Ice_and_Fire"; do
        sed -i "s|scene_path:.*|scene_path: data/extracted_scenes/${TITLE}.json|" config/play.yaml
        sed -i "s|title:.*|title: ${TITLE}|" config/evaluate.yaml
        
        python -u simulator.py --config_file config/play.yaml --log_file "simulation_${MODEL}_${TITLE}.log"
        python quick_start_eval.py --config_file config/evaluate.yaml --log_file "evaluate_${MODEL}_${TITLE}.log"
    done
done

echo "All models tested!"
```

### Model Naming in Output Files

Output files are named based on the model names:

```
output/record/{title}/character/{narrator}_{character}_character.jsonl
output/evaluation/detail/{title}/{judger}_{narrator}_character_evaluation_detail.csv
```

Example:
```
output/record/Harry_Potter/character/mistral-large-2512_labs-mistral-small-creative_character.jsonl
output/evaluation/detail/Harry_Potter/mistral-large-2512_mistral-large-2512_character_evaluation_detail.csv
```

### Available Model Examples

**Mistral AI:**
- `mistral-small-latest`
- `mistral-small-creative`
- `mistral-medium-latest`
- `mistral-large-latest`
- `mistral-large-2512`
- `labs-mistral-small-creative`

**OpenAI:**
- `gpt-4o`
- `gpt-4o-mini`
- `gpt-4-turbo`
- `gpt-3.5-turbo`

**Local Models (via vLLM or similar):**
- Set `api_base: http://localhost:8000/v1`
- Use the model name as configured in your local server

---

## Understanding the Workflow

```
┌─────────────────┐
│  Simulation     │ - Uses narrator + character models
│  (simulator.py) │ - Generates roleplay conversations
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Record Files   │ - Saved to output/record/
│  (.jsonl)       │ - Contains character responses
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Evaluation     │ - Uses judger model
│  (quick_start)  │ - Scores responses on 7 metrics
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Results        │ - CSV with detailed scores
│  (.csv)         │ - Knowledge, Emotion, Personality, etc.
└─────────────────┘
```

---

## Output Structure

```
output/
├── record/
│   ├── Harry_Potter/
│   │   ├── character/
│   │   │   └── {narrator}_{character}_character.jsonl
│   │   ├── narrator/
│   │   ├── plot/
│   │   ├── round/
│   │   └── all/
│   ├── The_Lord_of_the_Rings/
│   ├── The_Matrix/
│   ├── Twilight/
│   └── A_Song_of_Ice_and_Fire/
├── evaluation/
│   ├── summary_results.csv          # Generated by analyze_results.py
│   ├── all_results.xlsx             # Optional Excel export
│   └── detail/
│       ├── Harry_Potter/
│       │   └── {judger}_{narrator}_character_evaluation_detail.csv
│       ├── The_Lord_of_the_Rings/
│       ├── The_Matrix/
│       ├── Twilight/
│       └── A_Song_of_Ice_and_Fire/
└── log/
    └── evaluation/
```

---

## Troubleshooting

### Python 3.13 Compatibility Error
```
TypeError: ForwardRef._evaluate() missing 1 required keyword-only argument: 'recursive_guard'
```

**Solution:** Use Python 3.11 or 3.12. Recreate your virtual environment:
```bash
rm -rf .venv
uv venv --python 3.11
source .venv/bin/activate
uv pip install -r requirements.txt
```

### Token Length Warnings
```
Token indices sequence length is longer than the specified maximum sequence length
```

**Solution:** Apply the tokenizer fixes in Step 3 above.

### Missing Module Errors
```
ModuleNotFoundError: No module named 'fastapi'
```

**Solution:** Install missing packages:
```bash
uv pip install fastapi uvicorn
```

### File Not Found During Evaluation
```
FileNotFoundError: output/record/Harry_Potter/character/...
```

**Solution:** Run the simulation first before evaluation. The character_llm and narrator_llm in evaluate.yaml must match the models used in play.yaml.

### Log File Directory Error
```
FileNotFoundError: [Errno 2] No such file or directory: '.../output/log/logs/simulation_Harry_Potter.log'
```

**Solution:** Don't use nested paths for log files. Use simple filenames like `simulation.log` instead of `logs/simulation.log`.

### Wrong Model Being Evaluated
```
Current Model: gpt-4o-mini
```

**Solution:** Make sure you applied the fix in Step 4 to `quick_start_eval.py`. The script should use `config.get('character_llm', 'gpt-4o-mini')` instead of hardcoded `'gpt-4o-mini'`.

---

## Security Note

⚠️ **Never commit API keys to version control!** 

Consider using environment variables:

```bash
export MISTRAL_API_KEY="your-key-here"
```

Then in your config files, you can reference it (though CharacterBox doesn't natively support env vars, you can use a wrapper script):

```bash
#!/bin/bash
sed -i "s|api_key:.*|api_key: ${MISTRAL_API_KEY}|" config/play.yaml
sed -i "s|api_key:.*|api_key: ${MISTRAL_API_KEY}|" config/evaluate.yaml
# ... run your evaluation
```

---

## Summary of Changes Made to Original Repo

1. ✅ Fixed Python version compatibility (3.11/3.12 instead of 3.13)
2. ✅ Updated all GPT2 tokenizer initializations to support 8192 tokens
3. ✅ Fixed `quick_start_eval.py` to use config file instead of hardcoded model
4. ✅ Configured both `play.yaml` and `evaluate.yaml` for Mistral API
5. ✅ Installed missing dependencies (fastapi, uvicorn)
6. ✅ Created batch scripts for running all English titles
7. ✅ Created results analysis script

---

## Quick Reference Card

### Run Single Evaluation

```bash
# 1. Simulation
python -u simulator.py --config_file config/play.yaml --log_file simulation.log

# 2. Evaluation  
python quick_start_eval.py --config_file config/evaluate.yaml --log_file evaluate.log

# 3. View Results
python analyze_results.py
```

### Run All English Titles

```bash
./run_english_titles.sh
python analyze_results.py
```

### Change Model Being Tested

1. Edit `config/play.yaml`: change `character_llm`
2. Edit `config/evaluate.yaml`: change `character_llm` to match
3. Re-run simulation and evaluation

---

## Questions?

Refer to the original CharacterBox documentation:
- README.md
- Experiment.md

Or check the paper for methodology details.

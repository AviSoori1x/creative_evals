# Understanding CharacterBox: How It Works and Why

This document explains the CharacterBox evaluation framework in detail - its motivation, architecture, and implementation.

---

## Core Motivation

**CharacterBox evaluates how well Large Language Models can roleplay fictional characters consistently and accurately.**

The key insight: Traditional LLM benchmarks test general capabilities (math, reasoning, knowledge), but **playing a character requires**:
- Deep knowledge of character backstory
- Consistent personality traits
- Appropriate emotional responses
- Behavioral patterns that match the character
- Adapting to evolving story contexts

This is particularly important for:
- Interactive storytelling applications
- Character-based chatbots
- Video game NPCs
- Entertainment/creative writing tools

---

## The Three-Actor System

CharacterBox uses a **multi-agent simulation** with three distinct roles:

### 1. **Character LLM** (The Model Being Tested)
- **Role**: Plays a specific character (e.g., Harry Potter, Frodo)
- **Input**: Character profile + current scene context + conversation history
- **Output**: What the character says and does
- **This is the model you're evaluating**

### 2. **Narrator LLM** (The Story Manager)
- **Role**: Controls the story progression and other NPCs
- **Input**: Scene description + character actions + plot requirements
- **Output**: Scene descriptions, other character responses, plot developments
- **Purpose**: Creates a dynamic story environment for the character to respond to

### 3. **Judger LLM** (The Evaluator)
- **Role**: Evaluates the character's responses
- **Input**: Character profile + scene context + character's response
- **Output**: Scores across 7 dimensions (see below)
- **Purpose**: Provides objective assessment of roleplay quality

---

## How the Evaluation Works

### Phase 1: Data Preparation (Already Done)

The `data` folder contains pre-processed scripts from 10 movies/shows:

```
data/extracted_scenes/Harry_Potter.json
```

Each JSON contains:
- **Character profiles**: Background, personality, relationships
- **Scene descriptions**: Settings, context, dramatic situations
- **Expected behaviors**: How characters should act in each scene

Example structure:
```json
{
  "characters": {
    "Harry Potter": {
      "background": "Orphaned wizard, lives with Dursleys...",
      "personality": "Brave, loyal, sometimes impulsive...",
      "relationships": {"Hermione": "best friend", ...}
    }
  },
  "scenes": [
    {
      "scene_id": 0,
      "setting": "Hogwarts Great Hall",
      "context": "First day sorting ceremony",
      "characters_present": ["Harry", "Hermione", "Ron"]
    }
  ]
}
```

### Phase 2: Simulation (`simulator.py`)

**What happens:**

1. **Load Scene**: Read scene data from JSON
2. **Initialize Agents**:
   - Character agent (your test model) gets character profile
   - Narrator agent gets scene description
3. **Multi-Round Dialogue**:
   ```
   Round 1:
   Narrator: "You enter the Great Hall. The ceiling shows a starry sky..."
   Character (Harry): "I look around nervously, wondering where to sit..."
   Narrator: "Hermione waves at you from the Gryffindor table..."
   Character (Harry): "I smile and walk toward her..."
   
   Round 2:
   Narrator: "Professor McGonagall calls your name for sorting..."
   Character (Harry): [responds based on character knowledge/personality]
   ...
   ```

4. **Record Everything**: Save to `output/record/Harry_Potter/`
   - `character.jsonl`: All character responses
   - `narrator.jsonl`: All narrator prompts
   - `round.jsonl`: Full conversation by round
   - `plot.jsonl`: Story progression

**Key Code (simplified from `simulator.py`):**

```python
# Load character and scene
character = Character(profile, character_llm)
narrator = Narrator(scene_description, narrator_llm)

# Run simulation rounds
for round_num in range(max_rounds):
    # Narrator sets the scene
    narrator_message = narrator.generate_scene()
    
    # Character responds
    character_response = character.act(
        current_scene=narrator_message,
        conversation_history=history
    )
    
    # Narrator reacts to character's actions
    narrator.update(character_response)
    
    # Record everything
    save_record(character_response, narrator_message, round_num)
```

### Phase 3: Evaluation (`evaluate.py` / `quick_start_eval.py`)

**What happens:**

1. **Load Simulation Records**: Read the `.jsonl` files from Phase 2
2. **For Each Character Response**:
   - Extract: character profile, scene context, character's action
   - Build evaluation prompt for Judger LLM
3. **Judger Scores 7 Dimensions** (1-10 scale):

   **a. Knowledge Accuracy**
   - Does the character know what they should know?
   - Example: Harry should know Hogwarts layout, not Voldemort's plans

   **b. Emotional Expression**
   - Are emotions appropriate for the scene?
   - Example: Fear when facing danger, joy when with friends

   **c. Personality Traits**
   - Does behavior match character personality?
   - Example: Harry should be brave but not reckless

   **d. Behavioral Accuracy**
   - Do actions align with how this character typically behaves?
   - Example: Harry would defend friends, not abandon them

   **e. Immersion**
   - Does response maintain story consistency?
   - Example: No modern slang in medieval settings

   **f. Adaptability**
   - Can character handle unexpected situations appropriately?
   - Example: Responding to plot twists naturally

   **g. Behavioral Coherence**
   - Are actions consistent across conversation rounds?
   - Example: Not suddenly changing personality mid-scene

4. **Save Results**: CSV with all scores + detailed critiques

**Key Code (simplified from `evaluate.py`):**

```python
# Load character response from simulation
character_response = load_record(record_file)

# Build evaluation prompt
eval_prompt = f"""
Character Profile: {character_profile}
Scene Context: {scene_description}
Character's Response: {character_response}

Evaluate this response on:
1. Knowledge Accuracy (1-10):
2. Emotional Expression (1-10):
...

Provide scores and detailed critique.
"""

# Get judger's evaluation
judger_response = judger_llm.generate(eval_prompt)

# Parse scores
scores = parse_evaluation(judger_response)
save_to_csv(scores)
```

---

## File Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│                   DATA PREPARATION                       │
│                                                          │
│  data/extracted_scenes/Harry_Potter.json                 │
│  ├── Character profiles                                  │
│  ├── Scene descriptions                                  │
│  └── Expected behaviors                                  │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│              SIMULATION (simulator.py)                   │
│                                                          │
│  config/play.yaml → Configure models                     │
│  ├── narrator_llm: mistral-large-2512                    │
│  └── character_llm: labs-mistral-small-creative          │
│                                                          │
│  Generates multi-round conversations                     │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                  SIMULATION OUTPUT                       │
│                                                          │
│  output/record/Harry_Potter/                             │
│  ├── character.jsonl ← Character responses               │
│  ├── narrator.jsonl ← Narrator prompts                   │
│  ├── round.jsonl ← Full conversations                    │
│  └── plot.jsonl ← Story progression                      │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│         EVALUATION (quick_start_eval.py)                 │
│                                                          │
│  config/evaluate.yaml → Configure judger                 │
│  └── judger_llm: mistral-large-2512                      │
│                                                          │
│  Reads simulation records                                │
│  Scores each response on 7 dimensions                    │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                  EVALUATION OUTPUT                       │
│                                                          │
│  output/evaluation/detail/Harry_Potter/                  │
│  └── mistral-large_mistral-large_character_eval.csv      │
│      ├── Knowledge Accuracy scores                       │
│      ├── Emotional Expression scores                     │
│      ├── Personality Traits scores                       │
│      ├── Behavioral Accuracy scores                      │
│      ├── Immersion scores                                │
│      ├── Adaptability scores                             │
│      ├── Behavioral Coherence scores                     │
│      └── Detailed critiques                              │
└─────────────────────────────────────────────────────────┘
```

---

## Why This Design?

### 1. **Separation of Concerns**
- Character model focuses only on roleplay
- Narrator handles story complexity
- Judger provides objective scoring
- Each can be a different model/size

### 2. **Dynamic Evaluation**
- Not just static Q&A
- Characters must adapt to evolving situations
- Tests consistency across multiple turns

### 3. **Multi-Dimensional Assessment**
- Single score can't capture roleplay quality
- 7 dimensions provide detailed analysis
- Identifies specific weaknesses (e.g., good knowledge but poor emotion)

### 4. **Scalable Testing**
- 10 different universes (Harry Potter, LOTR, etc.)
- Multiple scenes per universe
- Multiple rounds per scene
- Can test on 100s of interactions

### 5. **Cost-Effective**
- Can use small model for character (being tested)
- Use large model for judging (more accurate evaluation)
- Separate narrator allows story control without affecting character

---

## Example Walkthrough

Let's trace a single evaluation:

**Scene**: Harry Potter's first Potions class

**Simulation** (`simulator.py`):
```
Narrator (mistral-large-2512):
"You enter the dungeons for your first Potions class. 
Professor Snape's cold eyes fix on you as you find a seat."

Character (labs-mistral-small-creative as Harry):
"I try to avoid Snape's gaze and sit next to Ron, 
pulling out my Potions textbook nervously."

Narrator:
"Snape begins class. 'Potter,' he sneers, 'tell me, 
what would I get if I added powdered root of asphodel 
to an infusion of wormwood?'"

Character (Harry):
"I freeze, realizing I don't know the answer. 
'I... I don't know, sir,' I admit quietly."
```

**Evaluation** (`quick_start_eval.py`):

Judger analyzes Harry's second response:

```
Knowledge Accuracy: 8/10
✓ Correctly doesn't know advanced potion ingredients (first year)
✓ Honest about not knowing

Emotional Expression: 9/10
✓ Shows nervousness (freeze, quietly)
✓ Appropriate fear/respect for Snape

Personality Traits: 9/10
✓ Honest rather than making up answer (integrity)
✓ Admits weakness (humility)

Behavioral Accuracy: 8/10
✓ Avoiding Snape's gaze (typical Harry)
✓ Sitting with Ron (friendship)

Immersion: 10/10
✓ Uses appropriate language
✓ Responds to story context naturally

Adaptability: 7/10
✓ Handles unexpected question
− Could show more active problem-solving

Behavioral Coherence: 9/10
✓ Consistent nervousness across responses
✓ Maintains character voice

Critique: "Strong characterization of Harry as nervous 
but honest first-year student. Captures his relationship 
dynamics well. Could show slightly more determination..."
```

---

## Key Insights from the Code

### 1. **Prompt Engineering is Critical**

Look at `agents/character.py`:
```python
def build_character_prompt(profile, scene, history):
    return f"""You are {profile['name']}.

Background: {profile['background']}
Personality: {profile['personality']}
Current Scene: {scene}
Recent Events: {history}

Respond AS THIS CHARACTER. Stay in character.
What do you say and do?"""
```

The quality of character behavior depends heavily on:
- How well the profile is written
- How scene context is presented
- How conversation history is formatted

### 2. **Token Management**

Notice all the `max_token: 32768` settings:
- Character needs full history to maintain consistency
- Narrator needs scene context to generate coherent story
- Judger needs everything to evaluate properly

This is why we increased tokenizer limits to 32768.

### 3. **Record Everything**

The `.jsonl` files save:
- Every prompt sent to models
- Every response received
- Timestamps, model names, configurations

This allows:
- Debugging failed evaluations
- Analyzing model behavior patterns
- Reproducing results exactly

### 4. **Modular Architecture**

Each component is pluggable:
```python
# You can swap any component
character_llm = get_llm("labs-mistral-small-creative")  # Test model
narrator_llm = get_llm("mistral-large-2512")            # Story control
judger_llm = get_llm("gpt-4")                           # Accurate scoring

# Or even use local models
character_llm = get_llm("custom", api_base="http://localhost:8000")
```

---

## What Makes This Different from Other Benchmarks?

| Traditional Benchmarks | CharacterBox |
|----------------------|--------------|
| Single-turn Q&A | Multi-turn dialogue |
| Static questions | Dynamic story progression |
| Right/wrong answers | Subjective roleplay quality |
| General knowledge | Character-specific knowledge |
| Task completion | Personality consistency |
| No memory needed | Must maintain coherence |

**Example:**

Traditional: "What house is Harry Potter in?" → "Gryffindor" ✓

CharacterBox: 
```
Scene: Death Eater attacks Hogwarts
Character must:
- Know Gryffindor values (knowledge)
- Feel appropriate fear (emotion)
- Act bravely anyway (personality)
- Protect friends (behavior)
- Stay in 1990s Britain context (immersion)
- Adapt to chaos (adaptability)
- Be consistent with earlier actions (coherence)
```

---

## Limitations and Considerations

### 1. **Judger Bias**
- Evaluation quality depends on judger model
- Different judgers might score differently
- Strong models (GPT-4, Claude) recommended for judging

### 2. **Subjectivity**
- "Good" roleplay is somewhat subjective
- Multiple valid character interpretations exist
- Scores are guidelines, not absolute truth

### 3. **Context Length**
- Long conversations may exceed context limits
- Earlier interactions might be forgotten
- Need very long context models for full consistency

### 4. **Cost**
- Running 5 titles × 5 scenes × 5 rounds = 125+ API calls
- With 3 models (character, narrator, judger) = 375+ calls
- Can get expensive with large models

### 5. **Character Profile Quality**
- Evaluation quality limited by profile detail
- Some characters more complex than others
- Profiles might miss important traits

---

## Use Cases

### 1. **Model Selection for Character AI**
Compare models to find best for your character chatbot:
```
GPT-4: 8.5/10 average (expensive)
Claude-3: 8.3/10 average (good balance)
Mistral-Large: 7.8/10 average (cheaper)
Mistral-Small: 6.9/10 average (budget option)
```

### 2. **Fine-Tuning Validation**
Test if your fine-tuned model improved:
```
Base model: 6.5/10
After fine-tuning on character data: 7.8/10 ✓
```

### 3. **Prompt Engineering**
Compare different character prompt formats:
```
Prompt A: "You are Harry Potter..."  → 7.2/10
Prompt B: "Roleplay as Harry..."     → 7.8/10
Prompt C: "Continue as Harry..."     → 8.1/10 ✓
```

### 4. **Character Difficulty Analysis**
Find which characters are hardest to roleplay:
```
Easy: Ron Weasley (straightforward personality)
Medium: Hermione (complex but consistent)
Hard: Snape (requires subtle emotional layers)
```

---

## The Seven Evaluation Dimensions Explained

### 1. Knowledge Accuracy (1-10)
**What it measures:** Does the character know what they should know at this point in their story?

**Good example:**
- Harry (Year 1) doesn't know advanced spells ✓
- Harry remembers his parents died when he was a baby ✓

**Bad example:**
- Harry (Year 1) references events from Year 7 ✗
- Harry doesn't recognize Hermione (his best friend) ✗

### 2. Emotional Expression (1-10)
**What it measures:** Are the character's emotions appropriate for the situation?

**Good example:**
- Fear when facing Voldemort ✓
- Joy when Gryffindor wins the House Cup ✓

**Bad example:**
- Laughing when a friend is in danger ✗
- No emotional reaction to major events ✗

### 3. Personality Traits (1-10)
**What it measures:** Does behavior align with the character's core personality?

**Good example:**
- Harry shows bravery when friends need help ✓
- Hermione seeks logical explanations ✓

**Bad example:**
- Harry abandons friends to save himself ✗
- Hermione ignoring rules without good reason ✗

### 4. Behavioral Accuracy (1-10)
**What it measures:** Do actions match how this character typically behaves?

**Good example:**
- Harry using Expelliarmus (his signature spell) ✓
- Ron making chess-related strategic observations ✓

**Bad example:**
- Harry using Unforgivable Curses casually ✗
- Hermione refusing to study ✗

### 5. Immersion (1-10)
**What it measures:** Does the response maintain the story's world and atmosphere?

**Good example:**
- Using period-appropriate language ✓
- Referencing in-world locations/items ✓

**Bad example:**
- "Let me Google that spell" (anachronism) ✗
- References to real-world places ✗

### 6. Adaptability (1-10)
**What it measures:** Can the character handle unexpected situations appropriately?

**Good example:**
- Improvising when a plan fails ✓
- Adjusting behavior when learning new information ✓

**Bad example:**
- Ignoring major plot developments ✗
- Continuing same action despite changed circumstances ✗

### 7. Behavioral Coherence (1-10)
**What it measures:** Are actions consistent across conversation rounds?

**Good example:**
- Maintaining injured state from previous round ✓
- Following through on stated intentions ✓

**Bad example:**
- Suddenly having different abilities ✗
- Forgetting what happened moments ago ✗

---

## Technical Implementation Details

### Memory Management

The character agent uses a retrieval system to manage long conversations:

```python
class Character:
    def __init__(self, profile, llm):
        # Vector store for conversation history
        self.memory = TimeWeightedVectorStoreRetriever()
        self.profile = profile
        self.llm = llm
    
    def act(self, scene, history):
        # Retrieve relevant memories
        relevant_context = self.memory.get_relevant_documents(scene)
        
        # Build prompt with profile + scene + context
        prompt = self.build_prompt(self.profile, scene, relevant_context)
        
        # Generate response
        response = self.llm.generate(prompt)
        
        # Store this interaction
        self.memory.add_documents([response])
        
        return response
```

This allows:
- Handling conversations longer than context window
- Prioritizing recent and relevant memories
- Maintaining long-term consistency

### Multi-Scene Evaluation

The evaluation can test across multiple scenes:

```yaml
# config/evaluate.yaml
max_scenes: 5  # Test on 5 different scenes
max_rounds: 5  # Each scene has 5 conversation rounds
```

This means:
- 5 scenes × 5 rounds = 25 total character responses
- Tests consistency across different contexts
- More comprehensive evaluation

### Parallel Execution

The evaluation can run multiple judging tasks in parallel:

```python
with concurrent.futures.ThreadPoolExecutor() as executor:
    futures = []
    for scene in scenes:
        future = executor.submit(evaluate_scene, scene)
        futures.append(future)
    
    for future in concurrent.futures.as_completed(futures):
        result = future.result()
```

This speeds up evaluation significantly when testing multiple scenes.

---

## Summary

**CharacterBox is a framework for rigorously testing how well LLMs can maintain character consistency in interactive storytelling.**

**The workflow:**
1. **Simulation**: Character model interacts with narrator-driven story
2. **Recording**: All interactions saved for analysis
3. **Evaluation**: Judger scores character performance on 7 dimensions
4. **Analysis**: Aggregate scores reveal model strengths/weaknesses

**Why it matters:**
- Character AI is a growing application area
- Requires different skills than traditional benchmarks
- Provides multi-dimensional assessment
- Supports rigorous model comparison

**Key innovations:**
- Three-actor system (Character, Narrator, Judger)
- Dynamic, multi-turn evaluation
- 7-dimensional scoring framework
- Multiple story universes for testing
- Reproducible, recorded interactions

**Your setup:**
- Testing: `labs-mistral-small-creative` (Mistral's creative model)
- Against: 5 English story universes
- Using: `mistral-large-2512` for narration and judging
- Results: Quantitative scores + qualitative critiques

This lets you answer: **"How good is this model at playing fictional characters compared to alternatives?"**

---

## References

- Original Paper: [CharacterBox - Character Roleplay Evaluation]
- GitHub: https://github.com/Paitesanshi/CharacterBox
- Setup Guide: `SETUP_GUIDE.md`
- Configuration Examples: `config/play.yaml`, `config/evaluate.yaml`

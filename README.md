# SURE — Curated Bluesky Post Dataset

A pipeline for building a high-quality, topic-diverse dataset of Bluesky posts, ending with 2,550 curated posts across 17 topics annotated with style dimensions.

---

## Pipeline Overview

```
candidate_posts_eng10.jsonl.gz  (~2.2M filtered Bluesky posts)
        │
        ▼  step1_colab.py          [Colab T4 GPU]
        BGE-M3 embeddings
        → embeddings.npy  (2.2M × 1024)
        → post_ids.npy
        │
        ▼  step2_local.py          [Local]
        BERTopic clustering → 362 clusters, 17 hand-picked topics
        → post_topics.jsonl
        │
        ▼  lr_scorer.py            [Local]
        Logistic regression trained on 1,000 human labels
        Scores all posts in 17 topics, keeps top 500 per topic
        → probability_scores.jsonl  (98,953 posts scored)
        → candidate_pool_500.jsonl  (8,500 posts)
        │
        ▼  agent_qc.py             [Local]
        Claude Haiku scores each post 0.0–1.0 on quality
        Keeps top 150 per topic
        → rankings.jsonl    (8,500 posts ranked)
        → 2550_posts.jsonl  (2,550 posts)
        │
        ▼  label_axes.py           [Local]
        Claude Haiku annotates each post with 7 style dimensions
        → 2550_posts.jsonl  (updated in-place with axes_json)
```

---

## Scripts

| Script | Where to run | Description |
|---|---|---|
| `step1_colab.py` | Google Colab (T4 GPU) | Encodes 2.2M posts into 1024-dim BGE-M3 vectors |
| `step2_local.py` | Local | Clusters embeddings with BERTopic; assigns each post a topic |
| `lr_scorer.py` | Local | Trains logistic regression on human labels; scores all posts in 17 topics |
| `agent_qc.py` | Local | Uses Claude Haiku to score each post 0.0–1.0; selects top 150 per topic |
| `label_axes.py` | Local | Annotates each post with 7 style axes using Claude Haiku |

---

## Data Files

| File | Rows | Description |
|---|---|---|
| `all_labels.csv` | 1,000 | Human-labeled posts (label=1 good, label=0 bad) used to train the LR scorer |
| `probability_scores.jsonl` | 98,953 | LR probability score for every post in the 17 selected topics |
| `candidate_pool_500.jsonl` | 8,500 | Top 500 posts per topic by LR score (17 × 500) |
| `rankings.jsonl` | 8,500 | All candidate posts ranked by Claude Haiku agent score |
| `2550_posts.jsonl` | 2,550 | Final curated dataset — top 150 per topic with style axes |

---

## Raw Data

The pipeline starts from `candidate_posts_eng10.jsonl.gz` (~2.2M posts), pre-filtered from the Bluesky firehose to:

- ≥ 10 engagement
- English only
- ≥ 100 characters, ≥ 18 words
- No URLs, no replies / reposts / quotes
- No posts starting with `@`
- Max 3 hashtags, max 1 mention

This file is not included in the repo due to size.

---

## 17 Topics

Ancient History, Birds & Nature, Books & Reading, Capitalism, Climate & Energy, Economy & Jobs, Fitness & Gym, Food & Cooking, Gardening & Plants, Higher Education, Movies & Film, Music, Pets, Politics, Space & Astronomy, Sports, Video Games

---

## Quality Labeling Rubric

**Good (1)** — shares specific knowledge, insight, or a well-reasoned opinion; gives the reader something to think about; uses concrete facts, examples, or real observations; clearly in English.

**Bad (0)** — personal feelings or reactions with no substance; ranked / "top N" list format; not in English; low-effort commentary.

---

## Scaled Dataset — `scale_8000/`

An expanded version of the pipeline using the same 17 topics but 470 posts per topic (7,990 total), produced by deepening the candidate pool from 500 → 1,000 per topic and raising the agent QC cutoff from 150 → 470.

| File | Rows | Description |
|---|---|---|
| `scale_8000/candidate_pool_1000.jsonl` | 17,000 | Top 1,000 posts per topic by LR score |
| `scale_8000/rankings_8000.jsonl` | 17,000 | All candidates ranked by Claude Haiku agent score |
| `scale_8000/8000_posts.jsonl` | 7,990 | Final scaled dataset — top 470 per topic |

Agent score cutoffs per topic at 470 posts:

| Topic | Cutoff |
|---|---|
| Ancient History, Climate & Energy, Space & Astronomy | 0.70–0.75 |
| Birds & Nature, Food & Cooking, Gardening & Plants, Higher Education, Video Games | 0.65 |
| Economy & Jobs, Movies & Film, Sports | 0.60 |
| Books & Reading, Music | 0.55 |
| Capitalism, Fitness & Gym, Pets, Politics | 0.50 |

Note: style axis annotation (`axes_json`) is not yet included in `scale_8000/8000_posts.jsonl`.

---

## Style Axes (`axes_json` in `2550_posts.jsonl`)

Each post is annotated with 7 dimensions scored 0.0–1.0:

| Axis | 0 → 1 |
|---|---|
| `reading_level` | simple vocabulary → academic / complex |
| `background` | assumes no prior knowledge → assumes expert knowledge |
| `abstract_concrete` | vague general claims → specific facts / numbers |
| `tone` | analytical / neutral → emotional / charged |
| `humor` | earnest → witty / humorous |
| `narrativity` | pure argument → story / anecdote |
| `grounding` | direct statement → analogy / example-driven |


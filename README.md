# Feedback generation

This repository trains a small model to write tutor feedback on short-answer computer science questions. Given a question, the reference answer, the student's answer and which rubric key points were marked MET / NOT MET, the model writes a short paragraph telling the student what they got right, what is missing and how to improve.

A large **teacher** model writes the training data and a small **student** model is fine-tuned on it:

| Stage | Notebook | Model | What it does |
|---|---|---|---|
| 1. Teacher | [feedback_teacher_qwen3.5-27B.ipynb](feedback_teacher_qwen3.5-27B.ipynb) | Qwen3.5-27B-FP8 on vLLM | Writes feedback for all 3,778 rows and keeps only the feedback that passes quality checks. |
| 2. Student | [qlora_feedback_qwen3.5-9B.ipynb](qlora_feedback_qwen3.5-9B.ipynb) | Qwen3.5-9B with QLoRA | Fine-tunes on the teacher's feedback, then writes feedback for the unseen test set with and without the adapter. |
| 3. Judge | Last section of the teacher notebook | Qwen3.5-27B-FP8 | Scores the student's test-set feedback, base model against adapter. |

The trained adapter is on Hugging Face at `SmuFypTeam5/feedback`.

## Results

Measured on 389 test rows from questions that neither model saw in training.

| | Base Qwen3.5-9B | With feedback adapter |
|---|---|---|
| **Passes all four judge checks** | 79.7% | **89.7%** |
| Covers every missing key point | 94.9% | 97.2% |
| Does not treat a met key point as missing | 95.4% | 97.2% |
| Faithful to what the student wrote | 86.9% | 95.9% |
| Factually correct | 88.2% | 95.4% |
| Passes the format checks (no markdown, grading terms or overlong text) | 44.2% | 100% |
| ROUGE-L against the teacher's feedback | 0.299 | 0.392 |
| Mean length (words) | 90 | 70 |

The first five rows come from the Qwen3.5-27B judge. The last three are automatic checks that the student notebook computed when it ran on the cluster.

To see what the numbers mean in practice, read [Results/side_by_side_examples.md](Results/side_by_side_examples.md). It shows eight test rows with the feedback from both models, including two where the adapter did worse.

## Where to find things

```
feedback/
├── feedback_teacher_qwen3.5-27B.ipynb   Stage 1 and 3, saved with its outputs
├── qlora_feedback_qwen3.5-9B.ipynb      Stage 2, saved without outputs
├── Datasets/
│   ├── training_data_moh_eng_rubrics.csv    Source data
│   └── teacher_feedback/                    Everything the pipeline produced
├── Results/
│   └── side_by_side_examples.md             Base vs adapter on eight test rows
└── Archive/                                 Earlier experiments, not part of the final pipeline
```

| If you want to see | Look at |
|---|---|
| The teacher and judge prompts | Teacher notebook, section *Prompts* |
| The quality checks and why they were designed this way | Teacher notebook, sections *Prompts* and *Quality checks* |
| How much teacher feedback passed, and why the rest failed | Teacher notebook, section *Review* |
| The student prompt and LoRA settings | Student notebook, sections *Configuration* and *Prompt* |
| How the automatic checks are computed | Student notebook, section *Generation and automatic checks* |
| The judge's verdict on every test row | `Datasets/teacher_feedback/student_test_judgements.jsonl` |

## Data

**Source:** `Datasets/training_data_moh_eng_rubrics.csv` has 3,778 graded student answers to 95 questions. Each row has the question, the reference answer (`desired_answer`), the student's answer, the rubric key points (`criteria[]`) and the mark for each one (`criteria_score[]`).

**Pipeline outputs** in `Datasets/teacher_feedback/`:

| File | Contents |
|---|---|
| `training_data_moh_eng_rubrics_with_feedback.csv` | The source data plus `feedback` and `feedback_passed_checks`. This is the student's training data. |
| `training_data_moh_eng_rubrics_generations.jsonl` | Every draft the teacher wrote (6,308), with its rule-check result. |
| `training_data_moh_eng_rubrics_judgements.jsonl` | Every judge verdict on those drafts (4,037). |
| `student_test_judgements.jsonl` | The feedback from the base model and the adapter on the 389 test rows, with the judge's verdict on each (778 lines). |
| `sft_train.jsonl`, `sft_val.jsonl` | The teacher feedback in chat format, with the teacher notebook's own 90/10 split. Not used by the student notebook, because this split overlaps the test questions. |

## Method

**Teacher.** For each row the teacher writes one paragraph of feedback. Every draft must pass two gates:

- **Rule checks:** length, truncation, markdown, non-English text, stock phrases, and grading terms such as "rubric" or "marks".
- **LLM judge:** the same 27B model in thinking mode, with a separate auditing prompt. It checks four things: every missing point is covered, no met point is treated as missing, the student is not credited with something they didn't write, and the technical content is correct.

A draft that fails is regenerated with a new seed and a different opening and closing style, up to 4 attempts. In the end 3,533 of 3,778 rows (93.5%) passed. Only these are used to train the student.

**Student.** QLoRA on Qwen3.5-9B: 4-bit base model, LoRA rank 16, alpha 32, learning rate 1e-4, loss on the feedback text only. Questions are split 80/10/10 with seed 42, which gives 2,782 training rows, 386 validation rows and 389 test rows. Training stopped early after about 2 epochs, and the checkpoint with the lowest validation loss (0.717) was kept.

The base model and the question split are the same as the grading adapter's (in the project's `qlora/` folder). One vLLM server can therefore serve both adapters, and both are tested on the same unseen questions.

**Judge.** The student's test-set feedback goes through the same four judge checks the teacher's feedback had to pass. The base model gets the same prompt and decoding settings as the adapter, so the comparison isolates the effect of fine-tuning.

## Reproducing

Both notebooks need a Linux machine with a CUDA GPU. They were run on one NVIDIA L40S (48 GB). Use a separate Python environment for each, because vLLM pins its own PyTorch build.

1. Run the teacher notebook up to *Export*. Generation and judging are checkpointed, so an interrupted run resumes where it stopped.
2. Run the student notebook. It trains the adapter, writes the test-set feedback to `Datasets/teacher_feedback/student_test_predictions.jsonl` and uploads the adapter.
3. Run the teacher notebook again. Finished rows are reused, and the last section judges the student's feedback.

Run the notebooks from this folder or from its parent folder. For a non-interactive run (such as papermill), set `HF_TOKEN` before the upload step.

`student_test_predictions.jsonl` is not in the repository. The same feedback text is in `student_test_judgements.jsonl`.

## Archive

Earlier prompting experiments on older datasets, kept for reference. They are not part of the final pipeline, and their notebooks still point to the file paths used at the time.

| Path | Contents |
|---|---|
| `Archive/Qwen3.5-4B/` | First attempt: Qwen3.5-4B prompted directly, with its 50 sample outputs. |
| `Archive/Qwen3.5-9B/` | Qwen3.5-9B prompted directly with two prompt versions. `prompt_A/` and `prompt_B/` each hold the prompt and its 50 sample outputs. |
| `Archive/Datasets/` | The older datasets these experiments used. |

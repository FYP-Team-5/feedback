# Feedback generation

Trains a small model to write tutor feedback on short-answer questions, given the question, the reference answer, the student's answer and which rubric key points were marked MET / NOT MET.

1. **Teacher** (`feedback_teacher_qwen3.5-27B.ipynb`): Qwen3.5-27B-FP8 on vLLM writes feedback for every row of `Datasets/training_data_moh_eng_rubrics.csv`. Each draft goes through rule checks and an LLM judge, and is regenerated up to 4 times until it passes. Its last section judges the student's test-set outputs.
2. **Student** (`qlora_feedback_qwen3.5-9B.ipynb`): QLoRA on Qwen3.5-9B, trained on the teacher feedback that passed. It uses the same base model and the same 80/10/10 question split (seed 42) as the grading adapter in `qlora/`, so one vLLM server can serve both adapters and both share one unseen test set. The adapter is on Hugging Face at `SmuFypTeam5/feedback`.

## Run order

1. Run the teacher notebook up to the export. Generation and judging are checkpointed to JSONL and resume after an interruption.
2. Run the student notebook. It trains the adapter, writes the test-set outputs of the base model and the adapter to `Datasets/teacher_feedback/student_test_predictions.jsonl`, and uploads the adapter.
3. Run the teacher notebook again. Finished rows are reused, and the last section judges the student outputs.

Both notebooks expect the project root as the working directory, with this folder at `feedback/`. On the cluster they run under papermill, so set `HF_TOKEN` before the upload: papermill cannot answer a login prompt.

## Layout

| Path | Contents |
|---|---|
| `Datasets/training_data_moh_eng_rubrics.csv` | Source data: 3,778 graded answers to 95 questions. |
| `Datasets/teacher_feedback/` | Teacher outputs: `*_generations.jsonl` and `*_judgements.jsonl` (every attempt), `*_with_feedback.csv` (the source plus `feedback` and `feedback_passed_checks`), and `student_test_judgements.jsonl` (judge results for the student). |
| `runs/` | Executed notebooks, kept as a record of results. |
| `Archive/` | Earlier experiments: Qwen3.5-4B and Qwen3.5-9B prompting on the older datasets. |

`sft_train.jsonl` and `sft_val.jsonl` use the teacher notebook's own 90/10 split, which includes the grading model's test questions. The student notebook doesn't use them.

## Results (389 test rows)

| | Base Qwen3.5-9B | Feedback adapter |
|---|---|---|
| Passes all four judge checks | 79.7% | 89.7% |
| Faithful to the student's answer | 86.9% | 95.9% |
| Factually correct | 88.2% | 95.4% |
| Passes the format checks | 44.2% | 100% |
| ROUGE-L vs teacher | 0.299 | 0.392 |

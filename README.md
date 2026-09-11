# Feedback generation

`feedback.ipynb` processes `Datasets/training_data_mol.csv` one row at a time with `Qwen/Qwen3.5-9B`.

Run the notebook in order:

1. Load and validate the dataset. Source rows with inconsistent criteria-array lengths are retained and reported as warnings.
2. Run the dependency cell once, then load the tokenizer and model.
3. Run `run_feedback()` for the full dataset, or `run_feedback(limit=5)` for a smoke test.

The output is written to `Datasets/training_data_mol_with_feedback.csv`, preserving the original eight columns and adding `feedback` as the final column. Raw per-row results are appended to `Datasets/training_data_mol_feedback.jsonl` so interrupted runs can resume.

Qwen3.5-9B is resource intensive. A CUDA-enabled PyTorch environment with enough VRAM is recommended; the notebook uses 4-bit NF4 loading when CUDA is available.
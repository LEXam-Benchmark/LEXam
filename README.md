<div align="center" style="display: flex; align-items: center; justify-content: center; gap: 16px;">
  <img src="pictures/logo.png" alt="LEXam Logo" width="120" style="border: none;">
  <div style="text-align: left;">
    <h1 style="margin: 0;">LEXam: Benchmarking Legal Reasoning on 340 Law Exams</h1>
    <p style="margin: 6px 0 0;">A diverse, rigorous evaluation suite for legal AI from Swiss, EU, and international law examinations.</p>
  </div>
</div>

### This repository provides code for evaluating LLMs on ***LEXam***. 

[//]: [![Website](https://img.shields.io/badge/Website-lexam--benchmark.github.io-blue)](https://lexam-benchmark.github.io)

[![Homepage](https://img.shields.io/badge/LEXam-Homepage-blue)](https://lexam-benchmark.github.io/)
[![Data](https://img.shields.io/badge/Data-Hugging%20Face-FFD21E)](https://huggingface.co/datasets/LEXam-Benchmark/LEXam)
[![arXiv](https://img.shields.io/badge/arXiv-2505.12864-b31b1b)](https://arxiv.org/abs/2505.12864)
[![license](https://img.shields.io/github/license/LEXam-Benchmark/LEXam?label=Code%20License)](https://github.com/LEXam-Benchmark/LEXam/blob/main/LICENSE)
[![Data License](https://img.shields.io/badge/Data%20License-CC--BY--4.0-orange.svg)](https://github.com/LEXam-Benchmark/LEXam/blob/main/LICENSE_DATA)

## 🔥 News
- [2026/01] Our paper has been accepted to ***ICLR 2026!***
- [2025/12] We reorganized all multiple-choice questions into four separate files, `mcq_4_choices` (n = 1,655), `mcq_8_choices` (n = 1,463), `mcq_16_choices` (n = 1,028), and `mcq_32_choices` (n = 550), all with standardized features.
- [2025/11] We identified and corrected several annotation errors in the statements of the original multiple-choice questions.
- [2025/09] We updated our evaluation results on open questions using an ensemble LLM-as-A-Judge (GPT-4o + DeepSeek-V3 + Qwen3-32B). 
- [2025/05] Release of the first version of [paper](https://arxiv.org/abs/2505.12864), where we evaluate representative SoTA LLMs with evaluations stricly verified by legal experts.

## 🔄 Reproducing Paper Results or Evaluating Your Own Language Model

### Environment Preparation
```shell
git clone https://github.com/LEXam-Benchmark/LEXam
cd LEXam
conda create -n lexam python=3.11
conda activate lexam
cd lighteval
pip install -e .[dev]
cd ..
pip install -r requirements.txt

# Set API keys for inference and evaluation.
# OpenAI key is mandatory for our expert-verified grader, which is based on GPT-4o
EXPORT OPENAI_API_KEY="xxx"
EXPORT TOGETHER_API_KEY="xxx"
EXPORT DEEPSEEK_API_KEY="xxx"
EXPORT ANTHROPIC_API_KEY="xxx"
EXPORT GEMINI_API_KEY="xxx"
```

### Evaluating Non-Reasoning LLMs with [[Huggingface lighteval]](https://huggingface.co/docs/lighteval/index)
Huggingface lighteval provides the advantage of uniformly evaluating LLMs from different endpoints -- local vLLM, OpenAI, Anthropic, TogetherAI, Gemini ...

Together-AI, OpenAI, Gemini, and other API-based LLMs can be evaluated by:
```shell
MODEL="openai/gpt-4o-mini-2024-07-18" 

# Evaluating GPT-4o-mini on LEXam Open Question subset.
python -m lighteval endpoint litellm "${MODEL}" "community|lexamoq_open_question|0|0" --custom-tasks lighteval/community_tasks/lexam_oq_evals.py --output-dir outputs_oq --save-details --use-chat-template

# Evaluating GPT-4o-mini on LEXam Multiple-Choice Question subset.
python -m lighteval endpoint litellm "${MODEL}" "community|lexammcq_mcq_4_choices|0|0" --custom-tasks lighteval/community_tasks/lexam_mcq_evals.py --output-dir outputs_mcq --save-details --use-chat-template
```
- `MODEL`: the target LLM you are evaluating, e.g., `openai/gpt-4.1`, `together_ai/meta-llama/Llama-4-Maverick-17B-128E-Instruct-FP8`
- `--output-dir`: evaluation results will be saved to `--output-dir`.
- `--save-details`: details including prompts, LLM responses, LLM judges, and other evaluation metrics will be saved in `details`.

Local inference using vLLM:
```shell
MODEL="meta-llama/Llama-3.1-8B-Instruct" 
export HF_HOME="xxx"
export HUGGINGFACE_TOKEN="xxx"
huggingface-cli login --token $HUGGINGFACE_TOKEN

# Evaluating GPT-4o-mini on LEXam Open Question subset.
python -m lighteval vllm "pretrained=${MODEL},trust_remote_code=True,dtype=bfloat16" "community|lexamoq_open_question|0|0" --custom-tasks lighteval/community_tasks/lexam_oq_evals.py --output-dir outputs_oq --save-details --use-chat-template

# Evaluating GPT-4o-mini on LEXam Multiple-Choice Question subset.
python -m lighteval vllm "pretrained=${MODEL},trust_remote_code=True,dtype=bfloat16" "community|lexammcq_mcq_4_choices|0|0" --custom-tasks lighteval/community_tasks/lexam_mcq_evals.py --output-dir outputs_mcq --save-details --use-chat-template
```

### Evaluating Reasoning LLMs with LiteLLM directly.
Reasoning LLMs generate both a <think> scratch pad and the final answer after </think>. To only evaluate the answer, we do not use lighteval for reasoning LLMs.
```shell
MODEL="deepseek-reasoner"
python litellm_eval.py --input_file data/open_questions_test.xlsx --cache_name r1 --llm $MODEL --output_file lexam_oq_${MODEL}.csv --batch_size 2 --task_type open_quesitons
python litellm_eval.py --input_file data/MCQs_test.xlsx --cache_name r1 --llm $MODEL --output_file lexam_mcq_${MODEL}.csv --batch_size 2 --answer_field gold --task_type mcq_letters
```
- `MODEL` can be set to any model included in `MODEL_DICT` of `litellm_eval.py`, e.g., `o1`, `o3-mini`, `qwq-32b`.
- `--output_file`: DeepSeek-R1's answer to open/MC questions will be at `lexam_oq_deepseek-reasoner.csv` and `lexam_mcq_deepseek-reasoner.csv`
- `--task_type`: chose from ['mcq_letters', 'mcq_numbers', 'open_questions']. mcq_letters and _numbers differ by using ABCD or 1234 as choice labels.

Then evaluate the answers using our expert-verified LLM judge. This script will print the Mean and bootstrapped Variance of open question performance.
```shell
MODEL="deepseek-reasoner"
python customized_judge_async.py --input_file lexam_oq_${MODEL}.csv --output_file lexam_oq_${MODEL}_graded.csv --async_call --cache_name gpt4o --llm gpt-4o
```
- `--input_file`: Grade DeepSeek-R1's answer to open questions. Grading results at `lexam_oq_deepseek-reasoner_graded.csv`


Finally evaluate the accuracy of MCQs. This script will print accuracy and bootstrapped variance. No LLM call is involved in this script.
```shell
MODEL="deepseek-reasoner"
INPUT_FILE="lexam_mcq_${MODEL}.csv"
python evaluation.py --input_file $INPUT_FILE --response_field ${MODEL}_answer --task_type mcq_letters
```
## Licenses

- The **Code** in this repository is licensed under the [Apache License 2.0](LICENSE).
- The **Data** in this repository is licensed under the [Creative Commons Attribution 4.0 International License](LICENSE_DATA).

## Citation

If you find the dataset helpful, please consider citing ***LEXam***: 
```shell
@article{fan2025lexam,
  title      =   {LEXam: Benchmarking Legal Reasoning on 340 Law Exams},
  author     =   {Fan, Yu and Ni, Jingwei and Merane, Jakob and Tian, Yang and Hermstr{\"u}wer, Yoan and Huang, Yinya and Akhtar, Mubashara and Salimbeni, Etienne and Geering, Florian and Dreyer, Oliver and Brunner, Daniel and Leippold, Markus and Sachan, Mrinmaya and Stremitzer, Alexander and Engel, Christoph and Ash, Elliott and Niklaus, Joel},
  journal    =   {arXiv preprint arXiv:2505.12864},
  year       =   {2025}
}
```

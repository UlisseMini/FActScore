# FActScore

[![made-with-python](https://img.shields.io/badge/Made%20with-Python-red.svg)](#python)
[![PyPI version factscore](https://badge.fury.io/py/factscore.svg)](https://pypi.python.org/pypi/factscore/)
[![Downloads](https://pepy.tech/badge/factscore)](https://pepy.tech/project/factscore)

This is the official release accompanying our preprint, [FActScore: Fine-grained Atomic Evaluation of Factual Precision in Long Form Text Generation](https://tinyurl.com/FActScore). FActScore is available as a PIP package as well.

## Install
<!-- ```
conda create -n fs-env python=3.9
conda activate fs-env
pip install -r requirements.txt
``` -->

Make a new Python 3.7+ environment using `virtualenv` or `conda`.

```bash
pip install factscore
python -m spacy download en_core_web_sm
```

## Download the data

```bash
python -m factscore.download_data
```

Or, download it manually from this [Google Drive link](https://drive.google.com/drive/folders/1bLHGu_imkZVtX6O0mpZ-G0-4ofTLM1ZA?usp=sharing). Make a cache directory `.cache/factscore`, and place unzipped `demos` and `enwiki-20230401.db` in that directory.

## Running FActScore using a command line

We expect running FActScore costs about $1 of the API cost per 100 sentences. For instance, if you have 100 generations, each with 5 sentences on average, it costs $5 in total. 

```bash
python -m factscore.factscorer --input_path {input_path} --data_dir {data_dir} --model_name {estimator_name} --cache_dir {cache_dir} --openai_key {openai_key}
```

- `input_path` can be something like `data/unlabeled/InstructGPT.jsonl`. It should be a `.jsonl` format where each line contains `topic` (a topic entity that corresponds to the Wikipedia title) and `output` (a generation from the model).
- `model_name`: `retrieval+ChatGPT`, `retrieval+ChatGPT+npm`, two more configs (`retrieval+llama`, `retrieval+llama+npm`) coming soon!
- `data_dir`: Directory containing knowledge source, etc. `data` by default.
- `cache_dir`: Directory containing cache from API/models. `.cache/factscore` by default.
- `openai_key`: File containing OpenAI API Key.
- `use_atomic_facts`: If specified, it uses model-generated atomic facts released as part of our data instead of running the atomic fact generator. You can't specify it if you are running new model generations.
- `n_samples`: If specified, it runs the model on a subset of the data.
- `verbose`: If specified, it shows the progress bar.

For example,

```python
python -m factscore.factscorer \
    --input_path data/unlabeled/InstructGPT.jsonl \
    --model_name "retrieval+ChatGPT" \
    --data_dir "data" \
    --cache_dir ".cache/factscore" \
    --openai_key "api.key" \
    --verbose
```
It uses `enwiki-20230401` by default, and will download the database from our Google drive.

Instructions to use Instruct-LLAMA-7B or your own LM coming soon!

## To evaluate your own LM

There're two sets of prompt entities, `data/labeled/prompt_entities.txt` (183 entities) and `data/unlabeled/prompt_entities.txt` (500 entities). Each line contains the name of the person (which is also a corresponding Wikipedia title). You can use the labeled version if you want to be compatible with the data under `data/labeled` (Section 3 and Section 4.2 in the paper), and use the unlabeled version if you want to be compatible with the data under `data/unlabeled` (Section 4.3 in the paper).

You can prompt your LM with your own prompt (we used `Question: Tell me a bio of <entity>.`) and use the following code.

```python
from factscore.factscorer import FactScorer

fs = FactScorer(openai_key="...")

# topics: list of strings (human entities used to generate bios)
# generations: list of strings (model generations)
out = fs.get_score(topics, generations)
print (out["score"]) # FActScore
print (out["respond_ratio"]) # % of responding (not abstaining from answering)
print (out["num_facts_per_response"]) # average number of atomic facts per response
```

Alternatively, you can create a .jsonl file, where each line has `topic` (entity name, exactly same as the one from `.txt` file) and `output` (generation from LM), and then use a command line [above](#Running-FActScore-using-a-command-line).

We recommend using (A) `FactScorer(model_name="retrieval+ChatGPT")` (default) or (B) `FactScorer(model_name="retrieval+llama+npm")`. They have 0.99 Pearson correlation. Here're results of a range of models, which you can easily reproduce through [these command lines](#Running-FActScore-using-a-command-line).

| Model | FActScore from (A) | FActScore from (B) |
|---|---|---|
| [GPT-4](https://arxiv.org/abs/2303.08774) | 73.1 | 59.9 |
| [ChatGPT](https://openai.com/blog/chatgpt) | 71.6 | 60.4 |
| [Alpaca 65B](https://crfm.stanford.edu/2023/03/13/alpaca.html) | 55.6 | 46.3 |
| [InstructGPT](https://openai.com/research/instruction-following) | 52.8 | 41.7 |
| [Alpaca 13B](https://crfm.stanford.edu/2023/03/13/alpaca.html) | 47.7 | 40.3 |
| [Vicuna 13B](https://lmsys.org/blog/2023-03-30-vicuna/) | 46.6 | 40.7 |
| [Alpaca 7B](https://crfm.stanford.edu/2023/03/13/alpaca.html) | 39.7 | 36.5 |
| [Vicuna 7B](https://lmsys.org/blog/2023-03-30-vicuna/) | 38.9 | 36.9 |
| [MPT Chat 7B](https://www.mosaicml.com/blog/mpt-7b) | 30.1 | 27.9 |
| [Oasst Pythia 12B](https://huggingface.co/OpenAssistant/oasst-sft-1-pythia-12b) | 25.1 | 20.8 |
| [Dolly 12B](https://huggingface.co/databricks/dolly-v2-12b) | 21.7 | 17.1 |
| [StableLM tuned 7B](https://huggingface.co/stabilityai/stablelm-tuned-alpha-7b) | 17.3 | 16.3 |

```python
from factscore.factscorer import FactScorer

fs = FactScorer(openai_key="...")

scoreA = fs.get_score(topics, modelA_generations)["score"] # 0.667
scoreB = fs.get_score(topics, modelB_generations)["score"] # 0.010
```

## To use a custom knowledge source

By default, FActScore uses Wikipedia dump from 2023/04/01. But you can also use your own knowledge source!

The knolwedge source should be ready in a `.jsonl` format, where each line is a dictionary containing `title` and `text`. `text` can either be a string or a list of strings (e.g., sections).

```python
from factscore.factscorer import FactScorer

fs = FactScorer()

# this will create a database using your file
# for English Wikipedia (18GB)), it takes ~8 hours
# once DB file is created, you can reuse it by only specifying `db_path`
fs.register_knowledge_source(name_of_your_knowledge_source,
                             data_path=path_to_jsonl_file,
                             db_path=path_to_output_db_file)

# now, when you compute a score, specify knowledge source to use
out = fs.get_score(topics, generations, knowledge_source=name_of_your_knowledge_source)
print (out["score"]) # FActScore
print (out["respond_ratio"]) # % of responding (not abstaining from answering)
print (out["num_facts_per_response"]) # average number of atomic facts per response
```



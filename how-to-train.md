---
title: How to train a new language model from scratch using Transformers and Tokenizers
thumbnail: assets/EsperBERTo-thumbnail.png
---

# How to train a new language model from scratch using Transformers and Tokenizers

<small>Published Feb 14, 2020.</small>


Over the past few weeks, we made several improvements to our [`transformers`](https://github.com/huggingface/transformers) and [`tokenizers`](https://github.com/huggingface/tokenizers) libraries, with the goal of making it way easier to **train a new language model from scratch**.

In this post we'll demo how to train a “small” model (84 M parameters = 6 layers, 768 hidden size, 12 attention heads) – that’s the same number of layers & heads as DistilBERT – on **Esperanto**. We'll then fine-tune the model on a downstream task of part-of-speech tagging.

Esperanto is a *constructed language* with a goal of being easy to learn. We pick it for this demo for several reasons:
- it is a relatively low-resource language (even though it's spoken by ~2 million people) so this demo is less boring than training one more English model 😁
- its grammar is highly regular (e.g. all common nouns end in -o, all adjectives in -a) so we should get interesting linguistic results even on a small dataset.
- finally, the overarching goal at the foundation of the language is to bring people closer (fostering world peace and international understanding) which one could argue is aligned with the goal of the NLP community 💚

> N.B. You won't need to understand Esperanto to understand this post, but if you do want to learn it, [Duolingo](https://www.duolingo.com/enroll/eo/en/Learn-Esperanto) has a nice course with 280k active learners.

Our model is going to be called… wait for it… **EsperBERTo** 😂

<img src="/blog/assets/eo.svg" alt="Esperanto flag" style="margin: auto; display: block; width: 260px;">

## 1. Find a dataset

First, let us find a corpus of text in Esperanto. Here we'll use the Esperanto portion of the [OSCAR corpus](https://traces1.inria.fr/oscar/) from INRIA.
OSCAR is a huge multilingual corpus obtained by language classification and filtering of [Common Crawl](https://commoncrawl.org/) dumps of the Web.

<img src="/blog/assets/oscar.png" style="margin: auto; display: block; width: 260px;">

The Esperanto portion of the dataset is only 299M, so we'll concatenate with the Esperanto sub-corpus of the [Leipzig Corpora Collection](https://wortschatz.uni-leipzig.de/en/download), which is comprised of text from diverse sources like news, literature, and wikipedia.

The final training corpus has a size of 3 GB, which is still small – for your model, you will get better results the more data you can get to pretrain on. 


## 2. Train a tokenizer

We choose to train a byte-level Byte-pair encoding tokenizer (the same as GPT-2), with the same special tokens as RoBERTa. Let's pick its size to be 52,000.

We recommend training a byte-level BPE as 

```python
from pathlib import Path

from tokenizers.implementations.byte_level_bpe import ByteLevelBPETokenizer

paths = [str(x) for x in Path("./epo-data/").glob("**/*.txt")]

# Initialize a tokenizer
tokenizer = ByteLevelBPETokenizer()

# Customize training
tokenizer.train(files=paths, vocab_size=52_000, min_frequency=2, special_tokens=[
    "<s>",
    "<pad>",
    "</s>",
    "<unk>",
    "<mask>",
])

# Save files to disk
tokenizer.save(".", "esperberto")
```

And here's a slightly accelerated capture of the output:

![tokenizers](assets/tokenizers-fast.gif)

🔥🔥 Wow, that was fast! ⚡️🔥

We now have both a `vocab.json`, which is a list of the most frequent tokens ranked by frequency, and a `merges.txt` list of merges.

```json
{
	"<s>": 0,
	"<pad>": 1,
	"</s>": 2,
	"<unk>": 3,
	"<mask>": 4,
	"!": 5,
	"\"": 6,
	"#": 7,
	"$": 8,
	"%": 9,
	"&": 10,
	"'": 11,
	"(": 12,
	")": 13,
	# ...
}

# merges.txt
l a
Ġ k
o n
Ġ la
t a
Ġ e
Ġ d
Ġ p
# ...
```

What is great is that our tokenizer is optimized for Esperanto. Compared to a generic tokenizer trained for English, more native words are represented by a single, unsplit token. We also represent sequences in a more efficient manner. Here on this corpus, the average length of encoded sequences is ~30% smaller as when using the pretrained GPT-2 tokenizer.

Here's  how you can use it in `tokenizers`, including handling the RoBERTa special tokens – of course, you'll also be able to use it direcly from `transformers`.

```python
from tokenizers.implementations import ByteLevelBPETokenizer
from tokenizers.processors import BertProcessing


tokenizer = ByteLevelBPETokenizer(
    "./models/EsperBERTo-small/vocab.json",
    "./models/EsperBERTo-small/merges.txt",
)
tokenizer._tokenizer.post_processor = BertProcessing(
    ("</s>", tokenizer.token_to_id("</s>")),
    ("<s>", tokenizer.token_to_id("<s>")),
)
tokenizer.enable_truncation(max_length=512)

print(
    tokenizer.encode("Mi estas Julien.")
)
# Encoding(num_tokens=7, ...)
# tokens: ['<s>', 'Mi', 'Ġestas', 'ĠJuli', 'en', '.', '</s>']
```

## 3. Train a language model from scratch

We will now train our language model using the `run_language_modeling.py` script from `transformers` (newly renamed from `run_lm_finetuning.py` as it now supports training from scratch more seamlessly).

> We'll train a RoBERTa-like model, which is a BERT-like with a couple of changes (check the [documentation](https://huggingface.co/transformers/model_doc/roberta.html) for more details).

As the model is BERT-like, we'll train it on a task of *Masked language modeling*, i.e. the predict how to fill arbitrary tokens that we randomly mask in the dataset. This is taken care of by the example script.

We just need to do two things:
- implement a simple subclass of `Dataset` that loads data from our text files
	- Depending on your use case, you might not even need to write your own subclass of Dataset, if one of the provided examples (`TextDataset` and `LineByLineTextDataset`) works – but there are lots of custom tweaks that you might want to add based on what your corpus looks like.
- Choose and experiment with different sets of hyperparameters.


Here's a simple version of our EsperantoDataset. If your dataset is very large, you can opt to load and tokenize on the fly, not as a preprocessing step like here.

```python
class EsperantoDataset(Dataset):
    def __init__(self, evaluate: bool = false):
        tokenizer = ByteLevelBPETokenizer(
            "./models/EsperBERTo-small/vocab.json",
            "./models/EsperBERTo-small/merges.txt",
        )
        tokenizer._tokenizer.post_processor = BertProcessing(
            ("</s>", tokenizer.token_to_id("</s>")),
            ("<s>", tokenizer.token_to_id("<s>")),
        )
        tokenizer.enable_truncation(max_length=512)

        self.examples = []

        src_files = Path("./data/").glob("*-eval.txt") if evaluate else Path("./data/").glob("*-train.txt")
        for src_file in src_files:
            print("🔥", src_file)
            lines = src_file.open(encoding="utf-8").read().splitlines()
            self.examples += [x.ids for x in tokenizer.encode_batch(lines)]

    def __len__(self):
        return len(self.examples)

    def __getitem__(self, i):
        # We'll pad at the batch level.
        return torch.tensor(self.examples[i])
```

Here is one specific set of **hyper-parameters and arguments** we pass to the script:

```
	--output_dir ./models/EsperBERTo-small-v1
	--model_type roberta
	--mlm
	--config_name ./models/EsperBERTo-small
	--tokenizer_name ./models/EsperBERTo-small
	--do_train
	--do_eval
	--learning_rate 1e-4
	--num_train_epochs 5
	--save_total_limit 2
	--save_steps 2000
	--per_gpu_train_batch_size 16
	--evaluate_during_training
	--seed 42
```

As usual, pick the largest batch size you can fit on your GPU(s). 

**🔥🔥🔥 Let's start training!! 🔥🔥🔥**

Here you can check our Tensorboard for [one particular set of hyper-parameters](https://tensorboard.dev/experiment/8AjtzdgPR1qG6bDIe1eKfw/#scalars):

[![tb](assets/tensorboard.png)](https://tensorboard.dev/experiment/8AjtzdgPR1qG6bDIe1eKfw/#scalars)

## 4. Check that the LM actually trained

Aside from looking at the training and eval losses going down, the easiest way to check whether our language model is learning anything interesting is via the `FillMaskPipeline`.

Pipelines are simple wrappers around tokenizers and models, and the 'fill-mask' one will let you input a sequence containing a masked token (here, `<mask>`) and return a list of the most probable filled sequences, with their probabilities.

```python
from transformers import pipeline

fill_mask = pipeline(
    "fill-mask",
    model="./models/EspertBERTo-small",
    tokenizer="./models/EspertBERTo-small"
)

result = fill_mask("La suno <mask>.")
# {'score': 0.05867236852645874, 'sequence': '<s> La suno malaperis.</s>', 'token': 4286}
# {'score': 0.05144098773598671, 'sequence': '<s> La suno estas.</s>', 'token': 315}
# {'score': 0.03619285672903061, 'sequence': '<s> La suno dormas.</s>', 'token': 12445}
# {'score': 0.027554208412766457, 'sequence': '<s> La suno kreskas.</s>', 'token': 5019}
# {'score': 0.014812440611422062, 'sequence': '<s> La suno okazis.</s>', 'token': 1180}
```

```python
fill_mask("Bonan <mask>, virinojn!")

# Good <mask>, ladies!

# {'score': 0.02966943569481373, 'sequence': '<s> Bonan vorton, virinojn!</s>', 'token': 3246}
# {'score': 0.02172263152897358, 'sequence': '<s> Bonan tagon, virinojn!</s>', 'token': 2960}
# {'score': 0.020237216725945473, 'sequence': '<s> Bonan amon, virinojn!</s>', 'token': 6194}
# {'score': 0.01287781447172165, 'sequence': '<s> Bonan leteron, virinojn!</s>', 'token': 6098}
# {'score': 0.012435879558324814, 'sequence': '<s> Bonan vesperon, virinojn!</s>', 'token': 10577}
```

TODO replace with an actual Esperanto example or two (preferentially fun)

## 5. Fine-tune your LM on a downstream task

We now can fine-tune our new Esperanto language model on a downstream task of Part-of-speech tagging.

todo todo todo

Example of `TokenClassificationPipeline`

yadda yada

## 6. Share your model 🎉

- upload your model using the CLI
- write a README.md model card and add it to the repo under `model_cards/`. Your model card should ideally include a model description, training params (dataset, preprocessing, hyperparameters), evaluation results, intended uses & limitations, etc.


**TADA!**

[![tb](assets/model_page.png)](https://tensorboard.dev/experiment/8AjtzdgPR1qG6bDIe1eKfw/#scalars)

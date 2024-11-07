---
layout: post
title:  "🤗Huggingface summary"
date:   2024-10-06 12:00:00 +0100
categories: hugginface api
---
_(Edited: Nov 7, 2024)_

_DISCLAIMER: the information here might not be completely correct. Please, if you find any error open an [issue](https://github.com/mkmenta/mkmenta.github.io/issues). Thanks!_

This is just an overview and summary of the huggingface docs.

```
transformers==4.45.1
datasets==3.0.2
```

## Easy inference
To do simple inference use the [pipeline](https://huggingface.co/docs/transformers/main_classes/pipelines).
- You can do it by task (find all the tasks in the [task](https://huggingface.co/docs/transformers/main_classes/pipelines#transformers.pipeline) parameter docs of the API), and it will load a default model unless you specify it.
- Or you can do it by model, if it has its task defined in the hub.

```python
from transformers import pipeline

pipe = pipeline(task="text-generation", model="unsloth/Llama-3.2-1B-Instruct")
print(pipe("Is 9.11 larger than 9.9?"))
```

Output:
```
[{'generated_text': 'Is 9.11 larger than 9.9? No, it is not. '}]
```

## Loading a model
Using the Auto classes should be enough (they will detect the correct classes to use). However, sometimes like in the following case, there might be several possibilities and it might load the base class and not the one with the Causal Language Modeling head:

```python
from transformers import AutoConfig, AutoModel, AutoTokenizer, LlamaForCausalLM

config = AutoConfig.from_pretrained("unsloth/Llama-3.2-1B-Instruct")
model = AutoModel.from_pretrained("unsloth/Llama-3.2-1B-Instruct")
tokenizer = AutoTokenizer.from_pretrained("unsloth/Llama-3.2-1B-Instruct")

print(type(config))
print(type(model))
print(type(tokenizer))

model = LlamaForCausalLM.from_pretrained("unsloth/Llama-3.2-1B-Instruct")
print(type(model))
```

Output:
```
<class 'transformers.models.llama.configuration_llama.LlamaConfig'>
<class 'transformers.models.llama.modeling_llama.LlamaModel'>
<class 'transformers.tokenization_utils_fast.PreTrainedTokenizerFast'>
<class 'transformers.models.llama.modeling_llama.LlamaForCausalLM'>
```
then you can go to the [docs]([https://huggingface.co/docs/transformers/v4.45.1/en/model_doc/clip#transformers.CLIPModel](https://huggingface.co/docs/transformers/v4.45.1/en/model_doc/llama#transformers.LlamaForCausalLM)) and check the details on how to use it. Note that many models have their own classes, like in this case, that inherit from general classes from huggingface. So, you might need to check the parent class [GenerationMixin](https://huggingface.co/docs/transformers/main/en/main_classes/text_generation#transformers.GenerationMixin) to see the useful API.

```python
model = model.to('cuda')
text_inputs = tokenizer(["Is 9.11 larger than 9.9?"], return_tensors="pt").to('cuda')
outputs = model.generate(**text_inputs, max_new_tokens=32)
print(tokenizer.decode(outputs[0]))
```

Output:
```
<|begin_of_text|>Is 9.11 larger than 9.9? No, it is not.
9.11 is larger than 9.9.
9.11 is larger than 9.9.
9.11 is
```
NOTE: specifically for the case of `generate()`, `max_new_tokens` is a *kwarg* overriding a [GenerationConfig](https://huggingface.co/docs/transformers/main/en/main_classes/text_generation#transformers.GenerationConfig) parameter from `model.generation_config`. But the docs of that class are better explained [here](https://huggingface.co/docs/transformers/generation_strategies).

## Loading a dataset
🤗datasets is [very efficient](https://huggingface.co/docs/datasets/about_arrow). It does not load the dataset in memory by default and has cache systems built-in.

This is a sample code to load a subset of a dataset and pre-process it:

```python
from datasets import load_dataset
from transformers import AutoTokenizer
from pprint import pprint


dataset = load_dataset("mlabonne/FineTome-100k",
                       split="train[:100]")
tokenizer = AutoTokenizer.from_pretrained("unsloth/Llama-3.2-1B-Instruct")

x = dataset[0]
x['conversations'][0]['value'] = x['conversations'][0]['value'][:32]
x['conversations'][1]['value'] = x['conversations'][1]['value'][:32]
pprint(x)


def apply_template(examples):
    conversations = examples["conversations"]
    text = [
        (("INSTRUCTION: " if message['from'] == 'human' else "RESPONSE: ")
         + message['value'])
        for message in conversations]
    text = " ".join(text)
    return tokenizer(text)


dataset = dataset.map(apply_template, batched=False)
dataset = dataset.remove_columns("conversations")
dataset = dataset.remove_columns("source")
dataset = dataset.remove_columns("score")
print(tokenizer.decode(dataset[0]['input_ids'][:32]))
```

Output:
```
{'conversations': [{'from': 'human',
                    'value': 'Explain what boolean operators a'},
                   {'from': 'gpt',
                    'value': 'Boolean operators are logical op'}],
 'score': 5.212620735168457,
 'source': 'infini-instruct-top-500k'}

<|begin_of_text|>INSTRUCTION: Explain what boolean operators are, what they do, and provide examples of how they can be used in programming. Additionally, describe the concept of
```

## Building a dataset
I will show how it is done using an example. There are different ways of creating a dataset, but the one using a generator can be more efficient and it is general for any situation.

```python
import numpy as np
from datasets import Dataset, DatasetDict, load_dataset, disable_caching, load_from_disk


class MyGen:
    """Alternative 1."""

    def __init__(self, split):
        if split == 'train':
            i = 6
        elif split == 'val':
            i = 206
        elif split == 'test':
            i = 406
        self.data = np.arange(i, i + 100)

    def __call__(self):
        print("Called!")
        for i in self.data:
            yield {'myid': i}


class MyGen2(MyGen):
    """Alternative 2."""

    def __call__(self):
        print("Called!")
        return self

    def __iter__(self):
        self._idx = 0
        return self

    def __next__(self):
        if self._idx is None or self._idx >= len(self.data):
            raise StopIteration
        self._idx += 1
        return {'myid': self.data[self._idx - 1]}


def main():
    # This is useful if the data in MyGen is fetched from a database
    # for example, and you want to make sure that the data is fetched again
    disable_caching()

    # Create the datasets
    train = Dataset.from_generator(MyGen('train'))
    print(f"train,{len(train)}")
    val = Dataset.from_generator(MyGen2('val'))
    print(f"val,{len(val)}")
    test = Dataset.from_generator(MyGen('test'))
    print(f"test,{len(test)}")
    dsetdict = DatasetDict(train=train, validation=val, test=test)

    # Save the datasets to disk
    dsetdict.save_to_disk("mydatasets/playground")

    # Check that it works
    # Not working:
    loaded_dsetdict = load_dataset("mydatasets/playground")
    print(loaded_dsetdict['train'][0])
    loaded_train = load_dataset("mydatasets/playground", split='train')
    print(loaded_train[0])
    # Working:
    loaded_dsetdict = load_from_disk("mydatasets/playground")
    print(loaded_dsetdict['train'][0])
    loaded_dsetdict = DatasetDict.load_from_disk("mydatasets/playground")
    print(loaded_dsetdict['train'][0])


if __name__ == '__main__':
    main()
```

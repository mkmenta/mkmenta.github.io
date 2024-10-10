---
layout: post
title:  "Obscure memory leaks when training on PyTorch"
date:   2024-10-10 12:00:00 +0100
categories: pytorch memory-leak python
---

_DISCLAIMER: the information here might not be completely correct. Please, if you find any error open an [issue](https://github.com/mkmenta/mkmenta.github.io/issues). Thanks!_

This post is about "memory leaks" that I have encountered training AI models with PyTorch and that have been difficult to debug. 

## The Copy-on-access problem / the refcount problem

This is the [PyTorch issue](https://github.com/pytorch/pytorch/issues/13246#issuecomment-445770039) that talks about this problem (with the solution).

**TL;DR:** If you are using Python lists and dicts to store the data in your PyTorch dataset `__init__` and you are using `num_workers>0` in the PyTorch DataLoader, you are suffering this problem. But, maybe you won't notice until you store many elements in these variables.

The **best solution** I have found is:
- To store the data in `pandas>2.0.0` using the [PyArrow backend](https://pandas.pydata.org/docs/user_guide/pyarrow.html) or 🤗HuggingFace Datasets, if your data have strings.
- To store the data in either of the above, torch tensors or numpy arrays; if they are numerical values. Don't store strings in numpy arrays or tensors, you will either continue suffering the memory leak or allocate lots of unused memory (which could be even worse).

## The pin_memory memory leak
First of all, `pin_memory=True` was giving me a speedup of x1.2 when I suffered this memory leak. So even if the leak can be solved by setting it to False, consider giving it a try.

**TL;DR:** if you have `pin_memory=True` make sure that you don't keep any of the data returned by your dataset when iterating it. For example, don't do `all_images.append(batch_images)`. If you need to, make sure you are doing a `copy()` or a `tolist()` first. Take into account, that doing a `.numpy()` in a torch tensor does not count as a copy (I guess that `.cpu()`  does but I have not tried).

Code to reproduce the leak:
```python
import time
import tqdm
import torch
import numpy as np
from torch.utils.data import DataLoader, Dataset


class MyDataset(Dataset):
    """Dataset."""

    def __init__(self):
        """Initialize dataset."""
        self.numbers = np.arange(stop=200_000, dtype=np.int32)

    def __len__(self):
        """Return dataset length."""
        return len(self.numbers)

    def __getitem__(self, idx):
        """Get item from dataset."""
        n = self.numbers[idx]
        return (np.array(n, dtype=np.int32),
                # MEMORY LEAK CAUSE (part2):
                # Without returning a big tensor, the memory leak is so small
                # that it can't be perceived
                np.random.rand(256, 256, 3))


def main():
    # Dataset
    dataset = MyDataset()
    loader = DataLoader(dataset,
                        batch_size=128,
                        num_workers=8,
                        # MEMORY LEAK CAUSE (part1):
                        # With pin_memory=False, the memory leak does not happen
                        pin_memory=True)

    all_numbers = []
    with torch.inference_mode():
        for batch in tqdm.tqdm(loader, total=len(loader)):
            n, img = batch

            # MEMORY LEAK:
            all_numbers.append(n.numpy())
            # NO MEMORY LEAK:
            # all_numbers.append(n.numpy().copy())

            time.sleep(0.1)


if __name__ == "__main__":
    main()
```

What is probably happening here (according to my knowledge):
- `pin_memory=True` keeps memory allocated to speed up the data loading and transfer to GPU. So the returned values in 
   `__getitem__` always go to this allocated space.
- In our `__getitem__` we don't just return a small array (n) but also a big one `np.random.rand(256, 256, 3)` which makes 
  the memory leak more noticeable. Otherwise, it would be so small that it would be hard to perceive.
- The previous two points imply that pin memory is allocating a relevant amount of memory per each worker of the 
  `DataLoader` to put the returned NumPy arrays there.
- The actual memory leak occurs when in the loop we do not `.copy()` n, not allowing the garbage collector to free the 
  memory allocated by `pin_memory`. This is a special situation for two reasons:
    - Even if we do `.numpy()`, the memory used by the original tensor and the numpy array is actually the same. I.e. 
      doing `.numpy()` does not create any copy of the variable, we are still referencing the same memory space. (This 
      is probably to allow fast conversions between torch tensor and numpy).
    - As we are keeping n in `all_numbers` we don't allow to free that memory and on top of that, the big array is also
      kept in memory. This is because they share the same allocated memory space by pin memory. They are not 
      independent from each other. We can't keep n in `all_numbers` and let the big tensor be freed.
- So, for the next `__getitem__`, the worker needs to again allocate memory for n and the big array. This is the memory leak.

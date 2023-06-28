---
title: 'Accelerating Data Processing in Python with Parallelism'
date: 2023-06-28
permalink: /posts/2022/06/python-parallel/
tags:
  - Python
---

Parallel data processing is a technique used to speed up data processing tasks by dividing the data into smaller chunks and processing them simultaneously on multiple processors. It could be useful as long as do not need to be implemented sequentially. In contrast to threading, which is suitable for tasks that are I/O bounded but not tasks that are CPU bounded, multiprocessing runs on multiple cores and helps when tasks are CPU bounded. Python provides several ways to implement parallel data processing. 

In this post, we will introduce two of them
- [Pandarallel](#pandarallel)
- [Multiprocessing](#multiprocessing)

## Pandarallel

If you are using `apply` when processing your DataFrame, `pandarallel` is an easy-to-use tool that helps you parallelize your operations with minimal modification to your code.

**1. Installation**

```python
pip install pandarallel
```

**2. Initialization**

```python
from pandarallel import pandarallel
pandarallel.initialize(progress_bar=True)
```

If successfully initialized, you will see:

```
INFO: Pandarallel will run on 8 workers.
INFO: Pandarallel will use Memory file system to transfer data between the main process and workers.
```

The number of workers depends on the available cores of your device.

**3. Usage**

```python
# replace standard `apply` with `parallel_apply`
df['new_var'] = df.parallel_apply(func, axis=1) 
```

Note that `pandarallel` does not reduce your processing time to 1/n if it runs on n workers, maybe far from that. In my own experience using M1 Mac, `parallel_apply` takes around 1/3 of the time comparing to `apply`.

More details can be found at [PyPI](https://pypi.org/project/pandarallel/) or [Github](https://github.com/nalepae/pandarallel).

## Multiprocessing

Although Pandarallel is a simple and efficient tool, it lacks flexibility when it comes to more complex tasks. Suppose you have a list of firms and want to compute several different quarterly and annual variables, and save them in two different DataFrames. In this case, it is probably easier to use `multiprocessing`. 

Here is a sample code.

```python
import pickle
import pandas as pd
import multiprocessing

# define data processing function
def process_data(firm_list):
    ... ...
    return df_quarter, df_annual

if __name__ == '__main__':
    # Read in firm list
    with open('firm_list.pkl', 'rb') as f:
        firm_list = pickle.load(f)

    # Split data into groups
    num_groups = multiprocessing.cpu_count()
    group_size = len(firm_list) // num_groups + (len(firm_list) % num_groups > 0)
    groups = [firm_list[x:x+group_size] for x in range(0, len(firm_list), group_size)]

    # Create a pool of worker processes
    with multiprocessing.Pool() as pool:
        results = [pool.apply_async(process_data, args=(group,)) for group in groups]

    # Combine df_quarter, df_annual from all groups together after the process
    df_quarter_list = []
    df_annual_list = []
    for result in results:
        df_quarter, df_annual = result.get()
        df_quarter_list.append(df_quarter)
        df_annual_list.append(df_annual)   
    df_quarter_combined = pd.concat(df_quarter_list)
    df_annual_combined = pd.concat(df_annual_list)
```

Here is a very useful table on the difference between `Pool.apply`, `Pool.apply_async`, `Pool.map`, `Pool.map_async` from [stackoverflow](https://stackoverflow.com/questions/8533318/multiprocessing-pool-when-to-use-apply-apply-async-or-map). Similar to `map` and `apply`, `Pool.map` only accepts a single argument, while `Pool.apply` accepts multiple arguments. `async` methods are non-blocking and return an `AsyncResult` object, while others block until all calls are completed. The official documentation can be found [here](https://docs.python.org/3/library/multiprocessing.html).

|                  | Multi-args | Concurrence | Blocking | Ordered-results |
|------------------|------------|-------------|----------|-----------------|
|Pool.map          | no         | yes         |  yes     |    yes          |
|Pool.map_async    | no         | yes         |  no      |    yes          |
|Pool.apply        | yes        | no          |  yes     |    no           |
|Pool.apply_async  | yes        | yes         |  no      |    no           |
|Pool.starmap      | yes        | yes         |  yes     |    yes          |
|Pool.starmap_async| yes        | yes         |  no      |    no           |

> See also [concurrent.futures.ProcessPoolExecutor](https://docs.python.org/3/library/concurrent.futures.html#concurrent.futures.ProcessPoolExecutor) offers a higher level interface to push tasks to a background process without blocking execution of the calling process. Compared to using the `Pool` interface directly, the `concurrent.futures` API more readily allows the submission of work to the underlying process pool to be separated from waiting for the results.

In our simple example, this two approaches are very similar.
```python
import concurrent.futures

with concurrent.futures.ProcessPoolExecutor() as executor:
    results = [executor.submit(process_data, group) for group in groups]
    for f in concurrent.futures.as_completed(results):
        df_quarter, df_annual = f.result()
        df_quarter_list.append(df_quarter)
        df_annual_list.append(df_annual)
```

In conclusion, parallel processing in Python can significantly speed up your data processing tasks. By using Pandarallel or Multiprocessing, you can take advantage of multiple cores and efficiently process large datasets. 

This is all I have for this post. Enjoy your reading, and hope it helps!
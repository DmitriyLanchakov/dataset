# Working with large datasets

## Dataset
See [dataset.md](dataset.md)

## Index
Index holds a sequence of data item ids. As a dataset is split into batches you should have a mechanism to uniquely address each data item.
In simple cases it can be just a `numpy.arange`:
```python
dataset_index = DatasetIndex(np.arange(my_array.shape[0]))
```
See [index.md](index.md)

## Batch
See [batch.md](batch.md)

## Pipeline
See [pipeline.md](pipeline.md)

## Within-batch parallelism
In order to accelerate data processing you can run batch methods in parallel:
```python
from dataset import Batch, inbatch_parallel, action

class MyBatch(Batch):
    ...
    @action
    @inbatch_parallel(init='_init_fn', post='_post_fn', target='threads')
    def some_action(self, item):
        # process just one item from the batch
        return some_value
```
See [parallel.md](parallel.md)

## Inter-batch parallelism
To further increase pipeline performance and eliminate inter batch delays you may process several batches in parallel:
```python
some_pipeline.next_batch(BATCH_SIZE, prefetch=3)
```
The parameter `prefetch` defines how many additional batches will be processed in the background.
See [prefetch.md](prefetch.md)
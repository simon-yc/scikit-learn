# Documentation of Implemented Issues

## Issue #21335

<blockquote>
    <p>Issue Link: 
        <a href="https://github.com/scikit-learn/scikit-learn/issues/21335">(#21335) <code>ndcg_score</code> doesn't work with binary relevance and a list of 1 element </a>
    </p>
    <p>Targeted PR: 
        <a href=""> to do </a>
    </p>
</blockquote>

### Files changed
<ul>
    <li>
        <a href="#"><code>/sklearn/metrics/_ranking.py</code></a>
    </li>
    <li>
        <a href="#"><code>/sklearn/metrics/tests/test_ranking.py</code></a>
    </li>
</ul>

### Description
The function `ndcg_score` only works correctly when the number of elements is greater than 1. If the number of elements is equal to 1, a Value Error is raised unexpectedly.
<a href="https://github.com/scikit-learn/scikit-learn/issues/21335">(Source)
</a>
```python
>>> t = [[1]]
>>> p = [[0]]
>>> metrics.ndcg_score(t, p)
```
#### Expected Result
```python
No error is thrown
```
or 
```python
ValueError: Calculating the NDCG score with a single input is not meaningful.
```
#### Actual Result
```python
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/Users/cbournhonesque/.pyenv/versions/bento/lib/python3.8/site-packages/sklearn/utils/validation.py", line 63, in inner_f
    return f(*args, **kwargs)
  File "/Users/cbournhonesque/.pyenv/versions/bento/lib/python3.8/site-packages/sklearn/metrics/_ranking.py", line 1567, in ndcg_score
    _check_dcg_target_type(y_true)
  File "/Users/cbournhonesque/.pyenv/versions/bento/lib/python3.8/site-packages/sklearn/metrics/_ranking.py", line 1307, in _check_dcg_target_type
    raise ValueError(
ValueError: Only ('multilabel-indicator', 'continuous-multioutput', 'multiclass-multioutput') formats are supported. Got binary instead
```

###  Development and Implementation Process

#### Find cause of bug
The root cause of this issue is inside the `ndcg_score()` method</a> in `_forest.py` defined on line 1616. Inside this `ndcg_score()` method, the calls to `check_array` and `check_consistent_length` are successful and allows a single element as input. However, once we get to the method call `_check_dcg_target_type`, we obtain a `ValueError` telling us that the single element input is `binary`. This should not have been the case because technically a single element passed in as input should have passed this test case.

#### Find possible fixes for bug
We have two possible fixes for this issue:

1. Calculating the `ndcg_score` for a list of 1 element is trivial because it will simply return 1 if the binary relevance input is greater than 0, and 0 otherwise. We can implement a fix that does exactly this before the call to `_check_dcg_target_type` to return the correct value before a `ValueError` is thrown.

2. We implement a fix that still prohibits any single element input into the method `ndcg_score`, but before the call to `_check_dcg_target_type` to ensure that it does not throw the wrong `ValueError`. In this case, we will provide the user with better feedback as to why their single element input is not valid for `ndcg_score`.

#### Implementation of fix
We have chosen solution 2 from the above list. We chose it because a single element passed into `ndcg_score` is not meaningful as discussed above. The `ndcg_score` is a trivial computation in this case. Thus, we believe that the best fix we should implement is to provide the user with a better feedback stating that passing in a single input into `ndcg_score` is not meaningful in any way.

The added code checks the shape of the y_true input to the `ndcg_score` function in scikit learn's metrics module.

If y_true is a 2D array with only one column (i.e., a single input), the code raises a ValueError with the message: "Calculating the NDCG score with a single input is not meaningful."

This is because the normalized discounted cumulative gain (NDCG) is a measure of ranking quality that requires multiple relevance scores for different items in a ranking. Therefore, calculating the NDCG score for a single item ranking does not make sense.

The added code ensures that the input to the `ndcg_score` function meets the minimum requirements for meaningful NDCG score calculation, preventing potential errors or misleading results.
```python
    y_shape = y_true.shape

    if len(y_shape) == 2 and y_shape[1] == 1:
        raise ValueError(
            "Calculating the NDCG score with a single input is not meaningful."
        )
```

### Testing
to do

### Customer Acceptance Test
to do
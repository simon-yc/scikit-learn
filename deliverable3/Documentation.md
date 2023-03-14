# Documentation of Implemented Issues

## Issue #24037

<blockquote>
    <p>Issue Link: 
        <a href="https://github.com/scikit-learn/scikit-learn/issues/24037">(#24037) RandomForestClassifier <code>class_weight/max_samples</code> interaction can lead to ungraceful and nondescriptive failure</a>
    </p>
    <p>Targeted PR: 
        <a href="https://github.com/simon-yc/d01w23-team-deez/pull/2">(#2) Fix: Throw Value Error when <code>max_samples</code> too small</a>
    </p>
        <p>Team member involved: 

+ Abhay Patel (finding cause of bug and testing)
+ Tanzim Ahmed (finding fixes for bug and testing)
+ Tirth Patel (task implementation)
    </p>
</blockquote>

### Files changed
<ul>
    <li>
        <a href="#"><code>/sklearn/ensemble/_forest.py</code></a>
    </li>
    <li>
        <a href="#"><code>/sklearn/ensemble/tests/test_forest.py</code></a>
    </li>
</ul>

### Description
When using a RandomForestClassifier with the parameters ```class_weight='balanced_subsample'``` and ```max_samples```, a Value Error is raised when the value of ```max_samples``` is set to near 0. 
<a href="https://github.com/scikit-learn/scikit-learn/issues/24037#:~:text=from%20sklearn.datasets%20import%20load_wine%0Afrom%20sklearn.ensemble%20import%20RandomForestClassifier%0A%0AX%2C%20y%20%3D%20load_wine(return_X_y%3DTrue)%0A%0Aclf%20%3D%20RandomForestClassifier(max_samples%3D1e%2D4%2C%20class_weight%3D%27balanced_subsample%27)%0Aclf.fit(X%2Cy)">(Source)
</a>
```python
from sklearn.datasets import load_wine
from sklearn.ensemble import RandomForestClassifier

X, y = load_wine(return_X_y=True)

clf = RandomForestClassifier(max_samples=1e-4, class_weight='balanced_subsample')
clf.fit(X,y)
```
#### Exprected Result
```python
No error is thrown
```
or 
```python
ValueError: insufficient samples for max_samples value 0.0001
```
#### Actual Result
```python
IndexError: arrays used as indices must be of integer (or boolean) type
```
### Implementation Process

####  Find cause of bug

The root cause of this issue is inside the 
<a href="https://github.com/scikit-learn/scikit-learn/blob/main/sklearn/ensemble/_forest.py#:~:text=def%20fit(self%2C%20X%2C%20y%2C%20sample_weight%3DNone)%3A">
```fit() method``` 
</a> 
in _forest.py defined on line 317. The issue is that inside this fit() method, the 
<a href="https://github.com/scikit-learn/scikit-learn/blob/main/sklearn/ensemble/_forest.py#:~:text=.bootstrap%3A-,n_samples_bootstrap%20%3D%20_get_n_samples_bootstrap(,-n_samples%3DX">
```n_samples_bootstrap``` 
</a> 
variable is being set to ```0``` when it is expected to be ```>= 1```. The variable is initialized on line 403 in _forest.py where it is set to the return value of the function ```_get_n_samples_bootstrap``` which takes in the max_samples value as input as well as the n_samples value. Inside the 
<a href="https://github.com/scikit-learn/scikit-learn/blob/30bf6f39a7126a351db8971d24aa865fa5605569/sklearn/ensemble/_forest.py#L90:~:text=def%20_get_n_samples_bootstrap(n_samples%2C%20max_samples)%3A">
```_get_n_samples_bootstrap``` 
</a>
function, we enter the third if statement (line 119 in _forest.py) where we return the result of ```round(n_samples * max_samples)```. The result of the round is ```0``` because the product of n_samples and max_samples is too small (rounds to 0). This is where our issue comes from as ```n_samples_bootstrap``` is now set to ```0```. Later on, in line 473 in _forest.py, in the 
<a href="https://github.com/scikit-learn/scikit-learn/blob/30bf6f39a7126a351db8971d24aa865fa5605569/sklearn/ensemble/_forest.py#L90:~:text=on%20using%20threads.-,trees%20%3D%20Parallel(,-n_jobs%3Dself">
```Parallel loop``` 
</a>
we call the function 
<a href="https://github.com/scikit-learn/scikit-learn/blob/30bf6f39a7126a351db8971d24aa865fa5605569/sklearn/ensemble/_forest.py#L90:~:text=def%20_parallel_build_trees(">
```_parallel_build_trees``` 
</a>
where we are 
<a href="https://github.com/scikit-learn/scikit-learn/blob/30bf6f39a7126a351db8971d24aa865fa5605569/sklearn/ensemble/_forest.py#L90:~:text=indices%20%3D%20_generate_sample_indices(">
```using n_samples_bootstrap``` 
</a>
to generate our indices array (line 171 in _forest.py). Since our ```n_samples_bootstrap``` is set to 0, ```_generate_sample_indices``` returns an empty list, which causes the crash in 
<a href="https://github.com/scikit-learn/scikit-learn/blob/30bf6f39a7126a351db8971d24aa865fa5605569/sklearn/ensemble/_forest.py#L90:~:text=curr_sample_weight%20*%3D%20compute_sample_weight(%22balanced%22%2C%20y%2C%20indices%3Dindices)">
```compute_sample_weight``` 
</a>
on line 182 in _forest.py.

####  Find possible fixes for bug

We have three possible fixes for this issue:

1. The first fix that comes to mind is to change the `_get_n_samples_bootstrap` function to return 1 instead of 0 when the product of n_samples and max_samples is too small. This would fix the issue since the initial problem was that the `_get_n_samples_bootstrap` function returns `round(n_samples * max_samples)`, but since the value of max_samples is too small, it results in a 0 rather than a non-zero integer. This aproach solved the issue however, it would also change the behavior of the function and would change the overall behavior of the fit() function. 

2. The second fix is to change the `compute_sample_weight` function to handle the case where the input array is empty. This would fix the issue since the initial problem was that the `compute_sample_weight` function was being called with an empty array, which was a result of calling the `_generate_sample_indices` function that takes in the `n_samples_bootstrap` variable created by the `_get_n_samples_bootstrap` function, which we know returns an unwanted 0. However, this approach would also change the behavior of the function and the the overall behavior of the fit() function.

3. The third fix is to change the `_get_n_samples_bootstrap` function to throw a value error when the product of n_samples and max_samples is too small. This would fix the issue since the initial problem was that the `_get_n_samples_bootstrap` function returns `round(n_samples * max_samples)`, but since the value of max_samples is too small, it results in a 0 rather than a non-zero integer. This approach does not change the behavior of the function, but it does change the behavior of the `fit()` method in the sense that it would now throw an error instead of continuing on with the rest of the code.

####  Implementation of fix

We have chosen solution 3 from the above list. We chose it because this solution does not modify the behaviour of the `_get_n_samples_bootstrap`. All it does is restrict the user from using certain values for `max_samples`. 
We added an if statement within the method `_get_n_samples_bootstrap` in _forest.py under line 119 to check whether the rounding of `n_samples` * `max_samples` would return a value under 1. If this was the case, then the function will raise a ValueError with an informative message stating that the max_samples value is too low.

```python
if isinstance(max_samples, Real):
    result = round(n_samples * max_samples)
    if result < 1:
        msg = "insufficient samples for max_samples value {}"
        raise ValueError(msg.format(max_samples))
    return result
```

### Testing

#### Scenario #1: ```max_samples``` parameter is set to a value that is not too small

```python
def test_round_to_positive_int():
    msg = "Unexpected value error"

    X, y = datasets.load_wine(return_X_y=True)

    clf = RandomForestClassifier(max_samples=1e-2, class_weight='balanced_subsample')

    try:
        clf.fit(X,y)
    except ValueError:
        pytest.fail(msg)
```

#### Scenario #2: ```max_samples``` parameter is set to a value that is too small

```python
def test_round_to_zero_error():
    X, y = datasets.load_wine(return_X_y=True)

    clf = RandomForestClassifier(max_samples=1e-8, class_weight='balanced_subsample')

    with pytest.raises(ValueError):
        clf.fit(X,y)
```

### Customer Acceptance Test

Previously running the code snippet described in ```Description```, we ran into ```IndexError: arrays used as indices must be of integer (or boolean) type``` when the value of max_samples was too small.

After proposing the fix described in ```Implementation of fix```, we no longer get such error and properly executes the problematic code. Thus, we get now get the following after the fix:

```python
ValueError: insufficient samples for max_samples value 0.0001
```
<br></br>

## Issue #24862

<blockquote>
    <p>Issue Link: 
        <a href="https://github.com/scikit-learn/scikit-learn/issues/24862">(#24862) Make automatic parameter validation for scikit-learn public function </a>
    </p>
    <p>Targeted PR: 
        <a href="https://github.com/simon-yc/d01w23-team-deez/pull/4">(#4) Fix: Parameters validation for mean_squared_log_error</a>
    </p>
    <p>Team member involved: 

+ Linda Shi (task implementation and testing)
+ Flora Xie (task implementation and testing)
    </p>
</blockquote>

### Files changed
<ul>
    <li>
        <a href="#"><code>/sklearn/metrics/_regression.py</code></a>
    </li>
    <li>
        <a href="#"><code>/sklearn/tests/test_public_functions.py</code></a>
    </li>
</ul>

### Description
The main goal of this issue is to have a consistent way to raise an informative error message when the parameter passed in has an invalid type or value. Among the functions listed in the issue open to be fixed, our team implemented parameter validation for the function mean_squared_log_error. Currently, this function has no parameter validation so invalid parameters may lead to an unexpected result. After the implementation, if an invalid parameter is passed into the function, we expect an exception would occur with an informative error message. 

#### Implementation:
Our implementation is to decorate the function mean_squared_log_error with the decorator sklearn.utils._param_validation.validate_params. For each parameter of this function, we first check its constraints listed from both the docstring and the function's detailed implementation, then implement validation according to it. Our implementation addresses this issue by providing validations for each parameter that the function takes in, so it handles cases when invalid arguments are passed in. 

##### Changes made to the design/codebase:

Add parameter validation to the function mean_squared_log_error:

```python
@validate_params(
    {
        "y_true": ["array-like"],
        "y_pred": ["array-like"],
        "sample_weight": ["array-like", None],
        "multioutput": [StrOptions({"raw_values", "uniform_average"}), "array-like"],
        "squared": ["boolean"],
    }
)
def mean_squared_log_error(
    y_true, y_pred, *, sample_weight=None, multioutput="uniform_average", squared=True
):
... ...
```
Added the function into the PARAM_VALIDATION_FUNCTION_LIST for testing purpose:
```python
PARAM_VALIDATION_FUNCTION_LIST = [
    ...
"sklearn.metrics.mean_squared_log_error",
    ...
]
```
### Testing

#### Unit test
In the issue documentation step 5, it mentions that under sklearn/tests/test_public_functions.py, there already exists a test case called test_function_param_validation for checking parameter validation. To test our implementation, we only need to add the function we did the validation for to the common param validation test list. Thus, as the test case already exists, and by following the issue documentation, there is no need to add additional unit tests for our implementation. 

#### Customer Acceptance Test

1. As a user, if I try to call function mean_squared_log_error using the following parameter:
   + y_true = [3, 5, 2.5, 7]
   + y_pred = [2.5, 5, 4, 8]
   + sample_weight = None
   + multioutput = "uniform_average"
   + squared = False

    The expected output should be 0.199…, as all input parameters are valid. 
    
2. As a user, if I try to call function mean_squared_log_error using the following parameter:
   + y_true = “INVALID INPUT”
   + y_pred = [2.5, 5, 4, 8]
   + sample_weight = None
   + multioutput = "uniform_average"
   + squared = False
	
    An exception should occur with the error message below because the y_true parameter has an invalid value: 

```python
sklearn.utils._param_validation.InvalidParameterError: 
The 'y_true' parameter of mean_squared_log_error must be an array-like. Got ‘INVALID INPUT’ instead.
```

<br></br>

## Issue #21335

<blockquote>
    <p>Issue Link: 
        <a href="https://github.com/scikit-learn/scikit-learn/issues/21335">(#21335) <code>ndcg_score</code> doesn't work with binary relevance and a list of 1 element </a>
    </p>
    <p>Targeted PR: 
        <a href="https://github.com/simon-yc/d01w23-team-deez/pull/6">(#6) Fix: Throw correct Value Error when `ndcg_score` takes in a single input</a>
    </p>
        <p>Team member involved: 

+ Brandon Lo (task implementation and testing)
+ Simon Chau (task implementation and testing)
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
The function `ndcg_score` only works correctly when the number of elements is greater than 1. If the number of elements is equal to 1, an incorrect `ValueError` is raised unexpectedly.
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

###  Implementation Process

#### Find cause of bug
The root cause of this issue is inside the <a href="https://github.com/scikit-learn/scikit-learn/blob/main/sklearn/metrics/_ranking.py#:~:text=def%20ndcg_score(y_true%2C%20y_score%2C%20*%2C%20k%3DNone%2C%20sample_weight%3DNone%2C%20ignore_ties%3DFalse)%3A"> `ndcg_score`</a> method</a> in `_ranking.py`. Inside this `ndcg_score` method, the calls to `check_array` and `check_consistent_length` are successful and allows a single element as input. However, once we get to the method call `_check_dcg_target_type`, we obtain a `ValueError` telling us that the single element input is `binary`. This should not have been the case because a single element passed in as input into `ndcg_score` should have passed this test case.

#### Find possible fixes for bug
We have two possible fixes for this issue:

1. Calculating the `ndcg_score` for a list of 1 element is trivial because it will simply return 1 if the binary relevance input is greater than 0, and 0 otherwise. We can implement a fix that does exactly this before the call to `_check_dcg_target_type` to return the correct value before a `ValueError` is thrown.

2. We implement a fix that still prohibits any single element input into the method `ndcg_score`, but before the call to `_check_dcg_target_type` to ensure that it does not throw the wrong `ValueError`. In this case, we will provide the user with better feedback as to why their single element input is not valid for `ndcg_score`.

#### Implementation of fix
We have chosen solution 2 from the above list. We chose it because a single element passed into `ndcg_score` is not meaningful as discussed above. The `ndcg_score` is a trivial computation in this case. Thus, we believe that the best fix we should implement is to provide the user with a better feedback stating that passing in a single input into `ndcg_score` is not meaningful in any way.

The added code checks the shape of the y_true input to the `ndcg_score` function in scikit learn's metrics module. This specific method that was modified can be found inside the file <a href="https://github.com/scikit-learn/scikit-learn/blob/main/sklearn/metrics/_ranking.py#:~:text=def%20ndcg_score(y_true%2C%20y_score%2C%20*%2C%20k%3DNone%2C%20sample_weight%3DNone%2C%20ignore_ties%3DFalse)%3A"> _ranking.py</a>.

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
The following test cases were added inside the file <a href="https://github.com/scikit-learn/scikit-learn/blob/main/sklearn/metrics/tests/test_ranking.py"> test_ranking.py</a>.

#### Scenario \#1: `ndcg_score` is called with more than one input
```python
def test_ndcg_score_for_multiple_documents():
    msg = "Unexpected value error"

    t = [[1, 0]]
    p = [[0, 1]]

    try:
        ndcg_score(t, p)
    except ValueError:
        pytest.fail(msg)
```

#### Scenario \#2: `ndcg_score` is called with only one input
```python
def test_ndcg_score_for_single_document_error():
    error = "Calculating the NDCG score with a single input is not meaningful."

    t = [[1]]
    p = [[0]]

    with pytest.raises(ValueError, match=error):
        ndcg_score(t, p)
```

### Customer Acceptance Test
Previously running the code snippet described in `Description`, we ran into `ValueError: Only ('multilabel-indicator', 'continuous-multioutput', 'multiclass-multioutput') formats are supported. Got binary instead` when a single input was passed into `ndcg_score`. However, this is unexpected behaviour because the error message does not describe the issue being presented correctly.

After proposing the fix described in `Implementation of fix`, we obtain another `ValueError` which describes clearly that the issue is because an input of a single value is meaningless when running `ndcg_score`. Thus, we now get the following after the fix:
```python
ValueError: Calculating the NDCG score with a single input is not meaningful.
```

# Documentation of Implemented Issues

## Issue #24037

<blockquote>
    <p>Issue Link: 
        <a href="https://github.com/scikit-learn/scikit-learn/issues/24037">(#24037) RandomForestClassifier <code>class_weight/max_samples</code> interaction can lead to ungraceful and nondescriptive failure</a>
    </p>
    <p>Targeted PR: 
        <a href="https://github.com/scikit-learn/scikit-learn/issues/24037">(#2) Fix: Throw Value Error when <code>max_samples</code> too small</a>
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
### Development and Implementation Process

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

#### Scenario #1: max_samples parameter is set to a value that is not too small

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

#### Scenario #2: max_samples parameter is set to a value that is too small

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
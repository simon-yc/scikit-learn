# Documentation of Implemented Issues

## Issue #24037

<blockquote>
    <p>Issue Link: 
        <a href="https://github.com/scikit-learn/scikit-learn/issues/24862">(#24862) Make automatic parameter validation for scikit-learn public function </a>
    </p>
    <p>Targeted PR: 
        <a href="https://github.com/scikit-learn/scikit-learn/issues/24037">(#2) Fix: Throw Value Error when <code>max_samples</code> too small</a>
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
Added the function into the 

PARAM_VALIDATION_FUNCTION_LIST for testing purpose:
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
    
1. if I try to call function mean_squared_log_error using the following parameter:
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


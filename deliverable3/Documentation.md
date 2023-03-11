## (#24037) RandomForestClassifier class_weight/max_samples interaction can lead to ungraceful and nondescriptive failure

</br>

###  **Find cause of bug**

The root cause of this issue is that the n_samples_bootstrap variable is being set to 0 when it is expected to be >= 1. The variable is initialized in the fit() method on line 403 in _forest.py where it is set to the return value of the function _get_n_samples_bootstrap which takes in the max_samples value as input as well as the n_samples value. Inside the _get_n_samples_bootstrap function, we enter the third if statement (line 119 in _forest.py) where we return the result of round(n_samples * max_samples). The result of the round is 0 because the product of n_samples and max_sampels is too small (rounds to 0). This is where our issue comes from as n_samples_bootstrap is now set to 0. Later on, in line 473 in _forest.py, in the Parallel loop we call the function _parallel_build_trees where we are using n_samples_bootstrap to generate our indices array (line 171 in _forest.py). Since our n_samples_bootstrap is set to 0, _generate_sample_indices returns an empty list, which causes the crash in compute_class_weight on line 182 in _forest.py. 

Files of importance:

scikit-learn-main\sklearn\ensemble\_forest.py

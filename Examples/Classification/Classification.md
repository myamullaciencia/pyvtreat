# Using [vtreat](https://github.com/WinVector/pyvtreat) with Classification Problems

Nina Zumel and John Mount
November 2019

Note: this is a description of the [`Python` version of `vtreat`](https://github.com/WinVector/pyvtreat), the same example for the [`R` version of `vtreat`](https://github.com/WinVector/vtreat) can be found [here](https://github.com/WinVector/vtreat/blob/master/Examples/Classification/Classification.md).


## Preliminaries

Load modules/packages.


```python
import pkg_resources
import pandas
import numpy
import numpy.random
import seaborn
import matplotlib.pyplot as plt
import vtreat
import vtreat.util
import wvpy.util

numpy.random.seed(2019)
```

Generate example data. 

* `y` is a noisy sinusoidal function of the variable `x`
* `yc` is the output to be predicted: : whether `y` is > 0.5. 
* Input `xc` is a categorical variable that represents a discretization of `y`, along some `NaN`s
* Input `x2` is a pure noise variable with no relationship to the output


```python
def make_data(nrows):
    d = pandas.DataFrame({'x': 5*numpy.random.normal(size=nrows)})
    d['y'] = numpy.sin(d['x']) + 0.1*numpy.random.normal(size=nrows)
    d.loc[numpy.arange(3, 10), 'x'] = numpy.nan                           # introduce a nan level
    d['xc'] = ['level_' + str(5*numpy.round(yi/5, 1)) for yi in d['y']]
    d['x2'] = numpy.random.normal(size=nrows)
    d.loc[d['xc']=='level_-1.0', 'xc'] = numpy.nan  # introduce a nan level
    d['yc'] = d['y']>0.5
    return d

d = make_data(500)

d.head()
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>x</th>
      <th>y</th>
      <th>xc</th>
      <th>x2</th>
      <th>yc</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>-1.088395</td>
      <td>-0.956311</td>
      <td>NaN</td>
      <td>-1.424184</td>
      <td>False</td>
    </tr>
    <tr>
      <th>1</th>
      <td>4.107277</td>
      <td>-0.671564</td>
      <td>level_-0.5</td>
      <td>0.427360</td>
      <td>False</td>
    </tr>
    <tr>
      <th>2</th>
      <td>7.406389</td>
      <td>0.906303</td>
      <td>level_1.0</td>
      <td>0.668849</td>
      <td>True</td>
    </tr>
    <tr>
      <th>3</th>
      <td>NaN</td>
      <td>0.222792</td>
      <td>level_0.0</td>
      <td>-0.015787</td>
      <td>False</td>
    </tr>
    <tr>
      <th>4</th>
      <td>NaN</td>
      <td>-0.975431</td>
      <td>NaN</td>
      <td>-0.491017</td>
      <td>False</td>
    </tr>
  </tbody>
</table>
</div>




```python
outcome_name = 'yc'    # outcome variable / column
outcome_target = True  # value we consider positive
```

### Some quick data exploration

Check how many levels `xc` has, and their distribution (including `NaN`)


```python
d['xc'].unique()
```




    array([nan, 'level_-0.5', 'level_1.0', 'level_0.0', 'level_-0.0',
           'level_0.5'], dtype=object)




```python
d['xc'].value_counts(dropna=False)
```




    level_1.0     140
    NaN           109
    level_-0.5    103
    level_0.5      75
    level_0.0      37
    level_-0.0     36
    Name: xc, dtype: int64



Find the prevalence of `yc == True` (our chosen notion of "positive").


```python
numpy.mean(d[outcome_name] == outcome_target)
```




    0.35



Plot of `yc` versus `x`.


```python
seaborn.lineplot(x='x', y='yc', data=d)
```




    <matplotlib.axes._subplots.AxesSubplot at 0x1a19866310>




![png](output_14_1.png)


## Build a transform appropriate for classification problems.

Now that we have the data, we want to treat it prior to modeling: we want training data where all the input variables are numeric and have no missing values or `NaN`s.

First create the data treatment transform object, in this case a treatment for a binomial classification problem.


```python
transform = vtreat.BinomialOutcomeTreatment(
    outcome_name=outcome_name,      # outcome variable
    outcome_target=outcome_target,  # outcome of interest
    cols_to_copy=['y'],  # columns to "carry along" but not treat as input variables
)  
```

Use the training data `d` to fit the transform and the return a treated training set: completely numeric, with no missing values.
Note that for the training data `d`: `transform.fit_transform()` is **not** the same as `transform.fit().transform()`; the second call can lead to nested model bias in some situations, and is **not** recommended. 
For other, later data, not seen during transform design `transform.transform(o)` is an appropriate step.


```python
d_prepared = transform.fit_transform(d, d['yc'])
```

Now examine the score frame, which gives information about each new variable, including its type, which original variable it is  derived from, its (cross-validated) correlation with the outcome, and its (cross-validated) significance as a one-variable linear model for the outcome. 


```python
transform.score_frame_
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>variable</th>
      <th>orig_variable</th>
      <th>treatment</th>
      <th>y_aware</th>
      <th>has_range</th>
      <th>PearsonR</th>
      <th>R2</th>
      <th>significance</th>
      <th>vcount</th>
      <th>default_threshold</th>
      <th>recommended</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>x_is_bad</td>
      <td>x</td>
      <td>missing_indicator</td>
      <td>False</td>
      <td>True</td>
      <td>-0.051749</td>
      <td>0.002388</td>
      <td>0.213707</td>
      <td>2.0</td>
      <td>0.10</td>
      <td>False</td>
    </tr>
    <tr>
      <th>1</th>
      <td>xc_is_bad</td>
      <td>xc</td>
      <td>missing_indicator</td>
      <td>False</td>
      <td>True</td>
      <td>-0.387438</td>
      <td>0.169451</td>
      <td>0.000000</td>
      <td>2.0</td>
      <td>0.10</td>
      <td>True</td>
    </tr>
    <tr>
      <th>2</th>
      <td>x</td>
      <td>x</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>0.052826</td>
      <td>0.002158</td>
      <td>0.237141</td>
      <td>2.0</td>
      <td>0.10</td>
      <td>False</td>
    </tr>
    <tr>
      <th>3</th>
      <td>x2</td>
      <td>x2</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>0.069126</td>
      <td>0.003709</td>
      <td>0.121205</td>
      <td>2.0</td>
      <td>0.10</td>
      <td>False</td>
    </tr>
    <tr>
      <th>4</th>
      <td>xc_logit_code</td>
      <td>xc</td>
      <td>logit_code</td>
      <td>True</td>
      <td>True</td>
      <td>0.867915</td>
      <td>0.834531</td>
      <td>0.000000</td>
      <td>1.0</td>
      <td>0.20</td>
      <td>True</td>
    </tr>
    <tr>
      <th>5</th>
      <td>xc_prevalence_code</td>
      <td>xc</td>
      <td>prevalence_code</td>
      <td>False</td>
      <td>True</td>
      <td>0.567968</td>
      <td>0.341474</td>
      <td>0.000000</td>
      <td>1.0</td>
      <td>0.20</td>
      <td>True</td>
    </tr>
    <tr>
      <th>6</th>
      <td>xc_lev_level_1_0</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>0.849837</td>
      <td>0.645319</td>
      <td>0.000000</td>
      <td>4.0</td>
      <td>0.05</td>
      <td>True</td>
    </tr>
    <tr>
      <th>7</th>
      <td>xc_lev__NA_</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>-0.387438</td>
      <td>0.169451</td>
      <td>0.000000</td>
      <td>4.0</td>
      <td>0.05</td>
      <td>True</td>
    </tr>
    <tr>
      <th>8</th>
      <td>xc_lev_level_-0_5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>-0.373767</td>
      <td>0.158566</td>
      <td>0.000000</td>
      <td>4.0</td>
      <td>0.05</td>
      <td>True</td>
    </tr>
    <tr>
      <th>9</th>
      <td>xc_lev_level_0_5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>0.102752</td>
      <td>0.007894</td>
      <td>0.023774</td>
      <td>4.0</td>
      <td>0.05</td>
      <td>True</td>
    </tr>
  </tbody>
</table>
</div>



Note that the variable `xc` has been converted to multiple variables: 

* an indicator variable for each possible level (`xc_lev_level_*`)
* the value of a (cross-validated) one-variable model for `yc` as a function of `xc` (`xc_logit_code`)
* a variable that returns how prevalent this particular value of `xc` is in the training data (`xc_prevalence_code`)
* a variable indicating when `xc` was `NaN` in the original data (`xc_is_bad`, `x_is_bad`)

Any or all of these new variables are available for downstream modeling. `x` doesn't show as exciting a significance as `xc`, as we are only checking linear relations, and `x` is related to `y` in a very non-linear way.

The `recommended` column indicates which variables are non constant (`has_range` == True) and have a significance value smaller than `default_threshold`. See the section *Deriving the Default Thresholds* below for the reasoning behind the default thresholds. Recommended columns are intended as advice about which variables appear to be most likely to be useful in a downstream model. This advice attempts to be conservative, to reduce the possibility of mistakenly eliminating variables that may in fact be useful (although, obviously, it can still mistakenly eliminate variables that have a real but non-linear relationship to the output, as is the case with `x`, in  our example).

Let's look at the variables that are and are not recommended:


```python
# recommended variables
transform.score_frame_.loc[transform.score_frame_['recommended'], ['variable']]
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>variable</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>xc_is_bad</td>
    </tr>
    <tr>
      <th>4</th>
      <td>xc_logit_code</td>
    </tr>
    <tr>
      <th>5</th>
      <td>xc_prevalence_code</td>
    </tr>
    <tr>
      <th>6</th>
      <td>xc_lev_level_1_0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>xc_lev__NA_</td>
    </tr>
    <tr>
      <th>8</th>
      <td>xc_lev_level_-0_5</td>
    </tr>
    <tr>
      <th>9</th>
      <td>xc_lev_level_0_5</td>
    </tr>
  </tbody>
</table>
</div>




```python
# not recommended variables
transform.score_frame_.loc[~transform.score_frame_['recommended'], ['variable']]
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>variable</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>x_is_bad</td>
    </tr>
    <tr>
      <th>2</th>
      <td>x</td>
    </tr>
    <tr>
      <th>3</th>
      <td>x2</td>
    </tr>
  </tbody>
</table>
</div>



Notice that `d_prepared` only includes recommended variables (along with `y` and `yc`):


```python
d_prepared.head()
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>y</th>
      <th>yc</th>
      <th>xc_is_bad</th>
      <th>xc_logit_code</th>
      <th>xc_prevalence_code</th>
      <th>xc_lev_level_1_0</th>
      <th>xc_lev__NA_</th>
      <th>xc_lev_level_-0_5</th>
      <th>xc_lev_level_0_5</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>-0.956311</td>
      <td>False</td>
      <td>1.0</td>
      <td>-5.838582</td>
      <td>0.218</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>-0.671564</td>
      <td>False</td>
      <td>0.0</td>
      <td>-5.837275</td>
      <td>0.206</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0.906303</td>
      <td>True</td>
      <td>0.0</td>
      <td>1.047934</td>
      <td>0.280</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0.222792</td>
      <td>False</td>
      <td>0.0</td>
      <td>-5.790939</td>
      <td>0.074</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>-0.975431</td>
      <td>False</td>
      <td>1.0</td>
      <td>-5.837541</td>
      <td>0.218</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
  </tbody>
</table>
</div>



This is `vtreat`s default behavior; to include all variables in the prepared data, set the parameter `filter_to_recommended` to False, as we show later, in the *Parameters for `BinomialOutcomeTreatment`* section below.

## A Closer Look at `logit_code` variables

Variables of type `logit_code` are the outputs of a one-variable hierarchical logistic regression of a categorical variable (in our example, `xc`) against the centered output on the (cross-validated) treated training data. 

Let's see whether `xc_logit_code` makes a good one-variable model for `yc`. It has a large AUC:


```python
wvpy.util.plot_roc(prediction=d_prepared['xc_logit_code'], 
                   istrue=d_prepared['yc'],
                   title = 'performance of xc_logit_code variable')
```


![png](output_29_0.png)





    0.9862945054945056



This indicates that `xc_logit_code` is strongly predictive of the outcome. Negative values of `xc_logit_code` correspond strongly to negative outcomes, and positive values correspond strongly to positive outcomes.


```python
wvpy.util.dual_density_plot(probs=d_prepared['xc_logit_code'], 
                            istrue=d_prepared['yc'])
```


![png](output_31_0.png)


The values of `xc_logit_code` are in "link space". We can often visualize the relationship a little better by converting the logistic score to a probability.


```python
from scipy.special import expit  # sigmoid
from scipy.special import logit

offset = logit(numpy.mean(d_prepared.yc))
wvpy.util.dual_density_plot(probs=expit(d_prepared['xc_logit_code'] + offset),
                            istrue=d_prepared['yc'])                                   
```


![png](output_33_0.png)


Variables of type `logit_code` are useful when dealing with categorical variables with a very large number of possible levels. For example, a categorical variable with 10,000 possible values potentially converts to 10,000 indicator variables, which may be unwieldy for some modeling methods. Using a single numerical variable of type `logit_code` may be a preferable alternative.

## Using the Prepared Data in a Model

Of course, what we really want to do with the prepared training data is to fit a model jointly with all the (recommended) variables. 
Let's try fitting a logistic regression model to `d_prepared`.


```python
import sklearn.linear_model
import seaborn

not_variables = ['y', 'yc', 'prediction']
model_vars = [v for v in d_prepared.columns if v not in set(not_variables)]

fitter = sklearn.linear_model.LogisticRegression()
fitter.fit(d_prepared[model_vars], d_prepared['yc'])

# now predict
d_prepared['prediction'] = fitter.predict_proba(d_prepared[model_vars])[:, 1]

# look at the ROC curve (on the training data)
wvpy.util.plot_roc(prediction=d_prepared['prediction'], 
                   istrue=d_prepared['yc'],
                   title = 'Performance of logistic regression model on training data')
```

    /Users/johnmount/opt/anaconda3/envs/ai_academy_3_7/lib/python3.7/site-packages/sklearn/linear_model/logistic.py:432: FutureWarning: Default solver will be changed to 'lbfgs' in 0.22. Specify a solver to silence this warning.
      FutureWarning)



![png](output_36_1.png)





    0.9862945054945055



Now apply the model to new data.


```python
# create the new data
dtest = make_data(450)

# prepare the new data with vtreat
dtest_prepared = transform.transform(dtest)

# apply the model to the prepared data
dtest_prepared['prediction'] = fitter.predict_proba(dtest_prepared[model_vars])[:, 1]

wvpy.util.plot_roc(prediction=dtest_prepared['prediction'], 
                   istrue=dtest_prepared['yc'],
                   title = 'Performance of logistic regression model on test data')
```


![png](output_38_0.png)





    0.9726541064569234



## Parameters for `BinomialOutcomeTreatment`

We've tried to set the defaults for all parameters so that `vtreat` is usable out of the box for most applications.



```python
vtreat.vtreat_parameters()
```




    {'use_hierarchical_estimate': True,
     'coders': {'clean_copy',
      'deviation_code',
      'impact_code',
      'indicator_code',
      'logit_code',
      'missing_indicator',
      'prevalence_code'},
     'filter_to_recommended': True,
     'indicator_min_fraction': 0.1,
     'cross_validation_plan': vtreat.cross_plan.KWayCrossPlanYStratified(),
     'cross_validation_k': 5,
     'user_transforms': [],
     'sparse_indicators': True,
     'missingness_imputation': <function numpy.mean(a, axis=None, dtype=None, out=None, keepdims=<no value>)>,
     'check_for_duplicate_frames': True,
     'retain_cross_plan': False}



**use_hierarchical_estimate:**: When True, uses hierarchical smoothing when estimating `logit_code` variables; when False, uses unsmoothed logistic regression.

**coders**: The types of synthetic variables that `vtreat` will (potentially) produce. See *Types of prepared variables* below.

**filter_to_recommended**: When True, prepared data only includes variables marked as "recommended" in score frame. When False, prepared data includes all variables. See the Example below.

**indicator_min_fraction**: For categorical variables, indicator variables (type `indicator_code`) are only produced for levels that are present at least `indicator_min_fraction` of the time. A consequence of this is that 1/`indicator_min_fraction` is the maximum number of indicators that will be produced for a given categorical variable. To make sure that *all* possible indicator variables are produced, set `indicator_min_fraction = 0`

**cross_validation_plan**: The cross validation method used by `vtreat`. Most people won't have to change this.

**cross_validation_k**: The number of folds to use for cross-validation

**user_transforms**: For passing in user-defined transforms for custom data preparation. Won't be needed in most situations, but see [here](https://github.com/WinVector/pyvtreat/blob/master/Examples/UserCoders/UserCoders.ipynb) for an example of applying a GAM transform to input variables.

**sparse_indicators**: When True, use a (Pandas) sparse representation for indicator variables. This representation is compatible with `sklearn`; however, it may not be compatible with other modeling packages. When False, use a dense representation.

**missingness_imputation** The function or value that `vtreat` uses to impute or "fill in" missing numerical values. The default is `numpy.mean()`. To change the imputation function or use different functions/values for different columns, see the [Imputation example](https://github.com/WinVector/pyvtreat/blob/master/Examples/Imputation/Imputation.ipynb).

### Example: Use all variables to model, not just recommended


```python
transform_all = vtreat.BinomialOutcomeTreatment(
    outcome_name='yc',    # outcome variable
    outcome_target=True,  # outcome of interest
    cols_to_copy=['y'],   # columns to "carry along" but not treat as input variables
    params = vtreat.vtreat_parameters({
        'filter_to_recommended': False
    })
)  

transform_all.fit_transform(d, d['yc']).columns
```




    Index(['y', 'yc', 'x_is_bad', 'xc_is_bad', 'x', 'x2', 'xc_logit_code',
           'xc_prevalence_code', 'xc_lev_level_1_0', 'xc_lev__NA_',
           'xc_lev_level_-0_5', 'xc_lev_level_0_5'],
          dtype='object')




```python
transform_all.score_frame_
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>variable</th>
      <th>orig_variable</th>
      <th>treatment</th>
      <th>y_aware</th>
      <th>has_range</th>
      <th>PearsonR</th>
      <th>R2</th>
      <th>significance</th>
      <th>vcount</th>
      <th>default_threshold</th>
      <th>recommended</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>x_is_bad</td>
      <td>x</td>
      <td>missing_indicator</td>
      <td>False</td>
      <td>True</td>
      <td>-0.051749</td>
      <td>0.002388</td>
      <td>0.213707</td>
      <td>2.0</td>
      <td>0.10</td>
      <td>False</td>
    </tr>
    <tr>
      <th>1</th>
      <td>xc_is_bad</td>
      <td>xc</td>
      <td>missing_indicator</td>
      <td>False</td>
      <td>True</td>
      <td>-0.387438</td>
      <td>0.169451</td>
      <td>0.000000</td>
      <td>2.0</td>
      <td>0.10</td>
      <td>True</td>
    </tr>
    <tr>
      <th>2</th>
      <td>x</td>
      <td>x</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>0.052826</td>
      <td>0.002158</td>
      <td>0.237141</td>
      <td>2.0</td>
      <td>0.10</td>
      <td>False</td>
    </tr>
    <tr>
      <th>3</th>
      <td>x2</td>
      <td>x2</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>0.069126</td>
      <td>0.003709</td>
      <td>0.121205</td>
      <td>2.0</td>
      <td>0.10</td>
      <td>False</td>
    </tr>
    <tr>
      <th>4</th>
      <td>xc_logit_code</td>
      <td>xc</td>
      <td>logit_code</td>
      <td>True</td>
      <td>True</td>
      <td>0.867909</td>
      <td>0.831229</td>
      <td>0.000000</td>
      <td>1.0</td>
      <td>0.20</td>
      <td>True</td>
    </tr>
    <tr>
      <th>5</th>
      <td>xc_prevalence_code</td>
      <td>xc</td>
      <td>prevalence_code</td>
      <td>False</td>
      <td>True</td>
      <td>0.567968</td>
      <td>0.341474</td>
      <td>0.000000</td>
      <td>1.0</td>
      <td>0.20</td>
      <td>True</td>
    </tr>
    <tr>
      <th>6</th>
      <td>xc_lev_level_1_0</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>0.849837</td>
      <td>0.645319</td>
      <td>0.000000</td>
      <td>4.0</td>
      <td>0.05</td>
      <td>True</td>
    </tr>
    <tr>
      <th>7</th>
      <td>xc_lev__NA_</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>-0.387438</td>
      <td>0.169451</td>
      <td>0.000000</td>
      <td>4.0</td>
      <td>0.05</td>
      <td>True</td>
    </tr>
    <tr>
      <th>8</th>
      <td>xc_lev_level_-0_5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>-0.373767</td>
      <td>0.158566</td>
      <td>0.000000</td>
      <td>4.0</td>
      <td>0.05</td>
      <td>True</td>
    </tr>
    <tr>
      <th>9</th>
      <td>xc_lev_level_0_5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>0.102752</td>
      <td>0.007894</td>
      <td>0.023774</td>
      <td>4.0</td>
      <td>0.05</td>
      <td>True</td>
    </tr>
  </tbody>
</table>
</div>



Note that the prepared data produced by `fit_transform()` includes all the variables, including those that were not marked as "recommended". 

## Types of prepared variables

**clean_copy**: Produced from numerical variables: a clean numerical variable with no `NaNs` or missing values

**indicator_code**: Produced from categorical variables, one for each (common) level: for each level of the variable, indicates if that level was "on"

**prevalence_code**: Produced from categorical variables: indicates how often each level of the variable was "on"

**logit_code**: Produced from categorical variables: score from a one-dimensional model of the centered output as a function of the variable

**missing_indicator**: Produced for both numerical and categorical variables: an indicator variable that marks when the original variable was missing or  `NaN`

**deviation_code**: not used by `BinomialOutcomeTreatment`

**impact_code**: not used by `BinomialOutcomeTreatment`

### Example: Produce only a subset of variable types

In this example, suppose you only want to use indicators and continuous variables in your model; 
in other words, you only want to use variables of types (`clean_copy`, `missing_indicator`, and `indicator_code`), and no `logit_code` or `prevalence_code` variables.


```python
transform_thin = vtreat.BinomialOutcomeTreatment(
    outcome_name='yc',    # outcome variable
    outcome_target=True,  # outcome of interest
    cols_to_copy=['y'],   # columns to "carry along" but not treat as input variables
    params = vtreat.vtreat_parameters({
        'filter_to_recommended': False,
        'coders': {'clean_copy',
                   'missing_indicator',
                   'indicator_code',
                  }
    })
)

transform_thin.fit_transform(d, d['yc']).head()
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>y</th>
      <th>yc</th>
      <th>x_is_bad</th>
      <th>xc_is_bad</th>
      <th>x</th>
      <th>x2</th>
      <th>xc_lev_level_1_0</th>
      <th>xc_lev__NA_</th>
      <th>xc_lev_level_-0_5</th>
      <th>xc_lev_level_0_5</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>-0.956311</td>
      <td>False</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>-1.088395</td>
      <td>-1.424184</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>-0.671564</td>
      <td>False</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>4.107277</td>
      <td>0.427360</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0.906303</td>
      <td>True</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>7.406389</td>
      <td>0.668849</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0.222792</td>
      <td>False</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>-0.057044</td>
      <td>-0.015787</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>-0.975431</td>
      <td>False</td>
      <td>1.0</td>
      <td>1.0</td>
      <td>-0.057044</td>
      <td>-0.491017</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>0.0</td>
      <td>0.0</td>
    </tr>
  </tbody>
</table>
</div>




```python
transform_thin.score_frame_
```




<div>

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>variable</th>
      <th>orig_variable</th>
      <th>treatment</th>
      <th>y_aware</th>
      <th>has_range</th>
      <th>PearsonR</th>
      <th>R2</th>
      <th>significance</th>
      <th>vcount</th>
      <th>default_threshold</th>
      <th>recommended</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>x_is_bad</td>
      <td>x</td>
      <td>missing_indicator</td>
      <td>False</td>
      <td>True</td>
      <td>-0.051749</td>
      <td>0.002388</td>
      <td>0.213707</td>
      <td>2.0</td>
      <td>0.166667</td>
      <td>False</td>
    </tr>
    <tr>
      <th>1</th>
      <td>xc_is_bad</td>
      <td>xc</td>
      <td>missing_indicator</td>
      <td>False</td>
      <td>True</td>
      <td>-0.387438</td>
      <td>0.169451</td>
      <td>0.000000</td>
      <td>2.0</td>
      <td>0.166667</td>
      <td>True</td>
    </tr>
    <tr>
      <th>2</th>
      <td>x</td>
      <td>x</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>0.052826</td>
      <td>0.002158</td>
      <td>0.237141</td>
      <td>2.0</td>
      <td>0.166667</td>
      <td>False</td>
    </tr>
    <tr>
      <th>3</th>
      <td>x2</td>
      <td>x2</td>
      <td>clean_copy</td>
      <td>False</td>
      <td>True</td>
      <td>0.069126</td>
      <td>0.003709</td>
      <td>0.121205</td>
      <td>2.0</td>
      <td>0.166667</td>
      <td>True</td>
    </tr>
    <tr>
      <th>4</th>
      <td>xc_lev_level_1_0</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>0.849837</td>
      <td>0.645319</td>
      <td>0.000000</td>
      <td>4.0</td>
      <td>0.083333</td>
      <td>True</td>
    </tr>
    <tr>
      <th>5</th>
      <td>xc_lev__NA_</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>-0.387438</td>
      <td>0.169451</td>
      <td>0.000000</td>
      <td>4.0</td>
      <td>0.083333</td>
      <td>True</td>
    </tr>
    <tr>
      <th>6</th>
      <td>xc_lev_level_-0_5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>-0.373767</td>
      <td>0.158566</td>
      <td>0.000000</td>
      <td>4.0</td>
      <td>0.083333</td>
      <td>True</td>
    </tr>
    <tr>
      <th>7</th>
      <td>xc_lev_level_0_5</td>
      <td>xc</td>
      <td>indicator_code</td>
      <td>False</td>
      <td>True</td>
      <td>0.102752</td>
      <td>0.007894</td>
      <td>0.023774</td>
      <td>4.0</td>
      <td>0.083333</td>
      <td>True</td>
    </tr>
  </tbody>
</table>
</div>



## Deriving the Default Thresholds

While machine learning algorithms are generally tolerant to a reasonable number of irrelevant or noise variables, too many irrelevant variables can lead to serious overfit; see [this article](http://www.win-vector.com/blog/2014/02/bad-bayes-an-example-of-why-you-need-hold-out-testing/) for an extreme example, one we call "Bad Bayes". The default threshold is an attempt to eliminate obviously irrelevant variables early.

Imagine that you have a pure noise dataset, where none of the *n* inputs are related to the output. If you treat each variable as a one-variable model for the output, and look at the significances of each model, these significance-values will be uniformly distributed in the range [0:1]. You want to pick a weakest possible significance threshold that eliminates as many noise variables as possible. A moment's thought should convince you that a threshold of *1/n* allows only one variable through, in expectation. 

This leads to the general-case heuristic that a significance threshold of *1/n* on your variables should allow only one irrelevant variable through, in expectation (along with all the relevant variables). Hence, *1/n* used to be our recommended threshold, when we developed the R version of `vtreat`.

We noticed, however, that this biases the filtering against numerical variables, since there are at most two derived variables (of types *clean_copy* and *missing_indicator* for every numerical variable in the original data. Categorical variables, on the other hand, are expanded to many derived variables: several indicators (one for every common level), plus a *logit_code* and a *prevalence_code*. So we now reweight the thresholds. 

Suppose you have a (treated) data set with *ntreat* different types of `vtreat` variables (`clean_copy`, `indicator_code`, etc).
There are *nT* variables of type *T*. Then the default threshold for all the variables of type *T* is *1/(ntreat nT)*. This reweighting  helps to reduce the bias against any particular type of variable. The heuristic is still that the set of recommended variables will allow at most one noise variable into the set of candidate variables.

As noted above, because `vtreat` estimates variable significances using linear methods by default, some variables with a non-linear relationship  to the output may fail to pass the threshold. Setting the `filter_to_recommended` parameter to False will keep all derived variables in the treated frame, for the data scientist to filter (or not) as they will.



## Conclusion

In all cases (classification, regression, unsupervised, and multinomial classification) the intent is that `vtreat` transforms are essentially one liners.

The preparation commands are organized as follows:


 * **Regression**: [`Python` regression example](https://github.com/WinVector/pyvtreat/blob/master/Examples/Regression/Regression.md), [`R` regression example, fit/prepare interface](https://github.com/WinVector/vtreat/blob/master/Examples/Regression/Regression_FP.md), [`R` regression example, design/prepare/experiment interface](https://github.com/WinVector/vtreat/blob/master/Examples/Regression/Regression.md).
 * **Classification**: [`Python` classification  example](https://github.com/WinVector/pyvtreat/blob/master/Examples/Classification/Classification.md), [`R` classification example, fit/prepare interface](https://github.com/WinVector/vtreat/blob/master/Examples/Classification/Classification_FP.md), [`R` classification example, design/prepare/experiment interface](https://github.com/WinVector/vtreat/blob/master/Examples/Classification/Classification.md).
 * **Unsupervised tasks**: [`Python` unsupervised example](https://github.com/WinVector/pyvtreat/blob/master/Examples/Unsupervised/Unsupervised.md), [`R` unsupervised example, fit/prepare interface](https://github.com/WinVector/vtreat/blob/master/Examples/Unsupervised/Unsupervised_FP.md), [`R` unsupervised example, design/prepare/experiment interface](https://github.com/WinVector/vtreat/blob/master/Examples/Unsupervised/Unsupervised.md).
 * **Multinomial classification**: [`Python` multinomial classification example](https://github.com/WinVector/pyvtreat/blob/master/Examples/Multinomial/MultinomialExample.md), [`R` multinomial classification example, fit/prepare interface](https://github.com/WinVector/vtreat/blob/master/Examples/Multinomial/MultinomialExample_FP.md), [`R` multinomial classification example, design/prepare/experiment interface](https://github.com/WinVector/vtreat/blob/master/Examples/Multinomial/MultinomialExample.md).


Some `vtreat` common capabilities are documented here:

  * **Score Frame** [score_frame_](https://github.com/WinVector/pyvtreat/blob/master/Examples/ScoreFrame/ScoreFrame.md), using the `score_frame_` information.
  * **Cross Validation** [Customized Cross Plans](https://github.com/WinVector/pyvtreat/blob/master/Examples/CustomizedCrossPlan/CustomizedCrossPlan.md), controlling the cross validation plan.

These current revisions of the examples are designed to be small, yet complete.  So as a set they have some overlap, but the user can rely mostly on a single example for a single task type.


```python

```


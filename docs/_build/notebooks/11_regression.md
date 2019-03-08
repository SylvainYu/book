---
redirect_from:
  - "/notebooks/11-regression"
interact_link: content/notebooks/11_regression.ipynb
kernel_name: python3
title: 'Regression over space'
prev_page:
  url: /notebooks/07_local_autocorrelation
  title: 'Local Autocorrelation'
next_page:
  url: 
  title: ''
comment: "***PROGRAMMATICALLY GENERATED, DO NOT EDIT. SEE ORIGINAL FILES IN /content***"
---

# Spatial regression



{:.input_area}
```python
%matplotlib inline

from pysal.model import spreg
from pysal.lib import weights
from pysal.explore import esda
from scipy import stats
import statsmodels.formula.api as sm
import numpy
import pandas
import geopandas
import bookdata
import matplotlib.pyplot as plt
import seaborn
```


# Introduction
   
**NOTE**: this chapter was written building from this [Spatial Analysis note](http://darribas.org/spa_notes/sp_eco.html).

## *What* is spatial regression and *why* should I care?

Regression (and prediction more generally) provides us a perfect case to examine how spatial structure can help us understand and analyze our data. 
Usually, spatial structure helps models in one of two ways. 
The first (and most clear) way space can have an impact on our data is when the process *generating* the data is itself explicitly spatial.
Here, think of something like the prices for single family homes. 
It's often the case that individuals pay a premium on their house price in order to live in a better school district for the same quality house. 
Alternatively, homes closer to noise or chemical polluters like waste water treatment plants, recycling facilities, or wide highways, may actually be cheaper than we would otherwise anticipate. 
Finally, in cases like asthma incidence, the locations individuals tend to travel to throughout the day, such as their places of work or recreation, may have more impact on their health than their residential addresses. 
In this case, it may be necessary to use data *from other sites* to predict the asthma incidence at a given site. 
Regardless of the specific case at play, here, *geography is a feature*: it directly helps us make predictions about outcomes *because those outcomes obtain from geographical processes*. 

An alternative (and more skeptical understanding) reluctantly acknowledges geography's instrumental value. 
Often, in the analysis of predictive methods and classifiers, we are interested in analyzing what we get wrong.
This is common in econometrics; an analyst may be concerned that the model *systematically* mis-predicts some types of observations.
If we know our model routinely performs poorly on a known set of observations or type of input, we might make a better model if we can account for this. 
Among other kinds of error diagnostics, geography provides us with an exceptionally-useful embedding to assess structure in our errors.
Mapping classification/prediction error can help show whether or not there are *clusters of error* in our data.
If we *know* that errors tend to be larger in some areas than in other areas (or if error is "contagious" between observations), then we might be able to exploit this structure to make better predictions.

Spatial structure in our errors might arise from when geography *should be* an attribute somehow, but we are not sure exactly how to include it in our model. 
They might also arise because there is some *other* feature whose omission causes the spatial patterns in the error we see; if this additional feature were included, the structure would disappear. 
Or, it might arise from the complex interactions and interdependences between the features that we have chosen to use as predictors, resulting in intrinsic structure in mis-prediction. 
Most of the predictors we use in models of social processes contain *embodied* spatial information: patterning intrinsic to the feature that we get for free in the model. 
If we intend to or not, using a spatially-patterned predictor in a model can result in spatially-patterned errors; using more than one can amplify this effect. 
Thus, *regardless of whether or not the true process is explicitly geographic*, additional information about the spatial relationships between our observations or more information about nearby sites can make our predictions better. 

## The Data: San Diego AirBnB

To learn a little more about how regression works, we'll examine some information about AirBnB in San Diego, CA. 
This dataset contains house intrinsic characteristics, both continuous (number of beds as in `beds`) and categorical (type of renting or, in AirBnb jargon, property group as in the series of `pg_X` binary variables), but also variables that explicitly refer to the location and spatial configuration of the dataset (e.g. distance to Balboa Park, `d2balboa` or neigbourhood id, `neighbourhood_cleansed`).



{:.input_area}
```python
db = geopandas.read_file(bookdata.regression_airbnbs())
db.info()
```


{:.output .output_stream}
```
<class 'geopandas.geodataframe.GeoDataFrame'>
RangeIndex: 6110 entries, 0 to 6109
Data columns (total 20 columns):
accommodates          6110 non-null int64
bathrooms             6110 non-null float64
bedrooms              6110 non-null float64
beds                  6110 non-null float64
neighborhood          6110 non-null object
pool                  6110 non-null int64
d2balboa              6110 non-null float64
coastal               6110 non-null int64
price                 6110 non-null float64
log_price             6110 non-null float64
id                    6110 non-null int64
pg_Apartment          6110 non-null int64
pg_Condominium        6110 non-null int64
pg_House              6110 non-null int64
pg_Other              6110 non-null int64
pg_Townhouse          6110 non-null int64
rt_Entire_home/apt    6110 non-null int64
rt_Private_room       6110 non-null int64
rt_Shared_room        6110 non-null int64
geometry              6110 non-null object
dtypes: float64(6), int64(12), object(2)
memory usage: 954.8+ KB

```

These are the explanatory variables we will use throughout the chapter.



{:.input_area}
```python
variable_names = ['accommodates', 'bathrooms', 'bedrooms', 
                  'beds', 'rt_Private_room', 'rt_Shared_room',
                  'pg_Condominium', 'pg_House', 
                  'pg_Other', 'pg_Townhouse']
```


## Non-spatial regression, a (very) quick refresh

Before we discuss how to explicitly include space into the linear regression framework, let us show how basic regression can be carried out in Python, and how one can begin to interpret the results. By no means is this a formal and complete introduction to regression so, if that is what you are looking for, we recommend Gelman & Hill (2006), in particular chapters 3 and 4, which provide a fantastic, non-spatial introduction.

The core idea of linear regression is to explain the variation in a given (*dependent*) variable as a linear function of a collection of other (*explanatory*) variables. For example, in our case, we may want to express/explain the price of a house as a function of whether it is new and the degree of deprivation of the area where it is located. At the individual level, we can express this as:

$$
P_i = \alpha + \sum_k \beta_k X_{ki} + \epsilon_i
$$

where $P_i$ is the AirBnb price of house $i$, and $X$ is a set of covariates that we use to explain such price. $\beta$ is a vector of parameters that give us information about in which way and to what extent each variable is related to the price, and $\alpha$, the constant term, is the average house price when all the other variables are zero. The term $\epsilon_i$ is usually referred to as "error" and captures elements that influence the price of a house but are not included in $X$. We can also express this relation in matrix form, excluding subindices for $i$, which yields:

$$
P = \alpha + \beta X + \epsilon
$$

A regression can be seen as a multivariate extension of bivariate correlations. Indeed, one way to interpret the $\beta_k$ coefficients in the equation above is as the degree of correlation between the explanatory variable $k$ and the dependent variable, *keeping all the other explanatory variables constant*. When one calculates bivariate correlations, the coefficient of a variable is picking up the correlation between the variables, but it is also subsuming into it variation associated with other correlated variables -- also called confounding factors. Regression allows us to isolate the distinct effect that a single variable has on the dependent one, once we *control* for those other variables.

Practically speaking, linear regressions in Python are rather streamlined and easy to work with. There are also several packages which will run them (e.g. `statsmodels`, `scikit-learn`, `PySAL`). In the context of this chapter, it makes sense to start with `PySAL` as that is the only library that will allow us to move into explicitly spatial econometric models. To fit the model specified in the equation above with $X$ as the list defined, we only need the following line of code:



{:.input_area}
```python
m1 = spreg.OLS(db[['log_price']].values, db[variable_names].values,
                name_y='log_price', name_x=variable_names)
```


We use the command `OLS`, part of the `spreg` sub-package, and specify the dependent variable (the log of the price, so we can interpret results in terms of percentage change) and the explanatory ones. Note that both objects need to be arrays, so we extract them from the `pandas.DataFrame` object using `.values`.

In order to inspect the results of the model, we can call `summary`:



{:.input_area}
```python
print(m1.summary)
```


{:.output .output_stream}
```
REGRESSION
----------
SUMMARY OF OUTPUT: ORDINARY LEAST SQUARES
-----------------------------------------
Data set            :     unknown
Weights matrix      :        None
Dependent Variable  :   log_price                Number of Observations:        6110
Mean dependent var  :      4.9958                Number of Variables   :          11
S.D. dependent var  :      0.8072                Degrees of Freedom    :        6099
R-squared           :      0.6683
Adjusted R-squared  :      0.6678
Sum squared residual:    1320.148                F-statistic           :   1229.0564
Sigma-square        :       0.216                Prob(F-statistic)     :           0
S.E. of regression  :       0.465                Log likelihood        :   -3988.895
Sigma-square ML     :       0.216                Akaike info criterion :    7999.790
S.E of regression ML:      0.4648                Schwarz criterion     :    8073.685

------------------------------------------------------------------------------------
            Variable     Coefficient       Std.Error     t-Statistic     Probability
------------------------------------------------------------------------------------
            CONSTANT       4.3883830       0.0161147     272.3217773       0.0000000
        accommodates       0.0834523       0.0050781      16.4336318       0.0000000
           bathrooms       0.1923790       0.0109668      17.5419773       0.0000000
            bedrooms       0.1525221       0.0111323      13.7009195       0.0000000
                beds      -0.0417231       0.0069383      -6.0134430       0.0000000
     rt_Private_room      -0.5506868       0.0159046     -34.6244758       0.0000000
      rt_Shared_room      -1.2383055       0.0384329     -32.2198992       0.0000000
      pg_Condominium       0.1436347       0.0221499       6.4846529       0.0000000
            pg_House      -0.0104894       0.0145315      -0.7218393       0.4704209
            pg_Other       0.1411546       0.0228016       6.1905633       0.0000000
        pg_Townhouse      -0.0416702       0.0342758      -1.2157316       0.2241342
------------------------------------------------------------------------------------

REGRESSION DIAGNOSTICS
MULTICOLLINEARITY CONDITION NUMBER           11.964

TEST ON NORMALITY OF ERRORS
TEST                             DF        VALUE           PROB
Jarque-Bera                       2        2671.611           0.0000

DIAGNOSTICS FOR HETEROSKEDASTICITY
RANDOM COEFFICIENTS
TEST                             DF        VALUE           PROB
Breusch-Pagan test               10         322.532           0.0000
Koenker-Bassett test             10         135.581           0.0000
================================ END OF REPORT =====================================

```

A full detailed explanation of the output is beyond the scope of this chapter, so we will focus on the relevant bits for our main purpose. This is concentrated on the `Coefficients` section, which gives us the estimates for $\beta_k$ in our model. In other words, these numbers express the relationship between each explanatory variable and the dependent one, once the effect of confounding factors has been accounted for. Keep in mind however that regression is no magic; we are only discounting the effect of confounding factors that we include in the model, not of *all* potentially confounding factors.

Results are largely as expected: houses tend to be significantly more expensive if they accommodate more people (`accommodates`), if they have more bathrooms and bedrooms and if they are a condominium or part of the "other" category of house type. Conversely, given a number of rooms, houses with more beds (ie. listings that are more "crowded") tend to go for cheaper, as it is the case for properties where one does not rent the entire house but only a room (`rt_Private_room`) or even shares it (`rt_Shared_room`). Of course, you might conceptually doubt the assumption that it is possible to *arbitrarily* change the number of beds within an Airbnb without eventually changing the number of people it accommodates, but methods to address these concerns using *interaction effects* won't be discussed here. 

### Hidden Structures

In general, our model performs well, being able to predict slightly more than 65% ($R^2=0.67$) of the variation in the mean nightly price using the covariates we've discussed above.
But, our model might display some clustering in errors. 
To interrogate this, we can do a few things. 
One simple concept might be to look at the correlation between the error in predicting an airbnb and the error in predicting its nearest neighbor. 
To examine this, we first might want to split our data up by regions and see if we've got some spatial structure in our residuals. 
One reasonable theory might be that our model does not include any information about *beaches*, a critical aspect of why people live and vacation in San Diego. 
Therefore, we might want to see whether or not our errors are higher or lower depending on whether or not an airbnb is in a "beach" neighborhood, a neighborhood near the ocean:



{:.input_area}
```python
is_coastal = db.coastal.astype(bool)
coastal = m1.u[is_coastal]
not_coastal = m1.u[~is_coastal]
plt.hist(coastal, density=True, label='Coastal')
plt.hist(not_coastal, histtype='step',
         density=True, linewidth=4, label='Not Coastal')
plt.vlines(0,0,1, linestyle=":", color='k', linewidth=4)
plt.legend()
plt.show()
```



{:.output .output_png}
![png](../images/notebooks/11_regression_11_0.png)



While it appears that the neighborhoods on the coast have only slightly higher average errors (and have lower variance in their prediction errors), the two distributions are significantly distinct from one another when compared using a classic $t$ test:



{:.input_area}
```python
stats.ttest_ind(coastal, 
             not_coastal,
#             permutations=9999 not yet available in scipy
             )
```





{:.output .output_data_text}
```
Ttest_indResult(statistic=array([13.98193858]), pvalue=array([9.442438e-44]))
```



There are more sophisticated (and harder to fool) tests that may be applicable for this data, however. We cover them in the [Challenge](#Challenge) section. 

Additionally, it might be the case that some neighborhoods are more desirable than other neighborhoods due to unmodeled latent preferences or marketing. 
For instance, despite its presence close to the sea, living near Camp Pendleton -a Marine base in the North of the city- may incur some significant penalties on area desirability due to noise and pollution. 
For us to determine whether this is the case, we might be interested in the full distribution of model residuals within each neighborhood. 

To make this more clear, we'll first sort the data by the median residual in that neighborhood, and then make a box plot, which shows the distribution of residuals in each neighborhood:



{:.input_area}
```python
db['residual'] = m1.u
medians = db.groupby("neighborhood").residual.median().to_frame('hood_residual')

f = plt.figure(figsize=(15,3))
ax = plt.gca()
seaborn.boxplot('neighborhood', 'residual', ax = ax,
                data=db.merge(medians, how='left',
                              left_on='neighborhood',
                              right_index=True)
                   .sort_values('hood_residual'), palette='bwr')
f.autofmt_xdate()
plt.show()
```



{:.output .output_png}
![png](../images/notebooks/11_regression_16_0.png)



No neighborhood is disjoint from one another, but some do appear to be higher than others, such as the well-known downtown tourist neighborhoods areas of the Gaslamp Quarter, Little Italy, or The Core. 
Thus, there may be a distinctive effect of intangible neighborhood fashionableness that matters in this model. 

Noting that many of the most over- and under-predicted neighborhoods are near one another in the city, it may also be the case that there is some sort of *contagion* or spatial spillovers in the nightly rent price. 
This often is apparent when individuals seek to price their airbnb listings to compete with similar nearby listings. 
Since our model is not aware of this behavior, its errors may tend to cluster. 
One exceptionally simple way we can look into this structure is by examining the relationship between an observation's residuals and its surrounding residuals. 

To do this, we will use *spatial weights* to represent the geographic relationships between observations. 
We cover spatial weights in detail in another chapter, so we will not repeat ourselves here.
For this example, we'll start off with a $KNN$ matrix where $k=1$, meaning we're focusing only on the linkages of each airbnb to their closest other listing.



{:.input_area}
```python
knn = weights.Distance.KNN.from_dataframe(db, k=1)
```


This means that, when we compute the *spatial lag* of that knn weight and the residual, we get the residual of the airbnb listing closest to each observation.



{:.input_area}
```python
lag_residual = weights.spatial_lag.lag_spatial(knn, m1.u)
ax = seaborn.regplot(m1.u.flatten(), lag_residual.flatten(), 
                     line_kws=dict(color='orangered'),
                     ci=None)
ax.set_xlabel('Model Residuals - $u$')
ax.set_ylabel('Spatial Lag of Model Residuals - $W u$');
```



{:.output .output_png}
![png](../images/notebooks/11_regression_20_0.png)



In this plot, we see that our prediction errors tend to cluster!
Above, we show the relationship between our prediction error at each site and the prediction error at the site nearest to it. 
Here, we're using this nearest site to stand in for the *surroundings* of that Airbnb. 
This means that, when the model tends to overpredict a given Airbnb's nightly log price, sites around that Airbnb are more likely to *also be overpredicted*. 

An interesting property of this relationship is that it tends to stabilize as the number of nearest neighbors used to construct each Airbnb's surroundings increases.
Consult the [Challenge](#Challenge) section for more on this property. 

Given this behavior, let's look at the stable $k=20$ number of neighbors.
Examining the relationship between this stable *surrounding* average and the focal Airbnb, we can even find clusters in our model error. 
Recalling the *local Moran* statistics, we can identify certain areas where our predictions of the nightly (log) Airbnb price tend to be significantly off:



{:.input_area}
```python
knn.reweight(k=20, inplace=True)
outliers = esda.moran.Moran_Local(m1.u, knn, permutations=9999)
error_clusters = (outliers.q % 2 == 1) # only the cluster cores
error_clusters &= (outliers.p_sim <= .001) # filtering out non-significant clusters
db.assign(error_clusters = error_clusters,
          local_I = outliers.Is)\
  .query("error_clusters")\
  .sort_values('local_I')\
  .plot('local_I', cmap='bwr', marker='.');
```



{:.output .output_png}
![png](../images/notebooks/11_regression_23_0.png)



Thus, these areas tend to be locations where our model significantly underpredicts the nightly airbnb price both for that specific observation and observations in its immediate surroundings. 
This is critical since, if we can identify how these areas are structured &mdash; if they have a *consistent geography* that we can model &mdash; then we might make our predictions even better, or at least not systematically mis-predict prices in some areas while correctly predicting prices in other areas. 

Since significant under- and over-predictions do appear to cluster in a highly structured way, we might be able to use a better model to fix the geography of our model errors. 


## Bringing space into the regression framework

There are many different ways that spatial structure shows up in our models, predictions, and our data, even if we do not explicitly intend to study it.
Fortunately, there are nearly as many techniques, called *spatial regression* methods, that are designed to handle these sorts of structures.
Spatial regression is about *explicitly* introducing space or geographical context into the statistical framework of a regression. 
Conceptually, we want to introduce space into our model whenever we think it plays an important role in the process we are interested in, or when space can act as a reasonable proxy for other factors we cannot but should include in our model. 
As an example of the former, we can imagine how houses at the seafront are probably more expensive than those in the second row, given their better views. 
To illustrate the latter, we can think of how the character of a neighborhood is important in determining the price of a house; however, it is very hard to identify and quantify "character" *per se,* although it might be easier to get at its spatial variation, hence a case of space as a proxy.

Spatial regression is a large field of development in the econometrics and statistics literatures. 
In this brief introduction, we will consider two related but very different processes that give rise to spatial effects: spatial heterogeneity and spatial dependence. 
For more rigorous treatments of the topics introduced here, the reader is referred to [1-3].

### Spatial Feature Engineering

Using geographic information to "construct" new data is a common approach to bring in spatial information into geographic analysis. 
Often, this reflects the fact that processes are not the same everywhere in the map of analysis. 
Indeed, this heterogeneity can have a large impact on how a process is understood or modeled. 

Spatial heterogeneity (SH) arises when we cannot safely assume the process we are studying operates under the same "rules" throughout the geography of interest. In other words, we can observe SH when there are effects on the outcome variable that are intrinsically linked to specific locations. A good example of this is the case of seafront houses above: we are trying to model the price of a house and, the fact some houses are located under certain conditions (i.e. by the sea), makes their price behave differently.

The abstract concept of SH can be made operational in a model in several ways. We will explore the following three: proximity variables, spatial fixed-effects (FE), and spatial regimes &mdash; which itself is a generalization of FE.

#### Proximity variables

For a start, one relevant proximity-driven variable that could influence our model is based on the listings proximity to Balboa Park. A common tourist destination, Balboa park is a central recreation hub for the city of San Diego, containing many museums and the San Diego zoo. Thus, it could be the case that people searching for Airbnbs in San Diego are willing to pay a premium to live closer to the park. If this were true *and* we omitted this from our model, we may indeed see a significant spatial pattern caused by this distance decay effect. 

Therefore, this is sometimes called a *spatially-patterned omitted covariate*: geographic information our model needs to make good precitions which we have left out of our model. Therefore, let's build a new model containing this distance to Balboa Park covariate. First, though, it helps to visualize the structure of this distance covariate itself:



{:.input_area}
```python
db.plot('d2balboa', marker='.', s=5)
```





{:.output .output_data_text}
```
<matplotlib.axes._subplots.AxesSubplot at 0x1a1ccc31d0>
```




{:.output .output_png}
![png](../images/notebooks/11_regression_26_1.png)





{:.input_area}
```python
base_names = variable_names
balboa_names = variable_names + ['d2balboa']
```




{:.input_area}
```python
m2 = spreg.OLS(db[['log_price']].values, db[balboa_names].values, 
                  name_y = 'log_price', name_x = balboa_names)
```


Unfortunately, when you inspect the regression diagnostics and output, you see that this covariate is not quite as helpful as we might anticipate. It is not statistically significant at conventional significance levels, the model fit does not substantially change:



{:.input_area}
```python
print(m2.summary)
```


{:.output .output_stream}
```
REGRESSION
----------
SUMMARY OF OUTPUT: ORDINARY LEAST SQUARES
-----------------------------------------
Data set            :     unknown
Weights matrix      :        None
Dependent Variable  :   log_price                Number of Observations:        6110
Mean dependent var  :      4.9958                Number of Variables   :          12
S.D. dependent var  :      0.8072                Degrees of Freedom    :        6098
R-squared           :      0.6685
Adjusted R-squared  :      0.6679
Sum squared residual:    1319.522                F-statistic           :   1117.9338
Sigma-square        :       0.216                Prob(F-statistic)     :           0
S.E. of regression  :       0.465                Log likelihood        :   -3987.446
Sigma-square ML     :       0.216                Akaike info criterion :    7998.892
S.E of regression ML:      0.4647                Schwarz criterion     :    8079.504

------------------------------------------------------------------------------------
            Variable     Coefficient       Std.Error     t-Statistic     Probability
------------------------------------------------------------------------------------
            CONSTANT       4.3796237       0.0169152     258.9162210       0.0000000
        accommodates       0.0836436       0.0050786      16.4698200       0.0000000
           bathrooms       0.1907912       0.0110047      17.3371724       0.0000000
            bedrooms       0.1507462       0.0111794      13.4842887       0.0000000
                beds      -0.0414762       0.0069387      -5.9774814       0.0000000
     rt_Private_room      -0.5529958       0.0159599     -34.6490178       0.0000000
      rt_Shared_room      -1.2355206       0.0384618     -32.1232754       0.0000000
      pg_Condominium       0.1404588       0.0222251       6.3198282       0.0000000
            pg_House      -0.0133019       0.0146230      -0.9096565       0.3630396
            pg_Other       0.1411756       0.0227980       6.1924442       0.0000000
        pg_Townhouse      -0.0457839       0.0343557      -1.3326417       0.1826992
            d2balboa       0.0016453       0.0009673       1.7008587       0.0890205
------------------------------------------------------------------------------------

REGRESSION DIAGNOSTICS
MULTICOLLINEARITY CONDITION NUMBER           12.745

TEST ON NORMALITY OF ERRORS
TEST                             DF        VALUE           PROB
Jarque-Bera                       2        2710.322           0.0000

DIAGNOSTICS FOR HETEROSKEDASTICITY
RANDOM COEFFICIENTS
TEST                             DF        VALUE           PROB
Breusch-Pagan test               11         317.519           0.0000
Koenker-Bassett test             11         132.860           0.0000
================================ END OF REPORT =====================================

```

And, there still appears to be spatial structure in our model's errors:



{:.input_area}
```python
lag_residual = weights.spatial_lag.lag_spatial(knn, m2.u)
seaborn.regplot(m2.u.flatten(), lag_residual.flatten(), 
                line_kws=dict(color='orangered'),
                ci=None);
```



{:.output .output_png}
![png](../images/notebooks/11_regression_32_0.png)



Finally, the distance to Balboa Park variable does not fit our theory about how distance to amenity should affect the price of an Airbnb; the coefficient estimate is *positive*, meaning that people are paying a premium to be *further* from the Park. We will revisit this result later on, when we consider spatial heterogeneity and will be able to shed some light on this.

#### Spatial Fixed-Effects

While we've stipulated that our proximity variable might stand in for a difficult-to-measure premium individuals pay when they're close to a recreational zone. However, not all neighborhoods are created equal; some neighborhoods may be more lucrative than others, regardless of their proximity to Balboa Park. When this is the case, we need some way to account for the fact that each neighborhood may experience these kinds of *gestalt*, unique effects. One way to do this is through *spatial fixed effects*.

To illustrate these, let us consider the house price example from the previous section to introduce a more general illustration that relates to the second motivation for spatial effects ("space as a proxy"). Given we are only including two explanatory variables in the model, it is likely we are missing some important factors that play a role at determining the price at which a house is sold. Some of them, however, are likely to vary systematically over space (e.g. different neighborhood characteristics). If that is the case, we can control for those unobserved factors by using traditional dummy variables but basing their creation on a spatial rule. For example, let us include a binary variable for every neighborhood, indicating whether a given house is located within such area (`1`) or not (`0`). Mathematically, we are now fitting the following equation:

$$
\log{P_i} = \alpha_r + \sum_k \beta_k X_{ki} + \epsilon_i
$$

where the main difference is that we are now allowing the constant term, $\alpha$, to vary by neighbourhood $r$, $\alpha_r$.

Programmatically, we will show two different ways can estimate this: one,
using `statsmodels`; and two, with `PySAL`. First, we will use `statsmodels`. This package provides a formula-like API, which allows us to express the *equation* we wish to estimate directly:



{:.input_area}
```python
f = 'log_price ~ ' + ' + '.join(variable_names) + ' + neighborhood - 1'
print(f)
```


{:.output .output_stream}
```
log_price ~ accommodates + bathrooms + bedrooms + beds + rt_Private_room + rt_Shared_room + pg_Condominium + pg_House + pg_Other + pg_Townhouse + neighborhood - 1

```

The *tilde* operator in this statement is usually read as "log price is a function of ...", to account for the fact that many different model specifications can be fit according to that functional relationship between `log_price` and our covariate list. Critically, note that the trailing `-1` term means that we are fitting this model without an intercept term. This is necessary, since including an intercept term alongside unique means for every neighborhood would make the underlying system of equations underspecified.  

Using this expression, we can estimate the unique effects of each neighborhood, fitting the model in `statsmodels`: 



{:.input_area}
```python
m3 = sm.ols(f, data=db).fit()
print(m3.summary2())
```


{:.output .output_stream}
```
                           Results: Ordinary least squares
======================================================================================
Model:                      OLS                    Adj. R-squared:           0.709    
Dependent Variable:         log_price              AIC:                      7229.6640
Date:                       2018-07-23 21:57       BIC:                      7599.1365
No. Observations:           6110                   Log-Likelihood:           -3559.8  
Df Model:                   54                     F-statistic:              276.9    
Df Residuals:               6055                   Prob (F-statistic):       0.00     
R-squared:                  0.712                  Scale:                    0.18946  
--------------------------------------------------------------------------------------
                                       Coef.  Std.Err.    t     P>|t|   [0.025  0.975]
--------------------------------------------------------------------------------------
neighborhood[Balboa Park]              4.2808   0.0333 128.5836 0.0000  4.2155  4.3460
neighborhood[Bay Ho]                   4.1983   0.0769  54.6089 0.0000  4.0475  4.3490
neighborhood[Bay Park]                 4.3292   0.0510  84.9084 0.0000  4.2293  4.4292
neighborhood[Carmel Valley]            4.3893   0.0566  77.6126 0.0000  4.2784  4.5001
neighborhood[City Heights West]        4.0535   0.0584  69.4358 0.0000  3.9391  4.1680
neighborhood[Clairemont Mesa]          4.0953   0.0477  85.8559 0.0000  4.0018  4.1888
neighborhood[College Area]             4.0337   0.0583  69.2386 0.0000  3.9195  4.1479
neighborhood[Core]                     4.7262   0.0526  89.7775 0.0000  4.6230  4.8294
neighborhood[Cortez Hill]              4.6081   0.0515  89.4322 0.0000  4.5071  4.7091
neighborhood[Del Mar Heights]          4.4969   0.0543  82.7599 0.0000  4.3904  4.6034
neighborhood[East Village]             4.5455   0.0294 154.7473 0.0000  4.4879  4.6031
neighborhood[Gaslamp Quarter]          4.7758   0.0473 100.9589 0.0000  4.6831  4.8685
neighborhood[Grant Hill]               4.3067   0.0524  82.2442 0.0000  4.2041  4.4094
neighborhood[Grantville]               4.0533   0.0714  56.7719 0.0000  3.9133  4.1933
neighborhood[Kensington]               4.3027   0.0772  55.7511 0.0000  4.1514  4.4540
neighborhood[La Jolla]                 4.6821   0.0258 181.4137 0.0000  4.6315  4.7327
neighborhood[La Jolla Village]         4.3303   0.0772  56.0653 0.0000  4.1789  4.4817
neighborhood[Linda Vista]              4.1911   0.0569  73.6380 0.0000  4.0796  4.3027
neighborhood[Little Italy]             4.6667   0.0468  99.6364 0.0000  4.5749  4.7586
neighborhood[Loma Portal]              4.3019   0.0332 129.4346 0.0000  4.2368  4.3671
neighborhood[Marina]                   4.5583   0.0480  94.9761 0.0000  4.4642  4.6524
neighborhood[Midtown]                  4.3667   0.0284 153.7902 0.0000  4.3110  4.4223
neighborhood[Midtown District]         4.5849   0.0651  70.4436 0.0000  4.4573  4.7125
neighborhood[Mira Mesa]                3.9896   0.0561  71.1135 0.0000  3.8796  4.0995
neighborhood[Mission Bay]              4.5155   0.0224 201.3850 0.0000  4.4715  4.5594
neighborhood[Mission Valley]           4.2760   0.0742  57.6031 0.0000  4.1304  4.4215
neighborhood[Moreno Mission]           4.4009   0.0567  77.5773 0.0000  4.2897  4.5122
neighborhood[Normal Heights]           4.0974   0.0490  83.5821 0.0000  4.0013  4.1935
neighborhood[North Clairemont]         3.9844   0.0691  57.6209 0.0000  3.8489  4.1200
neighborhood[North Hills]              4.2534   0.0255 166.9470 0.0000  4.2035  4.3034
neighborhood[Northwest]                4.1738   0.0697  59.8572 0.0000  4.0371  4.3104
neighborhood[Ocean Beach]              4.4372   0.0301 147.4709 0.0000  4.3782  4.4961
neighborhood[Old Town]                 4.4202   0.0419 105.5098 0.0000  4.3380  4.5023
neighborhood[Otay Ranch]               4.1859   0.0816  51.2999 0.0000  4.0260  4.3459
neighborhood[Pacific Beach]            4.4388   0.0224 198.0136 0.0000  4.3949  4.4828
neighborhood[Park West]                4.4409   0.0448  99.1988 0.0000  4.3531  4.5287
neighborhood[Rancho Bernadino]         4.1809   0.0720  58.0598 0.0000  4.0397  4.3221
neighborhood[Rancho Penasquitos]       4.1624   0.0618  67.3789 0.0000  4.0413  4.2835
neighborhood[Roseville]                4.3870   0.0586  74.8346 0.0000  4.2721  4.5019
neighborhood[San Carlos]               4.3350   0.0830  52.2035 0.0000  4.1722  4.4978
neighborhood[Scripps Ranch]            4.0824   0.0762  53.5440 0.0000  3.9329  4.2318
neighborhood[Serra Mesa]               4.3130   0.0599  71.9725 0.0000  4.1955  4.4304
neighborhood[South Park]               4.2253   0.0536  78.7676 0.0000  4.1202  4.3305
neighborhood[University City]          4.1937   0.0370 113.4516 0.0000  4.1213  4.2662
neighborhood[West University Heights]  4.2977   0.0431  99.6359 0.0000  4.2131  4.3822
accommodates                           0.0728   0.0048  15.0672 0.0000  0.0633  0.0822
bathrooms                              0.1702   0.0105  16.2171 0.0000  0.1496  0.1908
bedrooms                               0.1686   0.0106  15.8731 0.0000  0.1478  0.1894
beds                                  -0.0416   0.0065  -6.3508 0.0000 -0.0544 -0.0287
rt_Private_room                       -0.4873   0.0154 -31.6225 0.0000 -0.5175 -0.4570
rt_Shared_room                        -1.2396   0.0368 -33.6657 0.0000 -1.3118 -1.1674
pg_Condominium                         0.1329   0.0210   6.3333 0.0000  0.0918  0.1741
pg_House                               0.0400   0.0144   2.7868 0.0053  0.0119  0.0681
pg_Other                               0.0610   0.0224   2.7290 0.0064  0.0172  0.1048
pg_Townhouse                          -0.0075   0.0324  -0.2323 0.8163 -0.0710  0.0560
--------------------------------------------------------------------------------------
Omnibus:                   1215.551             Durbin-Watson:                1.835   
Prob(Omnibus):             0.000                Jarque-Bera (JB):             4115.510
Skew:                      0.989                Prob(JB):                     0.000   
Kurtosis:                  6.500                Condition No.:                132     
======================================================================================


```

The approach above shows how spatial FE are a particular case of a linear regression with a categorical  variable. Neighborhood membership is modeled using binary dummy variables. Thanks to the formula grammar used in `statsmodels`, we can express the model abstractly, and Python parses it, appropriately creating binary variables as required.

The second approach leverages `PySAL` Regimes functionality, which allows the user to specify which variables are to be estimated separately for each "regime". In this case however, instead of describing the model in a formula, we need to pass each element of the model as separate arguments.



{:.input_area}
```python
# PySAL implementation
m4 = spreg.OLS_Regimes(db[['log_price']].values, db[variable_names].values,
                       db['neighborhood'].tolist(),
                       constant_regi='many', cols2regi=[False]*len(variable_names),
                       regime_err_sep=False,
                       name_y='log_price', name_x=variable_names)
print(m4.summary)
```


{:.output .output_stream}
```
REGRESSION
----------
SUMMARY OF OUTPUT: ORDINARY LEAST SQUARES - REGIMES
---------------------------------------------------
Data set            :     unknown
Weights matrix      :        None
Dependent Variable  :   log_price                Number of Observations:        6110
Mean dependent var  :      4.9958                Number of Variables   :          55
S.D. dependent var  :      0.8072                Degrees of Freedom    :        6055
R-squared           :      0.7118
Adjusted R-squared  :      0.7092
Sum squared residual:    1147.169                F-statistic           :    276.9408
Sigma-square        :       0.189                Prob(F-statistic)     :           0
S.E. of regression  :       0.435                Log likelihood        :   -3559.832
Sigma-square ML     :       0.188                Akaike info criterion :    7229.664
S.E of regression ML:      0.4333                Schwarz criterion     :    7599.137

------------------------------------------------------------------------------------
            Variable     Coefficient       Std.Error     t-Statistic     Probability
------------------------------------------------------------------------------------
Balboa Park_CONSTANT       4.2807664       0.0332917     128.5835585       0.0000000
     Bay Ho_CONSTANT       4.1982505       0.0768784      54.6089479       0.0000000
   Bay Park_CONSTANT       4.3292234       0.0509870      84.9083655       0.0000000
Carmel Valley_CONSTANT       4.3892614       0.0565535      77.6125622       0.0000000
City Heights West_CONSTANT       4.0535183       0.0583780      69.4357707       0.0000000
Clairemont Mesa_CONSTANT       4.0952589       0.0476992      85.8558747       0.0000000
College Area_CONSTANT       4.0336972       0.0582579      69.2386376       0.0000000
       Core_CONSTANT       4.7261863       0.0526433      89.7775229       0.0000000
Cortez Hill_CONSTANT       4.6080896       0.0515261      89.4322167       0.0000000
Del Mar Heights_CONSTANT       4.4969102       0.0543368      82.7599068       0.0000000
East Village_CONSTANT       4.5454690       0.0293735     154.7473234       0.0000000
Gaslamp Quarter_CONSTANT       4.7757987       0.0473044     100.9588995       0.0000000
 Grant Hill_CONSTANT       4.3067425       0.0523653      82.2441742       0.0000000
 Grantville_CONSTANT       4.0532975       0.0713962      56.7718990       0.0000000
 Kensington_CONSTANT       4.3026710       0.0771765      55.7510746       0.0000000
   La Jolla_CONSTANT       4.6820840       0.0258089     181.4136961       0.0000000
La Jolla Village_CONSTANT       4.3303114       0.0772369      56.0652857       0.0000000
Linda Vista_CONSTANT       4.1911487       0.0569155      73.6380443       0.0000000
Little Italy_CONSTANT       4.6667423       0.0468377      99.6363950       0.0000000
Loma Portal_CONSTANT       4.3019094       0.0332362     129.4346151       0.0000000
     Marina_CONSTANT       4.5582979       0.0479941      94.9761422       0.0000000
    Midtown_CONSTANT       4.3666608       0.0283936     153.7902257       0.0000000
Midtown District_CONSTANT       4.5849382       0.0650866      70.4436292       0.0000000
  Mira Mesa_CONSTANT       3.9895616       0.0561013      71.1135365       0.0000000
Mission Bay_CONSTANT       4.5154791       0.0224221     201.3849675       0.0000000
Mission Valley_CONSTANT       4.2759604       0.0742315      57.6030636       0.0000000
Moreno Mission_CONSTANT       4.4009417       0.0567298      77.5773078       0.0000000
Normal Heights_CONSTANT       4.0973996       0.0490225      83.5820603       0.0000000
North Clairemont_CONSTANT       3.9844398       0.0691492      57.6208858       0.0000000
North Hills_CONSTANT       4.2534252       0.0254777     166.9470009       0.0000000
  Northwest_CONSTANT       4.1737520       0.0697284      59.8572467       0.0000000
Ocean Beach_CONSTANT       4.4371642       0.0300884     147.4709376       0.0000000
   Old Town_CONSTANT       4.4201603       0.0418934     105.5097966       0.0000000
 Otay Ranch_CONSTANT       4.1859412       0.0815974      51.2999205       0.0000000
Pacific Beach_CONSTANT       4.4388288       0.0224168     198.0136040       0.0000000
  Park West_CONSTANT       4.4409072       0.0447677      99.1988153       0.0000000
Rancho Bernadino_CONSTANT       4.1809062       0.0720103      58.0598088       0.0000000
Rancho Penasquitos_CONSTANT       4.1624276       0.0617764      67.3788989       0.0000000
  Roseville_CONSTANT       4.3869921       0.0586225      74.8346070       0.0000000
 San Carlos_CONSTANT       4.3349911       0.0830403      52.2034885       0.0000000
Scripps Ranch_CONSTANT       4.0823805       0.0762435      53.5439686       0.0000000
 Serra Mesa_CONSTANT       4.3129674       0.0599252      71.9725317       0.0000000
 South Park_CONSTANT       4.2253108       0.0536428      78.7675791       0.0000000
University City_CONSTANT       4.1937181       0.0369648     113.4516038       0.0000000
West University Heights_CONSTANT       4.2976715       0.0431338      99.6358857       0.0000000
_Global_accommodates       0.0727766       0.0048301      15.0671860       0.0000000
   _Global_bathrooms       0.1702080       0.0104956      16.2171367       0.0000000
    _Global_bedrooms       0.1685720       0.0106200      15.8731267       0.0000000
        _Global_beds      -0.0415809       0.0065474      -6.3507569       0.0000000
_Global_rt_Private_room      -0.4872544       0.0154085     -31.6225002       0.0000000
_Global_rt_Shared_room      -1.2395926       0.0368206     -33.6656955       0.0000000
_Global_pg_Condominium       0.1329341       0.0209896       6.3333214       0.0000000
    _Global_pg_House       0.0399982       0.0143528       2.7867915       0.0053399
    _Global_pg_Other       0.0610112       0.0223565       2.7290143       0.0063707
_Global_pg_Townhouse      -0.0075250       0.0323876      -0.2323436       0.8162790
------------------------------------------------------------------------------------
Regimes variable: unknown

REGRESSION DIAGNOSTICS
MULTICOLLINEARITY CONDITION NUMBER           12.143

TEST ON NORMALITY OF ERRORS
TEST                             DF        VALUE           PROB
Jarque-Bera                       2        4115.510           0.0000

DIAGNOSTICS FOR HETEROSKEDASTICITY
RANDOM COEFFICIENTS
TEST                             DF        VALUE           PROB
Breusch-Pagan test               54         854.587           0.0000
Koenker-Bassett test             54         310.744           0.0000

REGIMES DIAGNOSTICS - CHOW TEST
                 VARIABLE        DF        VALUE           PROB
                 CONSTANT        44         913.016           0.0000
              Global test        44         913.016           0.0000
================================ END OF REPORT =====================================

```

Econometrically speaking, what the neighborhood FEs we have introduced imply is that, instead of comparing all house prices across San Diego as equal, we only derive variation from within each postcode. Remember that the interpretation of $\beta_k$ is the effect of variable $k$, *given all the other explanatory variables included remain constant*. By including a single variable for each area, we are effectively forcing the model to compare as equal only house prices that share the same value for each variable; or, in other words, only houses located within the same area. Introducing FE affords a higher degree of isolation of the effects of the variables we introduce in the model because we can control for unobserved effects that align spatially with the distribution of the FE introduced (by postcode, in our case).

To make a map of neighborhood fixed effects, we need to process the results from our model slightly.

First, we extract only the effects pertaining to the neighborhoods:



{:.input_area}
```python
neighborhood_effects = m3.params.filter(like='neighborhood')
neighborhood_effects.head()
```





{:.output .output_data_text}
```
neighborhood[Balboa Park]          4.280766
neighborhood[Bay Ho]               4.198251
neighborhood[Bay Park]             4.329223
neighborhood[Carmel Valley]        4.389261
neighborhood[City Heights West]    4.053518
dtype: float64
```



Then, we need to extract just the neighborhood name from the index of this Series. A simple way to do this is to strip all the characters that come before and after our neighborhood names:



{:.input_area}
```python
stripped = neighborhood_effects.index.str.strip('neighborhood[').str.strip(']')
neighborhood_effects.index = stripped
neighborhood_effects = neighborhood_effects.to_frame('fixed_effect')
neighborhood_effects.head()
```





<div markdown="0" class="output output_html">
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>fixed_effect</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Balboa Park</th>
      <td>4.280766</td>
    </tr>
    <tr>
      <th>Bay Ho</th>
      <td>4.198251</td>
    </tr>
    <tr>
      <th>Bay Park</th>
      <td>4.329223</td>
    </tr>
    <tr>
      <th>Carmel Valley</th>
      <td>4.389261</td>
    </tr>
    <tr>
      <th>City Heights West</th>
      <td>4.053518</td>
    </tr>
  </tbody>
</table>
</div>
</div>



Good, we're back to our raw neighborhood names. Now, we can join them back up with the neighborhood shapes:



{:.input_area}
```python
neighborhoods = geopandas.read_file(bookdata.san_diego_neighborhoods())
```




{:.input_area}
```python
ax = neighborhoods.merge(neighborhood_effects, how='left',
                    left_on='neighbourhood', 
                    right_index=True).plot('fixed_effect',
                                           figsize=(12,6))
ax.set_title("San Diego Neighborhood Fixed Effects")
plt.show()
```



{:.output .output_png}
![png](../images/notebooks/11_regression_46_0.png)



#### Spatial Regimes

At the core of estimating spatial FEs is the idea that, instead of assuming the dependent variable behaves uniformly over space, there are systematic effects following a geographical pattern that affect its behaviour. In other words, spatial FEs introduce econometrically the notion of spatial heterogeneity. They do this in the simplest possible form: by allowing the constant term to vary geographically. The other elements of the regression are left untouched and hence apply uniformly across space. The idea of spatial regimes (SRs) is to generalize the spatial FE approach to allow not only the constant term to vary but also any other explanatory variable. This implies that the equation we will be estimating is:

$$
\log{P_i} = \alpha_r + \sum_k \beta_{k-r} X_{ki} + \epsilon_i
$$

where we are not only allowing the constant term to vary by region ($\alpha_r$), but also every other parameter ($\beta_{k-r}$).

To illustrate this approach, we will use the "spatial differentiator" of whether a house is in a coastal neighbourhood or not (`coastal_neig`) to define the regimes. The rationale behind this choice is that renting a house close to the ocean might be a strong enough pull that people might be willing to pay at different *rates* for each of the house's characteristics.

To implement this in Python, we use the `OLS_Regimes` class in `PySAL`, which does most of the heavy lifting for us:



{:.input_area}
```python
m4 = spreg.OLS_Regimes(db[['log_price']].values, db[variable_names].values,
                          db['coastal'].tolist(),
                          constant_regi='many',
                          regime_err_sep=False,
                          name_y='log_price', name_x=variable_names)
print(m4.summary)
```


{:.output .output_stream}
```
REGRESSION
----------
SUMMARY OF OUTPUT: ORDINARY LEAST SQUARES - REGIMES
---------------------------------------------------
Data set            :     unknown
Weights matrix      :        None
Dependent Variable  :   log_price                Number of Observations:        6110
Mean dependent var  :      4.9958                Number of Variables   :          22
S.D. dependent var  :      0.8072                Degrees of Freedom    :        6088
R-squared           :      0.6853
Adjusted R-squared  :      0.6843
Sum squared residual:    1252.489                F-statistic           :    631.4283
Sigma-square        :       0.206                Prob(F-statistic)     :           0
S.E. of regression  :       0.454                Log likelihood        :   -3828.169
Sigma-square ML     :       0.205                Akaike info criterion :    7700.339
S.E of regression ML:      0.4528                Schwarz criterion     :    7848.128

------------------------------------------------------------------------------------
            Variable     Coefficient       Std.Error     t-Statistic     Probability
------------------------------------------------------------------------------------
          0_CONSTANT       4.4072424       0.0215156     204.8392695       0.0000000
      0_accommodates       0.0901860       0.0064737      13.9311338       0.0000000
         0_bathrooms       0.1433760       0.0142680      10.0487871       0.0000000
          0_bedrooms       0.1129626       0.0138273       8.1695568       0.0000000
              0_beds      -0.0262719       0.0088380      -2.9726102       0.0029644
   0_rt_Private_room      -0.5293343       0.0189179     -27.9805699       0.0000000
    0_rt_Shared_room      -1.2244586       0.0425969     -28.7452834       0.0000000
    0_pg_Condominium       0.1053065       0.0281309       3.7434523       0.0001832
          0_pg_House      -0.0454471       0.0179571      -2.5308637       0.0114032
          0_pg_Other       0.0607526       0.0276365       2.1982715       0.0279673
      0_pg_Townhouse      -0.0103973       0.0456730      -0.2276456       0.8199294
          1_CONSTANT       4.4799043       0.0250938     178.5260014       0.0000000
      1_accommodates       0.0484639       0.0078806       6.1497397       0.0000000
         1_bathrooms       0.2474779       0.0165661      14.9388057       0.0000000
          1_bedrooms       0.1897404       0.0179229      10.5864676       0.0000000
              1_beds      -0.0506077       0.0107429      -4.7107925       0.0000025
   1_rt_Private_room      -0.5586281       0.0283122     -19.7309699       0.0000000
    1_rt_Shared_room      -1.0528541       0.0841745     -12.5079997       0.0000000
    1_pg_Condominium       0.2044470       0.0339434       6.0231780       0.0000000
          1_pg_House       0.0753534       0.0233783       3.2232188       0.0012743
          1_pg_Other       0.2954848       0.0386455       7.6460385       0.0000000
      1_pg_Townhouse      -0.0735077       0.0493672      -1.4889984       0.1365396
------------------------------------------------------------------------------------
Regimes variable: unknown

REGRESSION DIAGNOSTICS
MULTICOLLINEARITY CONDITION NUMBER           14.033

TEST ON NORMALITY OF ERRORS
TEST                             DF        VALUE           PROB
Jarque-Bera                       2        3977.425           0.0000

DIAGNOSTICS FOR HETEROSKEDASTICITY
RANDOM COEFFICIENTS
TEST                             DF        VALUE           PROB
Breusch-Pagan test               21         443.593           0.0000
Koenker-Bassett test             21         164.276           0.0000

REGIMES DIAGNOSTICS - CHOW TEST
                 VARIABLE        DF        VALUE           PROB
                 CONSTANT         1           4.832           0.0279
             accommodates         1          16.736           0.0000
                bathrooms         1          22.671           0.0000
                 bedrooms         1          11.504           0.0007
                     beds         1           3.060           0.0802
           pg_Condominium         1           5.057           0.0245
                 pg_House         1          16.793           0.0000
                 pg_Other         1          24.410           0.0000
             pg_Townhouse         1           0.881           0.3480
          rt_Private_room         1           0.740           0.3896
           rt_Shared_room         1           3.309           0.0689
              Global test        11         328.869           0.0000
================================ END OF REPORT =====================================

```

### Spatial Dependence

As we have just discussed, SH is about effects of phenomena that are *explicitly linked*
to geography and that hence cause spatial variation and clustering. This
encompasses many of the kinds of spatial effects we may be interested in when we fit
linear regressions. However, in other cases, our focus is on the effect of the *spatial
configuration* of the observations, and the extent to which that has an effect on the
outcome we are considering. For example, we might think that the price of a house not
only depends on whether it is a townhouse or an appartment, but also on
whether it is surrounded by many more townhouses than skyscrapers with more
appartments. This, we could hypothesise, might be related to the different "look and feel" a
neighbourhood with low-height, historic buildings has as compared to one with
modern highrises. To the extent these two different spatial configurations
enter differently the house price determination process, we will be
interested in capturing not only the characteristics of a house, but also of
its surrounding ones.
This kind of spatial effect is fundamentally different
from SH in that is it not related to inherent characteristics of the geography but relates 
to the characteristics of the observations in our dataset and, specially, to their spatial
arrangement. We call this phenomenon by which the values of observations are related to
each other through distance *spatial dependence* ([Anselin, 1988](https://books.google.co.uk/books/about/Spatial_Econometrics_Methods_and_Models.html?id=3dPIXClv4YYC&redir_esc=y])).

There are several ways to introduce spatial dependence in an econometric framework, with varying degrees of econometric sophistication (see [Anselin, 2002](http://onlinelibrary.wiley.com/doi/10.1111/j.1574-0862.2002.tb00120.x/abstract) for a good overview). Common to all of them however is the way space is formally encapsulated: through *spatial weights matrices ($W$)*.

#### Exogenous effects

Let us come back to the house price example we have been working with. So far, we
have hypothesized that the price of a house rented in San Diego through AirBnb can
be explained using information about its own characteristics as well as some 
relating to its location such as the neighborhood or the distance to the main
park in the city. However, it is also reasonable to think that prospective renters
care about the larger area around a house, not only about the house itself, and would
be willing to pay more for a house that was surrounded by certain types of houses, 
and less if it was located in the middle of other types. How could we test this idea?

The most straightforward way to introduce spatial dependence in a regression is by 
considering not only a given explanatory variable, but also its spatial lag. In our
example case, in addition to including a dummy for the type of house (`pg_XXX`), we 
can also include the spatial lag of each type of house. This addition implies
we are also including as explanatory factor of the price of a given house the proportion 
neighbouring houses in each type. Mathematically, this implies estimating the following model:

$$
\log(P) = \alpha + X\beta + WX\beta + \epsilon
$$

where $WX\beta$ represents the spatial lag of (some of) the explanatory variables. In Python, 
we can calculate the spatial lag of each variable whose name starts by `pg_`
by first creating a list of all of those names, and then applying `PySAL`'s
`lag_spatial` to each of them:



{:.input_area}
```python
wx = [i for i in variable_names if 'pg_' in i]
wx = db[wx].apply(lambda y: weights.spatial_lag.lag_spatial(knn, y))\
           .rename(columns=lambda c: 'w_'+c)
```


Once computed, we can run the model using OLS estimation because, in this
context, the spatial  lags included do not violate any of the assumptions OLS
relies on (they are essentially additional exogenous variables):



{:.input_area}
```python
m5 = spreg.OLS(db[['log_price']].values, 
                  db[variable_names].join(wx).values,
                  name_y='l_price', name_x=variable_names+wx.columns.tolist())
print(m5.summary)
```


{:.output .output_stream}
```
REGRESSION
----------
SUMMARY OF OUTPUT: ORDINARY LEAST SQUARES
-----------------------------------------
Data set            :     unknown
Weights matrix      :        None
Dependent Variable  :     l_price                Number of Observations:        6110
Mean dependent var  :      4.9958                Number of Variables   :          15
S.D. dependent var  :      0.8072                Degrees of Freedom    :        6095
R-squared           :      0.6800
Adjusted R-squared  :      0.6792
Sum squared residual:    1273.933                F-statistic           :    924.9423
Sigma-square        :       0.209                Prob(F-statistic)     :           0
S.E. of regression  :       0.457                Log likelihood        :   -3880.030
Sigma-square ML     :       0.208                Akaike info criterion :    7790.061
S.E of regression ML:      0.4566                Schwarz criterion     :    7890.826

------------------------------------------------------------------------------------
            Variable     Coefficient       Std.Error     t-Statistic     Probability
------------------------------------------------------------------------------------
            CONSTANT       4.3205814       0.0234977     183.8727044       0.0000000
        accommodates       0.0809972       0.0050046      16.1843874       0.0000000
           bathrooms       0.1893447       0.0108059      17.5224026       0.0000000
            bedrooms       0.1635998       0.0109764      14.9047058       0.0000000
                beds      -0.0451529       0.0068249      -6.6159365       0.0000000
     rt_Private_room      -0.5293783       0.0157308     -33.6524367       0.0000000
      rt_Shared_room      -1.2892590       0.0381443     -33.7995105       0.0000000
      pg_Condominium       0.1063490       0.0221782       4.7952003       0.0000017
            pg_House       0.0327806       0.0156954       2.0885538       0.0367893
            pg_Other       0.0861857       0.0239774       3.5944620       0.0003276
        pg_Townhouse      -0.0277116       0.0338485      -0.8186965       0.4129916
    w_pg_Condominium       0.5928369       0.0689612       8.5966706       0.0000000
          w_pg_House      -0.0774462       0.0318830      -2.4290766       0.0151661
          w_pg_Other       0.4851047       0.0551461       8.7967121       0.0000000
      w_pg_Townhouse      -0.2724493       0.1223388      -2.2270058       0.0259833
------------------------------------------------------------------------------------

REGRESSION DIAGNOSTICS
MULTICOLLINEARITY CONDITION NUMBER           14.277

TEST ON NORMALITY OF ERRORS
TEST                             DF        VALUE           PROB
Jarque-Bera                       2        2458.006           0.0000

DIAGNOSTICS FOR HETEROSKEDASTICITY
RANDOM COEFFICIENTS
TEST                             DF        VALUE           PROB
Breusch-Pagan test               14         393.052           0.0000
Koenker-Bassett test             14         169.585           0.0000
================================ END OF REPORT =====================================

```

The way to interpret the table of results is similar to that of any other
non-spatial regression. The variables we included in the original regression
display similar behaviour, albeit with small changes in size, and can be
interpreted also in a similar way. The spatial lag of each type of property
(`w_pg_XXX`) is the new addition. We observe that, except for the case
of townhouses (same as with the binary variable, `pg_Townhouse`), they are all
significant, suggesting our initial hypothesis on the role of the surrounding
houses might indeed be at work here. As an illustration, being a condominium increases
the price, on average, 11% ($\beta_{pg\_Condominium}=0.11$) with respect to the benchmark, which is set to apartments. More relevant to this section, any given house surrounded by condominiums *also* receives a price premium. In this case, the interpretation is slightly different because the variable is not a dummy but a proportion in the range [0, 1]. A 10% increase the proportion of neighbors that are condominiums translates into a 6% increase in the property house price ($\beta_{w_pg\_Condominium = 0.6$). This interpretation comes from the following: increasing the prevalence of 
condos in the area surrounding a house by ten percent is associated with a change in the
log of the nightly rental price of about .06, which translates to around a 6% increase in the nightly
rental price of the house. Similar interpretations can be derived for all other spatially lagged variables.

Introducing a spatial lag of an explanatory variable, as we have just seen, is the most straightforward way of incorporating the notion of spatial dependence in a linear regression framework. It does not require additional changes, it can be estimated with OLS, and the interpretation is rather similar to interpreting non-spatial variables. The field of spatial econometrics however is a much broader one and has produced over the last decades many techniques to deal with spatial effects and spatial dependence in different ways. Although this might be an over simplification, one can say that most of such efforts for the case of a single cross-section are focused on two main variations: the spatial lag and the spatial error model. Both are similar to the case we have seen in that they are based on the introduction of a spatial lag, but they differ in the component of the model they modify and affect.

#### Spatial Error

The spatial error model includes a spatial lag in the *error* term of the equation:

$$
\log{P_i} = \alpha + \sum_k \beta_k X_{ki} + u_i
$$

$$
u_i = \lambda u_{lag-i} + \epsilon_i
$$

where $u_{lag-i} = \sum_j w_{i,j} u_j$. 
Although it appears similar, this specification violates the assumptions about the
error term in a classical OLS model. Hence, alternative estimation methods are
required. `PySAL` incorporates functionality to estimate several of the most
advanced techniques developed by the literature on spatial econometrics. For
example, we can use a general method of moments that account for 
heterogeneity (Arraiz et al., 2010):



{:.input_area}
```python
m6 = spreg.GM_Error_Het(db[['log_price']].values, db[variable_names].values,
                           w=knn, name_y='log_price', name_x=variable_names)
print(m6.summary)
```


{:.output .output_stream}
```
REGRESSION
----------
SUMMARY OF OUTPUT: SPATIALLY WEIGHTED LEAST SQUARES (HET)
---------------------------------------------------------
Data set            :     unknown
Weights matrix      :     unknown
Dependent Variable  :   log_price                Number of Observations:        6110
Mean dependent var  :      4.9958                Number of Variables   :          11
S.D. dependent var  :      0.8072                Degrees of Freedom    :        6099
Pseudo R-squared    :      0.6655
N. of iterations    :           1                Step1c computed       :          No

------------------------------------------------------------------------------------
            Variable     Coefficient       Std.Error     z-Statistic     Probability
------------------------------------------------------------------------------------
            CONSTANT       4.4262033       0.0217088     203.8898742       0.0000000
        accommodates       0.0695536       0.0063268      10.9934495       0.0000000
           bathrooms       0.1614101       0.0151312      10.6673765       0.0000000
            bedrooms       0.1739251       0.0146697      11.8560847       0.0000000
                beds      -0.0377710       0.0088293      -4.2779023       0.0000189
     rt_Private_room      -0.4947947       0.0163843     -30.1993140       0.0000000
      rt_Shared_room      -1.1613985       0.0515304     -22.5381175       0.0000000
      pg_Condominium       0.1003761       0.0213148       4.7092198       0.0000025
            pg_House       0.0308334       0.0147100       2.0960849       0.0360747
            pg_Other       0.0861768       0.0254942       3.3802547       0.0007242
        pg_Townhouse      -0.0074515       0.0292873      -0.2544285       0.7991646
              lambda       0.6448728       0.0186626      34.5543740       0.0000000
------------------------------------------------------------------------------------
================================ END OF REPORT =====================================

```

#### Spatial Lag

The spatial lag model introduces a spatial lag of the *dependent* variable. In the example we have covered, this would translate into:

$$
\log{P_i} = \alpha + \rho \log{P_{lag-i}} + \sum_k \beta_k X_{ki} + \epsilon_i
$$

Although it might not seem very different from the previous equation, this model violates 
the exogeneity assumption, crucial for OLS to work. 
Put simply, this occurs when $P_i$ exists on both "sides" of the equals sign.
In theory, since $P_i$ is included in computing $P_{lag-i}$, exogeneity is violated. 
Similarly to the case of
the spatial error, several techniques have been proposed to overcome this
limitation, and `PySAL` implements several of them. In the example below, we
use a two-stage least squares estimation (Anselin, 1988), where the spatial
lag of all the explanatory variables is used as instrument for the endogenous
lag:



{:.input_area}
```python
m7 = spreg.GM_Lag(db[['log_price']].values, db[variable_names].values,
                     w=knn, name_y='log_price', name_x=variable_names)
print(m7.summary)
```


{:.output .output_stream}
```
REGRESSION
----------
SUMMARY OF OUTPUT: SPATIAL TWO STAGE LEAST SQUARES
--------------------------------------------------
Data set            :     unknown
Weights matrix      :     unknown
Dependent Variable  :   log_price                Number of Observations:        6110
Mean dependent var  :      4.9958                Number of Variables   :          12
S.D. dependent var  :      0.8072                Degrees of Freedom    :        6098
Pseudo R-squared    :      0.7057
Spatial Pseudo R-squared:  0.6883

------------------------------------------------------------------------------------
            Variable     Coefficient       Std.Error     z-Statistic     Probability
------------------------------------------------------------------------------------
            CONSTANT       2.7440254       0.0727290      37.7294400       0.0000000
        accommodates       0.0697596       0.0048157      14.4859187       0.0000000
           bathrooms       0.1626725       0.0104007      15.6405467       0.0000000
            bedrooms       0.1604137       0.0104823      15.3033012       0.0000000
                beds      -0.0365065       0.0065336      -5.5874750       0.0000000
     rt_Private_room      -0.4981415       0.0151396     -32.9031780       0.0000000
      rt_Shared_room      -1.1157392       0.0365563     -30.5210777       0.0000000
      pg_Condominium       0.1072995       0.0209048       5.1327614       0.0000003
            pg_House      -0.0004017       0.0136828      -0.0293598       0.9765777
            pg_Other       0.1207503       0.0214771       5.6222929       0.0000000
        pg_Townhouse      -0.0185543       0.0322730      -0.5749190       0.5653461
         W_log_price       0.3416482       0.0147787      23.1175620       0.0000000
------------------------------------------------------------------------------------
Instrumented: W_log_price
Instruments: W_accommodates, W_bathrooms, W_bedrooms, W_beds,
             W_pg_Condominium, W_pg_House, W_pg_Other, W_pg_Townhouse,
             W_rt_Private_room, W_rt_Shared_room
================================ END OF REPORT =====================================

```

#### Other ways of bringing space into regression

While these are some kinds of spatial regressions, many other advanced spatial regression methods see routine use in statistics, data science, and applied analysis. For example, Generalized Additive Models [4,5] haven been used to apply spatial kernel smoothing directly within a regression function. Other similar smoothing methods, such as spatial Gaussian process models [6] or Kriging, conceptualize the dependence between locations as smooth as well. Other methods in spatial regression that consider graph-based geographies (rather than distance/kernel effects) include variations on conditional autoregressive model, which examines spatial relationships at locations *conditional* on their surroundings, rather than as jointly co-emergent with them. Full coverage of these topics is beyond the scope of this book, however, though [7] provides a detailed and comprehensive discussion. 

### Challenge

#### The random coast
In the section analyzing our naive model residuals, we ran a classic two-sample $t$-test to identify whether or not our coastal and not-coastal residential districts tended to have the same prediction errors. Often, though, it's better to use straightforward, data-driven testing and simulation methods than assuming that the mathematical assumptions of the $t$-statistic are met.

To do this, we can shuffle our assignments to coast and not-coast, and check whether or not there are differences in the distributions of the *observed* residual distributions and random distributions. In this way, we shuffle the observations that are on the coast, and plot the resulting cumulative distributions.

Below, we do this; running 100 simulated re-assignments of districts to either "coast" or "not coast," and comparing the distributions of randomly-assigned residuals to the observed distributions of residuals. Further, we do this plotting by the *empirical cumulative density function*, not the histogram directly. This is because the *empirical cumulative density function* is usually easier to examine visually, especially for subtle differences. 

Below, the black lines represent our simulations, and the colored patches below them represents the observed distribution of residuals. If the black lines tend to be on the left of the colored patch, then, the simulations (where prediction error is totally random with respect to our categories of "coastal" and "not coastal") tend to have more negative residuals than our actual model. If the black lines tend to be on the right, then they tend to have more positive residuals. As a refresher, positive residuals mean that our model is underpredicting, and negative residuals mean that our model is overpredicting. Below, our simulations provide direct evidence for the claim that our model may be systematically underpredicting coastal price and overpredicting non-coastal prices. 



{:.input_area}
```python
n_simulations = 100
f, ax = plt.subplots(1,2,figsize=(12,3), sharex=True, sharey=True)
ax[0].hist(coastal, color='r', alpha=.5, 
           density=True, bins=30, label='Coastal', 
           cumulative=True)
ax[1].hist(not_coastal, color='b', alpha=.5,
           density=True, bins=30, label='Not Coastal', 
           cumulative=True)
for simulation in range(n_simulations):
    shuffled_residuals = m1.u[numpy.random.permutation(m1.n)]
    random_coast, random_notcoast = (shuffled_residuals[is_coastal], 
                                     shuffled_residuals[~is_coastal])
    if simulation == 0:
        label = 'Simulations'
    else:
        label = None
    ax[0].hist(random_coast, 
                density=True, 
                histtype='step',
                color='k', alpha=.05, bins=30, 
                label=label, 
                cumulative=True)
    ax[1].hist(random_coast, 
                density=True, 
                histtype='step',
                color='k', alpha=.05, bins=30, 
                label=label, 
                cumulative=True)
ax[0].legend()
ax[1].legend()
plt.tight_layout()
plt.show()
```



{:.output .output_png}
![png](../images/notebooks/11_regression_60_0.png)



#### The K-neighbor correlogram

Further, it might be the case that spatial dependence in our mis-predictions only matters for sites that are extremely close to one another, and decays quickly with distance. 
To investigate this, we can examine the correlation between each sites' residual and the *average* of the $k$th nearest neighbors' residuals, increasing $k$ until the estimate stabilizes. 
This main idea is central to the geostatistical concept, the *correlogram*, which gives the correlation between sites of an attribute being studied as distance increases.

One quick way to check whether or not what we've seen is *unique* or *significant* is to compare it to what happens when we just assign neighbors randomly. 
If what we observe is substantially different from what emerges when neighbors are random, then the structure of the neighbors embeds a structure in the residuals. 
We won't spend too much time on this theory specifically, but we can quickly and efficiently compute the correlation between our observed residuals and the spatial lag of an increasing $k$-nearest neighbor set:



{:.input_area}
```python
correlations = []
nulls = []
for order in range(1, 51, 5):
    knn.reweight(k=order, inplace=True) #operates in place, quickly and efficiently avoiding copies
    knn.transform = 'r'
    lag_residual = weights.spatial_lag.lag_spatial(knn, m1.u)
    random_residual = m1.u[numpy.random.permutation(len(m1.u))] 
    random_lag_residual = weights.spatial_lag.lag_spatial(knn, random_residual) # identical to random neighbors in KNN 
    correlations.append(numpy.corrcoef(m1.u.flatten(), lag_residual.flatten())[0,1])
    nulls.append(numpy.corrcoef(m1.u.flatten(), random_lag_residual.flatten())[0,1])
```




{:.input_area}
```python
plt.plot(range(1,51,5), correlations)
plt.plot(range(1,51,5), nulls, color='orangered')
plt.hlines(numpy.mean(correlations[-3:]),*plt.xlim(),linestyle=':', color='k')
plt.hlines(numpy.mean(nulls[-3:]),*plt.xlim(),linestyle=':', color='k')
plt.text(s='Long-Run Correlation: ${:.2f}$'.format(numpy.mean(correlations[-3:])), x=25,y=.3)
plt.text(s='Long-Run Null: ${:.2f}$'.format(numpy.mean(nulls[-3:])), x=25, y=.05)
plt.xlabel('$K$: number of nearest neighbors')
plt.ylabel("Correlation between site \n and neighborhood average of size $K$")
plt.show()
```



{:.output .output_png}
![png](../images/notebooks/11_regression_63_0.png)



Clearly, the two curves are different. The observed correlation reaches a peak around $r=.34$ when around 20 nearest listings are used. This means that adding more than 20 nearest neighbors does not significantly change the correlation in the residuals. Further, the lowest correlation is for the single nearest neighbor, and correlation rapidly increases as more neighbors are added close to the listing. Thus, this means that there does appear to be an unmeasured spatial structure in the residuals, since they are more similar to one another when they are near than when they are far apart. Further, while it's not shown here (since computationally, it becomes intractable), as the number of nearest neighbors gets very large (approaching the number of observations in the dataset), the average of the $k$th nearest neighbors' residuals goes to zero, the global average of residuals. This means that the correlation of the residuals and a vector that is nearly constant begins to approach zero. 

The null correlations, however, use randomly-chosen neighbors (without reassignment).
Thus, since sampling is truly random in this case, each average of $k$ randomly-chosen neighbors is usually zero (the global mean). 
So, the correlation between the observed residual and the average of $k$ randomly-chosen residuals is also usually zero. 
Thus, increasing the number of randomly-chosen neighbors does not significantly adjust the long-run average of zero.
Taken together, we can conclude that there is distinct positive spatial dependence in the error. 
This means that our over- and under-predictions are likely to cluster. 

# References
[1] Anselin, L. Spatial externalities, spatial multipliers, and spatial econometrics. *International Regional Science Review* 26, 156–166 (2003).

[2] Anselin, L. & Rey, S. J. *Modern Spatial Econometrics in Practice, a Guide to GeoDa, GeoDaSpace, and PySAL*. (GeoDa Press, 2014).

[3] Gelman, A., & Hill, J. (2006). Data analysis using regression and multilevel/hierarchical models. Cambridge university press.

[4] Gibbons, S., Overman, H. G., & Pattacchini, E. Chapter 3 - Spatial Methods. *Handbook of Regional and Urban Economics, Vol. 5*. (Elsevier, 2015): 115-168.

[5] Wood, S. N. *Generalized additive models: an introduction with R.* Chapman and Hall/CRC, 2006. 

[6] Brunsdon, Chris, A. Stewart Fotheringham, and Martin E. Charlton. "Geographically weighted regression: a method for exploring spatial nonstationarity." *Geographical Analysis* 28, vol. 4 (1996): 281-298.

[7] Banerjee, S., A. E. Gelfand, A. O. Finley, and H. Sang. "Gaussian predictive process models for large spatial data sets." Journal of the Royal Statistical Society: Series B (Statistical Methodology) 70, no. 4 (2008): 825-848.

[8] Cressie, N., and Christopher K. W. *Statistics for spatio-temporal data.* (John Wiley & Sons, 2015).
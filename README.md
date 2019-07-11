## I. Introduction
Scientists at Los Alamos Laboratory have recently found a use for massive amounts of data generated by a “constant tremor” of fault lines where earthquakes are most common **[1-3]**. This data has previously been disregarded as noise. However, now, it has been proven useful through the lens of Machine Learning (ML) **[1-2]**. Following their recent publications, our goal is to build _Machine Learning regression models for the Laboratory Earthquake problem_ that if applied to real data, might help predict earthquakes before they happen

#### Environment Setup
<details><summary>CLICK TO EXPAND</summary>
<p>
  
```markdown
import os
from scipy import ndimage, misc
from matplotlib import pyplot as plt
import numpy as np
import pandas as pd
from sklearn.datasets import load_boston, load_diabetes, load_digits, load_breast_cancer
from keras.datasets import mnist
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import PolynomialFeatures
from sklearn.linear_model import Ridge
from sklearn.metrics import mean_absolute_error
from sklearn import datasets, linear_model
from sklearn.model_selection import train_test_split
from sklearn.model_selection import KFold
import statistics 
```

</p>
</details>

## II. Feature Extraction
%describe the process%

```python
your code here
```
To access original data set, see [LANL Earthquake Prediction Data Set](https://www.kaggle.com/c/LANL-Earthquake-Prediction/data)

To access processed data set, see [Data Extracted](extract_train_full.csv)

## IIIa. Linear Regression
#### Transform data
<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
target = pd.read_csv("extract_label_Jul08.csv", delimiter = ',')
target = target.as_matrix()
target = target[:,1]

features = pd.read_csv("extract_train_Jul08.csv", delimiter = ',')
features = features.as_matrix()
features = features[:, 1:17]
```

</p>
</details>

#### Plot Features Methods
<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
def plot_features(features, reg, features_poly, number):
    plt.ylabel("Target Values")
    plt.xlabel("Feature #" + str(number))
    plt.scatter(features[:,number-1], target, s=10, c='b', marker="s", label='original')
    plt.scatter(features[:,number-1], reg.predict(features_poly),  s=10, c='r', marker="o", label='predicted')
    plt.legend(loc='upper right')
```

</p>
</details>

#### Kfold Cross Validation for Linear and Polynomial Regression
<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
def K_Fold(features, target, numfolds, classifier):

    kf = KFold(n_splits=numfolds)
    kf.get_n_splits(features)

    i = 0
    mae = np.zeros(numfolds-1) 
    for train_index, test_index in kf.split(features):
        features_train, features_test = features[train_index], features[test_index]
        target_train, target_test = target[train_index], target[test_index]
    
        poly = PolynomialFeatures(degree=2)
        features_poly_train = features_train 
        features_poly_test = features_test
        if classifier == "polynomial" :
            features_poly_train = poly.fit_transform(features_train)
            features_poly_test = poly.fit_transform(features_test)
        elif classifier == "linear":
            features_poly_train = features_train 
            features_poly_test = features_test
        
        reg = LinearRegression().fit(features_poly_train, target_train)
    
        i = i+1
        if (i < numfolds):
            mae[i-1] = mean_absolute_error(target_test, reg.predict(features_poly_test))
            
    avrmae = (sum(mae)/(numfolds-1))
    var = (statistics.variance(mae))
    return avrmae, var
```

</p>
</details>

#### Perform linear regression on the data

```python
reg = LinearRegression().fit(features, target)

print("The loss values is: ", mean_absolute_error(target, reg.predict(features)))
```
The loss values is:  2.110853811043013

#### Perfom Kfold cross validation on the data

```python
i = np.array([5,10,50,100, 150])
mae = np.zeros(i.shape[0])
var = np.zeros(i.shape[0])
for numfold in range(i.shape[0]):
    m, v = K_Fold(features,target, i[numfold], "linear")
    mae[numfold] = m
    var[numfold] = v
plt.plot(i, mae, color = 'blue', label = 'Average MAE')
plt.plot(i, var, color = 'red', label = 'Variance of MAE')
plt.legend(loc='lower right')
```
![Linear Regression K Kold](https://github.com/hoangtung167/cx4240/blob/master/Linear%20Regression%20K%20Fold.png)


#### Graph the results

<details><summary>CLICK TO EXPAND</summary>
<p>
 
```python
fig, axes = plt.subplots(nrows=3, ncols=2)
fig.tight_layout()
fig.subplots_adjust(left=0.1, bottom=-1.2, right=0.9, top=0.9, wspace=0.4, hspace=0.2)

for i in range(6):
    plt.subplot(32*10 +(i+1))
    plot_features(features, reg, features, i+1)

fig, axes = plt.subplots(nrows=3, ncols=2)
fig.tight_layout()
fig.subplots_adjust(left=0.1, bottom=-1.2, right=0.9, top=0.9, wspace=0.4, hspace=0.2)

for i in range(6):
    plt.subplot(32*10 +(i+1))
    plot_features(features, reg, features, i+7)

fig, axes = plt.subplots(nrows=3, ncols=2)
fig.tight_layout()
fig.subplots_adjust(left=0.1, bottom=-1.2, right=0.9, top=0.9, wspace=0.4, hspace=0.2)

for i in range(6):
    plt.subplot(32*10 +(i+1))
    plot_features(features, reg, features, i+13)
```

</p>
</details>

![Linear Regression](https://github.com/hoangtung167/cx4240/blob/master/Linear%20Regression.png)

#### Analysis

## IIIb. Polynomial Regression
#### Perform polynomial regression on the data
 
```python
poly = PolynomialFeatures(degree=2)
features_poly = poly.fit_transform(features)
reg = LinearRegression().fit(features_poly, target)

print("The loss values is: ", mean_absolute_error(target, reg.predict(features_poly)))
```
The loss values is:  1.985654086901071

#### Perfom Kfold cross validation on the data

```python
i = np.array([5,10,50,100, 150])
mae = np.zeros(i.shape[0])
var = np.zeros(i.shape[0])
for numfold in range(i.shape[0]):
    m, v = K_Fold(features,target, i[numfold], "linear")
    mae[numfold] = m
    var[numfold] = v
plt.plot(i, mae, color = 'blue', label = 'Average MAE')
plt.plot(i, var, color = 'red', label = 'Variance of MAE')
plt.legend(loc='lower right')
```
![Polynomial Regression K Fold](https://github.com/hoangtung167/cx4240/blob/master/Polynomial%20Regression%20K%20Fold.png)


#### Graph the results

<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
fig, axes = plt.subplots(nrows=3, ncols=2)
fig.tight_layout()
fig.subplots_adjust(left=0.1, bottom=-1.2, right=0.9, top=0.9, wspace=0.4, hspace=0.2)

for i in range(6):
    plt.subplot(32*10 +(i+1))
    plot_features(features, reg, features_poly, i+1)

fig, axes = plt.subplots(nrows=3, ncols=2)
fig.tight_layout()
fig.subplots_adjust(left=0.1, bottom=-1.2, right=0.9, top=0.9, wspace=0.4, hspace=0.2)

for i in range(6):
    plt.subplot(32*10 +(i+1))
    plot_features(features, reg, features_poly, i+7)

fig, axes = plt.subplots(nrows=3, ncols=2)
fig.tight_layout()
fig.subplots_adjust(left=0.1, bottom=-1.2, right=0.9, top=0.9, wspace=0.4, hspace=0.2)

for i in range(4):
    plt.subplot(32*10 +(i+1))
    plot_features(features, reg, features_poly, i+13)
```

</p>
</details>

![Polynomial Regression](https://github.com/hoangtung167/cx4240/blob/master/Polynomial%20Regression.png)


#### Analysis

## IV. Decision Tree/ Random Forest

## V. Deep Learning with Fully Connected, LSTM or 1D-CNN layer

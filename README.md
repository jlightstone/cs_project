## I. Problem statements
**Please edit this part**

Earthquakes are devastating natural disasters and possess the potential to destroy buildings and cities. In the process they can injure and even kill hundreds and thousands of people. It is necessary to develop solutions to better understand seismic activity to enable accurate and precise prediction of when an earthquake will happen. Tools with this capability provide could equip officials with the necessary information to minimize risks for populations expected to experience seismic activity.  


Scientist have recently discovered that “constant tremors” measured along fault lines provide valuable information about earthquake activity. The goal of this project is to utilize a machine learning (ML) approach to analyze experimental data that simulates the tremors at fault lines. Ultimately, the goal of the project is to develop a generalized ML model capable of correctly model the tremor waves as a function of time till failure.    


![Introduction_Data](https://github.com/hoangtung167/cx4240/blob/master/CSV%20Files/Introduction_data.png)


The data above shows a continuous block of experimental data. This graph shows plotted accoustic viabrations as a function of time. The plot also shows a "time to failure" component (a.k.a. the time of earthquake). In order to predict the time to failure, statistical features must be extracted from the data. The features used to describe the data are discussed in the next section. 

#### Environment Setup

<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
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

Using resourses from [kaggle](https://www.kaggle.com/c/LANL-Earthquake-Prediction/discussion/94390#latest-554034), we determined 16 statistical features that serve as potential candidates for understanding our the experimental data. The features are broken into four different catagories- basic, fast fourier transformed, rolling window, and mel-frequency features. 

The basic features are calculate using simple statistics and include ‘mean’, ‘std’, and ‘skew’.  The fast fourier transformed features convert the time-domain signal into a frequency-domain signal, which results in real and imaginary numbers. The mean and standard deviation for the real and imaginary numbers were calculated, resulting in 4 more features. The rolling windows method resulted from breaking the larger data set into a rolling window size of 100. From this, the mean, standard deviation, and upper and lower percentiles subsets were used to extract an additional six features. Lastly, the Librosa toolbox was used to calculate the Mel-frequency cepstral coefficients of two features. 

![Feature Extraction Concept](https://github.com/hoangtung167/cx4240/blob/master/Graphs/Feature_Extraction_Concept.png)

### Visualization of 16 features

![Feature Visualization](https://github.com/hoangtung167/cx4240/blob/master/Graphs/Feature_Visualization.png)

Once enough features from the data are extracted, one can begin train different ML algorithms to determine which is most capable of modeling the given set. In the following sectin, we test multiple algorithms in order to determine which are best at predicting the time to failure for a slip plane. The feature importance for each ML method is also discussed. 

### Feature Extractions Methods

<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
def generate_feature_basic(seg_id, seg, X):
    xc = pd.Series(seg['acoustic_data'].values)
    
    X.loc[seg_id, 'index'] = seg_id
    X.loc[seg_id, 'mean'] = xc.mean()
    X.loc[seg_id,'std'] = xc.std()
    #X.loc[seg_id, 'kurt'] = xc.kurtosis()
    X.loc[seg_id, 'skew'] = xc.skew()
def generate_feature_FFT(seg_id, seg, X):
    xc = pd.Series(seg['acoustic_data'].values)
    zc = np.fft.fft(xc)
    realFFT, imagFFT = np.real(zc), np.imag(zc)
    
    X.loc[seg_id, 'FFT_mean_real'] = realFFT.mean()
    X.loc[seg_id, 'FFT_mean_imag'] = imagFFT.mean()
    X.loc[seg_id, 'FFT_std_real'] = realFFT.std()
    X.loc[seg_id, 'FFT_std_max'] = realFFT.max()
    
def generate_feature_Roll(seg_id, seg, X):
    xc = pd.Series(seg['acoustic_data'].values)
    
    windows = 100
    x_roll_std = xc.rolling(windows).std().dropna().values
    x_roll_mean = xc.rolling(windows).mean().dropna().values
    
    X.loc[seg_id, 'Roll_std_p05'] = np.percentile(x_roll_std, 5)
    X.loc[seg_id, 'Roll_std_p30'] = np.percentile(x_roll_std,30)
    X.loc[seg_id, 'Roll_std_p60'] = np.percentile(x_roll_std,60)
    X.loc[seg_id, 'Roll_std_absDiff'] = np.mean(np.diff(x_roll_std))
    
    X.loc[seg_id, 'Roll_mean_p05'] = np.percentile(x_roll_mean, 5)
    X.loc[seg_id, 'Roll_mean_absDiff'] = np.mean(np.diff(x_roll_mean))

def generate_feature_Melfrequency(seg_id, seg, X):
    xc = seg['acoustic_data'].values
    mfcc = librosa.feature.mfcc(xc.astype(np.float64))
    mfcc_mean = mfcc.mean(axis = 1)
    
    X.loc[seg_id, 'MFCC_mean02'] = mfcc_mean[2]
    X.loc[seg_id, 'MFCC_mean16'] = mfcc_mean[16]

chunksize = 150000
CsvFileReader = pd.read_csv('train.csv', chunksize = chunksize)
X, y = pd.DataFrame(), pd.DataFrame()

for seg_id, seg in tqdm_notebook(enumerate(CsvFileReader)):
    y.loc[seg_id, 'target'] = seg['time_to_failure'].values[-1]
    generate_feature_basic(seg_id, seg, X)
    generate_feature_FFT(seg_id, seg, X)
    generate_feature_Roll(seg_id, seg, X)
    generate_feature_Melfrequency(seg_id, seg, X)

X.to_csv('extract_train_Jul08.csv')
y.to_csv('extract_label_Jul08.csv')
X1, y1 = X.iloc[500:1000], y.iloc[500:1000]

plt.figure(figsize=(15, 15))
for i, col in enumerate(X.columns):
    ax1 = plt.subplot(4, 4, i + 1)
    plt.plot(X1[col], color='blue');plt.title(col);ax1.set_ylabel(col, color='b')
    ax2 = ax1.twinx(); plt.plot(y1, color='g'); ax2.set_ylabel('time_to_failure', color='g')
    ax1.legend(loc= 2);ax2.legend(['time_to_failure'], loc=1)
plt.subplots_adjust(wspace=0.5, hspace=0.3)
```

</p>
</details>

## III. Testing Maching Learning Models

### Linear Regression and Polynomial Regression Performance

![Comapare Predicted Values](https://github.com/hoangtung167/cx4240/blob/master/Graphs/Compare%20Predicted%20Values.png)
</p>

![Compare MAE](https://github.com/hoangtung167/cx4240/blob/master/Graphs/Compare%20MAE%20Linear%20Polynomial.png)


<details><summary>CLICK TO EXPAND</summary>
<p>

#### Compare the Mean Absolute Error
<details><summary>CLICK TO EXPAND</summary>
<p>

```python
fl = ['Linear Regression', 'Polynomial Regression']
t1, m1, v1, c1 = K_Fold(features,target, degree = 1, numfolds = 5, classifier = "linear")
t2, m2, v2, c2 = K_Fold(features,target, degree = 2, numfolds = 5, classifier = "polynomial")
mae = np.append(m1, m2)
var = np.append(v1,v2)
## coeff = reg.coef_.shape
materials = fl
x_pos = np.arange(len(fl))
CTEs = mae
error = var
# Build the plot
fig, ax = plt.subplots(1,2,figsize =(9,3))
ax[0].bar(x_pos, CTEs, yerr=error, align='center', color = ['red', 'green'], alpha=0.5, ecolor='black', capsize=10)
ax[0].set_ylim(0, 3)
ax[0].set_ylabel('Mean Absolute Error')
ax[0].set_xticks(x_pos)
ax[0].set_xticklabels(materials)
ax[0].set_title('Kfold results')
ax[0].yaxis.grid(True)

CTEs = [2.110853811043013, 1.985654086901071]

ax[1].bar(x_pos, CTEs, align='center', color = ['red', 'green'], alpha=0.5, ecolor='black', capsize=10)
ax[1].set_ylim(0, 3)
ax[1].set_ylabel('Mean Absolute Error')
ax[1].set_xticks(x_pos)
ax[1].set_xticklabels(materials)
ax[1].set_title('Training the whole set')
ax[1].yaxis.grid(True)

# Save the figure and show
plt.tight_layout()
plt.savefig('Compare MAE Linear Polynomial.png', dpi = 199)
plt.show()
```

</p>
</details>

#### Comparing Linear and Polynomial Regression
<details><summary>CLICK TO EXPAND</summary>
<p>

```python
reg = LinearRegression().fit(features, target)

indx = range(target.shape[0])
plt.axis([0, target.shape[0], -0.1, 16])
plt.title("Comparison - Linear Regression")
plt.ylabel("Time before failure(s)")
plt.xlabel("Index")
plt.plot(indx, reg.predict(features), linewidth = 3, label = 'Pred')
plt.plot(indx, target, linewidth = 2, label = 'Actual')
plt.legend(loc='upper right')
plt.savefig('Compare P Linear.png', dpi = 324)
plt.show()

poly = PolynomialFeatures(degree=2)
features_poly = poly.fit_transform(features)
reg = LinearRegression().fit(features_poly, target)

indx = range(target.shape[0])
plt.axis([0, target.shape[0], -0.1, 16])
plt.title("Comparison - Polynomial Regression")
plt.ylabel("Time before failure(s)")
plt.xlabel("Index")
plt.plot(indx, reg.predict(features_poly), linewidth = 3, label = 'Pred')
plt.plot(indx, target, linewidth = 2, label = 'Actual')
plt.legend(loc='upper right')
plt.savefig('Compare P Polynomial.png', dpi = 324)
plt.show()
```

</details>



### Linear Regression 
<details><summary>CLICK TO EXPAND</summary>
<p>

```python
target = pd.read_csv("extract_label_Jul08.csv", delimiter = ',')
target = target.as_matrix()
target = target[:,1]

features = pd.read_csv("extract_train_Jul08.csv", delimiter = ',')
features = features.as_matrix()
features = features[:, 1:17]

def K_Fold(features, target, degree, numfolds, classifier):
    numfolds += 1
    kf = KFold(n_splits=numfolds)
    kf.get_n_splits(features)

    i = 0
    mae = np.zeros(numfolds-1)
    coef = np.zeros([numfolds-1, 16])
    for train_index, test_index in kf.split(features):
        features_train, features_test = features[train_index], features[test_index]
        target_train, target_test = target[train_index], target[test_index]

        poly = PolynomialFeatures(degree)
        features_poly_train = features_train
        features_poly_test = features_test

        if classifier == "polynomial" :
            features_poly_train = poly.fit_transform(features_train)
            features_poly_test = poly.fit_transform(features_test)

        reg = LinearRegression().fit(features_poly_train, target_train)
        
        if classifier == "ridge" :
            clf = Ridge(alpha=0.001)
            reg = clf.fit(features_poly_train, target_train)
        if classifier == "linear":
            if (i < numfolds - 1):
                coef[i, :] = reg.coef_.reshape(1, 16)
        if classifier == "lasso":
            clf = linear_model.Lasso(alpha=0.1)
            reg = clf.fit(features_poly_train, target_train)
        if classifier == "huber":
            reg = HuberRegressor().fit(features_poly_train, target_train)

        i = i+1
        if (i < numfolds):
            mae[i-1] = mean_absolute_error(target_test, reg.predict(features_poly_test))

        avrmae = (sum(mae)/(numfolds-1))
        var = (statistics.variance(mae))
        return mae, avrmae, var, coef

def mv(coefmat):
    
    mean = np.zeros(coefmat.shape[1])
    var = np.zeros(coefmat.shape[1])
    for i in range(coefmat.shape[1]):
        mean[i] = np.mean(coefmat[:, i])
        var[i] = np.std(coefmat[:, i])
    return mean, var

reg = LinearRegression().fit(features, target)
print("The loss values is: ", mean_absolute_error(target, reg.predict(features)))
indx = range(target.shape[0])
plt.axis([0, target.shape[0], -0.1, 16])
plt.title("Comparison between predicted and actual target values")
plt.ylabel("Time before failure(s)")
plt.xlabel("Index")
plt.plot(indx, reg.predict(features), linewidth = 3, label = 'Pred')
plt.plot(indx, target, linewidth = 2, label = 'Actual')
plt.legend(loc='upper right')
plt.savefig('Linear Regression.png', dpi = 199)

#different linear regression types 
fl = ['Linear', 'Ridge', 'Lasso', 'Huber Regressor']
## coeff = reg.coef_.shape
materials = fl
x_pos = np.arange(len(fl))
t1, m, v, c = K_Fold(features,target,degree = 1, numfolds = 5, classifier = "linear")
t2, m1, v1, c1 = K_Fold(features,target,degree = 1, numfolds = 5, classifier = "ridge")
t3, m2, v2, c2 = K_Fold(features,target,degree = 1, numfolds = 5, classifier = "lasso")
t4, m3, v3, c3 = K_Fold(features,target,degree = 1, numfolds = 5, classifier = "huber")
tot = np.append(np.append(np.append(t1,t2), t3), t4)
mae = [m, m1, m2, m3]
var = [v, v1, v2, v3]
CTEs = mae
error = var

# Build the plot
fig, ax = plt.subplots()
ax.bar(x_pos, CTEs, yerr=error, align='center', color = ['black', 'red', 'green', 'blue', 'cyan'], alpha=0.5, ecolor='black', capsize=10)
ax.set_ylabel('Mean Absolute Error')
ax.set_xlabel('Types of Regressor')
ax.set_xticks(x_pos)
ax.set_xticklabels(materials)
ax.set_title('KFold Mean Absolute Error Values with Variances')
ax.yaxis.grid(True)

# Save the figure and show
plt.tight_layout()
plt.savefig('Linear Regression K Fold.png', dpi = 199)
plt.show()

#feature importance
fl = ['index', 'mean', 'std', 'skew', 'FFT_mean_real', 'FFT_mean_imag',
     'FFT_std_real', 'FFT_std_max', 'Roll_std_p05', 'Roll_std_p30',
      'Roll_std_p60', 'Roll_std_absDiff', 'Roll_mean_p05',
      'Roll_mean_absDiff', 'MFCC_mean02', 'MFCC_mean16']
t, m, v, c = K_Fold(features,target, degree = 1, numfolds = 5, classifier = "linear")
mean, error = mv(c)
## coeff = reg.coef_.shape
materials = fl
x_pos = np.arange(len(fl))
CTEs = -mean/ np.amax(-mean)
error = error/ np.amax(-mean)
# Build the plot
fig, ax = plt.subplots()
ax.bar(x_pos, CTEs, yerr=error, align='center', color = ['black', 'red', 'green', 'blue', 'cyan'], alpha=0.5, ecolor='black', capsize=10)
ax.set_ylabel('Gradient value of features')
ax.set_xticks(x_pos)
ax.set_xticklabels(materials, rotation = 'vertical')
ax.set_title('Features Importance')
ax.yaxis.grid(True)

# Save the figure and show
plt.tight_layout()
plt.savefig('bar_plot_with_error_bars.png', dpi = 199)
plt.show()
```
</p>
</details>

### Polynomial Regression 
<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
  
fl = ['1','2','3','4']
i = np.array([1,2,3,4])
## coeff = reg.coef_.shape
materials = fl
x_pos = np.arange(len(fl))
tot = np.zeros(1)
mae = np.zeros(i.shape[0])
var = np.zeros(i.shape[0])
for numfold in range(i.shape[0]):
    t, m, v, c = K_Fold(features,target, degree = i[numfold], numfolds = 5, classifier = "polynomial")
    mae[numfold] = m
    var[numfold] = v
    tot = np.append(tot, t, axis = 0)
CTEs = mae
error = var
# Build the plot
fig, ax = plt.subplots()
ax.bar(x_pos, CTEs, yerr=error, align='center', color = ['black', 'red', 'green', 'blue', 'cyan'], alpha=0.5, ecolor='black', capsize=10)
ax.set_ylabel('Mean Absolute Error')
ax.set_xlabel('Degree Levels')
ax.set_xticks(x_pos)
ax.set_xticklabels(materials)
ax.set_title('KFold Mean Absolute Error Values with Variances')
ax.yaxis.grid(True)
tot = np.delete(tot, 0)
# Save the figure and show
plt.tight_layout()
plt.savefig('Polynomial K Fold.png', dpi = 199)
plt.show()
```  
</p>
</details>

![Linear Regression](https://github.com/hoangtung167/cx4240/blob/master/Graphs/Linear%20Regression.png)  

###### Analysis
From the graph, we see that the linear regression model provide fairly acceptable prediction on the outcome - "Time before failure(s)". However, we can observe the tendency to center the values: the model can not predict high peak an show consistent trend of repeating height - nearly periodic. To combat this situation, we decided to use two different approaches: first is to use different type of regressor and compare and validate them using Kfold cross-validation, second is to use the polynomial regression. We suspect that there is no significant improvement when using different types of linear regression as all of them have a tendency to center the values.

#### Compare different types of regression models

![Linear Regression K Fold](https://github.com/hoangtung167/cx4240/blob/master/Graphs/Linear%20Regression%20K%20Fold.png)

##### Analysis
Just as what we predicted, using other types of regressor such as Ridge, Lasso, and Huber Regressor do not increase accuracy significantly. Specially, we event observe a worst model with Huber Regressor: higher Mean Absolute Error with higher variance



#### Feature Importance

We output and graphs the coefficients in the weight from linear regression model corresponding to features. This graphs will be able to tell us the gradient values of features and thus their respective importance.

### Polynomial Regression

#### Perform K-Fold to analyze and determine optimal degree
<details><summary>CLICK TO EXPAND</summary>
<p>




![Polynomial K Fold](https://github.com/hoangtung167/cx4240/blob/master/Graphs/Polynomial%20K%20Fold.png)

##### Analysis
From the graph, we can observe that the Mean Absolute Error has a tendency to increase with degree levels. Furthermore, the variance also increases significantly which implies that we may overfit the data with higher order models. Since degree equals 1 is the Linear Regression model, we decide to use degree of 2 to continue our analysis.

</p>
</details>

#### Perform Polynomial Regression with Degree of 2
<details><summary>CLICK TO EXPAND</summary>
<p>

```python
poly = PolynomialFeatures(degree=2)
features_poly = poly.fit_transform(features)
reg = LinearRegression().fit(features_poly, target)

print("The loss values is: ", mean_absolute_error(target, reg.predict(features_poly)))
```
The loss values is:  1.985654086901071
```python
indx = range(target.shape[0])
plt.axis([0, target.shape[0], -0.1, 16])
plt.title("Comparison between predicted and actual target values")
plt.ylabel("Time before failures")
plt.xlabel("Index")
plt.plot(indx, reg.predict(features_poly), linewidth = 3, label = 'Pred')
plt.plot(indx, target, linewidth = 2, label = 'Actual')
plt.legend(loc='upper right')
plt.savefig('Polynomial Regression.png', dpi = 199)
```
![Polynomial Regression](https://github.com/hoangtung167/cx4240/blob/master/Graphs/Polynomial%20Regression.png)
</p>
</details>

</p>
</details>


</p>
</details>


## V. Decision Tree/ Random Forest / LGB Classifier

We use 3 different types of Tree-Classifier for this classification.

<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
from sklearn.tree import DecisionTreeRegressor 
from sklearn.ensemble import RandomForestRegressor
import lightgbm as lgb
```

</p>
</details>

### Decision Tree

<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
model = DecisionTreeRegressor(min_samples_split = 25, random_state = 1, 
                                  criterion='mae',max_depth=5)
```

With no Index, Validation MeanAbsoluteError: Mean = 2.067 Std = 0.044

![Decision Tree without Index](https://github.com/hoangtung167/cx4240/blob/master/Graphs/DT_woIndex.png)

With Index, Validation MeanAbsoluteError: Mean = 1.717 Std = 0.083
![Decision Tree with Index](https://github.com/hoangtung167/cx4240/blob/master/Graphs/DT_withIndex.png)

</p>
</details>

### Random Forest

<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
model = RandomForestRegressor(max_depth=5,min_samples_split=9,random_state=0,
                                  n_estimators=50,criterion='mae')
```

Validation MeanAbsoluteError: Mean = 2.020 Std = 0.031

![Random Forest without Index](https://github.com/hoangtung167/cx4240/blob/master/Graphs/RF_woIndex.png)

With Index, Validation MeanAbsoluteError: Mean = 1.617 Std = 0.038
![Random Forest  with Index](https://github.com/hoangtung167/cx4240/blob/master/Graphs/RF_withIndex.png)

</p>
</details>

### Light Gradient Boosting Machine (LGBM)

<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
model = RandomForestRegressor(max_depth=5,min_samples_split=9,random_state=0,
                                  n_estimators=50,criterion='mae')
```

Validation MeanAbsoluteError: Mean = 2.024 Std = 0.033

![LGBM without Index](https://github.com/hoangtung167/cx4240/blob/master/Graphs/LGBM_woIndex.png)

Validation MeanAbsoluteError: Mean = 0.680 Std = 0.036
![LGBM  with Index](https://github.com/hoangtung167/cx4240/blob/master/Graphs/LGBM_withIndex.png)

</p>
</details>

### Tree-based Technique Comparison
<details><summary>CLICK TO EXPAND</summary>
<p>
  
Cross Validation score for 5 folds:
![Tree_score](https://github.com/hoangtung167/cx4240/blob/master/Graphs/Tree_score.png)

Features Importance:
![Tree_feature_importance](https://github.com/hoangtung167/cx4240/blob/master/Graphs/Tree_feature_importance.png)
  
</p>
</details>

## VI. SVM/ Neural Nets
<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
from keras.models import Sequential
from keras.layers import Dense

from sklearn.svm import SVR
from sklearn.feature_selection import RFE
```

</p>
</details>

### Nerual Net (NN)
<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
model = Sequential()
model.add(Dense(32, activation='relu'))
model.add(Dense(16, activation='relu'))
model.add(Dense(1, activation='relu'))
model.compile(loss='mean_squared_error',
              optimizer='sgd',
              metrics=['accuracy'])
```

Validation MeanAbsoluteError: Mean = 2.113 Std = 0.033

![NN without Index](https://github.com/hoangtung167/cx4240/blob/master/Graphs/NN_without_Index.png)

Validation MeanAbsoluteError: Mean = 2.071 Std = 0.034
![NN with Index](https://github.com/hoangtung167/cx4240/blob/master/Graphs/NN_with_index.png)

</p>
</details>

### Support Vector Machine (SVM)
<details><summary>CLICK TO EXPAND</summary>
<p>
  
```python
model = SVR(kernel='linear')
```

Validation MeanAbsoluteError: Mean = 2.099 Std = 0.037

![SVM linear without Index](https://github.com/hoangtung167/cx4240/blob/master/Graphs/SVM_linear_withoutIndex.png)

Validation MeanAbsoluteError: Mean = 2.065 Std = 0.038
![SVM linear with Index](https://github.com/hoangtung167/cx4240/blob/master/Graphs/SVM_linear_withIndex.png)

```python
model = SVR(kernel='rbf')
```


</p>
</details>


## IV. Principal Component Analysis - PCA

Principal component analysis (PCA) is a technique used for understanding the dimensional structure of a data set. PCA transforms data in a way that converts a set of orthogonally correlated observations into a set of linearly uncorrelated variables called principal components.  This transformation maximizes the largest possible variance between each principal component.

In this work we use three different visualization methods to help understand the dimensional structure of the data and reduce the dimensionality of the dataset using PCA. 

 ### Running PCA 
<details><summary>CLICK TO EXPAND</summary>
<p>

```python
  import numpy as np
  import seaborn as sns
  import matplotlib.pyplot as plt
  import pandas as pd
  from sklearn.preprocessing import StandardScaler
  from sklearn.decomposition import PCA

  train = pd.read_csv('extract_train_Jul08.csv')
  train = train.drop(['index'], axis = 1)
  train = train.drop(train.columns[0],axis = 1)

  scaler=StandardScaler() #instantiate
  scaler.fit(train) # compute the mean and standard which will be used in the next command
  X_scaled=scaler.transform(train)
  pca=PCA() 
  pca.fit(X_scaled) 
  X_pca=pca.transform(X_scaled)
  ex_variance=np.var(X_pca,axis=0)
  ex_variance_ratio = ex_variance/np.sum(ex_variance)
  print(ex_variance_ratio)
```
</p>
</details>

 ### Pricipal Component Proportionality
<details><summary>CLICK TO EXPAND</summary>
<p>

```python
plt.figure(figsize=(10,5))
plt.bar(np.arange(1,16),pca.explained_variance_ratio_, linewidth=3)
plt.plot(np.arange(1,16),np.cumsum(pca.explained_variance_ratio_), linewidth=3, c = 'r', label = 'Cumulative Proportion')
plt.legend()
plt.xlabel('Principal Component')
plt.ylabel('Variance Proportion')
plt.grid()
plt.plot([0.99]*16, '--')
ex_variance=np.var(X_pca,axis=0)
ex_variance_ratio = ex_variance/np.sum(ex_variance)
print(ex_variance_ratio)
```
</p>
</details>

The x-axis of the graph below indicated each principal component for the featurized data set, while the y-axis accounts for the proportionality of the total variance contained within the data set. As expected, the first principal component accounts for the largest amount of variance. Each consecutive principal component accounts for slightly less variance than component before it. 

The red line shows the cumulative proportional variance after each principal component is formed. The dashed line is an indication of 99% variance of the data. One can see that the dashed line crosses the cumulative sum (red) line at the 9th principal component. This indicated that 99% of the variance within the data is accounted for when the dimensionality of the data is reduced from 16 dimensions down to 9 dimensions. 


![Principal Components Visualization](https://github.com/hoangtung167/cx4240/blob/master/Graphs/principal_component_visualization.png)

 ### Feature Variance for Principal Components 1 & 2
<details><summary>CLICK TO EXPAND</summary>
<p>
 
```python
plt.matshow([pca.components_[0]],cmap='viridis')
plt.yticks([0],['1st Comp'],fontsize=10)
plt.colorbar()
plt.xticks(range(len(train.columns)),train.columns,rotation=65,ha='left')
plt.show()

plt.matshow([pca.components_[1]],cmap='viridis')
plt.yticks([0],['2nd Comp'],fontsize=10)
plt.colorbar()
plt.xticks(range(len(train.columns)),train.columns,rotation=65,ha='left')
plt.show()
```
</p>
</details>

The two plots show the contributing variance of each feature in the first and second principal components. Yellow indicates a high positive variance while purple indicates a high negative variance. In the first principal component the features contributing to the most variance are the ‘Roll_std_pXX’ features as well as the “MFCC_mean02” feature. In the second principal component the “mean”, “FFT_std_max”, and “index” features contribute to the most variance. Knowing this correlation relationship could provide a framework for identifying the features providing the most significant variation within the model. 


![First Principal Component](https://github.com/hoangtung167/cx4240/blob/master/Graphs/first_principal_component.png)

![second Principal Component](https://github.com/hoangtung167/cx4240/blob/master/Graphs/second_principal_component.png)

 ### Visualizing Feature Correlation
<details><summary>CLICK TO EXPAND</summary>
<p>
 
```python
features = test.columns
plt.figure(figsize=(8,8))
s=sns.heatmap(test.corr(),cmap='coolwarm') 
s.set_yticklabels(s.get_yticklabels(),rotation=30,fontsize=7)
s.set_xticklabels(s.get_xticklabels(),rotation=30,fontsize=7)
plt.show()
```
</p>
</details>

The final graph within this section is a heat map which shows the correlation between different features. Dark red indicates that features have a strong positive correlation while dark blue indicates that there is a strong negative correlation between features. This heat map provides insight into which features are linearly independent and which variables linearly dependent. For example, the “Roll_std_p60” and “skew” features are linearly independent and have nearly zero correlation with other features. On the other hand, “Roll_std_60” is strongly correlated with 7 other features. 


![Feature Correlation](https://github.com/hoangtung167/cx4240/blob/master/Graphs/heat_map.png)

 ### Saving Reduced Dimensionality Matrix and Feature Importance
<details><summary>CLICK TO EXPAND</summary>
<p>
 
```python
a = np.abs(pca.components_[0])
a = a/np.max(a)
df = pd.DataFrame()
df['features'] = test.columns
df['importance'] = a
df.to_csv('PCA_extracted.csv')
print(df.shape)

pca=PCA(n_components = 9) 
pca.fit(X_scaled) 
X_pca=pca.transform(X_scaled)
df = pd.DataFrame(X_pca)
df.to_csv('pca_exported_9features.csv')
```
</p>
</details>


## VII. Summary

#### Compare the loss values and variances across methods

![Summary_MAE_Score](https://github.com/hoangtung167/cx4240/blob/master/CSV%20Files/Summary_MAE_Score.png)

#### Compare the feature importance across methods

![Summary_Feature_Importance](https://github.com/hoangtung167/cx4240/blob/master/CSV%20Files/Summary_Feature_Importance.png)

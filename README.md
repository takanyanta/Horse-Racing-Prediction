# Horse-racing
Web Scraping, Regression

## Purpose
* For my interest in horse racing(stochastic game)
* Expectation to be considered as the part of my investment

## Process

### 1. Collecting past data
* By using the technique of Web Scraping, extracting the race data from <a href="https://www.netkeiba.com/">netkeiba.com</a>.
* As there are enormous race data, it takes tremendous times by extracting all data. So focusing on "not retired horse(Genekiba)".

![Extract the frame](https://github.com/takanyanta/Horse-racing/blob/main/netkeiba.png "process1")

#### 1-1. Collecting URLs

```python
import csv
from urllib.request import urlopen
from bs4 import BeautifulSoup
import ssl
ssl._create_default_https_context = ssl._create_unverified_context
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import Select
import selenium
import pandas as pd 
import time
import os

def select_element_fx(name, value):
    element_name = driver.find_element_by_name(name)
    select_element = Select(element_name)
    select_element.select_by_value(value)
    
driver = webdriver.Chrome("chromedriver.exe")
driver.get('https://db.netkeiba.com/?pid=horse_search_detail')

# Define under age
select_element_fx('under_age', 'none')
# Define over age
select_element_fx('over_age', 'none')
# Define Displayed results
select_element_fx('list', '100')
# Click Genekiba
driver.find_element_by_id("check_other_01").click()

# Click Search Button
elements = driver.find_element_by_class_name("button")
driver.execute_script("arguments[0].click();", elements)

# get all URLs

time.sleep(0.1)
k = 0
l = 0
while True:
    url_list = []
    try:
        page = driver.find_elements_by_xpath('/html/body/div[1]/div[2]/div/div/div[2]')[0].text
        page =  page.split('件中')[1].split('件目')[0]
        pagemax = int(page.split('～')[1])-int(page.split('～')[0]) + 1
        for i in range(2, 2+ pagemax ,1):
            elem = driver.find_elements_by_xpath('//*[@id="contents_liquid"]/div/form/table/tbody/tr[{}]/td[2]/a'.format(i))[0].get_attribute("href")
            url_list.append(elem)

        if k == 0:
            driver.find_elements_by_xpath('/html/body/div[1]/div[2]/div/div/div[2]/a')[0].click()
            k += 1

        else:
            driver.find_elements_by_xpath('/html/body/div[1]/div[2]/div/div/div[2]/a[2]')[0].click()
    except IndexError:
        break

    pd.DataFrame(url_list, columns=['URL']).to_csv('\Uma{:07d}.csv'.format(l), encoding = "shift_jis")
    l += 1
    
pd.DataFrame(url_list, columns=['URL']).to_csv(r'\Uma{:07d}.csv'.format(l), encoding = "shift_jis")
```

#### 1-2. Download data from each URL

```python
def uma_get(driver, url):

    driver.get(url)

    # Get Table information
    tableElem = driver.find_element_by_class_name("db_h_race_results")
    trs = tableElem.find_elements(By.TAG_NAME, "tr")

    # Get without Header
    res1_ = []
    for i in range(1,len(trs)):
        tds = trs[i].find_elements(By.TAG_NAME, "td")
        temp=[]
        for j in range(0,len(tds)):
            temp.append(tds[j].text)
        res1_.append(temp)

    col1 = ["日付", "開催", "天気", "R", "レース名", "映像", "頭数", "枠番", 
            "馬番", "オッズ", "人気", "着番", "騎手", "斤量", "距離", "馬場",
            "馬場指数", "タイム", "着差", "タイム指数", "通過", "ペース", 
           "上り", "馬体重", "厩舎コメント", "備考", "勝ち馬(2着馬)", "賞金"]

    df = pd.DataFrame(res1_, columns=col1)

    # Get Table information
    tableElem = driver.find_element_by_class_name("db_prof_table")
    trs = tableElem.find_elements(By.TAG_NAME, "tr")

    # Get without Header
    res2_ = []
    for i in range(0,len(trs)):
        tds = trs[i].find_elements(By.TAG_NAME, "td")
        for j in range(0,len(tds)):
            res2_.append(tds[j].text)
            
    uma_name = driver.find_elements_by_xpath("/html/body/div[1]/div[2]/div[2]/div[1]/div[1]/div[1]/h1")[0].text

    df["生年月日"] = res2_[0]
    df["調教師"] = res2_[1]
    df["馬主"] = res2_[2]
    df["生産者"] = res2_[3]
    df["産地"] = res2_[4]
    df["セリ取引価格"] = res2_[5]
    df["獲得賞金"] = res2_[6]
    df["通算成績"] = res2_[7]
    df["主な勝鞍"] = res2_[8]
    df["近親馬"] = res2_[9]
    df["馬名"] = uma_name
    
    return df
    
driver = webdriver.Chrome("chromedriver.exe")
df = uma_get(driver, 'https://db.netkeiba.com/horse/2018105845/')
```

### 2. Descriptive analysis
* Becoming suspicious about common knowledge about horse racing

| Common knowledge | Explanation |
----|---- 
| The horse which runs in the inner frame tends to win | The one which runs in the outer frame tends to lose because of sands falled down on the face |
| For the the turf race, when field conditions becomes wose, the speed tends to be slow  | TD4 |

#### 2-1. Importing Data

* Translating Columns-Name from Japanese to English

| Japanese | English | Japanese | English | Japanese | English | Japanese | English |
----|---- |---- |---- |---- |---- |---- |---- 
|日付|Date|着番|Arrival_Number|上り|Final_Time|セリ取引価格|Trading_Price|
|開催|Place|騎手|Jockey|馬体重|Horse_Weight|獲得賞金|Accumulate_Prize|
|天気|Weather|斤量|Weight|厩舎コメント|Comment|通算成績|Accumulate_record|
|R|R|距離|Distance_and_Type|備考|Others|主な勝鞍|Main_Win_Race|
|レース名|Race_Name|馬場|Course_Condition|勝ち馬(2着馬)|Winning_Horse(Second_Winner)|近親馬|Relative_Horse|
|映像|Movie|馬場指数|Course_Condition_Index|賞金|Prize|馬名|Horse_Name|
|頭数|Number_of_Heads|タイム|Time|生年月日|Birthday|-|-|
|枠番|Frame_Number|着差|Difference|馬調教師主|Trainer|-|-|
|馬番|Horse_Number|タイム指数|Time_Index|馬主|Owner|-|-|
|オッズ|Odds|通過|Passing|生産者|Producer|-|-|
|人気|Popularity|ぺース|Pace|産地|Origin|-|-|

```python
import glob
import numpy as np
import tqdm
import pandas as pd
import matplotlib.pyplot as plt
import datetime
import seaborn as sns
from scipy.stats import gaussian_kde
from mypulp import *
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import train_test_split, cross_val_score, GridSearchCV
from sklearn.linear_model import Ridge, Lasso
from sklearn.ensemble import RandomForestRegressor
from xgboost import XGBRegressor
pd.set_option('display.max_columns' ,100)
pd.set_option('display.max_rows' ,100)
from hyperopt import fmin, tpe, hp, STATUS_OK, Trials, space_eval
from sklearn.metrics import r2_score

df = pd.DataFrame()
for i in glob.glob("*csv"):
    df = pd.concat([df, pd.read_csv(i, encoding="cp932")])

del df["Unnamed: 0"]

del df["Unnamed: 0.1"]

df.columns.values

cols_ = ["Date", "Place", "Weather", "R", "Race_Name", "Movie", "Number_of_Heads", "Frame_Number", "Horse_Number", "Odds", "Popularity", 
"Arrival_Number", "Jockey", "Weight", "Distance_and_Type", "Course_Condition", "Course_Condition_Index", "Time", "Difference", "Time_Index","Passing", 
"Pace", "Final_Time","Horse_Weight", "Comment", "Others", "Winning_Horse(Second_Winner)", "Prize", "Birthday", 
"Trainer", "Owner", "Producer", "Origin", "Trading_Price", "Accumulate_Prize", "Accumulate_record", "Main_Win_Race", "Relative_Horse", 
"Horse_Name"]

rename_cols={}
for i in range(len(cols_)):
    rename_cols[df.columns.values[i]] = cols_[i]

df = df.rename(columns=rename_cols)

df.head(5)
```


#### 2-2. Preprocessing Data

* Preprocessing data which makes data bocome able to be analyzed.

```python
df["Date"] = pd.to_datetime(df["Date"])

df["Weather"] = df["Weather"].apply(lambda x: str(x).replace(" ", "").replace("小雨", "drizzle").replace("雨", "rain").replace("小雪", " light_snow").replace("晴", "sunny").replace("雪", "snow").replace("曇", "cloudy"))

df["R"] = df["R"].apply(lambda x :np.nan if x == " " else  int(x))
df["Number_of_Heads"] = df["Number_of_Heads"].apply(lambda x :np.nan if x == " " else  int(x))
df["Frame_Number"] = df["Frame_Number"].apply(lambda x :np.nan if x == " " else  int(x))
df["Horse_Number"] = df["Horse_Number"].apply(lambda x :np.nan if x == " " else  int(x))
df["Odds"] = df["Odds"].apply(lambda x :np.nan if x == " " else  float(str(x).replace(",", "")))
df["Popularity"] = df["Popularity"].apply(lambda x :np.nan if x == " " else  int(x))
df["Weight"] = df["Weight"].apply(lambda x:float(x))

df["Difference"] = df["Difference"].apply(lambda x :np.nan if x == " " else  float(x))

df["Distance_and_Type"] = df["Distance_and_Type"].fillna("無0")
distance_len = []
distance_type = []
for i in range(len(df)):
    if df["Distance_and_Type"].iat[i][0:1].isdigit() == False:
        if len(df["Distance_and_Type"].iat[i]) > 1:
            distance_type.append(df["Distance_and_Type"].iat[i][0:1])
            distance_len.append(int(df["Distance_and_Type"].iat[i][1:]))
        else:
            distance_type.append(df["Distance_and_Type"].iat[i][0:1])
            distance_len.append(0)
    else:
        distance_type.append("無")
        distance_len.append(int(df["Distance_and_Type"].iat[i][0:]))

df["Distance"] = distance_len

df["Type"] = distance_type

df["Type"] = df["Type"].apply(lambda x: x.replace("ダ", "DIRT").replace("芝", "TURF").replace("障", "OBSTACLE").replace("無", ""))

df["Time"] = df["Time"].apply(lambda x : float(x.split(":")[0])+float(x.split(":")[1])/60 if (":" in x) else np.nan)
df["Horse_Weight_Actual"] = df["Horse_Weight"].apply(lambda x : float(x.split("(")[0]) if "(" in x else np.nan)
df["Horse_Weight_Change"] = df["Horse_Weight"].apply(lambda x : float(x.split("(")[1].split(")")[0]) if "(" in x else np.nan)
df["Prize"] = df["Prize"].apply(lambda x : 0 if " " in x else float( x.replace(",", "")) )
df["Birthday"] = df["Birthday"].apply(lambda x : datetime.datetime(int( x.split("年")[0] ),
                                              int( x.split("年")[1].split("月")[0] ),
                                              int( x.split("年")[1].split("月")[1].split("日")[0] ),) if "月" in x else np.nan)

df["Weight_per_Horse_Weight"] = df["Weight"]/df["Horse_Weight_Actual"]

df["MPM"] = df["Distance"]/df["Time"]

df["MPM"] = df["MPM"].apply(lambda x : x if x != np.inf else np.nan)

df["Old"] = df.apply(lambda row: (row["Date"] - row["Birthday"]).total_seconds()/60/60/24/365.25, axis=1)

df["Course_Condition"] = df["Course_Condition"].apply(lambda x : str(x).replace("良", "Good_to_Firm/Standard").replace("稍", "Good").replace("重", "Yielding/Muddy").replace("不", "Soft/Sloppy"))

df["Foreign_or_Domestic"] = df["Trainer"].apply(lambda x : "Foreign" if "海外" in str(x) else "Domestic")

df.groupby("Foreign_or_Domestic")["MPM"].agg(["mean", "std", "count", "max", "min"])
```

#### 2-3. Descreptive Analysis

* First, we focus on "DIRT" and "TURF" because the number of "OBSTACLE" is much less than the others.

|Race Type|Data|
---|---
|DIRT|154483|
|TURF|46129|
|OBSTACLE|2007|

* Second, *MPM* has outliers, so choose upper 99% of data.

| class | Histgram |
----|----
|Before|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/1.png "process1") |
|After|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/2.png "process1")|

```python
use_ = []
dirt_min =  np.percentile(df1[df1["Type"] == "DIRT"].dropna()["MPM"], 1)
turf_min =  np.percentile(df1[df1["Type"] == "TURF"].dropna()["MPM"], 1)
for i in range(len(df1)):
    if df1["Type"].iat[i] == "DIRT" and df1["MPM"].iat[i] >=dirt_min:
        use_.append(True)
    elif df1["Type"].iat[i] == "TURF" and df1["MPM"].iat[i] >= turf_min:
        use_.append(True)
    else:
        use_.append(False)

df2 = df1[np.array(use_)].reset_index(drop=True)
```

* Below is the result of descriptive analysis

| Name | Chart | Explanation|
---|---|---
|Origin|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/3.png "process1")|Some Origins tends to win, but their race count is not a lot|
|Jockey|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/4.png "process1")|Some Jockeys tends to win, but their race count is not a lot|
|Trainer|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/5.png "process1")|No correlation can be seen|
|Horse_Weight_Actual|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/6.png "process1")|No correlation can be seen|
|Horse_Weight_Change|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/7.png "process1")|No correlation can be seen|
|Weight_per_Horse_Weight|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/8.png "process1")|No correlation can be seen|
|Old|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/9.png "process1")|No correlation can be seen|
|Number_of_Heads|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/12.png "process1")|There is a correlation between MPM and Number of heads|
|Frame_Number|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/13.png "process1")|No correlation can be seen|
|Weather|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/14.png "process1")|There is a correlation between MPM and weather|
|Course_Condition|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/15.png "process1")|There is a correlation between MPM and course condition|
|Distance|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/17.png "process1")|There is a correlation between MPM and distance|

### 3. Build Regression Model

* Choose below 12 features for building regression model; ["MPM","Weather", "Number_of_Heads", "Frame_Number", "Horse_Number", "Course_Condition", "Distance", "Horse_Weight_Actual","Horse_Weight_Change", "Weight_per_Horse_Weight",  "Old", "Interval"]
* And use the previous -1th~-3th race result.
* Define the data which can be integrated to whole race result as validation data.

```python
df3 = df3.sort_values(["Horse_Name", "Date"]).reset_index(drop=True)

select1 = df3["Course_Condition"] != "nan"

select2 =  (df3["Weather"] != "") | (df3["Weather"] != "nan")

df3.columns.values

df3["Interval"] = df3["Date"].diff().apply(lambda x : x.days)

for_rename = [ ["{}_{}".format(df3.columns.values[j],i) for j in range(len(df3.columns.values))] for i in range(1, 4, 1)]

dict1 = {}
dict2 = {}
dict3 = {}
for i in range(len( for_rename[0] )) :
    dict1[df3.columns.values[i]] = for_rename[0][i]
for i in range(len( for_rename[1] )) :
    dict2[df3.columns.values[i]] = for_rename[1][i]
for i in range(len( for_rename[2] )) :
    dict3[df3.columns.values[i]] = for_rename[2][i]

df4 = df3.copy()

df4 = pd.merge(df4,  df3.shift(periods=1).rename(columns=dict1),  left_index=True, right_index=True)

df4 = pd.merge(df4,  df3.shift(periods=2).rename(columns=dict2),  left_index=True, right_index=True)

df4 = pd.merge(df4,  df3.shift(periods=3).rename(columns=dict3),  left_index=True, right_index=True)

select3 = (df4["Horse_Name"] == df4["Horse_Name_1"]) & (df4["Horse_Name_1"] == df4["Horse_Name_2"]) & (df4["Horse_Name_2"] == df4["Horse_Name_3"])

df5 = df4[select1 & select2 & select3].reset_index(drop=True)

df5.columns.values[:int(len(df5.columns.values)/4)]

cols = ["MPM","Weather", "Number_of_Heads", "Frame_Number", "Horse_Number", "Course_Condition", "Distance", "Horse_Weight_Actual", 
        "Horse_Weight_Change", "Weight_per_Horse_Weight",  "Old", "Interval"]

analyze_cols = cols + ["{}_{}".format(j, i) for j in cols for i in range(1, 4,1)]

df6 = df5[df5[analyze_cols].isnull().sum(axis=1) == 0]

df_DIRT = df6[df6["Type"] == "DIRT"].reset_index(drop=True)
df_TURF = df6[df6["Type"] == "TURF"].reset_index(drop=True)

Num_heads_dirt = df_DIRT.groupby(['Date', 'Place',  'R', 'Race_Name'])["Number_of_Heads"].mean()

R_count_dirt = df_DIRT.groupby(['Date', 'Place',  'R', 'Race_Name'])["R"].count()

valid_race_dirt = Num_heads_dirt[Num_heads_dirt == R_count_dirt]

Num_heads_turf = df_TURF.groupby(['Date', 'Place',  'R', 'Race_Name'])["Number_of_Heads"].mean()

R_count_turf = df_TURF.groupby(['Date', 'Place',  'R', 'Race_Name'])["R"].count()

valid_race_turf = Num_heads_turf[Num_heads_turf == R_count_turf]

valid_race_dirt1 = valid_race_dirt.reset_index().rename(columns={"Number_of_Heads":"Valid"})
valid_race_turf1 = valid_race_turf.reset_index().rename(columns={"Number_of_Heads":"Valid"})

df_DIRT_add_Valid = pd.merge(df_DIRT, valid_race_dirt1, on = ['Date', 'Place',  'R', 'Race_Name'], how="left" )
df_TURF_add_Valid = pd.merge(df_TURF, valid_race_turf1, on = ['Date', 'Place',  'R', 'Race_Name'], how="left" )

df_DIRT_analyze = df_DIRT_add_Valid[analyze_cols+["Valid"]]
df_TURF_analyze = df_TURF_add_Valid[analyze_cols+["Valid"]]

df_DIRT_analyze_dummies = pd.get_dummies(df_DIRT_analyze,drop_first=True )
df_TURF_analyze_dummies = pd.get_dummies(df_TURF_analyze,drop_first=True )

df_DIRT_analyze_dummies_traintest = df_DIRT_analyze_dummies[df_DIRT_analyze_dummies["Valid"].isnull()]
df_TURF_analyze_dummies_traintest = df_TURF_analyze_dummies[df_TURF_analyze_dummies["Valid"].isnull()]
df_DIRT_analyze_dummies_valid = df_DIRT_analyze_dummies[~df_DIRT_analyze_dummies["Valid"].isnull()]
df_TURF_analyze_dummies_valid = df_TURF_analyze_dummies[~df_TURF_analyze_dummies["Valid"].isnull()]

del df_DIRT_analyze_dummies_traintest["Valid"]
del df_TURF_analyze_dummies_traintest["Valid"]
del df_DIRT_analyze_dummies_valid["Valid"]
del df_TURF_analyze_dummies_valid["Valid"]

X_train_dirt, X_test_dirt, y_train_dirt, y_test_dirt = train_test_split(df_DIRT_analyze_dummies_traintest.drop("MPM", axis=1), df_DIRT_analyze_dummies_traintest[["MPM"]])
X_train_turf, X_test_turf, y_train_turf, y_test_turf = train_test_split(df_TURF_analyze_dummies_traintest.drop("MPM", axis=1), df_TURF_analyze_dummies_traintest[["MPM"]])
X_valid_dirt, y_valid_dirt = df_DIRT_analyze_dummies_valid.drop("MPM", axis=1), df_DIRT_analyze_dummies_valid[["MPM"]]
X_valid_turf, y_valid_turf = df_TURF_analyze_dummies_valid.drop("MPM", axis=1), df_TURF_analyze_dummies_valid[["MPM"]]

sc_dirt_X = StandardScaler()
sc_dirt_y = StandardScaler()
sc_turf_X = StandardScaler()
sc_turf_y = StandardScaler()

X_train_dirt_std = sc_dirt_X.fit_transform(X_train_dirt)
X_test_dirt_std = sc_dirt_X.transform(X_test_dirt)
X_valid_dirt_std = sc_dirt_X.transform(X_valid_dirt)

y_train_dirt_std = sc_dirt_y.fit_transform(y_train_dirt)
y_test_dirt_std = sc_dirt_y.transform(y_test_dirt)
y_valid_dirt_std = sc_dirt_y.transform(y_valid_dirt)

X_train_turf_std = sc_turf_X.fit_transform(X_train_turf)
X_test_turf_std = sc_turf_X.transform(X_test_turf)
X_valid_turf_std = sc_turf_X.transform(X_valid_turf)

y_train_turf_std = sc_turf_y.fit_transform(y_train_turf)
y_test_turf_std = sc_turf_y.transform(y_test_turf)
y_valid_turf_std = sc_turf_y.transform(y_valid_turf)
```

* For the Interpretation of the model, use Lasso-Regression with L1-regularization
* As the result of hyperparameter tuning, "alpha" should be set to 0.006739 for both DIRT and TURF.

```python
#Define parameter which is wanted to be searched
hyperopt_parameters = {
    'alpha': hp.loguniform('alpha', -5, 0),
}

def objective(args):
    estimator = Lasso(**args)
    
    estimator.fit(X_train_dirt_std, y_train_dirt_std)
    
    predicts = estimator.predict(X_test_dirt_std)
    R2 = r2_score(y_test_dirt_std, predicts)
    return -1*R2

max_evals = 200
trials = Trials()

best = fmin(
    objective,
    hyperopt_parameters,
    algo=tpe.suggest,
    max_evals=max_evals,
    trials=trials,
    verbose=1
)
```
* The result of DIRT

| Class | Chart | Explanation |
---|---|---
|Feature Importance(Absolute value of Coef)|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/18.png "process1")|Numboer of heads, MPM and Distance are strongly correlated|
|Parity plot(Train)|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/19.png "process1")|R2-score : 0.73089|
|Parity plot(Test)|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/21.png "process1")|R2-score : 0.72800|
|Parity plot(Valid)|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/21.png "process1")|R2-score : 0.62278|

* The result of TURF

| Class | Chart | Explanation |
---|---|---
|Feature Importance(Absolute value of Coef)|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/22.png "process1")|MPM, Distance and Course Condition are strongly correlated|
|Parity plot(Train)|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/23.png "process1")|R2-score : 0.72167|
|Parity plot(Test)|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/24.png "process1")|R2-score : 0.71530|
|Parity plot(Valid)|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/25.png "process1")|R2-score : 0.75462|

### 4. Betting Strategy

* For thinking simple, focus on "Win(Tansho)"

#### 4-1 Result of Predicting

##### DIRT
* The percentage of predicting "Win" is **8.19%**(=0.59/(0.59+0.83+0.71+5.11))

|-|Actual 1th|Actual 2th|Actual 3th|Actual 4th~|
---|---|---|---|---
|**Predict 1th**|0.59%|0.83%|0.71%|5.11%|
|**Predict 2th**|1.66%|0.83%|0.95%|3.80%|
|**Predict 3th**|0.71%|0.83%|0.59%|5.11%|
|**Predict 4th~**|4.28%|4.88%|4.88%|64.16%|

##### TURF
* The percentage of predicting "Win" is **17.4%**(=1.47/(1.47+0.86+0.36+5.78))

|-|Actual 1th|Actual 2th|Actual 3th|Actual 4th~|
---|---|---|---|---
|**Predict 1th**|1.47%|0.86%|0.36%|5.78%|
|**Predict 2th**|0.73%|0.98%|1.23%|5.54%|
|**Predict 3th**|0.36%|0.98%|1.479%|5.66%|
|**Predict 4th~**|5.91%|5.66%|5.54%|57.3%|

#### 4-2 Odds distribution of 1th

|-|Chart|Explanation|
---|---|---
|DIRT|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/26.png "process1")|Mean : 7.3, Median : 3.4|
|TURF|![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/27.png "process1")|Mean : 9.7, Median : 4.7|

#### 4-3 Compute the expected value

* DIRT : 8.19%×7.3=0.59 **< 1**
* TURF : 17.4%×9.7=1.68 **> 1**

## Conclusion
* We should bet not on DIRT race but on TURF race.
* The race cards are shown at Friday evening and the acutal horse weight are shown 40minute before the race. So we should run built model every weekend.

![Extract the frame](https://github.com/takanyanta/Horse-Racing-Analytics/blob/main/pic/27.png "process1")

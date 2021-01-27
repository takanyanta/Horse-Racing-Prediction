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

#### 1. Importing Data

* First, translate Columns-Name from Japanese to English

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


#### 2. Preprocessing Data

* Second, preprocess data which makes data bocome able to be analyzed.

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

#### 3. Descreptive Analysis

### 3. Optimization of betting
* As there are many betting styles, focused to the below; 

| Betting styles | Explanation |
----|---- 
| Win(Tansho) | predicting the horse which goals first  |
| Place, Show(Fukusyo)  | predicting the horse which goals within third(Second when the number of starters is less than 8) |
| Quinella(Umaren)  | predicting the two horses which goal within first and second, in no particular order |
| Exacta(Umatan)  | redicting the two horses which goal within first and second, in particular order |
| Quinella Place(Waido)  | predicting the two horses which goal within third, in no particular order |
| Trio(Sanrenpuku)  | predicting the three horses which goal within third, in no particular order |
| Trifecta(Sanrentan)  | predicting the three horses which goal within third, in particular order  |

### 4. Test by real race
* 

## Conclusion
* 
* 
* 


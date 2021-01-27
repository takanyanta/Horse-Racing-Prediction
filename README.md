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

| Japanese Word | English Word |
----|---- 
|日付|Date||着番|
|開催|Place|騎手||
|天気|Place|斤量||
|R|Place|距離||
|レース名|Place|馬場||
|映像|Place|馬場指数||
|頭数|Place|タイム||
|枠番|Place|着差||
|馬番|Place|タイム指数||
|オッズ|Place|通過||
|人気|Place|ぺース||


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


# Demo about realtime analytics of user behavior
- This project is designed based on mapR Tech blog.
- https://www.mapr.com/blog/real-time-user-profiles-spark-drill-and-mapr-db


## Part 1. Demo scenario
### 1). User Story
- Music streaming site를 제공하는 회사에서 수많은 사용자들의 행위(behavior)를 조회하고자 함.
- 하루 중 가장 많이 사용하는지? 남성/여성 및 연령의 비율은? Mobile/PC의 비율은? 접속지역은?
- 이러한 정보를 다양한 chart, map으로 시각화하여 실시간으로 보고자 함.


- 또한 실시간으로 수집된 사용자 정보를 활용하여 마케팅 대상을 분류할 수 있는 머신러닝 모델 학습
- 학습된 모델을 이용해 사용자별로 마케팅 적용대상을 분류하여 접속시에 광고를 전달하고자 함.
- 이를 위해 많은 사용자들의 log를 실시간으로 수집하여 분산처리 및 시각화하는 기술/시간/자원 필요.

### 2). Data Model
- Individual customers listening to individual tracks

   ![Image of tracks table] (https://www.mapr.com/sites/default/files/blogimages/blog_RealTimeUser-table1.png)

- Customer information

   ![Image of customers table] (https://www.mapr.com/sites/default/files/blogimages/blog_RealTimeUser-table2.png)

- Previous ad clicks : indicating which ad was played to the user and whether or not they clicked on it (clicks table)

```
 EventID, CustID, AdClicked,           Localtime
 0,       109,    "ADV_FREE_REFERRAL", "2014-12-18 08:15:16"
```

- Customer behaviors (live table) : summary data about listening habits, for example what time(s) of day were they listening, how much listening on a mobile device, and how many unique tracks they played

### 3). Define demo level
#### Stage 1. Simple realtime visualization
- 사용자의 접속로그를 logstsh로 수집하여 Elasticsearch로 저장한 후, kibana를 이용하여 빠르게 시각화


#### Stage 2. Stage1 + distributed processing with apache spark
- logstash에서  kafka로 저장하고, 이를 spark에서 실시간 집계를 수행 -> ES
- customerid, trackid와 상세정보를 join하여 데이터를 추가한다. -> ES
- 특정 시간(30분) 이내에 같은 곡을 3번 이상 들은 사용자는 해당곡을 관심 list로 등록 -> Redis, ES


#### Stage 3. Stage 1~2 + classify user by mllib(logistic regression)
- 특정 등급(Gold)이하의 사용자에게 이벤트 광고
- (display ad about one day gold grade upgrade offer)
- 주기적으로 ml model(logistic regression)을 학습하고, 에러율을 체크하여 모델 사용여부를 판단
- 에러가 일정수준(에를 들어 10% 이하)이면 해당 사용자를 광고할 타겟으로 분류
- Stage2의 spark streaming에 광고타겟 사용자일 경우 SMS전송 기능 추가 (Redis로 저장 -> SMS 서버에서 발송)


#### Stage 4. Stage 1~3 + advanced visualization for web page with c3.js or slamData
- nodejs를 이용하여 직접 실시간 데이터의 시각화



## Part 2. Implementing Stage 1
### data generator
- 실시간으로 데이터가 유입될 수 있도록 data generator에서 특정 file에 write (10 records / sec)
- tracks.csv -> data generator(date를 현재 시간으로 조정) -> tracks_live.log
- 1,000,000건이 저장된 tracks.csv에서 파일을 읽어서, 랜덤한 시간 간격으로 tracks_live.csv에 쓴다.
- 이는 실시간으로 사용자들이 접속하는 log를 재연하기 위한 용도로 사용
```{.python}
#-*- coding: utf-8 -*-
import time
from random import random
from bisect import bisect

def weighted_choice(choices):
    values, weights = zip(*choices)
    total = 0
    cum_weights = []
    for w in weights:
        total += w
        cum_weights.append(total)
    x = random() * total
    i = bisect(cum_weights, x)
    return values[i]

r_fname = "tracks.csv"
w_fname = "tracks_live.csv"

rf = open(r_fname)
wf = open(w_fname, "a+")

try:
    num_lines = sum(1 for line in rf)
    rf.seek(0)
    lines = 0
    while (1):
        line = rf.readline()
        wf.write(line)
        wf.flush()

        stime = weighted_choice([(1,30), (0.5,20), (2,20), (0.8,10), (0.3,10), (0.1,5), (3,5)])
        print(stime)
        time.sleep(stime)
        lines += 1

        if(lines == num_lines):
            break
finally:
    rf.close()
    wf.close()
    print "close file"
```

### logstash
#### install
 - [link] 참고.d

#### configuration (collect logs and save to ES, logstash_stage1.conf)
 -  input 
  -  data_generator.py가 write하는 tracks_live.csv에 내용이 추가되면 읽어서 ES에 저장한다.
 - filtersssss
  - 
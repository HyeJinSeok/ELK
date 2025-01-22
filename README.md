# ELK 기반 주문 데이터 모니터링


## 팀원 소개


|<img src="https://avatars.githubusercontent.com/u/193798531?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/74342019?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/153366521?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/127267532?v=4" width="150" height="150"/>|
|:-:|:-:|:-:|:-:|
|김리영 (Kim Ri-yeong)<br/>[@riyeong0916](https://github.com/riyeong0916)|Ryan Na<br/>[@CooolRyan](https://github.com/CooolRyan)|Park ji hye<br/>[@parkjhhh](https://github.com/parkjhhh)|[@HyeJinSeok](https://github.com/HyeJinSeok)|


## 프로젝트 소개


실제 elastic search가 활용되는 분야를 고민하던 중 로그 수집과 분석, 추천 알고리즘을 떠올릴 수 있었다.<br>
그 중 고객 데이터를 활용해  


![image](https://github.com/user-attachments/assets/ac8ca101-4d65-46db-9713-7ea7cdf3d126)


무신사에서는 일부 크롤링을 허용해주는 것을 robots.txt를 통해 확인할 수 있었다. 직접 크롤링하는데 시간이 걸리기 때문에 [해시스크래퍼](https://www.hashscraper.com/) 사이트에서 제공하는 무료 크롤링 기능을 활용해 상품 데이터를 추출해와서 활용했다.<br>
크롤링한 데이터를 기반으로 주문자 정보 더미 데이터를 만드는 과정을 ChatGPT를 통해 진행했다.


### 1. 데이터 파이프라인 구축 
MySQL 데이터베이스에서 데이터를 추출하여 Logstash를 통해 Elasticsearch로 전송하는 데이터 파이프라인을 구축하는 방법을 실습함
<br>

![image](https://github.com/user-attachments/assets/cfb9084e-3dfe-416e-8784-711cefbd065f)


### 2. 데이터 처리

DB의 테이블 데이터를 수집, 변환, 필터링하고 Elasticsearch에 적합하게 저장함
<br>

![image](https://github.com/user-attachments/assets/3c889787-f50e-4981-ba3a-3bf68abe4c28)


### 3. Elasticsearch 활용

Elasticsearch를 활용해 빠른 검색과 분석을 수행하며, 특히 Kibana를 이용한 데이터 시각화로 분석 효율성을 극대화함.
<br>

![image](https://github.com/user-attachments/assets/b8a56d4a-615c-482e-9e9e-9270f546285c)

<br>

<br>

## JDBC를 통한 Logstash와 Elastic Search 연동


```
input {
  jdbc {
    jdbc_driver_library => "D:/woorifisa/05.ELK/logstash-7.11.1-windows-x86_64/logstash-7.11.1/logstash-core/lib/jars/mysql-connector-j-9.1.0.jar"
    jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:8888/musinsa?useSSL=false&characterEncoding=utf8&serverTimezone=Asia/Seoul"
    jdbc_user => "user01"
    jdbc_password => "user01"
    statement => "SELECT * FROM `customerorder` WHERE updated_at > :sql_last_value"
    schedule => "*/5 * * * * *"  # 5초마다 실행
  }
}
```


##  데이터 필터링 


```
filter {
  # category 필드를 > 기호로 분리
  mutate {
    split => { "category" => ">" }
  }
  # 상의 카테고리와 하의 카테고리로 매핑
  mutate {
    add_field => {
      "top_category" => "%{[category][0]}"
      "sub_category" => "%{[category][1]}"
    }
  }
  # address 필드를 띄어쓰기로 분리
  mutate {
    split => { "address" => " " }
  }
  # city와 state를 분리하는 Ruby 스크립트
  ruby {
    code => '
      if event.get("address")
        parts = event.get("address")
        if parts.length >= 2
          event.set("city", parts[0])
          event.set("state", parts[1])
        end
      end
    '
  }
}
```
<br><br><br><br>


데이터 가공을 위해 먼저 카테고리 필터링과 주문자 주소 정보 필터링을 진행했다.








```
filter{
  # order_date 필드를 날짜(date)와 시간(time)으로 분리
  ruby {
    code => '
      if event.get("order_date")
        order_date = event.get("order_date").to_s
        if order_date.include?("T")
          date_time = order_date.split("T")
          if date_time.length == 2
            event.set("date", date_time[0])  # 날짜 부분
            event.set("time", date_time[1].gsub("Z", ""))  # 시간 부분 (Z 제거)
          else
            event.tag("_order_date_split_failed")
          end
        else
          event.tag("_order_date_format_invalid")
        end
      else
        event.tag("_order_date_missing")
      end
    '
  }
}
```


## Kibana를 통한 시각화




## 무신사 검색 정렬 알고리즘 톺아보기


- 무신사의 검색 알고리즘의 문제점 <br>


![image](https://github.com/user-attachments/assets/34fb0a63-4553-470c-b2fd-c3f6f3274560)
<br>


- 적합도 반영 전략


![image](https://github.com/user-attachments/assets/52c41069-5728-433f-b088-b553cf1ccf9d)


### 스코어링 전략


<br>

무신사에서 "후드"를 검색하게 되면 후드 집업이 아닌 후드티 상품이 상위 데이터로 검색되는 화면을 볼 수 있다.


![image](https://github.com/user-attachments/assets/edbaf461-23d6-4621-a513-47c4ac83d638)


이에 반해 "후드 집업"을 검색하게 되면 결과 화면에는 후드티가 아닌 후드 집업 상품이 상위 데이터로 나타난다.<br>
그러나 "후드 집업" 혹은 "후드"를 검색하면 무조건 해당 제품만 검색되는 것은 아니다.<br>


![image](https://github.com/user-attachments/assets/da52d143-ed2e-4ac0-901a-2ca062e8139a)


해당 결과 화면에서는 후드 집업을 검색한 결과 중 상위 데이터의 일부는 후드 집업과 관련이 없는 상품이 존재하는 것을 볼 수 있다.


![image](https://github.com/user-attachments/assets/2347fec6-ed6f-4edb-b045-54d90b25b0b0)


또한 후드를 검색했을 때 모자가 달린 패딩과 같은 관련성이 적은 아우터가 검색 결과에 포함된 것을 볼 수 있다.<br>
이는 검색 엔진에서 단순히 키워드를 통한 결과를 반환하는 것이 아님을 알 수 있다.




![image](https://github.com/user-attachments/assets/b49d9b6a-4445-443b-aab0-7bb81a9072d5)![image](https://github.com/user-attachments/assets/7cd07175-7b4c-4421-bb77-d6985f550580)


스코어링 수식에서 카테고리와 브랜드의 가중치 또한 포함되는 것을 알 수 있었고 지퍼가 없는 "후드"를 원하는 사용자가 후드라는 단어를 검색하면 "후드 티셔츠" 카테고리의 상품이 상위에 나오게 된다.


이에 반해 "후드 집업"을 검색하게 되면 후드 집업 카테고리에 속한 제품들이 나오게 된다.


이를 통해 결과 화면에 단순히 키워드에 대한 score 전략으로는 충분한 데이터 필터링이 되지 않기 때문에 카테고리의 가중치가 더 클 것이라 추론했다.


현재 사용 중인 Elastic Search에서 브랜드 관련 Field를 보유하고 있지만 이에 대한 가중치 기준을 세우는데 어려움이 존재한다.


이로 인해 좀 더 확실한 기준을 세울 수 있는 카테고리를 상품과 함께 연동되어 가중치 계산이 되도록 진행했다.<br><br>



![image](https://github.com/user-attachments/assets/5a36be55-484e-4436-8f1e-7234326a030b)


"후드"라는 키워드를 검색하게 되면 후드 집업이라는 단순히 후드라는 상품명을 가진 제품이 더 상위에 포진해야 된다.


그러나 현재 쿼리를 통한 검색 결과를 보게 되면 후드라는 단어만 상품명에 포함한 제품과 후드 집업이라는 단어를 상품명에 포함한 제품의 score가 동일한 것을 볼 수 있다.


이는 BM25 알고리즘에서 하나의 document 내의 단어 개수 중 검색 키워드 출현 개수 비율을 반영하고 있으며 두 개의 데이터 모두 3개의 단어와 하나의 "후드" 단어를 포함하고 있기 때문이라 예상해볼 수 있었다.


```
POST /musinsa/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "product_name": "후드"
        }
      },
      "functions": [
        {
          "filter": {
            "term": {
              "sub_category.keyword": "후드 티셔츠"
            }
          },
          "weight": 3.0  // 카테고리: 상의 > 후드 티셔츠
        },
        {
          "filter": {
            "term": {
              "sub_category.keyword": "후드 집업"
            }
          },
          "weight": 0.5  // 카테고리: 아우터 > 후드 집업
        },
        {
          "filter": {
            "match_phrase": {
              "product_name": "후드"
            }
          },
          "weight": 2.0  // 상품명: "후드"
        }
      ],
      "score_mode": "sum",  // 가중치를 단순 합산
      "boost_mode": "sum"  // 기본 BM25 스코어와 가중치를 합산
    }
  }
}
```



1차적으로 진행한 가중치 개선에서 "후드"라는 단어와 함께 하위 카테고리의 가중치를 조정했고 후드티의 score가 대체적으로 후드 집업 상품에 비해 높아진 것을 확인할 수 있었다.


```
        "_index" : "musinsa",
        "_type" : "_doc",
        "_id" : "111",
        "_score" : 3.4028943,
        "_source" : {
          "category" : [
            "아우터 ",
            " 후드 집업"
          ],
          "top_category" : "아우터 ",
          "@timestamp" : "2025-01-21T16:56:20.123Z",
          "product_name" : "투웨이 후드 집업",
          "sub_category" : " 후드 집업",
        }
      },
      {
        "_index" : "musinsa",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 3.4028943,
        "_source" : {
          "category" : [
            "상의 ",
            " 후드 티셔츠"
          ],
          "top_category" : "상의 ",
          "@timestamp" : "2025-01-21T16:56:20.122Z",
          "product_name" : "미니멀 오버핏 후드",
          "sub_category" : " 후드 티셔츠",
        }
      },
```


그러나 문장에 존재하는 단어의 개수가 같은 경우에 후드 집업과 후드티의 score가 동일한 결과를 일부 확인할 수 있었다.


이를 좀 더 개선하기 위해 상위 카테고리와 하위 카테고리 모두 가중치를 조정했다. 무신사의 카테고리 분류에서는 후드 집업은 아우터로, 후드 티셔츠는 상의로 분류하고 있는 점을 고려해 조정했다.


```
POST /musinsa/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "product_name": "후드"
        }
      },
      "functions": [
        {
          "filter": {
            "term": {
              "top_category": "상의"
            }
          },
          "weight": 3.0  // "상의" 카테고리 가중치
        },
        {
          "filter": {
            "term": {
              "top_category": "아우터"
            }
          },
          "weight": 0.5  // "아우터" 카테고리 가중치
        },
        {
          "filter": {
            "term": {
              "sub_category": "후드 티셔츠"
            }
          },
          "weight": 2.0  // "후드 티셔츠" 하위 카테고리 가중치
        },
        {
          "filter": {
            "term": {
              "sub_category": "후드 집업"
            }
          },
          "weight": 0.5  // "후드 집업" 하위 카테고리 가중치
        },
        {
          "filter": {
            "match_phrase": {
              "product_name": "후드"
            }
          },
          "weight": 1.5  // "후드" 포함 가중치
        },
        {
          "filter": {
            "match_phrase": {
              "product_name": "후드 집업"
            }
          },
          "weight": 0.5  // "후드 집업" 포함 가중치 감소
        }
      ],
      "score_mode": "sum",  // 점수 합산
      "boost_mode": "sum"  // 기본 BM25와 합산
    }
  }
}
```


이를 통해 확실한 score 차이를 확인할 수 있었다.


```
        "_index" : "musinsa",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 5.902894,
        "_source" : {
          "category" : [
            "상의 ",
            " 후드 티셔츠"
          ],
          "top_category" : "상의 ",
          "product_name" : "그래픽 베이직 후드",
          "sub_category" : " 후드 티셔츠",
        }
      },
      {
        "_index" : "musinsa",
        "_type" : "_doc",
        "_id" : "111",
        "_score" : 3.9028943,
        "_source" : {
          "category" : [
            "아우터 ",
            " 후드 집업"
          ],
          "top_category" : "아우터 ",
          "product_name" : "투웨이 후드 집업",
          "sub_category" : " 후드 집업"
        }
```


### Nori를 사용한 Analyze




## 트러블 슈팅


### Timezone 세팅 문제


기본적으로 Logstash는 UTC 시간대를 표준으로 사용하고 있으며 이로 인해 데이터 삽입 과정에서 KST로 설정된 시간과 9시간 차이나는 값으로 삽입이 이루어짐.


### DB connection 문제
![image](https://github.com/user-attachments/assets/bb6100ea-93e9-41ac-9090-bd1b89d9e9c3)


원격 데이터베이스에 접근하지 못하는 문제가 발생했다.


```
set global max_connect_errors = 10000;
set global max_connections=300;
```

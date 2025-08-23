# ELK를 활용한 쇼핑몰 데이터 분석 및 검색 최적화

<br>

ELK 스택을 활용해 온라인 쇼핑몰(MUSINSA)의 주문 데이터를 처리하고 **검색 품질을 개선**하는 과정을 구현했습니다. <br>

저희는 ElasticSearch의 대표적인 활용 분야 중 ▲데이터 수집 및 분석과 ▲추천 알고리즘 개선에 주목했습니다. <br>

이를 기반으로 고객 구매 내역 데이터를 분석하고 **검색 알고리즘을 최적화**하는 방법을 탐구했습니다. <br>

또한 Kibana를 통해 고객 구매 패턴, 매출 현황, 브랜드별 인기도 등을 직관적으로 파악할 수 있는 **시각화 대시보드**를 구축했습니다. <br>

<br>

## ◈ 팀원 소개

<br>

|<img src="https://avatars.githubusercontent.com/u/193798531?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/74342019?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/153366521?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/127267532?v=4" width="150" height="150"/>|
|:-:|:-:|:-:|:-:|
|[@riyeong0916](https://github.com/riyeong0916)|[@CooolRyan](https://github.com/CooolRyan)|[@parkjhhh](https://github.com/parkjhhh)|[@HyeJinSeok](https://github.com/HyeJinSeok)|

<br>

---

<br>

## 1. 데이터 수집 및 전처리

<br>

• 온라인 커머스 플랫폼 '무신사(MUSINSA)'는 robots.txt를 통해 일부 크롤링을 허용하고 있음 <br>

• 상품 데이터를 직접 크롤링하는 데 시간이 많이 소요되므로 [해시스크래퍼](https://www.hashscraper.com/)의 무료 크롤링 기능을 활용함 <br>

• 테이블 스키마 설계를 위해 Kaggle의 [Amazon Seller - Order Status Prediction](https://www.kaggle.com/datasets/pranalibose/amazon-seller-order-status-prediction) 데이터셋을 참고함 <br>

• 프로젝트 목적에 맞게 데이터를 전처리하고 ChatGPT로 **주문자 더미 데이터**를 생성함 <br>

<br><br>

### 📌해시스크래퍼로 크롤링한 무신사 상품 테이블


<table>
  <tr>
    <td>카테고리</td>
    <td>정렬기준</td>
    <td>상품명</td>
    <td>브랜드</td>
    <td>품번</td>
    <td>판매가</td>
  </tr>
  <tr>
    <td>아우터 > 후드 집업(인터크루)</td>
    <td>무신사 추천순</td>
    <td>[23FW] 우먼스 융기모 세미크롭 후드집업 비바마젠타 ITX4DH54AVM</td>
    <td>인터크루</td>
    <td>5007029826</td>
    <td>69,900</td>
  </tr>
  <tr>
    <td>아우터 > 후드 집업(아임낫어휴먼비잉)</td>
    <td>무신사 추천순</td>
    <td>Basic Logo Zip-up Hoodie - ROYAL BLUE</td>
    <td>아임낫어휴먼비잉</td>
    <td>hbre206</td>
    <td>69,000</td>
  </tr>
</table>

<br>

### 📌Amazon Seller 스키마


<table>
  <tr>
    <td>Order No</td>
    <td>Order Date</td>
    <td>Buyer</td>
    <td>Ship City</td>
    <td>Ship State</td>
    <td>SKU</td>
    <td>Description</td>
    <td>Quantity</td>
    <td>Item Total</td>
    <td>Shipping Fee</td>
    <td>COD</td>
    <td>Order Status</td>
  </tr>
</table>

<br>

### 📌고객 주문을 관리하는 customer_order 테이블 정의

```
CREATE TABLE customer_order (

  order_no      INT PRIMARY KEY,
  order_date    DATETIME,
  buyer         VARCHAR(10),
  gender        ENUM('남성', '여성'),
  address       VARCHAR(50),
  category      VARCHAR(50),
  brand         VARCHAR(30),
  product_name  VARCHAR(100),
  price         BIGINT,
  order_status  ENUM('구매확정', '환불', '교환', '배송준비중'),
  updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP

);
```

<br>

### 📌customer_order 테이블 예시

![alt text](/images/table.png)

<br><br>

---

<br>

## 2. Logstash: Data Ingestion

<br>

• Logstash의 파이프라인은 **input → filter → output** 구조임 (.conf 파일)

• **원천 DB**에서 데이터를 주기적으로 끌어와, 필드를 전처리한 뒤 Elasticsearch에 **문서로 색인**하는 과정 <br>

> − Input 단계 : 원천 MySQL에서 데이터 끌어옴 (JDBC) <br>
> − Filter 단계 : 끌어온 데이터를 전처리 및 가공 <br>
> − Output 단계 : Elasticsearch에 색인 <br>

<br>

### ① JDBC 입력 설정


• 원격 MySQL에서 customer_order 테이블을 읽어오도록 **input { jdbc { ... } }** 구성함 <br>

• 연결 정보 및 드라이버를 지정함 <br>

• **:sql_last_value** 기준으로 신규/변경분만 가져오도록 구성함 <br>

```
input {

  jdbc {
    jdbc_driver_library => "D:/woorifisa/.../mysql-connector-j-9.1.0.jar"
    jdbc_driver_class   => "com.mysql.cj.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:8888/musinsa?useSSL=false&characterEncoding=utf8&serverTimezone=Asia/Seoul"
    jdbc_user => "user01"
    jdbc_password => "user01"
    statement  => "SELECT * FROM customer_order WHERE updated_at > :sql_last_value"
    schedule   => "*/5 * * * * *"   # 5초 주기
  }

}
```

<br>

### ② 데이터 필터링 


• category를 **> 기준으로 분해**해 top_category, sub_category 생성함 <br>

• address를 **공백 기준으로 분해**해 city, state 추출함 <br>

• order_date에서 날짜/시간을 분리 저장함 <br>

![image](https://github.com/user-attachments/assets/43487d17-d2c1-4afd-8050-f88eaf3e48a6)![image](https://github.com/user-attachments/assets/f80c44cd-c2c1-4f73-989b-c7c9abeffb95)
![image](https://github.com/user-attachments/assets/9967ac39-d6ae-46f8-8555-ee0efa7ef456)

```
filter {

  # 카테고리 분해 및 매핑
  mutate { split => { "category" => ">" } }
  mutate {
    add_field => {
      "top_category" => "%{[category][0]}"
      "sub_category" => "%{[category][1]}"
    }
  }
  mutate {
    strip => ["top_category","sub_category"]
    gsub  => [
      "top_category", "^\s+|\s+$", "",
      "sub_category", "^\s+|\s+$", ""
    ]
  }
```

<details>
  <summary>filter 코드 이어서 보기</summary>
  
<br>

```conf
  # 주소 파싱
  mutate { split => { "address" => " " } }
  ruby {
    code => '
      parts = event.get("address")
      if parts && parts.length >= 2
        event.set("city",  parts[0])
        event.set("state", parts[1])
      end
    '
  }

  # 주문일시 분해
  ruby {
    code => '
      if event.get("order_date")
        ts = event.get("order_date").to_s
        if ts.include?("T")
          d, t = ts.split("T", 2)
          if t
            event.set("date", d)
            event.set("time", t.gsub("Z",""))
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
</details>

<br>

### ③ Elasticsearch 색인


• 전처리된 이벤트를 Elasticsearch **인덱스 musinsa**로 색인함 <br>

• 중복 방지를 위해 **document_id = order_no**로 고정해 idempotent하게 갱신되도록 함

```
output {

  elasticsearch {
    hosts       => ["http://localhost:9200"]
    index       => "musinsa"
    document_id => "%{order_no}"
    action      => "index"
  }
  stdout { codec => rubydebug }

}
```

<br><br>

---

<br>

## 3. Kibana를 통한 시각화

<br>

• 상품 구매 내역 데이터를 기반으로 모니터링 대시보드를 구축함 <br>

• Kibana에서 **musinsa 인덱스**를 연결해 Lens로 그래프를 만듦 <br>

<br>

### < 일별 주문 건수 및 매출 >

<img src="https://github.com/user-attachments/assets/d3c8c4cf-cb32-4866-98fa-741a416e1267" alt="Image" style="width:70%;">

<br>

### < 2025년 주문 현황 >

<img src="https://github.com/user-attachments/assets/cdae1158-01dc-4726-b1ab-b7cfabe91b13" alt="Image" style="width:50%;">

<br>

### < 2025년 일일 매출 >

<img src="https://github.com/user-attachments/assets/69eb93f4-70f8-4a1a-8573-0a09876221b6" alt="Image" style="width:50%;">

<br>

### < 우수회원 >

<img src="https://github.com/user-attachments/assets/243ff5c7-ea8d-450b-9b1a-4c70020680c2" alt="Image" style="width:50%;">

<br>

### < 브랜드별 성별 인기도 >

<img src="https://github.com/user-attachments/assets/cdee7b43-a5fc-4f04-8810-8c6d3237b87b" alt="Image" style="width:50%;">

<br><br>

---

<br>

## 4. 기존 검색 알고리즘의 문제점

<br>

• 무신사 사이트는 **키워드 기반(Best Match 25) 검색 + 동의어 사전** 조합으로 매칭을 수행함 <br>

• 그러나 이는 카테고리 및 의도 신호 부재로, **텍스트 일치도만 높은 결과**가 의도와 무관하게 상위로 노출될 가능성이 있음 <br>

• 예) ‘삭스’가 들어간 **부츠 카테고리 상품** (=삭스 부츠)도 양말 질의와 텍스트 일치도가 높아져 **상단에 노출**되는 상황 <br>

<img src="https://github.com/user-attachments/assets/34fb0a63-4553-470c-b2fd-c3f6f3274560" alt="검색 문제 도해" width="600">

<br>

> [!NOTE]
> #### - 문제점 관찰 방법 <br>
> - “후드” 검색 ⇒ 후드티가 상단에 많이 노출됨 <br>
> - “후드 집업” 검색 ⇒ 후드 집업이 상단이지만 비의도 상품이 일부 끼어듦 <br>
> - “후드” 검색에서 후드 달린 패딩 등 관련도 낮은 아우터가 상단에 뜨는 경우가 있음 <br><br>
> #### - 원인 분석 <br>
> - BM25는 텍스트 일치도 중심이라 카테고리 의도를 반영하지 못함 <br>
> -  “후드” vs “후드 집업”처럼 단어 수 및 구성이 비슷하면 BM25 점수도 비슷해 구분력 떨어짐 <br>
> -  brand 필드는 존재하나 일관된 가중치 기준 잡기 어려워 노이즈 가능 <br><br>
> #### - 결론 <br>
> - 단순 키워드 점수만으론 부족하기 때문에 랭킹에서 카테고리 가중치를 최우선으로 둬야 함 <br>
> - “후드” 질의는 상의/후드 티셔츠 우선, “후드 집업” 질의는 아우터/후드 집업 우선으로 정렬되어야 함

<br>

### < ‘후드’ 질의에 따른 Elasticsearch hits 일부 >


![image](https://github.com/user-attachments/assets/5a36be55-484e-4436-8f1e-7234326a030b)


− ‘후드’ 검색 시, 상품명에 ‘후드’만 있는 상품과 ‘후드 집업’이 함께 상단에 노출됨 <br>

− 이때 두 결과의 랭킹 점수(_score)가 1.6701219로 거의 동일해 우선순위 구분이 어려움 <br>

<br>

---

<br>

## 5. 의도 기반 랭킹 시스템 설계

<br>

• 검색 결과가 **카테고리 및 구문 신호** 중심으로 재정렬되는 의도 기반의 랭킹이 필요함 <br>

• 필드 가중치 우선순위를 "**category > product_name > brand**"로 둠으로써 의도 카테고리 일치도와 정렬 안정성을 높임 <br>

• 질의어가 제품명의 문구와 정확히 일치하면 match_phrase로 **추가 가중치**를 부여함 <br>

• 비의도 카테고리에는 **페널티**를 적용해 점수를 낮춤 <br>

• 동의어 확장은 유지하되, **최종 랭킹**은 카테고리 신호 중심으로 재정렬함

<br>

![image](https://github.com/user-attachments/assets/52c41069-5728-433f-b088-b553cf1ccf9d)

<br>

### 🎯Scoring 전략 (1) : function_score로 가중치 합산하기

```

// 🔸top_category : 1차 분류(의도 카테고리) → 가장 큰 weight
// 🔸sub_category : 2차 세부 분류(후드 티셔츠 vs 후드 집업) → 보조 weight
// 🔸product_name : 구문 일치 보정(match_phrase)

// 🔸스코어 결합 : score_mode=sum, boost_mode=sum → 최종점수 = BM25 + (모든 weight의 합)


POST /musinsa/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "product_name": "후드" } },
      "functions": [
        { "filter": { "term": { "top_category": "상의" } },        "weight": 3.0 },
        { "filter": { "term": { "top_category": "아우터" } },       "weight": 0.5 },
        { "filter": { "term": { "sub_category": "후드 티셔츠" } },   "weight": 2.0 },
        { "filter": { "term": { "sub_category": "후드 집업" } },     "weight": 0.5 },
        { "filter": { "match_phrase": { "product_name": "후드" } },  "weight": 1.5 },
        { "filter": { "match_phrase": { "product_name": "후드 집업" } }, "weight": 0.5 }
      ],
      "score_mode": "sum",
      "boost_mode": "sum"
    }
  }
}
```

<br>

### 수행 결과

<br>

### 🎯Scoring 전략 (2) : script_score로 BM25에 직접 보정하기


```

// 🔸top_category : 상의 +3.0, 아우터 +0.5 (1차 분류, 가장 큰 영향)
// 🔸sub_category : 후드 티셔츠 +2.0, 후드 집업 +0.5 (2차 분류, 보조 영향)

// 🔸스코어 결합 : script_score → 최종점수 = BM25(_score) + 카테고리 가중치(직접 가산)


POST /musinsa/_search
{
  "query": {
    "script_score": {
      "query": { "match": { "product_name": "후드" } },
      "script": {
        "source": """
          double score = _score;
          if (doc['top_category'].value == '상의')          score += 3.0;
          else if (doc['top_category'].value == '아우터')    score += 0.5;
          if (doc['sub_category'].value == '후드 티셔츠')     score += 2.0;
          else if (doc['sub_category'].value == '후드 집업') score += 0.5;
          return score;
        """
      }
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
<br>


BM25 값을 직접 조정하게 되면서 score 값 또한 더 확실한 차이를 만들어낼 수 있었다.


```
      {
        "_index" : "musinsa",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 4.402894,
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
        "_score" : 1.9028943,
        "_source" : {
          "category" : [
            "아우터 ",
            " 후드 집업"
          ],
          "top_category" : "아우터 ",
          "product_name" : "투웨이 후드 집업",
          "sub_category" : " 후드 집업",
      },
```
<br>

---

<br>

## Nori를 사용한 Analyze


![image](https://github.com/user-attachments/assets/e0311c39-5271-45bb-97fa-706c58f60b09)
<br><br>


무신사의 검색 서비스 프로토타입에서 형태소 분석을 위한 토크나이저로 Nori(Lucene의 한글 형태소 분석기)의 파이썬 버전 Pynori를 사용했다.


이를 통해 동의어 및 사용자 사전을 커스텀하게 적용할 수 있기 때문에, 기존에 무신사 검색 서비스에서 사용 중이던 사전을 그대로 적용하여 최대한 실제 서비스와 비슷한 시뮬레이션 환경을 구현해냈다.


여기서 착안해 elastice search에서 제공하는 nori를 통한 상품 데이터 분석을 진행해보고자 했다.


```
elasticsearch-plugin install analysis-nori
```


먼저 nori를 사용하기 위한 plugin 설치를 진행한다.


```
PUT /musinsa_with_nori
{
  "settings": {
    "analysis": {
      "analyzer": {
        "nori_analyzer": {
          "type": "custom",
          "tokenizer": "nori_tokenizer",
          "filter": ["lowercase", "nori_readingform", "nori_part_of_speech"]
        }
      },
      "filter": {
        "nori_part_of_speech": {
          "type": "nori_part_of_speech",
          "stoptags": ["E", "IC", "J", "MAG", "MAJ", "MM", "SP", "SSC", "SSO", "SY", "UNA", "UNKNOWN", "VA", "VCN", "VCP", "VV", "VX", "XPN", "XR", "XSA", "XSN", "XSV"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "product_name": {
        "type": "text",
        "analyzer": "nori_analyzer",
        "fielddata": true  // fielddata 활성화
      },
      "category": {
        "type": "text",
        "analyzer": "nori_analyzer",
        "fielddata": true
      },
      "sub_category": {
        "type": "text",
        "analyzer": "nori_analyzer",
        "fielddata": true
      },
      "top_category": {
        "type": "text",
        "analyzer": "nori_analyzer",
        "fielddata": true
      },
      "price": { "type": "long" },
      "brand": { "type": "keyword" },
      "updated_at": { "type": "date" }
    }
  }
}
```


이 후 nori를 사용하기 위한 새로운 인덱스를 생성한다. 기존에 생성된 인덱스를 수정하는 것이 불가능 하기 때문에 새로운 인덱스를 생성하고 기존 인덱스의 데이터를 이관하는 작업을 진행했다.


```
POST /_reindex
{
  "source": {
    "index": "musinsa"
  },
  "dest": {
    "index": "musinsa_with_nori"
  }
}
```


재인덱싱 과정 이후 aggregation function을 통해 search를 진행하게 되면 형태소 분석을 통한 파싱이 진행된다.


```
GET /musinsa_with_nori/_search
{
  "size": 0,
  "aggs": {
    "product_name_count": {
      "terms": {
        "field": "product_name",
        "size": 10
      }
    }
  }
}
```


- 결과 화면 비교 및 개선




기존의 aggregation function을 통한 count를 진행하게 되면 product_name의 전체 value를 기준으로 count 되는 모습을 보여주었다.


![image](https://github.com/user-attachments/assets/2fe5532c-f622-4977-a5f4-149054f6efce)
<br><br>


그러나 field에 nori 사용을 위한 analyze를 추가한 뒤 aggregation function을 사용하면 형태소 별 분리가 된 결과를 볼 수 있다.


![image](https://github.com/user-attachments/assets/047958a7-bf11-47cc-b810-6bcd9f07122d)
<br><br>


이 때 "후드 집업" 이라는 단어가 "후드", "집", "업"으로 분리되는 것을 확인할 수 있다.


이는 nori analyze에서 제공하는 복합어 처리 과정에서 단어가 분리되는 것이다. 그러나 이런 형태로 데이터 분석을 진행하게 된다면 정확한 분석이 불가능할 것이라 생각했다.


```
"analysis": {
  "tokenizer": {
    "nori_tokenizer": {
      "type": "nori_tokenizer",
      "decompound_mode": "mixed"  // 복합어 및 원래 단어
    }
  }
```


이를 해결하기 위해 인덱스 생성 시 decompound_mode 설정을 mix로 설정해 복합어 분리가 되지 않은 데이터 또한 같이 확인할 수 있었다.


```
     "tokenizer": {
        "nori_tokenizer": {
          "type": "nori_tokenizer",
          "user_dictionary": "analysis/userdict_ko.txt"
        }
      }
```
사용자 정의 사전을 포함한 nori 활용을 위해서는 위의 코드와 같이 user_dictionary를 포함해서 사용할 수 있다.



## 트러블 슈팅


### Timezone 세팅 및 새로운 정보 update 문제


기본적으로 Logstash는 UTC 시간대를 표준으로 사용하고 있으며 이로 인해 데이터 삽입 과정에서 KST로 설정된 시간과 9시간 차이나는 값으로 삽입이 이루어진다.


Elastic Search 또한 마찬가지였고 이로 인해 시각이 시스템 상의 시간과 9시간 차이 나게 저장되는 것을 확인할 수 있었다.


```
SELECT * FROM `customerorder` WHERE updated_at > :sql_last_value
```


updated_at을 기준으로 새로 추가된 데이터를 ES 상에 추가하기 때문에 시간을 정상적으로 인식하지 못하게 되었다.


```
filter{ 
            ... 
            date{ 
              match => ["insert_ts","YYYYMMddHHmmss"] 
              timezone => "Asia/Seoul" <- 이부분
            } 
            ... 
      }
```


이를 해결하기 위해 filter에서 date 설정을 진행했으나 반영되지 않는 것을 확인했다.


데이터베이스의 order_no과 document의 _id 값을 매핑이 되도록 해 덮어쓰기를 진행하는 과정을 통해 데이터가 지속적으로 쌓이지 않도록 했다.


그러나 이를 반영하지 못하는 팀원의 케이스도 발생했으며 이로 인해 5초마다 업데이트 될때마다 새로운 정보를 업데이트 하는 것이 아닌 처음부터 모든 데이터를 업데이트 하게 되어 데이터가 116,110까지 쌓이게 되고, 비효율적으로 업데이트를 하게되었다.




<br>

<p align="center">
  <img src="https://github.com/user-attachments/assets/44b5a3f4-65c9-40c1-97c2-e628a113444d" alt="트러블슈팅-before">
</p>

<br>


시간 설정에 대해 찾아보던 중 **ElasticSearch의 기존 timezone은 UTC 기준이다 색인도 UTC 기준으로 하길 권고 하고 바꾸지 않기를 권고하고 있다**는 것을 알게 되었다.


이를 수정하기 위해 order_no 칼럼을 기준으로 새로운 데이터가 업데이트 되도록 설정했다.  
또한 마지막으로 가져온 정보를 `C:\\00.dataSet\\logstash_jdbc_last_run` 경로에 저장하여 Logstash를 실행하면 이 파일을 읽고 마지막으로 실행된 `order_no`를 확인하고 새로운 데이터를 업데이트했다.

```ruby
use_column_value => true
tracking_column => "order_no"
tracking_column_type => "numeric"
last_run_metadata_path => "C:\\00.dataSet\\logstash_jdbc_last_run"
schedule => "*/5 * * * * *"  # 5초마다 실행
```

가장 마지막 order_no 내용만 업데이트 되고 있음을 확인했다.
<br>

![트러블슈팅-after](https://github.com/user-attachments/assets/a7f799ee-6ea8-47cc-af2a-7773d0c1912f)

<br>

### DB connection 문제
![image](https://github.com/user-attachments/assets/bb6100ea-93e9-41ac-9090-bd1b89d9e9c3)


원격 데이터베이스에 접근하지 못하는 문제가 발생했다.


원격 서버에서 MySQL 서버로 연결한 뒤 close를 하면 MySQL은 비정상적인 접속으로 판단해 해당 IP를 블락 처리한다고 한다.

이 때 MySQL에서 이와 같은 비정상적인 접속 요청 수를 카운팅해서 global.max_connect_errors에 지정된 값을 넘기면 자동 블락 처리가 되어 생긴 이슈다.


```
set global max_connect_errors = 10000;
set global max_connections=300;
flush hosts;
```


위의 명령어를 통해 max_connect_errors 값을 증가시키고 그 동안의 접속 요청 기록을 초기화 해주는 과정을 통해 정상적으로 원격 데이터베이스에 다시 접속할 수 있었다.

<br>

### Elastic Search의 text와 keyword


- Elasticsearch 필드 타입: `keyword` vs `text`


| **특징**                | **`keyword`**                                          | **`text`**                                              |
|------------------------|-----------------------------------------------------|-------------------------------------------------------|
| **용도**               | 정렬, 집계, 필터링에 적합 (정확히 일치하는 값을 비교).       | 검색, 분석, 자연어 처리를 위해 적합.                        |
| **저장 방식**            | 전체 문자열이 인덱싱되고, 분석되지 않음.                     | 텍스트를 분석(토크나이징)하여 여러 토큰으로 저장.               |
| **토크나이저**            | 없음 (문자열 전체를 하나의 값으로 저장).                   | 텍스트를 공백, 구두점 등을 기준으로 나누어 여러 토큰으로 저장.     |
| **검색 동작**            | 정확히 일치하는 값만 검색 가능.                              | 텍스트의 부분적 일치(예: "후드" 포함) 검색 가능.               |
| **Aggregation 지원 여부** | 지원 (집계와 정렬 가능).                                    | 기본적으로 지원되지 않음 (집계를 위해선 `fielddata` 활성화 필요). |
| **데이터 크기**           | 저장 크기가 작음 (원본 값만 저장).                          | 토큰화된 데이터를 추가로 저장하므로 저장 크기가 더 큼.             |

---

<br>

### **문제**
`text` 필드를 Aggregation에 사용하려고 하면 다음과 같은 오류가 발생한다:

```json
{
  "error": {
    "type": "illegal_argument_exception",
    "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [field_name] in order to load field data by uninverting the inverted index."
  }
}
```


text 필드가 검색과 분석을 위해 설계되어 있으며, Aggregation처럼 정확한 값을 요구하는 작업을 지원하지 않기 때문이다.


또한 keyword에 analyzer를 설정할 수 없었다.


keyword 타입은 데이터를 토큰화하거나 분석하지 않는다.


반면 **analyzer**는 텍스트를 분리하고 형태소 분석을 수행하도록 설계되었다.


두 개념이 충돌하므로 keyword 타입에서 analyzer를 설정할 수 없다.


- 해결책


Aggregation에서 text 활용법


```
PUT /index_name/_mapping
{
  "properties": {
    "field_name": {
      "type": "text",
      "fielddata": true
    }
  }
}
```


field에 대한 fielddata를 true로 설정해주는 과정을 통해 text 필드를 Aggregation function에 사용할 수 있다.

<br>

## 회고


석혜진 : ElasticSearch와 MySQL을 연결하기 위해 Logstash에 JDBC 설정을 구성하는 프로세스를 이해하게 되었다. 이후 Kibana를 활용하여 데이터를 시각화하는 과정에서 Logstash의 conf 파일에 필터를 구성하고, KQL 명령어를 익히는 실습을 수행했다. 추후에는 대용량 데이터를 삽입하고 효율적으로 전처리하는 과정을 실습해볼 계획이다.


박지혜 : ElasticSearch와 Logstash, Kibana를 통해 실생활에 사용되는 데이터를 로그하고 모니터링 해보았다. 원격 데이터베이스를 연결하는 과정에서 JDBC Driver의 버전이 달라서 연결이 정상적으로 이루어지지 않는 이슈가 발생하긴 했지만 다시 Driver을 8버전으로 설치하고 logstash를 실행시켜보니 연결이 정상적으로 되었다. 이런 DB연결 문제, 예를 들어 버전이 달라서 생기는 문제를 더 자세히 학습하고 보다 더 순조롭게 프로젝트를 진행할 수 있도록 해야겠다.


나원호 : 기존 추천 알고리즘은 elastic search로 기반으로 이루어진다는 정보를 알고 있었지만 이에 대해 정확하게 어떤 식으로 구현되는지 궁금증을 가졌다. 이번 기회를 통해 score 가중치를 조정하며 사용자를 고려한 서비스를 구현하는 방식에 대해 알아볼 수 있었고 단어를 필터링하는 과정에서 사용할 수 있는 nori의 방식을 알 수 있었다.

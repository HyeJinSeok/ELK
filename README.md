# ELK를 활용한 쇼핑몰 데이터 분석 및 검색 최적화

<br>
  
ELK 스택을 활용해 온라인 쇼핑몰(MUSINSA)의 주문 데이터를 처리하고 **검색 품질을 개선**하는 과정을 구현했습니다. <br>

저희는 ElasticSearch의 대표적인 활용 분야 중 ▲데이터 수집 및 분석과 ▲추천 알고리즘 개선에 주목했습니다. <br>

이를 기반으로 고객 구매 내역 데이터를 분석하고 **검색 알고리즘을 최적화**하는 방법을 탐구했습니다. <br>

또한 Kibana를 통해 고객 구매 패턴, 매출 현황, 브랜드별 인기도 등을 파악할 수 있는 **시각화 대시보드**를 구축했습니다. <br><br>


<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk1.png" width=700 alt="ELK 다이어그램">

<br>

**• 프로젝트 기간** <br>

📆 2025.01.14 ~ 01.20

<br>

**• 팀원 소개**


|<img src="https://avatars.githubusercontent.com/u/193798531?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/74342019?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/153366521?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/127267532?v=4" width="150" height="150"/>|
|:-:|:-:|:-:|:-:|
|[@riyeong0916](https://github.com/riyeong0916)|[@CooolRyan](https://github.com/CooolRyan)|[@parkjhhh](https://github.com/parkjhhh)|[@HyeJinSeok](https://github.com/HyeJinSeok)|

<br>

**• 목차** <br>

1. 데이터 수집 및 전처리 <br>

2. Logstash로 구현한 데이터 인제스트 <br>

3. Kibana를 통한 시각화 <br>

4. 기존 검색 알고리즘의 문제점 <br>

5. 의도 기반 랭킹 시스템 설계 <br>

6. 한국어 토큰화 기반 검색 품질 개선 <br>

7. 트러블 슈팅 <br>

8. 회고 <br>

<br>

---

<br>

## 1. 데이터 수집 및 전처리

<br>

<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk2.png" width=300 alt="데이터 수집">


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

<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk3.png" width=1000 alt="커스텀 테이블">

<br><br>

---

<br>

## 2. Logstash로 구현한 데이터 인제스트
<br>

• Logstash의 파이프라인은 **input → filter → output** 구조임 (.conf 파일)

• **원천 DB**에서 데이터를 주기적으로 끌어와, 필드를 전처리한 뒤 Elasticsearch에 **문서로 색인**하는 과정 <br>

<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk4.png" width=700 alt="Logstash pipeline">

<br>

### ① JDBC 입력 설정


• 원격 MySQL에서 customer_order 테이블을 읽어오도록 **input { jdbc { ... } }** 구성함 <br>

• 연결 정보 및 드라이버를 지정함 <br>

• **:sql_last_value** 기준으로 신규/변경분만 가져오도록 구성함 <br>

```
input {

  jdbc {
    jdbc_driver_library => "/path/mysql-connector-j-9.1.0.jar"
    jdbc_driver_class   => "com.mysql.cj.jdbc.Driver"

    jdbc_connection_string => "jdbc:mysql://${MYSQL_HOST}:${MYSQL_PORT}/${MYSQL_DB}?${MYSQL_PARAMS}"
    jdbc_user              => "${MYSQL_USER}"
    jdbc_password          => "${MYSQL_PASSWORD}"

    statement => "SELECT * FROM customer_order WHERE updated_at > :sql_last_value"
    schedule  => "*/5 * * * * *"  # 5초 주기
  }

}
```

<br>

### ② 데이터 필터링 


• category를 **'>' 기준으로 분해**해 top_category, sub_category 생성함 <br>

• address를 **공백 기준으로 분해**해 city, state 추출함 <br>

• order_date에서 날짜/시간을 분리 저장함 <br>

<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk5.png" width=400 alt="tasble_schema">


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

#### < 일별 주문 건수 및 매출 >

<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk6.png" width=750 alt="graph1">


<br>

#### < 2025년 주문 현황 >

<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk7.png" width=600 alt="graph2">

<br>

#### < 2025년 일일 매출 >

<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk8.png" width=600 alt="graph3">

<br>

#### < 우수회원 >

<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk10.png" width=600 alt="graph4">

<br>

#### < 브랜드별 성별 인기도 >

<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk9.png" width=600 alt="graph5">

<br><br>

---

<br>

## 4. 기존 검색 알고리즘의 문제점

<br>

• 무신사 사이트는 **키워드 기반(Best Match 25) 검색 + 동의어 사전** 조합으로 매칭을 수행함 <br>

• 그러나 이는 카테고리 및 의도 신호 부재로, **텍스트 일치도만 높은 결과**가 의도와 무관하게 상위로 노출될 가능성이 있음 <br>

• 예) ‘삭스’가 들어간 **부츠 카테고리 상품** (=삭스 부츠)도 양말 질의와 텍스트 일치도가 높아져 **상단에 노출**되는 상황 <br>


<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk11.png" width=600 alt="problem">

<br>

> [!NOTE]
> #### - 문제 관찰 방법 <br>
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


< 후드 질의에 따른 **Elasticsearch hits** 일부 >


<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk12.png" width=800 alt="hits">


− ‘후드’ 검색 시 상품명에 ‘후드’만 있는 상품과 ‘후드 집업’이 함께 상단에 노출됨 <br>

− 이때 두 결과의 **랭킹 점수(_score)가 1.6701219로 거의 동일**해 우선순위 구분이 어려움 <br>

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


<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk13.png" width=600 alt="solution">

<br>

### 🎯Scoring 전략 (1) : function_score로 가중치 합산하기


• **top_category**: 1차 분류(의도 카테고리) → **가장 큰 weight** 부여 <br> 

• **sub_category**: 2차 세부 분류(후드 티셔츠 vs 후드 집업) → 보조 weight 설정 <br> 

• product_name: 구문 일치 보정(match_phrase) 적용 <br>

• 스코어 결합: **BM25 + (모든 weight의 합)**


```
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

### 💡수행 결과


• ‘후드’ 질의에서 상의/후드 티셔츠 문서 _score ≈ 5.90 <br>

• ‘후드’ 질의에서 아우터/후드 집업 문서 _score ≈ 3.90 <br>

• top_category=상의 / sub_category=후드 티셔츠 / match_phrase("후드") 보정이 합산돼 **BM25 + 가중치가 크게 올라감** <br>

• **랭킹이 의도 카테고리(상의) 중심으로 재정렬**되어, 비의도 카테고리(아우터)가 상위 노출에서 후순위로 내려감


<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk14.png" width=800 alt="result1">

<br>

### 🎯Scoring 전략 (2) : script_score로 BM25에 직접 보정하기


• **top_category**: 상의 +3.0, 아우터 +0.5 (**1차 분류**, 가장 큰 영향) <br>

• **sub_category**: 후드 티셔츠 +2.0, 후드 집업 +0.5 (**2차 분류**, 보조 영향) <br>

• 스코어 결합: **BM25(_score) + 카테고리 가중치(직접 가산)**


```
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

### 💡수행 결과


• ‘후드’ 질의에서 상의/후드 티셔츠 문서 _score ≈ 4.40 <br>

• ‘후드’ 질의에서 아우터/후드 집업 문서 _score ≈ 1.90 <br>

• function_score 대비 분리폭(Δscore)이 더 커져, **랭킹이 상의(후드 티셔츠) 우선으로 안정화**됨

<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk15.png" width=800 alt="result1">

<br>

---

<br>

## 6. 한국어 토큰화 기반 검색 품질 개선

<br>

• 한국어 상품명 및 카테고리를 **형태소 단위로 표준화**하면 검색 정확도와 집계 신뢰도를 올릴 수 있음 <br>

• **Nori**는 Elasticsearch에서 쓰는 한국어 형태소 분석기로, **analysis-nori 플러그인**으로 제공됨 <br>

• [Elasticsearch Nori 설치 공식 문서 바로가기](https://www.elastic.co/docs/reference/elasticsearch/plugins/analysis-nori)

<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk16.png" width=400 alt="nori">

<br>

### < 한국어 토큰화 적용 플로우 >


① **커스텀 analyzer 구성**: 한국어 형태소 분석 (소문자 변환 / 읽기형 / 불용품사 처리) 조합 <br>

② 필드 적용: product_name, category, sub_category, top_category 등에 동일 분석 규칙 적용함 <br>

③ 새 인덱스 생성 → 재인덱싱: 기존 인덱스는 변경 불가하기 때문에 **새 인덱스** 만든 뒤 데이터를 이관함 <br>

④ 복합어 처리: 복합어는 **원형 토큰과 부분 토큰**을 동시에 유지하도록 설정함 <br>

⑤ 사용자 사전: **브랜드명을 사전에 추가**해 한 토큰으로 인식시키고 매칭 정확도 높임 <br>

⑥ 토큰 단위 집계: 집계를 문자열 전체가 아니라 **형태소 토큰 기준**으로 돌려 상위 키워드 및 패턴 파악 쉽게 함 <br>

<br>

### < 결과 화면 비교 >


• Nori 적용 전 — **문자열 전체값** 기준 집계 (좌) / Nori 적용 후 — 형태소 **토큰** 기준 집계 (우)


<img src="https://raw.githubusercontent.com/HyeJinSeok/ELK/main/images/elk17.png" width=700 alt="nori_compare">

<br>

• "후드 집업"이 토큰화에서 **"후드", "집", "업"으로** 과분해되어 집계/검색이 노이즈 많아짐 <br>

• 인덱스 생성 시 nori_tokenizer.decompound_mode: **"mixed"** 로 설정해야 함 <br>

• user_dictionary에 도메인 용어로 등록해 하나의 토큰으로 고정할 수도 있음 <br>

```
      "tokenizer": {
        "nori_tokenizer": {
          "type": "nori_tokenizer",
          "decompound_mode": "mixed",                    // 원형+부분 토큰 동시 유지
          "user_dictionary": "analysis/userdict_ko.txt"  // 사용자 사전 경로
        }
      },
```

<br>

---

<br>

## 7. 트러블 슈팅

<br>

### 🔴문제 1 - 증분 수집 & 타임존 이슈


• Logstash와 Elasticsearch 기본이 UTC라서, **KST(+9h) 기준** updated_at 비교가 어긋나 **증분 수집이 틀어짐** <br>

• ``WHERE updated_at > :sql_last_value``로 돌리니 팀원 환경에서 매 실행마다 **전량 재수집**(문서 116,110건까지 누적)이 발생함 <br>

• ``filter { date { timezone => "Asia/Seoul" } }``로 보정 시도했지만 반영 안 됨 (파이프라인/필드 타입 조건 불일치)


### 🟢해결책


• 증분 기준을 **시간이 아닌 단조 증가 PK**로 변경: order_no 사용 <br>

• Logstash JDBC 입력에 tracking 기능 적용: 

```
input {
  jdbc {
    # ... (연결 설정)
    statement => "SELECT * FROM customer_order WHERE order_no > :sql_last_value"
    use_column_value       => true
    tracking_column        => "order_no"  #✅
    tracking_column_type   => "numeric"
    last_run_metadata_path => "C:\\00.dataSet\\logstash_jdbc_last_run"
    schedule               => "*/5 * * * * *"  
  }
}
```

<br>

### 🔴문제 2 - DB connection 오류


• 원격 MySQL 접속이 반복 실패 후 **IP가 차단**됨 <br>

• MySQL이 비정상 접속 누적을 ``global.max_connect_errors``로 카운트해 **임계 초과 시 자동 블록**된 상태 


### 🟢해결책

```
-- 비정상 접속 허용 임계 상향
SET GLOBAL max_connect_errors = 10000;

-- 동시 접속 여유 확대(상황에 맞게)
SET GLOBAL max_connections = 300;

-- 누적된 차단 기록 초기화
FLUSH HOSTS;
```

<br>

### 🔴문제 3 - Elasticsearch 집계 오류


• ``text``는 분석 및 토크나이즈용이라 집계(terms)·정렬을 기본적으로 지원 안 함 <br>

• ``keyword``는 비토큰화라 analyzer 적용 불가함 (전처리는 normalizer로)

```
{
  "error": {
    "type": "illegal_argument_exception",
    "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [field_name] in order to load field data by uninverting the inverted index."
  }
}
```

### 🟢해결책


• 검색과 집계 **역할을 분리**함 <br>

• 검색: ``text`` <br>

• 집계/정렬/필터: ``keyword`` (멀티필드로 추가)

```
PUT /index_name
{
  "mappings": {
    "properties": {
      "product_name": {
        "type": "text",                  // 검색 (형태소 및 부분일치)
        "fields": {
          "raw": { "type": "keyword" }   // 집계, 정렬, 필터
        }
      }
    }
  }
}
```

• 집계할 때는 .raw 필드 사용:

```
GET /index_name/_search
{
  "size": 0,
  "aggs": {
    "top_products": {
      "terms": { "field": "product_name.raw" }
    }
  }
}
```

<br>

---

<br>

## 8. 회고

<br>

- **👧🏻석혜진**: ElasticSearch와 MySQL을 연결하기 위해 Logstash에 JDBC 설정을 구성하는 프로세스를 이해하게 되었다. 이후 Kibana를 활용하여 데이터를 시각화하는 과정에서 Logstash의 conf 파일에 필터를 구성하고, KQL 명령어를 익히는 실습을 수행했다. 추후에는 대용량 데이터를 삽입하고 효율적으로 전처리하는 과정을 실습해볼 계획이다.

<br>

- **👩🏼‍🦰박지혜**: ElasticSearch와 Logstash, Kibana를 통해 실생활에 사용되는 데이터를 로그하고 모니터링 해보았다. 원격 데이터베이스를 연결하는 과정에서 JDBC Driver의 버전이 달라서 연결이 정상적으로 이루어지지 않는 이슈가 발생하긴 했지만 다시 Driver을 8버전으로 설치하고 logstash를 실행시켜보니 연결이 정상적으로 되었다. 이런 DB연결 문제, 예를 들어 버전이 달라서 생기는 문제를 더 자세히 학습하고 보다 더 순조롭게 프로젝트를 진행할 수 있도록 해야겠다.

<br>

- **👦🏻나원호**: 기존 추천 알고리즘은 elastic search로 기반으로 이루어진다는 정보를 알고 있었지만 이에 대해 정확하게 어떤 식으로 구현되는지 궁금증을 가졌다. 이번 기회를 통해 score 가중치를 조정하며 사용자를 고려한 서비스를 구현하는 방식에 대해 알아볼 수 있었고 단어를 필터링하는 과정에서 사용할 수 있는 nori의 방식을 알 수 있었다.

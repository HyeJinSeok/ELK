## 트러블 슈팅

### 새로운 정보 update 문제

5초마다 업데이트 될때마다 새로운 정보를 업데이트 하는 것이 아닌 처음부터 모든 데이터를 업데이트 하게 되어 데이터가 116,110까지 쌓이게 되고, 비효율적으로 업데이트를 하게됨.
<br>

<p align="center">
  <img src="https://github.com/user-attachments/assets/44b5a3f4-65c9-40c1-97c2-e628a113444d" alt="트러블슈팅-before">
</p>

<br>
order_no 칼럼을 기준으로 새로운 데이터가 업데이트 되도록 설정.  
또한 마지막으로 가져온 정보를 `C:\\00.dataSet\\logstash_jdbc_last_run` 경로에 저장하여 Logstash를 실행하면 이 파일을 읽고 마지막으로 실행된 `order_no`를 확인하고 새로운 데이터를 업데이트함.

```ruby
use_column_value => true
tracking_column => "order_no"
tracking_column_type => "numeric"
last_run_metadata_path => "C:\\00.dataSet\\logstash_jdbc_last_run"
schedule => "*/5 * * * * *"  # 5초마다 실행
```

가장 마지막 order_no 내용만 업데이트 되고 있음을 확인.
<br>

![트러블슈팅-after](https://github.com/user-attachments/assets/a7f799ee-6ea8-47cc-af2a-7773d0c1912f)



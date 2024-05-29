
## 실무개선 Project
### 실무개선 Project 기대효과 분석
- 주소검토 실패건을 사람이 직접 처리하는 데 소요되는 N시간 이상의 리소스를 줄일 수 있다.
  - 주소검토시 실패건에 대해 담당자가 직접 단건으로 처리할 경우 개당 약 1분이 소요된다. 
- third party 이슈로 인한 주소검토 실패율을 낮출 수 있다.
  - 가끔 랜덤하게 실패가 발생하는 경우가 많았는데, 재시도 프로세스를 통해 시스템 에러를 줄일 수 있다.   
- 주소검토 이력관리를 통해 에러 트래킹의 용이성을 높일 수 있다. 
  - 재시도 횟수, 실패 사유 등에 대한 확인이 가능하다.
  - 필요하다면 이력관리 메뉴를 통해 담당자의 업무 편의를 높이거나, batch 로 실패건에 대한 후처리도 가능하다.
 
### 실무개선 Project 프로세스
```mermaid
flowchart TB

%% Colors %%
    classDef red fill:#FF3399,stroke:#000,stroke-width:2px,color:#fff
    classDef green fill:#007700,stroke:#000,stroke-width:2px,color:#fff
    classDef sky fill:#0000CD,stroke:#000,stroke-width:2px,color:#fff

    명부업로드-->|insert|register[(명부)]
    명부업로드-->|insert|address[(소재지)]
    register-->명부저장
    address-->명부저장
    명부저장-->|after commit|MQSender
    MQSender-.->|1개씩 요청|MQ
    subgraph "소재지 주소검토(loop)"
        direction TB
        MQConsumer-.->MQ
        MQConsumer-->주소검토
        주소검토-->|검색요청|thirdParty-->|검색결과|이력저장-->|insert|requestHistory[(주소검토이력)]
        requestHistory-->Result{성공여부}
        Result-->|yes|결과저장-->|주소검토결과 update|소재지[(소재지)]
        결과저장-->|insert|addressDetail[(소재지 등기정보)]
        Result-->|no|재시도
        재시도-->|재시도횟수 +1|retry{재시도 max 초과여부}
        retry-->|yes|결과저장
        retry-->|no|주소검토
        소재지-->명부등록완료처리
        addressDetail-->명부등록완료처리
        명부등록완료처리-.->Scheduler:::green-.->|명부등록상태 update|union[(조합)]
    end
    union-->주소검토결과{주소검토 미완료건 존재여부}
    주소검토결과-->|no|명부전달
    주소검토결과-->|yes|개별작업
    
    thirdParty(third party):::red
    MQ(MQ):::green
```

  

### ERD
- 변경사항: 주소검토 히스토리 table 추가

```mermaid
erDiagram

"조합" {
    int seq "PK"
    string status "명부 등록상태"
}

"조합원명부" {
    int seq "PK"
    int unionSeq "FK"
    string name "이름"
}

"소재지" {
    int seq "PK"
    int registerSeq "FK"
    string address "주소"
}

"소재지 등기정보" {
    int seq "PK"
    int addressSeq "FK"
    int number "등기번호"
    string address "등기주소"
}

"주소검토 히스토리" {
    int seq "PK"
    int addressDetailSeq "FK"
    int retryCount "재시도 횟수"
    string errorMessage "에러 메시지"
    boolean isCompleted "처리완료여부"
    date completedAt "처리일시"
}

    "조합" ||--o{ "조합원명부" : contains
    "조합원명부" ||--o{ "소재지" : contains
    "소재지" ||--o{ "소재지 등기정보" : contains
    "소재지 등기정보" ||--|| "주소검토 히스토리" : has
    
```




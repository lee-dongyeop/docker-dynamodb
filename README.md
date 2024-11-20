# docker-dynamodb
> AWS DynamoDB를 Docker 환경으로 구성하는 메뉴얼입니다. <br/>
> AWS CLI를 사용하며, 기본적인 AWS 자격 인증 절차는 생략합니다. 

<br/>

## 0. AWS DynamoDB와 DynamoDB Docker Container 비교
1. 성능 및 확장성 부족
    - **무제한 트래픽 처리** 및 **자동 확장** 기능을 제공받지 못함
    - 테라바이트 이상의 데이터를 처리할 수 있는 분산 스토리지를 지원받지 못함
2. AWS 통합 및 관리 기능 부족 (VPC, IAM, CloudWatch, 백업/복원)
    - **IAM 역할 및 정책**: DynamoDB에 대한 접근 제어.
    - **CloudWatch 모니터링**: 성능 및 사용량을 모니터링하고 로그 관리.
    - **DynamoDB 백업 및 복원**: AWS는 클릭 몇 번으로 데이터를 백업하고 복원할 수 있습니다.
3. AWS의 보안 메커니즘 이용 불가
    - AWS에서는 TLS/SSL로 데이터를 암호화하며, 데이터 저장 시 서버 측 암호화를 기본으로 제공
    - `dynamodb-local`에서는 이러한 암호화 기능이 기본적으로 제공되지 않으므로 데이터가 노출될 가능성이 존재

→ 개발 및 테스트용 서버에서만 Docker 환경으로 구성하시고, 상용 환경에서는 AWS DynamoDB를 이용하시길 바랍니다.

<br/>

## 1. AWS DynamoDB Docker 환경 구성
### 1-1. docker-compose 파일 작성
```yml
# docker-compose-dynamodb.yml
version: '3.8'
services:
  dynamodb-local:
    image: amazon/dynamodb-local:latest
    container_name: dynamodb-local
    ports:
      - "8000:8000"
    volumes:
      - ./dynamodb-data:/data
    entrypoint: ["java", "-jar", "DynamoDBLocal.jar", "-sharedDb", "-dbPath", "/data"]
```

### 1-2. docker-compose 실행
```shell
# 실행
$ docker-compose -f docker-compose-dynamodb.yml up -d

# 종료
$ docker-compose -f docker-compose-dynamodb.yml down -d
```

### 1-3. Docker Container 기동 확인
```shell
$ docker ps

CONTAINER ID   IMAGE                          COMMAND                   CREATED          STATUS          PORTS                    NAMES
685a9b8a21dc   amazon/dynamodb-local:latest   "java -jar DynamoDBL…"   23 minutes ago   Up 23 minutes   0.0.0.0:8000->8000/tcp   dynamodb-local
```

<br/>

## 2. DynamoDB Table 구성
> 아래 예시는 문자 전송 내역을 예시로 하였습니다.

### 2-1. DynamoDB 테이블 생성
- `--attribute-definitions` : 미리 테이블의 Key를 정의
  - `AttributeType` : 데이터 타입을 의미. `S(문자열), N(숫자), BOOL(Boolean), B(Binary), SS(Set), NS(Number Set), M(Map)`
- `--key-schema` : 데이터를 고유하게 식별할 수 있는 키 정의
  - `KeyType`
    - `HASH` : [필수] 기본 키(Partition Key)를 의미
    - `RANGE` : [선택] 정렬시 사용할 Sort Key
- `--billing-mode` : 비용 청구 방식
  - `PAY_PER_REQUEST` : 사용한 만큼 요금이 청구
  - `PROVISIONED` : 미리 설정한만큼 요금이 청구 (기본값)
```shell
$ aws dynamodb create-table \
    --table-name message_log \
    --attribute-definitions \
        AttributeName=phoneNumber,AttributeType=S \
        AttributeName=sentTimestamp,AttributeType=S \
    --key-schema \
        AttributeName=phoneNumber,KeyType=HASH \
        AttributeName=sentTimestamp,KeyType=RANGE \
    --billing-mode PAY_PER_REQUEST \
    --endpoint-url http://localhost:8000
```

### 2-2. DynamoDB 테이블 목록 조회
```shell
$ aws dynamodb list-tables --endpoint-url http://localhost:8000

{
    "TableNames": [
        "message_log"
    ]
}
```

### 2-3. 예시 데이터 삽입 (1) & (2)
- NoSQL이므로 Key만 정의된 대로 삽입하면, 그 외 값들은 달라도 됨.
```shell
# 예시 데이터 1
$ aws dynamodb put-item \
    --table-name message_log \
    --item '{
        "phoneNumber": {"S": "123-456-7890"},
        "sentTimestamp": {"S": "2024-11-20T10:00:00"}
    }' \
    --endpoint-url http://localhost:8000

# 예시 데이터 2
$ aws dynamodb put-item \
    --table-name message_log \
    --item '{
        "phoneNumber": {"S": "01012345678"},
        "sentTimestamp": {"S": "2024-11-19T15:30:00Z"},
        "messageContent": {"S": "Hello!"},
        "senderNumber": {"S": "02머시기 회사번호"},
        "success": {"BOOL": true},
        "countryName": {"S": "KOREA"},
        "countryCodeNum": {"N": "82"}
    }' \
    --endpoint-url http://localhost:8000
```

### 2-4. 예시 데이터 조회
```shell
$ aws dynamodb scan --table-name message_log --endpoint-url http://localhost:8000
```

> IDE(DataGrip)으로 조회시
<img width="1539" alt="스크린샷 2024-11-20 오후 3 33 48" src="https://github.com/user-attachments/assets/a3a30ee0-8cdf-464a-a490-5f71dca920c0">

---
layout: posts
title: API Gateway의 로깅 설정과 Cloudwatch logs 보관주기 설정 자동화
tags: [aws, eventbridge, gateway]
sitemap :
  priority : 1.0
---

AWS managed 서비스는 각각의 로깅방식을 제공하고 대부분 Cloudwatch logs를 통해 지원한다. API Gateway도 마찬가지로 AWS 콘솔 설정을 통해 실행 로그와 액세스 로그를 기록할 수 있다. 다만 로그 보관주기를 Gateway 콘솔에서 설정할 수 없다는 단점이 있다. 보관주기 설정을 위해서는 Cloudwatch logs 콘솔에서 해당 log group 의 보관주기를 설정하거나 CLI를 통한 추가 작업이 필요하다.

API Gateway 로그 설정방법과 커스텀 액세스 로그 포맷 구성을 위한 필드에는 무엇이 있는지 알아보고, 마지막으로 로그 보관주기 설정을 자동화하기 위한 EventBridge 구성 방법까지 알아보자.

---

## API Gateway 로그 설정 ##
로그 활성화는 API Gateway > API > Stage > Logs/Tracing 탭에서 설정이 가능하며 CloudWatch Settings 메뉴에서 실행 로그를 Custom Access Logging 메뉴에서 액세스 로그를 설정할 수 있다.

### CloudWatch Settings ###
실행 로그에서는 오류 또는 요청 또는 응답 파라미터나 페이로드 데이터, Lambda 권한 부여자가 사용하는 데이터, API 키가 필요한지 여부, 사용량 계획이 활성화되는지 여부 등의 정보가 포함되어 있다. 환경이나 목적별 필요한 옵션들을 활성화하여 관리한다.

![CloudWatch Settings](/assets/images/api-gateway-logging/image_01.png)

|Option|Value|Description|
|------|---|---|
|Enable CloudWatch Logs|true/false|실행 로깅 활성화|
|Log level|Info/Error|기록될 로그레벨|
|Log full requests/response data|true/false|요청/응답의 페이로드 데이터 기록 활성화|
|Enable Detailed CloudWatch Metrics|true/false|[metric 데이터](https://docs.aws.amazon.com/ko_kr/apigateway/latest/developerguide/api-gateway-metrics-and-dimensions.html) 기록 활성화|
<br/>

### Custom Access Logging ###
액세스 로그를 활성화하면 클라이언트 정보나 접근 경로, 응답 상태코드, 지연시간 등의 다양한 정보 조회가 가능해진다. 제공되는 필드들을 조합하여 커스텀 로그 포멧 설정이 가능하다. 모든 필드에 대한 정보는 [공식 문서](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#context-variable-reference)를 참고한다.

![Custom Access Logging](/assets/images/api-gateway-logging/image_02.png)

|Option|Value|Description|
|------|---|---|
|Enable Access Logging|true/false|액세스 로그 기록 활성화|
|Access Log Destination ARN|이번 예시에서는 Cloudwatch loggroup|리소스가 기록될 목적지 ARN|
|Log Format|원하는 형식의 로그 포멧|커스텀 액세스 로그 포맷|
<br/>

액세스 로그 저장소는 Cloudwatch와 Kinesis가 가능하나 이번 글에서는 Cloudwatch를 사용한다.

커스텀 로그 포멧은 아래와 같이 미리 제공되는 필드들을 값으로 사용할 수 있으며 키값은 자유롭게 설정 가능하다.

```json
{
  "requestId":"$context.requestId",
  "ip": "$context.identity.sourceIp",
  "requestTime":"$context.requestTime",
  "httpMethod":"$context.httpMethod",
  "resourcePath":"$context.resourcePath",
  "status":"$context.status",
  "protocol":"$context.protocol",
  "responseLength":"$context.responseLength",
  "integrationStatus":"$context.integrationStatus",
  "integrationLatency":"$context.integration.latency",
  "userAgent":"$context.identity.userAgent"
}

```

위 예시에서 사용되는 필드의 설명은 아래와 같다. 모든 필드에 대한 설명은 [공식 문서](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#context-variable-reference)를 참고한다.

|Key|Value|Description|
|------|---|---|
|requestId|$context.requestId|API Gateway가 API 요청에 할당하는 ID|
|ip|$context.identity.sourceIp|API Gateway 요청자 IP 주소|
|userAgent|$context.identity.userAgent|요청 User-Agent 헤더|
|requestTime|$context.requestTime|[CLF 형식](https://httpd.apache.org/docs/current/logs.html#common)의 요청 시간(dd/MMM/yyyy:HH:mm:ss +-hhmm)|
|httpMethod|$context.httpMethod|요청 메소드 타입. DELETE, GET, HEAD, OPTIONS, PATCH, POST, PUT|
|path|$context.path|요청 경로|
|resourcePath|$context.resourcePath|리소스에 대한 경로. 예를 들어 https://{rest-api-id}.execute-api.{region}.amazonaws.com/{stage}/root/child의 요청 URI의 경우, $context.resourcePath 값은 /root/child|
|stage|$context.stage|API 요청 stage 이름|
|status|$context.status|메서드 응답 상태 코드|
|protocol|$context.protocol|요청 프로토콜입니다(예: HTTP/1.1)|
|responseLength|$context.responseLength|응답 페이로드 길이
|responseLatency|$context.responseLatency|응답 지연 시간(ms)|
|integrationStatus|$context.integrationStatus|integration endpoint 응답 상태 코드|
|integrationLatency|$context.integration.latency|integration endpoint 에서 gateway 요청을 처리한 시간|

---

## CloudWatch log group 보관주기 설정 ##
스테이지 로그 설정으로 생성된 execution, access log 의 Cloudwatch loggroup 의 보관주기는 콘솔이나 AWS CLI를 통해 직접 수정해야 한다.

### AWS Cloudwatch Console 사용 ###
CloudWatch > Log groups > Log group 선택 > Actions > Edit retention setting 진입하여 원하는 보관주기를 설정한다.

![Edit retiontion setting](/assets/images/api-gateway-logging/image_03.png)

### AWS CLI 사용 ###
아래 CLI 명령어를 통해 log group 이름 조건으로 보관주기 설정이 가능하다.

```shell
aws logs put-retention-policy \
  --log-group-name {log group name} \
  --retention-in-days {Expire events after}
```

---

## API Gateway 로그 보관주기 자동화 ##
로그 활성화 작업 후 콘솔과 CLI를 통해 보관주기 설정이 가능하나 스테이지를 생성할때마다 수동 작업이 필요하므로 실수로 잘못된 설정을 하거나 놓쳐 로그가 무기한 보관될 우려도 있다. 이를 방지하기 위해 EventBridge와 Lambda를 통해 자동화 작업을 진행한다.

### API Gateway의 Cloudwatch log group 생성 이벤트 확인 ###
AWS 서비스의 대부분의 이벤트는 Cloudtrail을 통해 이벤트가 수집된다. Cloudwatch log 또한 마찬가지이기에 API Gateway 로그를 활성화한 후 접근이력이 있다면 액세스 로그 또는 실행로그를 위한 log group이 생성되었을 것이고 이는 Cloudtrail에서 조회 가능하다.

CloudTrail > Event History 메뉴에서 Event name 이 CreateLogGroup 이고 sourceIPAddress 가 apigateway.amazonaws.com 인 이벤트를 찾아 레코드를 확인할 수 있다.

```json
# 이벤트 레코드 예시
{
    "eventVersion": "1.08",
    "userIdentity": {
        ...
    },
    "eventTime": "2022-01-11T23:23:31Z",
    "eventSource": "logs.amazonaws.com",
    "eventName": "CreateLogGroup",
    "awsRegion": "ap-northeast-2",
    "sourceIPAddress": "apigateway.amazonaws.com",
    "userAgent": "apigateway.amazonaws.com",
    "requestParameters": {
        "logGroupName": "{log group name}"
    },
    "responseElements": null,
    "requestID": "d7a6a82e-bd50-4fe3-83bf-8de52b993efc",
    "eventID": "79df2553-8197-4aa2-8d96-43d14979389f",
    "readOnly": false,
    "eventType": "AwsApiCall",
    "apiVersion": "20140328",
    "managementEvent": true,
    "recipientAccountId": "230667876840",
    "eventCategory": "Management"
}
```

### Cloudwatch log group 보관주기 변경을 위한 Lambda 생성 ###
선호하는 언어의 AWS SDK를 찾아 log group의 retention 변경 방법을 확인한다. 아래 예시에서는 python을 사용한다.
```python
import json
import boto3

def lambda_handler(event, context):
    print(f"Event message: {event}")
    
    detail = event['detail']
    log_group_name = detail['requestParameters']['logGroupName']
    response = putRetentionPolicy(log_group_name)
    print(response)

def putRetentionPolicy(log_group_name):
    client = boto3.client('logs')
    response = client.put_retention_policy(
        logGroupName=log_group_name,
        retentionInDays=90)
    return response
```
EventBridge를 통해 이벤트 레코드를 Lambda에서 전달받아 사용하기에 위 단락에서 다룬 Log group 생성 이벤트 레코드를 참고하여 log group 이름을 추출할 수 있고 보관주기를 변경할 수 있다.

### EventBridge 룰 생성 ###
이제 모든 준비가 되었으니 EventBride 룰을 생성하고 log group의 보관주기를 변경하는 Lambda 함수를 target으로 설정하자.

EventBridge > Rules > Create rule을 통해 진입한다. 중요한 부분은 룰 패턴을 정의하는 것이다.

![Event rule pattern](/assets/images/api-gateway-logging/image_04.png)

![Event rule pattern](/assets/images/api-gateway-logging/image_05.png)

API Gateway가 log group을 생성한 경우에 보관주기 변경 Lambda를 실행하도록 설정한다.

### 보관주기 자동화 동작 확인 ###
API Gateway의 특정 스테이지에 진입해 실행 로그 또는 액세스 로그를 활성화한다. 이후 접근 가능한 스테이지의 API를 호출해 로그가 기록되도록 유도한다. 이후 CloudWatch 로그 그룹의 보관주기가 원하는 값으로 변경되었는지 확인한다.

## 결론 ##
로그 그룹 보관주기를 관리하는 방법은 여러가지가 있을 것이다. 가이드를 통해 관련 부서에게 위임하는 방법. 또는 AWS Config 등을 통해 옵션을 통제하거나 모니터링해 올바른 설정을 유도하는 방법도 있을 것이다. 이번 글에서는 자동화에 초점을 맞추었고 이를 해결하기 위한 방법은 이외에도 많을 것이고 더 좋은 방법도 있을 것이다.

방식보다 중요한 것은 사람의 실수로 인해 발생할 수 있는 작업들을 자동화하는 행위 자체라고 생각한다. 모든 것을 자동화할 필요는 없지만 지속적인 자동화는 장기적으로 볼 때 효율적인 리소스 운영과 일관된 설정을 유지관리해나가는 측면에서 큰 도움이 될 것이다.
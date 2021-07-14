---
layout: posts
title: Eventbridge 를 활용하여 AWS 서비스 이벤트 다루기
tags: [aws, eventbridge, event, rule]
sitemap :
  priority : 1.0
---

AWS 상에서 다양한 아키텍처를 구성하다보면 서비스의 상태를 모니터링하거나 이벤트 알림을 원하는 채널에서 받을 필요가 있습니다.
원하는 서비스의 콘솔을 통해 제공하는 경우도 있으나 디테일한 설정은 어렵거나 제공되지 않는 경우가 많습니다.

이를 해결하기 위한 방법 중 한가지로 Eventbridge 를 소개합니다. 이번 글에서는 Eventbridge 의 여러 역할 중 AWS 서비스 이벤트 데이터 전송에 대해서 다루려 합니다.

Eventbridge 는 AWS 서비스 이벤트를 사용하여 이벤트 기반 애플리케이션을 대규모로 손쉽게 구축할 수 있는 서버리스 이벤트 버스입니다. 이러한 이벤트 소스의 실시간 이벤트 스트림을 원하는 대상으로 전송하는 역할을 합니다. 또한 이벤트 패턴을 통해 룰을 설정하고 데이터 필터링하고 전송할 대상을 선택할 수 있습니다. 이를 통해 이벤트 생산자와 소비자의 분리가 가능합니다.

## Eventbridge 를 통한 AWS 서비스 이벤트 처리 ##
여러 이벤트 소스 중 AWS Service 는 이미 생성되어 있는 Default eventbus 에 수많은 이벤트를 전송하고 있습니다. 현재 EventBridge 를 지원되고 있는 AWS 서비스 목록은 [공식 문서](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/EventTypes.html)를 참고합니다. 

Default EventBus 를 통해 전송된 이벤트는 사용자가 지정한 Rule 을 통해 필터링 되고 대상에게 전달됩니다. 대상을 통해 다양한 방식으로 데이터를 처리할 수 있습니다. 예를 들면 이벤트 데이터를 파싱하여 원하는 메시지 형태로 Slack 에 전송하는 AWS Lambda Function 을 사용할 수 있고, 또는 SNS Topic 으로 전송하고 해당 Topic 을 Subscribe 하는 대상들이 처리할 수 있도록 구성할 수도 있습니다.

![EventBridge 를 사용한 AWS 서비스 이벤트 알림처리](/assets/images/eventbridge-for-aws-service-event/image_01.png)

---

## Rule 을 사용한 데이터 필터링 ##
앞서 되었던 Rule 의 역할은 이벤트 데이터를 필터링하고 특정 대상에게 전달하는 것입니다. 그 중 먼저 필터링에 대해서 얘기해보겠습니다.

각각의 AWS Service 이벤트는 JSON 형태의 컨텐츠를 EventBridge 의 Default EventBus 에 전달하게 됩니다. Rule 은 이 이벤트 컨텐츠의 패턴을 정의하고 이와 일치하는 데이터를 대상에게 전달할 수 있습니다. 이벤트 패턴에 작성되지 않은 항목은 일치 여부를 판단하지 않고 통과됨에 주의합니다.

CodeBuild 프로젝트에서 빌드를 생성한 후 상태 변화를 발송하는 이벤트인 `CodeBuild State Change Event` 를 특정 패턴과 일치하는 경우에만 대상으로 전송하는 예제를 만들어보겠습니다.

필터링 조건은 아래와 같습니다.
1. 상태가 `SUCCEEDED` | `FAILED` | `STOPPED` 인 경우
2. CodebBuild 프로젝트 이름이 `test-codebuild-project` 인 경우

이제 AWS Console 을 사용해 Rule 을 등록해보겠습니다.

---

### 1. AWS Console > Amazon EventBridge > Events > Rules > Create rule 버튼을 통해 생성을 시작합니다.

![EventBrdige 콘솔에서 Rule 생성 화면 진입 방법](/assets/images/eventbridge-for-aws-service-event/image_02.png)

### 2. 패턴 정의 섹션에서 필요한 항목을 선택한 결과로 생성된 이벤트 패턴을 확인합니다.

아래 이미지 처럼 필요한 항목들을 선택해줍니다. Specific state 의 경우 다중 선택이 가능하니 필요한 세가지 상태를 추가합니다.

![Rule 에서 원하는 Codebuild 이벤트와 디테일 조건 선택하기](/assets/images/eventbridge-for-aws-service-event/image_03.png)

### 3. 선택한 항목을 기반으로 이벤트 패턴이 정상적으로 생성되었는지 확인합니다.

이벤트 패턴 하단에 선택한 상태별 샘플 이벤트를 제공하니 이를 참고하면 좋습니다. 리스트 타입을 활용하여 키값이 리스트 값 중 하나인지 필터링하는 패턴을 작성할 수 있습니다. 이번 예제에서는 아래와 같이 빌드 상태 조건에 사용되었습니다.
```json
{
  ...
  "detail": {
    "build-status": ["SUCCEEDED", "FAILED", "STOPPED"]  # 세 가지 상태중 하나만 일치한다면 통과
  }
  ...
}
```

![Rule 에서 원하는 Codebuild 이벤트와 디테일 조건 선택하기](/assets/images/eventbridge-for-aws-service-event/image_04.png)

### 4. 선택한 항목 외 필요한 패턴을 샘플 이벤트를 기반으로 추가합니다.

콘솔에서 제공된 샘플 이벤트를 참고하여 필요한 항목을 찾아 이벤트 패턴을 수정해줍니다. 

```json
# 샘플 이벤트
{
  "version": "0",
  "id": "bfdc1220-60ff-44ad-bfa7-3b6e6ba3b2d0",
  "detail-type": "CodeBuild Build State Change",
  "source": "aws.codebuild",
  "account": "123456789012",
  "time": "2017-07-12T00:42:28Z",
  "region": "us-east-1",
  "resources": ["arn:aws:codebuild:us-east-1:123456789012:build/SampleProjectName:ed6aa685-0d76-41da-a7f5-6d8760f41f55"],
  "detail": {
    "build-status": "SUCCEEDED",
    "project-name": "SampleProjectName",
    "build-id": "arn:aws:codebuild:us-east-1:123456789012:build/SampleProjectName:ed6aa685-0d76-41da-a7f5-6d8760f41f55",
    "current-phase": "COMPLETED",
    "current-phase-context": "[]",
    "version": "1"
  }
}
```

지금은 프로젝트 이름이 필요하니 detail > project-name 을 사용하여 아래와 같이 패턴을 사용할 수 있습니다.

```json
{
  "source": "aws.codebuild",
  "detail-type": "CodeBuild Build State Change",
  "detail": {
    "build-status": ["SUCCEEDED", "FAILED", "STOPPED"],
    "project-name": "test-codebuild-project"  # 프로젝트 이름 패턴 추가
  }
}
```

그럼 콘솔에서 패턴을 수정 후 등록해봅시다.

`Event pattern` > `Edit` > 수정된 패턴 적용 > `Save`

![선택된 값을 기준으로 생성된 이벤트 패턴 확인](/assets/images/eventbridge-for-aws-service-event/image_05.png)

간혹 `Save` 를 하지 않고 진행하는 경우가 있는데 이럴 경우 패턴이 원하는대로 적용되지 않으니 주의합니다.

### 5. Rule 을 통과한 이벤트가 전달될 대상을 선택하고 지금까지 작성한 내용을 기반으로 Rule 을 생성합니다.

이제 Rule 을 통과한 이벤트를 전달할 대상을 선택합니다. 대상은 다중 선택이 가능합니다.

![이벤트 전달할 대상 선택](/assets/images/eventbridge-for-aws-service-event/image_06.png)

예를 들어 AWS Lambda function 을 선택하면 실제 생성되어 있는 function 을 선택할 수 있습니다. 원하는 대상을 선택한 후 `Configure input` 에서 원하는 형태를 선택하여 이벤트 컨텐츠를 여러 형태로 변환하여 대상에게 전달할 수 있습니다.

4가지 옵션의 대략적인 내용을 정리해보겠습니다.

* Matched events - 전달된 이벤트 값을 그대로 대상에게 전달
* Part of the matched event - 전달된 이벤트 값 중 일부만을 선택해서 대상에게 전달
  * 예시) $.detail,$.source - detail 과 source 키 항목만을 대상에게 전달
* Constant (JSON text) - 전달된 이벤트 값을 무시하고 새로 설정한 값을 대상에게 전달
* Input transfomer - 전달된 이벤트 값을 원하는 형태로 변환
  * 보통 사람이 읽을 수 있는 형태로 전달하기 위해 사용된다. 예를 들면 최종 대상이 이메일 발송인 경우 포함될 문장을 생성하는 형태
  * 관련 [AWS 공식 블로그 문서](https://aws.amazon.com/ko/premiumsupport/knowledge-center/cloudwatch-human-readable-notifications/) 참고

지금은 CodeBuild 서비스에서 생성된 값을 그대로 대상에게 전달하는 `Matched events` 를 선택하고 하단의 `Create` 버튼을 통해 Rule 을 생성합니다.

이제 모든 작업이 완료되었습니다. 실제 Codebuild 이벤트가 발생하게 되면 rule 상세화면에서 진입가능한 Cloudwatch monitoring 을 통해 통계를 확인할 수 있습니다.

![rule 모니터링](/assets/images/eventbridge-for-aws-service-event/image_07.png)

---

## 결론 ##

EventBridge 의 활용도는 무궁무진하지만 오늘은 AWS Service event 를 다루는 방법을 소개했습니다. 이벤트 기반으로 동작하는 AWS Service 들을 별도 구축 없이 Default EventBus 를 통해서 안정적으로 대상에게 전달할 수 있었습니다. 이벤트를 발생시키는 AWS Service 와 실제 이를 처리하는 대상을 분리함으로써 서로의 종속성 없이 개발할 수 있는 환경을 구성할 수 있었습니다.

또한 AWS Amplify 와 같이 console 을 통한 배포 Notification 에 대한 조건이나 전달할 대상 선택에 제한이 있는 경우 EventBus 로 관련 이벤트를 전달하고 있다면 직접 EventBridge 를 통해 원하는 방향대로 수정할 수 있는 유연함을 갖추었습니다. 시간이 된다면 이런 예제들을 정리하고 공유하고자 합니다.

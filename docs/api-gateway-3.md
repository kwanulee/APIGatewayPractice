
## 3. API Gateway를 통한 Device Shadow 액세스 하기
AWS IoT 플랫폼에서는 원하는 프로그래밍 언어를 기반으로 [AWS SDK](https://aws.amazon.com/ko/tools/)를 사용하여 디바이스 섀도에 액세스 할 수 있습니다. 이 장에서는 **Java용 AWS SDK**를 사용하여 Device Shadow를 액세스하는 **Lambda 함수**를 정의하고, 이 Lambda 함수를 **API Gateway**와 통합하여 2절에서 정의한 **REST API**를 구축하는 방법을 설명합니다.

AWS SDK for Java를 프로젝트에서 사용하려면 pom.xml 파일에서 종속성으로 필요한 개별 모듈을 선언해야 합니다. 디바이스 섀도에 액세스 하기 위해서는 **aws-java-sdk-iot**를 종속성으로 선언해야 합니다.

### 3.1 [디바이스 목록 조회 REST API 구축하기](api-gateway-3.1.html)
### 3.2 [디바이스 상태 조회 REST API 구축하기](api-gateway-3.2.html)
### 3.3 [디바이스 상태 변경 REST API 구축하기](api-gateway-3.3.html)
### 3.4 [디바이스 로그 조회 REST API 구축하기](api-gateway-3.4.html)

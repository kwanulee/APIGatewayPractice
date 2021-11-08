#### 1단계: AWS Toolkit을 통해 Lambda 함수 생성


- **사전준비 사항**
	- [DynamoDB를 이용한 백엔드 구축하기](dynamodb.html) 단원의 [3. DynamoDBd와 AWS Lambda를 이용한 IoT 백엔드 구축 실습 (수정)](dynamodb.html#4)의 실습이 완료된 상태여야 합니다.

1. 다음 정보를 바탕으로 AWS Lambda 프로젝트를 Eclipse용 AWS Toolkit을 이용하여 생성한다.
	- **Project name**: *LogDeviceLambdaJavaProject*
	- **Class Name**: *LogDeviceHandler*
	- **Input Type**에서 *Custom*을 선택합니다. 

2. Eclipse 프로젝트 탐색기를 사용하여 *LogDeviceLambdaJavaProject* 프로젝트에서 *LogDeviceHandler.java*를 열고, 다음 코드로 바꿉니다.

	```java
	import java.text.ParseException;
	import java.text.SimpleDateFormat;
	import java.util.Iterator;
	import java.util.TimeZone;
	
	import com.amazonaws.services.dynamodbv2.AmazonDynamoDB;
	import com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilder;
	import com.amazonaws.services.dynamodbv2.document.DynamoDB;
	import com.amazonaws.services.dynamodbv2.document.Item;
	import com.amazonaws.services.dynamodbv2.document.ItemCollection;
	import com.amazonaws.services.dynamodbv2.document.QueryOutcome;
	import com.amazonaws.services.dynamodbv2.document.Table;
	import com.amazonaws.services.dynamodbv2.document.spec.QuerySpec;
	import com.amazonaws.services.dynamodbv2.document.utils.NameMap;
	import com.amazonaws.services.dynamodbv2.document.utils.ValueMap;
	import com.amazonaws.services.lambda.runtime.Context;
	import com.amazonaws.services.lambda.runtime.RequestHandler;
	
	public class LogDeviceHandler implements RequestHandler<Event, String> {
	
		private DynamoDB dynamoDb;
		private String DYNAMODB_TABLE_NAME = "Logging";
		
	    @Override
	    public String handleRequest(Event input, Context context) {
	    	this.initDynamoDbClient();
	    	
	    	Table table = dynamoDb.getTable(DYNAMODB_TABLE_NAME);
	    	
	    	long from=0;
	    	long to=0;
			try {
				SimpleDateFormat sdf = new SimpleDateFormat ( "yyyy-MM-dd HH:mm:ss");
				sdf.setTimeZone(TimeZone.getTimeZone("Asia/Seoul"));
				
				from = sdf.parse(input.from).getTime() / 1000;
				to = sdf.parse(input.to).getTime() / 1000;
			} catch (ParseException e1) {
				e1.printStackTrace();
			}
			
	        QuerySpec querySpec = new QuerySpec()
	        		.withKeyConditionExpression("deviceId = :v_id and #t between :from and :to")
	        		.withNameMap(new NameMap().with("#t", "time"))
	                .withValueMap(new ValueMap().withString(":v_id",input.device).withNumber(":from", from).withNumber(":to", to));	
	    
	        ItemCollection<QueryOutcome> items=null;
	    	try {    		
	    		items = table.query(querySpec);
	        }
	        catch (Exception e) {
	            System.err.println("Unable to scan the table:");
	            System.err.println(e.getMessage());
	    	}
	        
	        return getResponse(items);
	    }
	    
	    private String getResponse(ItemCollection<QueryOutcome> items) {
	    	
	    	Iterator<Item> iter = items.iterator();
	        String response = "{ \"data\": [";
	        for (int i =0; iter.hasNext(); i++) {
	        	if (i!=0) 
	        		response +=",";
	        	response += iter.next().toJSON();
	        }
	        response += "]}";
	        return response;
	    }
	
		private void initDynamoDbClient() {
			AmazonDynamoDB client = AmazonDynamoDBClientBuilder.standard().build();
	
			this.dynamoDb = new DynamoDB(client);
		}
	}
	
	class Event {
		public String device;
		public String from;
		public String to;
	}
	```

4. **Lambda에 함수를 업로드하려면**, Eclipse 코드 창에서 마우스 오른쪽 버튼을 클릭하고 **[AWS Lambda]**와 **[Upload function to AWS Lambda]**를 차례대로 선택합니다.
5. **[Select Target Lambda Function]** 페이지에서 사용할 AWS 리전을 선택합니다. 이 리전은 Amazon S3 버킷에 대해 선택한 리전과 동일해야 합니다.
6. 새 Lambda 함수 생성을 선택하고 함수 이름(예: *LogDeviceFunction*)을 입력한 후, [**Next**]를 선택합니다.
7. **함수 구성(Function Configuration** 페이지에서 대상 Lambda 함수에 대한 설명을 입력하고 함수에서 사용할 **IAM 역할 선택**합니다.
	- 사용할 역할은 **AmazonDynamoDBFullAccess** 정책이 연결되어 있어야 합니다. 만약 이러한 역할이 없다면, IAM 콘솔을 통해 해당 역할을 생성합니다.
		- 다음 [링크](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/lambda-intro-execution-role.html)를 통해 역할 생성에 대해서 자세히 살펴보세요.
8. Lambda 함수 코드를 저장할 S3 버킷을 선택합니다. 만약 새로운 Amazon S3 버킷을 생성하고 싶은 경우에는 **생성** 버튼을 클릭하고 버킷 생성 대화 상자에 버킷 이름을 입력합니다.
9. **Finish**를 선택하여 Lambda 함수를 AWS에 업로드합니다. 
10. **Lambda 함수를 실행하려면**, Eclipse 코드 창에서 마우스 오른쪽 버튼을 클릭하고 AWS Lambda를 선택한 후 **Run Function on AWS Lambda**(AWS Lambda에서 함수 실행)를 선택합니다. 
11. **Enter the JSON input for your function**이 선택된 상태에서 입력 창에 다음 입력 문자열을 입력한다.
	- device 속성: 조회할 사물의 이름이 *MyMKRWiFi1010*인 경우를 가정
	- from 속성: 시작 시간 (yyyy-MM-dd hh:mm:ss) 형식의 문자열
	- to 속성: 종료 시간  (yyyy-MM-dd hh:mm:ss) 형식의 문자열

		```JSON
		{ "device": "MyMKRWiFi1010", "from":"2019-11-29 17:56:26", "to": "2019-12-07 18:56:26"}	
		```
11. **Invoke** 버튼을 클릭한 후, **Eclipse Console** 창에 다음과 같은 결과가 출력되는 지 확인합니다.

	```
	Skip uploading function code since no local change is found...
Invoking function...
==================== FUNCTION OUTPUT ====================
"{ \"data\": [{\"temperature\":\"23.10\",\"LED\":\"ON\",\"time\":1575677556,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:12:36\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677561,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:12:41\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677651,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:14:11\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677656,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:14:16\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677661,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:14:21\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677666,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:14:26\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677671,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:14:31\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677676,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:14:36\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677681,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:14:41\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677686,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:14:46\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677691,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:14:51\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677696,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:14:56\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677711,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:15:11\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677716,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:15:16\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677721,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:15:21\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677726,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:15:26\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677731,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:15:31\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677736,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:15:36\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677741,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:15:41\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677746,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:15:46\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677751,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:15:51\"},{\"temperature\":\"23.30\",\"LED\":\"ON\",\"time\":1575677771,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:16:11\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677776,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:16:16\"},{\"temperature\":\"23.30\",\"LED\":\"ON\",\"time\":1575677796,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:16:36\"},{\"temperature\":\"23.20\",\"LED\":\"ON\",\"time\":1575677801,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:16:41\"},{\"temperature\":\"23.20\",\"LED\":\"OFF\",\"time\":1575677811,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:16:51\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575677836,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:17:16\"},{\"temperature\":\"23.20\",\"LED\":\"OFF\",\"time\":1575677846,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:17:26\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575677866,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:17:46\"},{\"temperature\":\"23.20\",\"LED\":\"OFF\",\"time\":1575677871,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:17:51\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575677906,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:18:26\"},{\"temperature\":\"23.20\",\"LED\":\"OFF\",\"time\":1575677911,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:18:31\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575677921,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:18:41\"},{\"temperature\":\"23.20\",\"LED\":\"OFF\",\"time\":1575677926,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 09:18:46\"},{\"temperature\":\"0.00\",\"LED\":\"OFF\",\"time\":1575681573,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:19:33\"},{\"temperature\":\"22.30\",\"LED\":\"OFF\",\"time\":1575681578,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:19:38\"},{\"temperature\":\"22.40\",\"LED\":\"OFF\",\"time\":1575681598,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:19:58\"},{\"temperature\":\"22.50\",\"LED\":\"OFF\",\"time\":1575681603,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:20:03\"},{\"temperature\":\"22.60\",\"LED\":\"OFF\",\"time\":1575681623,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:20:23\"},{\"temperature\":\"22.70\",\"LED\":\"OFF\",\"time\":1575681633,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:20:33\"},{\"temperature\":\"22.70\",\"LED\":\"ON\",\"time\":1575681649,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:20:49\"},{\"temperature\":\"22.80\",\"LED\":\"ON\",\"time\":1575681658,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:20:58\"},{\"temperature\":\"22.80\",\"LED\":\"OFF\",\"time\":1575681659,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:20:59\"},{\"temperature\":\"22.90\",\"LED\":\"OFF\",\"time\":1575681673,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:21:13\"},{\"temperature\":\"23.00\",\"LED\":\"OFF\",\"time\":1575681703,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:21:43\"},{\"temperature\":\"23.10\",\"LED\":\"OFF\",\"time\":1575681738,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:22:18\"},{\"temperature\":\"23.20\",\"LED\":\"OFF\",\"time\":1575681768,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:22:48\"},{\"temperature\":\"23.00\",\"LED\":\"OFF\",\"time\":1575681973,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:26:13\"},{\"temperature\":\"23.10\",\"LED\":\"OFF\",\"time\":1575681978,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:26:18\"},{\"temperature\":\"23.20\",\"LED\":\"OFF\",\"time\":1575681993,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:26:33\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682103,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:28:23\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682158,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:29:18\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682258,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:30:58\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682268,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:31:08\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682278,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:31:18\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682303,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:31:43\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682313,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:31:53\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682328,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:32:08\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682338,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:32:18\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682348,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:32:28\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682353,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:32:33\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682378,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:32:58\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682383,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:33:03\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682388,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:33:08\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682393,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:33:13\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682403,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:33:23\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682408,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:33:28\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682413,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:33:33\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682428,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:33:48\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682433,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:33:53\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682438,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:33:58\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682478,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:34:38\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682483,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:34:43\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682502,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:35:02\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682512,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:35:12\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682517,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:35:17\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682522,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:35:22\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682557,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:35:57\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682562,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:36:02\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682577,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:36:17\"},{\"temperature\":\"23.20\",\"LED\":\"OFF\",\"time\":1575682722,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:38:42\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682727,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:38:47\"},{\"temperature\":\"23.20\",\"LED\":\"OFF\",\"time\":1575682732,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:38:52\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682737,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:38:57\"},{\"temperature\":\"23.20\",\"LED\":\"OFF\",\"time\":1575682742,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:39:02\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682747,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:39:07\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682857,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:40:57\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682872,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:41:12\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682877,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:41:17\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575682882,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:41:22\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575682887,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:41:27\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575683152,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:45:52\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575683551,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:52:31\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575683806,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:56:46\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575683811,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:56:51\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575683816,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 10:56:56\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575684080,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:01:20\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575684085,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:01:25\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575684090,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:01:30\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575684570,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:09:30\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575684575,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:09:35\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575684670,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:11:10\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575684675,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:11:15\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575684969,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:16:09\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575684974,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:16:14\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575684979,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:16:19\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685009,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:16:49\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685014,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:16:54\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685024,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:17:04\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575685049,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:17:29\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685149,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:19:09\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575685159,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:19:19\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685164,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:19:24\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575685169,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:19:29\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685174,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:19:34\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575685244,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:20:44\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685249,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:20:49\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575685454,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:24:14\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685459,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:24:19\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575685533,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:25:33\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685538,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:25:38\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575685558,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:25:58\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685563,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:26:03\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575685583,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:26:23\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685588,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:26:28\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575685593,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:26:33\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685613,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:26:53\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685698,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:28:18\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685703,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:28:23\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685713,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:28:33\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685718,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:28:38\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685728,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:28:48\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685738,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:28:58\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685743,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:29:03\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685748,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:29:08\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685788,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:29:48\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685793,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:29:53\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685838,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:30:38\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685843,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:30:43\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685898,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:31:38\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685908,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:31:48\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685913,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:31:53\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685918,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:31:58\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685928,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:32:08\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685938,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:32:18\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685963,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:32:43\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685968,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:32:48\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575685988,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:33:08\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575685998,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:33:18\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686003,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:33:23\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686008,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:33:28\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686013,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:33:33\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686023,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:33:43\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686043,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:34:03\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686053,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:34:13\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686058,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:34:18\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686063,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:34:23\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686078,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:34:38\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686083,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:34:43\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686093,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:34:53\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686113,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:35:13\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686118,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:35:18\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686128,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:35:28\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686167,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:36:07\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686172,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:36:12\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686177,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:36:17\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686497,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:41:37\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686502,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:41:42\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686507,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:41:47\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686522,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:42:02\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686612,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:43:32\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686622,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:43:42\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686632,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:43:52\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686637,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:43:57\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686642,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:44:02\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686647,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:44:07\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686667,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:44:27\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686672,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:44:32\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686682,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:44:42\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575686687,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:44:47\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575686996,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:49:56\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575687111,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:51:51\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687121,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:52:01\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575687126,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:52:06\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687136,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:52:16\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575687186,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:53:06\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687216,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:53:36\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575687221,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:53:41\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687231,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:53:51\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575687306,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:55:06\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687311,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:55:11\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575687351,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:55:51\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687401,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:56:41\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575687406,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:56:46\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687411,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:56:51\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575687416,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:56:56\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687431,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:57:11\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575687441,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:57:21\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687446,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:57:26\"},{\"temperature\":\"23.30\",\"LED\":\"OFF\",\"time\":1575687451,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:57:31\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687466,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:57:46\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575687510,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 11:58:30\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687610,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 12:00:10\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575687615,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 12:00:15\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687645,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 12:00:45\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575687650,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 12:00:50\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687705,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 12:01:45\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575687710,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 12:01:50\"},{\"temperature\":\"23.40\",\"LED\":\"OFF\",\"time\":1575687715,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 12:01:55\"},{\"temperature\":\"23.50\",\"LED\":\"OFF\",\"time\":1575687720,\"deviceId\":\"MyMKRWiFi1010\",\"timestamp\":\"2019-12-07 12:02:00\"}]}"
==================== FUNCTION LOG OUTPUT ====================
START RequestId: 81172ac4-f728-4f4e-a984-40b7491d71e1 Version: $LATEST
END RequestId: 81172ac4-f728-4f4e-a984-40b7491d71e1
REPORT RequestId: 81172ac4-f728-4f4e-a984-40b7491d71e1	Duration: 8416.52 ms	Billed Duration: 8500 ms	Memory Size: 512 MB	Max Memory Used: 144 MB	Init Duration: 328.57 ms	
	
	``` 

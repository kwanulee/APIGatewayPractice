#### 1단계: AWS Toolkit을 통해 Lambda 함수 생성


- **사전준비 사항**
	- [DynamoDB를 이용한 백엔드 구축하기](dynamodb.html) 단원의 [3. DynamoDBd와 AWS Lambda를 이용한 IoT 백엔드 구축 실습](dynamodb.html#3)의 실습이 완료된 상태여야 합니다.

1. 다음 정보를 바탕으로 AWS Lambda 프로젝트를 Eclipse용 AWS Toolkit을 이용하여 생성한다.
	- **Project name**: *LogDeviceLambdaJavaProject*
	- **Class Name**: *LogDeviceHandler*
	- **Input Type**에서 *Custom*을 선택합니다. 

2. Eclipse 프로젝트 탐색기를 사용하여 *LogDeviceLambdaJavaProject* 프로젝트에서 *LogDeviceHandler.java*를 열고, 다음 코드로 바꿉니다.
	
	```java
	import java.text.ParseException;
	import java.util.Iterator;
	
	import com.amazonaws.services.dynamodbv2.AmazonDynamoDB;
	import com.amazonaws.services.dynamodbv2.AmazonDynamoDBClientBuilder;
	import com.amazonaws.services.dynamodbv2.document.DynamoDB;
	import com.amazonaws.services.dynamodbv2.document.Item;
	import com.amazonaws.services.dynamodbv2.document.ItemCollection;
	import com.amazonaws.services.dynamodbv2.document.ScanOutcome;
	import com.amazonaws.services.dynamodbv2.document.Table;
	import com.amazonaws.services.dynamodbv2.document.spec.ScanSpec;
	import com.amazonaws.services.dynamodbv2.document.utils.NameMap;
	import com.amazonaws.services.dynamodbv2.document.utils.ValueMap;
	import com.amazonaws.services.lambda.runtime.Context;
	import com.amazonaws.services.lambda.runtime.RequestHandler;
	
	public class LogDeviceHandler implements RequestHandler<Event, String> {
	
		private DynamoDB dynamoDb;
		
	    @Override
	    public String handleRequest(Event input, Context context) {
	    	this.initDynamoDbClient();
	    	
	    	Table table = dynamoDb.getTable(input.device);
	    	
	    	long from=0;
	    	long to=0;
			try {
				SimpleDateFormat sdf = new SimpleDateFormat ( "yyyy-MM-dd HH:mm:ss");
				sdf.setTimeZone(TimeZone.getTimeZone("Asia/Seoul"));
			
				from = sdf.parse(input.from).getTime() / 1000;
				to = sdf.parse(input.to).getTime() / 1000;
			} catch (ParseException e1) {
				// TODO Auto-generated catch block
				e1.printStackTrace();
			}
	 
	    	ScanSpec scanSpec = new ScanSpec()
	                .withFilterExpression("#t between :from and :to").withNameMap(new NameMap().with("#t", "time"))
	                .withValueMap(new ValueMap().withNumber(":from", from).withNumber(":to", to));
	           
	    	
	    	ItemCollection<ScanOutcome> items=null;
	    	try {
	    		items = table.scan(scanSpec);
	        }
	        catch (Exception e) {
	            System.err.println("Unable to scan the table:");
	            System.err.println(e.getMessage());
	    	}
	        
	        return getResponse(items);
	    }
	    
	    private String getResponse(ItemCollection<ScanOutcome> items) {
	    	
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
			AmazonDynamoDB client = AmazonDynamoDBClientBuilder.standard().withRegion("ap-northeast-2").build();
	
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
	- device 속성: 조회할 사물의 로그를 기록한 DynamoDB 테이블이 *DeviceData*인 경우를 가정
	- from 속성: 시작 시간 (yyyy-MM-dd hh:mm:ss) 형식의 문자열
	- to 속성: 종료 시간  (yyyy-MM-dd hh:mm:ss) 형식의 문자열

		```JSON
		{ "device": "DeviceData", "from":"2019-11-29 00:00:00", "to": "2019-11-29 18:09:36"}		
		```
11. **Invoke** 버튼을 클릭한 후, **Eclipse Console** 창에 다음과 같은 결과가 출력되는 지 확인합니다.

	```
	Skip uploading function code since no local change is found...
	Invoking function...
	==================== FUNCTION OUTPUT ====================
	"{ \"data\": [{\"clientToken\":\"a\",\"temperature\":\"24.60\",\"LED\":\"OFF\",\"time\":1575018207,\"timestamp\":\"2019-11-29 18:03:27\"},{\"clientToken\":\"a\",\"temperature\":\"24.90\",\"LED\":\"OFF\",\"time\":1575018547,\"timestamp\":\"2019-11-29 18:09:07\"},{\"clientToken\":\"a\",\"temperature\":\"25.10\",\"LED\":\"OFF\",\"time\":1575018576,\"timestamp\":\"2019-11-29 18:09:36\"},{\"clientToken\":\"a\",\"temperature\":\"24.70\",\"LED\":\"OFF\",\"time\":1575018192,\"timestamp\":\"2019-11-29 18:03:12\"},{\"clientToken\":\"a\",\"temperature\":\"24.50\",\"LED\":\"OFF\",\"time\":1575018197,\"timestamp\":\"2019-11-29 18:03:17\"},{\"clientToken\":\"a\",\"temperature\":\"25.00\",\"LED\":\"OFF\",\"time\":1575017786,\"timestamp\":\"2019-11-29 17:56:26\"},{\"clientToken\":\"a\",\"temperature\":\"25.00\",\"LED\":\"OFF\",\"time\":1575018566,\"timestamp\":\"2019-11-29 18:09:26\"},{\"clientToken\":\"a\",\"temperature\":\"25.00\",\"LED\":\"OFF\",\"time\":1575018551,\"timestamp\":\"2019-11-29 18:09:11\"},{\"clientToken\":\"a\",\"temperature\":\"25.00\",\"LED\":\"OFF\",\"time\":1575018571,\"timestamp\":\"2019-11-29 18:09:31\"},{\"clientToken\":\"a\",\"temperature\":\"25.00\",\"LED\":\"OFF\",\"time\":1575018561,\"timestamp\":\"2019-11-29 18:09:21\"},{\"clientToken\":\"a\",\"temperature\":\"25.00\",\"LED\":\"OFF\",\"time\":1575018556,\"timestamp\":\"2019-11-29 18:09:16\"},{\"clientToken\":\"a\",\"temperature\":\"24.60\",\"LED\":\"OFF\",\"time\":1575018202,\"timestamp\":\"2019-11-29 18:03:22\"},{\"clientToken\":\"a\",\"temperature\":\"24.70\",\"LED\":\"OFF\",\"time\":1575018212,\"timestamp\":\"2019-11-29 18:03:32\"}]}"
	==================== FUNCTION LOG OUTPUT ====================
	START RequestId: e403e7bc-490b-465a-b7d8-b553029262c1 Version: $LATEST
	END RequestId: e403e7bc-490b-465a-b7d8-b553029262c1
	REPORT RequestId: e403e7bc-490b-465a-b7d8-b553029262c1	Duration: 156.98 ms	Billed Duration: 200 ms	Memory Size: 512 MB	Max Memory Used: 144 MB	
	
	``` 


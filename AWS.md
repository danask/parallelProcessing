네! `aws` 또는 `awslocal` CLI에서 **로컬 디렉토리를 S3로 통째로 업로드(copy)** 할 수 있고,
**서브디렉토리 구조도 유지**할 수 있는 커맨드가 있습니다. 아래처럼 사용하세요:

---

## ✅ `awslocal s3 cp`로 디렉토리 전체 복사 (서브디렉토리 포함)

```bash
awslocal s3 cp ./local-dir s3://your-bucket-name/ --recursive
```

* `./local-dir` → 복사할 **로컬 디렉토리 경로**
* `s3://your-bucket-name/` → 복사할 **S3 버킷 경로**
* `--recursive` → **하위 디렉토리 포함해서 전체 복사**

---

### 📂 예시: 로컬 구조

```
./local-dir/
├── a.txt
├── sub/
│   └── b.txt
```

### 📦 실행

```bash
awslocal s3 cp ./local-dir s3://test-bucket/ --recursive
```

### 📦 결과 (`awslocal s3 ls s3://test-bucket/ --recursive`)

```
2025-06-06 16:45:00      123 a.txt
2025-06-06 16:45:01      456 sub/b.txt
```

> S3에 **서브디렉토리 경로 그대로 반영**됩니다.

---

## ✅ 반대로 S3 → 로컬 디렉토리로 복사할 때도

```bash
awslocal s3 cp s3://test-bucket/ ./downloaded/ --recursive
```

---

## 🎯 요약

| 작업             | 커맨드 예시                                                 |
| -------------- | ------------------------------------------------------ |
| 디렉토리 → S3 업로드  | `awslocal s3 cp ./local-dir s3://bucket/ --recursive`  |
| S3 → 디렉토리 다운로드 | `awslocal s3 cp s3://bucket/ ./local-dir/ --recursive` |

---

필요하시면:

* 특정 확장자만 복사 (`--exclude` / `--include`)
* 파일 존재 시 덮어쓰기 방지
* 업로드 후 URL 출력

등도 같이 도와드릴 수 있어요.


---

좋아요!
`LocalStack`이 정상 작동 중이라면 이제 아래처럼 S3 버킷에 **파일을 업로드하고**,
**Spring Boot 애플리케이션에서 해당 파일을 읽어들이는 기능**을 구현하실 수 있습니다.

---

## ✅ 전체 흐름 요약



1. S3에 버킷 만들기
2. S3에 파일 업로드 (`CLI` 또는 `Spring`에서)
3. Spring Boot에서 해당 파일 읽기
4. (선택) 다운로드하거나 처리

---

## 🔹 1. S3에 파일 업로드 (CLI에서 해보기)

```bash
brew intall awscli-local
awslocal s3 ls

echo "Hello from LocalStack!" > hello.txt

awslocal s3 mb s3://test-bucket
awslocal s3 cp hello.txt s3://test-bucket/hello.txt
```

> 이렇게 하면 `test-bucket`에 `hello.txt`라는 객체가 업로드됩니다.

---

## 🔹 2. Spring Boot에서 S3Client 구성 (이미 하셨다면 패스)

```java
@Configuration
public class AwsClientConfig {

    @Bean
    public S3Client s3Client() {
        return S3Client.builder()
                .endpointOverride(URI.create("http://localhost:4566"))
                .region(Region.US_EAST_1)
                .credentialsProvider(
                        StaticCredentialsProvider.create(AwsBasicCredentials.create("test", "test"))
                )
                .build();
    }
}
```

---

## 🔹 3. 업로드된 파일 읽기 (Spring Boot에서)

```java
@Service
public class S3FileService {

    private final S3Client s3Client;

    public S3FileService(S3Client s3Client) {
        this.s3Client = s3Client;
    }

    public String readFileFromS3(String bucketName, String key) {
        ResponseBytes<GetObjectResponse> objectBytes = s3Client.getObjectAsBytes(
                GetObjectRequest.builder()
                        .bucket(bucketName)
                        .key(key)
                        .build()
        );

        return objectBytes.asUtf8String(); // 문자열로 읽기
    }
}
```

---

## 🔹 4. Controller에서 호출하거나 테스트

### 테스트로 확인

```java
@SpringBootTest
class S3FileServiceTest {

    @Autowired
    S3FileService s3FileService;

    @Test
    void testReadFile() {
        String content = s3FileService.readFileFromS3("test-bucket", "hello.txt");
        System.out.println(content);
        assertEquals("Hello from LocalStack!", content.trim());
    }
}
```

---

## 🔹 5. 파일을 Spring에서 업로드하기 (프로그래밍 방식)

```java
public void uploadFile(String bucketName, String key, String content) {
    s3Client.putObject(
            PutObjectRequest.builder()
                    .bucket(bucketName)
                    .key(key)
                    .build(),
            RequestBody.fromString(content)
    );
}
```

---

## 🔍 파일 업로드 후 확인 CLI

```bash
awslocal s3 ls s3://test-bucket
```

결과 예시:

```
2025-06-06 16:30:00       24 hello.txt
```

---

## 🎁 보너스: Multipart 파일 업로드 예제 (Controller + S3)

```java
@RestController
@RequestMapping("/api/files")
public class FileUploadController {

    private final S3FileService s3FileService;

    public FileUploadController(S3FileService s3FileService) {
        this.s3FileService = s3FileService;
    }

    @PostMapping("/upload")
    public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) throws IOException {
        String key = file.getOriginalFilename();
        String content = new String(file.getBytes(), StandardCharsets.UTF_8);
        s3FileService.uploadFile("test-bucket", key, content);
        return ResponseEntity.ok("Uploaded: " + key);
    }
}
```

---

## 🧪 테스트 시 주의할 점

* LocalStack의 S3는 기본적으로 `http`입니다 (https 아님).
* 파일명이 중복되면 덮어씌워짐.
* 버킷을 먼저 생성하고 나서 파일을 업로드하세요.

---

필요하시면:

* `파일 리스트 API`
* `파일 삭제 API`
* `Testcontainers` 연동 예제

도 추가로 도와드릴 수 있어요.
어떤 방식으로 연동하실 계획인지(예: 프론트에서 파일 올리는 경우 등) 알려주시면 더 맞춤 예제 드릴게요!


-----

Spring Boot 애플리케이션에서 AWS Step Functions를 사용해 Athena 쿼리를 실행하고 그 응답을 받는 간단한 예제를 설명드리겠습니다. 이 과정은 다음 단계로 이루어집니다:

1. Athena 쿼리 실행을 Step Functions에서 시작
2. Step Functions에서 쿼리 결과를 기다리고 폴링
3. 쿼리 완료 후 결과를 반환

### 1. AWS SDK 의존성 추가

`pom.xml`에 AWS SDK와 Step Functions, Athena 의존성을 추가해야 합니다.

```xml
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-stepfunctions</artifactId>
    <version>1.12.300</version> <!-- 최신 버전 사용 -->
</dependency>

<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-athena</artifactId>
    <version>1.12.300</version> <!-- 최신 버전 사용 -->
</dependency>
```

### 2. Athena 쿼리 상태기계(State Machine) 정의

AWS 콘솔 또는 CloudFormation을 사용하여 Step Functions 상태기계를 생성합니다. 상태기계는 Athena 쿼리를 시작하고 결과를 폴링합니다.

#### 상태기계 정의 (예시)

```json
{
  "Comment": "Athena Query Execution using Step Functions",
  "StartAt": "StartAthenaQuery",
  "States": {
    "StartAthenaQuery": {
      "Type": "Task",
      "Resource": "arn:aws:states:::athena:startQueryExecution",
      "Parameters": {
        "QueryString": "SELECT * FROM your_table LIMIT 10",
        "QueryExecutionContext": {
          "Database": "your_database"
        },
        "ResultConfiguration": {
          "OutputLocation": "s3://your-output-bucket/"
        }
      },
      "Next": "WaitForQuery"
    },
    "WaitForQuery": {
      "Type": "Wait",
      "Seconds": 10,
      "Next": "CheckQueryStatus"
    },
    "CheckQueryStatus": {
      "Type": "Task",
      "Resource": "arn:aws:states:::athena:getQueryExecution",
      "Parameters": {
        "QueryExecutionId.$": "$.QueryExecutionId"
      },
      "End": true
    }
  }
}
```

이 상태기계는 Athena 쿼리를 실행하고 그 결과를 폴링하여 처리합니다.

### 3. Spring Boot에서 Step Functions 호출

이제 Spring Boot에서 Step Functions를 사용하여 Athena 쿼리를 실행하고 결과를 기다리는 코드를 작성합니다.

#### Step Functions 실행 서비스

```java
import com.amazonaws.services.stepfunctions.AWSStepFunctions;
import com.amazonaws.services.stepfunctions.AWSStepFunctionsClientBuilder;
import com.amazonaws.services.stepfunctions.model.StartExecutionRequest;
import com.amazonaws.services.stepfunctions.model.StartExecutionResult;
import org.springframework.stereotype.Service;

@Service
public class StepFunctionService {

    private final AWSStepFunctions stepFunctionsClient = AWSStepFunctionsClientBuilder.defaultClient();

    public String startStepFunction(String stateMachineArn, String query) {
        String input = String.format("{\"QueryString\": \"%s\"}", query);

        StartExecutionRequest request = new StartExecutionRequest()
                .withStateMachineArn(stateMachineArn)
                .withInput(input);

        StartExecutionResult result = stepFunctionsClient.startExecution(request);
        return result.getExecutionArn(); // 실행된 Step Function의 ARN을 반환
    }
}
```

#### Athena 쿼리 실행 및 결과 대기 컨트롤러

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AthenaQueryController {

    private final StepFunctionService stepFunctionService;

    public AthenaQueryController(StepFunctionService stepFunctionService) {
        this.stepFunctionService = stepFunctionService;
    }

    @GetMapping("/execute-query")
    public String executeAthenaQuery(@RequestParam String query) {
        // Step Functions 상태기계 ARN을 입력
        String stateMachineArn = "arn:aws:states:your-region:your-account-id:stateMachine:your-state-machine-name";

        // Athena 쿼리를 실행하고 Step Functions를 통해 처리
        String executionArn = stepFunctionService.startStepFunction(stateMachineArn, query);

        return "Step Function execution started with ARN: " + executionArn;
    }
}
```

### 4. Step Functions 결과 확인

Step Functions에서 `executionArn`을 사용하여 실행 상태를 확인할 수 있습니다. 이를 통해 Athena 쿼리의 실행이 완료되었는지 확인합니다. 필요에 따라 결과를 S3 버킷에서 가져와 사용할 수 있습니다.

### 요약

이 예제에서는 Spring Boot에서 Step Functions를 사용하여 Athena 쿼리를 비동기로 실행하고, 쿼리 완료 여부를 Step Functions 상태기계를 통해 확인하는 방법을 설명했습니다.

--------------------------

AWS LocalStack은 로컬 환경에서 AWS 서비스를 에뮬레이트하는 도구로, 실제 AWS 서비스를 사용하는 것과 비슷한 환경을 제공합니다. 여기서는 Spring Boot 서비스에서 AWS LocalStack을 사용하여 S3에 업로드된 CSV 파일을 읽고 파싱하는 로직을 만드는 과정을 안내해 드리겠습니다.

먼저, 프로젝트에 필요한 의존성을 추가해야 합니다. Maven을 사용하는 경우 `pom.xml` 파일에 다음 의존성을 추가할 수 있습니다.

```xml
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-s3</artifactId>
    <version>1.11.1013</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

그런 다음, AWS S3 서비스에 연결하고 파일을 읽고 파싱하는 Spring Boot 서비스를 만들어야 합니다. 아래는 간단한 예제 코드입니다.

```java
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.model.GetObjectRequest;
import com.amazonaws.services.s3.model.S3Object;
import com.amazonaws.services.s3.model.S3ObjectInputStream;
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVRecord;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.List;

@RestController
public class S3Controller {

    @Autowired
    private AmazonS3 amazonS3;

    @GetMapping("/read-csv-from-s3")
    public void readCSVFromS3(@RequestParam String bucketName, @RequestParam String fileName) {
        try {
            S3Object s3Object = amazonS3.getObject(new GetObjectRequest(bucketName, fileName));
            S3ObjectInputStream inputStream = s3Object.getObjectContent();
            
            // CSV 파싱
            InputStreamReader reader = new InputStreamReader(inputStream);
            CSVParser csvParser = new CSVParser(reader, CSVFormat.DEFAULT);
            List<CSVRecord> records = csvParser.getRecords();
            
            // CSV 데이터 처리
            for (CSVRecord record : records) {
                // 각 레코드 처리
                String column1 = record.get(0);
                String column2 = record.get(1);
                // 필요한 로직 수행
            }
            
            // 리소스 해제
            csvParser.close();
            reader.close();
            inputStream.close();
        } catch (IOException e) {
            e.printStackTrace();
            // 예외 처리
        }
    }
}
```

위의 코드에서 `/read-csv-from-s3` 엔드포인트는 S3 버킷에서 CSV 파일을 읽고 파싱하는 엔드포인트를 나타냅니다. `bucketName`과 `fileName`은 요청 매개변수로 전달됩니다. 이 코드를 실행하기 위해서는 `AmazonS3` 클라이언트가 주입되어야 합니다. 이를 위해서는 AWS SDK를 설정하고, AWS LocalStack의 S3 엔드포인트와 연결해야 합니다.

AWS SDK를 사용하여 AWS LocalStack의 S3 서비스에 연결하려면, AWS S3 클라이언트를 빈으로 등록하고 AWS LocalStack의 S3 엔드포인트를 사용하도록 구성해야 합니다. 이를 위해서는 `application.properties` 파일에 AWS S3 설정을 추가해야 합니다.

```properties
# AWS S3 설정
cloud.aws.region.static=us-east-1
cloud.aws.credentials.accessKey=test
cloud.aws.credentials.secretKey=test
cloud.aws.s3.endpoint=http://localhost:4566
```

위의 설정에서 `endpoint` 속성은 AWS LocalStack의 S3 엔드포인트 주소로 설정됩니다. 여기서는 기본 포트 4566을 사용합니다. 또한, AWS 액세스 키와 시크릿 키는 테스트용으로 임의의 값으로 설정되어 있습니다. 실제 환경에서는 액세스 키와 시크릿 키를 보안에 주의하여 설정해야 합니다.

이제 Spring Boot 애플리케이션을 실행하고 `/read-csv-from-s3` 엔드포인트를 호출하여 S3에서 CSV 파일을 읽고 파싱할 수 있습니다. 이를 통해 AWS LocalStack을 사용하여 로컬에서 S3와 상호작용하는 Spring Boot 서비스를 구축할 수 있습니다.


AWS LocalStack을 사용하여 스프링 부트 서비스에서 S3에 파일을 읽는 것을 테스트하려면 몇 가지 단계를 따라야 합니다. 여기에 대략적인 절차를 안내해 드리겠습니다.

1. **LocalStack 설치**: 먼저 LocalStack을 설치하고 실행해야 합니다. 이를 위해 Docker를 사용하면 편리합니다. Docker를 설치한 후에는 LocalStack을 실행할 수 있습니다.

2. **스프링 부트 프로젝트 설정**: 프로젝트에 AWS SDK를 추가하고, LocalStack의 S3 엔드포인트 및 자격 증명을 사용하도록 구성해야 합니다. `application.properties`나 `application.yml` 파일을 사용하여 AWS 자격 증명 및 S3 엔드포인트를 구성할 수 있습니다.

    예를 들어, `application.yml` 파일에 다음과 같은 설정을 추가할 수 있습니다:
    ```yaml
    cloud:
      aws:
        credentials:
          accessKey: test
          secretKey: test
        region:
          static: us-east-1
        s3:
          endpoint: http://localhost:4566
    ```

3. **테스트 코드 작성**: 스프링 부트 서비스에서 S3에 파일을 읽는 코드에 대한 테스트를 작성해야 합니다. 이를 위해 JUnit 또는 TestNG와 같은 테스트 프레임워크를 사용할 수 있습니다. 테스트 코드에서는 LocalStack의 S3 서비스를 사용하여 파일을 읽어와야 합니다.

    예를 들어, 다음은 S3 버킷에서 파일을 읽는 메서드를 포함한 테스트 클래스의 예입니다:
    ```java
    import org.junit.jupiter.api.Test;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import software.amazon.awssdk.core.ResponseInputStream;
    import software.amazon.awssdk.services.s3.S3Client;
    import software.amazon.awssdk.services.s3.model.GetObjectRequest;
    import software.amazon.awssdk.services.s3.model.S3Object;
    
    import static org.junit.jupiter.api.Assertions.assertNotNull;
    
    @SpringBootTest
    public class S3ServiceTest {
    
        @Autowired
        private S3Client s3Client;
        
        @Test
        public void testReadFileFromS3() {
            String bucketName = "your-bucket-name";
            String key = "your-file-key";
            
            GetObjectRequest getObjectRequest = GetObjectRequest.builder()
                    .bucket(bucketName)
                    .key(key)
                    .build();
    
            ResponseInputStream<S3Object> objectResponseInputStream = s3Client.getObject(getObjectRequest);
            assertNotNull(objectResponseInputStream);
            
            // 여기서부터는 파일을 읽는 로직에 대한 테스트를 진행할 수 있습니다.
        }
    }
    ```

4. **테스트 실행**: 테스트를 실행하여 LocalStack을 통해 S3에 파일을 읽는 동작이 예상대로 수행되는지 확인합니다.

이러한 단계를 따라 LocalStack을 사용하여 스프링 부트 서비스에서 S3에 파일을 읽는 것을 테스트할 수 있습니다.



로컬에 구성된 Docker LocalStack 컨테이너에 파일을 복사하는 것은 다음과 같은 절차를 따릅니다.

1. **Docker LocalStack 컨테이너에 파일을 복사하는 방법**
   
   일반적으로 Docker 컨테이너에 파일을 복사하는 방법은 다음과 같습니다.

   ```bash
   docker cp /로컬/파일/경로 컨테이너이름:/컨테이너/내부/경로
   ```

   예를 들어, 로컬에 있는 `example.csv` 파일을 Docker LocalStack 컨테이너 내부의 `/tmp` 디렉터리로 복사하려면 다음 명령어를 사용할 수 있습니다.

   ```bash
   docker cp example.csv localstack:/tmp
   ```

   이제 `example.csv` 파일이 Docker LocalStack 컨테이너 내부의 `/tmp` 디렉터리에 복사되었습니다.

2. **LocalStack에서 S3로 파일 업로드**

   이제 복사된 파일을 LocalStack의 S3에 업로드할 수 있습니다. 이 작업은 Spring Boot 서비스에서 LocalStack의 S3 클라이언트를 사용하여 수행할 수 있습니다. 예를 들어, Java에서는 다음과 같이 업로드할 수 있습니다.

   ```java
   import com.amazonaws.services.s3.AmazonS3;
   import com.amazonaws.services.s3.model.PutObjectRequest;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.stereotype.Service;
   import java.io.File;

   @Service
   public class S3Service {

       @Autowired
       private AmazonS3 amazonS3;

       public void uploadFileToS3(String bucketName, String key, File file) {
           amazonS3.putObject(new PutObjectRequest(bucketName, key, file));
       }
   }
   ```

   위의 코드에서 `uploadFileToS3` 메서드를 호출하여 로컬 파일을 LocalStack의 S3에 업로드할 수 있습니다.

3. **Spring Boot에서 파일 업로드**

   Spring Boot에서는 위에서 작성한 `S3Service`를 사용하여 파일을 업로드할 수 있습니다.

   ```java
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.web.bind.annotation.PostMapping;
   import org.springframework.web.bind.annotation.RequestParam;
   import org.springframework.web.bind.annotation.RestController;
   import org.springframework.web.multipart.MultipartFile;

   @RestController
   public class FileUploadController {

       @Autowired
       private S3Service s3Service;

       @PostMapping("/upload")
       public String uploadFile(@RequestParam("file") MultipartFile file) {
           // 로컬 파일을 S3에 업로드
           s3Service.uploadFileToS3("your-bucket-name", "your-key-name", convertMultiPartFileToFile(file));
           return "File uploaded successfully!";
       }

       // MultipartFile을 File로 변환하는 메서드
       private File convertMultiPartFileToFile(MultipartFile file) {
           // 구현 필요
       }
   }
   ```

   위의 코드에서 `/upload` 엔드포인트를 호출하여 로컬 파일을 LocalStack의 S3에 업로드할 수 있습니다. `MultipartFile`을 `File`로 변환하는 방법은 구현해야 하며, 일반적으로 `multipartFile.transferTo(new File("원하는_파일_경로"))` 메서드를 사용하여 변환할 수 있습니다.

이제 이러한 절차를 따라서 파일을 Docker LocalStack 컨테이너에 복사하고, LocalStack의 S3에 업로드할 수 있습니다.


------------------------------

가상으로 Kafka 이벤트를 테스트하는 것은 흔한 시나리오입니다. 이를 위해 대부분의 경우 내장 Kafka 인스턴스 또는 모의(Mock) Kafka 클러스터를 사용합니다. 여기에는 몇 가지 단계가 포함됩니다.

1. **의존성 추가**: 먼저 프로젝트에 Kafka 클라이언트 라이브러리를 추가해야 합니다. 이는 Maven이나 Gradle과 같은 빌드 관리 도구를 사용하여 수행됩니다.

2. **테스트 환경 설정**: 테스트 환경에서는 내장 Kafka 서버를 사용하거나, 모의(Mock) Kafka 클러스터를 실행해야 합니다. 내장 Kafka를 사용하는 경우 테스트 프로퍼티 파일에 Kafka 브로커의 위치와 관련된 설정을 추가해야 합니다.

    예를 들어, `application.yml` 파일에 내장 Kafka를 사용하는 경우 다음과 같이 설정할 수 있습니다:
    ```yaml
    spring:
      kafka:
        bootstrap-servers: localhost:9092
    ```

3. **Kafka 이벤트 생성**: Kafka 이벤트를 생성하여 테스트하는 로직을 작성합니다. 이는 실제로 메시지를 보내는 프로듀서 코드를 포함할 수 있습니다.

4. **테스트 코드 작성**: Kafka 이벤트를 테스트하는 코드를 작성합니다. 테스트 코드에서는 모의(Mock) Kafka 클러스터나 내장 Kafka를 사용하여 메시지를 송수신하는 작업을 수행해야 합니다.

    예를 들어, 모의(Mock) Kafka 클러스터를 사용하는 테스트 코드는 다음과 같을 수 있습니다:
    ```java
    import org.apache.kafka.clients.consumer.ConsumerRecord;
    import org.junit.jupiter.api.Test;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.kafka.core.KafkaTemplate;
    import org.springframework.kafka.listener.ConcurrentMessageListenerContainer;
    import org.springframework.kafka.listener.MessageListener;
    import org.springframework.kafka.test.context.EmbeddedKafka;
    
    import java.util.concurrent.BlockingQueue;
    import java.util.concurrent.LinkedBlockingQueue;
    import java.util.concurrent.TimeUnit;
    
    import static org.junit.jupiter.api.Assertions.assertEquals;
    
    @SpringBootTest
    @EmbeddedKafka(partitions = 1, topics = "test-topic")
    public class KafkaEventTest {
    
        @Autowired
        private KafkaTemplate<String, String> kafkaTemplate;
    
        private BlockingQueue<ConsumerRecord<String, String>> records = new LinkedBlockingQueue<>();
    
        @Autowired
        private ConcurrentMessageListenerContainer<String, String> container;
    
        @Test
        public void testSendReceiveKafkaMessage() throws Exception {
            container.setupMessageListener((MessageListener<String, String>) records::add);
    
            kafkaTemplate.send("test-topic", "test-message");
    
            ConsumerRecord<String, String> received = records.poll(10, TimeUnit.SECONDS);
            assertEquals("test-message", received.value());
        }
    }
    ```

5. **테스트 실행**: 테스트를 실행하여 Kafka 이벤트가 예상대로 처리되는지 확인합니다.

이러한 단계를 따라 가상으로 Kafka 이벤트를 테스트할 수 있습니다.


로컬 환경에서 Kafka 이벤트를 받아서 Spring Boot에서 처리하는 방법은 비슷합니다. 먼저 Kafka를 로컬에서 실행하고, Spring Boot 애플리케이션을 작성하여 Kafka 이벤트를 수신하고 처리하는 코드를 구현해야 합니다.

아래는 이를 수행하기 위한 간단한 예제 코드입니다. 먼저, Kafka를 로컬에서 실행하고, Spring Boot 애플리케이션에서 Kafka 이벤트를 수신하는 리스너를 작성하겠습니다.

1. Kafka 서버 실행:

Kafka를 로컬에서 실행하고 토픽을 생성합니다. 이 예제에서는 Docker를 사용하여 Kafka를 실행하는 방법을 안내하겠습니다.

```bash
docker run -d --rm \
    --name kafka \
    -p 9092:9092 \
    -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
    -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
    wurstmeister/kafka:2.12-2.8.0
```

2. Spring Boot 애플리케이션 작성:

이제 Spring Boot 애플리케이션에서 Kafka 이벤트를 수신하고 처리하는 코드를 작성합니다.

```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

@Service
public class KafkaConsumer {

    @KafkaListener(topics = "test-topic", groupId = "group-id")
    public void listen(String message) {
        // Kafka 이벤트 처리 로직 작성
        System.out.println("Received Kafka message: " + message);
    }
}
```

위의 코드에서 `@KafkaListener` 어노테이션은 `test-topic` 토픽에서 메시지를 수신하고, `listen` 메서드가 이벤트를 처리합니다. 이 예제에서는 간단히 받은 메시지를 콘솔에 출력하는 것으로 처리하였습니다.

3. Kafka Producer 구현 (옵션):

이벤트를 Kafka 토픽으로 보내려면 Kafka Producer를 구현해야 합니다. 아래는 Kafka Producer를 간단히 구현하는 예제입니다.

```java
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

@Component
public class KafkaProducer {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public KafkaProducer(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendMessage(String message) {
        kafkaTemplate.send("test-topic", message);
    }
}
```

위의 코드에서 `KafkaTemplate`을 사용하여 `test-topic` 토픽으로 메시지를 보내는 `sendMessage` 메서드를 구현하였습니다.

이제 Spring Boot 애플리케이션을 실행하고 Kafka로부터 메시지를 수신하려면 Kafka Producer를 통해 메시지를 보내고, Spring Boot 애플리케이션은 해당 메시지를 수신하여 처리합니다. 이를 통해 로컬 환경에서 Kafka 이벤트를 처리하는 Spring Boot 애플리케이션을 구축할 수 있습니다.

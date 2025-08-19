Studio 3T에서 일정 시간이 지나면 자동 로그인이 차단되는 현상은 주로 **MongoDB의 인증 세션 타임아웃** 또는 **Studio 3T 자체 설정**과 관련이 있습니다. 해결하려면 아래 두 가지 원인을 확인하세요.

---

### ✅ 1. **MongoDB 세션 타임아웃 설정 확인**

MongoDB는 SCRAM 인증 방식에서 **default로 세션 타임아웃**이 있습니다. 특히 **MongoDB Enterprise**나 **Atlas** 환경에서 `authMechanismProperties`에 따라 세션이 만료될 수 있습니다.

* **해결 방법**:

  * **`authMechanismProperties`에 `SERVICE_REALM` 또는 `SASL_PLAIN` 사용 시 Kerberos 등에서 세션 만료가 생김.**
  * **Atlas나 클라우드 환경**에서는 `Idle Session Timeout` 설정이 기본 적용 → UI에서 해제 불가.
  * **온프레미스 MongoDB**는 `--setParameter authenticationMechanisms=SCRAM-SHA-1,SCRAM-SHA-256` 옵션 사용 시 타임아웃 영향.
  * 이 경우 DB 쪽에서 세션 제한을 늘리거나 없애야 함.
    `mongod.conf` 예시:

    ```yaml
    setParameter:
      authenticationMechanisms: SCRAM-SHA-1,SCRAM-SHA-256
      saslauthdIdleTimeout: 86400  # 초 단위 (24시간)
    ```

---

### ✅ 2. **Studio 3T 자동 로그인 유지 설정**

Studio 3T는 기본적으로 **Connection Manager**에서 `Save Password`를 체크하지 않으면 로그인 유지가 안 됩니다.
추가로, **비밀번호를 암호화하여 로컬에 저장**할지 여부는 Preferences에서 관리합니다.

* **방법**:

  1. **Connection Manager** 열기 (`Ctrl` + `Shift` + `C`)
  2. 사용하는 Connection 선택 → `Edit`
  3. **Authentication** 탭 → `Save Password` 체크
  4. **SSL** / **SSH** 터널 사용 시 → Key Passphrase 저장도 체크
  5. **Preferences → Security → Auto-lock application after X minutes** 설정 해제

     * `Auto-lock application`이 활성화되어 있으면 일정 시간 후 세션이 만료됩니다.
  6. 비밀번호 저장 시 Studio 3T가 로컬에 AES 암호화로 저장합니다.

---

### ✅ 3. **OS 정책 (윈도우, 맥) 영향**

* 회사 정책으로 **Windows Credential Manager**나 **macOS Keychain**이 일정 시간 후 세션을 끊을 수도 있음.
* 이 경우 Studio 3T는 비밀번호를 저장했더라도 인증 토큰이 만료될 수 있음 → OS 보안 정책 확인 필요.

---

### ✅ 핵심 해결 포인트

* Studio 3T에서 `Save Password` + `Preferences → Security → Auto-lock` 끄기
* MongoDB 서버 세션 타임아웃 → `saslauthdIdleTimeout` 등 파라미터 조정
* 회사 보안 정책에서 Keychain/Credential 제한이 있는지 확인

---

필요하면 **Studio 3T에서 완전 자동 로그인 (비밀번호 입력 없이)** 되는 설정법과
**Atlas 환경에서 Timeout 안 걸리게 유지하는 방법**도 상세히 정리해줄까요?  (Atlas는 특히 `Idle Session Timeout` 강제 정책이 있어서 우회 방법이 따로 있습니다.)


----

IntelliJ IDEA에서 **"Search for class"**에 해당 클래스가 나타나지만, 의존성을 추가해도 프로젝트에서 사용되지 않는 경우에는 몇 가지 원인과 해결 방법이 있을 수 있습니다. 다음 해결책들을 순서대로 시도해 보세요.

### 1. **Maven Reimport (의존성 강제 재설정)**
먼저 IntelliJ에서 Maven 의존성을 강제로 다시 가져오는 작업을 시도하세요.

#### 해결 방법:
- IntelliJ IDEA 상단의 `View` → `Tool Windows` → `Maven` 창을 엽니다.
- `Maven` 창에서 프로젝트 이름에 마우스 오른쪽 클릭하고, `Reimport`를 선택합니다.
  
이렇게 하면 `pom.xml` 파일에 추가된 의존성들이 강제로 다시 가져와집니다.

### 2. **의존성 범위 확인 (`pom.xml`에서 scope 확인)**
`pom.xml` 파일에 추가된 의존성의 범위(`scope`)가 제한적일 수 있습니다. 예를 들어, 의존성 범위가 `test`나 `provided`로 설정된 경우, 해당 라이브러리는 특정 환경에서만 사용됩니다.

#### 해결 방법:
- `pom.xml` 파일에서 의존성 범위가 `test`, `provided` 등으로 설정되어 있다면 이를 제거하거나 `compile`로 변경해 보세요.
  
```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>your-library</artifactId>
  <version>1.0.0</version>
  <scope>compile</scope> <!-- 이 부분을 확인 -->
</dependency>
```

### 3. **의존성 충돌 문제**
다른 라이브러리와 버전 충돌이 발생할 수 있습니다. 특정 버전의 라이브러리가 프로젝트에 포함되지 않거나, 잘못된 버전이 추가될 수 있습니다.

#### 해결 방법:
- 터미널에서 다음 명령을 사용해 의존성 트리를 확인하고 충돌을 해결할 수 있습니다:
  ```bash
  mvn dependency:tree
  ```
  이 명령어를 통해 동일한 라이브러리의 버전 충돌을 확인하고, 필요한 경우 `pom.xml`에서 명시적으로 버전을 설정하여 문제를 해결합니다.

### 4. **로컬 Maven 캐시 문제**
로컬 `.m2` 캐시가 손상되었거나 라이브러리가 잘못 다운로드된 경우에도 문제가 발생할 수 있습니다.

#### 해결 방법:
- `.m2/repository` 폴더에서 해당 라이브러리 폴더를 삭제한 후, 다시 의존성을 다운로드하세요.
- IntelliJ에서 `mvn clean install` 명령을 실행하여 다시 빌드해 봅니다.

### 5. **프로젝트 구조 및 모듈 설정 확인**
의존성은 추가되었지만, 프로젝트의 모듈 설정에서 해당 라이브러리가 포함되지 않을 수 있습니다.

#### 해결 방법:
- `File` → `Project Structure` → `Modules`로 이동하여, `Dependencies` 탭에서 필요한 라이브러리가 추가되어 있는지 확인합니다.
- 라이브러리가 추가되어 있지 않다면, 여기서 수동으로 추가할 수 있습니다.

### 6. **IntelliJ 캐시 무효화 및 재시작**
IntelliJ IDEA의 캐시 문제로 인해 의존성이 제대로 반영되지 않을 수 있습니다.

#### 해결 방법:
- `File` → `Invalidate Caches / Restart` → `Invalidate and Restart`를 선택하여 캐시를 무효화하고 IDE를 다시 시작합니다.

### 7. **Maven 전역 설정 확인**
Maven의 전역 설정(`~/.m2/settings.xml`)이나 `pom.xml`에서 저장소 설정이 잘못되어 있으면 의존성이 제대로 다운로드되지 않거나 반영되지 않을 수 있습니다.

#### 해결 방법:
- `~/.m2/settings.xml` 또는 프로젝트의 `pom.xml`에서 Maven 저장소 설정이 올바른지 확인합니다.
  
  기본적으로 Maven Central을 사용하고 있지 않다면, 아래와 같이 `pom.xml`에 중앙 저장소를 추가합니다:
  
  ```xml
  <repositories>
    <repository>
      <id>central</id>
      <url>https://repo.maven.apache.org/maven2</url>
    </repository>
  </repositories>
  ```

### 8. **특정 라이브러리 문제 (사설 저장소)**
사설 저장소에서 제공되는 라이브러리나 커스텀 라이브러리의 경우, 배포가 제대로 이루어지지 않아 Maven이 해당 라이브러리를 찾지 못할 수도 있습니다.

#### 해결 방법:
- 라이브러리의 출처가 사설 저장소일 경우, 해당 저장소 설정이 제대로 되어 있는지 확인하세요. 또는 라이브러리 배포가 제대로 되었는지 확인해야 합니다.

---

위의 해결 방법을 단계적으로 진행하면, 의존성을 추가했음에도 불구하고 IntelliJ에서 제대로 인식하지 않는 문제를 해결할 수 있습니다.

--------------------------


`.m2` 폴더에 `lastUpdated` 파일이 생성되었다는 것은 해당 라이브러리가 제대로 다운로드되지 않았거나 다운로드 중 문제가 발생했음을 의미합니다. 이 때문에 프로젝트의 External Libraries에서도 해당 라이브러리가 보이지 않을 수 있습니다. 이를 해결하기 위한 몇 가지 방법을 제안합니다.

-----------------------

### 1. **Maven 의존성 강제 업데이트**
`lastUpdated` 파일이 생성된 경우 해당 의존성을 강제로 다시 다운로드할 수 있습니다.

- IntelliJ의 터미널에서 다음 명령을 실행하세요:
  ```bash
  mvn clean install -U
  ```
  `-U` 옵션은 Maven이 의존성을 강제로 다시 업데이트하도록 합니다.

### 2. **손상된 의존성 삭제**
- `.m2/repository` 폴더 내에서 문제가 발생한 라이브러리의 캐시를 삭제하고 다시 다운로드를 시도해 볼 수 있습니다.
  1. 해당 의존성의 경로로 이동 (`.m2/repository/그룹아이디/아티팩트아이디/버전`).
  2. 해당 폴더를 삭제합니다.
  3. 터미널에서 다시 Maven을 실행하여 라이브러리를 재다운로드합니다:
     ```bash
     mvn clean install
     ```

### 3. **Maven Repositories 설정 확인**
- `pom.xml`에 정의된 Maven 저장소가 불안정하거나 접근할 수 없는 경우에도 `lastUpdated` 파일이 생길 수 있습니다. 이 경우 저장소 설정을 확인하고, 문제 해결을 위해 다른 Maven 저장소(예: Maven Central)를 추가할 수 있습니다.

#### 예시:
```xml
<repositories>
  <repository>
    <id>central</id>
    <url>https://repo.maven.apache.org/maven2</url>
  </repository>
</repositories>
```

### 4. **프로젝트 구조의 의존성 확인**
Maven 의존성 파일이 제대로 다운로드되지 않으면 IntelliJ에서 프로젝트 구조에 나타나지 않습니다.

1. IntelliJ IDEA에서 `View` → `Tool Windows` → `Maven` 선택.
2. `Maven` 창에서 `Dependencies`를 확인하고, 의존성이 있는지 확인합니다.
3. `pom.xml`에 문제가 있는 경우, IntelliJ가 경고를 표시하므로 의존성 설정을 다시 한번 확인하세요.

### 5. **네트워크 설정 확인**
간혹 네트워크 문제로 인해 Maven이 의존성을 다운로드하지 못하는 경우도 있습니다. 이때 프록시 설정 등을 확인하세요.

- IntelliJ에서 `File` → `Settings` → `Build, Execution, Deployment` → `Build Tools` → `Maven` → `Importing`에서 네트워크 설정을 확인합니다.

### 6. **Maven Reimport 시도**
- IntelliJ에서 `Reimport`를 다시 시도하여 의존성을 강제로 불러옵니다.
  - `Maven` 창에서 프로젝트 이름에 마우스 우클릭 → `Reimport` 선택.

### 7. **Maven 전역 설정 확인**
- Maven 전역 설정(`~/.m2/settings.xml`)에 문제가 있을 수 있습니다. 프록시나 저장소 설정이 잘못되어 있으면 의존성 다운로드가 실패할 수 있습니다.

위 단계를 수행하면 `lastUpdated` 문제를 해결하고 의존성을 프로젝트 내에서 정상적으로 사용할 수 있을 것입니다.

---------------------

IntelliJ IDEA에서 Maven 기반 프로젝트의 경우, `m2` 폴더에 라이브러리가 다운로드되었지만 프로젝트 내에서 라이브러리가 보이지 않는 문제는 여러 원인이 있을 수 있습니다. 다음 단계를 통해 해결해 보세요.

### 1. **Maven Reimport 시도**
- IntelliJ IDEA에서 프로젝트가 Maven 의존성을 제대로 읽지 못할 수 있습니다. 이를 해결하려면 Maven 프로젝트를 다시 불러오십시오.
  - IntelliJ 상단의 `View` → `Tool Windows` → `Maven` 선택.
  - `Maven` 창에서 프로젝트 이름에 마우스 우클릭 → `Reimport` 선택.

### 2. **의존성 업데이트**
- 터미널에서 `mvn clean install` 명령을 실행하거나, IntelliJ 내에서 `mvn clean install`을 실행하여 의존성을 업데이트하세요.

### 3. **프로젝트 구조 확인**
- IntelliJ에서 `File` → `Project Structure`로 이동.
- `Modules` 탭에서 `Dependencies` 항목을 확인하고, 필요한 Maven 라이브러리가 목록에 포함되어 있는지 확인하세요.

### 4. **Maven Repositories 설정 확인**
- 프로젝트의 `pom.xml` 파일에서 라이브러리를 참조하는 Maven 저장소가 제대로 설정되어 있는지 확인하세요. 혹시 Maven 저장소가 제대로 설정되어 있지 않으면 라이브러리를 다운로드할 수 없습니다.

### 5. **IntelliJ 캐시 무효화 및 재시작**
- IntelliJ 캐시 문제로 인해 라이브러리가 보이지 않을 수 있습니다.
  - `File` → `Invalidate Caches` → `Invalidate and Restart` 선택.

### 6. **IDE 환경설정 확인**
- IntelliJ IDEA의 `Settings` → `Build, Execution, Deployment` → `Build Tools` → `Maven`에서 `Local Repository` 경로가 올바른지 확인합니다.

위 단계를 진행한 후에도 해결되지 않으면, `pom.xml` 파일이나 라이브러리 의존성 문제일 가능성이 있으니, `pom.xml` 내 의존성 설정을 다시 확인해 보세요.

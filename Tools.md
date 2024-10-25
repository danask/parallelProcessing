
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

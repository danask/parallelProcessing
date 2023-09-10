https://m.blog.naver.com/PostView.naver?blogId=writer0713&logNo=222249266550&proxyReferer=https:%2F%2Fl.facebook.com%2F

IntelliJ IDEA에서 Spring Boot 프로젝트에서 사용되는 공통 패키지를 라이브러리로 만들려면 다음 단계를 따를 수 있습니다. 이것은 IntelliJ IDEA에서 진행하는 일반적인 방법입니다.

1. IntelliJ IDEA에서 새 프로젝트를 엽니다.

2. "File" 메뉴에서 "New"를 선택한 다음 "Module"을 선택합니다.

3. "Create New Module" 대화 상자가 열립니다. 여기서 "Java"를 선택하고 "Next"를 클릭합니다.

4. 프로젝트의 공통 패키지 및 클래스가 포함된 디렉토리를 지정하십시오. 이 디렉토리를 라이브러리 모듈로 사용하려는 패키지와 클래스로 채워넣으십시오.

5. 다음 화면에서 모듈의 이름을 입력하고 "Finish"를 클릭합니다. 모듈 이름은 나중에 라이브러리로 참조할 때 사용됩니다.

6. 이제 프로젝트에 공통 모듈이 추가되었습니다. 이 모듈을 라이브러리로 사용하려면 다음 단계로 진행하십시오.

7. 프로젝트의 다른 모듈에서 이 공통 모듈을 사용하려면 해당 모듈을 선택한 다음 "File" 메뉴에서 "Project Structure"를 선택합니다.

8. 왼쪽 창에서 "Modules"을 선택한 다음 오른쪽 창에서 "Dependencies" 탭을 선택합니다.

9. "+" 버튼을 클릭하고 "Module Dependency"를 선택합니다. 그런 다음 사용할 공통 모듈을 선택합니다.

10. "OK"를 클릭하여 변경 사항을 저장합니다.

이제 다른 Spring Boot 모듈에서 공통 패키지 및 클래스를 사용할 수 있게 됩니다. 이 모듈은 다른 모듈에서 의존성으로 추가되어 공통 코드를 재사용할 수 있게 됩니다.

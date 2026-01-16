## VS Code와 Git 연동 및 저장소 연결

Visual Studio Code(VS Code)는 Git 제어 기능을 내장하고 있어, 터미널 명령어를 일일이 입력하지 않고도 소스 코드의 변경 사항을 추구하고 원격 저장소(GitHub 등)와 동기화할 수 있습니다.

---

### 1. 사전 준비 사항

연동을 시작하기 전, 시스템에 Git이 설치되어 있어야 합니다.

1. **Git 설치 확인**: 터미널(또는 CMD)에서 `git --version`을 입력합니다.
2. **사용자 설정**: 설치 후 처음이라면 사용자 이름과 이메일을 등록해야 합니다.
```bash
git config --global user.name "Your Name"
git config --global user.email "your_email@example.com"

```



---

### 2. VS Code에서 로컬 저장소 초기화

작업 중인 프로젝트 폴더를 Git 저장소로 만드는 과정입니다.

1. VS Code 왼쪽 사이드바에서 **Source Control(소스 제어)** 아이콘(세 개의 점과 선 모양)을 클릭합니다.
2. **Initialize Repository(저장소 초기화)** 버튼을 클릭합니다.
3. 폴더 내 파일들이 'U'(Untracked) 상태로 변하며 Git 관리 대상이 됩니다.

---

### 3. 원격 저장소(GitHub) 연결하기

로컬에서 작업한 코드를 온라인 저장소에 업로드하기 위한 연결 단계입니다.

#### 3.1. GitHub 계정 로그인 방식 (권장)

1. 사이드바의 **Source Control** 탭으로 이동합니다.
2. **Publish to GitHub** 버튼을 클릭합니다.
3. 상단 팝업에서 GitHub 계정 로그인을 진행하고, 저장소 이름을 입력한 뒤 'Public' 또는 'Private'을 선택합니다.

#### 3.2. 기존 원격 저장소 URL 연결 방식

이미 GitHub에 저장소를 생성했다면 URL을 직접 등록할 수 있습니다.

1. VS Code 터미널(`Ctrl + ` `)을 엽니다.
2. 아래 명령어를 입력하여 로컬과 원격(Remote)을 연결합니다.
```bash
git remote add origin https://github.com/사용자이름/저장소이름.git

```



---

### 4. 핵심 워크플로우: Add, Commit, Push

VS Code UI를 통해 명령어를 대신하는 방법입니다.

1. **Stage (Add)**: 변경된 파일 옆의 **+** 아이콘을 눌러 'Staged Changes' 상태로 올립니다.
2. **Commit**: 상단 텍스트 박스에 커밋 메시지(예: "Initial commit")를 적고 **Commit** 버튼을 누릅니다.
3. **Push**: 하단의 **Sync Changes** 또는 **...** 메뉴에서 **Push**를 선택하여 GitHub로 업로드합니다.

---

### 5. AWS 환경에서의 Git 사용 시 비용 및 보안 주의사항

Node.js 프로젝트 등을 AWS와 연동할 때 주의할 점입니다.

* **비밀번호 노출 금지 (중요)**: AWS Access Key나 데이터베이스 비밀번호가 포함된 `.env` 파일이 Git에 업로드되지 않도록 주의하십시오. 반드시 `.gitignore` 파일에 `.env`를 추가해야 합니다.
* **AWS CodeCommit (유료 전환 유의)**: GitHub 대신 AWS의 자체 Git 저장소인 CodeCommit을 사용할 경우:
* **무료**: 활성 사용자 5명까지 매월 무료.
* **과금**: 사용자 5명 초과 시 인당 월 $1.00의 비용이 발생하며, 저장 용량 및 API 요청 횟수에 따라 추가 과금이 발생할 수 있습니다.



---

Next Step: .gitignore 설정 방법 및 브랜치(Branch) 관리 전략

.gitignore 작성법 | 브랜치 생성 및 병합 | 충돌(Conflict) 해결 방법

본 내용은 강의 교재 형식을 준수하여 작성되었습니다. 깃허브 마크다운 형식을 지원하며, AWS 사용 시 발생할 수 있는 보안 및 비용 문제를 명시하였습니다.

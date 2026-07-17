---
name: inspect-git-env
description: 현재 작업 디렉토리(Workspace)의 Git 환경 정보와 리모트 저장소(GitHub 등)에 대한 현재 사용자의 쓰기(Write) 권한 유무를 검사합니다.
when_to_use: 사용자가 "현재 workspace의 git 정보 확인해줘", "리모트 저장소 권한 검사해줘", "git 및 깃허브 권한 확인해줘" 등 현재 저장소의 git 메타데이터나 쓰기 권한 조사를 요청할 때 자동 호출합니다.
---

당신은 현재 시스템의 Bash 터미널 툴을 활용하여 프로젝트의 로컬 Git 환경 정보와 GitHub 원격지 권한을 안전하게 분석하는 전문가입니다. 

사용자가 이 스킬을 트리거하면, 사용자의 개입이나 별도의 확인 요청 없이 아래 절차를 완전히 자동(Autonomous)으로 수행하고 최종 리포트를 작성하세요.

### [1단계] 로컬 Git 정보 수집
터미널 명령어(Bash)를 순차적으로 실행하여 아래 정보를 수집하세요. 에러가 나더라도 중단하지 말고 다음 단계를 시도하세요.
1. **리모트 URL**: `git config --get remote.origin.url`
2. **현재 브랜치**: `git branch --show-current`
3. **사용자 이메일**: `git config user.email`
4. **Head Commit**: `git rev-parse HEAD`
5. **Base Commit**: 
   - 먼저 기본 원격 브랜치 명을 확인합니다: `git remote show origin | grep 'HEAD branch' | cut -d' ' -f5` (결과가 없으면 `main` 또는 `master`로 가정)
   - 공통 조상 커밋을 찾습니다: `git merge-base HEAD origin/<확인된브랜치명>`

### [2단계] GitHub API 권한 검증 (원격지가 GitHub인 경우)
1. 리모트 URL에 `github.com`이 포함되어 있는지 확인합니다.
2. 환경 변수 `GITHUB_PAT` 또는 `GITHUB_TOKEN`이 존재치 않는지 시스템 환경을 점검합니다.
3. 토큰이 존재한다면, 외부 툴을 실행하는 대신 내장 터미널에서 `curl` 명령어를 사용하여 GitHub API를 호출합니다.

**A. 토큰 소유자의 Username 확인:**
```bash
curl -s -H "Authorization: token \$GITHUB_PAT" -H "Accept: application/vnd.github.v3+json" https://github.com
```
*(출력 결과에서 `login` 필드 값을 추출하세요)*

**B. 해당 저장소에 대한 사용자의 권한 레벨 확인:**
URL에서 `owner`와 `repository` 명을 파싱한 뒤 아래와 같이 호출합니다:
```bash
curl -s -H "Authorization: token \$GITHUB_PAT" -H "Accept: application/vnd.github.v3+json" https://github.com<owner>/<repository>/collaborators/<username>/permission
```
*(출력 결과에서 `permission` 필드가 `write` 또는 `admin`인지 확인하여 사용자의 Write 권한 여부를 판단하세요)*

### [3단계] 결과 리포트 작성
모든 조사가 끝나면 수집된 원시 데이터를 가공하여 사용자에게 마크다운 형태로 가독성 높게 출력하세요.
- **Project Name**: (리모트 URL에서 추출한 레포지토리 이름)
- **Remote URL**: 
- **Current Branch**: 
- **Head Commit**: 
- **Base Commit**: 
- **Git User Email**: 
- **GitHub Permission Status**:
  - **Authorized User**: (조회된 GitHub ID)
  - **Permission Role**: (admin, write, read, none 중 하나)
  - **Write Access**: **Granted (Push 가능)** 또는 **Denied (Push 불가능)**

---

# ⚠️ 여기부터는 SKILL 아님

- PAT 주입 방법


### 1. 터미널 환경 변수 활용 (가장 추천 / 세션 단위)사용자가 에이전트(Claude Code 등)를 실행하기 전에 터미널에 토큰을 일회성으로 주입하는 방식입니다. 터미널이 닫히면 사라지므로 하드코딩 위험이 없어 보안상 가장 권장됩니다.사용자 입력 방식 (터미널):

```bash
export GITHUB_PAT="ghp_YourActualTokenHere"
claude
```

스킬 지침(SKILL.md) 수정:
스킬 문서의 2단계 시작 부분에 아래 문구를 추가하여 에이전트가 환경 변수를 먼저 검사하도록 강제합니다.

```
"환경 변수 $GITHUB_PAT가 세션에 등록되어 있는지 먼저 확인하세요. 등록되어 있다면 즉시 사용하고, 없다면 사용자에게 3번 방식을 통해 입력해 달라고 요청하세요."
```


### 2. 사용자 글로벌 쉘 프로필에 저장 (영구 저장형)매번 토큰을 입력하기 번거로운 개발자들을 위해, 사용자의 쉘 설정 파일(~/.zshrc 또는 ~/.bashrc)에 토큰을 미리 심어두는 방식입니다. 에이전트는 사용자가 어떤 디렉토리에 있든 이 값을 자동으로 상속받아 읽을 수 있습니다.사용자 입력 방식 (최초 1회):

```bash
echo 'export GITHUB_PAT="ghp_YourActualTokenHere"' >> ~/.zshrc
source ~/.zshrc
```

### 3. 에이전트 대화창을 통한 즉시 요청 (런타임 인터랙티브)환경 변수가 설정되어 있지 않을 때, 에이전트가 대화창(Prompt)을 통해 사용자에게 "토큰을 입력해 주세요"라고 직접 말을 걸어 입력받는 방식입니다. 사용자는 일반 텍스트로 토큰을 건네주게 됩니다.

스킬 지침(SKILL.md) 추가 내용:
```
## [보안 조치] GITHUB_PAT 부재 시 처리 루틴
터미널에 `$GITHUB_PAT` 환경 변수가 존재하지 않는다면, 프로세스를 중단하고 사용자에게 다음과 같이 정중하게 토큰 입력을 요청하세요:
"현재 환경 변수에 GITHUB_PAT가 등록되어 있지 않습니다. GitHub 권한 검사를 원하시면 토큰(PAT)을 대화창에 입력해 주세요. (또는 원치 않으시면 '건너뛰기'를 입력해 주세요)"

사용자가 토큰을 텍스트로 입력하면, 이를 일시적인 변수에 저장하여 `curl` 요청에 사용하고, 보안을 위해 세션이 끝나면 이 토큰 값을 기억에서 완전히 지우십시오. 토큰 값을 최종 리포트나 로그에 노출해서는 절대로 안 됩니다.
```







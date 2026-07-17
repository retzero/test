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

# SVN → GitHub Enterprise 마이그레이션 가이드

## 1. 개요(Overview)

본 문서는 **SVN(Subversion) 저장소를 GitHub Enterprise 저장소로 마이그레이션**하는 절차를 설명합니다.

SVN과 Git은 저장소 구조와 이력 관리 방식이 다르기 때문에, 일반적으로 다음 흐름으로 이전합니다.

```text
SVN Repository
      ↓
Local Git Repository로 변환
      ↓
변환 결과 검증
      ↓
GitHub Enterprise Repository에 Push
```

즉, 이 방식은 **SVN 저장소를 직접 GitHub Enterprise로 전송하는 방식이 아니라**, 로컬 PC 또는 작업 서버에서 SVN 이력을 Git 저장소로 변환한 뒤 GitHub Enterprise에 업로드하는 방식입니다.

이 방식의 장점은 다음과 같습니다.

- 기존 SVN 커밋 이력을 최대한 보존할 수 있습니다.
- SVN 사용자명을 Git Author 정보로 매핑할 수 있습니다.
- SVN의 `trunk`, `branches`, `tags` 구조를 Git 브랜치와 태그로 변환할 수 있습니다.
- GitHub Enterprise에 반영하기 전에 로컬에서 충분히 검증할 수 있습니다.
- 문제가 생겨도 GitHub Enterprise 저장소에 Push하기 전까지는 대상 저장소에 영향을 주지 않습니다.

> 이 문서는 고객이 이미 보유한 GitHub Enterprise 환경을 대상으로 합니다. 별도의 GitHub Enterprise Trial 생성 절차는 포함하지 않습니다.

---

## 2. 마이그레이션 전체 흐름

아래 순서로 작업합니다.

| 단계 | 작업 | 설명 |
|---:|---|---|
| 1 | 사전 준비 | Git, SVN Client, GitHub Enterprise 접근 권한을 준비합니다. |
| 2 | SVN 구조 확인 | SVN 저장소가 표준 구조인지 확인합니다. |
| 3 | Author Mapping 작성 | SVN 사용자명을 Git Author 형식으로 변환할 매핑 파일을 만듭니다. |
| 4 | SVN → Local Git 변환 | `git svn clone` 명령으로 SVN 저장소를 로컬 Git 저장소로 변환합니다. |
| 5 | 변환 결과 검증 | 커밋 이력, 브랜치, 태그, 작성자 정보를 확인합니다. |
| 6 | GitHub Enterprise 저장소 생성 | GitHub Enterprise에 빈 Repository를 생성합니다. |
| 7 | Remote 등록 | 로컬 Git 저장소에 GitHub Enterprise 저장소 주소를 연결합니다. |
| 8 | Push 수행 | 브랜치와 태그를 GitHub Enterprise로 업로드합니다. |
| 9 | 최종 검증 | GitHub Enterprise UI에서 이관 결과를 확인합니다. |

---

## 3. 사전 준비 사항

### 3.1 필수 소프트웨어

아래 표는 GitHub Markdown 미리보기에서 깨지지 않도록 표준 Markdown 형식으로 작성되어 있습니다.

| 소프트웨어 | 용도 | 다운로드 |
|---|---|---|
| Git | Git 명령어 사용 및 로컬 Git 저장소 관리 | [Git Downloads](https://git-scm.com/downloads) |
| Git for Windows | Windows 환경에서 Git Bash, Git Credential Manager, Git 명령어 사용 | [Git for Windows](https://gitforwindows.org/) |
| Apache Subversion(SVN) Client | SVN 저장소 접근 및 `svn` 명령어 사용 | [Apache Subversion Packages](https://subversion.apache.org/packages.html) |
| Git SVN | SVN 저장소를 Git 저장소로 변환하기 위한 `git svn` 명령어 | Git 설치 패키지 또는 OS 패키지 매니저에서 설치 |

> Windows에서는 보통 **Git for Windows** 설치 후 Git Bash 또는 PowerShell에서 Git 명령을 사용할 수 있습니다.  
> 단, 설치 방식에 따라 `git svn`이 포함되지 않을 수 있으므로 반드시 아래 버전 확인 명령을 실행해야 합니다.

### 3.2 설치 확인

설치가 완료되면 터미널에서 다음 명령어를 실행합니다.

```bash
git --version
svn --version
git svn --version
```

정상 예시는 다음과 비슷합니다.

```text
git version 2.x.x
svn, version 1.x.x
git-svn version 2.x.x
```

`git svn --version` 실행 시 오류가 발생하면 `git svn` 구성 요소가 설치되어 있지 않은 것입니다.

#### Windows에서 확인할 점

Windows 환경에서는 다음 중 하나를 사용합니다.

| 실행 환경 | 설명 |
|---|---|
| Git Bash | Git for Windows 설치 시 함께 제공되는 터미널입니다. Git 명령 사용에 가장 무난합니다. |
| PowerShell | Windows 기본 터미널입니다. Git이 PATH에 등록되어 있어야 합니다. |
| CMD | 사용 가능하지만 긴 명령어 입력과 복사/붙여넣기 편의성은 Git Bash 또는 PowerShell이 더 좋습니다. |

---

## 4. 권한 준비

마이그레이션을 위해서는 **Source(SVN)** 와 **Target(GitHub Enterprise)** 양쪽 권한이 모두 필요합니다.

### 4.1 Source: SVN 권한

SVN 저장소에서 최소한 다음 권한이 필요합니다.

| 권한 | 필요 여부 | 설명 |
|---|---:|---|
| Repository Read 권한 | 필수 | SVN 저장소의 전체 이력을 읽기 위해 필요합니다. |
| Branch/Tag Read 권한 | 필수 | SVN의 branches, tags 이력을 함께 가져오기 위해 필요합니다. |
| 계정 인증 정보 | 필수 | SVN이 인증을 요구하는 경우 사용자명/비밀번호 또는 인증 토큰이 필요합니다. |

SVN 저장소 접근 테스트는 다음처럼 수행합니다.

```bash
svn list https://svn.example.com/project
```

정상적으로 접근되면 `trunk`, `branches`, `tags` 또는 프로젝트 디렉터리 목록이 출력됩니다.

### 4.2 Target: GitHub Enterprise 권한

GitHub Enterprise에서는 다음 권한이 필요합니다.

| 권한 | 필요 여부 | 설명 |
|---|---:|---|
| Repository 생성 권한 | 필수 | 대상 Organization 또는 User 영역에 새 Repository를 만들 수 있어야 합니다. |
| Push 권한 | 필수 | 변환된 Git 이력을 대상 Repository에 업로드하기 위해 필요합니다. |
| PAT 또는 SSH Key | 권장 | HTTPS Push 시 PAT, SSH Push 시 SSH Key가 필요합니다. |
| Branch Protection 관리 권한 | 상황별 필요 | 기본 브랜치 보호 정책 때문에 Push가 거부되는 경우 확인이 필요합니다. |

> 신규 Repository는 가능하면 **빈 Repository**로 생성하는 것을 권장합니다. README, `.gitignore`, License를 미리 만들면 첫 Push 시 이력 충돌이 발생할 수 있습니다.

---

## 5. SVN 저장소 구조 확인

SVN 저장소는 보통 다음과 같은 표준 구조를 사용합니다.

```text
project/
 ├── trunk
 ├── branches
 └── tags
```

각 디렉터리의 의미는 다음과 같습니다.

| SVN 디렉터리 | Git 변환 후 의미 |
|---|---|
| `trunk` | 기본 브랜치 역할을 합니다. 일반적으로 Git의 `main` 또는 `master`가 됩니다. |
| `branches` | Git의 브랜치로 변환됩니다. |
| `tags` | Git의 태그로 변환됩니다. |

SVN 구조를 확인하려면 다음 명령을 실행합니다.

```bash
svn list https://svn.example.com/project
```

표준 구조라면 다음처럼 출력됩니다.

```text
trunk/
branches/
tags/
```

### 5.1 표준 구조가 아닌 경우

예를 들어 SVN 구조가 다음처럼 되어 있을 수 있습니다.

```text
project/
 ├── Main
 ├── Branches
 └── Releases
```

이 경우 `git svn clone` 실행 시 표준 옵션인 `--stdlayout`을 사용할 수 없습니다. 대신 다음처럼 경로를 직접 지정해야 합니다.

```bash
git svn clone https://svn.example.com/project \
  --trunk=Main \
  --branches=Branches \
  --tags=Releases \
  --authors-file=authors.txt
```

---

## 6. Author Mapping 파일 생성

SVN은 커밋 작성자를 보통 단순 사용자명으로 저장합니다.

예:

```text
john
alice
admin
```

Git은 작성자를 다음 형식으로 저장합니다.

```text
Name <email@example.com>
```

따라서 SVN 사용자명을 Git Author 형식으로 변환하기 위해 `authors.txt` 파일을 만듭니다.

### 6.1 authors.txt 예시

```text
john = John Smith <john@example.com>
alice = Alice Kim <alice@example.com>
admin = Repository Admin <admin@example.com>
```

작성 규칙은 다음과 같습니다.

```text
SVN사용자명 = 표시이름 <이메일주소>
```

### 6.2 SVN 사용자 목록 확인

SVN 로그에서 사용자명을 확인할 수 있습니다.

```bash
svn log https://svn.example.com/project --quiet
```

출력 예시는 다음과 같습니다.

```text
r123 | john | 2024-01-10 10:00:00 +0900
r124 | alice | 2024-01-11 11:00:00 +0900
```

여기서 `john`, `alice`가 `authors.txt`에 들어가야 하는 SVN 사용자명입니다.

### 6.3 누락된 Author가 있을 때

`git svn clone` 실행 중 Author가 누락되면 다음과 유사한 오류가 발생할 수 있습니다.

```text
Author: unknown_user not defined in authors.txt file
```

이 경우 `authors.txt`에 해당 사용자를 추가한 후 다시 실행합니다.

```text
unknown_user = Unknown User <unknown_user@example.com>
```

---

## 7. SVN 저장소를 로컬 Git 저장소로 변환

이 단계에서는 SVN 저장소를 직접 GitHub Enterprise로 올리는 것이 아니라, 먼저 **로컬 Git 저장소로 변환**합니다.

### 7.1 작업 디렉터리 생성

```bash
mkdir svn-to-ghe-migration
cd svn-to-ghe-migration
```

### 7.2 표준 SVN 구조인 경우

SVN 저장소가 `trunk`, `branches`, `tags` 구조라면 다음 명령을 사용합니다.

```bash
git svn clone https://svn.example.com/project \
  --stdlayout \
  --authors-file=authors.txt \
  project-git
```

명령어 의미는 다음과 같습니다.

| 옵션 | 설명 |
|---|---|
| `git svn clone` | SVN 저장소를 Git 저장소로 변환합니다. |
| `https://svn.example.com/project` | Source SVN 저장소 URL입니다. |
| `--stdlayout` | SVN 표준 구조인 `trunk`, `branches`, `tags`를 사용한다는 의미입니다. |
| `--authors-file=authors.txt` | SVN 사용자명을 Git Author로 변환할 매핑 파일입니다. |
| `project-git` | 로컬에 생성될 Git 저장소 디렉터리 이름입니다. |

### 7.3 사용자 정의 SVN 구조인 경우

SVN 구조가 표준이 아니라면 다음처럼 직접 지정합니다.

```bash
git svn clone https://svn.example.com/project \
  --trunk=Main \
  --branches=Branches \
  --tags=Tags \
  --authors-file=authors.txt \
  project-git
```

### 7.4 대용량 저장소 주의사항

SVN 이력이 많거나 바이너리 파일이 많은 경우 변환 시간이 오래 걸릴 수 있습니다.

권장 사항은 다음과 같습니다.

- 안정적인 네트워크 환경에서 수행합니다.
- 노트북 절전 모드를 해제합니다.
- 가능하면 작업 서버에서 수행합니다.
- 중간에 실패하면 오류 메시지를 확인하고 누락된 Author 또는 접근 권한을 먼저 점검합니다.

---

## 8. 변환 결과 검증

변환이 완료되면 로컬 Git 저장소로 이동합니다.

```bash
cd project-git
```

### 8.1 커밋 이력 확인

```bash
git log --oneline --decorate --graph --all
```

확인할 내용은 다음과 같습니다.

- 과거 SVN 커밋 이력이 보이는지
- 최신 커밋이 포함되어 있는지
- 브랜치와 태그가 예상대로 연결되어 있는지

### 8.2 브랜치 확인

```bash
git branch -a
```

SVN branches가 Git 브랜치로 변환되었는지 확인합니다.

### 8.3 태그 확인

```bash
git tag
```

SVN tags가 Git 태그로 변환되었는지 확인합니다.

### 8.4 Author 정보 확인

```bash
git log --format="%an <%ae>" | sort | uniq
```

확인할 내용은 다음과 같습니다.

- `authors.txt`에 정의한 이름과 이메일로 표시되는지
- `unknown`, `no author`, 잘못된 이메일 등이 없는지

---

## 9. Git 기본 브랜치 정리

SVN의 `trunk`는 Git 변환 후 보통 `master` 브랜치가 됩니다.

GitHub Enterprise에서 기본 브랜치를 `main`으로 운영하려면 다음 명령으로 브랜치명을 변경합니다.

```bash
git branch -M main
```

현재 브랜치를 확인합니다.

```bash
git branch
```

예상 출력:

```text
* main
```

> 조직의 표준 브랜치명이 `master`라면 이 단계는 생략해도 됩니다.

---

## 10. GitHub Enterprise 저장소 생성

GitHub Enterprise 웹 화면에서 대상 Repository를 생성합니다.

### 10.1 생성 위치

다음 중 하나에 생성합니다.

| 위치 | 설명 |
|---|---|
| Organization Repository | 회사/팀 Organization 아래 생성하는 방식입니다. 일반적으로 권장됩니다. |
| User Repository | 개인 계정 아래 생성하는 방식입니다. 테스트나 개인 프로젝트에 적합합니다. |

### 10.2 생성 시 권장 설정

Repository 생성 화면에서 다음을 권장합니다.

| 항목 | 권장값 | 이유 |
|---|---|---|
| Repository name | 기존 SVN 프로젝트명과 동일 또는 표준 명명 규칙 사용 | 관리와 추적이 쉽습니다. |
| Visibility | 조직 정책에 맞게 선택 | Private/Internal/Public 중 선택합니다. |
| README 생성 | 선택하지 않음 | 기존 이력과 충돌 방지 |
| `.gitignore` 생성 | 선택하지 않음 | 기존 파일과 충돌 방지 |
| License 생성 | 선택하지 않음 | 기존 이력과 충돌 방지 |

> 반드시 빈 Repository로 생성하는 것을 권장합니다.

---

## 11. GitHub Enterprise 인증 준비

GitHub Enterprise에 Push하려면 HTTPS 또는 SSH 인증 중 하나를 사용합니다.

### 11.1 HTTPS + PAT 방식

HTTPS Remote URL 예시는 다음과 같습니다.

```text
https://github.company.com/org/project.git
```

이 방식에서는 Push 시 GitHub Enterprise 계정과 Personal Access Token(PAT)이 필요할 수 있습니다.

PAT에는 일반적으로 다음 권한이 필요합니다.

| 대상 | 필요한 권한 |
|---|---|
| Repository Push | `repo` 또는 Repository write 권한 |
| Organization Repository | 조직 정책에 따라 SSO 승인 또는 추가 권한 필요 가능 |

### 11.2 SSH Key 방식

SSH Remote URL 예시는 다음과 같습니다.

```text
git@github.company.com:org/project.git
```

이 방식에서는 로컬 PC의 SSH Public Key가 GitHub Enterprise 계정에 등록되어 있어야 합니다.

SSH 연결 테스트:

```bash
ssh -T git@github.company.com
```

정상 연결되면 인증 성공 메시지가 표시됩니다.

---

## 12. Remote 등록

로컬 Git 저장소에 GitHub Enterprise Repository 주소를 등록합니다.

### 12.1 HTTPS 방식

```bash
git remote add origin https://github.company.com/org/project.git
```

### 12.2 SSH 방식

```bash
git remote add origin git@github.company.com:org/project.git
```

### 12.3 Remote 확인

```bash
git remote -v
```

예상 출력:

```text
origin  https://github.company.com/org/project.git (fetch)
origin  https://github.company.com/org/project.git (push)
```

Remote URL을 잘못 등록한 경우 다음처럼 수정합니다.

```bash
git remote set-url origin https://github.company.com/org/project.git
```

---

## 13. GitHub Enterprise로 Push

### 13.1 전체 브랜치 Push

```bash
git push origin --all
```

이 명령은 로컬에 있는 모든 브랜치를 GitHub Enterprise로 업로드합니다.

### 13.2 전체 태그 Push

```bash
git push origin --tags
```

이 명령은 로컬에 있는 모든 Git 태그를 GitHub Enterprise로 업로드합니다.

### 13.3 기본 브랜치만 먼저 Push하고 싶은 경우

처음에는 기본 브랜치만 Push해서 정상 여부를 확인할 수도 있습니다.

```bash
git push -u origin main
```

정상 확인 후 다른 브랜치와 태그를 Push합니다.

```bash
git push origin --all
git push origin --tags
```

---

## 14. 최종 검증

GitHub Enterprise 웹 화면에서 다음 항목을 확인합니다.

| 검증 항목 | 확인 방법 |
|---|---|
| 기본 브랜치 | Repository 메인 화면에서 기본 브랜치가 `main` 또는 `master`인지 확인 |
| 커밋 이력 | Commits 메뉴에서 과거 이력이 보이는지 확인 |
| 브랜치 | Branches 메뉴에서 SVN branch가 반영되었는지 확인 |
| 태그 | Tags/Releases 메뉴에서 SVN tag가 반영되었는지 확인 |
| 파일 | 주요 소스 파일, 설정 파일, 문서 파일이 정상적으로 보이는지 확인 |
| Author | 커밋 작성자 이름과 이메일이 의도대로 표시되는지 확인 |
| 권한 | 팀원이 Clone/Pull/Push 가능한지 확인 |

로컬에서도 다음 명령으로 확인할 수 있습니다.

```bash
git remote show origin
git ls-remote --heads origin
git ls-remote --tags origin
```

---

## 15. 운영 전환 시 권장 절차

실제 운영 저장소를 전환할 때는 다음 절차를 권장합니다.

### 15.1 사전 리허설

운영 반영 전에 테스트용 GitHub Enterprise Repository를 만들어 한 번 변환해 봅니다.

확인할 내용:

- 변환 시간이 얼마나 걸리는지
- Author Mapping 누락이 있는지
- 브랜치와 태그가 정상 변환되는지
- 대용량 파일 또는 특수 파일명 문제가 있는지

### 15.2 Freeze Window 확보

최종 마이그레이션 시점에는 SVN 저장소 변경을 잠시 중단하는 것이 좋습니다.

예:

1. 개발팀에 SVN Commit 중단 공지
2. SVN 최종 Revision 번호 확인
3. 최종 `git svn clone` 또는 동기화 수행
4. GitHub Enterprise Push
5. GitHub Enterprise 기준으로 개발 재개

### 15.3 SVN 읽기 전용 전환

마이그레이션 완료 후에는 혼선을 막기 위해 SVN 저장소를 읽기 전용으로 전환하거나 안내 문구를 남기는 것이 좋습니다.

예:

```text
이 저장소는 GitHub Enterprise로 이전되었습니다.
신규 Commit은 GitHub Enterprise Repository를 사용하십시오.
```

---

## 16. 문제 해결(Troubleshooting)

### 16.1 `git svn` 명령을 찾을 수 없는 경우

증상:

```text
git: 'svn' is not a git command
```

확인 사항:

- Git 설치 여부 확인
- Git for Windows 설치 여부 확인
- `git svn` 구성 요소 포함 여부 확인
- Linux/macOS에서는 패키지 매니저로 `git-svn` 별도 설치 필요 가능

예시:

```bash
git --version
git svn --version
```

### 16.2 Author Mapping 오류

증상:

```text
Author: user1 not defined in authors.txt file
```

해결:

1. 오류에 표시된 SVN 사용자명을 확인합니다.
2. `authors.txt`에 해당 사용자를 추가합니다.
3. 변환 명령을 다시 실행합니다.

예:

```text
user1 = User One <user1@example.com>
```

### 16.3 Push가 거부되는 경우

증상:

```text
remote: Permission denied
fatal: unable to access ...
```

확인 사항:

- GitHub Enterprise Repository URL이 맞는지 확인
- 현재 로그인한 계정이 대상 Repository에 Push 권한이 있는지 확인
- HTTPS 사용 시 PAT 권한 확인
- SSH 사용 시 SSH Key 등록 여부 확인
- Organization SSO 정책이 있는 경우 PAT/SSH Key 승인 여부 확인
- Branch Protection Rule이 Push를 막고 있는지 확인

현재 Remote URL 확인:

```bash
git remote -v
```

### 16.4 브랜치 또는 태그가 누락된 경우

원인:

- SVN 구조가 표준이 아닌데 `--stdlayout`을 사용했을 수 있습니다.
- `--branches`, `--tags` 경로가 실제 SVN 구조와 다를 수 있습니다.

해결:

```bash
svn list https://svn.example.com/project
```

실제 구조를 확인한 뒤 다시 변환합니다.

```bash
git svn clone https://svn.example.com/project \
  --trunk=실제_trunk_경로 \
  --branches=실제_branches_경로 \
  --tags=실제_tags_경로 \
  --authors-file=authors.txt \
  project-git
```

### 16.5 GitHub Enterprise에 README가 이미 있어서 Push 충돌이 나는 경우

증상:

```text
! [rejected] main -> main (fetch first)
```

원인:

GitHub Enterprise Repository 생성 시 README, License, `.gitignore`를 함께 생성해 로컬 이력과 원격 이력이 달라졌기 때문입니다.

권장 해결:

- 가능하면 GitHub Enterprise Repository를 삭제 후 빈 Repository로 다시 생성합니다.
- 운영 정책상 삭제가 어렵다면 원격 내용을 먼저 Pull/Merge해야 하지만, 마이그레이션 저장소에서는 빈 Repository 재생성이 더 깔끔합니다.

---

## 17. 체크리스트

마이그레이션 전에 아래 항목을 확인합니다.

| 구분 | 체크 항목 | 완료 |
|---|---|---|
| Source | SVN URL 확인 |  |
| Source | SVN Read 권한 확인 |  |
| Source | SVN 구조 확인 |  |
| Source | 최종 SVN Revision 확인 |  |
| Author | SVN 사용자 목록 추출 |  |
| Author | `authors.txt` 작성 |  |
| Local | `git svn clone` 완료 |  |
| Local | 커밋 이력 확인 |  |
| Local | 브랜치 확인 |  |
| Local | 태그 확인 |  |
| Target | GitHub Enterprise 빈 Repository 생성 |  |
| Target | PAT 또는 SSH Key 준비 |  |
| Target | Remote 등록 |  |
| Target | Branch Push 완료 |  |
| Target | Tag Push 완료 |  |
| 검증 | GitHub Enterprise UI 확인 |  |
| 전환 | SVN Freeze Window 적용 |  |
| 전환 | 팀에 GitHub Enterprise 사용 안내 |  |

---

## 18. 명령어 요약

표준 SVN 구조 기준 전체 명령어 예시는 다음과 같습니다.

```bash
# 1. 작업 디렉터리 생성
mkdir svn-to-ghe-migration
cd svn-to-ghe-migration

# 2. SVN 구조 확인
svn list https://svn.example.com/project

# 3. SVN → Git 변환
git svn clone https://svn.example.com/project \
  --stdlayout \
  --authors-file=authors.txt \
  project-git

# 4. 로컬 Git 저장소 이동
cd project-git

# 5. 변환 결과 확인
git log --oneline --decorate --graph --all
git branch -a
git tag
git log --format="%an <%ae>" | sort | uniq

# 6. 기본 브랜치명을 main으로 변경하는 경우
git branch -M main

# 7. GitHub Enterprise Remote 등록
git remote add origin https://github.company.com/org/project.git

# 8. Push
git push origin --all
git push origin --tags

# 9. 원격 확인
git remote show origin
git ls-remote --heads origin
git ls-remote --tags origin
```

---

## 19. 참고 자료(Reference)

### GitHub 공식 문서

- [Importing a Subversion repository](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/importing-a-subversion-repository)
- [Migration paths to GitHub](https://docs.github.com/en/enterprise-server/migrations/overview/migration-paths-to-github)

### Microsoft 공식 문서

- [Perform a migration from SVN to Git](https://learn.microsoft.com/azure/devops/repos/git/perform-migration-from-svn-to-git)
- [Azure Repos Git documentation](https://learn.microsoft.com/azure/devops/repos/git/)

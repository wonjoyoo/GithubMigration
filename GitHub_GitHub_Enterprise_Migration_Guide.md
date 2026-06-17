# GitHub 개인 계정 → GitHub Enterprise Cloud Migration Guide

## 1. Overview

본 문서는 **GitHub 개인 계정(User Account)에 존재하는 Repository를 고객사의 GitHub Enterprise Cloud Organization으로 Migration하는 절차**를 설명합니다.

GitHub 개인 계정에 있는 Repository를 Enterprise Organization으로 옮기는 방법은 크게 두 가지가 있습니다.

### 1.1 방식 비교

| 방식 | 설명 | Source Repository 유지 여부 | 주요 특징 |
|---|---|---:|---|
| **직접 Transfer** | 개인 계정의 Repository를 Target Enterprise Organization으로 직접 Transfer | ❌ 유지되지 않음 | Repository 소유권 자체가 이동합니다. 가장 단순하지만 Source Repository는 개인 계정에서 사라집니다. |
| **Source Org 경유 Migration** | 개인 계정 Repository를 일반 Organization으로 Transfer한 뒤, GEI로 Target Enterprise Organization에 Migration | ✅ 유지됨 | GEI가 Target Organization에 Repository를 새로 생성하여 복사(Migration)합니다. Source Organization의 Repository는 유지됩니다. |

본 가이드는 두 번째 방식인 **Source Organization 경유 Migration 방식**을 기준으로 작성되었습니다.

GitHub Enterprise Importer(GEI)는 **GitHub 개인 계정(User Account)을 Source로 직접 사용할 수 없습니다.**  
따라서 개인 계정의 Repository를 먼저 일반 GitHub Organization으로 Transfer한 뒤, 해당 Organization을 Source로 지정하여 GitHub Enterprise Organization으로 Migration을 수행합니다.

```text
개인 GitHub 계정 (SOURCE_USER)
            │
            │ 1) Repository Transfer
            ▼
일반 GitHub Organization (SOURCE_ORG)
            │
            │ 2) GitHub Enterprise Importer Migration
            ▼
GitHub Enterprise Organization (TARGET_ORG)
```

이 절차를 사용하면 다음과 같은 장점이 있습니다.

- 개인 계정의 Repository를 Enterprise Organization으로 직접 Transfer하지 않아도 됩니다.
- GEI Migration 이후에도 Source Organization에 Repository가 유지됩니다.
- 여러 Repository를 동일한 변수와 스크립트 구조로 반복 Migration할 수 있습니다.
- 코드뿐 아니라 GEI가 지원하는 범위 내에서 Issues, Pull Requests, Releases, Wiki 등의 메타데이터도 함께 Migration할 수 있습니다.

---

## 2. 사전 준비 사항

Migration을 진행하기 전에 아래 항목이 준비되어 있어야 합니다.

| 구분 | 설명 |
|---|---|
| GitHub 개인 계정 | Source Repository를 보유한 개인 계정 |
| GitHub Enterprise Organization | 고객사가 이미 보유하고 있는 Target Organization |
| Source Organization | 개인 계정 Repository를 임시로 보관할 일반 GitHub Organization |
| GitHub CLI (`gh`) | GitHub API 및 GEI 실행에 사용 |
| GitHub Enterprise Importer Extension (`gh-gei`) | Migration 실행 도구 |
| Source PAT | Source Organization 접근용 Classic PAT |
| Target PAT | Target Enterprise Organization 접근용 Classic PAT |

> 본 문서는 고객사가 이미 GitHub Enterprise Cloud 환경을 보유하고 있다는 전제로 작성되었습니다.  
> GitHub Enterprise Trial 생성 절차는 포함하지 않습니다.

---

## 3. 변수 정의

아래 변수는 이후 명령어에서 공통으로 사용됩니다.

| 변수명 | 의미 | 예시 |
|---|---|---|
| `SOURCE_USER` | 개인 GitHub 계정명 | `wyattyoo` |
| `SOURCE_ORG` | 중간 Source Organization명 | `wyattyooorg` |
| `TARGET_ORG` | GitHub Enterprise Organization명 | `m2cloudorg` |
| `REPO_NAME` | Migration할 Repository명 | `test1` |
| `TARGET_REPO` | Target Organization에 생성할 Repository명 | `test1` |
| `TARGET_VISIBILITY` | Target Repository 공개 여부 | `private` 또는 `public` |

예시:

```text
SOURCE_USER       = wyattyoo
SOURCE_ORG        = wyattyooorg
TARGET_ORG        = m2cloudorg
REPO_NAME         = test1
TARGET_REPO       = test1
TARGET_VISIBILITY = private
```

---

## 4. Source Organization 생성

GEI는 개인 GitHub 계정을 Source로 직접 사용할 수 없으므로, 개인 계정 Repository를 먼저 보관할 **Source Organization**을 생성해야 합니다.

이 작업은 최초 한 번만 수행하면 됩니다.

### 생성 절차

1. GitHub에 개인 계정으로 로그인합니다.
2. 우측 상단의 **+** 버튼을 클릭합니다.
3. **New organization**을 선택합니다.
4. Free Plan 또는 고객 정책에 맞는 Plan을 선택합니다.
5. Organization 이름을 입력합니다.
   - 예: `wyattyooorg`
6. 생성 절차를 완료합니다.

생성 후 Source Organization URL 예시는 다음과 같습니다.

```text
https://github.com/wyattyooorg
```

---

## 5. Personal Access Token(PAT) 생성

GEI Migration에는 **Source PAT**와 **Target PAT**가 필요합니다.

- **Source PAT**: Source Organization(`SOURCE_ORG`)에 접근 가능한 계정에서 생성
- **Target PAT**: GitHub Enterprise Organization(`TARGET_ORG`)에 접근 가능한 계정에서 생성

> Fine-grained PAT가 아닌 **Classic Personal Access Token(PAT)** 사용을 권장합니다.

<details>
<summary><strong>Source PAT / Target PAT 생성 방법 펼치기</strong></summary>

### 5.1 GitHub 메뉴 이동

1. GitHub에 로그인합니다.
2. 우측 상단의 **프로필 아이콘**을 클릭합니다.
3. **Settings**를 선택합니다.
4. 좌측 메뉴를 아래쪽으로 스크롤합니다.
5. 하단의 **Developer settings**를 클릭합니다.
6. **Personal access tokens**를 펼칩니다.
7. **Tokens (classic)**을 선택합니다.
8. **Generate new token**을 클릭합니다.
9. **Generate new token (classic)**을 선택합니다.

### 5.2 Source PAT 생성

Source PAT는 `SOURCE_ORG`에 접근할 수 있는 계정으로 생성합니다.

권장 Scope:

| Scope | 용도 |
|---|---|
| `repo` | private/public Repository 접근 |
| `admin:org` | Organization 정보 및 Migration 관련 접근 |
| `workflow` | GitHub Actions workflow 관련 데이터 Migration |

생성한 토큰은 아래 환경 변수에 사용합니다.

```text
GH_SOURCE_PAT
```

### 5.3 Target PAT 생성

Target PAT는 `TARGET_ORG`에 접근할 수 있고, Migration을 실행할 권한이 있는 계정으로 생성합니다.

권장 Scope:

| Scope | 용도 |
|---|---|
| `repo` | Target Repository 생성 및 접근 |
| `admin:org` | Target Organization 접근 및 Migration 권한 |
| `workflow` | GitHub Actions workflow 관련 데이터 Migration |

생성한 토큰은 아래 환경 변수에 사용합니다.

```text
GH_PAT
```

### 5.4 주의사항

- 토큰은 생성 직후 한 번만 확인할 수 있습니다.
- 생성 후 안전한 위치에 복사해 두어야 합니다.
- SAML SSO를 사용하는 Organization의 경우, 생성한 PAT에 대해 SSO Authorization이 필요할 수 있습니다.
- 만료 기간은 고객 보안 정책에 맞게 설정합니다.

</details>

---

## 6. GitHub CLI 설치

### 6.1 Windows (PowerShell)

```powershell
winget install --id GitHub.cli
gh --version
```

설치 후 `gh` 명령이 인식되지 않으면 PowerShell을 종료한 뒤 다시 실행합니다.

### 6.2 Linux

Ubuntu/Debian 예시:

```bash
sudo apt update
sudo apt install gh
gh --version
```

### 6.3 macOS

```bash
brew install gh
gh --version
```

---

## 7. GitHub Enterprise Importer Extension 설치

```bash
gh extension install github/gh-gei
gh extension upgrade github/gh-gei
```

설치 확인:

```bash
gh gei --help
```

---

## 8. GitHub CLI 로그인
### GitHub CLI 로그인 (`gh auth login`)

`gh auth login`은 **Source Repository를 관리하는 GitHub 계정**으로 로그인하는 것을 권장합니다.

이유는 Repository Transfer 단계에서 `gh api /repos/{owner}/{repo}/transfer` API를 호출하기 위해 Source Repository에 대한 관리자 권한이 필요하기 때문입니다.

반면 GitHub Enterprise Importer(GEI)에서 Target Organization에 대한 인증은 `GH_PAT` 환경 변수에 설정한 **Target Classic PAT**를 사용합니다.

정리하면 다음과 같습니다.

| 용도 | 권장 계정 |
|------|----------|
| `gh auth login` | Source Repository 관리자 계정 |
| `GH_SOURCE_PAT` | Source Organization 접근 가능한 Classic PAT |
| `GH_PAT` | Target Enterprise Organization 접근 가능한 Classic PAT |

### Windows / Linux / macOS 공통

```bash
gh auth login
```

일반적인 선택 예시는 다음과 같습니다.

```text
GitHub.com
HTTPS
Login with a web browser
```

로그인 상태 확인:

```bash
gh auth status
```

---

## 9. 변수 설정

### 9.1 Windows (PowerShell)

```powershell
$SOURCE_USER="wyattyoo"
$SOURCE_ORG="wyattyooorg"
$TARGET_ORG="m2cloudorg"
$REPO_NAME="test1"
$TARGET_REPO="test1"
$TARGET_VISIBILITY="private"
```

### 9.2 Linux / macOS (Bash)

```bash
SOURCE_USER="wyattyoo"
SOURCE_ORG="wyattyooorg"
TARGET_ORG="m2cloudorg"
REPO_NAME="test1"
TARGET_REPO="test1"
TARGET_VISIBILITY="private"
```

---

## 10. 개인 계정 Repository를 Source Organization으로 Transfer

이 단계에서는 개인 계정의 Repository를 Source Organization으로 Transfer합니다.

예시:

```text
wyattyoo/test1 → wyattyooorg/test1
```

### 10.1 Windows (PowerShell)

```powershell
gh api `
  -X POST `
  "/repos/$SOURCE_USER/$REPO_NAME/transfer" `
  -f new_owner="$SOURCE_ORG"
```

### 10.2 Linux / macOS (Bash)

```bash
gh api \
  -X POST \
  "/repos/$SOURCE_USER/$REPO_NAME/transfer" \
  -f new_owner="$SOURCE_ORG"
```

### 10.3 Transfer 확인

Windows / Linux / macOS 공통:

```bash
gh repo view "$SOURCE_ORG/$REPO_NAME" --web
```

---

## 11. GEI용 PAT 환경 변수 설정

### 11.1 Windows (PowerShell)

```powershell
$env:GH_SOURCE_PAT="SOURCE_CLASSIC_PAT"
$env:GH_PAT="TARGET_CLASSIC_PAT"
```

### 11.2 Linux / macOS (Bash)

```bash
export GH_SOURCE_PAT="SOURCE_CLASSIC_PAT"
export GH_PAT="TARGET_CLASSIC_PAT"
```

---

## 12. GitHub Enterprise Organization으로 Migration

이 단계에서는 `SOURCE_ORG`의 Repository를 `TARGET_ORG`로 Migration합니다.

예시:

```text
wyattyooorg/test1 → m2cloudorg/test1
```

### 12.1 Windows (PowerShell)

```powershell
gh gei migrate-repo `
  --github-source-org $SOURCE_ORG `
  --source-repo $REPO_NAME `
  --github-target-org $TARGET_ORG `
  --target-repo $TARGET_REPO `
  --target-repo-visibility $TARGET_VISIBILITY
```

### 12.2 Linux / macOS (Bash)

```bash
gh gei migrate-repo \
  --github-source-org "$SOURCE_ORG" \
  --source-repo "$REPO_NAME" \
  --github-target-org "$TARGET_ORG" \
  --target-repo "$TARGET_REPO" \
  --target-repo-visibility "$TARGET_VISIBILITY"
```

---

## 13. Migration 로그 다운로드

Migration 실패 또는 상세 확인이 필요한 경우 로그를 다운로드합니다.

### 13.1 Windows (PowerShell)

```powershell
gh gei download-logs `
  --github-target-org $TARGET_ORG `
  --target-repo $TARGET_REPO
```

### 13.2 Linux / macOS (Bash)

```bash
gh gei download-logs \
  --github-target-org "$TARGET_ORG" \
  --target-repo "$TARGET_REPO"
```

---

## 14. Migration 결과 확인

Migration 완료 후 아래 항목을 확인합니다.

| 확인 항목 | 설명 |
|---|---|
| Repository 생성 여부 | `TARGET_ORG/TARGET_REPO` 생성 확인 |
| Branch / Tag | Source와 동일한 Branch, Tag 존재 여부 확인 |
| Issues | Issues Migration 여부 확인 |
| Pull Requests | PR 및 상태 확인 |
| Releases | Release 및 Asset 확인 |
| Wiki | Wiki 사용 시 Migration 여부 확인 |
| Actions | Workflow 파일 및 설정 확인 |
| Migration Log | Target Repository에 생성되는 Migration Log Issue 확인 |

---

## 15. 여러 Repository Migration

여러 Repository를 Migration할 경우, Repository 목록을 배열로 정의하여 반복 처리할 수 있습니다.

### 15.1 Windows (PowerShell)

```powershell
$SOURCE_USER="wyattyoo"
$SOURCE_ORG="wyattyooorg"
$TARGET_ORG="m2cloudorg"
$TARGET_VISIBILITY="private"

$REPOS=@(
  "test1",
  "test2",
  "test3"
)

foreach ($REPO_NAME in $REPOS) {
  $TARGET_REPO=$REPO_NAME

  Write-Host "Transfer: $SOURCE_USER/$REPO_NAME -> $SOURCE_ORG/$REPO_NAME"

  gh api `
    -X POST `
    "/repos/$SOURCE_USER/$REPO_NAME/transfer" `
    -f new_owner="$SOURCE_ORG"

  Start-Sleep -Seconds 10

  Write-Host "Migration: $SOURCE_ORG/$REPO_NAME -> $TARGET_ORG/$TARGET_REPO"

  gh gei migrate-repo `
    --github-source-org $SOURCE_ORG `
    --source-repo $REPO_NAME `
    --github-target-org $TARGET_ORG `
    --target-repo $TARGET_REPO `
    --target-repo-visibility $TARGET_VISIBILITY
}
```

### 15.2 Linux / macOS (Bash)

```bash
SOURCE_USER="wyattyoo"
SOURCE_ORG="wyattyooorg"
TARGET_ORG="m2cloudorg"
TARGET_VISIBILITY="private"

REPOS=(
  "test1"
  "test2"
  "test3"
)

for REPO_NAME in "${REPOS[@]}"; do
  TARGET_REPO="$REPO_NAME"

  echo "Transfer: $SOURCE_USER/$REPO_NAME -> $SOURCE_ORG/$REPO_NAME"

  gh api \
    -X POST \
    "/repos/$SOURCE_USER/$REPO_NAME/transfer" \
    -f new_owner="$SOURCE_ORG"

  sleep 10

  echo "Migration: $SOURCE_ORG/$REPO_NAME -> $TARGET_ORG/$TARGET_REPO"

  gh gei migrate-repo \
    --github-source-org "$SOURCE_ORG" \
    --source-repo "$REPO_NAME" \
    --github-target-org "$TARGET_ORG" \
    --target-repo "$TARGET_REPO" \
    --target-repo-visibility "$TARGET_VISIBILITY"
done
```

---

## 16. 자주 발생하는 오류 및 해결 방법

### 16.1 `The source org with login ... could not be found`

원인:

- `--github-source-org`에 개인 계정명을 입력한 경우
- Source PAT가 Source Organization에 접근 권한이 없는 경우

해결:

- 개인 Repository를 먼저 Source Organization으로 Transfer합니다.
- `SOURCE_ORG` 값이 Organization login과 일치하는지 확인합니다.
- `GH_SOURCE_PAT` 권한을 확인합니다.

---

### 16.2 `gh` 명령어를 찾을 수 없음

원인:

- GitHub CLI 설치 후 PATH가 현재 Shell에 반영되지 않음

해결:

- PowerShell 또는 Terminal을 재시작합니다.
- Windows에서는 아래 경로에 `gh.exe`가 있는지 확인합니다.

```powershell
Test-Path "C:\Program Files\GitHub CLI\gh.exe"
```

---

### 16.3 Target Repository가 이미 존재함

원인:

- `TARGET_ORG`에 동일한 이름의 Repository가 이미 존재함

해결:

- 기존 Repository를 삭제하거나
- `TARGET_REPO` 값을 다른 이름으로 지정합니다.

---

## 17. 참고 문서

- [GitHub Docs - About GitHub Enterprise Importer](https://docs.github.com/enterprise-cloud@latest/migrations/using-github-enterprise-importer/understanding-github-enterprise-importer/about-github-enterprise-importer)
- [GitHub Docs - About migrations between GitHub products](https://docs.github.com/en/enterprise-cloud@latest/migrations/using-github-enterprise-importer/migrating-between-github-products/about-migrations-between-github-products)
- [GitHub Docs - Migrating repositories from GitHub.com to GitHub Enterprise Cloud](https://docs.github.com/en/migrations/using-github-enterprise-importer/migrating-between-github-products/migrating-repositories-from-githubcom-to-github-enterprise-cloud)
- [GitHub Docs - Managing access for migrations between GitHub products](https://docs.github.com/en/migrations/using-github-enterprise-importer/migrating-between-github-products/managing-access-for-a-migration-between-github-products)
- [GitHub Docs - Transferring a repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/transferring-a-repository)
- [GitHub Docs - REST API: Transfer a repository](https://docs.github.com/en/rest/repos/repos#transfer-a-repository)

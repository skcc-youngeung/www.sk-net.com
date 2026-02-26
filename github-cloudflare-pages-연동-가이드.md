# GitHub (기업 Organization) + Cloudflare Pages 연동 가이드

**작성일**: 2026-02-26
**프로젝트**: www.sk-net.com
**레포지토리**: https://github.com/skcc-youngeung/www.sk-net.com

---

## 개요

이 문서는 **기업 GitHub Organization 계정**과 **Cloudflare Pages**를 연동하여
`git push` 한 번으로 자동 배포되는 환경을 구축하는 전체 과정을 기록합니다.

### 최종 배포 흐름

```
로컬 파일 수정
    ↓
git commit + git push
    ↓
GitHub Actions 자동 실행
    ↓
Cloudflare Pages 자동 배포
    ↓
www.sk-net.com 반영 (약 1분 내)
```

### 왜 Cloudflare 직접 연동이 아닌 GitHub Actions를 사용하는가?

Cloudflare Pages에는 GitHub 레포를 직접 연결하는 OAuth 방식이 있습니다.
그러나 **GitHub Organization 계정**을 연결할 때 다음 오류가 발생합니다:

```
API Request Failed: GET /api/v4/organizations?parent.id=null (502)
```

이 오류는 Cloudflare와 GitHub Organization API 간 호환 문제로 발생하며,
**GitHub Actions + Wrangler(Cloudflare CLI)** 방식으로 우회하면 안정적으로 동작합니다.

---

## 사전 준비

- GitHub Organization 계정 (예: `skcc-youngeung`)
- Cloudflare 계정 및 Pages 프로젝트 생성 완료
- 로컬 Git 설치

---

## 1단계: Cloudflare Pages 프로젝트 생성

GitHub Actions가 배포할 대상 프로젝트를 먼저 만들어야 합니다.

1. [Cloudflare 대시보드](https://dash.cloudflare.com) 접속
2. 왼쪽 사이드바 → **Workers & Pages**
3. **Create application** → **Pages** 탭
4. **"Direct Upload"** 선택 (Git 연동 없이 생성)
5. 프로젝트명 입력 (예: `www-sk-net-com`)
6. 임시 파일(아무 `.html` 파일) 업로드 → 완료

> **중요**: 프로젝트명을 기억해두세요. 나중에 GitHub Secret에 등록합니다.

---

## 2단계: Cloudflare API 토큰 발급

GitHub Actions에서 Cloudflare에 배포할 때 사용하는 인증 토큰입니다.

1. Cloudflare 대시보드 → 우측 상단 프로필 아이콘 → **My Profile**
2. 왼쪽 메뉴 → **API Tokens**
3. **Create Token** 클릭

### 템플릿 선택 화면

토큰 생성 화면에 여러 템플릿이 표시됩니다.
**"Cloudflare Pages: Edit" 템플릿이 없는 경우** (계정 설정에 따라 표시되지 않을 수 있음):
→ 템플릿 목록 하단의 **"사용자 설정 토큰 생성"** 클릭

### 사용자 설정 토큰 권한 설정

| 항목 | 설정값 |
|------|--------|
| 토큰 이름 | `www.sk-net.com deploy` (원하는 이름) |
| 권한 | **계정** → **Cloudflare Pages** → **편집** |
| 계정 리소스 | 모든 계정 포함 |

> 권한 항목은 **계정** 수준에서 **Cloudflare Pages - 편집** 하나만 추가하면 됩니다.
> 다른 권한(DNS, Workers 등)은 필요 없습니다.

4. **"Continue to summary"** → **"Create Token"**
5. 생성된 토큰 복사 (**이 화면에서 한 번만 표시됨, 반드시 복사**)

---

## 3단계: Cloudflare Account ID 확인

1. Cloudflare 대시보드 → **Workers & Pages**
2. 오른쪽 사이드바에 **Account ID** 표시됨
3. 복사해두기

---

## 4단계: GitHub Personal Access Token (PAT) 발급

로컬에서 회사 GitHub 레포에 push할 때 사용하는 인증 토큰입니다.

> **주의**: 개인 계정과 회사(Organization) 계정의 인증이 다를 경우 필요합니다.

1. GitHub에서 **회사 계정으로 로그인**
2. 우측 상단 프로필 → **Settings**
3. 좌측 하단 → **Developer settings**
4. **Personal access tokens** → **Tokens (classic)**
5. **Generate new token** 클릭

### 토큰 타입 선택

두 가지 옵션이 표시됩니다:

| 옵션 | 설명 | 선택 |
|------|------|------|
| Generate new token | Fine-grained (세밀한 권한 설정) | - |
| **Generate new token (classic)** | 범용 토큰, 설정 간단 | **이것 선택** |

### 토큰 설정

| 항목 | 설정값 |
|------|--------|
| Note | `www.sk-net.com deploy` (메모용) |
| Expiration | `90 days` 또는 `No expiration` |
| Scopes | `repo` 전체 체크 + **`workflow` 체크** |

> **중요**: `workflow` 권한을 반드시 체크해야 합니다.
> 이 권한이 없으면 `.github/workflows/` 폴더의 파일을 push할 때 다음 오류가 발생합니다:
> ```
> refusing to allow a Personal Access Token to create or update workflow
> without `workflow` scope
> ```

6. **Generate token** → 생성된 토큰 복사

### Git remote URL에 토큰 적용

```bash
git remote set-url origin https://[PAT토큰]@github.com/[org명]/[레포명].git

# 예시
git remote set-url origin https://ghp_xxxxxxxxxxxx@github.com/skcc-youngeung/www.sk-net.com.git
```

> **보안 주의**: PAT 토큰은 채팅, 문서, 코드에 절대 기록하지 마세요.
> 토큰이 노출되었다면 즉시 GitHub → Settings → Developer settings → 해당 토큰 **Delete** 후 재발급하세요.

---

## 5단계: GitHub Secrets 등록

GitHub Actions가 Cloudflare에 배포할 때 사용하는 인증 정보를 안전하게 저장합니다.

접속 URL:
```
https://github.com/[org명]/[레포명]/settings/secrets/actions
```

**"New repository secret"** 으로 3개 등록:

| Secret Name | 값 | 출처 |
|---|---|---|
| `CLOUDFLARE_API_TOKEN` | 2단계에서 발급한 Cloudflare API 토큰 | Cloudflare → API Tokens |
| `CLOUDFLARE_ACCOUNT_ID` | 3단계에서 확인한 Account ID | Cloudflare → Workers & Pages 사이드바 |
| `CLOUDFLARE_PROJECT_NAME` | Cloudflare Pages 프로젝트명 | 1단계에서 만든 프로젝트명 |

---

## 6단계: GitHub Actions 워크플로우 파일 생성

프로젝트 루트에 다음 경로로 파일을 생성합니다:

```
프로젝트루트/
└── .github/
    └── workflows/
        └── deploy.yml
```

**deploy.yml 내용:**

```yaml
name: Deploy to Cloudflare Pages

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      deployments: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy to Cloudflare Pages
        uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          projectName: ${{ secrets.CLOUDFLARE_PROJECT_NAME }}
          directory: .
          gitHubToken: ${{ secrets.GITHUB_TOKEN }}
```

> **wrangler-action vs pages-action**
> 처음에는 `cloudflare/wrangler-action@v3`을 사용했으나 Organization 환경에서 오류가 발생했습니다.
> `cloudflare/pages-action@v1`이 Pages 배포에 특화되어 있어 더 안정적으로 동작합니다.

---

## 7단계: 정적 파일 Cloudflare Pages 빌드 설정

Cloudflare Pages에서 GitHub Actions로 배포할 때 빌드 설정은 **모두 비워둡니다**.

순수 HTML/CSS/JS/이미지 등 정적 파일은 별도의 빌드 과정이 없기 때문입니다.

| 항목 | 설정값 |
|------|--------|
| 프레임워크 미리 설정 | 없음 |
| 빌드 명령 | (비워둠) |
| 빌드 출력 디렉터리 | (비워둠) |

---

## 8단계: .gitignore 설정

GitHub에 올리지 않아야 할 파일/폴더를 지정합니다.

```gitignore
# PDCA 개발 문서 - GitHub에 업로드하지 않음
docs/

# macOS 시스템 파일
.DS_Store
```

> `docs/` 폴더를 제외하는 이유: 개발 계획/설계 문서는 내부용으로,
> 실제 웹사이트에 노출될 필요가 없습니다.

---

## 9단계: 첫 Push 및 배포 확인

```bash
# 파일 스테이징
git add .

# 커밋
git commit -m "feat: 초기 웹사이트 배포"

# Push (GitHub Actions 자동 트리거)
git push origin main
```

### 배포 확인 방법

**GitHub Actions 로그 확인:**
```
https://github.com/[org명]/[레포명]/actions
```
성공 시 로그에 표시:
```
✨ Deployment complete!
URL: https://[hash].[프로젝트명].pages.dev
```

**Cloudflare Pages 대시보드 확인:**
- Workers & Pages → 프로젝트 → **Deployments** 탭
- 최근 배포 시각이 push한 시각과 일치하면 성공

---

## 일상적인 업데이트 방법

```bash
# 1. 파일 수정 후
git add .

# 2. 변경 내용 커밋
git commit -m "업데이트 내용 설명"

# 3. GitHub에 Push → 자동 배포 시작
git push origin main

# 약 1분 후 www.sk-net.com 새로고침하면 반영됨
```

---

## 트러블슈팅

### 문제 1: Cloudflare OAuth 연동 시 502 오류

```
API Request Failed: GET /api/v4/organizations?parent.id=null (502)
```

**원인**: GitHub Organization 계정을 Cloudflare Pages에 직접 OAuth 연동할 때 발생
**해결**: Cloudflare 직접 연동 대신 GitHub Actions + `cloudflare/pages-action` 방식 사용

---

### 문제 2: workflow 파일 Push 거부

```
refusing to allow a Personal Access Token to create or update workflow
`.github/workflows/deploy.yml` without `workflow` scope
```

**원인**: GitHub PAT 토큰에 `workflow` 권한이 없음
**해결**: PAT 토큰 재발급 시 Scopes에서 `workflow` 체크 추가

---

### 문제 3: Push 시 remote rejected (fetch first)

```
! [rejected] main -> main (fetch first)
```

**원인**: 원격 레포에 로컬에 없는 커밋이 존재 (GitHub 웹에서 직접 파일 업로드 등)
**해결**:
```bash
git pull origin main --rebase
git push origin main
```

---

### 문제 4: GitHub Actions exit code 1 오류

**원인 후보**:
1. Cloudflare Pages 프로젝트가 존재하지 않음 → 1단계 프로젝트 먼저 생성 필요
2. `CLOUDFLARE_PROJECT_NAME` Secret이 실제 프로젝트명과 불일치
3. Cloudflare API 토큰 권한 부족

**해결**: GitHub Actions 로그에서 상세 오류 메시지 확인 후 해당 항목 수정

---

## 인프라 구성 요약

```
[로컬 개발 환경]
  www.sk-net.com/
  ├── index.html
  ├── back.png
  ├── favicon.ico
  ├── SK-Net_Terms-2024-v11.pdf
  ├── .gitignore          ← docs/ 제외
  └── .github/
      └── workflows/
          └── deploy.yml  ← 자동 배포 설정

[GitHub]
  레포: skcc-youngeung/www.sk-net.com
  Secrets:
    - CLOUDFLARE_API_TOKEN
    - CLOUDFLARE_ACCOUNT_ID
    - CLOUDFLARE_PROJECT_NAME

[Cloudflare Pages]
  프로젝트: www-sk-net-com
  도메인: www.sk-net.com
  배포 방식: GitHub Actions → pages-action
```

---

## 관련 링크

| 항목 | URL |
|------|-----|
| GitHub 레포 | https://github.com/skcc-youngeung/www.sk-net.com |
| GitHub Actions | https://github.com/skcc-youngeung/www.sk-net.com/actions |
| GitHub Secrets 설정 | https://github.com/skcc-youngeung/www.sk-net.com/settings/secrets/actions |
| GitHub PAT 발급 | https://github.com/settings/tokens |
| Cloudflare 대시보드 | https://dash.cloudflare.com |
| Cloudflare API Tokens | https://dash.cloudflare.com/profile/api-tokens |
| 웹사이트 | https://www.sk-net.com |

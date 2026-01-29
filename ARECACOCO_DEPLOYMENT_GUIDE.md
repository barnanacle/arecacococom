# arecacoco.com 카페24 배포 설정 가이드

## 📋 프로젝트 개요

- **프로젝트명**: arecacoco.com
- **GitHub 레포지토리**: `https://github.com/barnanacle/arecacococom`
- **도메인**: arecacoco.com
- **배포 대상 경로**: `/var/www/html/` (카페24 호스팅 서버 루트)
- **목표**: GitHub `main` 브랜치에 push 시 자동으로 카페24 서버에 배포

---

## 🔧 필요한 작업

### 1. GitHub Secrets 설정

GitHub 레포지토리 → Settings → Secrets and variables → Actions에서 다음 시크릿 추가:

| Secret Name | 설명 | 값 |
|-------------|------|-----|
| `CAFE24_FTP_HOST` | FTP 서버 주소 | (기존 magic8ball과 동일) |
| `CAFE24_FTP_USERNAME` | FTP 사용자명 | (기존 magic8ball과 동일) |
| `CAFE24_FTP_PASSWORD` | FTP 비밀번호 | (기존 magic8ball과 동일) |
| `CAFE24_SSH_HOST` | SSH 서버 주소 | (기존 magic8ball과 동일) |
| `CAFE24_SSH_USERNAME` | SSH 사용자명 | (기존 magic8ball과 동일) |
| `CAFE24_SSH_KEY` | SSH 개인키 | (기존 magic8ball과 동일) |

> **참고**: 같은 카페24 호스팅 서버를 사용하므로 Secrets 값은 magic8ball 레포지토리와 동일합니다.

---

### 2. 디렉토리 구조 생성

프로젝트 루트에 `.github/workflows/` 디렉토리를 생성하고 `deploy.yml` 파일을 추가합니다.

```
arecacoco.com/
├── .github/
│   └── workflows/
│       └── deploy.yml    ← 이 파일 생성
├── index.html
├── (기타 웹 파일들)
└── ...
```

---

### 3. deploy.yml 파일 작성

`.github/workflows/deploy.yml` 파일 내용:

```yaml
name: Deploy to Cafe24

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Deploy to Cafe24 via FTP
      uses: SamKirkland/FTP-Deploy-Action@v4.3.4
      with:
        server: ${{ secrets.CAFE24_FTP_HOST }}
        username: ${{ secrets.CAFE24_FTP_USERNAME }}
        password: ${{ secrets.CAFE24_FTP_PASSWORD }}
        server-dir: '/'
        log-level: verbose
        exclude: |
          **/.git*
          **/.git*/**
          **/node_modules/**
          **/.env
          **/.env.*
          **/README.md
          **/.gitignore
          **/.github/**
          **/.DS_Store
          **/Thumbs.db
          **/*.log
          **/*.tmp
          **/*.temp

    - name: Fix permissions after FTP upload
      uses: appleboy/ssh-action@v1.0.3
      with:
        host: ${{ secrets.CAFE24_SSH_HOST }}
        username: ${{ secrets.CAFE24_SSH_USERNAME }}
        key: ${{ secrets.CAFE24_SSH_KEY }}
        script: |
          echo "🔧 FTP 업로드 후 권한 수정 중..."

          # 권한 설정
          chown -R www-data:www-data /var/www/html
          chmod -R 755 /var/www/html

          # 업로드된 파일들 확인
          echo "📁 업로드된 파일들:"
          ls -la /var/www/html/

          echo "✅ arecacoco.com 배포 완료!"
        port: 22
        timeout: 30s
        command_timeout: 2m
```

---

## ⚠️ 중요 차이점 (magic8ball vs arecacoco.com)

| 항목 | magic8ball | arecacoco.com |
|------|------------|---------------|
| 배포 경로 | `/var/www/html/magic8ball/` | `/var/www/html/` (루트) |
| server-dir | `/var/www/html/magic8ball/` | `/` |
| Node.js 필요 | ✅ (서버 앱) | ❌ (정적 웹페이지) |
| PM2 재시작 | ✅ 필요 | ❌ 불필요 |
| npm install | ✅ 필요 | ❌ 불필요 |

---

## 📝 체크리스트

### GitHub 설정
- [ ] 레포지토리 `barnanacle/arecacococom` 생성 완료
- [ ] GitHub Secrets 6개 모두 설정
- [ ] `.github/workflows/deploy.yml` 파일 생성

### 로컬 프로젝트
- [ ] Git 초기화: `git init`
- [ ] 원격 저장소 연결: `git remote add origin https://github.com/barnanacle/arecacococom.git`
- [ ] 첫 커밋 및 푸시: `git add . && git commit -m "Initial commit" && git push -u origin main`

### 배포 확인
- [ ] GitHub Actions 탭에서 워크플로우 실행 확인
- [ ] https://arecacoco.com 접속하여 배포 결과 확인

---

## 🔄 배포 플로우 다이어그램

```
[로컬 개발]
    ↓ git push origin main
[GitHub Repository]
    ↓ trigger: push to main
[GitHub Actions]
    ↓ FTP-Deploy-Action
[카페24 서버 /var/www/html/]
    ↓ SSH로 권한 수정
[https://arecacoco.com 서비스]
```

---

## 🛠️ 트러블슈팅

### FTP 업로드 실패 시
1. GitHub Secrets 값 확인 (특히 비밀번호 특수문자)
2. FTP 포트(21) 방화벽 확인
3. server-dir 경로 확인 (`/`로 설정되어야 함)

### SSH 연결 실패 시
1. SSH 키 형식 확인 (PEM 포맷)
2. SSH 포트(22) 방화벽 확인
3. known_hosts 관련 이슈는 appleboy/ssh-action이 자동 처리

### 권한 문제 발생 시
1. www-data 그룹 권한 확인
2. 디렉토리 권한 755, 파일 권한 644 확인

---

## 📎 참고 파일

이 가이드는 `magic8ball` 프로젝트의 `.github/workflows/deploy.yml`을 참고하여 작성되었습니다.
정적 웹사이트 배포에 맞게 Node.js 및 PM2 관련 단계를 제거한 간소화 버전입니다.

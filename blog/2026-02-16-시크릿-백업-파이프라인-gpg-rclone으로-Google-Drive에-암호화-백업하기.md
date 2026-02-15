---
title: "시크릿 백업 파이프라인: gpg + rclone으로 Google Drive에 암호화 백업하기"
date: 2026-02-16
tags: [devops, security, backup, gpg, rclone]
---

# 시크릿 백업 파이프라인: gpg + rclone으로 Google Drive에 암호화 백업하기

## 문제: 흩어진 시크릿, 불안한 마음

1인 개발자로 여러 프로젝트를 운영하다 보면 `.env` 파일, SSH 키, API 키, 인증서가 곳곳에 흩어진다. contents-hub의 DB 비밀번호, econipass의 Keycloak 시크릿, church-finance의 Gmail SMTP 키... 이 파일들이 한순간에 날아간다면?

맥북이 고장 나거나, 실수로 `rm -rf`를 때리거나, SSD가 뻗거나. 시크릿은 코드와 달리 Git에 올릴 수 없다. 그래서 별도의 백업 파이프라인이 필요했다.

## 설계 원칙

1. **암호화 필수** — 클라우드에 평문 시크릿을 올리면 안 된다
2. **자동화 가능** — 명령 하나로 전체 백업이 되어야 한다
3. **보존 정책** — 무한정 쌓이면 안 되고, 최근 5개만 유지
4. **복원 가능** — 백업이 있어도 복원이 어려우면 무용지물

## 도구 선택

### rclone — 클라우드용 rsync

[rclone](https://rclone.org)은 40개 이상의 클라우드 스토리지를 커맨드라인에서 다루는 도구다. "rsync for cloud storage"라는 별명답게, 로컬 파일을 Google Drive, S3, Dropbox 등에 업로드/다운로드/동기화할 수 있다.

```bash
brew install rclone          # macOS
sudo apt install rclone      # Ubuntu

# Google Drive remote 설정 (1회)
rclone config create gdrive drive \
  client_id="..." client_secret="..."
```

설정 후에는 이렇게 쓴다:

```bash
rclone copy local-file gdrive:secrets-backup/   # 업로드
rclone copy gdrive:secrets-backup/file ./        # 다운로드
rclone lsf gdrive:secrets-backup/                # 목록
rclone delete gdrive:secrets-backup/old-file     # 삭제
```

### gpg — AES-256 대칭 암호화

gpg의 `--symmetric` 모드를 사용하면 패스프레이즈 하나로 파일을 AES-256 암호화할 수 있다. 키 쌍을 관리할 필요 없이 패스프레이즈만 기억하면 된다.

```bash
# 암호화
gpg --symmetric --cipher-algo AES256 secrets.tar.gz
# → secrets.tar.gz.gpg 생성 (패스프레이즈 입력)

# 복호화
gpg --output secrets.tar.gz --decrypt secrets.tar.gz.gpg
```

## 파이프라인 구조

전체 흐름은 이렇다:

```
[secrets-index.md]  인덱스 파일에서 경로 추출
        ↓
[staging 디렉토리]  프로젝트별로 파일 복사
        ↓
[tar.gz]            하나의 아카이브로 압축
        ↓
[tar.gz.gpg]        AES-256 암호화
        ↓
[Google Drive]      rclone으로 업로드
        ↓
[보존 정책]         최근 5개만 유지, 나머지 자동 삭제
        ↓
[로컬 정리]         임시 파일 삭제
```

### 시크릿 인덱스가 핵심

백업 대상을 자동 탐색하지 않는다. `secrets-index.md`에 수동으로 등록된 파일만 백업한다. 이유는 명확하다:

- 자동 탐색은 불필요한 파일을 포함할 위험이 있다
- 어떤 시크릿이 어디에 있는지 한눈에 파악할 수 있다
- 프로젝트가 추가되면 인덱스에 한 줄 추가하면 끝이다

### 스테이징 디렉토리 구조

```
~/elon/secrets-backup/2026-02-16/
├── projects/
│   ├── contents-hub/        .env, docker-compose.yml, docker-compose.prod.yml
│   ├── econipass/           .env.local, docker-compose.yml, docker-compose.prod.yml
│   ├── church-finance/      .env, gdrive_config.py
│   └── linked-insight/      .env
├── infra/
│   ├── ssh-keys/            .pem, .crt, .csv
│   ├── tccli/               default.credential
│   └── credentials/         gdrive-credentials.json, gdrive-token.json
└── manifest.json            복원에 필요한 원본 경로 매핑
```

`manifest.json`이 중요하다. 백업 파일의 상대 경로와 원본 절대 경로를 매핑해두어, 복원할 때 각 파일이 어디로 돌아가야 하는지 알 수 있다.

## Claude Code에서의 한계와 해결

한 가지 걸림돌이 있었다. gpg의 패스프레이즈 입력은 대화형(interactive) pinentry를 사용하는데, Claude Code의 비대화형 터미널에서는 이게 동작하지 않는다.

해결책: **2단계 분리**

1. **Claude Code가 하는 일** — 파일 수집, 스테이징, tar 압축, 실행 스크립트 생성
2. **사용자가 별도 터미널에서 하는 일** — 스크립트 실행 (gpg 패스프레이즈 입력 → 암호화 → 업로드)

```bash
# Claude Code가 생성하는 스크립트
bash ~/elon/secrets-backup/backup-upload.sh
```

이 방식으로 자동화할 수 있는 부분은 최대한 자동화하고, 패스프레이즈 입력이라는 보안상 불가피한 수동 단계만 사용자에게 맡긴다.

## 복원 흐름

복원도 동일한 2단계 패턴을 따른다:

1. Claude Code가 복원 스크립트 생성 → 사용자가 실행 (다운로드 + gpg 복호화)
2. Claude Code가 `manifest.json`을 읽고, 파일별로 diff 비교 후 복원

이미 존재하는 파일은 diff를 보여주고 덮어쓸지 확인한다. 없는 파일은 바로 복원하고 `chmod 600`으로 권한을 설정한다.

## 정리

| 구성 요소 | 역할 |
|----------|------|
| `secrets-index.md` | 어떤 시크릿이 어디에 있는지 (Single Source of Truth) |
| `tar` | 여러 파일을 하나로 묶기 |
| `gpg --symmetric` | AES-256 패스프레이즈 암호화 |
| `rclone` | Google Drive 업로드/다운로드 |
| `manifest.json` | 원본 경로 매핑 (복원용) |
| 보존 정책 | 최근 5개만 유지 |

비용: 0원. Google Drive 무료 15GB면 시크릿 백업에는 차고 넘친다.

하나의 명령(`/secrets backup`)으로 모든 프로젝트의 시크릿을 암호화해서 클라우드에 올린다. 맥북이 내일 당장 고장 나도, 새 맥에서 `/secrets restore` 한 번이면 모든 시크릿이 제자리로 돌아온다. 그게 이 파이프라인의 전부다.

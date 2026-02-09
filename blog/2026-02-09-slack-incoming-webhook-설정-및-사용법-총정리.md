---
title: "Slack Incoming Webhook 설정 및 사용법 총정리"
date: "2026-02-09"
tags: [slack, webhook, guide, automation]
---

# [Guide] Slack Incoming Webhook 설정 및 사용법 총정리

외부 서비스의 알림을 슬랙으로 수신하기 위한 가장 보편적인 방법인 **Incoming Webhook** 설정 과정을 단계별로 안내합니다.

## 1. Slack App 생성 및 설정

슬랙 API는 보안과 관리 편의성을 위해 개별 'App'을 통해 웨브훅을 관리합니다.

* **앱 생성**: [Slack API 사이트](https://api.slack.com/apps)에서 `Create New App`을 클릭한 뒤 `From scratch`를 선택하여 앱을 생성합니다.
* **기능 활성화**: 왼쪽 사이드바 메뉴 중 **Features > Incoming Webhooks** 페이지로 이동하여 상단의 `Activate Incoming Webhooks` 스위치를 **On**으로 변경합니다.

## 2. Webhook URL 발급 프로세스

워크스페이스의 권한 설정에 따라 발급 방식이 두 가지로 나뉩니다.

* **관리자 승인이 필요한 경우**: 하단에 `Request to Add New Webhook` 버튼이 나타납니다. 이를 클릭하여 채널을 선택하고 승인을 요청해야 합니다.
* **승인 완료 및 일반 상태**: 승인이 완료되면 `Add New Webhook to Workspace` 버튼이 활성화됩니다.
* **최종 발급**: 위 버튼을 눌러 메시지를 게시할 최종 채널을 선택하면 `https://hooks.slack.com/services/...`로 시작하는 고유 URL이 생성됩니다.

## 3. 실전 사용 및 테스트

생성된 URL은 HTTP POST 방식을 통해 데이터를 전달받습니다. 터미널에서 아래 명령어로 즉시 테스트가 가능합니다.

```bash
curl -X POST -H 'Content-type: application/json' \
--data '{"text":"[시스템 알림] 테스트 메시지입니다."}' \
YOUR_WEBHOOK_URL
```

* **메시지 포맷**: `{"text": "내용"}` 구조의 JSON 데이터를 기본으로 하며, 이모지 코드(`:white_check_mark:`)나 줄바꿈(`\n`) 등을 포함할 수 있습니다.

## 4. 보안 및 운영 주의사항

웨브훅 주소는 해당 채널의 전송 권한을 가진 키와 같으므로 관리에 주의해야 합니다.

* **보안 노출 방지**: URL이 외부에 공개될 경우 누구나 해당 채널에 메시지를 보낼 수 있습니다. GitHub 등 공개 저장소에 소스 코드를 올릴 때는 반드시 환경 변수 처리하여 주소가 직접 노출되지 않도록 해야 합니다.
* **전송 속도 제한(Rate Limiting)**: 단시간에 과도한 양의 메시지를 보낼 경우 슬랙 정책에 따라 일시적으로 전송이 차단될 수 있습니다.

---

이제 발급받은 URL을 코드 내 설정값으로 입력하여 자동화를 시작하실 수 있습니다.

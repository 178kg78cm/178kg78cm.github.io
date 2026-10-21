---
layout: post
title:  "자동차 보안 기술 : 위협, 법규, 차량 내부 보안"
description: "자동차 보안이 필요한 이유와 주요 공격 경로, 법규와 TARA, HSM·Secure Boot·SecOC·IDS 등 차량 내부 보안 기술을 정리합니다."
date:   2025-03-13 +0900
categories: Security
tags: [Security, ISO21434, SecOC, HSM]
---

## 왜 자동차 보안인가

차량은 이제 수십~수백 개의 ECU, 수천만 줄의 코드, 그리고 외부와 연결된 통신 모듈(텔레매틱스, Wi-Fi, Bluetooth, V2X)을 가진 **네트워크 장비**다.
연결이 늘어날수록 공격 표면(Attack Surface)도 넓어진다.

- **2015 Jeep Cherokee 해킹** : 보안 연구자 Charlie Miller와 Chris Valasek가 인포테인먼트 유닛의 셀룰러 연결을 통해 원격으로 CAN 버스에 접근, 조향·브레이크·변속에 개입하는 시연을 했다. 이 일로 약 140만 대가 리콜되었다.
- 이 사건 이후 "보안은 IT의 일"이 아니라 **차량 안전의 일부**로 다뤄지기 시작했다.

## 주요 공격 경로

| 구분 | 경로 | 예시 |
| --- | --- | --- |
| 원격 | 텔레매틱스, OTA 서버, 모바일 앱 | 서버 탈취 후 악성 업데이트 배포 |
| 근거리 | Bluetooth, Wi-Fi, 스마트키, TPMS | 릴레이 어택으로 차량 문 열기 |
| 물리 | OBD-II 포트, ECU 디버그 포트(JTAG) | 진단 명령 위조, 펌웨어 추출 |
| 내부 네트워크 | CAN, Ethernet | 메시지 스푸핑, 재전송(Replay), DoS |

CAN은 설계 당시 폐쇄망을 가정했기 때문에 **송신자 인증이 없다.** 버스에 접근만 하면 누구나 어떤 ID의 메시지든 보낼 수 있다는 것이 근본적인 약점이다.

## 법규와 표준

- **UN R155 (CSMS, Cyber Security Management System)**
  - 제조사가 차량 전 생애주기에 걸친 사이버보안 관리 체계를 갖추고 인증받아야 한다.
- **UN R156 (SUMS, Software Update Management System)**
  - 소프트웨어 업데이트(OTA 포함)를 안전하게 관리하는 체계를 요구한다.
  - EU에서는 2022년 7월 신규 차종, 2024년 7월부터 모든 신차에 적용되었다.
- **ISO/SAE 21434**
  - R155가 "무엇을 갖춰야 하는가"라면, 21434는 "어떻게 개발하는가"를 정의하는 엔지니어링 표준이다.
  - 핵심 활동이 **TARA(Threat Analysis and Risk Assessment)** 다.

### TARA 흐름

1. **자산 식별** : 보호해야 할 대상(키, 펌웨어, 제어 메시지 등)과 보안 속성(기밀성·무결성·가용성)
2. **위협 시나리오 도출** : STRIDE 등으로 자산별 위협 정의
3. **영향도 평가** : 안전, 재무, 운행, 개인정보(SFOP) 관점
4. **공격 경로 분석 / 공격 가능성(Attack Feasibility) 평가** : 필요 시간, 전문성, 장비, 대상 지식, 기회의 창
5. **위험 결정 및 대응** : 회피, 감소, 공유, 수용 → 사이버보안 목표와 요구사항으로 연결

기능안전의 HARA가 "고장이 나면 얼마나 위험한가"를 본다면, TARA는 "**누군가 의도적으로 공격하면** 얼마나 위험한가"를 본다.

## 차량 내부 보안 기술

### HSM (Hardware Security Module)
- MCU 안에 분리된 보안 코어/영역으로 **암호 키 저장과 암호 연산**을 담당한다.
- 메인 코어가 탈취되어도 키는 HSM 밖으로 나오지 않는다. (예: Infineon AURIX의 HSM, SHE)

### Secure Boot
- 부트 단계마다 다음 단계 이미지의 **서명(또는 MAC)을 검증**한 뒤 실행한다.
- 변조된 펌웨어는 부팅되지 않는다. 신뢰의 시작점(Root of Trust)은 HSM과 OTP 영역이다.

### SecOC (Secure Onboard Communication)
- AUTOSAR 표준 모듈로, CAN/Ethernet 메시지에 **MAC과 Freshness Value**를 붙여 전송한다.
  - MAC : 송신자 인증과 무결성 확보 (CMAC-AES128 등)
  - Freshness Value : 카운터/타임스탬프로 **재전송 공격 방지**
- CAN Classic은 8바이트라 MAC을 잘라서(truncated) 넣어야 하므로, 여유가 있는 CAN FD와 함께 쓰이는 경우가 많다.

### Secure Gateway / 방화벽
- 외부와 연결된 도메인(인포테인먼트, 텔레매틱스)과 제어 도메인(파워트레인, 섀시)을 분리하고, 허용된 메시지만 라우팅한다.
- 진단 접근은 UDS **Security Access(0x27)** 나 인증서 기반 인증(0x29)으로 제한한다.

### IDS (Intrusion Detection System)
- 메시지 주기, 페이로드 범위, 발생 빈도를 감시해 **평소와 다른 트래픽을 탐지**하고 보안 관제(VSOC)로 보고한다.

관련 글 : [자동차 사이버보안 개요](/2025-03-11-SecureCoding/)

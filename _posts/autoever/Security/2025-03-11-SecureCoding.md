---
layout: post
title:  "자동차 사이버보안 개요 : ISO 21434와 A-SPICE for Cybersecurity"
description: "자동차 사이버보안 관련 표준인 ISO/SAE 21434와 A-SPICE for Cybersecurity, 공격 가능성 평가 등 주요 용어를 정리합니다."
categories: Security
tags: [Security, ISO21434, A-SPICE]
---
## Cybersecurity Overview

자동차 사이버 보안의 중요성이 커지고 있다. 차량이 외부 네트워크와 연결되고(커넥티드카, OTA), 소프트웨어가 기능을 결정하는 SDV로 바뀌면서 **해킹이 곧 안전 문제**가 되었다.

자율주행차에서는 안전을 세 가지 관점으로 나눠 본다.

| 표준 | 다루는 위험 |
| --- | --- |
| ISO 26262 (기능안전) | 하드웨어/소프트웨어 **고장** |
| ISO 21448 (SOTIF) | 고장은 없지만 **의도된 기능의 한계** (센서 인식 한계 등) |
| ISO/SAE 21434 (사이버보안) | **의도적인 공격** |

## Cybersecurity related Standards

1. **ISO/SAE 21434 Cybersecurity Engineering**
   - 차량 E/E 시스템의 개념 단계부터 개발, 생산, 운영, 폐기까지 **전 생애주기의 사이버보안 활동**을 정의한다.
   - 핵심은 TARA(위협 분석 및 위험 평가)와, 그 결과로 도출되는 사이버보안 목표/요구사항이다.
2. **A-SPICE for Cybersecurity**
   - A-SPICE의 프로세스 평가 방식에 보안 프로세스를 추가한 것이다.
   - SEC.1(보안 요구사항), SEC.2(보안 구현), SEC.3(위험 처리 검증), SEC.4(위험 처리 확인), MAN.7(사이버보안 위험 관리), ACQ.2(공급자 요구사항) 등이 추가된다.

## 용어 정리

### Cybersecurity specification

- 사이버 보안 요구사항과, 그것과 상호작용하는 architectural design을 함께 기술한 것
- "무엇을 지켜야 하는가(요구사항)"와 "어느 구성요소가 그것을 담당하는가(설계)"가 함께 있어야 검증할 수 있다.

### Architectural design

- 정적 설계와 동적 설계로 구분된다.
  - **정적 설계** : 구성 요소와 인터페이스, 데이터 흐름 (구조)
  - **동적 설계** : 상태 전이, 시퀀스, 타이밍 (동작)
- 보안 관점에서는 **신뢰 경계(Trust Boundary)**, 즉 외부 입력이 들어오는 지점을 설계에서 명확히 드러내는 것이 중요하다.

### Verification & Validation

- 검증과 확인
- 고객의 입장에서의 검사이면 **Validation** / 설계자의 관점에서 잘 만들어졌는지에 대한 검사이면 **Verification**
- 보안에서는 Verification으로 보안 요구사항이 구현되었는지 확인하고, Validation으로 **실제 공격에 견디는지**(침투 테스트, 퍼징) 확인한다.

### Automotive SPICE 4.0

- 2023년 말 발표된 Automotive SPICE 4.0에서는 고객의 요구사항과 맞는지 확인하는 **Validation Process Group(VAL.1)** 이 새로 생겼다.
- 머신러닝(MLE), 하드웨어(HWE) 엔지니어링 프로세스 그룹도 추가되었다.

### Attack feasibility

공격이 실제로 성공하기 얼마나 쉬운가를 평가하는 TARA 단계다. ISO 21434에서는 공격 가능성 평가 방법으로 공격 잠재력 기반(Attack potential), CVSS 기반, 공격 벡터 기반을 제시한다. 공격 잠재력 기반 평가에서는 다음 요소를 점수화한다.

| 요소 | 의미 |
| --- | --- |
| Elapsed Time | 공격 준비/수행에 걸리는 시간 |
| Specialist Expertise | 필요한 전문성 수준 |
| Knowledge of the Item | 대상 시스템에 대한 정보 접근성 |
| Window of Opportunity | 접근 기회 (원격/물리적 접근 필요 여부) |
| Equipment | 필요한 장비 (일반/특수/맞춤 장비) |

공격 가능성(High~Very Low)과 피해 영향도(Severe~Negligible)를 조합해 **위험 값**을 정하고, 그에 따라 대응 방안을 결정한다.

### R&R

- 'Role & Responsibilities'의 약자로, 조직에서 구성원이 맡은 역할과 책임을 뜻한다. 업무분장 규정으로도 불리며, 회사의 체계화를 위해 활용한다.
- 21434는 조직 차원의 사이버보안 관리(정책, 역할, 역량, 문화)를 요구하기 때문에 R&R 정의가 출발점이 된다.

### 보안 문화

강의에서는 A-SPICE Culture와 Cybersecurity Culture가 잘 형성되어 있는 회사로 LG, 현대모비스를 예로 들었다.

### DIA → CIA

- 기능안전에서 OEM과 공급사가 안전 활동의 책임을 나누는 문서가 **DIA (Development Interface Agreement)** 다.
- 사이버보안에서는 같은 역할을 **CIA (Cybersecurity Interface Agreement)** 가 한다. 누가 TARA를 하고, 누가 취약점을 관리하고, 사고 발생 시 누가 대응하는지를 합의한다.

### 기능과 비기능

- **기능 요구사항** : 입력이 있으면 출력이 있는 결과물 (예: 버튼을 누르면 문이 열린다)
- **비기능 요구사항** : 기능을 구현하는 데 필요한 다양한 품질 조건 (성능, 안전성, **보안성**, 신뢰성)
- 보안 요구사항은 대부분 비기능 요구사항이라 놓치기 쉽다. 그래서 TARA로 체계적으로 도출해야 한다.

## 시큐어 코딩

| 취약점 | 예방 |
| --- | --- |
| 버퍼 오버플로 | 경계 검사, `strcpy` 대신 길이 제한 함수 사용 |
| 정수 오버플로 | 연산 전 범위 확인 |
| 입력값 미검증 | 외부 입력(통신 메시지, 진단 요청)은 항상 범위·길이 검증 |
| 초기화되지 않은 변수 | 선언 시 초기화 |
| 하드코딩된 키 | 키는 HSM에 저장 |

- 코딩 표준 : **MISRA C**(안전), **CERT C**(보안), AUTOSAR C++14
- 정적 분석 도구로 자동 검사하고, 리뷰로 보완한다.

다음 글 : [자동차 보안 기술 : 위협, 법규, 차량 내부 보안](/2025-03-13-Automative_Security/)

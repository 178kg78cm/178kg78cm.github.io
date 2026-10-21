---
layout: post
title: "차량용 이더넷 (1) : CAN 노드 구조와 차량 E/E 아키텍처"
description: "CAN 노드의 Host, Controller, Transceiver 구조와 송수신 흐름, 차량 E/E 아키텍처, ECC·Lockstep 같은 하드웨어 안전 메커니즘을 정리합니다."
date:   2025-03-19 +0900
categories: Network
tags: [Ethernet]
---

두 가닥의 선으로 통신을 가능하게 한 것이 CAN 통신이다.

- 차량 통신에서 먼저 따져야 할 질문은 **"CAN 통신이 맞나?"** 이다.
  - 통신에는 예측 불가능한 지연이 생길 수 있다.
  - 중요한 것은 빠르기보다 **deadline 내에 끝낼 수 있는가**이다. Yes라면 실시간 통신(Real-time)이다.

## CAN Node의 구조와 기능 정리

하나의 CAN Node는 다음과 같은 3가지 주요 구성 요소로 이루어져 있다.

``` text
[ Application (Host) ]
         ↓
[ CAN Controller ]
         ↓
[ CAN Transceiver ]
         ↓
[ CAN Bus ]
```

---

### 1. Host

**설명**  

- Application 코드 및 AUTOSAR 통신 스택이 위치한 영역
- CAN 통신을 위한 데이터 생성 및 처리 수행

**기능**  

- CAN 메시지 생성 (`Rte_Write`, `CanIf_Transmit` 등)
- 수신 메시지 처리 (`Rte_Read`, 신호 해석 등)
- AUTOSAR 모듈들 (SWC, RTE, COM, PduR 등) 포함

---

### 2. CAN Controller

**설명**  

- MCU 내부에 존재하는 하드웨어 모듈
- CAN 프로토콜 처리 담당

**기능**  

- 프레임 인코딩/디코딩 (ID, DLC, CRC, ACK 등)
- Tx/Rx Mailbox 관리
- 에러 감지 및 상태 제어 (Bus-Off, Passive 등)
- Bit Timing 및 Baudrate 설정

---

### 3. CAN Transceiver

**설명**  

- 디지털 신호 ↔ 전기 신호를 변환하는 외부 칩
- CAN Controller와 물리적 CAN Bus 사이 연결

**기능**  

- 디지털 신호 → Differential Signal (CAN_H / CAN_L)
- 수신 전압 차 → 디지털 신호 복원
- 버스 보호 (ESD, 전압 제한, 절연 등)
- 예: TJA1040, MCP2551, TJA1051 등

---

## 송수신 흐름 예시

### 송신 흐름

1. Host에서 메시지 생성
2. CAN Controller가 CAN 프레임 생성
3. Transceiver가 전기 신호로 변환
4. CAN Bus를 통해 전송

### 수신 흐름

1. Transceiver가 신호를 수신하여 디지털화
2. CAN Controller가 프레임 디코딩
3. Host에 메시지 전달 (Application 처리)

---

| 구성 요소         | 위치         | 주요 기능 |
|------------------|--------------|-----------|
| **Host**         | MCU 내 Application 영역 | 데이터 처리, AUTOSAR 통신 스택 |
| **CAN Controller** | MCU 내부 IP 모듈 | 프로토콜 처리, 프레임 제어 |
| **CAN Transceiver** | 외부 IC | 전기적 신호 변환, 보호 기능 |

- CAN FD (Flexible Datarate)

- Flexray
최대 10Mbps 속도
현대는 사용하지 않고 고급차들이 많이 사용

- Ethernet
Ethernet RJ-45 (사무용, 두 쌍 이상의 선)
BroadR-Reach
=> Ethernet AVB / TSN

- Powertrain
일반적으로 10ms로 CAN 통신해서 RPM의 값을 전송한다.

- ADAS(Advanced Driver Assistance System)
대규모 센서들이 많기 때문에 **대용량 데이터 전송**과 **실시간 데이터 전송**이 둘 다 중요한 도메인이다. 그러나 Linux 기반 운영체제를 사용하기 때문에 End to End Latency는 250us~1ms로 긴시간이 필요하다. 20-100Mbps bandwidth가 필요하고 통신속도는 1~500ms까지 소요된다.

### 차량 E/E Architecture

SDV로의 변화를 위해 차량 E/E를 단계적으로 변화를 주고 있다.

1. Distributed
　전통적인 분산시스템을 가진 아키텍쳐

2. Domain centralized
　기존 제어기는 단순 센서 및 엑추에이터 기능을 수행하고 통합 제어 등을 DCU에서 수행하는 형태로 변화가 필요함
일반적으로 기존의 도메인에서 가지고 있는 기능 중 기존의 ECU는 단순 센서 및 액추에이터 역할을 수행하고 통합제어가 필요한 기능은 DCU에서 수행함
DCU간에 고속의 Ethernet으로 연결되어 다양한 도메인의 정보를 빠르게 처리할 수 있음
기존 제어기 대비 고성능의 CPU, 필요시 GPU도 탑재

3. Vehicle centralized
　고성능 Vehicle computer가 지역별로 배치된 Zone Control Unit의 정보를 수집하여 통합 제어

Vehicle Centralized Architecture이랑 Zonal Architecture을 같이 사용한다.
Central Computer은 일반적으로 2대 이상있다.
main 1개, backup 및 sub 1 + @ 개

- **Event Triggered vs Time Triggered** : 이벤트가 생길 때마다 보내는 방식보다, 정해진 시간 슬롯에 보내는 방식이 **결정적(Deterministic)** 이라 실시간 제어에 유리하다.




## ISO 26262와 하드웨어 안전 메커니즘

ISO 26262에는 위험, 보안, 문제가 생길 수 있는 부분을 보완할 수 있는 메커니즘이 정의되어 있다. 중앙 컴퓨터처럼 여러 기능을 한곳에 모을수록, 하드웨어 고장을 **스스로 검출**하는 장치가 필수가 된다. 대표적인 것이 아래 세 가지다.

## ECC

ECC(Error Correcting Code)는 데이터 전송 또는 저장 시 발생할 수 있는 오류를 검출하고 수정할 수 있는 코드를 의미한다.

- 데이터 무결성 보장
- 메모리에 데이터와 함께 검사 비트를 저장해서, 읽을 때 비트 오류를 확인한다.
- 일반적인 SECDED 방식은 **1비트 오류는 수정(Single Error Correction), 2비트 오류는 검출(Double Error Detection)** 한다.
- 우주선(Cosmic ray)이나 전기적 노이즈로 생기는 **소프트 에러(Bit Flip)** 대응에 쓰인다. TC275 같은 차량용 MCU는 Flash와 RAM에 ECC를 기본으로 갖고 있다.

## Lockstep

두 개 이상의 프로세서가 같은 명령어를 동시에 실행하여 결과를 비교하고 일치 여부로 오류를 검출한다.

- 메인 코어와 체커 코어가 같은 연산을 하고, 비교기(Comparator)가 매 클럭 결과를 비교한다.
- 결과가 다르면 즉시 오류를 보고하고 안전 상태(Safe State)로 전환한다.
- 두 코어가 같은 원인으로 동시에 틀리는 것(공통 원인 고장)을 막기 위해, 클럭을 몇 사이클 어긋나게 하거나 물리적으로 떨어뜨려 배치한다.
- 오류를 **검출**할 뿐 **수정**하지는 않는다. ASIL-D 수준의 MCU에서 사용된다.

## Hamming Code

ECC의 가장 기본적인 형태로, 데이터 비트 사이에 **패리티 비트**를 끼워 넣어 1비트 오류의 위치까지 찾아낸다.

- 패리티 비트는 1, 2, 4, 8, ... (2의 거듭제곱) 위치에 둔다.
- 각 패리티 비트는 자기 위치 번호의 비트가 1인 위치들을 검사한다.

```text
Hamming(7,4) : 데이터 4비트 + 패리티 3비트

위치 :  1   2   3   4   5   6   7
비트 :  p1  p2  d1  p4  d2  d3  d4

p1 = d1 ^ d2 ^ d4     (위치 1,3,5,7 검사)
p2 = d1 ^ d3 ^ d4     (위치 2,3,6,7 검사)
p4 = d2 ^ d3 ^ d4     (위치 4,5,6,7 검사)
```

- 수신 측에서 같은 검사를 다시 해서 나온 3비트 값(신드롬)이 0이면 오류 없음, 0이 아니면 **그 값이 곧 오류 난 비트의 위치**다. 그 비트를 뒤집으면 수정된다.
- 예 : 신드롬이 `101`(5)이면 5번 비트(d2)가 틀린 것이다.
- 여기에 전체 패리티 1비트를 추가하면 2비트 오류까지 검출하는 SECDED가 된다.

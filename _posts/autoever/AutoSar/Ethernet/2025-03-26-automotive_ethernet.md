---
layout: post
title: "차량용 이더넷 (4) : 통신 스택, SOME/IP와 DoIP"
description: "AUTOSAR 이더넷 통신 스택과 CANoe 시뮬레이션, CAN 기반 sleep 제어, SOME/IP와 DoIP를 정리합니다."
date:   2025-03-19 +0900
categories: Network
tags: [Ethernet]
---

## AUTOSAR Communication Stack

PDU router에서 아래 레이어의 통신 방법을 몰라도 상위 통신 가능. RTE에는 data 즉 signal만 올라갈 수 있도록 함.

CAN과 이더넷은 PduR 아래에서 갈라진다.

```text
          SWC ─ RTE ─ COM ─ PduR
                             │
          ┌──────────────────┴──────────────────┐
        CanIf                                  SoAd (Socket Adaptor)
          │                                      │
        Can Driver                             TcpIp (TCP/UDP/IP)
          │                                      │
        CAN Transceiver                        EthIf ─ Eth Driver ─ Eth Transceiver
```

- **SoAd** : PDU를 소켓(IP 주소 + 포트)에 매핑한다. CanIf가 PDU를 CAN ID에 매핑하는 것과 같은 역할이다.
- 상위 계층(COM, RTE, SWC)은 **아래가 CAN인지 이더넷인지 모른다.**

## Simulation on CANoe

CANoe에서는 실제 ECU 없이 네트워크와 노드를 가상으로 만들어 통신을 검증할 수 있다.

1. 네트워크 정의 : CAN은 DBC, 이더넷/SOME-IP는 ARXML(또는 FIBEX)로 메시지와 서비스를 정의한다.
2. 노드 구성 : 시뮬레이션할 ECU는 가상 노드로, 실제 ECU는 Real 노드로 둔다(Rest-bus Simulation).
3. 동작 구현 : CAPL로 메시지 송신, 수신 처리, 타이머 동작을 작성한다.
4. 측정 : Trace, Graphics, State Tracker로 메시지 흐름과 신호 값을 확인한다.

자세한 CAN 실습은 [CANoe 실습](/2025-04-01-CANoe/) 글에 정리했다.

## CAN 기반 sleep 제어

Transceiver에 배터리 전압이 직접적으로 계속 인가되어 항상 켜져있다. 이 Transceiver를 기반으로 메세지가 전송될 때마다 나머지 CAN 및 ECU가 Wakeup해서 동작하게 된다.

- **Partial CAN Network (Partial Networking)**
  기존은 Message를 보내면 모든 장치가 깨어나는데, 부분 제어를 하는 방식으로 개선한 버전이다.
  트랜시버가 **미리 지정된 Wake-up 프레임(특정 ID/데이터)** 일 때만 ECU를 깨우므로, 필요한 ECU만 동작하고 나머지는 잠든 상태로 남아 **암전류(대기 전력)** 를 줄인다.
- 이더넷에도 같은 개념이 있다. OPEN Alliance TC10은 링크 단위의 Sleep/Wake-up을 정의한다.

## SOME/IP

**Scalable service-Oriented MiddlewarE over IP**

CAN은 "누가 받을지 모르고 신호를 뿌리는" 방식이라면, SOME/IP는 **서비스 단위로 필요한 ECU끼리 통신**하는 서비스 지향 미들웨어다.

| 구성 | 내용 |
| --- | --- |
| **Service** | 기능의 묶음 (Service ID로 식별) |
| **Method** | 원격 함수 호출 (Request/Response, Fire&Forget) |
| **Event** | 서버가 구독자에게 값을 알림 (Publish/Subscribe) |
| **Field** | 값 + Getter/Setter/Notifier |
| **SOME/IP-SD** | Service Discovery. 서비스 제공(Offer)과 구독(Subscribe)을 런타임에 협상 |

- 헤더에 Service ID, Method ID, Client ID, Session ID, Message Type 등이 들어간다.
- UDP와 TCP 모두 사용할 수 있다.
- AUTOSAR Adaptive Platform(ara::com)의 기본 통신 수단이며, Classic Platform에서도 SomeIpXf, SD 모듈로 지원한다.
- 같은 서비스 지향 미들웨어로 **DDS**도 있다. 로봇(ROS 2) 분야에서 많이 쓰이며, QoS 설정이 풍부한 것이 특징이다.

## DoIP

**Diagnostics over Internet Protocol (ISO 13400)**

진단 프로토콜 UDS(ISO 14229)를 이더넷/IP 위에서 주고받기 위한 전송 규격이다. CAN에서 UDS를 나르는 것이 CAN TP(ISO 15765)라면, 이더넷에서는 DoIP가 그 역할을 한다.

1. **Vehicle Discovery** : 진단기가 UDP 브로드캐스트로 차량을 찾고, 차량은 VIN과 논리 주소를 응답한다.
2. **Routing Activation** : TCP 연결 후 진단 통신을 활성화한다.
3. **Diagnostic Message** : UDS 요청/응답을 DoIP 헤더에 담아 전송한다. 게이트웨이(DoIP Edge Node)가 필요하면 내부 CAN ECU로 변환해 전달한다.

| 구분 | CAN 진단 | DoIP |
| --- | --- | --- |
| 전송 | ISO-TP (프레임당 8/64바이트) | TCP/IP |
| 속도 | ~500 kbps | 100 Mbps 이상 |
| 용도 | 일반 진단 | 대용량 SW 리프로그래밍, OTA, 원격 진단 |

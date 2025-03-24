---
layout: post
title:  "Autosar (4) : Heating Project"
date:   2025-03-19 +09001
categories: Autoever
tag: "autosar"
---

## 프로젝트 개요
　신호에 따라서 온열기구를 동작시킨다. 단, Heating기능이 따로 없으므로 LED를 제어하는 방식으로 대체한다. 

- LED1 (Port E Pin 4): ECU가 동작 중임을 표시 (GPIO Out)
- LED2 (Port E Pin 5): 열선 시트가 동작 중임을 표시 (GPIO Out)
- LED4 (Port E Pin 7): 열선 시트 강도 (PWM)
- Potentiometer (Port B Pin 4): 열선 시트 강도 조절용 다이얼 (ADC)

![alt text](practice_ASW_architecture.png)
각 4개의 component를 구현한다. 

- Symbol 설명
  - ▶--▶
    Send - Receiver communication (queue를 사용한다.)
  - O--C
    Server - Client
  - Server runable
    서버에 연결되어 있는 블록으로 client에서 server를 호출할 때마다 동작
  - Runable
    client 사이에 연결되어 있는 블록으로 특정 시간 마다 동작한다. 여기서는 10ms인것을 확인할 수 있다.


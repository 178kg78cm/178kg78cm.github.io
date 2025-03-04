---
layout: post
# title:  "임베디드MCU | ADC"
date:   2025-02-18 +09001
categories: 임베디드MCU
tag: ["Infineon", "TC275"]
---

## 코드 프로파일링
성능 분석은 프로그램의 시간 복잡도 및 공간, 특정 명령어 이용, 함수 호출 주기와 빈도 등을 측정하는 동적 프로그램 분석의 한 형태.
``` c
    P10_IOCR0.U &= ~(0x1f<<19); // bit 0 초기화
    P10_IOCR0.U |= (0x10<<19); //
        

    while(1)
    {
        P10_OUT.U = 0x1<<2;
    }
```

``` assembly
48            P10_IOCR0.U &= ~(0x1f<<19); // bit 0 초기화
8000006c:   movh.a  a15,0xF004
80000070:   ld.w  d15,[a15]-0x4FF0
80000074:   insert  d15,d15,0x00,0x13,0x05
80000078:   movh.a  a15,0xF004
8000007c:   st.w  [a15]-0x4FF0,d15
49            P10_IOCR0.U |= (0x10<<19); //
80000080:   movh.a  a15,0xF004
80000084:   ld.w  d15,[a15]-0x4FF0
80000088:   insert  d15,d15,0x01,0x17,0x01
8000008c:   movh.a  a15,0xF004
   st.w  [a15]-0x4FF0,d15
```

- Reg 직접 제어
![alt text](/assets/images/regi.png)
instrction 6C~C4

- Ifx 함수 제어
![alt text](/assets/images/func1.png)
![alt text](/assets/images/func2.png)
instrction 6C~E4

Disassembly의 차이가 32만큼 차이가 난다.

함수를 사용해서 코드를 짠다
이후 최적화할 때, 함수를 레지스터를 직접 접근하는 방향으로 수정한다

---

## 컴파일러 최적화
컴파일러 개발자는 최적화 기법 중 **최소한의 노력으로 최대한의 효과**를 낼 수 있는 가장 효율적인 방법을 골라서 컴파일러 최적화에 적용해야한다.

- 핍홀 최적화
- 지역 최적화
- 전역 최적화
- 루프 최적화
- 프로시저 간 최적화
- 

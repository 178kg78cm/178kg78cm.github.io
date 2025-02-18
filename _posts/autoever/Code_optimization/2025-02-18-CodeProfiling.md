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
80000090:   st.w  [a15]-0x4FF0,d15
```
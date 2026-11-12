# Real-Time ADCP (Adaptive DMA Communication Pipeline)

__Working in Progress...__

극한의 시스템 부하 상황에서도 수신 인터럽트를 동적으로 제어하여, 고우선순위 제어 태스크의 마감 시간(Deadline)을 보장하는 하드 리얼타임(Hard Real-Time) 통신 파이프라인.
- 본 프로젝트는 제한된 자원(ex. CPU Core 1개)으로 예측 불가능한 고부하 극한 상황에서 발생할 수 있는 시스템 지연을 원천 차단하기 위해 설계. 통신 패킷이 짧은 주기로 쏟아지는 환경에서도 수신 인터럽트(ISR)의 간섭을 능동적으로 통제함으로써, 고우선순위 핵심 제어 태스크가 어떠한 방해 없이 안전하게 실행 마감 시간(Deadline)을 지키도록 보장.

## 🛠 Development Environment

- MCU / Board: STM32F407G-DISC1
  - D-Cache가 없어 DMA와 Cache Coherency 가능성 낮음
- Sensor: TF-Luna LiDAR (1D 거리 측정 센서)
  - USART 기준 Packet 규격

Byte No.|Field Name|Description|Value
---|---|---|---
Byte 0|Header 1|Frame Start Identifier|`0x59`
Byte 1|Header 1|Frame Start Identifier|`0x59`
Byte 2|Dist_Low|Distance Value(Low Byte)|`Distance = Dist_Low + Dist_High << 8`
Byte 3|Dist_High|Distance Value(High Byte)|`Distance = Dist_Low + Dist_High << 8`
Byte 4|Strength_Low|Signal Strength(Low Byte)|`Strength = Strength_Low + Strength_High << 8`
Byte 5|Strength_High|Signal Strength(High Byte)|`Strength = Strength_Low + Strength_High << 8`
Byte 6|Temp_Low|Internal Temperature(Low Byte)|`Temp`(°C) `= (Temp_Low + Temp_High << 8) / 8.0 - 256.0`
Byte 7|Temp_High|Internal Temperature(High Byte)|`Temp`(°C) `= (Temp_Low + Temp_High << 8) / 8.0 - 256.0`
Byte 8|Checksum|Lower 8 bits of the Sum of the Preceding 8 Bytes|`(Byte0 + Byte1 + ... + Byte7) & 0xFF`

- IDE: STM32CubeIDE 1.19.0
  - STM32CubeMX 통합 (.ioc)
- RTOS: FreeRTOS v202411.00
- Debugging / Profiling: SystemView V4.10b (J-Link 연결)
  - 실행 과정을 Continuous Mode로 기록
- Sensor Viewer: Benewake LiDAR Viewer
  - TF-Luna의 Frame Rate, Baud Rate 설정 

## 🚀 Project Progress (현재 진행 상황)

### 1. 하드웨어 환경 구성 완료
- STM32F407G-DISC1 및 TF-Luna 센서 하드웨어 결선 및 초기 설정 완료.

Voltage and Current of each Hardware

Hardware|Voltage|Consumption/Supply Electrical Current|Description
---|---|---|---
9V 1A Adoptor|9V|Max 1A supply|Convert 220v ➔ 9V
HW-131 Power Module|5V / 3.3V Converter|Max 700mA supply|Convert 9V ➔ 5V / 3.3V
TF-Luna LiDAR|5V (Communication 3.3V)|Avg 70mA / Max 150mA|5V for power / 3.3V for communication with the MCU
STM32F407G-DISC1|3.3V|Max 20 ~ 25mA per pin|Main board for this project

Pin Arrangement

Hardware|STM32F407 Pin|Bread Board
---|---|---
TF-Luna LiDAR|RXD ➔ PA2 / TXD ➔ PA3|GND ➔ (-) / VCC 5V ➔ (+)
Common GND|GND|GND

MCU Frequency

Bus|Frequency|Peripheral
---|---|---
AHB 1 Bus(SYSCLK)|168MHz|-
APB 1 Bus(APB 1 Peripheral Clock)|42MHz|PA2 (AF7: USART2_TX) / PA3 (AF7: USART2_RX)

TF-Luna Frequency

Communication|Frame Rate|Baud Rate
---|---|---
저속 통신|250Hz|115200bps
고속 통신|250Hz|921600bps

### 2. 순수 USART 통신 속도별 동작 분석 (SystemView 검증)

저속 및 고속 환경에서 기존 Interrupt / Polling 방식의 한계를 SystemView를 통해 확인하고 문제점을 도출.

- __저속 통신 환경 (115,200bps, 주기 4ms)__
  - 바이트 도착 간격이 약 86.8 µs
  - __Interrupt 방식__
    - TF-Luna의 바이트 수(9 byte)만큼 하드웨어 인터럽트(RXNE)가 발생.
    - 패킷당 바이트 단위(86.8 µs 간격)로 인터럽트가 발생하여 문맥 교환 오버헤드 및 CPU 소모 발생.
  - __Polling 방식__
    - 방식: CPU가 상태 플래그 RXNE(RXNEIE 비활성화)를 계속 확인(Spin Wait)하며 DR 레지스터에서 직접 바이트를 가져가는 방식.
    - Polling을 활용할 경우, USART의 Data Recieve Reigster는 1개(16byte Reserved, 16byte Use)만 사용하므로 polling 간격을 미세하게 두지 않는 이상 Overrun Error가 발생할 확률이 존재. Polling 주기를 86.8 µs 미만으로 잡아야 안전. 이 실험에서는 Spin Wait으로 진행.
    - 패킷 수신 내내 상태 플래그(RXNE)를 확인해야 하므로 CPU 자원의 심각한 낭비 발생.

- __고속 통신 환경 (921,600bps, 주기 4ms)__
  - 바이트 도착 간격이 10.85 µs로 극단적으로 짧아짐.
  - __Interrupt 방식__
    - 10.85 µs 안에 ISR을 처리해야 함. 
    - 바이트 단위 하드웨어 인터럽트의 ISR 소요 시간은 약 6.5 ~ 6.6 µs 소요. 
    - Single-Task인 경우 정상 동작하지만 Multi-Tasking인 경우 Context Switch의 오버헤드가 증가해 Overrun Error가 발생할 확률이 높아짐.
    - ISR로 인한 CPU 소모. 
  - __Polling 방식__
    - 저속 통신 환경에서 polling 방식과 비슷하지만 바이트 도착 간격이 약 10.85 µs으로 짧아졌고 data register가 1개 밖에 없어 촘촘한 polling 또는 Spin Wait을 하지 않는 이상 Overrun Error가 발생할 확률이 더 높아짐.
  - CPU 자원의 심각한 낭비 발생.
- __분석 (문제점)__
  - 저속 통신의 비효율: 매 바이트마다 하드웨어 인터럽트 발생 및 상태 플래그 상시 감시로 CPU 소모 및 ORE 발생 가능.
  - 고속 통신의 비효율: 바이트 도착 간격이 10.85 µs로 짧아져 ORE 발생 가능성이 더 높아지고 높은 CPU 소모 발생.

### 3. DMA 기반 아키텍처 도입

고속 통신(921,600bps)에서의 Overrun을 방지하고 CPU 소모 문제를 해결하기 위해 DMA를 도입.

- __테스트 진행 중인 항목__
  - `4ms-115200bps/921600bps-DMA-Interrupt`
    - 동작 원리
      - DMA 컨트롤러가 CPU 대신 패킷을 메모리(버퍼)에 직접 옮겨주고, 패킷 1개(9바이트)가 완전히 도착했을 때 IDLE line의 HIGH를 감지해 단 한 번의 Interrupt로 CPU(Task)에 알려줌.
    - 장점
      - Interrupt를 통해 고속 통신 환경에서 상대적으로 응답성이 높은 동시에 바이트 수신 중 CPU를 과도하게 소모하지 않고 하드웨어 Overrun(ORE)이 발생하지 않을 것으로 예정.
  - `4ms-115200bps/921600bps-DMA-Polling-1ms/9ms`
    - 동작 원리
      - DMA가 백그라운드에서 패킷을 메모리(Circular 링 버퍼)에 Byte Stream으로 옮겨두고, CPU는 매번 깨어나지 않고 가끔씩(자신의 스케줄에 맞춰) 버퍼 인덱스를 확인해 쌓여 있는 데이터에 대해 Batch Processing을 진행. Polling의 주기를 1ms 또는 9ms로 설정하여 진행 중.
          - Polling의 주기를 1ms로 설정하면 4ms 주기 동안 4번의 polling 시도를 하며 그 중 1번 실제 데이터를 가져옴.
          - Polling의 주기를 9ms로 설정하면 버퍼에 2~3개의 패킷이 적재되어 있어 매 polling마다 데이터를 가져옴. 
    - __ADCP 핵심 알고리즘 구현__
      - __Producer-Consumer 패턴__: DMA(생산자)가 하드웨어 수준에서 CPU 간섭 없이 Stream 데이터를 링 버퍼에 적재하고, Task(소비자)는 정해진 주기에 깨어나 버퍼에 쌓인 만큼만 소비.
      - __Sliding Window__: read_idx를 1바이트씩 옮기며 노이즈를 거르고 유실 바이트를 감지하여 불량 패킷은 버리고 패킷 헤더를 추적하여 패킷 파싱을 진행.
      - __TCP 고안 바이트 스트림 신뢰성 적용__: 패킷 경계가 쪼개진 데이터들도 선형 탐지 과정을 통해 9바이트 패킷으로 재조립해 내는 무손실 아키텍처를 고안 및 적용.

### 4. 현실적인 Multi-Tasking 및 가변 실행시간/주기 환경 (문제 제기)

DMA 도입으로 통신상의 데이터 유실 문제는 해결했으나, 실제 임베디드 제어 시스템에서는 추가적인 동적 변수들이 존재.

- __다중 Task 우선순위 간섭__: 시스템 내에는 통신보다 우선순위가 훨씬 높은 핵심 제어 Task들이 존재하며, 상황에 따라 이들의 연산 부하(실행 시간)는 지속적으로 변동
- __고정 주기 처리의 자원 낭비__: 드론/로봇 환경에서는 상황에 따라 동적으로 데이터가 필요(충돌 위험이 낮을 때는 저빈도, 근접 시에는 고빈도). 4ms 주기로 들어오는 센서 데이터를 항상 즉각 처리하는 것은 불필요한 자원 낭비.
  - (현재 STM32F4보드에서 TF-Luna에게 주기 변경 명령어가 먹히지 않는 상황 발생... 현재 해결 중....)
- __실험의 필요성__: '제어 Task의 동적 부하'와 '가변적인 패킷 수신 주기'가 혼재하는 환경에서 DMA + Interrupt와 DMA + Polling 방식이 각각 시스템 안정성에 어떤 영향을 미치는지 대조 검증이 필요.

## 🎯 Future Plans (향후 계획)

현재의 DMA 파이프라인을 바탕으로, 실제 임베디드 Multi-Tasking 환경에서의 task들의 deadline을 지키기 위한 __적응형 제어 메커니즘__ 을 완성해 나갈 계획.

### 1. Multi-Tasking 및 가변 실행시간(고부하/저부하)/주기 환경 시나리오 실험
- 가설 검증 (SystemView 활용):
  - 여유 상황 (저부하): DMA + Interrupt와 DMA + Polling 모두 최상위 제어 Task에 지장을 주지 않고 동작하는지 확인.
  - 극한 상황 (고부하): 고우선순위 Task 실행 시간이 길어질 때, DMA + Interrupt가 강제 간섭하여 Deadline Miss 및 시스템 다운을 유발하는지 증명. 반면, DMA + Polling은 인터럽트 간섭 없이 안전하게 최상위 Task의 Deadline을 지켜내는지 대조 실험 진행.
- 실행 방법
  - 우선순위가 높은 Dummy Task 여러 개를 만들어 진행할 예정.

### 2. 적응형(Adaptive) 통신 파이프라인 구축 (최종 목표)
- 저부하 모드 (DMA + IDLE Interrupt):
  - 시스템 리소스에 여유가 있는 상황(일반 주행 등)에서는 패킷 도착 즉시 데이터를 처리하여 빠른 응답성(정밀한 거리 데이터)을 확보.
- 고부하 모드 (DMA + Polling):
  - 복잡한 회피 알고리즘 연산 등으로 CPU 부하가 심해지면, 능동적으로 UART 수신 인터럽트(IDLE 포함)를 차단.
  - DMA 링 버퍼의 크기를 넉넉하게 잡고 이를 활용해 통신 Task의 주기를 늦추고, 고우선순위 제어 작업이 끝난 후 남은 IDLE 시간에 데이터를 일괄 처리(Batch Processing)하여 CPU 오버헤드를 극단적으로 최소화.

__최종 지향점__: 고정된 단일 통신 방식에 얽매이지 않고, 시스템 부하와 외부 환경에 맞춰 필요한 순간에 통신 전략을 동적으로 전환하여 고우선순위 Task의 Deadline을 보장하는 무결성 시스템을 완성.

License
- This project is licensed under the MIT License - see the LICENSE file for details.

### v.2.0 아키텍처 리팩토링 및 검증

**1. AP/HW 계층 분리**
- AP/HW 계층을 분리하고 계층간의 의존성을 분리했습니다.
- 계층 분리의 동작을 검증하기 위한 메인 루프만을 테스트한 버전입니다.

**2. 한계점과 향후 계획**
- 현재 sw_tim 업데이트 이벤트의 ISR에서 메인루프의 지연으로 인해 전역변수가 유실 될 수 있다는 점을 확인했습니다.
- 전역변수의 한계를 느끼고 RTOS의 메세지 큐를 활용한 모듈간 통신을 검토중입니다.

---

### v.2.1 RTOS 아키텍처 사전 준비 및 STOPWATCH_TASK 추가
**1. RTOS task 실행 계층 추가**
- FreeRTOS Scheduler에 의해 실행되는 Task 코드를 scheduler 계층으로 분리했습니다.
- 해당 계층에 stopwatch_task.c를 추가하여 스톱워치 모듈의 실행 흐름을 관리하도록 구성했습니다.

**2.ST-LINK 동작 검증**
- ST-LINK로 분 단위 COUNT에 breakpoint를 걸고 동작을 검증했습니다.

---

### v.2.2 UART_CMD 파서 모듈 추가 및 검증
**1. UART_CMD_PROCESS 파서 모듈 추가**
- UART의 ISR에서 UART_CMD_TASK로 전송한 1byte data를 모듈 내부 정적 배열에 저장 한 뒤
- 명령문의 끝 지점에서 문자열을 비교해 CMD_STATE를 변경하는 모듈을 추가했습니다.
- 시스템이 시작 될때 정적 배열로 데이터를 넣어 CMD_STATE의 상태가 변경되는 것을 확인하여 모듈의 검증을 완료 했습니다.

- 

**2. UART_CMD_PROCESS 한계점**
- 현재 모듈의 구조는 UART_CMD_TASK가 명령 대상 채널을 미리 지정하고, 파서 모듈은 해당 채널의 명령어 문자열만 비교하는 방식입니다.
- UART의 ISR에서 전송되는 1byte data로는 문장이 완성 되기 전까지는 타겟을 지정하기 어렵다는 것을 파악했습니다.

**3. 향후 UART_CMD_PROCESS 리팩토링 계획**
- 향후 파서 모듈이 문자열 명령을 분석하여 target과 command를 포함한 결과를 반환하고
- UART_CMD_TASK는 해당 target을 기준으로 각 Task Queue에 명령을 전달하는 구조로 리팩토링할 계획입니다.
- 이를 통해 향후 UART_CMD로 제어할 디바이스를 추가하더라도 기존 로직에서 구조를 바꾸지 않고 확장이 가능 할 것입니다.

### v.2.3 UART_CMD_TASK 실행 모듈 추가 및 RX_ISR TO TASK 흐름 검증

**1. UART_driver UART_RX 통합 검증**

- 기존에 제작했던 UART_driver의 RX_ISR를 TASK와 Rtos_Queue로 연결했습니다.

- tera term에서 "SW_START" 명령어를 송신하고 ISR에서 UartRxQueueHandle를 통해 1BYTE DATA가 TASK로 전달 되는 것을 확인했습니다.

- 이를 통해 UART_RX_IT Rtos_queue UART_CMD_TASK UART_CMD_PARSER가 하나의 흐름으로 동작하는 것을 검증 했습니다.



**2. RTOS_Queue기반 UART_TASK 동작 검증**

- UART_CMD_TASK는 xQueueReceive의 portMAX_DELAY를 사용하여 UART_CMD가 RX될 때까지 TASK를 Blocked 상태로 대기하는 구조입니다.

- UART_RX_ISR에서 데이터가 전송되면 UART_CMD_TASK가 실행되어 CMD 파서를 동작 시킵니다.

- 문자열이 완성되고 명령이 테이블에 있다면 파서에서 파싱된 데이터와 타겟을 TASK로 전달하고,

- true를 반환해 타겟의 switch case문을 동작시킬 예정입니다.



**3. 문제 해결 과정**

- UART_RX_ISR과 TASK의 연결 흐름을 확인 할 때 ISR에 진입하지 않는 것을 확인했고, NVIC TABLE ENABLE을 통해 ISR의 진입을 확인했습니다.

- UART_RX_ISR 진입은 확인했으나 teraterm의 터미널로 data를 송신하면 보드가 뻗어버리는 문제가 발생했습니다

- HardFualt가 일어난 것으로 판단하고, RTOS_Queue의 생성 타이밍을 확인, 생성타이밍을 추가하여 Queue의 데이터 흐름을 검증 했습니다.



**4. RTOS_Queue적용을 통해 배운점**

- 기존에는 RTOS를 TASK를 실행하는 상위계층이라고 생각했습니다.

- 하지만 ISR에서 생성된 데이터를 TASK로 전달하고 파서에서 파싱된 데이터로 다른 TASK를 전달하는 구조를 생각하면서 단순 TASK의 실행 계층은 아니라는 점을 파악했습니다

- RTOS는 하드웨어 이벤트와 애플리케이션 로직을 연결하며 TASK실행을 주체하는 계층이라는 것 확인했습니다.



**5. 향후 계획**

- 현재는 UART RX ISR → `UART_CMD_TASK` → `uart_cmd_process()`까지의 데이터 흐름과 target 기준 switch-case 진입까지 검증했습니다.

- 아직 파싱된 command를 `STOPWATCH_TASK`로 전달하는 구조는 구현하지 않았습니다.

- Stop_watch_task로 전달된 명령이 stop_watch 모듈의 동작을 제어하는 것까지 데이터의 흐름을 이어볼 생각입니다. 



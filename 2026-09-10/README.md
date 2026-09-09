## 목표

HWD SoT Builder의 궁극적인 목표는 사람의 설계 의도를 `SoT(Source of Truth)`라는 기본 단위로 분해하고, 이 SoT로부터 설계 문서, RTL, 검증 항목 같은 아티팩트를 생성하는 것이다.

```text
사람의 설계 의도
      ↓
검토 가능한 SoT 세트
      ↓
문서 · RTL · 검증 아티팩트
```

AI가 곧바로 RTL을 작성하게 하는 대신, 먼저 사람이 읽고 검토하고 변경할 수 있는 설계 기준을 만든다. 요구사항이 바뀌면 SoT를 먼저 갱신하고, 영향을 받는 아티팩트를 다시 만든다.

## 1. 요구사항이나 설계 문서를 입력한다

새 프로젝트를 만든 뒤 요구사항이나 기존 설계 문서를 입력한다. 입력은 한 번에 모두 넣지 않아도 된다. 프로젝트를 진행하면서 요구사항, 보완 설명, 변경 요청을 계속 추가할 수 있다.

AI backend는 새 입력과 지금까지 확정된 Decision, SoT를 함께 분석한다.

- 원문에 이미 적힌 사실은 SoT로 정리한다.
- 사용자의 선택이 필요한 내용은 Decision 후보로 남긴다.
- 원문과 기존 SoT가 충돌하면 관련 Stage를 다시 검토한다.

### 예시 입력

입력 이벤트의 개수를 세는 8비트 counter를 설계한다.

Clock rising edge에서 event=1이면 count를 1 증가시킨다.
현재 count 값은 count[7:0]으로 출력한다.

Reset 방식, count가 255일 때 다음 event를 처리하는 방법,
overflow 알림 출력이 필요한지는 아직 정하지 않았다.

이 입력에서 다음은 이미 정해져 있다.

- 설계 대상: 입력 이벤트를 세는 counter
- Count 폭: 8비트
- 증가 조건: rising edge에서 event=1
- 출력: count[7:0]

다음은 아직 결정되지 않았다.

- Reset을 synchronous로 할지 asynchronous로 할지
- Reset polarity와 event가 동시에 들어올 때의 우선순위
- 255 다음에 0으로 돌아갈지, 255에서 멈출지
- Overflow 알림을 만들지, 만든다면 pulse인지 sticky인지

명시된 사실은 SoT로 옮기고, 미정 항목은 Decision Map에서 사용자가 결정한다.

## 2. REQ → ARCH → MOD → VER 순서로 진행한다

현재 구현은 네 Stage로 설계를 구체화한다.

```text
REQ → ARCH → MOD → VER
```

Stage는 사용자가 현재 어느 수준에서 작업하고 있는지 알려주는 길잡이다.

```text
REQ   무엇을 보장할지 결정
ARCH  어떤 구조와 정책으로 보장할지 결정
MOD   어떤 RTL module과 동작으로 구현할지 결정
VER   어떻게 검사하고 PASS/FAIL을 판정할지 결정
```

## 3. Decision Map을 상위부터 펼쳐 간다

Decision Map은 아직 확정되지 않은 설계 선택을 계층적으로 정리한 지도다. AI가 모든 질문을 한꺼번에 나열하지 않고 큰 영역부터 제안한다. 사용자가 구조를 승인하면 AI가 여러 하위 영역을 다시 펼친다. 이 과정을 반복해 실제 질문에 답할 수 있는 Decision Leaf에 도달한다.

### 1단계: AI가 상위 결정 영역을 제안한다

```text
8-bit event counter decisions
├─ Counter behavior
├─ Reset behavior
└─ Status reporting
```

사용자는 빠진 영역이나 잘못된 분류가 없는지 확인한다. 구조가 적절하면 승인하고, 아니면 수정이나 재생성을 요청한다.

### 2단계: 승인된 영역의 하위 Map을 펼친다

상위 구조가 승인되면 AI가 각 Branch 아래에 필요한 하위 결정을 제안한다.

```text
8-bit event counter decisions                    승인됨
├─ Counter behavior                              승인됨
│  ├─ Maximum-count behavior                     새 제안
│  └─ Reset/event simultaneous priority          새 제안
├─ Reset behavior                                승인됨
│  ├─ Reset timing                               새 제안
│  └─ Reset polarity                             새 제안
└─ Status reporting                              승인됨
   ├─ Overflow output                            새 제안
   └─ Overflow flag lifetime                     새 제안
```

여러 Branch가 함께 펼쳐질 수 있다. 서로 독립적인 결정은 sibling으로 나란히 두고 각각 승인한다. 한 결정의 답에 따라 필요한 다음 결정이 달라진다면 해당 Branch만 더 펼친다.

### 3단계: 여러 Decision Leaf가 만들어진다

더 나눌 필요가 없는 항목은 Leaf가 된다.

```text
8-bit event counter decisions
├─ Counter behavior
│  ├─ Maximum-count behavior              Leaf
│  └─ Reset/event priority                Leaf
├─ Reset behavior
│  ├─ Reset timing                        Leaf
│  └─ Reset polarity                      Leaf
└─ Status reporting
   └─ Overflow output
      ├─ Output required?                 Leaf
      └─ Pulse or sticky?                 Leaf
```

- `Branch`는 여러 하위 결정을 묶는 설계 영역이다.
- `Leaf`는 사용자가 독립적으로 확정할 수 있는 하나의 설계 결정이다.

## 4. Leaf의 질문 카드에 답한다

Leaf에 도달하면 AI가 구체적인 질문과 선택지를 제시한다.

```text
[Maximum-count behavior]
count가 255일 때 event가 들어오면 어떻게 처리할까요?

A. 0으로 돌아간다.             Wrap-around
B. 255를 유지한다.             Saturation

[Reset timing]
Reset을 언제 적용할까요?

A. Clock edge와 관계없이 적용한다.  Asynchronous reset
B. Clock rising edge에서 적용한다.   Synchronous reset

[Overflow output]
255에서 event가 들어왔음을 외부에 알릴까요?

A. 별도 출력 없음
B. 한 cycle pulse 출력
C. Software가 clear할 때까지 sticky 출력
```

예를 들어 사용자가 다음과 같이 선택했다고 가정한다.

- 255에서 saturation
- Synchronous active-low reset
- Reset이 event보다 우선
- Overflow는 한 cycle pulse

이 답은 단순한 대화 기록이 아니라 확정된 Decision으로 저장된다. AI backend는 원문과 이 Decision을 근거로 SoT를 만든다.

## 5. 각 Stage에서 정의하는 내용과 SoT 예시

아래 예시는 같은 8비트 event counter가 Stage를 지나며 어떻게 구체화되는지 보여준다.

### REQ: 외부에 보장해야 하는 동작과 제약

REQ는 구현 방법보다 사용자가 기대하는 동작을 정의한다.

REQ / Atomic
- Rising edge에서 event=1이면 count를 1 증가시켜야 한다.
- Count는 8비트 값으로 제공해야 한다.
- Count가 255이면 추가 event에도 255를 유지해야 한다.
- Overflow event가 발생하면 한 cycle 동안 알려야 한다.

### ARCH: 구조, 인터페이스와 설계 정책

ARCH는 REQ를 만족시키기 위한 상태, 데이터 흐름, 인터페이스와 정책을 정의한다.
```text
ARCH / Container: Counter state architecture
├─ ARCH / Atomic: 8비트 count state를 하나 둔다.
├─ ARCH / Atomic: event가 유효하고 count<255이면 next_count=count+1이다.
├─ ARCH / Atomic: count=255이면 next_count=255이다.
└─ ARCH / Atomic: overflow pulse는 255에서 event가 들어온 cycle에 생성한다.
```

현재 구현에서는 clock, reset, event, count, overflow 같은 인터페이스 계약도 ARCH에서 함께 다룬다.

### MOD: RTL module과 구체적인 구현 책임

MOD는 module definition, instance, port와 clock edge에서의 구체적인 상태 전이를 정의한다.

```text
MOD / Container: event_counter module
├─ MOD / Atomic: input  clk, reset_n, event
├─ MOD / Atomic: output count[7:0], overflow
├─ MOD / Atomic: reset_n=0인 rising edge에서 count와 overflow를 0으로 만든다.
├─ MOD / Atomic: event=1이고 count<255이면 count를 1 증가시킨다.
└─ MOD / Atomic: event=1이고 count=255이면 count를 유지하고 overflow를 1로 만든다.
```

### VER: 검증 조건, stimulus와 판정 기준

VER는 무엇을 확인할지만 적는 것이 아니라 테스트가 끝나고 PASS/FAIL을 판단할 수 있도록 조건과 expected result를 정의한다.

VER / Atomic: Reset test
- reset_n=0으로 rising edge를 한 번 발생시킨다.
- 다음 cycle에 count=0, overflow=0인지 확인한다.

VER / Atomic: Increment test
- count=0에서 event pulse를 세 번 입력한다.
- 출력이 1, 2, 3 순서로 변하는지 확인한다.

VER / Atomic: Saturation test
- count를 254까지 증가시킨다.
- event 한 번 후 count=255인지 확인한다.
- event를 한 번 더 넣어도 count=255인지 확인한다.
- 두 번째 event에서 overflow가 정확히 한 cycle만 1인지 확인한다.

상위 Stage의 계약은 하위 Stage의 근거가 된다.

```text
REQ saturation
   ↓
ARCH next-count policy
   ↓
MOD counter state transition
   ↓
VER 254→255와 255→255 검사
```

## 6. Stage와 Decision Map은 현재 위치를 보여준다

설계가 커지면 사용자는 AI가 무엇을 하는지보다 자신이 지금 어디에 있는지를 알아야 한다.

- Stage는 현재 다루는 설계 수준을 보여준다.
- Decision Map은 현재 결정 영역과 Leaf까지의 경로를 보여준다.
- 열린 sibling Leaf는 앞으로 답해야 할 다른 결정을 보여준다.
- Stage checkpoint는 이번 단계에서 확정된 Decision과 생성·변경된 SoT를 요약한다.

```text
현재 Stage: ARCH

현재 위치:
8-bit event counter decisions
→ Counter behavior
→ Maximum-count behavior

현재 질문:
255 다음 event를 wrap-around와 saturation 중 어떻게 처리할 것인가?

같은 Stage에 남은 결정:
- Reset timing
- Reset polarity
- Reset/event priority
- Overflow output
```

이 구조를 통해 여러 질문과 AI 작업이 반복되더라도 사용자가 방향을 잃지 않고 현재 위치, 완료한 결정, 남은 결정을 확인할 수 있다.

## 7. SoT Explorer에서 현재 설계를 확인한다

SoT도 Tree 구조를 가진다.

- `Container`는 의미 있는 설계 범위나 entity를 묶는다.
- `Atomic`은 독립적으로 검토하고 변경하고 검증할 수 있는 최소 설계 계약이다.
- Atomic 아래에는 child를 두지 않는다.
- Type이 다른 SoT는 `IMPLEMENTS`, VERIFIES 같은 relation으로 연결한다.

```text
MOD: event_counter                         Container
├─ Interface                              Container
│  ├─ Input ports                         Atomic
│  └─ Output ports                        Atomic
└─ Counter behavior                       Container
   ├─ Reset transition                    Atomic
   ├─ Increment transition                Atomic
   └─ Saturation transition               Atomic
```

SoT Explorer에서는 현재까지 생성된 canonical SoT를 Type별 Tree로 볼 수 있다. 각 SoT의 원문 Source, 연결된 Decision, relation, revision과 변경 이력도 확인할 수 있다.

## 8. VER까지 끝나면 하나의 SoT 세트가 완성된다

REQ, ARCH, MOD, VER가 모두 닫히면 현재 설계를 설명하는 canonical SoT 세트가 만들어진다.

```text
Canonical SoT Set
├─ 사람이 읽는 설계 문서
├─ RTL
├─ 검증 계획과 assertion 후보
└─ 테스트벤치와 실행 결과
```

`SoT Complete`는 설계 명세 workflow가 끝났다는 뜻이다. RTL이 이미 옳거나 모든 검증이 끝났다는 뜻은 아니다.

## 9. 검증 실패는 원인에 맞는 계층에서 고친다

### SoT는 충분하지만 생성된 RTL이 틀린 경우

```text
SoT: count는 255에서 saturation한다.
RTL: 255 다음에 0으로 돌아간다.
```

SoT를 약하게 만들지 않고 RTL 생성 또는 구현을 고친다.

### SoT가 부족하거나 서로 모순되는 경우

```text
REQ: Reset 후 count=0
ARCH: Reset timing이 결정되지 않음
```

관련 Stage와 Decision Map을 다시 열어 설계를 보완한 뒤 아티팩트를 다시 생성한다.

### 검증 방법이 잘못된 경우

테스트가 필요한 상태에 도달하지 못하거나 expected value가 틀렸다면 VER 계약과 테스트 방법을 고친다. DUT가 테스트에 맞도록 요구사항을 변경해서는 안 된다.

## 전체 흐름

```text
Source 입력
  ↓
명시된 사실을 SoT로 반영
  ↓
Decision Map의 여러 Branch를 상위부터 승인하며 전개
  ↓
여러 Leaf의 질문 카드에 답해 설계 선택 확정
  ↓
REQ → ARCH → MOD → VER SoT 생성
  ↓
SoT Explorer에서 현재 설계와 근거 확인
  ↓
완성된 SoT 세트에서 문서·RTL·검증 아티팩트 생성
  ↓
검증 실패 원인을 SoT·생성·검증 계층으로 나눠 수정
  ↓
요구 변경 시 영향받은 Stage를 다시 열고 반복
```

HWD SoT Builder가 만들려는 것은 한 번 생성하고 버리는 RTL이 아니다. 요구사항, 설계 결정, 구현과 검증이 함께 변경될 수 있는 추적 가능한 설계 기준이다.

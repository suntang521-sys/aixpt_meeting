# 탑 워크플로우와 Agentic Macro Loop

## 1. 탑 워크플로우

탑은 **코드**이며, AI 모듈의 호출 순서와 사람에게 제어를 돌려주는 시점을 결정한다.
REQ → ARCH → MOD → VER 순서로 진행하고, 각 stage에서 같은 흐름을 사용한다.

```text
[사람] 설계 요구 입력 / 분석 시작
   |
[AI: Requirements] 원문 분석·분류·검토
   |
[코드] 원문 + stage별 분석 목록 저장 (별도 분석 승인 없음)
   |
[사람] SoT 구성 시작
   |
   v
[코드: SELECT] 현재 stage의 다음 미완료 범위를 선택
   | 범위 있음
   v
[AI: SoT PREPARE] 현재 범위의 SoT 초안 작성·검토
   |
   +-- 선결 결정 필요 --> [AI: Decision GENERATE] 질문 카드 생성
   |                           |
   |                       [사람] 선택 / 직접 답변 / 답변 코멘트
   |                           |
   |                       [AI: Decision INTERPRET] 답변 해석
   |                           +-- 모호함 --> 남은 부분만 재질문
   |                           +-- 명확함 --> 같은 범위 PREPARE
   |
   +-- 선결 결정 없음
          |
      [사람] SoT + 답변 원문/해석/규칙 반영 관계 확인
          +-- 제안 코멘트 --> 같은 범위 PREPARE
          +-- 답변 정정 --> INTERPRET
          +-- 제안 승인
                 |
             [코드] 승인된 묶음 저장 -> SELECT로 복귀

SELECT에서 남은 범위가 없으면:
   [코드] Close 사전 검사
      -> [AI: SoT REVIEW_STAGE] 승인된 stage 전체 읽기 전용 검토
           +-- 미완료 --> 지원되는 범위 재개 또는 NEEDS_WORK
           +-- 준비됨 --> [사람] Close 승인 / 검토 코멘트
                              +-- 코멘트 --> REVIEW_STAGE
                              +-- 승인 --> [코드] CLOSED 저장
                                               |
                                    [사람] 다음 stage 시작
                                               |
                                    [코드] 다음 stage 준비
                                               |
                                    [사람] SoT 구성 시작 -> SELECT
```

- 범위는 승인된 계층 순서로 선택한다. `ROOT`는 상위 구성, `EXPAND`는 하위 확장, `REFINE`은 허용된 범위 보완이다.
- 질문은 정식 제안 전에 한다. 답변 해석만 별도로 재승인하지 않고, SoT 제안과 함께 승인한다.
- 질문 자체의 수정 코멘트는 `GENERATE`로 돌아간다. 이미 부분 답변한 카드의 재생성에는 제한이 있다.
- 단계 검토에서 발견한 순수 참조 누락은 **코드의 보완 제안 → 사람 승인 → 새 단계 검토**로 처리할 수 있다.
- 승인된 계약의 의미 변경은 `NEEDS_IMPACT`, 기술 오류·한도 소진은 중지로 반환한다. 지원되는 재시도는 사람의 명시적 요청으로 수행한다.

## 2. Agentic Macro Loop

```text
Context Build -> Generate -> Validate -- 성공 --> 결과 반환
                               |
                              실패
                               |
                            Diagnose
                               |
                       Repair / Regenerate
                               |
                               +----------> Validate

필요한 권한·근거 부족 또는 안전하게 계속할 수 없음 -> 진단 반환 / 중지
```

공통 실행기 `runMacro()`에 작업별 단계를 주입한다. 실행 순서는 공통이고, 생성·검토 기준은 작업별로 다르다.
AI 검토자는 작성자의 추론·대화 이력을 받지 않는다. `read_records`로 필요한 근거를 추가 조회할 수 있으며,
툴 호출·수정도 동일 작업의 시간·토큰·횟수 한도를 공유한다.

수정 후에는 다시 검증하며, 초안이 바뀌면 이전 검토를 재사용하지 않는다.
정상적인 추가 질문이나 `ready: false`는 유효한 결과다. Macro 성공은 사람의 승인이나 stage Close를 뜻하지 않는다.

## 3. 모듈별 Agentic Macro Loop 구성 테이블

각 표의 **행은 실행 단계**, **열은 코드·AI·Skill·Tool의 역할**이다. 아래는 제안이 아니라 현재 구현이다.

- **코드**: 컨텍스트 조립, 형식 검사, 분기, 저장 등 프로그램이 수행하는 일.
- **AI**: 실제 LLM 호출로 수행하는 일. `—`는 해당 단계에 AI 호출이 없다는 뜻.
- **Skill**: AI에게 주입하는 작업 지침 파일. 별도로 실행되는 에이전트나 Tool이 아니다. Context Build에서는 코드가 삽입하고, AI 호출 시 적용한다.
- **Tool**: AI가 호출할 수 있는 읽기 전용 `read_records`. **필요 시**는 근거가 부족하거나 재확인이 필요할 때 호출 가능하다는 뜻이며, 매번 호출하지는 않는다. 일반 코드 함수는 Tool로 표시하지 않는다.
- **완료 반환** 행은 성공 이후의 코드 처리이며, 별도의 AI 작업이 아니다.

### 3.1 Requirements — 요구사항 분석

입력: 사용자 원문·제공된 근거 → 출력: 근거·stage가 연결된 분석 목록 또는 진단.

실제로는 **추출 Macro → 검토 Macro**로 구성된다.
짧은 입력은 전체를, 긴 입력은 절별로 추출·검토한 뒤 최종 CROSS 검토를 수행한다.

**A. 추출 Macro**

| 단계 | 코드 역할 | AI 역할 | Skill | Tool | 입력 → 출력 |
| --- | --- | --- | --- | --- | --- |
| Context Build | 원문 블록·절·근거 인덱스와 작성 컨텍스트 조립 | — | `skill.md` 삽입 | — | 원문 → 작성 컨텍스트 |
| Generate | 호출·응답 기록 | 의미 추출·분류 | `skill.md` | `read_records`: 필요 시 | 컨텍스트 → 분석 후보 JSON |
| Validate | 형식·근거·절 범위 검사; 필요한 미전달 근거 보충 | 근거 보충 시에만 재추출 | 재추출 시 `skill.md` | 재추출 시 사용 가능 | 후보 → 통과 또는 오류 |
| Diagnose | 오류를 LOCAL 수정 또는 중지로 분기 | — | — | — | 오류 → 수정 지시 |
| Repair / Regenerate | 오류 피드백 조립 | 잘못된 추출 결과 수정 | `skill.md` | `read_records`: 필요 시 | 오류·후보 → 수정 후보 → Validate |
| 완료 반환 | 후보를 검토 Macro로 전달 | — | — | — | 추출 통과 후보 → 검토 대상 |

**B. 검토 Macro — 전체 / 절별 / CROSS**

| 단계 | 코드 역할 | AI 역할 | Skill | Tool | 입력 → 출력 |
| --- | --- | --- | --- | --- | --- |
| Context Build | 작성 대화와 분리된 검토 컨텍스트 조립 | — | `review.md` 또는 `cross-review.md` 삽입 | — | 후보·근거 → 검토 컨텍스트 |
| Generate | 현재 후보를 검토 대상으로 고정; 새 후보 생성 없음 | — | — | — | 현재 후보 → 고정된 검토 대상 |
| Validate | 검토 JSON·근거·PATCH 대상 검사 | 의미·누락·분류 검토; 필요 시 수정 PATCH 제안 | `review.md` / CROSS는 `cross-review.md` | `read_records`: 필요 시 | 후보 → 통과 / PATCH / 진단 |
| Diagnose | 유효 PATCH는 수정으로, 잘못된 검토 출력은 중지로 분기 | — | — | — | 검토 결과 → 수정 또는 중지 |
| Repair / Regenerate | 검증된 PATCH 적용; 영향받은 절의 검토 무효화 | — | — | — | PATCH·후보 → 수정 후보 → Validate |
| 완료 반환 | 필요한 절별·CROSS 검토가 모두 끝나면 `ACCEPTED` 반환 | — | — | — | 검토 통과 목록 → 탑 |

공유 한도: LOCAL 2회·SEMANTIC 2회·CROSS 2회, 총 6회. 절·페이지마다 한도를 새로 받지 않는다.
CROSS 수정이 절에 영향을 주면 해당 절을 다시 검토한 뒤 CROSS로 돌아간다.

### 3.2 SoT — PREPARE (ROOT / EXPAND / REFINE)

입력: 현재 범위·분석 목록·관련 SoT·근거·잠정 답변·코멘트 → 출력: 검토된 SoT 제안 또는 추가 입력 진단.

| 단계 | 코드 역할 | AI 역할 | Skill | Tool | 입력 → 출력 |
| --- | --- | --- | --- | --- | --- |
| Context Build | 편집 범위·관련 책임·근거·답변을 묶음 | — | stage별 작성 Skill 삽입 | — | 범위·근거 → 작성 컨텍스트 |
| Generate | 호출·응답 기록 | 상위 구성 / 하위 확장 / 범위 보완 초안 작성 | stage별 작성 Skill | `read_records`: 필요 시 | 컨텍스트 → SoT 초안 JSON |
| Validate | 형식·범위·참조·규칙 보존 검사; 독립 검토 컨텍스트 구성; 검토 결과 검사 | 의미·책임·커버리지 검토 | stage별 검토 Skill | `read_records`: 필요 시 | 초안 → 통과 또는 오류·수정 의견 |
| Diagnose | 초안 오류 / 검토 오류와 LOCAL / SEMANTIC 구분 | — | — | — | 실패 → 수정 대상·피드백 |
| Repair / Regenerate | 피드백 조립; MOD/VER 미승인 ROOT의 제한된 참조 메타데이터는 코드 보완 가능 | 코드 보완 대상이 아니면 초안 또는 검토 출력 수정 | 수정 대상의 작성 / 검토 Skill | AI 수정 시 `read_records`: 필요 시 | 실패 결과 → 수정 결과 → Validate |
| 완료 반환 | 제안을 입력·범위에 연결해 `REVIEWED` 반환; 정본 저장은 하지 않음 | — | — | — | 검토 통과 제안 → 탑의 질문 / 사람 승인 |

공유 한도: LOCAL 2회·SEMANTIC 2회. **AI 수정이든 코드 보완이든 초안이 바뀌면 새 독립 검토**를 받는다.

### 3.3 SoT — REVIEW_STAGE (단계 전체 검토)

입력: 승인된 stage 전체·원문·답변·상위 stage 근거·Close 기준 → 출력: `STAGE_REVIEW(ready, checks)` 또는 진단.

| 단계 | 코드 역할 | AI 역할 | Skill | Tool | 입력 → 출력 |
| --- | --- | --- | --- | --- | --- |
| Context Build | 승인된 stage와 검토용 근거 묶음 준비 | — | 검토 Skill은 Validate의 컨텍스트 구성 때 삽입 | — | 승인 상태 → 검토용 묶음 |
| Generate | 승인된 상태를 변경 없는 검토 스냅샷으로 구성 | — | — | — | 승인 상태 → 읽기 전용 스냅샷 |
| Validate | 독립 검토 컨텍스트 구성; 검토 근거·연결·Close 기준 검사 | 단계 전체 완결성·일관성 검토 | stage별 검토 Skill | `read_records`: 필요 시 | 스냅샷 → 준비됨 / 미완료 / 오류 |
| Diagnose | 잘못된 검토 출력만 교정 대상으로 분류 | — | — | — | 검토 오류 → 교정 지시 |
| Repair / Regenerate | 오류 피드백 조립; 승인된 SoT는 유지 | 잘못된 검토 출력 수정 | stage별 검토 Skill | `read_records`: 필요 시 | 검토 오류 → 수정 검토 → Validate |
| 완료 반환 | `ready`·검사 결과를 탑에 전달 | — | — | — | 검토 결과 → Close 승인 / 미완료 처리 |

LOCAL 교정은 2회. 유효한 `ready: false`는 정상 출력이며, SoT를 자동 수정하거나 `true`가 될 때까지 반복하지 않는다.

### 3.4 Decision — GENERATE (질문 카드 생성)

입력: 현재 초안·소유 범위가 정해진 미정 사항·근거·질문 코멘트 → 출력: 검토된 `CARD_SET` 또는 진단.

| 단계 | 코드 역할 | AI 역할 | Skill | Tool | 입력 → 출력 |
| --- | --- | --- | --- | --- | --- |
| Context Build | 질문 대상·기존 조건·근거 묶음 구성 | — | 질문 생성 Skill 삽입 | — | 미정 사항·근거 → 생성 컨텍스트 |
| Generate | 호출·응답 기록 | 질문·선택지·선택적 추천 생성 | `generate.md` 계열 | `read_records`: 필요 시 | 컨텍스트 → 카드 후보 JSON |
| Validate | 형식·소유 범위·근거 검사; 독립 검토 컨텍스트 구성; 검토 출력 검사 | 질문의 충실성·답변 가능성 검토 | `generate-review.md` 계열 | `read_records`: 필요 시 | 카드 후보 → 통과 또는 수정 의견 |
| Diagnose | 카드 오류 / 검토 오류와 LOCAL / SEMANTIC 구분 | — | — | — | 실패 → 수정 대상·피드백 |
| Repair / Regenerate | 해당 역할의 피드백·컨텍스트 구성 | 카드 또는 검토 출력 수정 | 해당 역할의 생성 / 검토 Skill | `read_records`: 필요 시 | 실패 결과 → 수정 결과 → Validate |
| 완료 반환 | 카드 세트를 입력·버전에 연결해 반환 | — | — | — | `CARD_SET` → 탑 → 사람의 답변 |

공유 한도: LOCAL 2회·SEMANTIC 2회. 카드가 바뀌면 새 독립 검토를 받는다.

### 3.5 Decision — INTERPRET (답변 해석)

입력: 정확한 카드 버전·실제 선택/답변/코멘트·이전 답변·관련 근거 → 출력: 검토된 `ANSWER_SET` 또는 진단.

| 단계 | 코드 역할 | AI 역할 | Skill | Tool | 입력 → 출력 |
| --- | --- | --- | --- | --- | --- |
| Context Build | 원문 답변·선택지·기존 제약을 묶음 | — | 답변 해석 Skill 삽입 | — | 카드·답변 → 해석 컨텍스트 |
| Generate | 호출·응답 기록 | 답변 의미·남은 모호함·승인 계약 변경 여부 분석 | `interpret.md` 계열 | `read_records`: 필요 시 | 컨텍스트 → 해석 후보 JSON |
| Validate | 형식·버전·근거 검사; 독립 검토 컨텍스트 구성; 검토 출력 검사 | 실제 답변 충실성·모호함·제약 보존 검토 | `interpret-review.md` 계열 | `read_records`: 필요 시 | 해석 후보 → 통과 또는 수정 의견 |
| Diagnose | 해석 오류 / 검토 오류와 LOCAL / SEMANTIC 구분 | — | — | — | 실패 → 수정 대상·피드백 |
| Repair / Regenerate | 해당 역할의 피드백·컨텍스트 구성 | 해석 또는 검토 출력 수정 | 해당 역할의 해석 / 검토 Skill | `read_records`: 필요 시 | 실패 결과 → 수정 결과 → Validate |
| 완료 반환 | 해석 결과를 입력·카드 버전에 연결해 반환 | — | — | — | `ANSWER_SET` → SoT 준비 / 재질문 / Impact 진단 |

공유 한도: LOCAL 2회·SEMANTIC 2회. 해석이 바뀌면 새 독립 검토를 받는다.
`NEEDS_CLARIFICATION`·`CHANGE_REQUEST`는 유효한 결과이며, 사람의 추가 답변은 Macro 밖의 탑에서 받는다.

**Skill 파일 선택 규칙**

- Requirements: 해당 모듈의 `skill.md`, `review.md`, `cross-review.md`.
- SoT: REQ는 `skill.md` / `review.md`; ARCH·MOD·VER는 각각 `arch-`·`mod-`·`ver-` 접두사를 붙인다.
- Decision: REQ는 표의 파일명 그대로 사용하며, ARCH·MOD·VER는 같은 접두사를 붙인다.

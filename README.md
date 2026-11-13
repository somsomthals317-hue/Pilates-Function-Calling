# 🚀 필라테스 상담 챗봇 Function Calling

> 6가지 핵심 기능을 수행하는 Function Calling 기반 상담 챗봇 개발 및 Qwen LoRA Fine-Tuning을 통한 함수 선택·파라미터 추출 성능 개선

---

## 📌 1. 프로젝트 개요 (Overview)

- **개발 기간:** 2026.11.04 ~ 2026.11.12
- **프로젝트 유형:** 개인 프로젝트
- **개발 목적:** 자연어를 통해 필라테스 이용권, 수업 시간표, 예약 내역, 운영 정책 등을 조회하는 Function Calling 상담 챗봇 구현
- **주요 성과:**
  - 필라테스 센터 운영을 위한 관계형 DB 8개 테이블 및 핵심 함수 6개 설계
  - Multi-Turn 대화 데이터 약 80개, 총 412개 턴 구축
  - Qwen2.5-7B-Instruct 기반 LoRA Fine-Tuning 적용
  - 함수 선택 정확도 56.57% → 85.00% (+28.33%p)
  - 파라미터 값 정확도 47.42% → 73.40% (+25.98%p)
  - Exact Match 31.25% → 66.67% (+35.42%p)
  - Multi-Call Exact Match 8.33% → 75.00% (+66.67%p)

실제 필라테스 운동 예약 앱인 '바디코디'를 사용하며 경험한 예약 과정을 바탕으로, 자연어로 센터 정보를 조회할 수 있는 상담 챗봇을 기획했습니다.

기존 조회형 챗봇을 넘어 **사용자 요청에 적합한 함수를 선택하고 필요한 파라미터를 추출하는 Function Calling 능력**을 개선하는 것을 목표로 했습니다.

---

## 🛠️ 2. 기술 스택 (Tech Stack)

- **Language:** Python
- **LLM:** Qwen2.5-7B-Instruct, GPT-4o
- **Fine-Tuning:** LoRA, PEFT, SFT
- **Database:** 관계형 DB, SQL
- **Data Generation:** OpenAI API, Function Calling Prompt
- **Evaluation:** Tool Decision, Tool Selection, Parameter Accuracy, Exact Match, Multi-Call Exact Match

---

## 🏗️ 3. 시스템 아키텍처 (Architecture)

### 전체 파이프라인

**Domain & Function Design → DB 구축 → Function Calling 대화 데이터 생성 → LoRA Fine-Tuning → 함수 호출 성능 평가**

### ① Domain & Function Design

실제 필라테스 센터의 운영 환경을 가정하여 총 8개의 관계형 DB 테이블을 구성했습니다.

- `users`: 사용자 정보
- `memberships`: 이용권 정보
- `user_memberships`: 사용자별 이용권 정보
- `instructors`: 강사 정보
- `classes`: 수업 시간표
- `bookings`: 예약 내역
- `attendances`: 출석 정보
- `policies`: 센터 운영 정책

PK, FK를 기반으로 테이블 간 관계를 정의하고, 사용자별 이용권·예약·출석 정보를 연결했습니다.

이후 DB 조회를 위한 6가지 Function Calling 함수를 설계했습니다.

| 구분 | Function | 주요 기능 |
|---|---|---|
| 기본 조회 | `get_membership_info()` | 이용권 정보 조회 |
| 기본 조회 | `get_class_schedule()` | 수업 시간표 조회 |
| 예약 가능 여부 | `get_available_classes()` | 예약 가능한 수업 조회 |
| 유저 정보 | `get_my_memberships()` | 사용자 이용권 조회 |
| 유저 정보 | `get_my_bookings()` | 사용자 예약 내역 조회 |
| 운영 정책 | `search_policy_info()` | 센터 운영 정책 검색 |

### ② Function Calling Dialogue Dataset Generation

Qwen의 Function Calling Prompt Template을 기반으로 Multi-Turn 대화 데이터를 생성했습니다.

**Available Scenarios**
- 이용권 상담 비교
- 특정 강사의 시간표 조회
- 여러 요일의 수업 시간표 조회
- 예약 가능 여부 확인
- 복수 정보 조회를 위한 Multi-Call 구성

**Unavailable Scenarios**
- 예약 생성 및 취소
- 결제 및 환불 처리
- 출석 등록
- 이외 구현하지 않은 업무 요청

**데이터 생성 과정**
- GPT-4o를 활용하여 5~6턴의 Multi-Turn 대화 생성
- `TOOL_CALL`, `CLARIFY`, `DIRECT_ANSWER`, `FOLLOW_UP`의 행동 유형 반영
- 사용자 요청에 따라 함수 호출 또는 추가 질문이 필요한 상황 구성
- Available / Unavailable 시나리오를 혼합하여 다양한 상담 상황 반영

**최종 학습 데이터: 약 80개 대화, 총 412개 턴**

### ③ Qwen LoRA Fine-Tuning

- **Base Model:** Qwen2.5-7B-Instruct
- **Fine-Tuning:** LoRA + SFT
- **Target Modules:** Attention + MLP
- **LoRA Rank (r):** 16
- **LoRA Alpha:** 32
- **Epoch:** 3

사용자 요청에 따라 적절한 함수 이름과 파라미터를 출력하도록 학습하고, Multi-Turn 환경에서의 함수 호출 패턴을 반영했습니다.

### ④ Function Calling Evaluation

Base 모델과 Fine-Tuned 모델의 함수 호출 결과를 동일한 평가 데이터에서 비교했습니다.

**평가 과정**

1. 총 130개의 Assistant 응답을 대상으로 평가
2. Base 모델 및 Fine-Tuned 모델의 응답 생성
3. 생성된 `<tool_call>` 구간을 파싱하여 함수 이름 및 파라미터 추출
4. Ground Truth와 비교하여 함수 호출 정확도 평가

---

## ✨ 4. 주요 기능 및 본인 역할 (Key Features & Contributions)

### ① 필라테스 도메인 DB 및 함수 설계

- 센터 운영 환경을 반영한 8개 관계형 DB 테이블 구성
- 사용자별 이용권, 예약, 출석 내역을 연결하는 데이터 구조 설계
- 6가지 Function Calling 함수 및 입출력 스키마 정의
- 자연어 질문을 DB 조회와 연결하는 함수 실행 구조 구현

### ② Multi-Turn Function Calling 데이터 구축

- Qwen Function Calling 형식에 맞는 학습 데이터 생성
- 단일 함수 호출뿐 아니라 여러 함수가 필요한 Multi-Call 시나리오 구성
- 정보 부족 시 추가 질문을 수행하는 `CLARIFY` 유형 반영
- 지원하지 않는 요청에 대한 행동 경계 학습 데이터 구성

### ③ LoRA 기반 Function Calling 성능 개선

- Qwen2.5-7B-Instruct 모델에 LoRA Fine-Tuning 적용
- Attention 및 MLP 모듈을 학습 대상으로 설정
- 함수 선택 및 파라미터 추출 성능 비교 평가

**Base Model vs Fine-Tuned Model**

| Evaluation Metric | Base | Fine-Tuned | Improvement |
|---|---:|---:|---:|
| Tool Decision | 85.38% | 91.54% | +6.16%p |
| Tool Selection | 56.57% | 85.00% | +28.33%p |
| Params Selection | 51.55% | 74.47% | +22.92%p |
| Params Value | 47.42% | 73.40% | +25.98%p |
| Exact Match | 31.25% | 66.67% | +35.42%p |
| Multi-Call Exact Match | 8.33% | 75.00% | +66.67%p |

**평가 지표**
- **Tool Decision:** 함수 호출 여부 및 호출하지 않아야 하는 상황 판단
- **Tool Selection:** 정답 함수 이름 일치 여부
- **Params Selection:** 파라미터 Key 일치 여부
- **Params Value:** 파라미터 Value 일치 여부
- **Exact Match:** 함수 이름, Key, Value 전체 일치 여부
- **Multi-Call Exact Match:** 복수 함수 호출의 이름과 파라미터 일치 여부

### ④ 자연어 기반 복합 정보 조회

- 사용자 질문 한 번으로 이용권, 수업 시간표, 예약 잔여석 등 여러 정보 조회
- 여러 함수 호출 결과를 연결하여 하나의 응답 생성
- 정형화된 함수 호출을 통해 자연어 요청을 DB 조회 작업으로 연결

---

## 💡 5. 배운 점 (Troubleshooting)

### ① 함수 호출 정확도는 함수 선택뿐 아니라 파라미터 추출까지 평가해야 한다는 점

Function Calling은 적절한 함수 이름을 선택하더라도 파라미터가 잘못 추출되면 의도한 결과를 얻기 어렵습니다.

이에 따라 Tool Decision, Tool Selection, Params Selection, Params Value, Exact Match를 구분하여 평가했습니다.

특히 Exact Match가 31.25%에서 66.67%로 개선된 결과를 통해, **함수 이름과 파라미터를 함께 평가하는 것의 중요성**을 확인했습니다.

### ② Multi-Turn 환경에서는 행동 판단을 학습 데이터에 반영해야 한다는 점

실제 상담 과정에서는 모든 사용자 질문에 함수를 호출할 수 있는 것은 아닙니다.

사용자 정보가 부족하거나 지원하지 않는 업무를 요청하는 경우를 고려하여 `TOOL_CALL`, `CLARIFY`, `DIRECT_ANSWER`, `FOLLOW_UP` 유형을 구분했습니다.

이를 통해 Function Calling 학습에서는 함수 호출 형식뿐 아니라 **어떤 상황에서 호출하고, 추가 질문하거나 직접 답변해야 하는지에 대한 판단 기준**도 중요하다는 점을 학습했습니다.

### ③ 다양한 함수 호출 시나리오를 학습 데이터로 구성하는 것의 중요성

GPT-4o를 활용한 Multi-Turn 데이터 생성 과정에서 Function Calling Schema 불일치 오류가 발생했습니다.

또한 API 호출 비용과 생성 횟수 제한으로 충분한 학습 데이터를 확보하지 못해, 다양한 사용자 표현과 복합·예외 시나리오를 반영하는 데 한계가 있었습니다.

이러한 경험을 통해 학습 데이터의 양뿐 아니라 **함수 스키마의 일관성, 시나리오 다양성, 생성 결과 검증**이 중요하다는 점을 확인했습니다.

### ④ 실제 서비스 적용을 위한 추가 개선 방향

이번 프로젝트는 Function Calling Fine-Tuning의 효과를 검증하는 프로토타입으로 구현했습니다.

실제 서비스에 적용하기 위해서는 다음과 같은 개선이 필요합니다.

- **데이터 다양성 확대:** 다양한 사용자 표현과 예외 상황을 반영한 학습 데이터 추가
- **함수 실행 오류 처리:** 잘못된 파라미터, 조회 실패 등 함수 실행 단계의 오류 대응
- **대화 상태 관리:** 반복적인 질문과 연속적인 요청에서 사용자 맥락 유지
- **모델 배포 환경 구축:** 안정적인 추론 및 DB 연결을 위한 서비스 환경 개선

---

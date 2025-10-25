# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

OpenAI Agents SDK를 사용하여 구축된 다중 에이전트 고객 지원 시스템입니다. Streamlit 기반 음성 인터페이스를 통해 기술 지원, 청구, 주문 관리, 계정 관리 등 다양한 도메인으로 대화를 라우팅하는 triage 시스템을 사용합니다.

## 개발 환경 설정

```bash
# 의존성 설치 (uv 패키지 매니저 사용)
uv pip install -e .

# Streamlit 애플리케이션 실행
streamlit run main.py

# Python 버전: 3.13 이상 필요
```

## 핵심 아키텍처

### 다중 에이전트 시스템

프로젝트는 **에이전트 핸드오프(handoff) 패턴**을 사용하여 특수 에이전트들 사이에서 대화를 라우팅합니다:

1. **Triage Agent** (`my_agents/triage_agent.py`): 진입점 에이전트로 고객 문의를 분류하고 전문 에이전트로 핸드오프
2. **Technical Support Agent** (`my_agents/technical_agent.py`): 제품 문제 및 기술 문제 해결
3. **Billing Agent** (`my_agents/billing_agent.py`): 결제, 환불, 구독 처리
4. **Order Agent** (`my_agents/order_agent.py`): 배송, 반품, 주문 추적
5. **Account Agent** (`my_agents/account_agent.py`): 로그인, 비밀번호 재설정, 계정 설정

### 핸드오프 메커니즘

핸드오프는 `triage_agent.py`에서 정의되며 다음과 같이 작동합니다:

- `make_handoff()` 함수는 `HandoffData` 입력 타입으로 구조화된 핸드오프를 생성
- `handoff_filters.remove_all_tools`는 핸드오프 후 대상 에이전트가 이전 도구에 접근하지 못하도록 보장
- `handle_handoff()` 콜백은 핸드오프 정보를 Streamlit 사이드바에 표시
- 에이전트는 handoffs 리스트의 `make_handoff(agent)` 항목을 통해 연결됨

### Guardrails 시스템

프로젝트는 입력 및 출력 guardrails를 구현합니다:

**입력 Guardrails** (`triage_agent.py`):
- `off_topic_guardrail`: 요청이 고객 지원 도메인(계정, 청구, 주문, 기술)과 관련이 있는지 확인
- `InputGuardrailTripwireTriggered` 예외를 발생시켜 허용되지 않는 요청 차단

**출력 Guardrails** (`output_guardrails.py`):
- `technical_output_guardrail`: 기술 에이전트 응답에 청구/주문/계정 정보가 부적절하게 포함되지 않도록 보장
- 전용 guardrail 에이전트를 사용하여 응답을 분석하고 위반 시 tripwire 트리거

### Context 시스템

모든 에이전트는 `UserAccountContext`(`models.py`)를 공유합니다:
- 고객 정보 포함: `customer_id`, `name`, `tier`, `email`
- `RunContextWrapper`를 통해 전달되어 대화 전반에 걸쳐 사용자 컨텍스트 유지
- 동적 instructions 함수가 이 컨텍스트를 사용하여 고객별 행동 맞춤화

### 도구(Tools) 구조

모든 도구는 `tools.py`에 정의되며 `@function_tool` 데코레이터 사용:
- 모든 도구는 첫 번째 매개변수로 `context: UserAccountContext`를 받음
- 도구는 도메인별로 그룹화됨: Technical, Billing, Order, Account
- `AgentToolUsageLoggingHooks` 클래스는 Streamlit 사이드바에 도구 사용 로깅

### Session 관리

- SQLite 기반 세션 저장소 (`SQLiteSession`) 사용
- 대화 히스토리를 `customer-support-memory.db`에 저장
- Streamlit의 `session_state`에서 세션 상태 유지
- 사이드바의 "Reset memory" 버튼으로 세션 지우기

### 음성 입력 처리

Streamlit의 `audio_input` 위젯 사용:
- WAV 오디오를 numpy 배열로 변환 (`convert_audio` 함수)
- `AudioInput` 버퍼로 변환 후 에이전트로 전달
- 현재 `main.py`의 `run_agent` 함수가 불완전함 - 아직 음성 처리가 완전히 구현되지 않음

## 에이전트 수정 시

### 새 에이전트 추가
1. `my_agents/`에 새 에이전트 파일 생성
2. 동적 instructions 함수 정의 (`RunContextWrapper`를 매개변수로 받음)
3. `Agent` 인스턴스 생성하고 도구, guardrails, hooks 할당
4. `triage_agent.py`의 handoffs 리스트에 추가
5. Triage Agent instructions를 업데이트하여 새 도메인 포함

### 새 도구 추가
1. `tools.py`에 `@function_tool` 데코레이터로 함수 정의
2. 첫 번째 매개변수는 항상 `context: UserAccountContext`
3. 적절한 에이전트의 tools 리스트에 추가
4. 에이전트 instructions를 업데이트하여 도구 사용 시기 설명

### 동적 Instructions 패턴

모든 전문 에이전트는 동적 instructions 함수를 사용합니다:
```python
def dynamic_agent_instructions(
    wrapper: RunContextWrapper[UserAccountContext],
    agent: Agent[UserAccountContext],
):
    return f"""
    고객 {wrapper.context.name}을 위한 instructions...
    Tier: {wrapper.context.tier}
    """
```

이 패턴을 통해 고객 컨텍스트를 기반으로 런타임에 맞춤화된 instructions 제공.

## 데이터베이스 파일

- `customer-support-memory.db`: SQLiteSession 대화 히스토리
- `ai-memory.db`, `chat-gpt-clone-memory.db`: 레거시 또는 실험용 세션 파일
- 이러한 파일들은 `.gitignore`에 포함되어야 함

## 중요한 의존성

- `openai-agents[voice]`: 코어 에이전트 프레임워크
- `streamlit`: UI 프레임워크
- `python-dotenv`: 환경 변수 관리 (.env에서 OpenAI API 키 로드)
- `sounddevice`, `numpy`: 음성 처리용

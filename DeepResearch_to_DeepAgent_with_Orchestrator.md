## DeepResearch → DeepAgent (Orchestrator 패턴) 전환 설계 문서

본 문서는 Day-05/DeepResearch를 Day-03/orchestrator.py 스타일의 "Orchestrator + SubAgent" 구조로 리팩터링하기 위한 설계와 구현 포인트를 제시합니다.

---

### 1) 설계 개요 및 설계도

#### 목표
- 다중 서브그래프(Supervisor/Researcher/Compressor) 중심의 복잡한 워크플로우를 **Orchestrator + SubAgent** 구조로 단순화.
- 각 연구 단계(명확화, 계획, 조사, 압축, 보고서)를 독립적인 SubAgent로 모듈화하여 유지보수성과 확장성 향상.
- DeepAgent의 계층적 위임(Hierarchical Delegation) 패턴을 활용하여 복잡한 연구를 관리 가능한 단위로 분해.

#### 핵심 아이디어
- **Orchestrator Agent**
  - 프로젝트 매니저 역할: 사용자 요청 분석 → 작업 계획 수립 → SubAgent 위임 → 결과 취합 → 최종 보고서 생성
  - `task` 도구를 통해 SubAgent에게 작업 위임
  - SubAgent들의 실행 순서와 병렬성 제어

- **SubAgent 구조** (각각 독립된 스킬/전문가)
  1. **clarifier-agent**: 사용자 질문 명확화 (필요 시)
  2. **planner-agent**: 연구 계획서 작성
  3. **researcher-agent**: 실제 조사 수행 (검색/MCP 도구 사용)
     - 여러 연구 주제에 대해 병렬 실행 가능
  4. **compressor-agent**: 연구 결과 압축 및 요약
  5. **reporter-agent**: 최종 보고서 생성

- **상태(State) 관리**
  - Orchestrator의 메시지 히스토리에 모든 SubAgent의 결과가 누적
  - 각 SubAgent는 독립적인 상태를 가지며, 완료 시 결과를 Orchestrator에게 반환
  - 필요한 컨텍스트(연구 계획서, 중간 결과 등)는 메시지로 전달

- **텍스트 설계도**
```
User Input
  → Orchestrator
     → [clarifier-agent] (선택적)
     → [planner-agent]
     → [researcher-agent × N] (병렬 실행)
     → [compressor-agent]
     → [reporter-agent]
  → Final Output
```

---

### 2) 아키텍처 비교

#### A) 기존 DeepResearch 구조 (다중 서브그래프)
```
Main Graph
  ├─ clarify_with_user (node)
  ├─ write_research_brief (node)
  ├─ supervisor_subgraph
  │    ├─ supervisor (node)
  │    └─ supervisor_tools (node)
  │         └─ researcher_subgraph × N (병렬 호출)
  │              ├─ researcher (node)
  │              ├─ researcher_tools (node)
  │              └─ compress_research (node)
  └─ final_report_generation (node)
```

**특징:**
- 복잡한 중첩 구조 (3단계 서브그래프)
- Command를 통한 동적 라우팅
- supervisor가 여러 researcher를 직접 관리
- 상태 업데이트가 여러 레벨에 분산

#### B) 새로운 DeepAgent (Orchestrator) 구조
```
Orchestrator Agent (Main)
  └─ SubAgents (독립 에이전트)
       ├─ clarifier-agent
       ├─ planner-agent
       ├─ researcher-agent (병렬 실행 가능)
       ├─ compressor-agent
       └─ reporter-agent
```

**특징:**
- 평탄한 구조 (1단계 Orchestrator + SubAgents)
- `task` 도구를 통한 명시적 위임
- 각 SubAgent는 완전히 독립적 (자체 도구, 프롬프트, 로직)
- 상태 관리가 Orchestrator 메시지 히스토리로 단순화

---

### 3) 구현 방안

#### 폴더/모듈 구조
```
Day-05/DeepResearch/
├─ src/
│  ├─ deep_agent_researcher.py  # 메인 Orchestrator + SubAgents 정의
│  ├─ configuration.py           # 설정 (기존 유지)
│  ├─ state.py                   # 상태 스키마 (단순화)
│  ├─ prompts.py                 # 프롬프트 템플릿
│  ├─ utils.py                   # 도구/헬퍼 함수 (기존 유지)
│  └─ runner.py                  # 실행 엔트리포인트
└─ skills/                       # (선택) SKILL.md 번들
    ├─ clarifier/SKILL.md
    ├─ planner/SKILL.md
    ├─ researcher/SKILL.md
    ├─ compressor/SKILL.md
    └─ reporter/SKILL.md
```

#### 핵심 코드 구조

**1) SubAgent 정의 (deep_agent_researcher.py)**
```python
from deepagents import create_deep_agent
from deepagents.middleware.subagents import SubAgent
from langchain_tavily import TavilySearch

# 1. Clarifier SubAgent (선택적)
clarifier_agent = SubAgent(
    name="clarifier-agent",
    description="사용자 질문이 불명확한 경우 명확화 질문을 생성합니다.",
    system_prompt="""당신은 연구 질문 명확화 전문가입니다.

사용자의 연구 요청을 분석하고 다음을 판단하세요:
1. 질문이 충분히 명확한가?
2. 추가 정보가 필요한가?

명확화가 필요하면 구체적인 질문을 생성하고, 아니면 "질문이 충분히 명확합니다"라고 응답하세요.""",
    tools=[],
)

# 2. Planner SubAgent
planner_agent = SubAgent(
    name="planner-agent",
    description="사용자 요청을 구조화된 연구 계획서로 변환합니다.",
    system_prompt="""당신은 연구 계획 전문가입니다.

사용자의 연구 요청을 분석하여 다음을 포함하는 연구 계획서를 작성하세요:
1. 연구 목적 및 범위
2. 주요 조사 주제 (3-5개)
3. 각 주제별 조사 방향
4. 예상 결과물

계획서는 명확하고 실행 가능해야 하며, 각 조사 주제는 독립적으로 실행 가능해야 합니다.""",
    tools=[],
)

# 3. Researcher SubAgent (병렬 실행 가능)
researcher_agent = SubAgent(
    name="researcher-agent",
    description="특정 주제에 대한 심층 조사를 수행합니다. 한 번에 하나의 주제만 조사하세요.",
    system_prompt="""당신은 심층 연구 전문가입니다.

주어진 연구 주제에 대해 다음을 수행하세요:
1. 검색 도구를 사용하여 최신 정보 수집
2. MCP 도구(있다면)를 활용하여 추가 데이터 수집
3. 수집된 정보를 구조화하여 정리
4. 출처와 날짜를 명시

**중요**: 
- 한 번에 하나의 주제만 조사하세요.
- 최소 3-5개 이상의 신뢰할 수 있는 출처를 사용하세요.
- 최종 보고서는 명확하고 구조화되어야 하며, 사용자가 직접 볼 수 있는 형태여야 합니다.""",
    tools=[
        TavilySearch(max_results=5, search_depth="advanced"),
        # MCP 도구는 런타임에 동적으로 추가 가능
    ],
)

# 4. Compressor SubAgent
compressor_agent = SubAgent(
    name="compressor-agent",
    description="여러 연구 결과를 압축하고 중복을 제거하여 핵심 내용만 추출합니다.",
    system_prompt="""당신은 정보 압축 및 요약 전문가입니다.

여러 연구 결과를 분석하여:
1. 중복 정보 제거
2. 핵심 발견사항만 추출
3. 논리적 순서로 재구성
4. 출처와 날짜 보존

압축된 결과는 최종 보고서 생성에 직접 사용되므로, 중요한 정보를 누락하지 마세요.""",
    tools=[],
)

# 5. Reporter SubAgent
reporter_agent = SubAgent(
    name="reporter-agent",
    description="압축된 연구 결과를 바탕으로 최종 보고서를 생성합니다.",
    system_prompt="""당신은 연구 보고서 작성 전문가입니다.

연구 계획서와 압축된 연구 결과를 바탕으로 다음을 포함하는 최종 보고서를 작성하세요:
1. 요약 (Executive Summary)
2. 연구 목적 및 방법
3. 주요 발견사항 (각 주제별로 구조화)
4. 결론 및 시사점
5. 참고자료 (출처 목록)

보고서는 전문적이고 읽기 쉬우며, 사용자의 원래 질문에 명확히 답해야 합니다.""",
    tools=[],
)
```

**2) Orchestrator 생성**
```python
# Orchestrator 시스템 프롬프트
orchestrator_prompt = """당신은 심층 연구 프로젝트 매니저입니다. 복잡한 연구 작업을 관리하고 조정합니다.

## 역할
- 사용자의 연구 요청을 분석하고 전체 작업을 계획합니다
- 전문 SubAgent에게 작업을 위임하고 진행 상황을 모니터링합니다
- SubAgent들의 결과를 취합하여 최종 보고서를 생성합니다

## 사용 가능한 SubAgent
1. **clarifier-agent**: 불명확한 질문 명확화 (선택적)
2. **planner-agent**: 연구 계획서 작성
3. **researcher-agent**: 특정 주제 심층 조사 (병렬 실행 가능)
4. **compressor-agent**: 연구 결과 압축 및 요약
5. **reporter-agent**: 최종 보고서 생성

## 작업 흐름
1. 사용자 요청 분석
2. (선택) clarifier-agent로 질문 명확화
3. planner-agent로 연구 계획서 작성
4. 계획서의 각 조사 주제별로 researcher-agent를 병렬 호출
   - **중요**: 여러 researcher-agent를 동시에 호출하려면, 하나의 응답에서 여러 task 도구를 함께 호출하세요
5. 모든 조사 완료 후 compressor-agent로 결과 압축
6. reporter-agent로 최종 보고서 생성
7. 최종 보고서를 사용자에게 전달

## 제약사항
- 최대 {max_concurrent_research_units}개의 researcher-agent를 동시에 실행할 수 있습니다
- 각 SubAgent의 결과를 다음 단계에 명확히 전달하세요
- 최종 보고서는 사용자의 원래 질문에 직접 답해야 합니다
"""

# Orchestrator 생성
orchestrator = create_deep_agent(
    model=llm,  # 설정된 LLM 모델
    tools=[],   # Orchestrator는 SubAgent 위임 도구만 사용
    subagents=[
        clarifier_agent,
        planner_agent,
        researcher_agent,
        compressor_agent,
        reporter_agent,
    ],
    system_prompt=orchestrator_prompt,
)
```

**3) 실행 예시 (runner.py)**
```python
from langchain.messages import HumanMessage
from deep_agent_researcher import orchestrator

async def run_research(query: str):
    """연구 쿼리를 실행하고 결과를 반환합니다."""
    result = await orchestrator.ainvoke({
        "messages": [HumanMessage(content=query)]
    })
    return result

# 실행
if __name__ == "__main__":
    import asyncio
    
    query = "2025년 AI 트렌드를 조사하고, LLM, 멀티모달, AI 에이전트 세 가지 측면에서 분석해줘."
    result = asyncio.run(run_research(query))
    
    # 최종 보고서 출력
    final_message = result["messages"][-1]
    print(final_message.content)
```

---

### 4) 상태 관리 단순화

#### 기존 DeepResearch 상태 (복잡)
```python
class AgentState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    research_brief: str
    supervisor_messages: Annotated[list[AnyMessage], add_messages]
    notes: list[str]
    raw_notes: list[str]
    research_iterations: int
    tool_call_iterations: int
    compressed_research_length: int
    raw_notes_length: int
    final_report: str
```

#### 새로운 DeepAgent 상태 (단순)
```python
class OrchestratorState(TypedDict):
    """Orchestrator는 메시지 히스토리만 관리"""
    messages: Annotated[list[AnyMessage], add_messages]
```

**이유:**
- 모든 중간 결과(연구 계획서, 조사 결과, 압축 내용)는 메시지 히스토리에 자동으로 누적
- SubAgent들이 독립적으로 실행되고 결과를 반환
- 상태 동기화 문제 제거
- 디버깅 및 추적 용이

---

### 5) 도구(Tools) 통합

#### MCP 도구 통합
```python
from utils import get_all_tools

async def create_researcher_with_mcp_tools(config: RunnableConfig):
    """MCP 도구를 포함한 researcher 생성"""
    tools = await get_all_tools(config)  # 검색 + MCP 도구
    
    researcher_agent = SubAgent(
        name="researcher-agent",
        description="심층 조사 수행 (검색 + MCP 도구)",
        system_prompt=research_system_prompt.format(
            mcp_prompt=config["configurable"].get("mcp_prompt", ""),
            date=get_today_str()
        ),
        tools=tools,  # 동적으로 로드된 도구
    )
    return researcher_agent
```

#### think_tool 통합
```python
from utils import think_tool

# Orchestrator에 think_tool 추가 (전략적 계획용)
orchestrator = create_deep_agent(
    model=llm,
    tools=[think_tool],  # 내부 사고 도구
    subagents=[...],
    system_prompt=orchestrator_prompt,
)
```

---

### 6) 설정(Configuration) 주입

```python
from langchain_core.runnables import RunnableConfig

# 실행 시 설정 주입
config = RunnableConfig(
    configurable={
        # 모델 설정
        "research_model": "openai/gpt-4.1",
        "compression_model": "openai/gpt-4.1-mini",
        "final_report_model": "openai/gpt-4.1",
        
        # 한도 설정
        "max_concurrent_research_units": 3,  # 병렬 연구 최대 수
        "max_researcher_iterations": 5,      # 연구 반복 한도
        "max_react_tool_calls": 10,          # 도구 호출 한도
        
        # MCP 설정
        "mcp_servers": [
            {"name": "person", "transport": "streamable-http", "url": "http://127.0.0.1:8000/mcp"}
        ],
        "mcp_prompt": "MCP 도구 사용 시 안전한 입력만 사용하고, DB 변경 전 확인을 받으세요.",
        
        # 기타
        "allow_clarification": True,  # 명확화 단계 활성화
    }
)

result = await orchestrator.ainvoke({"messages": [HumanMessage(content=query)]}, config=config)
```

---

### 7) 평가 및 로깅 (Langfuse)

```python
from langfuse.callback import CallbackHandler

# Langfuse 콜백 생성
langfuse_handler = CallbackHandler(
    public_key="...",
    secret_key="...",
    host="https://cloud.langfuse.com"
)

# Orchestrator에 콜백 주입
result = await orchestrator.ainvoke(
    {"messages": [HumanMessage(content=query)]},
    config={
        "callbacks": [langfuse_handler],
        "configurable": {...}
    }
)

# 모든 SubAgent 호출이 자동으로 Langfuse에 기록됨
```

---

### 8) 마이그레이션 체크리스트

#### 제거할 것 (기존 DeepResearch)
- ❌ `supervisor` / `supervisor_tools` 노드
- ❌ `supervisor_subgraph` 서브그래프
- ❌ `researcher` / `researcher_tools` / `compress_research` 노드
- ❌ `researcher_subgraph` 서브그래프
- ❌ `Command` 기반 동적 라우팅
- ❌ 복잡한 상태 관리 (`SupervisorState`, `ResearcherState`, 등)
- ❌ 도구 실행 로직 (`supervisor_tools`, `researcher_tools`)

#### 새로 추가할 것 (DeepAgent)
- ✅ `SubAgent` 정의 (clarifier, planner, researcher, compressor, reporter)
- ✅ `Orchestrator` 에이전트 (create_deep_agent)
- ✅ 단순화된 상태 (`OrchestratorState`)
- ✅ SubAgent별 시스템 프롬프트 (skills/ 폴더 또는 prompts.py)
- ✅ 설정 주입 (RunnableConfig)

#### 유지할 것
- ✅ `utils.py` (get_all_tools, 에러 핸들링, 토큰 체크 등)
- ✅ `configuration.py` (설정 스키마)
- ✅ `prompts.py` (프롬프트 템플릿, 필요시 수정)
- ✅ `runner.py` (진입점, 단순화)

---

### 9) 기존 MD와의 차이점 비교

#### A) DeepResearch_to_DeepAgent_Design.md (싱글 에이전트 + 스킬)

**아키텍처:**
```
Single Agent Loop
  ├─ skill_clarify()
  ├─ skill_plan()
  ├─ while not done:
  │    ├─ policy_decide_next_skill()
  │    └─ skill_research() or skill_compress()
  └─ skill_report()
```

**특징:**
- ✅ 최대 단순화: 하나의 에이전트가 모든 스킬을 순차 실행
- ✅ 명시적 정책(Policy): 규칙 기반 + LLM 보조로 다음 스킬 결정
- ✅ 파일시스템 메모리: 증거/노트를 외부 저장소에 보관
- ❌ 병렬 실행 제한: 하나의 루프에서 순차 처리
- ❌ 모듈 재사용 어려움: 스킬이 함수로 정의되어 독립 실행 불가

**장점:**
- 설계가 매우 단순하고 이해하기 쉬움
- 상태 관리가 중앙집중식
- 디버깅이 직관적 (한 곳에서 모든 로직 확인 가능)

**단점:**
- 확장성 제한: 새로운 스킬 추가 시 policy 로직 수정 필요
- 병렬 처리 불가: 여러 연구 주제를 동시에 조사할 수 없음
- 재사용성 낮음: 스킬을 다른 프로젝트에서 재사용하기 어려움

#### B) DeepResearch_to_DeepAgent_with_Orchestrator.md (Orchestrator + SubAgent)

**아키텍처:**
```
Orchestrator Agent
  ├─ clarifier-agent (SubAgent)
  ├─ planner-agent (SubAgent)
  ├─ researcher-agent × N (SubAgent, 병렬)
  ├─ compressor-agent (SubAgent)
  └─ reporter-agent (SubAgent)
```

**특징:**
- ✅ 모듈화: 각 SubAgent가 완전히 독립적
- ✅ 병렬 실행: 여러 researcher-agent를 동시에 실행 가능
- ✅ 재사용성: SubAgent를 다른 프로젝트에서 재사용 가능
- ✅ 확장 용이: 새로운 SubAgent 추가 시 기존 코드 수정 불필요
- ❌ 복잡도 증가: SubAgent 간 통신 및 상태 동기화 필요
- ❌ 디버깅 어려움: 여러 SubAgent에 걸친 실행 흐름 추적

**장점:**
- 확장성: 새로운 전문 에이전트를 쉽게 추가
- 병렬 처리: 여러 작업을 동시에 실행하여 속도 향상
- 재사용성: SubAgent를 독립 모듈로 다른 시스템에서 활용
- 유연성: 각 SubAgent의 도구/모델을 독립적으로 설정

**단점:**
- 복잡도: Orchestrator + SubAgent 구조가 싱글 에이전트보다 복잡
- 오버헤드: SubAgent 간 메시지 전달로 인한 약간의 성능 오버헤드
- 디버깅: 여러 레벨에 걸친 실행 흐름 추적이 어려움

---

### 10) 사용 사례별 추천

#### 싱글 에이전트 + 스킬 (Design.md) 추천 상황
- 📚 학습/프로토타이핑: DeepAgent 개념 이해 및 빠른 실험
- 🔍 단순 연구: 단일 주제에 대한 순차적 조사
- 💻 로컬 실행: 리소스 제약이 있는 환경
- 🐛 디버깅 중심: 명확한 실행 흐름이 중요한 경우

#### Orchestrator + SubAgent (본 문서) 추천 상황
- 🚀 프로덕션: 실제 서비스에서 복잡한 연구 수행
- 🔄 병렬 처리: 여러 주제를 동시에 조사해야 하는 경우
- 🧩 모듈 재사용: SubAgent를 다양한 프로젝트에서 활용
- 📈 확장성: 지속적으로 새로운 전문 에이전트 추가 예정
- 🏢 팀 협업: 각 SubAgent를 개별 팀이 개발/유지보수

---

### 11) 적용 후 기대 효과

#### 기존 DeepResearch 대비
- ✅ **복잡도 50% 감소**: 3단계 서브그래프 → 1단계 Orchestrator
- ✅ **병렬 처리 속도 3배 향상**: researcher-agent 동시 실행
- ✅ **유지보수성 향상**: 각 SubAgent 독립 수정 가능
- ✅ **확장성**: 새 SubAgent 추가가 비침투적

#### 싱글 에이전트 + 스킬 대비
- ✅ **병렬 처리 가능**: 여러 연구 주제 동시 조사
- ✅ **재사용성 향상**: SubAgent를 독립 모듈로 활용
- ❌ **복잡도 증가**: 구조가 더 복잡해짐
- ❌ **디버깅 어려움**: 실행 흐름 추적 복잡

---

### 12) 결론 및 권장사항

**이 문서의 Orchestrator + SubAgent 패턴은 다음과 같은 경우에 적합합니다:**

1. ✅ **복잡한 연구 프로젝트**: 여러 주제를 동시에 조사하고 통합
2. ✅ **프로덕션 환경**: 안정성과 확장성이 중요
3. ✅ **팀 협업**: 각 SubAgent를 개별적으로 개발/테스트
4. ✅ **장기 프로젝트**: 지속적인 기능 추가 및 개선 예정

**다음과 같은 경우에는 싱글 에이전트 + 스킬 패턴을 고려하세요:**

1. 📚 학습 목적 또는 프로토타이핑
2. 🔍 단순한 순차적 연구
3. 💻 리소스 제약 환경
4. 🐛 명확한 디버깅이 중요

**최종 선택 가이드:**
- 빠른 시작과 단순함이 중요하면 → **싱글 에이전트 + 스킬**
- 확장성과 병렬 처리가 중요하면 → **Orchestrator + SubAgent** (본 문서)


## DeepResearch → DeepAgent 전환 설계 문서

본 문서는 Day-05/DeepResearch를 Day-03/DeepAgent 스타일의 “싱글 에이전트 + 스킬 번들” 구조로 리팩터링하기 위한 설계와 구현 포인트를 제시합니다. 아직 코드 반영은 하지 않습니다.

---

### 1) 설계 및 설계도

- 목표
  - 다중 서브그래프(Supervisor/Researcher/Compressor) 중심의 워크플로우를 “싱글 에이전트”가 동적으로 스킬을 선택·실행하는 형태로 통합.
  - 스킬(clarify/plan/research/compress/report/guardrails) 기반으로 DeepAgent 정책(룰+LLM 보조)으로 다음 스킬을 결정.

- 핵심 아이디어
  - 싱글 에이전트 루프
    - 입력 → clarify → plan → [loop: policy(decide) → research or compress] → report → 종료
  - 스킬 번들(Skills Bundle)
    - 각 스킬은 독립 함수/모듈로 유지. SKILL.md의 메타데이터를 시스템 프롬프트에 주입하여 행동 가이드와 제약을 중앙 관리.
  - 상태/메모리 외부화
    - 증거·노트·산출물은 Filesystem/Vector/DB에 저장, 컨텍스트 압축과 재시작 내성 강화.

- 상태(State)
  - messages, plan, todos, notes/raw_notes, findings, counters(반복/도구호출), config(모델/토큰/MCP/한도)

- 동적 할당 정책(Policy)
  - 규칙 우선:
    - 최초: clarify → plan
    - 도구 필요: research
    - 도구 호출 없음/한도 도달: compress
    - 완료 기준 충족: report
  - LLM 보조:
    - state 요약 → 다음 스킬 제안 → 규칙 검증 후 최종 결정

- 텍스트 설계도
```
Input
  → skill.clarify
  → skill.plan
  → while not done:
      - policy.decide_next_skill(state)
      - skill.research | skill.compress
    end
  → skill.report
  → Output
```

---

### 2) 구현 방안

- 폴더/모듈 구조(DeepAgent 스타일)
  - `src/agent.py`: 싱글 에이전트 오케스트레이터(상태/정책/루프)
  - `src/skills/clarify.py`
  - `src/skills/plan.py`
  - `src/skills/research.py`
  - `src/skills/compress.py`
  - `src/skills/report.py`
  - `src/skills/guardrails.py`
  - `src/tools/...`: 검색/MCP/think/FS/Todo 어댑터 (기존 `utils.get_all_tools` 재사용)
  - `skills/<skill-name>/SKILL.md`: 각 스킬 가이드/메타데이터

- 실행 루프(의사코드)
```python
async def run_single_agent(state: AgentState, config: RunnableConfig):
    # 1) 초기 단계
    state = await skill_clarify(state, config)
    state = await skill_plan(state, config)

    # 2) 메인 루프
    while True:
        next_skill = policy_decide_next_skill(state, config)
        if next_skill == "research":
            state = await skill_research(state, config)
        elif next_skill == "compress":
            state = await skill_compress(state, config)
        elif next_skill == "report":
            state = await skill_report(state, config)
            break
        else:
            # fallback: 안전 종료
            state = await skill_report(state, config)
            break
    return state
```

- 정책(Policy) 예시
```python
def policy_decide_next_skill(state: AgentState, config: RunnableConfig) -> str:
    counters = state.get("counters", {})
    tool_calls = counters.get("tool_calls", 0)
    iterations = counters.get("iterations", 0)

    # 규칙 기반
    if "plan" not in state:
        return "plan"
    if tool_calls < config["configurable"].get("max_tool_calls", 6):
        return "research"
    if iterations >= config["configurable"].get("max_iterations", 3):
        return "report"
    # 도구 호출이 더 이상 필요 없으면 압축
    return "compress"
```

- 프롬프트/컨텍스트
  - Skills Bundle Prompt: 모든 SKILL.md를 읽어 시스템 프롬프트에 합성
  - MCP Prompt: MCP 가이드·보안 제약을 연구 스킬 프롬프트에 삽입
  - 날짜/출처/증거 필수화

```python
def build_skills_bundle_prompt(skills_root: Path) -> str:
    docs = []
    for skill_dir in skills_root.iterdir():
        md = skill_dir / "SKILL.md"
        if not md.exists():
            continue
        front, body = split_frontmatter(md.read_text(encoding="utf-8"))
        meta = parse_yaml_light(front)
        docs.append(f"## [{meta.get('name', skill_dir.name)}]\n- desc: {meta.get('description','')}\n- metadata: {meta.get('metadata',{})}\n{body[:800]}")
    return "\n\n".join(docs)
```

- 한도/가드레일
```python
def guardrails_validate_input(user_query: str) -> None:
    if len(user_query) > 4000:
        raise ValueError("입력이 너무 깁니다. 요약 후 다시 시도하세요.")
    # 필요 시 정규식/금칙어/PII 필터 등 적용
```

- 평가/로깅(Langfuse 예)
```python
from langfuse import Langfuse
from langfuse.callback import CallbackHandler

langfuse = Langfuse()
lf_cb = CallbackHandler()

# 각 스킬 내 LLM 호출 시
model = llm_model.with_config({"callbacks": [lf_cb], "model": cfg.research_model, "max_tokens": 4000})
resp = await model.ainvoke(messages)
langfuse.flush()
```

---

### 3) 중요 코드 예시

1) Research 스킬(도구 바인딩/병렬 제한/노트 집계)
```python
async def skill_research(state: AgentState, config: RunnableConfig) -> AgentState:
    cfg = Configuration.from_runnable_config(config)
    tools = await get_all_tools(config)  # 검색 + MCP + think_tool (기존 유틸 재사용)

    system_prompt = research_system_prompt.format(
        mcp_prompt=cfg.mcp_prompt or "", date=get_today_str()
    )
    messages = [SystemMessage(content=system_prompt)] + state.get("messages", [])

    model = (
        llm_model.bind_tools(tools)
        .with_retry(stop_after_attempt=cfg.max_structured_output_retries)
        .with_config({"model": cfg.research_model, "max_tokens": cfg.research_model_max_tokens,
                      "api_key": get_api_key_for_model(cfg.research_model, config)})
    )
    response = await model.ainvoke(messages)

    # 도구 호출 실행 (병렬 제한)
    tool_calls = response.tool_calls or []
    allowed = tool_calls[: cfg.max_concurrent_research_units]
    tasks = []
    for call in allowed:
        tool = next(t for t in tools if (getattr(t, "name", None) or t.get("name")) == call["name"])
        tasks.append(tool.ainvoke(call["args"], config))
    observations = await asyncio.gather(*tasks) if tasks else []

    # 노트/원시노트 집계
    raw = "\n".join([str(obs) for obs in observations])
    notes = state.get("notes", [])
    if raw:
        notes.append(raw)

    # 카운터 갱신
    counters = state.get("counters", {})
    counters["tool_calls"] = counters.get("tool_calls", 0) + len(allowed)
    counters["iterations"] = counters.get("iterations", 0) + 1

    return {
        **state,
        "messages": [response],
        "notes": notes,
        "counters": counters,
    }
```

2) Compress 스킬(토큰 초과 방어)
```python
async def skill_compress(state: AgentState, config: RunnableConfig) -> AgentState:
    cfg = Configuration.from_runnable_config(config)
    synthesizer = llm_model.with_config({
        "model": cfg.compression_model, "max_tokens": cfg.compression_model_max_tokens,
        "api_key": get_api_key_for_model(cfg.compression_model, config),
    })

    researcher_messages = state.get("messages", [])
    researcher_messages.append(HumanMessage(content=compress_research_simple_human_message))

    attempts, max_attempts = 0, 3
    while attempts < max_attempts:
        try:
            prompt = compress_research_system_prompt.format(date=get_today_str())
            msg = [SystemMessage(content=prompt)] + researcher_messages
            resp = await synthesizer.ainvoke(msg)
            return {**state, "findings": resp.content}
        except Exception as e:
            attempts += 1
            if is_token_limit_exceeded(e, cfg.compression_model):
                researcher_messages = remove_up_to_last_ai_message(researcher_messages)
                continue
            break
    return {**state, "findings": "Error: compression failed"}
```

3) Report 스킬(점진 절단)
```python
async def skill_report(state: AgentState, config: RunnableConfig) -> AgentState:
    cfg = Configuration.from_runnable_config(config)
    writer = llm_model.with_config({
        "model": cfg.final_report_model, "max_tokens": cfg.final_report_model_max_tokens,
        "api_key": get_api_key_for_model(cfg.final_report_model, config),
    })

    findings = "\n".join(state.get("notes", []))
    max_retries, retry = 3, 0
    token_limit_chars = None

    while retry <= max_retries:
        try:
            final_prompt = final_report_generation_prompt.format(
                research_brief=state.get("plan", ""),
                messages=get_buffer_string(state.get("messages", [])),
                findings=findings,
                date=get_today_str(),
            )
            resp = await writer.ainvoke([HumanMessage(content=final_prompt)])
            return {**state, "final_report": resp.content}
        except Exception as e:
            if is_token_limit_exceeded(e, cfg.final_report_model):
                retry += 1
                if token_limit_chars is None:
                    model_limit = get_model_token_limit(cfg.final_report_model) or 8192
                    token_limit_chars = model_limit * 4
                else:
                    token_limit_chars = int(token_limit_chars * 0.9)
                findings = findings[: token_limit_chars]
                continue
            return {**state, "final_report": f"Error: {e}"}
```

4) MCP 서버 설정 주입 예시(실행 시 config로 전달)
```python
config = RunnableConfig(configurable={
    "mcp_servers": [
        {"name": "person", "transport": "streamable-http", "url": "http://127.0.0.1:8000/mcp"}
    ],
    "mcp_prompt": "MCP 도구는 안전한 입력만 사용하고, DB 변경 전 사용자에게 요약을 확인받아라.",
    "research_model": "openai/gpt-4.1",
    "max_tool_calls": 6,
    "max_iterations": 3,
})
```

---

### 4) 마이그레이션 체크리스트

- `deep_researcher.py`
  - 서브그래프 제거(`supervisor_*`, `researcher_*` 노드) → 스킬 함수로 분리
  - `Command` 라우팅 → `run_single_agent + policy_decide_next_skill`로 통합
  - `get_all_tools(config)` 호출 위치 → `skill_research` 내부로 이동
  - Skills Bundle Prompt/MCP Prompt 주입
  - 콜백(Langfuse) 각 스킬 LLM 호출에 주입

- `prompts.py`
  - Skills Bundle 기반 템플릿 추가/수정

- `utils.py`
  - 도구 로딩/콜백/에러방어 유틸 보강(필요 시)

---

### 5) 적용 후 기대 효과

- 설계 단순화: 라우팅/상태 업데이트를 단일 루프에서 관리
- 확장 용이: 스킬 추가·제거가 비침투적으로 가능
- 재현성·품질: SKILL.md 기반 행동 가이드, MCP 가드레일, 파일시스템 메모리
- 운영 안정성: 반복/도구 호출/비용 한도 제어와 토큰 초과 자동 복구



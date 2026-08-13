# Agents CLI in Agent Platform

## 사전 준비 (Before You Begin)

### 1. Cloud Console 접속
1. Google Cloud Console 접속
2. Cloud Shell 접속 (우측 상단)
3. Cloud Shell Editor 활성화 (Cloud Shell 에서 open Editor)
  - 상단 메뉴 - View - Terminal
  - 상단 메뉴 - View - Toggle Hidden Files

### 2. 필수 API 활성화
터미널에서 아래 명령을 실행하여 필요한 Cloud API를 활성화합니다.
```bash
gcloud services enable aiplatform.googleapis.com \
  run.googleapis.com \
  cloudtrace.googleapis.com \
  cloudbuild.googleapis.com
```
검색창에서 "Agent Platform" 검색 후 진입, 상단의 Enable APIs 를 선택해서 관련 API 들을 활성화

### 3. 인증 진행
```bash
gcloud auth login
gcloud auth application-default login
```

### 4. 환경 변수 설정
`YOUR_PROJECT_ID` 부분을 본인의 GCP 프로젝트 ID로 변경하여 실행합니다.
```bash
export GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
export GOOGLE_CLOUD_LOCATION=global
```

---

## 2. Agents CLI 설치 (Install Agents CLI)

```bash
uvx google-agents-cli setup
```

#### 설치 확인
```bash
agents-cli --version
# 출력 예시: agents-cli, version 1.3.1
```

---

## 3. 에이전트 프로젝트 생성
ADK Agent 개발을 위한 프로젝트 구조를 즉시 생성하기 위해 퀵 모드를 사용합니다:
```bash
agents-cli scaffold create customer-support-agent --prototype --yes
```

```bash
cd customer-support-agent
```

---

## 4. 로컬 테스트 (Test Locally with Playground)

대화형 UI를 통해 에이전트 동작을 테스트합니다.

### 1. 의존성 패키지 설치
```bash
agents-cli install
```
*(내부적으로 `uv sync`를 실행하여 `.venv` 환경을 구축합니다.)*

### 2. Playground 실행
```bash
agents-cli playground
```


### 3. 로컬 테스트
1. 접속 주소: [http://127.0.0.1:8080/dev-ui/?app=app](http://127.0.0.1:8080/dev-ui/?app=app)
2. 샘플 질문 테스트:
   - `"What's the weather in San Francisco?"`
   - `"도쿄의 날씨는 어때?"`
   - `"What time is it in San Francisco?"`
   - `"서울의 날씨와 현재 시간을 알려주세요"`

---

## 5. CLI에서 실행 (Run from Command Line)

웹 브라우저를 켜지 않고 터미널에서 빠르게 에이전트를 테스트합니다.

### 실행 명령
```bash
agents-cli run "What's the weather in San Francisco?"
```

#### 실행 결과 예시
```text
[user]: What's the weather in San Francisco?
[root_agent]:
[tool_call: get_weather({"query": "San Francisco"})]
[tool_response: get_weather -> {"result": "It's 60 degrees and foggy."}]
The weather in San Francisco is 60 degrees and foggy.

Session: fb30f7f7-147e-4697-8aaa-706d604589fa (resume with --session-id)
```

---

## 6. 에이전트 평가 (Evaluate Your Agent) (선택사항)

### 평가 디렉토리 구조 (`tests/eval/`)
- `tests/eval/datasets/`: 평가용 데이터셋 (JSON 형식)
- `tests/eval/eval_config.yaml`: 평가 메트릭 및 평가기 설정
- `tests/eval/response_quality.py`: LLM-as-judge 기반 응답 품질 채점 로직 (Score 1~5점)

---

### 평가 실행 (Evaluation Run)

**옵션 1: 추론 및 채점 일괄 실행 (권장)**
```bash
agents-cli eval run
```

**옵션 2: 2단계 분리 실행**
```bash
# 1단계: 평가 데이터셋 기반 에이전트 추론 실행 및 트레이스 생성
agents-cli eval generate

# 2단계: 생성된 트레이스 및 응답 채점 (LLM-as-judge)
agents-cli eval grade
```

#### 실행 결과 확인
평가가 완료되면 케이스별 점수(Score 1~5)와 LLM 평가자의 피드백/이유(Explanation)가 터미널에 요약 출력됩니다.

---

## 7. Agent Runtime 배포

Google Cloud의 서버리스 관리형 환경인 **Agent Runtime** 배포 아키텍처를 프로젝트에 반영합니다.

### 1. 프로젝트 설정 파일 업데이트
```bash
agents-cli scaffold enhance --deployment-target agent_runtime --yes
```

### 2. .env 파일 설정 추가 (Telemetry for ADK)
.env 파일에 다음 내용 추가
```bash
# Telemetry for ADK
GOOGLE_CLOUD_AGENT_ENGINE_ENABLE_TELEMETRY=true
OTEL_SEMCONV_STABILITY_OPT_IN="gen_ai_latest_experimental"
OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=EVENT_ONLY
```

### 3. 배포 실행

  ```bash
  agents-cli deploy --project $GOOGLE_CLOUD_PROJECT --region us-central1
  ```

> ⏱️ 최초 배포에는 약 **5~10분**이 소요됩니다. (이후 재배포는 더 빠르게 진행됩니다.)

#### 완료 결과 출력 예시
```text
✅ Deployment successful!
Agent Runtime ID: projects/.../locations/us-central1/reasoningEngines/XXXXXXXXXXXXXXXXXX
Service Account: service-XXXXXXXXX@gcp-sa-aiplatform-re.iam.gserviceaccount.com

📊 Open Console Playground: https://console.cloud.google.com/vertex-ai/agents/agent-engines/locations/us-central1/agent-engines/XXXXXXXXXXXXXXXXXX/playground?project=YOUR_PROJECT_ID
```

---

## 8. 배포된 에이전트 테스트 및 모니터링 (Test and Monitor Deployed Agent)

### 배포 에이전트 테스트 방법

#### 옵션 1: Console Playground (가장 쉬운 방법)
배포 완료 메시지에 출력된 Google Cloud Console 링크를 클릭하여 브라우저 대화 UI에서 배포된 에이전트를 테스트합니다.

#### 옵션 2: agents-cli run --url
터미널에서 원격 배포 엔드포인트로 직접 쿼리를 전송합니다:
```bash
RUNTIME_ID=$(jq -r .remote_agent_runtime_id deployment_metadata.json)

agents-cli run \
  --url "https://us-central1-aiplatform.googleapis.com/v1/${RUNTIME_ID}" \
  --mode adk \
  "What's the weather in San Francisco?"
```

---

## 9. MemoryBank 활성화

### app/agent.py에 Memory Bank 사용을 위한 콜백 함수 추가
```python
from google.adk.agents.callback_context import CallbackContext

async def add_session_to_memory_callback(callback_context: CallbackContext):
    await callback_context.add_session_to_memory()
    return None
```
https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/memory-bank/adk-quickstart#manage-memories
```python
root_agent = Agent(
  name="root_agent",
  model=Gemini(
      model=MODEL,
      retry_options=types.HttpRetryOptions(attempts=3),
  ),
  instruction="You are a helpful AI assistant designed to provide accurate and useful information.",
  # 다음 한 라인을 주석 처리하고 아래 두 라인을 추가
  # tools=[get_weather, get_current_time],
  tools=[get_weather, get_current_time, PreloadMemoryTool()],
  after_agent_callback=add_session_to_memory_callback,
)
```

최종 결과
```python
# ruff: noqa
# Copyright 2026 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

import datetime
from zoneinfo import ZoneInfo

from google.adk.agents import Agent
from google.adk.apps import App
from google.adk.models import Gemini
from google.genai import types

from google.adk.agents.callback_context import CallbackContext
from google.adk.tools import preload_memory

### Memory Bank 사용을 위한 콜백 함수 추가
async def add_session_to_memory_callback(callback_context: CallbackContext):
    # Session 내의 정보들을 Memory Bank에 저장
    await callback_context.add_session_to_memory()
    return None

MODEL = "gemini-3.6-flash"

def get_weather(query: str) -> str:
    """Simulates a web search. Use it get information on weather.

    Args:
        query: A string containing the location to get weather information for.

    Returns:
        A string with the simulated weather information for the queried location.
    """
    if "sf" in query.lower() or "san francisco" in query.lower():
        return "It's 60 degrees and foggy."
    return "It's 90 degrees and sunny."


def get_current_time(query: str) -> str:
    """Simulates getting the current time for a city.

    Args:
        city: The name of the city to get the current time for.

    Returns:
        A string with the current time information.
    """
    if "sf" in query.lower() or "san francisco" in query.lower():
        tz_identifier = "America/Los_Angeles"
    else:
        return f"Sorry, I don't have timezone information for query: {query}."

    tz = ZoneInfo(tz_identifier)
    now = datetime.datetime.now(tz)
    return f"The current time for query {query} is {now.strftime('%Y-%m-%d %H:%M:%S %Z%z')}"


root_agent = Agent(
    name="root_agent",
    model=Gemini(
        model=MODEL,
        retry_options=types.HttpRetryOptions(attempts=3),
    ),
    instruction="You are a helpful AI assistant designed to provide accurate and useful information.",
    ### PreloadMemoryTool() : 에이전트가 매 턴 시작 시 메모리를 검색하고 검색된 메모리를 시스템 메시지로 주입.
    tools=[get_weather, get_current_time, preload_memory],
    ### after_agent_callback: 에이전트 추론 완료 후 실행할 콜백 함수. 세션 메모리를 저장하는데 사용.
    after_agent_callback=add_session_to_memory_callback,
)

app = App(
    root_agent=root_agent,
    name="app",
)

```

### 코드 변경한 부분을 재 배포
```bash
agents-cli deploy --project $GOOGLE_CLOUD_PROJECT --region us-central1
```
---

## 11. Gemini Enterprise에 게시 (Publish to Gemini Enterprise)

Gemini Enterprise 를 먼저 활성화 합니다.

### 1. 대상 앱 목록 조회
```bash
agents-cli publish gemini-enterprise --list
```

### 2. 에이전트 등록
```bash
# agents-cli publish gemini-enterprise --list 명령어의 결과를 아래 환경변수에 등록
export GE_APP_ID=YOUR_GE_APP_ID

agents-cli publish gemini-enterprise \
  --gemini-enterprise-app-id "${GE_APP_ID}" \
  --display-name "Customer Support Agent" \
  --description "Answers weather and time questions" \
  --tool-description "Use this tool to ask the customer support agent."
```

---

# (선택사항) Antigravity CLI

### 1. Antigravity CLI 설치 (선택 사항)
Antigravity CLI를 설치합니다. (https://antigravity.google/download)

```bash
# Antigravity CLI 다운로드 및 설치
curl -fsSL https://antigravity.google/cli/install.sh | bash
```
```bash
# PATH 환경 변수 등록 및 반영
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
```

#### Antigravity CLI 실행
```bash
agy
# 메뉴 선택: Use a Google Cloud Project -> Continue with Google Cloud
```

#### ADK MCP (Model Context Protocol) 설정 (선택 사항)
[ADK MCP 가이드](https://adk.dev/tutorials/coding-with-ai/#antigravity)에 따라 Antigravity CLI MCP 설정 파일(`~/.gemini/antigravity-cli/mcp_config.json`)을 생성합니다:

```bash
cat << 'EOF' > ~/.gemini/antigravity-cli/mcp_config.json
{
  "mcpServers": {
    "adk-docs-mcp": {
      "command": "uvx",
      "args": [
        "--with",
        "mcp<2",
        "--from",
        "mcpdoc",
        "mcpdoc",
        "--urls",
        "AgentDevelopmentKit:https://adk.dev/llms.txt",
        "--transport",
        "stdio"
      ]
    }
  }
}
EOF
```

---

이 실습은 [Google Codelab: Agents CLI in Agent Platform](https://codelabs.developers.google.com/agents-cli-agent-platform/agents-cli-agent-platform)을 바탕으로 한 실습 가이드입니다.
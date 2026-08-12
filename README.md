# Agents CLI in Agent Platform

이 실습은 [Google Codelab: Agents CLI in Agent Platform](https://codelabs.developers.google.com/agents-cli-agent-platform/agents-cli-agent-platform)을 바탕으로 한 실습 가이드입니다.

---

## 1. 사전 준비 (Before You Begin)

### 1. Cloud Console 접근
1. Google Cloud Console 접속
2. CloudShell 연결 (Open Editor 버튼으로 Cloud Shell Editor에 들어가서 Terminal View 열어서 작업 추천)
   - View - Toggle Hidden Files
   - View - Terminal

### 2. 필수 API 활성화
터미널에서 아래 명령을 실행하여 필요한 Cloud API를 활성화합니다.
```bash
gcloud services enable aiplatform.googleapis.com \
  run.googleapis.com \
  cloudtrace.googleapis.com \
  cloudbuild.googleapis.com
```

### 3. 인증 진행
```bash
gcloud auth login
gcloud auth application-default login
```

### 4. 환경 변수 설정
`YOUR_PROJECT_ID` 부분을 본인의 GCP 프로젝트 ID로 변경하여 실행합니다.
```bash
export GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
export GOOGLE_CLOUD_LOCATION=us-central1
```

---

## 2. Antigravity CLI 및 Agents CLI 설치

### 1. Antigravity CLI 설치 (선택 사항)
Antigravity CLI를 설치합니다. (https://antigravity.google/download)

```bash
# Antigravity CLI 다운로드 및 설치
curl -fsSL https://antigravity.google/cli/install.sh | bash

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

### 2. Agents CLI 설치 (Install Agents CLI)

```bash
uvx google-agents-cli setup
```

#### 설치 확인
```bash
agents-cli --version
# 출력 예시: agents-cli, version 0.1.2
```

---

## 3. 에이전트 프로젝트 생성

### 🤖 Coding Agent 모드
Antigravity CLI에 다음과 같이 요청합니다:
> "prototype 템플릿을 사용하여 customer-support-agent라는 이름의 새로운 ADK 에이전트 프로젝트를 생성해 줘."

### 👤 수동 (Manual) 모드
실습과 완벽히 일치하는 프로젝트 구조를 즉시 생성하기 위해 퀵 모드를 사용합니다:
```bash
agents-cli scaffold create customer-support-agent --prototype --yes
```

---

## 4. 에이전트 코드 탐색 (Explore the Agent Code)

프로젝트 디렉토리로 이동합니다:
```bash
cd customer-support-agent
```

환경 변수 파일(`.env`) 로딩을 위해 `app/agent.py` 파일에 다음 내용을 추가합니다:

```python
from dotenv import load_dotenv
load_dotenv()
```

---

## 5. Playground로 로컬 테스트 (Test Locally with Playground)

대화형 UI를 통해 에이전트 동작을 테스트합니다.

### 1. 의존성 패키지 설치
```bash
agents-cli install
```
*(내부적으로 `uv sync`를 실행하여 `.venv` 환경을 구축합니다.)*

### 2. Playground 실행
- **🤖 Coding Agent**: Antigravity CLI에 다음과 같이 요청합니다:
  > "Start the playground for my agent"
- **👤 수동 모드**:
  ```bash
  agents-cli playground
  ```

### 3. 로컬 테스트
1. 접속 주소: [http://127.0.0.1:8080/dev-ui/?app=app](http://127.0.0.1:8080/dev-ui/?app=app)
2. 예시 질문 테스트:
   - `"What's the weather in San Francisco?"`
   - `"What's the weather in Tokyo?"`
   - `"What time is it in San Francisco?"`

---

## 6. CLI에서 실행 (Run from Command Line)

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

## 7. 에이전트 평가 (Evaluate Your Agent)

Agents CLI의 평가(Evaluation) 워크플로우는 데이터셋 정의, 에이전트 추론 실행(`eval generate`), 그리고 LLM-as-judge 기반 자동 채점(`eval grade`)으로 구성됩니다.

### 평가 디렉토리 구조 (`tests/eval/`)
- `tests/eval/datasets/`: 평가용 데이터셋 (JSON 형식)
- `tests/eval/eval_config.yaml`: 평가 메트릭 및 평가기 설정
- `tests/eval/response_quality.py`: LLM-as-judge 기반 응답 품질 채점 로직 (Score 1~5점)

---

### 1. 평가 데이터셋 수정
`tests/eval/datasets/basic-dataset.json` 파일을 편집하여 에이전트 테스트 케이스 및 기대 응답(reference)을 작성합니다:

```json
{
  "eval_cases": [
    {
      "eval_case_id": "weather_san_francisco",
      "prompt": {
        "role": "user",
        "parts": [{"text": "What's the weather like in San Francisco?"}]
      },
      "reference": {
        "response": {
          "role": "model",
          "parts": [{"text": "It's 60 degrees and foggy."}]
        }
      }
    },
    {
      "eval_case_id": "weather_tokyo",
      "prompt": {
        "role": "user",
        "parts": [{"text": "What's the weather in Tokyo?"}]
      },
      "reference": {
        "response": {
          "role": "model",
          "parts": [{"text": "It's 90 degrees and sunny."}]
        }
      }
    },
    {
      "eval_case_id": "time_san_francisco",
      "prompt": {
        "role": "user",
        "parts": [{"text": "What time is it in San Francisco?"}]
      }
    }
  ]
}
```

---

### 2. 평가 설정 확인 (`tests/eval/eval_config.yaml`)
`eval_config.yaml`은 평가 시 실행할 메트릭을 정의합니다:

```yaml
metrics_to_run:
  - custom_response_quality

custom_metrics:
  - name: custom_response_quality
    custom_function_file: response_quality.py
  - name: agent_turn_count
    custom_function: |
      def evaluate(instance):
          turns = (instance.get("agent_data") or {}).get("turns", [])
          return {'score': len(turns)}
```

---

### 3. 평가 실행 (Evaluation Run)

#### 🤖 Coding Agent 모드
> "Run the evaluations for my agent"

#### 👤 수동 (Manual) 모드

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

## 8. Agent Runtime 배포 설정 추가 (Add Agent Runtime Deployment)

Google Cloud의 서버리스 관리형 환경인 **Agent Runtime** 배포 아키텍처를 프로젝트에 반영합니다.

### 🤖 Coding Agent 모드
> "Add Agent Runtime deployment to my project"

### 👤 수동 (Manual) 모드
```bash
agents-cli scaffold enhance --deployment-target agent_runtime --yes
```

#### 주요 변경점
- **추가**: `app/agent_runtime_app.py`, `deployment_metadata.json`
- **삭제**: `Dockerfile`, `app/fast_api_app.py` (관리형 환경이므로 커스텀 Dockerfile 불필요)
- **보존**: `app/agent.py` (작성한 에이전트 코드는 변경 없이 그대로 유지)

---

## 9. Agent Runtime에 배포 (Deploy to Agent Runtime)

### 1. 패키지 의존성 잠금
```bash
uv lock
```

### 2. 배포 실행
`YOUR_PROJECT_ID`를 본인의 GCP 프로젝트 ID로 변경합니다.

- **🤖 Coding Agent**: `"Deploy my agent to Agent Runtime in project YOUR_PROJECT_ID, region us-central1"`
- **👤 수동 모드**:
  ```bash
  agents-cli deploy --project YOUR_PROJECT_ID --region us-central1
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

## 10. 배포된 에이전트 테스트 및 모니터링 (Test and Monitor Deployed Agent)

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

#### 옵션 3: Python SDK (`vertexai.agent_engines`)
```python
import vertexai
from vertexai import agent_engines

vertexai.init(project="YOUR_PROJECT_ID", location="us-central1")

# deployment_metadata.json의 remote_agent_runtime_id 값 사용
remote_agent = agent_engines.get(
    "projects/.../locations/us-central1/reasoningEngines/..."
)

session = remote_agent.create_session(user_id="user-1")
for event in remote_agent.stream_query(
    user_id="user-1",
    session_id=session["id"],
    message="What's the weather in San Francisco?",
):
    print(event)
```

### 모니터링 도구 활용
Agent Runtime은 Google Cloud의 모니터링 시스템과 자동으로 연동됩니다.

- **Cloud Trace**: [Trace Explorer](https://console.cloud.google.com/traces/list)에서 `service.name = customer-support-agent` 필터링 후 모델 추론, 도구 호출, 지연 시간(Latency) 분석
- **Cloud Logging**: [Logs Explorer](https://console.cloud.google.com/logs/query)에서 `resource.type="aiplatform.googleapis.com/ReasoningEngine"` 검색하여 에이전트 로그 확인
- **Cloud Monitoring**: [Metrics Explorer](https://console.cloud.google.com/monitoring/metrics-explorer)에서 `request_count`, `request_latencies`, `instance_count` 지표 모니터링

---

## 11. (선택) Gemini Enterprise에 게시 (Publish to Gemini Enterprise)

사내 구독 중인 Gemini Enterprise 환경에 에이전트를 마켓플레이스 형태로 게시할 수 있습니다.

### 1. 대상 앱 목록 조회
```bash
agents-cli publish gemini-enterprise --list
```

### 2. 에이전트 등록
```bash
agents-cli publish gemini-enterprise \
  --gemini-enterprise-app-id "projects/PROJECT_NUMBER/locations/global/collections/default_collection/engines/YOUR_APP_ID" \
  --display-name "Customer Support Agent" \
  --description "Answers weather and time questions" \
  --tool-description "Use this tool to ask the customer support agent."
```

---

## 12. 마침 (Congratulations)

🎉 **축하합니다!** Agents CLI와 ADK를 사용하여 에이전트 생성, 테스트, 평가 및 프로덕션 배포까지 모든 과정을 성공적으로 완료하셨습니다.

### 🚀 추가 학습 & 확장 방향
- **도구 추가**: 데이터베이스 쿼리, REST API 통신 도구 추가
- **RAG 연동**: 벡터 검색 기반 문서 기반 Q&A 에이전트 구현
- **A2A (Agent-to-Agent)**: 에이전트 간 협업 아키텍처 구축
- **공식 문서**: [Agents CLI Guide](https://google.github.io/agents-cli/) | [ADK Documentation](https://adk.dev/)

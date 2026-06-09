# 제10장 에이전트 통신 프로토콜

이전 장들에서 우리는 추론, 도구 호출, 메모리 기능을 갖춘 완전한 독립 실행형 에이전트를 구축했습니다. 그러나 더 복잡한 AI 시스템을 만들려고 하면 자연스럽게 다음과 같은 질문이 생깁니다. **에이전트는 외부 세계와 어떻게 효율적으로 상호작용할 수 있을까요? 여러 에이전트는 서로 어떻게 협력할 수 있을까요?**

이것이 바로 에이전트 통신 프로토콜이 해결하려는 핵심 문제입니다. 이 장에서는 HelloAgents 프레임워크에 세 가지 통신 프로토콜을 도입합니다. 에이전트와 도구 사이의 표준화된 통신을 위한 **MCP (Model Context Protocol)**, 에이전트 간 피어 투 피어 협업을 위한 **A2A (Agent-to-Agent Protocol)**, 대규모 에이전트 네트워크 구축을 위한 **ANP (Agent Network Protocol)**입니다. 이 세 프로토콜은 함께 에이전트 통신을 위한 인프라 계층을 형성합니다.

이 장의 학습을 통해 에이전트 통신 프로토콜의 설계 철학과 실무 기술을 익히고, 세 가지 주요 프로토콜의 설계 차이를 이해하며, 실제 문제를 해결하기 위해 적절한 프로토콜을 선택하는 방법을 배울 것입니다.

## 10.1 에이전트 통신 프로토콜 기초

### 10.1.1 통신 프로토콜이 필요한 이유

7장에서 구축한 ReAct 에이전트를 떠올려 봅시다. 이 에이전트는 이미 강력한 추론 및 도구 호출 능력을 갖추고 있습니다. 전형적인 사용 시나리오를 살펴보겠습니다.

```python
from hello_agents import ReActAgent, HelloAgentsLLM
from hello_agents.tools import CalculatorTool, SearchTool

llm = HelloAgentsLLM()
agent = ReActAgent(name="AI Assistant", llm=llm)
agent.add_tool(CalculatorTool())
agent.add_tool(SearchTool())

# 에이전트는 독립적으로 작업을 완료할 수 있음
response = agent.run("Search for the latest AI news and calculate the total market value of related companies")
```

이 에이전트는 잘 동작하지만, 세 가지 근본적인 한계에 직면합니다. 첫 번째는 **도구 통합의 딜레마**입니다.
GitHub API, 데이터베이스, 파일 시스템과 같은 새로운 외부 서비스에 접근해야 할 때마다 전용 Tool 클래스를 작성해야 합니다.
이는 노동 집약적일 뿐만 아니라 서로 다른 개발자가 작성한 도구가 서로 호환되지 않는 문제를 낳습니다.
두 번째는 **능력 확장의 병목**입니다.
에이전트의 능력은 미리 정의된 도구 집합에 제한되며,
새로운 서비스를 동적으로 발견하고 사용할 수 없습니다.
마지막은 **협업의 부재**입니다. 연구자 + 작성자 + 편집자처럼 여러 전문 에이전트가 협력해야 할 만큼 작업이 복잡해지면, 우리는 수동 오케스트레이션으로만 이들의 작업을 조율할 수 있습니다.

이러한 한계를 더 구체적인 예로 이해해 봅시다. 지능형 리서치 어시스턴트를 만들고 싶다고 가정해 보겠습니다. 이 어시스턴트는 다음이 필요합니다.

```python
# 전통적인 방식: 각 서비스를 수동으로 통합
class GitHubTool(BaseTool):
    """GitHub API 어댑터를 직접 작성해야 함"""
    def run(self, repo_url):
        # 많은 API 호출 코드...
        pass

class DatabaseTool(BaseTool):
    """데이터베이스 어댑터를 직접 작성해야 함"""
    def run(self, query):
        # 데이터베이스 연결 및 쿼리 코드...
        pass

class WeatherTool(BaseTool):
    """날씨 API 어댑터를 직접 작성해야 함"""
    def run(self, location):
        # 날씨 API 호출 코드...
        pass

# 새 서비스마다 이 과정을 반복해야 함
agent.add_tool(GitHubTool())
agent.add_tool(DatabaseTool())
agent.add_tool(WeatherTool())
```

이 방식에는 명확한 문제가 있습니다. 코드가 중복되고(각 도구가 HTTP 요청, 오류 처리, 인증 등을 처리해야 함), 유지보수가 어렵고(API 변경 시 관련 도구를 모두 수정해야 함), 재사용이 불가능하며(다른 개발자의 도구를 직접 사용할 수 없음), 확장성이 낮습니다(새 서비스를 추가하려면 많은 코딩 작업이 필요함).

**통신 프로토콜의 핵심 가치**는 바로 이러한 문제를 해결하는 데 있습니다.
통신 프로토콜은 표준화된 인터페이스 규격을 제공하여, 에이전트가 각 서비스마다 전용 어댑터를 작성하지 않고도 다양한 외부 서비스에 통일된 방식으로 접근할 수 있게 합니다.
이는 인터넷의 TCP/IP 프로토콜과 비슷합니다. TCP/IP 덕분에 서로 다른 장치들이 각 장치 유형마다 전용 통신 코드를 작성하지 않고도 서로 통신할 수 있습니다.

통신 프로토콜을 사용하면 위 코드는 다음과 같이 단순화됩니다.

```python
from hello_agents.tools import MCPTool

# MCP 서버에 연결하여 모든 도구를 자동으로 획득
mcp_tool = MCPTool()  # 내장 서버가 기본 도구를 제공

# 또는 전문 MCP 서버에 연결
github_mcp = MCPTool(server_command=["npx", "-y", "@modelcontextprotocol/server-github"])
database_mcp = MCPTool(server_command=["python", "database_mcp_server.py"])

# 에이전트는 어댑터를 직접 작성하지 않고도 모든 능력을 자동으로 획득
agent.add_tool(mcp_tool)
agent.add_tool(github_mcp)
agent.add_tool(database_mcp)
```

통신 프로토콜이 가져오는 변화는 근본적입니다. **표준화된 인터페이스**는 서로 다른 서비스가 통일된 접근 방식을 제공하도록 만들고, **상호 운용성**은 서로 다른 개발자의 도구가 매끄럽게 통합되도록 하며, **동적 발견**은 에이전트가 런타임에 새로운 서비스와 능력을 발견하게 하고, **확장성**은 시스템에 새로운 기능 모듈을 쉽게 추가할 수 있게 합니다.

### 10.1.2 세 가지 프로토콜 설계 철학 비교

에이전트 통신 프로토콜은 단일한 해법이 아니라 서로 다른 통신 시나리오를 위해 설계된 일련의 표준입니다. 이 장에서는 현재 주류로 여겨지는 세 가지 프로토콜인 MCP, A2A, ANP를 예로 들어 실습합니다. 먼저 개요를 비교해 보겠습니다.

**(1) MCP: 에이전트와 도구 사이의 다리**

MCP (Model Context Protocol)는 Anthropic 팀이 제안한 프로토콜<sup>[1]</sup>로, 핵심 설계 철학은 **에이전트와 외부 도구/리소스 사이의 통신 방식을 표준화하는 것**입니다.
여러분의 에이전트가 파일 시스템, 데이터베이스, GitHub, Slack 등 다양한 서비스에 접근해야 한다고 상상해 보세요.
전통적인 방식은 각 서비스마다 전용 어댑터를 작성하는 것인데, 이는 노동 집약적일 뿐만 아니라 유지보수도 어렵습니다.
MCP는 모든 서비스를 동일한 방식으로 접근할 수 있도록 통일된 프로토콜 규격을 정의합니다.

MCP의 설계 철학은 "컨텍스트 공유"입니다.
MCP는 단순한 RPC (Remote Procedure Call) 프로토콜이 아닙니다.
더 중요한 점은 에이전트와 도구가 풍부한 컨텍스트 정보를 공유할 수 있게 한다는 것입니다.
그림 10.1처럼 에이전트가 코드 저장소에 접근할 때, MCP 서버는 파일 내용뿐만 아니라 코드 구조, 의존성 관계, 커밋 기록 같은 컨텍스트 정보도 제공할 수 있습니다. 이를 통해 에이전트는 더 지능적인 의사결정을 내릴 수 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-1.png" alt="" width="85%"/>
  <p>그림 10.1 MCP 설계 철학</p>
</div>

**(2) A2A: 에이전트 간 대화**

A2A (Agent-to-Agent Protocol) 프로토콜은 Google 팀이 제안한 프로토콜<sup>2</sup>로, 핵심 설계 철학은 **에이전트 사이의 피어 투 피어 통신을 구현하는 것**입니다.
에이전트와 도구 사이의 통신에 초점을 맞추는 MCP와 달리, A2A는 에이전트들이 서로 어떻게 협력하는지에 초점을 맞춥니다.
이 설계는 에이전트들이 인간 팀처럼 대화하고, 협상하고, 협업할 수 있게 합니다.

A2A의 설계 철학은 "피어 투 피어 통신"입니다. 그림 10.2처럼 A2A 네트워크에서 각 에이전트는 서비스 제공자인 동시에 서비스 소비자입니다.
에이전트는 능동적으로 요청을 시작할 수도 있고 다른 에이전트의 요청에 응답할 수도 있습니다. 이러한 피어 투 피어 설계는 중앙 집중식 코디네이터의 병목을 피하게 하며, 에이전트 네트워크를 더 유연하고 확장 가능하게 만듭니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-2.png" alt="" width="85%"/>
  <p>그림 10.2 A2A 설계 철학</p>
</div>

**(3) ANP: 에이전트 네트워크를 위한 인프라**

ANP (Agent Network Protocol)는 개념적 프로토콜 프레임워크<sup>3</sup>입니다. 현재는 오픈소스 커뮤니티가 유지하고 있으며 아직 성숙한 생태계를 갖추지는 못했습니다.
핵심 설계 철학은 **대규모 에이전트 네트워크를 위한 인프라를 구축하는 것**입니다.
MCP가 "도구에 어떻게 접근할 것인가"를 해결하고 A2A가 "다른 에이전트와 어떻게 대화할 것인가"를 해결한다면,
ANP는 "대규모 네트워크에서 에이전트를 어떻게 발견하고 연결할 것인가"를 해결합니다.

ANP의 설계 철학은 "탈중앙화된 서비스 발견"입니다.
수백 또는 수천 개의 에이전트가 포함된 네트워크에서, 에이전트는 필요한 서비스를 어떻게 찾을 수 있을까요?
그림 10.3처럼 ANP는 서비스 등록, 발견, 라우팅 메커니즘을 제공하여, 에이전트가 모든 연결 관계를 미리 설정하지 않고도 네트워크 안의 다른 서비스를 동적으로 발견할 수 있게 합니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-3.png" alt="" width="85%"/>
  <p>그림 10.3 ANP 설계 철학</p>
</div>

마지막으로 표 10.1의 비교표를 통해 세 프로토콜의 차이를 더 명확히 이해해 봅시다.

<div align="center">
  <p>표 10.1 세 가지 프로토콜 비교</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-table-1.png" alt="" width="85%"/>
</div>

**(4) 적절한 프로토콜은 어떻게 선택할까?**

현재 프로토콜들은 아직 초기 개발 단계에 있습니다. MCP 생태계는 상대적으로 성숙했지만, 각 도구의 최신성은 유지보수자에게 달려 있습니다. 대기업이 뒷받침하는 MCP 도구를 선택하는 것을 더 권장합니다.

프로토콜 선택의 핵심은 자신의 요구사항을 이해하는 데 있습니다.

- 에이전트가 외부 서비스(파일, 데이터베이스, API)에 접근해야 한다면 **MCP**를 선택하세요.
- 여러 에이전트가 작업을 협업해야 한다면 **A2A**를 선택하세요.
- 대규모 에이전트 생태계를 구축하려면 **ANP**를 고려하세요.

### 10.1.3 HelloAgents 통신 프로토콜 아키텍처 설계

세 프로토콜의 설계 철학을 이해했으니, 이제 HelloAgents 프레임워크에서 이를 어떻게 구현하고 사용하는지 살펴보겠습니다.
우리의 설계 목표는 다음과 같습니다. **학습자가 가장 단순한 방식으로 이러한 프로토콜을 사용할 수 있게 하면서도, 복잡한 시나리오를 처리할 충분한 유연성을 유지하는 것**입니다.

그림 10.4처럼 HelloAgents 통신 프로토콜 아키텍처는 아래에서 위로 프로토콜 구현 계층, 도구 캡슐화 계층, 에이전트 통합 계층의 3계층 설계를 채택합니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-4.png" alt="" width="85%"/>
  <p>그림 10.4 HelloAgents 통신 프로토콜 설계</p>
</div>

**(1) 프로토콜 구현 계층**: 이 계층에는 세 프로토콜의 구체적인 구현이 포함됩니다.
MCP는 FastMCP 라이브러리를 기반으로 구현되어 클라이언트와 서버 기능을 제공합니다. A2A는 Google의 공식 a2a-sdk를 기반으로 구현됩니다. ANP는 우리가 자체 개발한 경량 구현으로,
서비스 발견과 네트워크 관리 기능을 제공합니다. 물론 현재 공식 [구현](https://github.com/agent-network-protocol/AgentConnect)도 있지만, 향후 반복 개발을 고려하여 여기서는 개념만 시뮬레이션합니다.

**(2) 도구 캡슐화 계층**: 이 계층은 프로토콜 구현을 통일된 Tool 인터페이스로 캡슐화합니다. MCPTool, A2ATool, ANPTool은 모두 BaseTool을 상속하며 일관된 `run()` 메서드를 제공합니다.
이 설계를 통해 에이전트는 서로 다른 프로토콜을 동일한 방식으로 사용할 수 있습니다.

**(3) 에이전트 통합 계층**: 이 계층은 에이전트와 프로토콜이 만나는 지점입니다. 모든 에이전트(ReActAgent, SimpleAgent 등)는 Tool System을 통해 프로토콜 도구를 사용하며, 하위 프로토콜 세부 사항을 신경 쓸 필요가 없습니다.

### 10.1.4 이 장의 학습 목표와 빠른 체험

먼저 10장의 학습 내용을 살펴보겠습니다.

```
hello_agents/
├── protocols/                          # 통신 프로토콜 모듈
│   ├── mcp/                            # MCP 프로토콜 구현 (Model Context Protocol)
│   │   ├── client.py                   # MCP 클라이언트 (5가지 전송 방식 지원)
│   │   ├── server.py                   # MCP 서버 (FastMCP 래퍼)
│   │   └── utils.py                    # 유틸리티 함수 (create_context/parse_context)
│   ├── a2a/                            # A2A 프로토콜 구현 (Agent-to-Agent Protocol)
│   │   └── implementation.py           # A2A 서버/클라이언트 (a2a-sdk 기반, 선택 의존성)
│   └── anp/                            # ANP 프로토콜 구현 (Agent Network Protocol)
│       └── implementation.py           # ANP 서비스 발견/등록 (개념적 구현)
└── tools/builtin/                      # 내장 도구 모듈
    └── protocol_tools.py               # 프로토콜 도구 래퍼 (MCPTool/A2ATool/ANPTool)
```

이 장의 내용은 주로 응용에 초점을 맞추며, 학습 목표는 자신의 프로젝트에 프로토콜을 적용할 수 있는 능력을 갖추는 것입니다.
또한 현재 프로토콜이 초기 개발 단계에 있으므로, 바퀴를 다시 발명하는 데 너무 많은 노력을 들일 필요는 없습니다.
실습을 시작하기 전에 개발 환경을 준비해 봅시다.

```bash
# HelloAgents 프레임워크 설치 (10장 버전)
pip install "hello-agents[protocol]==0.2.2"

# NodeJS 설치는 Additional-Chapter의 문서 참고
```

가장 단순한 코드로 세 프로토콜의 기본 기능을 체험해 보겠습니다.

```python
from hello_agents.tools import MCPTool, A2ATool, ANPTool

# 1. MCP: 도구 접근
mcp_tool = MCPTool()
result = mcp_tool.run({
    "action": "call_tool",
    "tool_name": "add",
    "arguments": {"a": 10, "b": 20}
})
print(f"MCP 계산 결과: {result}")  # 출력: 30.0

# 2. ANP: 서비스 발견
anp_tool = ANPTool()
anp_tool.run({
    "action": "register_service",
    "service_id": "calculator",
    "service_type": "math",
    "endpoint": "http://localhost:8080"
})
services = anp_tool.run({"action": "discover_services"})
print(f"발견된 서비스: {services}")

# 3. A2A: 에이전트 통신
a2a_tool = A2ATool("http://localhost:5000")
print("A2A 도구가 성공적으로 생성되었습니다")
```

이 간단한 예제는 세 프로토콜의 핵심 기능을 보여줍니다. 다음 섹션들에서는 각 프로토콜의 상세한 사용법과 모범 사례를 깊이 있게 학습하겠습니다.

## 10.2 MCP 프로토콜 실전

이제 MCP를 깊이 살펴보고, 에이전트가 외부 도구와 리소스에 접근하도록 만드는 방법을 익혀 봅시다.

### 10.2.1 MCP 프로토콜 개념 소개

**(1) MCP: 에이전트를 위한 "USB-C"**

여러분의 에이전트가 동시에 많은 일을 해야 한다고 상상해 보세요. 예를 들면 다음과 같습니다.

- 로컬 파일 시스템에서 문서 읽기
- PostgreSQL 데이터베이스 조회
- GitHub에서 코드 검색
- Slack 메시지 전송
- Google Drive 접근

전통적으로는 각 서비스마다 어댑터 코드를 작성하고, 서로 다른 API, 인증 방식, 오류 처리 등을 다루어야 했습니다. 이는 노동 집약적일 뿐만 아니라 유지보수도 어렵습니다. 더 중요한 점은 LLM 플랫폼마다 함수 호출 구현 방식이 크게 달라서, 모델을 바꿀 때 많은 코드를 다시 작성해야 한다는 것입니다.

MCP의 등장은 이 모든 것을 바꾸었습니다. USB-C가 다양한 장치의 연결 방식을 통일한 것처럼, **MCP는 에이전트와 외부 도구 사이의 상호작용 방식을 통일했습니다**. Claude, GPT 또는 다른 모델을 사용하더라도 MCP 프로토콜을 지원하기만 하면 동일한 도구와 리소스에 매끄럽게 접근할 수 있습니다.

**(2) MCP 아키텍처**

MCP 프로토콜은 Host, Client, Servers의 3계층 아키텍처 설계를 채택합니다. 그림 10.5의 시나리오를 통해 이 컴포넌트들이 어떻게 함께 동작하는지 이해해 봅시다.

Claude Desktop을 사용하면서 "내 데스크톱에는 어떤 문서가 있나요?"라고 질문한다고 가정해 보겠습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-5.png" alt="" width="85%"/>
  <p>그림 10.5 MCP 사례 시연</p>
</div>

**3계층 아키텍처의 책임:**

1. **Host (호스트 계층)**: Claude Desktop은 Host 역할을 하며, 사용자 질문을 받고 Claude 모델과 상호작용합니다. Host는 사용자가 직접 상호작용하는 인터페이스이며 전체 대화 흐름을 관리합니다.

2. **Client (클라이언트 계층)**: Claude 모델이 파일 시스템에 접근해야 한다고 판단하면, Host에 내장된 MCP Client가 활성화됩니다. Client는 적절한 MCP Server와 연결을 수립하고 요청을 보내며 응답을 받는 일을 담당합니다.

3. **Server (서버 계층)**: 파일 시스템 MCP Server가 호출되어 실제 파일 스캔 작업을 수행하고, 데스크톱 디렉터리에 접근한 뒤 발견된 문서 목록을 반환합니다.

**완전한 상호작용 흐름:** 사용자 질문 → Claude Desktop (Host) → Claude 모델 분석 → 파일 정보 필요 판단 → MCP Client 연결 → 파일 시스템 MCP Server → 작업 실행 → 결과 반환 → Claude가 답변 생성 → Claude Desktop에 표시

이 아키텍처 설계의 장점은 **관심사의 분리**에 있습니다. Host는 사용자 경험에 집중하고, Client는 프로토콜 통신에 집중하며, Server는 구체적인 기능 구현에 집중합니다. 개발자는 Host와 Client의 구현 세부 사항을 신경 쓰지 않고 해당 MCP Server 개발에만 집중하면 됩니다.

**(3) MCP의 핵심 능력**

표 10.2처럼 MCP 프로토콜은 세 가지 핵심 능력을 제공하며, 이를 통해 완전한 도구 접근 프레임워크를 구성합니다.

<div align="center">
  <p>표 10.2 MCP 핵심 능력</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-table-2.png" alt="" width="85%"/>
</div>

이 세 능력의 차이는 다음과 같습니다. **Tools는 능동적**이고(작업 실행), **Resources는 수동적**이며(데이터 제공), **Prompts는 지시적**입니다(템플릿 제공).

**(4) MCP 워크플로**

그림 10.6의 구체적인 예시를 통해 MCP의 전체 워크플로를 이해해 봅시다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-6.png" alt="" width="85%"/>
  <p>그림 10.6 MCP 사례 시연</p>
</div>

핵심 질문은 이것입니다. **Claude(또는 다른 LLM)는 어떤 도구를 사용할지 어떻게 결정할까요?**

사용자가 질문하면 전체 도구 선택 과정은 다음과 같습니다.

1. **도구 발견 단계**: MCP Client가 Server에 연결된 뒤 먼저 `list_tools()`를 호출하여 사용 가능한 모든 도구의 설명 정보(도구 이름, 기능 설명, 매개변수 정의)를 가져옵니다.

2. **컨텍스트 구축**: Client는 도구 목록을 LLM이 이해할 수 있는 형식으로 변환하고 시스템 프롬프트에 추가합니다. 예를 들면 다음과 같습니다.
   ```
   You can use the following tools:
   - read_file(path: str): Read the content of the file at the specified path
   - search_code(query: str, language: str): Search in the codebase
   ```

3. **모델 추론**: LLM은 사용자의 질문과 사용 가능한 도구를 분석하여 도구를 호출할지, 어떤 도구를 호출할지 결정합니다. 이 결정은 도구 설명과 현재 대화 컨텍스트를 기반으로 합니다.

4. **도구 실행**: LLM이 도구 사용을 결정하면 Client는 MCP Server를 통해 선택된 도구를 실행하고 결과를 얻습니다.

5. **결과 통합**: 도구 실행 결과가 LLM으로 다시 전달되고, LLM은 이 결과를 결합해 최종 답변을 생성합니다.

이 과정은 **완전히 자동화**되어 있으며, LLM은 도구 설명의 품질에 따라 도구 사용 여부와 사용 방식을 결정합니다. 따라서 명확하고 정확한 도구 설명을 작성하는 것이 매우 중요합니다.

**(5) MCP와 Function Calling의 차이**

많은 개발자가 묻습니다. **이미 Function Calling을 사용하고 있는데 왜 MCP가 또 필요할까요?** 표 10.3을 통해 차이를 이해해 봅시다.

<div align="center">
  <p>표 10.3 Function Calling과 MCP 비교</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-table-3.png" alt="" width="85%"/>
</div>

여기서는 에이전트가 GitHub 저장소와 로컬 파일 시스템에 접근해야 하는 예를 사용하여, 같은 작업을 두 가지 방식으로 구현하는 방법을 자세히 비교합니다.

**방법 1: Function Calling 사용**

```python
# 1단계: 각 LLM 제공자에 맞는 함수 정의
# OpenAI 형식
openai_tools = [
    {
        "type": "function",
        "function": {
            "name": "search_github",
            "description": "Search GitHub repositories",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Search keywords"}
                },
                "required": ["query"]
            }
        }
    }
]

# Claude 형식
claude_tools = [
    {
        "name": "search_github",
        "description": "Search GitHub repositories",
        "input_schema": {  # 주의: parameters가 아님
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search keywords"}
            },
            "required": ["query"]
        }
    }
]

# 2단계: 도구 함수를 직접 구현
def search_github(query):
    import requests
    response = requests.get(
        "https://api.github.com/search/repositories",
        params={"q": query}
    )
    return response.json()

# 3단계: 서로 다른 모델 응답 형식 처리
# OpenAI 응답
if response.choices[0].message.tool_calls:
    tool_call = response.choices[0].message.tool_calls[0]
    result = search_github(**json.loads(tool_call.function.arguments))

# Claude 응답
if response.content[0].type == "tool_use":
    tool_use = response.content[0]
    result = search_github(**tool_use.input)
```

**방법 2: MCP 사용**

```python
from hello_agents.protocols import MCPClient

# 1단계: 커뮤니티가 제공하는 MCP 서버에 연결 (직접 구현할 필요 없음)
github_client = MCPClient([
    "npx", "-y", "@modelcontextprotocol/server-github"
])

fs_client = MCPClient([
    "npx", "-y", "@modelcontextprotocol/server-filesystem", "."
])

# 2단계: 통일된 호출 방식 (모델 독립적)
async with github_client:
    # 도구 자동 발견
    tools = await github_client.list_tools()

    # 도구 호출 (표준화된 인터페이스)
    result = await github_client.call_tool(
        "search_repositories",
        {"query": "AI agents"}
    )

# 3단계: MCP를 지원하는 모든 모델이 사용할 수 있음
# OpenAI, Claude, Llama 등이 모두 같은 MCP 클라이언트를 사용
```

먼저 명확히 해야 할 점은 Function Calling과 MCP가 경쟁 관계가 아니라 상호 보완적이라는 것입니다. Function Calling은 대형 언어 모델의 핵심 능력으로, 모델이 언제 함수를 호출해야 하는지 이해하고 해당 호출 매개변수를 정확히 생성할 수 있게 하는 모델 고유의 지능을 반영합니다. 반면 MCP는 인프라 프로토콜의 역할을 하며, 도구가 모델과 어떻게 연결되는지를 공학적 수준에서 해결하고, 도구를 표준화된 방식으로 설명하고 호출합니다.

간단한 비유로 이해해 볼 수 있습니다. Function Calling은 "전화 거는 방법"이라는 기술을 배우는 것과 같습니다. 언제 전화를 걸고, 상대와 어떻게 소통하며, 언제 끊을지 포함합니다. 반면 MCP는 전 세계적으로 통일된 "전화 통신 표준"으로, 어떤 전화기든 다른 전화기에 성공적으로 전화를 걸 수 있도록 보장합니다.

이 상호 보완 관계를 이해했으니, 이제 HelloAgents에서 MCP 프로토콜을 사용하는 방법을 살펴보겠습니다.

### 10.2.2 MCP Client 사용

HelloAgents는 FastMCP 2.0을 기반으로 완전한 MCP 클라이언트 기능을 구현합니다. 우리는 서로 다른 사용 시나리오에 맞게 비동기 API와 동기 API를 모두 제공합니다. 대부분의 애플리케이션에서는 동시 요청과 장시간 실행 작업을 더 잘 처리할 수 있는 비동기 API를 권장합니다. 아래에서는 단계별 작업 시연을 제공합니다.

**(1) MCP Server에 연결하기**

MCP 클라이언트는 여러 연결 방식을 지원하며, 가장 일반적인 방식은 Stdio 모드(표준 입력/출력을 통한 로컬 프로세스와의 통신)입니다.

```python
import asyncio
from hello_agents.protocols import MCPClient

async def connect_to_server():
    # 방법 1: 커뮤니티 제공 파일 시스템 서버에 연결
    # npx는 @modelcontextprotocol/server-filesystem 패키지를 자동으로 다운로드하고 실행
    client = MCPClient([
        "npx", "-y",
        "@modelcontextprotocol/server-filesystem",
        "."  # 루트 디렉터리 지정
    ])

    # async with를 사용해 연결이 올바르게 종료되도록 보장
    async with client:
        # 여기서 클라이언트 사용
        tools = await client.list_tools()
        print(f"사용 가능한 도구: {[t['name'] for t in tools]}")

    # 방법 2: 사용자 정의 Python MCP 서버에 연결
    client = MCPClient(["python", "my_mcp_server.py"])
    async with client:
        # 클라이언트 사용...
        pass

# 비동기 함수 실행
asyncio.run(connect_to_server())
```

**(2) 사용 가능한 도구 발견하기**

연결에 성공하면 보통 첫 단계는 서버가 어떤 도구를 제공하는지 조회하는 것입니다.

```python
async def discover_tools():
    client = MCPClient(["npx", "-y", "@modelcontextprotocol/server-filesystem", "."])

    async with client:
        # 사용 가능한 모든 도구 가져오기
        tools = await client.list_tools()

        print(f"서버가 {len(tools)}개의 도구를 제공합니다:")
        for tool in tools:
            print(f"\n도구 이름: {tool['name']}")
            print(f"설명: {tool.get('description', '설명 없음')}")

            # 매개변수 정보 출력
            if 'inputSchema' in tool:
                schema = tool['inputSchema']
                if 'properties' in schema:
                    print("매개변수:")
                    for param_name, param_info in schema['properties'].items():
                        param_type = param_info.get('type', 'any')
                        param_desc = param_info.get('description', '')
                        print(f"  - {param_name} ({param_type}): {param_desc}")

asyncio.run(discover_tools())

# 출력 예시:
# 서버가 5개의 도구를 제공합니다:
#
# 도구 이름: read_file
# 설명: 파일 내용 읽기
# 매개변수:
#   - path (string): 파일 경로
#
# 도구 이름: write_file
# 설명: 파일 내용 쓰기
# 매개변수:
#   - path (string): 파일 경로
#   - content (string): 파일 내용
```

**(3) 도구 호출하기**

도구를 호출할 때는 도구 이름과 JSON Schema에 맞는 매개변수만 제공하면 됩니다.

```python
async def use_tools():
    client = MCPClient(["npx", "-y", "@modelcontextprotocol/server-filesystem", "."])

    async with client:
        # 파일 읽기
        result = await client.call_tool("read_file", {"path": "my_README.md"})
        print(f"파일 내용:\n{result}")

        # 디렉터리 나열
        result = await client.call_tool("list_directory", {"path": "."})
        print(f"현재 디렉터리 파일: {result}")

        # 파일 쓰기
        result = await client.call_tool("write_file", {
            "path": "output.txt",
            "content": "Hello from MCP!"
        })
        print(f"쓰기 결과: {result}")

asyncio.run(use_tools())
```

참고용으로 MCP 서비스를 더 안전하게 호출하는 방식은 다음과 같습니다.

```python
async def safe_tool_call():
    client = MCPClient(["npx", "-y", "@modelcontextprotocol/server-filesystem", "."])

    async with client:
        try:
            # 존재하지 않을 수 있는 파일 읽기 시도
            result = await client.call_tool("read_file", {"path": "nonexistent.txt"})
            print(result)
        except Exception as e:
            print(f"도구 호출 실패: {e}")
            # 재시도하거나 기본값을 사용하거나 사용자에게 오류를 보고할 수 있음

asyncio.run(safe_tool_call())
```

**(4) 리소스 접근하기**

도구 외에도 MCP 서버는 리소스를 제공할 수 있습니다.

```python
# 사용 가능한 리소스 나열
resources = client.list_resources()
print(f"사용 가능한 리소스: {[r['uri'] for r in resources]}")

# 리소스 읽기
resource_content = client.read_resource("file:///path/to/resource")
print(f"리소스 내용: {resource_content}")
```

**(5) 프롬프트 템플릿 사용하기**

MCP 서버는 미리 정의된 프롬프트 템플릿을 제공할 수 있습니다.

```python
# 사용 가능한 프롬프트 나열
prompts = client.list_prompts()
print(f"사용 가능한 프롬프트: {[p['name'] for p in prompts]}")

# 프롬프트 내용 가져오기
prompt = client.get_prompt("code_review", {"language": "python"})
print(f"프롬프트 내용: {prompt}")
```

**(6) 완전한 예제: GitHub MCP 서비스 사용**

캡슐화된 MCP Tools를 사용하여 커뮤니티가 제공하는 GitHub MCP 서비스를 사용하는 방법을 완전한 예제로 살펴보겠습니다.

```python
"""
GitHub MCP 서비스 예제

주의: 환경 변수를 설정해야 함
    Windows: $env:GITHUB_PERSONAL_ACCESS_TOKEN="your_token_here"
    Linux/macOS: export GITHUB_PERSONAL_ACCESS_TOKEN="your_token_here"
"""

from hello_agents.tools import MCPTool

# GitHub MCP 도구 생성
github_tool = MCPTool(
    server_command=["npx", "-y", "@modelcontextprotocol/server-github"]
)

# 1. 사용 가능한 도구 나열
print("📋 사용 가능한 도구:")
result = github_tool.run({"action": "list_tools"})
print(result)

# 2. 저장소 검색
print("\n🔍 저장소 검색:")
result = github_tool.run({
    "action": "call_tool",
    "tool_name": "search_repositories",
    "arguments": {
        "query": "AI agents language:python",
        "page": 1,
        "perPage": 3
    }
})
print(result)

```

### 10.2.3 MCP 전송 방식 설명

MCP 프로토콜의 중요한 특징 중 하나는 **전송 방식에 독립적**이라는 점입니다. 이는 MCP 프로토콜 자체가 특정 전송 방식에 의존하지 않으며, 서로 다른 통신 채널 위에서 실행될 수 있음을 의미합니다. HelloAgents는 FastMCP 2.0을 기반으로 완전한 전송 방식 지원을 제공하므로, 실제 시나리오에 따라 가장 적절한 전송 모드를 선택할 수 있습니다.

**(1) 전송 방식 개요**

HelloAgents의 `MCPClient`는 다섯 가지 전송 방식을 지원하며, 각 방식은 서로 다른 사용 사례를 갖습니다. 표 10.4에 나와 있습니다.

<div align="center">
  <p>표 10.4 MCP 전송 방식 비교</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-table-4.png" alt="" width="85%"/>
</div>

**(2) 전송 방식 사용 예**

```python
from hello_agents.tools import MCPTool

# 1. Memory Transport - 메모리 전송 (테스트용)
# 매개변수를 지정하지 않으면 내장 데모 서버 사용
mcp_tool = MCPTool()

# 2. Stdio Transport - 표준 입력/출력 전송 (로컬 개발)
# 명령 목록을 사용해 로컬 서버 시작
mcp_tool = MCPTool(server_command=["python", "examples/mcp_example_server.py"])

# 3. Stdio Transport with Args - 매개변수를 포함한 명령 전송
# 추가 매개변수 전달 가능
mcp_tool = MCPTool(server_command=["python", "examples/mcp_example_server.py", "--debug"])

# 4. Stdio Transport - 커뮤니티 서버 (npx 방식)
# npx를 사용해 커뮤니티 MCP 서버 시작
mcp_tool = MCPTool(server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."])

# 5. HTTP/SSE/StreamableHTTP Transport
# 주의: MCPTool은 주로 Stdio 및 Memory 전송을 위한 것
# HTTP/SSE 등 원격 전송에는 MCPClient를 직접 사용하는 것을 권장
```

**(3) Memory Transport**

사용 사례: 단위 테스트, 빠른 프로토타이핑

```python
from hello_agents.tools import MCPTool

# 내장 데모 서버 사용 (Memory 전송)
mcp_tool = MCPTool()

# 사용 가능한 도구 나열
result = mcp_tool.run({"action": "list_tools"})
print(result)

# 도구 호출
result = mcp_tool.run({
    "action": "call_tool",
    "tool_name": "add",
    "arguments": {"a": 10, "b": 20}
})
print(result)
```

**(4) Stdio Transport - 표준 입력/출력 전송**

사용 사례: 로컬 개발, 디버깅, Python 스크립트 서버

```python
from hello_agents.tools import MCPTool

# 방법 1: 사용자 정의 Python 서버 사용
mcp_tool = MCPTool(server_command=["python", "my_mcp_server.py"])

# 방법 2: 커뮤니티 서버 사용 (파일 시스템)
mcp_tool = MCPTool(server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."])

# 도구 나열
result = mcp_tool.run({"action": "list_tools"})
print(result)

# 도구 호출
result = mcp_tool.run({
    "action": "call_tool",
    "tool_name": "read_file",
    "arguments": {"path": "README.md"}
})
print(result)
```

**(5) HTTP Transport**

사용 사례: 프로덕션 환경, 원격 서비스, 마이크로서비스 아키텍처

```python
# 주의: MCPTool은 주로 Stdio 및 Memory 전송을 위한 것
# HTTP/SSE 등 원격 전송에는 하위 MCPClient 사용을 권장

import asyncio
from hello_agents.protocols import MCPClient

async def test_http_transport():
    # 원격 HTTP MCP 서버에 연결
    client = MCPClient("http://api.example.com/mcp")

    async with client:
        # 서버 정보 가져오기
        tools = await client.list_tools()
        print(f"원격 서버 도구: {len(tools)}개 도구")

        # 원격 도구 호출
        result = await client.call_tool("process_data", {
            "data": "Hello, World!",
            "operation": "uppercase"
        })
        print(f"원격 처리 결과: {result}")

# 주의: 실제 HTTP MCP 서버가 필요함
# asyncio.run(test_http_transport())
```

**(6) SSE Transport - Server-Sent Events 전송**

사용 사례: 실시간 통신, 스트리밍 처리, 장기 연결

```python
# 주의: MCPTool은 주로 Stdio 및 Memory 전송을 위한 것
# SSE 전송에는 하위 MCPClient 사용을 권장

import asyncio
from hello_agents.protocols import MCPClient

async def test_sse_transport():
    # SSE MCP 서버에 연결
    client = MCPClient(
        "http://localhost:8080/sse",
        transport_type="sse"
    )

    async with client:
        # SSE는 특히 스트리밍 처리에 적합
        result = await client.call_tool("stream_process", {
            "input": "Large data processing request",
            "stream": True
        })
        print(f"스트리밍 처리 결과: {result}")

# 주의: SSE를 지원하는 MCP 서버가 필요함
# asyncio.run(test_sse_transport())
```

**(7) StreamableHTTP Transport - 스트리밍 HTTP 전송**

사용 사례: 양방향 스트리밍 통신이 필요한 HTTP 시나리오

```python
# 주의: MCPTool은 주로 Stdio 및 Memory 전송을 위한 것
# StreamableHTTP 전송에는 하위 MCPClient 사용을 권장

import asyncio
from hello_agents.protocols import MCPClient

async def test_streamable_http_transport():
    # StreamableHTTP MCP 서버에 연결
    client = MCPClient(
        "http://localhost:8080/mcp",
        transport_type="streamable_http"
    )

    async with client:
        # 양방향 스트리밍 통신 지원
        tools = await client.list_tools()
        print(f"StreamableHTTP 서버 도구: {len(tools)}개 도구")

# 주의: StreamableHTTP를 지원하는 MCP 서버가 필요함
# asyncio.run(test_streamable_http_transport())
```

### 10.2.4 에이전트에서 MCP Tools 사용하기

앞에서는 MCP 클라이언트를 직접 사용하는 방법을 배웠습니다. 하지만 실제 애플리케이션에서는 호출 코드를 수동으로 작성하기보다 에이전트가 MCP 도구를 **자동으로** 호출하도록 만드는 편이 좋습니다. HelloAgents는 `MCPTool` 래퍼를 제공하여 MCP 서버를 에이전트의 도구 체인에 매끄럽게 통합할 수 있게 합니다.

**(1) MCP 도구의 자동 확장 메커니즘**

HelloAgents의 `MCPTool`에는 **자동 확장**이라는 기능이 있습니다. MCP 도구를 Agent에 추가하면, MCP 서버가 제공하는 모든 도구가 자동으로 독립 도구로 확장되어 Agent가 일반 도구처럼 호출할 수 있습니다.

**방법 1: 내장 데모 서버 사용**

앞서 구현했던 계산기 도구 함수를 여기서는 MCP 서비스로 변환합니다. 이것이 가장 단순한 사용 방식입니다.

```python
from hello_agents import SimpleAgent, HelloAgentsLLM
from hello_agents.tools import MCPTool

agent = SimpleAgent(name="Assistant", llm=HelloAgentsLLM())

# 설정 없이 내장 데모 서버를 자동 사용
mcp_tool = MCPTool(name="calculator")
agent.add_tool(mcp_tool)
# ✅ MCP 도구 'calculator'가 6개의 독립 도구로 확장됨

# 에이전트가 확장된 도구를 직접 사용할 수 있음
response = agent.run("Calculate 25 times 16")
print(response)  # 출력: The result of 25 times 16 is 400
```

**자동 확장 후의 도구**:

- `calculator_add` - 덧셈 계산기
- `calculator_subtract` - 뺄셈 계산기
- `calculator_multiply` - 곱셈 계산기
- `calculator_divide` - 나눗셈 계산기
- `calculator_greet` - 친근한 인사
- `calculator_get_system_info` - 시스템 정보 가져오기

Agent가 호출할 때는 예를 들어 `[TOOL_CALL:calculator_multiply:a=25,b=16]`처럼 매개변수만 제공하면 되고, 시스템이 타입 변환과 MCP 호출을 자동으로 처리합니다.

**방법 2: 외부 MCP 서버에 연결하기**

실제 프로젝트에서는 더 강력한 MCP 서버에 연결해야 합니다. 이러한 서버는 다음과 같을 수 있습니다.

- **커뮤니티가 제공하는 공식 서버**(예: 파일 시스템, GitHub, 데이터베이스 등)
- **직접 작성한 사용자 정의 서버**(비즈니스 로직 캡슐화)

```python
from hello_agents import SimpleAgent, HelloAgentsLLM
from hello_agents.tools import MCPTool

agent = SimpleAgent(name="File Assistant", llm=HelloAgentsLLM())

# 예제 1: 커뮤니티 제공 파일 시스템 서버에 연결
fs_tool = MCPTool(
    name="filesystem",  # 고유 이름 지정
    description="Access local file system",
    server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."]
)
agent.add_tool(fs_tool)

# 예제 2: 사용자 정의 Python MCP 서버에 연결
# 사용자 정의 MCP 서버 작성 방법은 10.5절 참고
custom_tool = MCPTool(
    name="custom_server",  # 다른 이름 사용
    description="Custom business logic server",
    server_command=["python", "my_mcp_server.py"]
)
agent.add_tool(custom_tool)

# 이제 에이전트가 이 도구들을 자동으로 사용할 수 있음!
response = agent.run("Please read the my_README.md file and summarize its main content")
print(response)
```

여러 MCP 서버를 사용할 때는 각 MCPTool에 서로 다른 이름을 반드시 지정하세요. 이 이름은 확장된 도구 이름의 접두사로 추가되어 충돌을 방지합니다. 예를 들어 `name="fs"`는 `fs_read_file`, `fs_write_file` 등으로 확장됩니다. 특정 비즈니스 로직을 캡슐화하기 위해 직접 MCP 서버를 작성해야 한다면 10.5절을 참고하세요.

**(2) MCP 도구 자동 확장의 동작 방식**

자동 확장 메커니즘을 이해하면 MCP 도구를 더 잘 사용할 수 있습니다. 내부 동작을 자세히 살펴보겠습니다.

```python
# 사용자 코드
fs_tool = MCPTool(name="fs", server_command=[...])
agent.add_tool(fs_tool)

# 내부에서 일어나는 일:
# 1. MCPTool이 서버에 연결하고 14개의 도구를 발견
# 2. 각 도구에 대한 래퍼 생성:
#    - fs_read_text_file (매개변수: path, tail, head)
#    - fs_write_file (매개변수: path, content)
#    - ...
# 3. Agent의 도구 레지스트리에 등록

# Agent 호출
response = agent.run("Read README.md")

# Agent 내부:
# 1. fs_read_text_file 호출이 필요하다고 식별
# 2. 매개변수 생성: path=README.md
# 3. 래퍼가 MCP 형식으로 변환:
#    {"action": "call_tool", "tool_name": "read_text_file", "arguments": {"path": "README.md"}}
# 4. MCP 서버 호출
# 5. 파일 내용 반환
```

시스템은 도구 매개변수 정의를 기반으로 타입을 자동 변환합니다.

```python
# Agent가 계산기 호출
agent.run("Calculate 25 times 16")

# Agent가 생성: a=25,b=16 (문자열)
# 시스템이 자동 변환: {"a": 25.0, "b": 16.0} (숫자)
# MCP 서버가 올바른 숫자 타입을 수신
```

**(3) 실전 사례: 지능형 문서 어시스턴트**

완전한 지능형 문서 어시스턴트를 만들어 봅시다. 여기서는 간단한 다중 에이전트 오케스트레이션으로 시연합니다.

```python
"""
다중 에이전트 협업 지능형 문서 어시스턴트

두 개의 SimpleAgent를 사용해 역할을 분담:
- Agent1: GitHub 검색 전문가
- Agent2: 문서 생성 전문가
"""
from hello_agents import SimpleAgent, HelloAgentsLLM
from hello_agents.tools import MCPTool
from dotenv import load_dotenv

# .env 파일에서 환경 변수 로드
load_dotenv(dotenv_path="../HelloAgents/.env")

print("="*70)
print("다중 에이전트 협업 지능형 문서 어시스턴트")
print("="*70)

# ============================================================
# Agent 1: GitHub Search Expert
# ============================================================
print("\n[1단계] GitHub 검색 전문가 생성 중...")

github_searcher = SimpleAgent(
    name="GitHub Search Expert",
    llm=HelloAgentsLLM(),
    system_prompt="""You are a GitHub search expert.
Your task is to search GitHub repositories and return results.
Please return clear, structured search results, including:
- Repository name
- Brief description

Keep it concise, don't add extra explanations."""
)

# GitHub 도구 추가
github_tool = MCPTool(
    name="gh",
    server_command=["npx", "-y", "@modelcontextprotocol/server-github"]
)
github_searcher.add_tool(github_tool)

# ============================================================
# Agent 2: Document Generation Expert
# ============================================================
print("\n[2단계] 문서 생성 전문가 생성 중...")

document_writer = SimpleAgent(
    name="Document Generation Expert",
    llm=HelloAgentsLLM(),
    system_prompt="""You are a document generation expert.
Your task is to generate structured Markdown reports based on provided information.

The report should include:
- Title
- Introduction
- Main content (listed in points, including project names, descriptions, etc.)
- Summary

Please output the complete Markdown format report content directly, do not use tools to save."""
)

# 파일 시스템 도구 추가
fs_tool = MCPTool(
    name="fs",
    server_command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "."]
)
document_writer.add_tool(fs_tool)

# ============================================================
# 작업 실행
# ============================================================
print("\n" + "="*70)
print("작업 실행 시작...")
print("="*70)

try:
    # 1단계: GitHub 검색
    print("\n[3단계] Agent1이 GitHub 검색 중...")
    search_task = "Search for GitHub repositories about 'AI agent', return the top 5 most relevant results"

    search_results = github_searcher.run(search_task)

    print("\n검색 결과:")
    print("-" * 70)
    print(search_results)
    print("-" * 70)

    # 2단계: 보고서 생성
    print("\n[4단계] Agent2가 보고서 생성 중...")
    report_task = f"""
Based on the following GitHub search results, generate a Markdown format research report:

{search_results}

Report requirements:
1. Title: # AI Agent Framework Research Report
2. Introduction: Explain this is a GitHub project survey about AI Agents
3. Main findings: List found projects and their features (including names, descriptions, etc.)
4. Summary: Summarize common characteristics of these projects

Please output the complete Markdown format report directly.
"""

    report_content = document_writer.run(report_task)

    print("\n보고서 내용:")
    print("=" * 70)
    print(report_content)
    print("=" * 70)

    # 3단계: 보고서 저장
    print("\n[5단계] 보고서를 파일로 저장 중...")
    import os
    try:
        with open("report.md", "w", encoding="utf-8") as f:
            f.write(report_content)
        print("✅ 보고서가 report.md에 저장되었습니다")

        # 파일 검증
        file_size = os.path.getsize("report.md")
        print(f"✅ 파일 크기: {file_size} bytes")
    except Exception as e:
        print(f"❌ 저장 실패: {e}")

    print("\n" + "="*70)
    print("작업 완료!")
    print("="*70)

except Exception as e:
    print(f"\n❌ 오류: {e}")
    import traceback
    traceback.print_exc()

```

이 과정에서 `github_searcher`는 `gh_search_repositories`를 호출하여 GitHub 프로젝트를 검색합니다. 얻은 결과는 `document_writer`의 입력으로 전달되어 보고서 생성을 더 안내하고, 최종적으로 보고서를 report.md에 저장합니다.

### 10.2.5 MCP 커뮤니티 생태계

MCP 프로토콜의 큰 장점은 **풍부한 커뮤니티 생태계**입니다. Anthropic과 커뮤니티 개발자들은 파일 시스템, 데이터베이스, API 서비스 등 다양한 시나리오를 포괄하는 수많은 기성 MCP 서버를 만들었습니다. 이는 도구 어댑터를 처음부터 작성할 필요 없이 검증된 서버를 바로 사용할 수 있음을 의미합니다.

MCP 커뮤니티를 위한 세 가지 리소스 저장소는 다음과 같습니다.

1. **Awesome MCP Servers** (https://github.com/punkpeye/awesome-mcp-servers)
   - 커뮤니티가 유지보수하는 엄선된 MCP 서버 목록
   - 다양한 서드파티 서버 포함
   - 기능별로 분류되어 찾기 쉬움

2. **MCP Servers Website** (https://mcpservers.org/)
   - 공식 MCP 서버 디렉터리 웹사이트
   - 검색 및 필터링 기능 제공
   - 사용 설명과 예제 포함

3. **Official MCP Servers** (https://github.com/modelcontextprotocol/servers)
   - Anthropic이 공식 유지보수하는 서버
   - 가장 높은 품질과 가장 완전한 문서
   - 자주 사용되는 서비스 구현 포함

표 10.5와 10.6은 자주 사용되는 공식 MCP 서버와 인기 있는 커뮤니티 MCP 서버를 보여줍니다.

<div align="center">
  <p>표 10.5 자주 사용되는 공식 MCP 서버</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-table-5.png" alt="" width="85%"/>
</div>

<div align="center">
  <p>표 10.6 인기 있는 커뮤니티 MCP 서버</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-table-6.png" alt="" width="85%"/>
</div>

참고할 만한 특히 흥미로운 사례 아이디어는 다음과 같습니다.

1. **자동화된 웹 테스트 (Playwright)**

   ```python
   # 에이전트가 자동으로 수행 가능:
   # - 브라우저를 열어 웹사이트 방문
   # - 폼 작성 및 제출
   # - 스크린샷으로 결과 검증
   # - 테스트 보고서 생성
   playwright_tool = MCPTool(
       name="playwright",
       server_command=["npx", "-y", "@playwright/mcp"]
   )
   ```

2. **지능형 노트 어시스턴트 (Obsidian + Perplexity)**
   ```python
   # 에이전트가 수행 가능:
   # - 최신 기술 뉴스 검색 (Perplexity)
   # - 구조화된 노트로 정리
   # - Obsidian 지식 베이스에 저장
   # - 노트 사이의 링크 자동 구축
   ```

3. **프로젝트 관리 자동화 (Jira + GitHub)**
   ```python
   # 에이전트가 수행 가능:
   # - GitHub Issues에서 Jira 작업 생성
   # - 코드 커밋을 Jira와 동기화
   # - Sprint 진행 상황 자동 업데이트
   # - 프로젝트 보고서 생성
   ```

5. **콘텐츠 제작 워크플로 (YouTube + Notion + Spotify)**

   ```python
   # 에이전트가 수행 가능:
   # - YouTube 동영상 자막 가져오기
   # - 콘텐츠 요약 생성
   # - Notion 데이터베이스에 저장
   # - 배경 음악 재생 (Spotify)
   ```

이 섹션의 설명을 통해 더 많은 MCP 구현 사례를 탐색해 보시기 바랍니다. HelloAgents에 대한 기여도 환영합니다! 다음으로 A2A 프로토콜을 학습하겠습니다.

## 10.3 A2A 프로토콜 실전

A2A (Agent-to-Agent)는 에이전트 간 직접 통신과 협업을 지원하는 프로토콜입니다.

### 10.3.1 프로토콜 설계 동기

MCP 프로토콜은 에이전트와 도구 사이의 상호작용을 해결했고, A2A 프로토콜은 에이전트 간 협업 문제를 해결합니다. 연구자, 작성자, 편집자처럼 다중 에이전트 협업이 필요한 작업에서는 에이전트들이 소통하고, 작업을 위임하고, 능력을 협상하며, 상태를 동기화해야 합니다.

전통적인 중앙 코디네이터(스타 토폴로지) 솔루션에는 세 가지 주요 문제가 있습니다.

- **단일 장애 지점**: 코디네이터 장애가 전체 시스템 마비로 이어집니다.
- **성능 병목**: 모든 통신이 중앙 노드를 거치므로 동시성이 제한됩니다.
- **확장 어려움**: 에이전트를 추가하거나 수정하려면 중앙 로직을 변경해야 합니다.

A2A 프로토콜은 피어 투 피어(P2P) 아키텍처(메시 토폴로지)를 채택하여 에이전트들이 직접 통신하게 함으로써 위 문제를 근본적으로 해결합니다. 핵심은 **Task**와 **Artifact**라는 두 추상 개념이며, 이것이 MCP와의 가장 큰 차이입니다. 표 10.7에 나와 있습니다.

<div align="center">
  <p>표 10.7 A2A 핵심 개념</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-table-7.png" alt="" width="85%"/>
</div>

협업 프로세스를 관리하기 위해 A2A는 작업을 위한 표준화된 생명주기를 정의합니다. 여기에는 생성, 협상, 위임, 진행 중, 완료, 실패 같은 상태가 포함되며, 그림 10.7에 나와 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-7.png" alt="" width="85%"/>
  <p>그림 10.7 A2A 작업 생명주기</p>
</div>

이 메커니즘은 에이전트들이 작업 협상, 진행 상황 추적, 예외 처리를 수행할 수 있게 합니다.

A2A 요청 생명주기는 에이전트 발견, 인증, send message API, send message stream API라는 네 가지 주요 단계를 상세히 보여주는 시퀀스입니다. 아래 그림 10.8은 공식 웹사이트의 흐름도를 차용한 것으로, 클라이언트, A2A 서버, 인증 서버 사이의 상호작용을 보여줍니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-8.png" alt="" width="85%"/>
  <p>그림 10.8 A2A 요청 생명주기</p>
</div>

### 10.3.2 A2A 프로토콜 실전

기존 A2A 구현의 대부분은 `Sample Code`이며, Python 구현조차 꽤 번거롭습니다. 따라서 여기서는 A2A-SDK를 통해 일부 기능을 구현하면서 프로토콜의 아이디어를 시뮬레이션하는 방식을 채택합니다.

**(2) 간단한 A2A Agent 만들기**

다시 계산기 사례를 시연으로 사용하여 A2A 에이전트를 만들어 봅시다.

```python
from hello_agents.protocols.a2a.implementation import A2AServer, A2A_AVAILABLE

def create_calculator_agent():
    """계산기 에이전트 생성"""
    if not A2A_AVAILABLE:
        print("❌ A2A SDK가 설치되어 있지 않습니다. 다음을 실행하세요: pip install a2a-sdk")
        return None

    print("🧮 계산기 에이전트 생성 중")

    # A2A 서버 생성
    calculator = A2AServer(
        name="calculator-agent",
        description="Professional mathematical calculation agent",
        version="1.0.0",
        capabilities={
            "math": ["addition", "subtraction", "multiplication", "division"],
            "advanced": ["power", "sqrt", "factorial"]
        }
    )

    # 기본 계산 스킬 추가
    @calculator.skill("add")
    def add_numbers(query: str) -> str:
        """덧셈 계산"""
        try:
            # "calculate 5 + 3" 형식의 간단한 파싱
            parts = query.replace("calculate", "").replace("plus", "+").replace("add", "+")
            if "+" in parts:
                numbers = [float(x.strip()) for x in parts.split("+")]
                result = sum(numbers)
                return f"Calculation result: {' + '.join(map(str, numbers))} = {result}"
            else:
                return "Please use format: calculate 5 + 3"
        except Exception as e:
            return f"Calculation error: {e}"

    @calculator.skill("multiply")
    def multiply_numbers(query: str) -> str:
        """곱셈 계산"""
        try:
            parts = query.replace("calculate", "").replace("times", "*").replace("×", "*")
            if "*" in parts:
                numbers = [float(x.strip()) for x in parts.split("*")]
                result = 1
                for num in numbers:
                    result *= num
                return f"Calculation result: {' × '.join(map(str, numbers))} = {result}"
            else:
                return "Please use format: calculate 5 * 3"
        except Exception as e:
            return f"Calculation error: {e}"

    @calculator.skill("info")
    def get_info(query: str) -> str:
        """에이전트 정보 가져오기"""
        return f"I am {calculator.name}, can perform basic mathematical calculations. Supported skills: {list(calculator.skills.keys())}"

    print(f"✅ 계산기 에이전트가 성공적으로 생성되었습니다. 지원 스킬: {list(calculator.skills.keys())}")
    return calculator

# 에이전트 생성
calc_agent = create_calculator_agent()
if calc_agent:
    # 스킬 테스트
    print("\n🧪 에이전트 스킬 테스트:")
    test_queries = [
        "Get information",
        "Calculate 10 + 5",
        "Calculate 6 * 7"
    ]

    for query in test_queries:
        if "information" in query.lower():
            result = calc_agent.skills["info"](query)
        elif "+" in query:
            result = calc_agent.skills["add"](query)
        elif "*" in query or "×" in query:
            result = calc_agent.skills["multiply"](query)
        else:
            result = "Unknown query type"

        print(f"  📝 쿼리: {query}")
        print(f"  🤖 응답: {result}")
        print()
```

**(2) 사용자 정의 A2A Agent**

자신만의 A2A 에이전트도 만들 수 있습니다. 간단한 시연은 다음과 같습니다.

```python
from hello_agents.protocols.a2a.implementation import A2AServer, A2A_AVAILABLE

def create_custom_agent():
    """사용자 정의 에이전트 생성"""
    if not A2A_AVAILABLE:
        print("먼저 A2A SDK를 설치하세요: pip install a2a-sdk")
        return None

    # 에이전트 생성
    agent = A2AServer(
        name="my-custom-agent",
        description="My custom agent",
        capabilities={"custom": ["skill1", "skill2"]}
    )

    # 스킬 추가
    @agent.skill("greet")
    def greet_user(name: str) -> str:
        """사용자에게 인사"""
        return f"Hello, {name}! I am a custom agent."

    @agent.skill("calculate")
    def simple_calculate(expression: str) -> str:
        """간단한 계산"""
        try:
            # 안전한 계산 (기본 연산만 지원)
            allowed_chars = set('0123456789+-*/(). ')
            if all(c in allowed_chars for c in expression):
                result = eval(expression)
                return f"Calculation result: {expression} = {result}"
            else:
                return "Error: Only basic mathematical operations supported"
        except Exception as e:
            return f"Calculation error: {e}"

    return agent

# 사용자 정의 에이전트 생성 및 테스트
custom_agent = create_custom_agent()
if custom_agent:
    # 스킬 테스트
    print("인사 스킬 테스트:")
    result1 = custom_agent.skills["greet"]("Zhang San")
    print(result1)

    print("\n계산 스킬 테스트:")
    result2 = custom_agent.skills["calculate"]("10 + 5 * 2")
    print(result2)
```

### 10.3.3 HelloAgents A2A Tools 사용하기

HelloAgents는 통일된 A2A 도구 인터페이스를 제공합니다.

**(1) A2A Agent Server 만들기**

먼저 Agent 서버를 만들어 봅시다.

```python
from hello_agents.protocols import A2AServer
import threading
import time

# 리서처 Agent 서비스 생성
researcher = A2AServer(
    name="researcher",
    description="Agent responsible for searching and analyzing materials",
    version="1.0.0"
)

# 스킬 정의
@researcher.skill("research")
def handle_research(text: str) -> str:
    """리서치 요청 처리"""
    import re
    match = re.search(r'research\s+(.+)', text, re.IGNORECASE)
    topic = match.group(1).strip() if match else text

    # 실제 리서치 로직 (여기서는 단순화)
    result = {
        "topic": topic,
        "findings": f"Research results about {topic}...",
        "sources": ["Source 1", "Source 2", "Source 3"]
    }
    return str(result)

# 백그라운드에서 서비스 시작
def start_server():
    researcher.run(host="localhost", port=5000)

if __name__ == "__main__":
    server_thread = threading.Thread(target=start_server, daemon=True)
    server_thread.start()

    print("✅ Researcher Agent 서비스가 http://localhost:5000 에서 시작되었습니다")

    # 프로그램 계속 실행
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("\n서비스가 중지되었습니다")
```

**(2) A2A Agent Client 만들기**

이제 서버와 통신할 클라이언트를 만들어 봅시다.

```python
from hello_agents.protocols import A2AClient

# researcher Agent에 연결할 클라이언트 생성
client = A2AClient("http://localhost:5000")

# 리서치 요청 전송
response = client.execute_skill("research", "research AI applications in healthcare")
print(f"받은 응답: {response.get('result')}")

# 출력:
# 받은 응답: {'topic': 'AI applications in healthcare', 'findings': 'Research results about AI applications in healthcare...', 'sources': ['Source 1', 'Source 2', 'Source 3']}
```

**(3) Agent Network 만들기**

여러 Agent 간 협업을 위해 여러 Agent를 서로 연결할 수 있습니다.

```python
from hello_agents.protocols import A2AServer, A2AClient
import threading
import time

# 1. 여러 Agent 서비스 생성
researcher = A2AServer(
    name="researcher",
    description="Researcher"
)

@researcher.skill("research")
def do_research(text: str) -> str:
    import re
    match = re.search(r'research\s+(.+)', text, re.IGNORECASE)
    topic = match.group(1).strip() if match else text
    return str({"topic": topic, "findings": f"Research results for {topic}"})

writer = A2AServer(
    name="writer",
    description="Writer"
)

@writer.skill("write")
def write_article(text: str) -> str:
    import re
    match = re.search(r'write\s+(.+)', text, re.IGNORECASE)
    content = match.group(1).strip() if match else text

    # 리서치 데이터 파싱 시도
    try:
        data = eval(content)
        topic = data.get("topic", "Unknown topic")
        findings = data.get("findings", "No research results")
    except:
        topic = "Unknown topic"
        findings = content

    return f"# {topic}\n\nBased on research: {findings}\n\nArticle content..."

editor = A2AServer(
    name="editor",
    description="Editor"
)

@editor.skill("edit")
def edit_article(text: str) -> str:
    import re
    match = re.search(r'edit\s+(.+)', text, re.IGNORECASE)
    article = match.group(1).strip() if match else text

    result = {
        "article": article + "\n\n[Edited and optimized]",
        "feedback": "Article quality is good",
        "approved": True
    }
    return str(result)

# 2. 모든 서비스 시작
threading.Thread(target=lambda: researcher.run(port=5000), daemon=True).start()
threading.Thread(target=lambda: writer.run(port=5001), daemon=True).start()
threading.Thread(target=lambda: editor.run(port=5002), daemon=True).start()
time.sleep(2)  # 서비스 시작 대기

# 3. 각 Agent에 연결할 클라이언트 생성
researcher_client = A2AClient("http://localhost:5000")
writer_client = A2AClient("http://localhost:5001")
editor_client = A2AClient("http://localhost:5002")

# 4. 협업 워크플로
def create_content(topic):
    # 1단계: 리서치
    research = researcher_client.execute_skill("research", f"research {topic}")
    research_data = research.get('result', '')

    # 2단계: 작성
    article = writer_client.execute_skill("write", f"write {research_data}")
    article_content = article.get('result', '')

    # 3단계: 편집
    final = editor_client.execute_skill("edit", f"edit {article_content}")
    return final.get('result', '')

# 사용
result = create_content("AI applications in healthcare")
print(f"\n최종 결과:\n{result}")
```

### 10.3.4 에이전트에서 A2A Tools 사용하기

이제 A2A를 HelloAgents 에이전트에 통합하는 방법을 살펴보겠습니다.

**(1) A2ATool 래퍼 사용**

```python
from hello_agents import SimpleAgent, HelloAgentsLLM
from hello_agents.tools import A2ATool
from dotenv import load_dotenv

load_dotenv()
llm = HelloAgentsLLM()

# researcher Agent 서비스가 이미 http://localhost:5000 에서 실행 중이라고 가정

# 코디네이터 Agent 생성
coordinator = SimpleAgent(name="Coordinator", llm=llm)

# A2A 도구 추가, researcher Agent에 연결
researcher_tool = A2ATool(
    name="researcher",
    description="Researcher Agent, can search and analyze materials",
    agent_url="http://localhost:5000"
)
coordinator.add_tool(researcher_tool)

# Coordinator가 researcher Agent를 호출할 수 있음
response = coordinator.run("Please have the researcher help me research AI applications in education")
print(response)
```

**(2) 실전 사례: 지능형 고객 서비스 시스템**

세 개의 Agent로 완전한 지능형 고객 서비스 시스템을 만들어 봅시다.

- **Receptionist**: 고객 질문 유형 분석
- **Technical Expert**: 기술 질문 답변
- **Sales Consultant**: 영업 질문 답변

```python
from hello_agents import SimpleAgent, HelloAgentsLLM
from hello_agents.tools import A2ATool
from hello_agents.protocols import A2AServer
import threading
import time
from dotenv import load_dotenv

load_dotenv()
llm = HelloAgentsLLM()

# 1. 기술 전문가 Agent 서비스 생성
tech_expert = A2AServer(
    name="tech_expert",
    description="Technical expert, answers technical questions"
)

@tech_expert.skill("answer")
def answer_tech_question(text: str) -> str:
    import re
    match = re.search(r'answer\s+(.+)', text, re.IGNORECASE)
    question = match.group(1).strip() if match else text
    # 실제 애플리케이션에서는 LLM 또는 지식 베이스를 호출
    return f"Technical answer: Regarding '{question}', I suggest you check our technical documentation..."

# 2. 영업 상담 Agent 서비스 생성
sales_advisor = A2AServer(
    name="sales_advisor",
    description="Sales consultant, answers sales questions"
)

@sales_advisor.skill("answer")
def answer_sales_question(text: str) -> str:
    import re
    match = re.search(r'answer\s+(.+)', text, re.IGNORECASE)
    question = match.group(1).strip() if match else text
    return f"Sales answer: Regarding '{question}', we have special offers..."

# 3. 서비스 시작
threading.Thread(target=lambda: tech_expert.run(port=6000), daemon=True).start()
threading.Thread(target=lambda: sales_advisor.run(port=6001), daemon=True).start()
time.sleep(2)

# 4. 접수 Agent 생성 (HelloAgents의 SimpleAgent 사용)
receptionist = SimpleAgent(
    name="Receptionist",
    llm=llm,
    system_prompt="""You are a customer service receptionist, responsible for:
1. Analyzing customer question types (technical questions or sales questions)
2. Forwarding questions to appropriate experts
3. Organizing expert answers and returning them to customers

Please remain polite and professional."""
)

# 기술 전문가 도구 추가
tech_tool = A2ATool(
    agent_url="http://localhost:6000",
    name="tech_expert",
    description="Technical expert, answers technical-related questions"
)
receptionist.add_tool(tech_tool)

# 영업 상담 도구 추가
sales_tool = A2ATool(
    agent_url="http://localhost:6001",
    name="sales_advisor",
    description="Sales consultant, answers price and purchase-related questions"
)
receptionist.add_tool(sales_tool)

# 5. 고객 문의 처리
def handle_customer_query(query):
    print(f"\n고객 문의: {query}")
    print("=" * 50)
    response = receptionist.run(query)
    print(f"\n고객 서비스 답변: {response}")
    print("=" * 50)

# 서로 다른 유형의 질문 테스트
if __name__ == "__main__":
    handle_customer_query("How do I call your API?")
    handle_customer_query("What is the price of the enterprise version?")
    handle_customer_query("How do I integrate it into my Python project?")
```

**(3) 고급 사용: Agent Negotiation**

A2A 프로토콜은 Agent 간 협상 메커니즘도 지원합니다.

```python
from hello_agents.protocols import A2AServer, A2AClient
import threading
import time

# 협상이 필요한 두 Agent 생성
agent1 = A2AServer(
    name="agent1",
    description="Agent 1"
)

@agent1.skill("propose")
def handle_proposal(text: str) -> str:
    """협상 제안 처리"""
    import re

    # 제안 파싱
    match = re.search(r'propose\s+(.+)', text, re.IGNORECASE)
    proposal_str = match.group(1).strip() if match else text

    try:
        proposal = eval(proposal_str)
        task = proposal.get("task")
        deadline = proposal.get("deadline")

        # 제안 평가
        if deadline >= 7:  # 최소 7일 필요
            result = {"accepted": True, "message": "Proposal accepted"}
        else:
            result = {
                "accepted": False,
                "message": "Timeline too tight",
                "counter_proposal": {"deadline": 7}
            }
        return str(result)
    except:
        return str({"accepted": False, "message": "Invalid proposal format"})

agent2 = A2AServer(
    name="agent2",
    description="Agent 2"
)

@agent2.skill("negotiate")
def negotiate_task(text: str) -> str:
    """협상 시작"""
    import re

    # 작업과 마감 기한 파싱
    match = re.search(r'negotiate\s+task:(.+?)\s+deadline:(\d+)', text, re.IGNORECASE)
    if match:
        task = match.group(1).strip()
        deadline = int(match.group(2))

        # agent1에 제안 전송
        proposal = {"task": task, "deadline": deadline}
        return str({"status": "negotiating", "proposal": proposal})
    else:
        return str({"status": "error", "message": "Invalid negotiation request"})

# 서비스 시작
threading.Thread(target=lambda: agent1.run(port=7000), daemon=True).start()
threading.Thread(target=lambda: agent2.run(port=7001), daemon=True).start()
```

## 10.4 ANP 프로토콜 실전

MCP 프로토콜이 도구 호출을 해결하고 A2A 프로토콜이 피어 투 피어 에이전트 협업을 해결한 뒤, ANP 프로토콜은 대규모 개방형 네트워크 환경에서의 에이전트 관리 문제 해결에 초점을 맞춥니다.

10.2절과 10.3절에서 우리는 MCP(도구 접근)와 A2A(에이전트 협업)를 학습했습니다. 이제 **대규모 개방형 에이전트 네트워크** 구축에 초점을 맞추는 ANP (Agent Network Protocol) 프로토콜을 학습해 봅시다.

### 10.4.1 프로토콜 목표

네트워크에 서로 다른 기능을 가진 많은 에이전트(예: 자연어 처리, 이미지 인식, 데이터 분석 등)가 포함되면, 시스템은 일련의 도전에 직면합니다.

- **서비스 발견**: 새로운 작업이 들어왔을 때 해당 작업을 처리할 수 있는 에이전트를 어떻게 빠르게 찾을 것인가?
- **지능형 라우팅**: 여러 에이전트가 같은 작업을 처리할 수 있다면, 가장 적합한 에이전트(예: 부하, 비용 등을 기준으로)를 어떻게 선택하고 작업을 배정할 것인가?
- **동적 확장**: 새로 참여한 에이전트를 다른 구성원이 발견하고 호출할 수 있게 하려면 어떻게 해야 하는가?

ANP의 설계 목표는 위의 서비스 발견, 라우팅 선택, 네트워크 확장성 문제를 해결하기 위한 표준화된 메커니즘을 제공하는 것입니다.

설계 목표를 달성하기 위해 ANP는 표 10.8과 같은 핵심 개념을 정의합니다.

<div align="center">
  <p>표 10.8 ANP 핵심 개념</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-table-8.png" alt="" width="85%"/>
</div>

또한 공식 [Getting Started Guide](https://github.com/agent-network-protocol/AgentNetworkProtocol/blob/main/docs/chinese/ANP入门指南.md)를 참고하여 ANP의 아키텍처 설계를 소개합니다. 그림 10.9에 나와 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-9.png" alt="" width="85%"/>
  <p>그림 10.9 ANP 전체 프로세스</p>
</div>

이 흐름도에서 주요 단계는 다음과 같습니다.

**1. 서비스 발견 및 매칭:** 먼저 Agent A는 공개 발견 서비스를 사용하여 의미적 또는 기능적 설명을 기반으로 질의하고, 자신의 작업 요구사항에 맞는 Agent B를 찾습니다. 발견 서비스는 각 에이전트가 노출하는 표준 엔드포인트(`.well-known/agent-descriptions`)를 미리 크롤링하여 인덱스를 구축함으로써, 서비스 수요자와 제공자 사이의 동적 매칭을 실현합니다.

**2. DID 기반 신원 확인:** 상호작용이 시작될 때 Agent A는 자신의 DID가 포함된 요청에 개인 키로 서명합니다. Agent B는 이를 받은 뒤 DID를 파싱하여 해당 공개 키를 얻고, 이를 사용해 서명의 진위와 요청의 무결성을 검증합니다. 이로써 양측 사이에 신뢰할 수 있는 통신이 수립됩니다.

**3. 표준화된 서비스 실행:** 신원 확인이 통과되면 Agent B가 요청에 응답하고, 양측은 예약, 조회 등과 같은 서비스 호출이나 데이터 교환을 미리 정의된 표준 인터페이스와 데이터 형식에 따라 수행합니다. 표준화된 상호작용 프로세스는 크로스 플랫폼 및 크로스 시스템 상호 운용성을 달성하는 기반입니다.

요약하면, 이 메커니즘의 핵심은 DID를 사용해 탈중앙화된 신뢰 기반을 구축하고, 표준화된 설명 프로토콜을 활용해 동적 서비스 발견을 달성하는 것입니다. 이 방식은 중앙 조정 없이도 에이전트들이 인터넷 위에서 안전하고 효율적으로 협업 네트워크를 형성할 수 있게 합니다.

### 10.4.2 ANP 서비스 발견 사용하기

**(1) 서비스 발견 센터 만들기**

```python
from hello_agents.protocols import ANPDiscovery, register_service

# 서비스 발견 센터 생성
discovery = ANPDiscovery()

# Agent 서비스 등록
register_service(
    discovery=discovery,
    service_id="nlp_agent_1",
    service_name="NLP Processing Expert A",
    service_type="nlp",
    capabilities=["text_analysis", "sentiment_analysis", "ner"],
    endpoint="http://localhost:8001",
    metadata={"load": 0.3, "price": 0.01, "version": "1.0.0"}
)

register_service(
    discovery=discovery,
    service_id="nlp_agent_2",
    service_name="NLP Processing Expert B",
    service_type="nlp",
    capabilities=["text_analysis", "translation"],
    endpoint="http://localhost:8002",
    metadata={"load": 0.7, "price": 0.02, "version": "1.1.0"}
)

print("✅ 서비스 등록 완료")
```

**(2) 서비스 발견하기**

```python
from hello_agents.protocols import discover_service

# 유형별 검색
nlp_services = discover_service(discovery, service_type="nlp")
print(f"{len(nlp_services)}개의 NLP 서비스를 찾았습니다")

# 부하가 가장 낮은 서비스 선택
best_service = min(nlp_services, key=lambda s: s.metadata.get("load", 1.0))
print(f"최적 서비스: {best_service.service_name} (부하: {best_service.metadata['load']})")
```

**(3) Agent Network 구축하기**

```python
from hello_agents.protocols import ANPNetwork

# 네트워크 생성
network = ANPNetwork(network_id="ai_cluster")

# 노드 추가
for service in discovery.list_all_services():
    network.add_node(service.service_id, service.endpoint)

# 연결 수립 (능력 매칭 기반)
network.connect_nodes("nlp_agent_1", "nlp_agent_2")

stats = network.get_network_stats()
print(f"✅ 네트워크 구축 완료, 총 {stats['total_nodes']}개 노드")
```

### 10.4.3 실전 사례

완전한 분산 작업 스케줄링 시스템을 만들어 봅시다.

```python
from hello_agents.protocols import ANPDiscovery, register_service
from hello_agents import SimpleAgent, HelloAgentsLLM
from hello_agents.tools.builtin import ANPTool
import random
from dotenv import load_dotenv

load_dotenv()
llm = HelloAgentsLLM()

# 1. 서비스 발견 센터 생성
discovery = ANPDiscovery()

# 2. 여러 계산 노드 등록
for i in range(10):
    register_service(
        discovery=discovery,
        service_id=f"compute_node_{i}",
        service_name=f"Compute Node {i}",
        service_type="compute",
        capabilities=["data_processing", "ml_training"],
        endpoint=f"http://node{i}:8000",
        metadata={
            "load": random.uniform(0.1, 0.9),
            "cpu_cores": random.choice([4, 8, 16]),
            "memory_gb": random.choice([16, 32, 64]),
            "gpu": random.choice([True, False])
        }
    )

print(f"✅ {len(discovery.list_all_services())}개의 계산 노드가 등록되었습니다")

# 3. 작업 스케줄러 Agent 생성
scheduler = SimpleAgent(
    name="Task Scheduler",
    llm=llm,
    system_prompt="""You are an intelligent task scheduler, responsible for:
1. Analyzing task requirements
2. Selecting the most suitable compute node
3. Assigning tasks

When selecting nodes, consider: load, CPU cores, memory, GPU, and other factors."""
)

# ANP 도구 추가
anp_tool = ANPTool(
    name="service_discovery",
    description="Service discovery tool, can find and select compute nodes",
    discovery=discovery
)
scheduler.add_tool(anp_tool)

# 4. 지능형 작업 할당
def assign_task(task_description):
    print(f"\n작업: {task_description}")
    print("=" * 50)

    # Agent가 지능적으로 노드 선택
    response = scheduler.run(f"""
    Please select the most suitable compute node for the following task:
    {task_description}

    Requirements:
    1. List all available nodes
    2. Analyze characteristics of each node
    3. Select the most suitable node
    4. Explain selection reasoning
    """)

    print(response)
    print("=" * 50)

# 서로 다른 유형의 작업 테스트
assign_task("Train a large deep learning model, requires GPU support")
assign_task("Process large amounts of text data, requires high memory")
assign_task("Run lightweight data analysis task")
```

다음은 로드 밸런싱 예제입니다.

```python
from hello_agents.protocols import ANPDiscovery, register_service
import random

# 서비스 발견 센터 생성
discovery = ANPDiscovery()

# 같은 유형의 여러 서비스 등록
for i in range(5):
    register_service(
        discovery=discovery,
        service_id=f"api_server_{i}",
        service_name=f"API Server {i}",
        service_type="api",
        capabilities=["rest_api"],
        endpoint=f"http://api{i}:8000",
        metadata={"load": random.uniform(0.1, 0.9)}
    )

# 로드 밸런싱 함수
def get_best_server():
    """부하가 가장 낮은 서버 선택"""
    servers = discovery.discover_services(service_type="api")
    if not servers:
        return None

    best = min(servers, key=lambda s: s.metadata.get("load", 1.0))
    return best

# 요청 할당 시뮬레이션
for i in range(10):
    server = get_best_server()
    print(f"요청 {i+1} -> {server.service_name} (부하: {server.metadata['load']:.2f})")

    # 부하 업데이트 (시뮬레이션)
    server.metadata["load"] += 0.1
```

## 10.5 사용자 정의 MCP 서버 구축

이전 섹션들에서는 기존 MCP 서비스를 사용하는 방법을 배웠습니다. 또한 서로 다른 프로토콜의 특성도 학습했습니다. 이제 직접 MCP 서버를 구축하는 방법을 알아보겠습니다.

### 10.5.1 첫 번째 MCP 서버 만들기

**(1) 왜 사용자 정의 MCP 서버를 만들어야 할까?**

공개 MCP 서비스를 직접 사용할 수도 있지만, 많은 실제 애플리케이션 시나리오에서는 특정 요구사항을 충족하기 위해 사용자 정의 MCP 서버를 구축해야 합니다.

주요 동기는 다음과 같습니다.

- **비즈니스 로직 캡슐화**: 기업 고유의 비즈니스 프로세스나 복잡한 작업을 표준화된 MCP 도구로 캡슐화하여 에이전트가 통일된 방식으로 호출할 수 있게 합니다.
- **비공개 데이터 접근**: 내부 데이터베이스, API 또는 공개 네트워크에 노출할 수 없는 기타 비공개 데이터 소스에 접근하기 위한 안전하고 제어 가능한 인터페이스 또는 프록시를 만듭니다.
- **성능 최적화**: 호출 빈도가 높거나 응답 지연 요구사항이 엄격한 애플리케이션 시나리오를 위해 심층 최적화를 수행합니다.
- **사용자 정의 기능 확장**: 표준 MCP 서비스가 제공하지 않는 특정 기능을 구현합니다. 예를 들어 독점 알고리즘 모델 통합이나 특정 하드웨어 장치 연결이 있습니다.

**(2) 교육 사례: 날씨 조회 MCP 서버**

간단한 날씨 조회 서버부터 시작하여 MCP 서버 개발을 단계적으로 배워 보겠습니다.

```python
#!/usr/bin/env python3
"""날씨 조회 MCP 서버"""

import json
import requests
import os
from datetime import datetime
from typing import Dict, Any
from hello_agents.protocols import MCPServer

# MCP 서버 생성
weather_server = MCPServer(name="weather-server", description="Real weather query service")

CITY_MAP = {
    "Beijing": "Beijing", "Shanghai": "Shanghai", "Guangzhou": "Guangzhou",
    "Shenzhen": "Shenzhen", "Hangzhou": "Hangzhou", "Chengdu": "Chengdu",
    "Chongqing": "Chongqing", "Wuhan": "Wuhan", "Xi'an": "Xi'an",
    "Nanjing": "Nanjing", "Tianjin": "Tianjin", "Suzhou": "Suzhou"
}


def get_weather_data(city: str) -> Dict[str, Any]:
    """wttr.in에서 날씨 데이터 가져오기"""
    city_en = CITY_MAP.get(city, city)
    url = f"https://wttr.in/{city_en}?format=j1"
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    data = response.json()
    current = data["current_condition"][0]

    return {
        "city": city,
        "temperature": float(current["temp_C"]),
        "feels_like": float(current["FeelsLikeC"]),
        "humidity": int(current["humidity"]),
        "condition": current["weatherDesc"][0]["value"],
        "wind_speed": round(float(current["windspeedKmph"]) / 3.6, 1),
        "visibility": float(current["visibility"]),
        "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    }


# 도구 함수 정의
def get_weather(city: str) -> str:
    """지정한 도시의 현재 날씨 가져오기"""
    try:
        weather_data = get_weather_data(city)
        return json.dumps(weather_data, ensure_ascii=False, indent=2)
    except Exception as e:
        return json.dumps({"error": str(e), "city": city}, ensure_ascii=False)


def list_supported_cities() -> str:
    """지원되는 모든 중국 도시 나열"""
    result = {"cities": list(CITY_MAP.keys()), "count": len(CITY_MAP)}
    return json.dumps(result, ensure_ascii=False, indent=2)


def get_server_info() -> str:
    """서버 정보 가져오기"""
    info = {
        "name": "Weather MCP Server",
        "version": "1.0.0",
        "tools": ["get_weather", "list_supported_cities", "get_server_info"]
    }
    return json.dumps(info, ensure_ascii=False, indent=2)


# 도구를 서버에 등록
weather_server.add_tool(get_weather)
weather_server.add_tool(list_supported_cities)
weather_server.add_tool(get_server_info)


if __name__ == "__main__":
    weather_server.run()
```

**(3) 사용자 정의 MCP 서버 테스트**

그런 다음 테스트 스크립트를 만듭니다.

```python
#!/usr/bin/env python3
"""날씨 조회 MCP 서버 테스트"""

import asyncio
import json
import sys
import os

sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..', 'HelloAgents'))
from hello_agents.protocols.mcp.client import MCPClient


async def test_weather_server():
    server_script = os.path.join(os.path.dirname(__file__), "14_weather_mcp_server.py")
    client = MCPClient(["python", server_script])

    try:
        async with client:
            # 테스트 1: 서버 정보 가져오기
            info = json.loads(await client.call_tool("get_server_info", {}))
            print(f"서버: {info['name']} v{info['version']}")

            # 테스트 2: 지원 도시 나열
            cities = json.loads(await client.call_tool("list_supported_cities", {}))
            print(f"지원 도시: {cities['count']}개 도시")

            # 테스트 3: 베이징 날씨 조회
            weather = json.loads(await client.call_tool("get_weather", {"city": "Beijing"}))
            if "error" not in weather:
                print(f"\n베이징 날씨: {weather['temperature']}°C, {weather['condition']}")

            # 테스트 4: 선전 날씨 조회
            weather = json.loads(await client.call_tool("get_weather", {"city": "Shenzhen"}))
            if "error" not in weather:
                print(f"선전 날씨: {weather['temperature']}°C, {weather['condition']}")

            print("\n✅ 모든 테스트 완료!")

    except Exception as e:
        print(f"❌ 테스트 실패: {e}")


if __name__ == "__main__":
    asyncio.run(test_weather_server())
```

**(4) Agent에서 사용자 정의 MCP 서버 사용하기**

```python
"""Agent에서 Weather MCP Server 사용하기"""

import os
from dotenv import load_dotenv
from hello_agents import SimpleAgent, HelloAgentsLLM
from hello_agents.tools import MCPTool

load_dotenv()


def create_weather_assistant():
    """날씨 어시스턴트 생성"""
    llm = HelloAgentsLLM()

    assistant = SimpleAgent(
        name="Weather Assistant",
        llm=llm,
        system_prompt="""You are a weather assistant that can query city weather.
Use the get_weather tool to query weather, supports Chinese city names.
"""
    )

    # 날씨 MCP 도구 추가
    server_script = os.path.join(os.path.dirname(__file__), "14_weather_mcp_server.py")
    weather_tool = MCPTool(server_command=["python", server_script])
    assistant.add_tool(weather_tool)

    return assistant


def demo():
    """데모"""
    assistant = create_weather_assistant()

    print("\n베이징 날씨 조회:")
    response = assistant.run("How's the weather in Beijing today?")
    print(f"답변: {response}\n")


def interactive():
    """대화형 모드"""
    assistant = create_weather_assistant()

    while True:
        user_input = input("\nYou: ").strip()
        if user_input.lower() in ['quit', 'exit']:
            break
        response = assistant.run(user_input)
        print(f"Assistant: {response}")


if __name__ == "__main__":
    import sys
    if len(sys.argv) > 1 and sys.argv[1] == "demo":
        demo()
    else:
        interactive()
```

```
🔗 MCP 서버에 연결 중...
✅ 연결 성공!
🔌 연결 해제됨
✅ Tool 'mcp_get_weather' registered.
✅ Tool 'mcp_list_supported_cities' registered.
✅ Tool 'mcp_get_server_info' registered.
✅ MCP tool 'mcp' expanded into 3 independent tools

You: I want to query Beijing's weather
🔗 MCP 서버에 연결 중...
✅ 연결 성공!
🔌 연결 해제됨
Assistant: 현재 베이징의 날씨는 다음과 같습니다:

- 온도: 10.0°C
- 체감 온도: 9.0°C
- 습도: 94%
- 날씨 상태: 약한 비
- 풍속: 1.7 m/s
- 가시거리: 10.0 km
- 타임스탬프: 2025년 10월 9일 13:46:40

우산을 챙기고 날씨 변화에 맞춰 옷차림을 조절하세요.
```

### 10.5.2 MCP 서버 업로드하기

우리는 실제 날씨 조회 MCP 서버를 만들었습니다. 이제 전 세계 개발자가 우리 서비스를 사용할 수 있도록 Smithery 플랫폼에 게시해 봅시다.

(1) Smithery란 무엇인가?

[Smithery](https://smithery.ai/)는 MCP 서버를 위한 공식 게시 플랫폼으로, Python의 PyPI나 Node.js의 npm과 비슷합니다. Smithery를 통해 사용자는 다음을 할 수 있습니다.

- 🔍 MCP 서버 발견 및 검색
- 📦 한 번의 클릭으로 MCP 서버 설치
- 📊 서버 사용 통계와 평점 확인
- 🔄 서버 업데이트 자동 획득

(2) 게시 준비

먼저 프로젝트를 표준 게시 형식으로 정리해야 합니다. 이 폴더는 참고할 수 있도록 `code` 디렉터리에 정리되어 있습니다.

```
weather-mcp-server/
├── README.md           # 프로젝트 문서
├── LICENSE            # 오픈소스 라이선스
├── Dockerfile         # Docker 빌드 설정 (권장)
├── pyproject.toml     # Python 프로젝트 설정 (필수)
├── requirements.txt   # Python 의존성
├── smithery.yaml      # Smithery 설정 파일 (필수)
└── server.py          # MCP 서버 메인 파일
```

`smithery.yaml`은 Smithery 플랫폼을 위한 설정 파일입니다.

```yaml
name: weather-mcp-server
displayName: Weather MCP Server
description: Real-time weather query MCP server based on HelloAgents framework
version: 1.0.0
author: HelloAgents Team
homepage: https://github.com/yourusername/weather-mcp-server
license: MIT
categories:
  - weather
  - data
tags:
  - weather
  - real-time
  - helloagents
  - wttr
runtime: container
build:
  dockerfile: Dockerfile
  dockerBuildPath: .
startCommand:
  type: http
tools:
  - name: get_weather
    description: Get current weather for a city
  - name: list_supported_cities
    description: List all supported cities
  - name: get_server_info
    description: Get server information
```

설정 설명:

- `name`: 서버의 고유 식별자(소문자, 하이픈 구분)
- `displayName`: 표시 이름
- `description`: 간단한 설명
- `version`: 버전 번호(시맨틱 버저닝 준수)
- `runtime`: 런타임 환경(python/node)
- `entrypoint`: 진입 파일
- `tools`: 도구 목록

`pyproject.toml`은 Python 프로젝트의 표준 설정 파일입니다. Smithery는 이후 서버로 패키징하기 때문에 이 파일을 요구합니다.

```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "weather-mcp-server"
version = "1.0.0"
description = "Real-time weather query MCP server based on HelloAgents framework"
readme = "README.md"
license = {text = "MIT"}
authors = [
    {name = "HelloAgents Team", email = "xxx"}
]
requires-python = ">=3.10"
dependencies = [
    "hello-agents>=0.2.1",
    "requests>=2.31.0",
]

[project.urls]
Homepage = "https://github.com/yourusername/weather-mcp-server"
Repository = "https://github.com/yourusername/weather-mcp-server"
"Bug Tracker" = "https://github.com/yourusername/weather-mcp-server/issues"

[tool.setuptools]
py-modules = ["server"]
```

설정 설명:

- `[build-system]`: 빌드 도구 지정(setuptools)
- `[project]`: 프로젝트 메타데이터
  - `name`: 프로젝트 이름
  - `version`: 버전 번호(시맨틱 버저닝 준수)
  - `dependencies`: 프로젝트 의존성 목록
  - `requires-python`: Python 버전 요구사항
- `[project.urls]`: 프로젝트 관련 링크
- `[tool.setuptools]`: setuptools 설정

Smithery가 Dockerfile을 자동 생성하긴 하지만, 사용자 정의 Dockerfile을 제공하면 배포 성공 가능성을 높일 수 있습니다.

```dockerfile
# weather-mcp-server를 위한 멀티 스테이지 빌드
FROM python:3.12-slim-bookworm as base

# 작업 디렉터리 설정
WORKDIR /app

# 시스템 의존성 설치
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

# 프로젝트 파일 복사
COPY pyproject.toml requirements.txt ./
COPY server.py ./

# Python 의존성 설치
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# 환경 변수 설정
ENV PYTHONUNBUFFERED=1
ENV PORT=8081

# 포트 노출 (Smithery는 8081 사용)
EXPOSE 8081

# 헬스 체크
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import sys; sys.exit(0)"

# MCP 서버 실행
CMD ["python", "server.py"]
```

Dockerfile 설정 설명:

- **Base Image**: `python:3.12-slim-bookworm` - 경량 Python 이미지
- **Working Directory**: `/app` - 애플리케이션 루트 디렉터리
- **Port**: `8081` - Smithery 플랫폼 표준 포트
- **Start Command**: `python server.py` - MCP 서버 실행

여기서는 `hello-agents` 저장소를 Fork하고, `code` 안의 소스 코드를 가져와 자신의 GitHub에서 `weather-mcp-server`라는 이름의 저장소를 만든 뒤, `yourusername`을 자신의 GitHub 사용자 이름으로 변경해야 합니다.

(3) Smithery에 제출하기

브라우저를 열고 [https://smithery.ai/](https://smithery.ai/)에 접속합니다. GitHub 계정으로 Smithery에 로그인합니다. 페이지에서 "Publish Server" 버튼을 클릭하고 GitHub 저장소 URL `https://github.com/yourusername/weather-mcp-server`를 입력한 뒤 게시가 완료될 때까지 기다립니다.

게시가 완료되면 그림 10.10과 비슷한 페이지를 볼 수 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-10.png" alt="" width="85%"/>
  <p>그림 10.10 Smithery 게시 성공 페이지</p>
</div>

서버가 성공적으로 게시되면 사용자는 다음과 같은 방식으로 사용할 수 있습니다.

방법 1: Smithery CLI 사용

```bash
# Smithery CLI 설치
npm install -g @smithery/cli

# 서버 설치
smithery install weather-mcp-server
```

방법 2: Claude Desktop에서 설정

```json
{
  "mcpServers": {
    "weather": {
      "command": "smithery",
      "args": ["run", "weather-mcp-server"]
    }
  }
}
```

방법 3: HelloAgents에서 사용

```python
from hello_agents import SimpleAgent, HelloAgentsLLM
from hello_agents.tools.builtin.protocol_tools import MCPTool

agent = SimpleAgent(name="Weather Assistant", llm=HelloAgentsLLM())

# Smithery로 설치한 서버 사용
weather_tool = MCPTool(
    server_command=["smithery", "run", "weather-mcp-server"]
)
agent.add_tool(weather_tool)

response = agent.run("How's the weather in Beijing today?")
```

물론 이는 예시에 불과하며, 더 많은 사용법은 직접 탐색해 볼 수 있습니다. 아래 그림 10.11은 MCP 도구가 성공적으로 게시되었을 때 포함되는 정보를 보여줍니다. 서비스 이름 "Weather", 고유 식별자 `@jjyaoao/weather-mcp-server`, 상태 정보가 표시됩니다. Tools 영역에는 방금 구현한 메서드가 표시되고, Connect 영역에는 이 서비스를 연결하고 사용하는 데 필요한 기술 정보가 제공됩니다. 여기에는 서비스의 **접근 URL 주소**와 여러 언어/환경의 **설정 코드 스니펫**이 포함됩니다. 더 자세히 알고 싶다면 이 [링크](https://smithery.ai/server/@jjyaoao/weather-mcp-server)를 클릭할 수 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/10-figures/10-11.png" alt="" width="85%"/>
  <p>그림 10.11 Smithery에 성공적으로 게시된 MCP 도구</p>
</div>

이제 여러분만의 MCP 서버를 만들 차례입니다!

## 10.6 장 요약

이 장에서는 에이전트 통신을 위한 세 가지 핵심 프로토콜인 MCP, A2A, ANP를 체계적으로 소개하고, 이들의 설계 철학, 적용 시나리오, 실무 방법을 살펴보았습니다.

**프로토콜 포지셔닝:**

- **MCP (Model Context Protocol)**: 에이전트와 도구 사이의 다리로서 통일된 도구 접근 인터페이스를 제공하며, 개별 에이전트의 능력을 강화하는 데 적합합니다.
- **A2A (Agent-to-Agent Protocol)**: 에이전트 사이의 대화 시스템으로서 직접 통신과 작업 협상을 지원하며, 소규모 팀의 긴밀한 협업에 적합합니다.
- **ANP (Agent Network Protocol)**: 에이전트를 위한 "인터넷"으로서 서비스 발견, 라우팅, 로드 밸런싱 메커니즘을 제공하며, 대규모 개방형 에이전트 네트워크 구축에 적합합니다.

**HelloAgents 통합 솔루션**

`HelloAgents` 프레임워크에서 이 세 프로토콜은 모두 도구(Tool)로 통일 추상화되어 매끄러운 통합을 달성합니다. 이를 통해 개발자는 에이전트에 서로 다른 수준의 통신 능력을 유연하게 추가할 수 있습니다.

```python
# 통일된 Tool 인터페이스
from hello_agents.tools import MCPTool, A2ATool, ANPTool

# 모든 프로토콜은 Tools로 Agent에 추가 가능
agent.add_tool(MCPTool(...))
agent.add_tool(A2ATool(...))
agent.add_tool(ANPTool(...))
```

**실무 경험 요약**

- 불필요한 중복 개발을 줄이기 위해 성숙한 커뮤니티 MCP 서비스를 우선적으로 사용하세요.
- 시스템 규모에 따라 적절한 프로토콜을 선택하세요. 소규모 협업 시나리오에는 A2A를 권장하고, 대규모 네트워크 시나리오에는 ANP를 사용해야 합니다.

이 장을 완료한 뒤에는 다음을 권장합니다.

1. **직접 실습**:
   - 자신만의 MCP 서버 구축
   - 프로토콜을 사용해 다중 에이전트 협업 시스템 만들기
   - MCP, A2A, ANP의 조합 적용 전략 탐구
2. **심화 학습**:
   - MCP 공식 문서 읽기: https://modelcontextprotocol.io
   - A2A 공식 문서 읽기: https://a2a-protocol.org/latest/
   - ANP 공식 문서 읽기: https://agent-network-protocol.com/guide/
3. **커뮤니티 참여**:
   - 커뮤니티에 새로운 MCP 서비스 기여
   - 직접 개발한 에이전트 구현 사례 공유
   - 관련 프로토콜의 기술 표준 논의에 참여하거나, Issues에서 질문하거나, HelloAgents가 새로운 예제 사례를 지원하도록 직접 돕기

**10장 완료를 축하합니다!**

이제 여러분은 에이전트 통신 프로토콜의 핵심 지식을 익혔습니다. 계속 정진하세요! 🚀

## 연습문제

> **참고**: 일부 연습문제에는 표준 답안이 없습니다. 초점은 학습자의 에이전트 통신 프로토콜에 대한 종합적 이해와 실천 능력을 기르는 데 있습니다.

1. 이 장에서는 MCP, A2A, ANP라는 세 가지 에이전트 통신 프로토콜을 소개했습니다. 다음을 분석해 보세요.

   - 10.1.2절은 세 프로토콜의 설계 철학을 비교했습니다. 깊이 분석해 보세요. 왜 MCP는 "컨텍스트 공유"를 강조하고, A2A는 "대화형 협업"을 강조하며, ANP는 "네트워크 토폴로지"를 강조할까요? 이러한 설계 철학은 각각 어떤 핵심 문제를 해결하나요?
   - "지능형 고객 서비스 시스템"을 구축한다고 가정해 보겠습니다. 이 시스템은 다음 기능이 필요합니다. (1) 고객 데이터베이스와 주문 시스템 접근, (2) 여러 전문 고객 서비스 에이전트가 협력하여 복잡한 문제 처리, (3) 대규모 동시 사용자 요청 지원. 각 기능에 가장 적합한 프로토콜을 선택하고 이유를 설명하세요.
   - 세 프로토콜을 조합해서 사용할 수 있을까요? MCP, A2A, ANP를 동시에 사용해 완전한 에이전트 시스템을 구축하는 실제 적용 시나리오를 설계해 보세요. 시스템 아키텍처 다이어그램을 그리고 각 프로토콜의 책임을 설명하세요.

2. MCP (Model Context Protocol)는 에이전트-도구 통신을 위한 표준 프로토콜입니다. 10.2절의 내용을 바탕으로 깊이 생각해 보세요.

   > **참고**: 이 문제는 실습형 문제이며 실제 조작을 권장합니다.

   - 10.2.3절의 MCP 서버 구현에서 우리는 `list_tools`와 `call_tool` 같은 핵심 메서드를 정의했습니다. 이 구현을 확장하여 다음 도구를 제공하는 새로운 MCP 서버를 추가하세요. (1) 데이터베이스 쿼리 도구, (2) 데이터 시각화 도구, (3) 보고서 생성 도구. 도구들이 협력하여 복잡한 데이터 분석 작업을 완료할 수 있어야 합니다.
   - MCP 프로토콜은 "Resources"와 "Prompts"라는 두 가지 중요한 개념을 지원하지만, 이 장은 주로 "Tools"에 초점을 맞춥니다. MCP 공식 문서를 참고하여 Resources와 Prompts의 설계 목적을 이해하고, 이 세 핵심 개념을 사용해 더 강력한 에이전트 시스템을 구축하는 애플리케이션 시나리오를 설계하세요.
   - MCP는 JSON-RPC 2.0을 하위 통신 프로토콜로 사용하고 stdio를 통해 프로세스 간 통신을 수행합니다. 이 설계의 장점과 한계는 무엇인지 분석해 보세요. 원격 MCP 서버(HTTP/WebSocket으로 접근)를 지원해야 한다면 현재 구현을 어떻게 확장해야 할까요?

3. A2A (Agent-to-Agent Protocol)는 에이전트 간 대화형 협업을 지원합니다. 10.3절의 내용을 바탕으로 다음 확장 실습을 완료하세요.

   > **참고**: 이 문제는 실습형 문제이며 실제 조작을 권장합니다.

   - 10.3.4절의 "리서치 팀" 사례에서 연구자와 작성자는 A2A 프로토콜을 통해 협력하여 논문 작성을 완료합니다. 이 사례를 확장하여 논문 품질을 검토하고 수정 제안을 제공할 수 있는 세 번째 에이전트 "Reviewer"를 추가하세요. 세 에이전트 간 협업 프로세스를 설계하고 완전한 코드를 구현하세요.
   - A2A 프로토콜은 `task`와 `task_result` 같은 메시지 유형을 정의합니다. 협업 중 충돌이 발생한다면(예: 두 에이전트가 같은 문제에 대해 서로 다른 의견을 갖는 경우), 충돌 해결 메커니즘을 어떻게 설계해야 할까요? "negotiation"과 "voting" 같은 메시지 유형을 추가하여 A2A 프로토콜을 확장해 보세요.
   - A2A 프로토콜과 6장에서 소개한 AutoGen, CAMEL 같은 다중 에이전트 프레임워크를 비교해 보세요. 표준 프로토콜로서 A2A와 이러한 프레임워크의 관계는 무엇인가요? 서로 대체할 수 있을까요? A2A 프로토콜 기반 에이전트가 AutoGen 프레임워크의 에이전트와 통신할 수 있는 솔루션을 설계하세요.

4. ANP (Agent Network Protocol)는 대규모 에이전트 네트워크를 지원합니다. 10.4절의 내용을 바탕으로 깊이 분석해 보세요.

   - 10.4.2절은 스타, 메시, 계층형 등 ANP의 네트워크 토폴로지 설계를 소개했습니다. 어떤 시나리오에서 어떤 토폴로지 구조를 선택해야 하는지 분석하세요. 네트워크 규모가 10개 에이전트에서 1000개 에이전트로 확장된다면 토폴로지 구조는 어떻게 진화해야 할까요?
   - ANP 프로토콜은 "라우팅"과 "발견" 메커니즘을 지원하여 에이전트가 적절한 협업 파트너를 동적으로 찾을 수 있게 합니다. "지능형 라우팅 알고리즘"을 설계해 보세요. 작업 유형, 에이전트 능력, 네트워크 부하 등의 요소를 기반으로 최적의 메시지 라우팅 경로를 자동으로 선택해야 합니다.
   - 10.4.4절의 "스마트 시티" 사례에서 여러 에이전트는 도시 시스템을 관리하기 위해 협력합니다. 핵심 에이전트(예: 교통 관리 에이전트)가 장애를 일으킨다면 전체 시스템은 어떻게 대응해야 할까요? 장애 감지, 백업 전환, 상태 복구 등의 기능을 포함한 "장애 허용 메커니즘"을 설계하세요.

5. 에이전트 통신 프로토콜의 보안과 개인정보 보호는 실제 애플리케이션에서 핵심 문제입니다. 다음을 생각해 보세요.

   - 10.2.4절의 MCP 클라이언트 구현에서 에이전트는 MCP 서버가 제공하는 모든 도구를 호출할 수 있습니다. 이 설계에는 어떤 보안 위험이 있을까요? MCP 서버가 파일 삭제나 시스템 명령 실행 같은 위험한 작업을 제공한다면, 권한 제어 메커니즘을 어떻게 설계해야 할까요?
   - A2A와 ANP 프로토콜은 여러 에이전트 간 통신을 포함하며, 여기에는 사용자 개인정보, 영업 비밀 같은 민감한 정보가 포함될 수 있습니다. "종단 간 암호화" 솔루션을 설계해 보세요. 전송 중 메시지가 도청되거나 변조되지 않도록 보장하면서, 에이전트 신원 인증과 접근 제어를 지원해야 합니다.
   - 대규모 에이전트 네트워크에서 악성 에이전트는 허위 정보를 전송하거나 서비스 거부 공격을 수행하거나 다른 에이전트의 데이터를 훔칠 수 있습니다. "신뢰 평가 시스템"을 설계해 보세요. 과거 행동, 협업 품질, 커뮤니티 평가 등의 요소를 기반으로 각 에이전트의 신뢰도를 동적으로 평가하고, 그에 따라 통신 전략을 조정해야 합니다.

## 참고문헌

[1] Anthropic. (2024). *Model Context Protocol*. Retrieved October 7, 2025, from https://modelcontextprotocol.io/

[2] The A2A Project. (2025). *A2A Protocol: An open protocol for agent-to-agent communication*. Retrieved October 7, 2025, from https://a2a-protocol.org/

[3] Chang, G., Lin, E., Yuan, C., Cai, R., Chen, B., Xie, X., & Zhang, Y. (2025). *Agent Network Protocol technical white paper*. arXiv. https://doi.org/10.48550/arXiv.2508.00007


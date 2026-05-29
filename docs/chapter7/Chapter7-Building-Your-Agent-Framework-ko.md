# 제7장 자신만의 Agent 프레임워크 구축하기

이전 장들에서는 Agent의 기본 개념을 설명하고, 주류 프레임워크들이 제공하는 개발 편의성을 직접 경험해 보았습니다. 이번 장부터는 더 도전적이고 가치 있는 단계인 **처음부터 자신만의 Agent 프레임워크인 HelloAgents를 구축하는 과정**에 들어섭니다.

학습 과정의 연속성과 재현성을 보장하기 위해, HelloAgents는 버전 반복(iteration)을 통해 개발을 진행할 것입니다. 각 장에서는 이전 장의 내용을 기반으로 새로운 기능 모듈을 추가하고, Agent 관련 지식 요소를 통합하여 구현해 나갑니다. 최종적으로는 이렇게 직접 구축한 프레임워크를 활용하여 이 책의 후속 장에서 다룰 고급 애플리케이션 사례들을 효율적으로 구현할 것입니다.

## 7.1 전체 프레임워크 아키텍처 디자인

### 7.1.1 자신만의 Agent 프레임워크를 구축해야 하는 이유

현재 빠르게 발전하고 있는 Agent 기술 생태계에는 이미 시장에 출시된 검증된 Agent 프레임워크들이 많이 존재합니다. 그렇다면 왜 우리는 굳이 처음부터 새로운 프레임워크를 구축해야 할까요?

**(1) 시장 프레임워크의 빠른 반복과 한계점**

Agent 분야는 새로운 개념이 끊임없이 등장하며 급격하게 발전하는 영역입니다. 각 프레임워크는 저마다의 지향점과 Agent 디자인에 대한 이해를 담고 있지만, Agent의 핵심적인 지식 요소는 일관됩니다.

- **과도한 추상화의 복잡성**: 많은 프레임워크가 범용성을 추구하다 보니 수많은 추상화 계층과 설정 옵션을 도입하게 되었습니다.
LangChain을 예로 들면, 체인 호출 메커니즘은 유연하지만 초보자에게는 학습 곡선이 매우 가파릅니다.
간단한 작업을 수행하기 위해서도 이해해야 할 개념이 너무 많기 때문입니다.
- **빠른 반복으로 인한 불안정성**: 상용 프레임워크들은 시장 점유율을 차지하기 위해 API 인터페이스를 빈번하게 변경합니다.
이로 인해 개발자들은 버전이 업그레이드될 때마다 기존 코드가 작동하지 않는 문제에 직면하게 되며, 유지보수 비용이 지속적으로 높게 유지됩니다.
- **블랙박스 구현 로직**: 상당수 프레임워크는 핵심 로직을 너무 단단하게 캡슐화하여 개발자가 Agent의 내부 작동 메커니즘을 이해하기 어렵게 만들고,
깊이 있는 커스터마이징 기능을 제공하지 못합니다.
문제가 발생했을 때 공식 문서나 커뮤니티 지원에만 의존해야 하며,
특히 커뮤니티 활동이 활발하지 않은 경우 피드백을 받기까지 매우 오랜 시간이 걸려 개발 효율성에 부정적인 영향을 미칩니다.
- **복잡한 의존성**: 성숙한 프레임워크들은 대량의 의존성 패키지를 동반하여 설치 크기가 크고, 다른 프로젝트 코드와 협업할 때 의존성 충돌 문제를 일으킬 가능성이 있습니다.

**(2) 사용자에서 빌더로의 역량 도약**

자신만의 Agent 프레임워크를 직접 구축하는 것은 단순히 도구를 사용하는 '사용자'에서 도구를 만드는 '빌더'로 거듭나는 과정입니다. 이러한 변화가 가져다주는 가치는 장기적이고 강력합니다.

- **Agent 작동 원리에 대한 깊은 이해**: 각 컴포넌트를 직접 구현하면서 개발자는 Agent의 사고 과정, 도구 호출 메커니즘, 그리고 다양한 디자인 패턴의 장단점과 차이점을 진정으로 이해하게 됩니다.
- **완전한 제어 권한 확보**: 프레임워크를 직접 구축한다는 것은 모든 코드 라인을 완벽하게 제어할 수 있음을 의미합니다. 서드파티 프레임워크의 디자인 철학에 얽매이지 않고, 특정 요구사항에 맞춰 정밀한 튜닝을 수행할 수 있습니다.
- **시스템 디자인 역량 함양**: 프레임워크 구축 과정에는 모듈러 디자인, 인터페이스 추상화, 예외 처리 등 핵심적인 소프트웨어 공학 역량이 포함되어 있어 개발자의 장기적인 성장에 큰 도움이 됩니다.

**(3) 맞춤형 요구사항과 깊은 숙련도의 필요성**

실제 애플리케이션 환경에서 Agent에 대한 요구사항은 시나리오에 따라 크게 다르며, 종종 범용 프레임워크를 기반으로 한 2차 개발이 요구됩니다.

- **특정 도메인을 위한 최적화 요구**: 금융, 의료, 교육 등 수직적 도메인(vertical domains)에서는 최적화된 프롬프트 템플릿, 특수 도구 통합, 맞춤형 보안 전략이 필수적입니다.
- **성능 및 리소스의 정밀한 제어**: 프로덕션 환경에서는 응답 시간, 메모리 사용량, 동시 처리 역량 등에 대한 엄격한 요구조건이 따릅니다. 범용 프레임워크가 제공하는 일괄적인 솔루션은 이러한 미세한 요구사항을 충족하기 어렵습니다.
- **학습 및 교수 활동을 위한 투명성 요구**: 교육적인 관점에서 학습자는 Agent가 구축되는 모든 단계를 명확히 확인하고 다양한 패러다임의 작동 메커니즘을 이해하기를 기대합니다. 이를 위해서는 프레임워크의 관찰 가능성(observability)과 해석 가능성(interpretability)이 매우 높아야 합니다.

### 7.1.2 HelloAgents 프레임워크의 디자인 철학

새로운 Agent 프레임워크를 구축할 때 중요한 것은 기능의 개수가 아니라, 디자인 철학이 기존 프레임워크의 가려운 부분을 긁어줄 수 있는지 여부입니다.
HelloAgents 프레임워크의 디자인은 한 가지 핵심 질문을 중심으로 전개됩니다. **"어떻게 해야 학습자가 빠르게 시작하면서도 Agent의 작동 원리를 깊이 있게 이해할 수 있을까?"**

어떤 성숙한 프레임워크든 처음 접할 때는 풍부한 기능에 매료되기 쉽지만,
이내 한 가지 문제에 봉착하게 됩니다.
간단한 작업을 하나 수행하려 해도 Chain, Agent, Tool, Memory, Retriever 등 수십 가지의 서로 다른 개념을 이해해야 한다는 점입니다.
각 개념마다 추상화 계층이 존재하므로 학습 곡선이 극도로 가파릅니다. 이러한 복잡성은 강력한 기능을 제공하긴 하지만, 초보자에게는 장벽이 됩니다.
HelloAgents 프레임워크는 기능적 완결성과 학습 친화성 사이에서 균형을 찾고자 하며, 다음과 같은 네 가지 핵심 디자인 철학을 확립했습니다.

**(1) 경량성과 교육 친화성 간의 균형**

훌륭한 교육용 프레임워크는 완벽한 가독성을 갖추어야 합니다.
HelloAgents는 핵심 코드를 장별로 분리하여 구성하였으며, 한 가지 단순한 원칙을 따릅니다.
**"일정한 프로그래밍 기본기를 갖춘 개발자라면 누구나 합리적인 시간 내에 프레임워크의 작동 원리를 완벽히 이해할 수 있어야 한다.
"** 의존성 관리 측면에서 프레임워크는 미니멀리즘 전략을 취합니다. OpenAI 공식 SDK와 몇 가지 필수 기본 라이브러리를 제외하고는 무거운 의존성을 도입하지 않습니다.
문제가 발생했을 때 복잡한 의존 관계 속에서 답을 찾을 필요 없이, 프레임워크 자체의 코드를 직접 추적하여 원인을 진단할 수 있습니다.

**(2) 표준 API 기반의 실용적인 선택**

OpenAI의 API는 업계 표준이 되었으며, 거의 모든 주요 LLM 제공업체들이 이 인터페이스와 호환되도록 노력하고 있습니다. HelloAgents는 추상화된 독자적인 인터페이스를 재발명하는 대신, 이 표준 위에서 구축하는 것을 선택했습니다. 이 결정은 몇 가지 이유에서 비롯되었습니다. 첫째는 호환성 보장입니다. HelloAgents 사용법을 마스터하고 나면, 다른 프레임워크로 마이그레이션하거나 기존 프로젝트에 통합할 때 기본 API 호출 로직이 완벽하게 일치합니다. 둘째는 학습 비용의 절감입니다. 이미 익숙한 표준 인터페이스를 기반으로 모든 작업이 이루어지므로, 새로운 개념 모델을 배울 필요가 없습니다.

**(3) 단계별 학습 경로의 세심한 디자인**

HelloAgents는 명확한 학습 경로를 제공합니다. 각 장의 학습 코드는 pip를 통해 다운로드할 수 있는 역사적 버전으로 저장되므로 코드 사용 비용을 걱정할 필요가 없으며, 모든 핵심 기능은 직접 작성하게 됩니다. 이 디자인을 통해 자신의 필요와 페이스에 맞춰 나아갈 수 있습니다. 각 업그레이드는 개념적 비약이나 이해의 공백 없이 자연스럽게 이어집니다. 이 장의 내용은 이전 6개 장의 내용을 바탕으로 하고 있으며, 마찬가지로 향후 고급 지식을 학습하기 위한 프레임워크 기초를 튼튼히 다져 줍니다.

**(4) 통합된 "도구" 추상화: 모든 것은 도구이다**

가볍고 교육 친화적인 철학을 철저히 구현하기 위해, HelloAgents는 아키텍처 측면에서 중요한 단순화를 단행했습니다. **핵심 Agent 클래스를 제외하고는 모두 '도구(Tool)'로 취급한다는 점입니다.** 다른 여러 프레임워크에서 독립적으로 학습해야 하는 Memory, RAG (검색 증강 생성), RL (강화 학습), MCP (모델 컨텍스트 프로토콜) 등의 모듈이 HelloAgents에서는 모두 하나의 "도구"로 통일되게 추상화됩니다. 이러한 디자인의 본래 의도는 불필요한 추상화 계층을 제거하여, 학습자가 "Agent가 도구를 호출한다"는 가장 직관적인 핵심 로직으로 돌아갈 수 있게 함으로써 빠른 시작과 깊은 이해를 동시에 달성하는 데 있습니다.

### 7.1.3 이 장의 학습 목표

먼저 제7장의 핵심 학습 내용을 살펴보겠습니다.

```
hello-agents/
├── hello_agents/
│   │
│   ├── core/                     # 핵심 프레임워크 레이어
│   │   ├── agent.py              # Agent 기본 클래스
│   │   ├── llm.py                # HelloAgentsLLM 통합 인터페이스
│   │   ├── message.py            # 메시지 시스템
│   │   ├── config.py             # 설정 관리
│   │   └── exceptions.py         # 예외 시스템
│   │
│   ├── agents/                   # Agent 구현 레이어
│   │   ├── simple_agent.py       # SimpleAgent 구현
│   │   ├── react_agent.py        # ReActAgent 구현
│   │   ├── reflection_agent.py   # ReflectionAgent 구현
│   │   └── plan_solve_agent.py   # PlanAndSolveAgent 구현
│   │
│   ├── tools/                    # 도구 시스템 레이어
│   │   ├── base.py               # 도구 기본 클래스
│   │   ├── registry.py           # 도구 등록 메커니즘
│   │   ├── chain.py              # 도구 체인 관리 시스템
│   │   ├── async_executor.py     # 비동기 도구 실행기
│   │   └── builtin/              # 내장 도구 세트
│   │       ├── calculator.py     # 계산기 도구
│   │       └── search.py         # 검색 도구
└──
```

구체적인 코드를 작성하기 전에 명확한 아키텍처 청사진을 세워야 합니다. HelloAgents의 아키텍처 디자인은 "계층 간 디커플링, 단일 책임, 통합 인터페이스"라는 핵심 원칙을 따르며,
이는 코드를 체계적으로 조직하고 장별로 내용을 확장하기 쉽게 만들어 줍니다.

**빠른 시작: HelloAgents 프레임워크 설치**

독자들이 이번 장의 전체 기능을 빠르게 경험해 볼 수 있도록 직접 설치 가능한 Python 패키지를 제공합니다. 다음 명령어를 사용하여 이번 장에 해당하는 버전을 설치할 수 있습니다.

```bash
# hello-agents 프레임워크 코드 링크: https://github.com/jjyaoao/HelloAgents
# Python 버전은 >= 3.10이어야 합니다.
pip install "hello-agents==0.1.1"
```

이 장을 학습하는 방법은 두 가지가 있습니다:

1. **체험형 학습**: `pip`를 사용해 프레임워크를 직접 설치하고, 예제 코드를 실행하며 다양한 기능을 빠르게 체험합니다.
2. **심화 학습**: 이 장의 내용을 따라 처음부터 각 컴포넌트를 직접 구현하며 프레임워크의 디자인 개념과 구현 세부사항을 깊이 있게 이해합니다.

우선은 "체험 후 구현" 학습 경로를 따르는 것을 추천합니다. 이 장에서는 완전한 테스트 파일을 제공합니다. 핵심 기능을 재작성하고 테스트를 실행하여 구현이 올바른지 검증해 볼 수 있습니다. 이 학습 방식은 실용성과 학습 효과를 모두 보장합니다. 프레임워크의 구현 세부사항을 깊이 이해하거나 개발에 직접 참여하고 싶다면 [GitHub 리포지토리](https://github.com/jjyaoao/helloagents)를 방문해 보시기 바랍니다.

시작하기 전에, Hello-agents를 사용해 단 30초 만에 간단한 Agent를 구축하는 경험을 해보세요!

```python
# 같은 디렉토리에 있는 .env 파일에 LLM API를 설정합니다. 코드 폴더 안의 .env.example을 참고하거나 이전 장 실습에서 사용한 .env 파일을 재사용할 수 있습니다.
from hello_agents import SimpleAgent, HelloAgentsLLM
from dotenv import load_dotenv

# 환경 변수 로드
load_dotenv()

# LLM 인스턴스 생성 - 프러임워크가 자동으로 제공업체(provider)를 감지합니다.
llm = HelloAgentsLLM()

# 또는 제공업체를 수동으로 지정할 수 있습니다 (선택 사항)
# llm = HelloAgentsLLM(provider="modelscope")

# SimpleAgent 생성
agent = SimpleAgent(
    name="AI 비서",
    llm=llm,
    system_prompt="당신은 유용한 AI 비서입니다."
)

# 기본 대화 진행
response = agent.run("안녕하세요! 자기소개 부탁드려요.")
print(response)

# 도구 기능 추가 (선택 사항)
from hello_agents.tools import CalculatorTool
calculator = CalculatorTool()
# 호출하려면 7.4.1에서 MySimpleAgent를 구현해야 하며, 이후 장에서 이 호출 방식을 지원할 예정입니다.
agent.add_tool(calculator)

# 이제 도구를 사용할 수 있습니다.
response = agent.run("2 + 3 * 4 계산을 도와주세요.")
print(response)

# 대화 기록 보기
print(f"이전 메시지 개수: {len(agent.get_history())}")
```
## 7.2 HelloAgentsLLM 확장

이 섹션의 내용은 4.1.3절에서 작성했던 `HelloAgentsLLM`을 기반으로 한 반복적 업그레이드입니다. 이 기본적인 클라이언트를 다양한 모델 호출에 유연하게 대응할 수 있는 모델 호출 허브로 변환할 것입니다. 이 업그레이드는 주로 다음 세 가지 목표를 중심으로 이루어집니다:

1. **멀티 제공업체 지원**: OpenAI, ModelScope, Zhipu AI 등 다양한 주류 LLM 서비스 제공업체 간의 원활한 전환을 구현하여, 프레임워크가 특정 벤더에 종속되는 것을 방지합니다.
2. **로컬 모델 통합**: 데이터 프라이버시 및 비용 제어 요구사항을 충족하기 위해, 3.2.3절의 Hugging Face Transformers 솔루션을 보완할 수 있는 프로덕션급 로컬 배포 솔루션인 VLLM과 Ollama를 도입합니다.
3. **자동 감지 메커니즘**: 프레임워크가 환경 정보를 기반으로 사용 중인 LLM 서비스 유형을 지능적으로 유추하는 자동 인식 메커니즘을 구축하여 사용자의 설정 과정을 단순화합니다.

### 7.2.1 멀티 제공업체 지원

기존에 정의한 `HelloAgentsLLM` 클래스는 `api_key`와 `base_url`이라는 두 가지 핵심 매개변수를 통해 OpenAI 인터페이스와 호환되는 모든 서비스에 연결할 수 있습니다. 이는 이론적으로 범용성을 보장하지만, 실제 응용에서는 서비스 제공업체마다 환경 변수 명명법, 기본 API 주소, 권장 모델 등이 다릅니다. 사용자가 서비스 제공업체를 바꿀 때마다 코드를 수동으로 조회하고 수정해야 한다면 개발 효율성이 크게 저하될 것입니다. 이 문제를 해결하기 위해 `provider`를 도입합니다. 개선 아이디어는 `HelloAgentsLLM` 내부에서 다양한 서비스 제공업체의 설정 세부사항을 처리하도록 하여 사용자에게 일관되고 간결한 호출 경험을 제공하는 것입니다. 구체적인 구현 세부사항은 7.2.3절 "자동 감지 메커니즘"에서 자세히 다루기로 하고, 여기서는 이 메커니즘을 사용하여 프레임워크를 어떻게 확장하는지에 초점을 맞추겠습니다.

아래에서는 `HelloAgentsLLM`을 상속하여 ModelScope 플랫폼 지원을 추가하는 방법을 보여주겠습니다. 독자 여러분이 프레임워크를 '사용'하는 방법뿐만 아니라 '확장'하는 방법도 마스터하기를 바랍니다. 이미 설치된 라이브러리의 소스 코드를 직접 수정하는 것은 라이브러리 업그레이드를 어렵게 만들기 때문에 권장되지 않는 방식입니다.

**(1) 커스텀 LLM 클래스 생성 및 상속**

프로젝트 디렉토리에 `my_llm.py` 파일이 있다고 가정해 보겠습니다. 먼저 `hello_agents` 라이브러리에서 `HelloAgentsLLM` 기본 클래스를 임포트한 다음, 이를 상속받는 `MyLLM`이라는 새 클래스를 생성합니다.

```python
# my_llm.py
import os
from typing import Optional
from openai import OpenAI
from hello_agents import HelloAgentsLLM

class MyLLM(HelloAgentsLLM):
    """
    상속을 통해 ModelScope 지원을 추가하는 커스텀 LLM 클라이언트
    """
    pass # 우선 비워 둡니다.
```

**(2) `__init__` 메서드 오버라이딩을 통한 신규 제공업체 지원**

다음으로 `MyLLM` 클래스의 `__init__` 메서드를 오버라이딩합니다. 우리의 목표는 사용자가 `provider="modelscope"`를 전달하면 커스텀 로직을 실행하고, 그렇지 않으면 부모 클래스인 `HelloAgentsLLM` 내부의 원래 로직을 호출하여 OpenAI와 같은 다른 기본 제공업체를 계속 지원할 수 있도록 하는 것입니다.

```python
class MyLLM(HelloAgentsLLM):
    def __init__(
        self,
        model: Optional[str] = None,
        api_key: Optional[str] = None,
        base_url: Optional[str] = None,
        provider: Optional[str] = "auto",
        **kwargs
    ):
        # 처리하고자 하는 제공업체가 'modelscope'인지 확인합니다.
        if provider == "modelscope":
            print("커스텀 ModelScope Provider 사용 중")
            self.provider = "modelscope"

            # ModelScope 인증 자격 증명 해석
            self.api_key = api_key or os.getenv("MODELSCOPE_API_KEY")
            self.base_url = base_url or "https://api-inference.modelscope.cn/v1/"

            # 인증 키 존재 여부 검증
            if not self.api_key:
                raise ValueError("ModelScope API 키를 찾을 수 없습니다. MODELSCOPE_API_KEY 환경 변수를 설정해 주세요.")

            # 기본 모델 및 기타 매개변수 설정
            self.model = model or os.getenv("LLM_MODEL_ID") or "Qwen/Qwen2.5-VL-72B-Instruct"
            self.temperature = kwargs.get('temperature', 0.7)
            self.max_tokens = kwargs.get('max_tokens')
            self.timeout = kwargs.get('timeout', 60)

            # 확보한 매개변수로 OpenAI 클라이언트 인스턴스 생성
            self._client = OpenAI(api_key=self.api_key, base_url=self.base_url, timeout=self.timeout)

        else:
            # modelscope가 아닌 경우, 부모 클래스의 원래 로직을 호출하여 처리합니다.
            super().__init__(model=model, api_key=api_key, base_url=base_url, provider=provider, **kwargs)

```

이 코드는 "오버라이딩(overriding)" 기법의 핵심을 보여줍니다. `provider="modelscope"`인 경우를 가로채 특별히 처리하고, 그 외의 모든 경우는 `super().__init__(...)`를 통해 부모 클래스로 전달하여 프레임워크 고유의 기능을 모두 보존합니다.

**(3) 커스텀 `MyLLM` 클래스 사용**

이제 프로젝트 비즈니스 로직에서 순정 `HelloAgentsLLM`을 사용하는 것과 동일하게 직접 만든 `MyLLM` 클래스를 사용할 수 있습니다.

먼저 `.env` 파일에 ModelScope API 키를 설정합니다:

```bash
# .env 파일
MODELSCOPE_API_KEY="your-modelscope-api-key"
```

그 다음, 메인 프로그램에서 `MyLLM`을 임포트하여 사용합니다:

```python
# my_main.py
from dotenv import load_dotenv
from my_llm import MyLLM # 주의: 여기서 자체 작성한 클래스를 임포트합니다.

# 환경 변수 로드
load_dotenv()

# 오버라이딩한 클라이언트를 인스턴스화하고 제공업체 지정
llm = MyLLM(provider="modelscope")

# 메시지 준비
messages = [{"role": "user", "content": "안녕하세요, 자기소개 부탁드립니다."}]

# 호출 진행. think 및 기타 메서드는 부모 클래스에서 상속받으므로 오버라이딩할 필요가 없습니다.
response_stream = llm.think(messages)

# 응답 출력
print("ModelScope 응답:")
for chunk in response_stream:
    # chunk는 이미 my_llm 내부에서 출력되므로 여기서는 그냥 pass 합니다.
    # print(chunk, end="", flush=True)
    pass
```

위 단계를 통해 `hello-agents` 라이브러리의 소스 코드를 수정하지 않고도 새로운 기능을 성공적으로 확장했습니다. 이 방법은 코드의 간결성과 유지보수성을 보장할 뿐만 아니라, 향후 `hello-agents` 라이브러리가 업그레이드되더라도 개발자가 커스터마이징한 기능이 유실되지 않도록 지켜줍니다.

### 7.2.2 로컬 모델 호출

3.2.3절에서는 Hugging Face Transformers 라이브러리를 사용해 오픈소스 모델을 로컬에서 구동하는 방법을 배웠습니다. 이 방법은 입문 학습 및 기능 검증에 적합하지만, 대규모 동시 요청을 처리할 때 내부 구현의 성능 제약이 있어 프로덕션 환경의 첫 번째 선택지로는 적합하지 않습니다.

로컬 환경에서 프로덕션 수준의 고성능 모델 추론 서비스를 구현하기 위해, 오픈소스 커뮤니티는 VLLM 및 Ollama와 같은 뛰어난 도구들을 내놓았습니다. 이 도구들은 Continuous Batching 및 PagedAttention 등의 기술을 통해 모델 처리량(throughput)과 운영 효율성을 대폭 개선하며, 모델을 OpenAI 표준과 호환되는 API 서비스로 캡슐화해 줍니다. 즉, 우리는 이 도구들을 `HelloAgentsLLM`에 원활하게 통합할 수 있습니다.

**VLLM**

VLLM은 LLM 추론을 위해 설계된 고성능 Python 라이브러리입니다. PagedAttention과 같은 고급 기술을 활용하여 표준 Transformers 구현보다 수 배 높은 처리량을 달성할 수 있습니다.
로컬에서 VLLM 서비스를 배포하는 전체 단계는 다음과 같습니다:

먼저 사용자의 하드웨어 환경(특히 CUDA 버전)에 맞게 VLLM을 설치해야 합니다.
버전 미스매치 문제를 피하기 위해 [공식 문서](https://docs.vllm.ai/en/latest/getting_started/installation.html)를 참고하여 설치하는 것이 좋습니다.

```python
pip install vllm
```

설치가 완료되면 다음 명령어를 실행하여 OpenAI와 호환되는 API 서비스를 구동합니다.
VLLM은 지정된 모델의 가중치를 Hugging Face Hub에서 자동으로 다운로드합니다(로컬에 없는 경우).
여기서는 Qwen1.5-0.5B-Chat 모델을 예로 들어 보겠습니다.

```bash
# VLLM 서비스를 시작하고 Qwen1.5-0.5B-Chat 모델을 로드합니다.
python -m vllm.entrypoints.openai.api_server \
    --model Qwen/Qwen1.5-0.5B-Chat \
    --host 0.0.0.0 \
    --port 8000
```

서비스가 시작되면 `http://localhost:8000/v1` 주소로 OpenAI 호환 API가 노출됩니다.

**Ollama**

Ollama는 모델 다운로드, 설정, 서비스 구동을 단일 명령어로 통합하여 로컬 모델 관리 및 배포를 한층 더 간소화함으로써 빠른 시작에 매우 적합합니다. Ollama [공식 웹사이트](https://ollama.com)를 방문하여 해당 운영체제용 클라이언트를 다운로드하고 설치하세요.

설치 후 터미널을 열고 다음 명령어를 실행하여 모델을 다운로드하고 실행합니다(Llama 3 예시). Ollama는 모델 다운로드, 서비스 캡슐화 및 하드웨어 가속 설정을 자동으로 처리합니다.

```bash
# 첫 실행 시에는 모델이 자동으로 다운로드되며, 이후 실행 시에는 서비스가 즉시 시작됩니다.
ollama run llama3
```

터미널에 모델의 인터랙티브 프롬프트가 나타나면 서비스가 백그라운드에서 정상적으로 구동된 것입니다. Ollama는 기본적으로 `http://localhost:11434/v1` 주소에서 OpenAI 호환 API 인터페이스를 제공합니다.

**`HelloAgentsLLM`과 통합**

VLLM과 Ollama 모두 업계 표준 API를 따르기 때문에 `HelloAgentsLLM`에 이를 통합하는 것은 매우 간단합니다. 클라이언트를 인스턴스화할 때 새로운 `provider`로 지정해 주기만 하면 됩니다.

예를 들어, 로컬에서 실행 중인 **VLLM** 서비스에 연결하는 방법은 다음과 같습니다:

```python
llm_client = HelloAgentsLLM(
    provider="vllm",
    model="Qwen/Qwen1.5-0.5B-Chat", # 서비스를 구동할 때 지정한 모델명과 일치해야 합니다.
    base_url="http://localhost:8000/v1",
    api_key="vllm" # 로컬 서비스는 대개 실제 API Key를 요구하지 않으므로 비어 있지 않은 아무 문자열이나 입력하면 됩니다.
)
```

또는 환경 변수를 설정하고 클라이언트가 자동 감지하도록 하여 소스 코드 수정 없이 구현할 수 있습니다:

```bash
# .env 파일 설정
LLM_BASE_URL="http://localhost:8000/v1"
LLM_API_KEY="vllm"

# Python 코드에서 바로 인스턴스화
llm_client = HelloAgentsLLM() # 자동으로 vllm으로 감지됩니다.
```

마찬가지로 로컬 **Ollama** 서비스에 연결하는 것도 아주 단순합니다:

```python
llm_client = HelloAgentsLLM(
    provider="ollama",
    model="llama3", # `ollama run`에서 지정한 모델명과 일치해야 합니다.
    base_url="http://localhost:11434/v1",
    api_key="ollama" # 로컬 서비스이므로 임의의 키 값을 사용합니다.
)
```

이러한 단일화된 설계 덕분에 우리의 Agent 핵심 코드는 전혀 변경하지 않고도 클라우드 API와 로컬 모델 사이를 자유롭게 전환할 수 있습니다. 이는 향후 애플리케이션 개발, 배포, 비용 제어 및 데이터 프라이버시 보호에 큰 유연성을 부여합니다.

### 7.2.3 자동 감지 메커니즘

사용자의 설정 부담을 최소화하고 "설정보다 관례(convention over configuration)" 원칙을 따르기 위해, `HelloAgentsLLM` 내부에는 두 가지 핵심 보조 메서드인 `_auto_detect_provider`와 `_resolve_credentials`가 설계되어 있습니다. 이들은 함께 연동되며, `_auto_detect_provider`는 환경 정보를 기반으로 서비스 제공업체를 유추하고, `_resolve_credentials`는 해당 유추 결과를 바탕으로 세부 매개변수 설정을 완료합니다.

`_auto_detect_provider` 메서드는 다음 우선순위에 따라 환경 정보에서 서비스 제공업체를 자동으로 판별합니다:

1. **최우선 순위: 특정 서비스 제공업체의 환경 변수 확인** 이는 가장 직접적이고 신뢰할 수 있는 판단 근거입니다. 프레임워크는 `MODELSCOPE_API_KEY`, `OPENAI_API_KEY`, `ZHIPU_API_KEY` 등의 환경 변수가 존재하는지 순차적으로 검사합니다. 발견되는 즉시 해당 서비스 제공업체로 판정합니다.

2. **차선 순위: `base_url` 기준 판단** 사용자가 특정 서비스 제공업체의 키를 설정하지 않았지만 범용 `LLM_BASE_URL`을 설정한 경우, 프레임워크는 이 URL을 분석합니다.
   - **도메인 매칭**: URL에 `"api-inference.modelscope.cn"`, `"api.openai.com"` 등의 특징적인 문자열이 포함되어 있는지 확인하여 클라우드 서비스 제공업체를 식별합니다.
   - **포트 매칭**: URL에 `:11434` (Ollama), `:8000` (VLLM) 등 로컬 서비스의 표준 포트가 포함되어 있는지 확인하여 로컬 배포 솔루션을 식별합니다.

3. **보조 판단: API 키 형식 분석** 위의 두 가지 방법으로 판별할 수 없는 특정 경우, 프레임워크는 범용 환경 변수인 `LLM_API_KEY`의 포맷을 분석해 봅니다. 예를 들어, 일부 서비스 제공업체의 API 키는 고유한 접두사나 인코딩 형식을 지니고 있습니다. 다만 이 방식은 모호성(예: 여러 제공업체의 키 포맷이 유사함)이 있을 수 있어 우선순위가 낮으며 보조 수단으로만 사용됩니다.

핵심 코드는 다음과 같습니다:

```python
def _auto_detect_provider(self, api_key: Optional[str], base_url: Optional[str]) -> str:
    """
    LLM 제공업체를 자동으로 감지합니다.
    """
    # 1. 특정 제공업체의 환경 변수 확인 (최우선 순위)
    if os.getenv("MODELSCOPE_API_KEY"): return "modelscope"
    if os.getenv("OPENAI_API_KEY"): return "openai"
    if os.getenv("ZHIPU_API_KEY"): return "zhipu"
    # ... 기타 서비스 제공업체 환경 변수 검사

    # 범용 환경 변수 획득
    actual_api_key = api_key or os.getenv("LLM_API_KEY")
    actual_base_url = base_url or os.getenv("LLM_BASE_URL")

    # 2. base_url 기반 판단
    if actual_base_url:
        base_url_lower = actual_base_url.lower()
        if "api-inference.modelscope.cn" in base_url_lower: return "modelscope"
        if "open.bigmodel.cn" in base_url_lower: return "zhipu"
        if "localhost" in base_url_lower or "127.0.0.1" in base_url_lower:
            if ":11434" in base_url_lower: return "ollama"
            if ":8000" in base_url_lower: return "vllm"
            return "local" # 기타 로컬 포트

    # 3. API 키 포맷 기반의 보조 판단
    if actual_api_key:
        if actual_api_key.startswith("ms-"): return "modelscope"
        # ... 기타 키 포맷 판별

    # 4. 판별 불가 시 기본값 'auto' 반환, 범용 설정 적용
    return "auto"
```

제공업체(`provider`)가 결정되고 나면(사용자 지정 혹은 자동 감지), `_resolve_credentials` 메서드가 실행되어 서비스 제공업체별 세부 설정을 처리합니다. 이 메서드는 `provider` 값에 따라 해당하는 환경 변수를 능동적으로 탐색하고 적절한 기본 `base_url`을 세팅합니다. 핵심 구현의 일부는 다음과 같습니다:

```python
def _resolve_credentials(self, api_key: Optional[str], base_url: Optional[str]) -> tuple[str, str]:
    """provider에 근거하여 API 키와 base_url을 해석합니다."""
    if self.provider == "openai":
        resolved_api_key = api_key or os.getenv("OPENAI_API_KEY") or os.getenv("LLM_API_KEY")
        resolved_base_url = base_url or os.getenv("LLM_BASE_URL") or "https://api.openai.com/v1"
        return resolved_api_key, resolved_base_url

    elif self.provider == "modelscope":
        resolved_api_key = api_key or os.getenv("MODELSCOPE_API_KEY") or os.getenv("LLM_API_KEY")
        resolved_base_url = base_url or os.getenv("LLM_BASE_URL") or "https://api-inference.modelscope.cn/v1/"
        return resolved_api_key, resolved_base_url

    # ... 기타 서비스 제공업체 처리 로직
```

간단한 예시를 통해 자동 감지 기능이 주는 편리함을 경험해 보겠습니다. 로컬 Ollama 서비스를 사용하려는 사용자는 `.env` 파일을 다음과 같이 구성하기만 하면 됩니다:

```bash
LLM_BASE_URL="http://localhost:11434/v1"
LLM_MODEL_ID="llama3"
```

`LLM_API_KEY`를 설정하거나 코드 내에서 `provider`를 지정할 필요가 전혀 없습니다. 그런 다음 Python 코드에서 간단히 `HelloAgentsLLM`을 호출합니다:

```python
from dotenv import load_dotenv
from hello_agents import HelloAgentsLLM

load_dotenv()

# provider를 인자로 넘기지 않아도 프레임워크가 자동으로 감지합니다.
llm = HelloAgentsLLM()
# 프레임워크 내부 로그에 provider가 'ollama'로 감지되었다고 표시됩니다.

# 이후 호출 방식은 완전히 동일합니다.
messages = [{"role": "user", "content": "안녕하세요!"}]
for chunk in llm.think(messages):
    print(chunk, end="")

```

이 과정에서 `_auto_detect_provider` 메서드는 `LLM_BASE_URL` 내의 `"localhost"`와 `:11434`를 파싱하여 제공업체가 `"ollama"`임을 성공적으로 유추합니다. 이어서 `_resolve_credentials` 메서드가 Ollama에 맞는 정확한 기본 매개변수들을 매핑해 줍니다.

4.1.3절의 초기 구현과 비교했을 때, 현재의 `HelloAgentsLLM`은 다음과 같은 상당한 이점을 제공합니다:

<div align="center">
  <p>표 7.1 HelloAgentLLM 버전별 기능 비교</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/7-figures/table-01.png" alt="" width="90%"/>
</div>

표 7.1에서 볼 수 있듯이, 이러한 진화는 프레임워크 설계의 중요한 원칙인 **"단순하게 시작하여 점진적으로 완성해 나간다"**를 잘 구현하고 있습니다. 우리는 인터페이스의 단순함을 유지하면서도 기능의 완결성을 높였습니다.
## 7.3 프레임워크 인터페이스 구현

이전 섹션에서는 대형 언어 모델과의 통신이라는 핵심 과제를 해결해 주는 핵심 컴포넌트인 `HelloAgentsLLM`을 구축했습니다. 하지만 프레임워크가 실질적으로 작동하고 데이터 흐름을 관리하며 설정을 제어하고 예외를 처리하기 위해서는, 상위 애플리케이션 구축을 위한 명확하고 통합된 구조를 지원하는 일련의 인터페이스와 컴포넌트가 더 필요합니다. 이 섹션에서는 다음 세 가지 핵심 파일을 다룹니다:

- `message.py`: 프레임워크 내의 통합 메시지 포맷을 정의하여 Agent와 모델 간의 정보 전달 규격을 보장합니다.
- `config.py`: 중앙 집중식 설정 관리 솔루션을 제공하여 프레임워크 동작을 쉽게 조정하고 확장할 수 있도록 돕습니다.
- `agent.py`: 모든 Agent의 추상 기본 클래스(`Agent`)를 정의하여 향후 다양한 유형의 Agent를 구현할 수 있는 일관된 인터페이스와 명세를 제공합니다.

### 7.3.1 Message 클래스

Agent와 LLM 간의 인터랙션 과정에서 대화 기록은 매우 중요한 맥락(context) 정보입니다. 이 정보를 표준화된 방식으로 관리하기 위해 간단한 `Message` 클래스를 설계했습니다.
이 클래스는 후속 장인 컨텍스트 엔지니어링 장에서 더욱 확장될 예정입니다.

```python
"""메시지 시스템"""
from typing import Optional, Dict, Any, Literal
from datetime import datetime
from pydantic import BaseModel

# 메시지 역할 유형을 정의하여 허용되는 값을 제한합니다.
MessageRole = Literal["user", "assistant", "system", "tool"]

class Message(BaseModel):
    """메시지 클래스"""

    content: str
    role: MessageRole
    timestamp: datetime = None
    metadata: Optional[Dict[str, Any]] = None

    def __init__(self, content: str, role: MessageRole, **kwargs):
        super().__init__(
            content=content,
            role=role,
            timestamp=kwargs.get('timestamp', datetime.now()),
            metadata=kwargs.get('metadata', {})
        )

    def to_dict(self) -> Dict[str, Any]:
        """딕셔너리 포맷으로 변환 (OpenAI API 포맷)"""
        return {
            "role": self.role,
            "content": self.content
        }

    def __str__(self) -> str:
        return f"[{self.role}] {self.content}"
```

이 클래스의 설계에는 몇 가지 핵심 포인트가 있습니다. 첫째, `typing.Literal`을 사용하여 `role` 필드의 값을 `"user"`, `"assistant"`, `"system"`, `"tool"`의 네 가지 유형으로 엄격히 제한했습니다.
이는 OpenAI API 사양과 직접 대응되어 타입 안정성을 보장합니다.
둘째, 핵심 필드인 `content`와 `role` 외에도 `timestamp`와 `metadata`를 추가하여 로깅 및 향후 기능 확장을 위한 공간을 마련해 두었습니다.
마지막으로 `to_dict()` 메서드는 내부에서 사용하는 `Message` 객체를 OpenAI API와 호환되는 딕셔너리 포맷으로 변환하는 핵심 기능을 담당하며, "내부는 풍부하게, 외부는 호환성 있게"라는 디자인 원칙을 잘 구현하고 있습니다.

### 7.3.2 Config 클래스

`Config` 클래스의 역할은 코드 내부의 하드코딩된 설정 매개변수들을 중앙 집중화하고 환경 변수로부터 값을 읽어오는 작업을 지원하는 것입니다.

```python
"""설정 관리"""
import os
from typing import Optional, Dict, Any
from pydantic import BaseModel

class Config(BaseModel):
    """HelloAgents 설정 클래스"""

    # LLM 설정
    default_model: str = "gpt-3.5-turbo"
    default_provider: str = "openai"
    temperature: float = 0.7
    max_tokens: Optional[int] = None

    # 시스템 설정
    debug: bool = False
    log_level: str = "INFO"

    # 기타 설정
    max_history_length: int = 100

    @classmethod
    def from_env(cls) -> "Config":
        """환경 변수로부터 설정을 생성합니다."""
        return cls(
            debug=os.getenv("DEBUG", "false").lower() == "true",
            log_level=os.getenv("LOG_LEVEL", "INFO"),
            temperature=float(os.getenv("TEMPERATURE", "0.7")),
            max_tokens=int(os.getenv("MAX_TOKENS")) if os.getenv("MAX_TOKENS") else None,
        )

    def to_dict(self) -> Dict[str, Any]:
        """딕셔너리로 변환"""
        return self.dict()
```

첫째, 설정 항목들을 `LLM 설정`, `시스템 설정` 등으로 논리적으로 구분하여 한눈에 파악하기 쉬운 구조로 만들었습니다.
둘째, 각 설정 항목이 합리적인 기본값을 가지고 있어 설정이 없어도 프레임워크가 정상 작동하도록 보장합니다.
가장 핵심은 `from_env()` 클래스 메서드입니다. 이 메서드는 사용자가 코드를 수정하지 않고도 환경 변수를 지정하여 기본 설정을 쉽게 오버라이딩할 수 있도록 지원하며, 이는 다양한 서버나 배포 환경에 적용할 때 특히 유용합니다.

### 7.3.3 Agent 추상 기본 클래스

`Agent` 클래스는 전체 프레임워크의 최상위 추상화 레이어입니다. 이는 Agent가 지녀야 할 공통 동작과 속성을 정의할 뿐,
구체적인 구현 방식은 신경 쓰지 않습니다. Python의 `abc` (Abstract Base Classes) 모듈을 이용해 이를 구현함으로써, 향후 구현할 모든 구체적인 Agent(예: `SimpleAgent`, `ReActAgent` 등)들이 동일한 "인터페이스 사양"을 강제적으로 따르도록 설계했습니다.

```python
"""Agent 기본 클래스"""
from abc import ABC, abstractmethod
from typing import Optional, Any
from .message import Message
from .llm import HelloAgentsLLM
from .config import Config

class Agent(ABC):
    """Agent 기본 클래스"""

    def __init__(
        self,
        name: str,
        llm: HelloAgentsLLM,
        system_prompt: Optional[str] = None,
        config: Optional[Config] = None
    ):
        self.name = name
        self.llm = llm
        self.system_prompt = system_prompt
        self.config = config or Config()
        self._history: list[Message] = []

    @abstractmethod
    def run(self, input_text: str, **kwargs) -> str:
        """Agent 실행 메서드"""
        pass

    def add_message(self, message: Message):
        """대화 기록에 메시지 추가"""
        self._history.append(message)

    def clear_history(self):
        """대화 기록 초기화"""
        self._history.clear()

    def get_history(self) -> list[Message]:
        """대화 기록 가져오기"""
        return self._history.copy()

    def __str__(self) -> str:
        return f"Agent(name={self.name}, provider={self.llm.provider})"
```

이 클래스의 설계는 객체지향 프로그래밍의 추상화 원칙을 잘 보여줍니다. 첫째, `ABC`를 상속하여 인스턴스를 직접 생성할 수 없는 추상 클래스로 정의했습니다. 생성자인 `__init__`은 Agent의 핵심 의존성(이름, LLM 인스턴스, 시스템 프롬프트, 설정)을 명확하게 정의합니다. 가장 중요한 부분은 `@abstractmethod` 데코레이터가 붙은 `run` 메서드로, 이 메서드는 모든 서브클래스가 강제로 구현하도록 함으로써 모든 Agent가 통합된 실행 진입점(entry point)을 갖도록 보장합니다. 또한 기본 클래스는 대화 기록을 관리할 수 있는 공통 메서드를 제공하며, 이는 `Message` 클래스와 유기적으로 연계되어 컴포넌트 간의 긴밀한 통합을 구현합니다.

이로써 우리는 `HelloAgents` 프레임워크의 핵심 기초 컴포넌트 설계를 완료했습니다.
## 7.4 Agent 패러다임의 프레임워크 구현

이 섹션의 내용은 4장에서 작성했던 세 가지 클래식 Agent 패러다임(ReAct, Plan-and-Solve, Reflection)을 기반으로 프레임워크 리팩터링을 진행하고,
기본적인 대화 패러다임으로 SimpleAgent를 추가하는 내용입니다. 우리는 이 독립적인 Agent 구현체들을 통일된 아키텍처 기반의 프레임워크 컴포넌트로 전환할 것입니다. 이번 리팩터링은 다음 세 가지 핵심 목표를 중심으로 진행됩니다:

1. **프롬프트 엔지니어링의 체계적인 개선**: 4장의 프롬프트를 깊이 있게 최적화하여 특정 작업 지향형 설계에서 일반화된(generalized) 설계로 전환하고, 포맷 제약 사항과 역할 정의를 강화합니다.
2. **인터페이스 및 포맷의 표준화와 통일**: 통일된 Agent 기본 클래스와 표준 실행 인터페이스를 설정하여 모든 Agent가 동일한 초기화 매개변수, 메서드 시그니처 및 대화 기록 관리 메커니즘을 따르도록 합니다.
3. **고도의 설정이 가능한 커스터마이징 역량**: 사용자가 직접 정의하는 프롬프트 템플릿, 설정 매개변수, 그리고 실행 전략을 지원합니다.

### 7.4.1 SimpleAgent

SimpleAgent는 가장 기본적인 Agent 구현체로, 프레임워크 기반 위에서 대화형 Agent를 구축하는 방법을 보여줍니다.
기존 `SimpleAgent` 클래스를 확장하고 핵심 메서드를 오버라이딩하여 확장성이 뛰어난 버전을 만들어 보겠습니다. 먼저 프로젝트 디렉토리에 `my_simple_agent.py` 파일을 생성합니다.

```python
# my_simple_agent.py
from typing import Optional, Iterator
from hello_agents import SimpleAgent, HelloAgentsLLM, Config, Message

class MySimpleAgent(SimpleAgent):
    """
    재작성된 단순 대화형 Agent
    SimpleAgent를 상속하여 커스텀 Agent를 구축하는 방법을 보여줍니다.
    """

    def __init__(
        self,
        name: str,
        llm: HelloAgentsLLM,
        system_prompt: Optional[str] = None,
        config: Optional[Config] = None,
        tool_registry: Optional['ToolRegistry'] = None,
        enable_tool_calling: bool = True
    ):
        super().__init__(name, llm, system_prompt, config)
        self.tool_registry = tool_registry
        self.enable_tool_calling = enable_tool_calling and tool_registry is not None
        print(f"✅ {name} 초기화 완료, 도구 호출: {'활성화됨' if self.enable_tool_calling else '비활성화됨'}")
```

다음으로 `run` 메서드를 오버라이딩합니다. SimpleAgent는 선택적으로 도구 호출 기능을 지원하여 후속 장에서의 확장을 용이하게 돕습니다:

```python
# my_simple_agent.py에 이어서 작성
import re

class MySimpleAgent(SimpleAgent):
    # ... 이전 __init__ 메서드 생략

    def run(self, input_text: str, max_tool_iterations: int = 3, **kwargs) -> str:
        """
        재작성된 run 메서드 - 기본적인 대화 로직을 구현하며, 선택적으로 도구 호출을 지원합니다.
        """
        print(f"🤖 {self.name}이(가) 처리 중: {input_text}")

        # 메시지 목록 빌드
        messages = []

        # 시스템 메시지 추가 (도구 정보가 포함될 수 있음)
        enhanced_system_prompt = self._get_enhanced_system_prompt()
        messages.append({"role": "system", "content": enhanced_system_prompt})

        # 대화 기록 메시지 추가
        for msg in self._history:
            messages.append({"role": msg.role, "content": msg.content})

        # 현재 사용자 메시지 추가
        messages.append({"role": "user", "content": input_text})

        # 도구 호출이 활성화되지 않은 경우, 단순 대화 로직 사용
        if not self.enable_tool_calling:
            response = self.llm.invoke(messages, **kwargs)
            self.add_message(Message(input_text, "user"))
            self.add_message(Message(response, "assistant"))
            print(f"✅ {self.name} 응답 완료")
            return response

        # 다중 라운드 도구 호출을 지원하는 로직 실행
        return self._run_with_tools(messages, input_text, max_tool_iterations, **kwargs)

    def _get_enhanced_system_prompt(self) -> str:
        """도구 정보가 포함된 강화된 시스템 프롬프트를 빌드합니다."""
        base_prompt = self.system_prompt or "당신은 유용한 AI 비서입니다."

        if not self.enable_tool_calling or not self.tool_registry:
            return base_prompt

        # 사용 가능한 도구 설명 가져오기
        tools_description = self.tool_registry.get_tools_description()
        if not tools_description or tools_description == "No tools available":
            return base_prompt

        tools_section = "\n\n## 사용 가능한 도구\n"
        tools_section += "질문에 답하기 위해 다음 도구들을 사용할 수 있습니다:\n"
        tools_section += tools_description + "\n"

        tools_section += "\n## 도구 호출 형식\n"
        tools_section += "도구를 사용해야 할 때는 반드시 다음 형식을 준수해 주세요:\n"
        tools_section += "`[TOOL_CALL:{tool_name}:{parameters}]`\n"
        tools_section += "예: `[TOOL_CALL:search:Python 프로그래밍]` 또는 `[TOOL_CALL:memory:recall=사용자 정보]`\n\n"
        tools_section += "도구 실행 결과는 대화에 자동으로 삽입되며, 그 결과를 바탕으로 계속 답변을 이어갈 수 있습니다.\n"

        return base_prompt + tools_section
```

이제 도구 호출의 핵심 로직을 구현합니다:

```python
# my_simple_agent.py에 이어서 작성
class MySimpleAgent(SimpleAgent):
    # ... 이전 메서드 생략

    def _run_with_tools(self, messages: list, input_text: str, max_tool_iterations: int, **kwargs) -> str:
        """도구 호출을 지원하는 실행 로직"""
        current_iteration = 0
        final_response = ""

        while current_iteration < max_tool_iterations:
            # LLM 호출
            response = self.llm.invoke(messages, **kwargs)

            # 도구 호출이 있는지 검사
            tool_calls = self._parse_tool_calls(response)

            if tool_calls:
                print(f"🔧 {len(tool_calls)}개의 도구 호출이 감지되었습니다.")
                # 모든 도구 호출을 실행하고 결과 수집
                tool_results = []
                clean_response = response

                for call in tool_calls:
                    result = self._execute_tool_call(call['tool_name'], call['parameters'])
                    tool_results.append(result)
                    # 응답에서 도구 호출 마커를 제거
                    clean_response = clean_response.replace(call['original'], "")

                # 도구 호출을 수행했던 조각이 반영된 메시지 추가
                messages.append({"role": "assistant", "content": clean_response})

                # 도구 실행 결과 추가
                tool_results_text = "\n\n".join(tool_results)
                messages.append({"role": "user", "content": f"도구 실행 결과:\n{tool_results_text}\n\n이 결과들을 종합하여 최종 답변을 작성해 주세요."})

                current_iteration += 1
                continue

            # 도구 호출이 더 이상 없으면 이것이 최종 답변입니다.
            final_response = response
            break

        # 최대 반복 횟수를 초과했으나 최종 응답이 없는 경우, 마지막으로 한번 더 LLM을 호출하여 강제 종료
        if current_iteration >= max_tool_iterations and not final_response:
            final_response = self.llm.invoke(messages, **kwargs)

        # 대화 기록에 저장
        self.add_message(Message(input_text, "user"))
        self.add_message(Message(final_response, "assistant"))
        print(f"✅ {self.name} 응답 완료")

        return final_response

    def _parse_tool_calls(self, text: str) -> list:
        """텍스트 안에서 도구 호출 마커를 파싱합니다."""
        pattern = r'\[TOOL_CALL:([^:]+):([^\]]+)\]'
        matches = re.findall(pattern, text)

        tool_calls = []
        for tool_name, parameters in matches:
            tool_calls.append({
                'tool_name': tool_name.strip(),
                'parameters': parameters.strip(),
                'original': f'[TOOL_CALL:{tool_name}:{parameters}]'
            })

        return tool_calls

    def _execute_tool_call(self, tool_name: str, parameters: str) -> str:
        """도구를 실행합니다."""
        if not self.tool_registry:
            return "❌ 오류: 도구 등록기(Tool registry)가 구성되지 않았습니다."

        try:
            # 매개변수 파싱 진행
            param_dict = self._parse_tool_parameters(tool_name, parameters)
            return self.tool_registry.execute_tool(tool_name, param_dict)
        except Exception as e:
            return f"❌ 도구 실행 중 예외 발생: {str(e)}"
```

커스텀 Agent에 스트리밍 응답 기능 및 편의 메서드도 추가할 수 있습니다:

```python
# my_simple_agent.py에 이어서 작성
class MySimpleAgent(SimpleAgent):
    # ... 이전 메서드들 생략

    def _parse_tool_parameters(self, tool_name: str, parameters: str) -> dict:
        """도구 매개변수를 지능적으로 파싱합니다."""
        param_dict = {}

        if '=' in parameters:
            # 포맷: key=value 또는 action=search,query=Python
            if ',' in parameters:
                # 다중 매개변수: action=search,query=Python,limit=3
                pairs = parameters.split(',')
                for pair in pairs:
                    if '=' in pair:
                        key, value = pair.split('=', 1)
                        param_dict[key.strip()] = value.strip()
            else:
                # 단일 매개변수: key=value
                key, value = parameters.split('=', 1)
                param_dict[key.strip()] = value.strip()
        else:
            # 매개변수가 키-값 형태가 아닐 경우 도구 유형에 맞게 지능적으로 해석
            if tool_name == 'search':
                param_dict = {'query': parameters}
            elif tool_name == 'memory':
                param_dict = {'action': 'search', 'query': parameters}
            else:
                param_dict = {'input': parameters}

        return param_dict

    def stream_run(self, input_text: str, **kwargs) -> Iterator[str]:
        """
        커스텀 스트리밍 실행 메서드
        """
        print(f"🌊 {self.name} 스트리밍 처리 시작: {input_text}")

        messages = []

        if self.system_prompt:
            messages.append({"role": "system", "content": self.system_prompt})

        for msg in self._history:
            messages.append({"role": msg.role, "content": msg.content})

        messages.append({"role": "user", "content": input_text})

        # LLM 스트리밍 호출
        full_response = ""
        print("📝 실시간 응답: ", end="")
        for chunk in self.llm.stream_invoke(messages, **kwargs):
            full_response += chunk
            print(chunk, end="", flush=True)
            yield chunk

        print()  # 개행

        # 완성된 대화를 히스토리에 저장
        self.add_message(Message(input_text, "user"))
        self.add_message(Message(full_response, "assistant"))
        print(f"✅ {self.name} 스트리밍 응답 완료")

    def add_tool(self, tool) -> None:
        """Agent에 도구를 추가합니다 (편의 메서드)."""
        if not self.tool_registry:
            from hello_agents import ToolRegistry
            self.tool_registry = ToolRegistry()
            self.enable_tool_calling = True

        self.tool_registry.register_tool(tool)
        print(f"🔧 도구 '{tool.name}'가 추가되었습니다.")

    def has_tools(self) -> bool:
        """사용 가능한 도구가 있는지 검사합니다."""
        return self.enable_tool_calling and self.tool_registry is not None

    def remove_tool(self, tool_name: str) -> bool:
        """도구를 제거합니다 (편의 메서드)."""
        if self.tool_registry:
            self.tool_registry.unregister(tool_name)
            return True
        return False

    def list_tools(self) -> list:
        """모든 사용 가능한 도구를 나열합니다."""
        if self.tool_registry:
            return self.tool_registry.list_tools()
        return []
```

이를 테스트할 수 있는 `test_simple_agent.py` 테스트 파일을 작성합니다:

```python
# test_simple_agent.py
from dotenv import load_dotenv
from hello_agents import HelloAgentsLLM, ToolRegistry
from hello_agents.tools import CalculatorTool
from my_simple_agent import MySimpleAgent

# 환경 변수 로드
load_dotenv()

# LLM 인스턴스 생성
llm = HelloAgentsLLM()

# 테스트 1: 기본 대화형 Agent (도구 없음)
print("=== 테스트 1: 기본 대화 ===")
basic_agent = MySimpleAgent(
    name="일반 비서",
    llm=llm,
    system_prompt="당신은 친절한 AI 비서입니다. 명료하고 간결하게 답변해 주세요."
)

response1 = basic_agent.run("안녕하세요, 자기소개 부탁드려요.")
print(f"기본 대화 응답: {response1}\n")

# 테스트 2: 도구가 포함된 Agent
print("=== 테스트 2: 도구 강화 대화 ===")
tool_registry = ToolRegistry()
calculator = CalculatorTool()
tool_registry.register_tool(calculator)

enhanced_agent = MySimpleAgent(
    name="도구 활용 비서",
    llm=llm,
    system_prompt="당신은 도구를 활용해 사용자를 돕는 지능형 비서입니다.",
    tool_registry=tool_registry,
    enable_tool_calling=True
)

response2 = enhanced_agent.run("15 * 8 + 32 계산을 도와주세요.")
print(f"도구 활용 응답: {response2}\n")

# 테스트 3: 스트리밍 응답
print("=== 테스트 3: 스트리밍 응답 ===")
print("스트리밍 응답 시작: ", end="")
for chunk in basic_agent.stream_run("인공지능이 무엇인지 설명해 주세요."):
    pass  # 스트리밍 결과는 stream_run 내부에서 실시간으로 출력됩니다.

# 테스트 4: 도구의 동적 추가 및 관리
print("\n=== 테스트 4: 동적 도구 관리 ===")
print(f"도구 추가 전 활성화 여부: {basic_agent.has_tools()}")
basic_agent.add_tool(calculator)
print(f"도구 추가 후 활성화 여부: {basic_agent.has_tools()}")
print(f"등록된 도구 목록: {basic_agent.list_tools()}")

# 대화 히스토리 확인
print(f"\n대화 기록: 총 {len(basic_agent.get_history())}개의 메시지")
```

이 섹션에서는 `Agent` 기본 클래스를 상속함으로써 프레임워크 명세를 엄격히 준수하고 완전한 기능을 갖춘 기본적인 대화형 Agent인 `MySimpleAgent`를 구축했습니다. 이 Agent는 기본 대화뿐만 아니라 선택적 도구 호출 기능, 스트리밍 응답, 그리고 편리한 도구 동적 관리 기능을 내장하고 있습니다.

### 7.4.2 ReActAgent

프레임워크 기반의 ReActAgent는 핵심 실행 로직을 기존대로 유지하면서, 프롬프트를 고도화하고 프레임워크의 도구 시스템과 연동하는 설계를 통해 코드 구조와 유지보수성을 향상시켰습니다.

**(1) 프롬프트 템플릿 고도화**

기존의 포맷 요구사항을 그대로 보존하되, Agent가 혼동하지 않고 한 번에 하나의 단계만 처리할 수 있도록 명문화하고 두 가지 유형의 Action 시나리오를 명확히 제한합니다.

```python
MY_REACT_PROMPT = """당신은 추론과 행동 능력을 겸비한 AI 비서입니다. 문제를 분석하여 Thought를 정리하고, 적절한 도구를 호출하여 정보를 획득(Observation)한 뒤, 최종적으로 정확한 답변을 제공합니다.

## 사용 가능한 도구
{tools}

## 작동 순서 (Workflow)
반드시 다음 포맷을 엄격히 준수하여 응답해 주세요. 한 번의 호출에는 단 하나의 단계만 수행해야 합니다:

Thought: 현재 직면한 문제를 분석하고 필요한 정보나 수행할 행동을 정리합니다.
Action: 취할 행동을 선택합니다. 다음 중 하나여야 합니다:
- `{{tool_name}}[{{tool_input}}]` - 정의된 도구를 호출합니다.
- `Finish[최종 답변]` - 충분한 정보를 확보하여 사용자에게 최종 답변을 제공할 때 사용합니다.

## 주의 사항
1. 매 응답은 Thought와 Action 파트를 모두 포함해야 합니다.
2. 도구 호출 형식은 반드시 tool_name[parameters] 형식을 엄격히 지켜야 합니다.
3. 충분한 정보를 바탕으로 확신이 설 때만 Finish를 선언하세요.
4. 도구가 반환한 정보가 부족하다면 다른 매개변수나 다른 도구를 사용하여 추론을 이어가세요.

## 현재 작업
**질문:** {question}

## 실행 이력 (History)
{history}

이제 추론과 행동을 시작해 주세요:
"""
```

**(2) 재작성된 ReActAgent의 전체 구현**

ReActAgent를 재작성하기 위해 `my_react_agent.py` 파일을 생성합니다:

```python
# my_react_agent.py
import re
from typing import Optional, List, Tuple
from hello_agents import ReActAgent, HelloAgentsLLM, Config, Message, ToolRegistry

class MyReActAgent(ReActAgent):
    """
    재작성된 ReAct Agent - 추론(Reasoning)과 행동(Acting)을 결합한 Agent
    """

    def __init__(
        self,
        name: str,
        llm: HelloAgentsLLM,
        tool_registry: ToolRegistry,
        system_prompt: Optional[str] = None,
        config: Optional[Config] = None,
        max_steps: int = 5,
        custom_prompt: Optional[str] = None
    ):
        super().__init__(name, llm, system_prompt, config)
        self.tool_registry = tool_registry
        self.max_steps = max_steps
        self.current_history: List[str] = []
        self.prompt_template = custom_prompt if custom_prompt else MY_REACT_PROMPT
        print(f"✅ {name} 초기화 완료, 최대 추론 단계: {max_steps}")
```

각 매개변수의 의미는 다음과 같습니다:

- `name`: Agent 이름.
- `llm`: 대형 언어 모델과의 통신을 담당하는 `HelloAgentsLLM` 인스턴스.
- `tool_registry`: Agent가 사용할 수 있는 도구를 관리하고 실행하는 `ToolRegistry` 인스턴스.
- `system_prompt`: Agent의 역할과 행위 규칙을 세팅하는 시스템 프롬프트.
- `config`: 프레임워크 레벨 설정을 주입하기 위한 설정 객체.
- `max_steps`: ReAct 루프가 무한 루프에 빠지는 것을 방지하기 위한 최대 실행 단계 수.
- `custom_prompt`: 기본 ReAct 프롬프트를 대체할 수 있는 커스텀 프롬프트 템플릿.

프레임워크 기반의 ReActAgent는 다음과 같이 추론-행동 실행 프로세스를 명확하게 한 단계씩 실행합니다:

```python
    def run(self, input_text: str, **kwargs) -> str:
        """ReAct Agent 실행"""
        self.current_history = []
        current_step = 0

        print(f"\n🤖 {self.name} 질문 처리 시작: {input_text}")

        while current_step < self.max_steps:
            current_step += 1
            print(f"\n--- {current_step}단계 ---")

            # 1. 프롬프트 생성
            tools_desc = self.tool_registry.get_tools_description()
            history_str = "\n".join(self.current_history)
            prompt = self.prompt_template.format(
                tools=tools_desc,
                question=input_text,
                history=history_str
            )

            # 2. LLM 호출
            messages = [{"role": "user", "content": prompt}]
            response_text = self.llm.invoke(messages, **kwargs)

            # 3. 출력 분석 및 파싱
            thought, action = self._parse_output(response_text)

            # 4. 완료 조건 체크
            if action and action.startswith("Finish"):
                final_answer = self._parse_action_input(action)
                self.add_message(Message(input_text, "user"))
                self.add_message(Message(final_answer, "assistant"))
                return final_answer

            # 5. 도구 실행
            if action:
                tool_name, tool_input = self._parse_action(action)
                observation = self.tool_registry.execute_tool(tool_name, tool_input)
                self.current_history.append(f"Action: {action}")
                self.current_history.append(f"Observation: {observation}")

        # 최대 단계를 도달했음에도 답을 내지 못한 경우
        final_answer = "죄송합니다. 제한된 단계 내에서 작업을 완료할 수 없었습니다."
        self.add_message(Message(input_text, "user"))
        self.add_message(Message(final_answer, "assistant"))
        return final_answer
```

이와 같은 리팩터링을 통해 ReAct 패러다임을 프레임워크에 매끄럽게 통합했습니다. 핵심 개선점은 단일화된 `ToolRegistry` 인터페이스를 적극 활용하고, 고도로 세분화된 프롬프트 설계를 통해 Agent의 추론-행동 실행 루프 안정성을 높인 것입니다. ReAct 테스트 사례는 도구 사용이 필요하므로, 테스트 코드는 문서 뒤쪽에 함께 수록해 두었습니다.

### 7.4.3 ReflectionAgent

이 유형의 Agent는 이미 4장에서 핵심 로직을 구현했기 때문에, 여기서는 프레임워크용 프롬프트 템플릿만 먼저 소개합니다.
코드 생성 등 특정 분야에 국한되었던 4장의 프롬프트와 달리, 프레임워크 버전은 텍스트 작성, 분석, 창작 등 다양한 시나리오에 유연하게 대응할 수 있는 범용적인(generalized) 구조로 설계되었으며, `custom_prompts` 파라미터를 통해 언제든지 재정의할 수 있습니다.

```python
DEFAULT_PROMPTS = {
    "initial": """
다음 요구사항에 맞춰 과제를 완성해 주세요:

과제: {task}

완전하고 정확한 답변을 제공해 주시기 바랍니다.
""",
    "reflect": """
다음 응답을 꼼꼼하게 검토하고, 발생 가능한 잠재적 문제점이나 개선이 필요한 영역을 식별해 주세요:

# 원본 과제:
{task}

# 현재 응답 내용:
{content}

응답 내용의 품질을 분석하고, 구체적인 개선 제안 및 미흡한 점을 작성해 주세요.
만약 현재 응답이 매우 훌륭하고 수정이 필요 없다면 반드시 "No improvement needed"라고 답변해 주세요.
""",
    "refine": """
다음 피드백을 기반으로 응답을 개선해 주세요:

# 원본 과제:
{task}

# 이전 응답 내용:
{last_attempt}

# 피드백 내용:
{feedback}

개선된 최종 응답을 작성해 주시기 바랍니다.
"""
}
```

독자 여러분은 4장의 코드와 위의 ReAct 구현 사례를 참고하여 자신만의 `MyReflectionAgent`를 설계해 볼 수 있습니다. 다음은 아이디어를 검증하기 위한 간단한 테스트 예시입니다.

```python
# test_reflection_agent.py
from dotenv import load_dotenv
from hello_agents import HelloAgentsLLM
from my_reflection_agent import MyReflectionAgent

load_dotenv()
llm = HelloAgentsLLM()

# 기본 범용 프롬프트 템플릿 사용
general_agent = MyReflectionAgent(name="자기성찰 비서", llm=llm)

# 4장과 유사한 코드 생성용 커스텀 프롬프트 정의
code_prompts = {
    "initial": "당신은 Python 전문가입니다. 다음 함수를 작성해 주세요: {task}",
    "reflect": "다음 코드의 알고리즘 효율성을 분석해 주세요:\n작업: {task}\n코드: {content}",
    "refine": "피드백을 바탕으로 코드를 최적화해 주세요:\n작업: {task}\n피드백: {feedback}"
}
code_agent = MyReflectionAgent(
    name="코드 작성 비서",
    llm=llm,
    custom_prompts=code_prompts
)

# 테스트 실행
result = general_agent.run("인공지능의 발전 역사에 대해 짧은 글을 작성해 주세요.")
print(f"최종 결과: {result}")
```

### 7.4.4 PlanAndSolveAgent

4장에서 자유로운 텍스트 형식으로 실행 계획(plan)을 수립했던 것과 달리, 프레임워크 버전은 Planner가 반드시 계획을 Python List 형식으로 출력하도록 제한을 두어 후속 단계의 실행 안정성을 확보하고 완전한 예외 처리 메커니즘을 제공합니다. 프레임워크 기반의 Plan-and-Solve 프롬프트 설계는 다음과 같습니다:

```python
# 기본 planner 프롬프트 템플릿
DEFAULT_PLANNER_PROMPT = """
당신은 최고의 AI 계획 설계 전문가입니다. 당신의 임무는 사용자가 제기한 복잡한 문제를 여러 개의 간단한 단계로 이루어진 행동 계획(action plan)으로 세분화하는 것입니다.
각 단계는 독립적이고 실행 가능해야 하며, 엄격한 논리적 순서로 구성되어야 합니다.
출력 형식은 반드시 subtask들을 담은 문자열 목록 형태의 Python List 형식이어야 합니다.

질문: {question}

반드시 다음 예시 형식을 엄격히 준수하여 계획을 출력해 주세요:
```python
["1단계 설명", "2단계 설명", "3단계 설명", ...]
```
"""

# 기본 executor 프롬프트 템플릿
DEFAULT_EXECUTOR_PROMPT = """
당신은 최고의 AI 계획 실행 전문가입니다. 주어진 계획에 맞춰 엄격하고 꼼꼼하게 문제를 단계별로 해결하는 것이 임무입니다.
당신에게는 원본 질문, 전체 계획, 그리고 지금까지 수행한 단계별 실행 결과(이력)가 주어집니다.
현재 집중해야 하는 "현재 단계"의 문제를 해결하는 데 집중하고, 부가적인 설명이나 사족 없이 오직 "현재 단계"에 대한 답변(결과)만을 출력하세요.

# 원본 질문:
{question}

# 전체 계획:
{plan}

# 과거 실행 이력 및 결과:
{history}

# 현재 처리해야 하는 단계:
{current_step}

오직 "현재 단계"에 대한 답변만을 간결하게 출력해 주세요:
"""
```

이번 장에서도 자신만의 Agent를 설계하고 검증해 볼 수 있는 `test_plan_solve_agent.py` 테스트 파일을 제공합니다.

```python
# test_plan_solve_agent.py
from dotenv import load_dotenv
from hello_agents.core.llm import HelloAgentsLLM
from my_plan_solve_agent import MyPlanAndSolveAgent

# 환경 변수 로드
load_dotenv()

# LLM 인스턴스 생성
llm = HelloAgentsLLM()

# 커스텀 PlanAndSolveAgent 생성
agent = MyPlanAndSolveAgent(
    name="계획 실행 비서",
    llm=llm
)

# 복잡한 논리 문제 테스트
question = "한 과일 가게에서 월요일에 사과 15개를 팔았습니다. 화요일에 판 사과 개수는 월요일의 2배입니다. 수요일에 판 사과 개수는 화요일보다 5개 적습니다. 과일 가게가 이 3일 동안 판매한 총 사과 개수는 몇 개입니까?"

result = agent.run(question)
print(f"\n최종 답변: {result}")

# 대화 기록 보기
print(f"대화 기록: {len(agent.get_history())}개의 메시지")
```

또한, `custom_prompts` 매개변수를 이용하여 특정 연산 작업에 특화된 프롬프트를 로드하는 방식도 구현해 볼 수 있습니다:

```python
# 수학 문제 해결에 특화된 커스텀 프롬프트 정의
math_prompts = {
    "planner": """
당신은 수학 문제의 해결 계획을 설계하는 전문가입니다. 주어진 수학 문제를 여러 단계의 단순 연산 프로세스로 분해해 주세요:

질문: {question}

출력 형식:
```python
["1단계 계산", "2단계 계산", "최종 합산"]
```
""",
    "executor": """
당신은 정밀한 수학 연산 전문가입니다. 아래 명시된 현재 연산 단계를 수행해 주세요:

질문: {question}
전체 계획: {plan}
연산 이력: {history}
현재 연산 단계: {current_step}

다른 군더더기 없이 오직 연산된 "숫자 결과"만을 출력하세요:
"""
}

# 커스텀 수학 프롬프트를 사용하는 Agent 인스턴스 생성
math_agent = MyPlanAndSolveAgent(
    name="수학 전문 연산 비서",
    llm=llm,
    custom_prompts=math_prompts
)

# 테스트 진행
math_result = math_agent.run(question)
print(f"수학 전문 Agent의 연산 결과: {math_result}")
```

표 7.2에 정리된 바와 같이, 이러한 프레임워크 리팩터링을 통해 우리는 4장에서 구축한 개별 Agent의 핵심 기능을 모두 유지하는 것은 물론, 코드 구성, 유지보수성, 그리고 유연한 확장성을 대폭 강화했습니다. 모든 Agent가 저마다의 개성을 잃지 않으면서 프레임워크가 제공하는 단일화된 인프라의 혜택을 누리게 되었습니다.

<div align="center">
  <p>표 7.2 각 장의 Agent 구현 특징 비교</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/7-figures/table-02.png" alt="" width="90%"/>
</div>

### 7.4.5 FunctionCallAgent

FunctionCallAgent는 Hello-Agents 버전 0.2.8 이후 도입된 컴포넌트로,
OpenAI의 네이티브 함수 호출(Function Calling) 메커니즘을 기반으로 합니다.
OpenAI가 공식적으로 지원하는 API 함수 호출 규격을 이용하여 Agent를 구축하는 방법을 보여줍니다.
이 Agent는 다음과 같은 핵심 함수들을 구현하고 있습니다:

- `_build_tool_schemas`: 도구 명세(Tool descriptions)를 OpenAI Function Calling 스키마 규격으로 자동 변환합니다.
- `_extract_message_content`: OpenAI API 응답으로부터 텍스트 콘텐츠를 정확히 추출합니다.
- `_parse_function_call_arguments`: 모델이 반환하는 JSON 형태의 인자 문자열을 파이썬 딕셔너리로 안전하게 변환합니다.
- `_convert_parameter_types`: 스키마 정의에 맞춰 매개변수 타입을 보정합니다.

이러한 설계는 단순히 프롬프트 제약 조건에만 기대어 도구를 호출하게 만드는 방식에 비해, API 스키마 레벨에서의 정밀한 검증을 거치므로 실행 시 강건성(robustness)이 훨씬 뛰어납니다.

```python
def _invoke_with_tools(self, messages: list[dict[str, Any]], tools: list[dict[str, Any]], tool_choice: Union[str, dict], **kwargs):
        """OpenAI 클라이언트를 직접 호출하여 함수 호출(Function Calling)을 수행합니다."""
        client = getattr(self.llm, "_client", None)
        if client is None:
            raise RuntimeError("HelloAgentsLLM 클라이언트가 정상적으로 초기화되지 않아 함수 호출을 실행할 수 없습니다.")

        client_kwargs = dict(kwargs)
        client_kwargs.setdefault("temperature", self.llm.temperature)
        if self.llm.max_tokens is not None:
            client_kwargs.setdefault("max_tokens", self.llm.max_tokens)

        return client.chat.completions.create(
            model=self.llm.model,
            messages=messages,
            tools=tools,
            tool_choice=tool_choice,
            **client_kwargs,
        )

# 내부 로직은 OpenAI 네이티브 함수 호출 사양을 래핑합니다.
# OpenAI 네이티브 함수 호출 사용 예제
from openai import OpenAI
client = OpenAI()

tools = [
  {
    "type": "function",
    "function": {
      "name": "get_current_weather",
      "description": "지정된 위치의 현재 날씨 정보를 획득합니다.",
      "parameters": {
        "type": "object",
        "properties": {
          "location": {
            "type": "string",
            "description": "도시 및 주 정보 (예: San Francisco, CA)",
          },
          "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
        },
        "required": ["location"],
      },
    }
  }
]
messages = [{"role": "user", "content": "오늘 보스턴의 날씨는 어떤가요?"}]
completion = client.chat.completions.create(
  model="gpt-5",
  messages=messages,
  tools=tools,
  tool_choice="auto"
)

print(completion)
```
## 7.5 도구 시스템

이 섹션의 내용은 앞서 구축한 Agent 인프라를 바탕으로 도구 시스템의 설계 및 구현 방식을 심층적으로 살펴보는 것입니다. 우리는 인프라 설계부터 시작하여 점진적으로 커스텀 도구의 설계 방식을 다룰 것입니다.
이 섹션의 학습 목표는 주로 다음 세 가지 핵심적인 측면을 중심으로 이루어집니다:

1. **단일화된 도구 추상화 및 관리**: 도구의 개발, 등록, 탐색 및 실행을 위한 통합 인프라를 구축하기 위해 표준화된 Tool 기본 클래스와 ToolRegistry 등록 메커니즘을 정립합니다.
2. **실습 기반의 도구 개발**: 수학 연산 도구를 대표적인 사례로 사용하여 사용자가 직접 커스텀 도구를 설계하고 구현하는 완전한 프로세스를 마스터하도록 돕습니다.
3. **고급 통합 및 최적화 전략**: 다중 소스 검색 도구 설계를 통해 여러 외부 API 서비스를 연계하고, 백엔드의 자동 선택,
결과 병합, 그리고 장애 극복(fault tolerance) 메커니즘을 어떻게 구현하는지 보여줌으로써 복잡한 비즈니스 시나리오에서의 도구 설계 아키텍처 역량을 배양합니다.

### 7.5.1 도구 기본 클래스 및 등록 메커니즘 디자인

확장 가능한 도구 시스템을 구축할 때 가장 먼저 해결해야 할 일은 표준화된 기본 인프라를 구축하는 것입니다. 이 인프라에는 Tool 기본 클래스, ToolRegistry 등록기, 그리고 도구 관리 메커니즘이 포함됩니다.

**(1) 도구 기본 클래스의 추상적 설계**

Tool 기본 클래스는 모든 도구가 반드시 준수해야 하는 인터페이스 명세를 규정하는 도구 시스템의 최상위 추상화 요소입니다:

```python
class Tool(ABC):
    """도구 기본 클래스"""

    def __init__(self, name: str, description: str):
        self.name = name
        self.description = description

    @abstractmethod
    def run(self, parameters: Dict[str, Any]) -> str:
        """도구를 실행합니다."""
        pass

    @abstractmethod
    def get_parameters(self) -> List[ToolParameter]:
        """도구의 매개변수 정의 목록을 가져옵니다."""
        pass
```

이 설계는 객체지향 설계의 핵심 개념을 반영하고 있습니다. 통일된 `run` 메서드 인터페이스를 지님으로써 모든 도구는 딕셔너리 형태의 매개변수를 입력받고 문자열 결과를 반환하는 일관된 형태를 유지하며 실행됩니다.
또한 도구는 스스로를 설명할 수 있는 자가 설명 능력을 가집니다. `get_parameters` 메서드를 통해 호출자에게 자신이 요구하는 매개변수를 명확히 전달하며, 이러한 메커니즘은 자동화된 문서 생성 및 매개변수 유효성 검증의 탄탄한 토대를 제공합니다. 또한 이름(name)과 설명(description)과 같은 메타데이터는 도구 시스템에 뛰어난 탐색(discoverability) 능력과 직관적인 이해를 부여합니다.

**(2) ToolParameter 매개변수 정의 시스템**

복잡한 매개변수 검증 및 사양 정의를 돕기 위해 `ToolParameter` 클래스를 설계했습니다:

```python
class ToolParameter(BaseModel):
    """도구 매개변수 정의"""
    name: str
    type: str
    description: str
    required: bool = True
    default: Any = None
```

이 구조는 도구가 요구하는 매개변수 규격을 정확하게 표현할 수 있게 해주며, 타입 검사, 기본값 주입, 그리고 자동 문서 생성을 안정적으로 지원합니다.

**(3) ToolRegistry 구현**

ToolRegistry는 도구 시스템의 중앙 제어 허브로,
도구의 등록, 발견, 실행 등의 핵심 관리 기능을 전담합니다. 이번 장에서는 주로 다음과 같은 함수들을 설계하여 활용합니다:

```python
class ToolRegistry:
    """HelloAgents 도구 등록기"""

    def __init__(self):
        self._tools: dict[str, Tool] = {}
        self._functions: dict[str, dict[str, Any]] = {}

    def register_tool(self, tool: Tool):
        """Tool 객체를 등록합니다."""
        if tool.name in self._tools:
            print(f"⚠️ 경고: 도구 '{tool.name}'이(가) 이미 존재하므로 덮어씁니다.")
        self._tools[tool.name] = tool
        print(f"✅ 도구 '{tool.name}' 등록 완료.")

    def register_function(self, name: str, description: str, func: Callable[[str], str]):
        """
        단순한 함수를 도구로 바로 등록합니다 (편의 메서드).

        Args:
            name: 도구 이름
            description: 도구 설명
            func: 호출할 함수. 문자열 인자를 받아 문자열 결과를 반환해야 함.
        """
        if name in self._functions:
            print(f"⚠️ 경고: 도구 '{name}'이(가) 이미 존재하므로 덮어씁니다.")

        self._functions[name] = {
            "description": description,
            "func": func
        }
        print(f"✅ 도구 '{name}' 등록 완료.")
```

`ToolRegistry`는 두 가지 형태의 등록 편의성을 열어 둡니다:

1. **Tool 객체 직접 등록**: 매개변수 상세 정의나 타입 검증이 따르는 정밀하고 복잡한 도구 구현에 어울립니다.
2. **함수 직접 래핑 등록**: 기존에 사용하던 함수를 빠르게 도구 인터페이스로 변환하여 융합할 때 실용적입니다.

**(4) 도구 탐색 및 관리 메커니즘**

등록기는 등록된 도구의 정보를 텍스트 포맷으로 가공하여 시스템 외부에 알리는 핵심적인 도구 발견 기능을 구비하고 있습니다:

```python
def get_tools_description(self) -> str:
    """등록된 모든 도구의 포맷팅된 설명 문자열을 반환합니다."""
    descriptions = []

    # Tool 객체들의 설명 취합
    for tool in self._tools.values():
        descriptions.append(f"- {tool.name}: {tool.description}")

    # 함수 도구들의 설명 취합
    for name, info in self._functions.items():
        descriptions.append(f"- {name}: {info['description']}")

    return "\n".join(descriptions) if descriptions else "No tools available"
```

이 메서드를 통해 생성된 설명 텍스트는 Agent의 프롬프트 템플릿에 그대로 투입되어, Agent가 현재 자신이 동원할 수 있는 도구 목록과 사용법을 인지하는 정보원으로 작용합니다.

### 7.5.2 커스텀 도구 개발

이제 인프라가 설계되었으므로, 실질적인 커스텀 도구 개발 방식을 알아보겠습니다.
수학 연산 도구는 직관적이면서도 구조적으로 단순하여 입문에 매우 적합한 사례입니다.
가장 간단한 구현은 `ToolRegistry`의 함수 등록 편의 기능을 이용하는 것입니다.

자신만의 수학 연산 함수를 작성해 봅시다. 먼저 프로젝트 디렉토리에 `my_calculator_tool.py` 파일을 생성합니다:

```python
# my_calculator_tool.py
import ast
import operator
import math
from hello_agents import ToolRegistry

def my_calculate(expression: str) -> str:
    """수식 연산을 처리하는 단순한 계산 함수"""
    if not expression.strip():
        return "계산할 수식이 비어 있습니다."

    # 지원할 기본 산술 연산자 목록 정의
    operators = {
        ast.Add: operator.add,      # +
        ast.Sub: operator.sub,      # -
        ast.Mult: operator.mul,     # *
        ast.Div: operator.truediv,  # /
    }

    # 지원할 기본 수학 함수 및 상수 정의
    functions = {
        'sqrt': math.sqrt,
        'pi': math.pi,
    }

    try:
        node = ast.parse(expression, mode='eval')
        result = _eval_node(node.body, operators, functions)
        return str(result)
    except Exception:
        return "연산 처리에 실패했습니다. 수식 형식을 확인해 주세요."

def _eval_node(node, operators, functions):
    """추상 구문 트리(AST) 노드를 안전하게 순회하며 연산값을 구합니다."""
    if isinstance(node, ast.Constant):
        return node.value
    elif isinstance(node, ast.BinOp):
        left = _eval_node(node.left, operators, functions)
        right = _eval_node(node.right, operators, functions)
        op = operators.get(type(node.op))
        return op(left, right)
    elif isinstance(node, ast.Call):
        func_name = node.func.id
        if func_name in functions:
            args = [_eval_node(arg, operators, functions) for arg in node.args]
            return functions[func_name](*args)
    elif isinstance(node, ast.Name):
        if node.id in functions:
            return functions[node.id]

def create_calculator_registry():
    """계산기 도구가 사전 등록된 도구 등록기 인스턴스를 생성합니다."""
    registry = ToolRegistry()

    # 연산 처리 함수를 등록기 인터페이스에 바인딩
    registry.register_function(
        name="my_calculator",
        description="간단한 산술 연산 (+,-,*,/) 및 제곱근(sqrt) 함수 연산을 지원하는 도구입니다.",
        func=my_calculate
    )

    return registry
```

이 도구는 사칙연산 연산자뿐만 아니라 대표적인 수학 함수와 상수까지 아우르고 있어 일상의 다양한 연산 요구에 충분히 부응합니다. 독자 여러분은 이 로직을 더욱 풍성하게 개량하여 성능을 향상시킬 수도 있습니다. 검증을 돕는 `test_my_calculator.py` 예시 파일은 다음과 같습니다:

```python
# test_my_calculator.py
from dotenv import load_dotenv
from my_calculator_tool import create_calculator_registry

# 환경 변수 로드
load_dotenv()

def test_calculator_tool():
    """자체 연산 도구 단위 테스트"""

    # 계산기가 내장된 등록기 생성
    registry = create_calculator_registry()

    print("🧪 커스텀 연산 도구 검증 시작\n")

    # 검증할 테스트 연산 수식 목록
    test_cases = [
        "2 + 3",           # 기본 덧셈
        "10 - 4",          # 기본 뺄셈
        "5 * 6",           # 기본 곱셈
        "15 / 3",          # 기본 나눗셈
        "sqrt(16)",        # 제곱근 연산
    ]

    for i, expression in enumerate(test_cases, 1):
        print(f"테스트 {i}: {expression}")
        result = registry.execute_tool("my_calculator", expression)
        print(f"결과: {result}\n")

def test_with_simple_agent():
    """SimpleAgent와의 연동 작동 테스트"""
    from hello_agents import HelloAgentsLLM

    # LLM 클라이언트 설정
    llm = HelloAgentsLLM()

    # 도구 등록기 로드
    registry = create_calculator_registry()

    print("🤖 SimpleAgent 연동 동작 흐름 시뮬레이션:")

    # Agent가 도구를 호출해야 하는 복합 질문 정의
    user_question = "sqrt(16) + 2 * 3 연산을 도와줘."

    print(f"사용자 질문: {user_question}")

    # 도구를 기계적으로 작동하여 값 계산
    calc_result = registry.execute_tool("my_calculator", "sqrt(16) + 2 * 3")
    print(f"도구 연산 결과: {calc_result}")

    # LLM이 연산된 사실 자료에 의거하여 답변하게 조율
    final_messages = [
        {"role": "user", "content": f"연산된 결과는 {calc_result}입니다. 이 계산값을 참고하여 질문에 한국어 자연어로 답해 주세요: {user_question}"}
    ]

    print("\n🎯 SimpleAgent 최종 응답:")
    response = llm.think(final_messages)
    for chunk in response:
        print(chunk, end="", flush=True)
    print("\n")

if __name__ == "__main__":
    test_calculator_tool()
    test_with_simple_agent()
```

이와 같이 실용적인 계산기 개발을 경험해 보며 간단한 함수를 만들고 `ToolRegistry`에 얹어 `SimpleAgent`와 매끄럽게 연결하는 실습을 끝마쳤습니다. 아울러 작동 흐름의 구조적 이해를 보완하기 위해 그림 7.1을 수록해 두었습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/7-figures/01.png" alt="" width="90%"/>
  <p>그림 7.1 HelloAgents 기반의 SimpleAgent 동작 프로세스</p>
</div>

### 7.5.3 다중 소스 검색 도구

엔지니어링 실무에서는 더 신뢰할 수 있고 균형 잡힌 응답 결과를 생성하기 위해 여러 개의 서드파티 검색 엔진이나 외부 정보 시스템을 한데 모으는 작업이 자주 발생합니다.
1장에서는 Tavily API를 사용했었고 4장에서는 SerpApi Google API를 학습해 보았으므로, 이번 장에서는 이 둘을 지능적으로 교차 연동하는 하이브리드 검색 도구를 작성해 보겠습니다.
관련 패키지가 미설치 상태라면 다음 명령어로 먼저 라이브러리를 추가해 줍니다:

```bash
pip install "hello-agents[search]==0.1.1"
```

**(1) 검색 도구의 통합 인터페이스 설계**

HelloAgents 프레임워크가 제공하는 `SearchTool`의 고급 하이브리드 검색 설계 구조입니다:

```python
class SearchTool(Tool):
    """
    지능형 하이브리드 검색 도구

    다양한 검색 백엔드를 관리하며, 가용한 키 상태에 따라 최적의 소스를 결정합니다:
    1. 하이브리드 모드 (hybrid) - Tavily 혹은 SerpApi 자동 판별 선택
    2. Tavily API (tavily) - LLM 및 AI에 친화적인 요약 검색 제공
    3. SerpApi (serpapi) - 구글 포털 등 전통 웹 환경 검색 제공
    """

    def __init__(self, backend: str = "hybrid", tavily_key: Optional[str] = None, serpapi_key: Optional[str] = None):
        super().__init__(
            name="search",
            description="인터넷 실시간 검색을 수행하는 검색 엔진 도구입니다. 하이브리드 감지를 통해 장애 조치 및 최적의 검색을 진행합니다."
        )
        self.backend = backend
        self.tavily_key = tavily_key or os.getenv("TAVILY_API_KEY")
        self.serpapi_key = serpapi_key or os.getenv("SERPAPI_API_KEY")
        self.available_backends = []
        self._setup_backends()
```

이 패턴의 강점은 복잡한 설정 없이도 시스템이 환경 내에 등록된 API 키를 읽어 적당한 활성 검색 엔진 목록을 자동 수립해 두는 데 있습니다.

**(2) Tavily 및 SerpApi 검색 소스의 통합 전략**

하이브리드 검색 백엔드 결정 핵심 로직 설계 부분입니다:

```python
def _search_hybrid(self, query: str) -> str:
    """하이브리드 모드 - 가용한 수단을 동원해 성공할 때까지 장애 복구를 처리합니다."""
    # AI 최적화 강점을 가진 Tavily 우선 처리
    if "tavily" in self.available_backends:
        try:
            return self._search_tavily(query)
        except Exception as e:
            print(f"⚠️ Tavily 검색 도중 예외 발생: {e}")
            # Tavily 작동 불능 시 차선책인 SerpApi 호출 시도
            if "serpapi" in self.available_backends:
                print("🔄 SerpApi 백엔드로 즉시 우회합니다.")
                return self._search_serpapi(query)

    # Tavily 키가 없거나 비활성 상태면 SerpApi 백엔드에 의존
    elif "serpapi" in self.available_backends:
        try:
            return self._search_serpapi(query)
        except Exception as e:
            print(f"⚠️ SerpApi 검색 도중 예외 발생: {e}")

    # 모든 리소스 조달이 불가능할 경우 안내
    return "❌ 가용한 검색 소스가 없습니다. .env에 TAVILY_API_KEY 혹은 SERPAPI_API_KEY가 있는지 확인해 주세요."
```

이 구현은 고가용성 설계 개념을 고스란히 투영하고 있습니다. 특정 경로가 일시적인 통신 장애나 키 만료로 무너졌을 때,
시스템이 조용히 하위 예비 경로로 질의를 전달해 복원성을 유지하고,
최종적인 실패 시 명료하게 관리자에게 에러 요인을 제공합니다.

**(3) 검색 결과의 통일된 포맷 처리**

서로 다른 벤더 제품들은 API 결과값으로 돌려주는 데이터 스키마와 키 이름이 각양각색입니다. 프레임워크는 이를 하나의 문자열 레이아웃으로 균일하게 다듬어 줍니다:

```python
def _search_tavily(self, query: str) -> str:
    """Tavily API 정보 정제"""
    response = self.tavily_client.search(
        query=query,
        search_depth="basic",
        include_answer=True,
        max_results=3
    )

    result = f"🎯 Tavily AI 직접 답변: {response.get('answer', '직접 답변을 생성하지 못했습니다.')}\n\n"

    for i, item in enumerate(response.get('results', [])[:3], 1):
        result += f"[{i}] {item.get('title', '')}\n"
        result += f"    {item.get('content', '')[:200]}...\n"
        result += f"    출처: {item.get('url', '')}\n\n"

    return result
```

이 철학에 기반하여, 우리만의 지능형 클래스형 검색 도구를 작성해 보겠습니다. 프로젝트 디렉토리에 `my_advanced_search.py` 파일을 생성합니다:

```python
# my_advanced_search.py
import os
from typing import Optional, List, Dict, Any
from hello_agents import ToolRegistry

class MyAdvancedSearchTool:
    """
    다중 소스 통합 및 장애 조치 패턴을 보여주는 고급 커스텀 검색 도구 클래스
    """

    def __init__(self):
        self.name = "my_advanced_search"
        self.description = "다중 백엔드를 활용해 복합 결과를 지능적으로 필터링하여 응답하는 웹 검색 도구입니다."
        self.search_sources = []
        self._setup_search_sources()

    def _setup_search_sources(self):
        """가용한 API 키 유무에 따른 동적 소스 스캔 설정"""
        # Tavily 라이브러리 검출
        if os.getenv("TAVILY_API_KEY"):
            try:
                from tavily import TavilyClient
                self.tavily_client = TavilyClient(api_key=os.getenv("TAVILY_API_KEY"))
                self.search_sources.append("tavily")
                print("✅ Tavily 검색 엔진 사용이 활성화되었습니다.")
            except ImportError:
                print("⚠️ Tavily SDK 라이브러리가 패키지에 탐지되지 않습니다.")

        # SerpApi 라이브러리 검출
        if os.getenv("SERPAPI_API_KEY"):
            try:
                import serpapi
                self.search_sources.append("serpapi")
                print("✅ SerpApi 검색 엔진 사용이 활성화되었습니다.")
            except ImportError:
                print("⚠️ SerpApi 라이브러리가 패키지에 탐지되지 않습니다.")

        if self.search_sources:
            print(f"🔧 탐색 가능한 활성 채널: {', '.join(self.search_sources)}")
        else:
            print("⚠️ 어떠한 검색 소스 자격 키도 검출되지 않았습니다. API 키 구성을 확인해 주세요.")

    def search(self, query: str) -> str:
        """지능형 다중 경로 검색 연동 실행"""
        if not query.strip():
            return "❌ 오류: 검색어가 비어 있습니다."

        if not self.search_sources:
            return """❌ 사용 가능한 검색 수단이 부재합니다. 아래 중 최소 하나의 환경 변수 구성을 마쳐 주세요:

1. Tavily AI: TAVILY_API_KEY 환경 변수 등록
   발급 주소: https://tavily.com/

2. SerpAPI: SERPAPI_API_KEY 환경 변수 등록
   발급 주소: https://serpapi.com/

키를 환경 설정에 입력한 뒤 터미널/애플리케이션을 다시 켜서 구동해 주세요."""

        print(f"🔍 실시간 검색 정보 쿼리 시작: {query}")

        # 백엔드 스캔 작동
        for source in self.search_sources:
            try:
                if source == "tavily":
                    result = self._search_with_tavily(query)
                    if result and "not found" not in result.lower():
                        return f"📊 Tavily AI 검색 결과 요약:\n\n{result}"

                elif source == "serpapi":
                    result = self._search_with_serpapi(query)
                    if result and "not found" not in result.lower():
                        return f"🌐 SerpApi Google 일반 검색 결과:\n\n{result}"

            except Exception as e:
                print(f"⚠️ {source} API 작동 지연 또는 실패: {e}")
                continue

        return "❌ 가용한 채널이 전부 정상 응답을 돌려주지 못했습니다. 네트워크 상황을 체크해 주세요."

    def _search_with_tavily(self, query: str) -> str:
        """Tavily API 세부 핸들링"""
        response = self.tavily_client.search(query=query, max_results=3)

        if response.get('answer'):
            result = f"💡 AI 직접 요약: {response['answer']}\n\n"
        else:
            result = ""

        result += "🔗 관련 기사 내용:\n"
        for i, item in enumerate(response.get('results', [])[:3], 1):
            result += f"[{i}] {item.get('title', '')}\n"
            result += f"    {item.get('content', '')[:150]}...\n\n"

        return result

    def _search_with_serpapi(self, query: str) -> str:
        """SerpApi 구글 검색 핸들링"""
        import serpapi

        search = serpapi.GoogleSearch({
            "q": query,
            "api_key": os.getenv("SERPAPI_API_KEY"),
            "num": 3
        })

        results = search.get_dict()

        result = "🔗 구글 유기적(Organic) 검색 정보:\n"
        if "organic_results" in results:
            for i, res in enumerate(results["organic_results"][:3], 1):
                result += f"[{i}] {res.get('title', '')}\n"
                result += f"    {res.get('snippet', '')}\n\n"

        return result

def create_advanced_search_registry():
    """고급 검색 기능이 연동된 도구 등록기 생성"""
    registry = ToolRegistry()

    # 검색 클라이언트 구축
    search_tool = MyAdvancedSearchTool()

    # 검색 클라이언트의 메서드를 함수형 도구로 직접 등록
    registry.register_function(
        name="advanced_search",
        description="Tavily와 SerpAPI를 모두 연동하고 감지하여 다각도로 풍부한 최신 웹 정보를 전달하는 도구입니다.",
        func=search_tool.search
    )

    return registry
```

이 기능을 테스트할 수 있는 검증 파일인 `test_advanced_search.py` 코드 예시입니다:

```python
# test_advanced_search.py
from dotenv import load_dotenv
from my_advanced_search import create_advanced_search_registry, MyAdvancedSearchTool

# 환경 변수 로드
load_dotenv()

def test_advanced_search():
    """다중 검색 도구 연동 검증"""

    # 고급 검색 도구 등록기 인스턴스 생성
    registry = create_advanced_search_registry()

    print("🔍 고급 교차 검색 도구 검증 프로세스 가동\n")

    test_queries = [
        "Python 프로그래밍 언어의 역사",
        "인공지능 분야의 최신 기술 트렌드",
        "2026년 세계 IT 산업 전망"
    ]

    for i, query in enumerate(test_queries, 1):
        print(f"테스트 {i}: {query}")
        result = registry.execute_tool("advanced_search", query)
        print(f"결과:\n{result}")
        print("-" * 60 + "\n")

def test_api_configuration():
    """예외적인 API 키 미비 상황 동작 체크"""
    print("🔧 API 설정 감지 기능 테스트:")

    # 단독 객체 강제 활성화 시도
    search_tool = MyAdvancedSearchTool()

    # 키 환경을 일부 지우고 실행하면 안내가 정확히 제공되는지 점검
    result = search_tool.search("기계 학습 기본 알고리즘")
    print(f"보안/예외 상황 피드백 결과:\n{result}")

def test_with_agent():
    """Agent 프롬프트용 텍스트 명세 취합 테스트"""
    print("\n🤖 Agent 프롬프트용 도구 연동 기능 검증:")
    print("고급 다중 소스 검색 도구가 준비되어 Agent 프롬프트에 활용될 수 있습니다.")

    # 도구 명세가 잘 정제되는지 출력 확인
    registry = create_advanced_search_registry()
    tools_desc = registry.get_tools_description()
    print(f"통합 설명 텍스트:\n{tools_desc}")

if __name__ == "__main__":
    test_advanced_search()
    test_api_configuration()
    test_with_agent()
```

클래스 기반 구조로 검색 도구를 직접 개발해 보았습니다. 이러한 클래스 방식은 API 커넥터 유지나 시스템의 연결 상태값 관리 등 '도구 내부에 고유한 상태 정보'를 누적하여 관리할 필요가 있을 때 매우 강력하게 작용합니다.

### 7.5.4 도구 시스템의 고급 기능

기본 도구 작성 및 다중 소스 통합을 익혔으니, 도구 시스템의 심화 고급 주제로 넘어가겠습니다. 이 기술들은 Agent가 운영 단계의 프로덕션 망에서 안전하고 강건하게 활동하는 밑거름이 됩니다.

**(1) 도구 체인 호출 메커니즘**

하나의 질문에 대답하기 위해 여러 도구를 순차적으로 엮어 파이프라인처럼 작동시켜야 하는 상황이 빈번합니다. 우리는 6장에서 배운 워크플로(Graph) 개념을 차용하여 순차적 도구 실행을 돕는 도구 체인 시스템을 구성할 수 있습니다:

```python
# tool_chain_manager.py
from typing import List, Dict, Any, Optional
from hello_agents import ToolRegistry

class ToolChain:
    """여러 도구를 단계별로 실행하는 도구 체인"""

    def __init__(self, name: str, description: str):
        self.name = name
        self.description = description
        self.steps: List[Dict[str, Any]] = []

    def add_step(self, tool_name: str, input_template: str, output_key: str = None):
        """
        체인에 실행 단계를 추가합니다.

        Args:
            tool_name: 호출할 도구명
            input_template: 입력 형식 템플릿 (이전 노드의 결과 매핑용 변수 지원)
            output_key: 해당 단계의 출력 결과를 참조할 이름 정보
        """
        self.steps.append({
            "tool_name": tool_name,
            "input_template": input_template,
            "output_key": output_key or f"step_{len(self.steps)}_result"
        })

    def execute(self, registry: ToolRegistry, initial_input: str, context: Dict[str, Any] = None) -> str:
        """도구 체인 프로세스 실행"""
        context = context or {}
        context["input"] = initial_input

        print(f"🔗 도구 체인 파이프라인 작동: {self.name}")

        for i, step in enumerate(self.steps, 1):
            tool_name = step["tool_name"]
            input_template = step["input_template"]
            output_key = step["output_key"]

            # 컨텍스트 기반 변수 바인딩 치환
            try:
                tool_input = input_template.format(**context)
            except KeyError as e:
                return f"❌ 도구 체인 연계 예외: 템플릿 매핑 과정에서 필요한 변수 {e}가 결여되었습니다."

            print(f"  단계 {i}: '{tool_input[:50]}...' 내용을 도구 '{tool_name}'로 연산 진행")

            # 도구 구동
            result = registry.execute_tool(tool_name, tool_input)
            context[output_key] = result

            print(f"  ✅ 단계 {i} 완료, 결과 크기: {len(result)} 자")

        # 가장 마지막 체인 노드가 도출한 데이터를 최종값으로 결정
        final_result = context[self.steps[-1]["output_key"]]
        print(f"🎉 도구 체인 워크플로 '{self.name}' 실행 완료")
        return final_result

class ToolChainManager:
    """도구 체인 종합 관리 매니저"""

    def __init__(self, registry: ToolRegistry):
        self.registry = registry
        self.chains: Dict[str, ToolChain] = {}

    def register_chain(self, chain: ToolChain):
        """작성된 체인을 관리 풀에 등록"""
        self.chains[chain.name] = chain
        print(f"✅ 도구 체인 '{chain.name}' 등록 완료")

    def execute_chain(self, chain_name: str, input_data: str, context: Dict[str, Any] = None) -> str:
        """등록된 특정 명칭의 체인 실행"""
        if chain_name not in self.chains:
            return f"❌ 오류: 지정하신 도구 체인 '{chain_name}'이(가) 존재하지 않습니다."

        chain = self.chains[chain_name]
        return chain.execute(self.registry, input_data, context)

    def list_chains(self) -> List[str]:
        """등록된 전체 도구 체인 명칭 리스트 반환"""
        return list(self.chains.keys())

# 체인 활용 시나리오 예제
def create_research_chain() -> ToolChain:
    """정보 검색 -> 연산 처리 -> 요약 정리 순서의 연구용 도구 체인 생성 예시"""
    chain = ToolChain(
        name="research_and_calculate",
        description="실시간 웹 정보를 조회한 후 특정 수학 지표를 추출해 연산을 가하는 도구 체인"
    )

    # 1. 정보 검색 수행
    chain.add_step(
        tool_name="search",
        input_template="{input}",
        output_key="search_result"
    )

    # 2. 획득한 검색 결과를 참조하여 계산기 호출
    chain.add_step(
        tool_name="my_calculator",
        input_template="수집된 텍스트 데이터의 세부 지표 수식을 풀어주세요: {search_result}",
        output_key="calculation_result"
    )

    return chain
```

**(2) 비동기 도구 실행 지원**

상당수 외부 웹 검색이나 대규모 DB 조회 등의 태스크는 응답 지연을 초래합니다. 전체 프레임워크가 멈춰 서는 것을 방지하기 위해, 비동기 호출 병렬 구조를 설계합니다:

```python
# async_tool_executor.py
import asyncio
import concurrent.futures
from typing import Dict, Any, List, Callable
from hello_agents import ToolRegistry

class AsyncToolExecutor:
    """비동기 병렬 도구 실행기"""

    def __init__(self, registry: ToolRegistry, max_workers: int = 4):
        self.registry = registry
        self.executor = concurrent.futures.ThreadPoolExecutor(max_workers=max_workers)

    async def execute_tool_async(self, tool_name: str, input_data: str) -> str:
        """특정 개별 도구의 비동기 실행"""
        loop = asyncio.get_event_loop()

        def _execute():
            return self.registry.execute_tool(tool_name, input_data)

        # 동기형 레거시 도구 함수를 ThreadPool에 실어 이벤트 루프 지연 회피
        result = await loop.run_in_executor(self.executor, _execute)
        return result

    async def execute_tools_parallel(self, tasks: List[Dict[str, str]]) -> List[str]:
        """여러 도구 태스크들을 한번에 병렬 비동기 동시 실행"""
        print(f"🚀 총 {len(tasks)}개의 도구 실행 작업을 병렬로 개시합니다.")

        # 코루틴 태스크 집합 생성
        async_tasks = []
        for task in tasks:
            tool_name = task["tool_name"]
            input_data = task["input_data"]
            async_task = self.execute_tool_async(tool_name, input_data)
            async_tasks.append(async_task)

        # 비동기 동시 대기 처리
        results = await asyncio.gather(*async_tasks)

        print(f"✅ 모든 병렬 도구 실행 태스크가 완료되었습니다.")
        return results

    def __del__(self):
        """할당된 리소스 해제"""
        if hasattr(self, 'executor'):
            self.executor.shutdown(wait=True)

# 비동기 병렬 실행 시뮬레이션
async def test_parallel_execution():
    """병렬 실행 테스트 흐름 예시"""
    from hello_agents import ToolRegistry

    registry = ToolRegistry()
    # 이미 도구들이 등록기에 바인딩 완료되었다고 가정

    executor = AsyncToolExecutor(registry)

    # 병렬로 동시에 밀어 넣을 다중 작업 명세 구성
    tasks = [
        {"tool_name": "search", "input_data": "Python 언어 트렌드"},
        {"tool_name": "search", "input_data": "기계학습 모델 종류"},
        {"tool_name": "my_calculator", "input_data": "1024 * 8"},
        {"tool_name": "my_calculator", "input_data": "sqrt(256)"},
    ]

    # 실행
    results = await executor.execute_tools_parallel(tasks)

    for i, result in enumerate(results):
        print(f"작업 {i+1} 연산 결과 (첫 100자): {result[:100]}...")
```

경험에 비추어 볼 때 도구 시스템을 효과적으로 구축하기 위해 다음 지침을 지키는 것이 유익합니다. 첫째, 각 컴포넌트는 단일 책임의 성격에만 골몰해 기능적으로 단순해야 하고, 둘째, 도구의 통일된 입출력 포맷 규격 하에 예외 및 예방 검증 코드가 성실히 수반되어야 합니다. 또한, IO 병목이 동반되는 지연이 큰 동작은 적합한 ThreadPool 및 비동기 처리기를 적극 고려하는 것이 시스템 가용성에 좋습니다.
## 7.6 장 요약

본격적인 요약에 앞서, 독자분들께 기쁜 소식을 전해드립니다. 이번 장에서 직접 구현하고 검증했던 모든 클래스와 기능들에 대한 전체 실행 테스트 케이스가 GitHub에 마련되어 있습니다. [이 예제 링크](https://github.com/jjyaoao/HelloAgents/blob/main/examples/chapter07_basic_setup.py)를 방문하여 직접 코드를 열람하고 실행해 볼 수 있습니다. 이 파일에는 네 가지 Agent 패러다임 구동 쉘, 도구 통합 및 비동기 파이프라인 관리 모듈의 작동 모습, 그리고 사용자 인터랙션 예제가 풍부하게 포함되어 있습니다. 작성하신 코드의 유효성을 점검하거나 프레임워크의 상세 설계를 실감나게 학습하고 싶다면 이 예제들이 큰 이정표가 되어줄 것입니다.

이번 7장을 돌이켜보면 우리는 꽤 도전적인 여정을 소화해 냈습니다. 바로 자신만의 Agent 프레임워크인 HelloAgents를 바닥부터 한 단계씩 건설한 것입니다. 이 건설의 과정은 "계층별 격리(decoupling), 단일 책임 분담, 그리고 상호 인터페이스 통일"이라는 튼튼한 소프트웨어 엔지니어링의 준칙을 철저하게 따랐습니다.

프레임워크 내부를 다질 때 우리는 네 가지 핵심 Agent 작동 패러다임을 재정비하여 얹었습니다. 단순 소통을 전담하는 SimpleAgent부터 시작하여, 추론과 행위를 맞물린 ReActAgent, 결과물을 성찰하여 연마하는 ReflectionAgent, 문제를 세분 계획하여 차례로 요리하는 PlanAndSolveAgent에 이르기까지 일관된 구조로 격상시켰습니다. 또한 Agent의 손발이 되어줄 도구(Tool) 파이프라인은 복합 통합 및 비동기 처리 기법을 총동원하여 프로덕션에서도 손색없을 탄탄한 형태를 지니게 되었습니다.

가장 중요한 사실은 이번 7장 프레임워크의 성공적인 구축이 끝이 아닌, 더 넓은 고급 지식으로 향하는 교두보라는 점입니다. 우리는 처음 시작할 때부터 후속 고급 모듈이 추가될 수 있도록 세심하게 아키텍처 확장 공간을 배정해 두었습니다. 즉, 이번에 마련한 통합 LLM 브리지, 통일된 메시지 규격, 그리고 도구 등록 허브는 앞으로 마주할 수많은 과제들의 핵심 주춧돌이 됩니다. 당장 8장에서 배우게 될 지능형 메모리(Memory) 및 지식 데이터베이스 결합 검색 증강 생성(RAG) 모듈이 이 구조 위에서 작동하게 되며, 9장의 고밀도 프롬프트 컨텍스트 엔지니어링도 이번 메시지 인터페이스를 뼈대로 삼습니다. 아울러 10장의 다기능 Agent 프로토콜도 여기에 새로운 도구(Tool)를 확장 등록하는 방식으로 구현됩니다.

다음 장에서는 이 뼈대 위에 기억력과 지식을 불어넣는 메모리 및 RAG 시스템을 이식해 보겠습니다. 8장에서 더 흥미진진한 이야기로 다시 만나요!


## 연습 문제

1. 이번 장에서는 `HelloAgents` 프레임워크의 기초 뼈대를 설계하며 "자신만의 Agent 프레임워크를 개발해야 하는 당위성"에 대해 고민해 보았습니다. 다음 논제를 분석해 주세요:
   - 7.1.1절에서는 현재 상용 혹은 오픈소스로 풀려 있는 주류 프레임워크들이 가지는 주요 한계 네 가지를 꼽았습니다. 독자 여러분이 [6장 연습 문제](../chapter6/第六章%20框架开发实践.md#习题)를 포함한 이전 단계에서 직접 타사 프레임워크를 이용해 개발했던 경험에 빗대어, 이러한 한계점들이 왜 실무 생산성을 저해하는지 사례를 들어 설명해 보세요.
   - `HelloAgents`는 디자인의 미니멀리즘을 위해 "모든 부가 시스템(Memory, RAG, MCP 등)은 도구(Tool)의 하위 부류로 간주하여 통합 추상화한다"는 파격적인 설계 철학을 제안했습니다. 이와 같은 단순화의 명확한 엔지니어링상 장점은 무엇이며, 반대로 어떤 복잡한 시나리오에서 이 설계가 역효과나 제한 요인을 가질 수 있을지 의견을 말해 주세요.
   - 4장 등에서 클래스나 인프라 틀 없이 날것으로 구현했던 코드 상태와 이번 장에서 프레임워크 옷을 입힌 뒤의 구조를 비교했을 때, 어떤 부분이 구체적으로 유지보수하기 편리해졌나요? 만일 독자 본인이 이 프레임워크를 전면 재설계한다면 어떠한 소프트웨어적 가치(예: 확장성, 속도, 타입 안정성 등)를 최우선 가치로 두겠습니까?

2. 7.2절에서는 멀티 모델 벤더와 로컬 VLLM/Ollama 호환 처리가 가능하도록 `HelloAgentsLLM`을 개량했습니다.
   > **안내**: 실제 개발 환경을 설정하여 직접 확인해 보시는 것을 권장합니다.
   - 7.2.1절의 연동 설계 구조를 응용하여, `Gemini` 혹은 `Anthropic`, `Claude` 같은 신규 대형 모델 공급사 중 하나를 프레임워크 인터페이스에 녹여내는 커스텀 클래스를 작성해 보세요. 상속 패턴을 이용하고 공급사가 지정한 특정 환경 변수 접두사가 있을 때 자동으로 제공업체(provider) 이름이 감지되도록 메서드를 구현해 주세요.
   - 7.2.3절의 제공업체 자동 식별 규칙은 3개의 계층 구조를 갖습니다. 만일 사용자가 로컬 컴퓨터의 시스템 환경에 `OPENAI_API_KEY`도 세팅해 두고 동시에 `LLM_BASE_URL="http://localhost:11434/v1"`도 동시에 할당해 두었다면, 프레임워크는 궁극적으로 어떤 LLM을 제공업체로 선택할까요? 이 판정 우선순위 설계에 모순이나 충돌은 없는지, 더 합리적인 방식이 있다면 대안을 작성해 주세요.
   - 본문에서 다룬 `VLLM` 및 `Ollama` 외에도 최근 추론 병렬 처리 속도로 급부상하는 `SGLang` 엔진이 있습니다. 웹 검색 등을 동원해 `SGLang`이 내부적으로 지닌 처리 원리나 특장점을 파악한 뒤, 이들 세 도구(`VLLM`, `SGLang`, `Ollama`)의 자원 요구량, 속도, 입문 난이도, 모바일/일반 디바이스 탑재 가능 여부를 종합적으로 대조 평가해 주세요.

3. 7.3절에서는 프레임워크 자료구조의 삼총사인 `Message`, `Config`, `Agent`를 정립했습니다. 다음 논제를 분석해 주세요:
   - `Message` 설계 시 일반적인 파이썬 딕셔너리가 아닌 `Pydantic`의 `BaseModel`을 기반으로 삼아 얻는 시스템 설계상 이점은 무엇인가요?
   - `Agent` 클래스에 설계된 핵심 골격을 보면 내부의 저수준 실행 루프를 캡슐화하고 상속받은 구체 클래스에 구체적 거동(`run`, `_execute` 등)을 떠넘기는 설계 양식을 지닙니다. 이러한 디자인 패턴의 공식 명칭은 무엇이며, 이 구조를 채택함으로써 아키텍처의 균일성을 유지하는 데 어떤 이득이 주어지나요?
   - `Config` 객체에 대개 싱글톤(Singleton) 아키텍처 패턴이 자주 고려됩니다. 싱글톤 패턴이란 무엇이며, 왜 설정 인스턴스 전역 공유에 싱글톤이 필수적으로 논의되는지, 그리고 만일 멀티스레드나 분산 병렬 요청 상황에서 매번 설정을 재생성해 쓸 때 어떤 공유 메모리 동기화 버그가 나타날 수 있는지 적어주세요.

4. 7.4절에서는 네 가지 대표 Agent 구동 모델을 프레임워크에 안착시켰습니다.
   > **안내**: 실제 개발 환경을 설정하여 직접 확인해 보시는 것을 권장합니다.
   - 4장의 ReAct 구현체 코드와 7.4.2절에 구현한 프레임워크 기반의 ReActAgent 클래스를 나란히 열어 두고, 3가지 뚜렷한 코드 발전적 변화 사항을 요약해 주세요. 이 변화들이 왜 코드의 강건성에 기여하는지도 함께 분석해야 합니다.
   - `ReflectionAgent`는 내부적으로 피드백 루프를 반복하여 완성도를 끌어올립니다. 여기에 "품질 판단 임계값 감지기"를 더해 리팩터링해 봅시다. LLM에게 현재 작성 단계의 결과에 대해 1~10점 사이의 점수를 별도 부여하도록 쿼리하고, 점수가 사용자가 임의로 지정한 임계값(예: 8점) 이상을 달성했다면 추가 반복을 스킵하고 즉시 결과를 반환하도록 조기 종료(early-stopping) 논리를 추가해 보세요.
   - 기존의 단선적인 추론 경로에서 벗어나, 각 분기마다 LLM이 여러 개의 생각 후보(branch)를 뻗어나가고 이를 탐색 평가하여 최고의 논리 지점을 택해 진행하는 `Tree-of-Thought Agent` 클래스를 직접 작성해 보세요. 이 클래스 역시 `Agent` 기본 추상 클래스를 올바르게 상속하여 규격을 맞춰야 합니다.

5. 7.5절에서는 도구를 다루는 인프라를 마련했습니다. 다음 논제들을 고려해 주세요:
   - `BaseTool` 추상 클래스가 모든 하위 도구에게 통일된 입출력 포맷 규격을 지키도록 추상 인터페이스 메서드를 강제하는 것이 왜 시스템 통합 관점에서 중요한가요? 만일 검색 도구처럼 하나의 실행 메서드 반환으로 제목, 요약문, 웹 링크 등 구조화된 복합 결과 세트를 도출해야 한다면, 이 통합 인터페이스의 포맷 형식을 해치지 않고 어떻게 이를 처리하는 것이 가장 정석에 가깝습니까?
   - 7.5.3절의 `ToolChain` 기능을 실무 워크플로에 적용해 봅니다. 최소 3개 이상의 상이한 도구(예: 실시간 환율 검색 -> 단가 계산 -> 이메일 발송 템플릿 생성 등)가 결합되어 일하는 가상의 시나리오를 정의하고, 도구 체인이 각 노드의 아웃풋을 다음 노드의 인풋 템플릿으로 변환 전달하는 관계를 도표나 텍스트 다이어그램으로 설계해 보세요.
   - 비동기 실행 모듈(`AsyncToolExecutor`)은 내부적으로 ThreadPool을 쥐고 일을 나누어 처리합니다. 일반적으로 어떠한 성격의 도구 태스크(예: 계산 vs 네트워크 웹 스크래핑)들이 이 비동기 병렬 이득을 가장 크게 취할 수 있는지 원인을 기술해 주세요.

6. 프레임워크의 확장성은 설계 우수성을 평가하는 리트머스 시험지입니다. 여러분은 이제 다음의 요구사항을 `HelloAgents` 아키텍처에 구현하려 합니다:
   - 먼저, Agent가 답변 한 글자씩 실시간으로 타이핑하듯 출력하여 주는 "스트리밍 출력 연동"을 프레임워크 기본 속성으로 입혀야 합니다. 어느 모듈(어느 클래스 및 메서드)이 수정되거나 어떤 Generator 형태의 설계 인터페이스가 추가로 노출되어야 하는지 개발 구현 설계 기획안을 수립해 주세요.
   - 대화가 깊어지면 과거 메시지를 다듬거나 특정 과거의 대답 시점으로 되돌아가 다른 대답 분기를 타는 "멀티턴 대화 복원 및 브랜치 이력 관리 모듈"이 필요해집니다. 이 이력 관리를 담당할 신규 클래스는 어떻게 명세되어야 하고, 7.3절에서 구현한 `Message` 구조와 어떻게 결합하는 것이 유연할까요?
   - 마지막으로, 코어 프레임워크 파일을 전혀 건드리지 않고 개발자들이 서드파티 플러그인(Plug-in)을 작성하여 등록하는 것만으로 새로운 Agent 종류나 커스텀 도구를 프레임워크에 추가 적용할 수 있게 해주는 "플러그인 로더 메커니즘"을 설계하려 합니다. 플러그인 폴더를 자동으로 스캔(reflection)하여 컴포넌트를 주입하는 구조의 간이 흐름 설계도를 작성하고 설명해 주세요.

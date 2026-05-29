# 4장: 클래식 에이전트 패러다임 구축하기

이전 장에서는 현대 에이전트의 "뇌" 역할을 하는 대규모 언어 모델에 대해 깊이 탐구했습니다. 우리는 모델 내부의 트랜스포머 아키텍처, 모델과의 상호 작용 방법, 모델의 기능적 한계에 대해 배웠습니다. 이제는 이러한 이론적 지식을 실전으로 옮겨, 우리의 손으로 직접 에이전트를 구축해 볼 시간입니다.

현대 에이전트의 핵심 기능은 대규모 언어 모델의 추론 능력을 외부 세계와 연결하는 데 있습니다. 사용자의 의도를 자율적으로 이해하고, 복잡한 작업을 분해하며, 코드 인터프리터, 검색 엔진, API와 같은 일련의 "도구"를 호출하여 정보를 얻고 작업을 실행함으로써 목표를 달성할 수 있습니다. 하지만 에이전트가 만능은 아닙니다. 대형 모델 고유의 "환각(hallucination)" 문제, 복잡한 작업에서 발생할 수 있는 추론 루프, 잘못된 도구 사용과 같은 에이전트의 기능적 한계 상황도 존재합니다.

에이전트의 "생각(thinking)"과 "행동(acting)" 과정을 더 잘 조직화하기 위해 업계에서는 여러 가지 클래식 아키텍처 패러다임이 등장했습니다. 이 장에서는 가장 대표적인 세 가지 패러다임에 초점을 맞추고, 처음부터 단계별로 이를 직접 구현해 보겠습니다.

- **ReAct (Reasoning and Acting):** "생각"과 "행동"을 긴밀하게 결합하여 에이전트가 실행하면서 생각하고 동적으로 조정할 수 있도록 하는 패러다임입니다.
- **Plan-and-Solve:** 에이전트가 먼저 전체 행동 계획을 생성한 다음 이를 엄격하게 실행하는 "행동하기 전에 생각하기" 패러다임입니다.
- **Reflection (성찰):** 에이전트에 "자가 성찰" 능력을 부여하여 자가 비판과 수정을 통해 결과를 최적화하는 패러다임입니다.

이러한 내용을 이해하고 나면 다음과 같은 의문이 들 수 있습니다. LangChain이나 LlamaIndex처럼 이미 우수한 프레임워크가 많이 존재하는데, 왜 굳이 "바퀴를 다시 발명"해야 할까요? 그 이유는 성숙한 프레임워크가 엔지니어링 효율성 면에서는
큰 장점을 갖지만, 고도로 추상화된 도구를 직접 가져다 쓰는 것만으로는 내부 설계 메커니즘이 어떻게 작동하는지,
어떤 이점을 주는지 깊이 이해하기 어렵기 때문입니다. 둘째,
이 과정을 통해 프로젝트에서 마주하는 실제 엔지니어링 문제를 직접 경험해볼 수 있습니다. 프레임워크는 모델 출력 형식 파싱,
도구 호출 실패 시 재시도, 에이전트가 무한 루프에 빠지는 것 방지 등 수많은 문제를 대신 처리해 줍니다. 이러한 문제를 직접 다루어 보는 것이 시스템 설계 능력을 기르는 가장 직접적인 방법입니다. 마지막으로 가장 중요한 점은,
설계 원리를 마스터해야 프레임워크의 단순한 "사용자"에서 지능형 애플리케이션의 "창조자"로 진정으로 거듭날 수 있기 때문입니다. 표준 컴포넌트가 여러분의 복잡한 요구사항을 충족하지 못할 때,
여러분은 에이전트를 고도로 커스터마이징하거나 심지어 바닥부터 완전히 새로운 에이전트를 직접 구축할 수 있는 능력을 갖추게 될 것입니다.

## 4.1 환경 준비 및 기본 도구 정의

에이전트 구축을 시작하기 전에 개발 환경을 설정하고 몇 가지 기본 컴포넌트를 정의해야 합니다. 이를 통해 중복 작업을 피하고 이후 다양한 패러다임을 구현할 때 핵심 로직에 더 집중할 수 있습니다.

### 4.1.1 의존성 설치

이 책의 실습 부분은 주로 Python 언어를 사용하며, Python 3.10 이상 버전을 권장합니다. 먼저 대규모 언어 모델과의 상호 작용을 위한 `openai` 라이브러리와 API 키를 안전하게 관리하기 위한 `python-dotenv` 라이브러리가 설치되어 있는지 확인해 주세요.

터미널에서 다음 명령을 실행합니다.

```bash
pip install openai python-dotenv
```

### 4.1.2 API 키 설정

코드를 더 범용적으로 만들기 위해 모델 서비스 관련 정보(모델 ID, API 키, 서비스 주소)를 환경 변수로 통일하여 설정합니다.

1. 프로젝트 루트 디렉터리에 `.env`라는 이름의 파일을 생성합니다.
2. 이 파일에 아래 내용을 추가합니다. 필요에 따라 OpenAI 공식 서비스나 OpenAI 인터페이스와 호환되는 로컬/제3자 서비스를 가리키도록 설정할 수 있습니다.
3. API 키 획득 방법을 잘 모르겠다면 [환경 구성(Environment Configuration)](https://github.com/datawhalechina/hello-agents/blob/main/Extra-Chapter/Extra07-환경配置.md)을 참고하세요.

```bash
# .env 파일
LLM_API_KEY="YOUR-API-KEY"
LLM_MODEL_ID="YOUR-MODEL"
LLM_BASE_URL="YOUR-URL"
```

우리가 작성할 코드는 이 파일에서 환경 변수 설정을 자동으로 로드합니다.

### 4.1.3 기본 LLM 호출 함수 캡슐화

코드 구조를 더 명확하고 재사용 가능하게 만들기 위해 전용 LLM 클라이언트 클래스를 정의해 보겠습니다. 이 클래스는 모델 서비스와의 상호 작용 세부 정보를 캡슐화하여 메인 로직이 에이전트 구축에만 집중할 수 있도록 돕습니다.

```python
import os
from openai import OpenAI
from dotenv import load_dotenv
from typing import List, Dict

# .env 파일에서 환경 변수 로드
load_dotenv()

class HelloAgentsLLM:
    """
    "Hello Agents" 책을 위한 맞춤형 LLM 클라이언트.
    OpenAI 인터페이스와 호환되는 모든 서비스를 호출하는 데 사용되며, 기본적으로 스트리밍 응답을 사용합니다.
    """
    def __init__(self, model: str = None, apiKey: str = None, baseUrl: str = None, timeout: int = None):
        """
        클라이언트를 초기화합니다. 전달된 파라미터를 우선시하며, 제공되지 않은 경우 환경 변수에서 로드합니다.
        """
        self.model = model or os.getenv("LLM_MODEL_ID")
        apiKey = apiKey or os.getenv("LLM_API_KEY")
        baseUrl = baseUrl or os.getenv("LLM_BASE_URL")
        timeout = timeout or int(os.getenv("LLM_TIMEOUT", 60))
        
        if not all([self.model, apiKey, baseUrl]):
            raise ValueError("모델 ID, API 키, 서비스 주소는 제공되거나 .env 파일에 정의되어 있어야 합니다.")

        self.client = OpenAI(api_key=apiKey, base_url=baseUrl, timeout=timeout)

    def think(self, messages: List[Dict[str, str]], temperature: float = 0) -> str:
        """
        대규모 언어 모델을 호출하여 생각을 수행하고 그 응답을 반환합니다.
        """
        print(f"🧠 {self.model} 모델 호출 중...")
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=temperature,
                stream=True,
            )
            
            # 스트리밍 응답 처리
            print("✅ 대규모 언어 모델 응답 성공:")
            collected_content = []
            for chunk in response:
                if not chunk.choices:
                    continue
                content = chunk.choices[0].delta.content or ""
                print(content, end="", flush=True)
                collected_content.append(content)
            print()  # 스트리밍 출력 종료 후 줄바꿈
            return "".join(collected_content)

        except Exception as e:
            print(f"❌ LLM API 호출 중 오류 발생: {e}")
            return None

# --- 클라이언트 사용 예시 ---
if __name__ == '__main__':
    try:
        llmClient = HelloAgentsLLM()
        
        exampleMessages = [
            {"role": "system", "content": "당신은 Python 코드를 작성하는 유용한 비서입니다."},
            {"role": "user", "content": "퀵정렬(quicksort) 알고리즘을 작성해줘."}
        ]
        
        print("--- LLM 호출 중 ---")
        responseText = llmClient.think(exampleMessages)
        if responseText:
            print("\n\n--- 전체 모델 응답 ---")
            print(responseText)

    except ValueError as e:
        print(e)


>>>
--- LLM 호출 중 ---
🧠 xxxxxx 모델 호출 중...
✅ 대규모 언어 모델 응답 성공:
퀵정렬은 매우 효율적인 정렬 알고리즘입니다...
```

---

## 4.2 ReAct

LLM 클라이언트를 준비한 후, 첫 번째이자 가장 클래식한 에이전트 패러다임인 **ReAct (Reason + Act)**를 구축해 보겠습니다. ReAct는 2022년에 Shunyu Yao 등이 제안했습니다<sup>[1]</sup>.
핵심 아이디어는 **추론(Reasoning)**과 **행동(Acting)**을 명시적으로 결합하여 "생각-행동-관찰" 루프를 형성함으로써 인간이 문제를 해결하는 방식을 모방하는 것입니다.

### 4.2.1 ReAct 워크플로우

ReAct가 등장하기 전에는 주류 방법론이 두 가지 카테고리로 나뉘어 있었습니다. 하나는 **Chain-of-Thought**처럼 복잡한 논리적 추론을 유도할 수는 있지만 외부 세계와 상호 작용할 수 없어 사실적 환각에 취약한 "순수 생각(pure thinking)" 유형이었습니다. 다른 하나는 모델이 실행할 행동을 직접 출력하지만 계획 및 오류 수정 능력이 결합되지 않은 "순수 행동(pure action)" 유형이었습니다.

ReAct의 기발함은 **생각과 행동이 상호 보완적**이라는 점을 깨달은 데 있습니다. 생각이 행동을 가이드하고, 행동 결과가 다시 생각을 수정합니다.
이를 위해 ReAct 패러다임은 특별한 프롬프트 엔지니어링을 사용하여 모델의 각 단계 출력이 고정된 궤적을 따르도록 유도합니다.

- **Thought (생각):** 에이전트의 "내적 독백"입니다. 현재 상황을 분석하고, 작업을 분해하며, 다음 계획을 수립하거나 이전 단계의 결과를 반성합니다.
- **Action (행동):** 에이전트가 취하기로 결정한 구체적인 조치로, 일반적으로 `Search['화웨이 최신 폰']`처럼 외부 도구를 호출하는 것입니다.
- **Observation (관찰):** 외부 도구가 `Action`을 실행한 후 반환한 결과로, 검색 결과 요약이나 API 반환 값 등입니다.

에이전트는 이 **Thought -> Action -> Observation** 루프를 계속 반복하며, 새로운 관찰 결과를 이력에 추가하여 지속적으로 확장되는 컨텍스트를 형성합니다. 이 과정은 `Thought`에서 최종 답변을 찾았다고 판단하여 결과를 출력할 때까지 계속됩니다. 이 프로세스는 강력한 시너지 효과를 냅니다. **추론은 행동에 목적성을 더하고, 행동은 추론에 사실적 근거를 제공합니다.**

이 과정을 수식으로 표현하면 그림 4.1과 같습니다. 구체적으로 매 시간 단계 $t$에서 에이전트의 정책(즉, 대규모 언어 모델 $\pi$)은 초기 질문 $q$와 이전의 모든 "행동-관찰" 이력 궤적 $((a_1,o_1),\dots,(a_{t-1},o_{t-1}))$을 바탕으로 현재의 생각 $th_t$와 행동 $a_t$를 생성합니다.

$$\left(th_t,a_t\right)=\pi\left(q,(a_1,o_1),\ldots,(a_{t-1},o_{t-1})\right)$$

이어서 환경의 도구 $T$가 행동 $a_t$를 실행하고 새로운 관찰 결과 $o_t$를 반환합니다.

$$o_t = T(a_t)$$

이 루프는 모델이 생각 $th_t$에서 작업이 완료되었다고 판단할 때까지 계속되며 새로운 $(a_t,o_t)$ 쌍을 이력에 추가해 나갑니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/4-figures/4-1.png" alt="ReAct 패러다임의 생각-행동-관찰 시너지 루프" width="90%"/>
 ㅓ <p>그림 4.1 ReAct 패러다임의 생각-행동-관찰 시너지 루프</p>
</div>

이 메커니즘은 특히 다음과 같은 시나리오에 적합합니다.

- **외부 지식이 필요한 작업**: 실시간 정보(날씨, 뉴스, 주가) 조회, 전문 도메인 지식 검색 등.
- **정밀한 계산이 필요한 작업**: LLM의 계산 오류를 방지하기 위해 수학 문제를 계산기 도구에 위임.
- **API 상호 작용이 필요한 작업**: 데이터베이스 조작, 특정 기능을 완료하기 위해 서비스 API 호출 등.

따라서 우리는 대규모 언어 모델이 자체 지식베이스만으로는 직접 대답할 수 없는 질문에 답할 수 있도록 **외부 도구를 사용하는 능력**을 갖춘 ReAct 에이전트를 구축할 것입니다. 예를 들어, "화웨이의 최신 휴대폰은 무엇인가요? 주요 셀링 포인트는 무엇인가요?"라는 질문은 에이전트가 온라인 검색이 필요함을 이해하고, 도구를 호출해 결과를 찾고, 답변을 요약해야 합니다.

### 4.2.2 도구 정의 및 구현

대규모 언어 모델이 에이전트의 뇌라면, **도구(Tools)**는 외부 세계와 상호 작용하기 위한 "손과 발"입니다. ReAct 패러다임이 우리가 설정한 문제를 실제로 해결할 수 있으려면 에이전트에게 외부 도구를 호출하는 능력을 주어야 합니다.

이 섹션에서 설정한 목표인 "화웨이의 최신 휴대폰"에 대한 질문에 답하기 위해 에이전트에게 웹 검색 도구를 제공해야 합니다.
여기서는 API를 통해 정형화된 Google 검색 결과를 제공하고 "답변 요약 상자"나 정확한 지식 그래프 정보를 직접 반환할 수 있는 **SerpApi**를 선택합니다.

먼저 라이브러리를 설치해야 합니다.

```bash
pip install google-search-results
```

동시에 [SerpApi 공식 홈페이지](https://serpapi.com/)로 이동하여 무료 계정을 등록하고 API 키를 얻은 다음, 프로젝트 루트 디렉터리의 `.env` 파일에 추가해야 합니다.

```bash
# .env 파일
# ... (이전 LLM 설정 유지)
SERPAPI_API_KEY="YOUR_SERPAPI_API_KEY"
```

다음으로 코드를 통해 이 도구를 정의하고 관리하겠습니다. 단계별로 진행합니다. 먼저 도구의 핵심 기능을 구현한 다음 일반적인 도구 실행기를 구축합니다.

(1) 검색 도구의 핵심 로직 구현

잘 정의된 도구는 다음 세 가지 핵심 요소를 포함해야 합니다.

1. **Name (이름)**: `Action`에서 에이전트가 호출할 간결하고 고유한 식별자 (예: `Search`).
2. **Description (설명)**: 이 도구의 용도를 설명하는 명확한 자연어 설명입니다. **이것은 전체 메커니즘에서 가장 중요한 부분**인데, 대규모 언어 모델이 이 설명을 바탕으로 어떤 상황에서 어떤 도구를 사용할지 결정하기 때문입니다.
3. **Execution Logic (실행 로직)**: 실제로 작업을 수행하는 함수 또는 메서드입니다.

우리의 첫 번째 도구는 쿼리 문자열을 받아 검색 결과를 반환하는 `search` 함수입니다.

```python
from serpapi import SerpApiClient

def search(query: str) -> str:
    """
    SerpApi를 기반으로 하는 실용적인 웹 검색 엔진 도구입니다.
    검색 결과를 지능적으로 파싱하여 직접적인 답변이나 지식 그래프 정보를 우선시합니다.
    """
    print(f"🔍 [SerpApi] 웹 검색 실행 중: {query}")
    try:
        api_key = os.getenv("SERPAPI_API_KEY")
        if not api_key:
            return "오류: .env 파일에 SERPAPI_API_KEY가 구성되지 않았습니다."

        params = {
            "engine": "google",
            "q": query,
            "api_key": api_key,
            "gl": "cn",  # 국가 코드
            "hl": "zh-cn", # 언어 코드
        }
        
        client = SerpApiClient(params)
        results = client.get_dict()
        
        # 지능형 파싱: 가장 직접적인 답변 찾기를 최우선으로 함
        if "answer_box_list" in results:
            return "\n".join(results["answer_box_list"])
        if "answer_box" in results and "answer" in results["answer_box"]:
            return results["answer_box"]["answer"]
        if "knowledge_graph" in results and "description" in results["knowledge_graph"]:
            return results["knowledge_graph"]["description"]
        if "organic_results" in results and results["organic_results"]:
            # 직접적인 답변이 없으면 처음 3개 유기적 검색 결과의 요약본 반환
            snippets = [
                f"[{i+1}] {res.get('title', '')}\n{res.get('snippet', '')}"
                for i, res in enumerate(results["organic_results"][:3])
            ]
            return "\n\n".join(snippets)
        
        return f"죄송합니다. '{query}'에 대한 정보를 찾을 수 없습니다."

    except Exception as e:
        return f"검색 중 오류 발생: {e}"
```

위 코드에서는 먼저 `answer_box`(Google의 추천 스니펫 답변 상자)나 `knowledge_graph`(지식 그래프) 정보가 존재하는지 확인합니다. 존재한다면 가장 정밀한 답변을 직접 반환합니다. 그렇지 않으면 일반 검색 결과 상위 3개의 요약본을 반환하도록 대체합니다. 이러한 "지능형 파싱"은 LLM에 고품질의 정보 입력을 제공할 수 있습니다.

(2) 범용 도구 실행기 구축

에이전트가 여러 도구를 사용해야 할 때(예: 검색 외에 계산기, 데이터베이스 조회 등도 필요할 때), 우리는 이러한 도구들을 등록하고 발송할 통일된 관리자가 필요합니다. 이를 위해 `ToolExecutor` 클래스를 생성합니다.

```python
from typing import Dict, Any

class ToolExecutor:
    """
    도구를 관리하고 실행하는 역할을 담당하는 도구 실행기입니다.
    """
    def __init__(self):
        self.tools: Dict[str, Dict[str, Any]] = {}

    def registerTool(self, name: str, description: str, func: callable):
        """
        도구함에 새로운 도구를 등록합니다.
        """
        if name in self.tools:
            print(f"경고: '{name}' 도구가 이미 존재하므로 덮어씁니다.")
        self.tools[name] = {"description": description, "func": func}
        print(f"도구 '{name}'가 등록되었습니다.")

    def getTool(self, name: str) -> callable:
        """
        이름으로 도구의 실행 함수를 가져옵니다.
        """
        return self.tools.get(name, {}).get("func")

    def getAvailableTools(self) -> str:
        """
        사용 가능한 모든 도구의 서식화된 설명 문자열을 가져옵니다.
        """
        return "\n".join([
            f"- {name}: {info['description']}" 
            for name, info in self.tools.items()
        ])
```

(3) 테스트

이제 `search` 도구를 `ToolExecutor`에 등록하고 호출을 시뮬레이션하여 전체 프로세스가 제대로 작동하는지 확인해 보겠습니다.

```python
# --- 도구 초기화 및 사용 예시 ---
if __name__ == '__main__':
    # 1. 도구 실행기 초기화
    toolExecutor = ToolExecutor()

    # 2. 실용적인 검색 도구 등록
    search_description = "웹 검색 엔진입니다. 현재 사건, 사실 정보 및 자체 지식베이스에 없는 정보에 대한 질문에 답해야 할 때 이 도구를 사용하십시오."
    toolExecutor.registerTool("Search", search_description, search)

    # 3. 사용 가능한 도구 출력
    print("\n--- 사용 가능한 도구 목록 ---")
    print(toolExecutor.getAvailableTools())

    # 4. 에이전트의 Action 호출 시뮬레이션, 이번에는 실시간 질문을 던집니다.
    print("\n--- Action 실행: Search['NVIDIA의 최신 GPU 모델은 무엇인가요'] ---")
    tool_name = "Search"
    tool_input = "NVIDIA의 최신 GPU 모델은 무엇인가요"

    tool_function = toolExecutor.getTool(tool_name)
    if tool_function:
        observation = tool_function(tool_input)
        print("--- 관찰 (Observation) ---")
        print(observation)
    else:
        print(f"오류: 이름이 '{tool_name}'인 도구를 찾을 수 없습니다.")

>>>
도구 'Search'가 등록되었습니다.

--- 사용 가능한 도구 목록 ---
- Search: 웹 검색 엔진입니다. 현재 사건, 사실 정보 및 자체 지식베이스에 없는 정보에 대한 질문에 답해야 할 때 이 도구를 사용하십시오.

--- Action 실행: Search['NVIDIA의 최신 GPU 모델은 무엇인가요'] ---
🔍 [SerpApi] 웹 검색 실행 중: NVIDIA의 최신 GPU 모델은 무엇인가요
--- 관찰 (Observation) ---
[1] GeForce RTX 50 시리즈 그래픽 카드
GeForce RTX™ 50 시리즈 GPU는 NVIDIA Blackwell 아키텍처를 기반으로 하여 게이머와 크리에이터에게 새로운 게임 경험을 선사합니다. RTX 50 시리즈는 강력한 AI 컴퓨팅 파워를 갖추어 한층 업그레이드된 경험과 실감 나는 그래픽을 제공합니다.

[2] GeForce 시리즈 최신 세대 및 이전 세대 그래픽 카드 비교
최신 RTX 30 시리즈 그래픽 카드를 이전 RTX 20 시리즈, GTX 10 및 900 시리즈 그래픽 카드와 비교합니다. 사양, 기능, 기술 지원 등을 확인하세요.

[3] GeForce 그래픽 카드 | NVIDIA
DRIVE AGX. AI 기반 지능형 차량 시스템을 위한 강력한 차량용 컴퓨팅 파워 · Clara AGX. 혁신적인 의료 기기 및 이미징을 위한 AI 컴퓨팅. 게이밍 및 창작. GeForce. 그래픽 카드, 게이밍 솔루션, AI 탐색...
```

이로써 에이전트가 현실 세계의 인터넷과 연결할 수 있는 `Search` 도구를 갖추게 되었으며, 이는 후속 ReAct 루프의 든든한 기초가 됩니다.

### 4.2.3 ReAct 에이전트 코드 구현

이제 LLM 클라이언트와 도구 실행기라는 독립된 컴포넌트들을 조립하여 완전한 ReAct 에이전트를 구축하겠습니다. `ReActAgent` 클래스를 통해 핵심 로직을 캡슐화하겠습니다. 이해를 돕기 위해 이 클래스의 구현 프로세스를 핵심 부분으로 나누어 설명합니다.

(1) 시스템 프롬프트 설계

프롬프트는 전체 ReAct 메커니즘의 초석으로, 대규모 언어 모델에 작동 지침을 제공합니다. 우리는 사용 가능한 도구, 사용자의 질문, 중간 단계의 상호 작용 이력이 동적으로 삽입될 템플릿을 신중하게 설계해야 합니다.

```bash
# ReAct 프롬프트 템플릿
REACT_PROMPT_TEMPLATE = """
당신은 외부 도구를 호출할 수 있는 지능형 비서입니다.

사용 가능한 도구는 다음과 같습니다:
{tools}

반드시 다음 형식을 엄격히 준수하여 응답해 주세요:

Thought: 문제를 분석하고, 작업을 분해하며, 다음 행동을 계획하는 데 사용하는 생각 과정.
Action: 취하기로 결정한 행동으로, 반드시 다음 형식 중 하나여야 합니다:
- `{{tool_name}}[{{tool_input}}]`: 사용 가능한 도구를 호출합니다.
- `Finish[최종 답변]`: 최종 답변을 얻었다고 판단했을 때 사용합니다.
- 사용자의 최종 질문에 대답하기에 충분한 정보를 수집했다면, 반드시 Action: 필드 뒤에 `Finish[최종 답변]`을 사용하여 최종 답변을 출력해야 합니다.

이제 다음 문제를 해결하기 시작해 주세요:
Question: {question}
History: {history}
"""
```

이 템플릿은 에이전트와 LLM 간의 상호 작용 규격을 정의합니다.

- **역할 정의**: "당신은 외부 도구를 호출할 수 있는 지능형 비서입니다"를 통해 LLM의 역할을 설정합니다.
- **도구 목록 (`{tools}`)**: LLM이 사용할 수 있는 "손과 발"이 무엇인지 알려줍니다.
- **형식 약속 (`Thought`/`Action`)**: 가장 중요한 부분으로, LLM의 출력을 정형화하여 코드를 통해 의도를 정확하게 파싱할 수 있도록 강제합니다.
- **동적 컨텍스트 (`{question}`/`{history}`)**: 사용자의 원래 질문과 지속적으로 누적되는 상호 작용 이력을 주입하여 LLM이 완전한 컨텍스트를 기반으로 의사 결정을 내릴 수 있도록 합니다.

(2) 핵심 루프 구현

`ReActAgent` 의 핵심은 작업이 완료되거나 최대 단계 제한에 도달할 때까지 "프롬프트 포맷팅 -> LLM 호출 -> Action 실행 -> 결과 통합"을 지속적으로 반복하는 루프입니다.

```python
import re

class ReActAgent:
    def __init__(self, llm_client: HelloAgentsLLM, tool_executor: ToolExecutor, max_steps: int = 5):
        self.llm_client = llm_client
        self.tool_executor = tool_executor
        self.max_steps = max_steps
        self.history = []

    def run(self, question: str):
        """
        ReAct 에이전트를 실행하여 질문에 대답합니다.
        """
        self.history = []  # 매 실행마다 이력 초기화
        current_step = 0

        while current_step < self.max_steps:
            current_step += 1
            print(f"--- Step {current_step} ---")

            # 1. 프롬프트 포맷팅
            tools_desc = self.tool_executor.getAvailableTools()
            history_str = "\n".join(self.history)
            prompt = REACT_PROMPT_TEMPLATE.format(
                tools=tools_desc,
                question=question,
                history=history_str
            )

            # 2. LLM을 호출하여 생각 유도
            messages = [{"role": "user", "content": prompt}]
            response_text = self.llm_client.think(messages=messages)

            if not response_text:
                print("오류: LLM이 유효한 응답을 반환하는 데 실패했습니다.")
                break

            # ... (후속 파싱, 실행, 통합 단계)
```

`run` 메서드는 에이전트의 진입점입니다. 이 안의 `while` 루프가 ReAct 패러다임의 본체를 구성하며, `max_steps` 매개변수는 에이전트가 무한 루프에 빠져 자원을 소진하는 것을 방지하는 중요한 안전장치 역할을 합니다.

(3) 출력 파서 구현

LLM은 플레인 텍스트를 반환하므로 우리는 그 안에서 `Thought`와 `Action`을 정확히 추출해야 합니다. 이는 보통 정규 표현식을 사용하는 몇 가지 보조 파싱 함수를 통해 수행됩니다.

```python
# (이 메서드들은 ReActAgent 클래스의 일부입니다)
    def _parse_output(self, text: str):
        """LLM 출력을 파싱하여 Thought와 Action을 추출합니다.
        """
        # Thought: Action: 또는 텍스트 끝까지 매칭
        thought_match = re.search(r"Thought:\s*(.*?)(?=\nAction:|$)", text, re.DOTALL)
        # Action: 텍스트 끝까지 매칭
        action_match = re.search(r"Action:\s*(.*?)$", text, re.DOTALL)
        thought = thought_match.group(1).strip() if thought_match else None
        action = action_match.group(1).strip() if action_match else None
        return thought, action

    def _parse_action(self, action_text: str):
        """Action 문자열을 파싱하여 도구 이름과 입력을 추출합니다.
        """
        match = re.match(r"(\w+)\[(.*)\]", action_text, re.DOTALL)
        if match:
            return match.group(1), match.group(2)
        return None, None
```

- `_parse_output`: LLM의 전체 응답에서 `Thought`와 `Action` 두 가지 핵심 부분을 분리하는 역할을 합니다.
- `_parse_action`: `Action` 문자열을 추가로 파싱합니다. 예를 들어, `Search[화웨이 최신 폰]`에서 도구 이름인 `Search`와 도구 입력인 `화웨이 최신 폰`을 추출합니다.

(4) 도구 호출 및 실행

```python
# (이 로직은 run 메서드의 while 루프 내부에 위치합니다)
            # 3. LLM 출력 파싱
            thought, action = self._parse_output(response_text)

            if thought:
                print(f"Thought: {thought}")

            if not action:
                print("경고: 유효한 Action을 파싱하는 데 실패하여 프로세스가 종료됩니다.")
                break

            # 4. Action 실행
            if action.startswith("Finish"):
                # Finish 명령인 경우, 최종 답변을 추출하고 종료
                final_answer_match = re.match(r"Finish\[(.*)\]", action)
                final_answer = final_answer_match.group(1) if final_answer_match else action
                print(f"🎉 최종 답변: {final_answer}")
                return final_answer

            tool_name, tool_input = self._parse_action(action)
            if not tool_name or not tool_input:
                # ... 잘못된 Action 형식 처리 ...
                print("경고: Action 형식이 올바르지 않습니다.")
                continue

            print(f"🎬 Action: {tool_name}[{tool_input}]")

            tool_function = self.tool_executor.getTool(tool_name)
            if not tool_function:
                observation = f"오류: 이름이 '{tool_name}'인 도구를 찾을 수 없습니다."
            else:
                observation = tool_function(tool_input)  # 실제 도구 호출
```

이 코드는 `Action` 실행의 핵심입니다. 먼저 `Finish` 명령인지 확인하여 맞다면 프로세스를 종료합니다. 그렇지 않으면 `tool_executor`를 통해 해당 도구 함수를 얻어 실행하고 `observation`을 획득합니다.

(5) 관찰 결과의 통합

마지막 단계이자 폐쇄 루프를 형성하는 핵심은, 도구 실행 후 얻은 `Action` 자체와 `Observation`을 다시 이력에 추가하여 다음 루프를 위한 새로운 컨텍스트를 제공하는 것입니다.

```python
# (이 로직은 도구 호출 뒤에 이어지며, while 루프의 마지막에 위치합니다)
            print(f"👀 관찰 (Observation): {observation}")

            # 이번 라운드의 Action과 Observation을 이력에 추가
            self.history.append(f"Action: {action}")
            self.history.append(f"Observation: {observation}")

        # 루프 종료
        print("최대 단계에 도달하여 프로세스가 종료되었습니다.")
        return None
```

`self.history`에 `Observation`을 추가함으로써 에이전트는 다음 라운드에서 프롬프트를 생성할 때 이전 행동의 결과를 "볼" 수 있고, 이에 맞춰 새로운 생각과 계획을 세울 수 있습니다.

(6) 실행 인스턴스 및 분석

위의 모든 부분을 조립하면 완전한 `ReActAgent` 클래스가 완성됩니다. 전체 코드 실행 예시는 이 책의 동반 코드 저장소의 `code` 폴더에서 확인할 수 있습니다.

다음은 실제 실행 기록입니다.

```
도구 'Search'가 등록되었습니다.

--- Step 1 ---
🧠 xxxxxx 모델 호출 중...
✅ 대규모 언어 모델 응답 성공:
Thought: 이 질문에 답하기 위해, 화웨이에서 최근 출시한 휴대폰 모델과 주요 특징을 검색해야 합니다. 이 정보는 내 기존 지식베이스에 없을 수 있으므로 검색 엔진을 사용하여 최신 데이터를 얻어야 합니다.
Action: Search[화웨이 최신 휴대폰 모델 및 주요 셀링 포인트]
Thought: 이 질문에 답하기 위해, 화웨이에서 최근 출시한 휴대폰 모델과 주요 특징을 검색해야 합니다. 이 정보는 내 기존 지식베이스에 없을 수 있으므로 검색 엔진을 사용하여 최신 데이터를 얻어야 합니다.
🎬 Action: Search[화웨이 최신 휴대폰 모델 및 주요 셀링 포인트]
🔍 [SerpApi] 웹 검색 실행 중: 화웨이 최신 휴대폰 모델 및 주요 셀링 포인트
👀 관찰 (Observation): [1] 화웨이 휴대폰 - 화웨이 공식 웹사이트
스마트폰 ; Mate 시리즈. Extraordinary Flagship · HUAWEI Mate XTs. Extraordinary Master ; Pura 시리즈. Pioneer Imaging · HUAWEI Pura 80 Pro+ ; Pocket 시리즈. Aesthetic New Chapter. HUAWEI Pocket 2 ; nova 시리즈. Professional Portrait.

[2] 2025년 가장 가치 있는 화웨이 휴대폰은 무엇인가요? 화웨이 휴대폰 추천 및 시장 분석...
현재 화웨이 휴대폰의 가장 큰 판매 포인트는 HarmonyOS system과 화웨이라는 브랜드 신뢰도입니다. 예산에 따라 구매할 만한 몇 가지 화웨이 시리즈 휴대폰은 다음과 같습니다. 현재 가장 인기 있고 탑재된...

[3] 2025년 어떤 새 화웨이 휴대폰이 가장 가성비가 좋나요? 10가지 새 화웨이 휴대폰 추천
화웨이를 선택할 때는 주로 고급형 휴대폰을 추천하며, Mate 70과 Pura 70 시리즈가 최신 플래그십 모델입니다. HUAWEI Mate 70. 장점은 카메라 구성이 여전히 최상급이고, 전체 초점 거리 커버리지가 가능하여 전문 사진 촬영에 적합하며, 정교한 마감과 우수한 야외 낙하 방지 기능이 있습니다...

--- Step 2 ---
🧠 xxxxxx 모델 호출 중...
✅ 대규모 언어 모델 응답 성공:
Thought: 검색 결과에 따르면, 화웨이의 최신 플래그십 모델에는 Mate 70과 Pura 80 Pro+가 포함되어 있습니다. 최신 모델과 주요 판매 포인트를 결정하기 위해 이 정보에 초점을 맞추겠습니다. 제공된 링크에서 Mate 70 시리즈와 Pura 80 Pro+ 모두 최근 출시된 제품이지만, 어느 것이 진짜 "최신"인지는 추가 확인이 필요합니다. 한편, 이 정보들에서 주요 판매 포인트를 추출할 수 있습니다.
Action: Finish[최신 정보에 따르면, 화웨이의 최신 휴대폰은 HUAWEI Pura 80 Pro+ 또는 HUAWEI Mate 70일 가능성이 높습니다. 그중 HUAWEI Mate 70의 주요 셀링 포인트는 최상급 카메라 구성, 전문 사진 촬영에 적합한 전 초점 거리 커버리지, 정교한 마감, 우수한 야외 낙하 방지 등입니다. 반면 HUAWEI Pura 80 Pro+는 개척자적인 이미징(Pioneer Imaging) 기술을 강조하고 있습니다.]
Thought: 검색 결과에 따르면, 화웨이의 최신 플래그십 모델에는 Mate 70과 Pura 80 Pro+가 포함되어 있습니다. 최신 모델과 주요 판매 포인트를 결정하기 위해 이 정보에 초점을 맞추겠습니다. 제공된 링크에서 Mate 70 시리즈와 Pura 80 Pro+ 모두 최근 출시된 제품이지만, 어느 것이 진짜 "최신"인지는 추가 확인이 필요합니다. 한편, 이 정보들에서 주요 판매 포인트를 추출할 수 있습니다.
🎉 최종 답변: 최신 정보에 따르면, 화웨이의 최신 휴대폰은 HUAWEI Pura 80 Pro+ 또는 HUAWEI Mate 70일 가능성이 높습니다. 그중 HUAWEI Mate 70의 주요 셀링 포인트는 최상급 카메라 구성, 전문 사진 촬영에 적합한 전 초점 거리 커버리지, 정교한 마감, 우수한 야외 낙하 방지 등입니다. 반면 HUAWEI Pura 80 Pro+는 개척자적인 이미징(Pioneer Imaging) 기술을 강조하고 있습니다.
```

위의 출력을 보면 에이전트가 자신의 사고 사슬을 명확하게 보여주고 있습니다. 먼저 자신의 지식이 부족하여 검색 도구를 사용해야 함을 인식하고, 검색 결과를 기반으로 추론 및 요약을 진행하여 단 2단계 만에 최종 답변에 도달했습니다.

모델의 지식과 인터넷 정보는 끊임없이 업데이트되므로 실제 실행 결과는 이와 조금 다를 수 있습니다. 이 단락이 작성된 2025년 9월 8일 기준으로 검색 결과에 언급된 HUAWEI Mate 70과 HUAWEI Pura 80 Pro+는 당시 화웨이의 최신 플래그십 시리즈 휴대폰이 맞았습니다. 이는 실시간성 민감 문제를 처리하는 데 있어 ReAct 패러다임이 가진 강력한 역량을 잘 보여줍니다.

### 4.2.4 ReAct의 특징, 한계점 및 디버깅 기술

ReAct 에이전트를 직접 구현해 봄으로써 작동 프로세스를 마스터했을 뿐만 아니라 내부 메커니즘도 더 깊이 이해하게 되었을 것입니다. 모든 기술 패러다임에는 돋보이는 부분과 개선해야 할 영역이 공존합니다. 이 섹션에서는 ReAct를 요약해 봅니다.

(1) ReAct의 주요 특징

1. **높은 설명 가능성 (High Interpretability)**: ReAct의 가장 큰 장점 중 하나는 투명성입니다. `Thought` 사슬을 통해 우리는 각 단계에서 에이전트의 "심적 경로", 즉 왜 이 도구를 선택했는지, 다음에는 무엇을 할 계획인지를 명확하게 볼 수 있습니다. 이는 에이전트의 행동을 이해하고 신뢰하며 디버깅하는 데 필수적입니다.
2. **동적 계획 및 오류 수정 능력 (Dynamic Planning and Error Correction)**: 전체 계획을 한 번에 세우는 패러다임과 달리, ReAct는 "한 걸음 걷고, 한 걸음 보기" 방식입니다. 매 단계마다 외부 세계로부터 얻은 `Observation`에 근거하여 이후의 `Thought`와 `Action`을 동적으로 조정합니다. 이전 검색 결과가 만족스럽지 않다면 다음 단계에서 검색어를 수정하여 다시 시도할 수 있습니다.
3. **도구 시너지 능력 (Tool Synergy)**: ReAct 패러다임은 대규모 언어 모델의 추론 능력과 외부 도구의 실행 능력을 자연스럽게 결합합니다. LLM은 전략 수립(계획 및 추론)을 담당하고 도구는 특정 문제 해결(검색, 계산)을 담당하여 두 영역이 시너지를 내며 지식 실시간성, 계산 정확도 등 단일 LLM이 가진 고유한 한계를 돌파합니다.

(2) ReAct 고유의 한계점

1. **LLM 자체 능력에 대한 강력한 의존성**: ReAct 프로세스의 성공 여부는 기본 LLM의 종합적인 능력에 크게 의존합니다. LLM의 논리적 추론 능력, 지시문 준수 능력, 포맷팅 출력 능력이 부족하다면, `Thought` 단계에서 잘못된 계획을 세우거나 `Action` 단계에서 형식에 맞지 않는 지시를 생성하기 쉬워 전체 프로세스가 중단될 수 있습니다.
2. **실행 효율성 문제**: 단계별 특성 때문에 작업을 완료하려면 일반적으로 여러 번의 LLM 호출이 필요합니다. 호출할 때마다 네트워크 대기 시간과 컴퓨팅 비용이 동반됩니다. 많은 단계를 필요로 하는 복잡한 작업의 경우, 이러한 직렬 방식의 "생각-행동" 루프는 높은 총 시간과 비용을 초래할 수 있습니다.
3. **프롬프트의 취약성 (Prompt Fragility)**: 전체 메커니즘의 안정적인 작동은 세심하게 설계된 프롬프트 템플릿 위에서 성립합니다. 템플릿의 사소한 변경이나 단어 선택의 차이조차도 LLM의 행동에 영향을 미칠 수 있습니다. 또한 모든 모델이 일관되게 규정된 형식을 따를 수 있는 것은 아니므로 실제 적용에서 불확실성이 증가합니다.
4. **지역 최적해에 빠질 위험 (Local Optima)**: 단계별 의사 결정 모드는 에이전트에게 장기적인 글로벌 계획이 부족함을 의미합니다. 즉각적인 `Observation`에 끌려 장기적으로는 최선이 아니지만 단기적으로만 옳아 보이는 경로를 선택하거나, 심지어 특정 상황에서 제자리걸음을 하는 루프에 빠질 수 있습니다.

(3) 디버깅 기술

구축한 ReAct 에이전트가 예상치 못한 행동을 보일 때, 다음과 같은 측면에서 디버깅할 수 있습니다.

- **전체 프롬프트 확인**: 각 LLM 호출 전에 모든 이력이 포함된 포맷팅이 완료된 최종 전체 프롬프트를 출력해 보세요. 이는 LLM의 결정 원천을 추적하는 가장 직접적인 방법입니다.
- **원시 출력 분석**: 출력 파싱이 실패할 때(예: 정규식이 `Action`을 찾지 못할 때) LLM이 반환한 처리되지 않은 원시 텍스트를 반드시 출력해 보세요. 이를 통해 LLM이 형식을 따르지 않은 것인지 혹은 파싱 로직에 오류가 있는 것인지 파악할 수 있습니다.
- **도구 입출력 검증**: 에이전트가 생성한 `tool_input`이 도구 함수가 예상하는 형식인지 확인하고, 도구가 반환하는 `observation` 역시 에이전트가 이해하고 처리할 수 있는 형태인지 보장해야 합니다.
- **프롬프트 내 예시 조정 (Few-shot Prompting)**: 모델이 자주 실수를 한다면, 프롬프트 안에 한두 개의 완성도 높은 성공적인 "Thought-Action-Observation" 사례를 추가하여 모델이 예시를 보고 지시사항을 더 잘 따르도록 유도할 수 있습니다.
- **다른 모델 또는 파라미터 시도**: 더 성능이 좋은 모델로 전환하거나 출력의 결정론적 성격을 확보하기 위해 `temperature` 파라미터를 낮추는(보통 0으로 설정) 등의 방법을 쓰면 문제를 해결할 수 있습니다.

---

## 4.3 Plan-and-Solve

반응적이고 단계별 의사 결정을 내리는 에이전트 패러다임인 ReAct를 마스터한 후,
다음으로 스타일은 매우 다르지만 그 못지않게 강력한 패러다임인 **Plan-and-Solve**를 살펴보겠습니다. 이름 그대로 이 패러다임은 작업을 **계획 먼저 하고,
그 다음 실행(Plan first, then Solve)**하는 두 단계로 명시적으로 나눕니다.

ReAct가 현장의 단서(관찰)를 바탕으로 추론하고 수사 방향을 언제든 조정하는 노련한 형사라면,
Plan-and-Solve는 시공 전에 반드시 완벽한 설계도(계획)를 그린 다음 그 설계도에 따라 엄격하게 시공(실행)해 나가는 건축가에 가깝습니다.
실제로 현재 우리가 사용하는 많은 대형 모델 도구의 에이전트 모드에도 이러한 설계 패턴이 결합되어 있습니다.

### 4.3.1 Plan-and-Solve 작동 원리

Plan-and-Solve 프롬프팅은 2023년 Lei Wang 등이 제안했습니다<sup>[2]</sup>. 핵심 동기는 여러 단계를 거쳐야 하는 복잡한 문제를 다룰 때 생각의 사슬(Chain-of-thought)이 쉽게 "삼천포로 빠지는" 문제를 해결하는 것입니다.

매 단계마다 생각과 행동을 융합하는 ReAct와 달리, Plan-and-Solve는 그림 4.2와 같이 전체 프로세스를 두 가지 핵심 단계로 디커플링합니다.

1. **계획 단계 (Planning Phase)**: 먼저 에이전트는 사용자의 온전한 질문을 받습니다. 첫 번째 할 일은 바로 문제를 해결하거나 도구를 호출하는 것이 아니라,
**문제를 해체하여 명확하고 단계적인 행동 계획을 세우는 것**입니다. 이 계획 자체도 대규모 언어 모델 호출의 결과물입니다.
2. **실행 단계 (Solving Phase)**: 완전한 계획을 얻은 후 에이전트는 실행 단계에 진입합니다. **계획된 각 단계에 따라 엄격하게 순서대로 하나씩 실행**합니다.
각 단계의 실행은 독립적인 LLM 호출일 수도 있고 이전 단계 결과의 처리일 수도 있으며, 계획의 모든 단계가 완료되고 최종 답변을 얻을 때까지 반복됩니다.

이러한 "실행 전 계획" 전략은 장기적인 계획이 필요한 복잡한 작업을 처리할 때 에이전트가 일관되게 목표를 유지할 수 있게 도우며 중간 단계에서 길을 잃지 않도록 방지합니다.

이 두 단계 프로세스를 수식으로 표현해 볼 수 있습니다. 먼저 계획 모델 $\pi_{\text{plan}}$은 원본 질문 $q$를 기반으로 $n$개 단계가 포함된 계획 $P = (p_1, p_2, \dots, p_n)$을 생성합니다.

$$
P = \pi_{\text{plan}}(q)
$$

이어서 실행 단계에서 실행 모델 $\pi_{\text{solve}}$는 계획된 단계를 차례대로 완성해 나갑니다. $i$번째 단계에 대한 해법 $s_i$를 생성할 때 원본 질문 $q$, 전체 계획 $P$, 그리고 이전까지의 모든 단계 실행 결과인 $(s_1, \dots, s_{i-1})$에 의존하게 됩니다.

$$
s_i = \pi_{\text{solve}}(q, P, (s_1, \dots, s_{i-1}))
$$

최종 답변은 마지막 단계의 실행 결과 $s_n$이 됩니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/4-figures/4-2.png" alt="Plan-and-Solve 패러다임의 2단계 워크플로우" width="90%"/>
  <p>그림 4.2 Plan-and-Solve 패러다임의 2단계 워크플로우</p>
</div>

Plan-and-Solve는 명확하게 해체할 수 있는 강력한 구조를 가진 다음과 같은 복잡한 작업에 적합합니다.

- **여러 단계의 수학 문장제 문제**: 계산 단계를 먼저 나열한 다음 차례로 풀어야 할 때.
- **여러 정보원을 통합한 보고서 작성**: 보고서 구조(서론, 데이터 소스 A, 데이터 소스 B, 결론)를 먼저 계획하고 차례로 내용을 채워야 할 때.
- **코드 생성 작업**: 함수, 클래스, 모듈의 구조를 먼저 설계한 다음 차례로 구현해야 할 때.

### 4.3.2 계획 단계 (Planning Phase)

구조화된 추론 작업에서 Plan-and-Solve 패러다임의 장점을 돋보이게 하기 위해, 여기서는 외부 도구를 사용하지 않고 순수 프롬프트 설계를 통해 추론 작업을 해결해 보겠습니다.

이런 유형의 작업 특징은 단 한 번의 질의나 계산만으로는 답을 얻을 수 없고, 문제를 먼저 논리적으로 일관된 하위 단계들로 분해한 다음 순서대로 풀어야 한다는 것입니다.
이는 "먼저 계획하고 나중에 실행한다"는 Plan-and-Solve의 핵심 역량을 발휘하기에 완벽한 무대입니다.

**해결할 대상 문제는 다음과 같습니다:** "한 과일 가게에서 월요일에 사과 15개를 팔았습니다.
화요일에 판 사과의 수는 월요일의 두 배였습니다.
수요일에 판 사과의 수는 화요일보다 5개 적었습니다. 사흘 동안 판매된 사과는 모두 몇 개입니까?"

이 문제는 대형 언어 모델에게 아주 어렵지는 않지만, 명확한 논리 사슬을 담고 있습니다. 실제 논리 퍼즐 문제에서 대형 모델이 정확한 답변을 일관되게 출력하지 못할 때 이 디자인 패턴을 참고하여 나만의 에이전트를 구축해 볼 수 있습니다.
에이전트는 다음과 같이 행동해야 합니다.

1. **계획 단계**: 먼저 문제를 세 가지 독립적인 계산 단계로 분해합니다 (화요일 판매량 계산, 수요일 판매량 계산, 총 판매량 계산).
2. **실행 단계**: 계획을 엄격히 따라 단계별로 계산을 실행하고 각 단계의 결과를 다음 단계의 입력으로 사용하여 최종 합계를 도출합니다.

계획 단계의 목표는 대규모 언어 모델이 원본 문제를 전달받아 명확하고 단계적인 행동 계획을 출력하는 것입니다. 이 계획은 우리의 코드가 쉽게 파싱하고 하나씩 실행할 수 있도록 구조화되어야 합니다.
따라서 우리가 설계하는 프롬프트는 모델의 역할과 임무를 명확히 제시하고 출력 형식의 예시를 제공해야 합니다.

```python
PLANNER_PROMPT_TEMPLATE = """
당신은 최고의 AI 계획 전문가입니다. 당신의 임무는 사용자가 제기한 복잡한 문제를 여러 개의 단순한 단계로 구성된 행동 계획으로 분해하는 것입니다.
계획의 각 단계가 독립적이고 실행 가능한 하위 작업인지 확인하고 논리적 순서에 따라 엄격하게 배치해 주세요.
당신의 출력은 반드시 하위 작업을 설명하는 문자열들로 이루어진 Python 리스트 형식이어야 합니다.

Question: {question}

반드시 다음 형식에 맞춰 엄격하게 계획을 출력해 주세요. 접두사와 접미사로 ```python과 ```가 반드시 포함되어야 합니다:
```python
["단계 1", "단계 2", "단계 3", ...]
```
"""
```

이 프롬프트는 다음과 같은 핵심 사항을 통해 출력의 품질과 안정성을 보장합니다.
- **역할 설정**: "최고의 AI 계획 전문가"를 통해 모델의 전문적인 잠재력을 깨웁니다.
- **작업 설명**: "문제 분해"라는 목표를 명확히 정의합니다.
- **형식 제약**: 출력을 Python 리스트 형식의 문자열로 강제함으로써 후속 코드 파싱 작업을 단순화하고, 자연어 파싱보다 훨씬 안정적이고 신뢰할 수 있게 만듭니다.

다음으로 이 프롬프트 로직을 기획자 역할을 하는 `Planner` 클래스로 캡슐화합니다.

```python
import ast

class Planner:
    def __init__(self, llm_client):
        self.llm_client = llm_client

    def plan(self, question: str) -> list[str]:
        """
        사용자 질문을 바탕으로 행동 계획을 생성합니다.
        """
        prompt = PLANNER_PROMPT_TEMPLATE.format(question=question)

        # 계획 생성을 위해 간단한 메시지 목록 구축
        messages = [{"role": "user", "content": prompt}]

        print("--- 계획 생성 중 ---")
        # 스트리밍 출력을 통해 전체 계획을 얻어옴
        response_text = self.llm_client.think(messages=messages) or ""

        print(f"✅ 계획 생성 완료:\n{response_text}")

        # LLM이 출력한 리스트 문자열 파싱
        try:
            # ```python과 ``` 사이의 내용 찾기
            plan_str = response_text.split("```python")[1].split("```")[0].strip()
            # ast.literal_eval을 사용하여 문자열을 안전하게 실행하고 Python 리스트로 변환
            plan = ast.literal_eval(plan_str)
            return plan if isinstance(plan, list) else []
        except (ValueError, SyntaxError, IndexError) as e:
            print(f"❌ 계획 파싱 오류: {e}")
            print(f"원시 응답: {response_text}")
            return []
        except Exception as e:
            print(f"❌ 계획 파싱 중 알 수 없는 오류 발생: {e}")
            return []
```

### 4.3.3 실행기 및 상태 관리 (Executor and State Management)

기획자(`Planner`)가 명확한 행동 설계도를 그린 후에는 실행기(`Executor`)가 계획에 포함된 작업들을 하나씩 완료해 나가야 합니다. 실행기는 각 하위 문제를 해결하기 위해 대규모 언어 모델을 호출할 뿐만 아니라,
**상태 관리(State Management)**라는 매우 중요한 역할을 수행합니다. 각 단계의 실행 결과를 기록하여 후속 단계에 컨텍스트로 제공함으로써 전체 작업 체인 내에서 정보가 원활하게 흐르도록 보장해야 합니다.

실행기의 프롬프트는 기획자의 프롬프트와 다릅니다. 목표는 문제의 분해가 아니라, **기존 컨텍스트를 바탕으로 현재 단계를 해결하는 데 집중하는 것**입니다. 따라서 프롬프트에는 다음과 같은 핵심 정보가 포함되어야 합니다.

- **Original Question (원본 질문)**: 모델이 언제나 최종 목표를 이해하고 있도록 합니다.
- **Complete Plan (전체 계획)**: 모델이 전체 작업에서 현재 단계의 위치를 파악하게 합니다.
- **Historical Steps and Results (이전 단계 및 결과)**: 현재 단계의 직접적인 입력 정보로 쓰이도록 지금까지 완료된 작업들을 제공합니다.
- **Current Step (현재 단계)**: 지금 해결해야 할 특정 작업이 무엇인지 모델에게 명확히 알려줍니다.

```python
EXECUTOR_PROMPT_TEMPLATE = """
당신은 최고의 AI 실행 전문가입니다. 당신의 임무는 제공된 계획을 엄격하게 준수하여 문제를 단계별로 해결하는 것입니다.
당신은 원본 질문, 전체 계획, 그리고 지금까지 완료된 단계와 결과를 제공받게 됩니다.
"현재 단계"를 해결하는 데만 집중해 주시고, 다른 추가 설명이나 대화 없이 해당 단계의 최종 결과값만 출력해 주세요.

# 원본 질문:
{question}

# 전체 계획:
{plan}

# 이전 단계 및 결과:
{history}

# 현재 해결해야 할 단계:
{current_step}

추가 사족이나 설명 없이 현재 단계에 대한 정확한 답변 혹은 최종 값만 출력해 주세요.
"""
```

이 프롬프트의 강점은 모델이 "현재 단계"에 집중하도록 유지하는 동시에 이전 단계들의 결과를 컨텍스트 정보로 활용할 수 있도록 보장한다는 점입니다.
"추가 설명이나 대화 없이 최종 결과만 출력"하도록 제약하여 결과가 깔끔하게 유지되고 다음 단계에 전달하기 좋은 형태가 됩니다.

다음으로 상태를 추적하고 각 단계를 제어하는 `Executor` 클래스를 구현해 보겠습니다.

```python
class Executor:
    def __init__(self, llm_client):
        self.llm_client = llm_client

    def execute(self, question: str, plan: list[str]) -> str:
        """
        계획의 각 단계를 차례대로 실행하고 상태를 추적합니다.
        """
        history = []
        
        for i, step in enumerate(plan):
            print(f"\n-> 단계 {i+1}/{len(plan)} 실행 중: {step}")
            
            # 포맷팅용 이력 문자열 생성
            history_str = "\n".join(history) if history else "아직 완료된 단계가 없습니다."
            
            # 실행기 프롬프트 구축
            prompt = EXECUTOR_PROMPT_TEMPLATE.format(
                question=question,
                plan=str(plan),
                history=history_str,
                current_step=step
            )
            
            messages = [{"role": "user", "content": prompt}]
            
            # LLM을 호출하여 현재 단계를 실행
            step_result = self.llm_client.think(messages=messages) or ""
            step_result = step_result.strip()
            
            print(f"✅ 단계 {i+1} 완료, 결과: {step_result}")
            
            # 현재 단계와 결과를 이력에 기록
            history.append(f"단계: {step}\n결과: {step_result}")
            
        # 마지막 단계의 결과가 전체 질문에 대한 최종 답변이 됩니다.
        return step_result if history else ""
```

마지막으로 이 모든 조각들을 `PlanAndSolveAgent` 클래스로 통합합니다.

```python
class PlanAndSolveAgent:
    def __init__(self, llm_client):
        self.planner = Planner(llm_client)
        self.executor = Executor(llm_client)

    def run(self, question: str) -> str:
        print(f"\n--- 질문 처리 시작 ---\n질문: {question}")
        
        # 1단계: 계획 수립
        plan = self.planner.plan(question)
        if not plan:
            print("오류: 유효한 계획을 생성하지 못했습니다.")
            return ""
            
        # 2단계: 계획 실행
        final_answer = self.executor.execute(question, plan)
        
        print(f"\n--- 작업 완료 ---\n최종 답변: {final_answer}")
        return final_answer
```

이 `PlanAndSolveAgent` 클래스의 설계는 "상속보다 합성(composition over inheritance)" 원칙을 보여줍니다. 에이전트 자체는 복잡한 로직을 담지 않고 오케스트레이터 역할을 하며, 내부의 기획자와 실행기 컴포넌트를 명확하게 호출해 작업을 완수합니다.

### 4.3.4 실행 인스턴스 및 분석

전체 코드는 동반 코드 저장소의 `code` 폴더에서 확인할 수 있으며, 여기서는 최종 결과 로그만 보여드립니다.

```bash
--- 질문 처리 시작 ---
질문: 한 과일 가게에서 월요일에 사과 15개를 팔았습니다. 화요일에 판 사과의 수는 월요일의 두 배였습니다. 수요일에 판 사과의 수는 화요일보다 5개 적었습니다. 사흘 동안 판매된 사과는 모두 몇 개입니까?
--- 계획 생성 중 ---
🧠 xxxx 모델 호출 중...
✅ 대규모 언어 모델 응답 성공:
```python
["월요일 사과 판매량 확인: 15", "화요일 사과 판매량 계산: 월요일 수량 × 2 = 15 × 2 = 30", "수요일 사과 판매량 계산: 화요일 수량 - 5 = 30 - 5 = 25", "사흘간 총 판매량 계산: 월요일 + 화요일 + 수요일 = 15 + 30 + 25 = 70"]
```
✅ 계획 생성 완료:
```python
["월요일 사과 판매량 확인: 15", "화요일 사과 판매량 계산: 월요일 수량 × 2 = 15 × 2 = 30", "수요일 사과 판매량 계산: 화요일 수량 - 5 = 30 - 5 = 25", "사흘간 총 판매량 계산: 월요일 + 화요일 + 수요일 = 15 + 30 + 25 = 70"]
```

--- 계획 실행 중 ---

-> 단계 1/4 실행 중: 월요일 사과 판매량 확인: 15
🧠 xxxx 모델 호출 중...
✅ 대규모 언어 모델 응답 성공:
15
✅ 단계 1 완료, 결과: 15

-> 단계 2/4 실행 중: 화요일 사과 판매량 계산: 월요일 수량 × 2 = 15 × 2 = 30
🧠 xxxx 모델 호출 중...
✅ 대규모 언어 모델 응답 성공:
30
✅ 단계 2 완료, 결과: 30

-> 단계 3/4 실행 중: 수요일 사과 판매량 계산: 화요일 수량 - 5 = 30 - 5 = 25
🧠 xxxx 모델 호출 중...
✅ 대규모 언어 모델 응답 성공:
25
✅ 단계 3 완료, 결과: 25

-> 단계 4/4 실행 중: 사흘간 총 판매량 계산: 월요일 + 화요일 + 수요일 = 15 + 30 + 25 = 70
🧠 xxxx 모델 호출 중...
✅ 대규모 언어 모델 응답 성공:
70
✅ 단계 4 완료, 결과: 70

--- 작업 완료 ---
최종 답변: 70
```

위 출력 로그를 통해 Plan-and-Solve 패러다임의 워크플로우를 명확히 알 수 있습니다.

1. **계획 단계**: 에이전트는 먼저 `Planner`를 호출하여 복잡한 문장제 문제를 4개의 논리적 단계가 담긴 Python 리스트로 해체하는 데 성공합니다. 이 구조화된 계획이 후속 실행의 밑바탕이 됩니다.
2. **실행 단계**: `Executor`는 생성된 계획에 따라 단계별로 엄격하게 실행을 수행합니다. 매 단계마다 이전 이력 결과를 컨텍스트로 제공받으므로 올바른 정보가 전달됩니다(예: 2단계는 1단계 결과인 "15"를 바르게 활용하고, 3단계는 2단계 결과인 "30"을 활용함).
3. **결과**: 전체 프로세스의 논리가 명확하고 절차가 뚜렷하여 에이전트가 정확한 정답인 "70"에 무사히 도달할 수 있었습니다.

---

## 4.4 Reflection (자가 성찰)

우리가 구현한 ReAct와 Plan-and-Solve 패러다임은 에이전트가 한 번 작업을 완료하고 나면 워크플로우가 종료되는 방식입니다.
그러나 행동 궤적이든 최종 결과물이든 처음 생성해낸 초안 답변에는 오류가 섞여 있거나 더 개선할 여지가 있을 수 있습니다. 성찰(Reflection) 메커니즘의 핵심 아이디어는 에이전트가 스스로 일한 결과를 되돌아보고,
결점을 보완하며, 마치 인간처럼 반복적으로 개선할 수 있는 **사후 자가 수정 루프**를 도입하는 것입니다.

### 4.4.1 성찰 메커니즘의 핵심 개념

성찰 메커니즘은 초안을 다 쓰고 퇴고를 거치거나 수학 문제를 푼 다음 오답 노트를 작성하는 인간의 학습 과정에서 영감을 얻었습니다. 이 개념은 2023년 Shinn, Noah 등이 제안한 Reflexion 프레임워크를 비롯한 여러 연구에서 정식화되었습니다<sup>[3]</sup>. 핵심 워크플로우는 **실행 -> 성찰 -> 개선 (Execute -> Reflect -> Refine)**이라는 세 단계의 간결한 루프로 정리할 수 있습니다.

1. **실행 (Execution)**: 에이전트는 익숙한 방식(예: ReAct 또는 Plan-and-Solve)으로 작업을 완수하려 시도하여 초기 솔루션이나 행동 궤적을 만듭니다. 이를 "초안"으로 볼 수 있습니다.
2. **성찰 (Reflection)**: 다음으로 에이전트는 성찰 단계에 진입합니다. 독립적인 LLM 인스턴스(혹은 특별한 프롬프트로 세팅된 인스턴스)를 불러내어 "코드 리뷰어"나 "검토자"의 역할을 수행하게 합니다. 이 검토자는 첫 단계에서 생성된 "초안"을 다음 측면에서 다각도로 검토합니다.
   - **사실적 오류 (Factual Errors)**: 상식이나 알려진 사실에 위배되는 내용이 있는가?
   - **논리적 결함 (Logical Flaws)**: 추론 과정에 모순이나 일관되지 않은 점이 있는가?
   - **효율성 문제 (Efficiency Issues)**: 더 직관적이고 효율적으로 작업을 완료할 다른 대안이 있는가?
   - **정보 누락 (Missing Information)**: 문제의 핵심 제약 조건이나 특정 측면을 놓치지는 않았는가?
   검토를 마치면 이를 통해 구체적인 문제점과 개선 방안이 담긴 구조화된 **피드백 (Feedback)**을 생성합니다.
3. **개선 (Refinement)**: 마지막으로 에이전트는 "초안"과 "피드백"을 새로운 컨텍스트로 삼아 다시 한번 LLM을 호출하고, 피드백을 기반으로 초안을 수정하여 보완된 "수정본"을 생성하게 합니다.

그림 4.3에서 볼 수 있듯이, 이 루프는 성찰 단계에서 더 이상 문제점이 발견되지 않거나 미리 설정한 최대 반복 횟수에 도달할 때까지 여러 차례 반복될 수 있습니다. 이 최적화 과정을 공식으로 나타내면 다음과 같습니다. $O_i$가 $i$번째 반복에서 생성된 출력물이고($O_0$은 초기 출력), 성찰 모델 $\pi_{\text{reflect}}$가 $O_i$에 대한 피드백 $F_i$를 생성합니다.

$$
F_i = \pi_{\text{reflect}}(\text{Task}, O_i)
$$

이후 개선 모델 $\pi_{\text{refine}}$은 원본 작업, 이전 버전의 출력, 피드백을 한데 엮어 새 버전의 출력 $O_{i+1}$을 만들어 냅니다.

$$
O_{i+1} = \pi_{\text{refine}}(\text{Task}, O_i, F_i)
$$

<div align="center">
<img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/4-figures/4-3.png" alt="Reflection 메커니즘의 실행-성찰-개선 반복 루프" width="70%"/>
<p>그림 4.3 Reflection 메커니즘의 실행-성찰-개선 반복 루프</p>
</div>

이전의 두 패러다임과 비교했을 때 Reflection이 주는 가치는 다음과 같습니다.

- 에이전트 내부의 자가 오류 수정 루프를 제공하므로, 에이전트가 외부 도구 피드백(ReAct의 Observation)에만 전적으로 의존하지 않고 높은 수준의 논리적이고 전략적인 오류를 보완할 수 있게 됩니다.
- 1회성 실행에 그치지 않고 지속적인 최적화 과정을 이끌어내어 복잡한 작업의 최종 성공률과 답변 품질을 크게 향상시킵니다.
- 에이전트에게 일시적인 **"단기 기억(short-term memory)"**을 마련해 줍니다. "실행-성찰-개선"의 흐름 전체가 하나의 가치 있는 경험 기록이 되며, 에이전트는 최종 답변만 알게 되는 것이 아니라 결점이 있던 초안에서 최종본으로 어떻게 고쳐 나갔는지에 대한 과정을 함께 기억합니다. 더 나아가, 이 기억 시스템은 **다중 모달(multimodal)**로도 동작하여 텍스트뿐만 아니라 코드, 이미지 등 다양한 형태의 산출물을 검토하고 개선할 수 있게 해 주며 이는 더 강력한 멀티모달 에이전트 구축의 밑거름이 됩니다.

### 4.4.2 실습 설정 및 메모리 모듈 설계

Reflection 메커니즘을 실습에서 구체화하기 위해 메모리 관리 메커니즘을 도입하겠습니다.
성찰은 대개 정보의 저장 및 검색과 밀접하기 때문입니다.
컨텍스트가 너무 긴 상황에서 검토자가 모든 날것의 정보를 직접 훑어보며 성찰을 수행하게 하면 불필요한 정보 오버헤드가 발생합니다.
이번 실습 단계에서는 주로 **코드 생성 및 반복적인 코드 최적화**를 완수해 보겠습니다.

**목표 작업은 다음과 같습니다:** "1부터 n 사이의 모든 소수를 찾는 Python 함수를 작성해줘."

이 작업은 다음과 같은 이유로 Reflection 메커니즘을 테스트하기에 아주 좋습니다.

1. **명확한 성능 개선 경로가 존재함**: 모델이 처음에 작성하는 코드는 대개 단순하지만 비효율적인 이중 루프 방식일 가능성이 큽니다.
2. **분명한 지적 사항이 도출됨**: 성찰 과정을 거쳐 "지나치게 높은 시간 복잡도" 또는 "불필요한 중복 연산" 같은 한계를 쉽게 포착할 수 있습니다.
3. **확실한 리팩토링 방향성**: 피드백을 바탕으로 에라토스테네스의 체처럼 더 효율적인 알고리즘 버전으로 코드를 업그레이드할 수 있습니다.

Reflection의 정수는 반복(iteration)에 있고, 반복의 전제 조건은 이전의 시도와 그때 받았던 피드백을 기억하는 능력입니다.
따라서 이 패러다임을 구현하려면 "단기 기억" 모듈이 필수적으로 장착되어야 합니다.
이 메모리 모듈은 "실행-성찰" 루프의 매 이력을 빠짐없이 담아 두는 역할을 맡습니다.

```python
from typing import List, Dict, Any, Optional

class Memory:
    """
    에이전트의 실행 및 성찰 궤적을 저장하는 간단한 단기 기억 모듈입니다.
    """

    def __init__(self):
        """
        모든 기록을 담기 위해 빈 리스트를 초기화합니다.
        """
        self.records: List[Dict[str, Any]] = []

    def add_record(self, record_type: str, content: str):
        """
        기억 장치에 새 기록을 추가합니다.

        Parameters:
        - record_type (str): 기록의 유형 ('execution' 또는 'reflection').
        - content (str): 기록의 실제 내용 (예: 생성된 코드 또는 성찰 피드백).
        """
        record = {"type": record_type, "content": content}
        self.records.append(record)
        print(f"📝 메모리 업데이트됨, '{record_type}' 기록이 추가되었습니다.")

    def get_trajectory(self) -> str:
        """
        프롬프트 구축을 위해 메모리에 저장된 모든 기록을 하나의 연결된 텍스트로 서식화합니다.
        """
        trajectory_parts = []
        for record in self.records:
            if record['type'] == 'execution':
                trajectory_parts.append(f"--- 이전 시도 (코드) ---\n{record['content']}")
            elif record['type'] == 'reflection':
                trajectory_parts.append(f"--- 검토자 피드백 ---\n{record['content']}")

        return "\n\n".join(trajectory_parts)

    def get_last_execution(self) -> Optional[str]:
        """
        가장 최근에 실행되어 생성된 결과물(예: 최신 생성 코드)을 가져옵니다.
        존재하지 않으면 None을 반환합니다.
        """
        for record in reversed(self.records):
            if record['type'] == 'execution':
                return record['content']
        return None
```

이 `Memory` 클래스의 설계는 직관적이며 다음과 같은 구조를 띱니다.

- 리스트 `records`를 사용하여 각 동작과 성찰의 내용을 발생 순서대로 저장합니다.
- `add_record` 메서드는 메모리에 새로운 항목을 기록합니다.
- 핵심은 `get_trajectory` 메서드로, 메모리 궤적을 한 편의 텍스트로 "직렬화"하여 후속 프롬프트에 직접 삽입할 수 있도록 변환해 줌으로써 모델에 완전한 문맥적 흐름을 제공합니다.
- `get_last_execution`은 성찰을 진행할 최신 "초안"을 간편하게 집어 올 수 있게 돕습니다.

### 4.4.3 Reflection 에이전트 코드 구현

`Memory` 모듈을 발판 삼아, 이제 `ReflectionAgent` 클래스의 핵심 뼈대를 세워 보겠습니다. 에이전트의 워크플로우는 앞서 논의한 "실행-성찰-개선" 루프를 중심으로 움직이며,
세밀하게 직조된 프롬프트를 사용해 모델에 단계별 역할을 안내합니다.

(1) 프롬프트 설계

이전 패러다임들과 달리 Reflection 메커니즘은 다각적 역할을 담당하는 여러 개의 프롬프트가 맞물려 돌아갑니다.

1. **초기 실행 프롬프트 (Initial Execution Prompt)**: 에이전트가 맨 처음 문제 해결을 시도할 때 전달하는 프롬프트로, 군더더기 없이 지정된 작업물만 똑 떨어지게 완성하도록 이끕니다.

```bash
INITIAL_PROMPT_TEMPLATE = """
당신은 시니어 Python 개발자입니다. 다음 요구사항에 맞춰 Python 함수를 작성해 주세요.
작성하는 코드에는 완전한 함수 시그니처와 독스트링(docstring)이 반드시 포함되어야 하며 PEP 8 코딩 표준을 준수해야 합니다.

요구사항: {task}

설명이나 대화 텍스트는 덧붙이지 말고 코드 내용만 곧바로 출력해 주세요.
"""
```

2. **성찰 프롬프트 (Reflection Prompt)**: 성찰 메커니즘의 영혼에 해당하는 프롬프트입니다. 모델을 냉철하고 깐깐한 "코드 리뷰어"로 포지셔닝하여 이전 라운드에 생성된 코드를 세세하게 검토하고,
실질적으로 개선 가능한 피드백을 끌어냅니다.

```bash
REFLECT_PROMPT_TEMPLATE = """
당신은 코드 성능에 있어 완벽주의를 고수하는 지독하게 깐깐한 코드 리뷰 전문가이자 수석 알고리즘 엔지니어입니다.
당신이 할 일은 다음 Python 코드를 리뷰하여 <strong>알고리즘 효율성</strong> 측면에서 치명적인 병목 현상을 짚어내는 것입니다.

# 원본 작업:
{task}

# 리뷰할 코드:
```python
{code}
```

이 코드의 시간 복잡도를 철저히 분석하고, 성능을 크게 개선할 수 있는 <strong>알고리즘적으로 훨씬 뛰어난</strong> 다른 대안이 존재할지 고려해 보세요.
만약 개선할 점이 있다면 현재 알고리즘의 결함을 분명하게 지적하고, 구체적이고 실현 가능한 알고리즘 개선 제안(예: 단순 나눗셈 시도 방식 대신 에라토스테네스의 체 적용)을 제시해 주세요.
알고리즘 수준에서 코드가 이미 완벽하게 최적화된 상태일 때만 사족 없이 "no improvement needed"라고 단호하게 대답해 주세요.

피드백 내용만 다른 불필요한 설명 없이 직접 출력해 주세요.
"""
```

3. **개선 프롬프트 (Refinement Prompt)**: 피드백을 전달받은 모델이 이전 코드를 보완하여 새로운 최적화 코드를 구현하도록 돕는 프롬프트입니다.

```bash
REFINE_PROMPT_TEMPLATE = """
당신은 시니어 Python 개발자입니다. 코드 리뷰 전문가의 피드백을 바탕으로 기존에 작성했던 코드를 리팩토링하고 최적화하려 합니다.

# 원본 작업:
{task}

# 이전에 작성했던 코드:
{last_code_attempt}

# 리뷰어 피드백:
{feedback}

리뷰어의 피드백을 반영하여 알고리즘이 크게 최적화된 새로운 버전의 코드를 구현해 주세요.
함수 시그니처와 독스트링을 반드시 포함하고 PEP 8 표준을 지켜야 합니다.
다른 장황한 부가 설명은 생략하고 개선된 코드 블록만 즉시 출력해 주세요.
"""
```

(2) 에이전트 캡슐화 및 구현

이제 프롬프트 조합과 `Memory` 모듈을 하나로 엮어 `ReflectionAgent` 클래스를 최종 구현합니다.

```python
class ReflectionAgent:
    def __init__(self, llm_client, max_iterations=3):
        self.llm_client = llm_client
        self.memory = Memory()
        self.max_iterations = max_iterations

    def run(self, task: str):
        print(f"\n--- 작업 처리 시작 ---\n작업: {task}")

        # --- 1단계: 최초 시도 (Initial Attempt) ---
        print("\n--- 최초 시도 실행 중 ---")
        initial_prompt = INITIAL_PROMPT_TEMPLATE.format(task=task)
        initial_code = self._get_llm_response(initial_prompt)
        self.memory.add_record("execution", initial_code)

        # --- 2단계: 성찰 및 개선 반복 루프 ---
        for i in range(self.max_iterations):
            print(f"\n--- 반복 회차 {i+1}/{self.max_iterations} ---")

            # a. 성찰 (Reflection)
            print("\n-> 성찰을 수행하는 중...")
            last_code = self.memory.get_last_execution()
            reflect_prompt = REFLECT_PROMPT_TEMPLATE.format(task=task, code=last_code)
            feedback = self._get_llm_response(reflect_prompt)
            self.memory.add_record("reflection", feedback)

            # b. 종료 조건 확인
            if "no improvement needed" in feedback.lower():
                print("\n✅ 성찰 결과 코드가 이미 최적이며 추가 개선이 불필요하다고 판단되어 작업을 마칩니다.")
                break

            # c. 개선 (Refinement)
            print("\n-> 개선 작업을 수행하는 중...")
            refine_prompt = REFINE_PROMPT_TEMPLATE.format(
                task=task,
                last_code_attempt=last_code,
                feedback=feedback
            )
            refined_code = self._get_llm_response(refine_prompt)
            self.memory.add_record("execution", refined_code)

        final_code = self.memory.get_last_execution()
        print(f"\n--- 작업 최종 완료 ---\n최종 생성된 코드:\n```python\n{final_code}\n```")
        return final_code

    def _get_llm_response(self, prompt: str) -> str:
        """LLM을 호출하고 스트리밍 응답의 전체 결과를 수집하는 보조 메서드입니다."""
        messages = [{"role": "user", "content": prompt}]
        response_text = self.llm_client.think(messages=messages) or ""
        return response_text
```

### 4.4.4 실행 인스턴스 및 분석

전체 작동 코드는 동반 코드 저장소의 `code` 폴더에서 확인할 수 있으며, 아래는 실행 시 출력되는 결과의 한 예시입니다.

```python
--- 작업 처리 시작 ---
작업: 1부터 n 사이의 모든 소수를 찾는 Python 함수를 작성해줘.

--- 최초 시도 실행 중 ---
🧠 Calling xxxxxx model...
✅ 대규모 언어 모델 응답 성공:
```python
def find_primes(n):
    # (단순 Trial division 방식의 초기 코드 생략)
    ...
    return primes
```
📝 메모리 업데이트됨, 'execution' 기록이 추가되었습니다.

--- 반복 회차 1/2 ---

-> 성찰을 수행하는 중...
🧠 Calling xxxxxx model...
✅ 대규모 언어 모델 응답 성공:
현재 코드는 O(n * sqrt(n))의 시간 복잡도를 가집니다. n이 비교적 작을 때는 괜찮으나, n이 매우 커질 경우 성능이 급격히 저하될 위험이 있습니다. 매 숫자마다 나누어 떨어지는지 일일이 검사하므로 연산 비용이 큽니다.

시간 복잡도를 O(n log(log n))으로 대폭 낮출 수 있는 에라토스테네스의 체(Sieve of Eratosthenes) 알고리즘을 도입하면 소수 검색 연산 효율을 크게 극대화할 수 있을 것입니다.

추천되는 개선 방향 코드는 다음과 같습니다:
```python
def find_primes(n):
    ...
    return primes
```
📝 메모리 업데이트됨, 'reflection' 기록이 추가되었습니다.

-> 개선 작업을 수행하는 중...
🧠 Calling xxxxxx model...
✅ 대규모 언어 모델 응답 성공:
```python
def find_primes(n):
    # (에라토스테네스의 체를 구현한 코드)
    ...
    return primes
```
📝 메모리 업데이트됨, 'execution' 기록이 추가되었습니다.

--- 반복 회차 2/2 ---

-> 성찰을 수행하는 중...
🧠 Calling xxxxxx model...
✅ 대규모 언어 모델 응답 성공:
현재 코드는 시간 복잡도 O(n log log n), 공간 복잡도 O(n)을 갖는 에라토스테네스의 체 알고리즘을 올바르게 적용했습니다. 일반적인 범위 내에서 모든 소수를 구하기에 이미 매우 효율적이며 더 이상의 무리한 개선은 필요하지 않아 보입니다. 다만 극단적인 최적화가 필요한 특수한 경우 아래 방향도 검토해 볼 수 있습니다:

1. Segmented Sieve: n이 매우 거대하여 메모리 한계에 부딪힐 때, 구간을 잘게 쪼개어 체 연산을 돌려 메모리 점유율을 줄이는 기법입니다.
2. 홀수 전용 체 설계: 2를 제외한 모든 소수가 홀수라는 특징을 착안해 배열 메모리를 절반으로 절감하는 방식입니다.

그러나 이러한 극적 튜닝은 대다수 일반 개발 시나리오에서는 불필요하며 현재 구현이 이미 충분히 강건합니다. 따라서 general하게 볼 때 **no improvement needed**입니다.
📝 메모리 업데이트됨, 'reflection' 기록이 추가되었습니다.

✅ 성찰 결과 코드가 이미 최적이며 추가 개선이 불필요하다고 판단되어 작업을 마칩니다.

--- 작업 최종 완료 ---
최종 생성된 코드:
```python
def find_primes(n):
    """
    Finds all prime numbers between 1 and n using the Sieve of Eratosthenes algorithm.

    :param n: The upper limit of the range to find prime numbers.
    :return: A list of all prime numbers between 1 and n.
    """
    if n < 2:
        return []

    is_prime = [True] * (n + 1)
    is_prime[0] = is_prime[1] = False

    p = 2
    while p * p <= n:
        if is_prime[p]:
            for i in range(p * p, n + 1, p):
                is_prime[i] = False
        p += 1

    primes = [num for num in range(2, n + 1) if is_prime[num]]
    return primes
```
```

이 실습 예시는 Reflection 메커니즘이 에이전트를 어떻게 심화된 최적화로 끌고 가는지 그 저력을 훌륭하게 대변합니다.

1. **매서운 검토가 최적화의 첫단추**: 첫 성찰 때 "알고리즘 효율성에 집착하는 엄격한 코드 검토자" 프롬프트를 쥐여주자, 에이전트는 돌아가기만 하는 최초 코드에 만족하지 않고 정확하게 시간 복잡도 `O(n * sqrt(n))` 병목 지점을 짚어 내며 '에라토스테네스의 체'라는 알고리즘적 돌파구를 명확히 주문했습니다.
2. **점진적 개선 도출**: 피드백을 고스란히 안고 들어간 개선 단계에서, 에이전트는 복잡도를 `O(n log log n)`으로 끌어내리는 정교한 체 코드를 안정적으로 빌드하며 첫 번째 유의미한 자기 성장을 완성했습니다.
3. **수렴과 리액션 종료**: 이어진 두 번째 성찰에서 이미 성숙한 최적 코드를 직면한 에이전트는 한 차원 더 높은 아이디어(구간 분할 체, 홀수 압축 등)를 지식 차원에서 언급하는 한편, 실질적으로는 일반적인 비즈니스 요건에서 "더 이상의 고도화는 불필요하다"는 성숙한 수렴 결론을 내렸습니다. 이 성찰 텍스트가 우리의 종료 조건과 딱 맞아떨어지면서 연산 낭비 없이 루프가 아름답게 수렴하여 종료되었습니다.

이 실습은 잘 다듬어진 성찰 메커니즘이 단순 오류를 메우는 것을 넘어 **결과물의 품질과 효율성을 단계적으로 견인해 나가는 중추적 장치**임을 증명하며, 고난도 고품질 에이전트를 구성하는 중요한 핵심 기술이 됨을 알 수 있습니다.

### 4.4.5 Reflection 메커니즘의 손익 분석 (Cost-Benefit Analysis)

Reflection 메커니즘이 결과물의 완성도를 올리는 데 매력적이기는 하나, 공짜로 얻어지는 성능은 아닙니다. 현업 프로젝트에 녹여내기 전에는 이것이 야기하는 추가 자원 비용과 품질상의 이점을 냉정히 저울질해 보아야 합니다.

(1) 주요 자원 비용 (Costs)

1. **모델 호출 오버헤드의 가파른 증가**: 매 이터레이션 회차마다 최소 두 차례(성찰 한 번, 개선 한 번)의 추가 API 콜이 붙습니다. 루프 횟수가 누적될수록 API 사용 요금과 클라우드 추론 연산 인프라 소모량이 수직 상승합니다.
2. **응답 대기 레이턴시의 누적**: 성찰은 철저하게 앞 단계의 출력이 마무리되어야 다음 단계가 돌아가는 직렬(serial) 설계 방식입니다. 사용자가 마냥 기다리기 힘든 즉각적인 반응성(real-time) 중심의 서비스 모델에는 치명적인 걸림돌이 될 수 있습니다.
3. **프롬프트 엔지니어링 리소스**: 실습 코드에서 드러났듯 "최초 빌드", "성찰", "개선" 등 각 포인트마다 기대를 만족하는 출력을 내도록 정밀하게 조율된 여러 장의 프롬프트를 만들고 디버깅하는 일에 설계 공수가 많이 듭니다.

(2) 핵심 가치 이점 (Benefits)

1. **결과물의 완성도 점프**: '작동은 하는' 투박한 코드나 문서 결과물을 '알고리즘적, 논리적 미가 충만한' 명품으로 정제해 나가는 퇴고 효과가 탁월합니다. 미세한 논리 결점이 큰 위험을 부르는 실무 환경에서 이 차이는 매우 결정적입니다.
2. **시스템 견고함 및 예외 탄력성 상승**: 자가 성찰 흐름을 통해 초기 초안에서 모델 본인이 미처 신경 쓰지 못한 경계 조건(edge cases)의 예외 처리나 잠재적인 로직 결함을 미리 쓸어 담음으로써, 최종 도출본의 정합성을 한 단계 더 안전하게 다질 수 있습니다.

따라서 Reflection 메커니즘은 전형적인 **"레이턴시와 비용을 녹여서 압도적인 품질을 사는" 전략**입니다. 실시간 반응 속도에 목매지 않아도 되는 동시에 결과물의 정밀성과 정확성이 생명인 영역에 적극 추천됩니다. 예를 들어:

- 배포 시 크리티컬한 핵심 비즈니스 모듈 코드 생성이나 중요 정책 보고서 초안 최적화
- 학술 및 복잡한 정밀 수학 추론 과정
- 검증이 중요한 대형 의사결정 보조 시스템

반면, 짧고 가볍게 툭 치고 빠지는 대화형 챗봇 서비스나 대강 맞물려 돌아가는 정보만으로도 충분히 목적을 이루는 일상 작업이라면 ReAct 혹은 Plan-and-Solve 선에서 간결히 정리하는 편이 손익 측면에서 훨씬 합리적입니다.

---

## 4.5 장 요약

이번 장에서는 3장에서 다진 대형 언어 모델의 뇌 구조에 대한 이해를 토대로, "직접 바퀴를 만들어 굴려 보는" 구현 코딩을 통해 업계를 대표하는 세 가지 클래식 에이전트 설계 패러다임(ReAct, Plan-and-Solve, Reflection)을 바닥부터 조립해 보았습니다. 각각의 아키텍처 원리를 눈으로 보았을 뿐만 아니라 장단점과 실제 사용 환경도 케이스 스터디로 다각도로 분석했습니다.

**핵심 기술 요약:**

1. **ReAct**: 실시간 상황 정보 검색에 능동적으로 나서는 에이전트를 구축했습니다. "생각-행동-관찰" 시너지 루프를 발판 삼아 내장 지식만으로 해소하기 벅찬 현재의 웹 이슈를 검색 도구를 통해 메워 나갔습니다. **현장 기민성과 예외 자가 수정력**이 무기이며, 외부 지식을 끊임없이 참조해야 하는 탐색 과제에 최적입니다.
2. **Plan-and-Solve**: 큰 뼈대를 먼저 세운 뒤 조립하는 기획형 에이전트를 조립해 다단계 추론과 연산이 결합된 수학 문제를 깔끔하게 해소했습니다. 큰 덩어리의 미션을 직관적인 로드맵으로 쪼개서 하나씩 쳐 나가는 구조를 가집니다. **규격화된 논리와 연산의 안정성**이 우수하여 고정적인 단계별 추론 작업에 안성맞춤입니다.
3. **Reflection (자가 성찰 및 진화)**: 생성 결과물을 되돌아보며 최적 알고리즘으로 단계를 끌어올리는 성찰형 모델을 조립했습니다. "실행-성찰-개선" 피드백 루프는 단조로운 초안 수준의 코드를 O(n log log n) 성능의 에라토스테네스의 체로 이끌었습니다. **지독한 품질 개선 능력**이 강점이며, 정합성과 신뢰성이 절대적으로 필요한 고급 미션에 진가를 드러냅니다.

세 개의 패러다임은 에이전트가 문제에 대응하는 다양한 성격을 보여줍니다 (표 4.1 참고). 실무에서 이들을 활용할 때는 미션의 성격과 요구되는 레이턴시, 인프라 비용 한계를 고려해 가장 알맞은 단일 혹은 하이브리드 전략을 결합해야 합니다.

<div align="center">
<p>표 4.1 다양한 에이전트 루프의 선택 전략</p>
<img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/4-figures/4-4.png" alt="" width="70%"/>
</div>

지금까지 개별적으로 일하는 독립적인 단일 에이전트 설계 핵심을 탄탄히 마스터했습니다. 다음 장부터는 이 이론을 더 폭넓은 필드로 옮겨와 로우코드 에이전트 플랫폼들과 실무 연동형 경량 프레임워크들을 사용해 더 빠르고 효율적으로 에이전트를 조립하는 실습 단계로 나아가 보겠습니다.

---

## 연습 문제

> **주의**: 일부 문제에는 정답이 정해져 있지 않습니다. 학습자가 에이전트 패러다임 설계에 관한 종합적인 이해도와 실무 응용력을 스스로 기르는 데 주안점을 둡니다.

1. 이번 장에서는 세 가지 클래식 에이전트 패러다임인 `ReAct`, `Plan-and-Solve`, `Reflection`을 구현했습니다. 다음 사항을 고찰해 보세요.
   - 이 세 패러다임이 "생각"과 "행동"의 관계를 설계하고 구조화하는 본질적인 차이는 무엇입니까?
   - "스마트홈 제어 비서"(조명, 에어컨, 커튼 등을 제어하고 가계의 일상 패턴을 자율 학습해 자동 반영함)를 기획한다면, 어떤 패러다임을 메인 뼈대로 설정하겠습니까? 그 합당한 사유는 무엇입니까?
   - 이 세 패러다임을 하이브리드 형태로 뒤섞을 수도 있을까요? 그렇다면 어떤 혼합형 구조로 설계할 수 있으며, 이러한 혼합 구조가 빛을 발하는 비즈니스 시나리오는 무엇입니까?

2. 4.2절의 `ReAct` 에이전트 코드를 보면 정규식을 써서 LLM의 생각(`Thought`)과 행동(`Action`) 구문을 잘라냈습니다. 다음 사항을 점검해 보세요.
   - 현재처럼 정규식 기반으로 텍스트를 갈무리하는 방법이 내포하는 취약점은 무엇이며 어떤 응답 상황에서 오작동을 유발합니까?
   - 정규식 파싱 방식 외에 상용 프로젝트에서 훨씬 더 안정적으로 모델의 출력을 포맷팅하고 뽑아낼 수 있는 검증된 파싱 기법에는 무엇이 있습니까?
   - 이번 장의 코드를 직접 고쳐서 훨씬 안정적인 정형 출력 파이프라인을 부착해 보고, 기존 방식과의 장단점을 실험적으로 비교해 보세요.

3. 도구(Tools)의 호출은 에이전트의 존재 이유 중 하나입니다. 4.2.2항의 `ToolExecutor` 설계를 밑바탕으로 다음 심화 구현 문제를 수행해 보세요.
   > **주의**: 이 문항은 손으로 직접 구현해 보는 실습 문제입니다. 되도록 Python 코드를 실제로 편집해 확인하기를 권장합니다.
   - `ReAct` 에이전트의 도구함에 사칙 연산 및 복합 괄호 계산이 지원되는 "계산기" 도구를 탑재해 보세요 (예: `(123 + 456) * 789 / 12` 와 같은 수식의 연산 대행).
   - "도구 선택 실패 예외 처리(Error Handling)" 메커니즘을 구상하여 탑재해 보세요. 에이전트가 존재하지 않는 도구명을 부르거나, 인자 규격에 맞지 않는 파라미터를 넣어 에러를 반복할 때 스스로 어떻게 가이드라인을 타고 복원되도록 처리하시겠습니까?
   - 만약 에이전트가 쥐고 있는 도구의 개수가 50개, 100개 이상으로 비대해진다면 기존의 도구 설명 주입 방식이 정상 동작할까요? 서비스가 커지면서 다량의 도구가 추가되는 환경에서 에이전트의 도구 검색(Retrieval)과 조직 체계를 엔지니어링 관점에서 어떻게 고도화할 수 있습니까?

4. `Plan-and-Solve` 패러다임은 작업을 "계획"과 "실행"으로 쪼갭니다. 다음 질문에 대해 깊이 있게 분석해 보세요.
   - 4.3절의 구현에서 수립된 기획 설계도는 "정적(static)"입니다 (최초 1회 생성 뒤 수정 불가). 만약 실행을 돌리던 중 특정 하위 태스크가 물리적인 서버 다운이나 예기치 않은 오류로 좌초되었을 때, "동적 재계획(Dynamic Replanning)"을 태우려면 아키텍처를 어떻게 수정해야 합니까?
   - `Plan-and-Solve`와 `ReAct`를 일대일로 맞비교해 봅시다. "서울발 상하이행 출장 일정 일괄 예약 (항공편 수배, 비즈니스 호텔 낙점, 렌터카 비교 선택 등 포함)" 비즈니스 업무를 자동화할 때 어떤 구조가 더 뛰어난 성능을 냅니까? 그 사유는 무엇입니까?
   - "계층형 계획(Hierarchical Planning)" 구조를 구상해 보세요. 가장 상위 단계의 추상적인 가이드 마일스톤을 먼저 뽑은 뒤, 각 단계마다 상세한 세부 행동 목록을 하위로 쪼개어 들어가는 2단계 형태입니다. 이러한 구조가 복잡한 일처리에 주는 메리트는 무엇입니까?

5. `Reflection` 메커니즘은 "실행-성찰-개선" 피드백 루프를 타고 품질을 높입니다. 다음을 따져 보세요.
   - 4.4절의 소수 필터링 실습에서는 초안 빌더와 검토 전문가를 동일 모델로 설정했습니다. 이를 분리하여 실행은 가볍고 기민한 소형 모델에 맡기고, 성찰/검토만 크고 지능이 높은 초거대 플래그십 모델에 전담시키는 구조(Heterogeneous Model Layout)로 틀 경우 어떤 장단점이 유발되겠습니까?
   - 현재 성찰 코드의 종료 조건은 피드백 텍스트에 "no improvement needed" 패턴이 발견되거나 최대 횟수에 다다르는 형태입니다. 이 설계가 가진 약점은 무엇이며 훨씬 더 스마트하고 논리적인 루프 탈출 조건은 어떻게 고안할 수 있습니까?
   - "학술 논문 초고 집필 및 최적화 도우미"를 만든다고 가정해 봅시다. 문맥 흐름, 방법론의 논리적 독창성, 문장의 학술적 톤앤매너, 인용 문헌 포맷 등 다채로운 측면에서 논문을 검증하고 빌드업하는 "다차원 성찰(Multi-dimensional Reflection)" 모듈을 구상하고 설계도를 작성해 보세요.

6. 프롬프트 엔지니어링은 에이전트의 완성도를 매듭짓는 마침표입니다. 이 장에서 등장한 여러 개의 핵심 프롬프트 템플릿을 분석해 보세요.
   - 4.2.3항의 `ReAct` 프롬프트와 4.3.2항의 `Plan-and-Solve` 기획 프롬프트를 나란히 대조해 보면 구조 설계 측면에서 아주 뚜렷한 대비가 드러납니다. 이러한 각 템플릿 구조의 차이가 본인 패러다임의 작동 철학을 어떻게 뒷받침하고 있습니까?
   - 4.4.3항의 `Reflection` 성찰 프롬프트 안에는 "당신은 극도로 엄격하고 알고리즘 완성도에 목을 매는 전문가입니다"라는 페르소나를 탑재했습니다. 이 역할 지침을 "당신은 팀원의 가독성과 깔끔한 변수명 작성 방식을 지극히 중시하는 오픈소스 메인테이너입니다" 등으로 튜닝했을 때, 실제로 생성 및 개량되는 결과물 코드에 어떤 영향과 피드백의 성향 차이가 발생하는지 경험적으로 테스트하고 결과를 분석해 보세요.
   - 프롬프트에 `few-shot` 예시들을 삽입하는 기법은 출력 형식 엄수에 엄청난 버프를 줍니다. 본 장의 에이전트 프롬프트 중 하나를 골라 고품질 `few-shot` 데이터를 적용하고 개선율을 비교해 보세요.

7. 한 전자상거래 스타트업이 인건비 절감과 고객 반응성 개선을 목표로 사람을 대체할 "CS 전담 상담 에이전트"를 런칭하려고 합니다. 요구되는 스펙은 다음과 같습니다:
   a. 고객의 반품/환불 사유 유형을 정확히 독해함
   b. 고객의 주문 내역 및 택배사의 실시간 물류 이송 상태를 동적으로 확인해 올 수 있음
   c. 회사 내부의 약관 및 환불 정책 DB를 대조하여 환불 가능 여부를 합리적으로 승인함
   d. 상황에 맞는 공손한 어조의 메일 답장을 포맷팅하여 이메일 API를 통해 전송함
   e. 본인이 판단을 내리기에 정책적 충돌이 있어 확신도(Confidence Score)가 기준선 아래로 떨어질 경우, 스스로 성찰 단계를 가동해 대안을 제시하거나 상담원 배정을 결정함
   
   이 제품의 총괄 기획 팀장으로서:
   - 이번 장에서 학습한 세 아키텍처 중 어떤 패러다임(혹은 어떤 결합형 구조)을 시스템 엔진으로 발탁하겠습니까?
   - 에이전트의 손발이 되어 줄 도구함에 어떤 것들이 필요합니까? 용도가 정의된 도구명과 설명을 최소 3개 정의해 보세요.
   - 에이전트가 고객에게 친절함을 잃지 않으면서도 회사의 정책 규정을 엄격히 수호하도록 유도하는 프롬프트 조율 장치를 어떻게 설계하겠습니까?
   - 정식 런칭 후 현업 CS 시장에서 에이전트가 저지를 수 있는 주요 위험과 돌발 행동은 무엇이며 이를 기술적으로 방어할 백업 안전 솔루션은 어떻게 고안할 수 있습니까?

## 참고 문헌

[1] Yao S, Zhao J, Yu D, et al. React: Synergizing reasoning and acting in language models[C]//International Conference on Learning Representations (ICLR). 2023.

[2] Wang L, Xu W, Lan Y, et al. Plan-and-solve prompting: Improving zero-shot chain-of-thought reasoning by large language models[J]. arXiv preprint arXiv:2305.04091, 2023.

[3] Shinn N, Cassano F, Gopinath A, et al. Reflexion: Language agents with verbal reinforcement learning[J]. Advances in Neural Information Processing Systems, 2023, 36: 8634-8652.

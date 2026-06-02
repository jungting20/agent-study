# 제9장 컨텍스트 엔지니어링

이전 장들에서 우리는 에이전트를 위한 메모리 시스템과 RAG를 소개했습니다. 그러나 에이전트가 실제 복잡한 시나리오에서 안정적으로 "생각"하고 "행동"하도록 하려면 메모리와 검색만으로는 부족합니다. 모델에 적절한 "컨텍스트(context)"를 지속적이고 체계적으로 구성하기 위한 엔지니어링 방법론이 필요합니다. 이것이 바로 이번 장의 주제인 컨텍스트 엔지니어링(Context Engineering)입니다. 컨텍스트 엔지니어링은 "각 모델 호출 전에 재사용 가능하고, 측정 가능하며, 발전 가능한 방식으로 입력 컨텍스트를 조립하고 최적화하는 방법"에 초점을 맞추어 정확성, 견고성, 효율성을 향상시킵니다<sup>[1][2]</sup>.

독자들이 이 장의 전체 기능을 빠르게 경험할 수 있도록, 직접 설치 가능한 Python 패키지를 제공합니다. 다음 명령을 사용하여 이 장에 해당하는 버전을 설치할 수 있습니다:

```bash
pip install "hello-agents[all]==0.2.8"
```

이 장에서는 주로 컨텍스트 엔지니어링의 핵심 개념과 실무를 소개하며, HelloAgents 프레임워크에 컨텍스트 빌더와 두 가지 지원 도구를 추가합니다:

- **ContextBuilder** (`hello_agents/context/builder.py`): GSSC (Gather-Select-Structure-Compress) 파이프라인을 구현하여 통일된 컨텍스트 관리 인터페이스를 제공하는 컨텍스트 빌더
- **NoteTool** (`hello_agents/tools/builtin/note_tool.py`): 에이전트를 위한 영구 메모리 관리를 지원하는 구조화된 노트 도구
- **TerminalTool** (`hello_agents/tools/builtin/terminal_tool.py`): 파일 시스템 작업 및 에이전트를 위한 적시(just-in-time) 컨텍스트 검색을 지원하는 터미널 도구

이러한 컴포넌트들은 함께 완전한 컨텍스트 엔지니어링 솔루션을 구성하며, 이는 장기 작업 관리 및 에이전트 기반 검색(agentic search)을 구현하는 핵심 요소입니다. 이에 대해서는 후속 섹션에서 자세히 소개합니다.

프레임워크 설치 외에도 `.env` 파일에 LLM API를 설정해야 합니다. 이 장의 예제들은 주로 컨텍스트 관리와 지능형 의사결정을 위해 대규모 언어 모델(LLM)을 사용합니다.

설정이 완료되면 이 장의 학습 여정을 시작할 수 있습니다!

## 9.1 컨텍스트 엔지니어링이란 무엇인가

수년간 프롬프트 엔지니어링(Prompt Engineering)이 응용 AI의 초점이 된 후, 새로운 용어가 전면에 등장했습니다.
바로 **컨텍스트 엔지니어링(Context Engineering)**입니다.
오늘날 언어 모델로 시스템을 구축하는 것은 단순히 프롬프트에서 올바른 문구와 단어 선택을 찾는 것을 넘어, 더 거시적인 질문에 답하는 것입니다.
즉, **"어떤 종류의 컨텍스트 구성이 모델로 하여금 우리가 기대하는 행동을 생성할 가능성을 가장 높이는가?"**라는 질문입니다.

이른바 "컨텍스트"란 대규모 언어 모델(LLM)을 샘플링할 때 포함되는 토큰의 집합을 가리킵니다. 직면한 엔지니어링 과제는 LLM의 고유한 제약 조건 하에서 **이러한 토큰의 효용을 최적화**하여 기대하는 결과를 안정적으로 얻는 것입니다. LLM을 효과적으로 활용하기 위해서는 종종 "컨텍스트 안에서 생각하기"가 필요합니다. 즉, 호출할 때마다 LLM이 볼 수 있는 전반적인 상태를 검토하고 이 상태가 유도할 수 있는 행동을 예측하는 것입니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/9-figures/9-1.webp" alt="" width="85%"/>
  <p>그림 9.1 프롬프트 엔지니어링 vs 컨텍스트 엔지니어링</p>
</div>

이 섹션에서는 새롭게 떠오르는 컨텍스트 엔지니어링을 탐구하고, **통제 가능하고 효과적인** 에이전트를 구축하기 위한 정제된 멘탈 모델을 제공할 것입니다.

**컨텍스트 엔지니어링 vs. 프롬프트 엔지니어링**

그림 9.1에서 볼 수 있듯이, 주요 모델 제공업체들의 관점에서 컨텍스트 엔지니어링은 프롬프트 엔지니어링의 자연스러운 진화입니다. 프롬프트 엔지니어링은 더 나은 결과를 얻기 위해 LLM 지침을 작성하고 구성하는 방법(예: 시스템 프롬프트 작성 및 구조화 전략)에 초점을 맞추는 반면, 컨텍스트 엔지니어링은 **추론 단계에서 "최적의 정보 집합(토큰)"을 계획하고 유지하는 방법**을 다룹니다. 여기에는 프롬프트 자체뿐만 아니라 컨텍스트 창에 들어가는 다른 모든 정보가 포함됩니다.

LLM 엔지니어링의 초기 단계에서는 프롬프트가 주된 작업인 경우가 많았습니다. 일상적인 대화를 제외한 대부분의 사용 사례가 단일 턴 분류 또는 텍스트 생성을 위해 미세 조정된 프롬프트 최적화를 요구했기 때문입니다. 이름에서 알 수 있듯이 프롬프트 엔지니어링의 핵심은 "효과적인 프롬프트(특히 시스템 프롬프트)를 작성하는 방법"입니다. 그러나 더 긴 시간 범위에서 작동하고 여러 추론 라운드에 걸쳐 동작하는 더 강력한 에이전트를 엔지니어링하기 시작하면서, 시스템 지침, 도구, MCP (Model Context Protocol), 외부 데이터, 메시지 기록 등을 포함한 **전체 컨텍스트 상태**를 관리할 수 있는 전략이 필요해졌습니다.

루프에서 실행되는 에이전트는 다음 추론 라운드와 관련이 있을 수 있는 데이터를 지속적으로 생성합니다. 이 정보는 **주기적으로 정제**되어야 합니다. 따라서 컨텍스트 엔지니어링의 "예술과 기술"은 지속적으로 확장되는 "후보 정보 우주"로부터 **제한된 컨텍스트 창에 어떤 콘텐츠가 들어가야 하는지 식별**하는 데 있습니다.

## 9.2 컨텍스트 엔지니어링이 중요한 이유

모델의 속도가 점점 빨라지고 더 큰 규모의 데이터를 처리할 수 있게 되었지만, 우리는 인간과 마찬가지로 LLM도 특정 지점에서 "헤매거나" "혼란스러워하는" 현상을 관찰할 수 있습니다. 건더미 속 바늘 찾기(Needle-in-a-haystack) 벤치마크는 **컨텍스트 부패(context rot)** 현상을 보여줍니다. 즉, 컨텍스트 창 안의 토큰 수가 늘어남에 따라 컨텍스트로부터 정보를 정확히 리콜(recall)하는 모델의 능력이 실제로는 감소한다는 것입니다.

모델마다 기능 저하 곡선이 더 완만할 수 있지만, 이러한 특성은 거의 모든 모델에서 나타납니다. 따라서 **컨텍스트는 한계 수확 체감 법칙이 적용되는 제한된 자원으로 보아야 합니다**. 인간이 제한된 작업 기억 용량을 갖는 것처럼 LLM도 "어텐션 예산(attention budget)"을 갖습니다. 새로운 토큰이 추가될 때마다 이 예산의 일부를 소비하므로, LLM에 어떤 토큰을 제공해야 할지 훨씬 더 주의를 기울여야 합니다.

이러한 희소성은 우연한 것이 아니라 LLM의 아키텍처적 제약에서 비롯됩니다. 트랜스포머(Transformer)는 각 토큰이 컨텍스트 내의 **모든** 토큰과 연관 관계를 맺을 수 있도록 하여, 이론적으로 \(n^2\)개의 쌍별 어텐션(pairwise attention) 관계를 형성합니다. 컨텍스트 길이가 길어짐에 따라 이러한 쌍별 관계를 모델링하는 모델의 능력은 "한계에 도달"하여, 자연스럽게 "컨텍스트 스케일"과 "어텐션 집중도" 사이에 긴장감이 조성됩니다. 또한 모델의 어텐션 패턴은 훈련 데이터 분포에서 나옵니다. 짧은 시퀀스가 긴 시퀀스보다 더 일반적이므로, 모델은 "전체 컨텍스트 의존성"에 대한 경험이 부족하고 관련 전문 파라미터도 적습니다.

위치 인코딩 보간(position encoding interpolation) 같은 기술을 사용하면 추론 시 모델이 훈련 때보다 더 긴 시퀀스에 "적응"할 수 있지만, 이는 토큰 위치를 이해하는 정확도를 일부 희생하는 대가를 치릅니다. 전반적으로 이러한 요인들이 함께 작용하여 "절벽 형태"의 붕괴가 아닌 **성능 경사(performance gradient)**를 형성합니다. 즉, 긴 컨텍스트에서도 모델은 여전히 강력하지만 짧은 컨텍스트와 비교할 때 정보 검색 및 장거리 추론의 정밀도는 떨어집니다.

위와 같은 현실에 기반해 볼 때, **의식적인 컨텍스트 엔지니어링**은 탄탄한 에이전트를 구축하는 데 필수적입니다.

### 9.2.1 효과적인 컨텍스트의 "구조 분석"

"제한된 어텐션 예산"이라는 제약 조건 속에서 훌륭한 컨텍스트 엔지니어링의 목표는 **최대한 적으면서도 신호 밀도가 높은 토큰을 사용하여 예상된 결과를 얻을 확률을 극대화하는 것**입니다. 실무적으로 우리는 다음과 같은 컴포넌트들을 중심으로 엔지니어링할 것을 권장합니다:

- **시스템 프롬프트(System Prompt)**: 정보 계층 구조가 "적절한" 높이를 가진 명확하고 직관적인 언어입니다. 흔한 실수로는 다음과 같은 두 가지 극단이 있습니다:
  - 과도한 하드코딩(Over-hardcoding): 프롬프트에 복잡하고 깨지기 쉬운 if-else 로직을 작성하는 것으로, 장기적인 유지보수 비용이 높고 취약합니다.
  - 지나치게 모호함(Too vague): 거시적 목표와 일반적인 안내만 제공하고, 예상 출력에 대한 **구체적인 신호**가 결여되어 있거나 잘못된 "공유된 컨텍스트"를 가정하는 것입니다.
  프롬프트를 XML/Markdown으로 구분된 여러 섹션(예: `<background_information>`, `<instructions>`, 도구 안내, 출력 설명 등)으로 구성하는 것이 좋습니다. 형식이 무엇이든 추구하는 바는 **예상되는 동작을 완전히 요약할 수 있는 "최소한의 필수 정보 집합"**입니다("최소"가 "가장 짧음"을 의미하지는 않습니다). 먼저 가장 좋은 모델로 최소한의 프롬프트에서 실행해 본 다음, 실패 모드를 기반으로 명확한 지침과 예시를 추가하세요.

- **도구(Tools)**: 도구는 에이전트와 정보/행동 공간 사이의 계약을 정의하며 효율성을 촉진해야 합니다. 즉, 에이전트의 효율적인 행동을 유도하는 동시에 **토큰 친화적인(token-friendly)** 정보를 반환해야 합니다. 도구는 다음을 충족해야 합니다:
  - 겹침이 적은 단일 책임 및 명확한 인터페이스 시맨틱을 가질 것
  - 오류에 대해 견고할 것
  - 표현 및 추론 측면에서 모델의 강점을 완전히 활용할 수 있도록 명확하고 모호하지 않은 매개변수 설명을 가질 것
  흔히 나타나는 실패 모드는 "비대해진 도구 집합(bloated tool sets)"입니다. 기능 경계가 모호하여 "어떤 도구를 사용할지" 결정하는 것 자체가 모호해집니다. **인간 엔지니어도 어떤 도구를 사용해야 할지 구분할 수 없다면, 에이전트가 더 잘할 것이라고 기대하지 마십시오**. "최소 존립 가능한 도구 집합(Minimum Viable Tool Set, MVTS)"을 신중하게 식별하면 장기적인 상호작용에서 안정성과 유지보수성을 대폭 향상시킬 수 있는 경우가 많습니다.

- **퓨샷 예시(Few-shot Examples)**: 예시를 제공하는 것은 항상 권장되지만, 프롬프트에 "모든 경계 조건"을 채워 넣는 것은 권장하지 않습니다. "예상 동작"을 직접적으로 묘사하는 **다양하고 전형적인** 예시 세트를 신중하게 선택해 주세요. LLM에게 **잘 작성된 예시는 수백 마디의 말보다 값집니다**.

전반적인 핵심 원칙은 **충분하지만 압축된 정보**입니다. 그림 9.2에서 볼 수 있듯이, 이는 런타임에 진입하는 동적 검색의 과정입니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/9-figures/9-2.webp" alt="" width="85%"/>
  <p>그림 9.2 시스템 프롬프트 교정</p>
</div>

### 9.2.2 컨텍스트 검색 및 에이전트 기반 검색

간결한 정의: **에이전트 = 루프 속에서 도구를 자율적으로 호출하는 LLM**. 기본 모델의 능력이 향상됨에 따라 에이전트의 자율성 수준도 향상될 수 있습니다. 에이전트는 더 독립적으로 복잡한 문제 공간을 탐색하고 오류로부터 복구할 수 있습니다.

엔지니어링 실무는 점차 "추론 전 단판 검색(임베딩 검색)"에서 "**적시(Just-in-time, JIT) 컨텍스트**"로 전환하고 있습니다. 후자는 모든 관련 데이터를 미리 로드하지 않고 **가벼운 참조(references)**(파일 경로, 스토리지 쿼리, URL 등)를 유지하며, 런타임에 도구를 통해 필요한 데이터를 동적으로 로드합니다. 이를 통해 모델은 대상이 지정된 쿼리를 작성하고, 필요한 결과를 캐싱하며, 전체 데이터 블록을 한 번에 컨텍스트에 채워 넣지 않고도 `head`/`tail`과 같은 명령어로 대량의 데이터를 분석할 수 있습니다. 이 인지 패턴은 인간과 더 유사합니다. 우리는 모든 정보를 기억하지 않고, 필요할 때마다 파일 시스템, 수신함, 북마크 같은 외부 색인을 사용하여 정보를 추출합니다.

스토리지 효율성뿐만 아니라 **참조의 메타데이터(metadata of references)** 자체도 동작을 개선하는 데 도움이 됩니다. 디렉터리 계층 구조, 명명 규칙, 타임스탬프 등은 모두 암묵적으로 "목적과 적시성"을 전달합니다. 예를 들어, `tests/test_utils.py`와 `src/core/test_utils.py`는 서로 다른 의미적 함의를 갖습니다.

에이전트가 자율적으로 탐색하고 검색할 수 있도록 하면 **점진적 공개(progressive disclosure)**도 가능해집니다. 각 상호작용 단계는 새로운 컨텍스트를 생성하고, 이는 다시 다음 의사결정을 안내합니다. 파일 크기는 복잡성을 나타내고, 이름은 목적을, 타임스탬프는 관련성을 암시합니다. 에이전트는 레이어별로 이해를 쌓아가면서 현재 필요한 하위 집합만 작업 기억 장치에 유지하고 보조적인 영속성을 위해 "메모 작성"을 사용함으로써, "포괄성에 압도되는" 대신 집중력을 유지할 수 있습니다.

이에 대한 트레이드오프(trade-off)는 다음과 같습니다. 런타임 탐색은 미리 계산된 검색보다 느린 경우가 많으며, 모델이 올바른 도구와 휴리스틱을 갖추도록 보장하는 "의견이 반영된(opinionated)" 엔지니어링 설계가 필요합니다. 지침이 없으면 에이전트는 도구를 오용하거나 막다른 골목을 쫓거나 핵심 정보를 놓쳐 컨텍스트를 낭비할 수 있습니다.

많은 시나리오에서 **하이브리드 전략(hybrid strategy)**이 더 효과적입니다. 속도를 보장하기 위해 소량의 "가치가 높은" 컨텍스트를 미리 로드한 다음, 에이전트가 필요에 따라 자율적인 탐색을 계속하도록 허용하는 것입니다. 경계의 선택은 작업의 역동성과 적시성 요구 사항에 달려 있습니다. 엔지니어링 관점에서는 "프로젝트 컨벤션 설명(예: README/가이드)"과 같은 파일을 미리 로드하는 동시에 `glob`, `grep`과 같은 기본 기능(primitives)을 제공하여 에이전트가 특정 파일을 적시에 검색할 수 있도록 함으로써, 오래된 인덱스와 복잡한 구문 분석 트리의 매몰 비용을 우회할 수 있습니다.

### 9.2.3 장기 작업(Long-Horizon Tasks)을 위한 컨텍스트 엔지니어링

장기 작업은 컨텍스트 창을 초과하는 일련의 행동 시퀀스 속에서도 에이전트가 일관성, 컨텍스트 일치성 및 목표 지향성을 유지하도록 요구합니다. 예를 들어 대규모 코드베이스 마이그레이션이나 몇 시간에 걸쳐 체계적으로 진행되는 연구 조사가 이에 해당합니다. 컨텍스트 창을 무한정 늘리는 것만으로는 "컨텍스트 오염" 및 관련성 저하 문제를 해결할 수 없으므로, 이러한 제약 사항에 직접 대응하는 다음과 같은 엔지니어링 방법들이 필요합니다: **압축(Compaction)**, **구조화된 노트 작성(Structured note-taking)**, **서브 에이전트 아키텍처(Sub-agent architectures)**.

- **압축 (Compaction)**
  - 정의: 대화가 컨텍스트 제한에 가까워지면 충실도 높은 요약을 수행하고, 이 요약본으로 새 컨텍스트 창을 다시 시작하여 장기적인 일관성을 유지하는 기법입니다.
  - 실무: 모델이 아키텍처 결정 사항, 해결되지 않은 결함, 구현 세부 사항 등을 압축하여 유지하고, 반복적인 도구 출력과 노이즈는 버리도록 유도합니다. 새 창에는 압축된 요약과 함께 최근에 유의미하게 활용된 아티팩트(예: "최근에 액세스한 파일") 몇 개가 포함됩니다.
  - 튜닝 제안: 먼저 **재현율(recall)**을 최적화(핵심 정보가 누락되지 않도록 보장)한 다음 **정밀도(precision)**를 최적화(중복 콘텐츠 제거)합니다. 안전한 "가벼운" 압축 방식은 "오래전 기록된 도구 호출 및 결과"를 정리하는 것입니다.

- **구조화된 노트 작성 (Structured note-taking)**
  - 정의: "에이전트 메모리"라고도 부릅니다. 에이전트가 핵심 정보를 **컨텍스트 외부의 영구 스토리지**에 고정된 빈도로 기록하고, 후속 단계에서 필요할 때마다 다시 불러오는 기법입니다.
  - 가치: 극도로 낮은 컨텍스트 오버헤드로 지속적인 상태와 종속성을 유지합니다. 예를 들어 TODO 리스트, 프로젝트 `NOTES.md`, 핵심 결론/의존성/장애물의 인덱스를 관리하며 수십 번의 도구 호출과 여러 차례의 컨텍스트 리셋 과정 속에서도 진행 상황과 일관성을 유지할 수 있도록 돕습니다.
  - 참고: 비코딩 시나리오(예: 장기 전략적 작업, 게임/시뮬레이션에서의 목표 관리 및 통계 수치 카운팅 등)에서도 동등하게 효과적입니다. 제8장의 `MemoryTool`과 결합하면 파일 기반 혹은 벡터 기반 외부 메모리를 런타임에 쉽게 구현하고 가져올 수 있습니다.

- **서브 에이전트 아키텍처 (Sub-agent architectures)**
  - 아이디어: 메인 에이전트는 상위 수준의 계획 및 종합을 담당하고, 여러 전문 서브 에이전트들은 각각 "깨끗한 컨텍스트 창"에서 세부 작업을 수행하고 도구를 호출하며 탐색한 뒤, 최종적으로는 오직 **압축된 요약**(대개 1,000~2,000 토큰 내외)만 메인 에이전트에 반환하는 구조입니다.
  - 장점: 관심사의 분리(separation of concerns)를 달성합니다. 복잡한 검색 컨텍스트는 서브 에이전트 내부에 유지되는 한편, 메인 에이전트는 통합과 추론에만 집중합니다. 병렬 탐색이 필요한 복잡한 연구/분석 작업에 적합합니다.
  - 경험: 공개된 다중 에이전트(multi-agent) 연구 시스템에 따르면, 이 패턴은 복잡한 연구 작업에서 단일 에이전트 베이스라인보다 현저한 강점을 보여줍니다.

각 방법 간의 트레이드오프는 다음과 같은 경험 법칙(rules of thumb)을 따를 수 있습니다:

- **압축**: 긴 대화 일관성이 요구되며 컨텍스트 "릴레이"가 강조되는 작업에 적합합니다.
- **구조화된 노트 작성**: 마일스톤이나 단계별 결과물이 존재하는 반복적인 개발 및 연구에 적합합니다.
- **서브 에이전트 아키텍처**: 병렬 탐색으로 혜택을 얻을 수 있는 복잡한 연구 및 분석에 적합합니다.

모델의 기능이 계속 향상되더라도 "긴 상호작용 속에서 일관성과 초점을 유지하는 것"은 견고한 에이전트를 구축할 때 직면하는 핵심 과제로 남아있습니다. 세심하고 체계적인 컨텍스트 엔지니어링은 장기적으로도 핵심 가치를 유지할 것입니다.

## 9.3 Hello-Agents에서의 실무: ContextBuilder

이 섹션에서는 HelloAgents 프레임워크의 컨텍스트 엔지니어링 실무를 자세히 알아봅니다. 설계 동기, 핵심 데이터 구조, 구현 세부 사항부터 전체 사례 연구까지 점진적으로 시연하여 프로덕션 등급의 컨텍스트 관리 시스템을 구축하는 방법을 보여줄 것입니다. ContextBuilder의 설계 철학은 "단순함과 효율성"으로, 불필요한 복잡성을 제거하고 "관련성 + 최신성" 점수를 기반으로 균일하게 선택하여 에이전트 모듈화 및 유지보수성이라는 엔지니어링 지향성에 부합하도록 했습니다.

### 9.3.1 설계 동기 및 목표

ContextBuilder를 구축하기 전에 먼저 설계 목표와 핵심 가치를 명확히 해야 합니다. 훌륭한 컨텍스트 관리 시스템은 다음과 같은 주요 문제를 해결해야 합니다:

1. **단일 진입점(Unified Entry)**: "수집-선택-구조화-압축(Gather-Select-Structure-Compress)" 과정을 재사용 가능한 파이프라인으로 추상화하여, 에이전트 구현 시 반복적인 보일러플레이트 코드를 줄입니다. 이처럼 통일된 인터페이스 설계를 통해 개발자는 각 에이전트마다 컨텍스트 관리 로직을 중복 작성하는 수고를 덜 수 있습니다.

2. **안정적인 형태(Stable Form)**: 고정된 골격을 갖춘 컨텍스트 템플릿을 출력하여 디버깅, A/B 테스트 및 평가를 용이하게 합니다. 우리는 구획이 나누어진 템플릿 구조를 채택했습니다:
   - `[Role & Policies]`: 에이전트의 역할 정의와 행동 가이드라인 명시
   - `[Task]`: 현재 완료해야 할 구체적인 작업
   - `[State]`: 에이전트의 현재 상태와 컨텍스트 정보
   - `[Evidence]`: 외부 지식 베이스로부터 검색된 증거 정보
   - `[Context]`: 대화 이력 및 관련 메모리
   - `[Output]`: 기대하는 출력 형식과 요구 사항

3. **예산 수호자(Budget Guardian)**: 토큰 예산 내에서 가치가 높은 정보를 최대한 유지하고, 한도를 초과하는 컨텍스트에 대해서는 대체 압축 전략을 제공합니다. 이를 통해 방대한 정보가 주어지는 시나리오에서도 시스템이 안정적으로 실행되도록 보장합니다.

4. **최소한의 규칙(Minimum Rules)**: 소스나 우선순위 같은 세분화된 분류 차원을 도입하지 않아 복잡성이 증가하는 것을 방지합니다. 실무 경험상, 관련성과 최신성에 기반한 단순한 스코어링 메커니즘이 대부분의 시나리오에서 충분히 효과적임을 보여줍니다.

### 9.3.2 핵심 데이터 구조

ContextBuilder의 구현은 시스템의 설정과 정보 단위를 정의하는 두 가지 핵심 데이터 구조에 의존합니다.

(1) ContextPacket: 후보 정보 패키지

```python
from dataclasses import dataclass
from typing import Optional, Dict, Any
from datetime import datetime

@dataclass
class ContextPacket:
    """후보 정보 패키지

    Attributes:
        content: 정보 내용
        timestamp: 타임스탬프
        token_count: 토큰 수
        relevance_score: 관련성 점수 (0.0-1.0)
        metadata: 선택적 메타데이터
    """
    content: str
    timestamp: datetime
    token_count: int
    relevance_score: float = 0.5
    metadata: Optional[Dict[str, Any]] = None

    def __post_init__(self):
        """초기화 후 처리"""
        if self.metadata is None:
            self.metadata = {}
        # 관련성 점수가 유효한 범위 내에 있는지 확인
        self.relevance_score = max(0.0, min(1.0, self.relevance_score))
```

`ContextPacket`은 시스템 내 정보의 기본 단위입니다. 각각의 후보 정보는 `ContextPacket`으로 캡슐화되어 내용, 타임스탬프, 토큰 수, 관련성 점수와 같은 핵심 속성을 가집니다. 이처럼 단일화된 데이터 구조는 이후 수행되는 선택 및 정렬 로직을 대폭 간소화합니다.

(2) ContextConfig: 설정 관리

```python
@dataclass
class ContextConfig:
    """컨텍스트 구축 설정

    Attributes:
        max_tokens: 최대 토큰 수
        reserve_ratio: 시스템 지침을 위해 예약할 비율 (0.0-1.0)
        min_relevance: 최소 관련성 임계값
        enable_compression: 압축 활성화 여부
        recency_weight: 최신성 가중치 (0.0-1.0)
        relevance_weight: 관련성 가중치 (0.0-1.0)
    """
    max_tokens: int = 3000
    reserve_ratio: float = 0.2
    min_relevance: float = 0.1
    enable_compression: bool = True
    recency_weight: float = 0.3
    relevance_weight: float = 0.7

    def __post_init__(self):
        """설정 매개변수 유효성 검사"""
        assert 0.0 <= self.reserve_ratio <= 1.0, "reserve_ratio는 [0, 1] 범위여야 합니다"
        assert 0.0 <= self.min_relevance <= 1.0, "min_relevance는 [0, 1] 범위여야 합니다"
        assert abs(self.recency_weight + self.relevance_weight - 1.0) < 1e-6, \
            "recency_weight + relevance_weight는 1.0이어야 합니다"
```

`ContextConfig`는 설정 가능한 모든 매개변수를 캡슐화하여 시스템 동작을 유연하게 조정할 수 있게 합니다. 특히 `reserve_ratio` 매개변수는 시스템 지침과 같은 핵심 정보가 항상 충분한 공간을 확보하여 다른 정보에 의해 밀려나지 않도록 보장하므로 주목할 만합니다.

### 9.3.3 GSSC 파이프라인 상세 설명

ContextBuilder의 핵심은 컨텍스트 구축 프로세스를 4개의 명확한 단계로 분해한 GSSC (Gather-Select-Structure-Compress) 파이프라인입니다. 각 단계의 구현 세부 사항을 자세히 살펴보겠습니다.

(1) Gather: 다중 소스 정보 수집

첫 번째 단계는 다양한 소스에서 후보 정보를 수집하는 것입니다. 이 단계의 핵심은 결함 허용(fault tolerance)과 유연성입니다.

```python
def _gather(
    self,
    user_query: str,
    conversation_history: Optional[List[Message]] = None,
    system_instructions: Optional[str] = None,
    custom_packets: Optional[List[ContextPacket]] = None
) -> List[ContextPacket]:
    """모든 후보 정보 수집

    Args:
        user_query: 사용자 쿼리
        conversation_history: 대화 이력
        system_instructions: 시스템 지침
        custom_packets: 사용자 정의 정보 패키지

    Returns:
        List[ContextPacket]: 후보 정보 목록
    """
    packets = []

    # 1. 시스템 지침 추가 (가장 높은 우선순위, 점수 매기지 않음)
    if system_instructions:
        packets.append(ContextPacket(
            content=system_instructions,
            timestamp=datetime.now(),
            token_count=self._count_tokens(system_instructions),
            relevance_score=1.0,  # 시스템 지침은 항상 보존됨
            metadata={"type": "system_instruction", "priority": "high"}
        ))

    # 2. 메모리 시스템에서 관련 메모리 검색
    if self.memory_tool:
        try:
            memory_results = self.memory_tool.run({
                "action": "search",
                "query": user_query,
                "limit": 10,
                "min_importance": 0.3
            })
            # 메모리 결과를 파싱하고 ContextPacket으로 변환
            memory_packets = self._parse_memory_results(memory_results, user_query)
            packets.extend(memory_packets)
        except Exception as e:
            print(f"[WARNING] 메모리 검색 실패: {e}")

    # 3. RAG 시스템에서 관련 지식 검색
    if self.rag_tool:
        try:
            rag_results = self.rag_tool.run({
                "action": "search",
                "query": user_query,
                "limit": 5,
                "min_score": 0.3
            })
            # RAG 결과를 파싱하고 ContextPacket으로 변환
            rag_packets = self._parse_rag_results(rag_results, user_query)
            packets.extend(rag_packets)
        except Exception as e:
            print(f"[WARNING] RAG 검색 실패: {e}")

    # 4. 대화 이력 추가 (최근 N개만 유지)
    if conversation_history:
        recent_history = conversation_history[-5:]  # 기본적으로 최근 5개 항목만 유지
        for msg in recent_history:
            packets.append(ContextPacket(
                content=f"{msg.role}: {msg.content}",
                timestamp=msg.timestamp if hasattr(msg, 'timestamp') else datetime.now(),
                token_count=self._count_tokens(msg.content),
                relevance_score=0.6,  # 대화 이력 메시지의 기본 관련성 점수
                metadata={"type": "conversation_history", "role": msg.role}
            ))

    # 5. 사용자 정의 정보 패키지 추가
    if custom_packets:
        packets.extend(custom_packets)

    print(f"[ContextBuilder] {len(packets)}개의 후보 정보 패키지를 수집했습니다")
    return packets
```

이 구현은 몇 가지 중요한 설계 고려 사항을 보여줍니다:

- **결함 허용 메커니즘**: 각 외부 데이터 소스 호출은 `try-except`로 래핑되어 있어, 단일 소스가 실패하더라도 전체 프로세스에 영향을 미치지 않도록 보장합니다.
- **우선순위 처리**: 시스템 지침은 높은 우선순위로 표시되어 항상 보존됩니다.
- **이력 제한**: 대화 이력은 가장 최근 항목만 유지하여, 과거 정보로 인해 컨텍스트 창이 가득 차는 것을 방지합니다.

(2) Select: 지능형 정보 선택

두 번째 단계는 관련성과 최신성을 기반으로 후보 정보의 점수를 매기고 선택하는 것입니다. 이는 전체 파이프라인의 핵심이며 최종 컨텍스트의 품질을 직접 결정합니다.

```python
def _select(
    self,
    packets: List[ContextPacket],
    user_query: str,
    available_tokens: int
) -> List[ContextPacket]:
    """가장 관련성 높은 정보 패키지 선택

    Args:
        packets: 후보 정보 패키지 목록
        user_query: 사용자 쿼리 (관련성 계산용)
        available_tokens: 사용 가능한 토큰 수

    Returns:
        List[ContextPacket]: 선택된 정보 패키지 목록
    """
    # 1. 시스템 지침과 기타 정보 분리
    system_packets = [p for p in packets if p.metadata.get("type") == "system_instruction"]
    other_packets = [p for p in packets if p.metadata.get("type") != "system_instruction"]

    # 2. 시스템 지침이 차지하는 토큰 계산
    system_tokens = sum(p.token_count for p in system_packets)
    remaining_tokens = available_tokens - system_tokens

    if remaining_tokens <= 0:
        print("[WARNING] 시스템 지침이 전체 토큰 예산을 차지했습니다")
        return system_packets

    # 3. 기타 정보에 대한 종합 점수 계산
    scored_packets = []
    for packet in other_packets:
        # 관련성 점수 계산 (아직 계산되지 않은 경우)
        if packet.relevance_score == 0.5:  # 기본값인 경우 재계산 필요
            relevance = self._calculate_relevance(packet.content, user_query)
            packet.relevance_score = relevance

        # 최신성 점수 계산
        recency = self._calculate_recency(packet.timestamp)

        # 종합 점수 = 관련성 가중치 × 관련성 + 최신성 가중치 × 최신성
        combined_score = (
            self.config.relevance_weight * packet.relevance_score +
            self.config.recency_weight * recency
        )

        # 최소 관련성 임계값 미만의 정보 필터링
        if packet.relevance_score >= self.config.min_relevance:
            scored_packets.append((combined_score, packet))

    # 4. 점수 기준 내림차순 정렬
    scored_packets.sort(key=lambda x: x[0], reverse=True)

    # 5. 탐욕적(Greedy) 선택: 토큰 한도에 도달할 때까지 점수가 높은 순으로 채움
    selected = system_packets.copy()
    current_tokens = system_tokens

    for score, packet in scored_packets:
        if current_tokens + packet.token_count <= available_tokens:
            selected.append(packet)
            current_tokens += packet.token_count
        else:
            # 토큰 예산이 가득 차면 선택 중단
            break

    print(f"[ContextBuilder] {len(selected)}개의 정보 패키지를 선택했으며, 총 {current_tokens} 토큰입니다")
    return selected

def _calculate_relevance(self, content: str, query: str) -> float:
    """콘텐츠와 쿼리 간의 관련성 계산

    단순 키워드 오버랩 알고리즘을 사용합니다. 프로덕션 환경에서는 벡터 유사도 계산으로 대체할 수 있습니다.

    Args:
        content: 콘텐츠 텍스트
        query: 쿼리 텍스트

    Returns:
        float: 관련성 점수 (0.0-1.0)
    """
    # 토큰화 (간단한 구현, 실제 프로덕션에서는 더 정밀한 토크나이저 사용 권장)
    content_words = set(content.lower().split())
    query_words = set(query.lower().split())

    if not query_words:
        return 0.0

    # 자카드 유사도(Jaccard similarity)
    intersection = content_words & query_words
    union = content_words | query_words

    return len(intersection) / len(union) if union else 0.0

def _calculate_recency(self, timestamp: datetime) -> float:
    """시간적 최신성 점수 계산

    지수 감쇠(exponential decay) 모델을 사용하여 24시간 이내에는 높은 점수를 유지한 뒤 점차 감쇠합니다.

    Args:
        timestamp: 정보 타임스탬프

    Returns:
        float: 최신성 점수 (0.0-1.0)
    """
    import math

    age_hours = (datetime.now() - timestamp).total_seconds() / 3600

    # 지수 감쇠: 24시간 이내에는 높은 점수를 유지하고 이후 점차 감소
    decay_factor = 0.1  # 감쇠 계수
    recency_score = math.exp(-decay_factor * age_hours / 24)

    return max(0.1, min(1.0, recency_score))  # [0.1, 1.0] 범위로 제한
```

선택 단계의 핵심 알고리즘은 몇 가지 중요한 엔지니어링 고려 사항을 구체화합니다:

- **점수 메커니즘**: 설정 가능한 가중치와 함께 관련성 및 최신성의 가중 합을 사용합니다.
- **탐욕적 알고리즘**: 점수가 높은 순서대로 공간을 채워 나가므로, 제한된 예산 내에서 가장 가치 있는 정보가 우선적으로 선택되도록 보장합니다.
- **필터링 메커니즘**: `min_relevance` 매개변수를 통해 품질이 낮은 정보를 걸러냅니다.

(3) Structure: 구조화된 출력

세 번째 단계는 선택된 정보들을 구조화된 컨텍스트 템플릿으로 구성하는 것입니다.

```python
def _structure(self, selected_packets: List[ContextPacket], user_query: str) -> str:
    """선택된 정보 패키지들을 구조화된 컨텍스트 템플릿으로 구성

    Args:
        selected_packets: 선택된 정보 패키지 목록
        user_query: 사용자 쿼리

    Returns:
        str: 구조화된 컨텍스트 문자열
    """
    # 유형별 그룹화
    system_instructions = []
    evidence = []
    context = []

    for packet in selected_packets:
        packet_type = packet.metadata.get("type", "general")

        if packet_type == "system_instruction":
            system_instructions.append(packet.content)
        elif packet_type in ["rag_result", "knowledge"]:
            evidence.append(packet.content)
        else:
            context.append(packet.content)

    # 구조화된 템플릿 작성
    sections = []

    # [Role & Policies]
    if system_instructions:
        sections.append("[Role & Policies]\n" + "\n".join(system_instructions))

    # [Task]
    sections.append(f"[Task]\n{user_query}")

    # [Evidence]
    if evidence:
        sections.append("[Evidence]\n" + "\n---\n".join(evidence))

    # [Context]
    if context:
        sections.append("[Context]\n" + "\n".join(context))

    # [Output]
    sections.append("[Output]\nPlease provide accurate, evidence-based answers based on the above information.")

    return "\n\n".join(sections)
```

구조화 단계는 분산된 정보 패키지들을 명확한 섹션으로 구성합니다. 이 설계는 다음과 같은 몇 가지 장점이 있습니다:

- **가독성(Readability)**: 명확한 섹션을 통해 사람과 모델 모두 컨텍스트 구조를 쉽게 이해할 수 있습니다.
- **디버깅 용이성(Debuggability)**: 문제 위치를 더 수월하게 파악할 수 있어 어떤 영역에 문제가 되는 정보가 있는지 빠르게 식별할 수 있습니다.
- **확장성(Extensibility)**: 새로운 정보 소스를 추가할 때 새로운 섹션만 정의하면 됩니다.

(4) Compress: 대체 압축 (Fallback Compression)

네 번째 단계는 제한을 초과하는 컨텍스트를 압축하는 것입니다.

```python
def _compress(self, context: str, max_tokens: int) -> str:
    """제한을 초과하는 컨텍스트 압축

    Args:
        context: 원본 컨텍스트
        max_tokens: 최대 토큰 제한

    Returns:
        str: 압축된 컨텍스트
    """
    current_tokens = self._count_tokens(context)

    if current_tokens <= max_tokens:
        return context  # 압축 불필요

    print(f"[ContextBuilder] 컨텍스트 초과 ({current_tokens} > {max_tokens}), 압축을 실행합니다")

    # 섹션 압축: 구조적 무결성 유지
    sections = context.split("\n\n")
    compressed_sections = []
    current_total = 0

    for section in sections:
        section_tokens = self._count_tokens(section)

        if current_total + section_tokens <= max_tokens:
            # 완전히 보존
            compressed_sections.append(section)
            current_total += section_tokens
        else:
            # 일부만 보존
            remaining_tokens = max_tokens - current_total
            if remaining_tokens > 50:  # 최소 50 토큰은 유지
                # 단순 잘라내기 (프로덕션 환경에서는 LLM 요약 기능을 사용할 수 있음)
                truncated = self._truncate_text(section, remaining_tokens)
                compressed_sections.append(truncated + "\n[... Content compressed ...]")
            break

    compressed_context = "\n\n".join(compressed_sections)
    final_tokens = self._count_tokens(compressed_context)
    print(f"[ContextBuilder] 압축 완료: {current_tokens} -> {final_tokens} 토큰")

    return compressed_context

def _truncate_text(self, text: str, max_tokens: int) -> str:
    """텍스트를 지정된 토큰 수로 잘라냄

    Args:
        text: 원본 텍스트
        max_tokens: 최대 토큰 수

    Returns:
        str: 잘려진 텍스트
    """
    # 단순 구현: 문자 비율로 추정
    # 프로덕션 환경에서는 정확한 토크나이저를 사용해야 함
    char_per_token = len(text) / self._count_tokens(text) if self._count_tokens(text) > 0 else 4
    max_chars = int(max_tokens * char_per_token)

    return text[:max_chars]

def _count_tokens(self, text: str) -> int:
    """텍스트의 토큰 수 추정

    Args:
        text: 텍스트 내용

    Returns:
        int: 토큰 수
    """
    # 단순 추정: 중국어 1자 ≈ 1 토큰, 영어 1단어 ≈ 1.3 토큰
    # 프로덕션 환경에서는 실제 토크나이저를 사용해야 함
    chinese_chars = sum(1 for ch in text if '\u4e00' <= ch <= '\u9fff')
    english_words = len([w for w in text.split() if w])

    return int(chinese_chars + english_words * 1.3)
```

압축 단계의 설계는 "구조적 무결성 유지" 원칙을 반영합니다. 토큰 예산이 부족한 상황에서도 각 섹션의 핵심 정보를 보존하려고 시도합니다.

### 9.3.4 전체 사용 예시

이제 완전한 예제를 통해 실제 프로젝트에서 ContextBuilder를 사용하는 방법을 보여주겠습니다.

(1) 기본 사용법

```python
from hello_agents.context import ContextBuilder, ContextConfig
from hello_agents.tools import MemoryTool, RAGTool
from hello_agents.core.message import Message
from datetime import datetime

# 1. 도구 초기화
memory_tool = MemoryTool(user_id="user123")
rag_tool = RAGTool(knowledge_base_path="./knowledge_base")

# 2. ContextBuilder 생성
config = ContextConfig(
    max_tokens=3000,
    reserve_ratio=0.2,
    min_relevance=0.2,
    enable_compression=True
)

builder = ContextBuilder(
    memory_tool=memory_tool,
    rag_tool=rag_tool,
    config=config
)

# 3. 대화 이력 준비
conversation_history = [
    Message(content="I'm developing a data analysis tool", role="user", timestamp=datetime.now()),
    Message(content="Great! Data analysis tools usually need to handle large amounts of data. What tech stack do you plan to use?", role="assistant", timestamp=datetime.now()),
    Message(content="I plan to use Python and Pandas, and have completed the CSV reading module", role="user", timestamp=datetime.now()),
    Message(content="Good choice! Pandas is very powerful for data processing. Next you may need to consider data cleaning and transformation.", role="assistant", timestamp=datetime.now()),
]

# 4. 일부 메모리 추가
memory_tool.run({
    "action": "add",
    "content": "User is developing a data analysis tool using Python and Pandas",
    "memory_type": "semantic",
    "importance": 0.8
})

memory_tool.run({
    "action": "add",
    "content": "Completed development of CSV reading module",
    "memory_type": "episodic",
    "importance": 0.7
})

# 5. 컨텍스트 구축
context = builder.build(
    user_query="How to optimize Pandas memory usage?",
    conversation_history=conversation_history,
    system_instructions="You are a senior Python data engineering consultant. Your answers need to: 1) Provide specific actionable advice 2) Explain technical principles 3) Provide code examples"
)

print("=" * 80)
print("Built context:")
print("=" * 80)
print(context)
print("=" * 80)
```

(2) 실행 효과 입증

위 코드를 실행하면 다음과 같이 구조화된 컨텍스트 출력을 볼 수 있습니다:

```
================================================================================
Built context:
================================================================================
[Role & Policies]
You are a senior Python data engineering consultant. Your answers need to: 1) Provide specific actionable advice 2) Explain technical principles 3) Provide code examples

[Task]
How to optimize Pandas memory usage?

[Evidence]
Core strategies for Pandas memory optimization include:
1. Use appropriate data types (such as category instead of object)
2. Read large files in chunks
3. Use chunksize parameter
---
Data type optimization can significantly reduce memory usage. For example, downgrading int64 to int32 can save 50% memory.

[Context]
user: I'm developing a data analysis tool
assistant: Great! Data analysis tools usually need to handle large amounts of data. What tech stack do you plan to use?
user: I plan to use Python and Pandas, and have completed the CSV reading module
assistant: Good choice! Pandas is very powerful for data processing. Next you may need to consider data cleaning and transformation.
Memory: User is developing a data analysis tool using Python and Pandas
Memory: Completed development of CSV reading module

[Output]
Please provide accurate, evidence-based answers based on the above information.
================================================================================
```

이 구조화된 컨텍스트에는 필요한 모든 정보가 포함되어 있습니다:

- **[Role & Policies]**: AI의 역할과 답변 요구 사항을 명시합니다.
- **[Task]**: 사용자의 질문을 명확히 전달합니다.
- **[Evidence]**: RAG 시스템에서 검색된 관련 지식입니다.
- **[Context]**: 충분한 배경 정보를 제공하는 대화 이력 및 관련 메모리입니다.
- **[Output]**: LLM이 답변을 구성하는 방법을 안내합니다.

(3) 에이전트(Agent)와의 통합

마지막으로 ContextBuilder를 에이전트에 통합하는 방법을 살펴보겠습니다:

```python
from hello_agents import SimpleAgent, HelloAgentsLLM, ToolRegistry
from hello_agents.context import ContextBuilder, ContextConfig
from hello_agents.tools import MemoryTool, RAGTool

class ContextAwareAgent(SimpleAgent):
    """컨텍스트 인식 기능을 갖춘 에이전트"""

    def __init__(self, name: str, llm: HelloAgentsLLM, **kwargs):
```
        super().__init__(name=name, llm=llm, system_prompt=kwargs.get("system_prompt", ""))

        # 컨텍스트 빌더 초기화
        self.memory_tool = MemoryTool(user_id=kwargs.get("user_id", "default"))
        self.rag_tool = RAGTool(knowledge_base_path=kwargs.get("knowledge_base_path", "./kb"))

        self.context_builder = ContextBuilder(
            memory_tool=self.memory_tool,
            rag_tool=self.rag_tool,
            config=ContextConfig(max_tokens=4000)
        )

        self.conversation_history = []

    def run(self, user_input: str) -> str:
        """에이전트를 실행하고, 최적화된 컨텍스트를 자동으로 빌드합니다."""

        # 1. ContextBuilder를 사용하여 최적화된 컨텍스트 빌드
        optimized_context = self.context_builder.build(
            user_query=user_input,
            conversation_history=self.conversation_history,
            system_instructions=self.system_prompt
        )

        # 2. 최적화된 컨텍스트로 LLM 호출
        messages = [
            {"role": "system", "content": optimized_context},
            {"role": "user", "content": user_input}
        ]
        response = self.llm.invoke(messages)

        # 3. 대화 기록 업데이트
        from hello_agents.core.message import Message
        from datetime import datetime

        self.conversation_history.append(
            Message(content=user_input, role="user", timestamp=datetime.now())
        )
        self.conversation_history.append(
            Message(content=response, role="assistant", timestamp=datetime.now())
        )

        # 4. 중요한 상호작용을 메모리 시스템에 기록
        self.memory_tool.run({
            "action": "add",
            "content": f"Q: {user_input}\nA: {response[:200]}...",  # 요약
            "memory_type": "episodic",
            "importance": 0.6
        })

        return response

# 사용 예시
agent = ContextAwareAgent(
    name="Data Analysis Consultant",
    llm=HelloAgentsLLM(),
    system_prompt="You are a senior Python data engineering consultant.",
    user_id="user123",
    knowledge_base_path="./data_science_kb"
)

response = agent.run("How to optimize Pandas memory usage?")
print(response)
```

이러한 방식을 통해 ContextBuilder는 에이전트의 '컨텍스트 관리 두뇌'가 되어 정보 수집, 필터링 및 구성을 자동으로 처리하며, 에이전트가 항상 최적의 컨텍스트 하에서 추론하고 생성할 수 있도록 돕습니다.

### 9.3.5 모범 사례 및 최적화 권장사항

ContextBuilder를 실제로 적용할 때 다음과 같은 모범 사례들을 유념할 필요가 있습니다:

1. **토큰 예산의 동적 조정**: 작업 복잡도에 따라 `max_tokens`를 동적으로 조정합니다. 단순한 작업에는 더 적은 예산을 사용하고, 복잡한 작업에는 예산을 늘립니다.

2. **관련성 계산 최적화**: 프로덕션 환경에서는 단순한 키워드 중복 검사 대신 벡터 유사도 계산을 사용하여 검색 품질을 개선합니다.

3. **캐싱 메커니즘**: 변경되지 않는 시스템 지침 및 지식 베이스 콘텐츠의 경우, 캐싱 메커니즘을 구현하여 반복적인 계산을 피합니다.

4. **모니터링 및 로깅**: 향후 최적화를 위해 컨텍스트 빌드 시마다 통계 정보(선택된 정보 수, 토큰 사용률 등)를 기록합니다.

5. **A/B 테스트**: 핵심 매개변수(예: 관련성 가중치, 최신성 가중치)에 대해 A/B 테스트를 통해 최적의 구성을 찾습니다.

## 9.4 NoteTool: 구조화된 노트

NoteTool은 '장기 실행 작업(long-horizon tasks)'을 위해 제공되는 구조화된 외부 메모리 컴포넌트입니다. 헤더의 YAML 프런트 매터(front matter)로 핵심 정보를 기록하고, 본문에는 상태, 결론, 장애 요인(blockers), 후속 조치 사항(action items)을 기록하는 마크다운(Markdown) 파일을 캐리어로 사용합니다. 이러한 디자인은 인간의 가독성, 버전 관리 편의성, 컨텍스트로의 재주입 용이성을 결합하여 장기 실행 에이전트를 구축하기 위한 중요한 도구로 활용됩니다.

### 9.4.1 설계 철학 및 응용 시나리오

구현 세부 사항을 살펴보기 전에, 먼저 NoteTool의 설계 철학과 대표적인 응용 시나리오를 이해해 봅시다.

(1) 왜 NoteTool이 필요한가?

8장에서는 강력한 메모리 관리 기능을 제공하는 MemoryTool을 소개했습니다. 하지만 MemoryTool은 주로 **대화형 메모리**(단기 작업 메모리, 일화적 메모리, 의미적 메모리)에 중점을 둡니다. 장기적인 추적과 구조화된 관리가 필요한 **프로젝트 기반 작업**의 경우, 더 가볍고 인간 친화적인 기록 방식이 필요합니다.

NoteTool은 다음과 같은 기능을 제공하여 이 격차를 채워줍니다.

- **구조화된 기록**: 마크다운 + YAML 형식을 사용하여 기계 파싱과 인간의 읽기 및 편집에 모두 적합합니다.
- **버전 관리 친화적**: 평문 텍스트 형식으로, Git과 같은 버전 관리 시스템을 자연스럽게 지원합니다.
- **낮은 오버헤드**: 복잡한 데이터베이스 작업이 필요하지 않아 가벼운 상태 추적에 적합합니다.
- **유연한 분류**: `type`과 `tags`를 통해 노트를 유연하게 정리하여 다차원 검색을 지원합니다.

(2) 대표적인 응용 시나리오

NoteTool은 특히 다음과 같은 시나리오에 적합합니다.

**시나리오 1: 장기 프로젝트 추적**

에이전트가 며칠 또는 몇 주가 걸릴 수 있는 대규모 코드베이스 리팩터링 작업을 돕고 있다고 상상해 보십시오. NoteTool은 다음을 기록할 수 있습니다.

- `task_state`: 현재 단계의 작업 상태 및 진행 상황
- `conclusion`: 각 단계가 종료된 후의 핵심 결론
- `blocker`: 마주한 문제 및 장애 요인
- `action`: 다음 조치 계획

```python
# 작업 상태 기록
notes.run({
    "action": "create",
    "title": "Refactoring Project - Phase 1",
    "content": "Completed refactoring of data model layer, test coverage reached 85%. Next will refactor business logic layer.",
    "note_type": "task_state",
    "tags": ["refactoring", "phase1"]
})

# 장애 요인 기록
notes.run({
    "action": "create",
    "title": "Dependency Conflict Issue",
    "content": "Found some third-party library versions incompatible, need to resolve. Impact scope: 3 modules in business logic layer.",
    "note_type": "blocker",
    "tags": ["dependency", "urgent"]
})
```

**시나리오 2: 연구 작업 관리**

문헌 조사를 수행하는 지능형 연구 에이전트는 NoteTool을 사용하여 다음을 기록할 수 있습니다.

- 각 논문의 핵심 관점 (`conclusion`)
- 심층 조사할 주제 (`action`)
- 중요한 참고 문헌 (`reference`)

**시나리오 3: ContextBuilder와의 협업**

매 대화 라운드 전에 에이전트는 `search` 또는 `list` 작업을 통해 관련 노트를 검색하여 컨텍스트에 주입할 수 있습니다.

```python
# 에이전트의 run 메서드 내
def run(self, user_input: str) -> str:
    # 1. 관련 노트 검색
    relevant_notes = self.note_tool.run({
        "action": "search",
        "query": user_input,
        "limit": 3
    })

    # 2. 노트 콘텐츠를 ContextPacket으로 변환
    note_packets = []
    for note in relevant_notes:
        note_packets.append(ContextPacket(
            content=note['content'],
            timestamp=note['updated_at'],
            token_count=self._count_tokens(note['content']),
            relevance_score=0.7,
            metadata={"type": "note", "note_type": note['type']}
        ))

    # 3. 컨텍스트 빌드 시 노트를 전달
    context = self.context_builder.build(
        user_query=user_input,
        custom_packets=note_packets,
        ...
    )
```

### 9.4.2 저장 형식 상세 설명

NoteTool은 구조와 가독성의 균형을 맞춘 마크다운 + YAML 혼합 형식을 채택하고 있습니다.

(1) 노트 파일 형식

각 노트는 다음과 같은 형식을 가진 독립적인 `.md` 파일입니다.

```markdown
---
id: note_20250119_153000_0
title: Project Progress - Phase 1
type: task_state
tags: [refactoring, phase1, backend]
created_at: 2025-01-19T15:30:00
updated_at: 2025-01-19T15:30:00
---

# 프로젝트 진행 상황 - 1단계

## 완료 상태

데이터 모델 레이어의 리팩터링을 완료했습니다. 주요 변경 사항은 다음과 같습니다:

1. 엔티티 클래스 명명 규칙 통일
2. 코드 유지보수성 향상을 위해 타입 힌트 도입
3. 데이터베이스 쿼리 성능 최적화

## 테스트 커버리지

- 단위 테스트 커버리지: 85%
- 통합 테스트 커버리지: 70%

## 다음 단계

1. 비즈니스 로직 레이어 리팩터링
2. 종속성 충돌 문제 해결
3. 통합 테스트 커버리지를 85%로 상향
```

이 형식의 장점:

- **YAML 메타데이터**: 기계 파싱이 가능하여 정밀한 필드 추출 및 검색을 지원합니다.
- **마크다운 본문**: 인간이 읽기 쉬우며 다양한 서식(제목, 목록, 코드 블록 등)을 지원합니다.
- **ID 역할의 파일명**: 관리를 단순화하며, 각 노트의 파일명이 고유 식별자가 됩니다.

(2) 인덱스 파일

NoteTool은 노트를 빠르게 검색하고 관리하기 위해 `notes_index.json` 파일을 유지 관리합니다.

```json
{
  "note_20250119_153000_0": {
    "id": "note_20250119_153000_0",
    "title": "프로젝트 진행 상황 - 1단계",
    "type": "task_state",
    "tags": ["refactoring", "phase1", "backend"],
    "created_at": "2025-01-19T15:30:00",
    "updated_at": "2025-01-19T15:30:00",
    "file_path": "./notes/note_20250119_153000_0.md"
  }
}
```

이 인덱스 파일의 역할:

- **빠른 검색**: 각 파일을 일일이 열어볼 필요 없이 인덱스에서 직접 검색합니다.
- **메타데이터 관리**: 모든 노트의 메타데이터를 중앙에서 관리합니다.
- **무결성 검사**: 누락되거나 손상된 파일을 감지할 수 있습니다.

### 9.4.3 핵심 작업 상세 설명

NoteTool은 노트의 전체 수명 주기 관리를 다루는 7가지 핵심 작업을 제공합니다.

(1) create: 노트 생성

```python
def _create_note(
    self,
    title: str,
    content: str,
    note_type: str = "general",
    tags: Optional[List[str]] = None
) -> str:
    """노트를 생성합니다.

    Args:
        title: 노트 제목
        content: 노트 내용 (마크다운 형식)
        note_type: 노트 유형 (task_state/conclusion/blocker/action/reference/general)
        tags: 태그 목록

    Returns:
        str: 노트 ID
    """
    from datetime import datetime

    # 1. 고유 ID 생성
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    note_id = f"note_{timestamp}_{len(self.index)}"

    # 2. 메타데이터 구축
    metadata = {
        "id": note_id,
        "title": title,
        "type": note_type,
        "tags": tags or [],
        "created_at": datetime.now().isoformat(),
        "updated_at": datetime.now().isoformat()
    }

    # 3. 전체 마크다운 파일 콘텐츠 구축
    md_content = self._build_markdown(metadata, content)

    # 4. 파일에 저장
    file_path = os.path.join(self.workspace, f"{note_id}.md")
    with open(file_path, 'w', encoding='utf-8') as f:
        f.write(md_content)

    # 5. 인덱스 업데이트
    metadata["file_path"] = file_path
    self.index[note_id] = metadata
    self._save_index()

    return note_id

def _build_markdown(self, metadata: Dict, content: str) -> str:
    """마크다운 파일 콘텐츠를 구축합니다. (YAML + 본문)"""
    import yaml

    # YAML 프런트 매터
    yaml_header = yaml.dump(metadata, allow_unicode=True, sort_keys=False)

    # 결합된 형식
    return f"---\n{yaml_header}---\n\n{content}"
```

사용 예시:

```python
from hello_agents.tools import NoteTool

notes = NoteTool(workspace="./project_notes")

note_id = notes.run({
    "action": "create",
    "title": "Refactoring Project - Phase 1",
    "content": """## Completion Status
Completed refactoring of data model layer, test coverage reached 85%.

## Next Steps
Refactor business logic layer""",
    "note_type": "task_state",
    "tags": ["refactoring", "phase1"]
})

print(f"✅ 노트가 성공적으로 생성되었습니다. ID: {note_id}")
```

(2) read: 노트 읽기

```python
def _read_note(self, note_id: str) -> Dict:
    """노트 콘텐츠를 읽습니다.

    Args:
        note_id: 노트 ID

    Returns:
        Dict: 메타데이터와 본문을 포함하는 딕셔너리
    """
    if note_id not in self.index:
        raise ValueError(f"노트가 존재하지 않습니다: {note_id}")

    file_path = self.index[note_id]["file_path"]

    # 파일 읽기
    with open(file_path, 'r', encoding='utf-8') as f:
        raw_content = f.read()

    # YAML 메타데이터 및 마크다운 본문 파싱
    metadata, content = self._parse_markdown(raw_content)

    return {
        "metadata": metadata,
        "content": content
    }

def _parse_markdown(self, raw_content: str) -> Tuple[Dict, str]:
    """마크다운 파일을 파싱합니다. (YAML과 본문 분리)"""
    import yaml

    # YAML 구분 기호 찾기
    parts = raw_content.split('---\n', 2)

    if len(parts) >= 3:
        # YAML 프런트 매터가 있는 경우
        yaml_str = parts[1]
        content = parts[2].strip()
        metadata = yaml.safe_load(yaml_str)
    else:
        # 메타데이터가 없는 경우 전체를 본문으로 처리
        metadata = {}
        content = raw_content.strip()

    return metadata, content
```

(3) update: 노트 업데이트

```python
def _update_note(
    self,
    note_id: str,
    title: Optional[str] = None,
    content: Optional[str] = None,
    note_type: Optional[str] = None,
    tags: Optional[List[str]] = None
) -> str:
    """노트를 업데이트합니다.

    Args:
        note_id: 노트 ID
        title: 새로운 제목 (선택 사항)
        content: 새로운 내용 (선택 사항)
        note_type: 새로운 유형 (선택 사항)
        tags: 새로운 태그 (선택 사항)

    Returns:
        str: 작업 결과 메시지
    """
    if note_id not in self.index:
        raise ValueError(f"노트가 존재하지 않습니다: {note_id}")

    # 1. 기존 노트 읽기
    note = self._read_note(note_id)
    metadata = note["metadata"]
    old_content = note["content"]

    # 2. 필드 업데이트
    if title:
        metadata["title"] = title
    if note_type:
        metadata["type"] = note_type
    if tags is not None:
        metadata["tags"] = tags
    if content is not None:
        old_content = content

    # 타임스탬프 업데이트
    from datetime import datetime
    metadata["updated_at"] = datetime.now().isoformat()

    # 3. 재구축 및 저장
    md_content = self._build_markdown(metadata, old_content)
    file_path = metadata["file_path"]

    with open(file_path, 'w', encoding='utf-8') as f:
        f.write(md_content)

    # 4. 인덱스 업데이트
    self.index[note_id] = metadata
    self._save_index()

    return f"✅ 노트가 업데이트되었습니다: {metadata['title']}"
```

(4) search: 노트 검색

```python
def _search_notes(
    self,
    query: str,
    limit: int = 10,
    note_type: Optional[str] = None,
    tags: Optional[List[str]] = None
) -> List[Dict]:
    """노트를 검색합니다.

    Args:
        query: 검색 키워드
        limit: 반환할 수량 제한
        note_type: 유형 필터링 (선택 사항)
        tags: 태그 필터링 (선택 사항)

    Returns:
        List[Dict]: 일치하는 노트 목록
    """
    results = []
    query_lower = query.lower()

    for note_id, metadata in self.index.items():
        # 유형 필터
        if note_type and metadata.get("type") != note_type:
            continue

        # 태그 필터
        if tags:
            note_tags = set(metadata.get("tags", []))
            if not note_tags.intersection(tags):
                continue

        # 노트 내용 읽기
        try:
            note = self._read_note(note_id)
            content = note["content"]
            title = metadata.get("title", "")

            # 제목 및 내용에서 검색
            if query_lower in title.lower() or query_lower in content.lower():
                results.append({
                    "note_id": note_id,
                    "title": title,
                    "type": metadata.get("type"),
                    "tags": metadata.get("tags", []),
                    "content": content,
                    "updated_at": metadata.get("updated_at")
                })
        except Exception as e:
            print(f"[경고] 노트를 읽지 못했습니다 {note_id}: {e}")
            continue

    # 업데이트 시간 기준 정렬
    results.sort(key=lambda x: x["updated_at"], reverse=True)

    return results[:limit]
```

(5) list: 노트 목록 표시

```python
def _list_notes(
    self,
    note_type: Optional[str] = None,
    tags: Optional[List[str]] = None,
    limit: int = 20
) -> List[Dict]:
    """노트 목록을 표시합니다. (업데이트 시간 기준 내림차순)

    Args:
        note_type: 유형 필터링 (선택 사항)
        tags: 태그 필터링 (선택 사항)
        limit: 반환할 수량 제한

    Returns:
        List[Dict]: 노트 메타데이터 목록
    """
    results = []

    for note_id, metadata in self.index.items():
        # 유형 필터
        if note_type and metadata.get("type") != note_type:
            continue

        # 태그 필터
        if tags:
            note_tags = set(metadata.get("tags", []))
            if not note_tags.intersection(tags):
                continue

        results.append(metadata)

    # 업데이트 시간 기준 정렬
    results.sort(key=lambda x: x.get("updated_at", ""), reverse=True)

    return results[:limit]
```

(6) summary: 노트 요약

```python
def _summary(self) -> Dict[str, Any]:
    """노트 요약 통계를 생성합니다.

    Returns:
        Dict: 통계 정보
    """
    total_count = len(self.index)

    # 유형별 집계
    type_counts = {}
    for metadata in self.index.values():
        note_type = metadata.get("type", "general")
        type_counts[note_type] = type_counts.get(note_type, 0) + 1

    # 최근 업데이트된 노트
    recent_notes = sorted(
        self.index.values(),
        key=lambda x: x.get("updated_at", ""),
        reverse=True
    )[:5]

    return {
        "total_notes": total_count,
        "type_distribution": type_counts,
        "recent_notes": [
            {
                "id": note["id"],
                "title": note.get("title", ""),
                "type": note.get("type"),
                "updated_at": note.get("updated_at")
            }
            for note in recent_notes
        ]
    }
```

(7) delete: 노트 삭제

```python
def _delete_note(self, note_id: str) -> str:
    """노트를 삭제합니다.

    Args:
        note_id: 노트 ID

    Returns:
        str: 작업 결과 메시지
    """
    if note_id not in self.index:
        raise ValueError(f"노트가 존재하지 않습니다: {note_id}")

    # 1. 파일 삭제
    file_path = self.index[note_id]["file_path"]
    if os.path.exists(file_path):
        os.remove(file_path)

    # 2. 인덱스에서 제거
    title = self.index[note_id].get("title", note_id)
    del self.index[note_id]
    self._save_index()

    return f"✅ 노트가 삭제되었습니다: {title}"
```

### 9.4.4 ContextBuilder와의 심층 통합

NoteTool의 진정한 강력함은 ContextBuilder와 함께 결합하여 사용할 때 나타납니다. 구체적인 실제 통합 사례를 통해 이를 살펴보겠습니다.

(1) 시나리오 설정

우리가 다음과 같은 기능을 수행해야 하는 장기 프로젝트 도우미를 구축한다고 가정해 봅시다.

1. 프로젝트의 단계적 진행 상황 기록
2. 미결된 이슈 추적
3. 대화할 때마다 자동으로 관련 노트 검토
4. 과거 노트를 바탕으로 일관성 있는 추천 제공

(2) 구현 예시

```python
from hello_agents import SimpleAgent, HelloAgentsLLM
from hello_agents.context import ContextBuilder, ContextConfig, ContextPacket
from hello_agents.tools import MemoryTool, RAGTool, NoteTool
from datetime import datetime

class ProjectAssistant(SimpleAgent):
    """NoteTool과 ContextBuilder를 통합한 장기 프로젝트 도우미 에이전트"""

    def __init__(self, name: str, project_name: str, **kwargs):
        super().__init__(name=name, llm=HelloAgentsLLM(), **kwargs)

        self.project_name = project_name

        # 도구 초기화
        self.memory_tool = MemoryTool(user_id=project_name)
        self.rag_tool = RAGTool(knowledge_base_path=f"./{project_name}_kb")
        self.note_tool = NoteTool(workspace=f"./{project_name}_notes")

        # 컨텍스트 빌더 초기화
        self.context_builder = ContextBuilder(
            memory_tool=self.memory_tool,
            rag_tool=self.rag_tool,
            config=ContextConfig(max_tokens=4000)
        )

        self.conversation_history = []

    def run(self, user_input: str, note_as_action: bool = False) -> str:
        """어시스턴트를 실행하고 노트를 자동으로 통합합니다."""

        # 1. NoteTool에서 관련 노트 검색
        relevant_notes = self._retrieve_relevant_notes(user_input)

        # 2. 노트를 ContextPacket으로 변환
        note_packets = self._notes_to_packets(relevant_notes)

        # 3. 최적화된 컨텍스트 빌드
        context = self.context_builder.build(
            user_query=user_input,
            conversation_history=self.conversation_history,
            system_instructions=self._build_system_instructions(),
            custom_packets=note_packets
        )

        # 4. LLM 호출
        response = self.llm.invoke(context)

        # 5. 필요 시, 상호작용을 노트로 저장
        if note_as_action:
            self._save_as_note(user_input, response)

        # 6. 대화 기록 업데이트
        self._update_history(user_input, response)

        return response

    def _retrieve_relevant_notes(self, query: str, limit: int = 3) -> List[Dict]:
        """관련 노트를 검색합니다."""
        try:
            # blocker 및 action 유형 노트를 우선적으로 가져옵니다.
            blockers = self.note_tool.run({
                "action": "list",
                "note_type": "blocker",
                "limit": 2
            })

            # 일반 검색
            search_results = self.note_tool.run({
                "action": "search",
                "query": query,
                "limit": limit
            })

            # 병합 및 중복 제거
            all_notes = {note['note_id']: note for note in blockers + search_results}
            return list(all_notes.values())[:limit]

        except Exception as e:
            print(f"[WARNING] Note retrieval failed: {e}")
            return []

    def _notes_to_packets(self, notes: List[Dict]) -> List[ContextPacket]:
        """노트를 컨텍스트 패킷으로 변환"""
        packets = []

        for note in notes:
            content = f"[Note: {note['title']}]\n{note['content']}"

            packets.append(ContextPacket(
                content=content,
                timestamp=datetime.fromisoformat(note['updated_at']),
                token_count=len(content) // 4,  # 단순 추정
                relevance_score=0.75,  # 노트는 높은 관련성을 가짐
                metadata={
                    "type": "note",
                    "note_type": note['type'],
                    "note_id": note['note_id']
                }
            ))

        return packets

    def _save_as_note(self, user_input: str, response: str):
        """상호작용을 노트로 저장"""
        try:
            # 저장할 노트 타입 결정
            if "problem" in user_input.lower() or "blocker" in user_input.lower():
                note_type = "blocker"
            elif "plan" in user_input.lower() or "next" in user_input.lower():
                note_type = "action"
            else:
                note_type = "conclusion"

            self.note_tool.run({
                "action": "create",
                "title": f"{user_input[:30]}...",
                "content": f"## Question\n{user_input}\n\n## Analysis\n{response}",
                "note_type": note_type,
                "tags": [self.project_name, "auto_generated"]
            })

        except Exception as e:
            print(f"[WARNING] Failed to save note: {e}")

    def _build_system_instructions(self) -> str:
        """시스템 지침 빌드"""
        return f"""귀하는 {self.project_name} 프로젝트를 위한 장기 어시스턴트입니다.

귀하의 역할:
1. 과거 노트를 바탕으로 일관된 권장사항 제공
2. 프로젝트 진행 상황 및 미결 문제 추적
3. 답변 시 관련 과거 노트 참조
4. 구체적이고 실행 가능한 다음 단계 권장사항 제공

참고사항:
- blocker로 표시된 문제를 우선시합니다.
- 권장사항의 근거 출처(노트, 메모리 또는 지식 베이스)를 표시합니다.
- 전체적인 프로젝트 진행 상황을 파악하고 있어야 합니다."""

    def _update_history(self, user_input: str, response: str):
        """대화 기록 업데이트"""
        from hello_agents.core.message import Message

        self.conversation_history.append(
            Message(content=user_input, role="user", timestamp=datetime.now())
        )
        self.conversation_history.append(
            Message(content=response, role="assistant", timestamp=datetime.now())
        )

        # 기록 길이 제한
        if len(self.conversation_history) > 10:
            self.conversation_history = self.conversation_history[-10:]

# 사용 예시
assistant = ProjectAssistant(
    name="Project Assistant",
    project_name="data_pipeline_refactoring"
)

# 첫 번째 상호작용: 프로젝트 상태 기록
response = assistant.run(
    "데이터 모델 레이어의 리팩토링을 완료했으며, 테스트 커버리지는 85%에 도달했습니다. 다음 계획은 비즈니스 로직 레이어를 리팩토링하는 것입니다.",
    note_as_action=True
)

# 두 번째 상호작용: 문제 제기
response = assistant.run(
    "비즈니스 로직 레이어를 리팩토링할 때 의존성 버전 충돌 문제가 발생했습니다. 이를 어떻게 해결해야 하나요?"
)

# 노트 요약 조회
summary = assistant.note_tool.run({"action": "summary"})
print(summary)
```

(3) 실행 효과 실습 데모

```bash
[ContextBuilder] Collected 8 candidate information packages
[ContextBuilder] Selected 7 information packages, total 3500 tokens

✅ 어시스턴트 답변:

이전에 기록된 노트에서 이 문제가 언급된 것을 확인했습니다. [리팩토링 프로젝트 - 1단계] 노트에 따르면, 현재 테스트 커버리지가 85%에 도달하여 좋은 기반이 마련되었습니다.

의존성 버전 충돌 문제와 관련하여 다음과 같이 권장합니다:

1. **가상 환경 격리 사용**: 다른 모듈과의 의존성 충돌을 방지하기 위해 비즈니스 로직 레이어를 위한 독립적인 가상 환경을 생성합니다.
2. **버전 고정**: requirements.txt 파일에 모든 의존성의 정확한 버전을 명시적으로 지정합니다.
3. **pipdeptree 사용**: 의존성 트리를 분석하여 충돌의 근본 원인을 파악합니다.

이 문제를 blocker(차단 요인)로 표시하고 우선적으로 해결할 것을 권장합니다.

[출처: 노트 note_20250119_153000_0, 프로젝트 지식 베이스]

---

📋 노트 요약:
{
  "total_notes": 2,
  "type_distribution": {
    "action": 1,
    "blocker": 1
  },
  "recent_notes": [
    {
      "id": "note_20250119_154500_1",
      "title": "비즈니스 로직 레이어를 리팩토링할 때, 의존성 버전 충돌 문제가 발생했습니다...",
      "type": "blocker",
      "updated_at": "2025-01-19T15:45:00"
    },
    {
      "id": "note_20250119_153000_0",
      "title": "데이터 모델 레이어의 리팩토링을 완료했습니다...",
      "type": "action",
      "updated_at": "2025-01-19T15:30:00"
    }
  ]
}
```

### 9.4.5 베스트 프랙티스

실제로 NoteTool을 사용할 때, 다음의 베스트 프랙티스를 참고하면 더 강력한 장기 실행(Long-horizon) 에이전트를 구축하는 데 도움이 됩니다.

1. **합리적인 노트 분류**:
   - `task_state`: 단계별 진행 상황 및 상태 기록
   - `conclusion`: 중요한 결론 및 결과 기록
   - `blocker`: 진행을 가로막는 문제(최우선 순위) 기록
   - `action`: 향후 실행 계획 기록
   - `reference`: 중요한 참고 자료 기록

2. **정기적인 정리 및 아카이빙**:
   - 해결된 blocker는 conclusion으로 업데이트합니다.
   - 오래된 action은 즉시 삭제하거나 업데이트합니다.
   - 버전 관리를 위해 `["v1.0", "completed"]`와 같은 태그를 사용합니다.

3. **ContextBuilder와의 협업**:
   - 매 대화 단계 전에 관련 노트를 검색합니다.
   - 노트 유형에 따라 각기 다른 관련성 점수를 설정합니다(blocker > action > conclusion).
   - 컨텍스트 과부하를 방지하기 위해 노트 개수를 제한합니다.

4. **인간-기계 협업(Human-machine collaboration)**:
   - 노트는 사람이 읽을 수 있는 마크다운(Markdown) 형식이므로 직접 수동 편집이 가능합니다.
   - Git과 같은 버전 관리 도구를 사용하여 노트의 변경 이력을 추적합니다.
   - 주요 단계에서 에이전트가 생성한 노트를 직접 검토합니다.

5. **자동화된 워크플로우**:
   - 정기적으로 노트 요약 보고서를 생성합니다.
   - 노트를 기반으로 프로젝트 진행 문서를 자동으로 작성합니다.
   - 노트 내용을 다른 시스템(예: Notion, Confluence 등)에 동기화합니다.

## 9.5 TerminalTool: 즉각적인 파일 시스템 액세스

이전 장에서 대화식 메모리와 지식 검색 기능을 각각 제공하는 MemoryTool과 RAGTool을 소개했습니다. 하지만 실제의 많은 시나리오에서 에이전트는 로그 파일 확인, 코드베이스 구조 분석, 설정 파일 검색 등 **파일 시스템에 대한 즉각적인 접근 및 탐색**이 필요합니다. 이러한 경우에 TerminalTool이 사용됩니다.

TerminalTool은 에이전트에게 **안전한 명령줄(command-line) 실행 기능**을 제공하여 일반적인 파일 시스템 및 텍스트 처리 명령을 지원하는 동시에, 다층 보안 메커니즘을 통해 시스템의 안전을 보장합니다. 이 설계는 9.2.2절에서 언급한 "적시(Just-in-time, JIT) 컨텍스트" 개념을 구현한 것입니다. 즉, 에이전트는 모든 파일을 미리 로드할 필요 없이 필요할 때 필요한 만큼 탐색하고 조회할 수 있습니다.

### 9.5.1 설계 철학 및 보안 메커니즘

(1) 왜 TerminalTool이 필요한가?

장기 실행 에이전트를 구축할 때 다음과 같은 시나리오를 자주 마주하게 됩니다.

**시나리오 1: 코드베이스 탐색**

개발 어시스턴트가 사용자에게 대규모 코드베이스의 구조를 이해하도록 도와야 하는 경우:

```python
# 기존 방식: 모든 파일 사전 인덱싱 (비용이 많이 들고 최신 상태가 아닐 수 있음)
rag_tool.add_document("./project/**/*.py")  # 시간이 오래 걸리고 큰 저장 공간을 차지함

# TerminalTool 방식: 즉각적인 탐색
terminal.run({"command": "find . -name '*.py' -type f"})  # 빠르고 실시간임
terminal.run({"command": "grep -r 'class UserService' ."})  # 정확한 위치 탐색
terminal.run({"command": "head -n 50 src/services/user.py"})  # 필요한 만큼만 조회
```

**시나리오 2: 로그 파일 분석**

운영 어시스턴트가 애플리케이션 로그를 분석해야 하는 경우:

```python
# 로그 파일 크기 확인
terminal.run({"command": "ls -lh /var/log/app.log"})

# 최신 에러 로그 확인
terminal.run({"command": "tail -n 100 /var/log/app.log | grep ERROR"})

# 에러 유형 분포 집계
terminal.run({"command": "grep ERROR /var/log/app.log | cut -d':' -f3 | sort | uniq -c"})
```

**시나리오 3: 데이터 파일 미리보기**

데이터 분석 어시스턴트가 데이터 파일의 구조를 빠르게 파악해야 하는 경우:

```python
# CSV 파일의 처음 몇 줄 확인
terminal.run({"command": "head -n 5 data/sales.csv"})

# 라인 수 계산
terminal.run({"command": "wc -l data/*.csv"})

# 컬럼명 확인
terminal.run({"command": "head -n 1 data/sales.csv | tr ',' '\n'"})
```

이러한 시나리오들의 공통적인 특징은 **사전 인덱싱 및 벡터화가 아닌, 실시간의 경량화된 파일 시스템 액세스가 필요하다는 점**입니다. TerminalTool은 바로 이러한 "탐색형" 워크플로우를 위해 설계되었습니다.

(2) 보안 메커니즘 상세 설명

에이전트가 명령을 실행하도록 허용하는 것은 강력하지만 동시에 위험한 기능입니다. TerminalTool은 다층 보안 메커니즘을 통해 시스템 보안을 보장합니다.

**첫 번째 계층: 명령 화이트리스트(Command Whitelist)**

안전한 읽기 전용 명령만 허용하고, 시스템을 수정할 가능성이 있는 어떠한 작업도 철저히 금지합니다.

```python
ALLOWED_COMMANDS = {
    # 파일 목록 조회 및 정보
    'ls', 'dir', 'tree',
    # 파일 내용 조회
    'cat', 'head', 'tail', 'less', 'more',
    # 파일 검색
    'find', 'grep', 'egrep', 'fgrep',
    # 텍스트 처리
    'wc', 'sort', 'uniq', 'cut', 'awk', 'sed',
    # 디렉토리 작업
    'pwd', 'cd',
    # 파일 정보
    'file', 'stat', 'du', 'df',
    # 기타
    'echo', 'which', 'whereis',
}
```

에이전트가 화이트리스트 이외의 명령을 실행하려고 하면 즉시 차단됩니다.

```python
terminal.run({"command": "rm -rf /"})
# ❌ Command not allowed: rm
# Allowed commands: cat, cd, cut, dir, du, ...
```

**두 번째 계층: 작업 디렉토리 제한(샌드박스)**

TerminalTool은 지정된 작업 디렉토리 및 그 하위 디렉토리에만 접근할 수 있으며, 시스템의 다른 영역에는 접근할 수 없습니다.

```python
# 초기화할 때 작업 디렉토리를 지정합니다.
terminal = TerminalTool(workspace="./project")

# 허용됨: 작업 디렉토리 내부 파일에 접근
terminal.run({"command": "cat ./src/main.py"})  # ✅

# 금지됨: 작업 디렉토리 외부 파일에 접근
terminal.run({"command": "cat /etc/passwd"})  # ❌ 작업 디렉토리 외부 경로에 접근할 수 없습니다.

# 금지됨: ..을 통해 상위로 탈출 시도
terminal.run({"command": "cd ../../../etc"})  # ❌ 작업 디렉토리 외부 경로에 접근할 수 없습니다.
```

이러한 샌드박스 메커니즘을 통해 에이전트의 오작동이 있더라도 시스템의 다른 영역에는 영향을 주지 않도록 합니다.

**세 번째 계층: 시간 제한(Timeout) 제어**

무한 루프나 리소스 고갈을 방지하기 위해 각 명령에는 실행 시간 제한이 설정되어 있습니다.

```python
terminal = TerminalTool(
    workspace="./project",
    timeout=30  # 30초 시간 제한
)

# 명령 실행 시간이 30초를 초과하는 경우
terminal.run({"command": "find / -name '*.log'"})
# ❌ Command execution timeout (exceeded 30 seconds)
```

**네 번째 계층: 출력 크기 제한**

메모리 오버플로우를 방지하기 위해 명령 출력 크기를 제한합니다.

```python
terminal = TerminalTool(
    workspace="./project",
    max_output_size=10 * 1024 * 1024  # 10MB
)

# 출력이 10MB를 초과하는 경우
terminal.run({"command": "cat huge_file.log"})
# ... (콘텐츠의 처음 10MB 출력) ...
# ⚠️ Output truncated (exceeded 10485760 bytes)
```

이 네 가지 계층의 보안 메커니즘을 통해 TerminalTool은 강력한 기능을 제공함과 동시에 시스템의 보안을 극대화합니다.

### 9.5.2 핵심 기능 상세 설명

TerminalTool 구현은 명령 실행과 디렉토리 탐색(navigation)이라는 두 가지 핵심 기능에 초점을 맞춥니다.

(1) 명령 실행

핵심인 `_execute_command` 메서드는 명령의 실제 실행을 담당합니다.

```python
def _execute_command(self, command: str) -> str:
    """명령 실행"""
    try:
        # 현재 디렉토리에서 명령 실행
        result = subprocess.run(
            command,
            shell=True,
            cwd=str(self.current_dir),  # 현재 작업 디렉토리에서 실행
            capture_output=True,
            text=True,
            timeout=self.timeout,
            env=os.environ.copy()
        )

        # 표준 출력과 표준 에러 병합
        output = result.stdout
        if result.stderr:
            output += f"\n[stderr]\n{result.stderr}"

        # 출력 크기 체크
        if len(output) > self.max_output_size:
            output = output[:self.max_output_size]
            output += f"\n\n⚠️ Output truncated (exceeded {self.max_output_size} bytes)"

        # 리턴 코드 정보 추가
        if result.returncode != 0:
            output = f"⚠️ Command return code: {result.returncode}\n\n{output}"

        return output if output else "✅ Command executed successfully (no output)"

    except subprocess.TimeoutExpired:
        return f"❌ Command execution timeout (exceeded {self.timeout} seconds)"
    except Exception as e:
        return f"❌ Command execution failed: {e}"
```

이 구현의 핵심 포인트는 다음과 같습니다.

- **현재 디렉토리 인식**: `cwd` 파라미터를 사용하여 항상 올바른 디렉토리에서 명령이 수행되도록 합니다.
- **오류 처리**: 표준 에러를 캡처하고 병합하여 전체적인 진단 정보를 제공합니다.
- **리턴 코드 검사**: 0이 아닌 리턴 코드가 발생하는 경우 경고로 표시합니다.
- **결함 허용 설계(Fault-tolerant design)**: 타임아웃과 예외를 적절히 처리하여 에이전트 프로그램이 비정상 종료되지 않도록 방지합니다.

(2) 디렉토리 탐색

`cd` 명령에 대한 특수 처리는 파일 시스템에서 에이전트의 디렉토리 탐색을 지원합니다.

```python
def _handle_cd(self, parts: List[str]) -> str:
    """cd 명령 처리"""
    if not self.allow_cd:
        return "❌ cd command is disabled"

    if len(parts) < 2:
        # 매개변수가 없는 cd의 경우 현재 디렉토리 반환
        return f"Current directory: {self.current_dir}"

    target_dir = parts[1]

    # 상대 경로 처리
    if target_dir == "..":
        new_dir = self.current_dir.parent
    elif target_dir == ".":
        new_dir = self.current_dir
    elif target_dir == "~":
        new_dir = self.workspace
    else:
        new_dir = (self.current_dir / target_dir).resolve()

    # 작업 디렉토리 내에 있는지 확인
    try:
        new_dir.relative_to(self.workspace)
    except ValueError:
        return f"❌ Not allowed to access paths outside working directory: {new_dir}"

    # 디렉토리 존재 여부 확인
    if not new_dir.exists():
        return f"❌ Directory does not exist: {new_dir}"

    if not new_dir.is_dir():
        return f"❌ Not a directory: {new_dir}"

    # 현재 디렉토리 업데이트
    self.current_dir = new_dir
    return f"✅ Switched to directory: {self.current_dir}"
```

이 설계는 에이전트의 다단계 파일 시스템 탐색을 지원합니다.

```python
# 1단계: 프로젝트 구조 확인
terminal.run({"command": "ls -la"})

# 2단계: 소스 코드 디렉토리로 진입
terminal.run({"command": "cd src"})

# 3단계: 특정 파일 찾기
terminal.run({"command": "find . -name '*service*.py'"})

# 4단계: 파일 내용 확인
terminal.run({"command": "cat user_service.py"})
```

### 9.5.3 대표적인 사용 패턴

TerminalTool은 다양한 일반 파일 시스템 작업 패턴을 지원합니다.

(1) 탐색적 내비게이션(Exploratory Navigation)

에이전트는 전문 개발자처럼 프로젝트 코드베이스를 단계별로 탐색할 수 있습니다.

```python
from hello_agents.tools import TerminalTool

terminal = TerminalTool(workspace="./my_project")

# 1단계: 프로젝트 루트 디렉토리 확인
print(terminal.run({"command": "ls -la"}))
"""
total 24
drwxr-xr-x  6 user  staff   192 Jan 19 16:00 .
drwxr-xr-x  5 user  staff   160 Jan 19 15:30 ..
-rw-r--r--  1 user  staff  1234 Jan 19 15:30 README.md
drwxr-xr-x  4 user  staff   128 Jan 19 15:30 src
drwxr-xr-x  3 user  staff    96 Jan 19 15:30 tests
-rw-r--r--  1 user  staff   456 Jan 19 15:30 requirements.txt
"""

# 2단계: 소스 코드 디렉토리 구조 확인
terminal.run({"command": "cd src"})
print(terminal.run({"command": "tree"}))

# 3단계: 특정 패턴 검색
print(terminal.run({"command": "grep -r 'def process' ."}))
```

(2) 데이터 파일 분석

데이터 파일의 구조와 내용을 빠르게 이해할 수 있습니다.

```python
terminal = TerminalTool(workspace="./data")

# CSV 파일의 처음 몇 줄 확인
print(terminal.run({"command": "head -n 5 sales_2024.csv"}))
"""
date,product,quantity,revenue
2024-01-01,Widget A,150,4500.00
2024-01-01,Widget B,200,8000.00
2024-01-02,Widget A,180,5400.00
2024-01-02,Widget C,120,3600.00
"""

# 총 라인 수 계산
print(terminal.run({"command": "wc -l *.csv"}))
"""
  10234 sales_2024.csv
   8567 sales_2023.csv
  18801 total
"""

# 제품 카테고리 추출 및 개수 집계
print(terminal.run({"command": "tail -n +2 sales_2024.csv | cut -d',' -f2 | sort | uniq -c"}))
"""
  3456 Widget A
  4123 Widget B
  2655 Widget C
"""
```

(3) 로그 파일 분석

애플리케이션 로그의 실시간 분석을 통해 문제를 빠르게 찾아낼 수 있습니다.

```python
terminal = TerminalTool(workspace="/var/log")

# 최신 에러 로그 확인
print(terminal.run({"command": "tail -n 50 app.log | grep ERROR"}))

# 에러 유형 분포 집계
print(terminal.run({"command": "grep ERROR app.log | awk '{print $4}' | sort | uniq -c | sort -rn"}))
"""
  245 DatabaseConnectionError
  123 TimeoutException
   67 ValidationError
   34 AuthenticationError
"""

# 특정 시간대의 로그 검색
print(terminal.run({"command": "grep '2024-01-19 15:' app.log | tail -n 20"}))
```

(4) 코드베이스 분석

코드 리뷰 및 구조 파악을 지원합니다.

```python
terminal = TerminalTool(workspace="./codebase")

# 코드 라인 수 계산
print(terminal.run({"command": "find . -name '*.py' -exec wc -l {} + | tail -n 1"}))

# 모든 TODO 주석 검색
print(terminal.run({"command": "grep -rn 'TODO' --include='*.py'"}))

# 특정 함수의 정의 검색
print(terminal.run({"command": "grep -rn 'def process_data' --include='*.py'"}))

# 함수 구현부 보기
print(terminal.run({"command": "sed -n '/def process_data/,/^def /p' src/processor.py | head -n -1"}))
```

### 9.5.4 다른 도구와의 협업

TerminalTool의 진정한 강력함은 MemoryTool, NoteTool, ContextBuilder 등 다른 도구와 함께 유기적으로 사용할 때 발휘됩니다.

(1) MemoryTool과의 협업

TerminalTool을 통해 탐색된 핵심 정보는 메모리 시스템에 영구 저장될 수 있습니다.

```python
# TerminalTool을 사용하여 프로젝트 구조 탐색
structure = terminal.run({"command": "tree -L 2 src"})

# 시맨틱 메모리에 저장
memory_tool.run({
    "action": "add",
    "content": f"Project structure:\n{structure}",
    "memory_type": "semantic",
    "importance": 0.8,
    "metadata": {"type": "project_structure"}
})
```

(2) NoteTool과의 협업

중요한 발견 사항은 구조화된 노트로 즉시 기록될 수 있습니다.

```python
# 성능 병목 현상 발견
log_analysis = terminal.run({"command": "grep 'slow query' app.log | tail -n 10"})

# blocker 노트로 기록
note_tool.run({
    "action": "create",
    "title": "Database Slow Query Issue",
    "content": f"## 문제 설명\n시스템 성능에 영향을 주는 여러 슬로우 쿼리가 발견되었습니다.\n\n## 로그 분석\n```\n{log_analysis}\n```\n\n## 다음 단계 작업\n1. 슬로우 쿼리 SQL 분석\n2. 인덱스 추가\n3. 쿼리 로직 최적화",
    "note_type": "blocker",
    "tags": ["performance", "database"]
})
```

(3) ContextBuilder와의 협업

TerminalTool의 실시간 출력은 컨텍스트 생성 단계에서 다른 후보 정보들과 함께 포함될 수 있습니다.

```python
# 코드베이스 탐색
code_structure = terminal.run({"command": "ls -R src"})
recent_changes = terminal.run({"command": "git log --oneline -10"})

# ContextPacket으로 변환
from hello_agents.context import ContextPacket
from datetime import datetime

packets = [
    ContextPacket(
        content=f"Codebase structure:\n{code_structure}",
        timestamp=datetime.now(),
        token_count=len(code_structure) // 4,
        relevance_score=0.7,
        metadata={"type": "code_structure", "source": "terminal"}
    ),
    ContextPacket(
        content=f"Recent commits:\n{recent_changes}",
        timestamp=datetime.now(),
        token_count=len(recent_changes) // 4,
        relevance_score=0.8,
        metadata={"type": "git_history", "source": "terminal"}
    )
]

# 컨텍스트 빌드 시 해당 정보 포함
context = context_builder.build(
    user_query="How to refactor the user service module?",
    custom_packets=packets
)
```

## 9.6 실전 장기 실행 에이전트: 코드베이스 유지보수 어시스턴트

이제 ContextBuilder, NoteTool, TerminalTool을 하나로 결합하여 장기 실행(long-horizon)이 가능한 완성형 에이전트인 **코드베이스 유지보수 어시스턴트(Codebase Maintenance Assistant)**를 구축해 보겠습니다. 이 어시스턴트는 다음과 같은 기능을 수행할 수 있습니다.

1. 코드베이스 구조 탐색 및 이해
2. 발견된 문제점 및 개선 포인트 기록
3. 장기적인 리팩토링 태스크 추적
4. 컨텍스트 창 제한 내에서 대화의 일관성 유지

### 9.6.1 시나리오 설정 및 요구사항 분석

**비즈니스 시나리오**

약 50개의 Python 파일로 이루어진 중간 규모의 Python 웹 애플리케이션을 유지보수하고 있다고 가정해 보겠습니다. 이 코드베이스는 Flask 프레임워크로 작성되었으며 데이터 모델, 비즈니스 로직, API 인터페이스 등의 모듈을 포함하고 있고, 점진적으로 정리해 나가야 할 어느 정도의 기술 부채(technical debt)도 안고 있습니다. 이러한 상황에서 우리는 코드베이스를 탐색하고, 프로젝트 구조와 의존성 및 코드 스타일을 파악하며, 중복 코드나 과도한 복잡성, 테스트 부재 등의 문제점을 식별할 수 있는 똑똑한 어시스턴트가 필요합니다. 또한 어시스턴트는 할 일(to-do), 완료된 작업, blocker 등의 작업 진행 상황을 추적하고 기록하며, 과거의 컨텍스트를 토대로 일관성 있는 리팩토링 권장안을 제시해야 합니다.

**도전 과제 및 해결 방안**

이 시나리오는 몇 가지 전형적인 장기 실행 작업의 해결 과제들을 동반합니다. 첫째, 정보량이 컨텍스트 창 크기를 초과하는 문제입니다. 전체 코드베이스는 수만 줄에 달할 수 있으므로 한 번에 컨텍스트 창에 채워 넣을 수 없습니다. 우리는 이를 해결하기 위해 TerminalTool을 사용해 필요에 따라 실시간으로만 코드를 탐색하고 특정 파일만 열어보도록 설계했습니다. 둘째, 세션 간 상태 관리(cross-session state management) 문제입니다. 리팩토링 작업은 며칠 동안 지속될 수 있으므로 여러 세션에 걸쳐 진행 상황을 유지해야 합니다. 이를 위해 NoteTool을 사용하여 단계별 진행 상황, 할 일 목록, 주요 결정을 기록합니다. 마지막으로 컨텍스트의 품질과 관련성 문제입니다. 대화 단계마다 관련 있는 과거 정보를 복원하되 관련 없는 잡음(noise)에 묻히지 않아야 합니다. 이를 위해 ContextBuilder를 사용하여 컨텍스트를 지능적으로 필터링하고 조직화함으로써 신호 대 잡음비(signal density)를 극대화합니다.

### 9.6.2 시스템 아키텍처 설계

코드베이스 유지보수 어시스턴트는 그림 9.3에 표시된 것과 같이 3계층 아키텍처를 채택하고 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/9-figures/9-3.png" alt="" width="85%"/>
  <p>그림 9.3 코드베이스 유지보수 어시스턴트의 3계층 아키텍처</p>
</div>

### 9.6.3 핵심 구현

이제 이 시스템의 핵심 클래스를 구현해 보겠습니다.

```python
from typing import Dict, Any, List, Optional
from datetime import datetime
import json

from hello_agents import SimpleAgent, HelloAgentsLLM
from hello_agents.context import ContextBuilder, ContextConfig, ContextPacket
from hello_agents.tools import MemoryTool, NoteTool, TerminalTool
from hello_agents.core.message import Message


class CodebaseMaintainer:
    """코드베이스 유지보수 어시스턴트 - 장기 실행 에이전트 예시

    ContextBuilder + NoteTool + TerminalTool + MemoryTool 결합
    세션 간 코드베이스 유지보수 태스크 관리 구현
    """

    def __init__(
        self,
        project_name: str,
        codebase_path: str,
        llm: Optional[HelloAgentsLLM] = None
    ):
        self.project_name = project_name
        self.codebase_path = codebase_path
        self.session_id = f"session_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

        # LLM 초기화
        self.llm = llm or HelloAgentsLLM()

        # 도구 초기화
        self.memory_tool = MemoryTool(user_id=project_name)
        self.note_tool = NoteTool(workspace=f"./{project_name}_notes")
        self.terminal_tool = TerminalTool(workspace=codebase_path, timeout=60)

        # 컨텍스트 빌더 초기화
        self.context_builder = ContextBuilder(
            memory_tool=self.memory_tool,
            rag_tool=None,  # 본 예시에서는 RAG를 사용하지 않음
            config=ContextConfig(
                max_tokens=4000,
                reserve_ratio=0.15,
                min_relevance=0.2,
                enable_compression=True
            )
        )

        # 대화 기록
        self.conversation_history: List[Message] = []

        # 통계
        self.stats = {
            "session_start": datetime.now(),
            "commands_executed": 0,
            "notes_created": 0,
            "issues_found": 0
        }

        print(f"✅ 코드베이스 유지보수 에이전트가 초기화되었습니다: {project_name}")
        print(f"📁 작업 디렉터리: {codebase_path}")
        print(f"🆔 세션 ID: {self.session_id}")

    def run(self, user_input: str, mode: str = "auto") -> str:
        """에이전트 실행

        Args:
            user_input: 사용자 입력
            mode: 실행 모드
                - "auto": 도구 사용 여부를 자동으로 결정
                - "explore": 코드 탐색에 집중
                - "analyze": 문제 분석에 집중
                - "plan": 작업 계획에 집중

        Returns:
            str: 에이전트의 답변
        """
        print(f"\n{'='*80}")
        print(f"👤 사용자: {user_input}")
        print(f"{'='*80}\n")

        # 1단계: 모드에 따라 전처리 실행
        pre_context = self._preprocess_by_mode(user_input, mode)

        # 2단계: 관련 노트 검색
        relevant_notes = self._retrieve_relevant_notes(user_input)
        note_packets = self._notes_to_packets(relevant_notes)

        # 3단계: 최적화된 컨텍스트 빌드
        context = self.context_builder.build(
            user_query=user_input,
            conversation_history=self.conversation_history,
            system_instructions=self._build_system_instructions(mode),
            custom_packets=note_packets + pre_context
        )

        # 4단계: LLM 호출
        print("🤖 생각 중...")
        response = self.llm.invoke(context)

        # 5단계: 후처리
        self._postprocess_response(user_input, response)

        # 6단계: 대화 기록 업데이트
        self._update_history(user_input, response)

        print(f"\n🤖 에이전트: {response}\n")
        print(f"{'='*80}\n")

        return response

    def _preprocess_by_mode(
        self,
        user_input: str,
        mode: str
    ) -> List[ContextPacket]:
        """모드에 따라 전처리를 실행하고 관련 정보를 수집합니다."""
        packets = []

        if mode == "explore" or mode == "auto":
            # 탐색 모드: 프로젝트 구조를 자동으로 확인
            print("🔍 코드베이스 구조 탐색 중...")

            structure = self.terminal_tool.run({"command": "find . -type f -name '*.py' | head -n 20"})
            self.stats["commands_executed"] += 1

            packets.append(ContextPacket(
                content=f"[Codebase Structure]\n{structure}",
                timestamp=datetime.now(),
                token_count=len(structure) // 4,
                relevance_score=0.6,
                metadata={"type": "code_structure", "source": "terminal"}
            ))

        if mode == "analyze":
            # 분석 모드: 코드 복잡도 및 문제 확인
            print("📊 코드 품질 분석 중...")

            # 코드 라인 수 계산
            loc = self.terminal_tool.run({"command": "find . -name '*.py' -exec wc -l {} + | tail -n 1"})

            # TODO 및 FIXME 찾기
            todos = self.terminal_tool.run({"command": "grep -rn 'TODO\\|FIXME' --include='*.py' | head -n 10"})

            self.stats["commands_executed"] += 2

            packets.append(ContextPacket(
                content=f"[Code Statistics]\n{loc}\n\n[To-Do Items]\n{todos}",
                timestamp=datetime.now(),
                token_count=(len(loc) + len(todos)) // 4,
                relevance_score=0.7,
                metadata={"type": "code_analysis", "source": "terminal"}
            ))

        if mode == "plan":
            # 계획 모드: 최근 노트 로드
            print("📋 작업 계획 로드 중...")

            task_notes = self.note_tool.run({
                "action": "list",
                "note_type": "task_state",
                "limit": 3
            })

            if task_notes:
                content = "\n".join([f"- {note['title']}" for note in task_notes])
                packets.append(ContextPacket(
                    content=f"[Current Tasks]\n{content}",
                    timestamp=datetime.now(),
                    token_count=len(content) // 4,
                    relevance_score=0.8,
                    metadata={"type": "task_plan", "source": "notes"}
                ))

        return packets

    def _retrieve_relevant_notes(self, query: str, limit: int = 3) -> List[Dict]:
        """관련 노트 검색"""
        try:
            # 차단 요소(blockers) 검색 우선 처리
            blockers = self.note_tool.run({
                "action": "list",
                "note_type": "blocker",
                "limit": 2
            })

            # 관련 노트 검색
            search_results = self.note_tool.run({
                "action": "search",
                "query": query,
                "limit": limit
            })

            # 병합 및 중복 제거
            all_notes = {note.get('note_id') or note.get('id'): note for note in (blockers or []) + (search_results or [])}
            return list(all_notes.values())[:limit]

        except Exception as e:
            print(f"[WARNING] 노트 검색 실패: {e}")
            return []

    def _notes_to_packets(self, notes: List[Dict]) -> List[ContextPacket]:
        """노트를 컨텍스트 패킷으로 변환"""
        packets = []

        for note in notes:
            # 노트 타입에 따라 다른 관련성 점수 설정
            relevance_map = {
                "blocker": 0.9,
                "action": 0.8,
                "task_state": 0.75,
                "conclusion": 0.7
            }

            note_type = note.get('type', 'general')
            relevance = relevance_map.get(note_type, 0.6)

            content = f"[Note: {note.get('title', 'Untitled')}]\nType: {note_type}\n\n{note.get('content', '')}"

            packets.append(ContextPacket(
                content=content,
                timestamp=datetime.fromisoformat(note.get('updated_at', datetime.now().isoformat())),
                token_count=len(content) // 4,
                relevance_score=relevance,
                metadata={
                    "type": "note",
                    "note_type": note_type,
                    "note_id": note.get('note_id') or note.get('id')
                }
            ))

        return packets

    def _build_system_instructions(self, mode: str) -> str:
        """시스템 지침 구축"""
        base_instructions = f"""You are the codebase maintenance assistant for the {self.project_name} project.

Your core capabilities:
1. Use TerminalTool to explore codebase (ls, cat, grep, find, etc.)
2. Use NoteTool to record discoveries and tasks
3. Provide coherent recommendations based on historical notes

Current session ID: {self.session_id}
"""

        mode_specific = {
            "explore": """
Current mode: Explore codebase

You should:
- Actively use terminal commands to understand code structure
- Identify key modules and files
- Record project architecture in notes
""",
            "analyze": """
Current mode: Analyze code quality

You should:
- Find code issues (duplication, complexity, TODOs, etc.)
- Evaluate code quality
- Record discovered issues as blocker or action notes
""",
            "plan": """
Current mode: Task planning

You should:
- Review historical notes and tasks
- Formulate next action plan
- Update task status notes
""",
            "auto": """
Current mode: Auto decision

You should:
- Flexibly choose strategies based on user needs
- Use tools when needed
- Maintain professionalism and practicality in responses
"""
        }

        return base_instructions + mode_specific.get(mode, mode_specific["auto"])

    def _postprocess_response(self, user_input: str, response: str):
        """후처리: 답변을 분석하고 중요한 정보를 자동으로 기록합니다."""

        # 문제가 발견되면 자동으로 blocker 노트 생성
        if any(keyword in response.lower() for keyword in ["issue", "bug", "error", "blocker", "problem"]):
            try:
                self.note_tool.run({
                    "action": "create",
                    "title": f"Issue found: {user_input[:30]}...",
                    "content": f"## User Input\n{user_input}\n\n## Issue Analysis\n{response[:500]}...",
                    "note_type": "blocker",
                    "tags": [self.project_name, "auto_detected", self.session_id]
                })
                self.stats["notes_created"] += 1
                self.stats["issues_found"] += 1
                print("📝 이슈 노트가 자동으로 생성되었습니다.")
            except Exception as e:
                print(f"[WARNING] 노트 생성 실패: {e}")

        # 작업 계획을 세우는 경우 자동으로 action 노트 생성
        elif any(keyword in user_input.lower() for keyword in ["plan", "next", "task", "todo"]):
            try:
                self.note_tool.run({
                    "action": "create",
                    "title": f"Task planning: {user_input[:30]}...",
                    "content": f"## Discussion\n{user_input}\n\n## Action Plan\n{response[:500]}...",
                    "note_type": "action",
                    "tags": [self.project_name, "planning", self.session_id]
                })
                self.stats["notes_created"] += 1
                print("📝 실행 계획 노트가 자동으로 생성되었습니다.")
            except Exception as e:
                print(f"[WARNING] 노트 생성 실패: {e}")

    def _update_history(self, user_input: str, response: str):
        """대화 기록 업데이트"""
        self.conversation_history.append(
            Message(content=user_input, role="user", timestamp=datetime.now())
        )
        self.conversation_history.append(
            Message(content=response, role="assistant", timestamp=datetime.now())
        )

        # 기록 길이 제한 (최근 10라운드의 대화 유지)
        if len(self.conversation_history) > 20:
            self.conversation_history = self.conversation_history[-20:]

    # === 편의 메서드 ===

    def explore(self, target: str = ".") -> str:
        """코드베이스 탐색"""
        return self.run(f"Please explore the code structure of {target}", mode="explore")

    def analyze(self, focus: str = "") -> str:
        """코드 품질 분석"""
        query = f"Please analyze code quality" + (f", focusing on {focus}" if focus else "")
        return self.run(query, mode="analyze")

    def plan_next_steps(self) -> str:
        """다음 단계 계획"""
        return self.run("Based on current progress, plan next steps", mode="plan")

    def execute_command(self, command: str) -> str:
        """터미널 명령 실행"""
        result = self.terminal_tool.run({"command": command})
        self.stats["commands_executed"] += 1
        return result

    def create_note(
        self,
        title: str,
        content: str,
        note_type: str = "general",
        tags: List[str] = None
    ) -> str:
        """노트 생성"""
        result = self.note_tool.run({
            "action": "create",
            "title": title,
            "content": content,
            "note_type": note_type,
            "tags": tags or [self.project_name]
        })
        self.stats["notes_created"] += 1
        return result

    def get_stats(self) -> Dict[str, Any]:
        """통계 가져오기"""
        duration = (datetime.now() - self.stats["session_start"]).total_seconds()

        # 노트 요약 가져오기
        try:
            note_summary = self.note_tool.run({"action": "summary"})
        except:
            note_summary = {}

        return {
            "session_info": {
                "session_id": self.session_id,
                "project": self.project_name,
                "duration_seconds": duration
            },
            "activity": {
                "commands_executed": self.stats["commands_executed"],
                "notes_created": self.stats["notes_created"],
                "issues_found": self.stats["issues_found"]
            },
            "notes": note_summary
        }

    def generate_report(self, save_to_file: bool = True) -> Dict[str, Any]:
        """세션 보고서 생성"""
        report = self.get_stats()

        if save_to_file:
            report_file = f"maintainer_report_{self.session_id}.json"
            with open(report_file, 'w', encoding='utf-8') as f:
                json.dump(report, f, ensure_ascii=False, indent=2, default=str)
            report["report_file"] = report_file
            print(f"📄 보고서가 저장되었습니다: {report_file}")

        return report
```

### 9.6.4 전체 사용 예시

이제 완전한 사용 시나리오를 통해 이 장기(long-horizon) 에이전트의 워크플로우를 시연해 보겠습니다.

```python
# ========== 에이전트 초기화 ==========

from hello_agents import HelloAgentsLLM

maintainer = CodebaseMaintainer(
    project_name="my_flask_app",
    codebase_path="./my_flask_app",
    llm=HelloAgentsLLM()
)


# ========== 1일 차: 코드베이스 탐색 ==========
# 1. 초기 탐색
response = maintainer.explore()
"""
🔍 코드베이스 구조 탐색 중...
🤖 생각 중...

🤖 에이전트: 코드베이스를 스캔했습니다. 이 프로젝트는 Flask 웹 애플리케이션이며, 주로 다음과 같은 모듈을 포함하고 있습니다:

📁 프로젝트 구조:
- app/
  ├── models/      # 데이터 모델 (User, Product, Order 등)
  ├── routes/      # API 라우트
  ├── services/    # 비즈니스 로직
  └── utils/       # 유틸리티 함수
- tests/           # 테스트 파일
- migrations/      # 데이터베이스 마이그레이션
- config.py        # 설정 파일

총 약 3,500라인의 Python 코드입니다.

권장하는 다음 단계:
1. 핵심 모델 정의 검토
2. 라우트 및 API 디자인 확인
3. 테스트 커버리지 평가
"""

# 2. 특정 모듈 상세 분석
response = maintainer.run("Please analyze the data model design in the app/models/ directory")
"""
🤖 에이전트: app/models/ 디렉터리를 검토한 결과 다음과 같은 데이터 모델을 발견했습니다:

1. User (user.py)
   - 필드: id, username, email, password_hash, created_at
   - 관계: Order와 일대다(One-to-many) 관계
   - 문제점: ⚠️ 인덱스가 누락되었으며, email 필드에 고유(unique) 제약 조건이 설정되지 않았습니다.

2. Product (product.py)
   - 필드: id, name, description, price, stock
   - 상태: ✅ 잘 설계되었습니다.

3. Order (order.py)
   - 필드: id, user_id, product_id, quantity, status
   - 문제점: ⚠️ 생성 시간 필드가 누락되어 데이터 분석에 불리합니다.

권장 사항:
1. User.email에 고유 제약 조건 추가
2. Order에 created_at 및 updated_at 필드 추가
3. 소프트 삭제(soft delete) 지원 고려

이러한 문제들을 차단 요소(blocker) 노트로 기록했습니다.
"""
# [차단 요소(blocker) 노트 자동 생성됨]


# ========== 2일 차: 코드 품질 분석 ==========

# 1. 전반적인 품질 분석
response = maintainer.analyze()
"""
📊 코드 품질 분석 중...
🤖 생각 중...

🤖 에이전트: 코드 품질 분석 보고서:

📊 코드 통계:
- 총 라인 수: 3,542라인
- 평균 파일 길이: 약 150라인
- 가장 큰 파일: services/order_service.py (456라인)

⚠️ 발견된 문제점:

1. 코드 중복 (높음):
   - user_service.py와 product_service.py에 유사한 CRUD 로직이 존재함
   - 권장 사항: BaseService 기본 클래스 추출

2. 과도한 복잡도 (보통):
   - order_service.py::process_order 메서드의 중첩 수준이 8단계에 달함
   - 권장 사항: 여러 개의 작은 메서드로 분할

3. 테스트 부족 (높음):
   - 테스트 커버리지가 45%에 불과함
   - services/ 디렉터리에 테스트가 전혀 없음

4. TODO/FIXME (12개 항목):
   - 대부분 "나중에 최적화" 유형의 주석임
   - 권장 사항: 정리하거나 공식 작업으로 변환

이러한 문제들을 노트로 기록하고 우선순위를 표시했습니다.
"""
# [여러 개의 차단 요소(blocker) 노트 자동 생성됨]

# 2. 특정 문제 검토
response = maintainer.run("Please review the process_order method in order_service.py and provide refactoring recommendations")
"""
🤖 에이전트: process_order 메서드의 코드를 검토했습니다:

【기존 코드】
def process_order(self, order_id):
    order = Order.query.get(order_id)
    if order:
        if order.status == 'pending':
            product = Product.query.get(order.product_id)
            if product:
                if product.stock >= order.quantity:
                    # ... 8단계 중첩 계속 ...

【문제 분석】
1. 깊은 중첩으로 인한 가독성 저하
2. 에러 처리 부재
3. 비즈니스 로직과 데이터 액세스의 혼재

【리팩터링 권장 사항】
def process_order(self, order_id):
    # 1. 조기 반환(Early return) 패턴 적용, 중첩 축소
    order = self._get_order(order_id)
    if not order or order.status != 'pending':
        return False

    # 2. 메서드 추출, 단일 책임 원칙 준수
    if not self._check_stock(order):
        return False

    # 3. 일관성 확보를 위해 트랜잭션 사용
    with db.transaction():
        self._update_stock(order)
        self._update_order_status(order, 'processing')

    return True

【다음 단계】
1. 리팩터링 계획 노트를 생성했습니다.
2. 먼저 테스트 케이스를 작성하는 것을 권장합니다.
3. 기능을 유지하면서 점진적으로 리팩터링을 진행합니다.
"""
# [실행 계획(action) 노트 자동 생성됨]


# ========== 3일 차: 리팩터링 작업 계획 ==========

# 1. 진행 상황 검토
response = maintainer.plan_next_steps()
"""
📋 작업 계획 로드 중...
🤖 생각 중...

🤖 에이전트: 지난 이틀 동안의 탐색 및 분석을 바탕으로 리팩터링 작업 우선순위를 정리했습니다:

🔴 높은 우선순위 (이번 주 완료 목표):
1. [blocker] User.email에 고유 제약 조건 추가
   - 위험도: 낮음, 단순 데이터베이스 마이그레이션
   - 소요 시간: 1시간

2. [blocker] 코드 중복 감소를 위해 BaseService 추출
   - 위험도: 보통, 여러 서비스 클래스에 영향
   - 소요 시간: 반나절

3. [blocker] process_order 메서드 리팩터링
   - 위험도: 높음, 핵심 비즈니스 로직
   - 소요 시간: 1일 (테스트 작성 포함)

🟡 중간 우선순위 (다음 주 목표):
4. services/ 디렉터리에 대한 단위 테스트 추가
5. TODO 주석 정리
6. Order에 시간 필드 추가

🟢 낮은 우선순위 (추후 결정):
7. 성능 최적화
8. 문서 업데이트

권장 사항:
- 이번 주에는 먼저 데이터 모델 문제(작업 1, 6)를 해결합니다.
- 다음 주에는 코드 리팩터링(작업 2, 3)을 진행합니다.
- 테스트 커버리지를 지속적으로 높여 나갑니다(작업 4).

작업 상태 노트를 업데이트했으므로 언제든 진행 상황을 확인할 수 있습니다.
"""

# 2. 상세 리팩터링 계획 수동 생성
maintainer.create_note(
    title="주간 리팩터링 계획 - 1주 차",
    content="""## 목표
데이터 모델 레이어 최적화 완료

## 작업 체크리스트
- [ ] User.email에 고유 제약 조건 추가
- [ ] Order에 created_at, updated_at 필드 추가
- [ ] 데이터베이스 마이그레이션 스크립트 작성
- [ ] 관련 테스트 케이스 업데이트

## 일정
- 월요일: 마이그레이션 스크립트 설계
- 화요일~수요일: 마이그레이션 실행 및 테스트
- 목요일: 테스트 케이스 업데이트
- 금요일: 코드 리뷰(Code Review)

## 위험 요소
- 데이터베이스 마이그레이션이 운영 환경에 영향을 미칠 수 있으므로 트래픽이 적은 시간대에 실행해야 함
- 기존 데이터에 중복된 이메일이 존재할 수 있으므로 먼저 정리가 필요함
""",
    note_type="task_state",
    tags=["refactoring", "week1", "high_priority"]
)

print("✅ 상세 리팩터링 계획이 생성되었습니다.")


# ========== 일주일 후: 진행 상황 확인 ==========

# 노트 요약 보기
summary = maintainer.note_tool.run({"action": "summary"})
print("📊 노트 요약:")
print(json.dumps(summary, indent=2, ensure_ascii=False))
"""
{
  "total_notes": 8,
  "type_distribution": {
    "blocker": 3,
    "action": 2,
    "task_state": 2,
    "conclusion": 1
  },
  "recent_notes": [
    {
      "id": "note_20250119_160000_7",
      "title": "Weekly Refactoring Plan - Week 1",
      "type": "task_state",
      "updated_at": "2025-01-19T16:00:00"
    },
    ...
  ]
}
"""

# 전체 보고서 생성
report = maintainer.generate_report()
print("\n📄 세션 보고서:")
print(json.dumps(report, indent=2, ensure_ascii=False))
"""
{
  "session_info": {
    "session_id": "session_20250119_150000",
    "project": "my_flask_app",
    "duration_seconds": 172800  # 2일
  },
  "activity": {
    "commands_executed": 24,
    "notes_created": 8,
    "issues_found": 3
  },
  "notes": { ... }
}
"""
```

### 9.6.5 실행 효과 분석

이 완전한 사례 연구를 통해 우리는 장기(long-horizon) 에이전트의 몇 가지 주요 특징을 볼 수 있습니다. 첫째는 **세션 간 일관성(cross-session coherence)**입니다. 에이전트는 `NoteTool`을 통해 여러 날과 세션에 걸쳐 작업의 일관성을 유지합니다. 1일 차에 탐색한 문제는 2일 차 분석 시 자동으로 고려되고, 3일 차 계획 수립 단계에서는 이전 이틀간의 모든 발견 사항을 종합할 수 있으며, 일주일 후 확인 시 전체 기록이 그대로 보존됩니다. 둘째는 **지능적인 컨텍스트 관리**입니다. `ContextBuilder`는 각 대화에 대한 고품질 컨텍스트를 보장하며, 관련 노트(특히 blocker 유형)를 자동으로 수집하고 대화 모드에 따라 전처리 전략을 동적으로 조정하며 토큰 예산 내에서 가장 관련성 높은 정보를 선택합니다.

셋째는 **즉각적인 파일 시스템 액세스**입니다. `TerminalTool`은 전체 코드베이스의 사전 인덱싱 없이도 유연한 코드 탐색을 지원하고, 특정 파일의 콘텐츠를 즉시 볼 수 있으며, 복잡한 텍스트 처리(grep, awk 등)를 지원합니다. 넷째는 **자동화된 지식 관리**입니다. 시스템은 발견된 지식을 자동으로 관리하여, 문제가 발견되면 자동으로 blocker 노트를 생성하고 계획을 논의할 때는 자동으로 action 노트를 생성하며 메모리 시스템에 핵심 정보를 자동으로 저장합니다. 마지막으로 **인간-기계 협업**입니다. 이 시스템은 유연한 인간-기계 협업 모드를 지원하여 에이전트가 탐색과 분석을 자동으로 완료할 수 있도록 하고, 인간은 노트 시스템을 통해 개입하여 가이드할 수 있으며, 상세한 계획 노트를 수동으로 작성하는 것도 지원합니다.

이러한 기본 프레임워크는 코드베이스에 대한 벡터 인덱스를 구축하고 시맨틱 검색을 결합하기 위해 `RAGTool`을 통합하거나, 다중 에이전트 협업을 구현하기 위해 특화된 탐색기(explorer), 분석기(analyzer), 계획기(planner)로 분할하거나, 리팩터링 결과를 자동으로 검증하기 위해 테스트 도구를 통합하거나, 코드 변경 사항을 추적하기 위해 `TerminalTool`을 통해 git 명령을 실행하거나, Gradio/Streamlit을 사용하여 시각적 인터페이스를 구축하는 등 다양하게 확장할 수 있습니다.

## 9.7 단원 요약

이 장에서 우리는 컨텍스트 엔지니어링의 이론적 기초와 엔지니어링 실무에 대해 깊이 있게 탐구했습니다.

### 이론적 차원

1. **컨텍스트 엔지니어링의 본질**: "프롬프트 엔지니어링"에서 "컨텍스트 엔지니어링"으로의 진화이며, 그 핵심은 제한된 주의력 예산(attention budget)을 관리하는 데 있습니다.
2. **컨텍스트 부패(Context Rot)**: 긴 컨텍스트가 초래하는 성능 저하를 이해하고 컨텍스트를 희소한 자원으로 인식하는 것입니다.
3. **3가지 주요 전략**: 압축(Compaction), 구조화된 노트 작성(Structured note-taking), 서브 에이전트 아키텍처(Sub-agent architectures)입니다.

### 엔지니어링 실무

1. **ContextBuilder**: GSSC 파이프라인을 구현하여 통일된 컨텍스트 관리 인터페이스를 제공합니다.
2. **NoteTool**: Markdown+YAML의 하이브리드 형식으로 구조화된 장기 메모리를 지원합니다.
3. **TerminalTool**: 안전한 명령줄 도구로, 파일 시스템에 대한 즉각적인 액세스를 지원합니다.
4. **장기(Long-Horizon) 에이전트**: 3대 도구를 통합하여 세션 간 일관성을 유지하는 코드베이스 유지보수 에이전트를 구축합니다.

### 핵심 요약

- **계층적 설계**: 즉각적인 액세스(TerminalTool) + 세션 메모리(MemoryTool) + 영구 노트(NoteTool)
- **지능형 필터링**: 관련성 및 최신성에 기반한 점수 부여 매커니즘
- **보안 최우선**: 다중 계층 보안 매커니즘을 통해 시스템 안정성 확보
- **인간-기계 협업**: 자동화와 제어 가능성 사이의 균형

본 단원의 학습을 통해 여러분은 컨텍스트 엔지니어링의 핵심 기술을 마스터했을 뿐만 아니라, 더 중요하게는 긴 시간 범위에 걸쳐 일관성과 효과를 유지할 수 있는 에이전트 시스템을 구축하는 방법을 이해하게 되었습니다. 이러한 기술은 프로덕션 급의 에이전트 애플리케이션을 빌드하기 위한 중요한 토대가 될 것입니다.

다음 단원에서는 에이전트 통신 프로토콜을 살펴보고, 에이전트가 외부 세계와 더 광범위하게 상호작용하는 방법을 배울 것입니다.

## 연습 문제

> **참고**: 일부 연습 문제에는 표준 답안이 없습니다. 학습자가 컨텍스트 엔지니어링과 장기(long-horizon) 작업 관리에 대한 종합적인 이해와 실천 능력을 함양하는 데 중점을 둡니다.

1. 이 장에서는 컨텍스트 엔지니어링과 프롬프트 엔지니어링의 차이점을 소개했습니다. 다음을 분석해 보세요:

   - 9.1절에서는 "컨텍스트는 한계 수확 체감의 법칙이 적용되는 제한된 자원으로 보아야 한다"고 언급했습니다. **컨텍스트 부패(context rot)** 현상이란 무엇인지 설명해 보세요. 모델이 100K 또는 200K의 대형 컨텍스트 창을 지원하더라도 컨텍스트를 세심하게 관리해야 하는 이유는 무엇일까요?
   - 50개의 파일을 포함하는 코드베이스를 분석해야 하는 "코드 리뷰 에이전트"를 구축하려고 한다고 가정해 봅시다. 다음 두 가지 전략을 비교해 보세요: (1) 모든 파일의 콘텐츠를 한 번에 컨텍스트에 로드하는 전략, (2) 도구를 통해 필요할 때만 파일을 가져오는 JIT(Just-In-Time) 컨텍스트 전략. 각각의 장단점과 적합한 시나리오를 분석해 보세요.
   - 9.2.1절에서는 시스템 프롬프트의 두 가지 극단적인 위험 요소로 "과도한 하드코딩(over-hardcoding)"과 "지나치게 모호함(too vague)"을 들었습니다. 각각의 실제 사례를 들고 어떻게 적절한 균형을 찾을 수 있을지 설명해 보세요.

2. GSSC(Gather-Select-Structure-Compress) 파이프라인은 이 장의 핵심 기술입니다. 깊이 있게 고민해 보세요:

   > **참고**: 이 문제는 실습 문제이므로 실제 작업을 수행해 보는 것이 좋습니다.

   - 9.3절의 `ContextBuilder` 구현에서 네 단계는 각각 다른 책임을 집니다. 만약 특정 단계가 실패할 경우(예를 들어 Select 단계에서 무관한 정보를 선택하거나, Compress 단계에서 과도한 압축으로 인해 정보 유실이 발생할 경우), 최종 에이전트의 성능에 어떤 영향을 미칠지 분석해 보세요.
   - 9.3.4절의 코드를 바탕으로 `ContextBuilder`에 "컨텍스트 품질 평가" 기능을 추가해 보세요. 컨텍스트가 빌드될 때마다 정보 밀도, 관련성, 완전성을 자동으로 평가하고 최적화 제안을 제공하도록 합니다.
   - GSSC 파이프라인의 "압축(compress)" 단계에서는 지능형 요약을 위해 LLM을 사용합니다. 어떤 상황에서 단순 잘라내기(truncation)나 슬라이딩 윈도우(sliding window) 전략이 LLM 요약보다 더 적절할지 고민해 보세요. 여러 압축 방법의 장점을 결합한 하이브리드 압축 전략을 설계해 보세요.

3. `NoteTool`과 `TerminalTool`은 장기 작업을 지원하는 핵심 도구입니다. 9.4절과 9.5절을 바탕으로 다음 확장 실습을 완료해 보세요:

   > **참고**: 이 문제는 실습 문제이므로 실제 작업을 수행해 받는 것이 좋습니다.

   - `NoteTool`은 계층적 노트 시스템(프로젝트 노트, 작업 노트, 임시 노트)을 사용합니다. "자동 노트 정리" 메커니즘을 설계해 보세요. 임시 노트가 일정 개수 이상 쌓이면 에이전트가 이를 자동으로 분석하여 중요 정보를 작업 노트나 프로젝트 노트로 승격시키고, 중복된 콘텐츠를 정리하도록 합니다.
   - `TerminalTool`은 파일 시스템 조작 기능을 제공하지만, 9.5.2절에서는 보안 설계를 강조합니다. 현재의 보안 메커니즘(경로 유효성 검사, 명령어 화이트리스트, 권한 확인)이 충분한지 분석해 보세요. 에이전트가 민감한 파일에 액세스하거나 위험한 명령을 실행해야 할 경우, "인간-기계 협업 승인" 프로세스를 어떻게 설계해야 할까요?
   - `NoteTool`과 `TerminalTool`을 결합하여 "지능형 코드 리팩터링 에이전트"를 설계해 보세요. 이 에이전트는 코드베이스 구조를 분석하고 리팩터링 계획을 기록하며 리팩터링 작업을 단계별로 실행하고 진행 상황과 발견된 문제를 노트에 추적할 수 있어야 합니다. 전체 워크플로우 다이어그램을 그려 보세요.

4. 9.6절 of "장기 작업 관리" 사례를 통해 실제 애플리케이션에서 컨텍스트 엔지니어링의 가치를 보았습니다. 이에 대해 깊이 분석해 보세요:

   - 이 사례는 "계층적 컨텍스트 관리" 전략을 사용합니다: 즉각적인 액세스(`TerminalTool`) + 세션 메모리(`MemoryTool`) + 영구 노트(`NoteTool`). 이 세 계층이 어떻게 조화를 이루어야 할지 분석해 보세요. 어떤 정보를 어떤 계층에 배치해야 할까요? 정보의 중복 및 불일치를 방지하려면 어떻게 해야 할까요?
   - 작업 실행 도중 중단(예: 시스템 다운, 네트워크 연결 해제)이 발생하여 에이전트가 노트에서 상태를 복구하고 실행을 이어가야 한다고 가정해 봅시다. "중단점부터 재개(resume from breakpoint)" 메커니즘을 설계해 보세요. 노트에 충분한 상태 정보를 어떻게 기록할 수 있을까요? 복구된 상태가 올바른지 어떻게 검증할 수 있을까요?
   - 장기 작업은 흔히 여러 서브 태스크의 병렬 또는 직렬 실행을 포함합니다. "작업 의존성 관리" 시스템을 설계해 보세요. 이 시스템은 작업 간의 의존 관계(예: "작업 B는 작업 A가 완료된 후에 실행되어야 함")를 표현하고 작업 실행 순서를 자동으로 스케줄링할 수 있어야 합니다. 이 시스템을 `NoteTool`과 어떻게 통합해야 할까요?

5. 이 장에서는 "점진적 공개(progressive disclosure)" 개념이 여러 번 언급되었습니다. 생각해 보세요:

   - 9.2.2절에서 점진적 공개는 "각 상호작용 단계가 새로운 컨텍스트를 생성하며, 이는 다시 다음 의사결정을 안내한다"고 설명했습니다. 점진적 공개가 에이전트의 효율적인 작업 완료에 어떻게 도움이 되는지 보여주는 구체적인 애플리케이션 시나리오(예: 학술 논문 작성, 복잡한 문제 디버깅)를 설계해 보세요.
   - 점진적 공개의 잠재적 위험은 "비효율적인 탐색"입니다. 에이전트가 중요하지 않은 세부 사항에 시간을 낭비하거나 핵심 정보를 놓칠 수 있습니다. 휴리스틱 규칙이나 메타인지(metacognitive) 전략을 통해 에이전트가 "다음 탐색 대상"에 대해 더 스마트한 의사결정을 내릴 수 있도록 돕는 "탐색 가이드" 메커니즘을 설계해 보세요.
   - "점진적 공개"와 기존의 "모든 컨텍스트를 한 번에 로드하기"를 비교해 보세요. 전자가 명확한 우위를 점하는 작업 유형은 무엇일까요? 후자가 더 적절한 작업 유형은 무엇일까요? 서로 다른 유형의 작업에 대한 예시를 최소 3가지 이상 제시해 보세요.

## 참고 문헌

[1] Anthropic. Effective Context Engineering for AI Agents. `https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents`

[2] David Kim. Context-Engineering (GitHub). `https://github.com/davidkimai/Context-Engineering`

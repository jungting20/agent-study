# 5장: 로우코드 플랫폼으로 에이전트 구축하기

이전 장에서는 Python 코드를 작성하여 ReAct, Plan-and-Solve, Reflection 등 다양한 클래식 에이전트 워크플로를 처음부터 구현했습니다. 이 과정은 견고한 기술적 기반을 마련해 주었고, 에이전트의 내부 메커니즘을 깊이 이해할 수 있게 해 주었습니다. 그러나 빠르게 발전하는 분야에서 순수 코드 개발이 항상 가장 효율적인 선택은 아닙니다. 특히 아이디어를 빠르게 검증해야 하거나, 비전문 개발자가 구축에 참여해야 하는 시나리오에서는 더욱 그렇습니다.

## 5.1 플랫폼 기반 구축의 부상

기술이 성숙해지면서 점점 더 많은 역량이 "플랫폼화"되고 있습니다. 웹사이트 개발이 HTML/CSS/JS를 직접 작성하는 방식에서 WordPress, Wix 같은 웹사이트 빌더 플랫폼을 사용하는 방식으로 진화한 것과 같이, 에이전트 구축도 플랫폼화의 물결을 맞이했습니다. 이 장에서는 그래픽 기반의 모듈형 로우코드 플랫폼을 사용해 에이전트 애플리케이션을 빠르고 직관적으로 구축·디버깅·배포하는 방법에 집중하며, 관심사를 "구현 세부사항"에서 "비즈니스 로직"으로 옮깁니다.

### 5.1.1 로우코드 플랫폼이 필요한 이유

"바퀴를 다시 발명하는" 것은 깊은 학습에 매우 중요하지만, 엔지니어링 효율과 혁신을 추구하는 실무에서는 종종 거인의 어깨 위에 서야 합니다. 4장에서 `ReActAgent`, `PlanAndSolveAgent` 같은 재사용 가능한 클래스를 캡슐화했지만, 비즈니스 로직이 복잡해지면 순수 코드의 유지보수 비용과 개발 주기가 급격히 늘어납니다. 로우코드 플랫폼의 등장은 바로 이러한 pain point를 해결하기 위한 것입니다.

핵심 가치는 주로 다음 측면에서 드러납니다.

1. **기술 장벽 낮추기**: 로우코드 플랫폼은 API 호출, 상태 관리, 동시성 제어 같은 복잡한 기술 세부사항을 이해하기 쉬운 "노드" 또는 "모듈"로 캡슐화합니다. 사용자는 프로그래밍에 능숙할 필요 없이, 노드를 드래그하여 연결하기만 하면 강력한 워크플로를 구축할 수 있습니다. 이를 통해 PM, 디자이너, 도메인 전문가 같은 비기술 인력도 에이전트 설계와 제작에 참여할 수 있어 혁신의 경계가 크게 넓어집니다.
2. **개발 효율 향상**: 전문 개발자에게도 플랫폼은 큰 효율 향상을 가져올 수 있습니다. 프로젝트 초기에 아이디어를 빠르게 검증하거나 프로토타입을 만들어야 할 때, 로우코드 플랫폼을 사용하면 원래 며칠 걸릴 코딩 작업을 몇 시간, 심지어 몇 분 만에 끝낼 수 있습니다. 개발자는 저수준 엔지니어링 구현보다 비즈니스 로직 구성과 프롬프트 엔지니어링 최적화에 더 많은 에너지를 쓸 수 있습니다.
3. **더 나은 시각화와 관측 가능성**: 터미널에 로그를 출력하는 방식과 비교하면, 그래픽 플랫폼은 에이전트 실행 궤적을 end-to-end로 자연스럽게 시각화합니다. 각 노드 사이에서 데이터가 어떻게 흐르는지, 어느 구간이 가장 오래 걸리는지, 어떤 도구 호출이 실패하는지 명확히 볼 수 있습니다. 이런 직관적인 디버깅 경험은 순수 코드 개발과 비교할 수 없습니다.
4. **표준화와 모범 사례 축적**: 우수한 로우코드 플랫폼에는 보통 많은 업계 모범 사례가 내장되어 있습니다. 예를 들어 사전 구성된 ReAct 템플릿, 최적화된 지식 베이스 검색 엔진, 표준화된 도구 통합 규격 등이 있습니다. 이는 개발자가 "함정"에 빠지는 것을 막을 뿐 아니라, 모두가 같은 표준과 컴포넌트를 기반으로 개발하기 때문에 팀 협업도 더 원활해집니다.

요약하면, 로우코드 플랫폼은 코드를 대체하려는 것이 아니라 더 높은 수준의 추상화를 제공합니다. 지루한 저수준 구현에서 벗어나 에이전트의 "사고"와 "행동" 로직 자체에 더 집중할 수 있게 해 주며, 아이디어를 더 빠르고 더 잘 현실로 옮길 수 있습니다.

### 5.1.2 로우코드 플랫폼 선택하기

현재 에이전트 및 LLM 애플리케이션용 로우코드 플랫폼 시장은 각 플랫폼마다 고유한 포지셔닝과 장점을 가진 채로 활발히 성장하고 있습니다. 어떤 플랫폼을 선택할지는 핵심 요구사항, 기술 배경, 프로젝트의 최종 목표에 따라 달라지는 경우가 많습니다. 이후 내용에서는 Coze, Dify, n8n 세 가지 대표 플랫폼을 중심으로 소개하고 실습합니다. 그에 앞서 각 플랫폼을 간단히 살펴보겠습니다.

**Coze**

- **핵심 포지셔닝**: ByteDance가 출시한 Coze<sup>[1]</sup>는 제로코드/로우코드 Agent 구축 경험에 집중하여, 프로그래밍 배경이 없는 사용자도 쉽게 만들 수 있게 합니다.
- **기능 분석**: Coze는 매우 친숙한 시각적 인터페이스를 갖추고 있습니다. 사용자는 플러그인을 드래그 앤 드롭하고, 지식 베이스를 구성하고, 워크플로를 설정하는 방식으로 레고 블록을 조립하듯 에이전트를 만들 수 있습니다. 풍부한 내장 플러그인 라이브러리를 갖추고 있으며, Douyin, Feishu, WeChat Official Accounts 등 주류 플랫폼에 원클릭 배포를 지원해 배포 과정을 크게 단순화합니다.
- **대상 사용자**: AI 애플리케이션 입문 사용자, PM, 운영 담당자, 아이디어를 빠르게 인터랙티브 제품으로 바꾸고 싶은 개인 창작자.

**Dify**

- **핵심 포지셔닝**: Dify는 오픈소스의 풀기능 LLM 애플리케이션 개발·운영 플랫폼<sup>[2]</sup>으로, 프로토타입 구축부터 프로덕션 배포까지 원스톱 솔루션을 제공하는 것을 목표로 합니다.
- **기능 분석**: 백엔드 서비스와 모델 운영 개념을 통합하여 Agent 워크플로, RAG Pipeline, 데이터 어노테이션, 파인튜닝 등 다양한 역량을 지원합니다. 전문성, 안정성, 확장성을 추구하는 엔터프라이즈급 애플리케이션에 견고한 기반을 제공합니다.
- **대상 사용자**: 어느 정도 기술 배경이 있는 개발자, 확장 가능한 엔터프라이즈급 AI 애플리케이션을 구축해야 하는 팀.

**n8n**

- **핵심 포지셔닝**: n8n은 본질적으로 오픈소스 워크플로 자동화 도구<sup>[3]</sup>이며, 순수 LLM 플랫폼은 아닙니다. 최근 몇 년간 AI 역량을 적극적으로 통합하고 있습니다.

- **기능 분석**: n8n의 강점은 "연결"에 있습니다. 수백 개의 사전 구성 노드로 다양한 SaaS 서비스, 데이터베이스, API를 복잡한 자동화 비즈니스 프로세스로 쉽게 연결할 수 있습니다. 이 과정에 LLM 노드를 끼워 넣어 전체 자동화 체인의 일부로 만들 수 있습니다. LLM 기능만큼 전문화되어 있지는 않지만, 범용 자동화 역량은 독특합니다. 다만 학습 곡선도 상대적으로 가파른 편입니다.

- **대상 사용자**: AI 역량을 기존 비즈니스 프로세스에 깊이 통합하고, 고도로 맞춤화된 자동화를 구현해야 하는 개발자와 기업.

이후 소절에서는 이 플랫폼들을 하나씩 직접 다루며, 실제 조작을 통해 각각의 매력을 더 직관적으로 느껴 보겠습니다.

## 5.2 플랫폼 1: Coze

Coze는 정말 멋진 AI 에이전트 제작 도구입니다. 현재 시장에서 가장 널리 쓰이는 에이전트 플랫폼이기도 합니다. 직관적인 시각적 인터페이스와 풍부한 기능 모듈 덕분에, 대화할 수 있는 챗봇, 이야기를 자동으로 쓰는 창작 도구, 심지어 이야기를 영화 MV로 바꿔 주는 애플리케이션까지 다양한 유형의 에이전트를 쉽게 만들 수 있습니다. 특히 강력한 생태계 통합 역량이 하이라이트입니다. 개발한 에이전트는 WeChat, Feishu, Doubao 등 주류 플랫폼에 원클릭으로 배포할 수 있어 크로스 플랫폼 배포가 매끄럽습니다. 엔터프라이즈 사용자를 위해 Coze는 유연한 API 인터페이스도 제공하여, 에이전트 역량을 기존 비즈니스 시스템에 통합하는 "블록 조립식" AI 애플리케이션 구축을 지원합니다.

### 5.2.1 Coze의 기능 모듈

(1) 플랫폼 인터페이스 개요

전체 레이아웃 소개: 최근 Coze는 UI를 다시 업데이트했으며, 그림 5.1과 같습니다. 가장 왼쪽 사이드바는 Coze 플랫폼 홈페이지의 개발 워크스페이스로, 핵심 프로젝트 개발, 리소스 라이브러리, 효과 평가, 스페이스 설정을 포함합니다. 아래 영역은 Coze 개발을 위한 보조 자료 공간으로, 원클릭 복사용 공식 템플릿, Coze의 가장 큰 강점인 풍부한 플러그인 스토어, 다양한 에이전트가 모인 최대 규모의 에이전트 커뮤니티, API 테스트용 API 관리, 상세 튜토리얼 문서, 엔터프라이즈용 일반 관리 기능이 있습니다. 오른쪽에는 네 가지 템플릿이 있고, 상단에는 Coze 최신 업데이트 공지가 있어 최신 도구와 기능을 알 수 있습니다. 그 아래는 초보자 튜토리얼로, 클릭하면 초보자 튜토리얼 문서를 볼 수 있고 몇 분 만에 에이전트 구축을 시작할 수 있습니다. 다음은 팔로우와 에이전트 추천 영역으로, 좋아하는 AI 개발자를 팔로우하고 에이전트를 북마크해 자신만의 용도로 쓸 수 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-01.png" alt="Image description" width="90%"/>
  <p>그림 5.1 Coze Agent 플랫폼 전체 개요</p>
</div>

(2) 핵심 기능 소개

먼저 왼쪽 사이드바의 플러스 버튼을 클릭하면 에이전트 생성 진입점을 볼 수 있습니다. 현재 AI 애플리케이션은 두 가지 유형입니다. 하나는 에이전트 생성, 다른 하나는 애플리케이션입니다. 에이전트는 단일 에이전트 자율 계획 모드, 단일 에이전트 대화 플로우 모드, 멀티 에이전트 모드로 나뉩니다. AI 애플리케이션도 두 가지로, 데스크톱·웹용 UI를 설계할 수 있을 뿐 아니라 미니프로그램·H5용 인터페이스도 쉽게 만들 수 있습니다. 그림 5.2 참고.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-02.png" alt="Image description" width="90%"/>
  <p>그림 5.2 Coze Agent 생성 진입점</p>
</div>

프로젝트 스페이스는 에이전트 저장소입니다. 개발했거나 복사한 모든 에이전트·애플리케이션이 저장되며, Coze에서 에이전트를 개발할 때 가장 자주 방문하는 공간입니다. 그림 5.3.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-03.png" alt="Image description" width="90%"/>
  <p>그림 5.3 Coze Agent 프로젝트 스페이스</p>
</div>

리소스 라이브러리는 Coze 에이전트 개발의 핵심 무기고입니다. 워크플로, 지식 베이스, 카드, 프롬프트 라이브러리 등 에이전트 개발 도구가 저장됩니다. 어떤 에이전트를 만들 수 있는지는 먼저 모델 역량에 달려 있지만, 더 중요한 것은 에이전트에 어떤 "장비와 스킬"을 장착하느냐입니다. 모델은 에이전트의 하한을 정하지만, Coze 리소스 라이브러리는 에이전트 역량의 상한을 사실상 무한히 열어 줍니다. 그림 5.4.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-04.png" alt="Image description" width="90%"/>
  <p>그림 5.4 Coze Agent 리소스 라이브러리</p>
</div>

스페이스 설정에는 에이전트, 플러그인, 워크플로, 배포 채널을 통합 관리하는 채널과, 호출하는 각종 대형 모델을 볼 수 있는 모델 관리가 포함됩니다. 그림 5.5.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-05.png" alt="Image description" width="90%"/>
  <p>그림 5.5 Coze Agent 배포 채널</p>
</div>

Coze 에이전트 개발을 한마디로 정리한다면, 게임의 여러 구성 요소에 비유할 수 있습니다. 각 부분을 조합해 멋진 에이전트를 만드는 것은 "게임"을 하는 것과 매우 비슷합니다. 에이전트를 하나 완성할 때마다 보스를 물리친 것처럼 "경험치"나 "장비"를 많이 얻는 기분이 듭니다.

- 워크플로: 스테이지 클리어 루트 맵
- 대화 플로우: NPC 대화 클리어
- 플러그인: 캐릭터 스킬 카드
- 지식 베이스: 게임 백과사전
- 카드: 퀵 아이템 바
- 프롬프트: 캐릭터 조작 키
- 데이터베이스: "클라우드 세이브"
- 배포 관리: 스테이지 심사관
- 모델 관리: 게임 캐릭터 라이브러리 또는 캐릭터 생성 시스템
- 효과 평가: 스테이지 점수 시스템

### 5.2.2 "Daily AI Brief" 어시스턴트 구축하기

**사례 설명:** 이 실습 사례는 Coze 플랫폼의 플러그인 통합 역량을 깊이 분석하고, 독자가 처음부터 강력한 "Daily AI Brief" 에이전트를 구축하도록 안내합니다. 이 에이전트는 36Kr, Huxiu, IT Home, InfoQ, GitHub, arXiv 등 여러 정보 소스에서 AI 분야 최신 헤드라인, 학술 논문, 오픈소스 프로젝트 업데이트를 자동으로 수집하고, 구조적이고 전문적인 방식으로 생생하고 간결한 브리핑으로 통합합니다.

이 사례를 통해 다음 핵심 스킬을 체계적으로 익힐 수 있습니다.

  * **다중 소스 정보 집계:** Coze 플러그인 생태계를 활용해 크로스 플랫폼·크로스 타입 데이터 흐름을 seamless하게 통합
  * **에이전트 행동 정의:** 역할 설정과 프롬프트 엔지니어링으로 에이전트의 작업 실행과 콘텐츠 생성을 정밀 제어해, 출력이 사전 설정한 전문 기준을 충족하도록 보장
  * **자동화 워크플로 구축:** 데이터 수집, 콘텐츠 처리, 포맷 출력 등 여러 단계를 효율적인 자동화 워크플로로 연결하는 방법 학습

**Step 1: 정보 소스 플러그인 추가 및 구성**

"Daily AI Brief" 에이전트 구축의 첫 과제는 풍부하고 신뢰할 수 있는 정보 소스에 연결하는 것입니다. Coze 플랫폼에서는 해당 플러그인을 추가하고 구성하여 이를 달성합니다.

1.  **플러그인 통합:** Coze 플러그인 라이브러리에서 필요한 플러그인을 검색해 추가합니다. 예를 들어 **RSS** 플러그인(그림 5.6)으로 미디어 플랫폼 RSS를 구독하고, **GitHub** 플러그인(그림 5.7)으로 오픈소스 프로젝트를 추적하며, **arXiv** 플러그인(그림 5.8)으로 최신 학술 연구 결과를 가져옵니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-06.png" alt="Image description" width="90%"/>
  <p>그림 5.6 미디어 플랫폼용 RSS 소스 플러그인</p>
</div>
<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-07.png" alt="Image description" width="90%"/>
  <p>그림 5.7 GitHub 플러그인</p>
</div>
<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-08.png" alt="Image description" width="90%"/>
  <p>그림 5.8 Arxiv 플러그인</p>
</div>

2.  **개인화 구성:** 각 플러그인을 세밀하게 구성해 필요한 데이터를 정확히 가져오도록 합니다. 예를 들어 RSS 플러그인에는 36Kr, Huxiu 등의 RSS 구독 링크를 입력하고, GitHub 플러그인에는 모니터링할 키워드 쿼리 수와 최신 업데이트 설정을 지정하며, arXiv 플러그인에는 "LLM", "AI" 등 관심 키워드와 수량·최신 업데이트 설정을 정의합니다.

```
RSS Link Configuration

- **36Kr:** https://www.36kr.com/feed
- **Huxiu:** https://rss.huxiu.com/
- **IT Home:** http://www.ithome.com/rss/
- **InfoQ:** https://feed.infoq.com/ai-ml-data-eng/

GitHub Plugin Configuration

- q:AI
- per_page:10
- sort:updated

Arxiv Plugin Configuration

- count: 5
- search_query: AI
- sort_by: 2
```

3.  **오케스트레이션 및 연결:** 에이전트의 시각적 오케스트레이션 인터페이스에서 구성한 정보 소스 플러그인(`rss_24Hbj`, `searchRepository`, `arxiv` 등)을 데이터 입력 노드로 사용하고, 후속 논리 처리 모듈(**Large Model** 모듈 등)에 연결해 완전한 데이터 처리 경로를 구축합니다. 그림 5.9.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-09.png" alt="Image description" width="90%"/>
  <p>그림 5.9 Daily AI Brief 오케스트레이션 플로우차트</p>
</div>

**Step 2: 에이전트 역할 및 프롬프트 설정**

역할 설정과 프롬프트 작성은 에이전트 행동과 출력 품질을 정의하는 핵심 단계입니다. 이 단계는 추상적 지시를 에이전트가 이해하고 실행할 수 있는 구체적 작업으로 바꾸는 것을 목표로 합니다.

(1) 역할 설정

에이전트를 **시니어급 권위 있는 기술 미디어 에디터**로 설정합니다. 이 역할은 에이전트에 명확한 전문적 포지셔닝을 부여하여, 이후 콘텐츠 제작에서 전문 에디터의 사고 방식을 모방해 정보를 효율적으로 선별·통합·요약하도록 합니다.

(2) 프롬프트 작성 및 구조화

프롬프트는 에이전트가 작업을 실행하기 위한 지침서입니다. 지시가 명확하고 완전하며 제어 가능하도록 **System Prompt와 User Prompt**로 나눕니다.

**System Prompt**

시스템 프롬프트는 에이전트의 장기 행동 가이드라인과 출력 형식 규격을 정의합니다.

```
# Role
You are a senior and authoritative technology media editor, skilled at efficiently and precisely integrating and creating highly professional technology briefs, with deep analytical and integration capabilities especially in AI field technical developments, cutting-edge academic research results, and popular open-source projects.

## Workflow
### Daily Report Output Format
1. The daily report should prominently display "AI Daily Report", "by@jasonhuang", and the current date at the beginning, for example: "AI Daily Report | September 24, 2025 | by@jasonhuang".
2. <!!!important!!!> Add a unique Emoji symbol at the beginning of each title based on the different content of each AI technology news, each AI academic paper, and each AI open-source project.
3. All output content must be highly relevant to AI, LLM, AIGC, large models, and other technical topics, firmly excluding any irrelevant information, advertisements, and marketing content.
4. Must provide the original link for each item (including AI technology news, AI academic papers, AI open-source projects).
5. Provide a brief and precise summary description for each news item or project output.
```

**User Prompt**

사용자 프롬프트는 구체적 작업 지시와 데이터 소스를 정의합니다.

```
- **Information Extraction and Integration:** From input sources `{{articles}}`, `{{articles1}}`, `{{articles2}}`, and `{{articles3}}`, filter and extract article titles and corresponding links related to AI, large models, AIGC, LLM, and other topics, and organize them into the **"AI Technology News"** module.
- **Academic Paper Summary:** From input source `{{arxiv}}`, based on fields `arxiv_title` and `arxiv_link`, summarize and organize the latest paper content to form the **"AI Academic Papers"** module.
- **Open-Source Project Filtering:** From input source `{{GitHub}}`, filter out the **5 most prominent and influential AI open-source projects**. Extract the titles and corresponding links of these projects and organize them into the **"AI Open-Source Projects"** module.

# Attention
- Strictly follow the daily report output format defined in the system prompt.
- The total output content should be: **10 AI technology news items, 5 AI academic papers, 5 AI open-source projects**.
```

**Step 3: 테스트, 디버깅 및 멀티채널 배포**

에이전트 핵심 로직 구축을 마친 뒤에는 출력이 기대에 부합하는지 확인하기 위해 엄격한 테스트와 디버깅을 수행해야 합니다.

**실행 미리보기:** Coze 플랫폼 미리보기 인터페이스에서 에이전트를 실행하고 생성된 브리핑 콘텐츠를 확인합니다.

```
# AI Daily Report by@jasonhuang 2025-09-24

## 🚀 AI Technology News

🤖 **Zhiyuan Robot GO-1 Universal Embodied Foundation Model Fully Open-Sourced**
Link: https://36kr.com/p/3479085489708163?f=rss
Summary: Zhiyuan Robot announced the full open-sourcing of its GO-1 universal embodied foundation model, providing powerful AI foundational capabilities for the robotics field.

🔬 **Microsoft Overcomes Data Center Chip Cooling Bottleneck: Microfluidics + AI Precision Cooling**
Link: https://www.ithome.com/0/885/391.htm
Summary: Microsoft achieves precise temperature control of data center chips through the combination of microfluidic technology and AI algorithms, improving energy efficiency.
......

## 📚 AI Academic Papers

🧪 **Lyra: Generative 3D Scene Reconstruction via Video Diffusion Model Self-Distillation**
Link: http://arxiv.org/pdf/2509.19296v1
Summary: Proposes an innovative framework for 3D scene generation through video diffusion model self-distillation, without requiring multi-view training data.

📊 **The ICML 2023 Ranking Experiment: Examining Author Self-Assessment in ML/AI Peer Review**
Link: http://arxiv.org/pdf/2408.13430v3
Summary: Studies the effectiveness of author self-assessment in machine learning conference review processes and proposes methods to improve review mechanisms.
......

## 💻 AI Open-Source Projects

🤖 **llmling-agent - Multi-Agent Workflow Framework**
Link: https://github.com/phil65/llmling-agent
Summary: Multi-agent interaction framework supporting YAML configuration and programming methods, integrating MCP and ACP protocol support.

🚌 **College_EV_AI_Transportation - Campus AI Electric Transportation System**
Link: https://github.com/LuisMc2005v/College_EV_AI_Transportation
Summary: AI-driven campus electric transportation optimization system, achieving real-time tracking and efficient carpooling services.
......
```

브리핑의 내용 정확성, 형식 완전성, 언어 스타일을 꼼꼼히 확인합니다. 기대에 못 미치는 부분이 있으면 프롬프트 또는 플러그인 구성 단계로 돌아가 세밀하게 조정합니다. 예를 들어 내용이 충분히 간결하지 않으면 프롬프트의 요약 요구사항을 수정하고, 데이터 수집이 부정확하면 플러그인 구성 파라미터를 점검합니다.

**멀티채널 배포:** Coze는 에이전트를 WeChat, Doubao, Feishu 등 여러 주류 애플리케이션 플랫폼에 원클릭으로 배포할 수 있어 에이전트 적용 시나리오를 크게 확장합니다. 그림 5.10.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-10.png" alt="Image description" width="90%"/>
  <p>그림 5.10 Coze 플랫폼의 다양한 배포 채널</p>
</div>

에이전트를 배포한 뒤 Coze 스토어에서 만든 AI 에이전트를 볼 수 있고, AI 애플리케이션에 통합해 사용자에게 서비스할 수도 있습니다. 그림 5.11, 5.12. [Daily AI News Agent 체험 링크](https://www.coze.cn/store/agent/7506052197071962153?bot_id=true&bid=6hkt3je8o2g16)도 참고하세요.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-11.png" alt="Image description" width="90%"/>
  <p>그림 5.11 AI Agent - Daily AI News</p>
</div>

또한 이 [체험 링크](https://www.coze.cn/store/project/7458678213078777893?from=store_search_suggestion&bid=6gu3cmr7k5g1i)를 클릭하면 AI 애플리케이션에서 Daily AI News를 볼 수 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-12.png" alt="Image description" width="90%"/>
  <p>그림 5.12 AI 애플리케이션의 Daily AI News</p>
</div>

**배포 구성:** 자신의 에이전트를 배포하려면 배포 전에 적절한 이름, 아바타, 환영 메시지를 구성해 더 친숙한 사용자 경험을 제공해야 합니다. 그림 5.13, 5.14.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-13.png" alt="Image description" width="90%"/>
  <p>그림 5.13 에이전트 기본 정보 구성</p>
</div>
<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/coze-14.png" alt="Image description" width="90%"/>
  <p>그림 5.14 에이전트 오프닝 멘트 및 사전 질문 구성</p>
</div>

### 5.2.3 Coze의 장점과 한계 분석

**장점:**

  * **강력한 플러그인 생태계:** Coze 플랫폼의 핵심 강점은 풍부한 플러그인 라이브러리로, 에이전트가 외부 서비스와 데이터 소스에 쉽게 접근해 기능 확장성을 높일 수 있습니다.
  * **직관적인 시각적 오케스트레이션:** 플랫폼은 진입 장벽이 낮은 시각적 워크플로 오케스트레이션 인터페이스를 제공합니다. 사용자는 깊은 프로그래밍 지식 없이 "드래그 앤 드롭"으로 복잡한 워크플로를 구축할 수 있어 개발 난이도가 크게 낮아집니다.
  * **유연한 프롬프트 제어:** 정밀한 역할 설정과 프롬프트 작성으로 에이전트 행동과 콘텐츠 생성을 세밀하게 제어해 고도로 맞춤화된 출력을 달성할 수 있습니다. 프롬프트 관리와 템플릿도 지원해 개발을 크게 편리하게 합니다.
  * **편리한 멀티 플랫폼 배포:** 동일한 에이전트를 여러 애플리케이션 플랫폼에 배포해 seamless한 크로스 플랫폼 통합과 적용이 가능합니다. Coze는 계속해서 새 플랫폼을 생태계에 통합하고 있으며, 점점 더 많은 휴대폰·하드웨어 제조사가 Coze 에이전트 배포를 지원하고 있습니다.

**한계:**

  * **MCP 미지원:** 이 점이 가장 치명적이라고 볼 수 있습니다. Coze 플러그인 마켓은 매우 풍부하고 매력적이지만, MCP를 지원하지 않으면 발전을 제한하는 족쇄가 될 수 있습니다. 개방된다면 또 하나의 killer feature가 될 것입니다.
  * **일부 플러그인 구성의 높은 복잡도:** API Key 등 고급 파라미터가 필요한 플러그인은 사용자가 어느 정도 기술 배경을 갖춰야 올바르게 구성할 수 있습니다. 복잡한 워크플로 오케스트레이션도 제로 기반으로는 바로 익히기 어렵고, JavaScript나 Python 기초가 필요합니다.
  * **JSON 파일 가져오기 불가:** 이전에는 앱에 내보내기/가져오기 기능이 없었지만, 유료 버전에는 이제 있습니다. 다만 내보내기/가져오기 파일은 Dify나 N8n처럼 JSON이 아니라 ZIP입니다. 즉 앱에서 내보낸 뒤 ZIP만 가져올 수 있습니다. 우회 방법도 있습니다. 레이아웃 인터페이스에서 Ctrl+A로 전체 선택 후 Ctrl+C로 복사하고, 다른 빈 워크플로나 기존 워크플로에 붙여 넣을 수 있습니다.

## 5.3 플랫폼 2: Dify

### 5.3.1 Dify 및 생태계 소개

Dify는 Backend as a Service(BaaS)와 LLMOps 개념을 통합한 오픈소스 대규모 언어 모델(LLM) 애플리케이션 개발 플랫폼으로, 프로토타입 설계부터 프로덕션 배포까지 전 과정을 지원합니다. 그림 5.15. 계층형 모듈 아키텍처를 채택하여 데이터 계층, 개발 계층, 오케스트레이션 계층, 기반 계층으로 나뉘며, 각 계층은 분리되어 확장이 용이합니다.

Dify는 모델 중립성과 호환성이 높습니다. 오픈소스든 상용 모델이든 간단한 구성으로 통합할 수 있고, 통일된 인터페이스로 추론 역량을 호출할 수 있습니다. GPT, Deepseek, Llama 등 수백 개의 오픈소스·독점 LLM 통합을 내장 지원하며, OpenAI API와 호환되는 모든 모델도 사용할 수 있습니다.

동시에 Dify는 로컬 배포(공식 Docker Compose 원클릭 시작)와 클라우드 배포를 지원합니다. 로컬/프라이빗 환경에 직접 배포해 데이터 프라이버시를 보장하거나, 공식 SaaS 클라우드 서비스를 사용할 수 있습니다(아래 비즈니스 모델 절 참고). 이런 배포 유연성은 보안 요구가 있는 엔터프라이즈 내부망이나 운영 편의를 중시하는 개발자 그룹에 적합합니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-01.png" alt="Image description" width="90%"/>
  <p>그림 5.15 Dify 공식 웹사이트</p>
</div>

**Marketplace 플러그인 생태계:** Dify Marketplace는 원스톱 플러그인 관리와 원클릭 배포 기능을 제공하여, 개발자가 플러그인을 발견·확장·제출할 수 있게 해 커뮤니티에 더 많은 가능성을 열어 줍니다. 그림 5.16.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-02.png" alt="Image description" width="90%"/>
  <p>그림 5.16 Dify Marketplace 플러그인 생태계</p>
</div>

Marketplace에는 다음이 포함됩니다.

- Models
- Tools
- Agent Strategies
- Extensions
- Bundles

현재 Dify Marketplace에는 8,677개 이상의 플러그인이 있으며, 다양한 기능과 적용 시나리오를 포괄합니다. 공식 추천 플러그인 예시는 다음과 같습니다.
- Google Search: langgenius/google
- Azure OpenAI: langgenius/azure_openai
- Notion: langgenius/notion
- DuckDuckGo: langgenius/duckduckgo

Dify는 플러그인 개발자를 위한 강력한 개발 지원을 제공합니다. 인기 IDE와 seamless하게 협업하는 원격 디버깅 기능을 포함하며, 최소한의 환경 설정만 필요합니다. 개발자는 Dify SaaS 서비스에 연결하면서 모든 플러그인 작업을 로컬 환경으로 전달해 테스트할 수 있습니다. 이런 개발자 친화적 접근은 플러그인 창작자를 empower하고 Dify 생태계 혁신을 가속하는 것을 목표로 합니다. 모델은 통합할 수 있고, 프롬프트와 오케스트레이션은 복사할 수 있지만, 도구 플러그인의 존재와 풍부함이 에이전트가 더 좋은 결과나 예상 밖의 강력한 기능을 달성할 수 있는지를 직접 좌우하기 때문에 Dify가 현재 가장 성공적인 agent 플랫폼 중 하나가 될 수 있었습니다.

### 5.3.2 슈퍼 에이전트 개인 비서 구축하기

> **✨✨ 상세 조작 가이드**: **[Dify Agent 생성 단계별 튜토리얼](https://github.com/datawhalechina/hello-agents/blob/main/Extra-Chapter/Extra03-Dify智能体创建保姆级操作流程.md)** 참고

이전 Coze 사례에서는 daily AI brief 에이전트를 구축했습니다. 기능은 명확하지만 단일 브리핑 생성 역량에는 한계가 있습니다. 이 절에서는 Dify로 일상 Q&A, 카피 최적화, 멀티모달 생성, 데이터 분석 등 여러 시나리오를 아우르는 풀기능 슈퍼 에이전트 개인 비서를 구축합니다. 시작하기 전에 Dify의 주요 인터페이스와 기능 모듈을 간단히 살펴보겠습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-14.png" alt="Image description" width="90%"/>
  <p>그림 5.17 Dify Agent 구축 홈페이지</p>
</div>
<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-18.png" alt="Image description" width="90%"/>
  <p>그림 5.18 Dify 공식 템플릿 라이브러리</p>
</div>
<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-15.png" alt="Image description" width="90%"/>
  <p>그림 5.19 Dify 지식 베이스</p>
</div>
<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-16.png" alt="Image description" width="90%"/>
  <p>그림 5.20 Dify 플러그인 마켓</p>
</div>
<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-17.png" alt="Image description" width="90%"/>
  <p>그림 5.21 Dify 대형 모델 구성</p>
</div>

**(1) 플러그인 생성 및 MCP 구성**

에이전트를 구축하기 전에 필요한 플러그인 설치와 MCP 구성을 먼저 완료해야 합니다. 그림 5.22는 이 사례에 필요한 핵심 플러그인입니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-19.png" alt="Image description" width="90%"/>
  <p>그림 5.22 Dify 플러그인 설치 구성</p>
</div>

그림에서 빨간 박스로 표시된 플러그인은 Dify 플러그인 마켓에서 검색해 설치해야 합니다. 사용자는 상세 보기를 클릭해 각 플러그인의 구체적 기능을 확인할 수 있습니다.

다음으로 MCP(Model Context Protocol)를 구성합니다. MCP의 상세 원리는 여기서 확장하지 않고, 클라우드 배포 MCP 서비스 사용 방법을 중심으로 시연합니다. 이 사례는 국내 ModelScope 커뮤니티 MCP 마켓을 사용합니다. 그림 5.23.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-20.png" alt="Image description" width="90%"/>
  <p>그림 5.23 ModelScope 커뮤니티 MCP 마켓</p>
</div>

ModelScope 커뮤니티 MCP 마켓을 열고 hosted 타입을 선택합니다. Amap MCP를 예로 들면, 홈페이지에 진입한 뒤 오른쪽에서 SSE 모드를 선택하고 connection configuration을 클릭하면 전용 MCP 구성 JSON이 생성됩니다. 그림 5.24. MCP는 여러 통신 모드를 지원하지만, Dify에서 SSE 모드 통신이 더 매끄럽고 안정적이므로 SSE 모드를 권장합니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-21.png" alt="Image description" width="90%"/>
  <p>그림 5.24 Amap MCP 구성 예시</p>
</div>

**(2) 에이전트 설계 및 효과 시연**

이 사례에서는 다음 기능 모듈을 아우르는 종합 개인 비서를 만듭니다.

- 일상 생활 Q&A
- 카피 다듬기 및 최적화
- 멀티모달 콘텐츠 생성(이미지, 비디오)
- 데이터 조회 및 시각화 분석
- MCP 도구 통합(Amap, 식단 추천, 뉴스 정보)

전체 에이전트 오케스트레이션 아키텍처는 그림 5.25와 같습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-12.png" alt="Image description" width="90%"/>
  <p>그림 5.25 Agent 오케스트레이션</p>
</div>

멀티 에이전트 아키텍처에서는 question classifier로 지능형 라우팅을 사용합니다. classifier에서 각 에이전트의 핵심 기능과 작업 범위를 정의해 사용자 요청이 정확히 해당 처리 모듈로 분배되도록 합니다.

**Daily Assistant 모듈**

대형 언어 모델과 시간 도구를 구성한 기본 대화 모듈로, fallback 일반 Q&A 서비스 역할을 합니다.

프롬프트 구성:
```
# Role: Daily Question Consultation Expert

## Profile
- language: Chinese
- description: Specializes in answering general questions in users' daily lives, providing practical, accurate, and easy-to-understand advice and answers
- background: Possesses rich life experience and extensive knowledge reserves, skilled at simplifying complex problems
- personality: Kind and friendly, patient and meticulous, pragmatic and reliable
- expertise: Daily life, health and wellness, family management, interpersonal relationships, practical tips


## Skills

1. Problem Analysis Ability
   - Quick Understanding: Rapidly grasp the core points of user questions
   - Classification Recognition: Accurately judge the life domain to which the question belongs
   - Demand Mining: Deeply understand users' potential needs
   - Priority Sorting: Reasonably assess the importance and urgency of problems

2. Answer Providing Ability
   - Knowledge Integration: Comprehensively apply multi-domain knowledge to provide answers
   - Solution Formulation: Provide specific and feasible solutions
   - Step Decomposition: Break down complex problems into simple steps
   - Alternative Solutions: Prepare multiple backup solutions for users to choose from

3. Communication and Expression Ability
   - Popular Language: Use simple and easy-to-understand everyday language
   - Clear Logic: Organize answer content in a well-organized manner
   - Illustrative Examples: Help understanding through specific cases
   - Highlight Key Points: Emphasize key information and precautions

## Rules

1. Answer Principles:
   - Practicality First: Ensure the advice provided is actionable
   - Accuracy Guarantee: Give answers based on reliable information and common sense
   - Neutral and Objective: Avoid personal bias and subjective assumptions
   - Moderate Advice: Provide appropriate depth of answers based on problem complexity

2. Code of Conduct:
   - Timely Response: Quickly respond to users' questions
   - Patient and Meticulous: Maintain patience with repetitive or simple questions
   - Active Guidance: Encourage users to provide more background information
   - Continuous Improvement: Optimize answer quality based on feedback


## Workflows

- Goal: Provide users with practical and reliable daily problem solutions
- Step 1: Carefully read and understand the daily questions raised by users
- Step 2: Analyze the problem type and users' potential needs
- Step 3: Provide specific and feasible suggestions based on common sense and experience
- Step 4: Organize answer content in easy-to-understand language
- Step 5: Check the practicality and safety of the answer


## Initialization
As a daily question consultation expert, you must abide by the above Rules and execute tasks according to Workflows.
```

효과 시연은 그림 5.26과 같습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-03.png" alt="Image description" width="90%"/>
  <p>그림 5.26 Daily Assistant</p>
</div>

**Copywriting Optimization 모듈**

OpenAI 데이터 보고서에 따르면 60% 이상의 사용자가 ChatGPT를 텍스트 최적화 관련 작업(다듬기, 수정, 확장, 축약)에 사용합니다. 따라서 카피 최적화는 고빈도 수요 시나리오이며, 두 번째 핵심 기능 모듈로 둡니다.

프롬프트 구성:
```
# I. Role Setting (Role)
You are a professional copywriting optimization expert with rich experience in marketing copywriting and optimization, skilled at improving the attractiveness, conversion rate, and readability of copy. Your perspective is from the angle of the target audience and marketing goals, with professional boundaries limited to the copywriting optimization field, not involving technical implementation or product development.

# II. Background
The user has provided a piece of original copy that needs your optimization to improve its overall effectiveness. Background information includes: the copy may be used for marketing, brand promotion, or information communication scenarios, but the specific use is not detailed. The known condition is that the user hopes the copy is more attractive, clear, or persuasive, but has not provided the original copy content, so you need to work based on general optimization principles.

# III. Task Objectives (Task)
- Analyze and optimize the structure, language, and style of the copy to make it more in line with the preferences of the target audience.
- Improve the attractiveness, readability, and conversion potential of the copy, ensuring clear information delivery.
- Make adjustments according to common optimization principles (such as conciseness, emotional resonance, call to action, etc.), without content rewriting unless necessary.
- While maintaining core information, appropriately expand and enrich copy content to provide a more comprehensive optimized version.

# IV. Limitation Prompts (Limit)
- Avoid changing the core information or intent of the original copy unless explicitly requested by the user.
- Do not add fictional or irrelevant content, ensuring optimization is based on logic and best practices.
- Avoid using overly technical or professional terminology unless the target audience is professionals.
- Do not involve optimization of images, layouts, or other non-text elements.

# V. Output Format Requirements (Example)
The output should be optimized copy text with clear structure, fluent language, and substantial content. For example:
- If the original copy is "Our product is very good, come and buy it"
The optimized version can be: "In this era full of choices, what truly touches people's hearts is never exaggerated propaganda, but good products that can withstand the test of time and users. Our product is exactly that. It not only pays attention to details and quality in design but also continuously polishes and innovates in function, just to bring a better user experience to every user. Whether it's the texture of the appearance or the stability of performance, we always adhere to high standards and strict requirements, striving to make every customer who chooses us feel the surprise of value for money.
We deeply understand that purchasing a product is not just a simple consumption but a choice of lifestyle. Therefore, from material selection, craftsmanship to after-sales service, we have poured full sincerity and professionalism into every link, carefully guarding your every experience. Whether you pursue practicality, value quality, or want unique personalization, our products can provide you with ideal solutions.
Now, let us prove everything with action. A truly good product does not need too much embellishment; it itself is the best spokesperson. Act now, choose us, let quality change life, and have a different experience from now on!"
- The output should directly present optimized content without additional explanations or annotations unless requested by the user. Please ensure that the optimized copy content is richer and more complete, and the optimized copy text must exceed 500 words.
```

효과 시연은 그림 5.27과 같습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-04.png" alt="Image description" width="90%"/>
  <p>그림 5.27 Copywriting Assistant</p>
</div>

**Multimodal Generation 모듈**

이미지·비디오 생성은 또 다른 고빈도 적용 시나리오입니다. Doubao 이미지 생성, Google Imagen 같은 모델의 진화와 Keling, Google Veo 3, OpenAI Sora 2 같은 비디오 생성 기술의 돌파로, 멀티모달 콘텐츠 생성 품질은 실용 수준에 도달했습니다.

이 사례에서는 Doubao 플러그인으로 이미지·비디오 생성을 구현합니다. 구성 단계는 다음과 같습니다.

1. 워크플로에 Doubao 이미지/비디오 생성 플러그인 추가
2. 파라미터 구성(예: 이미지 비율 1:1, 모델 선택 doubao seedream)
3. 생성된 파일 출력

이미지 생성 구성과 효과는 그림 5.28, 5.29.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-13.png" alt="Image description" width="90%"/>
  <p>그림 5.28 Image Generation Settings</p>
</div>
<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-05.png" alt="Image description" width="90%"/>
  <p>그림 5.29 Image Generation Assistant</p>
</div>

비디오 생성 효과는 그림 5.30.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-06.png" alt="Image description" width="90%"/>
  <p>그림 5.30 Video Assistant</p>
</div>

**Data Query and Analysis 모듈**

데이터 처리는 에이전트의 중요한 역량 중 하나입니다. 이 모듈은 Dify에서 데이터베이스에 연결해 데이터 조회와 시각화 분석을 구현하는 방법을 보여 줍니다.

먼저 데이터 조회 도구 플러그인을 설치합니다. 이 사례에서는 `rookie-text2data` 플러그인을 사용합니다. 데이터 조회의 핵심은 대형 모델에 명확한 테이블 구조와 필드 정보를 제공해 정확한 SQL 쿼리문을 생성하게 하는 것입니다. 일반적인 방법은 다음과 같습니다.

- 데이터 테이블 DDL 문을 직접 제공
- 테이블명과 필드명 대응 관계 설명 제공

데이터베이스 연결 정보(IP 주소, DB 이름, 포트, 계정, 비밀번호 등)를 구성합니다. 그림 5.31. 조회 결과는 대형 모델 노드를 통해 정리하고 이해하기 쉬운 자연어 출력으로 변환해야 합니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-22.png" alt="Image description" width="90%"/>
  <p>그림 5.31 Database Configuration</p>
</div>

프롬프트 설정:

```
# I. Role Setting (Role)
You are a professional data query specialist, skilled at data organization, with clear logical thinking and concise expression ability.

# II. Background
The user has provided raw data queried from the database. This data may have issues such as inconsistent formats, missing fields, and duplicate records, and needs professional organization before effective display.

# III. Task Objectives (Task)
1. Summarize and organize raw data
2. Classify and sort data according to correct logic
3. Data display highlights key information and data insights
4. Provide easy-to-understand data display

# IV. Limitation Prompts (Limit)
1. Must not arbitrarily delete important data
2. Avoid using overly complex or professional statistical terminology
3. Must not tamper with the true values of raw data
4. Avoid displaying too much redundant information, keep it concise and clear
5. Must not leak sensitive data or personal privacy information

# V. Output Format Requirements (Example)
 Data Overview: Simply briefly explain the data content
```

효과 표시는 그림 5.32.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-07.png" alt="Image description" width="90%"/>
  <p>그림 5.32 Data Query Assistant</p>
</div>

프롬프트 설정:

```
# I. Role Setting (Role)
You are a professional data analyst with data organization, cleaning, and visualization capabilities, able to extract key information from raw data and transform it into intuitive visual displays.

# II. Background
The user has queried a batch of raw data from the database. This data may contain multiple fields, missing values, or inconsistent formats, and needs to be organized before generating visualization charts.

# III. Task Objectives (Task)
# Workflow
1. Data Analysis
Analyze, organize, and summarize data according to reasonable rules
2. Analysis & Visualization
Generate at least 1 chart (choose one or more from bar / line / pie chart)
Can call tools: "generate_pie_chart" | "generate_column_chart" | "generate_line_chart"

# IV. Limitation Prompts (Limit)
1. Avoid using overly complex chart types, ensure visualization results are easy to understand
2. Do not ignore data quality issues, must perform necessary data cleaning
3. Avoid using too many colors or elements in visualization, keep it concise and clear
4. Do not omit labeling and explanation of key data
5. Must perform summary and chart generation, regardless of data volume

# V. Output Format Requirements (Example)
Please output in the following format:
1. Data overview summary (do not output field names, do not list points, just a short paragraph)
2. Display generated charts
```

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-08.png" alt="Image description" width="90%"/>
  <p>그림 5.33 Data Analysis Assistant</p>
</div>

데이터 분석 어시스턴트의 유일한 차이는 데이터 시각화 도구, 즉 "generate_pie_chart" | "generate_column_chart" | "generate_line_chart" BI 차트 생성 도구 플러그인을 추가했다는 점입니다. 앞서 요구대로 설치했다면 바로 추가해 사용할 수 있으며, 위 프롬프트처럼 해당 설명을 추가하면 됩니다.

**MCP Tool Integration**

마지막으로 MCP 도구의 통합 적용입니다. MCP 구성은 이미 완료했으므로, 이제 에이전트에 통합합니다. 구성 단계는 다음과 같습니다.

1. MCP 호출을 지원하는 에이전트 전략 선택
2. ReAct 모드 선택
3. MCP 서비스 구성(`mcp-server` 접두사 삭제, SSE 모드 선택)
4. 해당 프롬프트 작성

구성 인터페이스는 그림 5.34.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-23.png" alt="Image description" width="90%"/>
  <p>그림 5.34 Agent MCP Configuration</p>
</div>

Amap assistant, dietary assistant, news assistant 효과는 각각 그림 5.35, 5.36, 5.37.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-09.png" alt="Image description" width="90%"/>
  <p>그림 5.35 Amap Assistant</p>
</div>

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-10.png" alt="Image description" width="90%"/>
  <p>그림 5.36 Dietary Assistant</p>
</div>

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/dify-11.png" alt="Image description" width="90%"/>
  <p>그림 5.37 News Assistant</p>
</div>

이 시점에서 풀기능 슈퍼 에이전트 개인 비서를 완성했습니다. 이 어시스턴트는 생활 여러 면을 아우릅니다. 새 옷이 필요하면 Doubao로 디자인을 생성하고, 외출 전에는 Amap assistant로 경로를 계획하며, 무엇을 먹을지 모를 때는 식단 추천을 받고, 학습 상황을 파악하고 싶을 때는 데이터 분석을 수행할 수 있습니다. 다양한 업무와 생활 작업을 처리할 수 있으며, 여러분도 더 창의적인 개인 agent assistant를 만들어 보시길 기대합니다.

### 5.3.3 Dify의 장점과 한계 분석

선도적인 AI 애플리케이션 개발 플랫폼으로서 Dify는 여러 측면에서 뚜렷한 장점을 보여 줍니다.

1. 핵심 장점

- **풀스택 개발 경험:** Dify는 RAG 파이프라인, AI 워크플로, 모델 관리 등을 하나의 플랫폼에 통합해 원스톱 개발 경험을 제공합니다.
- **로우코드와 높은 확장성의 균형:** Dify는 로우코드 개발의 편의성과 전문 개발의 유연성 사이에서 좋은 균형을 이룹니다.
- **엔터프라이즈급 보안과 컴플라이언스:** Dify는 AES-256 암호화, RBAC 권한 제어, 감사 로그를 제공해 엄격한 보안·컴플라이언스 요구를 충족합니다.
- **풍부한 도구 통합 역량:** Dify는 9000개 이상의 도구와 API 확장을 지원해 광범위한 기능 확장성을 제공합니다.
- **활발한 오픈소스 커뮤니티:** Dify는 활발한 오픈소스 커뮤니티를 갖추고 있어 풍부한 학습 자료와 지원을 제공합니다.

2. 주요 한계

- **가파른 학습 곡선:** 기술 배경이 전혀 없는 사용자에게는 여전히 학습 곡선이 있습니다.
- **성능 병목:** 고동시성 시나리오에서는 성능 이슈가 발생할 수 있어 적절한 최적화가 필요합니다. Dify 시스템의 핵심 서버 컴포넌트는 Python으로 구현되어 C++, Golang, Rust 같은 언어에 비해 상대적으로 성능이 낮습니다.
- **멀티모달 지원 부족:** 현재는 주로 텍스트 처리에 집중되어 있으며, 이미지·비디오·HTML 등 지원은 제한적입니다.
- **높은 Enterprise Edition 비용:** Dify enterprise edition 가격이 상대적으로 높아 소규모 팀 예산을 초과할 수 있습니다.
- **API 호환성 이슈:** Dify API 형식은 OpenAI와 호환되지 않아 일부 서드파티 시스템과의 통합을 제한할 수 있습니다.

## 5.4 플랫폼 3: n8n

앞서 소개했듯 n8n의 핵심 정체성은 범용 워크플로 자동화 플랫폼이며, 순수 LLM 애플리케이션 빌더는 아닙니다. 이 점을 이해하는 것이 n8n을 제대로 다루는 열쇠입니다. n8n으로 지능형 애플리케이션을 만들 때, 우리는 사실 더 큰 자동화 프로세스를 설계하고 있으며, 대규모 언어 모델은 그 과정에서 강력한 "처리 노드" 하나(또는 여러 개)일 뿐입니다.

### 5.4.1 n8n의 노드와 워크플로

n8n의 세계는 두 가지 가장 기본적인 개념, **Node**와 **Workflow**로 구성됩니다.

- **Node**: 노드는 워크플로에서 특정 작업을 수행하는 최소 단위입니다. 특정 기능을 가진 "블록"이라고 생각하면 됩니다. n8n은 이메일 전송, DB 읽기/쓰기, API 호출, 파일 처리 등 다양한 일반 작업을 아우르는 수백 개의 사전 구성 노드를 제공합니다. 각 노드는 입력과 출력을 갖고 그래픽 구성 인터페이스를 제공합니다. 노드는 대략 두 가지로 나뉩니다.
  - **Trigger Node**: 전체 워크플로의 시작점으로, 프로세스를 시작합니다. 예: "새 Gmail 이메일 수신", "매시간 한 번 트리거", "Webhook 요청 수신". 워크플로에는 반드시 하나, 그리고 하나만의 trigger node가 있어야 합니다.
  - **Regular Node**: 특정 데이터와 로직을 처리합니다. 예: "Google Sheets 스프레드시트 읽기", "OpenAI 모델 호출", "DB에 레코드 삽입".
- **Workflow**: 워크플로는 여러 노드가 연결된 자동화 플로우차트입니다. 데이터가 trigger node에서 시작해 서로 다른 노드 사이를 단계적으로 통과하며 처리되어 최종적으로 사전 설정한 작업을 완료하는 전체 경로를 정의합니다. 데이터는 구조화된 JSON 형식으로 노드 간에 전달되므로, 각 구간의 입력과 출력을 정밀하게 제어할 수 있습니다.

n8n의 진짜 힘은 강력한 "연결" 역량에 있습니다. 원래 분리되어 있던 애플리케이션과 서비스(회사 내부 CRM, 외부 소셜 미디어 플랫폼, DB, 대형 언어 모델)를 연결해, 이전에는 복잡한 코딩이 필요했던 end-to-end 비즈니스 프로세스 자동화를 구현할 수 있습니다. 다가올 실습에서는 이 노드·워크플로 시스템으로 AI 역량이 통합된 자동화 애플리케이션을 구축하는 방법을 직접 경험합니다.

### 5.4.2 지능형 이메일 어시스턴트 구축하기

n8n 환경 구성과 가장 기본적인 사용법은 프로젝트 `Additional-Chapter` 폴더에 문서가 있으므로 여기서는 많이 소개하지 않습니다. 이전 절에서 n8n의 기본 개념을 배웠고, 이 사례는 현대 AI Agent와 전통적 자동화 워크플로의 핵심 차이를 명확히 보여 줍니다. 전통적 프로세스는 선형적이지만, 우리가 만들 Agent는 사용자 이메일을 받아 핵심 **AI Agent node**로 "생각"하고, 여러 "도구" 중에서 자율적으로 의도를 이해·결정·선택한 뒤, 관련성 높은 답장을 자동 생성·발송할 수 있습니다.

전체 과정은 더 고급 의사결정 로직을 시뮬레이션합니다: `Receive -> AI Agent (Think -> Decide -> Tool Call) -> Reply`. 그림 5.38.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-01.png" alt="Image description" width="90%"/>
  <p>그림 5.38 통합 지능형 이메일 Agent 아키텍처 다이어그램</p>
</div>

도구를 여러 sub-workflow로 쪼개는 전통적 방식과 달리, n8n의 `AI Agent` node는 LLM, memory, tools 같은 컴포넌트를 통합 인터페이스에 모아 구축 과정을 크게 단순화합니다.

전체 구축 과정은 두 핵심 단계로 나뉩니다.

1. **Agent "Memory" 준비**: Agent용 private knowledge base를 로드하는 독립 프로세스 생성
2. **Agent 본체 구축**: 이메일 수신, 사고, 답장을 담당하는 main workflow 생성

### 5.4.3 Agent Private Knowledge Base 구축하기

Agent가 특정 도메인(개인 정보, 프로젝트 문서 등) 질문에 답하려면 먼저 "외부 두뇌", 즉 벡터 knowledge base를 준비해야 합니다.

n8n에서는 `Simple Vector Store` node로 메모리 내 knowledge base를 빠르게 구축할 수 있습니다. knowledge를 업데이트할 때만 보통 한 번 실행하면 됩니다.

**(1) Knowledge Source 정의**

먼저 `Code` node로 원본 knowledge text를 저장합니다. 간단하고 빠른 방법이며, 실제 프로젝트에서는 파일, DB 등에서도 가져올 수 있습니다.

- **Node**: `Code`
- **Content**: knowledge를 JSON 형식으로 작성

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-02.png" alt="Screenshot of knowledge base JSON text filled in Code node" width="90%"/>
  <p>그림 5.39 Code Node에서 Knowledge Source 정의</p>
</div>

```javascript
return [
  {
    "doc_id": "work-schedule-001",
    "content": "My working hours are Monday to Friday, 9 AM to 5 PM. The timezone is Australian Eastern Standard Time (AEST)."
  },
  {
    "doc_id": "off-hours-policy-001",
    "content": "During non-working hours (including weekends and public holidays), I cannot reply to emails immediately."
  },
  {
    "doc_id": "auto-reply-instruction-001",
    "content": "If an email is received during non-working hours, the AI assistant should inform the sender that the email has been received and I will process and reply as soon as possible between 9 AM and 5 PM on the next working day."
  }
];
```

**(2) Text Vectorization (Embeddings)**

컴퓨터는 텍스트를 직접 이해하지 못하므로 벡터로 변환해야 합니다. `Embeddings` node가 이 "번역" 작업을 담당합니다.

- **Node**: `Embeddings Google Gemini`, 모델은 `gemini-embedding-exp-03-07` 선택. 여기서는 Google API로 시연하며, Google API 획득 방법은 공식 문서 참고.
- **Configuration**: `Code` node 뒤에 연결하면 upstream에서 전달된 텍스트를 자동으로 벡터 데이터로 변환합니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-03.png" alt="" width="90%"/>
  <p>그림 5.40 Code 데이터 벡터화</p>
</div>

**(3) Vector Storage에 저장**

마지막으로 벡터화된 knowledge를 in-memory DB에 저장합니다. 그림 5.41.

- **Node**: `Simple Vector Store`
- **Configuration**:
  - **Operation Mode**: `Insert Documents`(쓰기 모드)
  - **Memory Key**: knowledge base에 고유 이름 부여, 예: `my-dailytime`. 이 Key는 DB의 "테이블명"과 같으며, Agent가 나중에 정보를 찾을 때 사용합니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-04.png" alt="" width="90%"/>
  <p>그림 5.41 Code 데이터를 Vector Storage에 저장</p>
</div>

구성을 마친 뒤 **이 프로세스를 수동으로 한 번 실행**합니다. 성공하면 private knowledge가 n8n memory에 로드됩니다. 그림 5.42.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-05.png" alt="" width="90%"/>
  <p>그림 5.42 Knowledge Base 로딩 워크플로 완료</p>
</div>

### 5.4.4 Agent Main Workflow 생성하기

도구가 준비되었으니 Agent main process를 구축합니다. 이메일 수신, 사고·의사결정, 적절한 시점에 방금 만든 도구 호출, 최종 이메일 답장 실행을 담당합니다.

(1) Gmail Trigger 구성

`Agent: Customer Support`라는 새 workflow를 만듭니다. `Gmail` node를 trigger로 사용하고 **Event**를 `Message Received`로 설정한 뒤 이메일 계정을 구성합니다. 새 이메일이 수신함에 들어올 때마다 workflow가 자동 트리거됩니다. 그림 5.43.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-06.png" alt="" width="90%"/>
  <p>그림 5.43 Gmail Node 생성</p>
</div>

구성 과정은 [n8n 공식 문서](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal#enable-apis)를 참고하세요. Gmail API는 [여기](https://console.cloud.google.com/apis/library/gmail.googleapis.com?project=apt-entropy-471905-b9)에서 구성합니다. credential을 생성하고 Web application 타입을 선택한 뒤 client ID와 client secret을 얻습니다. n8n이 제공하는 OAuth Redirect URL을 authorized redirect URIs에 추가해야 합니다. 동시에 [Audience](https://console.cloud.google.com/auth/audience?project=apt-entropy-471905-b9)의 Add users에 자신의 이메일 주소도 추가해야 합니다. 최종 구성 페이지는 그림 5.44.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-07.png" alt="" width="90%"/>
  <p>그림 5.44 Gmail 계정 로드 성공</p>
</div>

이제 `Fetch Test Event`를 클릭해 이메일을 가져올 수 있습니다. 그림 5.45!

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-08.png" alt="" width="90%"/>
  <p>그림 5.45 실시간 이메일 가져오기</p>
</div>

(2) AI Agent Node 구성

이것이 전체 workflow의 두뇌입니다. 노드 메뉴에서 `AI Agent` node를 드래그해 다음과 같이 구성합니다.

- **Chat Model**: 선택한 LLM 연결, 예: `Google Gemini Chat Model`. Agent의 "사고 핵심".
- **Memory**: `Simple Memory` node 연결. 같은 이메일 스레드에서 여러 왕복 이메일을 처리할 때 이전 대화 기록을 기억하게 합니다.
- **Tools**: 여기에 여러 도구를 연결할 수 있습니다. 이 사례에서는 두 도구를 연결합니다.
  1. `SerpAPI`: 4장 사례에서 사용한 API로, Agent가 공개 정보를 온라인 검색할 수 있게 합니다.
  2. `Simple Vector Store`: 1부에서 만든 private knowledge base를 조회할 수 있게 합니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-09.png" alt="" width="90%"/>
  <p>그림 5.46 AI Agent Node 설정</p>
</div>

Agent "사고"의 첫 단계입니다. `Gemini` node(또는 다른 LLM node)를 추가하고 mode를 `Chat`으로 설정합니다. 목표는 이메일 내용을 분석해 사용자 의도를 판단하는 것입니다. 프롬프트 설계가 매우 중요하며, 명확한 지시가 LLM이 작업을 더 정확히 수행하게 합니다. 이메일 본문과 제목(`{{ $json.snippet }}{{ $json.Subject }}`)을 변수로 Prompt에 전달합니다. API가 없다면 [Google AI Studio](https://aistudio.google.com/prompts/new_chat)에서 Get API key를 클릭해 생성할 수 있습니다.

AI Agent node에서는 주로 `User Message`와 `System Message`를 작성합니다. 그림 5.47.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-10.png" alt="" width="90%"/>
  <p>그림 5.47 AI Agent Node 상세</p>
</div>

이 사례에서 사용한 Prompt는 다음과 같습니다.

```json
# Prompt (User Message)
# Context Information
- Current Time: {{ new Date().toLocaleString('en-AU', { timeZone: 'Australia/Sydney', hour12: false }) }} (Sydney, Australia time)
- Sender: {{ $json.From }}
- Subject: {{ $json.Subject }}
- Email Body: {{ $json.snippet }}

# System Message
# Role and Goal
You are a 24/7 on-call, professional and efficient AI email assistant. Your task is: to do your best to answer all questions in emails using public information at the first opportunity, and add contextual status reminders at the beginning of replies based on my work schedule.

# Context Information
- Current Time: {{ new Date().toLocaleString('en-AU', { timeZone: 'Australia/Sydney', hour12: false }) }} (Sydney, Australia time)
- Email information is in the input data.

# Available Tools
- Simple Vector Store2: Used to query my exact working hours (e.g., Monday to Friday, 9 AM to 5 PM).
- SerpAPI: **[Primary Information Source]** Prioritize using this tool to search the internet to answer specific questions in emails.

# Execution Steps
1.  **Analyze the Question**: First, carefully read the email content and extract the sender's core question.

2.  **Parallel Information Gathering**: Execute the following two operations simultaneously to collect information:
    a. Use the `SerpAPI` tool to search online for answers to the sender's questions.
    b. Use the `Simple Vector Store2` tool to get my set exact working hours.

3.  **Draft Core Reply**: Based on the information collected by `SerpAPI`, clearly and directly answer the sender's question. This part will serve as the main body of the email reply.

4.  **Add Status Prefix and Integrate**:
    a. Compare "Current Time" with the working hours I obtained from the tool.
    b. **If currently "Non-working Hours"**: Create a status reminder prefix. This prefix **must include** the specific working hours obtained from `Simple Vector Store2`.
        * **Prefix Example**: "Hello, thank you for your email. You have contacted me during my non-working hours (my working hours are: [insert queried working hours here]). I will personally review this email on the next working day. In the meantime, here is a preliminary reply found for you based on public information:**<br><br>---<br><br>**"
    c. **If currently "Working Hours"**: Just use a simple greeting.
        * **Prefix Example**: "Hello, regarding your question, the reply is as follows:**<br><br>---<br><br>**"
    d. Concatenate the generated prefix and the core reply you drafted (result of step 3) to form the final email body.

5.  **Formatted Output**: You must output the finally generated email content in a strict JSON format. The format is as follows, do not add any additional explanations or text:
    {
      "shouldReply": true,
      "subject": "Re: [Original Email Subject]",
      "body": "[Here is the concatenated, complete email reply body, **all line breaks must use HTML <br> tags**]"
    }

# Rules and Restrictions
- **Always Try to Answer First**: At any time, your primary task is to use `SerpAPI` to provide valuable replies to users.
- **Must Declare Status**: If replying during non-working hours, you must clearly state this at the beginning of the email and attach my exact working hours.
- **Information Sources Must Be Accurate**: Working hours must strictly follow the results of `Simple Vector Store2`; question answers mainly come from `SerpAPI`, do not fabricate information.
- **Output Format**: **In the final output JSON, all line breaks in the `body` field must use `<br>` tags, not `\n`.**
```

(3) Agent Tools 구성

`Simple Vector Store` tool에는 앞서 저장한 knowledge를 올바르게 "읽을" 수 있도록 핵심 구성이 필요합니다.

- **Operation Mode**: `Retrieve Documents (As Tool for AI Agent)`(Agent tool로서의 읽기 모드)
- **Memory Key**: 1부와 **정확히 동일한** Key 입력, 즉 `my_private_knowledge`
- **Embeddings**: 1부와 **정확히 동일한** `Embeddings Google Gemini` 모델 사용

`Memory Key`와 `Embeddings` 모델이 완전히 일치해야 Agent가 올바른 "키"와 "언어"로 knowledge base에 접근할 수 있습니다. 그림 5.48.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-11.png" alt="" width="90%"/>
  <p>그림 5.48 Simple Vector Store Tool Configuration</p>
</div>

Description 파라미터는 AI Agent가 tool을 호출할 때의 설명 정의입니다. 해당 Prompt는 다음과 같습니다.

```json
This is the Simple Vector Store2 tool, used to query my personal information, especially my working hours and email reply policy. When you need to determine whether it is currently working hours, or need to inform the other party when I will reply to emails, you must use this tool.
```

Memory의 경우, 각 mailbox thread name을 고유 식별자로 사용해 저장 고유성을 보장합니다. Key는 `{{ $('Gmail').item.json.threadId }}`로 설정합니다.

(4) 최종 답장 발송

마지막 단계는 실행입니다. `AI Agent` node 출력을 `Gmail` node에 연결하고 **Operation**을 `Send`로 설정합니다. n8n expression으로 수신자, 제목, 본문을 `AI Agent`가 출력한 JSON 데이터의 해당 필드와 연결해 자동 이메일 답장을 구현합니다. 그림 5.49.

- **To**: `{{ $('Gmail').item.json.From }}`(또는 다른 trigger의 sender field)
- **Subject**: `Re:  {{ $('Gmail').item.json.Subject }}`
- **Message**: `{{ $json.output }}`

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-12.png" alt="" width="90%"/>
  <p>그림 5.49 최종 답장 Tool 다이어그램</p>
</div>

발송에 성공하면 개인 메일함에서 실제 회신 이메일 정보도 받을 수 있습니다. 그림 5.50.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-13.png" alt="" width="90%"/>
  <p>그림 5.50 개인 메일함 회신 이메일 형식</p>
</div>

이 시점에서 `AI Agent` node 기반의 통합 지능형 고객 지원이 완성되었습니다. 테스트 이메일을 보내 동작을 검증할 수 있습니다. 이 아키텍처는 확장성이 매우 강합니다. 앞으로 `AI Agent` node에 calendar, database, CRM 등 더 많은 tool을 직접 추가하고, Prompt에서 사용법만 가르치면 Agent 역량을 계속 강화할 수 있습니다.

### 5.4.5 n8n의 장점과 한계 분석

지능형 이메일 어시스턴트를 처음부터 구축하는 실습을 통해 n8n의 작동 방식을 직관적으로 이해했습니다. 강력한 로우코드 자동화 플랫폼으로서 n8n은 Agent 애플리케이션 개발에 뛰어나지만 만능은 아닙니다. 표 5.1처럼 장점과 잠재적 한계를 객관적으로 분석해 보겠습니다.

<div align="center">
  <p>표 5.1 n8n 플랫폼 장단점 요약</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/5-figures/n8n-14.png" alt="" width="90%"/>
</div>

먼저 n8n의 가장 큰 장점은 **개발 효율**에 있습니다. 복잡한 로직을 직관적인 시각적 workflow로 추상화합니다. 이메일 수신, AI 의사결정, tool 호출, 최종 답장까지 전체 데이터 흐름과 처리 체인이 캔버스에서 한눈에 보입니다. 이런 로우코드 특성은 기술 장벽을 크게 낮춰, 개발자가 Agent 핵심 로직을 빠르게 구축·검증할 수 있게 하고 아이디어에서 프로토타입까지의 거리를 크게 줄입니다.

둘째, 플랫폼은 **강력하고 통합성이 높습니다**. n8n은 Gmail, Google Gemini 등 수백 개의 일반 서비스를 쉽게 연결할 수 있는 풍부한 내장 node library를 갖추고 있습니다. 더 중요한 것은 고급 `AI Agent` node가 model, memory, tool 관리를 높은 수준으로 통합해, 복잡한 자율 의사결정을 하나의 node로 구현할 수 있다는 점입니다. 이는 전통적인 다중 node 수동 라우팅보다 훨씬 우아하고 강력합니다. 동시에 내장 기능으로 커버되지 않는 시나리오에서는 `Code` node로 커스텀 코드를 작성할 수 있어 기능 상한도 보장합니다.

마지막으로 **배포와 운영** 측면에서 n8n은 **private deployment**를 지원하며, 현재 비교적 간단하게 전체 프로젝트를 배포할 수 있는 private Agent 솔루션 중 하나입니다. 데이터 보안과 프라이버시를 중시하는 기업에 매우 중요합니다. 전체 서비스를 자체 서버에 배포해 내부 이메일, 고객 데이터 같은 민감 정보가 외부로 나가지 않게 할 수 있어 Agent 애플리케이션의 컴플라이언스 기반을 제공합니다.

물론 모든 도구에는 trade-off가 있습니다. n8n의 편리함을 누리면서도 한계를 인식해야 합니다.

**개발 효율** 뒤에는 **상대적으로 번거로운 디버깅과 오류 처리**가 있습니다. workflow가 복잡해지면 데이터 형식 오류가 발생했을 때 각 node의 input/output을 하나씩 확인해야 할 수 있어, 코드에서 breakpoint를 거는 것만큼 직접적이지 않을 때가 있습니다.

기능 측면에서 가장 큰 한계는 **내장 저장소의 비영속성**입니다. 사례에서 사용한 `Simple Memory`와 `Simple Vector Store`는 모두 memory 기반이라, n8n 서비스가 재시작되면 모든 대화 기록과 knowledge base가 사라집니다. 프로덕션 환경에는 치명적이므로, 실제 배포에서는 Redis, Pinecone 같은 외부 persistent DB로 교체해야 하며, 추가 구성·유지보수 비용도 발생합니다.

또한 **배포·운영**과 팀 협업 측면에서 n8n의 **버전 관리와 다인 협업은 전통적 코드만큼 성숙하지 않습니다**. workflow를 JSON으로 내보낼 수는 있지만, 변경 비교는 `git diff`만큼 명확하지 않고, 여러 사람이 동시에 같은 workflow를 편집하면 충돌이 쉽게 발생합니다.

마지막으로 **성능** 측면에서 n8n은 대부분의 기업 자동화와 중·저빈도 Agent 작업에는 충분합니다. 그러나 초고동시 요청을 처리해야 하는 시나리오에서는 node scheduling mechanism이 성능 오버헤드를 가져올 수 있어, 순수 코드 구현 서비스보다 다소 열세일 수 있습니다.

## 5.5 장 요약

이 장에서는 로우코드 플랫폼 기반 agent 애플리케이션 구축의 개념, 방법, 실습을 체계적으로 소개하며, "직접 코드 작성"에서 "플랫폼 기반 개발"로의 중요한 전환을 다룹니다.

첫 절에서는 로우코드 플랫폼 부상의 배경과 가치를 설명했습니다. 4장의 순수 코드 구현 agent와 비교하면, 로우코드 플랫폼은 그래픽·모듈형 방식으로 기술 장벽을 낮추고 개발 효율을 높이며 더 나은 시각적 디버깅 경험을 제공합니다. 이런 "더 높은 수준의 추상화"는 개발자가 저수준 구현 세부사항보다 비즈니스 로직과 prompt engineering에 집중하게 합니다.

이어서 특색 있는 세 대표 플랫폼을 깊이 실습했습니다.

**Coze**는 zero-code 친화적 경험과 풍부한 plugin ecosystem으로 두각을 나타냅니다. "Daily AI Brief" 사례를 통해 drag-and-drop 구성으로 다중 소스 정보를 빠르게 통합하고 여러 주류 플랫폼에 원클릭 배포하는 방법을 경험했습니다. Coze는 비기술 배경 사용자와 아이디어를 빠르게 검증해야 하는 시나리오에 특히 적합하지만, MCP 미지원과 표준화된 구성 파일 export 불가 같은 한계도 주목할 만합니다.

**Dify**는 오픈소스 enterprise-level platform으로 풀스택 개발 역량을 보여 줍니다. "Super Agent Personal Assistant" 사례는 daily Q&A, copywriting optimization, multimodal generation, data analysis, MCP tool integration 등 여러 모듈을 아우르며, 복잡한 비즈니스 시나리오에서 Dify의 강력한 orchestration 역량을 충분히 보여 줍니다. 풍부한 plugin market(8000+), 유연한 배포 방식, enterprise-level security feature 덕분에 professional developer와 enterprise team에게 이상적 선택입니다. 다만 상대적으로 가파른 learning curve와 고동시성 시나리오의 performance challenge도 함께 고려해야 합니다.

**n8n**은 독특한 "connection" 역량으로 또 다른 길을 엽니다. "Intelligent Email Assistant" 사례에서 AI 역량을 복잡한 business automation process에 seamless하게 끼워 넣는 방법을 보았습니다. n8n의 AI Agent node는 model, memory, tool을 높은 수준으로 통합하고, 수백 개의 preset node와 결합해 고도로 맞춤화된 automation solution을 구현할 수 있습니다. private deployment 지원은 data security를 중시하는 enterprise에 특히 중요합니다. 다만 내장 storage의 non-persistence와 version control의 미성숙은 production environment에서 추가 engineering processing이 필요합니다.

세 플랫폼의 비교 실습을 통해 다음 선택 가이드를 도출할 수 있습니다.
- **빠른 prototype validation, 비기술 사용자**: Coze 우선
- **enterprise-level application, 복잡한 business logic**: Dify 우선
- **deep business integration, automation process**: n8n 우선

강조할 점은, 로우코드 플랫폼이 code development를 대체하려는 것이 아니라 complementary choice를 제공한다는 것입니다. 실제 프로젝트에서는 단계별 요구에 따라 유연하게 전환할 수 있습니다. 아이디어 검증에는 low-code platform, fine-grained control에는 code; 표준화된 process에는 platform, special logic에는 code. 이런 "hybrid development" 사고방식이 agent engineering의 best practice입니다.

다음 장에서는 더 저수준 agent framework를 추가로 탐구하여, 독자가 더 안정적이고 흥미로운 애플리케이션을 구축할 수 있도록 돕겠습니다.

## 연습 문제

1. 이 장에서는 Coze, Dify, n8n 세 가지 특색 있는 로우코드 플랫폼을 소개했습니다. 다음을 분석해 보세요.

   - 세 플랫폼의 core positioning과 design philosophy 차이는 무엇인가요? 각각 agent development의 어떤 pain point를 해결하나요?
   - low-code platform과 pure code development는 각각 장단점이 있습니다. 또한 일부 기능은 platform, 일부는 code로 구현하는 "hybrid development" 모드도 있습니다. 세 개발 모드가 각각 어떤 시나리오에 적합한지 생각하고 예를 들어 설명해 보세요.

2. 5.2절 Coze 사례에서 "Daily AI Brief" agent를 구축했습니다. 이 사례를 바탕으로 확장해 보세요.

   > **Tip**: hands-on practice 문제이므로 실제 조작을 권장합니다.

   - 현재 brief generation은 passive trigger(사용자가 직접 질문)입니다. 이 agent를 매일 오전 8시에 brief를 자동 생성해 지정 Feishu group 또는 WeChat official account로 push하도록 바꾸려면?
   - brief quality는 prompt design에 크게 의존합니다. 5.2.2절 prompt를 최적화해 더 전문적이고 구조가 명확한 brief를 만들거나, "hot spot analysis", "trend prediction" 같은 새 기능을 추가해 보세요.
   - Coze가 현재 `MCP` protocol을 지원하지 않는 것은 중요한 limitation으로 여겨집니다(연습 문제 작성 시점에는 [`Coze Studio Q4 2025 Product Roadmap`](https://github.com/coze-dev/coze-studio/issues/2218)에 `feature-mcp`가 있었지만 아직 구현되지 않음). `MCP` protocol이 무엇인지, 왜 중요한지 간단히 설명하고, 향후 Coze가 MCP를 지원하면 어떤 새 가능성이 열릴지 서술해 보세요.

3. 5.3절 Dify 사례에서 풀기능 "Super Agent Personal Assistant"를 구축했습니다. 다음을 심층 분석해 보세요.

   - 사례는 "question classifier"로 intelligent routing을 사용해 서로 다른 요청을 sub-agent에 분배합니다. 이 multi-agent architecture의 장점은 무엇인가요? classifier 없이 single agent가 모든 task를 처리하면 어떤 문제가 생기나요?
   - data query module은 LLM에 명확한 table structure information을 제공해야 합니다. DB에 50개 table, 각 20개 field가 있다면 모든 `DDL`을 prompt에 넣으면 context가 너무 길어집니다. 이 문제를 해결할 smarter solution을 설계해 보세요.
   - Dify는 local deployment와 cloud deployment를 모두 지원합니다. data security, cost, performance, maintenance difficulty 측면에서 두 모드를 비교하고, 각각의 applicable scenario를 설명해 보세요.

4. 5.4절 n8n 사례에서 "Intelligent Email Assistant"를 구축했습니다. 다음을 생각해 보세요.

   > **Tip**: hands-on practice 문제이므로 실제 조작을 권장합니다.

   - 사례의 `Simple Vector Store`와 `Simple Memory`는 모두 memory 기반이라 service restart 후 data가 사라집니다. `n8n` documentation을 참고해 `Pinecone`, `Redis` 등 persistent storage solution으로 교체해 보고, configuration process를 설명해 보세요.
   - 현재 email assistant는 text email만 처리합니다. 사용자가 `PDF` document, image 같은 attachment를 보내면, workflow를 어떻게 확장해 agent가 attachment content를 이해하고 답장할 수 있게 할까요?
   - `n8n`의 core advantage는 "connection" 역량입니다. 더 복잡한 automation scenario를 설계해 보세요. e-commerce platform에서 customer가 order를 넣으면 confirmation email 전송, inventory DB update, logistics system notification, `CRM`에 customer information 기록 등 일련의 작업이 자동 트리거됩니다. workflow의 node connection diagram을 그리고 key configuration을 설명해 보세요.

5. prompt engineering은 low-code platform에서도 equally crucial합니다. 이 장의 여러 platform prompt design 사례를 분석해 보세요.

   - 5.2.2절(`Coze`), 5.3.2절(`Dify`), 5.4.4절(`n8n`) prompt design을 비교해 보세요. structure, style, focus의 차이는 무엇인가요? 이런 차이는 platform characteristics와 관련이 있나요?
   - `Dify` "Copywriting Optimization Module" prompt는 output이 "500 words를 초과"하도록 요구합니다. output length에 대한 이런 hard requirement가 reasonable한가요? output length를 제한해야 하는 상황과 model이 freely express하도록 해야 하는 상황은 각각 언제인가요?

6. tool과 plugin은 low-code platform의 core capability extension method입니다. 다음을 생각해 보세요.

   - `Coze`는 풍부한 plugin store, `Dify`는 8000+ plugin market, `n8n`은 수백 개의 preset node를 갖추고 있습니다. 세 platform 모두 필요한 specific tool(예: "회사 내부 system `API` 연결")이 없다면 어떻게 해결할까요?
   - 5.3.2절에서 `MCP` protocol로 Amap, dietary recommendation 등 service를 통합했습니다. 조사해 설명해 보세요. `MCP` protocol과 traditional `RESTful API`, `Tool Calling`의 차이는 무엇인가요? `MCP`가 agent tool invocation의 "new standard"로 불리는 이유는?
   - `Dify`용 custom plugin을 개발해 회사 내부 knowledge base system을 호출하게 하려면, `Dify` plugin development documentation을 참고해 development process와 key technical points를 outline해 보세요.

7. platform selection은 agent product 성공의 key decision 중 하나입니다. startup company의 technical leader라고 가정하고, 다음 세 AI application 각각에 가장 suitable한 platform(`Coze`, `Dify`, `n8n`, 또는 pure code development)을 선택하고 상세히 설명해 보세요.

   **Application A**: C-end user용 "AI Writing Assistant" mini-program. market demand 검증을 위해 빠르게 launch해야 하고 budget이 제한적이며, team에는 front-end engineer 1명과 product manager 1명만 있습니다.

   **Application B**: enterprise customer용 "Intelligent Contract Review System". sensitive legal document를 처리해야 하고 data가 customer private environment를 벗어나면 안 되며, customer existing OA system과 document management system과 deep integration이 필요합니다.

   **Application C**: internal "R&D Efficiency Improvement Tool". code review, test report generation, bug tracking, project progress synchronization 등 여러 R&D process link를 automate해야 하며, team technical capability가 강합니다.

   각 application에 대해 다음 dimension(포함하되 이에 국한되지 않음)으로 분석하세요.

   > **Tip**: platform capability가 requirement를 충족하는지, launch speed, development cost, operating cost, subsequent iteration difficulty, future function expansion space

   - Technical feasibility
   - Development efficiency
   - Cost control
   - Maintainability
   - Scalability
   - Data security and compliance

## 참고 문헌

[1] Coze - Next-generation AI application development platform. https://www.coze.cn/

[2] Dify - Open-source LLM application development platform. https://dify.ai/

[3] n8n - Workflow automation tool. https://n8n.io/


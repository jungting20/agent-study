# 8장 메모리와 검색 (Memory and Retrieval)

이전 장들에서 우리는 HelloAgents 프레임워크의 기본 아키텍처를 구축하고 다양한 에이전트 패러다임과 도구 시스템을 구현했습니다. 하지만 우리 프레임워크에는 여전히 중요한 기능이 빠져 있습니다. 바로 **메모리(Memory)**입니다. 에이전트가 이전의 상호작용을 기억하지 못하거나 역사적 경험으로부터 배우지 못한다면, 지속적인 대화나 복잡한 작업을 수행할 때 성능이 크게 제한될 것입니다.

이 장에서는 7장에서 구축한 프레임워크를 기반으로 HelloAgents에 두 가지 핵심 기능인 **메모리 시스템(Memory System)**과 **검색 증강 생성(RAG, Retrieval-Augmented Generation)**을 추가할 것입니다. 우리는 "프레임워크 확장 + 지식 대중화" 방식을 채택하여, 구축 과정에서 메모리와 RAG의 이론적 토대를 깊이 이해하고, 최종적으로 완전한 메모리 및 지식 검색 기능을 갖춘 에이전트 시스템을 구현할 것입니다.


## 8.1 인지 과학에서 에이전트 메모리까지

### 8.1.1 인간 메모리 시스템에서의 영감

에이전트의 메모리 시스템을 구축하기 전에,
먼저 인지 과학 관점에서 인간이 정보를 어떻게 처리하고 저장하는지 이해해 봅시다.
인간의 메모리는 단순히 정보를 저장하는 것이 아니라 중요성, 시간, 맥락에 따라 정보를 분류하고 조직하는 다층적 인지 시스템입니다.
인지 심리학은 메모리의 구조와 프로세스를 이해하기 위한 클래식한 이론적 프레임워크를 제공하며<sup>[1]</sup>, 이는 그림 8.1에 나와 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/8-figures/8-1.png" alt="인간 메모리 시스템 구조" width="85%"/>
  <p>그림 8.1 인간 메모리 시스템의 계층적 구조</p>
</div>

인지 심리학 연구에 따르면 인간의 메모리는 다음과 같은 단계로 나눌 수 있습니다:

1. **감각 메모리 (Sensory Memory)**: 매우 짧은 지속 시간 (0.5~3초), 거대한 용량, 감각기관을 통해 수신된 모든 정보를 일시적으로 저장하는 역할
2. **작업 메모리 (Working Memory)**: 짧은 지속 시간 (15~30초), 제한된 용량 (7±2개 아이템), 현재 작업에서의 정보 처리를 담당
3. **장기 메모리 (Long-term Memory)**: 긴 지속 시간 (평생 지속 가능), 거의 무제한의 용량, 다음과 같이 세분화됨:
   - **절차적 메모리 (Procedural Memory)**: 기술과 습관 (예: 자전거 타기)
   - **선언적 메모리 (Declarative Memory)**: 언어로 표현할 수 있는 지식, 다음과 같이 세분화됨:
     - **의미론적 메모리 (Semantic Memory)**: 일반적인 지식과 개념 (예: "파리는 프랑스의 수도이다")
     - **일화적 메모리 (Episodic Memory)**: 개인적인 경험과 사건 (예: "어제 회의 내용")

### 8.1.2 에이전트에게 메모리와 RAG가 필요한 이유

인간 메모리 시스템의 설계를 참조하면, 에이전트에게도 유사한 메모리 기능이 필요한 이유를 이해할 수 있습니다.
인간 지능의 중요한 특징은 과거의 경험을 기억하고, 그것으로부터 배우며, 이러한 경험을 새로운 상황에 적용하는 능력입니다.
마찬가지로 진정으로 지능적인 에이전트에게도 메모리 기능이 필요합니다.
LLM 기반 에이전트의 경우, 일반적으로 두 가지 근본적인 한계인 **대화 상태의 망각**과 **내장 지식의 한계**에 직면하게 됩니다.

(1) 한계 1: 무상태성(Statelessness)으로 인한 대화 망각

현재의 대형 언어 모델은 강력하지만 **무상태(stateless)**로 설계되었습니다.
이는 각각의 사용자 요청(또는 API 호출)이 독립적이고 서로 연관되지 않은 계산임을 의미합니다.
모델 자체는 이전 대화의 내용을 자동으로 "기억"하지 않습니다. 이는 다음과 같은 여러 문제를 야기합니다:

1. **맥락 손실 (Context Loss)**: 긴 대화에서 중요한 초기 정보가 컨텍스트 윈도우 한계로 인해 유실될 수 있음
2. **개인화 부족 (Lack of Personalization)**: 에이전트가 사용자의 선호도, 습관 또는 구체적인 요구사항을 기억하지 못함
3. **제한된 학습 능력 (Limited Learning Ability)**: 과거의 성공이나 실패로부터 배우고 개선할 수 없음
4. **일관성 문제 (Consistency Issues)**: 다턴 대화에서 모순된 답변을 제공할 수 있음

구체적인 예를 통해 이 문제를 이해해 봅시다:

```python
# 7장의 Agent 사용 방법
from hello_agents import SimpleAgent, HelloAgentsLLM

agent = SimpleAgent(name="학습 도우미", llm=HelloAgentsLLM())

# 첫 번째 대화
response1 = agent.run("제 이름은 홍길동이고, 파이썬을 배우고 있으며 기초 문법을 마스터했습니다.")
print(response1)  # "좋습니다! 파이썬 기초 문법은 프로그래밍의 중요한 기초입니다..."
 
# 두 번째 대화 (새 세션)
response2 = agent.run("제 학습 진도를 기억하시나요?")
print(response2)  # "죄송합니다, 귀하의 학습 진도를 알지 못합니다..."
```

이 문제를 해결하기 위해 우리 프레임워크는 메모리 시스템을 도입해야 합니다.

(2) 한계 2: 모델 내장 지식의 한계

대화 기록을 망각하는 것 외에도 LLM의 또 다른 핵심 한계는 지식이 **정적이고 제한적**이라는 점입니다.
이 지식은 전적으로 학습 데이터에서 비롯되므로 다음과 같은 문제를 수반합니다:

1. **지식의 시의성 (Knowledge Timeliness)**: 대형 모델은 학습 데이터 차단일(cutoff date)이 있어 최신 정보에 액세스할 수 없음
2. **도메인 특화 지식 (Domain-Specific Knowledge)**: 범용 모델은 특정 도메인에서 깊이가 부족할 수 있음
3. **사실적 정확성 (Factual Accuracy)**: 검색 검증을 통해 모델의 환각(hallucination) 현상을 줄여야 함
4. **설명 가능성 (Explainability)**: 답변의 신뢰성을 높이기 위해 정보 출처를 제공해야 함

이러한 한계를 극복하기 위해 RAG 기술이 등장했습니다. 핵심 아이디어는 모델이 답변을 생성하기 전에 외부 지식 베이스(예: 문서, 데이터베이스, API)에서 가장 관련성 높은 정보를 검색하고, 이 정보를 모델에 컨텍스트로 제공하는 것입니다.

### 8.1.3 메모리 및 RAG 시스템 아키텍처 설계

7장에서 확립한 프레임워크 기반과 인지 과학의 영감을 바탕으로 계층화된 메모리 및 RAG 시스템 아키텍처를 설계했습니다. 이는 그림 8.2에 나와 있습니다. 이 아키텍처는 인간 메모리 시스템의 계층적 구조를 반영할 뿐만 아니라 공학적 구현의 확장성도 충분히 고려했습니다. 구현상 우리는 메모리와 RAG를 두 개의 독립적인 도구로 설계합니다. `memory_tool`은 대화 중에 상호작용 정보를 저장하고 유지하는 역할을 담당하며, `rag_tool`은 사용자가 제공한 지식 베이스에서 관련 정보를 검색하여 컨텍스트로 제공하는 역할을 담당합니다. 또한 RAG 도구는 중요 검색 결과를 메모리 시스템에 자동으로 저장할 수 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/8-figures/8-2.png" alt="HelloAgents 메모리 및 RAG 시스템 아키텍처" width="95%"/>
  <p>그림 8.2 HelloAgents 메모리 및 RAG 시스템의 전체 아키텍처</p>
</div>

메모리 시스템은 4개 계층의 아키텍처 설계를 채택합니다:

```
HelloAgents 메모리 시스템
├── 인프라 계층 (Infrastructure Layer)
│   ├── MemoryManager - 메모리 관리자 (통합 스케줄링 및 조정)
│   ├── MemoryItem - 메모리 데이터 구조 (표준화된 메모리 아이템)
│   ├── MemoryConfig - 설정 관리 (시스템 매개변수 설정)
│   └── BaseMemory - 메모리 기반 클래스 (공통 인터페이스 정의)
├── 메모리 유형 계층 (Memory Types Layer)
│   ├── WorkingMemory - 작업 메모리 (임시 정보, TTL 관리)
│   ├── EpisodicMemory - 일화적 메모리 (구체적 사건, 시계열)
│   ├── SemanticMemory - 의미론적 메모리 (추상 지식, 그래프 관계)
│   └── PerceptualMemory - 지각 메모리 (다중 모달 데이터)
├── 저장 백엔드 계층 (Storage Backend Layer)
│   ├── QdrantVectorStore - 벡터 저장소 (고성능 의미론적 검색)
│   ├── Neo4jGraphStore - 그래프 저장소 (지식 그래프 관리)
│   └── SQLiteDocumentStore - 문서 저장소 (구조화된 영속성)
└── 임베딩 서비스 계층 (Embedding Service Layer)
    ├── DashScopeEmbedding - 알리바바 클라우드 임베딩 (클라우드 API)
    ├── LocalTransformerEmbedding - 로컬 임베딩 (오프라인 배포)
    └── TFIDFEmbedding - TFIDF 임베딩 (경량 백업용)
```

RAG 시스템은 외부 지식의 획득 및 활용에 초점을 맞춥니다:

```
HelloAgents RAG 시스템
├── 문서 처리 계층 (Document Processing Layer)
│   ├── DocumentProcessor - 문서 처리기 (다중 포맷 파싱)
│   ├── Document - 문서 객체 (메타데이터 관리)
│   └── Pipeline - RAG 파이프라인 (엔드투엔드 처리)
├── 임베딩 계층 (Embedding Layer)
│   └── Unified Embedding Interface - 메모리 시스템의 임베딩 서비스 재사용
├── 벡터 저장 계층 (Vector Storage Layer)
│   └── QdrantVectorStore - 벡터 데이터베이스 (네임스페이스 격리)
└── 지능형 Q&A 계층 (Intelligent Q&A Layer)
    ├── 다중 전략 검색 - 벡터 검색 + MQE + HyDE
    ├── 컨텍스트 구축 - 지능형 프래그먼트 병합 및 잘라내기
    └── LLM 강화 생성 - 맥락 기반의 정확한 Q&A
```

### 8.1.4 학습 목표 및 빠른 체험

먼저 8장의 핵심 학습 내용을 살펴보겠습니다:

```
hello-agents/
├── hello_agents/
│   ├── memory/                   # 메모리 시스템 모듈
│   │   ├── base.py               # 기본 데이터 구조 (MemoryItem, MemoryConfig, BaseMemory)
│   │   ├── manager.py            # 메모리 관리자 (통합 조정 및 스케줄링)
│   │   ├── embedding.py          # 통합 임베딩 서비스 (DashScope/Local/TFIDF)
│   │   ├── types/                # 메모리 유형 구현
│   │   │   ├── working.py        # 작업 메모리 (TTL 관리, 순수 인메모리)
│   │   │   ├── episodic.py       # 일화적 메모리 (이벤트 시퀀스, SQLite+Qdrant)
│   │   │   ├── semantic.py       # 의미론적 메모리 (지식 그래프, Qdrant+Neo4j)
│   │   │   └── perceptual.py     # 지각 메모리 (다중 모달, SQLite+Qdrant)
│   │   ├── storage/              # 저장 백엔드 구현
│   │   │   ├── qdrant_store.py   # Qdrant 벡터 저장소 (고성능 벡터 검색)
│   │   │   ├── neo4j_store.py    # Neo4j 그래프 저장소 (지식 그래프 관리)
│   │   │   └── document_store.py # SQLite 문서 저장소 (구조화된 영속성)
│   │   └── rag/                  # RAG 시스템
│   │       ├── pipeline.py       # RAG 파이프라인 (엔드투엔드 처리)
│   │       └── document.py       # 문서 처리기 (다중 포맷 파싱)
│   └── tools/builtin/            # 확장 빌트인 도구
│       ├── memory_tool.py        # 메모리 도구 (에이전트 메모리 기능)
│       └── rag_tool.py           # RAG 도구 (지능형 Q&A 기능)
└──
```

**빠른 시작: HelloAgents 프레임워크 설치**

독자들이 이번 장의 완전한 기능을 빠르게 경험할 수 있도록 직접 설치 가능한 Python 패키지를 제공합니다. 다음 명령을 통해 이번 장에 해당하는 버전을 설치할 수 있습니다:

```bash
# 버전 0.2.0에서 모델을 사용할 수 없는 문제가 발생하면 issue#320을 참조하거나 테스트를 위해 0.2.9 버전으로 전환하십시오.
pip install "hello-agents[all]==0.2.0"
python -m spacy download zh_core_web_sm
python -m spacy download en_core_web_sm
```

또한, `.env` 파일에 그래프 데이터베이스, 벡터 데이터베이스, LLM 및 임베딩 솔루션 API를 설정해야 합니다.
본 튜토리얼에서는 벡터 데이터베이스로 Qdrant를, 그래프 데이터베이스로 Neo4J를 사용하며,
임베딩으로는 Bailian(百炼) 플랫폼을 선호합니다. 사용 가능한 API가 없으면 로컬 배포 모델 솔루션으로 전환할 수 있습니다.

```bash
# ================================
# Qdrant 벡터 데이터베이스 설정 - API 키 획득: https://cloud.qdrant.io/
# ================================
# Qdrant 클라우드 서비스 사용 (권장)
QDRANT_URL=https://your-cluster.qdrant.tech:6333
QDRANT_API_KEY=your_qdrant_api_key_here

# 또는 로컬 Qdrant 사용 (Docker 필요)
# QDRANT_URL=http://localhost:6333
# QDRANT_API_KEY=

# Qdrant 컬렉션 설정
QDRANT_COLLECTION=hello_agents_vectors
QDRANT_VECTOR_SIZE=384
QDRANT_DISTANCE=cosine
QDRANT_TIMEOUT=30

# ================================
# Neo4j 그래프 데이터베이스 설정 - API 키 획득: https://neo4j.com/cloud/aura/
# ================================
# Neo4j Aura 클라우드 서비스 사용 (권장)
NEO4J_URI=neo4j+s://your-instance.databases.neo4j.io
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your_neo4j_password_here

# 또는 로컬 Neo4j 사용 (Docker 필요)
# NEO4J_URI=bolt://localhost:7687
# NEO4J_USERNAME=neo4j
# NEO4J_PASSWORD=hello-agents-password

# Neo4j 연결 설정
NEO4J_DATABASE=neo4j
NEO4J_MAX_CONNECTION_LIFETIME=3600
NEO4J_MAX_CONNECTION_POOL_SIZE=50
NEO4J_CONNECTION_TIMEOUT=60

# ==========================
# 임베딩 설정 예시 - 알리바바 클라우드 콘솔에서 획득: https://dashscope.aliyun.com/
# ==========================
# - 비어 있으면 dashscope는 기본적으로 text-embedding-v3를 사용하고, 로컬은 sentence-transformers/all-MiniLM-L6-v2를 사용합니다.
EMBED_MODEL_TYPE=dashscope
EMBED_MODEL_NAME=
EMBED_API_KEY=
EMBED_BASE_URL=
```

이번 장의 학습은 두 가지 방식으로 진행할 수 있습니다:

1. **체험 학습**: `pip`를 사용해 프레임워크를 직접 설치하고, 예제 코드를 실행하며 다양한 기능을 신속하게 경험
2. **심화 학습**: 장의 내용을 따라가며 각 컴포넌트를 처음부터 구현하고, 프레임워크의 설계 철학과 구현 세부사항을 깊이 이해

우리는 "체험 먼저, 그 후 구현"의 학습 경로를 채택하는 것을 권장합니다. 이 장에서는 완전한 테스트 파일을 제공합니다. 핵심 기능을 재작성하고 테스트를 실행하여 구현이 올바른지 확인할 수 있습니다.

7장에서 확립한 설계 원칙에 따라, 우리는 새로운 에이전트 클래스를 만들기보다 메모리와 RAG 기능을 표준 도구로 캡슐화합니다. 시작하기 전에 Hello-agents를 사용해 메모리와 RAG 기능을 갖춘 에이전트를 구축하는 30초 체험을 해봅시다!

```python
# 동일 폴더의 .env에 LLM API 설정
from hello_agents import SimpleAgent, HelloAgentsLLM, ToolRegistry
from hello_agents.tools import MemoryTool, RAGTool

# LLM 인스턴스 생성
llm = HelloAgentsLLM()

# 에이전트 생성
agent = SimpleAgent(
    name="지능형 조수",
    llm=llm,
    system_prompt="당신은 메모리 및 지식 검색 기능을 갖춘 AI 조수입니다."
)

# 도구 레지스트리 생성
tool_registry = ToolRegistry()

# 메모리 도구 추가
memory_tool = MemoryTool(user_id="user123")
tool_registry.register_tool(memory_tool)

# RAG 도구 추가
rag_tool = RAGTool(knowledge_base_path="./knowledge_base")
tool_registry.register_tool(rag_tool)

# 에이전트에 도구 설정
agent.tool_registry = tool_registry

# 대화 시작
response = agent.run("안녕하세요! 제 이름은 홍길동이고 파이썬 개발자라는 것을 기억해주세요.")
print(response)
```

모든 것이 올바르게 설정되었다면 다음과 같은 내용을 볼 수 있습니다:

```bash
[OK] SQLite database tables and indexes created
[OK] SQLite document storage initialized: ./memory_data\memory.db
INFO:hello_agents.memory.storage.qdrant_store:✅ Successfully connected to Qdrant cloud service: https://0c517275-2ad0-4442-8309-11c36dc7e811.us-east-1-1.aws.cloud.qdrant.io:6333
INFO:hello_agents.memory.storage.qdrant_store:✅ Using existing Qdrant collection: hello_agents_vectors
INFO:hello_agents.memory.types.semantic:✅ Embedding model ready, dimension: 1024
INFO:hello_agents.memory.types.semantic:✅ Qdrant vector database initialization complete
INFO:hello_agents.memory.storage.neo4j_store:✅ Successfully connected to Neo4j cloud service: neo4j+s://851b3a28.databases.neo4j.io
INFO:hello_agents.memory.types.semantic:✅ Neo4j graph database initialization complete
INFO:hello_agents.memory.storage.neo4j_store:✅ Neo4j index creation complete
INFO:hello_agents.memory.types.semantic:✅ Neo4j graph database initialization complete
INFO:hello_agents.memory.types.semantic:🏥 Database health status: Qdrant=✅, Neo4j=✅
INFO:hello_agents.memory.types.semantic:✅ Loaded Chinese spaCy model: zh_core_web_sm
INFO:hello_agents.memory.types.semantic:✅ Loaded English spaCy model: en_core_web_sm
INFO:hello_agents.memory.types.semantic:📚 Available language models: Chinese, English
INFO:hello_agents.memory.types.semantic:Enhanced semantic memory initialization complete (using Qdrant+Neo4j professional databases)
INFO:hello_agents.memory.manager:MemoryManager initialization complete, enabled memory types: ['working', 'episodic', 'semantic']
✅ Tool 'memory' registered.
INFO:hello_agents.memory.storage.qdrant_store:✅ Successfully connected to Qdrant cloud service: https://0c517275-2ad0-4442-8309-11c36dc7e811.us-east-1-1.aws.cloud.qdrant.io:6333
INFO:hello_agents.memory.storage.qdrant_store:✅ Using existing Qdrant collection: rag_knowledge_base
✅ RAG tool initialization successful: namespace=default, collection=rag_knowledge_base
✅ Tool 'rag' registered.
안녕하세요, 홍길동님! 만나서 반갑습니다. 파이썬 개발자이신 만큼 프로그래밍에 대한 열정이 남다르실 것 같습니다. 기술적인 질문이 있으시거나 파이썬 관련 주제로 대화가 필요하시다면 언제든 편하게 말씀해 주세요. 최선을 다해 돕겠습니다. 지금 바로 도와드릴 일이 있을까요?
```


## 8.2 메모리 시스템: 에이전트에게 메모리 부여하기

### 8.2.1 메모리 시스템 워크플로우

코드 구현 단계에 들어가기 전에, 먼저 메모리 시스템의 워크플로우를 정의해야 합니다. 이 워크플로우는 인지 과학의 메모리 모델을 참조하여 각 인지 단계를 특정 기술 컴포넌트 및 작업에 매핑합니다. 이 매핑 관계를 이해하는 것은 이후의 코드 구현에 큰 도움이 될 것입니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/8-figures/8-3.png" alt="메모리 형성 과정" width="90%"/>
  <p>그림 8.3 메모리 형성의 인지적 과정</p>
</div>

그림 8.3에 나타난 바와 같이 인지 과학 연구에 따르면 인간 메모리의 형성은 다음과 같은 단계를 거칩니다:

1. **부호화 (Encoding)**: 인지한 정보를 저장 가능한 형태로 변환
2. **저장 (Storage)**: 부호화된 정보를 메모리 시스템에 저장
3. **검색 (Retrieval)**: 필요에 따라 메모리에서 관련 정보를 추출
4. **공고화 (Consolidation)**: 단기 메모리를 장기 메모리로 변환
5. **망각 (Forgetting)**: 중요하지 않거나 오래된 정보를 삭제

이러한 영감을 바탕으로 HelloAgents를 위한 완전한 메모리 시스템을 설계했습니다. 핵심 아이디어는 인간의 뇌가 서로 다른 유형의 정보를 처리하는 방식을 모방하여 메모리를 여러 전문화된 모듈로 분할하고 지능적인 관리 메커니즘을 구축하는 것입니다. 그림 8.4는 메모리 추가, 검색, 공고화, 망각 등 주요 연결 고리를 포함한 이 시스템의 상세 워크플로우를 보여줍니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/8-figures/8-4.png" alt="메모리 시스템 워크플로우" width="95%"/>
  <p>그림 8.4 HelloAgents 메모리 시스템의 전체 워크플로우</p>
</div>

우리 메모리 시스템은 네 가지 유형의 메모리 모듈로 구성되며, 각 모듈은 특정 애플리케이션 시나리오와 생명주기에 최적화되어 있습니다:

첫째는 **작업 메모리 (Working Memory)**입니다. 에이전트의 "단기 메모리" 역할을 하며, 주로 현재 대화의 맥락 정보를 저장하는 데 사용됩니다. 빠른 액세스와 응답 속도를 보장하기 위해 용량은 의도적으로 제한되며(예: 기본 50개 항목), 생명주기는 단일 세션에 바인딩되어 세션이 종료되면 자동으로 지워집니다.

둘째는 **일화적 메모리 (Episodic Memory)**입니다. 구체적인 상호작용 이벤트와 에이전트의 학습 경험을 장기적으로 저장하는 역할을 담당합니다. 작업 메모리와 달리 일화적 메모리는 풍부한 맥락 정보를 포함하며, 시계열이나 주제별 회고 검색을 지원하여 에이전트가 과거 경험을 "복기"하고 학습할 수 있는 기반이 됩니다.

구체적인 이벤트에 대응하는 것은 **의미론적 메모리 (Semantic Memory)**로, 보다 추상적인 지식, 개념, 규칙을 저장합니다. 예를 들어 대화를 통해 파악한 사용자 선호도, 장기적으로 준수해야 하는 지침, 도메인 지식 포인트 등이 여기에 저장되기에 적합합니다. 이 부분의 메모리는 높은 영속성과 중요성을 가지며, 에이전트가 "지식 체계"를 형성하고 연상 추론을 수행하는 핵심이 됩니다.

마지막으로, 점점 더 풍부해지는 멀티미디어와 상호작용하기 위해 **지각 메모리 (Perceptual Memory)**를 도입했습니다. 이 모듈은 이미지, 오디오와 같은 다중 모달 정보를 특화하여 처리하며 교차 모달 검색을 지원합니다. 그 생명주기는 정보의 중요성과 사용 가능한 저장 공간을 바탕으로 동적으로 관리됩니다.

### 8.2.2 빠른 체험: 30초 만에 시작하는 메모리 기능

구현 세부사항을 살펴보기 전에, 메모리 시스템의 기본 기능을 신속하게 경험해 봅시다:

```python
from hello_agents import SimpleAgent, HelloAgentsLLM, ToolRegistry
from hello_agents.tools import MemoryTool

# 메모리 기능을 갖춘 에이전트 생성
llm = HelloAgentsLLM()
agent = SimpleAgent(name="메모리 도우미", llm=llm)

# 메모리 도구 생성
memory_tool = MemoryTool(user_id="user123")
tool_registry = ToolRegistry()
tool_registry.register_tool(memory_tool)
agent.tool_registry = tool_registry

# 메모리 기능 체험
print("=== 다중 메모리 추가 ===")

# 첫 번째 메모리 추가
result1 = memory_tool.execute("add", content="사용자 홍길동은 머신러닝과 데이터 분석에 집중하는 파이썬 개발자입니다.", memory_type="semantic", importance=0.8)
print(f"메모리 1: {result1}")

# 두 번째 메모리 추가
result2 = memory_tool.execute("add", content="이몽룡은 React와 Vue.js 개발에 능숙한 프론트엔드 엔지니어입니다.", memory_type="semantic", importance=0.7)
print(f"메모리 2: {result2}")

# 세 번째 메모리 추가
result3 = memory_tool.execute("add", content="성춘향은 사용자 경험 디자인과 요구사항 분석을 담당하는 제품 매니저입니다.", memory_type="semantic", importance=0.6)
print(f"메모리 3: {result3}")

print("\n=== 특정 메모리 검색 ===")
# 프론트엔드 관련 메모리 검색
print("🔍 '프론트엔드 엔지니어' 검색:")
result = memory_tool.execute("search", query="프론트엔드 엔지니어", limit=3)
print(result)

print("\n=== 메모리 요약 ===")
result = memory_tool.execute("summary")
print(result)
```
### 8.2.3 MemoryTool 상세 설명

이제 하향식(top-down) 접근 방식을 채택하여 MemoryTool에서 지원하는 구체적인 작업부터 시작해 내부 구현을 단계적으로 살펴보겠습니다. MemoryTool은 메모리 시스템의 통합 인터페이스로서 "통합 입력, 분산 처리" 아키텍처 패턴을 따릅니다:

```python
def execute(self, action: str, **kwargs) -> str:
    """메모리 작업 실행

    지원되는 작업:
    - add: 메모리 추가 (working/episodic/semantic/perceptual 4가지 유형 지원)
    - search: 메모리 검색
    - summary: 메모리 요약 획득
    - stats: 통계 정보 획득
    - update: 메모리 업데이트
    - remove: 메모리 삭제
    - forget: 메모리 망각 (여러 전략 지원)
    - consolidate: 메모리 공고화 (단기 → 장기)
    - clear_all: 모든 메모리 삭제
    """

    if action == "add":
        return self._add_memory(**kwargs)
    elif action == "search":
        return self._search_memory(**kwargs)
    elif action == "summary":
        return self._get_summary(**kwargs)
    # ... 기타 작업들
```

이 통합된 `execute` 인터페이스 설계는 에이전트의 호출 방식을 단순화합니다. 구체적인 작업은 `action` 매개변수를 통해 지정되며, `**kwargs`를 통해 각 작업마다 서로 다른 매개변수 요구사항을 수용할 수 있습니다. 여기서는 몇 가지 중요한 작업을 살펴보겠습니다:

(1) 작업 1: add

`add` 작업은 메모리 시스템의 근간입니다. 인간의 뇌가 인지한 정보를 메모리로 인코딩하는 과정을 모방합니다. 구현 시에는 단순히 메모리 콘텐츠만 저장하는 것이 아니라 각 메모리에 풍부한 맥락(context) 정보를 추가해야 합니다. 이 정보는 이후의 검색 및 관리에서 중요한 역할을 합니다.

```python
def _add_memory(
    self,
    content: str = "",
    memory_type: str = "working",
    importance: float = 0.5,
    file_path: str = None,
    modality: str = None,
    **metadata
) -> str:
    """메모리 추가"""
    try:
        # 세션 ID가 존재하는지 확인
        if self.current_session_id is None:
            self.current_session_id = f"session_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

        # 지각 메모리 파일 지원
        if memory_type == "perceptual" and file_path:
            inferred = modality or self._infer_modality(file_path)
            metadata.setdefault("modality", inferred)
            metadata.setdefault("raw_data", file_path)

        # 메타데이터에 세션 정보 추가
        metadata.update({
            "session_id": self.current_session_id,
            "timestamp": datetime.now().isoformat()
        })

        memory_id = self.memory_manager.add_memory(
            content=content,
            memory_type=memory_type,
            importance=importance,
            metadata=metadata,
            auto_classify=False
        )

        return f"✅ 메모리 추가 완료 (ID: {memory_id[:8]}...)"

    except Exception as e:
        return f"❌ 메모리 추가 실패: {str(e)}"
```

이 메서드는 주로 세 가지 핵심 작업을 수행합니다: 세션 ID 자동 관리(각 메모리가 명확한 세션에 귀속되도록 보장), 다중 모달 데이터의 지능형 처리(파일 유형을 자동 추론하고 관련 메타데이터 저장), 그리고 맥락 정보의 자동 보완(각 메모리에 타임스탬프와 세션 정보 추가)입니다. 그중에서도 `importance` 매개변수(기본값 0.5)는 메모리의 중요도를 표시하는 데 사용되며 값의 범위는 0.0에서 1.0 사이입니다. 이 매개변수는 인간의 뇌가 서로 다른 정보의 중요성을 평가하는 방식을 모방합니다. 이러한 설계 덕분에 에이전트는 서로 다른 시기의 대화를 자동으로 구별하고, 향후 검색 및 관리를 위한 풍부한 맥락 정보를 제공할 수 있게 됩니다.

각 메모리 유형별 사용 예시는 다음과 같습니다:

```python
# 1. 작업 메모리 - 임시 정보, 제한된 용량
memory_tool.execute("add",
    content="사용자가 방금 파이썬 함수에 대해 질문했습니다.",
    memory_type="working",
    importance=0.6
)

# 2. 일화적 메모리 - 구체적인 사건과 경험
memory_tool.execute("add",
    content="2024년 3월 15일, 사용자 홍길동이 자신의 첫 파이썬 프로젝트를 완료했습니다.",
    memory_type="episodic",
    importance=0.8,
    event_type="milestone",
    location="온라인 학습 플랫폼"
)

# 3. 의미론적 메모리 - 추상적인 지식과 개념
memory_tool.execute("add",
    content="파이썬은 인터프리터 방식의 객체 지향 프로그래밍 언어입니다.",
    memory_type="semantic",
    importance=0.9,
    knowledge_type="factual"
)

# 4. 지각 메모리 - 다중 모달 정보
memory_tool.execute("add",
    content="사용자가 함수 정의가 포함된 파이썬 코드 스크린샷을 업로드했습니다.",
    memory_type="perceptual",
    importance=0.7,
    modality="image",
    file_path="./uploads/code_screenshot.png"
)
```

(2) 작업 2: search

`search` 작업은 메모리 시스템의 핵심 기능입니다. 수많은 메모리 중에서 검색어(query)와 가장 관련성 높은 콘텐츠를 신속하게 찾아내야 합니다. 여기에는 의미론적 이해, 관련성 계산, 결과 정렬 등 여러 단계가 포함됩니다.

```python
def _search_memory(
    self,
    query: str,
    limit: int = 5,
    memory_types: List[str] = None,
    memory_type: str = None,
    min_importance: float = 0.1
) -> str:
    """메모리 검색"""
    try:
        # 매개변수 표준화
        if memory_type and not memory_types:
            memory_types = [memory_type]

        results = self.memory_manager.retrieve_memories(
            query=query,
            limit=limit,
            memory_types=memory_types,
            min_importance=min_importance
        )

        if not results:
            return f"🔍 '{query}'와(과) 관련된 메모리를 찾을 수 없습니다."

        # 결과 포맷팅
        formatted_results = []
        formatted_results.append(f"🔍 관련 메모리 {len(results)}개를 찾았습니다:")

        for i, memory in enumerate(results, 1):
            memory_type_label = {
                "working": "작업 메모리",
                "episodic": "일화적 메모리",
                "semantic": "의미론적 메모리",
                "perceptual": "지각 메모리"
            }.get(memory.memory_type, memory.memory_type)

            content_preview = memory.content[:80] + "..." if len(memory.content) > 80 else memory.content
            formatted_results.append(
                f"{i}. [{memory_type_label}] {content_preview} (중요도: {memory.importance:.2f})"
            )

        return "\n".join(formatted_results)

    except Exception as e:
        return f"❌ 메모리 검색 실패: {str(e)}"
```

검색 작업은 매개변수의 단수형과 복수형(`memory_type` 및 `memory_types`)을 모두 지원하도록 설계되어 사용자가 가장 자연스러운 방식으로 요구사항을 표현할 수 있게 합니다. 그중에서도 `min_importance` 매개변수(기본값 0.1)는 저품질 메모리를 필터링하는 데 사용됩니다. 검색 기능의 사용법은 다음 예시를 참고하세요:

```python
# 기본 검색
result = memory_tool.execute("search", query="파이썬 프로그래밍", limit=5)

# 특정 메모리 유형 지정 검색
result = memory_tool.execute("search",
    query="학습 진도",
    memory_type="episodic",
    limit=3
)

# 다중 유형 검색
result = memory_tool.execute("search",
    query="함수 정의",
    memory_types=["semantic", "episodic"],
    min_importance=0.5
)
```

(3) 작업 3: forget

망각 메커니즘은 가장 인지 과학적인 특징을 보여주는 기능입니다. 인간 뇌의 선택적 망각 과정을 모방하며 다음 세 가지 전략을 지원합니다: 중요도 기반(중요하지 않은 메모리 삭제), 시간 기반(오래된 메모리 삭제), 용량 기반(저장 공간이 한계에 도달하면 가장 덜 중요한 메모리부터 삭제).

```python
def _forget(self, strategy: str = "importance_based", threshold: float = 0.1, max_age_days: int = 30) -> str:
    """메모리 망각 (다양한 전략 지원)"""
    try:
        count = self.memory_manager.forget_memories(
            strategy=strategy,
            threshold=threshold,
            max_age_days=max_age_days
        )
        return f"🧹 메모리 {count}개를 망각했습니다 (전략: {strategy})"
    except Exception as e:
        return f"❌ 메모리 망각 실패: {str(e)}"
```

**세 가지 망각 전략의 사용 방법:**

```python
# 1. 중요도 기반 망각 - 중요도 임계값 미만의 메모리 삭제
memory_tool.execute("forget",
    strategy="importance_based",
    threshold=0.2
)

# 2. 시간 기반 망각 - 지정된 일수보다 오래된 메모리 삭제
memory_tool.execute("forget",
    strategy="time_based",
    max_age_days=30
)

# 3. 용량 기반 망각 - 메모리 개수가 제한을 초과하면 가장 중요도가 낮은 것부터 삭제
memory_tool.execute("forget",
    strategy="capacity_based",
    threshold=0.3
)
```

(4) 작업 4: consolidate

```python
def _consolidate(self, from_type: str = "working", to_type: str = "episodic", importance_threshold: float = 0.7) -> str:
    """메모리 공고화 (중요한 단기 메모리를 장기 메모리로 격상)"""
    try:
        count = self.memory_manager.consolidate_memories(
            from_type=from_type,
            to_type=to_type,
            importance_threshold=importance_threshold,
        )
        return f"🔄 메모리 {count}개를 장기 메모리로 공고화했습니다 ({from_type} → {to_type}, 임계값={importance_threshold})"
    except Exception as e:
        return f"❌ 메모리 공고화 실패: {str(e)}"
```

공고화(consolidate) 작업은 신경 과학의 메모리 공고화 개념을 도입하여, 인간의 뇌가 단기 메모리를 장기 메모리로 변환하는 과정을 모방합니다. 기본 설정은 중요도가 0.7을 초과하는 작업 메모리를 일화적 메모리로 변환하는 것입니다. 이 임계값은 진정으로 중요한 정보만 장기적으로 보존되도록 보장합니다. 전체 프로세스는 자동화되어 있으며 사용자가 특정 메모리를 수동으로 선택할 필요가 없습니다. 시스템이 조건에 맞는 메모리를 지능적으로 식별하여 유형 변환을 수행합니다.

**메모리 공고화의 사용 예시:**

```python
# 중요한 작업 메모리를 일화적 메모리로 변환
memory_tool.execute("consolidate",
    from_type="working",
    to_type="episodic",
    importance_threshold=0.7
)

# 중요한 일화적 메모리를 의미론적 메모리로 변환
memory_tool.execute("consolidate",
    from_type="episodic",
    to_type="semantic",
    importance_threshold=0.8
)
```

이러한 핵심 작업들의 협업을 통해 MemoryTool은 완전한 메모리 생명주기 관리 시스템을 구축합니다. 메모리 생성, 검색, 요약부터 망각, 공고화 및 관리에 이르기까지 폐쇄 루프 형태의 지능형 메모리 관리 시스템을 형성하여 에이전트에게 진정으로 인간다운 메모리 능력을 부여합니다.

### 8.2.4 MemoryManager 상세 설명

MemoryTool의 인터페이스 설계를 이해했으니, 이제 내부 구현을 들여다보고 MemoryTool이 MemoryManager와 어떻게 협업하는지 살펴봅시다. 이러한 계층화된 설계는 소프트웨어 공학의 '관심사 분리(separation of concerns)' 원칙을 반영합니다. MemoryTool은 사용자 인터페이스와 매개변수 처리에 집중하고, MemoryManager는 핵심 메모리 관리 로직을 담당합니다.

MemoryTool은 초기화 중에 MemoryManager 인스턴스를 생성하고 설정에 따라 다양한 유형의 메모리 모듈을 활성화합니다. 이 설계를 통해 사용자는 구체적인 요구사항에 따라 활성화할 메모리 유형을 선택할 수 있으며, 기능적 완결성을 확보하는 동시에 불필요한 자원 소모를 방지할 수 있습니다.

```python
class MemoryTool(Tool):
    """메모리 도구 - 에이전트에게 메모리 기능 제공"""

    def __init__(
        self,
        user_id: str = "default_user",
        memory_config: MemoryConfig = None,
        memory_types: List[str] = None
    ):
        super().__init__(
            name="memory",
            description="메모리 도구 - 대화 기록, 지식 및 경험을 저장하고 검색할 수 있습니다."
        )

        # 메모리 관리자 초기화
        self.memory_config = memory_config or MemoryConfig()
        self.memory_types = memory_types or ["working", "episodic", "semantic"]

        self.memory_manager = MemoryManager(
            config=self.memory_config,
            user_id=user_id,
            enable_working="working" in self.memory_types,
            enable_episodic="episodic" in self.memory_types,
            enable_semantic="semantic" in self.memory_types,
            enable_perceptual="perceptual" in self.memory_types
        )
```

MemoryManager는 메모리 시스템의 핵심 조정자로서 서로 다른 유형의 메모리 모듈을 관리하고 통합된 작업 인터페이스를 제공하는 책임을 집니다.

```python
class MemoryManager:
    """메모리 관리자 - 통합 메모리 작업 인터페이스"""

    def __init__(
        self,
        config: Optional[MemoryConfig] = None,
        user_id: str = "default_user",
        enable_working: bool = True,
        enable_episodic: bool = True,
        enable_semantic: bool = True,
        enable_perceptual: bool = False
    ):
        self.config = config or MemoryConfig()
        self.user_id = user_id

        # 저장 및 검색 컴포넌트 초기화
        self.store = MemoryStore(self.config)
        self.retriever = MemoryRetriever(self.store, self.config)

        # 다양한 유형의 메모리 초기화
        self.memory_types = {}

        if enable_working:
            self.memory_types['working'] = WorkingMemory(self.config, self.store)

        if enable_episodic:
            self.memory_types['episodic'] = EpisodicMemory(self.config, self.store)

        if enable_semantic:
            self.memory_types['semantic'] = SemanticMemory(self.config, self.store)

        if enable_perceptual:
            self.memory_types['perceptual'] = PerceptualMemory(self.config, self.store)
```

### 8.2.5 네 가지 유형의 메모리

이제 네 가지 메모리 유형의 구체적인 구현을 살펴봅시다. 각 메모리 유형은 고유한 특성과 적용 시나리오를 갖추고 있습니다:

(1) 작업 메모리 (Working Memory)

작업 메모리는 메모리 시스템에서 가장 활발하게 작동하는 부분입니다. 현재 대화 세션의 임시 정보를 저장하는 책임을 집니다. 작업 메모리의 설계 초점은 빠른 액세스와 자동 정리(cleanup)에 있으며, 이를 통해 시스템의 응답 속도와 자원 효율성을 보장합니다.

작업 메모리는 순수 인메모리(in-memory) 저장 솔루션을 채택하며, 자동 정리를 위해 TTL(Time To Live) 메커니즘을 결합합니다. 이 설계의 장점은 극도로 빠른 액세스 속도이지만, 시스템이 재시작되면 작업 메모리의 내용이 소실됨을 의미하기도 합니다. 이러한 특징은 작업 메모리의 포지셔닝(임시적이고 휘발성 있는 정보의 저장)에 완벽하게 부합합니다.

```python
class WorkingMemory:
    """작업 메모리 구현
    특징:
    - 제한된 용량 (기본 50개 항목) + TTL 자동 정리
    - 순수 인메모리 저장으로 극도로 빠른 액세스 제공
    - 하이브리드 검색: TF-IDF 벡터화 + 키워드 매칭
    """

    def __init__(self, config: MemoryConfig):
        self.max_capacity = config.working_memory_capacity or 50
        self.max_age_minutes = config.working_memory_ttl or 60
        self.memories = []

    def add(self, memory_item: MemoryItem) -> str:
        """작업 메모리 추가"""
        self._expire_old_memories()  # 만료된 데이터 정리

        if len(self.memories) >= self.max_capacity:
            self._remove_lowest_priority_memory()  # 용량 관리

        self.memories.append(memory_item)
        return memory_item.id

    def retrieve(self, query: str, limit: int = 5, **kwargs) -> List[MemoryItem]:
        """하이브리드 검색: TF-IDF 벡터화 + 키워드 매칭"""
        self._expire_old_memories()

        # TF-IDF 벡터 검색 시도
        vector_scores = self._try_tfidf_search(query)

        # 종합 점수 계산
        scored_memories = []
        for memory in self.memories:
            vector_score = vector_scores.get(memory.id, 0.0)
            keyword_score = self._calculate_keyword_score(query, memory.content)

            # 하이브리드 스코어링
            base_relevance = vector_score * 0.7 + keyword_score * 0.3 if vector_score > 0 else keyword_score
            time_decay = self._calculate_time_decay(memory.timestamp)
            importance_weight = 0.8 + (memory.importance * 0.4)

            final_score = base_relevance * time_decay * importance_weight
            if final_score > 0:
                scored_memories.append((final_score, memory))

        scored_memories.sort(key=lambda x: x[0], reverse=True)
        return [memory for _, memory in scored_memories[:limit]]
```

작업 메모리 검색은 하이브리드 검색 전략을 채택합니다. 먼저 의미론적 검색을 위해 TF-IDF 벡터화를 시도하고, 실패할 경우 키워드 매칭으로 대체합니다. 이 설계는 다양한 환경에서 안정적인 검색 서비스를 제공하도록 돕습니다. 스코어링 알고리즘은 의미적 유사성, 시간 감쇠(time decay), 중요도 가중치를 결합합니다. 최종 점수 공식은 `(유사성 × 시간 감쇠) × (0.8 + 중요도 × 0.4)`입니다.

(2) 일화적 메모리 (Episodic Memory)

일화적 메모리는 구체적인 사건과 경험을 저장하는 역할을 합니다. 설계의 초점은 사건의 완전성과 시계열 관계를 유지하는 것입니다. 일화적 메모리는 SQLite + Qdrant의 하이브리드 저장 솔루션을 사용합니다. SQLite는 구조화된 데이터 저장 및 복잡한 쿼리를 담당하고, Qdrant는 효율적인 벡터 검색을 수행합니다.

```python
class EpisodicMemory:
    """일화적 메모리 구현
    특징:
    - SQLite + Qdrant 하이브리드 저장 아키텍처
    - 시계열 및 세션 레벨의 검색 지원
    - 구조화된 필터링 + 의미론적 벡터 검색
    """

    def __init__(self, config: MemoryConfig):
        self.doc_store = SQLiteDocumentStore(config.database_path)
        self.vector_store = QdrantVectorStore(config.qdrant_url, config.qdrant_api_key)
        self.embedder = create_embedding_model_with_fallback()
        self.sessions = {}  # 세션 인덱스

    def add(self, memory_item: MemoryItem) -> str:
        """일화적 메모리 추가"""
        # 일화(episode) 객체 생성
        episode = Episode(
            episode_id=memory_item.id,
            session_id=memory_item.metadata.get("session_id", "default"),
            timestamp=memory_item.timestamp,
            content=memory_item.content,
            context=memory_item.metadata
        )

        # 세션 인덱스 업데이트
        session_id = episode.session_id
        if session_id not in self.sessions:
            self.sessions[session_id] = []
        self.sessions[session_id].append(episode.episode_id)

        # 영속성 저장 (SQLite + Qdrant)
        self._persist_episode(episode)
        return memory_item.id

    def retrieve(self, query: str, limit: int = 5, **kwargs) -> List[MemoryItem]:
        """하이브리드 검색: 구조화된 필터링 + 의미론적 벡터 검색"""
        # 1. 구조화된 사전 필터링 (시간 범위, 중요도 등)
        candidate_ids = self._structured_filter(**kwargs)

        # 2. 벡터 의미론적 검색
        hits = self._vector_search(query, limit * 5, kwargs.get("user_id"))

        # 3. 종합 스코어링 및 정렬
        results = []
        for hit in hits:
            if self._should_include(hit, candidate_ids, kwargs):
                score = self._calculate_episode_score(hit)
                memory_item = self._create_memory_item(hit)
                results.append((score, memory_item))

        results.sort(key=lambda x: x[0], reverse=True)
        return [item for _, item in results[:limit]]

    def _calculate_episode_score(self, hit) -> float:
        """일화적 메모리 스코어링 알고리즘"""
        vec_score = float(hit.get("score", 0.0))
        recency_score = self._calculate_recency(hit["metadata"]["timestamp"])
        importance = hit["metadata"].get("importance", 0.5)

        # 스코어링 공식: (벡터 유사성 × 0.8 + 시간적 최신성 × 0.2) × 중요도 가중치
        base_relevance = vec_score * 0.8 + recency_score * 0.2
        importance_weight = 0.8 + (importance * 0.4)

        return base_relevance * importance_weight
```

일화적 메모리의 검색 구현은 복잡한 다요소 스코어링 메커니즘을 보여줍니다. 의미적 유사성뿐만 아니라 시간적 최신성(temporal recency)에 대한 고려 사항도 통합하여 최종적으로 중요도 가중치로 조정됩니다. 스코어링 공식은 `(벡터 유사성 × 0.8 + 시간적 최신성 × 0.2) × (0.8 + 중요도 × 0.4)`로, 검색 결과가 의미적 및 시간적으로 모두 관련성이 높도록 보장합니다.

(3) 의미론적 메모리 (Semantic Memory)

의미론적 메모리는 메모리 시스템에서 가장 복잡한 부분입니다. 추상적인 개념, 규칙, 지식을 저장하는 역할을 담당합니다. 의미론적 메모리의 설계 초점은 지식의 구조화된 표현과 지능적인 추론 능력에 있습니다. 의미론적 메모리는 Neo4j 그래프 데이터베이스와 Qdrant 벡터 데이터베이스의 하이브리드 아키텍처를 채택합니다. 이 설계를 통해 시스템은 지식 그래프를 활용하여 빠른 의미론적 검색뿐만 아니라 복잡한 관계 추론을 수행할 수 있습니다.

```python
class SemanticMemory(BaseMemory):
    """의미론적 메모리 구현

    특징:
    - 텍스트 임베딩을 위해 HuggingFace의 한국어 사전 학습 모델을 사용
    - 빠른 유사도 매칭을 위한 벡터 검색
    - 엔티티와 관계의 저장을 위한 지식 그래프 저장소
    - 하이브리드 검색 전략: 벡터 + 그래프 + 의미론적 추론
    """

    def __init__(self, config: MemoryConfig, storage_backend=None):
        super().__init__(config, storage_backend)

        # 임베딩 모델 (통합 제공)
        self.embedding_model = get_text_embedder()

        # 전문 데이터베이스 스토리지
        self.vector_store = QdrantConnectionManager.get_instance(**qdrant_config)
        self.graph_store = Neo4jGraphStore(**neo4j_config)

        # 엔티티 및 관계 캐시
        self.entities: Dict[str, Entity] = {}
        self.relations: List[Relation] = []

        # NLP 프로세서 (한국어 및 영어 지원)
        self.nlp = self._init_nlp()
```

의미론적 메모리의 추가 프로세스는 지식 그래프 구축의 전체 워크플로우를 구현합니다. 시스템은 메모리 콘텐츠를 단순히 저장하는 것이 아니라 엔티티(entity)와 관계(relationship)를 자동으로 추출하여 구조화된 지식 표현을 구축합니다:

```python
def add(self, memory_item: MemoryItem) -> str:
    """의미론적 메모리 추가"""
    # 1. 텍스트 임베딩 생성
    embedding = self.embedding_model.encode(memory_item.content)

    # 2. 엔티티 및 관계 추출
    entities = self._extract_entities(memory_item.content)
    relations = self._extract_relations(memory_item.content, entities)

    # 3. Neo4j 그래프 데이터베이스에 저장
    for entity in entities:
        self._add_entity_to_graph(entity, memory_item)

    for relation in relations:
        self._add_relation_to_graph(relation, memory_item)

    # 4. Qdrant 벡터 데이터베이스에 저장
    metadata = {
        "memory_id": memory_item.id,
        "entities": [e.entity_id for e in entities],
        "entity_count": len(entities),
        "relation_count": len(relations)
    }

    self.vector_store.add_vectors(
        vectors=[embedding.tolist()],
        metadata=[metadata],
        ids=[memory_item.id]
    )
```

의미론적 메모리의 검색은 벡터 검색의 의미 이해 기능과 그래프 검색의 관계 추론 기능을 결합한 하이브리드 검색 전략을 구현합니다:

```python
def retrieve(self, query: str, limit: int = 5, **kwargs) -> List[MemoryItem]:
    """의미론적 메모리 검색"""
    # 1. 벡터 검색
    vector_results = self._vector_search(query, limit * 2, user_id)

    # 2. 그래프 검색
    graph_results = self._graph_search(query, limit * 2, user_id)

    # 3. 하이브리드 랭킹
    combined_results = self._combine_and_rank_results(
        vector_results, graph_results, query, limit
    )

    return combined_results[:limit]
```

하이브리드 랭킹 알고리즘은 다요소 스코어링 메커니즘을 채택합니다:

```python
def _combine_and_rank_results(self, vector_results, graph_results, query, limit):
    """결과 하이브리드 랭킹"""
    combined = {}

    # 벡터 검색 결과와 그래프 검색 결과 머지
    for result in vector_results:
        combined[result["memory_id"]] = {
            **result,
            "vector_score": result.get("score", 0.0),
            "graph_score": 0.0
        }

    for result in graph_results:
        memory_id = result["memory_id"]
        if memory_id in combined:
            combined[memory_id]["graph_score"] = result.get("similarity", 0.0)
        else:
            combined[memory_id] = {
                **result,
                "vector_score": 0.0,
                "graph_score": result.get("similarity", 0.0)
            }

    # 하이브리드 점수 계산
    for memory_id, result in combined.items():
        vector_score = result["vector_score"]
        graph_score = result["graph_score"]
        importance = result.get("importance", 0.5)

        # 기본 관련성 점수
        base_relevance = vector_score * 0.7 + graph_score * 0.3

        # 중요도 가중치 [0.8, 1.2]
        importance_weight = 0.8 + (importance * 0.4)

        # 최종 점수: 유사도 * 중요도 가중치
        combined_score = base_relevance * importance_weight
        result["combined_score"] = combined_score

    # 정렬 및 반환
    sorted_results = sorted(
        combined.values(),
        key=lambda x: x["combined_score"],
        reverse=True
    )

    return sorted_results[:limit]
```

의미론적 메모리의 스코어링 공식은 `(벡터 유사도 × 0.7 + 그래프 유사도 × 0.3) × (0.8 + 중요도 × 0.4)`입니다. 이 설계의 핵심 아이디어는 다음과 같습니다:

- **벡터 검색 가중치 (0.7)**: 의미론적 유사성이 주된 요소이며, 검색 결과가 쿼리와 의미적으로 밀접하게 연관되도록 보장합니다.
- **그래프 검색 가중치 (0.3)**: 관계 추론을 보완 요소로 사용하여, 개념 간의 암시적인 연관성을 발견합니다.
- **중요도 가중치 범위 [0.8, 1.2]**: 중요도가 유사도 랭킹에 과도한 영향을 미치지 않도록 방지하여 검색의 정확성을 유지합니다.

(4) 지각 메모리 (Perceptual Memory)

지각 메모리는 텍스트, 이미지, 오디오 등 다양한 모달리티(modality)의 데이터 저장을 지원합니다. 모달리티별로 격리된 저장 전략을 채택하여 서로 다른 모달리티의 데이터에 대해 독립적인 벡터 컬렉션을 생성합니다. 이 설계는 차원 불일치(dimension mismatch) 문제를 방지하고 검색 정확성을 보장합니다:

```python
class PerceptualMemory(BaseMemory):
    """지각 메모리 구현

    특징:
    - 다중 모달 데이터 지원 (텍스트, 이미지, 오디오 등)
    - 교차 모달 유사도 검색
    - 지각 데이터의 의미론적 이해
    - 콘텐츠 생성 및 검색 지원
    """

    def __init__(self, config: MemoryConfig, storage_backend=None):
        super().__init__(config, storage_backend)

        # 다중 모달 인코더
        self.text_embedder = get_text_embedder()
        self._clip_model = self._init_clip_model()  # 이미지 인코딩
        self._clap_model = self._init_clap_model()  # 오디오 인코딩

        # 모달리티별 격리된 벡터 저장소
        self.vector_stores = {
            "text": QdrantConnectionManager.get_instance(
                collection_name="perceptual_text",
                vector_size=self.vector_dim
            ),
            "image": QdrantConnectionManager.get_instance(
                collection_name="perceptual_image",
                vector_size=self._image_dim
            ),
            "audio": QdrantConnectionManager.get_instance(
                collection_name="perceptual_audio",
                vector_size=self._audio_dim
            )
        }
```

지각 메모리 검색은 동일 모달리티 검색과 교차 모달리티 검색을 모두 지원합니다. 동일 모달리티 검색은 정밀한 매칭을 위해 전문 인코더를 사용하는 반면, 교차 모달리티 검색은 보다 복잡한 의미론적 정렬 메커니즘을 요구합니다:

```python
def retrieve(self, query: str, limit: int = 5, **kwargs) -> List[MemoryItem]:
    """지각 메모리 검색 (모달리티 필터링 가능, 동일 모달리티 벡터 검색 + 시간/중요도 융합)"""
    user_id = kwargs.get("user_id")
    target_modality = kwargs.get("target_modality")
    query_modality = kwargs.get("query_modality", target_modality or "text")

    # 동일 모달리티 벡터 검색
    try:
        query_vector = self._encode_data(query, query_modality)
        store = self._get_vector_store_for_modality(target_modality or query_modality)

        where = {"memory_type": "perceptual"}
        if user_id:
            where["user_id"] = user_id
        if target_modality:
            where["modality"] = target_modality

        hits = store.search_similar(
            query_vector=query_vector,
            limit=max(limit * 5, 20),
            where=where
        )
    except Exception:
        hits = []

    # 융합 랭킹 (벡터 유사도 + 시간적 최신성 + 중요도 가중치)
    results = []
    for hit in hits:
        vector_score = float(hit.get("score", 0.0))
        recency_score = self._calculate_recency_score(hit["metadata"]["timestamp"])
        importance = hit["metadata"].get("importance", 0.5)

        # 스코어링 알고리즘
        base_relevance = vector_score * 0.8 + recency_score * 0.2
        importance_weight = 0.8 + (importance * 0.4)
        combined_score = base_relevance * importance_weight

        results.append((combined_score, self._create_memory_item(hit)))

    results.sort(key=lambda x: x[0], reverse=True)
    return [item for _, item in results[:limit]]
```

지각 메모리의 스코어링 공식은 `(벡터 유사도 × 0.8 + 시간적 최신성 × 0.2) × (0.8 + 중요도 × 0.4)`입니다. 지각 메모리의 스코어링 메커니즘은 교차 모달 검색도 지원하여, 통합된 벡터 공간을 통해 텍스트, 이미지, 오디오 등 서로 다른 모달리티 데이터의 의미 정렬을 수행합니다. 교차 모달 검색을 진행할 때 시스템은 검색 결과의 다양성과 정확성을 확보하기 위해 스코어링 가중치를 자동으로 조정합니다. 추가로, 지각 메모리의 시간적 최신성 계산은 지수 감쇠(exponential decay) 모델을 채택합니다:

```python
def _calculate_recency_score(self, timestamp: str) -> float:
    """시간적 최신성 점수 계산"""
    try:
        memory_time = datetime.fromisoformat(timestamp)
        current_time = datetime.now()
        age_hours = (current_time - memory_time).total_seconds() / 3600

        # 지수 감쇠: 24시간 이내에는 높은 점수를 유지하다가 이후 서서히 감쇠
        decay_factor = 0.1  # 감쇠 계수
        recency_score = math.exp(-decay_factor * age_hours / 24)

        return max(0.1, recency_score)  # 최소 기본 점수 0.1 유지
    except Exception:
        return 0.5  # 기본 중간 점수
```

이 시간 감쇠 모델은 인간 기억의 망각 곡선을 모방한 것으로, 지각 메모리 시스템이 시간적으로 보다 연관성 높은 메모리 내용을 우선적으로 검색할 수 있도록 보장합니다.
## 8.3 RAG 시스템: 지식 검색 강화

### 8.3.1 RAG 기초

HelloAgents의 RAG 시스템 구현을 살펴보기 전에, 먼저 RAG 기술의 기본 개념, 발전 역사, 핵심 원리를 이해해 봅시다. 이 텍스트는 RAG를 전제로 작성된 것이 아니므로, 시스템 설계 시 기술적 선택과 혁신을 더 잘 이해할 수 있도록 관련 개념을 빠르게 훑어보겠습니다.

(1) RAG란 무엇인가?

검색 증강 생성(RAG, Retrieval-Augmented Generation)은 정보 검색과 텍스트 생성을 결합한 기술입니다. 핵심 아이디어는 에이전트가 답변을 생성하기 전에 먼저 외부 지식 베이스에서 관련 정보를 검색한 다음, 검색된 정보를 대형 언어 모델에 맥락(context)으로 제공하여 더욱 정확하고 신뢰할 수 있는 답변을 생성하도록 돕는 것입니다.

따라서 검색 증강 생성(Retrieval-Augmented Generation)은 세 단어로 나눌 수 있습니다. **Retrieval(검색)**은 지식 베이스에서 관련 콘텐츠를 조회하는 것을 뜻하고, **Augmented(증강)**는 검색 결과를 프롬프트에 통합하여 모델의 생성을 보조하는 것을 의미하며, **Generation(생성)**은 정확성과 투명성이 결합된 답변을 출력하는 것을 가리킵니다.

(2) 기본 워크플로우

완전한 RAG 애플리케이션 워크플로우는 크게 두 가지 핵심 단계로 나뉩니다. **데이터 준비 단계**에서 시스템은 **데이터 추출**, **텍스트 분할(segmentation)**, **벡터화(vectorization)**를 통해 외부 지식을 검색 가능한 데이터베이스로 구축합니다. 이어서 **애플리케이션 단계**에서 시스템은 사용자의 **쿼리**에 응답하고, 데이터베이스에서 관련 정보를 **검색**하며, 이를 **프롬프트에 주입**하고, 최종적으로 대형 언어 모델을 구동하여 **답변을 생성**합니다.

(3) 발전 역사

첫 번째 단계: 나이브 RAG (Naive RAG, 2020-2021). RAG 기술의 태동기로서 과정이 직접적이고 단순하여 흔히 "검색 후 읽기(Retrieve-Read)" 모드로 불립니다. **검색 방법**: 주로 `TF-IDF`나 `BM25`와 같은 전통적인 키워드 매칭 알고리즘에 의존합니다. 이 방법은 자구 일치 효과는 좋으나 단어 빈도와 문서 빈도를 계산하여 관련성을 평가하므로 의미론적 유사성을 이해하기는 어렵습니다. **생성 모드**: 검색된 문서 콘텐츠를 가공하지 않고 프롬프트 맥락에 직접 병합하여 생성 모델로 전송합니다.

두 번째 단계: 고급 RAG (Advanced RAG, 2022-2023). 벡터 데이터베이스와 텍스트 임베딩 기술이 성숙함에 따라 RAG는 급격한 발전 단계에 접어들었습니다. 연구자와 개발자들은 "검색"과 "생성"의 다양한 단계에서 많은 최적화 기법을 도입했습니다. **검색 방법**: **밀집 임베딩(dense embedding)**에 기반한 의미론적 검색으로 전환되었습니다. 텍스트를 고차원 벡터로 변환함으로써 모델은 키워드뿐만 아니라 의미론적 유사성도 이해하고 매칭할 수 있게 되었습니다. **생성 모드**: 쿼리 재작성, 문서 청킹, 리랭킹(reranking) 등 다양한 최적화 기술이 도입되었습니다.

세 번째 단계: 모듈형 RAG (Modular RAG, 2023-현재). 고급 RAG를 바탕으로 현대 RAG 시스템은 모듈화, 자동화, 지능화 방향으로 한층 더 발전했습니다. 시스템의 다양한 부분이 플러그인 형태로 조립 가능한 독립 모듈로 설계되어 더욱 복잡하고 다양한 애플리케이션 시나리오에 적응할 수 있게 되었습니다. **검색 방법**: 하이브리드 검색, 다중 쿼리 확장, 가상 문서 임베딩(HyDE) 등이 도입되었습니다. **생성 모드**: 생각의 사슬(CoT) 추론, 자기 반성 및 수정 등이 사용됩니다.

### 8.3.2 RAG 시스템 작동 원리

구현 세부사항을 들여다보기 전에, 흐름도를 통해 HelloAgents RAG 시스템의 전체 워크플로우를 정리해 보겠습니다:

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/8-figures/8-5.png" alt="RAG 시스템 핵심 원리" width="85%"/>
  <p>그림 8.5 RAG 시스템의 핵심 작동 원리</p>
</div>

그림 8.5에 표시된 바와 같이, 이는 RAG 시스템의 두 가지 주요 작업 모드를 보여줍니다:
1. **데이터 처리 워크플로우**: 지식 문서를 처리하고 저장합니다. 여기서는 `Markitdown` 도구를 채택하여 유입되는 모든 외부 지식 소스를 처리하기에 앞서 일관되게 마크다운 포맷으로 변환하는 디자인 아이디어를 적용했습니다.
2. **쿼리 및 생성 워크플로우**: 쿼리를 기반으로 관련 정보를 검색하고 답변을 생성합니다.

### 8.3.3 빠른 체험: 30초 만에 시작하는 RAG 기능

RAG 시스템의 기본 기능을 신속하게 경험해 봅시다:

```python
from hello_agents import SimpleAgent, HelloAgentsLLM, ToolRegistry
from hello_agents.tools import RAGTool

# RAG 기능을 갖춘 에이전트 생성
llm = HelloAgentsLLM()
agent = SimpleAgent(name="지식 도우미", llm=llm)

# RAG 도구 생성
rag_tool = RAGTool(
    knowledge_base_path="./knowledge_base",
    collection_name="test_collection",
    rag_namespace="test"
)

tool_registry = ToolRegistry()
tool_registry.register_tool(rag_tool)
agent.tool_registry = tool_registry

# RAG 기능 체험
# 첫 번째 지식 추가
result1 = rag_tool.execute("add_text",
    text="파이썬은 1991년 귀도 반 로섬(Guido van Rossum)에 의해 처음 발표된 고급 프로그래밍 언어입니다. 파이썬의 설계 철학은 코드 가독성과 간결한 문법을 강조합니다.",
    document_id="python_intro")
print(f"지식 1: {result1}")

# 두 번째 지식 추가
result2 = rag_tool.execute("add_text",
    text="머신러닝은 컴퓨터가 데이터로부터 패턴을 학습할 수 있도록 알고리즘을 사용하는 인공지능의 한 분야입니다. 주로 지도 학습, 비지도 학습, 강화 학습의 세 가지 유형이 있습니다.",
    document_id="ml_basics")
print(f"지식 2: {result2}")

# 세 번째 지식 추가
result3 = rag_tool.execute("add_text",
    text="RAG(검색 증강 생성)는 정보 검색과 텍스트 생성을 결합한 AI 기술입니다. 관련 지식을 검색하여 대형 언어 모델의 생성 능력을 향상시킵니다.",
    document_id="rag_concept")
print(f"지식 3: {result3}")


print("\n=== 지식 검색 ===")
result = rag_tool.execute("search",
    query="파이썬 프로그래밍 언어의 역사",
    limit=3,
    min_score=0.1
)
print(result)

print("\n=== 지식 베이스 통계 ===")
result = rag_tool.execute("stats")
print(result)
```

이어서 HelloAgents RAG 시스템의 구체적인 구현을 자세히 살펴보겠습니다.

### 8.3.4 RAG 시스템 아키텍처 설계

이 섹션에서는 메모리 시스템 설명과는 다른 접근 방식을 채택합니다. `Memory_tool`은 시스템적인 구현인 반면, 우리 설계에서 RAG는 파이프라인으로 구성할 수 있는 도구로 정의되기 때문입니다. RAG 시스템의 핵심 아키텍처는 "5개 계층 7단계" 설계 패턴으로 요약할 수 있습니다:

```
사용자 계층 (User Layer): RAGTool 통합 인터페이스
  ↓
애플리케이션 계층 (Application Layer): 지능형 Q&A, 검색, 관리
  ↓
처리 계층 (Processing Layer): 문서 파싱, 청킹, 벡터화
  ↓
저장 계층 (Storage Layer): 벡터 데이터베이스, 문서 저장소
  ↓
기반 계층 (Foundation Layer): 임베딩 모델, LLM, 데이터베이스
```

이러한 계층화 설계의 장점은 전체 시스템의 안정성을 유지하면서 각 계층을 독립적으로 최적화하고 교체할 수 있다는 점입니다. 예를 들어 상위 비즈니스 로직에 영향을 주지 않고 임베딩 모델을 sentence-transformers에서 알리바바 Bailian API로 손쉽게 바꿀 수 있습니다. 마찬가지로 처리 워크플로우 코드는 완벽하게 재사용 가능하므로, 필요한 부분만 선택해 자신의 프로젝트에 이식할 수 있습니다. RAGTool은 RAG 시스템의 통합 진입점 역할을 수행하며 간결한 API 인터페이스를 제공합니다.

```python
class RAGTool(Tool):
    """RAG 도구

    완전한 RAG 기능 제공:
    - 다중 포맷 문서 추가 (PDF, Office, 이미지, 오디오 등)
    - 지능형 검색 및 리콜(recall)
    - LLM 강화 Q&A
    - 지식 베이스 관리
    """

    def __init__(
        self,
        knowledge_base_path: str = "./knowledge_base",
        qdrant_url: str = None,
        qdrant_api_key: str = None,
        collection_name: str = "rag_knowledge_base",
        rag_namespace: str = "default"
    ):
        # RAG 파이프라인 초기화
        self._pipelines: Dict[str, Dict[str, Any]] = {}
        self.llm = HelloAgentsLLM()

        # 기본 파이프라인 생성
        default_pipeline = create_rag_pipeline(
            qdrant_url=self.qdrant_url,
            qdrant_api_key=self.qdrant_api_key,
            collection_name=self.collection_name,
            rag_namespace=self.rag_namespace
        )
        self._pipelines[self.rag_namespace] = default_pipeline
```

전체 처리 워크플로우는 다음과 같습니다:
```
임의 포맷 문서 → MarkItDown 변환 → 마크다운 텍스트 → 지능형 청킹 → 벡터화 → 저장 및 검색
```

(1) 다중 모달 문서 로드

RAG 시스템의 핵심 장점 중 하나는 강력한 다중 모달 문서 처리 능력입니다. 시스템은 MarkItDown을 통합 문서 변환 엔진으로 사용하여 거의 모든 공통 문서 포맷을 지원합니다. MarkItDown은 마이크로소프트에서 개발한 오픈소스 범용 문서 변환 도구입니다. HelloAgents RAG 시스템의 핵심 컴포넌트로서 임의 포맷의 문서를 구조화된 마크다운 텍스트로 일관되게 변환하는 책임을 집니다. 입력 형식이 PDF, Word, Excel이든 이미지나 오디오든 최종적으로 표준 마크다운 형식으로 변환되어 통합 청킹, 벡터화, 저장 워크플로우로 유입됩니다.

```python
def _convert_to_markdown(path: str) -> str:
    """
    MarkItDown을 사용하는 범용 문서 리더 (향상된 PDF 처리 기능 포함).
    핵심 기능: 임의 포맷의 문서를 마크다운 텍스트로 변환

    지원 포맷:
    - 문서: PDF, Word, Excel, PowerPoint
    - 이미지: JPG, PNG, GIF (OCR 이용)
    - 오디오: MP3, WAV, M4A (텍스트 변환 이용)
    - 텍스트: TXT, CSV, JSON, XML, HTML
    - 코드: Python, JavaScript, Java 등
    """
    if not os.path.exists(path):
        return ""

    # PDF 파일은 향상된 처리 방식 사용
    ext = (os.path.splitext(path)[1] or '').lower()
    if ext == '.pdf':
        return _enhanced_pdf_processing(path)

    # 기타 포맷은 MarkItDown 통합 변환 사용
    md_instance = _get_markitdown_instance()
    if md_instance is None:
        return _fallback_text_reader(path)

    try:
        result = md_instance.convert(path)
        markdown_text = getattr(result, "text_content", None)
        if isinstance(markdown_text, str) and markdown_text.strip():
            print(f"[RAG] MarkItDown 변환 성공: {path} -> {len(markdown_text)} 자의 마크다운")
            return markdown_text
        return ""
    except Exception as e:
        print(f"[WARNING] MarkItDown 변환 실패 {path}: {e}")
        return _fallback_text_reader(path)
```

(2) 지능형 청킹 전략

MarkItDown 변환을 거치고 나면 모든 문서는 표준 마크다운 형식으로 통일됩니다. 이는 이후의 지능형 청킹(chunking)을 위한 구조화된 기반을 제공합니다. HelloAgents는 마크다운 구조적 특징을 십분 활용해 정밀하게 분할하는 마크다운 전용 지능형 청킹 전략을 구현했습니다.

마크다운 구조 인지 청킹 워크플로우:

```
표준 마크다운 텍스트 → 제목 계층 파싱 → 단락 의미론적 분할 → 토큰 계산 청킹 → 오버랩 전략 최적화 → 벡터화 준비
       ↓                ↓              ↓            ↓           ↓            ↓
    통일된 형식      #/##/###      의미 경계 보장     크기 제어    정보 연속성 유지   임베딩 벡터
    명확한 구조      계층 인지       무결성 보장     검색 최적화     컨텍스트 보존    유사도 매칭
```

모든 문서가 마크다운 형식으로 변환되었기 때문에, 시스템은 마크다운의 제목 구조(#, ##, ### 등)를 사용해 정밀한 의미 분할을 수행할 수 있습니다:

```python
def _split_paragraphs_with_headings(text: str) -> List[Dict]:
    """제목 계층을 기반으로 단락을 분할하여 의미적 무결성 유지"""
    lines = text.splitlines()
    heading_stack: List[str] = []
    paragraphs: List[Dict] = []
    buf: List[str] = []
    char_pos = 0

    def flush_buf(end_pos: int):
        if not buf:
            return
        content = "\n".join(buf).strip()
        if not content:
            return
        paragraphs.append({
            "content": content,
            "heading_path": " > ".join(heading_stack) if heading_stack else None,
            "start": max(0, end_pos - len(content)),
            "end": end_pos,
        })

    for ln in lines:
        raw = ln
        if raw.strip().startswith("#"):
            # 제목 라인 처리
            flush_buf(char_pos)
            level = len(raw) - len(raw.lstrip('#'))
            title = raw.lstrip('#').strip()

            if level <= 0:
                level = 1
            if level <= len(heading_stack):
                heading_stack = heading_stack[:level-1]
            heading_stack.append(title)

            char_pos += len(raw) + 1
            continue

        # 단락 내용 축적
        if raw.strip() == "":
            flush_buf(char_pos)
            buf = []
        else:
            buf.append(raw)
        char_pos += len(raw) + 1

    flush_buf(char_pos)

    if not paragraphs:
        paragraphs = [{"content": text, "heading_path": None, "start": 0, "end": len(text)}]

    return paragraphs
```

마크다운 단락 분할을 바탕으로, 시스템은 토큰 개수에 기준하여 지능형 청킹을 한 단계 더 진행합니다. 입력 파일이 이미 구조화된 마크다운 텍스트이므로 시스템은 청크 경계를 더 정밀하게 제어할 수 있어, 각 청크가 벡터화 처리에 적합하면서도 마크다운 구조의 무결성을 유지할 수 있도록 보장합니다:

```python
def _chunk_paragraphs(paragraphs: List[Dict], chunk_tokens: int, overlap_tokens: int) -> List[Dict]:
    """토큰 수 기반의 지능형 청킹"""
    chunks: List[Dict] = []
    cur: List[Dict] = []
    cur_tokens = 0
    i = 0

    while i < len(paragraphs):
        p = paragraphs[i]
        p_tokens = _approx_token_len(p["content"]) or 1

        if cur_tokens + p_tokens <= chunk_tokens or not cur:
            cur.append(p)
            cur_tokens += p_tokens
            i += 1
        else:
            # 현재 청크 생성
            content = "\n\n".join(x["content"] for x in cur)
            start = cur[0]["start"]
            end = cur[-1]["end"]
            heading_path = next((x["heading_path"] for x in reversed(cur) if x.get("heading_path")), None)

            chunks.append({
                "content": content,
                "start": start,
                "end": end,
                "heading_path": heading_path,
            })

            # 오버랩 구간 구축
            if overlap_tokens > 0 and cur:
                kept: List[Dict] = []
                kept_tokens = 0
                for x in reversed(cur):
                    t = _approx_token_len(x["content"]) or 1
                    if kept_tokens + t > overlap_tokens:
                        break
                    kept.append(x)
                    kept_tokens += t
                cur = list(reversed(kept))
                cur_tokens = kept_tokens
            else:
                cur = []
                cur_tokens = 0

    # 마지막 청크 처리
    if cur:
        content = "\n\n".join(x["content"] for x in cur)
        start = cur[0]["start"]
        end = cur[-1]["end"]
        heading_path = next((x["heading_path"] for x in reversed(cur) if x.get("heading_path")), None)

        chunks.append({
            "content": content,
            "start": start,
            "end": end,
            "heading_path": heading_path,
        })

    return chunks
```
```python
def _approx_token_len(text: str) -> int:
    """대략적인 토큰 길이 추정, 한영 혼용 텍스트 지원"""
    # CJK 문자는 1문자당 1토큰으로 계산
    cjk = sum(1 for ch in text if _is_cjk(ch))
    # 그 외 문자는 공백 기준으로 분할하여 계산
    non_cjk_tokens = len([t for t in text.split() if t])
    return cjk + non_cjk_tokens

def _is_cjk(ch: str) -> bool:
    """CJK 문자 여부 판별"""
    code = ord(ch)
    return (
        0x4E00 <= code <= 0x9FFF or  # CJK 통합 한자
        0x3400 <= code <= 0x4DBF or  # CJK 통합 한자 확장 A
        0x20000 <= code <= 0x2A6DF or # CJK 통합 한자 확장 B
        0x2A700 <= code <= 0x2B73F or # CJK 통합 한자 확장 C
        0x2B740 <= code <= 0x2B81F or # CJK 통합 한자 확장 D
        0x2B820 <= code <= 0x2CEAF or # CJK 통합 한자 확장 E
        0xF900 <= code <= 0xFAFF      # CJK 호환 한자
    )
```

(3) 통일된 임베딩 및 벡터 저장소

임베딩 모델은 RAG 시스템의 핵심입니다. 텍스트를 고차원 벡터로 변환하여 컴퓨터가 텍스트의 의미론적 유사성을 이해하고 비교할 수 있게 합니다. RAG 시스템의 검색 성능은 임베딩 모델의 품질과 벡터 저장소의 효율성에 크게 의존합니다. HelloAgents는 통일된 임베딩 인터페이스를 제공합니다. 시연을 위해 여기서는 Bailian API를 사용합니다. 설정되지 않은 경우 로컬의 `all-MiniLM-L6-v2` 모델로 전환됩니다. 두 솔루션이 모두 지원되지 않는 경우 백업용으로 TF-IDF 알고리즘이 구성되어 있습니다. 실제 사용 시 원하는 모델이나 API로 교체하거나 프레임워크를 확장해 보실 수 있습니다.

```python
def index_chunks(
    store = None,
    chunks: List[Dict] = None,
    cache_db: Optional[str] = None,
    batch_size: int = 64,
    rag_namespace: str = "default"
) -> None:
    """
    통일된 임베딩 및 Qdrant 저장소를 사용하여 마크다운 청크 색인 생성.
    로컬 sentence-transformers로의 폴백이 제공되는 Bailian API 사용.
    """
    if not chunks:
        print("[RAG] 색인할 청크가 없습니다.")
        return

    # 통일된 임베딩 모델 사용
    embedder = get_text_embedder()
    dimension = get_dimension(384)

    # 기본 Qdrant 저장소 생성
    if store is None:
        store = _create_default_vector_store(dimension)
        print(f"[RAG] 차원 {dimension}의 기본 Qdrant 저장소를 생성했습니다.")

    # 임베딩 품질 향상을 위해 마크다운 텍스트 전처리
    processed_texts = []
    for c in chunks:
        raw_content = c["content"]
        processed_content = _preprocess_markdown_for_embedding(raw_content)
        processed_texts.append(processed_content)

    print(f"[RAG] 임베딩 시작: total_texts={len(processed_texts)} batch_size={batch_size}")

    # 배치 인코딩
    vecs: List[List[float]] = []
    for i in range(0, len(processed_texts), batch_size):
        part = processed_texts[i:i+batch_size]
        try:
            # 통일된 임베더 사용 (내부적으로 캐싱 처리)
            part_vecs = embedder.encode(part)

            # List[List[float]] 형식으로 표준화
            if not isinstance(part_vecs, list):
                if hasattr(part_vecs, "tolist"):
                    part_vecs = [part_vecs.tolist()]
                else:
                    part_vecs = [list(part_vecs)]

            # 벡터 포맷 및 차원 처리
            for v in part_vecs:
                try:
                    if hasattr(v, "tolist"):
                        v = v.tolist()
                    v_norm = [float(x) for x in v]

                    # 차원 확인 및 조정
                    if len(v_norm) != dimension:
                        print(f"[WARNING] 벡터 차원 이상: expected {dimension}, actual {len(v_norm)}")
                        if len(v_norm) < dimension:
                            v_norm.extend([0.0] * (dimension - len(v_norm)))
                        else:
                            v_norm = v_norm[:dimension]

                    vecs.append(v_norm)
                except Exception as e:
                    print(f"[WARNING] 벡터 변환 실패: {e}, 제로 벡터 사용")
                    vecs.append([0.0] * dimension)

        except Exception as e:
            print(f"[WARNING] 배치 {i} 인코딩 실패: {e}")
            # 재시도 메커니즘 구현
            # ... 재시도 로직 ...

        print(f"[RAG] 임베딩 진행: {min(i+batch_size, len(processed_texts))}/{len(processed_texts)}")
```

### 8.3.5 고급 검색 전략

RAG 시스템의 검색 기능은 그 핵심 경쟁력입니다. 실제 애플리케이션에서는 사용자의 질문과 문서의 실제 텍스트 표현 사이에 단어 선택의 격차가 존재하여 관련 문서가 누락될 수 있습니다. 이 문제를 해결하기 위해 HelloAgents는 세 가지 상호 보완적인 고급 검색 전략인 다중 쿼리 확장(MQE, Multi-Query Expansion), 가상 문서 임베딩(HyDE, Hypothetical Document Embeddings), 그리고 통일된 확장 검색 프레임워크를 구현했습니다.

(1) 다중 쿼리 확장 (MQE)

다중 쿼리 확장(MQE)은 사용자의 질문과 의미적으로 유사한 다양한 검색어를 생성하여 검색 리콜(recall)을 개선하는 기술입니다. 핵심 통찰은 동일한 질문을 여러 가지 방식으로 표현할 수 있으며, 표현 방식에 따라 매칭되는 관련 문서가 달라질 수 있다는 점입니다. 예를 들어 "파이썬 배우는 법"은 "파이썬 초보자 튜토리얼", "파이썬 학습 방법", "파이썬 프로그래밍 가이드" 등으로 확장될 수 있습니다. 이러한 확장 질문들을 병렬로 실행하여 검색 결과를 통합하면, 어휘 선택의 격차로 인해 유실될 수 있는 문서를 방지하고 폭넓은 지식 매칭을 확보할 수 있습니다.

MQE의 장점은 사용자 질문이 가질 수 있는 여러 잠재적 의미를 자동으로 인식한다는 점이며, 특히 모호한 질문이나 전문 용어가 포함된 질문에 효과적입니다. 시스템은 LLM을 사용해 확장 질문을 생성하므로 확장의 다양성과 의미론적 연관성이 보장됩니다:

```python
def _prompt_mqe(query: str, n: int) -> List[str]:
    """LLM을 사용하여 다양한 쿼리 확장 생성"""
    try:
        from ...core.llm import HelloAgentsLLM
        llm = HelloAgentsLLM()
        prompt = [
            {"role": "system", "content": "당신은 검색어 확장 도우미입니다. 의미론적으로 동일하거나 보완적인 다양한 질문을 생성하십시오. 한국어를 사용하며 짧게 작성하고 문장 부호는 생략하십시오."},
            {"role": "user", "content": f"원본 쿼리: {query}\n상호 독립적인 다양한 확장 질문을 {n}개 제공하십시오. 한 라인에 하나씩 작성하십시오."}
        ]
        text = llm.invoke(prompt)
        lines = [ln.strip("- \t") for ln in (text or "").splitlines()]
        outs = [ln for ln in lines if ln]
        return outs[:n] or [query]
    except Exception:
        return [query]
```

(2) 가상 문서 임베딩 (HyDE)

가상 문서 임베딩(HyDE)은 "답변으로 답변을 찾는다"라는 핵심 아이디어를 가진 혁신적인 검색 기술입니다. 전통적인 검색 방법은 질문으로 문서를 매칭하지만, 질문과 답변은 의미 공간 상에서 표현 양식이 서로 다릅니다. 질문은 대개 의문문인 반면 문서는 서술문이기 때문입니다. HyDE는 LLM이 먼저 가상의 답변 단락을 생성하도록 한 다음, 이 가상 답변의 벡터를 사용해 실제 문서를 검색함으로써 질문과 문서 간의 의미 공간 격차를 좁힙니다.

이 방법의 장점은 가상 답변이 의미 공간에서 실제 답변 문서와 더 유사하다는 점입니다. 가상 답변의 내용이 완전하지 않거나 일부 오류가 있더라도, 그에 포함된 핵심 어휘, 개념, 표현 양식은 검색 시스템이 올바른 문서를 찾도록 효과적으로 유도합니다. 특히 전문 영역의 쿼리에 대해 HyDE는 전문 용어를 포함한 가상 문서를 생성하므로 검색 정확성을 크게 향상시킵니다:

```python
def _prompt_hyde(query: str) -> Optional[str]:
    """검색 정확성 개선을 위해 가상 문서 생성"""
    try:
        from ...core.llm import HelloAgentsLLM
        llm = HelloAgentsLLM()
        prompt = [
            {"role": "system", "content": "사용자의 질문을 바탕으로, 벡터 검색을 위한 가상의 답변 단락을 작성하십시오 (분석 과정 제외)."},
            {"role": "user", "content": f"질문: {query}\n핵심 전문 용어를 포함하고 객관적이며 중간 길이의 단락을 직접 작성하십시오."}
        ]
        return llm.invoke(prompt)
    except Exception:
        return None
```

(3) 확장 검색 프레임워크

HelloAgents는 MQE와 HyDE 두 가지 전략을 통일된 확장 검색 프레임워크로 통합했습니다. 시스템은 사용자가 `enable_mqe`와 `enable_hyde` 매개변수를 통해 특정 시나리오에 맞게 활성화할 전략을 선택할 수 있도록 설계되었습니다. 높은 리콜이 필요한 시나리오에서는 두 전략을 동시에 켜고, 성능이 중요한 시나리오에서는 기본 검색만 사용하도록 제어할 수 있습니다.

확장 검색의 핵심 메커니즘은 "확장-검색-병합"의 3단계 워크플로우입니다. 먼저 시스템은 원본 쿼리를 기반으로 여러 확장 검색어(MQE 질문과 HyDE 가상 문서 포함)를 생성합니다. 그 후 각 확장 질문에 대해 벡터 검색을 병렬로 실행하여 후보 문서 풀(pool)을 확보합니다. 마지막으로 중복을 제거하고 유사도 점수를 기준으로 정렬하여 가장 관련성 높은 상위 k개의 문서를 반환합니다. 이 설계의 뛰어난 점은 `candidate_pool_multiplier` 매개변수(기본값 4)를 통해 충분한 수의 후보 문서를 사전에 확보해 필터링하는 동시에, 지능적인 중복 제거를 통해 중복된 콘텐츠가 반환되지 않도록 방지하는 것입니다.

```python
def search_vectors_expanded(
    store = None,
    query: str = "",
    top_k: int = 8,
    rag_namespace: Optional[str] = None,
    only_rag_data: bool = True,
    score_threshold: Optional[float] = None,
    enable_mqe: bool = False,
    mqe_expansions: int = 2,
    enable_hyde: bool = False,
    candidate_pool_multiplier: int = 4,
) -> List[Dict]:
    """
    통일된 임베딩 및 Qdrant를 사용하여 쿼리 확장 검색 수행.
    """
    if not query:
        return []

    # 기본 저장소 생성
    if store is None:
        store = _create_default_vector_store()
```
    # 쿼리 확장
    expansions: List[str] = [query]

    if enable_mqe and mqe_expansions > 0:
        expansions.extend(_prompt_mqe(query, mqe_expansions))
    if enable_hyde:
        hyde_text = _prompt_hyde(query)
        if hyde_text:
            expansions.append(hyde_text)

    # 중복 제거 및 정리
    uniq: List[str] = []
    for e in expansions:
        if e and e not in uniq:
            uniq.append(e)
    expansions = uniq[: max(1, len(uniq))]

    # 후보군 풀(pool) 크기 할당
    pool = max(top_k * candidate_pool_multiplier, 20)
    per = max(1, pool // max(1, len(expansions)))

    # RAG 데이터 필터 구성
    where = {"memory_type": "rag_chunk"}
    if only_rag_data:
        where["is_rag_data"] = True
        where["data_source"] = "rag_pipeline"
    if rag_namespace:
        where["rag_namespace"] = rag_namespace

    # 확장된 모든 쿼리로부터 결과 수집
    agg: Dict[str, Dict] = {}
    for q in expansions:
        qv = embed_query(q)
        hits = store.search_similar(
            query_vector=qv,
            limit=per,
            score_threshold=score_threshold,
            where=where
        )
        for h in hits:
            mid = h.get("metadata", {}).get("memory_id", h.get("id"))
            s = float(h.get("score", 0.0))
            if mid not in agg or s > float(agg[mid].get("score", 0.0)):
                agg[mid] = h

    # 점수 기준으로 정렬 및 반환
    merged = list(agg.values())
    merged.sort(key=lambda x: float(x.get("score", 0.0)), reverse=True)
    return merged[:top_k]
```

실제 애플리케이션에서는 이 세 가지 전략을 결합하여 사용하는 것이 가장 효과적입니다. MQE는 표현 다양성 문제를 해결하고, HyDE는 질문과 문서 사이의 의미 공간 격차를 좁혀주며, 통합 프레임워크는 결과의 품질과 다양성을 보장합니다. 일반적인 질문에는 MQE를 켜는 것을 권장하고, 전문 영역의 쿼리에는 MQE와 HyDE를 동시에 켜는 것이 좋습니다. 성능에 민감한 시나리오에서는 기본 검색만 사용하거나 MQE만 단독으로 사용할 수 있습니다.

물론 그 밖에도 다양하고 흥미로운 기법들이 많습니다. 여기에 소개된 내용은 개념 확장을 돕기 위한 적절한 기본 안내이며, 실제 사용 사례에 적합한 솔루션을 찾기 위해서는 직접 탐색하고 적용해보는 자세가 필요합니다.


## 8.4 지능형 문서 Q&A 어시스턴트 구축

이전 섹션들에서 우리는 HelloAgents의 메모리 시스템과 RAG 시스템의 설계 및 구현을 자세히 살펴보았습니다. 이제, 이 두 시스템을 유기적으로 결합하여 어떻게 지능형 문서 Q&A 어시스턴트를 구축할 수 있는지 완전한 실무 사례를 통해 보여드리겠습니다.

### 8.4.1 프로젝트 배경 및 목표

실무 환경에서는 방대한 분량의 기술 문서, 학술 논문, 제품 매뉴얼 등의 PDF 파일을 처리해야 하는 경우가 빈번합니다. 전통적인 문서 독해 방식은 효율이 낮아 핵심 정보를 빠르게 파악하기 어렵고 지식 간의 유기적 연결 고리를 설정하기도 쉽지 않습니다.

이 사례에서는 Datawhale의 또 다른 대형 모델 튜토리얼인 'Happy-LLM'의 공개 베타 PDF 문서인 `Happy-LLM-0727.pdf`를 학습 데이터로 사용할 것입니다. 우리는 **Gradio 기반의 웹 애플리케이션**을 구축하여 RAGTool과 MemoryTool을 통합한 대화형 학습 조수를 구현하는 과정을 보여주겠습니다. 해당 PDF 파일은 이 [링크](https://github.com/datawhalechina/happy-llm/releases/download/v1.0.1/Happy-LLM-0727.pdf)에서 다운로드할 수 있습니다.

우리는 다음과 같은 기능을 구현하고자 합니다:

1. **지능형 문서 처리**: MarkItDown을 사용하여 PDF를 마크다운으로 통일되게 변환하고, 마크다운 구조 기반의 지능형 청킹 전략을 적용하며, 효율적인 벡터화 및 색인을 구축합니다.

2. **고급 검색 Q&A**: 다중 쿼리 확장(MQE)을 통한 검색 리콜 향상, 가상 문서 임베딩(HyDE)을 통한 검색 정확성 개선, 맥락을 인지한 지능형 Q&A를 수행합니다.

3. **다단계 메모리 관리**: 현재 학습 작업과 대화 맥락을 관리하는 작업 메모리, 학습 히스토리와 쿼리 로그를 기록하는 일화적 메모리, 개념적 지식과 이해를 저장하는 의미론적 메모리, 그리고 문서의 특성과 다중 모달 정보를 처리하는 지각 메모리를 통합 관리합니다.

4. **개인화된 학습 지원**: 학습 기록 기반의 개인화 추천, 메모리 공고화 및 선택적 망각, 그리고 학습 보고서 생성과 진도 추적 기능을 지원합니다.

전체 시스템의 실행 구조를 명확히 보여주기 위해 그림 8.6에 5단계의 연동 방식과 데이터 흐름을 도식화했습니다. 5개의 단계는 하나의 폐쇄 루프를 이룹니다: 1단계는 처리된 PDF 문서 정보를 메모리 시스템에 기록하고, 2단계의 검색 결과 역시 메모리 시스템에 기록됩니다. 3단계는 메모리 시스템의 전체 기능(추가, 검색, 공고화, 망각)을 작동시키며, 4단계는 RAG와 Memory를 결합하여 지능형 라우팅을 제공하고, 5단계는 축적된 통계 정보를 수집해 최종 학습 보고서를 생성합니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/8-figures/8-6.png" alt="지능형 Q&A 어시스턴트 5단계 실행 흐름" width="85%"/>
  <p>그림 8.6 지능형 Q&A 어시스턴트의 5단계 실행 흐름</p>
</div>

이제 이 웹 애플리케이션의 구현을 보여드리겠습니다. 전체 애플리케이션은 다음과 같이 3개의 핵심 요소로 구분됩니다:

1. **핵심 어시스턴트 클래스 (PDFLearningAssistant)**: RAGTool과 MemoryTool의 호출 로직을 캡슐화합니다.
2. **Gradio 웹 인터페이스**: 학습을 위한 직관적이고 친숙한 대화형 인터페이스를 제공합니다 (예제 코드를 참조하여 직접 살펴볼 수 있습니다).
3. **기타 핵심 기능**: 노트 작성, 학습 복기, 통계 확인, 그리고 보고서 생성 기능입니다.

### 8.4.2 핵심 어시스턴트 클래스 구현

먼저, RAGTool과 MemoryTool의 연동 로직을 하나로 묶어주는 핵심 어시스턴트 클래스 `PDFLearningAssistant`를 구현합니다.

(1) 클래스 초기화

```python
class PDFLearningAssistant:
    """지능형 문서 Q&A 어시스턴트"""

    def __init__(self, user_id: str = "default_user"):
        """학습 어시스턴트 초기화

        Args:
            user_id: 사용자 ID, 다중 사용자 간 데이터 격리에 사용
        """
        self.user_id = user_id
        self.session_id = f"session_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

        # 도구 초기화
        self.memory_tool = MemoryTool(user_id=user_id)
        self.rag_tool = RAGTool(rag_namespace=f"pdf_{user_id}")

        # 학습 통계
        self.stats = {
            "session_start": datetime.now(),
            "documents_loaded": 0,
            "questions_asked": 0,
            "concepts_learned": 0
        }

        # 현재 로드된 문서
        self.current_document = None
```

이 초기화 프로세스에서 내린 핵심적인 설계 결정들은 다음과 같습니다:

- **MemoryTool 초기화**: `user_id` 매개변수를 통해 사용자 수준의 메모리 격리를 지원합니다. 서로 다른 사용자의 학습 메모리는 독립적으로 운영되며, 각각 자신만의 작업 메모리, 일화적 메모리, 의미론적 메모리, 지각 메모리 공간을 가집니다.
- **RAGTool 초기화**: `rag_namespace` 매개변수를 통해 지식 베이스 네임스페이스를 분리합니다. `f"pdf_{user_id}"` 포맷을 사용함으로써 각 사용자별로 독립적인 PDF 지식 저장소가 확보됩니다.
- **세션 관리**: `session_id`는 개별 학습 세션의 전체 실행 프로세스를 추적하여 향후 학습 여정을 복기하고 분석하는 도구로 활용됩니다.
- **통계 정보**: `stats` 딕셔너리는 학습 보고서를 생성할 때 핵심 지표들을 산출하는 데 활용됩니다.

(2) PDF 문서 로드

```python
def load_document(self, pdf_path: str) -> Dict[str, Any]:
    """PDF 문서를 지식 베이스에 로드

    Args:
        pdf_path: PDF 파일 경로

    Returns:
        Dict: 성공 여부와 메시지를 담은 딕셔너리
    """
    if not os.path.exists(pdf_path):
        return {"success": False, "message": f"파일이 존재하지 않습니다: {pdf_path}"}

    start_time = time.time()

    # [RAGTool] PDF 처리: MarkItDown 변환 → 지능형 청킹 → 벡터화 수행
    result = self.rag_tool.execute(
        "add_document",
        file_path=pdf_path,
        chunk_size=1000,
        chunk_overlap=200
    )

    process_time = time.time() - start_time

    if result.get("success", False):
        self.current_document = os.path.basename(pdf_path)
        self.stats["documents_loaded"] += 1

        # [MemoryTool] 학습 메모리에 이벤트 기록
        self.memory_tool.execute(
            "add",
            content=f"문서 《{self.current_document}》 로드 완료",
            memory_type="episodic",
            importance=0.9,
            event_type="document_loaded",
            session_id=self.session_id
        )

        return {
            "success": True,
            "message": f"성공적으로 로드했습니다! (소요 시간: {process_time:.1f}초)",
            "document": self.current_document
        }
    else:
        return {
            "success": False,
            "message": f"로드 실패: {result.get('error', '알 수 없는 오류')}"
        }
```

우리는 단 한 줄의 코드로 PDF 문서를 처리할 수 있습니다:

```python
result = self.rag_tool.execute(
    "add_document",
    file_path=pdf_path,
    chunk_size=1000,
    chunk_overlap=200
)
```

이 호출은 RAGTool의 완전한 처리 워크플로우(MarkItDown 변환, 이미지 OCR 등 고급 처리, 지능형 청킹, 벡터화 저장)를 내부적으로 자동 실행합니다. 이 과정에 대해서는 앞서 8.3절에서 상세히 설명했으므로 여기서는 다음에만 주의하면 됩니다:

- **작업 유형**: `"add_document"` - 지식 베이스에 문서 추가
- **파일 경로**: `file_path` - PDF 파일 경로
- **청킹 설정**: `chunk_size=1000, chunk_overlap=200` - 텍스트 단락 크기 제어
- **반환값**: 처리 상태와 통계 정보를 담은 결과 딕셔너리

문서가 성공적으로 로드된 후에는 MemoryTool을 사용하여 일화적 메모리에 기록합니다:

```python
self.memory_tool.execute(
    "add",
    content=f"문서 《{self.current_document}》 로드 완료",
    memory_type="episodic",
    importance=0.9,
    event_type="document_loaded",
    session_id=self.session_id
)
```

**왜 일화적 메모리를 사용할까요?** 이 동작은 특정 시점에 일어난 고유한 사건(event)이므로 일화적 메모리에 저장하기에 완벽한 대상입니다. `session_id` 매개변수는 해당 이벤트를 현재의 학습 세션과 연동시켜 향후 사용자의 학습 경로 추적을 유연하게 해줍니다.

이 학습 기록은 이후 개인화 서비스를 제공하기 위한 단초가 됩니다:

- 사용자가 "이전에 로드했던 문서가 뭐였지?"라고 물으면 → 일화적 메모리에서 검색
- 시스템이 사용자의 학습 여정 및 문서 사용 현황을 장기적으로 파악 가능

### 8.4.3 지능형 Q&A 함수

문서가 로드되면 사용자는 문서에 대한 질문을 시작할 수 있습니다. 사용자의 질문을 처리하기 위해 `ask` 메서드를 다음과 같이 구현합니다:

```python
def ask(self, question: str, use_advanced_search: bool = True) -> str:
    """문서에 대한 질문 처리

    Args:
        question: 사용자 질문
        use_advanced_search: 고급 검색(MQE + HyDE) 사용 여부

    Returns:
        str: 생성된 답변
    """
    if not self.current_document:
        return "⚠️ 먼저 문서를 로드해 주십시오!"

    # [MemoryTool] 질문을 작업 메모리에 기록
    self.memory_tool.execute(
        "add",
        content=f"질문: {question}",
        memory_type="working",
        importance=0.6,
        session_id=self.session_id
    )

    # [RAGTool] 고급 검색을 활용하여 답변 도출
    answer = self.rag_tool.execute(
        "ask",
        question=question,
        limit=5,
        enable_advanced_search=use_advanced_search,
        enable_mqe=use_advanced_search,
        enable_hyde=use_advanced_search
    )

    # [MemoryTool] 일화적 메모리에 Q&A 상호작용 정보 기록
    self.memory_tool.execute(
        "add",
        content=f"'{question}'에 대한 학습 수행",
        memory_type="episodic",
        importance=0.7,
        event_type="qa_interaction",
        session_id=self.session_id
    )

    self.stats["questions_asked"] += 1

    return answer
```

우리가 `self.rag_tool.execute("ask", ...)`를 호출하면, RAGTool 내부에서 다음과 같은 고급 검색 워크플로우가 실행됩니다:

1. **다중 쿼리 확장 (MQE)**:
   ```python
   # 다양한 질문 생성
   expanded_queries = self._generate_multi_queries(question)
   # 예: 사용자가 "대형 언어 모델이란?"을 입력했을 때 다음과 같이 생성:
   # - "대형 언어 모델의 정의는 무엇입니까?"
   # - "LLM의 개념을 설명해 주십시오."
   # - "Large Language Model의 뜻은 무엇입니까?"
   ```
   MQE는 LLM을 사용해 의미가 같으면서 다르게 표현된 검색어들을 구성하여, 사용자의 의도를 다각도로 포착하고 검색 리콜(recall) 비율을 30%~50%가량 증대시킵니다.

2. **가상 문서 임베딩 (HyDE)**:
   - 가상의 답변 단락을 우선 생성하여 질문과 대상 문서 사이의 형태 차이(의미적 격차)를 해소합니다.
   - 생성된 가상 답변 벡터를 매개체로 색인 공간을 검색합니다.

이러한 고급 검색 기술의 내부 구현 세부사항은 앞서 8.3.5절에서 상세히 설명되었습니다.

### 8.4.4 기타 핵심 기능

문서 로드와 Q&A 기능 외에도 노트 작성, 학습 여정 복기, 통계 확인, 그리고 보고서 생성 기능을 추가로 지원합니다:

```python
def add_note(self, content: str, concept: Optional[str] = None):
    """학습 노트 추가"""
    self.memory_tool.execute(
        "add",
        content=content,
        memory_type="semantic",
        importance=0.8,
        concept=concept or "general",
        session_id=self.session_id
    )
    self.stats["concepts_learned"] += 1

def recall(self, query: str, limit: int = 5) -> str:
    """학습 여정 복기"""
    result = self.memory_tool.execute(
        "search",
        query=query,
        limit=limit
    )
    return result

def get_stats(self) -> Dict[str, Any]:
    """학습 통계 정보 획득"""
    duration = (datetime.now() - self.stats["session_start"]).total_seconds()
    return {
        "학습 진행 시간": f"{duration:.0f}초",
        "로드된 문서 수": self.stats["documents_loaded"],
        "질문한 횟수": self.stats["questions_asked"],
        "작성된 학습 노트": self.stats["concepts_learned"],
        "현재 로드된 문서": self.current_document or "로드되지 않음"
    }

def generate_report(self, save_to_file: bool = True) -> Dict[str, Any]:
    """학습 보고서 생성"""
    memory_summary = self.memory_tool.execute("summary", limit=10)
    rag_stats = self.rag_tool.execute("stats")

    duration = (datetime.now() - self.stats["session_start"]).total_seconds()
    report = {
        "session_info": {
            "session_id": self.session_id,
            "user_id": self.user_id,
            "start_time": self.stats["session_start"].isoformat(),
            "duration_seconds": duration
        },
        "learning_metrics": {
            "documents_loaded": self.stats["documents_loaded"],
            "questions_asked": self.stats["questions_asked"],
            "concepts_learned": self.stats["concepts_learned"]
        },
        "memory_summary": memory_summary,
        "rag_status": rag_stats
    }

    if save_to_file:
        report_file = f"learning_report_{self.session_id}.json"
        with open(report_file, 'w', encoding='utf-8') as f:
            json.dump(report, f, ensure_ascii=False, indent=2, default=str)
        report["report_file"] = report_file

    return report
```

이 각 메서드들은 다음 기능을 충족합니다:

- **add_note**: 작성한 학습 노트를 의미론적 메모리에 보관합니다.
- **recall**: 학습 여정과 대화 내역을 메모리 시스템에서 되짚어 검색합니다.
- **get_stats**: 현재 대화 및 학습 세션 동안의 주요 활동 통계를 집계합니다.
- **generate_report**: 종합적인 학습 정보를 정리하여 JSON 파일로 보고서를 파일 출력합니다.

### 8.4.5 실행 화면 및 인터페이스 예시

다음은 실제 실행 인터페이스의 예시입니다. 그림 8.7과 같이 애플리케이션의 메인 페이지에 진입한 후 먼저 어시스턴트를 초기화해야 합니다. 이는 데이터베이스 접속, 임베딩 모델 로드, API 세션 연동 등 전반적인 준비 작업을 수행하는 과정입니다. 초기화가 끝나면 PDF 파일을 선택해 지식 베이스에 로드할 수 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/8-figures/8-7.png" alt="Q&A 어시스턴트 메인 화면" width="85%"/>
  <p>그림 8.7 Q&A 어시스턴트의 초기 실행 화면</p>
</div>

첫 번째 핵심 기능은 지능형 문서 Q&A입니다. 로드된 문서를 기반으로 질문에 대한 답변을 찾고, 관련 근거 구절의 출처와 유사성 평가 점수를 함께 제시합니다. 이는 그림 8.8에 나와 있는 RAG 도구의 시각적 실행 사례입니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/8-figures/8-8.png" alt="Q&A 검색 결과 제시 화면" width="85%"/>
  <p>그림 8.8 문서 검색 및 Q&A 실행 결과 화면</p>
</div>

두 번째 기능은 학습 노트 생성입니다. 그림 8.9에 나타난 바와 같이 사용자는 관련 개념(concept) 항목을 선택하고 노트 본문을 기입할 수 있습니다. 기입된 내용은 Memory 도구를 거쳐 영구 저장 공간에 보관되며, 학습 종료 시 전체 결과 보고서에 요약 반영됩니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/8-figures/8-9.png" alt="학습 노트 등록 화면" width="85%"/>
  <p>그림 8.9 메모리 시스템을 활용한 학습 노트 보관 기능</p>
</div>

끝으로, 학습 여정 동안의 활동 지표를 한눈에 볼 수 있는 통계 대시보드와 보고서 출력 화면입니다. 그림 8.10에서 볼 수 있듯이 로드한 문서, 생성한 질문, 기록된 노트의 빈도가 시각적으로 집계되며, 우측에서 최종적으로 구성된 종합 JSON 보고서 파일 데이터를 확인할 수 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/8-figures/8-10.png" alt="통계 및 종합 학습 보고서 화면" width="85%"/>
  <p>그림 8.10 세션 통계 분석 및 최종 보고서 데이터 확인 화면</p>
</div>

이 Q&A 어시스턴트 사례를 통하여, 우리는 RAGTool과 MemoryTool을 유기적으로 활용해 실용적인 **웹 기반 지능형 문서 Q&A 서비스**를 조립해 보았습니다. 전체 어플리케이션의 완성본 코드는 `code/chapter8/11_Q&A_Assistant.py`에서 확인할 수 있습니다. 실행 후 웹 브라우저로 `http://localhost:7860`에 접근하면 직접 상호작용 서비스를 체험할 수 있습니다.

독자 여러분은 반드시 이 사례를 직접 실행하여 RAG와 메모리의 실질적인 장점을 체험하고, 프로젝트 성격에 맞춰 사용자 정의 설정을 유연하게 덧붙여 보시기를 권장합니다!


## 8.5 요약 및 향후 전망

이번 장에서는 HelloAgents 프레임워크에 두 핵심 기둥인 메모리 시스템과 RAG 시스템을 성공적으로 확장했습니다.

지식의 깊이를 더하고 배운 내용을 유연히 응용하고자 하는 독자분들께 다음 탐구 방향을 제안합니다:

1. 처음부터 단독으로 미니 메모리 모듈을 스크래치 설계해보고 기능적 점진적 고도화를 반복해 보십시오.
2. 다양한 성격의 임베딩 모델군과 검색 전략들을 한데 묶어 정량 테스트를 수행하고, 실 프로젝트 도메인에 알맞은 최적 조합을 발굴해 보십시오.
3. 구축한 RAG 및 메모리 연동 구조를 개인의 고유 어플리케이션 환경에 실제 탑재해 가며 실무 피드백을 통해 보완점을 찾아 보십시오.

**고급 기능으로의 확장 아이디어**

1. 지능형 메모리 및 정교한 RAG 구현으로 업계의 사랑을 받는 최첨단 오픈소스 저장소들의 아키텍처 패턴을 깊이 분석해 보십시오.
2. 텍스트 정보뿐 아니라 시각(이미지) 요소가 융합된 다중 모달 RAG 또는 이종 모달 데이터의 교차 검색 구조를 탐험해 보십시오.
3. HelloAgents 오픈소스 생태계에 기여자로 참여하여 참신한 기능 제안이나 코드 풀 리퀘스트를 올려 보십시오.

본 내용을 완독하면서 여러분은 메모리와 RAG의 기계적 명세를 파악하는 것을 넘어, 인지 과학이 조명한 생물학적 메커니즘을 어떻게 실용적인 컴퓨터 소프트웨어 설계 패턴으로 환유해 내는지 그 생각의 고리를 연결했을 것입니다. 이러한 융합적 관점은 다가오는 차세대 지능형 에이전트 시스템을 구상하는 데 더없이 단단한 디딤돌이 되어 줄 것입니다.

끝으로 마인드맵 형태로 일목요연하게 8장의 전체 지식 얼개를 정리한 그림 8.11을 확인해 보십시오:

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/8-figures/8-11.png" alt="HelloAgents 8장 지식 요약 마인드맵" width="85%"/>
  <p>그림 8.11 HelloAgents 8장 핵심 지식 체계 요약</p>
</div>

이번 단락을 통해 우리는 고립된 응답기 수준의 에이전트를 지나, 과거를 되짚고(Memory) 외부의 풍부한 자료(RAG)를 펼쳐 읽을 수 있는 진짜 "학습 조수"를 품게 되었습니다. 이 유연한 모듈 아키텍처는 기술 지원, 가상 비서, 스마트 헬프데스크 등 다양한 영역의 에이전트로 용이하게 탈바꿈할 수 있습니다.

다음 장에서는 대화의 품질을 극대화하고 맥락의 풍성함을 입혀줄 '컨텍스트 엔지니어링(Context Engineering)'에 대해 알아보겠습니다. 계속해서 학습을 이어가 봅시다!


## 연습문제

> **참고**: 일부 서술형 문항은 고정된 정답이 존재하지 않으며, 메모리 시스템과 RAG 기술에 대한 전체적인 인지적 깊이와 실무 설계 감각을 길러주는 것을 목적으로 합니다.

1. 이번 장에서는 작업 메모리, 일화적 메모리, 의미론적 메모리, 지각 메모리의 네 가지 메모리 유형을 제시했습니다. 다음 질문을 분석해 보십시오:
   - 8.2.5절에서 설계된 각 메모리 유형의 스코어링 공식들을 비교 분석해 보십시오. 특히 일화적 메모리는 왜 시간적 최신성(가중치 0.2)을 중시하고, 의미론적 메모리는 왜 그래프 검색의 관계성(가중치 0.3)에 더 큰 보완점을 부여하는지 그 인지적/기술적 타당성을 논해 보십시오.
   - 만약 여러분이 식단, 운동, 수면 지표를 체계적으로 기록하고 건강 관리 조언을 해주는 '개인 맞춤 헬스 어시스턴트'를 설계한다면, 네 가지 메모리 유형을 어떻게 각각 활용하여 일련의 서비스 시나리오로 통합하시겠습니까? 구체적으로 적어 보십시오.
   - 작업 메모리는 시간이 흐르면 만료되는 TTL(Time To Live) 메커니즘을 씁니다. 그렇다면 작업 메모리의 내용 중 어느 시점에 어떤 규칙으로 중요 정보를 '장기 메모리로 공고화'시켜야 할까요? 이를 지능적으로 감지할 수 있는 시스템 조건식을 직접 논리 구조로 제안해 보십시오.

2. 8.3절 RAG 시스템에서 MarkItDown을 통해 문서를 마크다운으로 통일되게 다듬는 설계적 장점에 기반해 아래 심화 연습문제를 해결해 보십시오:
   - 현재 구현된 지능형 청킹 알고리즘은 마크다운의 제목 구조(#, ##, ###)를 명확히 인지합니다. 만약 기승전결이 모호한 소설책이나 딱딱한 줄글 형태의 약관처럼 마크다운 헤더가 존재하지 않는 파일을 처리하려면, 단락 분할의 무결성을 지키기 위해 청킹 알고리즘을 어떻게 보완해야 할까요? '의미론적 단락 경계'를 식별할 수 있는 개선안을 아이디어 단계나 슈도 코드로 구상해 보십시오.
   - 8.3.5절에 제시된 고급 검색 기법인 MQE와 HyDE의 작동 장점을 되짚어 보십시오. 가령 '의료 질의 분석'이나 'IT 기술 매뉴얼 질문 대응' 같은 구체적 시나리오를 설정하고, 기본 키워드/벡터 검색, MQE, HyDE가 내놓는 질의 결과의 특성을 비교하여 각각 어떤 단점을 상호 보완해 주는지 논하십시오.
   - 임베딩 모델의 선택은 검색의 정밀도에 결정적인 영향을 줍니다. 이번 장에 등장한 세 솔루션(Bailian 클라우드 API, 로컬 트랜스포머, 가벼운 TF-IDF)을 정확성, 처리 레이턴시, 운영 비용, 오프라인 환경 작동성 등의 기준으로 상세 표로 대조해 보십시오.

3. 메모리의 선택적 '망각'은 유연한 시스템 구축을 돕는 훌륭한 생물학적 영감입니다. 8.2.3절의 MemoryTool을 바탕으로 다음 고도화 실습에 응해 보십시오:
   - 현재 구현된 세 망각 메커니즘(중요도 기반, 시간 흐름 기반, 메모리 수 한계 극복을 위한 용량 기반)을 뛰어넘어, 사용 빈도, 의미적 연결성, 중요도의 동적 추이를 종합 계산하여 자동으로 망각 순위를 도출하는 '복합 지능형 망각 점수 알고리즘'의 공식을 수립해 보십시오.
   - 오랜 시간 에이전트를 가동하면 데이터가 계속 축적됩니다. 최근에는 거의 찾지 않지만 삭제하기엔 아까운 메모리를 저렴한 아카이빙 스토리지(Cold Storage)로 이관하고, 차후 맥락상 필요할 때 다시 동적으로 복원해내는 '메모리 티어링(Archiving/Restoration)' 흐름을 기존의 네 가지 메모리 유형 아키텍처에 어떻게 녹여낼 수 있을지 구조를 제안해 보십시오.
   - 사용자가 민감한 개인 정보(예: 카드 번호 등)의 삭제를 요청했을 때 단순히 DB 테이블에서 행(row)을 지우는 것만으로 충분할까요? 벡터 데이터베이스 및 지식 그래프(Neo4j)와 연동된 환경에서 어떻게 정보가 확실히 지워졌음을(Right to be forgotten) 검증하고 지울 수 있을지 그 기술적 과제를 서술하십시오.

4. 8.4절 '지능형 학습 조수' 사례는 MemoryTool과 RAGTool을 엮어 풍부한 연동을 보였습니다. 다음 과제를 분석해 보십시오:
   - `ask()` 메서드는 문서 데이터(RAG)와 사용자의 과거 내역(Memory)을 모두 들여다봅니다. 사용자 질문에 따라 '이 질문은 지식 문서 검색(RAG)으로 보내야 할지, 아니면 대화의 흐름(Memory)에서 대답해야 할지' 판별하는 라우팅 전략(Intelligent Router)을 어떻게 설계할 수 있을까요?
   - 현재의 학습 결과 요약 기능(`generate_report()`)은 단순 통계 위주입니다. 이를 개선해 사용자의 질의 유형, 기록된 노트의 테마를 분석해 '사용자가 취약한 영역의 개념'을 포착하고 차후 학습 문서를 역으로 추천해주는 '지능형 피드백 생성기'를 덧붙이려면 기존 구조에서 어떤 요소를 추가해야 할지 설계안을 작성해 보십시오.
   - 개발한 조수를 여러 사용자가 함께 쓰는 멀티 테넌트(Multi-User) 서비스로 확장하려 합니다. Qdrant 벡터 DB와 Neo4j 그래프 DB 단에서 사용자별로 온전히 영역이 나뉘어 정보가 새어나가지 않도록 만드는 '멀티 테넌시 물리적/논리적 격리 설계 방안'을 제안해 보십시오.

5. 의미론적 메모리는 지식 그래프(Neo4j)의 관계 지식을 보완재로 삼습니다. 다음 설계안을 구상해 보십시오:
   - 비정형 텍스트에서 엔티티와 관계를 파싱해 Neo4j로 주입하는 자동 추출 방식의 오탐지(False Positive) 가능성을 짚어보십시오. 관계가 엉뚱하게 추출되는 사례를 막고 주입되는 지식의 질을 모니터링할 수 있는 '그래프 품질 평가 및 조정(Verification) 워크플로우'를 어떻게 프레임워크에 추가하면 좋을까요?
   - Neo4j의 다중 홉(Multi-hop) 경로 매칭 쿼리가 빛을 발할 수 있는 지식 관계 추론 상황의 가상 사례를 만들고, 일반적인 벡터 유사도 기반 키워드 검색으로는 도저히 매치할 수 없는 어떤 독특한 인지적 질의를 그래프 쿼리가 풀어낼 수 있는지 예시 쿼리와 함께 논해 보십시오.
   - "벡터 매칭 + 그래프 관계 검색"의 상호 보완 구조를 단일 벡터 데이터베이스 검색과 정량 비교해 보고, 시너지 효과가 가장 크게 발휘되는 구체적인 검색 질의 범주가 어떤 것인지 명확히 짚어 보십시오.

## 참고문헌

[1] Atkinson, R. C., & Shiffrin, R. M. (1968). Human memory: A proposed system and its control processes. In *Psychology of learning and motivation* (Vol. 2, pp. 89-195). Academic press.

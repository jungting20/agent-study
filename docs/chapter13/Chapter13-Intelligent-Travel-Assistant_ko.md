# 13장 지능형 여행 비서

이전 장들에서 우리는 다양한 에이전트 패러다임, 도구 시스템, 메모리 메커니즘, 프로토콜 통신, 성능 평가를 포함한 핵심 기능을 구현하며 HelloAgents 프레임워크를 처음부터 직접 구축했습니다. 이번 장부터는 완전히 새로운 단계로 접어듭니다. **지금까지 배운 모든 지식을 통합하여 완전한 실전 애플리케이션을 구축하는 것입니다.**

1장에서 우리가 처음으로 구축했던 에이전트를 기억하시나요? 그것은 `Thought-Action-Observation` 루프의 기본 원리를 보여주는 간단한 지능형 여행 비서였습니다.
이번 장에서 구현할 지능형 여행 비서는 다음과 같은 핵심 기능을 포함하는 완전한 프로젝트입니다.

**(1) 지능형 일정 계획**: 사용자가 목적지, 날짜, 선호도 등의 정보를 입력하면 시스템이 자동으로 관광지, 식당, 호텔을 포함한 완전한 일정 계획을 생성합니다.

**(2) 지도 시각화**: 관광지 위치를 지도에 표시하고 투어 경로를 그려주어 일정을 한눈에 명확하게 파악할 수 있도록 합니다.

**(3) 예산 계산**: 티켓, 호텔, 식사, 교통 비용을 자동으로 계산하여 예산 상세 정보를 표시합니다.

**(4) 일정 편집**: 관광지 추가, 삭제, 조정 기능을 지원하며 지도를 실시간으로 업데이트합니다.

**(5) 내보내기 기능**: PDF나 이미지로의 내보내기를 지원하여 저장 및 공유가 편리하도록 합니다.

## 13.1 프로젝트 개요 및 아키텍처 디자인

### 13.1.1 지능형 여행 비서가 필요한 이유

여행 계획을 세우는 것은 설레는 일이기도 하지만 동시에 피곤한 일이기도 합니다. 인터넷에서 관광지 정보를 검색하고, 다양한 가이드를 비교하고, 날씨 예보를 확인하고, 호텔을 예약하고, 예산을 계산하고, 경로를 계획해야 합니다. 이 과정은 몇 시간 또는 며칠이 걸릴 수도 있습니다. 그리고 이렇게 많은 시간을 들인 후에도 계획한 일정이 합리적인지, 중요한 관광지를 놓치지는 않았는지, 예산이 정확한지 확신하기 어렵습니다.

기존의 여행 계획 방식에는 몇 가지 문제점이 있습니다. 첫째는 **분산된 정보**입니다. 관광지 정보는 여행 웹사이트에, 날씨 정보는 날씨 웹사이트에, 호텔 정보는 예약 웹사이트에 흩어져 있어 사용자가 여러 웹사이트를 오가며 수동으로 이 정보를 통합해야 합니다. 둘째는 **개인화 부족**입니다. 대부분의 여행 가이드는 일반적인 정보만 제공하며 사용자의 개인적 선호도, 예산 제약, 여행 시간 및 기타 요소를 고려하지 않습니다. 마지막으로 **수정의 어려움**입니다. 일정을 변경하고 싶을 때 여행 전체를 다시 계획해야 할 수도 있습니다. 관광지의 순서, 시간 배치, 예산이 모두 서로 연결되어 있기 때문입니다.

AI 기술은 이러한 문제를 해결할 수 있는 새로운 가능성을 제공합니다. 사용자가 시스템에 "북경(베이징)을 3일 동안 여행하고 싶고, 역사와 문화를 좋아하며, 예산은 보통입니다"라고 말하기만 하면 시스템이 매일 방문할 관광지, 식사할 곳, 머무를 호텔, 필요한 예산 등을 포함한 완전한 일정 계획을 자동으로 생성해 준다고 상상해 보십시오. 또한, 이 계획은 조정이 가능하여 마음에 들지 않는 관광지를 삭제하거나 투어 순서를 조정하면 시스템이 지도와 예산을 자동으로 업데이트합니다.

이것이 바로 우리가 구축하고자 하는 지능형 여행 비서입니다. 이는 단순한 기술 데모가 아니라 실제로 유용한 애플리케이션입니다. 이 프로젝트를 통해 AI 기술을 실무 문제에 적용하는 방법, 다중 에이전트(Multi-Agent) 시스템을 디자인하는 방법, 그리고 완전한 웹 애플리케이션을 구축하는 방법을 배우게 될 것입니다.

### 13.1.2 기술 아키텍처 개요

시스템은 그림 13.1에 나타난 것처럼 4개의 레이어로 나뉘는 전형적인 **프론트엔드 및 백엔드 분리 아키텍처**를 채택하고 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/13-figures/13-1.png" alt="" width="85%"/>
  <p>그림 13.1 지능형 여행 비서 기술 아키텍처</p>
</div>

**(1) 프론트엔드 레이어 (Vue3+TypeScript)**: 양식 입력, 결과 표시, 지도 시각화를 포함하여 사용자 인터랙션과 데이터 표시를 담당합니다.

**(2) 백엔드 레이어 (FastAPI)**: API 라우팅, 데이터 검증, 비즈니스 로직을 담당합니다.

**(3) 에이전트 레이어 (HelloAgents)**: 작업 분할, 도구 호출, 결과 통합을 담당합니다. 4개의 특화된 에이전트(Agent)가 포함되어 있습니다.

**(4) 외부 서비스 레이어**: 고덕 지도(Amap) API, Unsplash API, LLM API를 포함하여 데이터와 기능을 제공합니다.

데이터 흐름 과정은 다음과 같습니다: 사용자가 프론트엔드에서 폼을 작성하면 → 백엔드에서 데이터를 검증하고 → 에이전트 시스템을 호출합니다 → 에이전트들은 순차적으로 관광지 검색, 날씨 조회, 호텔 추천, 일정 계획 에이전트를 호출합니다 → 각 에이전트는 MCP 프로토콜을 통해 외부 API를 호출합니다 → 결과를 통합하여 프론트엔드로 반환하고 → 프론트엔드가 이를 렌더링하여 표시합니다.

소스 코드 위치를 쉽게 찾을 수 있도록 돕는 프로젝트 참조 구조는 다음과 같습니다:

```
helloagents-trip-planner/
├── backend/                    # 백엔드 코드
│   ├── app/
│   │   ├── agents/            # 에이전트 구현
│   │   ├── api/               # API 라우트
│   │   ├── models/            # 데이터 모델
│   │   ├── services/          # 서비스 레이어
│   │   └── config.py          # 설정 파일
│   └── requirements.txt       # Python 의존성
│
└── frontend/                   # 프론트엔드 코드
    ├── src/
    │   ├── views/             # 페이지 컴포넌트
    │   ├── services/          # API 서비스
    │   ├── types/             # 타입 정의
    │   └── router/            # 라우트 설정
    └── package.json           # npm 의존성
```

자세한 아키텍처 디자인과 데이터 흐름은 이어지는 섹션에서 소개합니다.

### 13.1.3 빠른 체험: 5분 안에 프로젝트 실행하기

구현 세부 사항을 자세히 알아보기 전에, 먼저 프로젝트를 실행하여 최종 결과물을 확인해 보겠습니다. 이를 통해 전체 시스템을 직관적으로 이해할 수 있을 것입니다.

**환경 요구사항:**

- Python 3.10 이상
- Node.js 16.0 이상
- npm 8.0 이상

**API 키 발급:**

다음 API 키들을 준비해야 합니다:

- LLM API (OpenAI, DeepSeek 등)
- 고덕 지도(Amap) 웹 서비스 키: https://console.amap.com/ 에 방문하여 가입하고 애플리케이션 생성
- Unsplash Access Key: https://unsplash.com/developers 에 방문하여 가입하고 애플리케이션 생성

모든 API 키를 `.env` 파일에 저장합니다.

백엔드 시작:

```bash
# 1. 백엔드 디렉토리로 이동
cd helloagents-trip-planner/backend

# 2. 의존성 설치
pip install -r requirements.txt

# 3. 환경 변수 설정
cp .env.example .env
# .env 파일을 편집하여 API 키 입력

# 4. 백엔드 서비스 시작
uvicorn app.api.main:app --reload
# 또는
python run.py
```

성공적으로 시작되면 http://localhost:8000/docs 에 방문하여 API 문서를 확인할 수 있습니다.

새 터미널 창을 엽니다:

```bash
# 1. 프론트엔드 디렉토리로 이동
cd helloagents-trip-planner/frontend

# 2. 의존성 설치
npm install

# 3. 프론트엔드 서비스 시작
npm run dev
```

성공적으로 시작되면 http://localhost:5173 에 방문하여 애플리케이션을 사용할 수 있습니다.

핵심 기능 체험하기:

먼저 홈페이지 양식에 목적지 도시, 여행 날짜, 선호도, 예산, 교통 및 숙박 유형을 입력합니다. "계획 시작" 버튼을 클릭하면 시스템에 로딩 진행 표시줄이 나타나고 신속하게 결과 페이지를 생성합니다(그림 13.2 참조).

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/13-figures/13-2.png" alt="" width="85%"/>
  <p>그림 13.2 여행 비서 계획 진행 페이지</p>
</div>

로딩이 성공적으로 끝나면 페이지에는 그림 13.3과 13.4처럼 일정 개요, 예산 상세 정보, 관광지 지도, 일별 일정 상세 정보 및 날씨 정보가 명확하게 표시됩니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/13-figures/13-3.png" alt="" width="85%"/>
  <p>그림 13.3 여행 비서 계획 완료 페이지</p>
</div>

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/13-figures/13-4.png" alt="" width="85%"/>
  <p>그림 13.4 여행 비서 계획 완료 페이지</p>
</div>

사용자가 개인 맞춤형 조정을 원할 경우, 그림 13.5처럼 "일정 편집" 버튼을 클릭하여 관광지 순서를 자유롭게 조정하거나 특정 관광지를 삭제할 수 있습니다. 계획이 완료되면 "일정 내보내기" 드롭다운 메뉴를 통해 최종 계획을 이미지나 PDF 파일로 손쉽게 저장하여 언제든지 참조할 수 있습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/13-figures/13-5.png" alt="" width="85%"/>
  <p>그림 13.5 여행 비서 계획 완료 페이지</p>
</div>

## 13.2 데이터 모델 디자인

### 13.2.1 웹 애플리케이션에서의 데이터 흐름

지능형 여행 비서를 구축할 때 해결해야 할 핵심 문제는 다음과 같습니다. **여행 계획 데이터를 어떻게 표현하고 전달할 것인가?**

우리는 완전한 웹 애플리케이션에서 데이터가 어떻게 흐르는지 이해해야 합니다. 브라우저에서 사용자가 "계획 시작" 버튼을 누를 때 어떤 일이 일어나는지 상상해 보십시오.

사용자가 프론트엔드에서 작성한 양식 데이터(목적지, 날짜, 예산 등)는 HTTP 요청을 통해 백엔드 서버로 전송되어야 합니다. 백엔드가 데이터를 받으면 에이전트 시스템을 호출하여 처리합니다. 그 후 에이전트들은 데이터 획득을 위해 고덕 지도 API나 Unsplash API 같은 외부 서비스를 호출할 것입니다. 이러한 외부 API가 반환하는 데이터 포맷은 서로 다릅니다. 어떤 것은 `lng`를 쓰고, 어떤 것은 `lon`을 쓰며, 어떤 것은 `longitude`를 씁니다. 마지막으로 백엔드는 처리된 데이터를 프론트엔드로 반환해야 하고, 프론트엔드는 이를 사용자가 보는 페이지로 렌더링합니다.

이 과정에서 데이터는 여러 번 변환됩니다: 프론트엔드 폼 → HTTP 요청 → 백엔드 Python 객체 → 외부 API 응답 → 백엔드 Python 객체 → HTTP 응답 → 프론트엔드 TypeScript 객체 → 페이지 표시. 통일된 데이터 포맷이 없다면 각 변환 단계마다 오류가 발생할 수 있습니다. 이것이 바로 **데이터 모델**이 필요한 이유입니다.

### 13.2.2 딕셔너리에서 Pydantic 모델로

1장의 간단한 프로토타입부터 시작하겠습니다. 그 프로토타입에서 우리는 관광지 데이터를 나타내기 위해 Python 딕셔너리를 사용했습니다:

```python
# 1장 방식: 딕셔너리 사용
attraction = {
    "name": "자금성",
    "location": {"lng": 116.397128, "lat": 39.916527},
    "price": 60
}

# 데이터 접근
lng = attraction["location"]["lng"]
```

이 방식은 프로토타입 단계에서는 편리하지만, 실제 프로젝트에서는 많은 문제를 일으킵니다. 첫째는 **일관되지 않은 필드명** 문제입니다. 고덕 지도 API가 반환하는 위치 데이터는 `"116.397128,39.916527"`과 같은 문자열로, 경도와 위도로 수동으로 나누어야 합니다. Unsplash API는 `longitude`와 `latitude`를 사용할 수도 있습니다. 코드 곳곳에서 딕셔너리를 사용한다면 모든 위치에서 이러한 차이점을 직접 처리해야 합니다.

둘째는 **타입 안정성** 문제입니다. 실수로 `price`를 문자열 `"60"`으로 설정했다고 가정해 봅시다. Python에서는 즉시 에러가 발생하지 않지만, 총예산을 계산할 때 문제가 발생합니다. 더 나쁜 것은 이러한 종류의 에러는 런타임에만 발견할 수 있으며 에러 메시지가 발생한 위치를 찾기 어려울 수 있다는 점입니다.

마지막으로 **유지보수성** 문제입니다. 관광지에 새로운 필드(예: `rating`)를 추가해야 할 때, 코드의 여러 곳을 수정해야 합니다. 한 곳이라도 빠뜨리면 데이터 불일치가 발생합니다.

Pydantic이 솔루션을 제공합니다. Pydantic은 클래스를 사용해 데이터 구조를 정의하고 유효성 검사, 변환, 직렬화를 자동으로 처리할 수 있게 해주는 Python 데이터 검증 라이브러리입니다. 간단한 예를 살펴보겠습니다:

```python
from pydantic import BaseModel, Field

class Location(BaseModel):
    longitude: float = Field(..., description="경도")
    latitude: float = Field(..., description="위도")

class Attraction(BaseModel):
    name: str
    location: Location
    ticket_price: int = 0

# 객체 생성
attraction = Attraction(
    name="자금성",
    location=Location(longitude=116.397128, latitude=39.916527),
    ticket_price=60
)

# 타입 안정성이 확보된 접근
lng = attraction.location.longitude  # IDE가 코드 자동 완성을 제공합니다.
```

이 방식은 여러 장점이 있습니다. 첫째, 잘못된 타입을 전달하면(예: `ticket_price`를 문자열로 설정) Pydantic이 즉시 예외를 발생시켜 에러가 어디서 났는지 알려줍니다. 둘째, IDE가 타입 정의를 기반으로 자동 완성 및 타입 검사를 제공하여 오타를 크게 줄여줍니다. 마지막으로 데이터 구조를 수정해야 할 때 클래스 정의만 수정하면 해당 클래스를 사용하는 모든 곳이 자동으로 업데이트됩니다.

### 13.2.3 Pydantic의 핵심 개념

데이터 모델을 설계하기 전에 Pydantic의 핵심 개념 몇 가지를 먼저 이해해 보겠습니다. Pydantic의 기반은 `BaseModel` 클래스이며, 모든 데이터 모델은 이 클래스를 상속받아야 합니다. 각 필드는 타입을 지정할 수 있으며 Pydantic이 자동으로 타입 검사 및 변환을 수행합니다.

필드 정의는 `Field` 함수를 사용하며, 기본값, 설명, 검증 규칙 등을 지정할 수 있습니다. `...`은 이 필드가 필수(required)임을 의미합니다. 객체를 생성할 때 이 필드가 제공되지 않으면 Pydantic은 예외를 발생시킵니다. 또한 `Optional`을 사용해 선택적 필드를 나타내거나 직접 기본값을 제공할 수도 있습니다.

```python
from pydantic import BaseModel, Field
from typing import Optional, List

class Attraction(BaseModel):
    name: str = Field(..., description="관광지 이름")  # 필수
    rating: float = Field(default=0.0, ge=0, le=5)  # 기본값, 범위 검증
    visit_duration: int = Field(default=60, gt=0)  # 0보다 커야 함
    description: Optional[str] = None  # 선택적 필드
```

Pydantic은 중첩 모델(nested models)과 리스트도 지원합니다. 하나의 모델 안에서 다른 모델을 필드 타입으로 사용할 수 있으므로 복잡한 데이터 구조를 빌드할 수 있습니다. 예를 들어 관광지는 위치 정보를 포함하고, 여정은 여러 관광지를 포함합니다.

```python
class DayPlan(BaseModel):
    date: str
    attractions: List[Attraction]  # 관광지 목록
    hotel: Optional[Hotel] = None  # 선택적 호텔 정보
```

가장 강력한 기능 중 하나는 **커스텀 검증기(custom validators)**입니다. 때로는 외부 API가 반환하는 데이터 포맷이 우리의 요구사항과 맞지 않을 때가 있는데, 이때 `field_validator` 데코레이터를 사용하여 검증 및 변환 로직을 커스텀할 수 있습니다. 예를 들어 고덕 지도에서 반환하는 온도가 `"16°C"`와 같은 문자열일 때 이를 숫자로 변환해야 합니다:

```python
from pydantic import field_validator

class WeatherInfo(BaseModel):
    temperature: int

    @field_validator('temperature', mode='before')
    def parse_temperature(cls, v):
        """온도 문자열 파싱: "16°C" -> 16"""
        if isinstance(v, str):
            v = v.replace('°C', '').replace('℃', '').strip()
            return int(v)
        return v
```

이 검증기는 객체가 생성되기 전에 자동으로 실행되어 문자열을 정수로 변환합니다. 이렇게 하면 코드의 모든 곳에서 수동으로 온도 포맷을 처리할 필요가 없어집니다.

### 13.2.4 바텀업(Bottom-Up) 모델 디자인

이제 지능형 여행 비서의 데이터 모델 설계를 시작하겠습니다. 좋은 설계 원칙은 **바텀업(bottom-up)**입니다. 가장 기본적인 모델을 먼저 정의한 다음, 그것들을 점진적으로 결합하여 복잡한 구조를 만드는 것입니다. 이 방식의 장점은 각 모델이 단순하고 이해하기 쉬우며 유지보수가 용이하다는 것입니다.

가장 기본적인 모델은 **위치 정보**입니다. 관광지든 호텔이든 식당이든 모두 위치 정보가 필요합니다. 경도와 위도 좌표를 표현하는 `Location` 클래스를 정의합니다:

```python
class Location(BaseModel):
    """위치 정보 (경도 및 위도 좌표)"""
    longitude: float = Field(..., description="경도", ge=-180, le=180)
    latitude: float = Field(..., description="위도", ge=-90, le=90)
```

여기서는 범위 검증(`ge`는 크거나 같음, `le`는 작거나 같음)을 사용하여 경도와 위도 값이 합리적인 범위 내에 있도록 보장합니다.

다음은 **관광지 정보**입니다. 관광지는 이름, 주소, 위치, 방문 소요 시간, 설명, 평점, 이미지 및 티켓 가격 정보를 포함합니다. 필드 타입으로 중첩 모델인 `Location`을 사용하고 있음을 유념하십시오:

```python
class Attraction(BaseModel):
    """관광지 정보"""
    name: str = Field(..., description="관광지 이름")
    address: str = Field(..., description="주소")
    location: Location = Field(..., description="경위도 좌표")
    visit_duration: int = Field(..., description="추천 방문 소요 시간 (분)", gt=0)
    description: str = Field(..., description="관광지 설명")
    category: Optional[str] = Field(default="Attraction", description="관광지 카테고리")
    rating: Optional[float] = Field(default=None, ge=0, le=5, description="평점")
    image_url: Optional[str] = Field(default=None, description="이미지 URL")
    ticket_price: int = Field(default=0, ge=0, description="티켓 가격 (원/위안)")
```

마찬가지로 **식사 정보**와 **호텔 정보**를 정의합니다. 이 모델들은 이름, 주소, 위치, 비용 등의 기본 정보를 포함하는 유사한 구조를 가집니다:

```python
class Meal(BaseModel):
    """식사 정보"""
    type: str = Field(..., description="식사 타입: 아침/점심/저녁/간식")
    name: str = Field(..., description="식당/음식 이름")
    address: Optional[str] = Field(default=None, description="주소")
    location: Optional[Location] = Field(default=None, description="경위도 좌표")
    description: Optional[str] = Field(default=None, description="설명")
    estimated_cost: int = Field(default=0, description="예상 비용 (원/위안)")

class Hotel(BaseModel):
    """호텔 정보"""
    name: str = Field(..., description="호텔 이름")
    address: str = Field(default="", description="호텔 주소")
    location: Optional[Location] = Field(default=None, description="호텔 위치")
    price_range: str = Field(default="", description="가격 범위")
    rating: str = Field(default="", description="평점")
    distance: str = Field(default="", description="관광지와의 거리")
    type: str = Field(default="", description="호텔 타입")
    estimated_cost: int = Field(default=0, description="예상 비용 (원/위안/1박)")
```

**예산 정보**는 위치 정보는 없지만 다양한 지출의 요약을 담고 있는 특별한 모델입니다:

```python
class Budget(BaseModel):
    """예산 정보"""
    total_attractions: int = Field(default=0, description="총 관광지 티켓 비용")
    total_hotels: int = Field(default=0, description="총 호텔 비용")
    total_meals: int = Field(default=0, description="총 식사 비용")
    total_transportation: int = Field(default=0, description="총 교통 비용")
    total: int = Field(default=0, description="총 비용")
```

이제 이러한 기본 모델들을 결합하여 **일별 일정**을 구축할 수 있습니다. 일별 일정은 날짜, 설명, 교통 수단, 숙박 옵션, 호텔, 관광지 목록, 식사 목록을 포함합니다:

```python
class DayPlan(BaseModel):
    """일별 일정"""
    date: str = Field(..., description="날짜")
    day_index: int = Field(..., description="일차 번호 (0부터 시작)")
    description: str = Field(..., description="일별 일정 설명")
    transportation: str = Field(..., description="교통 수단")
    accommodation: str = Field(..., description="숙박 옵션")
    hotel: Optional[Hotel] = Field(default=None, description="호텔 정보")
    attractions: List[Attraction] = Field(default_factory=list, description="관광지 목록")
    meals: List[Meal] = Field(default_factory=list, description="식사 일정")
```

관광지 목록을 나타내기 위해 `List[Attraction]`을 사용했고, `default_factory=list`는 기본값이 빈 리스트임을 의미합니다.

**날씨 정보**는 고덕 지도에서 반환하는 온도 형식이 비표준적이기 때문에 특별한 처리가 필요합니다. 이를 처리하기 위해 커스텀 검증기를 사용합니다:

```python
class WeatherInfo(BaseModel):
    """날씨 정보"""
    date: str = Field(..., description="날짜")
    day_weather: str = Field(..., description="낮 날씨")
    night_weather: str = Field(..., description="밤 날씨")
    day_temp: int = Field(..., description="낮 온도 (섭씨)")
    night_temp: int = Field(..., description="밤 온도 (섭씨)")
    wind_direction: str = Field(..., description="풍향")
    wind_power: str = Field(..., description="풍력")

    @field_validator('day_temp', 'night_temp', mode='before')
    def parse_temperature(cls, v):
        """온도 문자열 파싱: "16°C" -> 16"""
        if isinstance(v, str):
            v = v.replace('°C', '').replace('℃', '').replace('°', '').strip()
            try:
                return int(v)
            except ValueError:
                return 0  # 에러 허용(예외 처리)
        return v
```

마지막으로 **전체 여행 계획**을 정의합니다. 이것은 모든 정보를 포함하는 최상위 모델입니다:

```python
class TripPlan(BaseModel):
    """여행 계획"""
    city: str = Field(..., description="목적지 도시")
    start_date: str = Field(..., description="시작 날짜")
    end_date: str = Field(..., description="종료 날짜")
    days: List[DayPlan] = Field(default_factory=list, description="일별 일정")
    weather_info: List[WeatherInfo] = Field(default_factory=list, description="날씨 정보")
    overall_suggestions: str = Field(..., description="종합 제안")
    budget: Optional[Budget] = Field(default=None, description="예산 정보")
```

이로써 전체 데이터 모델 설계를 완료했습니다. 가장 기본적인 `Location`부터 시작하여 `Attraction`, `Meal`, `Hotel`로 나아가고, 다시 `DayPlan`을 거쳐 최상위의 `TripPlan`에 도달하는 명확한 계층적 구조를 형성하게 되었습니다.

### 13.2.5 웹 애플리케이션에서 데이터 모델의 적용

이제 이 데이터 모델들이 실제 웹 애플리케이션에서 어떻게 사용되는지 보겠습니다. FastAPI에서 Pydantic 모델은 요청 및 응답의 타입 정의로 직접 사용될 수 있습니다. FastAPI는 데이터 검증, 직렬화 및 문서 생성을 자동으로 수행합니다.

```python
from fastapi import FastAPI
from app.models.schemas import TripPlanRequest, TripPlan

app = FastAPI()

@app.post("/api/trip/plan", response_model=TripPlan)
async def create_trip_plan(request: TripPlanRequest) -> TripPlan:
    """
    여행 계획 생성

    FastAPI가 다음을 자동으로 수행합니다:
    1. 요청 데이터 검증 (TripPlanRequest)
    2. 응답 데이터 검증 (TripPlan)
    3. OpenAPI 문서 생성
    """
    trip_plan = await generate_trip_plan(request)
    return trip_plan
```

사용자가 `/api/trip/plan`으로 POST 요청을 보낼 때, FastAPI는 자동으로 JSON 데이터를 `TripPlanRequest` 객체로 변환합니다. 데이터 형식이 올바르지 않으면(필수 필드 누락 또는 타입 불일치 등) FastAPI는 자동으로 400 에러를 반환하고 사용자에게 에러가 어디서 발생했는지 알려줍니다.

프론트엔드에서도 대응되는 TypeScript 타입을 정의해야 합니다. TypeScript와 Python은 서로 다른 언어이지만 데이터 구조는 동일합니다:

```typescript
interface Location {
  longitude: number;
  latitude: number;
}

interface Attraction {
  name: string;
  address: string;
  location: Location;
  visit_duration: number;
  ticket_price: number;
}

interface TripPlan {
  city: string;
  start_date: string;
  end_date: string;
  days: DayPlan[];
}
```

이렇게 하면 프론트엔드와 백엔드가 일관된 데이터 형식을 사용하게 됩니다. 백엔드가 `TripPlan` 객체를 반환할 때 프론트엔드는 별도의 변환 없이 이를 바로 사용할 수 있습니다. TypeScript의 타입 검사 기능도 많은 에러를 예방하는 데 도움을 줍니다.

## 13.3 다중 에이전트 협업 디자인

### 13.3.1 다중 에이전트가 필요한 이유

7장에서 우리는 SimpleAgent를 사용해 에이전트를 빌드하는 방법을 배웠습니다. SimpleAgent의 설계 철학은 단순하고 직관적입니다. `run()` 메서드가 호출될 때마다 에이전트는 사용자의 질문을 분석하고, 도구를 호출할지 결정한 후 결과를 반환합니다. 이 설계는 간단한 작업을 처리할 때 매우 유용하지만 여행 계획 수립과 같은 복잡한 작업을 만났을 때는 몇 가지 문제가 생깁니다.

만약 단일 에이전트로 여행 계획을 모두 완료하려고 한다면, 이 에이전트는 무엇을 해야 할까요? 첫째, 고덕 지도의 POI 검색 도구를 호출하여 관광지 정보를 검색해야 합니다. 둘째, 날씨 조회 도구를 호출하여 날씨 정보를 조회해야 합니다. 셋째, 다시 POI 검색 도구를 호출하여 호텔 정보를 검색해야 합니다. 마지막으로 이 모든 정보를 통합하여 완전한 여행 계획을 작성해야 합니다.

간단해 보이지만 실제 구동해보면 첫 번째 문제에 맞닥뜨리게 됩니다. 바로 **도구 호출의 제한**입니다. SimpleAgent는 한 번의 `run()` 호출에서 하나의 도구만 실행할 수 있습니다. 이는 우리가 `run()` 메서드를 여러 번 호출해야 하며, 각 호출마다 하나의 작업을 처리해야 함을 의미합니다. 그러나 이는 새로운 문제를 야기합니다. 여러 호출 사이에서 정보를 어떻게 전달할 것인가? 첫 번째 호출에서 얻은 관광지 정보를 두 번째 호출로 어떻게 전달할 것인가? 이러한 중간 결과를 수동으로 관리해야 하므로 코드가 매우 복잡해집니다.

물론 ReactAgent를 사용해 이 문제를 해결할 수도 있습니다. ReactAgent는 한 번의 호출로 여러 도구를 실행할 수 있으며 여러 라운드의 생각(Thinking)과 행동(Action)을 자동으로 진행합니다. 그러나 이는 **시간 비용**이라는 새로운 문제를 가져옵니다. ReactAgent의 생각 라운드마다 LLM을 호출해야 합니다. 3개의 도구를 호출해야 한다면 최소 3라운드의 생각이 필요하고 이는 최소 3번의 LLM 호출을 뜻합니다. 게다가 이 호출들은 순차적으로 실행되어 이전 호출이 끝나야 다음 호출을 시작할 수 있으므로 전체 소요 시간이 매우 길어집니다.

두 번째 문제는 **프롬프트의 복잡성**입니다. 하나의 에이전트가 모든 작업을 완료하게 하려면 각 작업의 실행 로직을 프롬프트에 상세히 설명해야 합니다. 예를 들어:

```python
COMPLEX_PROMPT = """당신은 여행 계획 비서입니다. 다음을 수행해야 합니다:
1. maps_text_search를 사용하여 사용자 선호도에 따른 키워드로 관광지를 검색합니다.
2. maps_weather를 사용하여 날씨를 조회하고, 향후 며칠 동안의 날씨 예보를 가져옵니다.
3. maps_text_search를 사용하여 사용자 요구에 맞는 유형의 호텔을 검색합니다.
4. 모든 정보를 통합하여 일별 관광지, 식사, 숙박 배치를 포함한 여행 계획을 생성합니다.
주의: 반드시 순서대로 실행해야 하며, 각 도구는 한 번만 호출할 수 있고, 출력은 JSON 형식이어야 합니다...
"""
```

이런 프롬프트는 몇 가지 문제가 있습니다. 첫째는 **유지보수의 어려움**입니다. 관광지 검색 로직을 수정하고 싶을 때(예: 평점 필터링 추가) 프롬프트 전체를 수정해야 하며, 이는 다른 부분에 쉽게 영향을 줄 수 있습니다. 둘째는 **에러가 발생하기 쉽다**는 점입니다. LLM이 여러 작업의 요구사항을 동시에 이해해야 하므로 서로 다른 작업의 형식과 매개변수를 혼동하기 쉽습니다. 마지막으로 **디버깅이 어렵다**는 점입니다. 생성된 계획이 예상과 다를 때 어느 단계에서 문제가 발생했는지 알기 어렵습니다. 관광지 검색이 부정확했는지, 날씨 조회가 실패했는지, 아니면 통합 로직에 문제가 있는지 파악하기 어렵기 때문입니다.

이러한 문제들을 마주했을 때 자연스럽게 떠오르는 아이디어는 바로 '복잡한 작업을 여러 개의 단순한 작업으로 분할하고 각 에이전트가 자신의 일에만 집중하게 할 수 없을까?' 하는 것입니다. 이것이 다중 에이전트 협업(Multi-Agent Collaboration)의 핵심 아이디어입니다.

현실 세계의 여행사를 생각해 보십시오. 여행사에 가서 여행 계획 상담을 받을 때 단 한 명의 직원에게만 서비스를 받지는 않습니다. 일반적으로 관광지 추천을 담당하는 전문 관광지 컨설턴트, 호텔 예약을 담당하는 호텔 컨설턴트, 그리고 모든 정보를 최종 일정표로 통합하는 일정 플래너가 따로 있습니다. 각자 자신의 전문 분야에 집중하고 마지막에 플래너가 모든 정보를 요약합니다. 이러한 분업과 협업은 한 사람이 모든 것을 처리하는 것보다 훨씬 효율적입니다.

### 13.3.2 에이전트 역할 디자인

작업 분할 원칙에 따라 우리는 그림 13.6과 같이 4개의 특화된 에이전트를 설계했습니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/13-figures/13-6.png" alt="" width="85%"/>
  <p>그림 13.6 다중 에이전트 협업 흐름</p>
</div>

- **AttractionSearchAgent (관광지 검색 전문가)**: 관광지 정보 검색에 집중합니다. 사용자의 선호도(예: "역사와 문화", "자연 경관")를 이해하고 고덕 지도의 POI 검색 도구를 호출하여 관련 관광지 목록을 반환하기만 하면 됩니다. 프롬프트는 선호도에 따라 키워드를 선택하고 도구를 호출하는 방법만 설명하므로 매우 간단합니다.

- **WeatherQueryAgent (날씨 조회 전문가)**: 날씨 정보 조회에 집중합니다. 도시 이름만 알면 날씨 조회 도구를 호출하고 향후 며칠간의 날씨 예보를 반환할 뿐입니다. 작업이 매우 명확하여 에러 발생 가능성이 거의 없습니다.

- **HotelAgent (호텔 추천 전문가)**: 호텔 정보 검색에 집중합니다. 사용자의 숙박 요구사항(예: "실속형", "고급형")을 파악하여 POI 검색 도구를 호출하고 조건에 맞는 호텔 목록을 반환합니다.

- **PlannerAgent (일정 계획 전문가)**: 모든 정보의 통합을 담당합니다. 앞선 세 에이전트의 출력값과 사용자의 원래 요구사항(날짜, 예산 등)을 입력받아 완전한 여행 계획을 생성합니다. 외부 도구를 전혀 호출하지 않으며 오직 정보 통합과 일정 배치에만 집중합니다.

이제 각 에이전트의 역할과 프롬프트를 상세하게 설계해 보겠습니다. 프롬프트를 설계할 때 우리는 다음과 같은 핵심 질문들을 고려해야 합니다: 이 에이전트에게 어떤 입력이 필요한가? 어떤 출력을 생성해야 하는가? 어떤 도구를 호출해야 하는가? 어떤 문제가 발생할 수 있는가?

**AttractionSearchAgent**의 작업은 사용자 선호도에 따라 관광지를 검색하는 것입니다. 입력값은 도시 이름과 사용자 선호도(예: "역사와 문화", "자연 경관")입니다. 키워드와 도시를 매개변수로 하여 `amap_maps_text_search` 도구를 호출해야 합니다. 출력값은 이름, 주소, 평점 등의 정보를 포함한 관광지 목록입니다.

```python
ATTRACTION_AGENT_PROMPT = """당신은 관광지 검색 전문가입니다.

**도구 호출 형식:**
`[TOOL_CALL:amap_maps_text_search:keywords=attraction,city=도시_이름]`

**예시:**
- `[TOOL_CALL:amap_maps_text_search:keywords=attraction,city=Beijing]`
- `[TOOL_CALL:amap_maps_text_search:keywords=museum,city=Shanghai]`

**중요 사항:**
- 반드시 도구를 사용하여 검색하고 정보를 날조하지 마십시오.
- 사용자의 선호도({preferences})를 기반으로 {city}의 관광지를 검색하십시오.
"""
```

이 프롬프트는 간결하지만 필요한 모든 정보를 담고 있습니다. 도구 호출 형식을 명확히 설명하고 구체적인 예시를 제공하며, 도구를 꼭 사용해야 한다는 점과 선호도에 기반해 검색해야 한다는 두 가지 중요 원칙을 강조합니다.

**WeatherQueryAgent**의 작업은 더 단순하여 날씨 조회만 하면 됩니다. 입력값은 도시 이름이고 출력값은 날씨 정보입니다.

```python
WEATHER_AGENT_PROMPT = """당신은 날씨 조회 전문가입니다.

**도구 호출 형식:**
`[TOOL_CALL:amap_maps_weather:city=도시_이름]`

{city}의 날씨 정보를 조회해 주십시오.
"""
```

**HotelAgent**의 작업은 호텔 검색입니다. 입력값은 도시 이름과 숙박 유형이며 출력값은 호텔 목록입니다.

```python
HOTEL_AGENT_PROMPT = """당신은 호텔 추천 전문가입니다.

**도구 호출 형식:**
`[TOOL_CALL:amap_maps_text_search:keywords=hotel,city=도시_이름]`

{city}에서 {accommodation} 조건을 만족하는 호텔을 검색해 주십시오.
"""
```

**PlannerAgent**는 모든 정보를 통합해야 하므로 가장 복잡합니다. 입력값은 사용자 요구사항과 세 에이전트의 출력값이며, 출력값은 JSON 형식의 완전한 여행 계획입니다.

```python
PLANNER_AGENT_PROMPT = """당신은 일정 계획 전문가입니다.

**출력 형식:**
반드시 다음 JSON 형식을 엄격히 준수하여 반환하십시오:
{
  "city": "도시 이름",
  "start_date": "YYYY-MM-DD",
  "end_date": "YYYY-MM-DD",
  "days": [...],
  "weather_info": [...],
  "overall_suggestions": "종합 제안",
  "budget": {...}
}

**계획 요구사항:**
1. weather_info는 매일의 날씨를 포함해야 합니다.
2. 온도는 순수 숫자 형식이어야 합니다 (뒤에 °C 없이).
3. 하루에 2~3개의 관광지를 배치하십시오.
4. 관광지 간 거리와 방문 시간을 고려하십시오.
5. 아침, 점심, 저녁 식사를 포함하십시오.
6. 실용적인 조언을 제공하십시오.
7. 예산 정보를 포함하십시오.
"""
```

### 13.3.3 에이전트 협업 흐름

이제 이 4개의 에이전트가 어떻게 협업하여 여행 계획 작업을 완수하는지 살펴보겠습니다. 전체 흐름은 5단계로 나눌 수 있습니다:

```python
class TripPlannerAgent:
    def __init__(self):
        self.attraction_agent = SimpleAgent(name="관광지 검색", prompt=ATTRACTION_PROMPT)
        self.weather_agent = SimpleAgent(name="날씨 조회", prompt=WEATHER_PROMPT)
        self.hotel_agent = SimpleAgent(name="호텔 추천", prompt=HOTEL_PROMPT)
        self.planner_agent = SimpleAgent(name="일정 계획", prompt=PLANNER_PROMPT)

    def plan_trip(self, request: TripPlanRequest) -> TripPlan:
        # 1단계: 관광지 검색
        attraction_response = self.attraction_agent.run(
            f"{request.city}에서 {request.preferences} 성향의 관광지를 검색해 주십시오."
        )

        # 2단계: 날씨 조회
        weather_response = self.weather_agent.run(
            f"{request.city}의 날씨를 조회해 주십시오."
        )

        # 3단계: 호텔 추천
        hotel_response = self.hotel_agent.run(
            f"{request.city}에서 {request.accommodation} 유형의 호텔을 검색해 주십시오."
        )

        # 4단계: 통합 및 계획 생성
        planner_query = self._build_planner_query(
            request, attraction_response, weather_response, hotel_response
        )
        planner_response = self.planner_agent.run(planner_query)

        # 5단계: JSON 파싱
        trip_plan = self._parse_trip_plan(planner_response)
        return trip_plan
```

이 흐름은 4개 단계를 순차적으로 실행하며, 각 단계의 출력이 다음 단계의 입력으로 사용됩니다. 13.2절에서 정의한 `TripPlanRequest`와 `TripPlan` Pydantic 모델을 사용하고 있음을 확인하십시오.

### 13.3.4 쿼리 생성

PlannerAgent는 모든 정보를 통합해야 합니다. 플래너용 쿼리는 필요한 모든 정보를 포함해야 하며 LLM이 정확하게 이해할 수 있도록 명확하고 질서정연하게 구성되어야 합니다.

```python
def _build_planner_query(
    self,
    request: TripPlanRequest,
    attraction_response: str,
    weather_response: str,
    hotel_response: str
) -> str:
    """계획 에이전트를 위한 쿼리 빌드"""
    return f"""
다음 정보를 바탕으로 {request.city}에 대한 {request.days}일간의 여행 계획을 생성해 주십시오:

**사용자 요구사항:**
- 목적지: {request.city}
- 일정: {request.start_date} ~ {request.end_date}
- 일수: {request.days}일
- 선호도: {request.preferences}
- 예산: {request.budget}
- 교통 수단: {request.transportation}
- 숙박 옵션: {request.accommodation}

**관광지 정보:**
{attraction_response}

**날씨 정보:**
{weather_response}

**호텔 정보:**
{hotel_response}

일별 관광지 배치, 식사 추천, 숙박 정보, 예산 세부 사항을 포함한 자세한 여행 계획을 생성해 주십시오.
"""
```

이러한 다중 에이전트 협업 설계를 통해 복잡한 여행 계획 작업을 4개의 간단한 서브 작업으로 해체할 수 있었습니다. 각 에이전트는 자신의 전문 영역에 집중할 수 있으며, 향후 새로운 기능 확장(예: 맛집 추천 에이전트, 교통 계획 에이전트 추가 등)을 위한 훌륭한 토대가 됩니다.

## 13.4 MCP 도구 통합 상세

### 13.4.1 API를 직접 호출하지 않는 이유

13.3절에서 우리는 여행 계획 작업을 완수하기 위해 협업하는 4개의 에이전트를 설계했습니다. 그중 AttractionSearchAgent, WeatherQueryAgent, HotelAgent는 모두 데이터를 얻기 위해 고덕 지도 API를 호출해야 합니다. 그렇다면 왜 에이전트 내부에서 고덕 지도의 HTTP API를 직접 호출하지 않을까요?

API를 직접 호출하는 방식이 어떻게 생겼는지 먼저 확인해 보겠습니다. 고덕 지도는 POI 검색 API를 제공하므로 다음과 같이 HTTP 요청을 구성하고 매개변수를 전달하며 응답을 파싱해야 합니다:

```python
import requests

def search_poi(keywords: str, city: str, api_key: str):
    """고덕 지도 POI 검색 API 직접 호출"""
    url = "https://restapi.amap.com/v3/place/text"
    params = {
        "keywords": keywords,
        "city": city,
        "key": api_key,
        "output": "json"
    }
    response = requests.get(url, params=params)
    data = response.json()
    return data
```

이 방법은 단순해 보이지만 실제 적용할 때는 몇 가지 문제가 발생합니다. 첫째는 **에이전트의 자율적 호출 불가**입니다. HelloAgents 프레임워크에서 에이전트는 프롬프트의 도구 호출 마커(예: `[TOOL_CALL:tool_name:arg1=value1]`)를 식별하여 도구를 호출합니다. 코드에서 API를 직접 호출하면 에이전트는 자율적인 의사결정 능력을 잃고 단순한 함수 호출기로 전락합니다.

둘째는 **복잡한 매개변수 전달**입니다. 고덕 지도 API에는 수많은 매개변수가 있습니다. 예를 들어 POI 검색은 `keywords`, `city`, `types`, `offset`, `page` 등 10개 이상의 매개변수가 있습니다. 에이전트가 이 매개변수들을 유연하게 사용하도록 하려면 프롬프트에 각 매개변수의 의미와 형식을 상세히 설명해야 하므로 프롬프트가 매우 복잡해집니다.

셋째는 **까다로운 응답 파싱**입니다. 고덕 지도 API가 반환하는 데이터는 비교적 복잡한 구조의 JSON 형식입니다. 이 데이터를 파싱하고 필요한 필드를 추출하는 코드를 직접 작성해야 합니다. API의 응답 형식이 바뀌면 파싱 코드도 수정해야 합니다.

마지막으로 **혼란스러운 도구 관리**입니다. 고덕 지도는 POI 검색, 날씨 조회, 경로 계획 등 10여 개의 서로 다른 API를 제공합니다. 각 API에 대해 개별 함수를 작성하고 에이전트의 도구 목록에 직접 등록해야 한다면 코드가 매우 길어질 것입니다. 또한 새 API를 추가하고 싶을 때 여러 곳을 수정해야 합니다.

### 13.4.2 고덕 지도(Amap) MCP 통합

MCP(Model Context Protocol)는 Anthropic이 제안한 LLM과 외부 도구를 연결하기 위한 표준 프로토콜입니다. 이 섹션에서는 프로젝트에 고덕 지도 MCP 서버를 통합하는 방법을 소개합니다. 우리 프로젝트는 Node.js로 구현된 MCP 서버인 `amap-mcp-server`를 사용합니다:

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/13-figures/13-7.png" alt="" width="85%"/>
  <p>그림 13.7 amap-mcp-server 도구 목록</p>
</div>

고덕 지도 MCP 서버는 표 13.1에 요약된 것처럼 다양한 도구를 제공하며 주로 다음과 같은 카테고리로 나뉩니다:

<div align="center">
  <p>표 13.1 고덕 지도 MCP 도구 분류</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/13-figures/13-table-1.png" alt="" width="85%"/>
</div>

MCP 프로토콜 덕분에 HelloAgents에서 매우 쉽게 이를 통합할 수 있습니다:

```python
from hello_agents.tools import MCPTool
from app.config import get_settings

settings = get_settings()

# MCP 도구 생성
mcp_tool = MCPTool(
    name="amap_mcp",
    command="npx",
    args=["-y", "@sugarforever/amap-mcp-server"],
    env={"AMAP_API_KEY": settings.amap_api_key},
    auto_expand=True
)
```

이 코드는 무엇을 할까요? 먼저 `command`와 `args`는 MCP 서버를 시작하는 방법을 지정합니다. `npx -y @sugarforever/amap-mcp-server`는 npm 저장소에서 `amap-mcp-server` 패키지를 다운로드하여 실행합니다. `env` 매개변수는 환경 변수를 전달하며, 여기서는 고덕 지도 API 키를 전달합니다.

**주의:** 이 문서의 일부 예제에서는 MCP(Model Context Protocol) 서비스를 구동하기 위해 `npx`를 사용합니다. 그러나 이 장에 해당하는 실제 코드 저장소에서는 `uvx`를 사용하고 있습니다. `npx`와 `uvx`는 거의 동일한 설계 원칙을 공유하며 단지 생태계의 차이일 뿐입니다. `npx`는 JavaScript/Node.js(npm 패키지)를 대상으로 하고 `uvx`는 Python(PyPI 패키지)을 대상으로 합니다. 두 방법 사이에 우열은 없으므로 필요에 맞게 선택하여 사용하시기 바랍니다.

`MCPTool` 객체를 생성하면 백엔드에서 MCP 서버 프로세스가 시작되고 표준 입출력(stdin/stdout)을 통해 서버와 통신합니다. 이는 프로세스 간 통신(IPC)을 사용하여 HTTP보다 더 효율적이고 관리하기 쉬운 MCP 프로토콜만의 특징입니다.

가장 중요한 매개변수는 `auto_expand=True`입니다. 이를 True로 설정하면 `MCPTool`은 MCP 서버가 제공하는 도구 목록을 자동으로 조회한 뒤 각 도구에 대한 독립적인 Tool 객체를 생성합니다. 우리가 단 하나의 `MCPTool`을 생성했음에도 에이전트가 16개의 도구를 사용할 수 있게 된 이유가 바로 이것입니다. 이 과정을 살펴보겠습니다:

```python
# 하나의 MCPTool 생성
mcp_tool = MCPTool(..., auto_expand=True)
agent.add_tool(mcp_tool)

# 에이전트는 실제로 16개의 도구를 획득합니다!
print(list(agent.tools.keys()))
# ['amap_maps_text_search', 'amap_maps_weather', ...]
```

그림 13.8에 나타난 것처럼 사용자가 북경(베이징) 관광지를 검색하기 원한다고 가정해 봅시다. AttractionSearchAgent는 "북경에서 역사와 문화 관련 관광지를 검색해 주십시오"라는 질의를 받습니다. 에이전트는 질의를 분석하고 `keywords=attraction, city=Beijing` 매개변수로 `amap_maps_text_search` 도구를 호출하기로 결정합니다.

<div align="center">
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/13-figures/13-8.png" alt="" width="85%"/>
  <p>그림 13.8 MCP 도구 호출 흐름</p>
</div>

에이전트는 도구 호출 마커를 생성합니다: `[TOOL_CALL:amap_maps_text_search:keywords=attraction,city=Beijing]`. HelloAgents 프레임워크는 이 마커를 파싱하여 도구 이름과 매개변수를 추출한 다음 대응되는 Tool 객체를 호출합니다.

이 Tool 객체는 `MCPTool`에 의해 자동으로 생성되었으며 호출 요청을 MCP 서버로 전송합니다. 구체적으로는 JSON-RPC 형식의 메시지를 구성하여 표준 입력(stdin)을 통해 서버 프로세스로 보냅니다:

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "amap_maps_text_search",
    "arguments": {
      "keywords": "attraction",
      "city": "Beijing"
    }
  }
}
```

MCP 서버는 이 메시지를 받아 매개변수를 파싱하고 고덕 지도 HTTP API를 호출합니다. HTTP 요청을 구성하고 API 키를 추가하여 요청을 보내고 응답을 받습니다.

고덕 지도 API는 관광지 목록, 주소, 좌표 등의 정보를 포함하는 JSON 데이터를 반환합니다. MCP 서버는 이 데이터를 파싱하고 주요 필드를 추출하여 응답 메시지를 구성한 뒤 표준 출력(stdout)을 통해 `MCPTool`로 반환합니다:

```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "다음 관광지들을 찾았습니다:\n1. 고궁박물원(자금성) - 주소: 동城区 경산전가 4호\n2. 천단공원 - 주소: 동城区 천단로\n..."
      }
    ]
  }
}
```

`MCPTool`은 응답을 받아 텍스트 콘텐츠를 추출하고 이를 에이전트로 반환합니다. 에이전트는 이 결과를 도구 호출의 출력값으로 사용하여 최종 답변을 계속 생성해 나갑니다.

이 과정이 복잡해 보일 수 있지만, 에이전트 입장에서는 관광지를 검색할 수 있는 `amap_maps_text_search`라는 도구가 존재한다는 것만 알면 됩니다. 모든 하부 세부 사항은 MCP 프로토콜과 `MCPTool`에 의해 캡슐화되어 있습니다.

### 13.4.3 MCP 인스턴스 공유

우리 다중 에이전트 시스템에서 3개의 에이전트는 모두 고덕 지도 도구를 사용해야 합니다. 그렇다면 각 에이전트가 각자 독립된 `MCPTool` 인스턴스를 생성해야 할까요, 아니면 동일한 인스턴스를 공유해야 할까요?

각 에이전트가 독립된 `MCPTool` 인스턴스를 생성하면 3개의 서버 프로세스가 동시에 실행된다는 것을 의미합니다. 각 프로세스가 독립적으로 고덕 지도 API를 호출하므로 API의 호출 속도 제한(rate limit)을 초과할 수 있습니다. 게다가 여러 프로세스는 더 많은 메모리와 CPU 리소스를 점유합니다.

더 나은 접근 방식은 모든 에이전트가 동일한 `MCPTool` 인스턴스를 공유하는 것입니다. 이렇게 하면 단 하나의 MCP 서버 프로세스만 구동되며 모든 API 호출이 이 프로세스를 통해 처리됩니다. 이는 리소스를 절약할 뿐만 아니라 API 호출 빈도를 더 잘 제어할 수 있게 해줍니다.

코드에서는 `TripPlannerAgent` 생성자에서 `MCPTool` 인스턴스를 생성한 다음 각 서브 에이전트의 도구 목록에 추가합니다:

```python
class TripPlannerAgent:
    def __init__(self):
        settings = get_settings()
        self.llm = HelloAgentsLLM()

        # 공유 MCP 도구 인스턴스 생성 (단 한 번만 생성)
        self.mcp_tool = MCPTool(
            name="amap_mcp",
            command="npx",
            args=["-y", "@sugarforever/amap-mcp-server"],
            env={"AMAP_API_KEY": settings.amap_api_key},
            auto_expand=True
        )

        # 여러 에이전트를 생성하고 동일한 MCP 도구를 공유
        self.attraction_agent = SimpleAgent(
            name="AttractionSearchAgent",
            llm=self.llm,
            system_prompt=ATTRACTION_AGENT_PROMPT
        )
        self.attraction_agent.add_tool(self.mcp_tool)  # 공유

        self.weather_agent = SimpleAgent(
            name="WeatherQueryAgent",
            llm=self.llm,
            system_prompt=WEATHER_AGENT_PROMPT
        )
        self.weather_agent.add_tool(self.mcp_tool)  # 공유

        self.hotel_agent = SimpleAgent(
            name="HotelAgent",
            llm=self.llm,
            system_prompt=HOTEL_AGENT_PROMPT
        )
        self.hotel_agent.add_tool(self.mcp_tool)  # 공유
```

이렇게 하면 3개의 에이전트 모두 고덕 지도의 16개 도구를 사용할 수 있지만, 그 아래에서는 단 하나의 MCP 서버 프로세스만 동작합니다. `TripPlannerAgent`의 `plan_trip` 메서드를 호출하면 3개의 에이전트가 순차적으로 도구를 호출하며 모든 요청은 동일한 MCP 서버를 통해 고덕 지도 API로 전송됩니다.

### 13.4.4 Unsplash 이미지 API 통합

고덕 지도 외에도 여행 계획을 더욱 생생하고 직관적으로 만들기 위해 관광지 이미지를 확보해야 합니다. 우리는 관광지 이미지를 검색하기 위해 Unsplash API를 사용합니다. 단, Unsplash는 해외 서비스이며 무료로 사용할 수 있는 몇 안 되는 이미지 API 중 하나이므로 검색 결과가 아주 정확하지는 않을 수 있다는 점에 유의하십시오. 실제 프로젝트에서는 Bing, Baidu 또는 고덕 지도의 POI 이미지 API를 사용하는 것을 고려해 볼 수 있으나, 일반적으로 유료 서비스입니다.

Unsplash API 통합은 비교적 단순합니다. API 호출을 캡슐화하기 위해 `UnsplashService` 클래스를 생성합니다:

```python
# backend/app/services/unsplash_service.py
import requests
from typing import Optional, List, Dict
import logging

logger = logging.getLogger(__name__)

class UnsplashService:
    """Unsplash 이미지 서비스"""

    def __init__(self, access_key: str):
        self.access_key = access_key
        self.base_url = "https://api.unsplash.com"

    def search_photos(self, query: str, per_page: int = 10) -> List[Dict]:
        """이미지 검색"""
        try:
            url = f"{self.base_url}/search/photos"
            params = {
                "query": query,
                "per_page": per_page,
                "client_id": self.access_key
            }

            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()

            data = response.json()
            results = data.get("results", [])

            # 이미지 URL 추출
            photos = []
            for result in results:
                photos.append({
                    "url": result["urls"]["regular"],
                    "description": result.get("description", ""),
                    "photographer": result["user"]["name"]
                })

            return photos

        except Exception as e:
            logger.error(f"이미지 검색 실패: {e}")
            return []

    def get_photo_url(self, query: str) -> Optional[str]:
        """단일 이미지 URL 가져오기"""
        photos = self.search_photos(query, per_page=1)
        return photos[0].get("url") if photos else None
```

이 서비스 클래스는 두 가지 메서드를 제공합니다: `search_photos`는 여러 이미지를 검색하고, `get_photo_url`은 단일 이미지의 URL을 가져옵니다. 우리는 이 서비스를 API 라우트에서 활용하여 각 관광지의 이미지를 가져옵니다:

```python
# backend/app/api/routes/trip.py
from app.services.unsplash_service import UnsplashService

unsplash_service = UnsplashService(settings.unsplash_access_key)

@router.post("/plan", response_model=TripPlan)
async def create_trip_plan(request: TripPlanRequest) -> TripPlan:
    # 여행 계획 생성
    trip_plan = trip_planner_agent.plan_trip(request)

    # 각 관광지의 이미지 가져오기
    for day in trip_plan.days:
        for attraction in day.attractions:
            if not attraction.image_url:
                image_url = unsplash_service.get_photo_url(
                    f"{attraction.name} {trip_plan.city}"
                )
                attraction.image_url = image_url

    return trip_plan
```

여기서는 Unsplash를 도구나 MCP 도구로 캡슐화하지 않고 API 라우트에서 직접 호출했다는 점에 유의하십시오. 이미지 검색은 에이전트의 지능적인 의사결정이 필요하지 않은 단순한 데이터 보강(Data Enhancement) 단계이기 때문입니다. 에이전트가 이미지 필요 여부를 자율적으로 판단하거나 다른 이미지 소스를 직접 선택하도록 만들고 싶다면 이를 도구(Tool)로 캡슐화하는 것을 고려할 수 있습니다.

## 13.5 프론트엔드 개발 상세

### 13.5.1 프론트엔드와 백엔드가 분리된 웹 아키텍처

프론트엔드 개발을 시작하기 전에 현대 웹 애플리케이션의 아키텍처 패턴을 이해해야 합니다. 초기 웹 개발에서는 프론트엔드와 백엔드가 한데 섞여 있었습니다. 예를 들어 PHP나 JSP 같은 기술은 HTML 템플릿과 비즈니스 로직 코드가 하나의 파일에 작성되었습니다. 이 방식은 소규모 프로젝트에는 편리하지만 규모가 커지면 많은 문제에 직면합니다. 프론트엔드와 백엔드 개발자 간의 빈번한 조율이 필요하고 코드 재사용이 어려우며 테스트가 곤란해집니다.

현대 웹 애플리케이션은 일반적으로 **프론트엔드와 백엔드가 분리된** 아키텍처를 채택합니다. 백엔드는 API 인터페이스만 제공하고 JSON 형식으로 데이터를 반환하는 역할을 수행합니다. 프론트엔드는 독립된 애플리케이션으로서 HTTP 요청을 통해 백엔드 API를 호출하고 데이터를 획득하여 페이지를 렌더링합니다. 이 아키텍처는 몇 가지 뚜렷한 장점이 있습니다: 프론트엔드와 백엔드를 독립적으로 개발, 배포, 테스트할 수 있고, 프론트엔드가 웹 애플리케이션이든 모바일 앱이든 데스크톱 앱이든 관계없이 동일한 백엔드 API 세트를 사용할 수 있으며, 프론트엔드가 최신 프레임워크와 도구 체인을 사용하여 더 나은 사용자 경험을 제공할 수 있습니다.

우리 지능형 여행 비서 프로젝트에서 백엔드는 Python과 FastAPI로 구현되어 여행 요구사항을 입력받아 계획을 반환하는 핵심 API 인터페이스 `POST /api/trip/plan`을 제공합니다. 프론트엔드는 Vue 3와 TypeScript로 구현된 단일 페이지 애플리케이션(SPA)입니다. 사용자가 브라우저에서 양식을 작성하고 "계획 시작" 버튼을 클릭하면 프론트엔드가 백엔드로 HTTP 요청을 보내고 응답을 기다린 후 결과 페이지를 렌더링합니다. 이 모든 과정에서 페이지 전체가 새로고침되지 않아 사용자 경험이 매우 매끄럽습니다.

프론트엔드 기술 스택은 개발 효율성, 성능, 생태계, 학습 곡선 등을 고려하여 선택해야 합니다. 표 13.2에 정리된 것처럼 이 프로젝트는 다음과 같은 기술 스택을 선택했습니다:

<div align="center">
  <p>표 13.2 프론트엔드 기술 스택</p>
  <img src="https://raw.githubusercontent.com/datawhalechina/Hello-Agents/main/docs/images/13-figures/13-table-2.png" alt="" width="85%"/>
</div>

프로젝트의 디렉터리 구조는 다음과 같습니다:

```
frontend/
├── src/
│   ├── views/              # 페이지 컴포넌트
│   │   ├── Home.vue        # 홈 페이지 (양식)
│   │   └── Result.vue      # 결과 페이지
│   ├── services/           # API 서비스
│   │   └── api.ts
│   ├── types/              # 타입 정의
│   │   └── index.ts
│   ├── router/             # 라우터 설정
│   │   └── index.ts
│   ├── App.vue
│   └── main.ts
├── package.json
├── vite.config.ts
└── tsconfig.json
```

`views` 디렉터리는 페이지 컴포넌트를, `services` 디렉터리는 API 호출 로직을, `types` 디렉터리는 TypeScript 타입 정의를, 그리고 `router` 디렉터리는 라우트 설정을 저장합니다.

### 13.5.2 타입 정의

13.2절에서 백엔드에서 Pydantic을 사용해 `Location`, `Attraction`, `DayPlan`, `TripPlan` 등의 데이터 모델을 정의했습니다. 프론트엔드에서도 이에 대응하는 TypeScript 타입을 정의해야 합니다.

이 타입들을 정의하는 방법을 보겠습니다. 먼저 가장 기본적인 경위도 좌표를 나타내는 `Location` 타입입니다:

```typescript
// frontend/src/types/index.ts
export interface Location {
  longitude: number
  latitude: number
}
```

이 타입 정의는 백엔드 Pydantic 모델과 정확히 매칭됩니다. TypeScript에서는 `interface` 키워드를 사용하여 타입을 정의하며 필드 타입은 콜론으로 구분하고 기본값은 생략한다는 점을 유념하십시오.

다음은 관광지 정보를 나타내는 `Attraction` 타입입니다:

```typescript
export interface Attraction {
  name: string
  address: string
  location: Location
  visit_duration: number
  description: string
  category?: string
  rating?: number
  image_url?: string
  ticket_price?: number
}
```

여기서 `Location` 타입을 필드 타입으로 중첩하여 사용하고 있습니다. 물음표 `?`는 선택적(optional) 필드임을 나타내며, 백엔드 Pydantic 모델의 `Optional`에 대응합니다.

동일한 방식으로 `Meal`, `Hotel`, `Budget`, `WeatherInfo` 등의 타입을 정의합니다. 마지막으로 최상위의 `TripPlan` 타입입니다:

```typescript
export interface TripPlan {
  city: string
  start_date: string
  end_date: string
  days: DayPlan[]
  weather_info: WeatherInfo[]
  overall_suggestions: string
  budget?: Budget
}
```

백엔드 요청 모델에 대응하는 요청 타입 `TripPlanRequest`도 존재합니다:

```typescript
export interface TripPlanRequest {
  city: string
  start_date: string
  end_date: string
  days: number
  preferences: string
  budget: string
  transportation: string
  accommodation: string
}
```

이 타입 정의들이 필요한 이유는 무엇일까요? 첫째, 우리가 API를 호출할 때 TypeScript는 전달하는 데이터가 `TripPlanRequest` 타입을 만족하는지 검사합니다. 실수로 `days`를 문자열로 적으면 즉시 에러를 표시합니다. 둘째, API 응답을 받을 때 TypeScript는 응답 데이터가 `TripPlan` 타입에 부합하는지 검증합니다. 백엔드의 데이터 구조가 변경되면 프론트엔드에서 즉시 알 수 있습니다. 마지막으로 IDE가 타입 정의를 기반으로 자동 완성을 제공합니다. `tripPlan.`을 입력하면 사용 가능한 모든 필드가 목록으로 자동으로 표시됩니다.

### 13.5.3 API 서비스 캡슐화

타입 정의를 마친 후 API 호출 부분을 캡슐화할 수 있습니다. `api.ts` 파일을 생성하고 Axios를 사용하여 HTTP 요청을 전송합니다:

```typescript
import axios from 'axios'
import type { TripPlanRequest, TripPlan } from '../types'

const api = axios.create({
  baseURL: 'http://localhost:8000/api',
  timeout: 120000, // 2분 제한시간 (Timeout)
  headers: {
    'Content-Type': 'application/json'
  }
})
```

여기서 Axios 인스턴스를 생성하고 기본 URL, 타임아웃, 요청 헤더를 구성합니다. 타임아웃을 2분으로 설정한 이유는 무엇일까요? 여행 계획을 생성하려면 여러 에이전트를 호출해야 하고, 각 에이전트는 LLM과 외부 API를 호출해야 하므로 전체 프로세스에 10~30초 이상이 소요될 수 있기 때문입니다. 타임아웃이 너무 짧으면 요청이 중간에 차단됩니다.

다음으로 인터셉터(Interceptors)를 추가합니다. 인터셉터는 요청을 보내기 전이나 응답을 받은 후에 로깅, 에러 처리, 인증 등의 공통 로직을 실행할 수 있게 해줍니다:

```typescript
// 요청 인터셉터
api.interceptors.request.use(
  config => {
    console.log('요청 전송:', config)
    return config
  },
  error => Promise.reject(error)
)

// 응답 인터셉터
api.interceptors.response.use(
  response => {
    console.log('응답 수신:', response)
    return response
  },
  error => {
    console.error('요청 실패:', error)
    return Promise.reject(error)
  }
)
```

마지막으로 프론트엔드에서 백엔드를 호출할 유일한 진입점인 API 함수를 정의합니다:

```typescript
// 여행 계획 생성
export const generateTripPlan = async (request: TripPlanRequest): Promise<TripPlan> => {
  const response = await api.post<TripPlan>('/trip/plan', request)
  return response.data
}
```

이 함수의 타입 시그니처를 보십시오. 매개변수는 `TripPlanRequest` 타입이고 반환값은 `Promise<TripPlan>` 타입입니다. 이는 호출자가 넘겨주는 매개변수가 형식에 맞는지 TypeScript가 검사하고 반환값을 올바르게 사용하는지도 보장해 줌을 의미합니다.

### 13.5.4 홈 폼 디자인

홈페이지는 사용자가 진입하는 지점이며 여행 요구사항을 작성하는 폼을 포함합니다. 우리는 코드를 구성하기 위해 Vue 3의 Composition API를 사용합니다:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { message } from 'ant-design-vue'
import { generateTripPlan } from '@/services/api'
import type { TripPlanRequest } from '@/types'

const router = useRouter()
const loading = ref(false)
const loadingProgress = ref(0)
const loadingStatus = ref('')

const formData = ref<TripPlanRequest>({
  city: '',
  start_date: '',
  end_date: '',
  days: 3,
  preferences: '역사와 문화',
  budget: '보통',
  transportation: '대중교통',
  accommodation: '실속형 호텔'
})
</script>
```

여기서 `ref`를 사용하여 반응형 변수를 만듭니다. `formData`는 `TripPlanRequest` 타입의 폼 데이터입니다. `loading`은 로딩 중인지 여부를, `loadingProgress`는 로딩 진행률을, `loadingStatus`는 로딩 상태 텍스트를 나타냅니다.

폼 제출 로직은 다음과 같습니다:

```typescript
const handleSubmit = async () => {
  loading.value = true
  loadingProgress.value = 0

  // 진행률 업데이트 시뮬레이션
  const progressInterval = setInterval(() => {
    if (loadingProgress.value < 90) {
      loadingProgress.value += 10
      if (loadingProgress.value <= 30) loadingStatus.value = '🔍 관광지 검색 중...'
      else if (loadingProgress.value <= 50) loadingStatus.value = '🌤️ 날씨 조회 중...'
      else if (loadingProgress.value <= 70) loadingStatus.value = '🏨 호텔 추천 중...'
      else loadingStatus.value = '📋 일정 생성 중...'
    }
  }, 500)

  try {
    const response = await generateTripPlan(formData.value)
    clearInterval(progressInterval)
    loadingProgress.value = 100
    router.push({ name: 'result', state: { tripPlan: response } })
  } catch (error) {
    clearInterval(progressInterval)
    message.error('계획 생성에 실패했습니다. 다시 시도해 주십시오.')
  } finally {
    loading.value = false
  }
}
```

이 코드는 몇 가지 작업을 처리합니다. 첫째, `loading`을 true로 설정하여 로딩 화면을 띄웁니다. 그런 다음 500밀리초마다 진행률 표시줄과 상태 텍스트를 업데이트하는 타이머를 작동시킵니다. 백엔드의 실제 진행률을 정확히 파악할 수 없기 때문에 시뮬레이션된 진행률이지만, 사용자가 시스템이 멈춘 것이 아니라 동작하고 있음을 느끼게 해줍니다.

다음으로 `generateTripPlan` 함수를 호출하여 API 요청을 보냅니다. 이는 비동기 작업이므로 `await`를 사용하여 응답을 대기합니다. 요청이 성공하면 타이머를 클리어하고 진행률을 100%로 설정한 후 결과 페이지로 이동하여 여행 계획 데이터를 전달합니다. 요청이 실패하면 에러 메시지를 보여줍니다. 성공 여부와 관계없이 최종적으로 `loading`을 false로 바꾸어 로딩 상태를 해제합니다.

템플릿 부분은 Ant Design Vue 컴포넌트를 사용합니다:

```vue
<template>
  <div class="home-container">
    <div class="page-header">
      <h1 class="page-title">✈️ 지능형 여행 비서</h1>
      <p class="page-subtitle">AI 기반 개인 맞춤형 여행 계획 수립</p>
    </div>

    <a-card class="form-card">
      <a-form :model="formData" @finish="handleSubmit">
        <a-form-item label="목적지 도시" name="city" :rules="[{ required: true }]">
          <a-input v-model:value="formData.city" placeholder="예: 북경" />
        </a-form-item>

        <!-- 추가 폼 항목들... -->

        <a-form-item>
          <a-button type="primary" html-type="submit" size="large" :loading="loading">
            계획 시작
          </a-button>
        </a-form-item>

        <!-- 로딩 진행 표시줄 -->
        <a-form-item v-if="loading">
          <a-progress :percent="loadingProgress" status="active" />
          <p>{{ loadingStatus }}</p>
        </a-form-item>
      </a-form>
    </a-card>
  </div>
</template>
```

`v-model:value` 디렉티브는 양방향 데이터 바인딩을 구현합니다. 사용자가 입력창에 글자를 쓰면 `formData.city`가 자동으로 업데이트되며 반대의 경우도 마찬가지입니다.

### 13.5.5 결과 페이지 표시

결과 페이지는 전체 애플리케이션의 핵심으로, 생성된 여행 계획을 표시합니다. 이 페이지는 일정 개요, 예산 상세 정보, 지도 시각화, 일별 일정 상세 정보 및 날씨 정보를 포함합니다.

첫째는 지도 시각화입니다. 고덕 지도 JS API를 사용하여 지도 위에 관광지 위치를 마킹합니다:

```typescript
import AMapLoader from '@amap/amap-jsapi-loader'

const initMap = async () => {
  const AMap = await AMapLoader.load({
    key: 'your_amap_web_key',
    version: '2.0'
  })

  map = new AMap.Map('amap-container', {
    zoom: 12,
    center: [116.397128, 39.916527]
  })

  // 관광지 마커 추가
  tripPlan.value.days.forEach((day) => {
    day.attractions.forEach((attraction, index) => {
      const marker = new AMap.Marker({
        position: [attraction.location.longitude, attraction.location.latitude],
        title: attraction.name,
        label: { content: `${index + 1}`, direction: 'top' }
      })
      map.add(marker)
    })
  })
}
```

이 코드는 고덕 지도 SDK를 먼저 로드하고 지도 인스턴스를 생성한 다음, 모든 관광지를 순회하며 각각 마커를 생성합니다. 마커의 좌표는 백엔드의 `Attraction` 객체에서 받아온 경위도 값을 이용합니다.

내보내기 기능은 `html2canvas`와 `jsPDF` 라이브러리를 활용합니다. `html2canvas`는 DOM 엘리먼트를 Canvas로 변환하고, 이후에 Canvas를 이미지나 PDF로 내보낼 수 있게 해줍니다:

```typescript
import html2canvas from 'html2canvas'
import jsPDF from 'jspdf'

// 이미지로 내보내기
const exportAsImage = async () => {
  const element = document.getElementById('trip-plan-content')
  const canvas = await html2canvas(element, { scale: 2 })
  const link = document.createElement('a')
  link.download = `${tripPlan.value.city} 여행 계획.png`
  link.href = canvas.toDataURL()
  link.click()
}

// PDF로 내보내기
const exportAsPDF = async () => {
  const element = document.getElementById('trip-plan-content')
  const canvas = await html2canvas(element, { scale: 2 })
  const imgData = canvas.toDataURL('image/png')
  const pdf = new jsPDF('p', 'mm', 'a4')
  const imgWidth = 210
  const imgHeight = (canvas.height * imgWidth) / canvas.width
  pdf.addImage(imgData, 'PNG', 0, 0, imgWidth, imgHeight)
  pdf.save(`${tripPlan.value.city} 여행 계획.pdf`)
}
```

이러한 프론트엔드 기술을 통해 우리는 하나의 완전한 웹 애플리케이션을 구현했습니다. 사용자는 브라우저에서 폼을 입력하고 요청을 제출한 뒤 AI가 여행 계획을 생성하는 것을 기다렸다가 상세 일정을 확인하고 지도 상의 위치를 확인하며 이미지나 PDF로 다운로드할 수 있습니다. 전체 과정이 매우 부드럽고 자연스러운데, 이것이 바로 모던 웹 애플리케이션의 매력입니다.

## 13.6 기능 구현 상세

이 섹션에서는 예산 계산, 로딩 진행 표시줄, 일정 편집, 내보내기 기능, 측면 내비게이션을 포함한 지능형 여행 비서의 핵심 기능 구현에 관해 알아봅니다.

### 13.6.1 예산 계산 기능

여행을 계획할 때 예산은 매우 중요한 고려사항입니다. 사용자는 이 여행에 대략 얼마가 필요하고 비용이 어디에 쓰이는지 파악하고 싶어 합니다. 지능형 여행 비서는 관광지 입장료, 호텔 숙박비, 식비, 교통비의 4가지 큰 범주로 나누어 자동으로 예산을 계산하는 기능을 제공합니다.

예산 계산 로직은 어디에 구현되어 있을까요? 우리는 백엔드의 PlannerAgent에 이를 구현하기로 결정했습니다. 왜 프론트엔드에서 계산하지 않을까요? 예산 추정은 관광지 티켓 가격, 호텔 가격 범위, 식사 기준, 교통수단 정보 등을 기반으로 해야 하는데, 이 정보는 이미 PlannerAgent가 여정을 생성할 때 함께 얻기 때문입니다. 프론트엔드에서 이를 계산한다면 이 로직을 중복 구현해야 하며 정확하지 않을 수도 있습니다.

PlannerAgent의 프롬프트에서 우리는 LLM이 예산 정보를 생성하도록 명확하게 요구합니다:

```python
PLANNER_AGENT_PROMPT = """
당신은 일정 계획 전문가입니다.

**출력 형식:**
반드시 다음 JSON 형식을 엄격히 준수하여 반환하십시오:
{
  ...
  "budget": {
    "total_attractions": 180,
    "total_hotels": 1200,
    "total_meals": 480,
    "total_transportation": 200,
    "total": 2060
  }
}

**계획 요구사항:**
...
7. 관광지 입장료, 호텔 가격, 식사 비용, 교통수단을 바탕으로 금액을 추정하여 예산 정보를 포함하십시오.
"""
```

LLM은 여정에 포함된 관광지, 호텔, 식사 구성에 맞추어 항목별 비용을 계산합니다. 예를 들어 일정에 자금성(입장료 60위안), 천단(15위안), 이화원(30위안)이 포함되어 있다면 총 관광지 입장료는 105위안으로 책정됩니다. 실속형 호텔(1박에 300위안)에 머무는 2박 3일 여행이라면 호텔 총비용은 600위안이 됩니다.

프론트엔드에서는 예산 정보를 표현하기 위해 Ant Design Vue의 Statistic 컴포넌트를 사용합니다. 이 컴포넌트는 통계 데이터를 표시하도록 설계되었으며 숫자 애니메이션, 접두사/접미사, 커스텀 스타일 등을 지원합니다:

```vue
<a-card v-if="tripPlan.budget" title="💰 예산 상세 정보">
  <a-row :gutter="16">
    <a-col :span="6">
      <a-statistic title="관광지 입장료" :value="tripPlan.budget.total_attractions" suffix="원" />
    </a-col>
    <a-col :span="6">
      <a-statistic title="호텔 숙박비" :value="tripPlan.budget.total_hotels" suffix="원" />
    </a-col>
    <a-col :span="6">
      <a-statistic title="식사 비용" :value="tripPlan.budget.total_meals" suffix="원" />
    </a-col>
    <a-col :span="6">
      <a-statistic title="교통비" :value="tripPlan.budget.total_transportation" suffix="원" />
    </a-col>
  </a-row>
  <a-divider />
  <a-row>
    <a-col :span="24" style="text-align: center;">
      <a-statistic
        title="예상 총비용"
        :value="tripPlan.budget.total"
        suffix="원"
        :value-style="{ color: '#cf1322', fontSize: '32px', fontWeight: 'bold' }"
      />
    </a-col>
  </a-row>
</a-card>
```

이 코드는 그리드 레이아웃(`a-row` 및 `a-col`)을 사용하여 4개의 비용 항목을 가로로 정렬해 보여줍니다. 각 비용은 `a-statistic` 컴포넌트로 제목과 값을 표시합니다. 구분선(`a-divider`)을 넣고 그 아래에는 큰 빨간색 글씨로 예상 총비용을 강조하여 나타냅니다.

`v-if="tripPlan.budget"`이라는 조건부 렌더링에 주목해 주십시오. 예산 정보는 선택 사항(Pydantic 모델에서 `Optional[Budget]`으로 정의됨)이므로, LLM이 예산 데이터를 생성하지 않은 경우 이 카드는 화면에 렌더링되지 않습니다. 이는 예외적인 데이터 상황에 대처하는 프론트엔드의 유연성을 보여줍니다.

### 13.6.2 로딩 진행 표시줄

여행 계획 생성은 많은 시간이 걸리는 작업입니다. 백엔드는 AttractionSearchAgent, WeatherQueryAgent, HotelAgent, PlannerAgent를 차례대로 호출해야 하고 각 에이전트는 LLM과 외부 API를 호출해야 합니다. 전체 작업에 10~30초가 소요됩니다. 사용자가 "계획 시작" 버튼을 눌렀는데 화면에 아무 반응이 없으면, 사용자는 시스템이 멈춘 것으로 오해하고 페이지를 새로고침하거나 버튼을 연속해서 클릭할 수 있습니다.

사용자 경험을 향상하기 위해 우리는 로딩 진행 표시줄과 상태 문구를 제공합니다. 비록 시뮬레이션된 과정이지만 사용자가 시스템이 정상 작동 중임을 알게 해줍니다:

```typescript
const loading = ref(false)
const loadingProgress = ref(0)
const loadingStatus = ref('')

const handleSubmit = async () => {
  loading.value = true
  loadingProgress.value = 0

  // 진행률 업데이트 시뮬레이션
  const progressInterval = setInterval(() => {
    if (loadingProgress.value < 90) {
      loadingProgress.value += 10
      if (loadingProgress.value <= 30) loadingStatus.value = '🔍 관광지 검색 중...'
      else if (loadingProgress.value <= 50) loadingStatus.value = '🌤️ 날씨 조회 중...'
      else if (loadingProgress.value <= 70) loadingStatus.value = '🏨 호텔 추천 중...'
      else loadingStatus.value = '📋 일정 생성 중...'
    }
  }, 500)

  try {
    const response = await generateTripPlan(formData.value)
    clearInterval(progressInterval)
    loadingProgress.value = 100
    loadingStatus.value = '✅ 완료!'
    router.push({ name: 'result', state: { tripPlan: response } })
  } catch (error) {
    clearInterval(progressInterval)
    message.error('계획 생성에 실패했습니다.')
  } finally {
    loading.value = false
  }
}
```

### 13.6.3 일정 편집 기능

AI가 작성한 여행 계획이 훌륭하더라도 사용자의 개인적 요구를 완벽히 충족하지는 못할 수 있습니다. 예를 들어, 특정 관광지가 마음에 들지 않아 빼고 싶거나 방문 순서를 바꾸고 싶을 수 있습니다. 우리는 사용자가 여정을 원하는 대로 꾸밀 수 있도록 편집 기능을 제공합니다.

편집 기능의 핵심은 **상태 관리(state management)**입니다. 우리는 현재의 여행 계획과 원본 여행 계획이라는 두 가지 상태를 유지해야 합니다. 사용자가 편집 모드로 진입하면 원본 데이터의 복사본을 만들어 보관합니다. 편집을 취소하면 원본 상태로 돌려놓고, 변경사항을 저장하면 현재 데이터를 업데이트합니다:

```typescript
const editMode = ref(false)
const originalPlan = ref<TripPlan | null>(null)

// 편집 모드 전환
const toggleEditMode = () => {
  editMode.value = true
  originalPlan.value = JSON.parse(JSON.stringify(tripPlan.value))
}
```

객체를 깊은 복사(Deep Copy)하기 위해 `JSON.parse(JSON.stringify(...))`를 사용했다는 점을 유념해 주십시오. 왜 그냥 대입하지 않을까요? JavaScript에서 객체는 참조(reference) 타입이므로 그냥 대입하면 `originalPlan`과 `tripPlan`이 동일한 객체를 가리키게 되어 한쪽을 수정하면 다른 한쪽도 변해 버립니다. 깊은 복사를 해야만 완벽하게 분리된 복사본이 만들어집니다.

관광지를 이동하는 로직은 배열 내부에서 두 원소의 위치를 스왑하는 것입니다:

```typescript
// 관광지 이동
const moveAttraction = (dayIndex: number, attractionIndex: number, direction: 'up' | 'down') => {
  const attractions = tripPlan.value.days[dayIndex].attractions
  const newIndex = direction === 'up' ? attractionIndex - 1 : attractionIndex + 1

  if (newIndex >= 0 && newIndex < attractions.length) {
    [attractions[attractionIndex], attractions[newIndex]] =
    [attractions[newIndex], attractions[attractionIndex]]
  }
}
```

임시 변수 없이도 배열 안의 원소를 우아하게 스왑할 수 있는 ES6의 구조 분해 할당(destructuring assignment) 문법을 사용했습니다.

관광지 삭제는 배열의 `splice` 메서드를 사용합니다:

```typescript
// 관광지 삭제
const deleteAttraction = (dayIndex: number, attractionIndex: number) => {
  tripPlan.value.days[dayIndex].attractions.splice(attractionIndex, 1)
}
```

변경사항을 최종 저장할 때는 관광지 위치가 변했을 수 있으므로 지도를 다시 초기화해야 합니다:

```typescript
// 변경사항 저장
const saveChanges = () => {
  editMode.value = false
  message.success('변경사항이 저장되었습니다.')
  initMap()  // 지도 재초기화
}

// 편집 취소
const cancelEdit = () => {
  if (originalPlan.value) {
    tripPlan.value = originalPlan.value
  }
  editMode.value = false
}
```

템플릿에서는 `editMode` 값에 의존하여 다른 UI를 노출합니다. 편집 모드일 때 각 관광지 옆에 위로 이동, 아래로 이동, 삭제 버튼이 노출됩니다:

```vue
<div v-if="editMode" class="edit-buttons">
  <a-button size="small" @click="moveAttraction(dayIndex, index, 'up')">위로</a-button>
  <a-button size="small" @click="moveAttraction(dayIndex, index, 'down')">아래로</a-button>
  <a-button size="small" danger @click="deleteAttraction(dayIndex, index)">삭제</a-button>
</div>
```

### 13.6.4 내보내기 기능

만족스러운 여행 계획을 완성한 후에 사용자는 이를 소장하거나 친구들과 나누고 싶을 것입니다. 우리는 이미지 내보내기와 PDF 내보내기라는 두 가지 공유 방식을 제공합니다.

내보내기 기능의 근간은 `html2canvas` 라이브러리입니다. 이 도구는 DOM 노드를 Canvas 객체로 변환하여 이미지 파일로 출력해 줍니다. 다만 여기에 기술적 난제가 하나 있는데, 지도가 Canvas로 그려져 있고 `html2canvas`가 중첩된 Canvas를 처리할 때 호환성 문제를 일으킨다는 점입니다.

지도 Canvas를 출력 전에 미리 이미지 데이터로 파싱해 두는 등의 여러 방식을 검토했으나, 고덕 지도의 내부 렌더링 매커니즘과 크로스 오리진(CORS) 규제 등으로 인해 문제를 완전히 해소하기는 어려웠습니다. 실제 제품을 개발할 때는 아래와 같은 대체안들을 고려해 볼 수 있습니다:

1. **고덕 지도의 정적 지도 API 활용**: dynamic map 대신 `maps_staticmap` 도구 등을 활용하여 정적 지도 이미지로 대체
2. **분리된 내보내기**: 지도 부분과 텍스트 일정을 따로 내보낸 후 백엔드 서버에서 병합 처리
3. **스크린샷 서비스 연동**: 백엔드에서 Puppeteer 같은 헤드리스 브라우저를 구동하여 캡처를 전담 수행
4. **출력 내용 간소화**: 내보낼 때 지도 영역을 일시 숨김 처리하고 여정 텍스트와 세부 안내 정보 위주로만 구성

현재 구현 단계에서는 내보낼 때 지도 영역을 임시로 숨기고 여정 본문과 관광지 정보 텍스트만 다운로드하는 간소화된 우회책을 채택했습니다. 이상적인 완성본은 아닐지라도 기능 자체를 정상 작동하게 보장해 줍니다.

이미지 내보내기 로직은 심플합니다:

```typescript
import html2canvas from 'html2canvas'

const exportAsImage = async () => {
  const element = document.getElementById('trip-plan-content')
  if (!element) return

  const canvas = await html2canvas(element, {
    backgroundColor: '#ffffff',
    scale: 2,
    useCORS: true
  })

  const link = document.createElement('a')
  link.download = `${tripPlan.value.city} 여행 계획.png`
  link.href = canvas.toDataURL('image/png')
  link.click()
  message.success('내보내기에 성공했습니다!')
}
```

`scale: 2`는 해상도를 2배 높여 출력본의 선명함을 확보해 줍니다. `useCORS: true`는 Unsplash 등 외부에서 동적으로 끌어온 이미지 리소스들의 CORS 로드를 보장해 줍니다.

PDF 내보내기는 몇 가지 절차가 더 필요합니다: Canvas를 먼저 뜨고 이미지 데이터로 바꾼 다음 PDF 문서 객체에 넣어 줍니다:

```typescript
import jsPDF from 'jspdf'

const exportAsPDF = async () => {
  // 먼저 지도 이미지 캡처
  await captureMapImage()

  const element = document.getElementById('trip-plan-content')
  if (!element) return

  const canvas = await html2canvas(element, {
    backgroundColor: '#ffffff',
    scale: 2,
    useCORS: true,
    allowTaint: true
  })

  // 지도 복원
  restoreMap()

  const pdf = new jsPDF('p', 'mm', 'a4')
  const imgData = canvas.toDataURL('image/png')
  const imgWidth = 210  // A4 가로폭 (mm)
  const imgHeight = (canvas.height * imgWidth) / canvas.width

  pdf.addImage(imgData, 'PNG', 0, 0, imgWidth, imgHeight)
  pdf.save(`${tripPlan.value.city} 여행 계획.pdf`)
  message.success('내보내기에 성공했습니다!')
}
```

Canvas 종횡비를 일정하게 유지하기 위해 이미지 세로 길이를 수학적으로 역산하여 적용해 주는 부분입니다. A4 종이 규격의 가로폭은 210mm이므로 이에 비례하는 세로폭을 계산합니다.

### 13.6.5 측면 내비게이션 및 앵커 이동

결과 화면에는 여정 요약, 예산, 지도, 일별 일정, 일기 예보 등 방대한 항목이 일렬로 나열되어 있습니다. 방문자가 원하는 영역으로 빠르게 이동하려 할 때 스크롤을 길게 내려야 하는 불편이 따릅니다. 스크롤 동선 단축을 위해 앵커 점프가 연동된 측면 플로팅 메뉴바를 구축했습니다.

측면 메뉴바는 Ant Design Vue의 Menu 컴포넌트를 이용합니다:

```vue
<a-menu
  v-model:selectedKeys="[activeSection]"
  mode="inline"
  @click="scrollToSection"
>
  <a-menu-item key="overview">📋 일정 개요</a-menu-item>
  <a-menu-item key="budget">💰 예산 상세 정보</a-menu-item>
  <a-menu-item key="map">🗺️ 지도 보기</a-menu-item>
  <a-menu-item key="days">📅 일별 일정</a-menu-item>
  <a-menu-item key="weather">🌤️ 날씨 예보</a-menu-item>
</a-menu>
```

메뉴 아이템을 선택(클릭)하면 `scrollToSection` 핸들러가 동작합니다:

```typescript
const activeSection = ref('overview')

// 목표 영역으로 부드럽게 스크롤
const scrollToSection = ({ key }: { key: string }) => {
  activeSection.value = key
  const element = document.getElementById(key)
  if (element) {
    element.scrollIntoView({ behavior: 'smooth', block: 'start' })
  }
}
```

`scrollIntoView`는 타깃 노드를 화면 중심이나 시작점으로 끌어다 주는 브라우저 고유 API입니다. `behavior: 'smooth'` 옵션을 통해 시점이 뚝 끊기지 않고 부드럽게 이동하게 만들어 줍니다. `block: 'start'`는 노드의 윗부분이 뷰포트 맨 위에 딱 맞게 스크롤되도록 잡아줍니다.

각 영역의 최상단 HTML 마크업에 매칭되는 ID들을 빠짐없이 심어둡니다:

```vue
<div id="overview">
  <!-- 일정 개요 영역 -->
</div>

<div id="budget">
  <!-- 예산 세부 영역 -->
</div>

<div id="map">
  <!-- 지도 노출 영역 -->
</div>
```

이 연계를 통해 메뉴 바를 터치하는 것만으로 스크롤이 매끄럽게 슬라이드하며 해당하는 정보 영역으로 방문자를 모십니다.

이러한 인터랙티브 기능군이 모여 지능형 여행 비서는 단순히 AI 텍스트만 뱉어내는 정적인 화면에서 탈피해, 실질적 비용을 가늠하는 예산 통계기, 막막한 대기를 달래는 진행바, 나만의 루트를 다듬는 편집기, 소장 가치를 극대화하는 내보내기, 긴 화면을 가뿐히 넘나드는 앵커바까지 아우르는, 실용적이고 종합적인 full-stack 웹 애플리케이션으로 성숙할 수 있었습니다.

## 13.7 결론

13장을 완주하신 것을 축하드립니다!

이 장을 통해 여러분은 단순히 완성형 지능형 여행 비서 애플리케이션을 빌드한 것을 넘어, 가장 핵심적인 역량들을 손에 쥐게 되었습니다:

1. **시스템 디자인 싱킹**: 거대하고 뒤엉킨 현실의 난제를 간결하고 명확한 미션 단위로 쪼개는 생각의 힘
2. **엔지니어링 실행력**: 머릿속 아키텍처 다이어그램을 실제 가동되는 코드로 변환해 내는 구현 역량
3. **풀스택 아우르기**: 프론트엔드의 화면 단과 백엔드의 연산 단을 톱니바퀴처럼 조립해 내는 연동 지식
4. **실무형 AI 응용 개발**: LLM을 장난감이 아닌, 일상 문제를 진짜로 해결해 주는 든든한 도구로 활용하는 안목

이번 여정은 마침표가 아닌, 새로운 출발선입니다. 이 뼈대를 토대로 여러분은 얼마든지 뻗어 나갈 수 있습니다:

- 확장 기능 추가
- 사용자 인터랙션 디테일 보완
- 타 도메인으로의 확장 (예: 스마트 쇼핑 도우미, 맞춤형 개인 학습 비서 등)
- 실제 서비스 배포를 통한 대고객 운영

배움의 가장 확실한 촉매는 몸소 겪는 삽질입니다. 예제 코드를 눈으로만 훑지 말고, 직접 부딪히고 깨지며 코드를 마음껏 확장해 보시기 바랍니다. 그러한 시도 하나하나가 다중 에이전트 시스템을 여러분의 완전한 무기로 만들어 줄 것입니다.

차세대 AI 애플리케이션 개발자로서 내딛는 여러분의 앞날을 진심으로 응원합니다!

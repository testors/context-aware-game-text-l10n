# 프로젝트 명세서: Context-Aware Recursive AI Translation System (v3.1)

## 1. 배경 및 핵심 혁신 (Background & Innovation)

### 1.1 배경: 기존 입력 방식의 한계 (AS-IS)
현재의 게임 텍스트 입력 방식은 **'단순 텍스트(Plain Text)'** 입력에 의존하고 있다.
* **방식:** 기획자가 *"전사 알렉스가 파멸의 검을 휘둘렀다."*라고 문장 전체를 문자열로 입력.
* **문제점:** AI 번역기는 텍스트만 보고 '파멸의 검'이 고유명사 아이템인지, 일반 명사인지 구분하지 못함. 이로 인해 용어 불일치와 치명적인 오번역(Hallucination)이 발생함.

### 1.2 혁신: 구조화된 태깅 시스템 도입 (TO-BE)
우리는 이번 프로젝트를 통해 **최초로 'ID 태깅(Tagging)' 시스템을 도입**한다.
* **개념:** 텍스트 내의 주요 고유명사를 텍스트가 아닌 **'불변의 ID 객체({ITEM_99})'**로 치환하여 저장한다.
* **효과:** 언어가 바뀌어도 ID는 변하지 않으므로, 데이터 무결성과 번역 일관성을 100% 보장한다.

### 1.3 해결 과제: 입력 비용의 최소화 (Smart Automation)
태깅 시스템은 번역 품질을 보장하지만, 기획자가 글을 쓸 때마다 `{NPC_05}` 같은 코드를 직접 찾아 입력해야 한다면 작업 효율이 급격히 떨어진다.
따라서 우리는 **'스마트 에디터(Smart Editor)'**를 통해 이 과정을 자동화하여, **"새로운 시스템을 도입하지만, 작업자의 입력 경험은 기존과 동일하게 유지"**하는 것을 목표로 한다.

---

## 2. 시스템 워크플로우 (Workflow)

### 2.1 지능형 입력 프로세스 (Smart Data Entry)
기획자는 복잡한 ID를 알 필요가 없다. 시스템이 백그라운드에서 텍스트를 데이터로 변환한다.

1.  **자연어 입력:** 기획자가 에디터에 한글로 *"전사 알렉스가 파멸의 검을 찾고 있다."*라고 입력한다.
2.  **실시간 감지 및 자동 태깅 (Auto-Tagging):**
    * 에디터는 입력된 문장에서 DB에 등록된 키워드('전사 알렉스', '파멸의 검')를 감지한다.
    * 사용자의 엔터(Enter) 입력 시점에, 해당 단어들을 내부적으로 **`{NPC_05}`**, **`{ITEM_99}`**로 자동 치환하여 저장한다.
    * *(사용자 화면에는 여전히 가독성을 위해 한글 원문이 하이라이트 되어 표시된다.)*
3.  **모호성 해결 (Disambiguation):**
    * 만약 '배'를 입력했는데 이것이 '선박(Ship)'인지 '과일(Pear)'인지 모호할 경우, 에디터는 팝업을 띄워 기획자가 문맥에 맞는 카테고리를 선택하게 한다.

### 2.2 번역 파이프라인 (Batch Translation Pipeline)
효율성을 위해 단일 문장이 아닌 **배치(Batch)** 단위로 번역을 수행한다.

1.  **동적 글로서리 생성 (On-Demand Glossary Construction):**
    * 번역 요청된 **여러 문장**에 포함된 모든 태그(Tag)를 수집한다.
    * 해당 태그들의 번역어 정보를 DB에서 조회하여, **이번 배치 요청에만 사용할 전용 글로서리(JSON)**를 생성한다.
    * *별도의 엑셀 용어집 파일은 존재하지 않으며 관리할 필요도 없다.*
2.  **AI 번역 요청:**
    * AI에게는 **'문장 배열(Array)'**과 **'동적 글로서리'**가 함께 전달된다.
    * AI는 글로서리를 참조하여 다수의 문장을 동시에 번역하며, 문맥의 일관성을 유지한다.

---

## 3. 데이터 구조 및 프롬프트 설계

### 3.1 요청 JSON 스키마 (Request Payload)
개발팀은 아래 구조를 참조하여, 사용자가 입력한 자연어에서 태그를 추출해 `global_glossary`를 **자동 생성**하는 로직을 구현해야 한다. `items` 배열을 통해 여러 문장을 한 번에 처리하는 구조임에 유의한다.

```json
{
  "system_instruction": "Translate the 'items' array based on the provided 'global_glossary'. The glossary is generated in real-time specifically for this batch. Strictly follow the target terms in the glossary.",
  
  "request_payload": {
    "source_lang": "ko",
    "target_lang": "en",
    
    // [System Generated] 배치 내 모든 문장에서 추출한 태그의 합집합 (중복 제거됨)
    "global_glossary": {
      "{NPC_05}": {
        "src": "전사 알렉스",
        "tgt": "Warrior Alex",
        "cat": "Character_Name" 
      },
      "{ITEM_99}": {
        "src": "파멸의 검",
        "tgt": "Blade of Ruin",
        "cat": "Weapon_Name"
      },
      "{SKILL_01}": {
        "src": "화염구",
        "tgt": "Fireball",
        "cat": "Skill_Name"
      }
    },

    // [Batch Items] 한 번의 API 호출로 처리할 문장 목록
    "items": [
      {
        "id": "SEQ_001",
        "category": "Quest_Description",
        "source_text": "전사 알렉스가 잃어버린 파멸의 검을 찾고 있다.", 
        "tag_text": "{NPC_05}가 잃어버린 {ITEM_99}를 찾고 있다."
      },
      {
        "id": "SEQ_002",
        "category": "System_Message",
        "source_text": "파멸의 검을 장착했습니다.",
        "tag_text": "{ITEM_99}을 장착했습니다." // 위 문장과 동일한 ITEM_99 태그 재사용
      },
      {
        "id": "SEQ_003",
        "category": "Dialogue",
        "source_text": "받아라! 이것이 나의 화염구다!",
        "tag_text": "받아라! 이것이 나의 {SKILL_01}다!"
      }
    ]
  }
}

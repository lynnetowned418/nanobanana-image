---
name: nanobanana-image
description: This skill should be used when the user asks to "이미지 만들어줘", "그림 생성해줘", "이미지 그려줘", "썸네일 만들어", "로고 만들어줘", "이미지 수정해줘", "그림 바꿔줘", "사진 편집해줘", "아까 만든 거 수정", "색감 바꿔줘", "배경 바꿔줘", "좀 더 밝게", "스타일 바꿔줘", "다시 그려줘", "재생성해줘", "image generate", "create image", "edit image", "modify image", "refine image". Make sure to use this skill whenever the user mentions image generation, editing, or modification, even if they use casual expressions like "그림 하나 뽑아줘", "이미지 좀 만들어봐", "아까 거 고쳐줘", or "다른 느낌으로 해줘".
---

# NanoBanana Image Generator

> 자연어로 이미지를 생성하고, 편집하고, 반복 수정하는 스킬. image_studio 시스템 프롬프트로 최적의 프롬프트를 만들고, google-genai SDK로 NanoBanana API를 호출한다.

---

## 사전 조건

`.env` 파일이 필요하다. 없으면 아래 안내를 출력하고 중단한다:

```
.env 파일이 없어요. 아래 단계를 따라주세요:

1. .env.example 파일을 복사: cp .env.example .env
2. NanoBanana API 키를 발급받아 NANOBANANA_API_KEY에 입력
3. 원하는 모델명을 NANOBANANA_MODEL에 입력 (기본: gemini-3-pro-image-preview)
```

필수 의존성: `google-genai` (≥1.56.0), `Pillow` (≥10.0.0). 미설치 시 스크립트가 안내 메시지를 출력한다.

---

## 워크플로우

### Step 0: 모드 자동 감지
**타입**: prompt (Claude 판단)

사용자 입력을 분석하여 3가지 모드 중 하나를 결정한다:

| 모드 | 조건 | 키워드 |
|------|------|--------|
| `MODE_NEW` | 새 이미지 생성 요청 (기본값) | "만들어줘", "생성해줘", "그려줘" |
| `MODE_EDIT` | 기존 이미지 파일 경로 포함 + 변형 요청 | "이 이미지 수정해줘", "[경로] 편집해줘" |
| `MODE_REFINE` | 이번 대화에서 이전 생성 이력 + 수정 요청 | "아까 거 바꿔줘", "색감 변경", "좀 더 밝게" |

감지 우선순위: MODE_EDIT (파일 경로 존재) > MODE_REFINE (이전 생성 이력 + 수정 동사) > MODE_NEW (기본)

수정 감지 키워드: "수정", "바꿔", "변경", "고쳐", "다시", "재생성", "좀 더", "덜", "밝게", "어둡게", "아까 거", "방금 거"

### Step 0.5: 설정 확인 (MODE_NEW 전용, 조건부)
**타입**: ask (조건부 실행)

사용자 프롬프트에서 모델/비율 힌트를 추출한다. 모든 힌트가 있거나 편집/멀티턴 모드면 이 단계를 스킵한다.

자동 매핑 규칙:

| 키워드 | 매핑 | 스킵 대상 |
|--------|------|-----------|
| "빨리", "시안", "초안", "대충" | Flash + 512 + minimal | 모델 질문 |
| "고품질", "4K", "정교하게", "세밀하게" | Pro + 4K + high | 모델 질문 |
| "세로", "포스터", "인스타" | 3:4 | 비율 질문 |
| "가로", "썸네일", "유튜브", "프레젠테이션" | 16:9 | 비율 질문 |
| "정사각형", "프로필", "아이콘", "로고" | 1:1 | 비율 질문 |
| "배너", "헤더" | 4:1 | 비율 질문 |

미확정 파라미터가 있으면 AskUserQuestion 1회 호출 (최대 2개 질문 동시):

```json
{
  "questions": [
    {
      "question": "어떤 스타일로 만들까요?",
      "header": "스타일",
      "options": [
        {"label": "추천대로 할게요", "description": "고품질 + 정사각형(1:1) — 가장 범용적인 설정"},
        {"label": "빠르게 시안만", "description": "초안 확인용 — 빠르지만 해상도 낮음 (5~10초)"},
        {"label": "최고 품질로", "description": "정교한 결과물 — 시간이 좀 더 걸림 (20~40초)"}
      ],
      "multiSelect": false
    },
    {
      "question": "이미지 비율은요?",
      "header": "비율",
      "options": [
        {"label": "정사각형 (1:1)", "description": "프로필, SNS 게시물, 아이콘"},
        {"label": "가로형 (16:9)", "description": "유튜브 썸네일, 프레젠테이션, 배경화면"},
        {"label": "세로형 (3:4)", "description": "인스타 피드, 포스터, 모바일"},
        {"label": "와이드 배너 (4:1)", "description": "웹 헤더, 이메일 배너"}
      ],
      "multiSelect": false
    }
  ]
}
```

파라미터 매핑:

| 선택 | 모델 | 해상도 | thinking |
|------|------|--------|----------|
| 추천대로 | gemini-3-pro-image-preview | 2K | high |
| 빠르게 시안만 | gemini-3.1-flash-image-preview | 512 | minimal |
| 최고 품질로 | gemini-3-pro-image-preview | 4K | high |

### Step 1: 이미지 프롬프트 생성
**타입**: rag + prompt

`references/image-studio-prompt.md` 파일을 Read 도구로 로드한다. 이 시스템 프롬프트의 지침을 내면화하여 사용자의 이미지 요청을 분석하고, 7가지 모드(인물/풍경/오브젝트/일러스트/썸네일/로고/컨셉트) 중 최적 모드를 자동 선택한 뒤, 영문 프롬프트를 생성한다.

모드별 프롬프트 차이:
- **MODE_NEW**: 사용자 요청 → 모드 선택 → 영문 프롬프트 생성 (200-500 단어)
- **MODE_EDIT**: "원본 이미지를 기반으로 [변경사항]을 적용하라" 형태의 영문 프롬프트
- **MODE_REFINE**: 이전 프롬프트 기반으로 수정 사항만 반영한 delta 프롬프트

### Step 1-E: 원본 이미지 검증 (MODE_EDIT 전용)
**타입**: prompt

MODE_EDIT인 경우, 사용자가 지정한 이미지 파일이 존재하는지 확인한다. 스크립트가 자동으로 포맷/크기/해상도를 검증한다.

### Step 2: SDK로 이미지 생성
**타입**: script

Bash 도구로 Python 스크립트를 실행한다:

**MODE_NEW**:
```bash
python3 "${SKILL_DIR}/scripts/generate_image.py" \
  --mode generate \
  --prompt "영문 프롬프트" \
  --output "저장 경로" \
  --aspect-ratio "비율" \
  --image-size "크기" \
  --thinking-level "수준" \
  --env-file "${SKILL_DIR}/.env"
```

**파라미터 유효값 (반드시 이 값만 사용):**
- `--aspect-ratio`: `1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `4:1`, `1:4`
- `--image-size`: `512`, `1K`, `2K`, `4K` (숫자 단독 사용 금지 — '2048' 아닌 '2K', '4096' 아닌 '4K')
- `--thinking-level`: `minimal`, `high`

**MODE_EDIT**:
```bash
python3 "${SKILL_DIR}/scripts/generate_image.py" \
  --mode edit \
  --prompt "편집 프롬프트" \
  --image "원본 경로" \
  --output "저장 경로" \
  --env-file "${SKILL_DIR}/.env"
```

**MODE_REFINE**:
```bash
python3 "${SKILL_DIR}/scripts/generate_image.py" \
  --mode chat \
  --prompt "수정 프롬프트" \
  --session-id "이전 세션 ID" \
  --output "저장 경로" \
  --env-file "${SKILL_DIR}/.env"
```

`SKILL_DIR`은 이 SKILL.md가 위치한 디렉토리의 절대 경로다.

### Step 3: 결과 반환
**타입**: generate

성공 시:
- 저장된 이미지 파일 경로
- 선택된 모드 (인물/풍경/로고 등)
- 사용된 프롬프트 요약 (한국어 2-3줄)
- session_id (내부 기억용, 사용자에게 노출하지 않음)
- "수정하고 싶으면 말해주세요" 안내

실패 시 에러 메시지와 해결 방법 안내.

### Step 4: 멀티턴 루프 대기

Step 3 완료 후, 대화 컨텍스트에 아래 정보를 유지한다:
- 마지막 생성 이미지 경로
- 마지막 사용 프롬프트
- session_id
- 선택된 설정 (모델, 비율 등)

사용자가 수정 요청 시 Step 0으로 자동 복귀 (MODE_REFINE).
세션 만료 시 새 세션으로 자동 전환하고 사용자에게 안내.

---

## References
- **`references/image-studio-prompt.md`** — 이미지 프롬프트 생성용 시스템 프롬프트 (image_studio v3.1). 7가지 모드별 워크플로우와 출력 템플릿 포함.

## Scripts
- **`scripts/generate_image.py`** — google-genai SDK 기반 이미지 생성 스크립트. 3가지 모드(generate/edit/chat) 지원. 멀티턴 세션 관리, 이미지 검증, SDK 에러 타입별 처리 포함.

## Settings

| 설정 | 기본값 | 변경 방법 |
|------|--------|-----------|
| API 키 | (없음, 필수) | `.env`의 `NANOBANANA_API_KEY` |
| 모델 | `gemini-3-pro-image-preview` | `.env`의 `NANOBANANA_MODEL` |
| 기본 비율 | (없음) | `.env`의 `NANOBANANA_ASPECT_RATIO` |
| 기본 크기 | (없음) | `.env`의 `NANOBANANA_IMAGE_SIZE` |
| 사고 수준 | (없음) | `.env`의 `NANOBANANA_THINKING_LEVEL` |
| 타임아웃 | 120초 | `.env`의 `NANOBANANA_TIMEOUT` |

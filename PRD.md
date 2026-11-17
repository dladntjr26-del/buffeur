# PRD: 사진 → SNS 포스팅 초안 생성기

## 1. 개요
사진 한 장을 올리면 AI가 사진의 특성을 분석하고, 그 특성으로 인스타그램과 스레드 포스팅 초안을 각각 작성해 보여 준다.

## 2. 목표
- 사진 등록부터 초안 출력까지 한 화면에서 끝낸다.
- 사진 분석과 글 작성을 분리해, 각 단계에 맞는 모델을 쓴다.
- API 키는 서버(Supabase Edge Function)에만 둔다.
- **AI는 무료 모델만 사용한다.** OpenRouter는 `:free` 모델만, Gemini는 무료 한도(free tier)가 있는 모델만 쓴다. 유료 모델로는 폴백하지 않는다.

### 비목표
- SNS 자동 게시, 로그인/회원 기능, 초안 저장 이력(1차 범위 제외)

## 3. 사용자 흐름
1. 사용자가 사진을 선택하거나 끌어다 놓는다.
2. 미리보기를 확인하고 "초안 만들기"를 누른다.
3. 로딩 표시 후 사진 특성 요약과 인스타·스레드 초안이 나온다.
4. 각 초안을 복사하거나 "다시 생성"을 누른다.

## 4. 기능 요구사항
### F1. 사진 등록
- JPEG, PNG, WEBP, GIF 지원. 한 장씩.
- 최대 크기: 원본 10MB 이하. 전송 전 브라우저에서 긴 변 1600px 이하로 줄여 base64로 변환한다.
- 형식/크기 위반 시 한국어 안내 메시지.

### F2. 사진 특성 추출 (OpenRouter)
- 모델: `particle-ai/particle-1:free` 우선, 실패(오류·타임아웃) 시 `nvidia/nemotron-3-nano-omni-30b-a3b-reasoning:free`로 대체.
- 요청: 이미지(base64 data URL) + "SNS 글쓰기에 쓸 특성을 JSON으로 추출" 프롬프트.
- 출력 JSON 항목:
  - `subject` 주요 피사체
  - `setting` 장소/배경
  - `mood` 분위기
  - `colors` 주요 색상 목록
  - `time_weather` 시간대/날씨 (알 수 있을 때)
  - `details` 눈에 띄는 디테일
  - `suggested_keywords` 해시태그용 키워드
- JSON 파싱 실패 시 텍스트 그대로 다음 단계에 넘기는 대체 처리.
- reasoning 모델 사용 시 `max_tokens` 1500 이상.

### F3. 포스팅 초안 작성 (Gemini)
- 모델: `gemini-3.5-flash-lite` (모델 목록으로 확인함).
- 입력: F2의 특성 JSON (이미지는 다시 보내지 않는다).
- 출력 두 가지:
  - **인스타그램**: 첫 줄 훅, 본문 3~5줄, 이모지 적당히, 해시태그 5~10개
  - **스레드**: 500자 이내, 대화체, 해시태그 0~2개
- 사진에 없는 사실(장소 이름, 인물 등)은 지어내지 않는다.
- 사용자가 선택 입력할 수 있는 옵션: 톤(친근/감성/유머/정보), 한마디 메모.

### F4. 결과 출력
- 사진 특성 요약(접이식), 인스타 초안, 스레드 초안을 카드로 표시.
- 각 카드에 "복사" 버튼, 전체에 "다시 생성" 버튼.
- 스레드 글자 수 표시.

## 5. 아키텍처
```
브라우저 (HTML/CSS/JS)
   │  POST /functions/v1/<함수명>  { image: dataURL, tone?, memo? }
   ▼
Supabase Edge Function
   ├─ 1) OpenRouter /chat/completions  (OPEN_ROUTER_KEY)  → 특성 JSON
   └─ 2) Gemini generateContent        (GEMINI_API_KEY)   → 초안 2종
   │
   ▼ { features, instagram, threads }
```
- 함수는 1~2개로 구성: 한 함수에서 두 단계를 순서대로 처리하거나, `analyze` / `draft`로 분리해도 된다.
- 시크릿: `OPEN_ROUTER_KEY`, `GEMINI_API_KEY` (Supabase Edge Function Secrets). 프론트에는 노출하지 않는다.

## 6. API 명세 (안)
`POST /functions/v1/photo-to-post`

요청
```json
{ "image": "data:image/jpeg;base64,...", "tone": "친근", "memo": "주말 산책" }
```
응답 200
```json
{
  "features": { "subject": "...", "setting": "...", "mood": "...", "colors": ["..."], "suggested_keywords": ["..."] },
  "instagram": "...",
  "threads": "..."
}
```
오류: 400(입력 오류), 405(POST 외), 502(외부 API 실패), 500(시크릿 없음). 본문은 `{ "error": "한국어 메시지" }`.

## 7. 비기능 요구사항
- 응답 시간: 보통 30초 이내. 프론트에 진행 상태 표시, 함수 내부 타임아웃 설정.
- 보안: Edge Function에는 인증(JWT)·호출 횟수 제한 등 접근 보안을 적용하지 않는다. 누구나 호출할 수 있다. API 키는 시크릿에 두고 응답에 포함하지 않는다.
- 비용: 무료 모델만 사용(OpenRouter `:free`, Gemini free tier). 코드의 허용 목록에 없는 모델과 `:free`가 아닌 OpenRouter ID는 호출 전에 거부한다. 무료 한도 초과(429) 시 유료로 넘어가지 않고 안내 메시지를 보여 준다.
- 접근성/반응형: 모바일 화면에서 사용 가능한 레이아웃.

## 8. 위험과 대응
| 위험 | 대응 |
|---|---|
| 무료 모델의 속도 제한·일시 장애 | 대체 모델로 재시도, 오류 메시지 안내 |
| reasoning 모델이 토큰을 소진해 답변이 끊김 | `max_tokens` 상향, 비어 있으면 대체 모델 |
| Gemini 무료 한도 여부 미확인 | 429 시 안내 메시지. 한도가 없으면 free tier 모델로 교체 |
| 공개 호출로 무료 한도 소진 | 보안은 적용하지 않기로 결정. 한도 초과(429) 시 안내 메시지만 표시 |
| 모델이 사진에 없는 내용을 지어냄 | 프롬프트에 "보이는 것만" 명시 |

## 9. 완료 기준
- [ ] 사진을 올리면 특성 JSON이 만들어진다.
- [ ] 같은 특성으로 인스타·스레드 초안이 각각 나온다.
- [ ] 키가 브라우저 네트워크 응답·소스 어디에도 보이지 않는다.
- [ ] 잘못된 파일/큰 파일/모델 오류에서 한국어 오류 메시지가 나온다.
- [ ] 모바일과 데스크톱에서 모두 사용 가능하다.
- [ ] 호출되는 모든 AI 모델이 무료 모델이다. (`:free` ID 또는 Gemini free tier)

## 10. 구현 순서
1. Gemini 모델 ID 확인, 시크릿 이름(`OPEN_ROUTER_KEY`, `GEMINI_API_KEY`) 확인
2. Edge Function 작성·배포 (특성 추출 → 초안 생성)
3. HTML/CSS/JS 화면 (업로드, 미리보기, 결과 카드)
4. 오류 처리·대체 모델 점검, 임시 함수 `tmp-openrouter-test` 삭제

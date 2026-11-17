# CLAUDE.md

사진을 올리면 SNS(인스타그램·스레드) 포스팅 초안을 만들어 주는 웹앱. 자세한 요구사항은 `PRD.md` 참고.

## 기술 스택
- 프론트엔드: 순수 HTML, CSS, JavaScript (빌드 도구·프레임워크 없음)
- 백엔드: Supabase Edge Function (Deno / TypeScript)
- 외부 API: OpenRouter(이미지 특성 추출), Google Gemini(초안 작성)

## 시크릿 (Supabase Edge Function Secrets)
| 이름 | 용도 |
|---|---|
| `OPEN_ROUTER_KEY` | OpenRouter 호출 |
| `GEMINI_API_KEY` | Gemini 호출 |

- 키는 Edge Function 안에서만 `Deno.env.get()`으로 읽는다. 프론트엔드 코드, 로그, 응답, 커밋에 절대 넣지 않는다.
- 시크릿 값에 공백·개행이 섞이면 헤더 오류가 나므로 읽은 뒤 `.trim()` 한다. (과거에 한글/공백이 섞여 저장된 적이 있다.)
- 실제 등록된 이름(2026-11-17 확인): `OPEN_ROUTER_KEY`(대문자), `gemini_api_key`(**소문자**). 함수는 `GEMINI_API_KEY`와 `gemini_api_key`, `OPEN_ROUTER_KEY`와 `open_ROUTER_KEY`를 모두 읽는다.

## 처리 흐름
1. 프론트가 이미지를 base64 data URL로 Edge Function에 전송
2. Edge Function이 OpenRouter 비전 모델로 사진 특성(JSON) 추출
3. 추출된 특성을 Gemini에 넘겨 인스타·스레드 초안 생성
4. 프론트가 두 초안을 출력

## 모델
- 이미지 분석: `particle-ai/particle-1:free` (기본), 실패 시 `nvidia/nemotron-3-nano-omni-30b-a3b-reasoning:free`
- 글 작성: `gemini-3.5-flash-lite` (모델 목록 API로 존재 확인, 2026-11-17). 모델 ID는 `GEMINI_MODEL` 상수 한 곳에 둔다. 무료 한도 적용 여부는 아직 미확인.
- **무료 모델만 사용한다.** OpenRouter는 ID가 `:free`로 끝나는 모델만 쓴다. 유료 ID(`particle-ai/particle-1` 등 `:free` 없는 것)로 폴백하지 않는다. Gemini도 무료 한도(free tier)가 있는 모델만 쓴다.
- 코드에 허용 모델 목록(allowlist)을 두고, `:free`가 아닌 OpenRouter 모델 ID는 호출 전에 거부한다.

## 테스트로 확인된 사실 (2026-11-17)
- OpenRouter 이미지 입력은 `messages[].content`에 `{type:"image_url", image_url:{url}}`로 보낸다. 이미지 URL보다 **base64 data URL이 안정적**이다.
- Particle은 이미지 1장당 16 MiB 이하, PNG/JPEG/WEBP/GIF만 허용한다.
- Nemotron nano omni는 reasoning 모델이라 `max_tokens`가 작으면(500) 답변이 끊긴다. 1500 이상으로 잡는다.
- `nvidia/nemotron-3-nano-30b-a3b`는 텍스트 전용이라 이미지 분석에 쓸 수 없다.
- 일부 응답은 앞에 공백/빈 줄이 붙는다. JSON 파싱 전에 `trim()` 한다.

## 코드 규칙
- 동작에 필요한 최소 입력 확인(이미지 형식, JSON 본문)만 하고 한국어 오류 메시지를 반환한다. 접근 보안 목적의 검증은 하지 않는다.
- CORS 헤더를 응답과 `OPTIONS`에 모두 둔다. (`ask-gemini` 함수 패턴을 따른다.)
- 외부 API 오류는 서버 로그에만 자세히 남기고, 클라이언트에는 일반화된 메시지만 준다.
- 사용자 화면 문구는 한국어.
- 프론트는 `innerHTML`에 모델 응답을 그대로 넣지 않는다. `textContent`를 쓴다.

## 파일 구조
- `index.html` — 화면 (업로드, 미리보기, 톤/메모, 결과 카드). 함수 주소가 코드에 들어 있다.
- `supabase/functions/photo-to-post/index.ts` — Edge Function (배포 이름 `photo-to-post`)
- 배포는 Supabase MCP `deploy_edge_function`으로 했다. 로컬 파일을 고치면 같은 내용으로 다시 배포해야 반영된다.
- 함수 오류 응답에는 디버깅용 `detail` 필드가 들어간다. (키는 포함되지 않는다.)

## 배포 / 주의
- Edge Function 보안은 적용하지 않는다. `verify_jwt: false`로 배포하고, 인증·호출 횟수 제한·Origin 검사를 추가하지 않는다. (사용자 결정)
- 테스트용 임시 함수 `tmp-openrouter-test`는 삭제 대상이다. (대시보드에서 삭제)
- Supabase 프로젝트: `연습` (`krelzutkcwuzoxbwbhni`, ap-northeast-2)

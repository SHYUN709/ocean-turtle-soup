# 🐢 바다거북스프

이상한 이야기의 진상을 예/아니오 질문만으로 밝혀내는 추리 게임이에요. 오리지널 문제 6개와 장면 일러스트가 들어 있어요.

## 플레이 방식
- **AI 판정 모드**: Gemini가 질문마다 예 / 아니오 / 상관없음을 판정하고, 힌트도 줘요.
- **출제자 모드**: 한 명이 출제자가 되어 '진상 보기'로 정답을 몰래 확인하고, 질문마다 버튼으로 답해요. AI 호출이 실패하거나 무료 한도를 넘으면 자동으로 이 모드로 바뀌고, 화면에 이유가 짧게 나와요.

## 구조
진상과 핵심 포인트는 **이 저장소에 없어요.** 소스 보기로는 정답을 볼 수 없어요.

| 용도 | 어디서 | 클라이언트가 보내는 것 | 받는 것 |
|---|---|---|---|
| 질문 판정 | AI Logic 서버 프롬프트 템플릿 `turtle-judge-v1` | 문제 플래그, 질문, 지금까지 문답 | `verdict` 하나 |
| 진상 제출 채점 | 템플릿 `turtle-guess-v1` | 문제 플래그, 추측, 문답 | `verdict`, `progress` |
| 힌트 | 템플릿 `turtle-hint-v1` | 문제 플래그, 문답 | 한 문장 |
| 진상 공개 | Firestore `puzzles/{문제ID}` (읽기 전용) | 문제 ID | 진상, 핵심 포인트 |

- 모델: `gemini-3.5-flash-lite` (Gemini Developer API, 무료 Spark 요금제)
- 판정 응답에는 자유 문장이 없어요. 허용 목록 밖의 값은 `invalid`로 처리하고, 안내 문구는 화면에서 고정 문장으로 보여줘요.
- 진상 공개를 Firestore로 둔 이유: Gemini 한도를 넘어도 출제자 모드의 '진상 보기'가 동작해야 해서예요.
- App Check(reCAPTCHA Enterprise)로 등록된 도메인에서 온 요청만 받아요.
- 질문 버튼에는 3초 쿨타임이 있어요.

> 한계: 진상 공개는 문제 ID만 있으면 읽을 수 있어서, 개발자 도구를 다룰 줄 아는 사람은 직접 불러올 수 있어요. 소스 보기로 바로 보이는 문제는 막지만, 작정한 사람까지 막으려면 유료 서버가 필요해요.

## 설정
`index.html` 스크립트 맨 위에 두 값을 넣어요.

```js
const FIREBASE_CONFIG = { apiKey:"...", authDomain:"...", projectId:"...", appId:"..." };
const RECAPTCHA_ENTERPRISE_SITE_KEY = "6L...";
```

- `firebaseConfig`의 `apiKey`는 Gemini 키가 아니라 프로젝트를 찾는 공개 식별자라서 저장소에 올라가도 괜찮아요.
- Gemini API 키, 서비스 계정 파일, App Check 디버그 토큰은 **절대 커밋하지 마세요.**
- 값이 비어 있으면 출제자 모드로 시작하지만, 진상 공개도 Firebase에서 가져오기 때문에 '진상 보기'는 동작하지 않아요.

### Firebase 콘솔에서 할 일
1. 게임 전용 새 프로젝트를 만들어요. 요금제는 Spark(무료)예요.
2. 웹 앱을 등록하고 `firebaseConfig`를 복사해요.
3. **AI Logic** → 시작하기 → **Gemini Developer API**를 선택해요.
4. **AI Logic → Prompt templates**에 템플릿 3개를 등록해요. ID는 위 표와 같아야 하고, 저장한 뒤 🔒 잠가요. 템플릿 원문은 진상이 들어 있어서 저장소 밖에서 따로 관리해요.
5. **Firestore**를 만들고(위치 `asia-northeast3`), 아래 규칙을 적용한 뒤 `puzzles` 컬렉션에 문서 6개를 넣어요.
   - 각 문서 필드: `truth`(string), `keys`(array)
6. [reCAPTCHA](https://console.cloud.google.com/security/recaptcha)에서 웹 키를 만들어요. 도메인은 `shyun709.github.io`만 넣어요.
7. **App Check → 앱**에서 웹 앱에 reCAPTCHA Enterprise를 등록해요.
8. **App Check → API**에서 Firebase AI Logic과 Cloud Firestore를 **적용(Enforced)**으로 바꿔요.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /puzzles/{puzzleId} {
      allow get: if true;
      allow list, write: if false;
    }
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

### 로컬 테스트
`localhost`에서 열면 App Check 디버그 토큰이 브라우저 콘솔에 찍혀요. 그 토큰을 **App Check → 앱 → 디버그 토큰 관리**에 등록하면 돼요. 테스트가 끝나면 지우세요.

## 무료 한도 메모
- Gemini 무료 한도에는 **분당 요청 수 제한**이 있고, 프로젝트 전체가 함께 써요. 여러 명이 동시에 플레이하면 걸릴 수 있어요.
  - 429(한도 초과)가 나면 5초 뒤 한 번 다시 시도해요. 시간 초과나 서버 오류도 한 번 다시 시도해요.
  - 그래도 실패하면 그 세션은 출제자 모드로 바꿔요. 새로고침하면 다시 AI로 시작해요.
- reCAPTCHA Enterprise는 월 10,000회 평가까지 무료예요. App Check 토큰 수명을 24시간으로 두어서, 플레이어 한 명당 하루 1~2회 정도만 써요.
- 판정 캐시를 Firestore에 공유하는 건 하지 않았어요. 클라이언트가 쓸 수 있게 열면 엉터리 판정을 써넣을 수 있어서예요. 필요하면 자주 나오는 질문만 콘솔에서 직접 넣어 두고 읽기 전용으로 쓰는 방식을 추천해요.

## 실행
`index.html` 하나로 돌아가요. 설치나 빌드가 필요 없어요.

# JavaScript 모듈: 파일을 나눠도 스코프는 나뉘지 않는다

## 개념 정의

모듈은 파일 단위로 독립된 스코프를 갖고, `export`로 명시한 것만 밖으로 내보내며, `import`로 필요한 것만 가져오는 코드 조직 방식이다. 브라우저에서는 `<script type="module">`로 로드한 파일이 모듈이 된다.

여기서 중요한 건 반대편이다. 일반 외부 스크립트(`<script src>`)는 **파일을 나눌 뿐 격리하지 않는다.** 브라우저는 그 파일을 읽어다 한 문서에 밀어 넣는 것에 가깝게 처리하고, 결과적으로 모든 파일이 하나의 전역 공간을 공유한다. 폴더를 아무리 정리해도 실행될 때는 한 덩어리다.

## 동작 원리

| | 일반 외부 스크립트 | 모듈 |
|---|---|---|
| 로드 | `<script src="file.js">` | `<script type="module" src="file.js">` |
| 스코프 | 전역 공유 (모든 파일이 한 공간) | 파일마다 독립 |
| 이름 충돌 | 발생한다 | 발생하지 않는다 |
| 파일 간 소통 | 전역 변수·전역 함수를 통한 간접 소통 | `import`/`export`로 명시적 소통 |
| 실행 시점 | 태그를 만나는 즉시 | HTML을 다 읽은 뒤 (자동 `defer`) |
| strict mode | 선택 | 항상 적용 |

전역 공유가 왜 문제인지는 이름이 겹치는 순간 드러난다. 두 파일이 `var count`를 각각 선언하면 뒤에 로드된 쪽이 앞을 조용히 덮어쓰고, `let`이나 `const`가 끼면 `Identifier 'count' has already been declared`로 스크립트가 죽는다. 모듈에서는 이 질문 자체가 사라진다. 파일이 스코프 경계이기 때문에 같은 이름을 써도 서로 모른다.

내보내는 방식은 세 가지다.

```js
// Named Export — 여러 개 내보낸다. 가져올 때 이름이 정확해야 한다
export const PI = 3.14;
export function add(a, b) { return a + b; }
import { add, PI } from "./math.js";

// Default Export — 한 모듈에 하나. 가져오는 쪽이 이름을 정한다
export default function () { … }
import anyName from "./math.js";
```

셋째는 둘을 섞는 방식인데, 주 기능만 default로 두고 나머지를 named로 내보낸다. 팀 코드에서 Named를 선호하는 이유는 이름이 고정돼 있어서 검색과 리팩터링이 되기 때문이다. Default는 파일마다 다른 이름으로 불릴 수 있어서 `import util from`이 실제로 무엇인지 파일을 열어봐야 안다.

## 직접 확인한 예시

내가 만든 [skala-front](https://github.com/hyun907/skala-front) 페이지가 마침 두 방식을 한 페이지에서 같이 쓰고 있어서, 배포본 콘솔에서 스코프 차이를 직접 확인했다.

```html
<script src="../js/theme.js"></script>
<script src="../js/upDown.js" defer></script>
<script src="../js/grade.js" defer></script>
<script src="../js/bag.js" defer></script>
<script src="../js/baseConverter.js" defer></script>
<script type="module" src="../js/realtimeInfo.js"></script>
```

앞의 다섯 개는 일반 스크립트, 마지막 하나만 모듈이다. 각 파일이 최상위에 선언한 것을 `window`에서 찾아봤다.

```js
// 일반 스크립트로 들어온 것
typeof window.startUpDown    // "function"   (upDown.js)
typeof window.showMyBag      // "function"   (bag.js)
typeof window.subjects       // "object"     (grade.js의 var subjects)
typeof window.myBag          // "object"     (bag.js의 var myBag)

// 모듈로 들어온 것
typeof window.renderWeather  // "undefined"  (realtimeInfo.js)
typeof window.CITY_COORDS    // "undefined"  (weatherAPI.js)
typeof window.fetchWeather   // "undefined"  (weatherAPI.js)
```

같은 페이지, 같은 브라우저인데 일반 스크립트가 선언한 건 함수든 변수든 전부 `window`에 붙어 있고, 모듈이 선언한 건 하나도 없다. `grade.js`의 `subjects`와 `bag.js`의 `myBag`이 나란히 전역에 떠 있다는 건, 내가 나중에 다른 파일에서 그 이름을 쓰면 서로 간섭한다는 뜻이다.

여기서 하나 더 확인된 게 있다. `upDown.js`부터 `baseConverter.js`까지는 `defer`가 붙어 있는데도 전역에 있다. **`defer`는 실행 시점만 미루지 스코프에는 아무 영향이 없다.** 격리는 `type="module"`만 해준다.

## 자주 발생하는 오류

- **`file://`로 열면 모듈이 로드되지 않는다.** CORS 정책 때문이라 로컬 서버가 필요하다. VS Code Live Server를 쓰는 이유가 편해서가 아니라 이것 때문이다. HTML을 더블클릭해서 열었는데 모듈만 안 돌아간다면 대개 이 경우다.
- **`type="module"`을 빠뜨리면** `import` 문에서 SyntaxError가 난다. 파일 내용은 멀쩡한데 문법 오류가 뜨면 여기를 먼저 본다.
- **확장자 생략.** 브라우저 네이티브 모듈은 `./math.js`처럼 확장자를 반드시 써야 한다. Vite나 Webpack은 생략을 허용해서, 번들러를 쓰다가 순수 브라우저 환경으로 옮기면 여기서 걸린다.
- **순환 참조.** A가 B를, B가 A를 import하면 한쪽이 `undefined`로 들어온다. 에러가 아니라 값이 비는 형태라 원인을 찾기 어렵다.
- **모듈은 자동 strict mode다.** 선언 없이 대입하던 코드를 모듈로 옮기면 그 순간 에러가 된다.

## 프로젝트 적용

skala-front에서 날씨 기능을 두 파일로 쪼갤 때 이 구조를 썼다.

```
weatherAPI.js     — 좌표 데이터와 서버 호출을 책임진다. DOM을 건드리지 않는다
realtimeInfo.js   — 화면만 책임진다. weatherAPI에서 필요한 것만 import
```

```js
// realtimeInfo.js
import { CITY_COORDS, fetchWeather } from "./weatherAPI.js";
```

파일을 나눈 것보다 중요한 건 `export`한 목록이 그 파일의 공개 계약이 된다는 점이다. `weatherAPI.js`가 내보내는 건 좌표 데이터와 호출 함수 두 개뿐이라, 화면 쪽 코드가 API 호출 방식을 알 필요가 없다. 나중에 다른 날씨 API로 바꿔도 `export`하는 이름만 유지하면 화면 코드는 그대로다.

같은 분리가 LLM 파이프라인에서도 그대로 쓰인다.

```
promptTemplates.js   — 프롬프트 문자열
llmClient.js         — API 호출
postProcess.js       — 응답 파싱·검증
```

프롬프트를 별도 모듈로 빼두면 프롬프트만 고치고 나머지를 안 건드릴 수 있다. 실험을 몇 번 돌릴 수 있느냐가 여기서 갈린다.

## 배운 점

모듈을 `import`/`export` 문법으로만 알고 있었다. 콘솔에서 `window`를 뒤져보니 문법 이전에 스코프 문제였다. 내 프로젝트의 전역에는 함수 일곱 개와 배열 두 개가 이름 그대로 떠 있었고, 파일을 나눠둔 것과는 아무 상관이 없었다.

그래서 지금은 파일 분리와 스코프 분리를 다른 작업으로 센다. 폴더 구조를 정리하는 건 사람이 읽기 좋으라고 하는 일이고, 실행 시점의 격리는 `type="module"`을 붙여야 생긴다. Vue나 React 코드가 전부 모듈 위에서 도는 이유도 같다. 컴포넌트가 서로 간섭하지 않는다는 전제 자체가 모듈 스코프에서 나온다.

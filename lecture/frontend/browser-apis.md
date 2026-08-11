# 브라우저가 제공하는 API: fetch는 404에 실패하지 않는다

## 개념 정의

JavaScript 언어 명세(ECMAScript)에 들어 있는 것과, 브라우저가 실행 환경으로서 빌려주는 것은 다른 층이다.

| 언어에 있는 것 | 브라우저가 주는 것 |
|---|---|
| `Array` `String` `Math` `JSON` `Date` `Promise` `RegExp` | `document` `window` `setTimeout` `fetch` `localStorage` `alert` |

**JavaScript 엔진에는 시계도, 네트워크도, 저장소도, 화면도 없다.** `setTimeout`은 JavaScript 함수가 아니라 브라우저가 대신 시간을 재주는 기능이고, `fetch`도 마찬가지다. 같은 언어인데 Node.js에는 `document`가 없고 브라우저에서는 파일 시스템에 접근할 수 없는 이유가 여기 있다.

브라우저 API는 크게 네 갈래다.

| 분류 | 대표 | 하는 일 |
|---|---|---|
| DOM API | `document.querySelector()` `addEventListener()` | HTML 조작, 이벤트 |
| Timer API | `setTimeout()` `setInterval()` | 시간 뒤 실행, 반복 실행 |
| Storage API | `localStorage` `sessionStorage` | 브라우저에 데이터 저장 |
| Network API | `fetch()` | 새로고침 없이 서버와 통신 |

## 동작 원리

**Timer API는 ID를 반환하고, 그 ID로만 취소할 수 있다.**

```js
const id = setInterval(tick, 1000);
clearInterval(id);   // 이 번호를 잃어버리면 멈출 방법이 없다
```

`setTimeout`은 3초 뒤 닫히는 배너 같은 것에도 쓰지만, 실제로 더 자주 쓰이는 쪽은 디바운스다. 입력할 때마다 예약을 취소하고 다시 걸어서, 타이핑이 멈춘 뒤에만 요청이 나가게 만든다.

**Storage API는 유지 기간과 공유 범위로 갈린다.**

| | LocalStorage | SessionStorage |
|---|---|---|
| 유지 기간 | 영구적. 브라우저를 꺼도 남는다 | 탭을 닫으면 삭제 |
| 공유 범위 | 같은 도메인의 여러 탭 | 그 탭 안에서만 |
| 쓰임 | 다크 모드 설정, 자동 로그인 토큰 | 일회성 폼 입력, 진행 상태 |

과거에 쓰던 Cookie는 용량이 4KB로 작고 매 요청마다 서버로 자동 전송된다. Web Storage는 자동 전송되지 않는다는 게 가장 큰 차이다.

**`fetch`는 Promise를 반환하고, `await`가 두 번 필요하다.**

```js
const response = await fetch(url);   // ① 응답 헤더가 도착할 때까지
const data = await response.json();  // ② 본문을 전부 받아 파싱할 때까지
```

응답이 왔다는 것과 본문이 다 왔다는 것은 다른 사건이다. 그래서 두 번 기다린다.

## 직접 확인한 예시

### fetch는 404를 실패로 보지 않는다

없는 주소를 불러서 `try/catch`가 잡는지 확인했다.

```js
try {
  const res = await fetch("https://hyun907.github.io/skala-front/없는파일.json");
  console.log("reject 안 됨. status =", res.status, "/ ok =", res.ok);
  const data = await res.json();
  console.log("json 파싱 성공:", data);
} catch (e) {
  console.log("throw:", e.constructor.name, "-", e.message);
}
```

```
reject 안 됨. status = 404 / ok = false
throw: SyntaxError - Unexpected token '<', "<!DOCTYPE "... is not valid JSON
```

`fetch` 자체는 아무 문제 없이 넘어갔다. 서버에 요청이 갔고 응답을 받았으니 통신은 성공한 것이고, 그 응답의 내용이 404라는 건 `fetch`의 관심사가 아니다. 예외가 터진 곳은 그다음 줄이다. 404 페이지의 HTML을 JSON으로 파싱하려다 SyntaxError가 났다.

에러 메시지가 문제의 원인과 한참 떨어져 있다는 게 이 실험의 진짜 소득이다. 주소를 틀렸는데 화면에는 JSON 파싱 오류가 뜬다. `response.ok`를 직접 확인하지 않으면 매번 이 경로로 돌아온다.

### localStorage는 무엇을 넣든 문자열로 만든다

```js
localStorage.setItem("__tmp_num", 25);
localStorage.getItem("__tmp_num");          // "25"
typeof localStorage.getItem("__tmp_num");   // "string"

localStorage.setItem("__tmp_obj", { a: 1 });
localStorage.getItem("__tmp_obj");          // "[object Object]"
```

숫자 `25`는 문자열 `"25"`로 돌아온다. 꺼내서 바로 계산에 쓰면 `+`가 덧셈이 아니라 문자열 이어붙이기가 된다. 객체는 더 나쁘다. `[object Object]`라는 문자열이 저장되고 원래 데이터는 사라지는데, 에러가 나지 않아서 저장에 성공한 것처럼 보인다. 객체를 넣을 때 `JSON.stringify()`가 선택이 아닌 이유다.

## 자주 발생하는 오류

- **`response.ok`를 확인하지 않는다.** 위 예시가 그 결과다. `fetch`는 네트워크 자체가 끊긴 경우에만 reject한다.
- **객체를 그대로 `setItem`에 넘긴다.** `[object Object]`가 조용히 저장된다. `JSON.stringify()`로 넣고 `JSON.parse()`로 꺼낸다.
- **`setInterval`을 지우지 않는다.** 화면을 떠나도 계속 돌면서 메모리를 먹고, 이미 사라진 요소를 건드리다 에러를 뿜는다. 화면을 정리하는 시점에 `clearInterval`을 같이 넣는다.
- **`localStorage` 용량 초과.** 보통 5~10MB이고 넘으면 `QuotaExceededError`가 난다. 대화 기록 같은 걸 무한정 쌓으면 도달한다.
- **환경을 착각한다.** `document`와 `localStorage`는 Node에 없고, `fs`는 브라우저에 없다. LLM에게 코드를 부탁할 때 실행 환경을 적지 않으면 두 세계의 코드가 섞여 나온다.

## 프로젝트 적용

[skala-front](https://github.com/hyun907/skala-front)의 다크 모드가 Storage API를 쓰는 부분이다.

```js
function saved() {
  try {
    return localStorage.getItem(KEY);
  } catch (e) {
    return null;   // 사생활 보호 모드 등에서 접근이 막힌 경우
  }
}

// 저장된 선택이 있으면 그대로, 없으면 운영체제 설정을 따른다
apply(saved() || (media.matches ? "dark" : "light"));
```

`localStorage` 접근을 `try/catch`로 감싼 건 브라우저 설정에 따라 접근 자체가 막힐 수 있기 때문이다. 여기서 예외가 그대로 올라가면 테마 스크립트가 죽고 페이지 전체가 기본 색으로 뜬다. 저장에 실패해도 전환은 동작하게 두는 쪽이 맞다.

날씨 호출 쪽에는 위에서 확인한 404 문제를 막는 줄이 들어가 있다.

```js
const response = await fetch(url);
if (!response.ok) {
  throw new Error("서버 응답 오류: " + response.status);
}
return await response.json();
```

`response.ok`를 먼저 확인하고 던지면, 호출한 쪽의 `catch`가 파싱 오류가 아니라 서버 응답 오류를 받는다. 화면에 띄울 메시지도 그때 제대로 고를 수 있다.

한편 이 목록에 없는 것도 정해져 있다. **LLM API 키는 브라우저 저장소에 두지 않는다.** `localStorage`는 페이지의 모든 스크립트가 읽을 수 있고 개발자 도구 Application 탭에서 그대로 보인다. 클라이언트로 내려간 값은 숨길 방법이 없으니, 키가 필요한 호출은 서버를 거친다.

## 배운 점

`fetch`에 `try/catch`를 둘렀으니 실패는 처리됐다고 생각하고 있었다. 404를 직접 던져보니 `catch`는 조용했고, 대신 엉뚱한 자리에서 JSON 파싱 오류가 났다. 통신이 성공한 것과 원하는 데이터를 받은 것이 다른 사건이라는 걸 도구가 구분하고 있는데, 나는 그걸 하나로 묶어 생각하고 있었다.

이 구분을 알고 나니 브라우저 API 전반이 비슷하게 읽힌다. `localStorage`가 객체를 받고도 에러를 안 내는 것, `setInterval`이 화면을 떠나도 계속 도는 것 모두 API는 시킨 일만 하고 나머지 판단은 부르는 쪽에 남긴다는 같은 성질이다. 실패를 어디서 감지할지는 API가 정해주지 않는다.

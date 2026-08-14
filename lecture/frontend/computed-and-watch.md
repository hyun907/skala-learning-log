# computed와 watch: 값을 만들 것인가, 일을 시킬 것인가

## 개념 정의

Vue에서 반응형 값이 바뀌었을 때 쓸 수 있는 도구가 두 개 있다. 이름만 보면 비슷해 보이지만 하는 일이 다르다.

- **`computed`** — 다른 반응형 값들로부터 **새 값을 계산해서 캐싱**한다
- **`watch`** — 지정한 값을 감시하다가 바뀌면 **함수를 실행**한다

한 문장으로 갈린다. **`computed`는 값을 만들고, `watch`는 일을 시킨다.**

*"A가 바뀌면 B라는 **값**이 필요하다"* 면 `computed`, *"A가 바뀌면 뭔가를 **해야** 한다"* 면 `watch`다.

```js
import { ref, computed, watch } from 'vue'

const count = ref(0)

// computed — 반환값이 있다
const double = computed(() => count.value * 2)
console.log(double.value)

// watch — 반환값이 없다. 부수 효과를 일으킨다
watch(count, (newVal, oldVal) => {
  console.log(`${oldVal} → ${newVal}`)
})
```

| 축 | `computed` | `watch` |
|---|---|---|
| 목적 | 값 도출 | 부수 효과 실행 |
| 반환 | 계산된 값 (읽기 전용) | 없음 |
| 이전 값 | 알 수 없다 | `oldValue`로 받는다 |
| 캐싱 | **한다** | 해당 없음 |
| 감시 대상 | 함수 안에서 읽은 값이 자동으로 대상 | 첫 인자로 명시 |
| 비동기 | 넣으면 안 된다 | 넣으라고 있는 것 |
| 초기 실행 | 처음 읽을 때 1회 | 기본은 실행 안 함 |

## 동작 원리

### computed의 정체는 캐싱이다

`computed`가 단순히 "템플릿을 깔끔하게 하는 문법 설탕"이 아니라는 게 여기서 드러난다.

```vue
<script setup>
const count = ref(0)
const dummy = ref(0)          // count와 아무 상관 없는 값

const doubleByMethod = () => {
  console.log('메서드 실행')
  return count.value * 2
}

const doubleByComputed = computed(() => {
  console.log('computed 실행')
  return count.value * 2
})
</script>

<template>
  <p>{{ doubleByMethod() }}</p>
  <p>{{ doubleByComputed }}</p>
  <button @click="count++">count +1</button>
  <button @click="dummy++">dummy +1</button>
</template>
```

`dummy` 버튼을 누르면 `count`는 그대로다. 그런데 콘솔에는 이렇게 찍힌다.

| | 콘솔 |
|---|---|
| 메서드 | `메서드 실행` **찍힌다** |
| computed | 안 찍힌다 |

`dummy`가 바뀌면 컴포넌트가 리렌더링되고, **템플릿 안의 모든 표현식이 다시 평가된다.** 메서드 호출도 당연히 다시 일어난다. 하지만 `computed`는 자기가 의존하는 값(`count`)이 안 바뀌었으므로 **저장해 둔 결과를 그대로 돌려준다.**

계산이 무거울수록 이 차이가 커진다. 1,000개 목록을 필터링하고 정렬하는 계산이라면, 무관한 값이 바뀔 때마다 그걸 다시 하느냐 마느냐의 차이다.

그리고 `computed`는 **읽기 전용**이다. 결과를 직접 대입하면 경고가 난다. 값의 출처가 의존 값이므로 결과만 바꾸는 건 논리적으로 성립하지 않는다.

### watch가 객체 내부 변경을 놓치는 이유

`watch`에서 가장 많이 걸리는 지점이다.

```js
const user = ref({ name: '홍길동', age: 20 })

watch(user, (n, o) => { ... })                  // user.value.name을 바꿔도 안 걸린다
watch(user, (n, o) => { ... }, { deep: true })  // 내부 변경도 감지된다
```

이건 Vue의 변덕이 아니라 **참조(주소)** 문제다. `watch`는 기본적으로 감시 대상의 주소가 바뀌었는지를 본다. `user.value.name = 'X'`는 객체 내부만 고칠 뿐 주소는 그대로라, 감시자 입장에서는 아무 일도 안 일어난 것이다.

`deep: true`는 내부를 재귀적으로 추적해서 이 문제를 푼다. **대가가 있다 — `oldValue`를 못 받는다.**

```js
watch(user, (newVal, oldVal) => {
  // deep일 때 newVal === oldVal (같은 객체를 가리키고 있다)
}, { deep: true })
```

무엇이 바뀌었는지는 알아도 **무엇에서 바뀌었는지는 알 수 없다.**

이전 값이 필요하면 첫 인자를 함수(getter)로 준다.

```js
watch(() => user.value.age, (newAge, oldAge) => {
  console.log(`${oldAge}세 → ${newAge}세`)     // 이전 값이 정상적으로 온다
})
```

감시 대상이 객체가 아니라 원시값 `age`가 되므로 `deep`이 필요 없고, 주소 문제도 발생하지 않는다.

| 목적 | 문법 | `oldValue` |
|---|---|---|
| 객체 안에서 뭐라도 바뀌면 | `watch(obj, cb, { deep: true })` | ❌ |
| 객체의 특정 속성이 바뀌면 | `watch(() => obj.value.prop, cb)` | ✅ |

## 직접 확인한 예시

위의 `dummy` 실험을 실제로 만들어 콘솔로 확인했다. 무관한 값을 바꿨을 때 메서드는 매번 실행되고 `computed`는 침묵한다는 게 눈으로 보인다. **캐싱이 무엇인지는 이 실험 하나로 끝난다.**

`watch`는 형태별로 전부 만들어 봤다 — 단일 ref, 배열로 여러 개, `deep`, getter, `watchEffect`. 그중 `deep` 예제에는 **실패하는 코드를 지우지 않고 주석으로 남겨 뒀다.**

```js
// 실패하는 예시 (가장 많이 범하는 오류)
watch(user, (n, o) => { ... })   // user.value.name을 바꿔도 안 걸린다
```

동작하는 코드만 남기면 나중에 다시 봤을 때 "왜 `deep`을 붙였더라"가 안 떠오른다. **실패한 경로를 같이 남겨야 그 줄이 왜 거기 있는지가 보존된다.**

## 자주 발생하는 오류

**`computed`로 될 일을 `watch`로 하는 것.** 가장 흔하다.

```js
// 파생 값을 watch로 만들기
const fullName = ref('')
watch([first, last], () => { fullName.value = first.value + last.value })

// computed로 충분하다
const fullName = computed(() => first.value + last.value)
```

`watch` 버전은 상태가 하나 더 생기고, 초기값이 비어 있고, 동기화 책임이 개발자에게 넘어온다. `computed`는 그 셋이 다 사라진다.

**`computed` 안에 비동기를 넣는 것.** `async`를 붙이면 반환되는 건 계산된 값이 아니라 Promise다. 값이 필요한 자리에 Promise가 들어가서 화면에 `[object Promise]`가 뜬다. 비동기는 `watch` 쪽 일이다.

**`watch`를 남발해서 흐름을 잃는 것.** `watch`는 명령적이라 여러 개가 서로를 건드리기 시작하면 "이 값이 왜 바뀌었지"를 역추적하기 어려워진다. 값으로 표현할 수 있는 건 최대한 `computed`로 두는 게 낫다.

## 프로젝트 적용

날씨 앱에서 두 도구가 나뉘는 경계가 뚜렷하게 드러났다.

```js
// computed: 화면에 뿌릴 값을 만든다
const avgTemp  = computed(() => list.value.reduce((s, x) => s + x.temp, 0) / list.value.length)
const filtered = computed(() => list.value.filter(x => x.city.includes(keyword.value)))

// watch: 화면 밖에 영향을 준다
watch(keyword, (newKeyword) => {
  router.push({ path: '/weather', query: { search: newKeyword || undefined } })
})
```

**`computed`는 화면 안, `watch`는 화면 밖(URL·서버·저장소).** 이게 두 도구를 나누는 실질적인 기준이었다.

LLM을 붙이는 서비스에서도 같은 구조가 나온다.

```js
// 전송 가능 여부는 다른 상태로부터 나오는 값이다
const canSend = computed(() => prompt.value.trim().length > 0 && !loading.value)

// 모델 교체는 연결을 다시 맺는 행동이다
watch(selectedModel, async (model) => {
  messages.value = []
  await reconnect(model)
})
```

## 배운 점

`computed`의 캐싱은 프론트엔드 기법이 아니라 **"파생 값을 언제 계산할 것인가"** 라는 훨씬 오래된 문제의 한 답이다.

| | 매번 계산 | 저장해 두고 재사용 |
|---|---|---|
| Vue | 메서드 호출 | `computed` |
| DB | View / 서브쿼리 | Materialized View |
| Vue 렌더링 | `v-if` (매번 생성) | `v-show` (만들어 두고 재사용) |

셋 다 같은 거래를 한다. **저장 공간을 내고 계산 횟수를 산다.** 인덱스를 정리하면서 "읽기를 사고 쓰기와 공간을 판다"고 적었던 문장이 여기에도 그대로 적용된다 — [인덱스 설계와 동작](../database/index-design.md)에 같은 구조를 정리해 두었다.

`computed`를 처음 배울 때는 새로운 문법을 하나 외우는 일처럼 느껴졌는데, 실제로는 DB에서 이미 만난 개념이 다른 이름으로 나온 것이었다. **계층이 바뀌어도 같은 판단이 반복된다는 걸 알고 나면 외울 것이 줄어든다.**

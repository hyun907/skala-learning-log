# Git 기본 흐름: Staging Area는 저장이 아니다

## 개념 정의

Git은 파일 변경 이력을 스냅샷 단위로 저장하고, 원격 저장소를 통해 공유하는 분산 버전 관리 시스템이다. 명령어를 외우는 것보다 **작업 → 스테이징 → 커밋 → 공유**라는 4단계 구조를 잡는 게 먼저다.

```
파일 수정 → git add → Staging Area → git commit → Local Repository → git push → Remote
```

`git add`는 이번 커밋에 이 변경을 포함하겠다고 고르는 행위이지, 저장이 아니다. 저장(확정)은 `git commit`이 한다.

## 동작 원리

Staging Area는 작업 트리(내가 지금 보는 파일)와 마지막 커밋 사이에 낀 **별도의 스냅샷 영역**이다. `git add`를 실행하는 순간 파일의 그 시점 내용이 스테이징 영역에 복사되고, 그 뒤에 파일을 또 고쳐도 스테이징 영역의 내용은 자동으로 갱신되지 않는다. 다시 `add`해야 갱신된다.

이걸 몰라서 흔히 하는 오해가 있다. add했으니 지금 파일 내용이 그대로 다음 커밋에 들어가리라는 것이다. 실제로 들어가는 건 **add한 순간의 스냅샷**이다.

## 직접 확인한 예시

파일을 `add`한 뒤 커밋 전에 다시 수정하면 어떻게 되는지 재현해봤다.

```bash
echo "v1" > file.txt
git add . && git commit -m "init"

echo "v2-staged" > file.txt
git add file.txt          # 여기서 v2-staged가 스냅샷됨

echo "v3-unstaged" > file.txt   # add 이후 또 수정

git status --short
```

```
MM file.txt
```

`M`이 두 번 찍힌다. 앞의 `M`은 스테이징 영역이 마지막 커밋과 다르다는 뜻이고(v1 → v2-staged), 뒤의 `M`은 작업 트리가 스테이징 영역과 다르다는 뜻이다(v2-staged → v3-unstaged).

```bash
git diff --staged   # v1 → v2-staged 만 보여줌
git diff             # v2-staged → v3-unstaged 만 보여줌

git commit -m "commit"
git show HEAD:file.txt   # → v2-staged
cat file.txt              # → v3-unstaged
```

**커밋된 건 `v2-staged`고, 지금 작업 트리에는 `v3-unstaged`가 남아 있다.** `add` 이후의 수정은 다시 `add`하지 않는 한 이번 커밋에 들어가지 않는다는 게 실제로 확인됐다.

## 자주 발생하는 오류

- **`add`한 파일을 커밋 전에 다시 고쳤는데, 그 최신 내용이 커밋에 안 들어가 있다.** 위 예시가 정확히 이 상황이다. `git status`에 `MM`이 보이면 다시 `add`가 필요하다는 신호다.
- **`git add .`로 의도치 않은 파일까지 스테이징된다.** 빌드 결과물, `.env`, `.DS_Store` 같은 파일이 섞여 들어갈 수 있다. `git status`로 무엇이 스테이징됐는지 먼저 확인하는 습관이 필요하다.
- **Conflict를 에러로 오해한다.** Conflict는 두 사람이 같은 부분을 다르게 고쳤으니 어느 쪽을 쓸지 사람이 정하라고 Git이 판단을 넘기는 절차다.

## 프로젝트 적용

커밋 단위를 의미 있게 쪼개려면 이 스냅샷 구조를 이해하고 있어야 한다. 파일 하나를 여러 번 고치면서 관련 있는 변경끼리 나눠 커밋하고 싶을 때, 지금 add한 게 정확히 어느 시점의 스냅샷인지 헷갈리면 엉뚱한 내용이 커밋에 섞인다. 커밋하기 직전에 `git diff --staged`로 지금 확정하려는 내용이 이게 맞는지 확인하는 걸 습관으로 삼고 있다.

## 배운 점

`add`는 저장이 아니라는 문장은 알고 있었지만, `MM` 상태를 직접 보기 전까지는 그게 정확히 무슨 뜻인지 몰랐다. 스테이징 영역이 작업 트리를 실시간으로 반영하는 게 아니라 **그 순간의 스냅샷을 따로 들고 있다**는 걸 확인하고 나서야, 커밋 직전에 `git diff --staged`로 확인하라는 조언이 왜 필요한지 이해됐다.

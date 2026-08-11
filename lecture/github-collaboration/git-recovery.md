# Git 되돌리기: restore, reset, revert, stash

## 개념 정의

Git에서 되돌리는 방법은 하나가 아니다. **어느 단계까지 갔는가**에 따라 맞는 명령어가 다르다.

| 상황 | 명령어 |
|---|---|
| 아직 커밋 안 한 파일 수정을 취소하고 싶다 | `git restore` |
| 방금 한 커밋을 취소하되 작업 내용은 남기고 싶다 | `git reset --soft HEAD~1` |
| 이미 push해서 공유된 커밋을 되돌리고 싶다 | `git revert` |
| 지금 작업을 잠시 치워두고 다른 일을 해야 한다 | `git stash` |

가장 중요한 분기점은 **그 변경을 이미 다른 사람과 공유했는가**다.

## 동작 원리

`git reset`은 커밋 히스토리 자체를 다시 쓴다. 로컬에서만 쓴 커밋이면 문제가 없지만, 이미 push해서 다른 사람이 받아 간 커밋을 `reset`으로 지우면 그 사람의 로컬 히스토리와 원격 히스토리가 어긋나서 다음 `pull`이 깨진다.

`git revert`는 반대로 접근한다. 기존 커밋을 지우지 않고, **그 변경을 취소하는 새 커밋을 추가**한다. 히스토리가 보존되므로 공유 저장소에서 안전하다.

```
reset  : 히스토리를 다시 쓴다 (과거를 바꾼다)
revert : 히스토리에 새 커밋을 추가한다 (과거는 그대로, 결과만 취소)
```

## 직접 확인한 예시

**1. `reset --soft`가 스테이징 상태를 남기는지 확인**

```bash
git commit -m "commit A"
echo "line2" >> file.txt
git add . && git commit -m "commit B (실수)"

git reset --soft HEAD~1
git log --oneline   # commit A만 남음 — B는 사라짐
git status --short  # M file.txt — 변경 내용은 스테이징 상태로 남아 있음
```

`reset --soft`는 커밋만 취소하고 파일 변경은 그대로 둔다. 커밋 메시지를 잘못 썼거나 커밋을 다시 나누고 싶을 때 쓴다.

**2. `revert`가 정말 히스토리를 보존하는지 확인**

```bash
git revert --no-edit <커밋ID>
git log --oneline
```

```
01f0ce9 Revert "commit B (실수, 재커밋)"
0fb437e commit B (실수, 재커밋)
d8ae048 commit A
```

원래 커밋(`0fb437e`)과 되돌리는 커밋(`01f0ce9`)이 **둘 다** 로그에 남는다. `reset`을 썼다면 `0fb437e` 자체가 사라졌을 것이다.

**3. `reset --hard`로 날린 커밋이 정말 복구 불가능한지 확인**

이게 가장 궁금했던 부분이다. `reset --hard`로 커밋을 지운 뒤:

```bash
git reset --hard HEAD~1   # commit 2를 날림
git log --oneline
# 3fbbea7 commit 1        ← commit 2가 사라진 것처럼 보임

git reflog
# 3fbbea7 HEAD@{0}: reset: moving to HEAD~1
# 6d4f618 HEAD@{1}: commit: commit 2 (날릴 커밋)   ← 여전히 남아 있다
# 3fbbea7 HEAD@{2}: commit (initial): commit 1

git reset --hard 6d4f618  # reflog에서 찾은 해시로 복구
git log --oneline
# 6d4f618 commit 2 (날릴 커밋)   ← 되살아났다
# 3fbbea7 commit 1
```

`reset --hard`는 삭제가 아니라 브랜치 포인터를 옮기는 동작이다. 커밋 객체 자체는 한동안 저장소에 남아 있고, `reflog`가 그 흔적을 추적한다. 다만 이건 안전망이지 대안은 아니다 — 오래 방치하면 결국 가비지 컬렉션 대상이 된다.

## 자주 발생하는 오류

- **`reset` 후 push했다가 pull이 깨진다.** 이미 공유된 커밋을 `reset`으로 지웠기 때문. 이 상황을 만들지 않는 게 최선이고, 이미 공유된 커밋은 `revert`를 써야 한다.
- **`git restore`로 파일을 되돌렸는데 작업 내용이 완전히 사라진다.** 커밋 전 변경 사항은 Git이 따로 보관하지 않는다. 확신이 안 서면 `restore` 대신 `stash`를 쓰는 편이 안전하다 — stash는 나중에 다시 꺼낼 수 있다.
- **`stash pop` 중 충돌이 난다.** stash해 둔 사이에 같은 부분이 바뀐 경우다. 일반 병합 충돌과 같은 방식으로 해결하면 된다.

## 프로젝트 적용

복구할 수 있다는 확신이 있어야 실험을 과감하게 할 수 있다. 실수해도 되돌릴 방법이 있다는 걸 직접 확인해 두면, AI가 생성한 코드나 낯선 라이브러리를 시험 삼아 적용해볼 때 부담이 줄어든다. 다만 그 확신은 어디까지가 안전망이고 어디부터가 진짜 위험한지 구분할 수 있을 때만 유효하다 — `reset --hard`도 무한정 안전하지는 않다.

## 배운 점

`reset`과 `revert`의 차이는 읽어서는 잘 안 외워졌다. 공유된 커밋은 revert라는 규칙 자체는 외우고 있었는데, 두 커밋이 로그에 나란히 남는 걸 눈으로 보고서야 왜 그게 안전한 되돌리기인지 감이 왔다. `reflog` 복구도 마찬가지다 — 복구될 수도 있다는 문장보다 실제로 사라진 커밋 해시를 찾아 되살려 본 경험이 `reset --hard`를 대하는 태도를 바꿨다. 여전히 급하게 쓸 명령어는 아니지만, 최소한 실수해도 끝이 아니라는 걸 알고 쓰는 것과 모르고 쓰는 건 다르다.

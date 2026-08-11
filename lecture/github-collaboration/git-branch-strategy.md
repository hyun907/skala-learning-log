# Git 브랜치 전략: fast-forward는 흔적을 남기지 않는다

## 개념 정의

브랜치 전략은 여러 사람이 하나의 저장소에서 동시에 작업할 때, 브랜치를 언제 만들고 언제 합칠지 정해 둔 팀의 규칙이다. 대표적으로 정식 출시 주기가 있는 프로젝트에 맞는 **Git Flow**(`main`/`develop`/`feature`/`release`/`hotfix`)와, 소규모·CI/CD 중심 프로젝트에 맞는 **GitHub Flow**(`main` + `feature` + Pull Request)가 있다.

## 동작 원리

두 전략의 실제 차이는 브랜치 이름 규칙보다 **머지 결과를 히스토리에 어떻게 남길 것인가**에서 더 실감 나게 드러난다. Git의 `merge`는 상황에 따라 두 가지 방식으로 동작한다.

- **fast-forward merge**: `main`이 그 이후로 변경되지 않았다면, Git은 그냥 브랜치 포인터를 앞으로 옮긴다. 별도의 머지 커밋이 생기지 않는다.
- **`--no-ff` merge**: fast-forward가 가능한 상황이어도 강제로 머지 커밋을 만든다. 이 커밋들이 원래 별도 브랜치에서 작업됐다는 흔적이 히스토리에 남는다.

## 직접 확인한 예시

같은 상황에서 fast-forward와 `--no-ff`가 히스토리를 어떻게 다르게 남기는지 비교했다.

```bash
git switch -c feature/a
echo "a" >> file.txt
git add . && git commit -m "feature A 작업"
git switch main
git merge feature/a
git log --oneline --graph
```

```
* f97aa2c feature A 작업
* e651b76 init
```

브랜치에서 작업했다는 흔적이 **전혀 남지 않는다.** 로그만 보면 `main`에 바로 커밋한 것과 구분이 안 된다.

```bash
git switch -c feature/b
echo "b" >> file.txt
git add . && git commit -m "feature B 작업"
git switch main
git merge --no-ff feature/b -m "Merge feature/b"
git log --oneline --graph
```

```
*   3a59db5 Merge feature/b
|\
| * 0e4a6ed feature B 작업
|/
* f97aa2c feature A 작업
* e651b76 init
```

이번엔 머지 커밋(`3a59db5`)이 생기고, 그래프에 브랜치가 갈라졌다 합쳐진 모양이 그대로 남는다.

## 자주 발생하는 오류

- **로컬에서 `feature` 브랜치를 fast-forward로 merge하고 나면, 이 기능이 브랜치에서 리뷰를 거쳐 들어왔다는 맥락이 로그에서 사라진다.** GitHub에서 Pull Request로 머지하면 기본적으로 머지 커밋이 남기 때문에 이 문제를 잘 못 느끼지만, 로컬에서 직접 `merge`할 때는 의식하지 않으면 fast-forward가 기본값이라 흔적이 사라진다.
- **`hotfix` 브랜치를 `develop`에서 따는 실수.** Git Flow에서 `hotfix`는 지금 운영 중인 코드를 고치는 것이라 `main`에서 따야 한다. `develop`에는 아직 배포 안 된 기능이 섞여 있어서, 거기서 따면 검증 안 된 코드가 같이 딸려 나간다.

## 프로젝트 적용

소규모 팀이나 혼자 하는 프로젝트에서는 Git Flow의 5종 브랜치가 과한 오버헤드다. `main` + `feature/*` + Pull Request로 시작하는 GitHub Flow가 기본값으로 더 현실적이다. 대신 로컬에서 브랜치를 병합할 일이 생기면 `--no-ff`를 의식적으로 쓴다 — 나중에 이 기능이 언제 어떤 단위로 만들어졌는지 로그에서 추적할 수 있어야 하기 때문이다.

## 배운 점

Git Flow는 무겁고 GitHub Flow는 가볍다는 설명을 나는 브랜치 개수 차이로만 읽고 있었다. fast-forward와 `--no-ff`의 로그를 나란히 놓고 보니 브랜치 전략의 진짜 쟁점 중 하나는 **나중에 이력을 봤을 때 브랜치 단위의 작업이 보이느냐**였다. 브랜치를 나눠서 작업해도 fast-forward로 합쳐버리면, 나눠서 작업한 의미가 히스토리에서는 사라진다.

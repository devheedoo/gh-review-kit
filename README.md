# gh-review-kit

[who-reviews-what](https://github.com/devheedoo/who-reviews-what)에서 실험한 리뷰 흐름을 다른 저장소에 적용하는 gh extension이다.
로직(reusable workflow)은 이 저장소에만 두고, 대상 저장소에는 이 저장소를 참조하는 짧은 caller와 그 저장소만의 기준을 둔다.

## 적용되는 것

| 대상 저장소에 생기는 것 | 하는 일 |
|---|---|
| `.github/workflows/review-kit-ci.yml` | 저장소가 정한 검증 명령(lint·typecheck·test·build)을 PR마다 돌린다 |
| `.github/workflows/review-kit-risk.yml` | 경로와 diff 크기로 `risk:low/mid/high` 라벨을 붙이고, high인데 사람 확인이 없으면 `risk-gate`를 실패시킨다 |
| `REVIEW.md` | 리뷰 태그 규칙(`[Important]`/`[Nit]`/`[Pre-existing]`)과 이 저장소에서 특히 볼 것 |
| 라벨 4개 | `risk:low`, `risk:mid`, `risk:high`, `reviewed-by-human` |
| ruleset `review-kit` (public만) | PR 필수, `review-kit-ci / ci`·`review-kit-risk / risk-gate` 통과, 리뷰 스레드 resolve, 삭제·force push 금지 |

## 설치

```bash
gh extension install devheedoo/gh-review-kit
```

## 사용

```bash
gh review-kit init <owner/repo> --dry-run   # 감지 결과와 만들 파일만 출력
gh review-kit init <owner/repo>             # 라벨 생성 + 설정 PR
# 설정 PR에서 high-paths와 REVIEW.md "이 저장소" 절을 채우고, reviewed-by-human 라벨을 붙여 병합
gh review-kit protect <owner/repo>          # 기본 브랜치에 ruleset (public만)
```

`init`이 감지하는 값

- 패키지 매니저: lockfile (여럿이거나 없으면 `--pm`으로 지정)
- Node 버전: `.nvmrc` → `.node-version` → `engines.node`, 정확한 버전이 아니면 22
- 검증 명령: `package.json`에 있는 `lint`, `typecheck`(`type-check`), `test`, `build`
- `enforce`: public이면 `true`, private이면 `false` (GitHub Free는 private 저장소에 ruleset을 걸 수 없어 경고만 남긴다)

저장소마다 사람이 정할 값은 `review-kit-risk.yml`의 `with:`와 `REVIEW.md`의 "이 저장소" 절뿐이다. 이 파일들은 공통 high 경로라, 기준을 바꾸는 PR도 사람 확인을 거친다.

## 사람 확인 흐름

1. risk:high PR의 diff를 직접 보고 `reviewed-by-human` 라벨을 붙인다.
2. workflow가 그 시점의 head 커밋에 `human-review` 상태를 남기고 `risk-gate`가 통과한다.
3. 새 커밋이 올라오면 라벨이 떨어지고 게이트가 다시 막힌다. 새 커밋까지 보고 다시 붙인다.

## 버전

caller는 `@v1`을 참조한다. 호환되는 수정은 `v1` 태그를 옮겨 모든 저장소에 반영하고, 입력이 바뀌는 수정은 `v2`로 올린다.

## 한계

- PR이 caller 파일을 고쳐 다른 workflow를 부르게 하면 게이트를 피할 수 있다. caller는 high 경로지만 그 판정도 PR 브랜치의 caller로 돈다. 완전히 막으려면 org의 required workflow가 필요하다.
- fork에서 온 PR은 토큰이 읽기 전용이라 라벨을 붙이지 못한다. 개인 저장소용이라 지원하지 않는다.
- 승인 수(0/1/2명)는 `reviewed-by-human` 라벨 하나로 대신한다.
- 모노레포, JS가 아닌 저장소의 CI는 지원하지 않는다 (위험도 분류만 적용된다).

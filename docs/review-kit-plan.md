# review-kit 계획

who-reviews-what에서 만든 리뷰 흐름(CI 게이트, 위험도 분류, 사람 확인, REVIEW.md 태그 규칙)을 다른 개인 저장소에 적용한다.
로직은 중앙 저장소 한 곳에 두고, 각 저장소에는 중앙을 참조하는 짧은 파일과 그 저장소에만 해당하는 기준만 둔다(② 참조 방식, 2026-11-19 결정).

## 목표와 범위

- 목표
  - 규칙을 고칠 때 중앙 저장소만 고치면 모든 적용 저장소에 반영된다.
  - 저장소마다 다른 기준(검증 명령, 위험 경로, 도메인별 리뷰 포인트)은 그 저장소에만 둔다.
  - 스크립트 한 번으로 새 저장소에 적용한다.
- 범위
  - 대상: `devheedoo` 개인 저장소 중 직접 고른 활성 저장소. 보관(archived) 저장소는 제외한다.
  - 제외: 회사 org(growdle) 저장소. 팀이 정할 일이고, p2s-web에는 이미 자체 PR 템플릿·리뷰어 자동 지정·risk 규칙이 있다.
  - 제외: 모노레포(`working-directory` 분기), JS가 아닌 저장소의 CI. 필요해지면 후속으로 다룬다.
- 영향 파일
  - 신규 저장소 `devheedoo/gh-review-kit`: reusable workflow 2개, 템플릿, 스크립트 (gh extension)
  - 신규 저장소 `devheedoo/.github`: 기본 PR 템플릿
  - 각 대상 저장소: caller workflow 2개, `REVIEW.md` (PR 하나로 추가)
  - 각 공개 대상 저장소의 GitHub 설정: 라벨 4개, ruleset 1개
  - `~/.claude/CLAUDE.md`: 리뷰·자체 검증 규칙 한 절
  - who-reviews-what: 바꾸지 않는다 (원문 재현 샘플로 그대로 둔다)
- 동작 변화
  - 대상 저장소의 PR에 `review-kit-ci / ci`, `review-kit-risk / risk-gate` 체크와 `risk:*` 라벨이 생긴다.
  - 공개 저장소는 ruleset 적용 후 main 직접 push가 막히고, 위 두 체크 통과와 리뷰 스레드 resolve가 병합 조건이 된다.
- 위험
  - 중앙 reusable workflow가 고장 나면 모든 적용 저장소의 PR이 함께 막힌다. `@v1` 태그를 옮기기 전에 시범 저장소에서 먼저 확인한다.
  - ruleset을 먼저 걸면 아직 없는 체크를 기다리느라 설정 PR이 병합되지 못한다. 그래서 `init`과 `protect`를 나눈다.
  - 대상 저장소에 이미 있는 workflow·템플릿을 덮어쓰면 기존 CI가 깨진다. 스크립트는 기존 파일을 덮어쓰지 않는다.

## 저장소마다 달라지는 기준

이 계획의 핵심이다. 값마다 어디에 두고, 누가 정하는지를 고정한다.

| 기준 | 두는 곳 | 스크립트가 정하는 방법 | 기본값 | 사람이 확인할 것 |
|---|---|---|---|---|
| 패키지 매니저 | ci caller `with.package-manager` | lockfile로 감지 (`package-lock.json`→npm, `pnpm-lock.yaml`→pnpm, `yarn.lock`→yarn, `bun.lock[b]`→bun) | npm | lockfile이 둘 이상이면 스크립트가 멈추고 묻는다 |
| Node 버전 | ci caller `with.node-version` | `.nvmrc` → `.node-version` → `package.json` `engines.node` 순 | 22 | 범위 표기(`>=18`)면 기본값을 쓰고 경고한다 |
| 검증 명령 | ci caller `with.commands` (줄마다 script 이름) | `package.json` scripts에서 `lint`, `typecheck`/`type-check`, `test`, `build` 중 있는 것만, 이 순서로 | 감지된 것 | `test`가 watch 모드(`vitest` 단독 등)면 CI에서 멈추지 않는다. PR에서 확인 |
| high 경로 (공통) | 중앙 reusable workflow | 고정 | `.github/`, 감지된 lockfile, `package.json`, 빌드·lint·tsconfig 설정 | 없음 |
| high 경로 (저장소별) | risk caller `with.high-paths` | 비워 둔다. 경로 이름으로 추측하지 않는다 | 없음 | 금액·인증·결제 등 그 저장소의 민감 경로를 PR에서 채운다 |
| low 경로 (저장소별) | risk caller `with.low-paths` | `*.md`만 넣는다 | `*.md` | 문구 전용 파일(`src/copy.ts` 같은 것)이 있으면 추가 |
| diff 크기 기준 | risk caller `with.size-threshold` | 고정 | 400 | 큰 생성 파일이 잦은 저장소면 조정 |
| 게이트 강제 여부 | risk caller `with.enforce` | 저장소 공개 여부로 정함: public→`true`, private→`false` | — | 없음 |
| 리뷰 포인트 | `REVIEW.md` 저장소별 절 ("특히 볼 것", "지적하지 않을 것" 추가분) | `TODO` 자리만 만든다 | 빈 절 | 사람이 채운다. 채우기 전에는 공통 태그 규칙만 적용된다 |
| 기본 브랜치 | ruleset 조건 | `~DEFAULT_BRANCH`로 지정해 감지하지 않는다 | — | 없음 |

- private 저장소에서 `enforce: false`면 `risk-gate`는 실패 대신 경고(`::warning::`)만 남긴다. GitHub Free에서는 private 저장소에 ruleset을 걸 수 없어 실패해도 막지 못하고, 빨간 체크만 쌓이기 때문이다.
- 저장소별 값은 전부 caller 2개와 `REVIEW.md`에만 있다. 이 세 파일 경로를 공통 high 경로에 넣어, 기준을 바꾸는 PR은 사람 확인을 거치게 한다.

## 결정 사항

- **설정은 caller의 `with:`에 둔다.** 별도 설정 파일(`.github/review-kit.yml`)은 두지 않는다.
  - 별도 파일을 base 브랜치에서 읽으면 PR이 자기 분류 기준을 못 바꾸지만, caller 파일 자체를 PR이 바꿀 수 있어 보안상 이득이 없다. 파일과 파싱만 늘어난다.
- **파일 이름에 `review-kit-` 접두사를 붙인다.** `.github/workflows/review-kit-ci.yml`, `review-kit-risk.yml`
  - 대상 저장소에 이미 있는 `ci.yml`과 충돌하지 않게 한다. 체크 이름은 `review-kit-ci / ci`, `review-kit-risk / risk-gate`가 되고 ruleset도 이 이름을 쓴다.
- **중앙 참조는 `@v1` 이동 태그로 한다.** 고치면 태그를 옮겨 전체에 반영하고, 깨지는 변경은 `v2`로 올린다.
  - SHA 고정은 안전하지만 "한 번 고치면 전체 반영"이라는 목표와 맞지 않는다.
- **`REVIEW.md`는 공통 블록 + 저장소별 절로 나눈다.**
  - 공통 블록(태그 3종, 지적 형식, 공통 "지적하지 않을 것")은 `<!-- review-kit:common v1 -->` 표시 사이에 복사한다. REVIEW.md는 사람과 다른 AI 리뷰어도 읽으므로 저장소 안에서 완결돼야 한다.
  - 표시를 두면 나중에 공통 블록만 갱신할 수 있다. 갱신 명령은 이번 범위가 아니다.
- **스크립트는 `init`과 `protect` 두 명령이다.** 순서 문제(체크가 생기기 전에 ruleset을 걸면 설정 PR이 막힘) 때문에 나눈다.
- **자체 검증과 REVIEW.md 연결은 전역 `~/.claude/CLAUDE.md`에 한 번만 둔다.** 저장소마다 CLAUDE.md를 고치지 않는다.
  - "PR 전에 `review-kit-ci.yml`의 `commands`를 로컬에서 먼저 돌린다", "리뷰할 때 저장소의 `REVIEW.md`를 읽고 태그 규칙을 따른다"
- **PR 템플릿은 `devheedoo/.github` 공개 저장소의 기본 템플릿으로 둔다.** 자기 템플릿이 없는 저장소에만 적용되므로 기존 템플릿을 건드리지 않는다.
- **who-reviews-what은 전환하지 않는다.** 원문 재현 샘플이라 파일이 저장소 안에서 다 읽혀야 한다.

## 열린 질문

권장안으로 확정했다 (2026-11-19).

- [x] 배포 형태 → **gh extension** `devheedoo/gh-review-kit` (`gh review-kit init|protect`). reusable workflow도 같은 저장소에 둔다.
- [x] CODEOWNERS → **만들지 않는다.** 혼자 쓰는 저장소에서는 효과가 없고 high 경로와 이중 관리만 생긴다.
- [x] `REVIEW.md` 저장소별 절 → **빈 `TODO`.** 초안 자동 생성은 필요하면 후속으로 다룬다.
- [x] private 저장소 → **설치하고 `enforce: false`.** Actions 사용 시간(Free 월 2,000분 [확인 필요])을 쓰는 점은 감안한다.
- [x] 시범 저장소 → **새 테스트 저장소** `devheedoo/review-kit-sandbox`. 확인 뒤 삭제 여부는 따로 묻는다.

## 단계

되돌리기 단위: 1~4는 `gh-review-kit` 저장소의 단계별 커밋(새 저장소라 main에 직접), 5는 `.github` 저장소, 6은 전역 설정, 7은 시범 저장소.
`@v1` 태그는 7단계 caller가 참조해야 하므로 4단계 뒤에 먼저 붙이고, 7단계에서 고칠 것이 생기면 옮긴다. 대상 저장소의 변경은 설정 PR 하나로 revert할 수 있고, ruleset은 `protect`가 출력한 id로 삭제한다.

- [x] 1. `gh-review-kit` 저장소 생성 + reusable `risk.yml` — who-reviews-what의 `risk-label.yml`을 옮기고 `high-paths`·`low-paths`·`size-threshold`·`enforce` 입력과 공통 high 경로(lockfile 종류별)를 추가 — 권장 모델: Opus 5.5 (분류·게이트 경계와 라벨 경합 처리를 그대로 옮겨야 함)
- [x] 2. reusable `ci.yml` — `package-manager`·`node-version`·`commands` 입력, 매니저별 설치 명령과 캐시 — 권장 모델: Sonnet 5.5 (명세가 정해진 워크플로)
- [x] 3. 템플릿 — caller 2개, `REVIEW.md`(공통 블록 + 저장소별 `TODO`) — 권장 모델: Sonnet 5.5 Medium (기존 문서에서 옮기는 작업)
- [x] 4. 스크립트 `init` / `protect` — 감지 → 파일 생성(기존 파일 건너뜀) → 라벨 생성 → 브랜치·PR / 공개 저장소에만 ruleset 적용, `--dry-run` 지원, 다시 실행해도 결과가 같게 — 권장 모델: Opus 5.5 (감지 분기와 순서 제약이 많음)
- [ ] 5. `devheedoo/.github` 저장소에 기본 PR 템플릿 — 권장 모델: Haiku 5.5 (파일 하나 복사)
- [ ] 6. `~/.claude/CLAUDE.md`에 자체 검증·REVIEW.md 절 추가 — 권장 모델: Sonnet 5.5 Medium (문서 한 절)
- [ ] 7. 시범 적용 — 테스트 저장소에 `init` → 설정 PR 병합 → `protect` → low·mid·high PR로 라벨·게이트·`reviewed-by-human` 흐름 확인 → `@v1` 태그 — 권장 모델: Opus 5.5 (실제 GitHub 동작 검증)

## 처리할 엣지 케이스

- 대상 저장소에 같은 이름의 workflow·`REVIEW.md`가 이미 있음 → 건너뛰고 목록으로 알린다.
- lockfile이 없거나 둘 이상 → 멈추고 매니저를 인자로 받는다.
- `package.json`이 없음 → ci caller를 만들지 않고, ruleset에서도 `review-kit-ci / ci`를 빼고 `risk-gate`만 요구한다.
- 라벨이 이미 있음(색·설명이 다름) → `--force`로 맞춘다.
- 기존 ruleset이 있음 → 지우지 않고 `review-kit` 이름의 ruleset을 따로 추가한다. 여러 ruleset은 겹쳐서 가장 엄격한 쪽이 적용된다.
- 설정 PR 자체 → `.github/` 경로라 high로 분류된다. `protect` 전이라 막히지는 않는다.
- private 저장소에서 `protect` 실행 → ruleset을 걸지 않고 이유를 출력한 뒤 정상 종료한다.
- fork에서 온 PR → 토큰이 읽기 전용이라 라벨을 못 붙인다. 개인 저장소라 지원하지 않고 README에 적는다.

## 알려진 한계

- PR이 caller 파일을 고쳐 다른 workflow를 부르게 하면 게이트를 피할 수 있다. caller는 high 경로라 사람 확인 라벨이 필요하지만, 그 판정 자체도 PR 브랜치의 caller로 돈다. 완전히 막으려면 org의 required workflow가 필요하고, 개인 계정에는 없다.
- 승인 수(0/1/2명)는 who-reviews-what과 같이 `reviewed-by-human` 라벨 하나로 대신한다.

# loop:review — 기계적 검사 · 수정 안전 루프

SKILL.md 의 **Phase 0.5** 와 **수정 요청을 받았을 때** 참조한다.

판단이 필요 없는 것부터 걷어내면, 렌즈 검사(Phase 2)를 설계와 로직에 쓸 수 있다.
리뷰에서 나오는 지적의 상당수는 **명령어 몇 줄로 30초에 잡힌다.**

---

## 준비 — base와 대상 파일 고정

**`main` 을 가정하지 않는다.** Phase 0 에서 확정한 base를 변수에 넣고 시작한다.

```bash
BASE=origin/release/1.4             # ← Phase 0 결과를 넣는다
FILES=$(git diff $BASE --name-only --diff-filter=ACM)
git diff --stat $BASE...HEAD
```

## 1. 디버그 잔여물

```bash
grep -rnE "console\.(log|debug)|debugger;|println\(|System\.out\.print|printStackTrace\(\)" $FILES
```

## 2. 비활성화된 테스트 (⚠️ 가장 조용한 결함)

**CI를 초록색으로 만든 채 커버리지를 비운다.** 통과 배지만 보면 절대 안 보인다.

```bash
grep -rnE "\.only\(|\.skip\(|xit\(|xdescribe\(|@Disabled|@Ignore" $FILES
```

## 3. 새로 들어간 TODO

기존 TODO 말고 **이번 diff에서 추가된 것만** 본다.

```bash
git diff $BASE | grep "^+" | grep -nE "TODO|FIXME|XXX|HACK"
```

## 4. 자격증명·비밀값

```bash
git diff $BASE | grep "^+" | grep -niE "(api[_-]?key|password|secret|token)[[:space:]]*[:=][[:space:]]*['\"][^'\"]{8,}"
git ls-files | grep -E "\.env$|\.pem$|credentials|id_rsa"
```

## 5. 커밋되면 안 되는 것

```bash
git status --porcelain
git ls-files | grep -E "\.idea|\.vscode/|\.DS_Store|node_modules|/build/|/target/|\.class$"
```

## 6. 빌드·타입·린트·테스트

**명령어를 추측하지 않는다.** 저장소에서 먼저 찾는다.

```bash
cat package.json 2>/dev/null | grep -A20 '"scripts"'   # JS/TS
ls gradlew pom.xml 2>/dev/null                          # JVM
```

찾은 것으로 돌린다. JVM은 **`clean` 을 붙여야 캐시 착시를 피한다.**
Turbo·Nx 같은 캐시 러너는 변경 파일이 있는데 cache hit이면 **패키지 안에서 직접 한 번 더** 돌린다.

## 7. 최신 base에서 재검증

내 브랜치에서만 통과하는 변경은 머지 후에 base를 깨뜨린다.

```bash
git fetch origin --quiet
git log --oneline HEAD..$BASE          # base가 앞서 나갔나?
```

**앞서 나갔으면** — 어느 쪽으로 맞출지는 push 여부로 갈린다:

| 상태 | 방법 | 이유 |
|---|---|---|
| 아직 push 안 함 | `git rebase $BASE` | 히스토리가 깨끗 |
| **이미 push 함** | `git merge $BASE` | rebase 하면 **force push가 필요**해진다 |

⚠️ **이미 push한 브랜치를 rebase 하지 않는다.** 리뷰어가 보던 커밋이 사라지고, 남이 그 브랜치를 받아갔으면 충돌이 난다.

맞춘 뒤 6번을 다시 돌린다. 여기서 깨지면 **남의 변경과의 충돌**이고, 지금 아는 게 머지 후보다 훨씬 싸다.

---

## 수정 요청을 받았을 때 (사용자가 "고쳐줘" 라고 했을 때만)

리뷰는 기본이 읽기 전용이다. 수정은 **한 건씩 고치고 즉시 검증한다.**

```bash
# 1. 고치기 전에 사본을 뜬다 — 이게 안전장치다
cp src/foo.ts /tmp/loop:review-backup-foo.ts

# 2. 수정

# 3. 즉시 검증 (그 파일만)
npx tsc --noEmit && npx eslint src/foo.ts

# 4. 실패하면 사본에서 복원하고 사람 판단으로 넘긴다
cp /tmp/loop:review-backup-foo.ts src/foo.ts
```

### 🛑 `git checkout -- <파일>` 로 되돌리지 않는다

그 파일에는 **아직 커밋되지 않은 작업이 통째로** 들어있을 수 있고, `git checkout` 은 그것까지 전부 날린다. 반드시 사본에서 복원한다.

### 루프 규칙

- **한 번에 한 건.** 여러 건 고치고 한 번에 검증하면 어느 수정이 깨뜨렸는지 모른다.
- 같은 파일을 **두 번 되돌리게 되면 멈추고** 사람에게 넘긴다.
- 수정 후에는 다시 훑는다 — 수정이 새 문제를 만들었을 수 있다. **재검토는 최대 2회.**

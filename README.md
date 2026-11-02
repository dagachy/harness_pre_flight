# condition

Claude Code 하네스 점검(condition) 스킬 번들.
"프로젝트 시작 전에 하네스 상태를 자동으로 감사한다"는 목적으로 모은 pre-flight 플러그인.

각 스킬은 책 *Harness Engineering with Claude Code* 의 특정 예제가 가르치는 원칙 하나를 **정적 점검**으로 변환한 것이다 (실행 없이 파일만 읽어 판정).

---

## 현재 포함된 스킬 (9종)

### A. 스킬 품질 (SKILL.md 대상)

- **separation-signals** — 분리 신호 3종(크기 / 도메인 분기 / 조건부 상세) regex 진단 *(ex-05-05)*
- **prose-pruner** — 컨텍스트 절약 3원칙(일반 지식 / 도메인 관례 / 산문 압축) LLM 진단·가지치기 *(ex-05-09)*
- **menu-pattern** — 라우터 메뉴판 원칙(R1 크기 / R2 조건부 링크 / R3 본문 금지 / R4 cross-ref 금지) regex 진단 *(ex-06-12)*
- **antipattern-detector** — 안티패턴 3종(거대 SKILL / references 부재 / 근거 누락) 심각도 라벨링 *(ex-05-12)*

### B. 에이전트 품질 (agent `.md` 대상)

- **agent-frontmatter** — frontmatter 필수 4필드(name/description/model/tools) + 본문 완비 검증 *(ex-07-13)*
- **agent-tools-contract** — tools 필드의 역할 적합성(권한 과소/과잉, 자동 커밋 리스크) heuristic 진단 *(ex-04-01 + ex-11-04)*

### C. 하네스 구조 (디렉토리 대상)

- **harness-structure** — 뼈대 구조(CLAUDE.md / .claude/agents / .claude/skills + frontmatter) 완비 진단 *(ex-02-02)*
- **team-size-governor** — 팀 규모 상한(전체 수 / Phase별 동시 활성) 거버넌스 *(ex-06-15 + ex-12-03)*
- **dag-validator** — 작업 의존성 DAG(순환 / dangling / 병렬 레벨) 검증 *(ex-11-05 + ex-13-01)*

---

## pre-flight 권장 순서

```
1. harness-structure         # 뼈대 있나?
2. agent-frontmatter         # agent 로드 가능한가?
3. agent-tools-contract      # agent 권한 적합한가?
4. separation-signals        # 스킬 분리 필요한가?
5. antipattern-detector      # 안티패턴 심각도는?
6. menu-pattern              # 라우터 적합한가?
7. prose-pruner              # 본문 다이어트 가능한가?
8. team-size-governor        # 팀 규모 적정한가?
9. dag-validator             # 의존성 DAG 유효한가?
```

pre-flight 전체 통과 후 프로젝트 kickoff.

---

## 설치 (한 줄)

```bash
git clone https://github.com/dagachy/condition.git ~/condition-src
bash ~/condition-src/scripts/install.sh
```

옵션:

```bash
bash ~/condition-src/scripts/install.sh --target /path/to/.claude
bash ~/condition-src/scripts/install.sh --symlink
```

기존 파일은 `.bak.<timestamp>` 로 백업.

---

## 사용법

### A. 스킬 품질 점검

```
~/.claude/skills/my-skill/SKILL.md 분리 신호 점검해줘      # separation-signals
~/.claude/skills/my-skill/SKILL.md 메뉴판 원칙 점검        # menu-pattern
~/.claude/skills/my-skill/SKILL.md 3원칙 다이어트해줘       # prose-pruner
~/.claude/skills/my-skill/SKILL.md 안티패턴 점검           # antipattern-detector
```

### B. 에이전트 품질 점검

```
~/.claude/agents/foo.md frontmatter 점검                 # agent-frontmatter
~/.claude/agents/foo.md 권한 적합성 점검                  # agent-tools-contract
```

### C. 하네스 구조 점검

```
<project-root> 하네스 구조 점검                           # harness-structure
<project-root> 팀 규모 점검                              # team-size-governor
tasks.json DAG 점검                                     # dag-validator
```

### 배치 모드

디렉토리를 지정하면 하위 모든 대상을 병렬 감사해 집계 표를 반환한다.

```
~/.claude/skills/ 전체 분리 신호 감사해줘
~/.claude/agents/ 전체 frontmatter 점검
```

### 임계값 override

세션 환경변수로 조정:

- `SIZE_THRESHOLD` (separation-signals, antipattern-detector)
- `MENU_MAX_LINES`, `MENU_MIN_LINKS` (menu-pattern)
- `MIN_REDUCTION_PCT` (prose-pruner)
- `MAX_TEAM_SIZE`, `MAX_CONCURRENT` (team-size-governor)
- `WHY_MIN`, `REF_DIR_NAMES` (antipattern-detector)
- `REQUIRED_FIELDS`, `RECOMMENDED_FIELDS`, `MIN_BODY_LINES` (agent-frontmatter)
- `READ_ONLY_KEYWORDS`, `ORCHESTRATOR_KEYWORDS`, `WRITE_TOOLS`, `DELEGATE_TOOLS` (agent-tools-contract)

---

## 자가 검증 결과 (모든 스킬 3/3 PASS)

| 스킬 | 샘플 디렉토리 | 결과 |
|------|---------------|------|
| separation-signals | `samples/{size,domain,conditional}/` | 3/3 (regex 결정론) |
| prose-pruner | `samples/{violations,clean}/` | 2/2 (anchor 기반) |
| menu-pattern | `samples/{compliant,violations}/` | 2/2 (regex 결정론) |
| **harness-structure** | `samples/{complete,partial,empty}/` | 3/3 (bash 결정론) |
| **agent-frontmatter** | `samples/{complete,missing-tools,no-body}/` | 3/3 (bash 결정론) |
| **agent-tools-contract** | `samples/{read-only-good,read-only-bad,bash-no-safety}/` | 3/3 (heuristic 결정론) |
| **dag-validator** | `samples/{valid-dag,cycle,dangling}/` | 3/3 (Python stdlib) |
| **team-size-governor** | `samples/{small-team,exceeds-total,exceeds-per-phase}/` | 3/3 (bash + Python) |
| **antipattern-detector** | `samples/{clean,giant-skill,no-why}/` | 3/3 (bash 결정론) |

재검증:

```
~/.claude/skills/<skill-name>/samples 전체 감사해줘
```

---

## 의존성

- `bash` 4+
- `wc`, `grep`, `find` (POSIX)
- `python` 3+ (dag-validator, team-size-governor matrix 모드)
- Windows: Git for Windows 또는 WSL 권장. Windows App Alias로 `python3` 가 Microsoft Store로 리다이렉션되므로 각 에이전트는 `python3 || python` fallback 사용.

---

## 업그레이드

```bash
cd ~/condition-src
git pull
bash scripts/install.sh
```

---

## 플러그인 확장 규약 (기여자용)

다른 "하네스 점검" 스킬을 이 번들에 추가하려면:

1. `.claude/skills/<new-skill>/SKILL.md` 폴더로 신규 스킬 추가.
2. 전용 에이전트가 필요하면 `.claude/agents/<new-agent>.md` 로 추가.
3. 각 스킬은 자기 `samples/`, `docs/` 를 자체 폴더에 둔다 (스킬 간 자원 격리).
4. 자가검증 샘플 3건 이상 포함 (bash/regex 결정론 또는 anchor 기반 heuristic).
5. `install.sh` 는 glob 기반이라 수정 불필요.
6. README "현재 포함된 스킬" 목록 + 자가 검증 표만 갱신.

---

## 더 자세한 사용 흐름

실제 커맨드와 예상 출력이 포함된 시나리오는 [`docs/USAGE-SCENARIO.md`](./docs/USAGE-SCENARIO.md) 참조.

각 스킬의 상세 원칙·탐지 공식·제한 사항은 각 스킬 폴더의 `docs/` 에 분산되어 있다:

```
.claude/skills/<skill>/docs/*.md
```

---

## 라이선스

[MIT](./LICENSE)

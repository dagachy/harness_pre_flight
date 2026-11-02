# 설치 및 사용 시나리오

실제 흐름대로 따라 할 수 있는 "상황 → 명령 → 결과" 순서의 시나리오 모음.
Claude Code CLI(또는 claude.ai/code)가 설치되어 있다고 가정한다.

---

## 시나리오 A — 신규 설치 (처음 쓰는 환경)

**상황.** 새 노트북 또는 신규 프로젝트. `~/.claude/` 는 비어 있거나 다른 스킬만 들어 있다.

**명령.**

```bash
git clone https://github.com/dagachy/condition.git ~/condition-src
bash ~/condition-src/scripts/install.sh
```

**예상 출력.**

```
mode: copy
target: /home/<user>/.claude
installed:
  /home/<user>/.claude/agents/skill-size-auditor.md
  /home/<user>/.claude/skills/separation-signals

done.
```

**확인.**

```bash
ls ~/.claude/agents/ | grep skill-size-auditor
ls ~/.claude/skills/ | grep separation-signals
```

둘 다 보이면 설치 성공. Claude Code 세션을 **새로 시작**하면 스킬이 로드된다 (이미 열린 세션은 재시작 필요).

---

## 시나리오 B — 단일 스킬 점검 (가장 가벼운 사용)

**상황.** 작성 중인 스킬 하나가 "너무 커지지 않았나?" 의심이 든다.

**명령 (Claude Code 세션에서).**

```
/Users/me/projects/foo/.claude/skills/report-maker/SKILL.md 분리 신호 점검해줘
```

(한국어 트리거 외에도 `"separation signals audit on <path>"` 로도 호출된다.)

**예상 출력.**

```
# skill-size-auditor report
- Target: /Users/me/projects/foo/.claude/skills/report-maker/SKILL.md
- Lines: 712
- Thresholds: size=500, domain=2, conditional=1
- Signals:
  - size: yes  (근거: 712줄 / 임계값 500)
  - domain-branch: no  (근거: 순수 분기 0건 / 임계값 2)
  - conditional-detail: no  (근거: 위임 패턴 0건 / 임계값 1)
- Recommendation: Phase 또는 역할 단위로 2개 이상 스킬로 분리할 것.
```

**다음 액션.** `Recommendation` 라인을 보고 직접 분리를 진행하거나, 결정을 보류하고 다른 스킬도 함께 점검(시나리오 C)한다.

---

## 시나리오 C — 디렉토리 전체 감사 (하네스 점검 루틴)

**상황.** 프로젝트 시작 전 또는 월 1회 하네스 상태 점검. `~/.claude/skills/` 또는 프로젝트 `.claude/skills/` 전체를 한 번에 돌린다.

**명령.**

```
~/.claude/skills 전체 분리 신호 감사해줘
```

또는 프로젝트 로컬:

```
.claude/skills 전체 분리 신호 점검
```

**예상 출력.**

```
| Skill                | Lines | size | domain | conditional | Recommendation                       |
|----------------------|------:|:----:|:------:|:-----------:|--------------------------------------|
| report-maker         |   712 | yes  |   no   |     no      | Phase 단위로 분리할 것.               |
| kpi-calculator       |    83 |  no  |  yes   |     no      | 도메인별 전문 스킬로 분리할 것.        |
| data-exporter        |    76 |  no  |   no   |     yes     | 위임 블록을 조건부/라우터 스킬로.      |
| harness              |   184 |  no  |   no   |     no      |                                      |
| tdd-cycle            |   203 |  no  |   no   |     no      |                                      |

분리 권고 3건 / 전체 5건
```

**다음 액션.** 권고된 3건을 우선순위대로(도메인 > 크기 > 조건부) 분리 작업 큐에 올린다.

---

## 시나리오 D — 임계값 조정 (프로젝트 특이성)

**상황.** 우리 팀은 스킬 1건이 300줄만 넘어도 분리 신호로 간주한다 (파일당 집중 유지 정책).

**명령.**

```
SIZE_THRESHOLD=300 ~/.claude/skills 전체 분리 신호 감사해줘
```

조건부 상세를 더 엄격히 보려면:

```
CONDITIONAL_MATCH_MIN=0 ~/.claude/skills/foo/SKILL.md 분리 신호 점검
```

(0으로 두면 위임 패턴이 하나만 있어도 발현. 디버깅용.)

**예상 효과.** 리포트 상단 `Thresholds:` 라인이 override 값으로 바뀌어 출력된다. 수치만 바뀔 뿐 포맷은 동일.

---

## 시나리오 E — 설치 검증 (의심이 들 때)

**상황.** "결과가 수상한데, 플러그인이 제대로 설치된 거 맞나?" 플러그인에 포함된 샘플 3건을 돌려 bench를 확인한다.

**명령.**

```
~/.claude/skills/separation-signals/samples 전체 분리 신호 감사해줘
```

**기대 결과.**

| 샘플 | size | domain | conditional |
|------|:----:|:------:|:-----------:|
| samples/size/SKILL.md | yes | no | no |
| samples/domain/SKILL.md | no | yes | no |
| samples/conditional/SKILL.md | no | no | yes |

**해석.** 3/3 일치하면 플러그인 정상. 1건이라도 어긋나면 아래 중 하나:
- 임계값이 override 상태 → 환경변수 리셋 후 재실행
- 설치가 중간에 멈춤 → `install.sh` 재실행
- 샘플 파일이 수정됨 → `git -C ~/condition-src status` 로 확인

---

## 시나리오 E.1 — 라우터 적합성 점검 (menu-pattern 단일)

**상황.** `sql-query` 같은 분기형 라우터가 "메뉴판"이 아니라 "조리법 덩어리"로 비대해지지 않았는지 의심.

**명령.**

```
~/.claude/skills/sql-query/SKILL.md 메뉴판 원칙 점검
```

**예상 출력.**

```
# menu-pattern-auditor report
- Router: /home/<user>/.claude/skills/sql-query/SKILL.md
- References: 2건 (/home/<user>/.claude/skills/sql-query/references)
- Thresholds: max_lines=30, min_links=2
- Rules:
  - R1 size:          fail  (근거: 59줄 / 최대 30)
  - R2 link-only:     pass  (근거: 링크 2건 / 최소 2)
  - R3 no-detail:     fail  (근거: 코드블록 8 / H3+ 6)
  - R4 no-cross-ref:  fail  (근거: references 간 참조 1건)
- Verdict: PATTERN BROKEN
- Hints: 라우터에 본문·코드 섞임 — references로 추출할 것.
```

**다음 액션.** Hints 라인대로 라우터의 본문·코드블록을 references/*.md 로 밀어내고 재진단.

---

## 시나리오 E.2 — 라우터 배치 감사

**상황.** 하네스에 라우터가 여러 개(분기형 스킬이 많은 프로젝트). 월 1회 전수 점검.

**명령.**

```
~/.claude/skills 전체 메뉴판 원칙 점검
```

**예상 출력.**

```
| Skill          | Lines | R1 | R2 | R3 | R4 | Verdict         |
|----------------|------:|:--:|:--:|:--:|:--:|-----------------|
| sql-query      |    14 |pass|pass|pass|pass| COMPLIANT       |
| report-router  |    42 |fail|pass|fail|pass| PATTERN BROKEN  |
| data-exporter  |    80 |fail|fail|fail|pass| PATTERN BROKEN  |

COMPLIANT 1건 / BROKEN 2건 / 전체 3건
```

**주의.** 자체 완결형(라우터 아님) 스킬은 R2 fail이 뜰 수 있다. **라우터로 설계한 스킬만** 결과를 신뢰한다.

---

## 시나리오 F.1 — 본문 다이어트 (prose-pruner 단일)

**상황.** `separation-signals` 가 분리 불필요로 판정했지만 본문이 장황해 보인다. 3원칙으로 다이어트하고 싶다.

**명령.**

```
~/.claude/skills/report-maker/SKILL.md 3원칙 다이어트해줘
```

**예상 출력 (요약).**

```
prose-pruner 완료
- Target: /home/<user>/.claude/skills/report-maker/SKILL.md
- 산출물: /home/<user>/.claude/skills/report-maker/audit/
  - diagnosis.md       (P1=2, P2=1, P3=3, FP=0)
  - sample-after.md    (320줄 → 210줄, 34% ↓)
  - tone-conversion.md (178자 → 18자, 90% ↓)
  - diff-summary.md
```

**다음 액션.** `sample-after.md` 를 본문 교체 후보로 리뷰. 수용 시 원 `SKILL.md` 에 반영하고 `audit/` 는 삭제 또는 커밋.

---

## 시나리오 F.2 — 배치 다이어트 (prose-pruner 전체)

**상황.** 하네스 리팩터 전에 `~/.claude/skills/` 전체 본문을 1회 다이어트하여 토큰 비용을 줄이고 싶다.

**명령.**

```
~/.claude/skills 전체 3원칙 다이어트해줘
```

**예상 출력.**

```
| Skill          | Lines(before→after) | Δ% | P1 | P2 | P3 | FP | Note        |
|----------------|---------------------|---:|---:|---:|---:|---:|-------------|
| report-maker   | 320 → 210           | 34 |  2 |  1 |  3 |  0 |             |
| kpi-calculator |  83 →  80           |  3 |  0 |  0 |  0 |  0 | 변경 거의 없음 |
| data-exporter  | 155 →  95           | 38 |  1 |  0 |  2 |  0 |             |

감소 기준(≥30%) 달성 2건 / 전체 3건
```

**다음 액션.** `audit/` 폴더가 각 스킬 디렉토리 아래 생성됨. 수용할 것만 원 파일로 반영.

---

## 시나리오 F — 업그레이드

**상황.** `condition` 저장소에 신규 감사 스킬이 추가되었거나 regex 패턴이 개선됐다.

**명령.**

```bash
cd ~/condition-src
git pull
bash scripts/install.sh
```

**예상 출력 (일부).**

```
  backup: /home/<user>/.claude/agents/skill-size-auditor.md -> /home/<user>/.claude/agents/skill-size-auditor.md.bak.20261102-143012
  backup: /home/<user>/.claude/skills/separation-signals -> /home/<user>/.claude/skills/separation-signals.bak.20261102-143012
mode: copy
target: /home/<user>/.claude
installed:
  /home/<user>/.claude/agents/skill-size-auditor.md
  /home/<user>/.claude/skills/separation-signals
  /home/<user>/.claude/skills/harness-health      ← 새 스킬이 추가됐다고 가정

done.
```

**다음 액션.** `.bak.*` 파일/폴더는 검증 후 수동 삭제. 세션 재시작으로 신규 스킬 로드.

---

## 시나리오 G — 심볼릭 모드 (개발/디버깅용)

**상황.** `condition` 저장소를 직접 수정하면서 즉시 반영을 보고 싶다.

**명령.**

```bash
bash ~/condition-src/scripts/install.sh --symlink
```

**효과.** 복사 대신 심볼릭 링크로 연결. 저장소에서 파일을 수정하면 `~/.claude/` 쪽도 즉시 반영 (세션은 재시작 필요).

**주의.** Windows에서는 심볼릭 생성에 관리자 권한 또는 developer mode가 필요할 수 있다. 안 되면 복사 모드로 돌리고, 수정 후 `install.sh` 재실행하는 방식으로 대체.

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| `error: cannot read <path>` | 지정한 SKILL.md 경로가 틀렸거나 권한 없음 | 절대 경로·존재 여부 확인 |
| 모든 스킬이 all-clear로 나온다 | 임계값이 너무 높음 또는 환경변수 override 상태 | `env | grep -E 'SIZE_THRESHOLD\|DOMAIN_MATCH_MIN\|CONDITIONAL_MATCH_MIN'` 로 확인 후 unset |
| 설치 후에도 스킬이 트리거되지 않음 | Claude Code 세션이 이전 로드 상태 | 세션 재시작 또는 `/clear` 후 재시작 |
| `install.sh: command not found: bash` | Windows cmd에서 실행 | Git Bash / WSL / PowerShell에서 `bash scripts/install.sh` |
| 리포트 포맷이 예상과 다름 | 플러그인 버전 상이 | `cat ~/condition-src/VERSION` 과 README의 "출력 스키마" 재확인 |
| prose-pruner `sample-after.md` 감소율 미달 | 원 본문이 이미 간결 또는 anchor 미인식 | `MIN_REDUCTION_PCT=10` 로 완화하거나, 명시 anchor(`<!-- PRINCIPLE N VIOLATION -->`) 로 ground truth 지정 후 재진단 |
| prose-pruner 산출물이 안 생김 | `OUTPUT_DIR` 쓰기 권한 없음 또는 세션에 에이전트 미로드 | 세션 재시작 후 재실행, 또는 `OUTPUT_DIR=/tmp/prose-pruner` 로 지정 |
| menu-pattern 자체완결 스킬에서 R2 fail 발생 | 라우터 아닌 스킬에 감사기 적용 | 라우터만 입력으로 넣거나 `MENU_MIN_LINKS=0` 으로 완화 |
| menu-pattern R4가 외부 URL까지 cross-ref로 집계 | references/*.md 가 외부 URL 포함 | URL은 `.md` 확장자 매치만 보므로 영향 없음. 그래도 fail이면 `REFS_DIR` 를 `/dev/null` 로 두어 R4 자동 pass |

---

## 시나리오 H — pre-flight 전수 점검 (0.2.0 신규 6종 통합)

**상황.** 새 프로젝트 kickoff 직전. 하네스 뼈대·에이전트·구조를 신규 6종(harness-structure / agent-frontmatter / agent-tools-contract / antipattern-detector / team-size-governor / dag-validator)으로 전수 점검.

### H.1 — harness-structure (뼈대 유무)

```
<project-root> 하네스 구조 점검
```

```
# harness-structure-auditor report
- Target: /path/to/my-project
- CLAUDE.md: found
- README.md: found
- .claude/agents/: 4 files
- .claude/skills/: 6 SKILL.md
- Agent frontmatter: 4/4
- Skill frontmatter: 6/6
- Verdict: COMPLETE
- Recommendations:
  - 없음
```

Verdict가 `INCOMPLETE` 또는 `EMPTY` 이면 즉시 Recommendations대로 보완 후 다음 단계.

### H.2 — agent-frontmatter (에이전트 로드 가능성)

```
<project-root>/.claude/agents 전체 frontmatter 점검
```

```
| Agent            | Required | Recommended | Body | Verdict    |
|------------------|:--------:|:-----------:|:----:|------------|
| pr-reviewer      |  4/4     |     1/1     |  OK  | COMPLETE   |
| orchestrator     |  3/4     |     0/1     |  OK  | BROKEN     |
```

`BROKEN` 은 에이전트 로드 자체 실패 위험 — 반드시 수정.

### H.3 — agent-tools-contract (권한 적합성)

```
<project-root>/.claude/agents 전체 권한 적합성 점검
```

```
| Agent            | Role         | Tools              | Verdict         |
|------------------|--------------|--------------------|:---------------:|
| pr-reviewer      | READ_ONLY    | Read, Grep         | COMPLIANT       |
| security-auditor | READ_ONLY    | Read, Write, Bash  | OVERPRIVILEGED  |
| orchestrator     | ORCHESTRATOR | Read, Bash         | UNDERPRIVILEGED |
| builder          | UNDETERMINED | Read, Write, Bash  | RISK            |
```

- OVERPRIVILEGED: `tools:` 에서 쓰기 도구 제거.
- UNDERPRIVILEGED: SendMessage/Task 추가.
- RISK: 본문에 "git commit 금지" 조항 명시.

### H.4 — antipattern-detector (스킬 안티패턴 라벨링)

```
<project-root>/.claude/skills 전체 안티패턴 점검
```

```
| Skill          | Giant | NoRef | NoWhy | Severity | Verdict     |
|----------------|:-----:|:-----:|:-----:|:--------:|-------------|
| report-maker   |  yes  |  no   |  no   |  High    | ANTIPATTERN |
| sql-query      |  no   |  yes  |  no   |  Medium  | ANTIPATTERN |
| kpi-calculator |  no   |  no   |  yes  |  Low     | ANTIPATTERN |
| harness        |  no   |  no   |  no   |  None    | CLEAN       |

High 1 / Medium 1 / Low 1 / Clean 1 / 전체 4
```

High부터 교정 큐 상단에 배치. `separation-signals` 와 함께 돌리면 "왜 분리 + 얼마나 급한가" 둘 다 확보.

### H.5 — team-size-governor (팀 규모 거버넌스)

디렉토리 모드:

```
<project-root> 팀 규모 점검
```

```
# team-size-governor-auditor report
- Target: /path/to/my-project
- Mode: directory
- Total/Max active: 7
- Cap: 5
- Verdict: EXCEEDS_TOTAL
- Recommendations:
  - 전체 에이전트 7명 (상한 5). Phase별 활성화로 동시성 분산 권장.
```

매트릭스 모드 (Phase별 활성):

```
phases.json 팀 규모 점검
```

```
# team-size-governor-auditor report
- Mode: matrix
- Max active: 7
- Cap: 5
- Violators: B(7)
- Verdict: EXCEEDS_PER_PHASE
- Recommendations:
  - Phase B에서 상한 초과. 역할 통합 또는 Phase 분할 권장.
```

### H.6 — dag-validator (의존성 그래프)

```
tasks.json DAG 점검
```

```
# dag-validator-auditor report
- Target: /path/to/tasks.json
- Tasks: 5
- Dangling refs: 없음
- Cycle: 없음
- Parallel levels: 4  [["fetch"],["parse"],["lint","test"],["report"]]
- Verdict: VALID
- Recommendations:
  - 없음. 병렬 레벨 4개로 fan-out 가능.
```

CYCLE / DANGLING 발견 시 작업자가 블록되므로 반드시 수정 후 kickoff.

### H.7 — pre-flight 체크리스트 (한 번에)

권장 실행 순서. 각 단계 **PASS 후 다음 단계**:

```
1. harness-structure          → COMPLETE?
2. agent-frontmatter          → BROKEN 0건?
3. agent-tools-contract       → OVERPRIVILEGED 0건?
4. separation-signals         → 분리 신호 평가
5. antipattern-detector       → High 0건?
6. menu-pattern (라우터만)     → PATTERN BROKEN 0건?
7. prose-pruner (선택적)      → 토큰 비용 사전 다이어트
8. team-size-governor         → EXCEEDS_* 0건?
9. dag-validator (오케스트레이터 있을 시) → VALID?
```

전부 통과하면 프로젝트 kickoff.

---

## 흐름 요약 (한 눈에)

```
설치                 →  시나리오 A (1회)
스킬 품질 점검       →  시나리오 B (ad hoc) / 시나리오 C (루틴)
팀 정책 반영         →  시나리오 D (환경변수 override)
라우터 적합성        →  시나리오 E.1 (단일) / 시나리오 E.2 (배치)
본문 다이어트        →  시나리오 F.1 (단일) / 시나리오 F.2 (배치)
의심 검증            →  시나리오 E (샘플 3/3 체크)
pre-flight 전수 점검  →  시나리오 H.1~H.7 (0.2.0 신규)
유지보수             →  시나리오 F (주기적 git pull)
플러그인 개발        →  시나리오 G (symlink)
```

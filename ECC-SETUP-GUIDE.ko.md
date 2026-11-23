# ECC 저장소 구조 및 설치·사용 가이드 (한국어)

이 저장소는 [affaan-m/ECC](https://github.com/affaan-m/ECC)의 `main` 브랜치
(커밋 `4adeee6`, 버전 **2.2.3**, MIT 라이선스)를 그대로 복사해 온 것입니다.
원본 README는 [`README.md`](README.md), 공식 한국어 번역은 [`docs/ko-KR/README.md`](docs/ko-KR/README.md)에 있습니다.

---

## 1. ECC란?

ECC는 **AI 코딩 에이전트(주로 Claude Code)용 플러그인/하네스**입니다.
에이전트가 아래 흐름으로 일하도록 에이전트·스킬·훅·규칙을 묶어 제공합니다.

```text
plan -> test -> implement -> review -> verify -> remember -> improve
```

| 구성 요소 | 개수 | 역할 |
|---|---:|---|
| Agents | 72 | 계획(planner), 리뷰(code-reviewer), 빌드 수리, 보안, 아키텍처 등 전문 서브에이전트 |
| Skills | 328 | TDD, 리서치, 보안, 문서, 프론트엔드, 데이터/ML, 운영 등 워크플로 |
| Commands | 99 | `/plan`, `/code-review`, `/build-fix` 같은 슬래시 명령(스킬로 이전 중인 호환 진입점) |
| Hooks | — | 세션 요약 저장, 지속 학습, 도구 실행 전후 검사 등 자동화 |
| Rules | 23개 팩 | 언어/프레임워크별 항상 적용되는 코딩 규칙 |
| AgentShield | — | 프롬프트·훅·MCP 설정·권한·시크릿 보안 스캔 |

지원 하네스: Claude Code(최우선), Codex, Qoder(네이티브 플러그인), 그 외 Cursor, OpenCode,
Gemini CLI, Zed, GitHub Copilot, Antigravity, Qwen, Kimi 등(기능 제한 어댑터).

---

## 2. 디렉터리 구조

```text
.
├── agents/              # 서브에이전트 정의(.md) 72개
├── skills/              # 스킬 328개 (각 폴더에 SKILL.md)
├── commands/            # 슬래시 명령 99개
├── legacy-command-shims/# 은퇴한 짧은 명령(/tdd, /eval 등) — 명시적으로 켤 때만 사용
├── hooks/               # hooks.json(Claude), codex/qoder/antigravity용 훅, memory-persistence
├── rules/               # 규칙 팩: common + 언어별(typescript, python, golang, java, kotlin,
│                        #   rust, swift, cpp, csharp, dart, php, ruby, perl, react, vue, angular,
│                        #   nuxt, react-native, web, fsharp, arkts)
├── contexts/            # 컨텍스트 프리셋
├── mcp-configs/         # MCP 서버 설정 예시
├── manifests/           # 설치 프로필/모듈/컴포넌트 정의(install-profiles.json 등)
├── scripts/             # Node.js 설치기·CLI(ecc.js, install-apply.js, doctor.js, uninstall.js …)
├── src/, ecc2/, pi/     # 런타임/차세대(ecc2) 소스
├── schemas/             # 설정 JSON 스키마
├── tests/               # 테스트 스위트 (node tests/run-all.js)
├── docs/                # 하네스별 가이드, 다국어 번역(docs/ko-KR 포함)
├── examples/, scaffolds/, workflows/, integrations/, plugins/, docker/, config/
│
├── .claude-plugin/      # Claude Code 플러그인/마켓플레이스 매니페스트
├── .codex-plugin/ .codex/      # Codex 플러그인
├── .qoder-plugin/       # Qoder 플러그인
├── .cursor/ .gemini/ .zed/ .opencode/ .kiro/ .qwen/ .kimi/ .trae/ .agents/ … # 각 하네스 어댑터
├── .claude/             # 이 저장소 자체를 Claude Code로 열 때 쓰는 설정
├── .github/             # CI 워크플로, Copilot 지시문/프롬프트, 이슈·PR 템플릿
│
├── install.sh / install.ps1    # 소스 체크아웃용 설치 진입점 (→ scripts/install-apply.js)
├── package.json         # npm 패키지 ecc-universal (Node.js >= 18)
├── ecc_dashboard.py     # 로컬 대시보드(Python)
├── AGENTS.md / CLAUDE.md # 에이전트용 프로젝트 지침
├── COMMANDS-QUICK-REF.md # 명령 빠른 참조
├── TROUBLESHOOTING.md   # 문제 해결
└── the-shortform-guide.md / the-longform-guide.md / the-security-guide.md  # 개념 가이드
```

---

## 3. 사전 요구 사항

- **Node.js 18 이상**, **Git**
- Claude Code 사용 시 **Claude Code CLI v2.1.0 이상** (`claude --version`으로 확인)

---

## 4. 설치 방법 (하네스마다 **한 가지 방법만** 선택)

> ⚠️ 같은 하네스에 여러 방식(예: 플러그인 + 수동 full 설치)을 겹쳐 설치하면
> 스킬·훅·명령이 중복됩니다. 하나만 고르세요.

### A. 권장: 가이드형 설치 마법사 (클론 불필요)

```bash
npx ecc-universal@2.2.3 setup
```

- 범위(scope) 선택: 개인 사용이면 **Global user**, 훅 프로필은 **Standard** 추천
- 이후 새 Claude Code 세션에서 `/plugin list`로 `ecc@ecc` 활성화 확인
- 같은 명령을 다시 실행하면 업데이트/범위 변경/훅 프로필 변경

여러 하네스(Claude Code + Codex + Kimi)를 한 번에:

```bash
npx ecc-universal@2.2.3 install --guided
# 비대화식 예시
npx ecc-universal@2.2.3 install --guided \
  --harness claude --harness codex \
  --claude-scope local --claude-hooks standard --profile core --yes
```

### B. Claude Code 네이티브 플러그인 명령

Claude Code 안에서:

```text
/plugin marketplace add https://github.com/affaan-m/ECC
/plugin install ecc@ecc
```

또는 `~/.claude/settings.json`에 선언:

```json
{
  "extraKnownMarketplaces": {
    "ecc": { "source": { "source": "github", "repo": "affaan-m/ECC" } }
  },
  "enabledPlugins": { "ecc@ecc": true }
}
```

**규칙(rules)은 플러그인으로 배포되지 않으므로** 필요한 것만 직접 복사합니다
(이 저장소에서 바로 실행 가능):

```bash
mkdir -p ~/.claude/rules/ecc
cp -R rules/common ~/.claude/rules/ecc/
cp -R rules/typescript ~/.claude/rules/ecc/   # 사용하는 언어로 교체
```

### C. 이 저장소(소스 체크아웃)에서 직접 설치

```bash
./install.sh --profile <프로필> --target <대상> [--enable-hooks | --no-hooks]
# Windows: .\install.ps1 --profile minimal --target claude
```

`install.sh`는 처음 실행 시 `npm install --ignore-scripts`로 의존성을 설치한 뒤
`scripts/install-apply.js`를 호출합니다. 훅이 포함되는 프로필은
`--enable-hooks` 또는 `--no-hooks`를 명시해야 실제로 기록합니다.

**설치 프로필** (`manifests/install-profiles.json`)

| 프로필 | 설명 |
|---|---|
| `minimal` | 저컨텍스트: 규칙·에이전트·명령·플랫폼 설정, 훅 런타임 제외 |
| `core` | 최소 기본 + 훅 |
| `developer` | 대부분의 앱 개발자용 기본 프로필 |
| `security` | 보안 중심 구성 |
| `research` | 리서치·콘텐츠 작성 중심 |
| `opencode` | OpenCode 기본 구성 |
| `full` | 모든 모듈 |

**주요 대상(target) 예시**

| 하네스 | 명령 |
|---|---|
| Claude Code (훅 없이) | `./install.sh --profile minimal --target claude` |
| Claude Code (core, 훅 끔) | `./install.sh --profile core --no-hooks --target claude` |
| Cursor | `./install.sh --profile minimal --target cursor` |
| Gemini CLI | `./install.sh --profile minimal --target gemini` |
| Zed | `./install.sh --profile minimal --target zed` |
| Qwen CLI | `./install.sh --profile minimal --target qwen` |
| Kimi Code | `./install.sh --profile minimal --target kimi` |
| GitHub Copilot | `./install.sh --profile full --target copilot` |
| OpenCode | `npm install && npm run build:opencode && ./install.sh --profile full --target opencode --enable-hooks` |

### D. Codex

```bash
codex plugin marketplace add affaan-m/ECC
codex plugin add ecc@ecc
codex plugin list --json
```

Codex 안에서는 `$configure-ecc`로 설정을 조정합니다.

---

## 5. 사용 방법

필요한 워크플로부터 시작하세요. (플러그인 설치 시 `/ecc:` 접두사, 수동 설치 시 짧은 형태)

| 하고 싶은 일 | 시작 명령 |
|---|---|
| 기능 개발 | `/ecc:plan "기능 설명"` → `tdd-workflow` 스킬 |
| 버그 수정 | 실패하는 테스트로 재현 → `tdd-workflow` |
| 코드 리뷰 | `/code-review` |
| GitHub 이슈 분석 | `/ecc:issue-review <번호>` |
| 빌드 오류 수리 | `/build-fix` |
| 데드 코드 정리 | `/refactor-clean` |
| 컨텍스트 사용량 점검 | `/context-budget` |
| 긴 세션 마무리/재개 | `/save-session` · `/learn-eval` / `/resume-session` |
| 에이전트 설정 보안 감사 | `/security-scan` 또는 `agentshield scan --path .` |
| ECC 재설정 | `/ecc:configure-ecc` |

전형적인 흐름:

```text
/ecc:plan "OAuth 로그인 추가"   -> planner가 구현 설계
tdd-workflow 스킬               -> tdd-guide가 테스트 우선 강제
/code-review                    -> code-reviewer가 새 컨텍스트에서 검토
/security-scan, /test-coverage  -> 배포 전 보안·커버리지(80%+) 확인
```

전체 명령 목록은 [`COMMANDS-QUICK-REF.md`](COMMANDS-QUICK-REF.md)를 참고하세요.

---

## 6. 점검·복구·제거

```bash
# npm 패키지로 설치한 경우 (설치했던 프로젝트 디렉터리에서)
npx ecc-universal@2.2.3 list-installed
npx ecc-universal@2.2.3 doctor
npx ecc-universal@2.2.3 repair
npx ecc-universal@2.2.3 uninstall --dry-run
npx ecc-universal@2.2.3 uninstall

# 이 소스 체크아웃에서
node scripts/ecc.js list-installed
node scripts/ecc.js doctor
node scripts/ecc.js repair
node scripts/uninstall.js --dry-run
node scripts/uninstall.js
```

여러 방식을 겹쳐 설치했다면: ① Claude 플러그인 제거 → ② ECC uninstall 실행 →
③ 수동 복사한 rules 폴더 삭제 → ④ 한 가지 방식으로 재설치.

---

## 7. 개발/테스트 (이 저장소를 수정할 경우)

```bash
npm install
node tests/run-all.js                 # 전체 테스트
ECC_TEST_JOBS=4 node tests/run-all.js # 병렬 실행
```

---

## 8. 이 사본에 대한 참고 사항

- `.github/workflows/`에 원본 프로젝트의 CI·릴리스·npm 배포·Discord 공지 등 워크플로가
  그대로 포함되어 있습니다. 이 저장소에서 GitHub Actions가 동작하면 일부는 원본 전용
  시크릿이 없어 실패할 수 있으니, 필요 없으면 해당 파일을 삭제하거나 Actions를 비활성화하세요.
- 원본 업데이트를 받으려면: `git remote add upstream https://github.com/affaan-m/ECC.git`
  후 필요한 변경을 수동 반영하세요(이 사본은 원본 커밋 히스토리 없이 파일만 복사했습니다).
- 라이선스: MIT (Copyright (c) 2026 Affaan Mustafa) — [`LICENSE`](LICENSE) 유지 필요.

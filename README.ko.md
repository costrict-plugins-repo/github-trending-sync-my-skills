# sync-my-skills

[English](README.md)

Claude Code 스킬을 여러 장비에서 동기화하고 관리하는 도구입니다.

## 만들게 된 이유

Claude Code의 스킬은 로컬 파일(`~/.claude/skills/`)로만 존재하고, 계정 연동으로 다른 장비에 동기화되는 기능이 없습니다.

회사, 집, 노트북 등 여러 장비에서 Claude Code를 사용하다 보면, 한 장비에서 만들거나 수정한 스킬을 다른 장비에서도 쓰고 싶은 순간이 옵니다. 매번 수동으로 파일을 복사하거나 symlink 명령어를 치는 건 번거롭고 실수하기 쉬워서, 이 과정을 자동화하는 스킬을 만들었습니다.

## 이런 분에게 유용합니다

- 여러 장비(회사, 집, 노트북 등)에서 Claude Code를 사용하는 경우
- 직접 만든 스킬을 GitHub에 백업하고 싶은 경우
- 다른 장비에서 동일한 스킬 환경을 빠르게 구성하고 싶은 경우

## 사전 요구사항

- [Claude Code](https://claude.ai/code) 설치
- [GitHub CLI (`gh`)](https://cli.github.com/) 설치 및 인증
- Git 설치

## 설치

Claude Code plugin으로 설치:

```
/plugin marketplace add kunsanglee/sync-my-skills
/plugin install sync-my-skills
```

또는 수동 설치:

```bash
mkdir -p ~/.claude/skills/my-skills
curl -o ~/.claude/skills/my-skills/SKILL.md \
  https://raw.githubusercontent.com/kunsanglee/sync-my-skills/main/skills/my-skills/SKILL.md
```

## 사용법

Claude Code 안에서 다음 명령어를 사용합니다.

### 초기 설정

```
/my-skills setup
```

- GitHub 계정을 확인하고 private repo를 생성 (또는 기존 repo를 clone)
- 설정 파일 (`~/.my-skills.yml`)을 생성
- 다른 장비에서 이미 push한 스킬이 있으면 설치 여부를 물어봄

### 스킬 Push (로컬 → GitHub)

```
/my-skills push                            # link된 모든 user level 스킬 push
/my-skills push memo                       # 특정 스킬만 push
/my-skills push memo recap                 # 여러 스킬 동시 push
/my-skills push backend-init -p            # project level 스킬 push (프로젝트 자동 감지)
/my-skills push backend-init -p retrievo   # 프로젝트명 직접 지정
```

처음 push하는 스킬은 자동으로 repo에 복사되고 symlink로 연결됩니다.

### 스킬 Pull (GitHub → 로컬)

```
/my-skills pull                            # 모든 업데이트 pull + 새 스킬 설치 확인
/my-skills pull memo                       # 특정 스킬만 pull
/my-skills pull memo recap                 # 여러 스킬 동시 pull
/my-skills pull -p retrievo                # retrievo 스킬 목록 표시 → 선택하여 pull
/my-skills pull backend-init -p retrievo   # 특정 project 스킬 직접 pull
```

### 스킬 목록 확인

```
/my-skills list                   # 전체 스킬 상태 (user + project level)
/my-skills list -p retrievo       # 특정 프로젝트 스킬만 확인
```

출력 예시:
```
Repo: username/my-skills (private)

── User Level ──
[linked]     memo            대화를 학습 노트로 변환
[linked]     recap           세션 작업 내역 요약
[local only] backend-init    Spring Boot 프로젝트 초기화
[other]      skill-creator   스킬 생성 및 개선 (plugin 설치)

── Project Level ──
  retrievo/
    [linked]     backend-init  Spring Boot + Kotlin DDD project init
    [remote]     claude-api    Build apps with Claude API
```

### 스킬 Unlink

```
/my-skills unlink memo                  # user level 스킬 unlink
/my-skills unlink memo recap            # 여러 스킬 unlink
/my-skills unlink backend-init -p       # project level 스킬 unlink
```

symlink를 제거하고 백업에서 원본을 복원합니다. repo에서 삭제되지 않으며, 로컬 연결만 해제됩니다.

## 동작 원리

```
~/.claude/skills/memo  →  (symlink)  →  ~/.my-skills-repo/user/memo
                                              ↕ (git push/pull)
                                        GitHub: username/my-skills
                                              ↕ (git push/pull)
[다른 장비] ~/.my-skills-repo/user/memo  ←  (symlink)  ←  ~/.claude/skills/memo
```

1. 스킬을 push하면 `~/.my-skills-repo/user/` (또는 project level은 `projects/<프로젝트>/`)에 복사되고, 원본은 symlink로 교체됩니다.
2. 이후 스킬을 수정하면 symlink를 통해 repo 디렉토리의 파일이 직접 수정됩니다.
3. `/my-skills push`로 변경사항을 GitHub에 반영하고, 다른 장비에서 `/my-skills pull`로 가져옵니다.

## Repo 구조

```
~/.my-skills-repo/
├── user/                       ← user level 스킬
│   ├── memo/SKILL.md
│   └── recap/SKILL.md
└── projects/                   ← project level 스킬 (프로젝트명별)
    ├── retrievo/
    │   └── backend-init/SKILL.md
    └── execute/
        └── frontend-init/SKILL.md
```

## User Level vs Project Level

| 구분 | 경로 | 적용 범위 |
|------|------|----------|
| User level | `~/.claude/skills/` | 모든 프로젝트에서 적용 |
| Project level | `{프로젝트}/.claude/skills/` | 해당 프로젝트에서만 적용 |

- 기본 동작은 user level
- `--project` (`-p`) 옵션으로 project level 스킬도 관리 가능

## 상태 라벨

| 라벨 | 의미 |
|------|------|
| `[linked]` | repo와 symlink 연결됨 — 동기화 상태 |
| `[remote]` | repo에 있지만 이 장비에 link되지 않음 |
| `[local only]` | 로컬에만 있고 repo에 미등록 |
| `[other]` | 다른 경로를 가리키는 symlink (예: plugin 설치) |

## FAQ

**Q: 기존 스킬이 삭제되나요?**
A: 아닙니다. push 시 기존 스킬은 `~/.my-skills-backup/`에 백업된 후 symlink로 교체됩니다.

**Q: 스킬을 수정하면 어떻게 되나요?**
A: symlink를 통해 repo 디렉토리의 파일이 직접 수정되므로, `/my-skills push`만 하면 변경사항이 GitHub에 반영됩니다.

**Q: 여러 장비에서 같은 스킬을 동시에 수정하면?**
A: Git 충돌이 발생할 수 있습니다. pull 시 충돌이 감지되면 해결 방법을 안내합니다.

**Q: repo는 public인가요?**
A: 기본적으로 private으로 생성됩니다. 개인 스킬이 포함될 수 있으므로 안전합니다.

**Q: 다른 프로젝트에서 같은 이름의 스킬을 push하면?**
A: 충돌 없습니다. project level 스킬은 `projects/<프로젝트명>/` 하위에 저장되므로 같은 스킬명이 여러 프로젝트에 존재할 수 있습니다.

## 피드백 및 기능 요청

사용 중 개선 아이디어나 버그를 발견하면 GitHub Issue로 알려주세요.

- [Issue 등록](https://github.com/kunsanglee/sync-my-skills/issues)
- 기능 요청, 버그 리포트, 사용 후기 모두 환영합니다

## Author

- [@kunsanglee](https://github.com/kunsanglee)

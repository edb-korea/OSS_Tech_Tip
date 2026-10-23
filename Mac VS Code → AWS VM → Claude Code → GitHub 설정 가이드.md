# Mac VS Code → AWS VM → Claude Code → GitHub 설정 가이드

**구조:** Mac의 VS Code는 화면(UI)만 담당하고, 코드·Claude Code·git은 모두 AWS VM에서 실행됩니다. VS Code Remote-SSH가 이 둘을 연결합니다.

## 1. AWS VM 사전 준비

- 보안 그룹(Security Group)에서 22번 포트를 **내 IP만** 허용합니다.
- 키 페어(.pem)를 Mac에 저장하고 권한을 설정합니다.

```bash
mkdir -p ~/.ssh && mv ~/Downloads/my-key.pem ~/.ssh/
chmod 400 ~/.ssh/my-key.pem
```

- VM을 stop/start하면 퍼블릭 IP가 바뀝니다. **Elastic IP**를 붙여두면 편합니다.

## 2. Mac SSH 설정

`~/.ssh/config`에 추가합니다.

```
Host aws-dev
  HostName <EC2 퍼블릭 IP 또는 DNS>
  User ubuntu            # Amazon Linux면 ec2-user
  IdentityFile ~/.ssh/my-key.pem
  ForwardAgent yes       # GitHub 키를 VM에 복사하지 않고 쓰기 위함
  ServerAliveInterval 60
```

```bash
ssh aws-dev    # 접속 확인
```

## 3. VS Code 원격 연결

1. 확장 **Remote - SSH**(Microsoft)를 설치합니다.
2. `Cmd+Shift+P` → `Remote-SSH: Connect to Host…` → `aws-dev` 선택
3. 새 창이 열리면 `File > Open Folder`로 VM의 프로젝트 폴더를 엽니다.

## 4. VM에 Claude Code 설치

VS Code 터미널(VM에서 실행됨)에서:

```bash
curl -fsSL https://claude.ai/install.sh | bash
claude --version
```

프로젝트 폴더에서 `claude`를 실행해 로그인합니다. 브라우저가 없는 VM이므로 URL이 표시되면 Mac 브라우저에서 열어 인증하고, 받은 코드를 터미널에 붙여넣습니다. API 키를 쓰려면 `export ANTHROPIC_API_KEY=...`를 사용합니다.

## 5. VS Code에서 Claude Code 확장 사용 (선택)

Extensions 탭에서 **Claude Code**를 검색해 `Install in SSH: aws-dev`로 **원격 쪽에** 설치합니다. 에디터 옆 패널에서 Claude와 대화하고 diff를 바로 확인할 수 있습니다. 터미널에서 `claude`를 쓰는 방식과 병행해도 됩니다.

## 6. GitHub 인증 (3가지 중 택1)

### A. SSH Agent Forwarding (권장: VM에 비밀키가 남지 않음)

Mac에서:

```bash
ssh-keygen -t ed25519 -C "you@email.com"      # 이미 있으면 생략
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub                      # GitHub > Settings > SSH keys에 등록
```

VM에서 확인:

```bash
ssh -T git@github.com    # "Hi <username>!" 이 나오면 성공
```

### B. gh CLI

```bash
# VM에서
sudo apt install gh      # 또는 공식 설치 방법 사용
gh auth login            # GitHub.com → SSH 또는 HTTPS → 웹 인증
```

### C. 레포 전용 Deploy Key

VM에서 키를 생성한 뒤 GitHub 레포 `Settings > Deploy keys`에 **Allow write access**로 등록합니다. 레포 하나만 접근 가능해 가장 안전하지만, 레포마다 키가 필요합니다.

## 7. Git 설정 및 레포 연결

```bash
git config --global user.name "Your Name"
git config --global user.email "you@email.com"
git config --global init.defaultBranch main

# 기존 레포 clone (SSH URL 사용)
git clone git@github.com:<you>/<repo>.git

# 기존 VM 코드를 GitHub에 올리는 경우
cd ~/project
git init && git add . && git commit -m "initial"
git remote add origin git@github.com:<you>/<repo>.git
git push -u origin main
```

먼저 `.gitignore`에 `.env`, 키, 시크릿이 포함되어 있는지 확인하세요.

## 8. 권장 작업 흐름

```bash
git switch -c feature/xyz
claude                      # "OO 기능 구현하고 테스트 돌려줘"
# Claude가 수정 → 직접 diff 검토
> "변경사항 커밋하고 push해줘"   # Claude에게 요청하거나 직접 git 실행
gh pr create --fill         # PR 생성 후 GitHub에서 머지
```

- `main`에 직접 push하지 말고 브랜치와 PR을 사용하세요. GitHub branch protection을 걸어두면 안전합니다.
- 프로젝트 루트에 `CLAUDE.md`를 만들어 빌드/테스트 명령, 코딩 규칙을 적어두면 Claude가 매번 참고합니다. (`/init`으로 자동 생성)
- `git push`처럼 외부에 영향을 주는 명령은 권한 확인 프롬프트가 뜹니다. 처음에는 하나씩 승인하고, 상시 허용은 `/permissions` 또는 `.claude/settings.json`에서 설정합니다.

## 9. 운영 팁

- 연결이 끊겨도 작업이 유지되도록 `tmux`를 사용하세요. VS Code가 끊겨도 Claude 세션이 살아 있습니다.
- Agent Forwarding은 VS Code 연결 중에만 동작합니다. 연결이 끊긴 상태에서 push해야 하면 B나 C 방식을 사용하세요.
- 퍼블릭 SSH 노출이 싫다면 AWS SSM Session Manager + ProxyCommand로 22번 포트 없이 접속할 수 있습니다.

# VS Code Remote-SSH로 AWS VM 접속 후 Claude Code 실행

## 1. 사전 준비

**로컬(내 PC)**
- VS Code 설치
- SSH 키 파일(`.pem`) 준비
- Windows는 OpenSSH Client 확인: PowerShell에서 `ssh -V`

**AWS**
- EC2 보안 그룹 인바운드 22번 포트(SSH)가 내 IP에 허용되어 있어야 함
- 인스턴스 퍼블릭 IP(또는 DNS)와 사용자명 확인

| AMI | 기본 사용자 |
|---|---|
| Amazon Linux / RHEL | `ec2-user` |
| Ubuntu | `ubuntu` |
| Debian | `admin` |
| CentOS | `centos` |

## 2. 키 파일 권한 설정 (Mac/Linux)

```bash
chmod 400 ~/.ssh/mykey.pem
```

## 3. 터미널에서 접속 테스트

VS Code 전에 먼저 터미널로 확인하면 원인 파악이 빠릅니다.

```bash
ssh -v -i ~/.ssh/mykey.pem ec2-user@<VM-IP>
```

## 4. SSH config 등록

`~/.ssh/config` (Windows: `C:\Users\<이름>\.ssh\config`)

```
Host my-aws-vm
    HostName 12.34.56.78
    User ec2-user
    IdentityFile ~/.ssh/mykey.pem
    IdentitiesOnly yes
    ServerAliveInterval 60
```

- `User` 누락 시 로컬 사용자명으로 접속 시도 → `Permission denied (publickey...)` 발생
- `IdentityFile` 누락/오타 시에도 동일 오류
- Bastion 경유 시 `ProxyJump bastion-host` 추가

`-i` 없이 `ssh my-aws-vm`으로 접속되는지 확인하세요. 되면 VS Code도 됩니다.

## 5. VS Code에서 접속

1. 확장 탭에서 **Remote - SSH** (Microsoft) 설치
2. `Cmd/Ctrl + Shift + P` → **Remote-SSH: Connect to Host...**
3. `my-aws-vm` 선택
4. 새 창이 열리고 VM에 VS Code Server 자동 설치 (1~2분)
5. 좌측 하단이 `SSH: my-aws-vm`이면 접속 완료
6. **File → Open Folder**로 VM의 소스 디렉터리 열기

## 6. VM에 Claude Code 설치

VS Code 통합 터미널(`` Ctrl + ` ``)은 VM 안의 셸입니다.

```bash
# Node.js 18+ 필요 (Amazon Linux 2023)
sudo dnf install -y nodejs npm

node -v

sudo npm install -g @anthropic-ai/claude-code
```

권한 오류 시 `sudo npm install -g ...` 또는 npm 전역 경로를 홈 디렉터리로 변경하세요.

## 7. 인증 with API 키

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
echo 'export ANTHROPIC_API_KEY="sk-ant-..."' >> ~/.bashrc
```

## 8. 실행

```bash
cd ~/myproject
claude
```

## 팁

- **VS Code 확장**: 접속 상태에서 Claude Code 확장을 "Install in SSH: my-aws-vm"으로 설치하면 diff 연동 사용 가능
- **연결 끊김 대비**: `tmux new -s claude` 안에서 실행, 재접속 후 `tmux attach -t claude`
- **변경 검토**: Source Control 탭에서 git diff 확인
- **IP 변경**: EC2 재시작 시 IP가 바뀌면 Elastic IP 할당 또는 config 갱신
- **권한 모드**: 운영 서버에서는 `--dangerously-skip-permissions` 사용 금지
- **보안**: 소스가 외부 API로 전송되므로 사내 정책 확인

## 트러블슈팅

| 증상 | 해결 |
|---|---|
| `Permission denied (publickey,gssapi-keyex,gssapi-with-mic)` | config의 `User`, `IdentityFile` 확인, 키 권한 400, `IdentitiesOnly yes` 추가 |
| `Connection timed out` | 보안 그룹 22번 포트, 퍼블릭 IP, 사내 방화벽 확인 |
| VS Code Server 설치 실패 | VM outbound 인터넷 확인. 폐쇄망은 오프라인 설치 필요 |
| `claude: command not found` | `npm bin -g` 경로가 PATH에 있는지 확인, 새 터미널 열기 |
| 터미널은 되는데 VS Code만 실패 | config에 `User`/`IdentityFile` 명시, Remote.SSH: Config File 경로 확인 |

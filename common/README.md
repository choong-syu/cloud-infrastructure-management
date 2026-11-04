# 공통 준비 - 학교 Ubuntu에서 AWS 실습 시작하기

[전체 실습 목록](../README.md) · [Week 07 시작하기 →](../week07/README.md)

이 문서는 처음 한 번 수행하는 **도구 설치**와 매 수업마다 수행하는 **AWS 임시 인증**을 안내합니다.
이후 7~11주차의 명령은 학교 Ubuntu의 같은 일반 사용자 Bash에서 실행합니다.

## 0. 내 컴퓨터에서 학교 VM으로 접속

Windows 터미널 또는 macOS/Linux 터미널에서 다음 형식으로 접속합니다. `studentNN`과 `vmNN`은 교수자가 배정한 자신의 값으로 바꿉니다. 공용 비밀번호를 교안에 넣지 않으며, 입력할 때 문자가 보이지 않는 것은 정상입니다.

```text
ssh "studentNN:vmNN@gate.su-cloud.syu.ac.kr" -p 2222
```

처음 접속할 때 호스트 키 확인이 표시되면 수업에서 안내받은 서버인지 확인합니다. 이후의 `sudo`·`apt`·`terraform` 명령은 **학교 Ubuntu 안**에서 실행합니다. AWS EC2로 접속하는 단계에서는 접속 대상을 별도로 표시합니다.

## 1. 도구 설치 — 처음 한 번

준비 환경은 Ubuntu 24.04, x86_64, Python 3.12, sudo를 사용할 수 있는 일반 사용자입니다. 아래 명령에서 오류가 발생하면 다음 단계로 진행하지 말고 원인을 해결합니다.

### 01. (학교 Ubuntu · 일반 사용자) 현재 사용자와 OS 확인

```bash
id -un
id -u
. /etc/os-release
printf "%s\n" "$PRETTY_NAME"
uname -m
python3 --version
```

**명령어 풀이**

- `id -un`은 현재 계정 이름, `id -u`는 UID를 출력합니다. UID는 0이 아니어야 합니다.
- `. /etc/os-release`는 OS 정보를 읽습니다. `uname -m`은 CPU 아키텍처입니다.
- 이 교안의 다운로드 파일은 `x86_64`용이며 Python은 3.12 계열을 기준으로 합니다.

**확인할 결과**

Ubuntu 24.04, x86_64, Python 3.12를 확인합니다. 다른 환경이면 교수자에게 확인합니다.

### 02. (학교 Ubuntu · 일반 사용자) 다운로드와 Python 가상환경 도구 설치

```bash
sudo apt-get update
sudo apt-get install -y curl unzip python3-venv
```

**명령어 풀이**

- `apt-get update`는 설치 가능한 패키지 목록을 갱신합니다.
- `curl`은 파일 다운로드, `unzip`은 압축 해제에 사용합니다.
- `python3-venv`는 실습용 AWS CLI·Ansible을 시스템 Python과 구분하여 설치할 때 필요합니다.

### 03. (학교 Ubuntu · 일반 사용자) 실습 도구 폴더 만들기

```bash
mkdir -p "$HOME/.local/share/cloud-infra-tools/bin"
mkdir -p "$HOME/.config/cloud-infra"
mkdir -p "$HOME/.cache/cloud-infra/providers"
mkdir -p "$HOME/cloud-infra"
chmod 700 "$HOME/.config/cloud-infra"
chmod 700 "$HOME/cloud-infra"
cd "$HOME/.local/share/cloud-infra-tools"
```

**명령어 풀이**

- 도구는 `~/.local/share/cloud-infra-tools`, 실습 파일은 `~/cloud-infra`에 둡니다.
- `mkdir -p`는 폴더가 없으면 만들고 있으면 유지합니다.
- 인증 폴더와 실습 폴더는 `chmod 700`으로 자신만 접근할 수 있게 합니다.
- `cd`는 현재 작업 폴더를 이동합니다. 파일을 만드는 명령은 현재 위치의 영향을 받습니다.

### 04. (학교 Ubuntu · 일반 사용자) 검증한 Terraform 버전 다운로드 및 무결성 확인

```bash
curl -fSLO https://releases.hashicorp.com/terraform/1.17.1/terraform_1.17.1_linux_amd64.zip
curl -fSLO https://releases.hashicorp.com/terraform/1.17.1/terraform_1.17.1_SHA256SUMS
grep 'terraform_1.17.1_linux_amd64.zip$' terraform_1.17.1_SHA256SUMS | sha256sum -c -
```

**명령어 풀이**

- Terraform 버전은 `1.17.1`로 고정합니다.
- `curl -fSLO`는 HTTP 오류를 실패로 처리하고 원래 파일명으로 저장합니다.
- `grep`은 Linux AMD64 압축 파일의 체크섬 행을 고릅니다.
- `sha256sum -c -`는 받은 파일의 SHA-256이 공식 체크섬과 일치하는지 확인합니다.

**확인할 결과**

`terraform_1.17.1_linux_amd64.zip: OK`가 나온 뒤에만 압축을 풉니다.

### 05. (학교 Ubuntu · 일반 사용자) Terraform과 AWS CLI 설치

```bash
unzip -oq terraform_1.17.1_linux_amd64.zip -d bin
python3 -m venv venv
venv/bin/python -m pip install 'awscli==1.46.2'
```

**명령어 풀이**

- `unzip -d bin`은 Terraform을 사용자 전용 `bin` 폴더에 풉니다.
- `python3 -m venv venv`는 `venv`라는 Python 가상환경을 만듭니다.
- `venv/bin/python -m pip`는 그 가상환경에 AWS CLI를 설치합니다. 시스템 Python에 `sudo pip`를 실행하지 않습니다.

### 06. (학교 Ubuntu · 일반 사용자) 매번 불러올 환경 설정 파일 작성

```bash
cat > "$HOME/.config/cloud-infra/env.sh" <<'EOF'
export PATH="$HOME/.local/share/cloud-infra-tools/bin:$HOME/.local/share/cloud-infra-tools/venv/bin:$PATH"
export AWS_SHARED_CREDENTIALS_FILE="$HOME/.config/cloud-infra/credentials"
export AWS_PROFILE=default
export AWS_DEFAULT_REGION=us-east-1
export AWS_REGION=us-east-1
export AWS_PAGER=""
export TF_PLUGIN_CACHE_DIR="$HOME/.cache/cloud-infra/providers"
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN AWS_SECURITY_TOKEN
EOF
```

**명령어 풀이**

- `export PATH=...`는 이번 셸에서 설치한 도구를 찾게 합니다.
- `AWS_SHARED_CREDENTIALS_FILE`은 수업 전용 자격 증명 파일 위치입니다. 기존 `~/.aws/credentials`는 덮어쓰지 않습니다.
- 리전은 `us-east-1`로 통일합니다. `AWS_PAGER=""`는 결과를 별도 화면으로 열지 않게 합니다.
- Provider 캐시는 주차별로 같은 큰 파일을 반복 다운로드하는 일을 줄입니다. 같은 캐시를 사용하는 `terraform init`은 동시에 여러 개 실행하지 않습니다.
- 마지막 `unset`은 이전 셸에 남은 임시 키가 전용 파일보다 우선 사용되는 일을 막습니다.
- `<<'EOF'`의 작은따옴표를 유지합니다. `$HOME` 등은 이 파일을 나중에 읽을 때 해석됩니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat "$HOME/.config/cloud-infra/env.sh"`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 07. (학교 Ubuntu · 일반 사용자) 설치 결과 확인

```bash
source "$HOME/.config/cloud-infra/env.sh"
terraform version
aws --version
```

**명령어 풀이**

- `source`는 현재 Bash에 파일의 환경 설정을 적용합니다. 새 SSH 접속 때마다 다시 실행합니다.
- Terraform `1.17.1`, AWS CLI `1.46.2`가 표시되는지 확인합니다.

**현재 위치:** 학교 VM에서 Terraform과 AWS CLI를 실행할 준비가 되었습니다. 아직 AWS 계정 인증은 하지 않았습니다.

## 2. AWS 임시 자격 증명 — 매 수업 시작 및 만료 시

브라우저에서 자신의 Learner Lab을 열어 **Start Lab**을 누르고 Ready 상태를 기다립니다. **AWS Details → AWS CLI → Show**에서 세 값을 확인합니다. 각 값에서 `=` 오른쪽의 값만 복사합니다. 특히 Session Token은 길어도 전체가 필요합니다.

아래 명령 전체를 먼저 붙여넣어 실행합니다. 이후 나타나는 질문에 한 값씩 붙여넣고 Enter를 누릅니다. 값이 화면에 보이지 않아도 입력되고 있습니다. 명령행에 실제 키를 직접 적거나 과제 화면에 포함하지 않습니다.

### 08. (학교 Ubuntu · 일반 사용자) 세 가지 임시 인증 값을 전용 파일에 저장

```bash
{
source "$HOME/.config/cloud-infra/env.sh"
read -rp 'Access Key ID: ' ci_access; printf '\n'
read -rp 'Secret Access Key: ' ci_secret; printf '\n'
read -rp 'Session Token: ' ci_token; printf '\n'
if [ -n "$ci_access" ] && [ -n "$ci_secret" ] && [ -n "$ci_token" ]; then
  (umask 077; printf '[default]\naws_access_key_id=%s\naws_secret_access_key=%s\naws_session_token=%s\n' \
    "$ci_access" "$ci_secret" "$ci_token" > "$AWS_SHARED_CREDENTIALS_FILE")
  chmod 600 "$AWS_SHARED_CREDENTIALS_FILE"
  printf '임시 자격 증명 저장 완료\n'
else
  printf '빈 값이 있습니다. 이 단계를 처음부터 다시 수행하세요.\n'
fi
unset ci_access ci_secret ci_token
}
```

**명령어 풀이**

- `read -r`은 값을 그대로 읽고 `-s`는 화면에 숨깁니다. `-p` 뒤는 질문 문구입니다.
- `umask 077`과 `chmod 600`은 파일을 자신만 읽고 쓸 수 있게 합니다.
- `[default]`는 이 전용 파일 안에서 사용할 프로파일 이름입니다.
- 셸 임시 변수는 저장 후 `unset`으로 제거합니다.

**확인할 결과**

세 값이 모두 있을 때만 저장 완료 문장이 나옵니다. 실제 값은 출력하지 않습니다.

### 09. (학교 Ubuntu · 일반 사용자) AWS 계정과 역할 확인

```bash
aws sts get-caller-identity
printf "리전: %s\n" "$AWS_DEFAULT_REGION"
```

**명령어 풀이**

- STS는 현재 자격 증명이 어느 AWS 계정·역할인지 알려 줍니다.
- `Account`가 자신의 Learner Lab 콘솔 계정과 같은지 직접 비교합니다.
- 인증 성공이 모든 서비스의 생성 권한까지 보장하지는 않습니다.

**확인할 결과**

Account와 Arn이 나오고 리전은 us-east-1이어야 합니다. 오류가 나면 주차별 apply를 시작하지 않습니다.

### 10. (학교 Ubuntu · 일반 사용자) "$HOME/cloud-infra/.gitignore" 작성

```bash
cat > "$HOME/cloud-infra/.gitignore" <<'EOF'
**/.terraform/
**/*.tfstate
**/*.tfstate.*
**/*.tfplan
**/crash.log
**/*.pem
**/id_ed25519
**/id_ed25519.pub
**/known_hosts
**/inventory.ini
**/connection.auto.tfvars.json
EOF
```

**명령어 풀이**

- Git을 사용할 때 State·저장된 계획·개인 키·접속 정보가 커밋되지 않도록 합니다.
- `.terraform.lock.hcl`은 제외하지 않습니다. 검증한 Provider 버전을 재현하는 데 사용합니다.
- `.gitignore`는 이미 Git이 추적하던 파일을 자동으로 제거하지 않습니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat "$HOME/cloud-infra/.gitignore"`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

## 3. 명령을 따라가는 방법

- `#### 01`처럼 번호가 붙은 명령을 위에서 아래로 수행합니다. 코드 블록 안의 `$` 프롬프트는 제공하지 않으므로 그대로 복사할 수 있습니다.
- 명령 블록은 실행용이고, `text`로 표시한 출력 예시는 입력하지 않습니다.
- `cat > 파일 <<'EOF'`는 첫 줄부터 마지막 `EOF`까지 한 번에 복사합니다. 같은 파일이 있으면 덮어쓰므로 안내한 주차별 폴더인지 `pwd`로 확인합니다.
- `plan -out=이름.tfplan`은 계획을 파일로 저장합니다. 그다음 `apply 이름.tfplan`은 검토한 계획을 **추가 yes 질문 없이 적용**합니다. 예상 자원 수를 확인한 뒤 실행합니다.
- 새로 접속하면 `source ~/.config/cloud-infra/env.sh`와 인증 확인부터 다시 수행합니다. 이전 셸에서 만든 `ci_...` 변수는 사라집니다. 주차별 명령에서 필요한 값을 다시 읽습니다.
- 매주 코드는 남기고 AWS 자원은 정리합니다. 다음 주는 새 디렉터리와 새 State를 사용합니다. 한 주차를 마치기 전에 다음 주차를 동시에 만들지 않습니다.

## 막혔을 때

EC2를 중지하는 것만으로 EBS 저장 비용까지 없어지지는 않습니다. 이 교안은 수업 끝에 인스턴스와 연결 자원을 삭제하며, 루트 디스크도 함께 삭제하도록 설정합니다. 공인 IPv4도 과금 항목이므로 실습 뒤에 남겨 두지 않습니다. 참고: [EC2 상태별 과금](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html), [공인 IPv4 요금](https://aws.amazon.com/vpc/pricing/). 예산 화면이 늦게 갱신될 수 있으므로 표시 금액만으로 정리 여부를 판단하지 않습니다.

| 증상 | 확인과 해결 |
| --- | --- |
| `terraform: command not found` | `source ~/.config/cloud-infra/env.sh` 후 `command -v terraform` 확인 |
| `Unable to locate credentials` | 인증 파일 저장 단계 완료 여부와 `AWS_SHARED_CREDENTIALS_FILE` 위치 확인 |
| `ExpiredToken` / `InvalidClientTokenId` | Lab이 Ready인지 확인하고 세 값을 모두 다시 입력 |
| 콘솔과 다른 Account | apply를 멈추고 환경 설정을 다시 source한 뒤 올바른 Lab 값으로 갱신 |
| `AccessDenied` | 실행한 서비스·작업·리전 확인. IAM 역할을 임의로 만들거나 권한 정책을 변경하지 말고 오류를 교수자에게 전달 |
| 명령 이후 `>`가 계속 나타남 | here-document 마지막 `EOF` 또는 따옴표가 빠졌는지 확인. Ctrl+C 후 해당 파일 작성 블록 전체를 다시 실행 |
| plan에 예상하지 않은 삭제·교체가 보임 | `pwd`, 해당 주차 `.tf`와 `.tfvars`, `terraform state list` 확인 후 진행 |

실습 중 토큰이 만료되어도 State를 삭제하지 않습니다. 인증을 갱신하고 같은 폴더에서 다시 계획을 확인합니다. 수업 후 모든 AWS 자원 정리를 확인했다면 `rm -f "$AWS_SHARED_CREDENTIALS_FILE"`로 수업 전용 임시 자격 증명 파일을 지울 수 있습니다. 다음 수업에서는 새 값을 저장합니다.

도구 설치 참고: [Terraform 공식 설치](https://developer.hashicorp.com/terraform/install), [임시 자격 증명](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html).

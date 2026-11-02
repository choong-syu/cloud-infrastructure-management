# Week 11 · Terraform과 Ansible로 웹 서버 구성 실습 가이드

[전체 실습 목록](../README.md) · [공통 준비](../common/README.md)

[← Week 10](../week10/README.md) · [실습 목록 →](../README.md)

지난주 모듈로 EC2 두 대를 준비하고, Terraform 출력을 보고 **직접 Inventory를 작성**합니다. Ansible로 nginx와 웹 페이지를 구성한 뒤 재실행·외부 변경·복구를 확인합니다.

## 학습 목표

Terraform의 인프라 관리와 Ansible의 OS 설정 관리를 연결하고 멱등성의 실제 결과를 읽습니다. 이론 45분 · 실습 80분 · 정리 15분. 10주차 dev/test를 모두 정리하고 모듈 코드만 보관한 상태로 시작합니다.

```text
Terraform apply → 출력 IP 확인 → 사람이 Inventory 입력
  → Ansible ping → 패키지·파일·서비스 설정
  → HTTP 확인 → 같은 Playbook 재실행 → 파일 드리프트 복구 → 전체 삭제
```

웹 접속도 학교 VM의 출구 IP에만 허용합니다. **학교 VM의 curl**로 확인합니다. 다른 출구 IP를 사용하는 개인 PC 브라우저에서 곧바로 열리지 않는 것은 정상입니다.

## 실습 준비와 설정값

| 항목 | 설정값 |
| --- | --- |
| 실행 환경 | 학교 Ubuntu 24.04 · 일반 사용자 Bash |
| AWS 환경 | 본인의 AWS Academy Learner Lab |
| 리전 | 미국 동부(버지니아 북부) `us-east-1` |
| 수업 시간 | 140분 · 도구 설치는 사전에 완료 |
| 작업 폴더 | `~/cloud-infra/week11` |

[공통 준비](../common/README.md)의 도구 설치·인증을 완료합니다. 이전 주차 AWS 자원은 삭제하고 코드와 State는 보관합니다. 이번 주차 폴더에 이미 실습 파일이나 자원이 있다면 먼저 기존 실습의 정리 단계를 마칩니다.

**실행 위치는 별도 표시가 없는 한 학교 Ubuntu 일반 사용자입니다.** 아래 블록을 번호 순서대로 전체 복사합니다. EC2 안에 대화형으로 들어가서 다음 Terraform 명령을 실행하지 않도록 주의합니다.

## 실습 절차

### 01. (학교 Ubuntu · 일반 사용자) 환경 불러오기와 주차 폴더 준비

```bash
source "$HOME/.config/cloud-infra/env.sh"
aws sts get-caller-identity
mkdir -p "$HOME/cloud-infra/week11"
cd "$HOME/cloud-infra/week11"
pwd
```

**명령어 풀이**

- `source`로 도구 경로·전용 인증 파일·리전을 설정합니다.
- `Account`가 자신의 Learner Lab 콘솔 계정과 같은지 확인한 뒤 진행합니다.
- 이 주차 폴더에 `.tf` 코드와 State가 함께 저장됩니다. `sudo terraform`은 사용하지 않습니다.

**확인할 결과**

현재 경로가 `/home/자신의계정/cloud-infra/week11`입니다.

### 02. (학교 Ubuntu · 일반 사용자) EC2 접속용 키 준비

```bash
mkdir -p "$HOME/.ssh"
chmod 700 "$HOME/.ssh"
if [ ! -f "$HOME/.ssh/cloud-infra-lab" ]; then
  ssh-keygen -t ed25519 -f "$HOME/.ssh/cloud-infra-lab" -N '' -C "$(id -un)-cloud-infra"
fi
chmod 600 "$HOME/.ssh/cloud-infra-lab"
ssh-keygen -lf "$HOME/.ssh/cloud-infra-lab.pub"
```

**명령어 풀이**

- 개인 키는 학교 VM 안에 보관합니다. `.pub` 공개 키만 AWS에 등록합니다.
- `-t ed25519`는 키 형식, `-f`는 저장 위치, `-N ''`는 이 수업용 키의 암호를 비워 두는 옵션입니다.
- `if`는 이미 있는 키를 덮어쓰지 않도록 합니다. 같은 키를 다음 주에도 재사용합니다.
- 마지막 명령은 공개 키의 지문을 보여 줍니다. 개인 키 내용을 화면에 출력하지 않습니다.

**확인할 결과**

키 지문이 표시됩니다. 이 개인 키는 Learner Lab 임시 API 키와 다른 용도입니다.

### 03. (학교 Ubuntu · 일반 사용자) 검증한 모듈 코드만 재사용

```bash
mkdir -p modules
cp -R "$HOME/cloud-infra/week10/modules/web_environment" modules/
```

**명령어 풀이**

10주차 모듈 폴더에는 코드만 있습니다. environments/dev 또는 test 폴더와 State는 복사하지 않습니다. 이번 주는 새 루트·새 State로 시작합니다.

### 04. (학교 Ubuntu · 일반 사용자) providers.tf 작성

```bash
cat > providers.tf <<'EOF'
terraform {
  required_version = "= 1.17.1"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 6.71.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
  default_tags {
    tags = {
      Course = "cloud-infra"
      Week   = "11"
    }
  }
}
EOF
```

**명령어 풀이**

이번 주 태그를 11로 설정합니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat providers.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 05. (학교 Ubuntu · 일반 사용자) variables.tf 작성

```bash
cat > variables.tf <<'EOF'
variable "name" {
  type = string
}
variable "vpc_cidr" {
  type    = string
  default = "10.70.0.0/16"
}
variable "ssh_cidr" {
  type = string
}
variable "public_key_path" {
  type    = string
  default = "~/.ssh/cloud-infra-lab.pub"
}
variable "ami_id" {
  type    = string
  default = "ami-0d6dd9e8265d48906"
}
variable "lesson_label" {
  type    = string
  default = "original"
}

variable "server_count" {
  type    = number
  default = 1
  validation {
    condition     = contains([1, 2], var.server_count)
    error_message = "server_count 는 1 또는 2여야 합니다."
  }
}

variable "enable_http" {
  type    = bool
  default = false
}
EOF
```

**명령어 풀이**

10주차와 같은 입력 인터페이스를 사용합니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat variables.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 06. (학교 Ubuntu · 일반 사용자) main.tf 작성

```bash
cat > main.tf <<'EOF'
module "web_environment" {
  source          = "./modules/web_environment"
  name            = var.name
  vpc_cidr        = var.vpc_cidr
  ssh_cidr        = var.ssh_cidr
  public_key_path = var.public_key_path
  ami_id          = var.ami_id
  lesson_label    = var.lesson_label
  server_count    = var.server_count
  enable_http     = var.enable_http
}
EOF
```

**명령어 풀이**

이번 루트 위치에 맞춰 source 경로만 ./modules/web_environment로 바꿉니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat main.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 07. (학교 Ubuntu · 일반 사용자) outputs.tf 작성

```bash
cat > outputs.tf <<'EOF'
output "vpc_id" {
  value = module.web_environment.vpc_id
}
output "instance_ids" {
  value = module.web_environment.instance_ids
}
output "public_ips" {
  value = module.web_environment.public_ips
}
output "subnet_ids" {
  value = module.web_environment.subnet_ids
}
EOF
```

**명령어 풀이**

두 서버의 공인 IP와 인스턴스 ID를 조회합니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat outputs.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 08. (학교 Ubuntu · 일반 사용자) 실습 입력값 작성

```bash
cat > terraform.tfvars <<EOF
name         = "ci-$(id -un)-w11"
vpc_cidr     = "10.100.0.0/16"
lesson_label = "original"
server_count = 2
EOF
```

**명령어 풀이**

- `terraform.tfvars`는 변수에 넣을 값을 적는 파일입니다. 파일 이름이 이와 같으면 자동으로 읽습니다.
- 여기서는 따옴표 없는 `<<EOF`를 사용해 `$(id -un)`이 현재 계정 이름으로 치환되게 합니다.
- `.tf`의 변수 선언은 입력 형식을 정하고, `.tfvars`는 이번 실행의 값을 정합니다.

### 09. (학교 Ubuntu · 일반 사용자) 학교에서 HTTP 접속을 허용하도록 입력 추가

```bash
printf '
enable_http = true
' >> terraform.tfvars
```

**명령어 풀이**

조건부 HTTP 규칙을 켭니다. 허용 출구 IP는 SSH와 같은 ssh_cidr이며 인터넷 전체에 공개하지 않습니다. 이 추가 명령은 한 번만 실행합니다.

### 10. (학교 Ubuntu · 일반 사용자) 학교 VM의 접속 주소를 변수 파일로 저장

```bash
ci_source_ip=$(curl -fsS https://checkip.amazonaws.com)
python3 - "$ci_source_ip" <<'PY'
import ipaddress, json, sys
address = str(ipaddress.IPv4Address(sys.argv[1].strip()))
with open("connection.auto.tfvars.json", "w") as f:
    json.dump({"ssh_cidr": address + "/32"}, f, indent=2)
print("허용할 학교 VM 출구 주소:", address + "/32")
PY
```

**명령어 풀이**

- `$(...)`는 명령 결과를 셸 변수에 넣습니다. 여기서는 학교 VM에서 AWS로 나가는 공인 주소를 구합니다.
- Python은 IPv4 형식을 확인하고 Terraform이 자동으로 읽는 `.auto.tfvars.json` 파일을 만듭니다.
- `/32`는 해당 주소 하나만 뜻합니다. SSH를 전 세계에 열지 않습니다.
- 같은 학교 출구 주소를 공유하는 환경이면 다른 학교 사용자도 네트워크 접근은 가능하지만 로그인에는 자신의 개인 키가 필요합니다.

**확인할 결과**

자신의 실행 환경에서 조회한 IPv4/32가 나옵니다. IP가 바뀌면 이 단계부터 다시 실행하고 plan/apply합니다.

### 11. (학교 Ubuntu · 일반 사용자) 초기화·형식 정리·문법 확인

```bash
terraform init
terraform fmt -recursive
terraform validate
```

**명령어 풀이**

- `init`은 Provider와 모듈을 준비하고 `.terraform.lock.hcl`에 Provider 선택을 기록합니다.
- `fmt`는 들여쓰기를 정리합니다. `validate`는 구성의 문법과 참조를 확인하지만 AWS 생성 권한까지 확인하지는 않습니다.

**확인할 결과**

`Success! The configuration is valid.`가 표시됩니다.

### 12. (학교 Ubuntu · 일반 사용자) 처음 생성할 자원의 계획 확인

```bash
terraform plan -out=create.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 14 to add, 0 to change, 0 to destroy.

### 13. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply create.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 14. (학교 Ubuntu · 일반 사용자) 두 EC2의 상태 검사 기다리기

```bash
ci_ids=$(terraform output -json instance_ids | python3 -c 'import json,sys; print(" ".join(json.load(sys.stdin)))')
aws ec2 wait instance-status-ok --instance-ids $ci_ids
terraform output public_ips
```

**명령어 풀이**

두 인스턴스의 상태 검사를 함께 기다립니다. 다음 Inventory에는 여기 나온 현재 주소를 사용합니다. EC2를 재생성했으면 예전 주소를 재사용하지 않습니다.

### 15. (학교 Ubuntu · 일반 사용자) Ansible 설치 및 버전 확인

```bash
"$HOME/.local/share/cloud-infra-tools/venv/bin/python" -m pip install 'ansible-core==2.21.5'
ansible --version
```

**명령어 풀이**

공통 준비의 Python 가상환경에 검증 버전을 설치합니다. Ansible 제어 프로그램은 학교 VM에서 실행하며 Windows PC에 설치하지 않습니다.

### 16. (학교 Ubuntu · 일반 사용자) 출력 주소를 읽고 Inventory 직접 작성

```bash
{
read -rp '첫 번째 public_ips 주소: ' ci_web1
read -rp '두 번째 public_ips 주소: ' ci_web2
python3 - "$ci_web1" "$ci_web2" <<'PY'
import ipaddress, pathlib, sys
ips = [str(ipaddress.IPv4Address(x.strip())) for x in sys.argv[1:]]
assert len(set(ips)) == 2, "서로 다른 두 서버의 IP를 입력하세요"
key = pathlib.Path.home() / ".ssh/cloud-infra-lab"
pathlib.Path("inventory.ini").write_text(
    "[web]\n" + "".join(f"web{i+1} ansible_host={ip}\n" for i,ip in enumerate(ips))
    + f"\n[web:vars]\nansible_user=ubuntu\nansible_ssh_private_key_file={key}\n"
    + "ansible_python_interpreter=/usr/bin/python3\n")
print("inventory.ini 작성 완료")
PY
unset ci_web1 ci_web2
}
cat inventory.ini
```

**명령어 풀이**

- 블록 전체를 복사한 다음 앞서 출력한 첫 번째·두 번째 IP를 질문에 직접 입력합니다.
- Terraform 출력과 Inventory 사이를 사람이 연결하는 단계입니다. 자동 Inventory 생성 도구를 사용하지 않습니다.
- `[web]`은 서버 그룹, web1/web2는 Ansible 별칭, ansible_host는 실제 접속 주소입니다.
- `[web:vars]`는 그룹 공통 로그인 사용자·키 경로·Python 경로입니다.
- IP 형식과 중복을 검사합니다. IP를 잘못 넣으면 여기서 멈추고 같은 단계를 다시 수행합니다.
- 바깥 `{ ... }`는 입력 명령 전체를 읽은 뒤 질문을 실행하게 하여 뒤의 코드 줄을 답변으로 소비하지 않게 합니다.

**확인할 결과**

서로 다른 IP 두 개와 ubuntu 계정·개인 키 경로를 확인합니다.

### 17. (학교 Ubuntu · 일반 사용자) ansible.cfg 작성

```bash
cat > ansible.cfg <<'EOF'
[defaults]
inventory = ./inventory.ini
host_key_checking = True
retry_files_enabled = False

[ssh_connection]
ssh_args = -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=./known_hosts
EOF
```

**명령어 풀이**

현재 폴더의 Inventory를 기본값으로 사용하고 서버 키를 known_hosts에 기록합니다. 암호 입력 없이 수업용 개인 키로 연결합니다. 처음 접속을 신뢰하는 방식의 의미는 8주차와 같습니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat ansible.cfg`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 18. (학교 Ubuntu · 일반 사용자) 서버 초기 설정 완료와 Ansible 접속 확인

```bash
ansible web -m raw -a 'cloud-init status --wait'
ansible web -m ping
```

**명령어 풀이**

- `raw`는 원격 셸 명령을 실행합니다. EC2 초기 설정이 끝나 패키지 관리 잠금이 풀릴 때까지 기다립니다.
- Ansible의 `ping`은 ICMP ping이 아닙니다. SSH 로그인과 원격 Python 실행 여부를 확인합니다.
- raw 명령은 단순 명령 실행이라 CHANGED로 표시될 수 있습니다. 이후 Playbook의 멱등성과 구분합니다.

**확인할 결과**

web1과 web2 모두 SUCCESS, ping 결과 pong.

### 19. (학교 Ubuntu · 일반 사용자) site.yml 작성

```bash
cat > site.yml <<'EOF'
---
- name: Configure class web servers
  hosts: web
  become: true
  gather_facts: false
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
        cache_valid_time: 3600
        lock_timeout: 180

    - name: Publish class page
      ansible.builtin.copy:
        dest: /var/www/html/index.html
        content: |
          Cloud infrastructure class
          Managed by Ansible
        owner: root
        group: root
        mode: '0644'

    - name: Keep nginx running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Check local HTTP response
      ansible.builtin.uri:
        url: http://127.0.0.1/
        return_content: true
      register: page
      failed_when: "page.status != 200 or 'Managed by Ansible' not in (page.content | default(''))"
EOF
```

**명령어 풀이**

- `hosts: web`은 두 서버를 선택하고 `become: true`는 필요한 작업을 sudo 권한으로 수행합니다.
- `apt state: present`는 nginx 설치 상태를 요구합니다. 매번 설치 명령을 무조건 실행하는 셸 스크립트와 다릅니다.
- `cache_valid_time: 3600`은 한 시간 이내 캐시를 재사용합니다. 나중에 다시 실행하면 캐시 갱신이 changed로 나타날 수 있습니다.
- `copy`는 파일 내용·소유자·권한을 일치시킵니다. 페이지 파일 변경만으로 nginx를 재시작할 필요는 없습니다.
- `service`는 실행·자동 시작 상태를 맞춥니다.
- `uri`는 EC2 내부에서 실제 HTTP 응답을 확인합니다. `register`로 결과를 저장하고 expected 문구가 없으면 실패로 처리합니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat site.yml`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 20. (학교 Ubuntu · 일반 사용자) 문법 확인 후 첫 설정 실행

```bash
ansible-playbook --syntax-check site.yml
ansible-playbook site.yml
```

**명령어 풀이**

첫 실행은 nginx 설치와 파일 생성 때문에 changed가 나옵니다. unreachable 또는 failed가 1 이상이면 다음 단계로 넘어가지 않습니다.

**확인할 결과**

두 서버 모두 unreachable=0, failed=0. 처음 실행에서 changed가 1 이상입니다.

### 21. (학교 Ubuntu · 일반 사용자) 학교 VM에서 웹 페이지 확인

```bash
ci_web1=$(awk '/^web1 / {sub("ansible_host=", "", $2); print $2}' inventory.ini)
ci_web2=$(awk '/^web2 / {sub("ansible_host=", "", $2); print $2}' inventory.ini)
curl --fail --max-time 10 "http://$ci_web1/"
curl --fail --max-time 10 "http://$ci_web2/"
```

**명령어 풀이**

`awk`가 Inventory에서 주소를 읽고 학교 VM의 curl로 HTTP 80번 경로를 확인합니다. 서버 내부 uri 확인과 달리 SG·인터넷 경로까지 통과하는 검증입니다.

**확인할 결과**

두 응답 모두 Cloud infrastructure class / Managed by Ansible을 포함합니다.

### 22. (학교 Ubuntu · 일반 사용자) 즉시 다시 실행해 멱등성 확인

```bash
ansible-playbook site.yml
```

**명령어 풀이**

같은 Playbook을 변경 없이 바로 다시 실행합니다. 패키지 캐시 유효 시간 안에서 불필요한 변경이 없는지 확인합니다.

**확인할 결과**

web1, web2 모두 changed=0, unreachable=0, failed=0.

### 23. (학교 Ubuntu · 일반 사용자) 첫 서버에서만 파일을 수동 변경

```bash
ci_web1=$(awk '/^web1 / {sub("ansible_host=", "", $2); print $2}' inventory.ini)
ssh -i "$HOME/.ssh/cloud-infra-lab" -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=./known_hosts "ubuntu@$ci_web1" 'printf "Changed outside Ansible
" | sudo tee /var/www/html/index.html'
curl --fail --max-time 10 "http://$ci_web1/"
```

**명령어 풀이**

Playbook 파일은 바꾸지 않고 EC2 내부 파일만 수동으로 바꿉니다. Terraform 밖의 인프라 태그 변경과 달리 이번에는 Ansible이 관리하는 OS 파일의 드리프트입니다.

### 24. (학교 Ubuntu · 일반 사용자) 다시 실행해 변경 없음 확인

```bash
terraform plan
```

**명령어 풀이**

- 같은 코드를 다시 실행해도 자원이 반복 생성되지 않는지 확인합니다.
- 외부 변경·인증 만료·잘못된 폴더 때문에 다른 결과가 나올 수 있으므로 실제 문구를 읽습니다.

**확인할 결과**

`No changes. Your infrastructure matches the configuration.`

Terraform이 No changes인 이유는 이번 `.tf`가 웹 페이지 파일 내용을 관리하지 않기 때문입니다. 도구마다 관리 대상으로 선언한 범위가 다릅니다.

### 25. (학교 Ubuntu · 일반 사용자) Playbook으로 파일 복구

```bash
ansible-playbook --diff site.yml
```

**명령어 풀이**

`--diff`는 파일 변경 전후 차이를 보여 줍니다. 실제 적용 명령이므로 web1 파일은 교안 내용으로 복구됩니다. 비밀 파일을 관리할 때는 diff 출력에 주의해야 합니다.

**확인할 결과**

web1 changed=1, web2 changed=0. 두 서버 모두 failed=0, unreachable=0.

### 26. (학교 Ubuntu · 일반 사용자) 복구 후 다시 변경 없음 확인

```bash
ansible-playbook site.yml
```

**명령어 풀이**

복구까지 완료하면 두 서버 모두 원하는 상태에 다시 도달합니다.

**확인할 결과**

두 서버 모두 changed=0, failed=0, unreachable=0.

**여기까지 필수 실습 완료입니다.** 추가 시간이 있고 교수자가 안내한 경우에만 자원을 정리하기 전에 [Handler](../extension-handlers/README.md), [내부 ALB](../extension-alb/README.md), [ASG](../extension-asg/README.md)를 수행합니다. ALB·ASG는 각 확장 교안의 정리까지 완료한 뒤 아래로 돌아옵니다. 선택 실습을 하지 않으면 바로 정리합니다.

## 자원 정리 — 제출 전에 수행

이 단계는 현재 폴더의 State가 관리하는 자원을 삭제합니다. 다른 주차 폴더에서 실행하지 않습니다. `terraform destroy`는 삭제 계획과 확인을 한 명령으로 수행하는 명령입니다. 여기서는 삭제 대상을 파일로 저장해 읽은 뒤 적용하는 같은 목적의 두 단계 절차를 사용합니다.

### 27. (학교 Ubuntu · 일반 사용자) 삭제 대상 확인

```bash
terraform plan -destroy -out=cleanup.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 0 to add, 0 to change, 14 to destroy.

### 28. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply cleanup.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 29. (학교 Ubuntu · 일반 사용자) 관리 자원이 남아 있지 않은지 확인

```bash
terraform state list
```

**명령어 풀이**

- 출력에 `aws_...` 또는 `module....aws_...`로 시작하는 관리 자원이 없어야 합니다.
- 코드·잠금 파일·비어 있는 State는 보관합니다. State를 지우는 것은 AWS 자원을 삭제하는 작업이 아닙니다.
- 구성 파일을 남겨 두었기 때문에 정리 후 일반 `terraform plan`을 실행하면 다시 생성할 계획이 나오는 것이 정상입니다.

**확인할 결과**

이번 구성에서는 아무 자원 주소도 출력되지 않습니다.

## 제출 및 확인 질문

- Inventory의 그룹·별칭·접속 주소·로그인 정보를 설명합니다.
- 첫 실행, 두 번째 실행, 수동 변경 후 복구 실행의 PLAY RECAP을 비교해 제출합니다.
- 두 서버 HTTP 응답과 Terraform 정리 완료를 제출합니다.
- 웹 파일이 바뀌어도 Terraform plan이 No changes인 이유를 설명합니다.

제출 화면에는 비밀번호·Access Key·Secret Key·Session Token·개인 키·State 원문을 포함하지 않습니다. `.tf` 코드와 본인이 정리한 결과를 제출합니다. 계정 ID·IP가 포함된 화면은 수업 제출 범위에서만 사용합니다.

## 막혔을 때

| 증상 | 확인 및 해결 |
| --- | --- |
| UNREACHABLE / timeout | Inventory가 이번 apply의 IP인지, 학교 VM에서 실행하는지, SG의 현재 /32 확인 |
| Permission denied | ansible_user=ubuntu, 키 경로·권한, EC2에 등록한 공개 키 확인 |
| apt 잠금 오류 | cloud-init 완료 확인. 다른 apt 작업이 끝날 때까지 기다린 뒤 재실행 |
| Ansible ping 성공·HTTP 실패 | ping은 SSH/Python 확인일 뿐. nginx 상태, HTTP 규칙 enable_http, curl 실행 위치 확인 |
| 두 번째 실행 changed=1 | 변경된 task를 확인. 한 시간 이후 apt 캐시 갱신인지, 수동 수정이 있었는지 구분 |
| 새 EC2에서 서버 키 불일치 | 해당 IP가 정말 재생성한 EC2인지 확인한 후 그 IP의 known_hosts 항목만 수정 |
| Inventory 입력 후 오류 | 첫·둘째 주소를 따옴표나 쉼표 없이 각각 입력했는지 확인 |

공통 인증·설치 오류는 [공통 준비의 문제 해결](../common/README.md#막혔을-때)을 확인합니다. 실제 검증 범위와 남은 제한은 [검증 리포트](../VERIFICATION.md)에 기록합니다.

---

[← Week 10](../week10/README.md) · [실습 목록 →](../README.md)

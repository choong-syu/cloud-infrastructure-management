# Week 10 · 모듈 재사용과 dev/test 환경 분리 실습 가이드

[전체 실습 목록](../README.md) · [공통 준비](../common/README.md)

[← Week 09](../week09/README.md) · [Week 11 →](../week11/README.md)

지난주 네트워크와 EC2 구성을 **작은 모듈 하나**로 묶습니다. dev와 test는 같은 모듈 코드를 사용하지만 서로 다른 루트 디렉터리와 State를 갖습니다.

## 학습 목표

재사용하는 코드와 관리 대상의 경계를 구분하고, dev 변경·삭제가 test를 바꾸지 않는지 확인합니다. 이론 55분 · 실습 70분 · 정리 15분. 9주차 자원을 먼저 삭제합니다. 실행 중인 자원을 모듈로 이동하는 실습은 아닙니다.

```text
week10/
├── modules/web_environment/  ← 재사용할 코드만 보관
└── environments/
    ├── dev/                 ← Terraform 실행 위치와 dev State
    └── test/                ← Terraform 실행 위치와 test State
```

`.tfvars` 파일만 바꿔 같은 폴더에서 실행하면 독립된 State가 생기지 않습니다. 이번에는 루트 디렉터리 자체를 분리합니다. 같은 모듈을 수정하면 두 환경의 다음 plan에 영향을 줄 수 있다는 점도 기억합니다.

## 실습 준비와 설정값

| 항목 | 설정값 |
| --- | --- |
| 실행 환경 | 학교 Ubuntu 24.04 · 일반 사용자 Bash |
| AWS 환경 | 본인의 AWS Academy Learner Lab |
| 리전 | 미국 동부(버지니아 북부) `us-east-1` |
| 수업 시간 | 140분 · 도구 설치는 사전에 완료 |
| 작업 폴더 | `~/cloud-infra/week10` |

[공통 준비](../common/README.md)의 도구 설치·인증을 완료합니다. 이전 주차 AWS 자원은 삭제하고 코드와 State는 보관합니다. 이번 주차 폴더에 이미 실습 파일이나 자원이 있다면 먼저 기존 실습의 정리 단계를 마칩니다.

**실행 위치는 별도 표시가 없는 한 학교 Ubuntu 일반 사용자입니다.** 아래 블록을 번호 순서대로 전체 복사합니다. EC2 안에 대화형으로 들어가서 다음 Terraform 명령을 실행하지 않도록 주의합니다.

## 실습 절차

### 01. (학교 Ubuntu · 일반 사용자) 환경 불러오기와 주차 폴더 준비

```bash
source "$HOME/.config/cloud-infra/env.sh"
aws sts get-caller-identity
mkdir -p "$HOME/cloud-infra/week10"
cd "$HOME/cloud-infra/week10"
pwd
```

**명령어 풀이**

- `source`로 도구 경로·전용 인증 파일·리전을 설정합니다.
- `Account`가 자신의 Learner Lab 콘솔 계정과 같은지 확인한 뒤 진행합니다.
- 이 주차 폴더에 `.tf` 코드와 State가 함께 저장됩니다. `sudo terraform`은 사용하지 않습니다.

**확인할 결과**

현재 경로가 `/home/자신의계정/cloud-infra/week10`입니다.

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

### 03. (학교 Ubuntu · 일반 사용자) 모듈·환경 디렉터리 만들기

```bash
mkdir -p modules/web_environment environments/dev environments/test
```

**명령어 풀이**

모듈 폴더에서는 apply하지 않습니다. 실제 실행은 각각의 environments/dev, environments/test에서 수행합니다.

### 04. (학교 Ubuntu · 일반 사용자) modules/web_environment/versions.tf 작성

```bash
cat > modules/web_environment/versions.tf <<'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 6.71.0"
    }
  }
}
EOF
```

**명령어 풀이**

자식 모듈이 필요로 하는 Provider를 선언합니다. 리전·인증 설정은 루트의 provider 블록을 상속합니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat modules/web_environment/versions.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 05. (학교 Ubuntu · 일반 사용자) modules/web_environment/variables.tf 작성

```bash
cat > modules/web_environment/variables.tf <<'EOF'
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

모듈의 입력 인터페이스입니다. 서버 수·이름·CIDR·접속 허용 주소를 루트에서 전달합니다. enable_http는 기본 false이며 다음 주 웹 서버 수업에서만 켭니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat modules/web_environment/variables.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 06. (학교 Ubuntu · 일반 사용자) modules/web_environment/network.tf 작성

```bash
cat > modules/web_environment/network.tf <<'EOF'
resource "aws_vpc" "lab" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags = {
    Name   = var.name
    Lesson = var.lesson_label
  }
}

resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.lab.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = ["us-east-1a", "us-east-1b"][count.index]
  map_public_ip_on_launch = true
  tags = { Name = "${var.name}-public-${count.index + 1}" }
}

resource "aws_internet_gateway" "lab" {
  vpc_id = aws_vpc.lab.id
  tags   = { Name = var.name }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.lab.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.lab.id
  }
  tags = { Name = "${var.name}-public" }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "web" {
  name_prefix = "${var.name}-"
  description = "Cloud infrastructure class"
  vpc_id      = aws_vpc.lab.id
  tags        = { Name = var.name }
}

resource "aws_vpc_security_group_ingress_rule" "ssh" {
  security_group_id = aws_security_group.web.id
  cidr_ipv4         = var.ssh_cidr
  from_port         = 22
  to_port           = 22
  ip_protocol       = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "outbound" {
  security_group_id = aws_security_group.web.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}

resource "aws_vpc_security_group_ingress_rule" "http" {
  count             = var.enable_http ? 1 : 0
  security_group_id = aws_security_group.web.id
  cidr_ipv4         = var.ssh_cidr
  from_port         = 80
  to_port           = 80
  ip_protocol       = "tcp"
}
EOF
```

**명령어 풀이**

9주차의 두 서브넷 네트워크를 재사용합니다. HTTP 규칙은 조건식 `var.enable_http ? 1 : 0`에 따라 1개 또는 0개 생성합니다. 기본값 false에서는 HTTP 규칙을 만들지 않습니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat modules/web_environment/network.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 07. (학교 Ubuntu · 일반 사용자) modules/web_environment/compute.tf 작성

```bash
cat > modules/web_environment/compute.tf <<'EOF'
resource "aws_key_pair" "lab" {
  key_name_prefix = "${var.name}-"
  public_key      = file(pathexpand(var.public_key_path))
}

resource "aws_instance" "web" {
  count                       = var.server_count
  ami                         = var.ami_id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public[count.index].id
  vpc_security_group_ids      = [aws_security_group.web.id]
  key_name                    = aws_key_pair.lab.key_name
  iam_instance_profile        = "LabInstanceProfile"
  associate_public_ip_address = true

  root_block_device {
    volume_type           = "gp3"
    volume_size           = 8
    encrypted             = true
    delete_on_termination = true
  }
  metadata_options {
    http_tokens = "required"
  }
  tags = { Name = "${var.name}-web-${count.index + 1}" }
  depends_on = [aws_route_table_association.public]
}
EOF
```

**명령어 풀이**

서버 생성 코드는 9주차의 count 구조를 유지합니다. 공통 모듈 내부에 특정 환경의 이름이나 CIDR을 고정하지 않습니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat modules/web_environment/compute.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 08. (학교 Ubuntu · 일반 사용자) modules/web_environment/outputs.tf 작성

```bash
cat > modules/web_environment/outputs.tf <<'EOF'
output "vpc_id" {
  value = aws_vpc.lab.id
}
output "instance_ids" {
  value = aws_instance.web[*].id
}
output "public_ips" {
  value = aws_instance.web[*].public_ip
}
output "subnet_ids" {
  value = aws_subnet.public[*].id
}
EOF
```

**명령어 풀이**

루트에서 사용할 결과만 모듈 출력으로 제공합니다. 루트는 module.web_environment.public_ips 같은 형태로 읽습니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat modules/web_environment/outputs.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 09. (학교 Ubuntu · 일반 사용자) environments/dev/providers.tf 작성

```bash
cat > environments/dev/providers.tf <<'EOF'
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
      Week   = "10"
    }
  }
}
EOF
```

**명령어 풀이**

루트에서 실제 Provider 리전과 공통 태그를 설정합니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat environments/dev/providers.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 10. (학교 Ubuntu · 일반 사용자) environments/dev/variables.tf 작성

```bash
cat > environments/dev/variables.tf <<'EOF'
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

루트도 외부에서 입력받을 변수를 선언해야 합니다. 루트 변수와 자식 모듈 변수는 자동 연결되지 않습니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat environments/dev/variables.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 11. (학교 Ubuntu · 일반 사용자) environments/dev/main.tf 작성

```bash
cat > environments/dev/main.tf <<'EOF'
module "web_environment" {
  source          = "../../modules/web_environment"
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

`source`는 이 루트 기준 상대 경로입니다. 오른쪽 var 값은 루트 입력이며, 왼쪽 인자는 자식 모듈에 전달할 입력 이름입니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat environments/dev/main.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 12. (학교 Ubuntu · 일반 사용자) environments/dev/outputs.tf 작성

```bash
cat > environments/dev/outputs.tf <<'EOF'
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

자식 모듈 출력을 루트 출력으로 다시 내보내야 루트에서 terraform output으로 볼 수 있습니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat environments/dev/outputs.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 13. (학교 Ubuntu · 일반 사용자) test 루트에 구성 코드만 복사

```bash
cp environments/dev/providers.tf environments/test/
cp environments/dev/variables.tf environments/test/
cp environments/dev/main.tf environments/test/
cp environments/dev/outputs.tf environments/test/
```

**명령어 풀이**

복사 대상을 네 개의 코드 파일로 한정합니다. `.terraform`, `terraform.tfstate`, 저장된 계획, 자격 증명을 다른 환경으로 복사하지 않습니다.

### 14. (학교 Ubuntu · 일반 사용자) 실습 입력값 작성

```bash
cat > environments/dev/terraform.tfvars <<EOF
name         = "ci-$(id -un)-dev"
vpc_cidr     = "10.80.0.0/16"
lesson_label = "original"
server_count = 1
EOF
```

**명령어 풀이**

- `terraform.tfvars`는 변수에 넣을 값을 적는 파일입니다. 파일 이름이 이와 같으면 자동으로 읽습니다.
- 여기서는 따옴표 없는 `<<EOF`를 사용해 `$(id -un)`이 현재 계정 이름으로 치환되게 합니다.
- `.tf`의 변수 선언은 입력 형식을 정하고, `.tfvars`는 이번 실행의 값을 정합니다.

### 15. (학교 Ubuntu · 일반 사용자) 실습 입력값 작성

```bash
cat > environments/test/terraform.tfvars <<EOF
name         = "ci-$(id -un)-test"
vpc_cidr     = "10.90.0.0/16"
lesson_label = "original"
server_count = 1
EOF
```

**명령어 풀이**

- `terraform.tfvars`는 변수에 넣을 값을 적는 파일입니다. 파일 이름이 이와 같으면 자동으로 읽습니다.
- 여기서는 따옴표 없는 `<<EOF`를 사용해 `$(id -un)`이 현재 계정 이름으로 치환되게 합니다.
- `.tf`의 변수 선언은 입력 형식을 정하고, `.tfvars`는 이번 실행의 값을 정합니다.

### 16. (학교 Ubuntu · 일반 사용자) dev 루트로 이동

```bash
cd "$HOME/cloud-infra/week10/environments/dev"
pwd
```

**명령어 풀이**

이제 dev의 코드·입력·State만 대상으로 Terraform을 실행합니다.

### 17. (학교 Ubuntu · 일반 사용자) 학교 VM의 접속 주소를 변수 파일로 저장

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

### 18. (학교 Ubuntu · 일반 사용자) 초기화·형식 정리·문법 확인

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

### 19. (학교 Ubuntu · 일반 사용자) 처음 생성할 자원의 계획 확인

```bash
terraform plan -out=create.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 12 to add, 0 to change, 0 to destroy.

### 20. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply create.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 21. (학교 Ubuntu · 일반 사용자) dev 자원 ID 기록

```bash
terraform output -json > dev-before.json
terraform state list
```

**명령어 풀이**

자원 주소가 `module.web_environment.aws_...`로 시작하는지 확인합니다. 모듈은 별도의 AWS 서비스가 아니라 코드 구성 단위입니다.

### 22. (학교 Ubuntu · 일반 사용자) test 루트로 이동

```bash
cd "$HOME/cloud-infra/week10/environments/test"
pwd
```

**명령어 풀이**

다른 디렉터리로 이동하면 다른 로컬 State를 사용합니다. 이후 plan/apply가 어느 환경인지 pwd로 다시 확인합니다.

### 23. (학교 Ubuntu · 일반 사용자) 학교 VM의 접속 주소를 변수 파일로 저장

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

### 24. (학교 Ubuntu · 일반 사용자) 초기화·형식 정리·문법 확인

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

### 25. (학교 Ubuntu · 일반 사용자) 처음 생성할 자원의 계획 확인

```bash
terraform plan -out=create.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 12 to add, 0 to change, 0 to destroy.

### 26. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply create.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 27. (학교 Ubuntu · 일반 사용자) test ID 보관과 환경 차이 확인

```bash
terraform output -json > test-before.json
python3 - <<'PY'
import json
dev = json.load(open("../dev/dev-before.json"))
test = json.load(open("test-before.json"))
assert dev["vpc_id"]["value"] != test["vpc_id"]["value"]
assert dev["instance_ids"]["value"] != test["instance_ids"]["value"]
print("dev/test의 VPC와 EC2가 서로 다름")
PY
```

**명령어 풀이**

모듈 코드가 같아도 각 루트에서 별개의 AWS 자원을 생성했는지 ID로 비교합니다.

### 28. (학교 Ubuntu · 일반 사용자) dev의 태그 입력만 변경

```bash
cd "$HOME/cloud-infra/week10/environments/dev"
sed -i 's/lesson_label = "original"/lesson_label = "dev-updated"/' terraform.tfvars
```

**명령어 풀이**

공유 모듈 코드가 아닌 dev 입력 파일을 바꿉니다. test 입력은 original 그대로입니다.

### 29. (학교 Ubuntu · 일반 사용자) 변경 계획 확인

```bash
terraform plan -out=dev-tag.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

0 to add, 1 to change, 0 to destroy. dev의 VPC 태그만 바뀝니다.

### 30. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply dev-tag.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 31. (학교 Ubuntu · 일반 사용자) test로 돌아와 영향 확인

```bash
cd "$HOME/cloud-infra/week10/environments/test"
```

**명령어 풀이**

현재 디렉터리를 test로 바꾼 뒤 별도로 plan을 실행합니다.

### 32. (학교 Ubuntu · 일반 사용자) 다시 실행해 변경 없음 확인

```bash
terraform plan
```

**명령어 풀이**

- 같은 코드를 다시 실행해도 자원이 반복 생성되지 않는지 확인합니다.
- 외부 변경·인증 만료·잘못된 폴더 때문에 다른 결과가 나올 수 있으므로 실제 문구를 읽습니다.

**확인할 결과**

`No changes. Your infrastructure matches the configuration.`

### 33. (학교 Ubuntu · 일반 사용자) dev 정리 위치로 이동

```bash
cd "$HOME/cloud-infra/week10/environments/dev"
```

**명령어 풀이**

dev만 먼저 삭제하고 test가 유지되는지 실제로 조회합니다.

## 자원 정리 — 제출 전에 수행

이 단계는 현재 폴더의 State가 관리하는 자원을 삭제합니다. 다른 주차 폴더에서 실행하지 않습니다. `terraform destroy`는 삭제 계획과 확인을 한 명령으로 수행하는 명령입니다. 여기서는 삭제 대상을 파일로 저장해 읽은 뒤 적용하는 같은 목적의 두 단계 절차를 사용합니다.

### 34. (학교 Ubuntu · 일반 사용자) 삭제 대상 확인

```bash
terraform plan -destroy -out=cleanup.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 0 to add, 0 to change, 12 to destroy.

### 35. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply cleanup.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 36. (학교 Ubuntu · 일반 사용자) 관리 자원이 남아 있지 않은지 확인

```bash
terraform state list
```

**명령어 풀이**

- 출력에 `aws_...` 또는 `module....aws_...`로 시작하는 관리 자원이 없어야 합니다.
- 코드·잠금 파일·비어 있는 State는 보관합니다. State를 지우는 것은 AWS 자원을 삭제하는 작업이 아닙니다.
- 구성 파일을 남겨 두었기 때문에 정리 후 일반 `terraform plan`을 실행하면 다시 생성할 계획이 나오는 것이 정상입니다.

**확인할 결과**

이번 구성에서는 아무 자원 주소도 출력되지 않습니다.

### 37. (학교 Ubuntu · 일반 사용자) dev 삭제 후 test가 살아 있는지 확인

```bash
cd "$HOME/cloud-infra/week10/environments/test"
ci_test_id=$(terraform output -json instance_ids | python3 -c 'import json,sys; print(json.load(sys.stdin)[0])')
aws ec2 describe-instances --instance-ids "$ci_test_id" --query 'Reservations[0].Instances[0].{ID:InstanceId,State:State.Name}'
terraform output -json > test-after.json
python3 - <<'PY'
import json
assert json.load(open("test-before.json")) == json.load(open("test-after.json"))
print("test 출력 ID와 주소 유지 확인")
PY
```

**명령어 풀이**

State의 ID 비교에 더해 AWS API로 test EC2가 실제 running인지 확인합니다. dev 삭제로 test를 삭제하면 안 됩니다.

### 38. (학교 Ubuntu · 일반 사용자) 다시 실행해 변경 없음 확인

```bash
terraform plan
```

**명령어 풀이**

- 같은 코드를 다시 실행해도 자원이 반복 생성되지 않는지 확인합니다.
- 외부 변경·인증 만료·잘못된 폴더 때문에 다른 결과가 나올 수 있으므로 실제 문구를 읽습니다.

**확인할 결과**

`No changes. Your infrastructure matches the configuration.`

## 자원 정리 — 제출 전에 수행

이 단계는 현재 폴더의 State가 관리하는 자원을 삭제합니다. 다른 주차 폴더에서 실행하지 않습니다. `terraform destroy`는 삭제 계획과 확인을 한 명령으로 수행하는 명령입니다. 여기서는 삭제 대상을 파일로 저장해 읽은 뒤 적용하는 같은 목적의 두 단계 절차를 사용합니다.

### 39. (학교 Ubuntu · 일반 사용자) 삭제 대상 확인

```bash
terraform plan -destroy -out=cleanup.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 0 to add, 0 to change, 12 to destroy.

### 40. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply cleanup.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 41. (학교 Ubuntu · 일반 사용자) 관리 자원이 남아 있지 않은지 확인

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

- 모듈 입력→내부 자원→모듈 출력→루트 출력 흐름을 예로 설명합니다.
- dev/test의 서로 다른 VPC·EC2 ID, dev 태그 변경 계획, test No changes를 제출합니다.
- dev 삭제 후 test running 조회와 마지막 두 환경 모두의 빈 State를 제출합니다.
- `.tfvars`만 두 개 만드는 것과 루트 디렉터리를 나누는 것의 차이를 설명합니다.

제출 화면에는 비밀번호·Access Key·Secret Key·Session Token·개인 키·State 원문을 포함하지 않습니다. `.tf` 코드와 본인이 정리한 결과를 제출합니다. 계정 ID·IP가 포함된 화면은 수업 제출 범위에서만 사용합니다.

## 막혔을 때

| 증상 | 해결 |
| --- | --- |
| Module not installed | 해당 dev/test 루트에서 terraform init 실행 |
| Unsupported argument | 루트에서 모듈에 전달한 이름이 modules/.../variables.tf 선언과 일치하는지 확인 |
| No outputs found | 모듈 출력뿐 아니라 루트 outputs.tf가 있는지, 그 루트에서 apply했는지 확인 |
| test 계획에 예상하지 않은 교체 | pwd와 test 입력 이름·CIDR을 확인. dev State를 복사하지 않았는지 확인 |
| 두 환경 모두 변경 계획 발생 | 공통 모듈을 바꿨는지 확인. 이번 실습은 dev의 tfvars 태그만 수정 |

공통 인증·설치 오류는 [공통 준비의 문제 해결](../common/README.md#막혔을-때)을 확인합니다. 실제 검증 범위와 남은 제한은 [검증 리포트](../VERIFICATION.md)에 기록합니다.

---

[← Week 09](../week09/README.md) · [Week 11 →](../week11/README.md)

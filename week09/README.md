# Week 09 · count·for_each와 State·드리프트 관리 실습 가이드

[전체 실습 목록](../README.md) · [공통 준비](../common/README.md)

[← Week 08](../week08/README.md) · [Week 10 →](../week10/README.md)

이번에는 서로 다른 AZ에 서브넷 두 개를 만들고 **EC2 1대 → 2대 → 1대**로 변경합니다. 이어 Terraform 밖에서 태그를 바꾼 뒤 차이를 발견하고 코드 값으로 복구합니다.

## 학습 목표

`count.index`·리스트·자원 주소를 읽고, 생성·수정·삭제·교체 계획을 구분합니다. 이어 이름이 있는 서브넷을 `for_each`로 만들어 두 방식의 식별 기준을 비교합니다. **A부와 B부 모두 필수**입니다.

140분 배분: 이론·코드 읽기 40분 · A부 count/State/드리프트 60분 · B부 for_each 비교 25분 · 두 구성 정리와 결과 확인 15분. 기존 시간에 실습을 추가하는 대신 이론 시간을 조정했습니다. 수업 운영 시간은 추정이며 사전 설치를 전제로 합니다.

8주차 자원은 삭제한 상태로 시작합니다. A부 자원을 정리한 다음 B부를 별도의 폴더·State에서 실행합니다. 실행 중인 count 자원을 for_each로 이전하는 작업은 [선택 심화](../extension-state-migration/README.md)로 분리합니다.

## 실습 준비와 설정값

| 항목 | 설정값 |
| --- | --- |
| 실행 환경 | 학교 Ubuntu 24.04 · 일반 사용자 Bash |
| AWS 환경 | 본인의 AWS Academy Learner Lab |
| 리전 | 미국 동부(버지니아 북부) `us-east-1` |
| 수업 시간 | 140분 · 도구 설치는 사전에 완료 |
| 작업 폴더 | `~/cloud-infra/week09` |

[공통 준비](../common/README.md)의 도구 설치·인증을 완료합니다. 이전 주차 AWS 자원은 삭제하고 코드와 State는 보관합니다. 이번 주차 폴더에 이미 실습 파일이나 자원이 있다면 먼저 기존 실습의 정리 단계를 마칩니다.

**실행 위치는 별도 표시가 없는 한 학교 Ubuntu 일반 사용자입니다.** 아래 블록을 번호 순서대로 전체 복사합니다. EC2 안에 대화형으로 들어가서 다음 Terraform 명령을 실행하지 않도록 주의합니다.

## A. count와 State·드리프트

### 01. (학교 Ubuntu · 일반 사용자) 환경 불러오기와 주차 폴더 준비

```bash
source "$HOME/.config/cloud-infra/env.sh"
aws sts get-caller-identity
mkdir -p "$HOME/cloud-infra/week09"
cd "$HOME/cloud-infra/week09"
pwd
```

**명령어 풀이**

- `source`로 도구 경로·전용 인증 파일·리전을 설정합니다.
- `Account`가 자신의 Learner Lab 콘솔 계정과 같은지 확인한 뒤 진행합니다.
- 이 주차 폴더에 `.tf` 코드와 State가 함께 저장됩니다. `sudo terraform`은 사용하지 않습니다.

**확인할 결과**

현재 경로가 `/home/자신의계정/cloud-infra/week09`입니다.

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

### 03. (학교 Ubuntu · 일반 사용자) providers.tf 작성

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
      Week   = "09"
    }
  }
}
EOF
```

**명령어 풀이**

같은 버전을 사용하고 주차 태그를 09로 구분합니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat providers.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 04. (학교 Ubuntu · 일반 사용자) variables.tf 작성

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
EOF
```

**명령어 풀이**

8주차 변수에 `server_count`를 추가합니다. 기본값은 1이고 validation으로 1 또는 2만 허용합니다. 학생별 인스턴스 수가 불필요하게 늘어나는 것을 막습니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat variables.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 05. (학교 Ubuntu · 일반 사용자) 실습 입력값 작성

```bash
cat > terraform.tfvars <<EOF
name         = "ci-$(id -un)-w09"
vpc_cidr     = "10.70.0.0/16"
lesson_label = "original"
server_count = 1
EOF
```

**명령어 풀이**

- `terraform.tfvars`는 변수에 넣을 값을 적는 파일입니다. 파일 이름이 이와 같으면 자동으로 읽습니다.
- 여기서는 따옴표 없는 `<<EOF`를 사용해 `$(id -un)`이 현재 계정 이름으로 치환되게 합니다.
- `.tf`의 변수 선언은 입력 형식을 정하고, `.tfvars`는 이번 실행의 값을 정합니다.

### 06. (학교 Ubuntu · 일반 사용자) 학교 VM의 접속 주소를 변수 파일로 저장

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

### 07. (학교 Ubuntu · 일반 사용자) network.tf 작성

```bash
cat > network.tf <<'EOF'
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
EOF
```

**명령어 풀이**

- `count = 2`는 같은 자원을 두 개 만듭니다. 주소는 `[0]`, `[1]`로 구분합니다.
- `count.index`는 0부터 시작합니다. 서브넷 CIDR은 10.70.0.0/24, 10.70.1.0/24가 됩니다.
- AZ 리스트에서 같은 인덱스를 골라 us-east-1a와 us-east-1b에 배치합니다. AZ 이름은 계정별 물리 위치를 동일하게 보장하지는 않습니다.
- Route Table association도 두 개를 만들어 각 서브넷에 연결합니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat network.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 08. (학교 Ubuntu · 일반 사용자) compute.tf 작성

```bash
cat > compute.tf <<'EOF'
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

- EC2의 `count = var.server_count`를 바꾸면 관리할 대수가 달라집니다.
- 첫 서버는 서브넷 `[0]`, 둘째는 `[1]`을 참조합니다.
- Name 태그는 index에 1을 더해 web-1, web-2로 읽기 쉽게 표시하지만 Terraform 주소는 계속 `[0]`, `[1]`입니다.
- 목록 중간의 항목을 지우거나 순서를 바꾸면 인덱스 기반 자원 대응이 달라질 수 있습니다. 이번에는 마지막 서버만 증감합니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat compute.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 09. (학교 Ubuntu · 일반 사용자) outputs.tf 작성

```bash
cat > outputs.tf <<'EOF'
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

`[*]`는 여러 인스턴스의 속성을 리스트로 모읍니다. 문자열 하나였던 지난주 출력과 형식이 달라집니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat outputs.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 10. (학교 Ubuntu · 일반 사용자) 초기화·형식 정리·문법 확인

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

### 11. (학교 Ubuntu · 일반 사용자) 처음 생성할 자원의 계획 확인

```bash
terraform plan -out=create.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 12 to add, 0 to change, 0 to destroy.

### 12. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply create.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 13. (학교 Ubuntu · 일반 사용자) 자원 주소와 두 AZ 확인

```bash
terraform state list
terraform state show 'aws_instance.web[0]'
ci_subnets=$(terraform output -json subnet_ids | python3 -c 'import json,sys; print(" ".join(json.load(sys.stdin)))')
aws ec2 describe-subnets --subnet-ids $ci_subnets --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}'
terraform output -json instance_ids > first-instance.json
```

**명령어 풀이**

- 대괄호가 있는 자원 주소는 셸 해석을 막기 위해 작은따옴표로 감쌉니다.
- `output -json`은 프로그램이 읽기 좋은 JSON으로 출력합니다.
- `$ci_subnets`는 AWS CLI에 여러 ID를 별도 인자로 전달하기 위해 이 위치에서 의도적으로 따옴표 없이 사용합니다. 값의 출처는 Terraform ID 리스트입니다.
- 첫 EC2 ID를 파일로 저장해 증감 후에도 유지되는지 비교합니다.

**확인할 결과**

서브넷 두 개가 서로 다른 AZ와 /24 CIDR을 가집니다.

### 14. (학교 Ubuntu · 일반 사용자) 서버 수를 2로 변경

```bash
sed -i 's/server_count = 1/server_count = 2/' terraform.tfvars
```

**명령어 풀이**

반복 자원 블록을 복사하지 않고 입력값 하나만 바꿉니다.

### 15. (학교 Ubuntu · 일반 사용자) 변경 계획 확인

```bash
terraform plan -out=scale-out.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 1 to add, 0 to change, 0 to destroy. 추가 주소는 aws_instance.web[1]입니다.

### 16. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply scale-out.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 17. (학교 Ubuntu · 일반 사용자) 두 서버의 ID 확인

```bash
terraform output instance_ids
terraform output public_ips
```

**명령어 풀이**

리스트 길이가 2인지 확인합니다. 기존 `[0]`의 ID와 첫 생성 때 저장한 ID가 같은지도 비교합니다.

### 18. (학교 Ubuntu · 일반 사용자) 다시 1대로 축소

```bash
sed -i 's/server_count = 2/server_count = 1/' terraform.tfvars
```

**명령어 풀이**

축소하면 `[1]`만 삭제되어야 합니다. `[0]` 교체가 계획되면 원인을 확인한 뒤 진행합니다.

### 19. (학교 Ubuntu · 일반 사용자) 변경 계획 확인

```bash
terraform plan -out=scale-in.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 0 to add, 0 to change, 1 to destroy. 삭제 주소는 aws_instance.web[1]입니다.

### 20. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply scale-in.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 21. (학교 Ubuntu · 일반 사용자) 첫 서버가 그대로 남았는지 비교

```bash
terraform output -json instance_ids > remaining-instance.json
python3 - <<'PY'
import json
before = json.load(open("first-instance.json"))
after = json.load(open("remaining-instance.json"))
assert before == after and len(after) == 1
print("첫 인스턴스 ID 유지 확인")
PY
```

**명령어 풀이**

`assert`는 조건이 맞지 않으면 오류를 냅니다. 비교 대상은 IP보다 안정적인 EC2 인스턴스 ID입니다.

### 22. (학교 Ubuntu · 일반 사용자) 교체 계획 읽기 — 적용하지 않음

```bash
terraform plan -replace='aws_instance.web[0]'
```

**명령어 풀이**

- `-replace`는 지정 자원을 교체하는 계획을 보여 줍니다. 이 명령만으로 서버를 교체하지 않습니다.
- 출력의 `-/+`와 1 add / 1 destroy를 확인합니다. 이번 단계는 plan만 실행하며 저장·적용하지 않습니다.

**확인할 결과**

1 to add, 0 to change, 1 to destroy. 뒤의 드리프트 단계와 혼동하지 않습니다.

### 23. (학교 Ubuntu · 일반 사용자) Terraform 밖에서 태그를 변경

```bash
ci_vpc_id=$(terraform output -raw vpc_id)
aws ec2 create-tags --resources "$ci_vpc_id" --tags Key=Lesson,Value=outside-terraform
aws ec2 describe-vpcs --vpc-ids "$ci_vpc_id" --query 'Vpcs[0].Tags'
```

**명령어 풀이**

- AWS CLI로 실제 VPC의 태그만 수정합니다. 콘솔에서 태그를 바꿔도 같은 종류의 외부 변경입니다.
- `.tfvars`의 lesson_label은 original 그대로입니다. 이 불일치가 이번 실습의 드리프트입니다.
- State 파일을 직접 편집하지 않습니다.

**확인할 결과**

AWS의 Lesson은 outside-terraform, 코드의 lesson_label은 original입니다.

### 24. (학교 Ubuntu · 일반 사용자) 변경 계획 확인

```bash
terraform plan -out=restore.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 0 to add, 1 to change, 0 to destroy. Lesson을 original로 복구합니다.

### 25. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply restore.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 26. (학교 Ubuntu · 일반 사용자) 다시 실행해 변경 없음 확인

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

### 27. (학교 Ubuntu · 일반 사용자) 삭제 대상 확인

```bash
terraform plan -destroy -out=cleanup.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 0 to add, 0 to change, 12 to destroy.

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

## A부 중간 확인

- 1→2→1 각각의 계획 요약과 첫 EC2 ID 유지 결과를 제출합니다.
- `[0]`, `[1]`, `count.index`, `[*]`의 의미를 설명합니다.
- 외부 변경 후 plan이 무엇을 복구하려 했는지 설명합니다.
- 교체 계획을 실제 적용했다면 어떤 영향이 있었을지 설명합니다.

제출 화면에는 비밀번호·Access Key·Secret Key·Session Token·개인 키·State 원문을 포함하지 않습니다. `.tf` 코드와 본인이 정리한 결과를 제출합니다. 계정 ID·IP가 포함된 화면은 수업 제출 범위에서만 사용합니다.

## 막혔을 때

| 증상 | 해결 |
| --- | --- |
| Invalid index | server_count 범위 1~2와 서브넷 count 2 확인 |
| 자원 주소를 못 찾음 | `terraform state list`의 실제 주소를 확인하고 대괄호 주소를 작은따옴표로 감싸기 |
| 2대 변경이 계획되지 않음 | terraform.tfvars의 server_count 확인. 다른 `.auto.tfvars`에 같은 값이 중복되지 않았는지 확인 |
| 태그 드리프트가 없음 | 이번 State의 VPC ID를 수정했는지, 코드 lesson_label은 original인지 확인 |

공통 인증·설치 오류는 [공통 준비의 문제 해결](../common/README.md#막혔을-때)을 확인합니다. 실제 검증 범위와 남은 제한은 [검증 리포트](../VERIFICATION.md)에 기록합니다.

<a id="for-each-required"></a>
## B. 필수 비교 — count의 인덱스와 for_each의 키

A부의 29번까지 마쳤더라도 이번 주 필수 실습은 아직 끝나지 않았습니다. 아래 30~44번을 이어서 수행합니다. A부와 다른 State를 사용하므로 기존 주소를 이전하는 실습이 아닙니다.

| 비교 | count | for_each |
| --- | --- | --- |
| 입력 | 개수 | map 또는 문자열 set |
| 주소 식별자 | `[0]`, `[1]` | `["blue"]`, `["green"]` |
| 자주 쓰는 값 | `count.index` | `each.key`, `each.value` |
| 이번 실습 | 동일한 서버 대수 증감 | 이름별 서브넷 관리 |
| 변경 시 주의 | 리스트 중간 삭제·재정렬 시 인덱스 대응이 달라질 수 있음 | 키 변경은 다른 주소가 됨. 키가 같아도 속성 변경이 교체를 요구할 수 있음 |

동일한 서버의 개수를 조절하는 경우 count가 간단합니다. 이름과 개별 설정이 있는 자원은 for_each로 의도를 표현하기 쉽습니다. for_each가 모든 변경에서 자원 ID를 보존하는 것은 아닙니다.

### 30. (학교 Ubuntu · 일반 사용자) 별도 실습 폴더 준비

```bash
source "$HOME/.config/cloud-infra/env.sh"
aws sts get-caller-identity
mkdir -p "$HOME/cloud-infra/week09/for-each"
cd "$HOME/cloud-infra/week09/for-each"
```

**명령어 풀이**

9주차 State를 복사하지 않고 새로운 루트에서 시작합니다.

### 31. (학교 Ubuntu · 일반 사용자) providers.tf 작성

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
      Week   = "09"
    }
  }
}
EOF
```

**명령어 풀이**

A부와 같은 Terraform·Provider 버전을 사용하고 Week 태그를 09로 설정합니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat providers.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 32. (학교 Ubuntu · 일반 사용자) main.tf 작성

```bash
cat > main.tf <<'EOF'
locals {
  networks = {
    blue  = "10.110.0.0/24"
    green = "10.110.1.0/24"
  }
}
resource "aws_vpc" "lab" {
  cidr_block = "10.110.0.0/16"
  tags      = { Name = "cloud-infra-for-each" }
}
resource "aws_subnet" "lab" {
  for_each   = local.networks
  vpc_id     = aws_vpc.lab.id
  cidr_block = each.value
  tags       = { Name = each.key }
}
output "subnet_ids" {
  value = { for key, subnet in aws_subnet.lab : key => subnet.id }
}
EOF
```

**명령어 풀이**

- locals는 이 구성 안에서 재사용할 계산값입니다. 외부 입력 변수와 구분합니다.
- for_each는 map의 키를 자원 주소에 사용합니다. each.key는 blue/green, each.value는 해당 CIDR입니다.
- 출력의 for 표현식은 키별 서브넷 ID를 map으로 만듭니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat main.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 33. (학교 Ubuntu · 일반 사용자) 초기화·형식 정리·문법 확인

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

### 34. (학교 Ubuntu · 일반 사용자) 처음 생성할 자원의 계획 확인

```bash
terraform plan -out=create.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 3 to add, 0 to change, 0 to destroy.

### 35. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply create.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 36. (학교 Ubuntu · 일반 사용자) 키 기반 주소와 ID 보관

```bash
terraform state list
terraform state show 'aws_subnet.lab["blue"]'
terraform output -json subnet_ids > before-subnets.json
```

**명령어 풀이**

주소는 `[0]` 대신 `["blue"]`처럼 이름으로 구분됩니다. 따옴표 전체를 작은따옴표로 감싸 셸 해석을 막습니다.

### 37. (학교 Ubuntu · 일반 사용자) 새로운 키 추가

```bash
sed -i '/networks = {/a\    amber = "10.110.2.0/24"' main.tf
terraform fmt
```

**명령어 풀이**

map에 amber를 추가합니다. 파일에서 앞에 끼워 넣어도 기존 blue/green의 키는 바뀌지 않습니다.

### 38. (학교 Ubuntu · 일반 사용자) 변경 계획 확인

```bash
terraform plan -out=add-key.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

1 to add, 0 to change, 0 to destroy. 추가 주소는 aws_subnet.lab["amber"]입니다.

### 39. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply add-key.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 40. (학교 Ubuntu · 일반 사용자) 기존 키의 ID 유지 확인

```bash
terraform output -json subnet_ids > after-subnets.json
python3 - <<'PY'
import json
before = json.load(open("before-subnets.json"))
after = json.load(open("after-subnets.json"))
assert all(after[k] == v for k,v in before.items()) and len(after) == 3
print("blue/green ID 유지, amber만 추가")
PY
```

**명령어 풀이**

키가 같은 기존 두 서브넷의 ID가 유지되는지 검사합니다. 키 자체를 변경하면 별개 자원으로 인식될 수 있습니다.

### 41. (학교 Ubuntu · 일반 사용자) 다시 실행해 변경 없음 확인

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

### 42. (학교 Ubuntu · 일반 사용자) 삭제 대상 확인

```bash
terraform plan -destroy -out=cleanup.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

Plan: 0 to add, 0 to change, 4 to destroy.

### 43. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply cleanup.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 44. (학교 Ubuntu · 일반 사용자) 관리 자원이 남아 있지 않은지 확인

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

- count와 for_each 자원 주소의 차이, 기존 서브넷 ID가 유지된 결과를 제출합니다.

제출 화면에는 비밀번호·Access Key·Secret Key·Session Token·개인 키·State 원문을 포함하지 않습니다. `.tf` 코드와 본인이 정리한 결과를 제출합니다. 계정 ID·IP가 포함된 화면은 수업 제출 범위에서만 사용합니다.

## 막혔을 때

키 이름을 바꾸는 실험은 삭제·생성 계획을 유발할 수 있습니다. 기존 자원을 이동하려면 별도의 State 이전 설계가 필요하므로 이 교안에서 임의로 바꾸지 않습니다.

공통 인증·설치 오류는 [공통 준비의 문제 해결](../common/README.md#막혔을-때)을 확인합니다. 실제 검증 범위와 남은 제한은 [검증 리포트](../VERIFICATION.md)에 기록합니다.

---

[← Week 08](../week08/README.md) · [Week 10 →](../week10/README.md)

# 선택 실습 – ASG 원하는 용량 0→1→0

[전체 실습 목록](../README.md) · [공통 준비](../common/README.md)

**11주차 26번까지 완료하고 자원 정리 전에** 수행합니다. 내부 ALB 선택 실습을 했다면 그 확장 자원을 먼저 정리합니다. 약 15~25분 추가입니다.

이번에는 Launch Template과 Auto Scaling Group으로 **원하는 서버 수(desired capacity) 0→1→0**을 확인합니다. 부하 기반 정책·ALB 연결·새 인스턴스 nginx 설치·동적 Inventory는 포함하지 않습니다. ASG 인스턴스는 기존 Ansible Inventory의 web1/web2와 별개의 세 번째 서버입니다.

## 실습 절차

### 01. (학교 Ubuntu · 일반 사용자) 11주차 작업 환경 준비

```bash
source "$HOME/.config/cloud-infra/env.sh"
cd "$HOME/cloud-infra/week11"
terraform state list
```

**명령어 풀이**

기존 두 EC2가 관리되고 있는 루트입니다. ASG가 만드는 인스턴스는 별도의 aws_instance 자원 주소로 State에 등록되지 않는다는 점을 이후 비교합니다.

### 02. (학교 Ubuntu · 일반 사용자) modules/web_environment/security-output.tf 작성

```bash
cat > modules/web_environment/security-output.tf <<'EOF'
output "security_group_id" {
  value = aws_security_group.web.id
}
EOF
```

**명령어 풀이**

ALB 선택 실습에서 이미 만든 파일이 있으면 같은 내용으로 유지합니다. 새 자원 생성 없이 기존 SG ID를 공개하는 출력입니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat modules/web_environment/security-output.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 03. (학교 Ubuntu · 일반 사용자) asg-extension.tf 작성

```bash
cat > asg-extension.tf <<'EOF'
variable "asg_desired" {
  type    = number
  default = 0
  validation {
    condition     = contains([0, 1], var.asg_desired)
    error_message = "asg_desired must be 0 or 1."
  }
}
resource "aws_launch_template" "class" {
  name_prefix   = "${var.name}-asg-"
  image_id      = var.ami_id
  instance_type = "t3.micro"
  iam_instance_profile {
    name = "LabInstanceProfile"
  }
  network_interfaces {
    associate_public_ip_address = true
    security_groups             = [module.web_environment.security_group_id]
  }
  block_device_mappings {
    device_name = "/dev/sda1"
    ebs {
      volume_type           = "gp3"
      volume_size           = 8
      encrypted             = true
      delete_on_termination = true
    }
  }
  metadata_options {
    http_tokens = "required"
  }
  tag_specifications {
    resource_type = "instance"
    tags = {
      Name   = "${var.name}-asg"
      Course = "cloud-infra"
      Week   = "11-extra"
    }
  }
}
resource "aws_autoscaling_group" "class" {
  name_prefix         = "${var.name}-"
  min_size            = 0
  max_size            = 1
  desired_capacity    = var.asg_desired
  vpc_zone_identifier = module.web_environment.subnet_ids
  launch_template {
    id      = aws_launch_template.class.id
    version = aws_launch_template.class.latest_version
  }
}
output "asg_name" {
  value = aws_autoscaling_group.class.name
}
EOF
```

**명령어 풀이**

- Launch Template은 ASG가 인스턴스를 만들 때 사용할 AMI·유형·네트워크·디스크 설정입니다.
- ASG의 min=0, max=1 범위에서 desired를 변경합니다. desired 0에서는 템플릿과 그룹만 만들고 서버는 만들지 않습니다.
- 기존 LabInstanceProfile을 사용합니다. 이 인스턴스에는 수업용 SSH 키를 등록하지 않으며 접속 실습을 하지 않습니다.
- latest_version은 템플릿의 구체적인 버전 번호를 참조합니다. 이번 실습에서는 템플릿 변경을 다루지 않습니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat asg-extension.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 04. (학교 Ubuntu · 일반 사용자) 확장 코드 확인

```bash
terraform fmt -recursive
terraform validate
```

**명령어 풀이**

Provider와 기존 모듈은 이미 초기화되어 있습니다.

### 05. (학교 Ubuntu · 일반 사용자) 변경 계획 확인

```bash
terraform plan -out=asg-add.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

2 to add, 0 to change, 0 to destroy. EC2 인스턴스는 아직 추가되지 않습니다.

### 06. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply asg-add.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 07. (학교 Ubuntu · 일반 사용자) 원하는 용량 1로 설정

```bash
printf 'asg_desired = 1
' > capacity.auto.tfvars
```

**명령어 풀이**

추가 자동 변수 파일로 원하는 용량을 지정합니다. 다른 파일에 같은 변수를 중복 지정하지 않습니다.

### 08. (학교 Ubuntu · 일반 사용자) 변경 계획 확인

```bash
terraform plan -out=asg-one.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

0 to add, 1 to change, 0 to destroy. 그룹 속성이 바뀌고 AWS가 서버를 생성합니다.

### 09. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply asg-one.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 10. (학교 Ubuntu · 일반 사용자) ASG 인스턴스가 InService가 될 때까지 확인

```bash
ci_asg=$(terraform output -raw asg_name)
ci_ready=false
for attempt in $(seq 1 60); do
  ci_count=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names "$ci_asg" --query 'length(AutoScalingGroups[0].Instances[?LifecycleState==`InService`])' --output text)
  if [ "$ci_count" = "1" ]; then ci_ready=true; break; fi
  sleep 5
done
test "$ci_ready" = true
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names "$ci_asg" --query 'AutoScalingGroups[0].{Desired:DesiredCapacity,Instances:Instances}'
terraform state list
```

**명령어 풀이**

- ASG 실행은 비동기이므로 최대 60회 조회하며 InService 1대를 기다립니다. 대기 시간을 제외한 API 시간도 더해질 수 있습니다.
- InService는 ASG 상태이며 nginx 설치나 HTTP 성공을 뜻하지 않습니다.
- State에는 그룹·템플릿이 있고, ASG가 만든 EC2는 별도의 aws_instance 주소로 나타나지 않습니다.

**확인할 결과**

Desired=1, Instances에 한 대가 InService로 표시됩니다. 대기 실패하면 그룹의 Activity history를 확인합니다.

### 11. (학교 Ubuntu · 일반 사용자) 원하는 용량을 다시 0으로 변경

```bash
printf 'asg_desired = 0
' > capacity.auto.tfvars
```

**명령어 풀이**

ASG가 만든 인스턴스를 그룹 정책에 따라 종료하도록 요구합니다. 기존 Terraform의 두 웹 서버에는 영향을 주지 않습니다.

### 12. (학교 Ubuntu · 일반 사용자) 변경 계획 확인

```bash
terraform plan -out=asg-zero.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

0 to add, 1 to change, 0 to destroy.

### 13. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply asg-zero.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 14. (학교 Ubuntu · 일반 사용자) ASG 인스턴스 목록이 비워질 때까지 확인

```bash
ci_asg=$(terraform output -raw asg_name)
ci_empty=false
for attempt in $(seq 1 60); do
  ci_count=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names "$ci_asg" --query 'length(AutoScalingGroups[0].Instances)' --output text)
  if [ "$ci_count" = "0" ]; then ci_empty=true; break; fi
  sleep 5
done
test "$ci_empty" = true
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names "$ci_asg" --query 'AutoScalingGroups[0].{Desired:DesiredCapacity,Instances:Instances}'
```

**명령어 풀이**

단순히 desired 값이 0인 것뿐 아니라 그룹의 인스턴스 목록이 실제로 비었는지 확인합니다.

**확인할 결과**

Desired=0, Instances=[]. 종료된 EC2 기록은 콘솔에 잠시 남을 수 있습니다.

### 15. (학교 Ubuntu · 일반 사용자) ASG 확장 파일 비활성화

```bash
mv asg-extension.tf asg-extension.tf.disabled
mv capacity.auto.tfvars capacity.auto.tfvars.disabled
```

**명령어 풀이**

변수 파일도 함께 비활성화해 선언되지 않은 변수 경고를 피합니다. 코드와 기록은 보관합니다.

### 16. (학교 Ubuntu · 일반 사용자) 변경 계획 확인

```bash
terraform plan -out=asg-remove.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

0 to add, 0 to change, 2 to destroy.

### 17. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply asg-remove.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 18. (학교 Ubuntu · 일반 사용자) 다시 실행해 변경 없음 확인

```bash
terraform plan
```

**명령어 풀이**

- 같은 코드를 다시 실행해도 자원이 반복 생성되지 않는지 확인합니다.
- 외부 변경·인증 만료·잘못된 폴더 때문에 다른 결과가 나올 수 있으므로 실제 문구를 읽습니다.

**확인할 결과**

`No changes. Your infrastructure matches the configuration.`

이제 [11주차 자원 정리](../week11/README.md#자원-정리--제출-전에-수행)로 돌아가 기존 14개 자원까지 삭제합니다.

오류 대응: InService에 도달하지 못하면 ASG Activity history의 인스턴스 유형·용량·권한·AMI 오류를 확인합니다. 단순히 Terraform apply가 성공했다고 실제 서버 준비까지 성공했다고 판단하지 않습니다.

---

[11주차로 돌아가기](../week11/README.md) · [전체 실습 목록](../README.md)

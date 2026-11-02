# 선택 실습 – 내부 ALB로 두 웹 서버 연결

[전체 실습 목록](../README.md) · [공통 준비](../common/README.md)

**11주차 26번까지 완료하고 자원 정리 전에** 수행합니다. 기존 두 nginx 서버를 대상으로 내부 Application Load Balancer를 추가합니다. 약 20~30분 추가이며 ALB가 살아 있는 동안 비용이 발생하므로 끝나면 이 확장 자원부터 삭제합니다.

내부 ALB는 공인 인터넷용 주소가 아닙니다. 학교 VM에서 SSH로 EC2에 원격 명령을 보내고, **VPC 안의 EC2에서 내부 ALB DNS로 HTTP 요청**합니다. 응답 문구가 같으므로 트래픽 분배 비율을 증명하는 실습은 아니며, 두 대상 healthy와 ALB 경유 응답을 확인합니다.

## 실습 절차

### 01. (학교 Ubuntu · 일반 사용자) 기존 웹 서버 환경 확인

```bash
source "$HOME/.config/cloud-infra/env.sh"
cd "$HOME/cloud-infra/week11"
ansible web -m ping
```

**명령어 풀이**

11주차의 두 nginx와 네트워크를 재사용합니다. Handler 선택 실습 완료 여부와 관계없이 수행할 수 있습니다.

### 02. (학교 Ubuntu · 일반 사용자) modules/web_environment/security-output.tf 작성

```bash
cat > modules/web_environment/security-output.tf <<'EOF'
output "security_group_id" {
  value = aws_security_group.web.id
}
EOF
```

**명령어 풀이**

자식 모듈의 SG ID를 출력으로 공개합니다. 이번 week11에 복사한 모듈만 수정합니다. 10주차 공통 모듈이나 State는 건드리지 않습니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat modules/web_environment/security-output.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 03. (학교 Ubuntu · 일반 사용자) alb-extension.tf 작성

```bash
cat > alb-extension.tf <<'EOF'
resource "aws_security_group" "alb" {
  name_prefix = "${var.name}-alb-"
  vpc_id      = module.web_environment.vpc_id
}
resource "aws_vpc_security_group_ingress_rule" "alb_http" {
  security_group_id = aws_security_group.alb.id
  cidr_ipv4         = var.vpc_cidr
  ip_protocol       = "tcp"
  from_port         = 80
  to_port           = 80
}
resource "aws_vpc_security_group_egress_rule" "alb_outbound" {
  security_group_id = aws_security_group.alb.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}
resource "aws_vpc_security_group_ingress_rule" "web_from_alb" {
  security_group_id            = module.web_environment.security_group_id
  referenced_security_group_id = aws_security_group.alb.id
  ip_protocol                  = "tcp"
  from_port                    = 80
  to_port                      = 80
}
resource "aws_lb" "internal" {
  name_prefix        = "ci-"
  internal           = true
  load_balancer_type = "application"
  subnets            = module.web_environment.subnet_ids
  security_groups    = [aws_security_group.alb.id]
}
resource "aws_lb_target_group" "web" {
  name_prefix          = "ci-"
  vpc_id               = module.web_environment.vpc_id
  port                 = 80
  protocol             = "HTTP"
  deregistration_delay = 5
  health_check {
    path                = "/"
    interval            = 10
    healthy_threshold   = 2
    unhealthy_threshold = 2
    matcher             = "200"
  }
}
resource "aws_lb_target_group_attachment" "web" {
  count            = 2
  target_group_arn = aws_lb_target_group.web.arn
  target_id        = module.web_environment.instance_ids[count.index]
  port             = 80
}
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.internal.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}
output "alb_dns" {
  value = aws_lb.internal.dns_name
}
output "target_group_arn" {
  value = aws_lb_target_group.web.arn
}
EOF
```

**명령어 풀이**

- `internal = true`로 내부 ALB를 만듭니다. 서로 다른 AZ의 기존 서브넷 두 개를 지정합니다.
- ALB SG는 VPC CIDR에서 들어오는 HTTP를 허용합니다. 웹 서버 SG에는 ALB SG에서 오는 HTTP를 추가합니다.
- Target Group은 웹 서버 목록과 상태 검사 방법, Listener는 들어온 HTTP를 어디로 보낼지를 정의합니다.
- attachment 두 개가 기존 EC2 두 대를 Target Group에 등록합니다.
- deregistration_delay를 5초로 줄여 실습 정리를 단축합니다. 운영 설정 권장값을 의미하지는 않습니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat alb-extension.tf`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 04. (학교 Ubuntu · 일반 사용자) 확장 코드 확인

```bash
terraform fmt -recursive
terraform validate
```

**명령어 풀이**

기존 루트에 자원을 추가했으므로 같은 State에서 확장 자원을 관리합니다.

### 05. (학교 Ubuntu · 일반 사용자) 변경 계획 확인

```bash
terraform plan -out=alb-add.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

9 to add, 0 to change, 0 to destroy. 기존 EC2를 교체하지 않습니다.

### 06. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply alb-add.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 07. (학교 Ubuntu · 일반 사용자) 두 대상 healthy 기다리기

```bash
ci_tg=$(terraform output -raw target_group_arn)
aws elbv2 wait target-in-service --target-group-arn "$ci_tg"
aws elbv2 describe-target-health --target-group-arn "$ci_tg" --query 'TargetHealthDescriptions[].{ID:Target.Id,State:TargetHealth.State}'
```

**명령어 풀이**

상태 검사는 비동기이므로 ALB 생성 완료 직후에는 initial일 수 있습니다. 둘 다 healthy인지 확인합니다.

**확인할 결과**

두 대상의 State가 모두 healthy입니다.

### 08. (학교 Ubuntu · 일반 사용자) VPC 내부에서 ALB 경유 요청

```bash
ci_alb_dns=$(terraform output -raw alb_dns)
ci_web1=$(awk '/^web1 / {sub("ansible_host=", "", $2); print $2}' inventory.ini)
ssh -i "$HOME/.ssh/cloud-infra-lab" -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=./known_hosts "ubuntu@$ci_web1" "curl --fail --max-time 15 http://$ci_alb_dns/"
```

**명령어 풀이**

외부에서 EC2로 SSH 접속한 뒤 EC2 안의 curl이 내부 DNS로 요청합니다. SSH 목적지 공인 IP와 HTTP 목적지 내부 ALB 주소를 구분합니다.

**확인할 결과**

Managed by Ansible이 포함된 응답입니다.

### 09. (학교 Ubuntu · 일반 사용자) 확장 구성 파일 비활성화

```bash
mv alb-extension.tf alb-extension.tf.disabled
```

**명령어 풀이**

Terraform은 .tf 확장자 파일을 읽습니다. 코드를 보관하되 활성 구성에서 빼면 해당 ALB 자원만 삭제할 계획을 만들 수 있습니다. State를 직접 지우지 않습니다.

### 10. (학교 Ubuntu · 일반 사용자) 변경 계획 확인

```bash
terraform plan -out=alb-remove.tfplan
```

**명령어 풀이**

- `plan`은 코드·State·AWS의 현재 상태를 비교합니다. 이 단계는 자원을 생성·변경하지 않습니다.
- `+`는 생성, `~`는 제자리 수정, `-`는 삭제, `-/+`는 교체입니다.
- `-out`은 이 계획을 저장합니다. 예상과 다르면 다음 apply를 실행하지 않습니다.

**확인할 결과**

0 to add, 0 to change, 9 to destroy. 기존 14개 자원은 삭제 대상이 아닙니다.

### 11. (학교 Ubuntu · 일반 사용자) 검토한 계획 적용

```bash
terraform apply alb-remove.tfplan
```

**명령어 풀이**

- 저장된 계획을 적용합니다. 추가 `yes` 질문이 없으므로 **직전 계획을 검토한 뒤** 실행합니다.
- 실행 중에는 Ctrl+C로 중단하지 말고 완료를 기다립니다. 오류가 나면 State를 지우지 말고 원인을 해결한 뒤 새로 plan합니다.

**확인할 결과**

`Apply complete!`와 변경 수가 표시됩니다. 출력 주소와 ID는 계정·실행마다 다릅니다.

### 12. (학교 Ubuntu · 일반 사용자) 다시 실행해 변경 없음 확인

```bash
terraform plan
```

**명령어 풀이**

- 같은 코드를 다시 실행해도 자원이 반복 생성되지 않는지 확인합니다.
- 외부 변경·인증 만료·잘못된 폴더 때문에 다른 결과가 나올 수 있으므로 실제 문구를 읽습니다.

**확인할 결과**

`No changes. Your infrastructure matches the configuration.`

ALB 확장 자원 정리를 확인한 뒤 [11주차 정리](../week11/README.md#자원-정리--제출-전에-수행)를 수행하거나 ASG 선택 실습으로 진행합니다. SG 출력 파일은 남아 있어도 추가 자원을 생성하지 않습니다.

오류 대응: unhealthy이면 nginx HTTP 응답, ALB→웹 SG 규칙, 대상 포트 80을 확인합니다. 학교 VM에서 내부 DNS 접속이 안 되는 것은 이 설계에서 정상입니다.

---

[11주차로 돌아가기](../week11/README.md) · [전체 실습 목록](../README.md)

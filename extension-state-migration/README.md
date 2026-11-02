# 선택 심화 설계안 – 실행 중인 자원의 count → for_each State 이전

[전체 목차](../README.md) · [9주차 필수 실습](../week09/README.md)

**검증 상태: 교수자용 설계안입니다. 아래 두 구성의 Terraform 문법·참조 검사는 통과했으며, 실제 AWS 자원의 주소 이전·ID 보존은 아직 실행 검증하지 않았습니다.** 새 검증을 시도했으나 AWS Academy 로그인 세션이 만료되어 인증 화면으로 이동했습니다. 학생에게 실행형 교안으로 배포하기 전에 교수자가 테스트 계정에서 아래 수락 기준을 확인해야 합니다. 앞서 AWS 검증을 마친 for_each 기본 실습과 혼동하지 않습니다.

## 필수 실습과 다른 점

9주차 B부는 별도 State에서 for_each 자원을 새로 만들고 키 추가를 비교합니다. 여기서는 **이미 존재하는 같은 AWS 자원을 유지하면서 Terraform 주소만 변경**하는 것이 목표입니다. EC2 실행 상태나 서비스 무중단까지 증명하는 실험은 아닙니다. 먼저 VPC·서브넷만으로 자원 식별 관계를 다룹니다.

| 현재 State 주소 | 새 코드 주소 | 유지할 실제 객체 |
| --- | --- | --- |
| `aws_subnet.lab[0]` | `aws_subnet.lab["blue"]` | blue 서브넷의 기존 AWS ID |
| `aws_subnet.lab[1]` | `aws_subnet.lab["green"]` | green 서브넷의 기존 AWS ID |

반복 문법만 바꾸면 기존 주소와 새 주소를 별개로 볼 수 있습니다. `moved` 블록으로 대응 관계를 선언하면 Terraform이 State의 주소 변경을 계획합니다. **물리적 서브넷이나 서버를 다른 장소로 옮기는 기능은 아닙니다.** [Terraform 공식 리팩터링 문서](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring).

## 교수자 검증용 시작 구성

별도 테스트 루트 하나에서 기존 교안과 동일한 Provider·리전을 사용합니다. 아래 main.tf로 VPC와 서브넷 두 개를 생성해 주소와 ID를 기록하는 설계입니다. 다른 주차의 State를 이 폴더로 복사하지 않습니다.

```hcl
locals {
  names = ["blue", "green"]
  cidrs = ["10.120.0.0/24", "10.120.1.0/24"]
}
resource "aws_vpc" "lab" {
  cidr_block = "10.120.0.0/16"
  tags      = { Name = "cloud-infra-state-migration" }
}
resource "aws_subnet" "lab" {
  count      = 2
  vpc_id     = aws_vpc.lab.id
  cidr_block = local.cidrs[count.index]
  tags       = { Name = local.names[count.index] }
}
output "subnet_ids" {
  value = { for i, subnet in aws_subnet.lab : local.names[i] => subnet.id }
}

```

## 같은 루트에서 사용할 전환 구성

아래 구성으로 main.tf를 교체하는 설계입니다. CIDR·Name·VPC 참조 등 AWS 속성은 유지합니다. 두 코드를 동시에 활성 `.tf` 파일로 두면 중복 선언 오류가 나므로 전환 전 코드는 `.tf.disabled` 등의 비활성 이름으로 보관합니다.

```hcl
locals {
  networks = {
    blue  = "10.120.0.0/24"
    green = "10.120.1.0/24"
  }
}
resource "aws_vpc" "lab" {
  cidr_block = "10.120.0.0/16"
  tags      = { Name = "cloud-infra-state-migration" }
}
resource "aws_subnet" "lab" {
  for_each   = local.networks
  vpc_id     = aws_vpc.lab.id
  cidr_block = each.value
  tags       = { Name = each.key }
}
moved {
  from = aws_subnet.lab[0]
  to   = aws_subnet.lab["blue"]
}
moved {
  from = aws_subnet.lab[1]
  to   = aws_subnet.lab["green"]
}
output "subnet_ids" {
  value = { for key, subnet in aws_subnet.lab : key => subnet.id }
}

```

`moved.from`은 현재 State에 기록된 주소, `moved.to`는 새 코드의 주소입니다. 임의로 인덱스와 키를 대응시키지 말고 실제 ID·CIDR·태그를 먼저 대조해야 합니다. `moved` 블록은 코드에 남겨 아직 이전하지 않은 같은 구성의 사용자가 경로를 따라갈 수 있게 합니다.

## 실행형 교안으로 전환하기 위한 검증 순서

1. **사전 확인:** 새로운 실습 전용 루트에서 count 구성으로 3개 자원을 생성하고, `terraform state list`·`terraform state show` 및 AWS 조회로 주소와 ID를 기록합니다.
2. **기록 보관:** 현재 State를 접근이 제한된 백업 파일로 보관합니다. 이 백업은 복구용이며 다른 루트에서 동시에 사용하는 State가 아닙니다. Git이나 학생 배포물에 넣지 않습니다.
3. **차이 관찰:** for_each 코드로 바꾸되 moved 블록을 제외한 상태에서 계획만 확인합니다. 예상치 못한 삭제·생성 계획을 실제 적용하지 않습니다.
4. **매핑 선언:** moved 두 개를 포함한 전환 코드로 문법 검사와 새 계획을 실행합니다. 예상은 `0 to add, 0 to change, 0 to destroy` 및 두 주소의 이동 표시입니다. 속성 변경이나 교체가 있다면 원인을 확인합니다.
5. **State 반영:** 검토한 계획을 적용해 주소 변경을 State에 기록합니다. plan만 확인한 것을 이전 완료로 판단하지 않습니다.
6. **보존 확인:** 두 실제 서브넷 ID가 전후 동일하고 State 주소가 blue/green으로 바뀌었는지 확인합니다. 다시 계획했을 때 변경이 없어야 합니다.
7. **정리:** 전환 후의 동일한 루트·State에서 VPC와 서브넷 3개를 삭제하고 잔존 여부를 조회합니다.

이 설계에서는 `terraform state mv`를 병행하지 않습니다. 선언형 moved 방식 하나에 집중하며 State 파일 직접 편집도 하지 않습니다. State 파일이 있다는 이유만으로 기존 자원을 자동으로 올바르게 연결하는 것은 아닙니다.

## 통과 기준

- 주소 매핑이 `[0]→blue`, `[1]→green`과 일치한다.
- 전환 적용에서 AWS 자원의 생성·수정·삭제가 0이다.
- 전후 VPC·서브넷 ID가 같고, 새 주소에서 같은 객체가 조회된다.
- 재실행은 No changes이며 최종 정리가 완료된다.

위 기준은 **기대 결과**이며 이번 개정에서 관측한 AWS 실행 결과는 아닙니다. 정적 검사 기록은 [검증 리포트](../VERIFICATION.md)를 확인합니다.

# 실습 검증 요약

[실습 목차](README.md)

검증·개정일: 2026-11-03 (한국 시간). 이 문서는 7~11주차와 선택 심화의 검증 결과를 요약합니다. 상세 실행 로그와 JSON은 교수자 보관용 원본 패키지에 있습니다.

## 검증 범위

교수자의 AWS Academy Learner Lab Student View 계정 한 개, us-east-1에서 학교 Ubuntu를 실행 환경으로 사용했습니다. 설치와 인증, Terraform의 계획·생성·수정·삭제, EC2 SSH, Ansible 설정과 복구를 실제 실행했습니다.

| 실습 | 확인한 결과 | 상태 |
| --- | --- | --- |
| 공통 준비 | 도구 설치·체크섬·임시 인증·STS 계정 확인 | 실행 통과 |
| 7주차 | VPC 생성, 태그 변경 시 ID 유지, No changes, 삭제 | AWS 실행 통과 |
| 8주차 | 네트워크·EC2 생성, 상태 검사, SSH ubuntu 로그인, cloud-init 완료, 삭제 | AWS 실행 통과 |
| 9주차 A부 | 두 AZ의 서브넷, EC2 1→2→1, 첫 서버 ID 유지, 외부 태그 변경 복구, 삭제 | AWS 실행 통과 |
| 9주차 B부 | for_each의 blue/green ID 유지, amber만 추가, 전체 삭제 | AWS 실행 통과 |
| 10주차 | dev/test 별도 State와 자원, dev 변경·삭제 후 test 유지, 두 환경 정리 | AWS 실행 통과 |
| 11주차 | 두 서버 nginx·HTTP, 멱등성, 파일 드리프트 복구, 전체 삭제 | AWS 실행 통과 |
| Handler·check/diff | 설정 변경 시 reload, 실제 /health 응답, 재실행 변경 없음 | 실행 통과 |
| 내부 ALB | 두 대상 healthy, EC2에서 내부 ALB 경유 HTTP, 확장 자원 삭제 | AWS 실행 통과 |
| ASG | 실제 InService 1대 확인, desired 0에서 인스턴스 목록 비움, 그룹·템플릿 삭제 | AWS 실행 통과 |
| count→for_each State 이전 | 전환 전·후 HCL의 init/validate 통과 | **정적 검사만 통과, AWS 실행 미검증** |

검증 버전은 Ubuntu 24.04 x86_64 / Python 3.12, Terraform 1.17.1, AWS Provider 6.71.0, AWS CLI 1.46.2, Ansible Core 2.21.5입니다. AMI는 us-east-1의 Canonical Ubuntu 24.04 AMD64 `ami-0d6dd9e8265d48906`입니다.

## Ansible 반복 실행 결과

| 실행 | web1 changed | web2 changed | unreachable / failed |
| --- | --- | --- | --- |
| 첫 설정 | 2 | 2 | 모두 0 |
| 즉시 재실행 | 0 | 0 | 모두 0 |
| web1 파일을 수동 변경한 뒤 복구 | 1 | 0 | 모두 0 |
| 복구 후 재실행 | 0 | 0 | 모두 0 |

웹 파일을 수동 변경했을 때 Terraform은 No changes였습니다. 이번 구성에서 웹 파일은 Ansible이 관리하기 때문입니다. apt 캐시 유효 시간 밖에서 실행하면 캐시 갱신이 changed로 나타날 수 있습니다.

## 개정 사항과 해석

- 기존에 실제 AWS에서 검증한 for_each 기본 실습을 9주차 B부 30~44번 필수로 편입했습니다. 작업 폴더와 Week 태그 표기를 바꾸었으며, 이 편집을 별도의 AWS 재실행으로 기록하지 않습니다.
- 9주차 B부는 A부 자원을 삭제한 뒤 별도 State로 새 자원을 만듭니다. 기존 count 자원을 유지하면서 주소를 이전하는 실습이 아닙니다.
- 실행 중 발견한 Inventory 작성 블록의 줄바꿈 오류는 수정 후 다시 실행해 통과했습니다. 배포한 교안에는 수정본이 반영돼 있습니다.
- State 이전 심화는 교수자용 설계안입니다. 전환 전·후 구성의 Terraform 정적 검사는 통과했으나, AWS Academy 로그인 만료로 실제 이전·ID 보존 검증은 완료하지 못했습니다. 학생 실행형 교안으로 확정하기 전에 별도 검증이 필요합니다.
- `--diff`는 11주차 파일 복구 단계에도 필수로 사용합니다. 선택으로 둔 부분은 Handler와 `--check`를 함께 사용하는 추가 설정 실습입니다.

## 자원 정리와 운영 조건

2026-11-03 **02:09:01 KST**의 최종 AWS 조회에서 활성 EC2, EBS, ENI, 기본 VPC 외 VPC, ALB, Target Group, ASG가 모두 0이었습니다. 테스트 키 페어·Launch Template·서브넷·Security Group·Route Table·Internet Gateway도 정리했고 기존 기본 VPC 1개는 유지했습니다. 이후 필수/심화 구분 개정과 업로드용 패키지 편집에서는 AWS 자원을 새로 만들지 않았습니다.

이번 결과는 해당 테스트 계정에 한정됩니다. 실제 학생 계정별 권한·예산, 동시 접속 부하, 세션 만료 후 재시작, 장기간 유지·복구, 전체 수업 비용은 별도 확인 대상입니다. 도구·AMI 가용성은 다음 수업 운영 전에 다시 확인합니다.

9주차의 140분 배분은 이론 40분, A부 60분, B부 25분, 정리 15분입니다. 이는 학생의 수행 시간을 실측한 결과가 아닌 수업 설계입니다. 도구 설치는 사전에 마치고 매주 AWS 자원을 삭제합니다.

S3 Backend, 새 IAM 사용자·역할, NAT Gateway, RDS, 인터넷 공개 ALB는 이 교안의 검증 범위에 포함하지 않았습니다. ASG 실습은 원하는 용량 0→1→0을 다루며 부하 기반 확장 정책·새 서버의 nginx 설정·동적 Inventory를 포함하지 않습니다.

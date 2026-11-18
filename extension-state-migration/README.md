# 9주차 구성 변경 안내

[전체 실습 목록](../README.md) · [9주차 이어서 진행](../week09/README.md#for-each-required)

이전 주소 이전 설계안은 현재 9주차에서 사용하지 않습니다. 현재 교안은 **VPC만 유지하고 기존 EC2·서브넷·부속 자원을 삭제한 뒤 새 for_each 서브넷으로 구성**합니다. 같은 폴더와 State를 사용하며 별도 State 주소 이전 절차는 없습니다.

Map은 `for_each = { ... }`에 직접 선언하고, output은 `aws_subnet.lab["blue"].id`, `aws_subnet.lab["green"].id`처럼 키를 직접 참조합니다. 이 페이지에서 별도 자원을 만들지 말고 9주차 27번부터 이어서 진행합니다.

기존 count 서브넷의 ID는 보존하지 않습니다. 재구성에서 유지하는 것은 VPC ID이며, 이후 amber 추가에서 새로 만든 blue·green의 ID를 유지하는지 비교합니다. 검증 범위는 [검증 리포트](../VERIFICATION.md)를 확인합니다.

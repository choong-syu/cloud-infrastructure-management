# 선택 실습 – nginx 설정 변경과 Handler

[전체 실습 목록](../README.md) · [공통 준비](../common/README.md)

**11주차 26번까지 완료하고 자원 정리 전에** 수행합니다. 학교 Ubuntu의 week11 폴더와 두 서버를 그대로 사용합니다. 약 15분 추가입니다.

페이지 파일만 바꾸면 nginx 재시작이 필요하지 않습니다. 이번에는 nginx의 **서버 설정 파일**을 바꾸고, 변경이 있을 때만 Handler로 reload합니다. `--check --diff`의 예측 결과와 실제 적용 결과도 비교합니다.

## 실습 절차

### 01. (학교 Ubuntu · 일반 사용자) 11주차 작업 폴더로 이동

```bash
source "$HOME/.config/cloud-infra/env.sh"
cd "$HOME/cloud-infra/week11"
ansible web -m ping
```

**명령어 풀이**

기존 두 웹 서버가 실행 중이며 Inventory가 유효한지 확인합니다.

### 02. (학교 Ubuntu · 일반 사용자) handler.yml 작성

```bash
cat > handler.yml <<'EOF'
---
- name: Add a health endpoint
  hosts: web
  become: true
  gather_facts: false
  tasks:
    - name: Configure nginx default server
      ansible.builtin.copy:
        dest: /etc/nginx/sites-available/default
        content: |
          server {
              listen 80 default_server;
              server_name _;
              root /var/www/html;
              index index.html;
              location /health {
                  default_type text/plain;
                  return 200 'healthy';
              }
              location / {
                  try_files $uri $uri/ =404;
              }
          }
        owner: root
        group: root
        mode: '0644'
      notify: Reload nginx

    - name: Test nginx configuration
      ansible.builtin.command: nginx -t
      changed_when: false

    - name: Run notified handlers before HTTP check
      ansible.builtin.meta: flush_handlers

    - name: Verify health endpoint after real apply
      ansible.builtin.uri:
        url: http://127.0.0.1/health
        return_content: true
      register: health
      failed_when: "health.status != 200 or (health.content | default('')) != 'healthy'"
      when: not ansible_check_mode

  handlers:
    - name: Reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
EOF
```

**명령어 풀이**

- 설정 파일이 실제 바뀐 경우에만 notify가 Handler를 예약합니다. 같은 Handler가 여러 번 통지되어도 처리 시점에는 한 번 실행됩니다.
- `nginx -t`가 실제 설정의 문법을 확인하며 changed_when: false로 조회 작업을 변경으로 표시하지 않습니다.
- flush_handlers는 HTTP 확인 전에 예약된 Handler를 실행합니다.
- check 모드에서는 아직 새 파일이 기록되지 않습니다. command가 건너뛰어질 수 있고 실제 설정 검사·HTTP 성공까지 보장하지 않으므로 HTTP 검사는 실제 적용 때만 수행합니다.
- 운영용 설정 배포에는 적용 전 임시 파일 검증·복구 절차가 더 필요합니다. 여기서는 수업의 고정된 작은 설정을 사용합니다.

**확인할 결과**

성공하면 별도 출력이 없습니다. `cat handler.yml`로 작성한 내용을 읽을 수 있습니다. 코드 블록은 첫 `cat`부터 마지막 `EOF`까지 전체를 복사합니다.

### 03. (학교 Ubuntu · 일반 사용자) 예측만 확인

```bash
ansible-playbook --check --diff handler.yml
```

**명령어 풀이**

실제 파일을 쓰지 않고 예상 차이를 읽습니다. check 모드는 모든 모듈과 외부 상황을 완벽히 모의 실행하는 기능은 아닙니다.

**확인할 결과**

파일 변경과 Handler 실행이 예상되지만 서버는 아직 변경되지 않습니다.

### 04. (학교 Ubuntu · 일반 사용자) 실제 적용

```bash
ansible-playbook --diff handler.yml
```

**명령어 풀이**

설정 검사가 성공한 뒤 nginx를 reload하고 /health 응답을 확인합니다.

**확인할 결과**

두 서버 각각 copy 1회 + Handler 1회로 changed=2, failed=0.

### 05. (학교 Ubuntu · 일반 사용자) 재실행 때 Handler가 생략되는지 확인

```bash
ansible-playbook handler.yml
```

**명령어 풀이**

같은 설정이면 copy가 changed가 아니므로 Handler를 예약하지 않습니다.

**확인할 결과**

두 서버 모두 changed=0, failed=0. RUNNING HANDLER가 나오지 않습니다.

이 선택 실습은 AWS 자원을 추가하지 않습니다. 이어서 다른 선택 실습을 하거나 [11주차 자원 정리](../week11/README.md#자원-정리--제출-전에-수행)로 돌아가 기존 14개 자원을 삭제합니다.

공식 참고: [Handlers](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_handlers.html).

---

[11주차로 돌아가기](../week11/README.md) · [전체 실습 목록](../README.md)

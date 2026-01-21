## Nginx 웹 서버 및 로드 밸런서 통합 구축 프로젝트

이 프로젝트는 **Ansible**을 활용하여 고가용성 웹 서비스의 기초가 되는 **로드 밸런싱(Load Balancing)** 환경을 자동화하여 구축하는 것을 목표로 합니다. 사용자의 요청을 받는 한 대의 로드 밸런서가 백엔드에 위치한 두 대의 웹 서버로 트래픽을 분산하는 구조를 실습합니다.

---

### 1. 프로젝트 개요 및 아키텍처

본 실습에서는 총 3대의 서버를 사용합니다.

* **Load Balancer (1대)**: 외부 사용자의 접속을 가장 먼저 받아 전면에 서는 서버입니다. Nginx의 `upstream` 모듈을 사용하여 트래픽을 분산합니다.
* **Web Servers (2대)**: 실제로 웹 콘텐츠를 제공하는 서버들입니다. 각 서버는 자신의 호스트 이름을 출력하여 로드 밸런싱이 정상적으로 작동하는지 확인시켜 줍니다.

---

### 2. 프로젝트 디렉토리 구조

Ansible 프로젝트를 체계적으로 관리하기 위한 표준 디렉토리 구성입니다. 깃허브(GitHub) 저장소에 바로 올릴 수 있는 형태입니다.

```text
nginx-lb-project/
├── inventory.yml                # 대상 서버들의 IP 주소와 그룹 정의
├── site_setup.yml               # 웹 서버 및 LB 구축을 위한 메인 플레이북
└── templates/                   # 동적 설정 생성을 위한 Jinja2 템플릿 폴더
    └── lb_nginx.conf.j2         # 로드 밸런서용 Nginx 설정 템플릿

```

---

### 3. 인벤토리 설정 파일 (`inventory.yml`)

각 서버의 역할을 그룹별로 정의하고 접속 정보를 입력합니다.

```yaml
all:                                      # 모든 호스트와 그룹을 포함하는 최상위 부모
  children:                               # 하위 그룹 정의 섹션
    load_balancer:                        # 로드밸런서 그룹 (트래픽 분산 역할)
      hosts:                              # 개별 호스트 목록
        lb_node:                          # 로드밸런서 호스트 별칭
          ansible_host: 192.168.10.100    # 로드밸런서 서버의 IP 주소
    web_servers:                          # 백엔드 웹 서버 그룹 (실제 서비스 제공)
      hosts:                              # 개별 호스트 목록
        web_01:                           # 웹 서버 1 별칭
          ansible_host: 192.168.10.101    # 웹 서버 1의 IP 주소
        web_02:                           # 웹 서버 2 별칭
          ansible_host: 192.168.10.102    # 웹 서버 2의 IP 주소

```

---

### 4. 통합 구축 Playbook (`site_setup.yml`)

웹 서버 구축과 로드 밸런서 설정을 순차적으로 진행하는 메인 실행 파일입니다.

```yaml
---
# [Step 1] 백엔드 웹 서버 구축 (web_servers 그룹 대상)
- name: 백엔드 웹 서버(Nginx) 구축
  hosts: web_servers                     # 인벤토리의 web_servers 그룹 대상
  become: true                           # 관리자(root) 권한으로 실행
  tasks:
    - name: Nginx 설치                    # 패키지 관리자를 통한 Nginx 설치
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true               # 설치 전 패키지 목록 업데이트

    - name: 서버 식별을 위한 index.html 생성 # 각 서버가 누구인지 확인하기 위한 페이지
      ansible.builtin.copy:
        content: "<h1>Hello! This is {{ inventory_hostname }}</h1>"
        dest: /var/www/html/index.html    # 웹 서버 기본 페이지 경로
        mode: '0644'                     # 파일 권한 설정

# [Step 2] 로드 밸런서 구축 (load_balancer 그룹 대상)
- name: Nginx 기반 로드 밸런서 구축
  hosts: load_balancer                   # 인벤토리의 load_balancer 그룹 대상
  become: true                           # 관리자 권한 실행
  tasks:
    - name: Nginx 패키지 설치              # 로드밸런서로 쓸 Nginx 설치
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: 기존 기본 설정 삭제             # 충돌 방지를 위한 기본 사이트 설정 제거
      ansible.builtin.file:
        path: /etc/nginx/sites-enabled/default
        state: absent                    # 파일 삭제 상태 유지

    - name: 로드 밸런싱 설정 파일 배포 (템플릿 활용) # Jinja2 템플릿을 실제 파일로 변환
      ansible.builtin.template:
        src: templates/lb_nginx.conf.j2  # 템플릿 원본 파일 경로
        dest: /etc/nginx/conf.d/load_balancer.conf # 생성될 설정 파일 경로
        mode: '0644'
      notify: Restart Nginx              # 파일 변경 시 재시작 핸들러 호출

  handlers:                              # notify에 의해 호출될 때만 작동
    - name: Restart Nginx
      ansible.builtin.service:
        name: nginx
        state: restarted                 # 설정 반영을 위해 서비스 재시작

```

---

### 5. Nginx 로드 밸런서 템플릿 (`templates/lb_nginx.conf.j2`)

웹 서버 그룹에 속한 서버들을 자동으로 인식하여 리스트업해주는 동적 설정 파일입니다.

```nginx
upstream my_backend_servers {             # 백엔드 서버 그룹 정의 시작
    # 인벤토리의 web_servers 그룹에 속한 모든 서버를 자동으로 순회하며 추가
    {% for host in groups['web_servers'] %}
    server {{ hostvars[host]['ansible_host'] }}:80 max_fails=3 fail_timeout=30s;
    {% endfor %}                         # 반복문 종료
}

server {                                 # 로드밸런서 가상 서버 설정
    listen 80;                            # 80번 포트에서 대기
    server_name localhost;                # 서버 식별 이름

    location / {                          # 모든 요청(/)에 대한 처리 규칙
        proxy_pass http://my_backend_servers; # 위에서 정의한 백엔드 그룹으로 전달
        proxy_set_header Host $host;      # 원본 요청의 호스트 헤더 유지
        proxy_set_header X-Real-IP $remote_addr; # 클라이언트 실제 IP 전달
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # 확인용: 어떤 백엔드 서버가 응답했는지 응답 헤더에 추가
        add_header X-Backend-Server $upstream_addr;
    }
}

```

---

### [과금 및 주의사항]

* **AWS EC2 인스턴스 비용**: 총 3대의 EC2(LB 1대, Web 2대)가 필요합니다. 프리티어 인스턴스(t2.micro 등)를 사용하더라도, 각 인스턴스에 할당된 **공인 IP(Public IP)** 사용에 대해 시간당 약 **$0.005**의 비용이 발생할 수 있습니다.
* **보안 그룹 (Security Group)**:
* **로드밸런서**: 모든 곳(0.0.0.0/0)으로부터 **TCP 80** 포트 허용이 필요합니다.
* **웹 서버**: 보안을 위해 로드밸런서 서버의 **사설 IP(Private IP)**로부터 오는 **TCP 80** 포트만 허용하는 구성을 권장합니다.


* **실습 종료 후**: 불필요한 과금을 막기 위해 모든 인스턴스를 **종료(Terminate)** 하십시오.

---

Next Step: Nginx 로드밸런싱 알고리즘 가중치(Weight) 적용 실습

프로젝트 구조, 자동화 구축, 동적 템플릿, AWS 비용 관리

사용자 맞춤 설정에 따라 깃허브 마크다운 형식을 적용하고, 프로젝트 설명과 디렉토리 구조를 최상단에 배치하여 통합 실습 가이드를 완성했습니다. 템플릿을 활용한 자동화 방식은 실무에서도 매우 중요한 개념입니다. 다음 단계로 로드밸런싱 알고리즘을 변경하는 방법이 궁금하신가요?

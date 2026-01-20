Nginx 기반 로드 밸런서 구축 실습 가이드 (Ansible YAML)
1. 인벤토리 설정 파일 (`inventory.yml`)

대상 서버들의 IP 주소를 그룹별로 정의합니다.

```

all:

  children:

    load_balancer:

      hosts:

        lb_node:

          ansible_host: 192.168.10.100  # 로드밸런서로 쓸 서버 IP

    web_servers:

      hosts:

        web_01:

          ansible_host: 192.168.10.101  # 실제 웹 서버 1

        web_02:

          ansible_host: 192.168.10.102  # 실제 웹 서버 2

```

2. 로드 밸런서 구성 Playbook (`lb_setup.yml`)

Nginx를 설치하고 로드 밸런싱 설정을 적용하는 메인 로직입니다.

```

---

- name: Nginx 기반 로드 밸런서 구축 실습

  hosts: load_balancer

  become: true

  tasks:

    - name: Nginx 패키지 설치

      ansible.builtin.apt:

        name: nginx

        state: present

        update_cache: true

    - name: 기존 기본 설정 삭제

      ansible.builtin.file:

        path: /etc/nginx/sites-enabled/default

        state: absent

    - name: 로드 밸런싱 설정 파일 배포 (템플릿 활용)

      ansible.builtin.template:

        src: templates/lb_nginx.conf.j2

        dest: /etc/nginx/conf.d/load_balancer.conf

        mode: '0644'

      notify: Restart Nginx

  handlers:

    - name: Restart Nginx

      ansible.builtin.service:

        name: nginx

        state: restarted

```

3. Nginx 설정 템플릿 (`templates/lb_nginx.conf.j2`)

인벤토리에 등록된 웹 서버 IP들을 자동으로 리스트업하여 로드 밸런싱 규칙을 만듭니다.

```

upstream my_backend_servers {

    # 인벤토리의 web_servers 그룹에 속한 모든 서버를 자동으로 추가

    {% for host in groups['web_servers'] %}

    server {{ hostvars[host]['ansible_host'] }}:80 max_fails=3 fail_timeout=30s;

    {% endfor %}

}

server {

    listen 80;

    server_name localhost;

    location / {

        proxy_pass http://my_backend_servers;

        proxy_set_header Host $host;

        proxy_set_header X-Real-IP $remote_addr;

        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        

        # 로드 밸런싱 작동 확인을 위한 헤더 추가

        add_header X-Backend-Server $upstream_addr;

    }

}

```

[과금 및 주의사항]

1. AWS EC2 비용: 인스턴스 3대 가동 시 프리티어라도 공인 IP당 약 $0.005/시간이 발생합니다.

2. 보안 그룹: LB 서버는 80포트 개방, Web 서버는 LB로부터의 트래픽을 허용해야 합니다.

3. 중지: 실습 종료 후 반드시 인스턴스를 종료하여 추가 과금을 방지하세요.

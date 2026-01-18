### Localhost 실행 시 SSH 키 없이 접속하는 방법 (Connection: Local)

Ansible은 기본적으로 모든 대상(로컬 포함)을 '원격 서버'로 간주하여 SSH 통신을 시도합니다. 내 컴퓨터(Localhost)에서 실행할 때 굳이 SSH 인증 과정을 거치지 않으려면, 연결 방식을 **`local`**로 명시하여 SSH를 우회하고 셸 명령어를 직접 실행하게 설정해야 합니다.

#### 1. 인벤토리 파일 수정 (가장 권장됨)

`inventory.ini` 파일에서 대상 호스트 옆에 `ansible_connection=local` 변수를 추가합니다.

**파일: `inventory.ini**`

```ini
[webservers]
# IP 대신 localhost 별칭을 사용하고 연결 방식을 local로 지정
localhost ansible_connection=local

```

* **원리:** Ansible이 SSH 클라이언트를 호출하지 않고, 현재 셸 세션에서 파이썬 스크립트를 직접 실행합니다.

#### 2. 명령어 옵션으로 일회성 실행

인벤토리 파일을 수정하지 않고 실행 시점에만 강제로 로컬 연결을 사용하려면 `-c` 옵션을 사용합니다.

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass -c local

```

* **`-c local`**: 연결 플러그인(Connection Plugin)을 `local`로 덮어씌웁니다.

#### 3. 주의사항: 권한 상승 (Sudo)

SSH 접속 문제는 해결되지만, 패키지 설치(`apt`)나 서비스 재시작(`systemctl`) 등 관리자 권한이 필요한 작업은 여전히 비밀번호가 필요할 수 있습니다.

* 만약 현재 사용자가 `sudo` 사용 시 비밀번호를 입력해야 한다면, 실행 시 **`-K`** (대문자 K) 옵션을 추가해야 합니다.
```bash
# -c local: SSH 생략 / -K: sudo 비밀번호 입력 요청
ansible-playbook -i inventory.ini site.yml --ask-vault-pass -c local -K

```



---

**Next Step:** Ansible 권한 상승 옵션 become, become_method, become_user 의 차이점

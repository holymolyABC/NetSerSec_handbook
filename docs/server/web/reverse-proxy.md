# URL의 포트를 생략하는 역방향 프록시

**상태:** 구성 예시, 실제 적용·검증 미완료. **대상:** Nginx가 이미 설치된 Linux, 같은 호스트의127.0.0.1:1025에서 앱이 응답하는 경우.

## 목적과 선택지

`http://agri.example.org:1025/login`을 `http://agri.example.org/login`으로 사용합니다. DNS는 IP를 안내할 뿐 일반 브라우저의 접속 포트를 정하지 않습니다. 포트 생략 시 HTTP는80, HTTPS는443을 사용합니다.

| 방법 | 적합한 상황 | 주의 |
|---|---|---|
| 역방향 프록시 | 기존 웹서버와 도메인별 공존 | Host·redirect·cookie·앱 설정 확인 |
| 앱 포트 변경 | 단독 서비스 | 기존80/443 충돌·TLS |
| 방화벽 포트 변환 | 전용IP, 외부80→내부1025 | 같은IP의 다른 웹서비스와 충돌 가능 |

보통 기존 웹서버에 해당 도메인의 가상 호스트를 추가합니다. 단순302로1025에 이동시키면 주소창에 포트가 다시 보입니다.

## 1. 조회

DNS 서버가 아닌 앱 서버에서 확인합니다.

```bash
sudo ss -lntp | egrep ':(80|443|1025)[[:space:]]'
nginx -v
apache2 -v
curl --max-time 5 -sS -D - -o /dev/null http://127.0.0.1:1025/login
```

Red Hat Apache는 httpd 명령을 사용할 수 있습니다. Apache가80을 사용하면 Nginx를 중복 실행하지 않습니다. 컨테이너나 다른 호스트라면127.0.0.1이 올바른 앱 주소인지 확인합니다.

`sudo nginx -T`로 실제 include 구조를 확인할 수 있지만 출력에 기밀이 포함될 수 있어 그대로 공유하지 않습니다. 백업은 include 대상 밖에 두어 중복 읽기를 피합니다.

## 2. 구성 예시

```nginx
server {
    listen 80;
    server_name agri.example.org;
    location / {
        proxy_pass http://127.0.0.1:1025;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

이 예시는 요청 경로를 유지합니다. 앱의 신뢰할 프록시 범위를 제한하고 임의의 전달 헤더를 무조건 신뢰하지 않습니다. WebSocket·긴 요청·파일 업로드는 요구사항별 추가 구성이 필요할 수 있습니다.

## 3. 생성·활성화·적용

**[주의: 서비스 영향 가능]** 기존 사이트·포트와 충돌할 수 있습니다. 기존 파일과 서비스·방화벽 상태를 백업합니다. 다음은 sites-enabled를 실제로 include하는 Ubuntu/Debian식 예시입니다. conf.d 방식은 경로를 바꿔야 합니다.

```bash
sudo vi /etc/nginx/sites-available/agri
```

위 내용을 저장한 뒤 같은 이름의 파일/링크가 없는 경우에만:

```bash
sudo ln -s /etc/nginx/sites-available/agri /etc/nginx/sites-enabled/agri
sudo nginx -t
```

검사 성공, 기존 Nginx가 systemd 관리하에 실행 중인 경우에만:

```bash
sudo systemctl reload nginx
```

미설치·중지·다른 서비스 관리자 환경은 별도 도입 계획이 필요합니다. 외부에서TCP80을 사용할 수 있도록 방화벽 경로도 확인합니다.

## 4. 검증

```bash
curl --max-time 5 -sS -D - -o /dev/null -H 'Host: agri.example.org' http://127.0.0.1/login
```

Location에 내부IP나1025가 남는지 확인합니다. 외부 브라우저에서 로그인·로그아웃·화면 이동·API·파일을 실제로 확인합니다. HEAD를 지원하지 않는 앱도 있어 curl -I 실패만으로 장애를 단정하지 않습니다.

HTTPS는443·유효한 인증서·갱신·방화벽을 별도 설정합니다. 이 HTTP 예시만으로 로그인 통신 암호화가 완료되지 않습니다. 외부1025를 닫으려면 기존 이용자와 프록시 출발지를 조사한 뒤 별도 변경으로 수행합니다.

## 5. 롤백

이번에 새로 만든 sites-enabled/agri 링크만 비활성화합니다. 기존 파일을 수정했다면 백업으로 복원합니다. nginx -t 성공 후reload하고 기존 사이트와1025 직접 접속을 검증합니다. 새 방화벽 정책도 변경 기록을 따라 되돌립니다.

[예시 파일](../../../examples/nginx/agri.conf.example) · [공식 자료](../../../REFERENCES.md)

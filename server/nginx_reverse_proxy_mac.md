# macOS에서 Nginx로 Reverse Proxy(프록시) 서버 구축 --- 상세 가이드

아래는 **Homebrew로 설치한 nginx 기준**의 단계별 가이드입니다. 로컬
개발/테스트용으로 `8080/8443` 포트를 사용하고, 로컬 신뢰 가능한 HTTPS는
**mkcert**로 만드는 방법까지 포함했습니다. (Apple Silicon / Intel 경로
차이는 설명에 포함했습니다.)

------------------------------------------------------------------------

## 1) 전제 / 준비

-   Homebrew가 설치되어 있어야 합니다. (없으면
    `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
    로 설치)
-   터미널 사용 권한(관리자 권한)이 필요할 수 있음.
-   `brew install nginx mkcert` 를 사용할 것입니다.

------------------------------------------------------------------------

## 2) 설치

``` bash
# nginx 설치
brew install nginx

# mkcert 설치 (로컬 신뢰 인증서 생성용)
brew install mkcert
```

설치 후 Homebrew prefix 확인:

``` bash
brew --prefix
# 보통 Intel: /usr/local
# Apple Silicon: /opt/homebrew
```

편의상 아래에서 `BREW_PREFIX=$(brew --prefix)` 로 표기하겠습니다.

-   nginx 구성 파일: `$BREW_PREFIX/etc/nginx/nginx.conf`
-   nginx 로그: `$BREW_PREFIX/var/log/nginx/{access.log,error.log}`

nginx 버전/경로 확인:

``` bash
nginx -v
nginx -t     # 설정 문법 검사
```

서비스 시작(사용자 로컬 에이전트로 시작):

``` bash
brew services start nginx
# 중지: brew services stop nginx
# 재시작: brew services restart nginx
```

수동으로 실행(디버깅 용):

``` bash
nginx            # 시작
nginx -s reload  # 설정 reload
nginx -s stop    # 중지
```

------------------------------------------------------------------------

## 3) 기본 Reverse Proxy 예제 (HTTP, 포트 8080 사용 --- 루트 권한 불필요)

`$BREW_PREFIX/etc/nginx/nginx.conf` 파일 내 `http { ... }` 블록 안에
아래 server 블록을 추가하세요. (또는 `conf.d/myapp.conf` 형태로 추가해도
됩니다 --- 반드시 http 블록 안에 위치)

``` nginx
# 예: proxy -> 로컬 앱(포트 3000)
server {
    listen 8080;
    server_name myapp.local;

    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 지원
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_buffering off; # 실시간 스트리밍/SSE 필요시
        proxy_redirect off;
    }
}
```

`myapp.local`을 도메인으로 쓰려면 `/etc/hosts`에 매핑 추가:

``` bash
sudo sh -c 'echo "127.0.0.1 myapp.local" >> /etc/hosts'
# 또는 sudo nano /etc/hosts 로 직접 편집
```

설정 문법 검사 및 reload:

``` bash
nginx -t
brew services restart nginx
# 또는 nginx -s reload
```

테스트:

``` bash
curl -v http://myapp.local:8080/
```

------------------------------------------------------------------------

## 4) HTTPS (로컬 신뢰된 인증서) --- mkcert 사용

개발환경에서 브라우저에 신뢰된 HTTPS 테스트를 할 때 mkcert를 사용합니다.

1.  로컬 CA 설치 (한 번만):

``` bash
mkcert -install
```

2.  인증서 생성 (현재 디렉터리에 `.pem`과 `-key.pem` 파일 생성):

``` bash
mkcert myapp.local 127.0.0.1 localhost ::1
# 생성파일 이름은 mkcert 버전에 따라 myapp.local.pem / myapp.local-key.pem
```

3.  nginx HTTPS server 블록 (예: 포트 8443 사용 --- 루트 권한 불필요):

``` nginx
server {
    listen 8443 ssl;
    server_name myapp.local;

    ssl_certificate     /path/to/myapp.local.pem;
    ssl_certificate_key /path/to/myapp.local-key.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_buffering off;
    }
}
```

HTTP에서 HTTPS로 리디렉션(선택):

``` nginx
server {
    listen 8080;
    server_name myapp.local;
    return 301 https://$host:8443$request_uri;
}
```

테스트:

``` bash
curl -v https://myapp.local:8443/    # mkcert로 신뢰된 인증서이면 인증서 오류 없음
```

> 참고: 포트 `443`과 `80`은 루트 권한이 필요합니다. 안전하게 개발하려면
> `8080/8443` 같은 \>1024 포트를 쓰는 것을 권장합니다. 꼭 80/443을 써야
> 하면 방법(권장하지 않음): `sudo nginx`로 실행하거나 macOS의 `pfctl`로
> 포트 포워딩을 설정해 80-\>8080, 443-\>8443 리다이렉트할 수
> 있습니다(설정 복잡).

`pfctl` 예시(고급 --- 신중히):

``` text
# /etc/pf.anchors/nginx_rdr 파일에
rdr pass on lo0 inet proto tcp from any to any port 80 -> 127.0.0.1 port 8080
rdr pass on lo0 inet proto tcp from any to any port 443 -> 127.0.0.1 port 8443

# 로드
sudo pfctl -f /etc/pf.conf   # /etc/pf.conf에 anchor 로드 설정 필요
sudo pfctl -e                # pf 활성화
```

(설정 실수시 네트워크 문제가 생길 수 있으므로 주의)

------------------------------------------------------------------------

## 5) 로드밸런싱(여러 백엔드) 예제

``` nginx
upstream backend_pool {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
}

server {
    listen 8080;
    server_name myapp.local;

    location / {
        proxy_pass http://backend_pool;
        # (나머지 proxy_set_header 등 동일)
    }
}
```

------------------------------------------------------------------------

## 6) 로그 및 문제 해결 팁

-   로그 확인:

    ``` bash
    tail -f $(brew --prefix)/var/log/nginx/access.log
    tail -f $(brew --prefix)/var/log/nginx/error.log
    ```

-   설정 문법 검사:

    ``` bash
    nginx -t
    ```

-   포트 바인딩 에러 (`bind() ... permission denied`): 포트 \<1024 사용
    시 루트 권한 필요. 포트 변경 또는 pfctl 사용 고려.

-   누가 포트를 쓰는지 확인:

    ``` bash
    sudo lsof -i :80
    ```

-   프로세스 확인:

    ``` bash
    ps aux | grep nginx
    ```

------------------------------------------------------------------------

## 7) 전체 예시 (최종 --- nginx.conf의 http 블록 안에 넣을 수 있는 통합 예시)

``` nginx
# http { ... } 내부에 추가
upstream backend {
    server 127.0.0.1:3000;
}

server {
    listen 8080;
    server_name myapp.local;
    client_max_body_size 50M;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_buffering off;
        proxy_redirect off;
    }
}

server {
    listen 8443 ssl;
    server_name myapp.local;

    ssl_certificate     /Users/you/certs/myapp.local.pem;
    ssl_certificate_key /Users/you/certs/myapp.local-key.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://backend;
        # 동일한 proxy 헤더들
    }
}
```

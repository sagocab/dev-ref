# Windows에서 Nginx Reverse Proxy 서버 구축 가이드

## 1) 준비

-   Windows 10/11 (관리자 권한 필요할 수 있음)

-   **Nginx Windows용 바이너리** 다운로드:\
    👉 [Nginx 공식 사이트](https://nginx.org/en/download.html)\
    (Stable 버전 `nginx/Windows-xx.zip` 파일 다운로드)

-   **압축 해제**: `C:\nginx` 경로에 두는 것을 권장

------------------------------------------------------------------------

## 2) Nginx 실행/중지

``` powershell
# 실행 (nginx.exe 위치에서 실행)
cd C:\nginx
start nginx

# 중지
nginx -s stop

# 설정 reload
nginx -s reload
```

> Windows에서는 `brew services` 같은 서비스 관리자가 없으므로, 수동
> 실행하거나 **서비스 등록**해야 합니다.\
> 자동 시작이 필요하다면 `nssm`(Non-Sucking Service Manager) 같은 툴로
> Windows 서비스 등록 가능.

------------------------------------------------------------------------

## 3) 기본 Reverse Proxy 설정

`C:\nginx\conf\nginx.conf` 수정 → `http { ... }` 블록 안에 추가

``` nginx
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

        proxy_buffering off;
        proxy_redirect off;
    }
}
```

------------------------------------------------------------------------

## 4) hosts 파일 수정

로컬에서 `myapp.local` 도메인을 쓰려면:

    C:\Windows\System32\drivers\etc\hosts

파일에 아래 추가 (관리자 권한으로 열어야 함):

    127.0.0.1   myapp.local

------------------------------------------------------------------------

## 5) HTTPS 설정 (Self-Signed 또는 mkcert)

### (1) OpenSSL 방식 (직접 self-signed 인증서 생성)

1.  OpenSSL 설치 (예: [Git for
    Windows](https://git-scm.com/download/win) 설치 시 포함)

2.  인증서 생성:

    ``` powershell
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout myapp.key -out myapp.crt
    ```

3.  `C:\nginx\conf\certs` 폴더에 저장

4.  `nginx.conf`에 HTTPS server 블록 추가:

    ``` nginx
    server {
        listen 8443 ssl;
        server_name myapp.local;

        ssl_certificate     C:/nginx/conf/certs/myapp.crt;
        ssl_certificate_key C:/nginx/conf/certs/myapp.key;
        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_ciphers         HIGH:!aNULL:!MD5;

        location / {
            proxy_pass http://127.0.0.1:3000;
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

### (2) mkcert 방식 (Windows 지원)

1.  `choco install mkcert` (Chocolatey 필요)\
    또는 [mkcert
    release](https://github.com/FiloSottile/mkcert/releases) 다운로드

2.  로컬 CA 설치:

    ``` powershell
    mkcert -install
    ```

3.  인증서 생성:

    ``` powershell
    mkcert myapp.local 127.0.0.1 ::1
    ```

    → `myapp.local.pem`, `myapp.local-key.pem` 생성

4.  nginx에 적용:

    ``` nginx
    ssl_certificate     C:/nginx/conf/certs/myapp.local.pem;
    ssl_certificate_key C:/nginx/conf/certs/myapp.local-key.pem;
    ```

------------------------------------------------------------------------

## 6) 로그 확인

-   로그 파일 위치:

        C:\nginx\logs\access.log
        C:\nginx\logs\error.log

------------------------------------------------------------------------

## 7) 자동 시작 (선택 사항)

Windows에서는 서비스 등록이 필요합니다. `nssm` 같은 툴을 사용:

``` powershell
nssm install nginx "C:\nginx\nginx.exe"
nssm start nginx
```

------------------------------------------------------------------------

✅ 여기까지 하면 Windows에서도 **Reverse Proxy (HTTP/HTTPS, WebSocket
지원 포함)** 환경을 구축할 수 있습니다.

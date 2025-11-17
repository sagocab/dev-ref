## 1) UTM 네트워크 모드 선택 (간단한 방법)
### 가장 쉬운 방법은 UTM의 Shared Network (SLIRP) 사용. 
- 이 모드는 별도 네트워크 설정 없이 호스트(맥)에서 포트포워딩으로 게스트에 접근하게 해줌.
- UTM 앱에서 VM 선택 → 우측 상단 Edit(또는 VM → Settings) → Network 
- Network Mode를 Shared Network로 설정
(브리지 모드로 할 수도 있지만 네트워크 환경에 따라 추가 설정 필요 — 여기서는 Shared Network 기준)

## 2) UTM에서 Port Forwarding 설정
### 같은 Network 설정 화면에서 Port Forwarding 또는 Forwarding 섹션을 찾음
#### 새 규칙 추가:
- Protocol: TCP
- Host IP: 127.0.0.1 (로컬에서만 접속하려면) 또는 0.0.0.0 (다른 장치에서 접근 허용하려면)
- Host Port: 2222 (예시 — 호스트에서 쓸 포트)
- Guest Port: 22 (게스트의 SSH 포트)
- Name: 적당히 ssh 등
- 저장하고 VM 재시작(필요 시)
- 팁: Host Port는 2222 외에 사용 가능한 포트로 정해도 됨. 이미 2222가 잡혀있으면 다른 포트 사용.
```
UTM Port Forwarding 설정 확인
   | Name    |Protocol  | Host Address | Host Port | Guest Address | Guest Port |
   |---------|----------|--------------|-----------|---------------|------------|
   | SSh     | TCP      | 127.0.0.1    | 2222      | 0.0.0.0       | 22         |
   | jenkins | TCP      | 127.0.0.1    | 9191      | 0.0.0.0       | 9191       | 
   | mysql   | TCP      | 127.0.0.1    | 3306      | 127.0.0.1     | 3306       |    
```
## 3) 게스트(리눅스)에서 OpenSSH Server 설치 및 활성화
```linux
# 패키지 설치
sudo apt update
sudo apt install -y openssh-server

# 서비스 상태 확인
sudo systemctl status ssh

# 서비스 자동 시작 및 시작
sudo systemctl enable --now ssh

# 설치 후 SSH가 22번 포트에서 리스닝하는지 확인:
sudo ss -ltnp | grep :22
```

# 4) 맥에서 접속
맥 터미널에서:
```linux
ssh -p 2222 sagocab@localhost
```



```
$ sudo apt install openjdk-17-jdk
$ java -version

# jenkins 패키지를 배포하는 리포지토리를 등록
$ curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
$ echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
  
# jenkins 설치 명령
$ sudo apt update
$ sudo apt install jenkins

# jenkins 서비스 실행 상태를 확인
$ sudo systemctl status jenkins

# 서비스 포트를 9191 로 변경
$ sudo vi /etc/default/jenkins
HTTP_PORT=9191

$ sudo vi /etc/init.d/jenkins
JENKINS_PORT=9191

$ sudo vi /lib/systemd/system/jenkins.service
Environment="JAVA_OPTS=-Xmx1014 -Xms1024 -Djava.awt.headless=true"
Environment="JENKINS_PORT=9191"

# 젠킨스를 다시 시작
$ sudo systemctl stop jenkins
$ sudo systemctl daemon-reload
$ sudo systemctl start jenkins
```
### 젠킨스 웹 주소
http://localhost:9191/
-- 
id / pwd : sagocab/1004jung!

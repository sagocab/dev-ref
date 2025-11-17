## MySql 설치
```linux
sudo apt-get update

# MySql 설치
sudo apt-get install mysql-server

mysql --version

# MySql 실행
sudo systemctl start mysql

# 시스템 재시작시 MySql 시작
sudo systemctl enable mysql

# MySql 상태
systemctl status mysql

# MySql 접속 (패스워드 생략)
sudo mysql -u root -p

# 사용자 계정 생성
CREATE USER 'sagocab'@'%' IDENTIFIED BY '1234';

ALTER USER 'sagocab'@'%' IDENTIFIED BY '1234';

DROP USER 'sagocab'@'localhost';

CREATE USER 'root'@'_gateway' IDENTIFIED BY '1004jung!';

#권한 부여
grant all privileges on mydb.* to 'sagocab'@'localhost';

#MySql 나오기
exit;

#DB 생성
CREATE DATABASE sagocab;
```
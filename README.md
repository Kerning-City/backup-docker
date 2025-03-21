# 🚀 Ubuntu에서 crontab을 이용한 docker 데이터 백업 자동화

이 문서는 **🚀 Ubuntu에서 crontab을 이용한 docker 데이터 백업 자동화**을 정리한 것입니다.

## 👨‍👨‍👦‍👦 Team

👥 팀명 : 커닝시티

|<img src="https://avatars.githubusercontent.com/u/71498489?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/123963462?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/95984922?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/143996082?v=4" width="150" height="150"/>|
|:-:|:-:|:-:|:-:|
|HanJH<br/>[@letsgojh0810](https://github.com/letsgojh0810)|태규<br/>[@EOTAEGYU](https://github.com/EOTAEGYU)|nahong_c<br/>[@HongChan1412](https://github.com/HongChan1412)|Jihoon Kim<br/>[@wild-turkey](https://github.com/wild-turkey)|

<br><br>

### 중요 파일
#### docker-compose.yaml
```shell
version: "1.0"

services:
  db:
    container_name: mysqldb
    image: mysql:8.0  # 명확한 버전 지정
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    networks:
      - spring-mysql-net
    volumes:
      - /mnt/mysql:/var/lib/mysql
    healthcheck:
      test: ['CMD-SHELL', 'mysqladmin ping -h 127.0.0.1 -u root --password=$${MYSQL_ROOT_PASSWORD} || exit 1']
      interval: 10s
      timeout: 2s
      retries: 100

  app:
    container_name: springbootapp
    build:
      context: .
      dockerfile: ./Dockerfile
    ports:
      - "8080:8080"
    environment:
      MYSQL_HOST: db
      MYSQL_PORT: 3306
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    depends_on:
      db:
        condition: service_healthy
    networks:
      - spring-mysql-net

networks:
  spring-mysql-net:
    driver: bridge
```
> `- /mnt/mysql:/var/lib/mysql` mysql데이터가 저장되는 폴더를 host의 /mnt/mysql과 연결

#### start.sh
```shell
#!/bin/bash

# 필수 파일 목록
FILES=("docker-compose.yaml" "dockerfile" "step06_SpringDataJPA-0.0.1-SNAPSHOT.jar")

# 모든 파일이 존재하는지 확인
for file in "${FILES[@]}"; do
    if [[ ! -f "$file" ]]; then
        echo "Error: '$file' not found."
        exit 1
    fi
done

# 파일이 모두 존재하면 docker-compose 실행
echo "All required files are present. Starting Docker Compose..."
docker-compose up -d
```
> 필수 파일인 `docker-compose.yaml`, `dockerfile`, `step06_SpringDataJPA-0.0.1-SNAPSHOT.jar` 있는지 확인 후 daemon상태로 docker-compose 실행

#### backup.sh
```shell
#!/bin/bash

# 복사 대상 디렉터리 설정
SOURCE_DIR="/mnt/mysql"
DEST_DIR="/backup/mysql_$(date +%Y%m%d_%H%M%S)"

# 백업 디렉터리 생성
mkdir -p "$DEST_DIR"

# 폴더 복사
cp -r "$SOURCE_DIR" "$DEST_DIR"

# 복사 완료 메시지 출력
echo "Backup completed: $DEST_DIR"
```
> mnt된`/mnt/mysql`을 주기적으로 백업하기 위해 `/backup/mysql_$(date +%Y%m%d_%H%M%S)`으로 저장

#### crontab
```shell
*/5 * * * * bash /home/ubuntu/06.dockerCompose/backup.sh >> /var/log/backup.log 2>&1
```
> 5분마다 실행될 수 있도록 crontab에 등록

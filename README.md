
# 🚀 MySQL 데이터 덤프 및 복원 자동화

**목표:**
- `mysqldb` 컨테이너에서 데이터 덤프를 자동화합니다.
- 덤프된 파일을 이용하여 `newmysqldb` 컨테이너에 데이터를 복원합니다.
- 이 모든 과정을 스크립트 또는 Docker Compose로 자동화합니다.



## 🛠️ 스크립트를 사용한 자동화

### 1. 데이터 덤프 자동화 스크립트 작성 (`dump_data.sh`)

```bash
#!/bin/bash

# 변수 설정
CONTAINER_NAME=mysqldb
DUMP_FILE_PATH=./mysql_dump.sql
MYSQL_ROOT_PASSWORD=your_password

# 데이터 덤프 실행
docker exec $CONTAINER_NAME mysqldump -uroot -p$MYSQL_ROOT_PASSWORD --all-databases > $DUMP_FILE_PATH

echo "데이터 덤프 완료: $DUMP_FILE_PATH"
```

**설명:**  
- `mysqldump` 명령어로 모든 데이터베이스를 덤프하고, **mysql_dump.sql** 파일에 저장합니다.



### 2. 데이터 복원 자동화 스크립트 작성 (`restore_data.sh`)

```bash
#!/bin/bash

# 변수 설정
NEW_CONTAINER_NAME=newmysqldb
DUMP_FILE_PATH=./mysql_dump.sql
MYSQL_ROOT_PASSWORD=your_password

# 새로운 컨테이너 실행
docker run -d \
  --name $NEW_CONTAINER_NAME \
  -e MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD \
  mysql:latest

# MySQL 서버가 시작될 때까지 대기
echo "MySQL 서버 시작 대기 중..."
sleep 20

# 데이터 복원 실행
docker exec -i $NEW_CONTAINER_NAME mysql -uroot -p$MYSQL_ROOT_PASSWORD < $DUMP_FILE_PATH

echo "데이터 복원 완료: $NEW_CONTAINER_NAME"
```

**설명:**  
- `newmysqldb` 컨테이너를 실행하고, 덤프된 데이터를 복원합니다.



### 3. 스크립트 실행 권한 부여 및 실행

```bash
chmod +x dump_data.sh restore_data.sh

# 데이터 덤프 실행
./dump_data.sh

# 데이터 복원 실행
./restore_data.sh
```



## 📦 Docker Compose를 사용한 자동화

### 1. `docker-compose.yml` 파일 작성

```yaml
version: '3.8'

services:
  mysqldb:
    image: mysql:latest
    container_name: mysqldb
    environment:
      - MYSQL_ROOT_PASSWORD=your_password
    volumes:
      - mysql-data:/var/lib/mysql

  newmysqldb:
    image: mysql:latest
    container_name: newmysqldb
    environment:
      - MYSQL_ROOT_PASSWORD=your_password
    volumes:
      - mysql-data:/var/lib/mysql
    depends_on:
      - mysqldb

volumes:
  mysql-data:
```

**설명:**  
- 두 개의 컨테이너가 **mysql-data** 볼륨을 공유하며, `newmysqldb`는 `mysqldb`가 실행된 후 시작됩니다.



### 2. 데이터베이스 초기화 스크립트 (`init.sql`)

```sql
CREATE DATABASE IF NOT EXISTS testdb;
USE testdb;
CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(255));
INSERT INTO users (name) VALUES ('Alice'), ('Bob'), ('Charlie');
```

**설명:**  
- MySQL 컨테이너가 시작될 때 `testdb` 데이터베이스를 생성하고, 초기 데이터를 삽입합니다.



### 3. `docker-compose.yml`에 초기화 스크립트 추가

```yaml
services:
  mysqldb:
    # ... 이전 내용 유지
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  newmysqldb:
    # ... 이전 내용 유지
    volumes:
      - mysql-data:/var/lib/mysql
```



### 4. Docker Compose 실행

```bash
docker-compose up -d
```



### 5. 데이터 복원 확인

```bash
docker exec -it newmysqldb mysql -uroot -pyour_password -e "
USE testdb;
SELECT * FROM users;
"
```



## ⏲️ 자동화된 데이터 백업 및 복원 스케줄링

### 1. 데이터 백업 스크립트 (`backup_data.sh`)

```bash
#!/bin/bash

# 변수 설정
CONTAINER_NAME=mysqldb
BACKUP_DIR=./backups
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
MYSQL_ROOT_PASSWORD=your_password

# 백업 디렉토리 생성
mkdir -p $BACKUP_DIR

# 데이터 백업 실행
docker exec $CONTAINER_NAME mysqldump -uroot -p$MYSQL_ROOT_PASSWORD --all-databases > $BACKUP_DIR/mysql_backup_$TIMESTAMP.sql

echo "데이터 백업 완료: $BACKUP_DIR/mysql_backup_$TIMESTAMP.sql"
```



### 2. 크론탭에 스케줄 추가

```bash
crontab -e
```

다음 내용을 추가하여 **매일 새벽 2시**에 백업을 실행합니다.

```bash
0 2 * * * /path/to/backup_data.sh
```

---

## ✅ 전체 자동화 과정 요약

1. **스크립트를 사용하여 데이터 덤프 및 복원**:
   - `dump_data.sh`와 `restore_data.sh`를 통해 데이터를 자동으로 덤프하고 복원합니다.

2. **Docker Compose로 컨테이너 관리**:
   - `docker-compose.yml`을 사용해 컨테이너를 자동으로 실행하고, 초기화 스크립트를 통해 데이터베이스와 테이블을 설정합니다.

3. **스케줄러를 사용한 데이터 백업**:
   - `backup_data.sh`를 크론탭에 등록하여 주기적인 데이터 백업을 자동화할 수 있습니다.


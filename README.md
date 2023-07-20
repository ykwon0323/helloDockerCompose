Docker Compose 사용해보기 - Docker 튜토리얼 자습서
======================
[참고 : Tutorial Page](https://docs.docker.com/compose/gettingstarted/)

> 전제조건
> - Docker Engine 및 Docker Compose를 독립 실행형 바이너리로 설치
> - Docker Engine 및 Docker Compose를 모두 포함하는 Docker Desktop 설치

# 1단계: 애플리케이션 종속성 정의
1 프로젝트의 디렉터리를 만든다
``` shell
    mkdir composetest
    cd composetest
```
2 프로젝트 디렉터리에서 호출되는 파일을 만들고 app.py 다음 코드를 붙여넣는다
``` python
import time

import redis
from flask import Flask

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)

```
3 프로젝트 디렉터리에서 호출되는 다른 파일을 만들고 requirements.txt 다음 코드를 붙여넣는다
``` txt
flask
redis

```
# 2단계: Dockerfile 만들기
- Dockerfile - Docker 이미지를 빌드하는 데 사용
- 이미지에는 Python 자체를 포함하여 Python 애플리케이션에 필요한 모든 종속성이 포함되어 있음

프로젝트 디렉터리에서 이름이 지정된 파일을 만들고 Dockerfile 다음코드를 붙여넣는다
```Docker
# syntax=docker/dockerfile:1
FROM python:3.7-alpine
WORKDIR /code
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
RUN apk add --no-cache gcc musl-dev linux-headers
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
EXPOSE 5000
COPY . .
CMD ["flask", "run"]
```
해석
- Python 3.7 이미지로 시작하는 이미지르 ㄹ빌드
- 작업 디렉토리를 설정 `/code`
- 명령에서 사용하는 환경변수를 설정 `flask`
- gcc 및 기타 종속성 설치
- `requirements.txt` Python 종속 항목을 복사하고 설치 
- 컨테이너가 포트 5000에서 수신 중임을 설명하는 메타데이터르 ㄹ이미지에 추가
- `.` 프로젝트의 현재 디렉터리를 `.`이미지의 workdir에 복사
- 컨테이너의 기본 명령을 설정 `flask run`

# 3단계: Compose 파일에서 서비스 정의
- `docker-compose.yml` 프로젝틍 디렉터리에 파일을 만들고 다음을 붙여넣음
```yml
services:
  web:
    build: .
    ports:
      - "8000:5000"
  redis:
    image: "redis:alpine"
```
- 이 Compose 파일은 두가지 서비스를 정의함 `web` 및 `redis`
- 서비스는 현재 디렉터리 `web`에서 빌드된 이미지를 사용
- `Dockerfile` 그런 다음 컨테이너와 호스트 시스템을 노출된 포트에 바인딩 `8000`
- 이 예제 서비스는 Flask 앱 서버의 기본 포트인 `5000`
- 이 서비스는 Docker Hub 레지스트리에서 가져온 `redis` 공용 Redis 이미지를 사용

# 4단계: Compose로 앱 빌드 및 실행
## 1 프로젝트 디렉터리에서 `docker compose up`을 실행하여 애플리케이션을 시작
```shell
docker compose up
```
- Compose는 Redis 이미지를 가져오고, 코드용 이미지를 빌드하고, 정의한 서비스를 시작합니다
- 이 경우 빌드 시 코드가 이미지에 정적으로 복사됨

## 2 실행 중인 응용 프로그램을 보려면 
http://localhost:8000/ 을 접속함 

## 3 페이지를 새로 고침함
숫자가 증가해야함

# 5단계: Compose 파일을 편집하여 바인드 마운트 추가
- `docker-compose.yml`프로젝트 디렉터리에서 편집하여 서비스에 대한 바인드 마운트 `web` 추가
```yml

services:
  web:
    build: .
    ports:
      - "8000:5000"
    volumes:
      - .:/code
    environment:
      FLASK_DEBUG: "true"
  redis:
    image: "redis:alpine"
```
새 `voulumes`키는 호스트의 프로젝트 디렉터리(현재 디렉터리)를 `/code` 컨테이너 내부에 탑재하므로 이미지를 다시 빌드하지 않고도 코드를 즉시 수정할 수 있음
키는 개발 모드에서 실행하고 변경 시 코드를 다시 로드하도록 지시하는 환경변수를 `environment`설정
이 모드는 개발 시에만 사용해야 함
`FLASK_DEBUG flas run`

# 6단계: Compose로 앱 다시 빌드 및 실행
프로젝트 디렉터리에서 `docker compose up` 업데이트된 Compose 파일로 앱 빌드를 입력하고 실행
```
docker compose up
```

# 7단계: 애플리케이션 업데이트
이제 애플리케이션 코드가 볼륨을 사용하여 컨테이너에 마운트 되기 때문에 이미지를 다시 빌드하지 않고도 코드를 변경하고 변경 사항을 즉시 확인할 수 있음
인사말을 변경 `app.py`하고 저장 
``` python
return 'Hello from Docker! I have been seen {} times. \n'.format(count)
```
# 8단계: 다른 명령으로 실험하기
백그라운드에서 서비스를 실행하려면 플래그 `-d`("분리" 모드용)를 에 전달 `docker compose up`하고 `docker compose ps` 현재 실행 중인 항목을 확인하는데 사용할 수 있음 
``` shell
docker compose up -d

Starting composetest_redis_1...
Starting composetest_web_1...
docker compose ps

       Name                      Command               State           Ports         
-------------------------------------------------------------------------------------
composetest_redis_1   docker-entrypoint.sh redis ...   Up      6379/tcp              
composetest_web_1     flask run                        Up      0.0.0.0:8000->5000/tcp
```
`docker compose run`명령을 사용하면 서비스에 대해 일회성 명령을 실행할 수 있음
```
docker compose run web env
```
`docker compose --help` 사용 가능한 다른 명령을 보려면 참조
`docker compose up -d`작업을 마친후 서비스를 중지
```shell
docker compose stop
```
명령을 사용하여 모든것을 중단하고 컨테이너를 완전히 제거 할 수 있음
`down`, `--volumes` Redis 컨테이너에서 사용하는 데이터 볼륨도 제거하렴녀 전달
```
docker compose down --volumes
```
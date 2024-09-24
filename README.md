# 🚀 Docker Image Optimization을 통한 Spring Boot 어플리케이션 성능 개선

## 🔧 개요

Docker 이미지를 최적화하는 것은 효율적인 리소스 활용, 더 빠른 배포 및 보안 강화를 위해 필수적입니다. Docker 이미지를 최적화하기 위한 방법들을 읽고 적용할 수 있는 방법들을 찾아보고 프로젝트에 적용해 보았습니다.

## 💡 제안된 방법

1. **경량화된 베이스 이미지 사용**: 최소한의 구성 요소를 가진 Alpine Linux와 같은 이미지를 사용해 이미지 크기를 줄이고 보안을 강화합니다.
2. **단일 책임 원칙**: 각 Docker 이미지가 하나의 역할만 수행하게 하여 관리와 확장성을 높입니다.
3. **멀티스테이지 빌드**: 빌드 시간에 필요한 의존성은 제거하고, 최종 프로덕션 이미지는 경량화하여 불필요한 파일과 의존성을 포함하지 않습니다.
    - 단일 Dockerfile에 다중 FROM 문을 적용하는 것을 멀티스테이지 빌드라고 합니다.
    
    `예시`
    
    ```bash
    # Build stage
    FROM node:14-alpine as build
    
    WORKDIR /app
    COPY package*.json ./
    RUN npm install
    COPY . .
    RUN npm run build
    
    # Production stage
    FROM node:14-alpine as production
    
    WORKDIR /app
    COPY --from=build /app/package*.json ./
    RUN npm ci --production
    COPY --from=build /app/dist ./dist
    
    CMD ["npm", "start"]
    ```
    
4. **레이어 최소화**: 여러 명령어를 하나의 `RUN` 명령어로 결합해 Docker 이미지 레이어 수를 줄이고 이미지 크기를 줄입니다.
5. **.dockerignore 사용**: 필요 없는 파일과 디렉토리를 빌드에서 제외해 빌드 시간과 이미지 크기를 줄입니다.
6. **특정 태그 사용**: `latest` 대신 특정 버전 태그를 사용해 빌드 일관성을 유지합니다.
7. **이미지 검토 및 보안 스캔**: 보안 취약점을 분석하고 이미지를 주기적으로 점검하여 안전하게 유지합니다.
8. **캐싱 구현:** Dockerfile을 캐시 활용도를 극대화하는 방식으로 구조화하여 Docker의 빌드 캐시를 활용합니다.

## 🧰 Spring Boot 애플리케이션 최적화

위 방법들을 토대로 실제 프로젝트에 적용할 수 있는 부분을 찾아보았습니다.

기존 Spring Boot 프로젝트에서 Docker를 통해 CI/CD를 적용했던 경험을 토대로 해당 프로젝트에서 사용했던 Dockerfile을 개선하여 성능을 최적화하고자 합니다.

### 📉 기존 Dockerfile

```bash
FROM openjdk:17-jdk-alpine
ARG JAR_FILE=build/libs/*.jar
COPY ${JAR_FILE} app.jar
COPY src/main/resources/application.yml /app/config/application.yml

ENTRYPOINT ["java","-jar","/app.jar", "--spring.config.location=file:/src/main/resources/application.yml"]
EXPOSE 8080

```

해당 Dockerfile의 단점은 다음과 같습니다.

1. **빌드와 실행 환경 분리**
    
    빌드 과정을 포함하지 않고 미리 빌드된 JAR 파일을 복사하여 실행하는 방식입니다. 멀티스테이지 빌드를 통해 빌드 도구와 프로덕션 실행환경을 구분하고자 합니다.
    
2. **유지보수성 개선**
    
    dockerfile 개발 당시 환경 변수 주입을 위해 application.yml 파일을 직접 이미지에 포함시켜 빌드를 진행했습니다. 이는 설정을 변경하기 위한 유연성이 부족하다는 단점이 있습니다.
    
3. **캐시 최적화**
    
    소스 파일 복사와 빌드 과정이 따로 이루어지지 않으므로 의존성 캐시를 활용할 수 없습니다.
    
    - `의존성 캐시` : 빌드 과정에서 한 번 설치된 외부 라이브러리나 모듈을 저장해, 이후 빌드 시 다시 다운로드하지 않고 빌드 시간을 줄이는 메커니즘

### 📈 개선된 Dockerfile 및 docker-compose.yml

**Dockerfile**

```bash
# 1. Build Stage: 애플리케이션 빌드 및 의존성 설치
FROM gradle:8.1.0-jdk17 AS build
WORKDIR /app

# 의존성 파일을 먼저 복사하여 캐시 최적화
COPY build.gradle settings.gradle ./
RUN gradle build --no-daemon

# 소스 파일을 복사한 후 빌드
COPY src ./src
RUN gradle bootJar --no-daemon

# 2. Production Stage: 경량화된 런타임 이미지
FROM openjdk:17-jdk-slim
WORKDIR /app

# 빌드된 애플리케이션 JAR 파일 복사
COPY --from=build /app/build/libs/*.jar app.jar

# 환경변수로 설정을 주입하는 방식
ENTRYPOINT ["java","-jar","/app/app.jar"]
EXPOSE 8080

```

**docker-compose.yml**

```bash
version: '3.8'
services:
  app:
    image: my-spring-app:latest
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: **
      SPRING_DATASOURCE_USERNAME: **
      SPRING_DATASOURCE_PASSWORD: ** 
      SPRING_DATASOURCE_DRIVER_CLASS_NAME: com.mysql.cj.jdbc.Driver
    volumes:
      - ./src/main/resources/application.yml:/app/config/application.yml
      - ./src/main/resources/application-secrets.yml:/app/config/application-secrets.yml
    depends_on:
      - db
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: **
      MYSQL_DATABASE: **
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:

```

**개선 과정**

1. **멀티스테이지 빌드를 통한 빌드와 실행 환경 분리**
    
    멀티스테이지 빌드를 사용하여 빌드와 프로덕션 환경을 분리하였습니다. 이는 빌드 도구나 불필요한 파일이 최종 실행 이미지에 포함되지 않아 이미지 크기가 작아지고, 배포가 더 효율적입니다.
    
2. **의존성 캐시 활용 최적화**
    
    의존성 관련 파일을 먼저 복사하고 의존성을 설치한 후에 소스 파일을 복사하는 방식으로, Gradle 캐시를 활용하여 빌드 시간을 대폭 감소하였습니다.
    
    빌드 시 변경되지 않은 의존성에 대해 캐시를 재사용해 빌드 성능을 향상시켰습니다.
    
3. **환경 파일 분리와 유지보수성 증대**
    
    docker compose를 적용하여 환경 별 설정 관리를 유연하게 진행할 수 있었습니다. 설정 변경 시에도 이미지를 다시 빌드할 필요가 없으며, 유지보수성 향상과 환경 설정의 유연성을 확보하였습니다.
    
** 추가 개선점** 

보안을 위해 환경 변수를 `.env` 파일로 분리하여 작성하는 것이 필요합니다.

### ⏳ 성능 측정

**기존 빌드 시간**

![image](https://github.com/user-attachments/assets/7b6e8ee3-4906-4d62-a3e3-27215c3b41e0)


**개선 후 빌드 시간**

![image](https://github.com/user-attachments/assets/e9471c19-166d-4ff8-b999-44440a26038a)

빌드 시간이 기존 3.1초에서 1.5초로 약 50% 가량 감소하였습니다.

또한 캐시 사용률과 병렬 처리 사용률이 증가한 것을 확인하여, 의존성이 많은 대형 프젝트일수록 해당 방법을 통한 성능 개선이 대폭 증대할 것으로 예상됩니다.

## 🏁 결론

Docker 이미지 최적화는 경량화된 베이스 이미지 사용, **멀티스테이지 빌드, 의존성 캐시 활용, 환경 파일의 분리**를 통해 애플리케이션의 빌드 성능과 배포 효율성을 크게 향상시키는 방법입니다. 기존 프로젝트에 해당 최적화 방안을 적용한 결과 빌드 시간이 약 50% 단축되었습니다.

이미지 경량화와 의존성 캐싱 활용을 통해 성능 개선을 실제로 체감할 수 있었습니다. 의존성이 많은 대형 프로젝트에서는 이러한 최적화로 빌드 속도 향상과 배포 효율성을 높일 수 있으며, 유연한 환경 설정 구성으로 운영 및 유지보수 측면에서도 긍정적인 영향을 미칠 것으로 기대됩니다.

## 🗒️ 참고자료

https://overcast.blog/docker-image-optimization-tips-tricks-6a17f687162b

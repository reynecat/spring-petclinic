# Spring PetClinic - Web/WAS 분리 아키텍처

## 개요

본 프로젝트는 웹 서버(Web)와 애플리케이션 서버(WAS)를 분리한 2-tier 아키텍처로 구성되어 있습니다. 이러한 분리는 성능, 확장성, 그리고 관심사의 분리(Separation of Concerns) 측면에서 이점을 제공합니다.

## 아키텍처 구성

```
[클라이언트]
    ↓
[Web 서버 - Nginx]  (포트 80)
    ↓
[WAS - Spring Boot] (포트 8080)
    ↓
[Database - PostgreSQL/MySQL]
```

## 1. Web 서버 (Nginx)

### 역할
- **리버스 프록시**: 모든 클라이언트 요청을 받아 WAS로 전달
- **정적 자원 캐싱**: 이미지, CSS, JS 파일 등의 캐싱 처리
- **로드 밸런싱**: 여러 WAS 인스턴스로 트래픽 분산 가능
- **헬스체크**: 웹 서버 상태 모니터링 엔드포인트 제공

### 구현 파일

#### [Dockerfile.web](Dockerfile.web)
```dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
```

- 경량 Alpine Linux 기반 Nginx 이미지 사용
- 포트 80으로 HTTP 요청 수신

#### [nginx.conf](nginx.conf)
주요 설정:

1. **업스트림 설정**
   ```nginx
   upstream backend {
       server was-service.was.svc.cluster.local:8080;
   }
   ```
   - Kubernetes 환경에서 WAS 서비스에 연결
   - DNS 이름: `was-service.was.svc.cluster.local`

2. **헬스체크 엔드포인트**
   ```nginx
   location /health {
       return 200 "healthy\n";
   }
   ```
   - `/health` 경로로 웹 서버 상태 확인

3. **정적 자원 캐싱**
   ```nginx
   location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
       proxy_pass http://backend;
       proxy_cache_valid 200 1h;
       expires 1h;
   }
   ```
   - 이미지, CSS, JS 파일은 1시간 캐싱
   - WAS 부하 감소 및 응답 속도 향상

4. **프록시 설정**
   ```nginx
   location / {
       proxy_pass http://backend;
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   }
   ```
   - 모든 요청을 WAS로 전달
   - 원본 클라이언트 IP 및 호스트 정보 헤더에 포함
   - 60초 타임아웃 설정

## 2. WAS (Web Application Server)

### 역할
- **비즈니스 로직 처리**: Spring Boot 애플리케이션 실행
- **동적 콘텐츠 생성**: Thymeleaf 템플릿 렌더링
- **데이터베이스 연동**: JPA/Hibernate를 통한 DB 처리
- **API 엔드포인트 제공**: RESTful API 서비스

### 구현 파일

#### [Dockerfile.was](Dockerfile.was)
```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/spring-petclinic-4.0.0-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- Eclipse Temurin Java 21 JRE 사용
- Spring Boot JAR 파일 실행
- 포트 8080으로 HTTP 요청 수신

### Spring Boot 애플리케이션
- **메인 클래스**: [PetClinicApplication.java](src/main/java/org/springframework/samples/petclinic/PetClinicApplication.java)
- **설정 파일**: [application.properties](src/main/resources/application.properties)
- **프로파일**: H2(기본), MySQL, PostgreSQL 지원

## 3. Kubernetes 배포 구성

### Web 서버 배포
Kubernetes에서 Web과 WAS는 별도의 네임스페이스와 서비스로 분리하여 배포됩니다:

```yaml
# Web 서버는 별도 네임스페이스에 배포되며
# nginx.conf의 upstream 설정을 통해 WAS와 연결
upstream backend {
    server was-service.was.svc.cluster.local:8080;
}
```

### WAS 배포 - [k8s/petclinic.yml](k8s/petclinic.yml)

#### Service 정의
```yaml
apiVersion: v1
kind: Service
metadata:
  name: petclinic
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
```
- NodePort 타입으로 외부 접근 가능
- 포트 80 → 컨테이너 포트 8080 매핑

#### Deployment 정의
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: petclinic
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: workload
          image: dsyer/petclinic
          ports:
            - containerPort: 8080
          livenessProbe:
            httpGet:
              path: /livez
              port: http
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
```

주요 기능:
- **헬스체크**: `/livez` (liveness), `/readyz` (readiness) 엔드포인트
- **데이터베이스 연결**: Secret을 통한 PostgreSQL 인증 정보 주입
- **프로파일**: `SPRING_PROFILES_ACTIVE=postgres` 환경 변수 설정

## 4. 데이터베이스 구성

### Kubernetes DB 배포 - [k8s/db.yml](k8s/db.yml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-db
spec:
  ports:
    - port: 5432
  selector:
    app: demo-db
```

- PostgreSQL 18.1 이미지 사용
- Secret을 통한 인증 정보 관리
- Service Binding 규격 지원

### Docker Compose DB - [docker-compose.yml](docker-compose.yml)

로컬 개발 환경용:
```yaml
services:
  mysql:
    image: mysql:9.5
    ports:
      - "3306:3306"
  postgres:
    image: postgres:18.1
    ports:
      - "5432:5432"
```

## 5. 분리 아키텍처의 장점

### 성능
- **정적 자원 캐싱**: Nginx가 정적 파일을 캐싱하여 WAS 부하 감소
- **연결 관리**: Nginx의 효율적인 커넥션 풀링

### 확장성
- **독립적 스케일링**: Web과 WAS를 각각 독립적으로 확장 가능
- **로드 밸런싱**: Nginx upstream을 통한 여러 WAS 인스턴스 지원

### 보안
- **외부 노출 최소화**: WAS는 내부 네트워크에만 노출
- **SSL/TLS 종료**: Nginx에서 SSL 처리 가능
- **헤더 제어**: X-Forwarded-* 헤더를 통한 원본 정보 보존

### 유지보수성
- **관심사 분리**: 웹 서버와 애플리케이션 로직 분리
- **독립적 배포**: Web과 WAS를 개별적으로 배포 및 업데이트 가능
- **모니터링**: 각 레이어별 독립적인 헬스체크

## 6. 배포 방법

### 로컬 개발
```bash
# WAS 빌드
./mvnw clean package

# Docker 이미지 빌드
docker build -f Dockerfile.was -t petclinic-was .
docker build -f Dockerfile.web -t petclinic-web .

# 컨테이너 실행
docker run -d -p 8080:8080 --name was petclinic-was
docker run -d -p 80:80 --name web petclinic-web
```

### Kubernetes 배포
```bash
# 데이터베이스 배포
kubectl apply -f k8s/db.yml

# WAS 배포
kubectl apply -f k8s/petclinic.yml

# Web 서버는 별도 네임스페이스에 배포 (설정 파일 필요)
```

## 7. 네트워크 플로우

1. **클라이언트 요청** → Nginx (포트 80)
2. **Nginx** → 요청 타입 확인
   - 정적 자원 (이미지, CSS, JS): 캐시 확인 → 캐시 미스 시 WAS 호출
   - 동적 콘텐츠: WAS로 프록시
3. **WAS** (포트 8080) → 요청 처리
   - 비즈니스 로직 실행
   - 필요시 데이터베이스 쿼리
4. **응답** → Nginx → 클라이언트

## 8. 모니터링 엔드포인트

### Web 서버
- `/health`: Nginx 상태 확인

### WAS
- `/livez`: 애플리케이션 생존 여부
- `/readyz`: 트래픽 수신 준비 상태
- `/actuator/*`: Spring Boot Actuator 엔드포인트

## 참고사항

- 현재 Kubernetes 설정([k8s/petclinic.yml](k8s/petclinic.yml))은 Web/WAS 통합 구성입니다
- 완전한 분리 배포를 위해서는 Web 서버용 별도 Deployment와 Service가 필요합니다
- [nginx.conf](nginx.conf)는 이미 분리된 환경을 가정한 설정을 포함하고 있습니다 (upstream: `was-service.was.svc.cluster.local`)

# KopoChat

Spring Boot + WebSocket(STOMP) 기반 실시간 채팅 서비스입니다. JWT 로그인, 1:1 DM, 그룹 채팅방, 초대/나가기, 읽음 처리를 지원합니다.

## 기술 스택

- Java 21, Spring Boot 3.5
- Spring Web, Spring WebSocket (STOMP + SockJS), Spring Security
- Spring Data JPA + MySQL
- JWT (jjwt 0.12.6)
- 프론트엔드: `src/main/resources/static/index.html` (정적 페이지)

## 로컬 실행

### 1. MySQL 준비

```sql
CREATE DATABASE kopochat;
```

테이블은 `spring.jpa.hibernate.ddl-auto=update` 설정에 따라 기동 시 자동 생성됩니다.

### 2. 로컬 설정 파일 작성

`src/main/resources/application-local.properties`를 만듭니다. 이 파일은 `.gitignore`에 등록되어 있으므로 커밋되지 않습니다.

```properties
spring.datasource.password=<MySQL 비밀번호>
JWT_SECRET=<Base64 인코딩된 32바이트 이상 키>
```

키는 `openssl rand -base64 32` 명령으로 만들 수 있습니다.

### 3. 실행

```bash
./gradlew bootRun --args='--spring.profiles.active=local'
```

`local` 프로파일 없이 실행하면 `JWT_SECRET` 값을 찾지 못해 기동이 실패합니다. 실행 후 http://localhost:8080 으로 접속합니다.

## 환경 변수

| 변수 | 기본값 | 설명 |
|---|---|---|
| `DB_URL` | `jdbc:mysql://localhost:3306/kopochat` | JDBC URL |
| `DB_USERNAME` | `root` | DB 사용자 |
| `SPRING_DATASOURCE_PASSWORD` | – | DB 비밀번호 |
| `JWT_SECRET` | – | JWT 서명 키 (필수) |

## API

### 인증 `/api/auth`

| 메서드 | 경로 | 설명 |
|---|---|---|
| POST | `/api/auth/signup` | 회원가입 |
| POST | `/api/auth/login` | 로그인 (JWT 발급) |
| POST | `/api/auth/logout` | 로그아웃 |

### 채팅 `/api/chat`

| 메서드 | 경로 | 설명 |
|---|---|---|
| GET | `/api/chat/users` | 사용자 목록 |
| GET | `/api/chat/rooms` | 내 채팅방 목록 |
| POST | `/api/chat/dm` | 1:1 DM 방 생성 또는 조회 |
| POST | `/api/chat/rooms` | 그룹 채팅방 생성 |
| POST | `/api/chat/rooms/{roomId}/invite` | 채팅방 초대 |
| GET | `/api/chat/rooms/{roomId}/members` | 채팅방 멤버 조회 |
| GET | `/api/chat/rooms/{roomId}/messages` | 메시지 이력 조회 |
| POST | `/api/chat/rooms/{roomId}/read` | 읽음 처리 |
| DELETE | `/api/chat/rooms/{roomId}/leave` | 채팅방 나가기 |

### WebSocket (STOMP)

- 연결 엔드포인트: `/ws-chat` (SockJS)
- 메시지 전송: `/app/chat.send`
- 구독 대상:
  - `/topic/room/{roomId}`: 해당 방의 새 메시지
  - `/topic/notify`: 전체 알림 (`NEW_MESSAGE` 이벤트)

## 프로젝트 구조

```
src/main/java/com/kopo/kopochat/
├── KopoChatApplication.java
├── SecurityConfig.java, JwtFilter.java, JwtUtil.java   # 인증
├── WebSocketConfig.java                                # STOMP 설정
├── AuthController.java                                 # 회원가입/로그인
├── ChatRoomController.java                             # 채팅방 REST API
├── ChatController.java                                 # 메시지 송수신
└── User, ChatRoom, ChatRoomMember, Message (+ Repository)
```

## 빌드와 배포

### Docker 이미지

```bash
docker build -t kopochat .
```

멀티 스테이지 빌드를 사용합니다. `gradle:jdk21`에서 bootJar를 만들고 `eclipse-temurin:21-jre`에서 실행합니다.

### CI/CD 흐름

```
git push → Jenkins (Jenkinsfile)
  ├─ Kaniko로 이미지 빌드 → std-harbor.kopoctc.kr/kopo02/kopochat:v<BUILD_NUMBER>
  └─ gitops 저장소(kopo021/gitops)의 apps/kopochat/deployment.yaml 이미지 태그 갱신
        → ArgoCD auto-sync (prune + selfHeal) → vcluster kopo02 / kopochat 네임스페이스
```

- 배포 매니페스트의 기준은 **`kopo021/gitops`의 `apps/kopochat/`** 한 곳입니다. 다른 매니페스트 저장소(`my-gitops`, `kopochat-manifest`)는 클러스터에 반영되지 않습니다.
- 클러스터를 `kubectl edit`나 `kubectl apply`로 직접 수정하지 마세요. selfHeal이 Git 상태로 되돌립니다.
- gitops 저장소에 수동으로 push할 때는 먼저 `git pull --rebase origin main`을 실행하세요. force push는 금지입니다.

### 시크릿

클러스터의 DB 계정과 JWT 키는 SealedSecrets(`mysql-secret`: `username`, `password`, `rootPassword`, `jwtSecret`)로 관리합니다. 값을 바꿀 때는 `kubeseal`로 다시 봉인한 뒤 gitops의 `apps/kopochat/mysql-sealedsecret.yaml`을 갱신합니다. 로컬과 클러스터의 JWT 키는 서로 다르므로, 로컬에서 발급받은 토큰은 클러스터에서 쓸 수 없습니다.

## 저장소

- GitHub: https://github.com/Shane-Ko/KopoChat (`origin`)
- GitLab: https://std-gitlab.kopoctc.kr/kopo02/kopochat (`gitlab`)

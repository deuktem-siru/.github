# 배포 가이드 — 득템시루

[문서 안내](../README.md#관련-문서)

EC2(Ubuntu 22.04)에서 systemd로 `/opt/deuktemsiru/app.jar`를 실행하고 RDS PostgreSQL 16에 연결하는 절차다.
`application-prod.properties`가 DB 접속 정보를 환경 변수로 받으며, 시작 시 Flyway 적용 후 JPA 스키마를 검증한다.
로컬 실행은 [Backend README](https://github.com/deuktem-siru/backend#시작하기), S3 전환 계획은 [구현 계획](../plan.md)를 따른다.

## 1. AWS 리소스

| 리소스 | 설정 |
| --- | --- |
| 보안 그룹 `sg-app` (EC2) | 22(내 IP), 8080(ALB 보안 그룹 또는 지정 대역) |
| 보안 그룹 `sg-db` (RDS) | 5432, 소스는 `sg-app`만 |
| RDS | PostgreSQL 16, 퍼블릭 액세스 No, 초기 DB `deuktemsiru` |
| EC2 | Ubuntu 22.04, `t3.small` 이상, 탄력적 IP 연결 |

앱 릴리스에는 HTTPS가 필요하다. Route53 도메인 + ACM 인증서 + ALB(443 → EC2 8080)를 구성하고, EC2의 8080 인바운드는 ALB 보안 그룹만 허용한다.

### 배포 전 DB 준비

현재 Flyway V1~V5는 기존 스키마의 변경분이며 초기 테이블 생성 SQL이 없다. 빈 RDS나 새 Compose 볼륨에 기본 `prod` 설정으로 실행하면 배포가 완료되지 않는다.

첫 배포 전에 JPA 엔티티와 V1~V5에 맞는 초기 스키마를 준비하고, 별도 테스트 DB에서 Flyway 적용과 `ddl-auto=validate` 기동을 확인한다. 초기 스키마 생성 SQL과 자동화 절차는 아직 마련되지 않았으므로 이 검증을 마친 DB를 배포 전제 조건으로 둔다. 개발용 `create-drop` 설정을 운영 DB에 적용하지 않는다.

## 2. EC2 초기 셋업

```bash
sudo apt update && sudo apt install -y openjdk-21-jre-headless
sudo useradd -r -m -s /bin/bash app
sudo mkdir -p /opt/deuktemsiru && sudo chown app:app /opt/deuktemsiru

sudo tee /etc/deuktemsiru.env >/dev/null <<'ENV'
SPRING_PROFILES_ACTIVE=prod
SPRING_DATASOURCE_URL=jdbc:postgresql://<RDS_ENDPOINT>:5432/deuktemsiru
SPRING_DATASOURCE_USERNAME=<USER>
SPRING_DATASOURCE_PASSWORD=<PASSWORD>
APP_JWT_SECRET=<64자 이상 랜덤 문자열>
ENV
sudo chmod 600 /etc/deuktemsiru.env && sudo chown app:app /etc/deuktemsiru.env

sudo tee /etc/systemd/system/deuktemsiru.service >/dev/null <<'UNIT'
[Unit]
Description=Deuktemsiru Backend
After=network.target
[Service]
User=app
WorkingDirectory=/opt/deuktemsiru
EnvironmentFile=/etc/deuktemsiru.env
ExecStart=/usr/bin/java -jar /opt/deuktemsiru/app.jar
SuccessExitStatus=143
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
UNIT
sudo systemctl daemon-reload && sudo systemctl enable deuktemsiru
```

`<...>`는 실제 운영 값으로 바꾼다. FCM 사용 시 `APP_FCM_ENABLED=true`와 서비스 계정 JSON 경로인 `GOOGLE_APPLICATION_CREDENTIALS`도 환경 파일에 설정한다.
운영자 승인·정산 API를 쓰려면 `APP_ADMIN_TOKEN=<32자 이상 랜덤 문자열>`도 추가한다. 설정하지 않으면 `/api/v1/admin/**`은 항상 401이다.

## 3. 수동 배포

자동 배포(CD) 워크플로는 사용하지 않으며, 서버 배포는 수동으로 진행한다.

로컬 `backend/`에서 JDK 21·Docker를 준비하고 실행한다. `<...>`는 실제 경로·호스트로 바꾼다.

```bash
./gradlew clean check bootJar
# build/libs/에서 *-plain.jar를 제외한 실행 JAR을 지정한다.
scp -i <SSH_KEY_PATH> <BOOT_JAR_PATH> ubuntu@<EC2_HOST>:/home/ubuntu/app.jar
ssh -i <SSH_KEY_PATH> ubuntu@<EC2_HOST>
```

EC2에서 아래 블록을 실행한다. JAR 교체 중에는 서비스를 중지하므로 잠시 중단되며, 헬스체크가 실패하면 로그를 출력한다.

```bash
bash <<'DEPLOY'
set -e
sudo systemctl stop deuktemsiru
sudo install -o app -g app -m 0644 /home/ubuntu/app.jar /opt/deuktemsiru/app.jar
sudo systemctl restart deuktemsiru
for i in $(seq 1 12); do
  if curl -fsS http://127.0.0.1:8080/actuator/health; then exit 0; fi
  sleep 5
done
sudo journalctl -u deuktemsiru -n 100 --no-pager
exit 1
DEPLOY
```

DB 스키마 변경이 포함되면 적용 전에 RDS 백업·스냅샷을 확인한다. Flyway 적용 후에는 이전 JAR만 복원해도 DB와 호환된다고 가정하지 않는다.

이미지는 기본 설정에서 `/opt/deuktemsiru/uploads/menu-images`에 저장된다. 위 JAR 교체·서비스 재시작으로 지워지지는 않지만, 인스턴스·볼륨 교체 시 별도 백업·복원이 필요하다.

## 4. 운영 확인

```bash
sudo systemctl status deuktemsiru
sudo journalctl -u deuktemsiru -f              # 실시간 로그
sudo systemctl restart deuktemsiru
curl -fsS http://127.0.0.1:8080/actuator/health
```

운영 전 `prod` 적용·운영 JWT 비밀값·debug 로그인 차단, RDS 자동 백업 7일 이상, CloudWatch 알람(CPU 80%, RDS 연결 수, 5xx 비율)을 확인한다.

## 5. 장애 점검

| 증상 | 점검 |
| --- | --- |
| 서비스 `failed` | `journalctl -u deuktemsiru -n 200` — 환경 변수 누락 / JDK 버전 |
| DB `Connection refused` | `sg-db`가 `sg-app`을 허용하는지, 엔드포인트·포트 |
| 배포가 반영 안 됨 | `/opt/deuktemsiru/app.jar` 타임스탬프 |
| OOM | 인스턴스 상향 또는 `JAVA_TOOL_OPTIONS=-Xmx512m` |

## 앱 릴리스 설정

Android 설정 파일·Firebase 파일·서명 키의 위치와 우선순위는 [Android 환경 설정](https://github.com/deuktem-siru/android-app#환경-설정)을 따른다. 릴리스 전에는 다음을 확인한다.

- `BACKEND_BASE_URL`은 `/`로 끝나는 HTTPS 호스트 루트다.
- 각 앱의 Kakao 네이티브 키·패키지명·릴리스 키 해시를 등록한다.
- 푸시를 사용할 앱의 Firebase 설정과 서버 FCM 활성화·서비스 계정 설정을 함께 준비한다.
- 기존 앱을 업데이트할 때는 기존 서명 키를 사용하고 실제 키·설정 파일을 커밋하지 않는다.

`android-app` 루트에서 실행한다.

```bash
./gradlew :buyer:assembleRelease :seller:assembleRelease
```

APK는 각 모듈의 `build/outputs/apk/release/`에 생성된다. 배포 전 서명 여부를 확인하고 두 앱의 실기기 로그인 → 주문 → 승인 → 픽업 → 푸시를 검증한다. 앱별 사용 방법은 [구매자](https://github.com/deuktem-siru/android-app/blob/main/buyer/README.md#사용-방법)·[판매자](https://github.com/deuktem-siru/android-app/blob/main/seller/README.md#사용-방법) 안내를 따른다.

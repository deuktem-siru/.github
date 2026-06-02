# 득템시루 운영 배포

EC2(Ubuntu 22.04)에서 systemd로 `/opt/deuktemsiru/app.jar`를 실행하고 RDS PostgreSQL 16에 연결하는 절차다.
`application-prod.properties`가 DB 접속 정보를 환경 변수로 받으며, 시작 시 Flyway 적용 후 JPA 스키마를 검증한다.
로컬 실행은 [Backend README](https://github.com/DeuktemSiru/Backend#시작하기), S3 전환 계획은 [plan.md](../plan.md), 진행 상태는 [tasks.md](../tasks.md)를 따른다.

## 1. AWS 리소스

| 리소스 | 설정 |
| --- | --- |
| 보안 그룹 `sg-app` (EC2) | 22(내 IP), 8080(ALB 보안 그룹 또는 지정 대역) |
| 보안 그룹 `sg-db` (RDS) | 5432, 소스는 `sg-app`만 |
| RDS | PostgreSQL 16, 퍼블릭 액세스 No, 초기 DB `deuktemsiru` |
| EC2 | Ubuntu 22.04, `t3.small` 이상, 탄력적 IP 연결 |

앱 릴리스에는 HTTPS가 필요하다. Route53 도메인 + ACM 인증서 + ALB(443 → EC2 8080)를 구성하고, EC2의 8080 인바운드는 ALB 보안 그룹만 허용한다.

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

## 3. 자동 배포

`Backend`의 main 푸시(또는 수동 실행)로 `.github/workflows/deploy.yml`이 JDK 21에서 `./gradlew clean check bootJar`를 돌리고, 실행 JAR을 SCP로 올린 뒤 아래 [수동 배포](#4-수동-배포)와 같은 서비스 교체·헬스체크를 SSH로 실행한다. 실패하면 `journalctl` 로그를 워크플로 출력에 남긴다.

저장소 Secrets에 `EC2_HOST`, `EC2_USER`(`ubuntu`), `EC2_SSH_KEY`(EC2 접속 개인키 전문)를 등록한다. 워크플로는 `sudo systemctl`·`sudo install`을 쓰므로 `EC2_USER`가 암호 없이 sudo를 실행할 수 있어야 한다.

## 4. 수동 배포

자동 배포가 막혔거나 첫 배포일 때 사용한다.

로컬 `Backend/`에서 JDK 21·Docker를 준비하고 실행한다. `<...>`는 실제 경로·호스트로 바꾼다.

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

## 5. 운영 확인

```bash
sudo systemctl status deuktemsiru
sudo journalctl -u deuktemsiru -f              # 실시간 로그
sudo systemctl restart deuktemsiru
curl -fsS http://127.0.0.1:8080/actuator/health
```

운영 전 `prod` 적용·운영 JWT 비밀값·debug 로그인 차단, RDS 자동 백업 7일 이상, CloudWatch 알람(CPU 80%, RDS 연결 수, 5xx 비율)을 확인한다.

## 6. 장애 점검

| 증상 | 점검 |
| --- | --- |
| 서비스 `failed` | `journalctl -u deuktemsiru -n 200` — 환경 변수 누락 / JDK 버전 |
| DB `Connection refused` | `sg-db`가 `sg-app`을 허용하는지, 엔드포인트·포트 |
| 배포가 반영 안 됨 | `/opt/deuktemsiru/app.jar` 타임스탬프 |
| OOM | 인스턴스 상향 또는 `JAVA_TOOL_OPTIONS=-Xmx512m` |

## 앱 릴리스 설정

두 앱 모두 각 저장소 루트의 `release.keystore`와 `local.properties`의 `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`를 사용합니다. 이 파일들은 커밋하지 않습니다. API·외부 서비스 설정과 빌드 명령은 [BuyerApp README](https://github.com/DeuktemSiru/BuyerApp#시작하기)와 [SellerApp README](https://github.com/DeuktemSiru/SellerApp#시작하기)를 따릅니다.

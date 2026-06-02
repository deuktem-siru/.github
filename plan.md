# PLAN — 득템시루

## 구현 방향

- 독립 저장소·CI 3개: `Backend/`(Kotlin·JDK 21·Spring Boot 4·JPA·PostgreSQL 16), `BuyerApp/`(XML·Navigation Component), `SellerApp/`(XML·Activity). 두 앱 minSdk 29, Retrofit 2·OkHttp Authenticator·`EncryptedSharedPreferences`.
- API 계약은 [contracts.md](reference/contracts.md)·Swagger(`/swagger-ui/index.html`, `/v3/api-docs`), 모델은 [data-model.md](reference/data-model.md). API 변경은 백엔드·두 앱 `ApiService.kt`/`ApiModels.kt`에 반영하고, DB 변경은 엔티티·새 Flyway SQL에 반영한다. 독립 저장소의 관련 변경과 배포 순서를 함께 관리한다.
- Flyway V1~V5, Actuator·Prometheus·Grafana. 인앱 알림이 기본이며 FCM은 `APP_FCM_ENABLED=true`로 활성화한다. 실행은 각 README, 배포는 [deploy.md](reference/deploy.md).

## 단계별 계획

| 단계 | 방법 |
| --- | --- |
| P0 기능·안전장치 | 결제 경로 통합 검사와 prod 구성 회귀를 고정한다. |
| P1 이미지 | `MenuImageStorageService` 저장·삭제만 S3 SDK로 교체한다. 호출부·반환 URL 유지, dev는 로컬, 새 인터페이스 없음. 버킷은 퍼블릭 차단 + CloudFront(OAC), EC2 IAM Role에 `s3:PutObject/GetObject/DeleteObject` 권한 부여. 기존 파일은 `aws s3 sync`로 이관하고 DB의 `/uploads/...` URL을 새 배포 URL에 맞춘다. |
| P1 승인·정산 (완료) | 운영자가 `X-Admin-Token`으로 호출하는 `/api/v1/admin/**`이 사업자·매장 승인/반려와 정산 지급/반려를 처리한다. 미승인 매장은 상품 등록과 구매자 노출·구매가 모두 막힌다. 송금은 수기 절차다. |
| P1 배포·릴리스 | Backend `.github/workflows/deploy.yml`은 [자동 배포](reference/deploy.md#3-자동-배포)로 완료했다. 남은 일은 두 앱의 HTTPS URL·서명·Kakao·FCM 구성과 실기기 점검이다. |
| P2 개선 | P1 이후에만 착수. 폴링은 푸시의 폴백으로 유지하고 신고·차트는 실제 운영 문제/유지보수 부담이 생길 때 검토한다. |

## 검증 전략

- CI `./gradlew check`: 테스트·Jacoco·미완성 Stub 검사. 결제는 `DeuktemsiruApplicationTests`와 실제 커밋을 사용하는 `OrderPaymentIntegrationTest`로 확인한다(동시 주문 12건/재고 5개 포함).
- 결제 통합 검사는 Docker 없으면 스킵되므로 CI 결과로 판정한다. `ProdSafetyTest`는 컨텍스트·Docker 없이 JWT 비밀값, prod 시드 제외·설정을 검사한다.
- 운영 배포와 실기기는 [완료 기준](spec.md#완료-기준)으로 확인한다.

## 리스크 및 미결정

- 정산 상태 전이는 운영자 API로 처리하되 실제 송금은 수기다. 송금 주체·주기를 확정해야 한다.
- 운영자 API는 `APP_ADMIN_TOKEN` 단일 공유 토큰이다. 운영자가 늘면 계정별 권한 모델을 검토한다.
- 이미지는 서버 로컬 `uploads/menu-images`에 있으므로 이관이 필요하다.
- 운영 시크릿은 현재 환경 파일을 사용하며, SSM Parameter Store / Secrets Manager 이전을 검토한다.
- k6의 p95·에러율 기준은 미정이다.

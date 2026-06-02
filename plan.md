# 득템시루 구현 계획

## 아키텍처

백엔드는 헥사고날·클린 아키텍처, Android는 MVVM·클린 아키텍처를 기준으로 패키지 의존성을 분리한다. API URL·JSON 계약, DB 스키마, 앱 ID와 저장소 키를 유지한 현재 구현을 설명한다.

### 백엔드 경계

| 패키지 | 책임 |
| --- | --- |
| `domain` | 업무 모델·상태와 JPA 매핑 |
| `application` | 유스케이스 서비스·DTO와 `port/out`의 저장소·외부 연동 계약 |
| `adapter/inbound/web` | HTTP 입력 검증·변환과 서비스 호출 |
| `adapter/out/persistence` | Spring Data JPA 구현과 페이지 타입 변환 |
| `adapter/out`의 나머지 패키지 | 인증·푸시·파일 등 외부 연동 구현 |
| `configuration` | Spring 빈 설정·조립과 초기 데이터 |
| `common` | 오류 처리 등 공통 지원 코드 |

소스 의존성은 `web → application → domain`, `출력 어댑터 → application/port/out` 방향이다. 실행 시 서비스는 출력 포트를 호출하고 Spring이 연결한 어댑터가 DB·외부 시스템에 접근한다. 컨트롤러는 별도 입력 포트 인터페이스 없이 애플리케이션 서비스를 직접 호출한다. 계층은 단일 백엔드 Gradle 프로젝트 안의 패키지 경계이며 독립 배포 서비스가 아니다.

저장소 포트는 `EntityStore`와 업무별 계약을 사용한다. Spring Data 저장소가 포트를 구현하며, 페이지 변환이 필요한 경로는 별도 영속성 어댑터를 사용한다. 포트에는 Spring Data 타입을 노출하지 않는다. 도메인의 JPA 매핑과 서비스의 Spring DI·트랜잭션 참조는 유지하므로 모든 코어가 프레임워크와 독립된 구조는 아니다.

패키지 기준은 `com.deuktemsiru`다. 서비스는 `application/service`, DTO는 `application/dto`에 둔다. `IntegrationPorts.kt`의 `IdentityProvider`·`TokenIssuer`·`ImageStorage`·`PushSender`를 인증·이미지·푸시 어댑터가 구현한다. 페이지 요청은 `PageQuery`를 사용한다.

### Android 경계

| 패키지 | 책임 |
| --- | --- |
| `domain/model`, `domain/repository` | 모델과 suspend 저장소 계약 |
| `data` | Retrofit·OkHttp, 저장소 구현·업로드·세션 저장 |
| `presentation/view` | 화면 표시·입력·탐색 |
| `presentation/viewmodel` | 요청 조정과 화면 상태 |
| 루트 `AppGraph` | 구체 저장소 연결과 ViewModel factory |

소스 의존성은 `View → ViewModel → domain`, `data → domain`이며, `AppGraph`에서 구현을 연결한다. ViewModel은 View·Context·ViewBinding을 보관하지 않고 `ViewModelProvider`로 얻는다. 단순 요청은 공통 `AppViewModel`의 suspend 호출과 요청 상태를 사용한다. 이 호출은 호출자 코루틴에서 실행되며, 모든 요청이 `viewModelScope`로 이동한 것은 아니다. 전용 ViewModel의 StateFlow와 기존 화면의 suspend 결과 렌더링을 함께 사용한다. View의 세션·설정 접근과 루트 Application·메시징 진입점은 남아 있다.

구매자 `buyer`·판매자 `seller`는 각각 `AppGraph`를 가지며, `common`은 실제 Android 라이브러리 모듈이다. 앱의 `domain` 기능 패키지는 결제·가격·판매 등 규칙을 포함하고, 목록 어댑터·차트는 `presentation/widget`에 둔다. 구매자는 Home·Store·Cart·PaymentViewModel, 판매자는 SalesViewModel을 사용한다. 홈·매출은 StateFlow를 관찰하고 결제는 중복 제출 상태를 관리한다. `data/remote/WireJson.kt`와 common의 별칭 어댑터가 JSON 호환성을 처리하며 도메인 모델에 Gson 애너테이션을 두지 않는다.

### 경계 검증과 적용 범위

백엔드 `CleanArchitectureTest`는 domain/application의 어댑터 참조, domain의 application 참조, application의 JPA 직접 참조, 포트의 Spring 참조, 웹 어댑터의 영속성 구현 참조·트랜잭션 선언을 검사한다. 소스 검사이므로 전체 런타임 의존성이나 업무 동작을 보장하지 않는다.

두 앱의 `CleanArchitectureTest`는 domain·ViewModel의 data 참조, UI·통신 라이브러리 참조와 presentation의 `RetrofitClient.api` 직접 호출을 검사한다. 모든 View의 data 참조를 금지하는 검사는 아니다. ViewModel·JSON 변환 단위 테스트와 기존 업무 테스트를 함께 실행한다. Docker 통합 검사와 실기기 검증은 별도로 확인한다.

## 현재 구조

- 코드는 `backend/`와 `android-app/` 독립 저장소에서 관리하고 각각 CI를 실행한다. 문서는 `docs/` 저장소에 둔다.
- 백엔드는 Kotlin·JDK 21·Spring Boot 4·JPA·PostgreSQL 16을 사용한다.
- Android는 구매자 `buyer`(XML·Navigation Component), 판매자 `seller`(XML·Activity), 공통 `common` 모듈로 구성한다. 두 앱은 minSdk 29이며 HTTP 인증·카카오 로그인·암호화 저장소·푸시 처리를 공유한다. 앱별 세션·저장소 이름과 판매자 mock 동작은 유지한다.
- API 계약은 [contracts.md](reference/contracts.md)·Swagger(`/swagger-ui/index.html`, `/v3/api-docs`), 모델은 [data-model.md](reference/data-model.md). API 변경은 백엔드·두 앱의 `ApiService.kt`/`ApiModels.kt`와 `common`의 관련 코드에 반영하고, DB 변경은 엔티티·새 Flyway SQL에 반영한다. 독립 저장소의 관련 변경과 배포 순서를 함께 관리한다.
- Flyway V1~V5, Actuator·Prometheus·Grafana. 인앱 알림이 기본이며 FCM은 `APP_FCM_ENABLED=true`로 활성화한다. 실행은 각 코드 저장소 README, 배포는 [deploy.md](reference/deploy.md).

## 구현 단계

| 단계 | 방법 |
| --- | --- |
| P0 기능·안전장치 | 결제 경로 통합 검사와 prod 구성 회귀를 고정한다. |
| P1 이미지 | `ImageStorage.save`의 로컬 구현인 `MenuImageStorageService`를 S3 구현으로 교체할 계획이다. 삭제 기능은 현재 포트에 없으며 필요 시 별도 계약으로 추가한다. 호출부·반환 URL 유지, `ImageStorage` 포트는 유지하고 dev는 로컬 구현을 사용한다. 버킷은 퍼블릭 차단 + CloudFront(OAC), EC2 IAM Role에 `s3:PutObject/GetObject/DeleteObject` 권한 부여. 기존 파일은 `aws s3 sync`로 이관하고 DB의 `/uploads/...` URL을 새 배포 URL에 맞춘다. |
| P1 승인·정산 (완료) | 운영자가 `X-Admin-Token`으로 호출하는 `/api/v1/admin/**`이 사업자·매장 승인/반려와 정산 지급/반려를 처리한다. 미승인 매장은 상품 등록과 구매자 노출·구매가 모두 막힌다. 송금은 수기 절차다. |
| P1 배포·릴리스 | 백엔드 `.github/workflows/deploy.yml`은 [자동 배포](reference/deploy.md#3-자동-배포)로 완료했다. 남은 일은 두 앱의 HTTPS URL·서명·Kakao·FCM 구성과 실기기 점검이다. |
| P2 개선 | P1 이후에만 착수. 폴링은 푸시의 폴백으로 유지하고 신고·차트는 실제 운영 문제/유지보수 부담이 생길 때 검토한다. |

## 검증 전략

- CI `./gradlew check`: 테스트·Jacoco·미완성 Stub 검사. 결제는 `DeuktemsiruApplicationTests`와 실제 커밋을 사용하는 `OrderPaymentIntegrationTest`로 확인한다(동시 주문 12건/재고 5개 포함).
- 결제 통합 검사는 Docker 없으면 스킵되므로 CI 결과로 판정한다. `ProdSafetyTest`는 컨텍스트·Docker 없이 JWT 비밀값, prod 시드 제외·설정을 검사한다.
- 운영 배포와 실기기는 [완료 기준](spec.md#완료-기준)으로 확인한다.

## 남은 결정과 제약

- 정산 상태 전이는 운영자 API로 처리하되 실제 송금은 수기다. 송금 주체·주기를 확정해야 한다.
- 운영자 API는 `APP_ADMIN_TOKEN` 단일 공유 토큰이다. 운영자가 늘면 계정별 권한 모델을 검토한다.
- 이미지는 서버 로컬 `uploads/menu-images`에 있으므로 이관이 필요하다.
- 운영 시크릿은 현재 환경 파일을 사용하며, SSM Parameter Store / Secrets Manager 이전을 검토한다.
- k6의 p95·에러율 기준은 미정이다.
- 운영 DB 초기 스키마 생성 절차가 필요하다. V1~V5만으로 빈 DB를 구성할 수 없다. [DB 준비](reference/deploy.md#배포-전-db-준비)를 따른다.
- Android 원격 저장소는 연결되어 있다. 원격 CI·릴리스 성공 여부는 해당 실행 결과로 별도 확인한다.

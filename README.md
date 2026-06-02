# 득템시루 문서

마감 임박 식품을 지역 픽업으로 연결하는 득템시루의 요구사항·설계·운영 문서입니다.

## 시작하기

1. [요구사항](spec.md)에서 제품 범위와 완료 기준을 확인합니다.
2. [구현 계획](plan.md)에서 현재 구조와 남은 작업을 확인합니다.
3. 서버 실행은 [Backend README](https://github.com/deuktem-siru/backend#시작하기), 앱 설정·빌드는 [Android README](https://github.com/deuktem-siru/android-app)를 따릅니다.

## 관련 문서

| 문서 | 내용 |
| --- | --- |
| [요구사항](spec.md) | 제품 범위·동작 규칙·완료 기준 |
| [구현 계획](plan.md) | 현재 구조·구현 단계·검증 전략·미결정 사항 |
| [API 계약](reference/contracts.md) | 인증·요청·응답·상태 전이·현재 제약 |
| [데이터 모델](reference/data-model.md) | 관계·테이블·저장 의미·무결성 |
| [배포 가이드](reference/deploy.md) | 환경 설정·배포·릴리스·운영 점검 |
| [작업 지침](AGENTS.md) | 문서별 책임·변경 원칙 |
| [프로젝트 소개](profile/README.md) | 핵심 기능·아키텍처·앱 화면·팀 구성 |

## 관련 저장소

| 저장소 | 작업 공간 경로 | 내용 |
| --- | --- | --- |
| [Backend](https://github.com/deuktem-siru/backend) | `backend/` | 서버·DB·API 테스트·배포 |
| [Android](https://github.com/deuktem-siru/android-app) | `android-app/` | 구매자 `buyer`·판매자 `seller`·공통 `common` |
| [문서](https://github.com/deuktem-siru/.github) | `docs/` | 이 문서와 GitHub 조직 소개 |

세 폴더는 같은 작업 공간에 있는 독립 Git 저장소입니다.

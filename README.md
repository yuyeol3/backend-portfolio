# 최유렬 | Backend Developer Portfolio

Java와 Spring으로 서비스 API와 데이터 수집기를 개발했습니다. 토큰 갱신 요청이 겹쳐 발생한 오류를 수정했고, 배포 환경 차이로 생긴 빌드 실패와 파일 저장 성능 저하를 재현하며 원인을 찾아 개선했습니다.

[GitHub](https://github.com/yuyeol3) / [개발 블로그](https://yuyeol3.github.io/)

## 핵심 경험

| 경험 | 직접 수행한 일 | 확인한 결과 |
| --- | --- | --- |
| Y-Fin | Refresh Token 동시성 수정, 금융 데이터 수집과 정규화, LLM 호출 실험 | 토큰 갱신 10개 요청에서 성공 1건, 401 응답 9건. 동일한 금융상품 97건의 배치 시간 38~48% 단축 |
| 카카오테크캠퍼스 | PR 검증과 ECR, SSM 배포 경로 구축, Docker 빌드 실패 대응 | 배포에서만 발견되던 오류를 PR 단계에서 재현하고, 수정 후 3개 구성요소 빌드 통과 |
| mc-distributed | 스냅샷 기록과 재생으로 병목 분석, 변경 파일만 stage하도록 개선 | 조정용 재생의 커밋 시간 0.87→9.7초 증가를 0.49~1.2초 범위로 완화. 별도 검증용 재생 0.37~0.91초 |

## 지원 직무와 연결되는 역량

| 직무에서 요구하는 역량 | 프로젝트 근거 |
| --- | --- |
| Java, Spring Boot, REST API, 인증과 보안 | Y-Fin의 OAuth2, JWT, Spring Security 인증, 사용자와 상품 API |
| 관계형 DB와 데이터 처리 | PostgreSQL과 JPA를 이용한 토큰 상태 변경, Spring Batch 외부 API 수집과 정규화 |
| AWS, 컨테이너, CI/CD | 카테캠의 EC2, Docker Compose, GitHub Actions, ECR, SSM 배포 |
| 안정성과 성능 개선 | 토큰 중복 갱신 방지, 배포 환경 차이 재현, 파일 커밋 비용 분석 |
| Git 협업과 배포 관리, 테스트 | 팀 PR에 테스트와 이미지 빌드 적용, 커밋 SHA 기반 이미지 추적, 수정 전후 실행 기록 |

## 목차

1. [Y-Fin](#1-y-fin): 토큰 갱신 동시성 / 외부 금융 데이터 처리
2. [카카오테크캠퍼스 팀 프로젝트](#2-카카오테크캠퍼스-팀-프로젝트): 배포 실패 재현과 CI 보완
3. [mc-distributed](#3-mc-distributed): 스냅샷 커밋 병목 분석
4. [추가 기술 경험](#추가-기술-경험)

---

## 1. Y-Fin

**사용자의 소득, 나이, 가구 조건에 맞는 금융상품을 찾고 우대금리 조건을 확인하는 서비스**

| 항목 | 내용 |
| --- | --- |
| 기간 / 인원 | 2026.03 ~ 진행 중 / 5인 팀, 백엔드 2인 |
| 본인 역할 | 인증과 인가 기반 및 데이터 수집기 구축, 상품과 공급자 API 개발, 추천 로직 유지보수와 리팩터링 |
| 주요 기술 | Java 21, Spring Boot, Spring Security, JPA, Spring Batch, PostgreSQL, Gemini |
| 데이터 범위 | 은행상품 328건 + 정부 정책상품 63건, 총 391건 정규화 |
| 수상 | 제7회 PNU 창의융합AI해커톤 창업트랙 최우수상 |
| 코드 | [서비스 백엔드](https://github.com/ApptiveDev/Fin-BE) / [수집기](https://github.com/ApptiveDev/Fin-API) / [해커톤 버전](https://github.com/PNU-2026-AI-Hackathon/pnuai-c-07-finfin2) |

### 토큰 갱신 요청이 겹칠 때 발생한 500 오류

같은 Refresh Token으로 갱신 요청이 겹치면 두 요청이 모두 기존 토큰을 조회했습니다. 이후 토큰을 삭제하고 재발급하는 과정에서 충돌이 발생해 500 오류가 반환됐습니다.

```mermaid
flowchart LR
    A["같은 토큰의<br/>갱신 요청"] --> B["토큰 조회 및<br/>사용자 확인"]
    B --> C["활성 토큰만<br/>조건부 비활성화"]
    C --> D{"변경 행 수 = 1"}
    D -->|예| E["새 토큰 발급<br/>200"]
    D -->|아니오| F["재사용 요청 거부<br/>401"]
```

이를 해결하기 위해 `isActive = true`인 토큰만 비활성화하는 UPDATE를 실행하고, 변경한 행 수가 1개인 요청에만 새 토큰을 발급했습니다. 토큰 조회 이후 다른 요청이 먼저 처리했더라도, DB 상태를 실제로 바꾼 요청만 재발급을 진행하도록 한 것입니다.

수정 후 10개 요청을 실행해 **200 응답 1건과 401 응답 9건**을 확인했습니다. 이미 사용한 토큰으로 들어온 요청은 500 오류 대신 인증 실패로 처리됐습니다. 해당 실행 결과와 코드를 PR에 남겼습니다.

[동시성 수정과 요청 결과](https://github.com/ApptiveDev/Fin-BE/pull/9) / [인증 기반 구현](https://github.com/ApptiveDev/Fin-BE/pull/3) / [인가 예외 처리](https://github.com/ApptiveDev/Fin-BE/pull/11)

### 금융 데이터 정규화에서 발생한 timeout과 파싱 실패

외부 API마다 데이터 구조가 달랐고, 우대금리의 일부 조건은 줄글로 제공됐습니다. 수집원별 데이터를 공통 상품과 옵션 모델로 정규화하고, 줄글 조건은 LLM으로 구조화했습니다. 수집 대상에서 사라진 상품은 비활성화하도록 동기화도 구현했습니다.

이 과정에서 일부 LLM 호출이 약 60초 뒤 timeout으로 끝나거나 응답 파싱에 실패했습니다. 배치 전체 시간만으로는 어떤 상품의 호출이 지연되는지 알기 어려워 상품별 호출 시간과 성공 여부를 기록했습니다.

```mermaid
flowchart LR
    A["금감원 API<br/>온통청년 API"] --> B["Batch 수집<br/>수집원별 정규화"]
    B --> C["LLM 보강<br/>응답 파싱<br/>실패 처리"]
    C --> D["공통 상품, 옵션<br/>PostgreSQL"]
```

동일한 FSS 원본 97건으로 temperature와 출력 토큰 상한 조합을 비교한 뒤 `temperature=0.3`, `max_output_tokens=2048`을 적용했습니다. 이 설정에서 배치 시간은 5분 30초에서 2분 52초~3분 23초로 줄었지만 파싱 실패는 남아 있었습니다.

남은 실패를 줄이기 위해 포괄 조건과 실제 우대 조건을 구분하고, 구간별 차등 금리를 별도 항목으로 추출하도록 프롬프트를 보완했습니다. 응답을 감싼 코드펜스를 제거하고 누락 필드는 기본값으로 처리했습니다. 실패한 상품을 계속 호출하지 않도록 6시간 재시도 쿨다운도 두었습니다.

| 검증 단계 | 변경 전 | 변경 후 | 해석 범위 |
| --- | --- | --- | --- |
| 설정 조합 비교 | 97건 배치 5분 30초, 실패 4건 | 선택 설정의 3회 실행 2분 52초~3분 23초, 실패 4~5건 | 전체 시간 약 38~48% 단축. 이 단계에서 파싱 실패는 남아 있었음 |
| 프롬프트 보완 | 우대조건 파싱 실패 | 7차례 개선 후 동일한 97건에서 파싱 실패 0건 | 조정에 사용한 데이터의 회귀 검증 |

temperature와 출력 상한을 함께 바꾼 실험이라 각 설정의 효과를 따로 구분할 수는 없습니다. 프롬프트 수정 후에는 같은 97건에서 파싱 실패가 0건이었으며, 새 상품에 대한 검증은 별도로 필요합니다. 수집기에는 실제 PostgreSQL을 사용하는 Testcontainers 통합 테스트를 구성했습니다.

[설정별 실행 결과와 프롬프트 수정](https://github.com/PNU-2026-AI-Hackathon/pnuai-c-07-finfin2/pull/40) / [97건 검토 자료](https://github.com/user-attachments/files/32417255/fss_normalization_v7_review.xlsx) / [상품 동기화 구현](https://github.com/ApptiveDev/Fin-API/pull/1)

---

## 2. 카카오테크캠퍼스 팀 프로젝트

**개인사업자의 카드 승인내역을 받아 필요경비 인정 여부와 근거를 제공하는 서비스**

| 항목 | 내용 |
| --- | --- |
| 기간 / 인원 | 2026.09 ~ 진행 중 / 6인 팀, 백엔드 2인 |
| 본인 역할 | 백엔드 개발, CI/CD 및 AWS 배포 기반 구축 |
| 주요 기술 | Java 21, Spring Boot, PostgreSQL, GitHub Actions, Docker Compose, AWS EC2, ECR, SSM, IAM, Caddy |
| 현재 단계 | 팀 서비스 개발 중. EC2 배포 경로 및 스택 기동을 확인했으며 서비스 전체 기능 완성은 진행 중 |
| 코드 | [팀 저장소](https://github.com/kakaotechcampus-4/ktc4-pusan-4) |

### CI는 통과했지만 배포 이미지 빌드가 실패한 원인

2026년 10월 5일, PR의 테스트와 웹 빌드는 통과했지만 release 배포에서 웹 Docker 이미지 빌드가 실패했습니다. 프런트 빌드가 읽는 `rules/categories.yaml`이 이미지에 없었습니다. CI 러너에는 저장소 전체가 있었지만 Dockerfile은 `frontend/`만 복사하고 있어 빌드 조건이 달랐습니다. 이미지 빌드 단계에서 멈췄기 때문에 EC2의 기존 버전은 유지됐습니다.

```mermaid
flowchart LR
    A["PR<br/>테스트와 빌드<br/>운영 이미지 빌드"] --> B["release<br/>OIDC 인증<br/>SHA 태그 / ECR"]
    B --> C["SSM 배포<br/>EC2 기동 확인"]
    C --> D["공개 HTTPS<br/>응답 확인"]
```

누락 파일을 추가하기 전에, 백엔드와 AI, 웹을 **배포와 같은 Dockerfile 및 빌드 컨텍스트로 PR에서 빌드**하도록 CI를 바꿨습니다. 이 변경만 반영했을 때 기존 `npm run build`는 통과했지만 새 `Build production image` 단계는 배포와 같은 오류로 실패했습니다. 추가한 검증이 실제로 파일 누락을 잡는다는 것을 확인한 뒤 Dockerfile에 카테고리 원본을 복사하도록 수정했습니다. 이후 세 이미지 빌드가 모두 통과했고 같은 날 release 배포도 성공했습니다.

이미지 빌드를 추가하면서 CI 시간은 늘었습니다. 한 차례의 전후 비교에서 가장 오래 걸린 백엔드 job은 1분 18초에서 1분 56초로 증가했습니다. 이 시간을 들여 배포에서 발생할 빌드 오류를 PR 단계에서 확인하도록 했습니다.

배포 경로도 직접 구축했습니다. GitHub OIDC로 AWS 임시 자격증명을 받고 SSM으로 EC2의 배포 스크립트를 실행했습니다. 이미지에는 커밋 SHA를 붙여 버전을 추적하고, `flock`으로 배포를 한 번에 하나씩 실행하도록 했습니다. 최신 release SHA와 다른 오래된 배포 요청은 거부했습니다.

Compose 기동 대기와 공개 HTTPS 응답 확인을 배포 과정에 넣었고, 이전 SHA를 지정하는 롤백 명령도 마련했습니다. 다만 이미지 롤백으로 DB 스키마까지 복원되지는 않아 스키마 복원은 별도로 처리해야 합니다.

[빌드 오류 수정 PR](https://github.com/kakaotechcampus-4/ktc4-pusan-4/pull/89) / [오류를 재현한 CI 실행](https://github.com/kakaotechcampus-4/ktc4-pusan-4/actions/runs/37260200228) / [수정 후 성공한 CI 실행](https://github.com/kakaotechcampus-4/ktc4-pusan-4/actions/runs/37260380195)

[배포 워크플로](https://github.com/kakaotechcampus-4/ktc4-pusan-4/blob/develop/.github/workflows/aws-release.yml) / [배포 스크립트](https://github.com/kakaotechcampus-4/ktc4-pusan-4/blob/develop/deploy/deploy.sh)

---

## 3. mc-distributed

**상시 게임 서버 없이 참가자의 PC가 호스트를 맡고, Git 저장소로 하나의 Minecraft 월드를 공유하는 프로그램**

| 항목 | 내용 |
| --- | --- |
| 담당한 작업 | 스냅샷 커밋 비용의 원인 분석, 변경 파일만 stage하는 개선 및 재생 실험 |
| 주요 기술 | Java, JGit, Paper, Git 기반 월드 버전 관리 |
| 검증 환경 | 로컬 Paper와 테스트 봇으로 세션 기록, Paper 없이 스냅샷 재생. 실제 원격 친구 환경은 미검증 |
| 코드 / 구조 | [저장소](https://github.com/yuyeol3/mc-distributed) / [아키텍처](https://github.com/yuyeol3/mc-distributed/blob/main/docs/ARCHITECTURE.md) |

### 월드가 커질수록 스냅샷 커밋이 느려지는 현상

월드 스냅샷을 커밋하는 시간이 회차마다 늘어났습니다. 조정용 재생에서 실제로 변경된 양은 비슷했지만 커밋 시간이 0.87초에서 9.7초로 증가했습니다. 커밋은 월드의 디스크 저장을 멈춘 구간에서 수행하므로 이 시간을 줄일 필요가 있었습니다.

```mermaid
flowchart TB
    A["기존<br/>전체 파일 add"] --> B["변경 없는 파일도<br/>읽기와 압축"]
    B --> C["월드가 커질수록<br/>커밋 비용 증가"]
    D["개선<br/>status로<br/>변경 파일 확인"] --> E["변경/신규 add<br/>삭제 파일<br/>rm --cached"]
    E --> F["커밋에 필요한<br/>파일만 처리"]
```

반복해서 비교할 수 있도록 실제 Paper와 봇으로 기록한 스냅샷 20개를 새 Git 저장소에 재생했습니다. 원인 분석에 쓰는 조정용 세션과 마지막 확인에 쓰는 검증용 세션은 따로 기록했습니다.

처음에는 압축 비용을 의심했습니다. 압축 수준을 낮춰도 커밋 시간은 계속 증가했고, 압축을 끄면 시간은 줄지만 `.git/objects` 크기가 340MB에서 472MB로 늘었습니다. 이어서 파일을 바꾸지 않고 커밋해 보니 이 경우에도 시간이 0.66초에서 8.5초로 증가했습니다. 실제 변경량보다 전체 파일을 stage하는 과정에 비용이 든다는 단서였습니다.

JGit의 `add "."`이 변경되지 않은 파일도 다시 읽고 압축하는 경로를 확인했습니다. `status`로 변경되거나 새로 생긴 파일을 찾고 해당 파일만 `add`하도록 바꿨습니다. 삭제한 파일은 `rm --cached`로 인덱스에 반영했습니다. 월드 저장 형식을 바꾸지 않고 처리 대상을 줄인 뒤 같은 조정용 세션을 재생했습니다.

| 측정 | 개선 전 | 개선 후 |
| --- | --- | --- |
| 조정용 스냅샷 재생 | 1→20회차에서 0.87→9.7초 | 0.49~1.2초 |
| 변경 없는 상태의 커밋 | 0.66→8.5초 | 29ms |
| 조정에 쓰지 않은 검증용 재생 | 동일 재생 조건의 개선 전 수치는 기록 없음 | 0.37~0.91초, 회차에 따른 증가 추세 미관찰 |

조정에 쓰지 않은 검증용 세션도 재생해 커밋 시간이 0.37~0.91초 범위에 머무는 것을 확인했습니다. 검증용 세션의 개선 전 기록은 실제 플레이 중 측정이라 재생 결과와 직접 비교한 단축률은 계산하지 않았습니다. 측정 대상은 로컬 커밋 시간으로, 게임 FPS나 네트워크 전송 성능은 이 실험에서 확인하지 않았습니다.

[압축 설정 비교와 스냅샷 재생 실험 기록](https://github.com/yuyeol3/mc-distributed/blob/main/experiments/E1-prime-results.md)

---

## 추가 기술 경험

Yacht Online은 3인 팀으로 개발한 실시간 게임입니다. 백엔드와 게임 서버 개발, Redis 연동, EC2 배포를 담당했습니다. 하나의 Spring Boot 코드베이스를 API 서버와 Game 서버 역할로 나누고, Redis Sorted Set의 방 개수와 heartbeat을 이용해 응답 가능한 서버를 선택했습니다. Game 서버는 10초마다 heartbeat을 갱신하고 30초 TTL로 갱신이 끊긴 서버를 후보에서 제외했습니다. Docker Compose 통합 실행 후 Game 서버 2대를 AWS에 배포했습니다.

[서버 레지스트리 코드](https://github.com/yuyeol3/yacht-backend/blob/865dd4056a69ab2da8875b1a9194779c79333096/src/main/java/io/github/yuyeol3/yachtbackend/server/GameServerRegistryService.java) / [프로젝트 구성도](https://github.com/yuyeol3/yacht-online/blob/main/docs/images/diagram.png)


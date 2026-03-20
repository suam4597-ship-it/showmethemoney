
# 투자용 해외 뉴스/공시 수집 시스템 설계서 v2
_검증 기준일: 2026-03-18 (Asia/Seoul)_  
_목표: Codex Agent가 그대로 구현 계획으로 사용할 수 있는 수준의 백엔드 중심 설계서_

---

## 0. 이 문서의 핵심 결론

이 프로젝트는 **“원격 백엔드 + 네이티브 데스크톱 앱”** 구조로 가야 한다.  
이유는 단순하다.

1. **공식 공시/거래소/외신/와이어/지표 소스가 지역별로 매우 분산되어 있고**, 폴링·재시도·체크포인트·중복 제거·상태머신·이력 관리가 필요하다.
2. “중복 없는 이벤트 전달”은 단순 기사 스크래핑이 아니라 **정규화된 이벤트 데이터 플랫폼**을 만드는 일이다.
3. **localhost를 띄우지 않는 네이티브 앱**이라는 조건 때문에, UI는 로컬 렌더러만 담당하고 핵심 수집/병합/판정/스케줄은 서버에서 책임지는 구조가 가장 안전하다.
4. 기술부채를 최소화하려면 처음부터 **공식 공시 우선 / 기사 보조 / 정본 이벤트 저장 / 증거 연결** 구조를 잡아야 한다.

이 문서의 추천 아키텍처는 다음과 같다.

- **Backend control plane:** Elixir + Phoenix + Oban + PostgreSQL
- **고성능/파싱/병합 엔진:** Rust
- **Native GUI:** Rust + Slint (1안), Tauri 2 (2안)
- **Object storage:** S3-compatible (예: MinIO/S3)
- **관측성:** OpenTelemetry + centralized logs + metrics
- **배포:** 단일 VM 또는 소형 k8s 이전의 container-based 배포부터 시작

---

## 1. 목표와 비목표

### 1.1 목표

이 시스템은 아래를 **매일 지역별로 거의 빠짐없이** 수집하고, **중복 없이** 사용자에게 전달해야 한다.

- 해외 지역별 주요 외신 핵심 뉴스
- 산업별 **수요/공급 변화**
- **혁신 기술 변화**와 산업적 파급
- 특정 제품/상품의 **부족**, 공급 차질, 수요 급증
- **M&A / 자금조달 / 대규모 자본 이동**
- **기업 실적 발표** 및 가이던스 변화
- 규제 심사/반독점/승인/결렬 등 **거래 상태 변화**
- 지역별 저녁~밤, 즉 **그날 뉴스 업로드 빈도가 크게 줄어드는 시점**에 묶음 전달

### 1.2 비목표

처음부터 아래까지 하지는 않는다.

- 초단타 매매용 초저지연 시스템
- 실시간 tick/quote 인프라
- 사용자 브라우저에서 각 사이트를 직접 스크래핑하는 구조
- 기사 전문 재배포를 전제로 한 설계
- LLM이 원문을 완전히 대체하는 구조

---

## 2. 설계 원칙

### 2.1 공식 공시가 정본이다
우선순위는 다음과 같다.

1. **공식 공시/거래소/규제기관**
2. 발행사 **IR 원문**
3. 라이선스가 명확한 **와이어 서비스** (Business Wire, GlobeNewswire, PR Newswire 등)
4. 신뢰 가능한 **외신/전문지**
5. 업계 지표/대체 데이터
6. 소셜/루머

### 2.2 저장 단위는 “기사”가 아니라 “이벤트”
동일 사건이 여러 소스로 들어와도 사용자에게는 하나의 이벤트로 보여야 한다.

예:
- 8-K 실적
- 회사 IR 보도자료
- 실적발표 슬라이드
- Reuters 기사
- AP 기사
- FT 해설 기사

위 5개가 들어와도 UI에는 **“실적 이벤트 1개 + 증거 5개”** 로 보이는 구조여야 한다.

### 2.3 중복 제거는 사후 필터가 아니라 핵심 도메인이다
중복 제거를 나중에 붙이면 실패한다.  
처음부터 데이터 모델이 **canonical_event / event_evidence** 구조여야 한다.

### 2.4 백엔드가 먼저다
이 프로젝트는 UI보다 먼저 아래가 완성되어야 한다.

- 소스 레지스트리
- 폴링/파싱/정규화
- 엔티티 매핑
- 이벤트 추출
- 이벤트 병합
- 이력과 수정 추적
- 발송 패킷 생성
- 재처리/백필
- 관측성/장애복구

### 2.5 “거의 빠짐없이”를 위해 지역별 합집합을 사용한다
특히 유럽은 단일 완전체 소스가 없다.  
따라서 국가/거래소/저장소/발행사 IR의 **합집합**이 필요하다.

### 2.6 기술부채 최소화
첫 버전부터 아래를 강제한다.

- 강한 타입
- 소스별 rate-limit / retries / idempotency
- event sourcing에 가까운 증거 저장
- 데이터 정합성 제약조건
- 운영 메트릭
- 백업/복구 전략
- 변경 가능성이 큰 모듈의 경계 명확화

---

## 3. 왜 “원격 백엔드 + 네이티브 앱”이 정답인가

### 3.1 로컬 수집형 앱의 문제
로컬 앱이 직접 모든 소스를 수집하려 하면 다음 문제가 생긴다.

- 사용자가 앱을 끄면 수집이 멈춤
- 각 지역별 시간대 스케줄이 불안정
- rate-limit / IP 블록 / 캡차 대응이 어려움
- 보안정보(쿠키, 토큰, 라이선스 키)를 클라이언트에 배포하게 됨
- 중복 제거 품질을 지속 개선하기 어려움
- 장애 복구/백필/재처리가 곤란

### 3.2 원격 백엔드 장점
- 항상 켜져 있는 수집 스케줄
- 소스별 전용 파이프라인 운영 가능
- 재시도/백필/상태머신/감사로그 구현 가능
- 라이선스 및 비공개 자격증명 보호
- 복수 사용자/팀으로 확장 가능
- UI는 네이티브 렌더링만 담당하므로 localhost 불필요

---

## 4. 권장 기술 스택

## 4.1 추천 1안 (가장 균형 좋음)

### Backend
- **Elixir**
- **Phoenix** (JSON API + WebSocket)
- **Oban** (job orchestration)
- **PostgreSQL**
- **S3-compatible object storage**
- **OpenTelemetry**
- **Rust sidecar services** (parser / deduper / entity matcher)

### Frontend
- **Rust + Slint**

### 왜 이 구성이 좋은가
- Elixir/OTP는 스케줄러/동시성/재시도/감시 트리에 강하다.
- Phoenix는 API/실시간 전달에 적합하다.
- Oban은 durable job queue로 매우 실용적이다.
- PostgreSQL 하나로도 초기 80% 요구를 충분히 커버한다.
- Rust는 CPU-bound 파서/병합/유사도 계산에 적합하다.
- Slint는 localhost 없이도 네이티브 GUI를 만들기 쉽다.

---

## 4.2 추천 2안

### Backend
- Elixir + Phoenix + Oban + PostgreSQL + Rust

### Frontend
- **Tauri 2 + Rust + TypeScript**

### 장점
- 풍부한 프론트엔드 생태계
- UI 컴포넌트 선택 폭이 넓음
- 데스크톱 앱 패키징이 성숙

### 단점
- 웹 UI 레이어가 추가되어 복잡도 증가
- WebView 의존
- 순수 네이티브 감각/메모리/디버깅 단순성은 Slint보다 떨어질 수 있음

---

## 4.3 추천하지 않는 1차 구조

### 올 Rust 풀스택
장점:
- 단일 언어
- 성능 우수

단점:
- 스케줄링/내결함성/운영 툴링의 초반 생산성이 낮아질 가능성
- 공시/뉴스 수집은 IO 중심이며, BEAM의 장점을 포기하게 됨

### Electron
장점:
- 빠른 UI 개발

단점:
- 네이티브 앱 조건에서 과한 리소스
- localhost는 아니어도 웹앱 사고방식이 강해짐
- 장기 기술부채 가능성 큼

---

## 5. Rust와 Elixir의 경계 설계

이 프로젝트의 가장 중요한 기술적 선택 중 하나는 Rust/Elixir 경계다.

## 5.1 권장 경계

### Elixir가 맡을 것
- 스케줄링
- 큐/재시도/백오프
- 소스별 ingest orchestration
- API
- 사용자 설정
- delivery packet 조립
- 이벤트 라이프사이클 관리
- 운영/관측성
- 데이터베이스 트랜잭션 조율

### Rust가 맡을 것
- 고성능 HTML/XML/XBRL/inline XBRL 파싱
- 문서 정규화
- 숫자/단위/통화 파싱
- 엔티티 후보 생성
- 유사도 계산
- 이벤트 병합 보조
- 번역 전/후 정규화
- 임베딩 생성 전처리
- 규칙 기반 extractor

## 5.2 Rustler vs Ports vs Sidecar
초기에는 다음 순서를 추천한다.

1. **Sidecar process 또는 Port 기반 통신**
2. 성숙 후에만 일부 순수 함수형 로직을 Rustler NIF로 이동

이유:
- NIF는 잘못 쓰면 BEAM 스케줄러에 영향을 줄 수 있다.
- 파서/병합은 실패 확률이 있고 입력이 더럽다.
- 외부 프로세스로 분리하면 crash isolation이 훨씬 좋다.

### 실무 권장안
- **Rust parser service**
  - stdin/stdout JSON Lines 기반 포트 또는 gRPC
- **Rust merge engine**
  - batch/stream input 받아 candidate set 계산
- **Elixir orchestrator**
  - 요청/응답 timeout, circuit breaker, retry 담당

---

## 6. 전체 시스템 아키텍처

```text
┌───────────────────────────── Native Desktop App (Rust + Slint) ─────────────────────────────┐
│ Feed / Event Detail / Company View / Calendar / Settings / Search / Local Cache            │
└───────────────────────────────────────▲───────────────────────────────────────────────────────┘
                                        │ HTTPS / WebSocket
┌───────────────────────────────────────┴───────────────────────────────────────────────────────┐
│                                   Phoenix API Layer                                           │
│ Auth / Delivery API / Search API / Event API / Admin API / WebSocket push                     │
└──────────────────────────────▲───────────────────────▲────────────────────────────────────────┘
                               │                       │
                               │                       │
┌──────────────────────────────┴──────────────┐ ┌──────┴───────────────────────────────────────┐
│        Elixir Domain Services               │ │              Oban Job System                  │
│ source registry / event lifecycle /         │ │ fetch / parse / normalize / enrich /         │
│ delivery windows / policy / audit /         │ │ dedupe / backfill / healthcheck / retry      │
│ rights control                              │ │                                              │
└──────────────────────▲──────────────────────┘ └──────▲───────────────────────────────────────┘
                       │                               │
                       │                               │
┌──────────────────────┴────────────────────────────────────────────────────────────────────────┐
│                               Rust Sidecar Services                                           │
│ parser / xbrl extractor / entity matcher / similarity / event candidate builder              │
└──────────────────────▲────────────────────────────────────────────────────────────────────────┘
                       │
                       │
┌──────────────────────┴──────────────────────────────┐  ┌──────────────────────────────────────┐
│ PostgreSQL                                          │  │ Object Storage                       │
│ sources / raw_docs / normalized_docs / entities /   │  │ raw html/pdf/xbrl/json attachments  │
│ canonical_events / evidence / delivery_packets /    │  │ snapshots / parser artifacts        │
│ checkpoints / audit / metrics                       │  │                                      │
└─────────────────────────────────────────────────────┘  └──────────────────────────────────────┘
```

---

## 7. 소스 계층 설계

## 7.1 소스 레벨

각 소스는 아래 레벨 중 하나로 등록한다.

- **L0_REGULATORY_PRIMARY**  
  거래소/규제기관/공식 공시 저장소  
  예: SEC EDGAR, TDnet, MOPS, CNINFO, RNS

- **L1_ISSUER_PRIMARY**  
  발행사 공식 IR / press release / results center

- **L2_LICENSED_WIRE**  
  Business Wire / GlobeNewswire / PR Newswire / licensed Reuters/AP feed

- **L3_EDITORIAL_TRUSTED**  
  Reuters, FT, WSJ, AP, Nikkei Asia, Caixin Global 등

- **L4_DOMAIN_INDICATOR**  
  IEA, OPEC, EIA, USDA, WSTS, 거래소 통계, SCFI

- **L5_ALTERNATIVE_SIGNAL**  
  산업 데이터, 대체 신호, 필요 시 제한적으로 사용

## 7.2 소스 등록 필드
`source_registry`

- `source_id`
- `source_name`
- `region`
- `country`
- `language`
- `source_level`
- `authority_rank`
- `delivery_method` (api/rss/html/pdf/email/sftp)
- `auth_mode`
- `poll_strategy`
- `rate_limit_policy`
- `legal_mode` (link_only / excerpt_only / licensed_fulltext / restricted_internal)
- `supports_incremental`
- `timezone`
- `market_calendar_code`
- `parser_profile`
- `dedupe_priority`
- `status`
- `owner_team`
- `notes`

---

## 8. 지역별 소스 설계

# 8.1 미국 (최우선)

## 8.1.1 정본 계층
- **SEC EDGAR APIs / submissions / companyfacts / latest filings**
- **SEC full-text search**
- **회사 IR 사이트**
- **8-K / 10-Q / 10-K / 20-F / 40-F / 6-K**
- **Schedule TO**
- **Schedule 13D / 13G**
- **S-3 / 424B / debt offering 관련 filing**

## 8.1.2 왜 중요한가
미국은 다음 이벤트를 가장 촘촘하게 잡을 수 있다.

- 실적 발표
- 가이던스 상향/하향
- M&A 계약 체결
- 자산 인수/처분
- 대규모 자금조달
- buyback / special dividend
- activism
- convertible / debt issuance

## 8.1.3 반드시 잡아야 하는 서류 타입
- `8-K Item 2.02` : 실적/재무정보 공표
- `8-K Item 1.01` : material definitive agreement
- `8-K Item 2.01` : completion of acquisition/disposition of assets
- `8-K Item 2.03` : creation of direct financial obligation
- `10-Q / 10-K`
- `20-F / 40-F / 6-K`
- `SC TO-*`
- `13D / 13D/A`
- `S-3`, `424B*`

## 8.1.4 외신/보조 계층
- Reuters Markets / Business / Deals
- AP Business
- WSJ Markets/Deals
- FT Companies/Markets
- Business Wire
- GlobeNewswire
- PR Newswire

## 8.1.5 미국 실적 커버리지 전략
미국은 “earnings calendar” 자체보다 **공식 filing + issuer IR + wire** 조합이 더 신뢰할 만하다.  
캘린더는 예고 용도이고, 정본은 **실제 공시 시점**이다.

## 8.1.6 미국 백엔드 운영 주의점
- EDGAR는 User-Agent 명시
- 공정 접근 정책 준수
- 초당 요청 제한 준수
- “latest filings”와 nightly index를 혼동하지 말 것
- filing accepted time과 public availability time 차이를 기록할 것

---

# 8.2 영국

## 8.2.1 정본 계층
- **RNS (Regulatory News Service)**
- 발행사 IR
- FCA NSM(필요 시 추가 검토)

## 8.2.2 특징
영국은 RNS가 핵심이다.  
실적, M&A, director dealings, trading update, contract win/loss, capital raise가 강하다.

## 8.2.3 주의점
- RNS는 강력하지만 유럽 전체를 커버하지 않는다.
- 영국 상장사 중심으로 이해해야 한다.
- 해외 이중상장사는 현지 거래소 공시와 cross-check 필요

---

# 8.3 유럽 대륙 (프랑스/네덜란드/벨기에/포르투갈 등 Euronext 계열 포함)

## 8.3.1 정본 계층
- **Euronext regulated information**
- **Issuer financial calendar**
- 발행사 IR

## 8.3.2 특징
- 실적, 분기/반기/연간 보고, M&A 관련 investor news, capital markets news를 제공
- 다만 국가별 발행사 관행 차이가 큼
- 공시 형식과 언어가 통일되지 않음

## 8.3.3 주의점
- “한 사이트로 전부”라는 가정 금지
- issuer calendar/IR를 반드시 합쳐야 함
- after-market-close release 관행이 많아 저녁 버퍼가 필요

---

# 8.4 독일/오스트리아/스위스 축

## 8.4.1 독일
정본:
- Deutsche Börse / Xetra 관련 공시 확인
- 발행사 IR
- **EQS News** (실무적으로 매우 중요)

필수 수집:
- ad hoc announcements
- financial reports
- voting rights
- managers’ transactions
- capital measures

## 8.4.2 스위스
정본:
- **SIX Official Notices**
- 발행사 corporate calendar
- 발행사 IR

주의:
- SIX의 통합 corporate calendar는 발행사 자발 입력 기반이므로 완전하지 않을 수 있다.
- 따라서 **SIX + issuer IR** 동시 수집이 필수다.

---

# 8.5 이탈리아

## 8.5.1 정본 계층
- Borsa Italiana issuer pages
- **Consob-authorized storage systems** (예: 1Info-Sdir, eMarket SDIR 등)
- issuer IR

## 8.5.2 매우 중요한 주의점
Borsa Italiana 페이지에 모든 regulated market issuer 공시가 다 모여 있다고 가정하면 안 된다.  
공식 저장 시스템과 발행사 공식 사이트를 함께 봐야 한다.

---

# 8.6 일본

## 8.6.1 정본 계층
- **TDnet**
- **JPX Company Announcements Service**
- **EDINET**
- 발행사 IR

## 8.6.2 일본 실적 전략
일본은 TDnet가 timely disclosure 정본이다.  
실적 요약, 수정, 자사주, 배당, 사업 재편 등 이벤트가 매우 강하다.

## 8.6.3 영어 커버리지 전략
- JPX Company Announcements Service의 영어 공시를 적극 활용
- 그러나 일본어 원문이 더 먼저/더 완전할 수 있다
- 영어가 불완전할 수 있으므로 **일본어 원문 우선 파싱 + 번역 보조**가 바람직하다

## 8.6.4 EDINET 역할
EDINET은 법정보고 계층이다.  
TDnet만으로 부족한 세부 보고서를 EDINET이 보완한다.

## 8.6.5 주의점
- Listed Company Search는 실시간 최신용이 아니라 보조 용도
- 최종 최신성 판단은 Company Announcements Service / TDnet 중심
- 2025~2026 전환 구간 동안 일부 영문 제공은 유예/예외가 있을 수 있음

---

# 8.7 대만

## 8.7.1 정본 계층
- **MOPS**
- TWSE / TPEx 공시/통계
- 발행사 IR

## 8.7.2 대만이 중요한 이유
대만은 반도체/전자 공급망 핵심 국가라서 아래가 특히 중요하다.

- 월매출
- 분기 재무
- 주요 공시
- 시설투자
- 공급 부족/수주 증가
- AI/서버/반도체 업황 변화

## 8.7.3 반드시 반영할 데이터
- MOPS material information
- 재무제표 / 연차보고서
- shareholder meeting 관련 자료
- **monthly operating revenues**
- TPEx quarterly financial reports

## 8.7.4 운영 강화 옵션
- **MOPS Push Server Service** 검토
- 영문 significant information 패키지 여부 검토
- 단, 영문보다 중국어 원문이 더 완전할 수 있다

---

# 8.8 중국 본토

## 8.8.1 정본 계층
- **CNINFO**
- **SSE announcements**
- **SZSE announcements**
- 발행사 IR / 회사 사이트

## 8.8.2 원칙
중국은 CNINFO가 매우 중요하지만, **CNINFO 단독 의존 금지**.  
SSE/SZSE와 issuer site cross-check가 필요하다.

## 8.8.3 언어 전략
중국은 **중국어 원문 우선**.  
영문 페이지는 보조로만 사용한다.

## 8.8.4 주의점
- 일부 기업은 지정 공시 매체와 CNINFO를 함께 표기
- 제3자 채널은 공식 채널보다 앞설 수 없다는 원칙을 기억
- 기업 구조조정/M&A는 거래소 공시와 규제 심사 단계를 분리해 추적해야 함

---

## 9. 실적 발표 커버리지를 “빠짐없이”에 가깝게 만드는 방법

## 9.1 핵심 원칙
실적은 “캘린더”가 아니라 아래 4층 구조로 수집해야 한다.

1. **공식 공시 정본**
2. **발행사 IR releases / results center**
3. **와이어 서비스**
4. **외신/분석 기사**

## 9.2 지역별 실적 파이프라인

### 미국
- SEC 8-K Item 2.02
- 10-Q / 10-K / 6-K / 20-F / 40-F
- issuer IR earnings release
- issuer earnings presentation / webcast info
- Business Wire / GlobeNewswire / PR Newswire
- Reuters/AP/WSJ/FT는 보조

### 유럽/영국
- RNS / Euronext regulated information / EQS / SIX / Consob storage systems
- issuer financial calendar
- issuer IR results page

### 일본
- TDnet financial results notices
- JPX Company Announcements Service
- issuer results presentation
- EDINET 법정보고

### 대만
- MOPS
- TWSE/TPEx financial reports
- MOPS material information
- monthly revenue와 실적을 연결

### 중국
- CNINFO 정기보고/중요공시
- SSE/SZSE announcement
- issuer IR

## 9.3 실적 이벤트 모델
`event_type = earnings`

하위 구분:
- preliminary
- scheduled
- released
- amended
- restated
- guidance_change
- post_call_update

필수 필드:
- `fiscal_year`
- `fiscal_quarter`
- `period_end`
- `reported_at`
- `revenue`
- `operating_income`
- `gross_margin`
- `eps`
- `guidance_revenue_low/high`
- `guidance_eps_low/high`
- `beat_miss_flags`
- `currency`
- `gaap_basis`
- `source_authority_rank`

---

## 10. M&A / 대규모 자금 이동 추적 설계

## 10.1 포함해야 할 이벤트 유형
- merger
- acquisition
- divestiture
- tender_offer
- strategic_review
- spin_off
- joint_venture
- minority_stake_purchase
- activist_stake
- follow_on_offering
- convertible_note
- bond_issue
- syndicated_loan
- buyback
- special_dividend
- bankruptcy / restructuring
- antitrust_review

## 10.2 M&A는 상태머신으로 관리
동일 거래가 며칠~수개월 동안 여러 단계로 진행되므로, 아래 상태머신이 필요하다.

- `rumor`
- `reported`
- `confirmed_talks`
- `signed`
- `shareholder_vote_pending`
- `regulatory_review`
- `second_request_or_remedy`
- `approved`
- `blocked`
- `terminated`
- `closed`

## 10.3 같은 거래의 중복을 막는 키
`deal_key` 예시:
- acquirer_entity_id
- target_entity_id
- announced_consideration_currency
- approximate_consideration_value
- deal_structure
- first_signed_date_bucket

## 10.4 국가별 M&A 정본 후보
- 미국: SEC + FTC/DOJ
- EU: European Commission competition merger cases
- 영국: CMA
- 일본: JFTC
- 대만: TFTC
- 중국: SAMR 관련 공개자료/공식 발표

## 10.5 실무적으로 중요한 점
외신은 “거래가 진행 중”을 먼저 알려줄 수 있다.  
하지만 canonical event의 상태 변경은 **공식 확인 또는 복수 고신뢰 소스**가 들어와야 한다.

---

## 11. 산업별 수요/공급, 부족, 수요 급증, 기술 변화 추적

## 11.1 별도 “산업 센서” 계층이 필요하다
기사는 사건을 말해주고, 지표는 왜 중요한지 설명한다.  
따라서 아래를 별도 파이프라인으로 운영한다.

## 11.2 추천 산업 센서 소스

### 에너지
- IEA Oil Market Report
- OPEC Monthly Oil Market Report
- EIA Weekly Petroleum Status Report

### 농업/원자재
- USDA WASDE

### 반도체
- WSTS forecast / historical billings
- TrendForce DRAM/NAND/Foundry/Server 관련 리서치

### 해운/물류
- Shanghai Shipping Exchange / SCFI
- 항공/해운 운임 지표 (필요 시 확장)

### 공급망
- 월매출/출하 데이터 (대만 등)
- 주요 부품 회사 실적 가이던스
- 설비투자 발표
- 생산차질 공시
- 화재/정전/지진 등 공급차질 사건

## 11.3 산업 이벤트 모델
`event_type = supply_demand_shift`

하위 구분:
- demand_spike
- demand_softening
- supply_shortage
- supply_recovery
- capacity_expansion
- capacity_cut
- price_increase
- price_decline
- technology_adoption
- product_launch_disruption

필수 필드:
- `industry`
- `product`
- `region_scope`
- `direction`
- `magnitude_text`
- `linked_entities[]`
- `supporting_indicators[]`

---

## 12. 외신 수집 전략

## 12.1 글로벌 backbone
- Reuters
- AP
- FT
- WSJ

## 12.2 아시아 보강
- Nikkei Asia
- Caixin Global
- SCMP Business/China

## 12.3 외신의 역할
외신은 정본이 아니라 다음 역할로 쓴다.

- 공식 공시 이전의 선행 신호
- 공식 공시를 투자 해석으로 연결
- 공급망/업계 문맥 제공
- 여러 지역 사건을 한 맥락으로 연결
- 루머와 공식 확인을 구분하는 보조 레이어

## 12.4 법적 정책
외신은 라이선스가 없으면 기본적으로:

- `link_only` 또는
- `excerpt_only`

로 다룬다.

원문 전문 저장/재배포는 라이선스 확인 후에만 허용한다.

---

## 13. 전달 시각 설계 (매우 중요)

사용자 요구:  
각 지역/국가별로 **저녁~밤, 즉 주요 뉴스 업로드 빈도가 크게 줄어드는 시점**에 받기.

이 요구를 만족하려면 **KST 기준 고정 시간이 아니라, 각 시장의 로컬 시간대를 기준**으로 스케줄을 계산해야 한다.

## 13.1 기본 원칙
- 스케줄은 **시장 로컬 시간 기준**
- 앱에서는 KST로 환산해 보여주되, 내부 스케줄은 IANA timezone 사용
- DST를 절대 하드코딩하지 말 것
- 거래소 휴장일/반일장/서머타임 전환 반영
- “대부분의 정보가 나온 뒤”를 목표로 하되, 너무 늦지 않게 설정

## 13.2 권장 기본 발송 시각
아래는 **운영 추천값**이다.  
절대적 진리가 아니라, 공식 장 종료 시각/공시 관행/저녁 배치 관행을 반영한 실무적 초기값이다.

### 미국 (ET 기준)
- **20:45 ET**
- 이유:
  - 정규장 종료 이후 실적/IR/8-K가 몰림
  - Nasdaq/NYSE 연장 거래와 post-market 반응이 20:00 ET 전후까지 존재
  - SEC 접수/인덱스 타이밍을 감안하면 20:45가 안전한 초기값

### 영국
- **20:45 Europe/London**
- 이유:
  - 장 종료 후 충분한 버퍼
  - RNS는 아침 배치가 강하지만 장중/장후도 존재
  - 저녁 시간대에 신규 밀도가 낮아짐

### 유럽 대륙 (파리/암스테르담/브뤼셀/프랑크푸르트 등)
- **20:30 local time**
- 이유:
  - 장 마감 이후 results release가 적지 않음
  - 19:00 전후 발표 사례가 있어 20:30이 안전
  - 국가별 격차를 감안한 범용값

### 스위스
- **20:30 Europe/Zurich**
- 이유:
  - issuer IR / calendar / 공시 반영 버퍼 필요

### 이탈리아
- **20:45 Europe/Rome**
- 이유:
  - authorized storage system 반영과 issuer IR 확인 버퍼 필요

### 일본
- **20:00 Asia/Tokyo**
- 이유:
  - 장 종료 이후 공시/IR 정리 시간 확보
  - 너무 늦게 잡을 필요는 상대적으로 적음

### 대만
- **20:00 Asia/Taipei**
- 이유:
  - 장 마감이 빠르고 MOPS/월매출/공시 정리가 비교적 일찍 끝남
  - 20시 이후 신규 밀도는 현저히 떨어질 가능성이 큼

### 중국 본토
- **20:30 Asia/Shanghai**
- 이유:
  - 거래소 장 종료 이후 공시/정기보고/회사 공지 확인 버퍼 필요
  - 저녁 시간대까지 반영 후 묶음 발송에 적합

## 13.3 브레이킹 예외 정책
사용자는 “그날 정리본”을 원하지만, 아래는 예외적으로 즉시 또는 추가 패킷 발송이 필요할 수 있다.

- 초대형 M&A 체결
- 반독점 승인/차단
- 파산/채무불이행
- 대규모 리콜/공장 셧다운
- 실적 대폭 하향/상향
- 국가 정책 충격

권장 정책:
- **기본은 1일 1회 지역별 정리**
- 단, `severity >= critical` 이면 별도 “긴급 보충 패킷” 허용

---

## 14. 백엔드 도메인 모델

## 14.1 핵심 테이블

### `source_registry`
소스 정의

### `source_credentials`
API key, session token, cookies, secret metadata

### `market_calendar`
거래소 영업일/반일장/DST/예외 일정

### `ingestion_checkpoint`
소스별 마지막 성공 위치
- last_cursor
- last_seen_timestamp
- last_seen_doc_id
- last_success_at
- consecutive_failures

### `raw_document`
원문 저장
- raw_document_id
- source_id
- fetched_at
- discovered_at
- source_doc_id
- source_url
- http_status
- content_type
- language_detected
- content_hash
- object_path
- legal_mode
- parser_status
- retry_count

### `normalized_document`
파싱/정규화 결과
- normalized_document_id
- raw_document_id
- title
- summary
- body_clean
- publication_time
- issuer_name_raw
- exchange_raw
- filing_type
- event_hints[]
- numeric_facts_json
- entities_candidate_json
- parser_version
- extraction_confidence

### `entity_master`
기업/기관/상품/기술/국가/산업 엔티티
- entity_id
- entity_type
- canonical_name
- aliases
- country
- primary_exchange
- ticker_list
- isin
- lei
- cik
- jp_corporate_no
- twse_code
- cn_stock_code
- confidence_meta

### `instrument_master`
주식/채권/전환사채/ETF 등

### `canonical_event`
정본 이벤트
- canonical_event_id
- event_type
- event_subtype
- state
- title_canonical
- entity_primary_id
- entity_secondary_id
- region
- countries[]
- industry
- occurred_at
- effective_at
- first_seen_at
- last_updated_at
- severity
- novelty_score
- confidence_score
- dedupe_fingerprint
- delivery_ready
- legal_mode_effective

### `event_evidence`
이벤트를 지지하는 증거 문서 연결
- canonical_event_id
- normalized_document_id
- authority_rank
- evidence_role (primary/secondary/contextual/rumor)
- extracted_claims_json
- extracted_numbers_json
- contradiction_flags
- linked_at

### `event_metrics`
실적/M&A 등 구조화 지표
- canonical_event_id
- metric_name
- metric_value_num
- metric_value_text
- unit
- currency
- period_label
- comparator
- source_priority

### `delivery_window`
시장별 발송 정책
- region_key
- timezone
- target_local_time
- grace_minutes
- critical_override_enabled
- holiday_policy

### `delivery_packet`
사용자에게 보낸 정리본
- packet_id
- user_id
- region_key
- window_date_local
- generated_at
- packet_version
- canonical_event_ids[]
- digest_markdown
- push_status
- ack_status

### `audit_log`
파이프라인 의사결정 기록
- job_id
- action
- input_refs
- output_refs
- decision_json
- created_at

---

## 15. 이벤트 병합(중복 제거) 설계

이 프로젝트의 성패를 좌우하는 핵심이다.

## 15.1 목표
동일 사건이 여러 소스에서 들어와도 **한 이벤트**로 묶는다.  
반대로 비슷해 보이지만 다른 사건은 **절대 합치지 않는다.**

## 15.2 병합 5단계

### 1단계: 엄격 키 병합
아래가 같으면 높은 확률로 동일 사건이다.
- SEC accession / filing id
- TDnet document id
- MOPS announcement id
- CNINFO announcement id
- RNS id
- deal id / case id / filing accession

### 2단계: 이벤트 키 병합
- `entity_id + event_type + fiscal_period`
- `acquirer + target + deal_type + signed_date_bucket`
- `entity + guidance_change + quarter`

### 3단계: 숫자 시그니처 병합
예:
- revenue, eps, guidance range, consideration value, premium
- 금액/통화/퍼센트 조합이 유사하면 같은 이벤트 후보

### 4단계: 텍스트/임베딩 유사도
- 제목 정규화
- boilerplate 제거 후 본문 유사도
- cross-language translation-normalized similarity

### 5단계: 권위 우선 규칙
공식 공시와 외신이 충돌하면:
- 정본은 공시
- 외신은 contextual evidence로 남긴다

## 15.3 병합 금지 규칙
다음은 유사해 보여도 합치면 안 된다.

- 같은 기업의 **Q1 실적** 과 **가이던스 수정**
- 같은 M&A의 **루머 단계** 와 **체결 단계** 를 하나로 완전히 덮어쓰기
- 같은 날 나온 **서로 다른 공장 증설 2건**
- 같은 산업 내 **서로 다른 기업의 공급 부족 기사**
- 한 회사의 **실적 발표** 와 **earnings call 코멘트 후 주가해설 기사**

## 15.4 정정/개정 관리
- `canonical_event.version`
- `is_amended`
- `supersedes_event_id`
- `restatement_reason`

중요:
정정은 “새 이벤트 생성”보다 “기존 이벤트 업데이트 + 이력 남김”이 더 적합한 경우가 많다.

---

## 16. 추출 파이프라인 상세

## 16.1 Ingest
- source fetch
- source metadata 저장
- checksum 계산
- object storage 업로드
- raw_document 생성

## 16.2 Parse
- HTML/PDF/XML/XBRL/JSON 별 파서
- boilerplate 제거
- 표/수치 추출
- 언어 판별
- 제목/본문 정규화

## 16.3 Normalize
- 회사명 정규화
- ticker/ISIN/CIK 등 식별자 매핑
- 날짜/시간대 통일
- 숫자/통화/단위 변환
- 이벤트 힌트 생성

## 16.4 Extract
- rule-based extractor
- filing type specific extractor
- M&A extractor
- earnings extractor
- supply-demand extractor

## 16.5 Candidate Build
- entity 후보 선택
- event subtype 후보 생성
- 숫자 시그니처 계산
- merge candidate 검색

## 16.6 Merge
- 기존 canonical_event와 비교
- confidence 점수 계산
- 새 이벤트 생성 또는 기존 이벤트 업데이트

## 16.7 Enrich
- 업계 지표 연결
- 국가/산업 태깅
- 외신 contextual attach
- 번역/요약
- severity 산정

## 16.8 Delivery Build
- 지역별 window 기준 이벤트 선택
- 중복 재검사
- 요약 생성
- 패킷 저장
- 알림 발송

---

## 17. 백엔드 스케줄링 설계

## 17.1 큐 분리
반드시 큐를 분리한다.

- `ingest_regulatory`
- `ingest_issuer`
- `ingest_wire`
- `ingest_editorial`
- `parse_heavy`
- `merge_core`
- `enrich_translation`
- `delivery_build`
- `backfill`
- `healthcheck`

## 17.2 Oban 설계 원칙
- unique job 적용
- retry with exponential backoff
- per-source concurrency 제한
- dead-letter 성격의 실패 분석 큐
- 오래된 원문 재파싱용 low-priority queue
- 지역별 peak time 분리

## 17.3 스케줄링 세분화
예시:
- SEC latest filings: 2~5분 주기
- TDnet / JPX: 2~5분 주기
- MOPS/CNINFO/거래소 announcement: 3~10분 주기
- 와이어/외신 RSS/API: 1~3분 또는 5분 주기
- 산업 지표: 일/주/월 주기
- issuer IR crawling: 공시 시즌에는 빈도 상향

## 17.4 체크포인트 전략
소스마다 다음 중 하나를 사용
- incremental cursor
- timestamp cursor
- page token
- source document id watermark
- content hash recent cache

---

## 18. 데이터베이스 설계 원칙

## 18.1 PostgreSQL 우선
초기 단계에서는 PostgreSQL을 중심으로 간다.

- transactional integrity
- JSONB 유연성
- FTS
- partitioning
- RLS
- logical replication
- PITR

## 18.2 파티셔닝
다음 테이블은 파티셔닝을 적극 검토한다.
- raw_document (by month)
- normalized_document (by month)
- event_evidence (by month or quarter)
- audit_log (by month)

## 18.3 인덱스
반드시 고려할 인덱스:
- source_id + discovered_at desc
- source_doc_id unique (nullable strategy carefully)
- content_hash
- canonical_event dedupe_fingerprint
- entity aliases gin/trgm
- normalized_document title/body tsvector
- event_metrics metric_name + metric_value_num
- delivery_packet user_id + generated_at desc

## 18.4 무결성 제약
- foreign key는 가능한 적극 사용
- unique constraint로 source duplication 방지
- enum-like domains는 application + db check로 이중 방어
- legal_mode downgrade는 허용, upgrade는 승인 필요

## 18.5 백업/복구
- WAL archiving
- PITR
- object storage 버전관리
- daily logical backup (schema + critical tables)
- 정기 restore drill

---

## 19. 검색 설계

## 19.1 1단계 검색
PostgreSQL FTS + trigram + structured filters로 시작한다.

필터:
- region
- country
- event_type
- entity
- industry
- severity
- source_level
- date range

## 19.2 2단계 검색
나중에 필요하면 pgvector를 추가한다.
초기에는 아래까지만 해도 충분할 수 있다.
- canonical_event embedding
- normalized_document embedding
- query expansion

## 19.3 검색과 중복 제거는 분리
검색이 유사하다고 이벤트가 같다는 뜻은 아니다.  
merge engine과 search engine을 분리해야 한다.

---

## 20. 관측성/운영

## 20.1 필수 메트릭
- source poll success rate
- parse success rate
- normalization confidence
- merge false-positive rate
- merge false-negative suspicion rate
- event creation latency
- delivery packet completeness score
- source freshness lag
- queue depth
- job retry count
- object storage write failures
- DB bloat indicators

## 20.2 로그
구조화 로그 필수:
- source_id
- job_id
- raw_document_id
- canonical_event_id
- parser_version
- decision_reason
- latency_ms

## 20.3 알람
- 특정 source 3회 연속 실패
- 주요 source freshness lag 증가
- delivery build miss
- parse regression
- dedupe spike
- DB replication/backups failure
- object storage checksum mismatch

## 20.4 운영 대시보드
- region freshness map
- source health board
- top failing parsers
- event merge review queue
- delivery packet preview
- licensing/rights violations board

---

## 21. 권한/법률/라이선스 설계

## 21.1 소스별 권리 모드
`legal_mode`
- `link_only`
- `excerpt_only`
- `licensed_fulltext`
- `restricted_internal`
- `do_not_store_body`

## 21.2 정책
- Reuters/AP/일부 외신은 라이선스 없는 전문 저장/재배포 금지 원칙
- 공식 공시/발행사 보도자료는 저장 가능성이 높지만, 각 사이트 정책 확인 필요
- 와이어는 계약/정책 확인 후 저장 범위 결정
- 법적 불확실성이 있으면 body는 저장하지 말고 summary + URL + checksum만 저장

## 21.3 UI 정책
사용자에게 보여주는 내용도 `legal_mode`에 따라 다르게 한다.
- link_only: 제목 + 메타 + 링크만
- excerpt_only: 제한된 발췌 + 링크
- licensed_fulltext: 전문 또는 구조화 요약
- do_not_store_body: 원문 재호출 또는 메타만

---

## 22. 번역/요약 전략

## 22.1 원문 우선 저장
일본어/중국어/독일어/프랑스어 등 원문을 우선 보존한다.

## 22.2 번역은 파생 데이터
- `translated_title`
- `translated_summary`
- `translation_model_version`
- `translation_confidence`

## 22.3 요약 전략
요약은 원문 대체가 아니라 아래 3단 구조를 추천:
- **한 줄 요약**
- **투자 관점 핵심**
- **증거/근거 목록**

## 22.4 LLM 사용 원칙
- 수치 추출의 최종 정본은 rule-based / parser-based
- LLM은 해석/요약/번역 보조
- M&A 상태/실적 수치의 최종 값은 LLM 단독 추출 금지

---

## 23. 네이티브 앱 설계 (Frontend)

## 23.1 왜 Slint를 1안으로 두는가
- Rust 친화적
- declarative UI
- localhost 불필요
- 배포가 단순
- 네이티브 데스크톱 감각이 좋음

## 23.2 앱 화면 구조
- 홈 피드
- 지역별 피드
- 기업 페이지
- 이벤트 상세
- 공급망/산업 대시보드
- 실적 캘린더/히스토리
- M&A 트래커
- 검색
- 설정
- 수동 검토/북마크

## 23.3 이벤트 상세 화면
필수 요소:
- canonical title
- event type / state
- 핵심 수치
- 관련 기업
- 국가/산업
- 원문 증거 목록
- 공식 공시 링크
- 외신 맥락 기사
- 정정 이력
- 사용자가 왜 봐야 하는지

## 23.4 로컬 캐시
- SQLite 또는 sled/sqlite 계열
- 마지막 패킷 오프라인 열람
- 읽음 상태/북마크/필터 로컬 저장
- 로컬에 라이선스 위반 원문이 저장되지 않도록 주의

---

## 24. API 설계

## 24.1 주요 엔드포인트
- `GET /v1/feed?region=us&date=2026-03-18`
- `GET /v1/events/:id`
- `GET /v1/entities/:id`
- `GET /v1/entities/:id/events`
- `GET /v1/regions`
- `GET /v1/delivery/windows`
- `GET /v1/search`
- `GET /v1/metrics/coverage`
- `POST /v1/preferences`
- `GET /v1/admin/source-health`
- `GET /v1/admin/review/merge-candidates`

## 24.2 WebSocket / push
- 새 패킷 도착
- 긴급 보충 패킷
- source outage 알림(관리자용)
- 수동 검토 결과 반영

---

## 25. 기술 스택별 장단점 비교

| 영역 | 후보 | 장점 | 단점 | 추천도 |
|---|---|---|---|---|
| Backend runtime | Elixir/OTP | 장애 격리, 동시성, supervision, 운영성 | 팀 러닝커브 | 매우 높음 |
| Backend runtime | Rust only | 성능/타입 안정성 | IO orchestration 생산성 낮을 수 있음 | 중간 |
| API layer | Phoenix | 성숙, websocket 강함 | Elixir 러닝커브 | 매우 높음 |
| Job system | Oban | Postgres 기반 durable jobs | 초대형 scale에서 Kafka보다 단순 | 매우 높음 |
| DB | PostgreSQL | 강한 정합성, FTS, JSONB, RLS, PITR | 문서 저장 대규모화 시 관리 필요 | 매우 높음 |
| Object store | MinIO/S3 | 원문 아카이브 적합 | 별도 운영 필요 | 높음 |
| GUI | Slint | Rust 친화, 네이티브, localhost 불필요 | 웹 생태계보다 컴포넌트 적음 | 매우 높음 |
| GUI | Tauri 2 | 생태계 풍부, 배포 성숙 | WebView/JS 계층 추가 | 높음 |
| Messaging | Postgres + Oban | 단순, 운영 쉬움 | 거대 규모 fanout엔 한계 | 높음 |
| Messaging | RabbitMQ | 라우팅 유연 | 운영 컴포넌트 증가 | 중간 |
| Messaging | Kafka | 대규모 스트리밍/리플레이 강함 | 초기 과설계 가능성 | 중간 |

---

## 26. “백엔드 기반 공사를 매우 탄탄하게” 하기 위한 구체 원칙

이 섹션이 가장 중요하다.

## 26.1 1단계에서 반드시 넣어야 할 것
아래는 “나중에”가 아니라 첫 단계부터 넣는다.

- source registry
- source credentials vault integration
- ingestion checkpoint
- object storage raw archive
- parser versioning
- canonical event model
- event evidence graph
- legal_mode enforcement
- delivery windows by timezone
- structured audit log
- retry/backoff/circuit breaker
- data retention policy
- restore procedure document

## 26.2 절대 처음부터 넣지 말 것
- Kafka
- microservice 8개 이상
- 벡터 DB 별도 구축
- 실시간 사용자별 개인화 ranking 대공사
- 과도한 AI 분류기

## 26.3 성공 기준
초기 성공 기준은 다음이어야 한다.
- 미국/일본 실적과 M&A 누락률이 낮다
- 같은 사건이 두 번 이상 거의 오지 않는다
- 장애 발생 시 재처리가 쉽다
- 소스 추가가 스키마 파괴 없이 가능하다
- 로컬 앱은 얇고 빠르다

---

## 27. 단계별 구현 계획 (한 번에 하나씩)

# Phase 0 — 토대 공사

목표: 아키텍처 뼈대와 데이터 모델 확정

### 해야 할 일
1. monorepo 생성
2. backend app 생성 (Phoenix API only)
3. Oban 도입
4. PostgreSQL 스키마 초안
5. object storage 연동
6. source_registry / raw_document / normalized_document / canonical_event / event_evidence 생성
7. delivery_window 테이블 생성
8. OpenTelemetry/logging 기본 구성
9. admin source health API 생성
10. market calendar 기본 구조 생성

### 종료 조건
- raw 문서를 저장할 수 있다
- parse 결과를 저장할 수 있다
- canonical_event를 생성할 수 있다
- 앱 없이도 API로 검증 가능하다

---

# Phase 1 — 미국 + 일본 최소 정본 시스템

목표: 가장 강한 공시 지역으로 핵심 도메인 완성

### 미국
- SEC EDGAR ingest
- 8-K / 10-Q / 10-K / 6-K / 20-F extractor
- issuer IR earnings release parser
- Reuters/AP/FT/WSJ 링크형 contextual attach
- M&A filing extractor

### 일본
- TDnet ingest
- JPX Company Announcements Service ingest
- EDINET ingest
- 일본어 원문 파서/번역 파이프라인

### 공통
- entity master v1
- earnings event model
- M&A event model
- delivery packet builder v1
- dedupe rules v1

### 종료 조건
- 미국/일본 실적/거래 이벤트가 하루 단위 digest로 정상 생성
- 중복 제거가 대체로 작동

---

# Phase 2 — 대만 + 중국 공급망 축 확장

목표: 반도체/전자 공급망 강화를 위한 아시아 심화

### 대만
- MOPS ingest
- monthly revenue ingest
- TWSE/TPEx financial report ingest
- material information parser

### 중국
- CNINFO ingest
- SSE/SZSE announcements ingest
- 중국어 원문 정규화
- issuer IR 보강

### 종료 조건
- 대만 월매출과 실적이 연결된다
- 중국 공시와 외신 맥락이 같이 묶인다

---

# Phase 3 — 유럽 분절 시장 통합

목표: 유럽의 “단일 소스 없음” 문제를 구조적으로 해결

### 영국
- RNS ingest

### Euronext
- regulated information + issuer financial calendar

### 독일/오스트리아
- EQS + issuer IR + Deutsche Börse 관련 페이지

### 스위스
- SIX official notices + issuer corporate calendar + issuer IR

### 이탈리아
- Borsa issuer pages + authorized storage system mapping + issuer IR

### 종료 조건
- 유럽은 국가별 편차가 있지만, 최소한 “공식 공시 합집합” 구조가 작동
- 단일 사이트 환상 없이 지역별 completeness가 측정됨

---

# Phase 4 — 산업 센서 계층

목표: 기사/공시를 공급·수요·가격·기술 변화와 연결

### 해야 할 일
- IEA/OPEC/EIA 파이프라인
- USDA WASDE
- WSTS / TrendForce
- 해운 운임 지표
- 공급 부족 이벤트 linking rules

### 종료 조건
- 이벤트 상세에서 “왜 중요한가”를 지표와 함께 보여줄 수 있다

---

# Phase 5 — 품질 고도화

목표: 운영성과 정확도 향상

### 해야 할 일
- merge review UI
- false positive/negative feedback loop
- translation QA
- ranking 개선
- watchlist / sector profile
- multi-user/team mode

---

## 28. Codex Agent용 monorepo 구조 제안

```text
repo/
  apps/
    backend/
      lib/
        backend/
          sources/
            sec/
            rns/
            tdnet/
            edinet/
            mops/
            cninfo/
            sse/
            szse/
            euronext/
            eqs/
            six/
            borsa/
            wires/
            editorial/
          ingest/
          parse/
          normalize/
          entities/
          events/
          delivery/
          observability/
          legal/
          market_calendar/
        backend_web/
          controllers/
          channels/
          json/
      priv/
        repo/migrations/
      test/
    desktop/
      src/
        app/
        screens/
        widgets/
        api/
        cache/
        models/
      ui/
    rust_services/
      parser_core/
      xbrl_core/
      entity_matcher/
      merge_engine/
      common_types/
  docs/
    architecture/
    runbooks/
    schemas/
    source_profiles/
  infra/
    docker/
    terraform/
    scripts/
```

---

## 29. Codex Agent용 구현 규칙

## 29.1 절대 규칙
- source별 parser를 섞지 말 것
- raw_document는 immutable
- normalized_document도 가능한 immutable + versioned
- canonical_event만 업데이트 가능
- 원문 삭제보다 legal_mode downgrade를 우선
- timezone은 항상 source-local + UTC 둘 다 저장
- 숫자는 원문 텍스트와 정규화 수치를 같이 저장
- merge decision은 audit_log에 남길 것

## 29.2 코딩 규칙
- Elixir: context boundary 엄격히 유지
- Rust: parser crate를 source profile별로 분리
- integration test는 source fixture 기반
- HTML snapshot fixture를 반드시 저장
- parser change는 regression fixture 통과 필수

## 29.3 테스트 전략
- source parser golden tests
- filing-type extractor tests
- dedupe property tests
- timezone/DST tests
- legal mode tests
- packet generation snapshot tests
- backfill idempotency tests

---

## 30. 리스크 레지스터 (작은 가능성까지 포함)

아래는 일부가 아니라 **가능한 한 넓게** 적는다.

## 30.1 소스/법률 리스크
1. 외신 전문 저장/재배포 라이선스 위반
2. 와이어 원문 사용 범위 오해
3. 사이트 이용약관 변경
4. robots 정책 변경
5. anti-bot 강화
6. API 폐쇄/유료화
7. 세션 기반 사이트 접근 실패
8. 공시 페이지 구조 변경
9. PDF 형식 변경
10. 공시 포맷의 언어 혼합
11. 원문 삭제/정정으로 링크 깨짐
12. 일부 국가에서 영문 페이지가 지연/불완전
13. 유럽 지역별 공시 저장소 분산으로 completeness 과신
14. 중국/일본/대만에서 영어만 보면 누락 발생
15. 기업 IR 사이트 redesign로 크롤러 깨짐
16. 보도자료 중복 syndication으로 과대 카운트

## 30.2 데이터 품질 리스크
17. 같은 회사 다른 ticker/ISIN 매핑 오류
18. ADR와 본주 혼동
19. 같은 기업의 자회사/사업부 이벤트를 모회사로 잘못 귀속
20. 환율/통화 단위 오해
21. million/billion/백만/억 단위 혼동
22. quarter/fiscal year 기준 차이
23. 일본/대만 회계 용어 해석 오류
24. 정정 공시를 새 이벤트로 잘못 처리
25. 서로 다른 거래를 하나로 합침 (false positive)
26. 같은 거래를 여러 개로 쪼갬 (false negative)
27. 루머 기사와 공식 체결을 같은 상태로 합침
28. 실적 수치와 guidance 수치를 혼동
29. headline만 보고 이벤트 유형 오분류
30. boilerplate가 실제 핵심 내용을 덮음
31. 표 파싱 실패
32. inline XBRL 파싱 오류
33. OCR 의존 시 수치 오인식
34. 시간대 변환 실수
35. DST 전환일 스케줄 오류
36. 반일장/휴장일 미반영
37. source timestamp와 fetch timestamp 혼동
38. file acceptance time과 public availability time 혼동

## 30.3 시스템 리스크
39. job 폭주
40. retry storm
41. source outage가 다른 큐를 막음
42. parser crash가 전체 파이프라인을 늦춤
43. sidecar backpressure로 timeout 증가
44. NIF 사용 시 VM 영향
45. DB connection pool 고갈
46. raw_document 급증으로 storage 비용 증가
47. event_evidence 급증으로 query 느려짐
48. partition maintenance 누락
49. autovacuum 부실
50. object storage 장애
51. checksum mismatch
52. backup은 있는데 restore 검증이 안 됨
53. migration rollback 실패
54. unique job 설정 오류로 중복 처리
55. idempotency key 누락으로 재처리 duplication
56. cache staleness
57. logical replication lag
58. alert는 울리는데 runbook이 없음
59. admin review queue 누적
60. source credentials 만료
61. secret rotation 실패
62. TLS/CA 문제로 외부 연결 실패

## 30.4 제품/UX 리스크
63. 너무 많은 알림으로 사용자 피로
64. 너무 적은 알림으로 누락 체감
65. 같은 사건의 맥락 기사가 사라져 “왜 중요한지” 전달 실패
66. 요약은 좋은데 원문 증거 연결이 약함
67. 중요도 점수가 납득되지 않음
68. 지역별 발송 시간이 너무 이르거나 늦음
69. KST 환산만 보다가 현지 기준을 잃어버림
70. 사용자가 “중복 제거”를 신뢰하지 못함
71. 고급 필터가 과도해 사용성이 떨어짐

## 30.5 조직/운영 리스크
72. 한 명만 소스 구조를 알고 있음
73. source parser 유지보수가 특정 개발자에게 종속
74. 문서화 부족
75. 테스트 fixture가 너무 적어 회귀를 못 잡음
76. 구현 속도 때문에 legal/reliability가 후순위로 밀림
77. 처음부터 소스 수를 과도하게 늘려 품질이 무너짐
78. 운영 비용 추정 실패
79. 상용 라이선스 계약 범위 오판
80. 지역 확장 시 품질 기준이 무너짐

## 30.6 리스크 대응 원칙
- “더 많은 소스”보다 “더 높은 정본성 + 더 좋은 병합” 우선
- 새 소스 추가 전 source profile 문서화
- parser 변경 시 회귀 테스트 강제
- legal_mode는 소스 등록 단계에서 의무화
- 운영 메트릭 없는 기능은 배포 금지
- 영어가 불완전한 지역은 원문 우선 파이프라인 유지
- 발송 시각은 하드코딩 금지, 운영 데이터 기반 튜닝

---

## 31. 초기 운영 정책

## 31.1 completeness score
각 지역별로 아래 점수를 추적한다.
- 공식 공시 수집 성공률
- 실적 이벤트 탐지율
- M&A 상태 업데이트율
- source freshness lag
- dedupe precision / recall 추정
- 수동 검토 비율

## 31.2 packet 품질 기준
한 패킷은 다음을 만족해야 한다.
- 중복 이벤트 없음
- 각 이벤트에 최소 1개 이상의 강한 증거
- 가능한 경우 공식 공시 우선 링크 포함
- 수치가 있는 이벤트는 구조화 수치 포함
- legal_mode 위반 없음

---

## 32. MVP 범위 추천

## 32.1 반드시 포함
- 미국
- 일본
- 대만
- 중국
- 영국
- 유럽(프랑스/독일/스위스/이탈리아 최소 축)
- 실적
- M&A
- 수요/공급 변화
- 산업 센서 일부
- 지역별 daily packet

## 32.2 나중에
- 사용자별 watchlist ranking
- 브로커/데이터벤더 연동
- 실시간 옵션 flow
- 포트폴리오 자동 분석
- 모델 기반 anomaly detection 고도화

---

## 33. 최종 추천안

### 아키텍처
- **Phoenix API + Oban + PostgreSQL + Object Storage + Rust Sidecars + Slint Desktop**

### 구현 순서
1. 데이터 모델
2. source registry
3. 미국/일본 정본 수집
4. canonical event / evidence
5. delivery packet
6. 대만/중국
7. 유럽
8. 산업 센서
9. 품질/운영 고도화

### 가장 중요한 KPI
- 기사 수가 아니라 **정확히 병합된 canonical event 수**
- 빠른 알림 수가 아니라 **누락 없이 신뢰 가능한 daily packet 완성도**
- 멋진 UI가 아니라 **백엔드 파이프라인의 정합성/복구성/확장성**

---

## 34. Codex Agent에 바로 줄 작업 지시문 예시

```md
우선 Phase 0만 구현하라.

목표:
- Phoenix API only backend 생성
- PostgreSQL 스키마 생성
- Oban 설정
- source_registry, raw_document, normalized_document, canonical_event, event_evidence, ingestion_checkpoint, delivery_window 테이블 생성
- object storage 업로드 abstraction 생성
- structured logging / OpenTelemetry 기본 설정
- admin source health endpoint 추가

제약:
- raw_document는 immutable
- timezone은 UTC와 source local 모두 저장
- legal_mode 필수
- parser_version 필수
- 모든 migration은 rollback 가능하게 작성
- 테스트 포함

완료 조건:
- fixture raw document를 넣으면 normalized_document 저장 가능
- canonical_event 생성 API 동작
- source health endpoint가 최근 ingest 상태를 반환
```

---

## 35. 마지막 체크리스트

시작 전에 반드시 확인할 것:

- [ ] 소스별 legal_mode 정의 완료
- [ ] market calendar / timezone 정책 확정
- [ ] canonical_event 스키마 확정
- [ ] dedupe policy 문서화
- [ ] source onboarding 템플릿 준비
- [ ] parser regression fixture 전략 준비
- [ ] backup + restore drill 계획 존재
- [ ] 운영 대시보드 최소 세트 정의
- [ ] Phase 1 범위가 “미국 + 일본”으로 고정됨
- [ ] UI는 backend 안정화 후 얇게 시작

---

이 설계의 핵심은 단 하나다.

> **뉴스 수집 앱을 만드는 것이 아니라, 공식 공시와 외신을 정규화해 “중복 없는 투자 이벤트”를 생성하는 백엔드 시스템을 짓는 것이다.**

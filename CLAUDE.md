# 컨텍스트 이전 프롬프트 — Gon (허형곤) NH농협 IT센터 프로젝트

---

## 나는 누구인가

나는 **허형곤(Gon)**이다. 식품서비스 매니저 7년차에서 백엔드 개발로 전향 중이며, 부트캠프(Java/Spring/React)를 수료하고 현재 **NH농협 IT센터 인덕원**에서 3개월 SI 프로젝트에 투입 중이다. 첫 실무 프로젝트이며, 팀에는 경력 13~15년의 개발자 2명과 함께 있다.

---

## 프로젝트 환경

- **팀**: 채널팀 (비교대출 모바일웹)
- **기술 스택**: Spring Boot + 농협 자체 프레임워크("프로프레임웍"), MyBatis, JSP 기반 모바일웹(MT), Eclipse STS
- **서버 환경**: dev 28700=MT(모바일웹), dev 28800=PT(기존 앱, PC브라우저에서 동작하는 유일한 테스트환경), predev=스마트데브 서버
- **네트워크**: 폐쇄망(업무 PC 인터넷 불가, 개인폰으로 검색/GPT 사용)
- **Git**: SourceTree, TI/PT/MT/CM 4개 별개 저장소. origin/feature/channel 브랜치 기준 작업+Push. 배포는 수석님이 SmartDev로 채널 브랜치 기준 진행
- **DB**: Oracle, AutoCommit OFF (반드시 수동 COMMIT)
- **암호화**: PETRA(농협 표준), dev서버에서는 자동 동작, 로컬은 평문

---

## 커밋 시 절대 금지 사항 (보안)

1. `|| true` 조건 우회
2. 하드코딩된 cusno(고객번호)
3. `[CBT]`/`## filtered` 등 임시 디버그 로그
4. `List2.add` 표시줄
5. 테스트용 System.out.println, console.log
6. .rej 파일
7. local.yml(비밀번호 파일) — git update-index --assume-unchanged 설정됨
8. RCIS9591IR_IN_MST.java 등 **자동생성 DTO 파일은 수정 금지** (황희중 과장이 등록해서 MCI 전문 DTO가 자동생성된 것)
9. 실제 고객번호/사번은 마스킹해서 캡처/공유
10. PL 데이터는 건드리지 말 것

---

## 현재 담당 과제들

### 1. CBT (신규 제휴사 사전공개) — 완료
- ps310(제휴사) + ps320(CBT 제휴사) → fisCodeList addAll
- ps315(상품) + ps325(CBT 상품) → lnList addAll
- CBT 대상이면 거래제한 무시, 정식출시 상품은 NOT EXISTS로 제외
- EnvUtil.isCbt()/getCbtYn()은 CBT와 무관(운영환경 판단용) → if문 자체 삭제됨
- cbtOnly 플래그로 CBT 전용 제휴사는 ps315 스킵+ps325만 실행

### 2. MCI 사업자 신용대출 가심사요청 — 코드 완료, 실데이터 대기
- 기존 직장인(RCIS9590IR) 구현에 사업자(RCIS9593IR) 추가
- **호출체인 (10/28 PL 피드백 반영 후)**: fcccb1100tC.reqLoanLimit → fcccb1100tS.execute() → fctcb1100tS.reqLoanLimitTc(jonCls="5" 분기) → fctcb1101tS.callBusinessMciEai → setBusinessLoanMciBody/setMciBody
- 분기 기준: jonCls="5"(개인사업자), TI Controller에서 @McuCallID 배열로 처리
- IN/OUT 비동기 구조: 가심사요청(IN) → 접수확인 → 역거래로 심사결과 수신
- **역거래(OUT)**: fctcb1200tC/tS/tD — 사업자/직장인 분기 없이 동일 경로 처리
- rqr_tr_tpc="03"(사업자 신규), **대환("04")은 범위 제외** (PL 지시)

#### 10/28 완료 사항
- isBiz 파라미터 제거 (fcccb1100tC, fcccb1100tS)
- reqLoanLimitBizTc 삭제 → reqLoanLimitTc에 jonCls="5" 사업자 분기 통합
- setMciBody 상품코드 분기: getEffcCore("aflcnC", aflcnC) → PD301
- common_ti_config.yml aflcnC PD301 매핑 추가
- 로컬 디버그: PD301 정상 확인
- dev 배포: MCI 가심사요청 송신 정상, 역거래 수신 정상 (제휴사 결과값 미유입으로 유효성 오류)

#### MCI 관련 임시코드 (삭제 예정, 커밋 금지)
1. scrpgSucYn 강제 "Y" 세팅 — **삭제 대상** (PL 확인됨: 스크래핑 성공 후에만 전송)
2. 디버그 System.out.println/log.info 전체
3. bodyMap 로그(resultBodyMap log.info)
4. resultMap OUT 로그

#### PL 대기/미해결 사항
- 스크래핑 진입점 연결: 황인준 과장이 처리 중 → 이게 돼야 TB_EF_FC_CH_HTAX_PSN_BZMN에 실데이터 적재
- 사업자 전용 4필드(io_dh_business_id/comm_bzmn_yn/nts_pnp_bzcc/io_ptax_dlq_yn): 스크래핑 연결 전까지 가값

---

## 소스 인스펙션 (코드 품질 검사) — 완료

분석 ID 325789 기준 수정 완료한 이슈들:
- **ARIAUtil.java**: FORWARD_NULL (line 129 decrypt), getPaddingCntPKCS7 null 체크 추가
- **TIUtil.java**: 배열 인덱스 미검증 → 범위 체크 추가, INTEGER_OVERFLOW (line 537/540) → long 캐스팅
- **fccmb0001tS.java**: NULL_RETURN_STD (line 50) → CipherUtil.java 내부 apiKey null 체크 추가
- **fcdcb5500tS.java**: UNCHECKED_NULL (lines 90/91/92)
- **ARIAEngine.java**: BAD_OPERATOR.MULTIPLE_VALUE_ASSIGNMENT (lines 353/356) → 체이닝 할당 분리
- **RCIS9591IR_IN_MST.java**: 자동생성 DTO 파일 — PL에게 예외 처리 확인 필요

---

## 단위테스트 (IQMS) — 진행 중

### 테스트 환경 제약
- dev PT(28800): PC 브라우저 동작O — 주요 테스트 환경
- dev MT(28700): PC 불가(앱 전용)
- 상품보기/제휴사 이동: opa.execute 앱 브릿지 필요 → PC 불가
- 로컬 MT: ServiceClient 빈 에러 → **10/27 해결 (buildpath에서 MT가 effccom을 보고 있어서 발생, 팀장이 수정)**
- **104번(실행결과통지) 역거래**: dev 환경에서 수신된 적 없음 → 관련 증빙 제한

### 테스트케이스 번호 및 진행 현황

**TS-CP-0028 (당행연계 가심사요청 IN, 139~146):**
- 139 대출금리한도조회요청 — 완료
- 140 상품정보조회 — 완료
- 141 암호화 — dev서버 PETRA 자동동작으로 가능
- 142 송신전문저장 — 완료
- 143 한도조회내역 저장여부 — 미진행
- 144 전문송신 — 완료 (MCI 송신(300) 내역 캡처)
- 145 대출금리한도조회요청 상태저장 — 양식 작성함
- 146 오류처리여부 — 코드 리뷰 기반 확인(에러 처리 소스코드 캡처)

**TS-CP-0029 (결과통지 역거래 수신, 147~150):**
- 147 결과통지전문수신 — 완료 (관리자페이지 + 코드 리뷰)
- 148 수신전문저장여부 — 완료 (관리자페이지 + DB 조회)
- 149 한도조회내역 수정 여부 — 양식 제공 완료
- 150 오류처리여부 — 완료 (fctcb1200tS E1/E2 에러 코드 캡처)

**TS-CP-0030 (대출신청접수, 151~152):**
- 151 대출실행내역 저장여부 — 양식 제공 완료 (fccb1400tD + fccb04_sql.xml INSERT 구조)
- 152 — 완료

**TS-CP-0031 (실행결과통지, 153~159):**
- 153 실행결과통지 전문 수신 — PASS (관리자페이지 수신 내역, 102번 데이터 기준)
- 154 수신전문 저장여부 — PASS (관리자페이지 상세)
- 155 대출실행내역 저장여부 — 진행중 (DB 두 테이블 비교 확인 완료, Orange DBMS 캡처 중)
- 156 대출실행내역 수정 여부 — 미진행
- 157 응답 전문 송신 확인 — **보류** (104번 역거래 수신 이력 없어 응답 송신 확인 불가)
- 158 송신 응답 전문 저장 여부 — **보류** (동일 사유)
- 159 오류 처리 여부 — **보류** (동일 사유)

**TS-CP-0031-089 (신용대출 상품 신청):**
- 89 신용대출 상품 신청 — PASS (상품목록→상세→신청→제휴사URL 이동 캡처)

### IQMS 증빙 양식 패턴
```
[테스트 방법]
~을 확인한다

[수행 내역]
1. ~한다
2. ~확인한다

[결과]
~확인하였다

[첨부]
(캡처 첨부)
```

### MCI 거래로그 번호 의미
- 100번 = 우리 WAS → MCI 송신
- 200번 = MCI에서 접수확인 수신
- 300번 = 제공기관 → MCI 응답
- 400번 = MCI → 우리 WAS 전달

---

## 브릿지URL 코드 추적 — 진행 중 (학습 목적)

### 배경
- PT에서 대출신청 시 브릿지URL로 이동하는 과정에서 이슈 발생
- 첫 신청하기 클릭 → value값 안 나옴 / 다시 상품보기 → URL인코딩값 나옴 (왔다 안 왔다)
- 팀장이 bridge_common.js 이전 분기 코드 수정 완료 → 내가 직접 추적해서 이해하고 싶음

### 파악된 코드 구조
- **파일**: MT/src/main/webapp/static/service/js/common/fc/bridge_common_ao.js
- **bridge_common.js 분기**:
  - `common_getDeviceOS() == "APP"` → `opa.execute` (네이티브 브릿지, 앱 전용)
  - else → `fg.sf.exec` (SB/웹 방식)
- **bridge_common_ao.js 함수**:
  - `$NH_OPA.OpenApp.openWebNoEnc` (line 89) — 인코딩 없이 외부 사이트 오픈
  - `$NH_OPA.OpenApp.openApp` (line 97) — 앱 링크
- **에러**: STS 콘솔에 `'openWebNoEnc' not found`

### 다음 확인할 것
1. `common_getDeviceOS()` 함수 위치 → PT(PC)에서 뭘 리턴하는지 확인
2. PC인데 APP 분기로 빠지는 건 아닌지 체크
3. MT static의 SB 코드는 지우면 안 됨 — 같은 페이지가 앱/웹 양쪽에서 열리므로 분기 조건이 맞는지가 핵심

### 대출신청접수 응답 구조 (fctcb1300tS)
- `fcmm00101: result: '00000', brpgUrl: 'https://cert-test.goryeob.co.kr/...'`
- 제휴사코드별 brpgUrl 분기: PD003→"40000950", 440→NH생명, 449→NH화재, 011→getNhbUrl
- 당행은 별도 MCI 전문 없이 대출실행내역 테이블 저장 후 brpgUrl만 리턴

### DB 분석 완료
- 송수신내역(TB_EF_FC_CH_CNTGRM_SNR) vs 대출실행관리내역(TB_EF_FC_CH_LN_EXEAMN)
- 같은 CMPR_RQ_SQNO로 조회 시 데이터 차이 → 정상 (모든 송수신 건이 실행관리까지 가지 않음)

---

## 내일(10/28) 할 일

- [ ] 155 DB 캡처 마무리 (Orange에서 필드+데이터 복사 → 메모장 캡처)
- [ ] 156 대출실행내역 수정 여부 양식 작성
- [ ] 브릿지URL 코드 추적 계속 — common_getDeviceOS() 확인
- [ ] 당행 계약대출상품 테스트 (팀장이 CI값 2건 등록해줌, PT/MT 모두 가능)
- [ ] 팀장에게 157~159 보류 처리 보고
- [ ] 임시코드 정리 (scrpgSucYn 강제 "Y", 디버그 로그 제거)
- [ ] 스크래핑 연결 후 4필드 실데이터 재테스트
- [ ] PD코드 ps110 ROWNUM=1 이슈 확인

---

## 내 커뮤니케이션 스타일 & 선호

- 한국어로 대화
- 직설적이고 간결하게. 빈말/칭찬 없이 약점부터 먼저 지적
- 동의하기 전에 반대 의견을 먼저 검토
- 디버깅 가르칠 때: 코드 주석 힌트는 안 읽음 → 실제 에러 재현하고 콘솔 출력 직접 확인시키기
- 질문으로 유도("이 변수에 뭐 들어있어?")하되, 3번 이상 막히면 직접 답 알려주기
- 작업 예시 보여주고 나머지는 직접 하게 하는 패턴이 가장 효과적
- 학은제 과제 대필 시: 마침표/쉼표 없이, "~거나", "~일 수도", "~라고 생각합니다" 등 사람이 쓴 듯한 부드러운 말투

---

## 주요 동료

- **PL (팀장)**: 직접 소스코드로 지도, 디버깅 브레이크포인트 방식 권장
- **강현모**: 카카오톡으로 기술 참고자료(API Gateway, MCI 전문 관련 사이트, 관리페이지 URL 등) 공유
- **황인준 과장**: 스크래핑 진입점 연결 담당
- **황희중 과장**: MCI 전문 DTO 등록(자동생성 파일 관리)
- **수석님**: SmartDev 배포 담당

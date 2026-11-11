# STS 실습 가이드 — 코드 추적 & 커버리지

> 폰으로 이 파일 보면서 STS에서 바로 따라하기
> 각 세션 30~40분 분량. 완료하면 체크(x) 표시

---

## 세션 1: STS 검색 도구 익히기 (기초)

### 1-1. 파일 검색 (Ctrl+Shift+R)
- [ ] `Ctrl+Shift+R` 누르기
- [ ] `fcccb1300` 입력 → 나오는 파일 목록 확인
- [ ] `fctcb1100` 입력 → tC, tS 파일 구분 확인
- [ ] `fcmam0106` 입력 → sC 파일 열기
- [ ] 와일드카드: `*0106*` 입력 → 관련 파일 전부 나오는지 확인

### 1-2. 텍스트 전체 검색 (Ctrl+H)
- [ ] `Ctrl+H` → File Search 탭 선택
- [ ] "jonCls" 검색 → Containing text에 입력
- [ ] File name patterns: `*.java` (자바만)
- [ ] Search 클릭 → 결과에서 **put/set하는 곳** 메모
- [ ] "ln_rq_job_c" 검색 → 같은 방식으로 **세팅하는 곳** 메모
- [ ] "FCT000002" 검색 → 에러코드가 정의된 위치 확인

### 1-3. 현재 파일 내 검색 (Ctrl+F)
- [ ] fcccb1300tS.java 열기
- [ ] `Ctrl+F` → "updtLoanData" 검색 → 메서드 위치로 이동
- [ ] `Ctrl+F` → "reqLoanExec" 검색 → 메서드 위치로 이동
- [ ] `Ctrl+F` → "brpgUrl" 검색 → 어디서 세팅하는지 확인

### 1-4. 메서드 목록 보기 (Ctrl+O)
- [ ] fcccb1300tS.java 열린 상태에서 `Ctrl+O`
- [ ] 나오는 메서드 목록 읽기: 각 이름으로 기능 추측해보기
  - updtLoanData = ?
  - setCoocon106Api = ?
  - reqLoanExec = ?
  - reqLoanCancel = ?
- [ ] fctcb1100tS.java 열고 `Ctrl+O` → 메서드 목록 확인

**세션 1 완료 기준**: 4가지 검색(Ctrl+Shift+R, Ctrl+H, Ctrl+F, Ctrl+O)을 망설임 없이 사용 가능

---

## 세션 2: Call Hierarchy — 호출 체인 따라가기

### 2-1. 기본 사용법
- [ ] fctcb1100tS.java 열기
- [ ] `reqLoanLimitTc` 메서드명 위에 커서 놓기
- [ ] `Ctrl+Alt+H` 누르기 → Call Hierarchy 뷰 열림
- [ ] 결과에서 "이 메서드를 호출하는 곳" 확인
- [ ] 호출자 클릭 → 해당 코드로 이동
- [ ] `Alt+←` → 원래 위치로 돌아오기

### 2-2. MCI 가심사요청 체인 추적
아래 체인을 Call Hierarchy로 직접 따라가며 확인:

```
시작: fctcb1101tS.callBusinessMciEai
  ↑ 누가 호출? → Ctrl+Alt+H
     → fctcb1100tS.reqLoanLimitTc (jonCls="5" 분기 안)
        ↑ 누가 호출?
           → fcccb1100tS.execute()
              ↑ 누가 호출?
                 → fcccb1100tC (CM 컨트롤러)
```

- [ ] fctcb1101tS 열기 → callBusinessMciEai에 `Ctrl+Alt+H`
- [ ] 호출자(fctcb1100tS) 확인 → 클릭해서 이동
- [ ] fctcb1100tS에서 reqLoanLimitTc에 `Ctrl+Alt+H`
- [ ] 호출자(fcccb1100tS) 확인 → 클릭해서 이동
- [ ] fcccb1100tS에서 execute()에 `Ctrl+Alt+H`
- [ ] 최종 호출자(fcccb1100tC) 확인
- [ ] **종이에 체인을 그려보기** (파일명 + 메서드명 + 화살표)

### 2-3. 대출신청접수 체인 추적
```
시작: fctcb1300tS (TI 서비스)
  ↑ 누가 호출?
     → fcccb1300tS (CM 서비스)
        ↑ 누가 호출?
           → fcccb1300tC (CM 컨트롤러)
```

- [ ] fctcb1300tS 열기 → 메인 메서드에 `Ctrl+Alt+H`
- [ ] 체인 끝까지 따라가기
- [ ] 종이에 그려보기

### 2-4. References — 변수 사용처 찾기
- [ ] fctcb1100tS.java 열기
- [ ] `jonCls` 변수 위에 커서 → `Ctrl+Shift+G`
- [ ] 이 변수가 사용되는 모든 곳 결과 확인
- [ ] 결과에서 **값을 세팅하는 곳**(= 왼쪽)과 **읽는 곳**(= 오른쪽) 구분

**세션 2 완료 기준**: Call Hierarchy로 3단계 이상 호출 체인을 혼자 따라갈 수 있음

---

## 세션 3: 컨트롤러 ↔ JSP 연결 찾기

### 3-1. 컨트롤러에서 JSP 찾기
- [ ] fcccb1300tC.java 열기
- [ ] 클래스 상단의 `@RequestMapping` 또는 서비스ID 매핑 확인
- [ ] 확인한 URL/서비스ID 메모 (예: fcccb1300)
- [ ] `Ctrl+H` → 파일 유형 `*.jsp,*.js` → 메모한 값 검색
- [ ] 검색 결과에서 JSP 파일 확인 → 열어보기

### 3-2. JSP에서 컨트롤러 찾기 (역방향)
- [ ] 찾은 JSP 파일 열기
- [ ] `Ctrl+F` → "fg.sf.exec" 검색
- [ ] fg.sf.exec 뒤의 서비스ID 확인 (예: "fcccb1300.reqLoanExec")
- [ ] 서비스ID에서 컨트롤러명 추출 → 해당 컨트롤러 열기
- [ ] 컨트롤러에서 → 서비스 호출 확인 (어떤 sS 메서드를 부르는지)

### 3-3. resultMap 응답 구조 이해
- [ ] 컨트롤러에서 `return resultMap` 하는 메서드 찾기
- [ ] resultMap에 어떤 키/값을 put하는지 확인
- [ ] JSP에서 그 키를 어떻게 받아 쓰는지 확인 (콜백 함수 안)

```
[컨트롤러]                    [JSP]
resultMap.put("brpgUrl", url)  →  function(result) { result.brpgUrl }
resultMap.put("result", "00000")  →  if(result.result == "00000")
```

- [ ] 실제 코드에서 이 패턴 3개 이상 찾아보기

### 3-4. 커버리지 연결
- [ ] 위에서 찾은 JSP가 어떤 화면인지 파악 (타이틀, 화면 구조)
- [ ] 그 화면을 dev에서 열면 어떤 컨트롤러 메서드가 실행되는지 정리
- [ ] **"이 화면 → 이 메서드"** 매핑표 종이에 작성

**세션 3 완료 기준**: JSP ↔ 컨트롤러 연결을 양방향으로 찾을 수 있음

---

## 세션 4: 데이터 흐름 추적 — 값이 어디서 오는가

### 4-1. 파라미터 origin 추적 연습
대상: `fctcb1100tS.java`의 `jonCls`

- [ ] `String jonCls = TIUtil.safeTrim(mapIn, "jonCls")` 찾기
- [ ] 질문: mapIn은 어디서 왔나? → 메서드 파라미터 확인
- [ ] 질문: 이 메서드를 호출하는 곳에서 mapIn에 뭘 넣었나? → Call Hierarchy
- [ ] 호출자로 이동 → mapIn 생성/세팅 코드 찾기
- [ ] **값의 경로를 메모**: [???] → mapIn.put("jonCls", ???) → TIUtil.safeTrim → jonCls

### 4-2. 값의 5가지 출처 구분 연습
fctcb1100tS.java에서 아래 변수들의 출처를 각각 찾아봐:

| 변수 | 출처 찾기 |
|------|----------|
| jonCls | [ ] 세션? 요청? DB? 설정? 외부? |
| aflcnC | [ ] 세션? 요청? DB? 설정? 외부? |
| cusno | [ ] 세션? 요청? DB? 설정? 외부? |

- [ ] 각 변수에 `Ctrl+Shift+G` → 세팅하는 곳 찾기
- [ ] 출처 구분 기준:
  - `session.getAttribute` / `dataMap.getin` → 세션
  - `request.getParameter` / Ajax 파라미터 → 화면 입력
  - DAO 호출 결과 → DB 조회
  - `getEffcCore` / `@Value` → 설정 파일
  - MCI/API 응답 → 외부 시스템

### 4-3. SQL까지 따라가기
- [ ] fctcb1100tS에서 DAO 호출하는 코드 찾기 (예: xxxD.selectXxx)
- [ ] DAO 파일 열기 → 메서드 확인
- [ ] 해당 SQL ID로 `*_sql.xml` 검색 (`Ctrl+H` → `*.xml`)
- [ ] SQL 쿼리 확인 → 어떤 테이블에서 어떤 컬럼을 가져오는지
- [ ] **전체 경로 메모**:
  ```
  [JSP 화면] → [컨트롤러] → [서비스] → [DAO] → [SQL] → [테이블.컬럼]
  ```

### 4-4. jonCls / lnRqJobC 미제 사건 도전
이건 시간 여유 있을 때 도전:

- [ ] `Ctrl+H` → "ln_rq_job_c" → 전체 프로젝트 → 결과 전부 메모
- [ ] 그 중 **= (대입)** 있는 줄만 골라내기 → 최초 세팅 위치
- [ ] `Ctrl+H` → "jonCls" → put 또는 = 있는 줄만 골라내기
- [ ] CM에서 TI로 호출하는 지점(fcccb1100tS → fctcb1100tS) 찾기
- [ ] 그 호출 코드에서 mapIn에 "jonCls" 키로 put하는 곳이 있는지 확인
- [ ] **찾으면 대성공. 못 찾으면 "여기까지 확인했다" 메모**

**세션 4 완료 기준**: 변수의 출처를 5가지 중 하나로 분류할 수 있음

---

## 세션 5: 커버리지 실전 — 빨간 줄 초록으로

### 5-1. 포탈 + STS + dev 3화면 세팅
- [ ] 왼쪽: 커버리지 포탈 (16.24.117.168:28080)
- [ ] 오른쪽: STS
- [ ] 브라우저 탭: dev PT 28800

### 5-2. 파일 선택 → 빨간 메서드 파악
- [ ] 배포리스트에서 네가 수정한 파일 1개 선택
- [ ] 포탈에서 파일 검색 → 커버리지 % 확인
- [ ] 빨간(0%) 메서드 목록 메모
- [ ] 각 메서드 이름으로 기능 추측

### 5-3. 시나리오 역추적
빨간 메서드 1개 골라서:

- [ ] STS에서 해당 메서드 열기
- [ ] `Ctrl+Alt+H` → 호출자 확인
- [ ] 호출자 → 컨트롤러까지 따라가기
- [ ] 컨트롤러 서비스ID → JSP 검색 (`Ctrl+H` → `*.jsp,*.js`)
- [ ] 해당 JSP가 어떤 화면인지 파악

### 5-4. 시나리오 실행 → 결과 확인
- [ ] dev에서 해당 화면 접속
- [ ] 빨간 메서드가 타는 동작 실행 (버튼 클릭, 화면 진입 등)
- [ ] 포탈에서 재확인 → 빨간 줄 초록 전환 확인
- [ ] 성공하면 다음 빨간 메서드로 반복

### 5-5. 못 태우는 건 분류
태울 수 없는 메서드는 사유 정리:

| 분류 | 대응 |
|------|------|
| 역거래/외부 의존 | PL에게 보고 |
| 테스트 데이터 없음 | 사유 메모 |
| 환경 제약 (MT 전용) | 폰으로 시도 |
| 인증 문제 | 다른 아이디 시도 |

**세션 5 완료 기준**: "빨간 메서드 → 시나리오 찾기 → 실행 → 초록" 사이클을 혼자 완주

---

## 세션 6: 환경/설정 파일 이해

### 6-1. build.gradle 읽기
- [ ] `Ctrl+Shift+R` → `build.gradle` 열기
- [ ] `dependencies` 블록 찾기 → 어떤 라이브러리 쓰는지 확인
- [ ] `implementation`, `compileOnly`, `testImplementation` 차이 이해
  - implementation = 실행 시 필요
  - compileOnly = 컴파일만 필요 (Lombok 등)
  - testImplementation = 테스트에서만 필요

### 6-2. application.yml 구조 파악
- [ ] `application.yml` 열기
- [ ] `spring.profiles` 확인 → 어떤 프로파일들이 있는지
- [ ] DB 연결 정보 구조 확인 (datasource)
- [ ] `application-dev.yml` vs `application-local.yml` 비교

### 6-3. common_ti_config.yml 확인
- [ ] `common_ti_config.yml` 열기
- [ ] 네가 추가한 `aflcnC: "fco0001": PD301` 매핑 찾기
- [ ] 코드에서 이 값을 읽는 곳 찾기: `Ctrl+H` → "getEffcCore"
- [ ] **설정값 → 코드 사용처** 연결 이해

### 6-4. WebSecurityConfig 읽기
- [ ] `WebSecurityConfig.java` 열기
- [ ] `permitAll()` 있는 URL 패턴 확인 → 인증 없이 접근 가능한 경로
- [ ] `authenticated()` 패턴 확인 → 로그인 필요한 경로
- [ ] 필터 체인 구조 파악 (어떤 순서로 체크하는지)

**세션 6 완료 기준**: 설정 파일이 코드에서 어떻게 사용되는지 연결할 수 있음

---

## 세션 7: JSP→JS 변수 전달 & 환경별 URL 분기 (팀장 다운로드 코드)

> 실제 커밋된 코드: fcmam0100r.jsp (메인 화면) mvDownLoad 함수
> 패턴: 서버 설정값 → JSP EL → JS 변수 → OS/앱타입 분기 → 외부 URL 호출

### 7-1. 전체 구조 파악

이 코드의 흐름:
```
[yml 설정파일]                     [JSP (서버사이드)]              [JS (클라이언트)]
MAIN_ANDROID_A=https://...  →  SpringContext.getProperty  →  ${MAIN_ANDROID_A}  →  mvDownLoad('A')
MAIN_IOS_S=https://...      →  <c:set var=.../>           →  ${MAIN_IOS_S}      →  mvDownLoad('S')
```

- [ ] fcmam0100r.jsp 열기 (`Ctrl+Shift+R`)
- [ ] `Ctrl+F` → "mvDownLoad" 검색 → 함수 위치 확인
- [ ] `Ctrl+F` → "MAIN_ANDROID" 검색 → JSP 상단의 `<c:set>` 태그 위치 확인
- [ ] 질문: `<c:set var="devOs" value="${RESULT_MAP.get('devOs')}"/>` — devOs는 어디서 오는가?

### 7-2. JSP 변수 선언부 분석

```jsp
<%-- 서버 설정에서 URL 가져오기 --%>
<c:set var="devOs" value="${RESULT_MAP.get('devOs')}"/>
<c:set var="MAIN_ANDROID_A" value="${SpringContext.getProperty('MAIN_ANDROID_A')}"/>
<c:set var="MAIN_ANDROID_S" value="${SpringContext.getProperty('MAIN_ANDROID_S')}"/>
<c:set var="MAIN_IOS_A" value="${SpringContext.getProperty('MAIN_IOS_A')}"/>
<c:set var="MAIN_IOS_S" value="${SpringContext.getProperty('MAIN_IOS_S')}"/>
```

- [ ] 각 변수의 출처 구분:
  - `devOs` → RESULT_MAP = 컨트롤러가 put한 값 (고객 디바이스 OS)
  - `MAIN_ANDROID_A` → SpringContext.getProperty = **yml 설정파일** 값
- [ ] `Ctrl+H` → "MAIN_ANDROID_A" → `*.yml,*.properties` → 어떤 설정파일에 정의됐는지 찾기
- [ ] 환경별(local/dev/run) 값이 다른지 확인 → **같은 키, 다른 값** 패턴 이해

### 7-3. HTML 버튼 ↔ JS 함수 연결

```html
<button class="btn-app-download ico-smart"
        aria-label="NH스마트뱅킹 다운로드"
        onclick="mvDownLoad('S')">다운로드</button>

<button class="btn-app-download ico-allone"
        aria-label="NHallone 다운로드"
        onclick="mvDownLoad('A')">다운로드</button>
```

- [ ] 'S' = 스마트뱅킹, 'A' = 올원뱅크 — appGbn 파라미터로 구분
- [ ] aria-label이 각 버튼마다 다른 이유 → 접근성 (스크린리더 구분용)
- [ ] onclick에서 JS 함수 직접 호출 — fg.sf.exec 패턴 아님 (외부 URL 단순 이동)

### 7-4. JS 함수 — OS + 앱타입 이중 분기

```javascript
function mvDownLoad(appGbn){
    var devOs = '${devOs}';  // ← JSP EL이 서버에서 치환됨
    if(devOs == "Android"){
        if(appGbn == "A"){ openWebPage('${MAIN_ANDROID_A}',function(ret){}); }
        else if(appGbn == "S"){ openWebPage('${MAIN_ANDROID_S}',function(ret){}); }
    } else if(devOs == "iOS"){
        if(appGbn == "A"){ openWebPage('${MAIN_IOS_A}',function(ret){}); }
        else if(appGbn == "S"){ openWebPage('${MAIN_IOS_S}',function(ret){}); }
    }
}
```

- [ ] `var devOs = '${devOs}'` — 이 줄이 **서버에서** 실행되면 `var devOs = 'Android'`로 치환됨
- [ ] 브라우저 소스보기(F12)에서 확인하면 이미 값이 들어가 있음
- [ ] `openWebPage` — 농협 프레임워크 함수, 외부 URL을 여는 공통 함수
- [ ] 질문: PC(PT)에서 이 함수 실행하면? → devOs가 Android/iOS 아니면 아무 동작 안 함

### 7-5. 핵심 패턴 정리 & 응용 연습

**패턴**: yml → SpringContext.getProperty → JSP `<c:set>` → JS `'${변수}'` → 분기 로직

- [ ] 이 패턴을 쓰는 다른 코드 찾기: `Ctrl+H` → "SpringContext.getProperty" → `*.jsp`
- [ ] getNhbUrl("03","11")과 비교: 같은 목적(환경별 URL), 다른 방식(JS 내 호출 vs JSP EL)
- [ ] ti_config.yml의 URL 매핑과 비교: `J011.URL.03.11` = getNhbUrl이 읽는 키

**쓰임 비교표**:
| 방식 | 코드 | 언제 쓰나 |
|------|------|----------|
| SpringContext.getProperty | `${SpringContext.getProperty('KEY')}` | 단순 값 1개 (URL, 플래그) |
| getNhbUrl | `getNhbUrl("03","11")` | TI config yml 계층 구조에서 꺼낼 때 |
| getEffcCore | `TIUtil.getEffcCore("aflcnC", value)` | TI 서비스 안에서 yml 매핑 읽을 때 |

- [ ] 위 3가지 방식 각각 실제 코드에서 1개씩 찾아보기

**세션 7 완료 기준**: JSP EL로 서버 값이 JS에 전달되는 구조를 설명할 수 있고, 환경별 URL 분기 패턴을 이해함

---

## 세션 8: MCI 전문 필드 세팅 경로 추적 & 접근성 ARIA 속성

### 8-1. MCI 전문의 두 가지 세팅 경로

MCI 전문은 **바디(setMciBody)**와 **공통헤더(setApiBody)** 두 곳에서 필드가 세팅됨.
같은 데이터(예: 주민번호)도 필드명이 다르게 들어감:

| 구분 | 메서드 | 파일 | 주민번호 필드명 |
|------|--------|------|---------------|
| 바디 | setMciBody | fctcb1101tS.java | pwize_tr_rlno (encrytedRlno) |
| 공통헤더 | setApiBody | fctcb1101tS.java | cusPwizeRlno (map101의 "RRNo") |

- [ ] fctcb1101tS.java에서 `setMciBody` 메서드 찾기 → 어떤 필드를 세팅하는지 목록화
- [ ] 같은 파일에서 `setApiBody` 메서드 찾기 → 어떤 필드를 세팅하는지 목록화
- [ ] 두 메서드가 겹치는 데이터(주민번호, 고객명 등)를 확인

**핵심**: 하나만 수정하면 다른 쪽이 덮어쓸 수 있음. 디버깅 시 콘솔에 찍히는 필드명이 어느 경로인지 먼저 확인

### 8-2. SQL에서 NULL 분기 처리 (NVL, CASE WHEN)

스크래핑 성공/실패를 Java if/else가 아닌 **SQL에서 처리**하는 패턴:

```sql
-- LEFT OUTER JOIN: 홈텍스 데이터 없으면 H 컬럼 전부 NULL
FROM 비교신청관리 R
LEFT OUTER JOIN 홈텍스사업자 H ON R.키 = H.키

-- NVL: NULL이면 기본값
NVL(H.COMM_BZMN_YN, 'N')  -- 스크래핑 실패 시 'N'

-- CASE WHEN: 조건별 치환
CASE WHEN H.NTS_BZCC IS NOT NULL 
     THEN H.NTS_BZCC              -- 스크래핑 성공: 원본값
     ELSE CASE R.BZMN_BZCC        -- 스크래핑 실패: 직접입력값 치환
          WHEN '1' THEN '369908'
          WHEN '2' THEN '523999'
          END
END
```

- [ ] fctcb01_sql.xml에서 ps111 쿼리 찾기 → LEFT OUTER JOIN 구조 확인
- [ ] NVL과 CASE WHEN이 각각 어떤 상황에 적합한지 구분

**핵심**: Java 분기 vs SQL 분기 — 쿼리 하나로 해결 가능하면 SQL이 깔끔. Java if/else 구조를 안 건드려도 됨

### 8-3. 웹 접근성 ARIA 속성 정리

| 속성 | 역할 | 사용법 |
|------|------|--------|
| aria-label | 요소에 직접 텍스트 삽입 | `aria-label="개인정보 동의 펼치기"` |
| aria-describedby | 다른 요소의 id 참조 | `aria-describedby="agChk3"` → id="agChk3"인 요소 텍스트를 읽음 |
| aria-expanded | 접힘/펼침 상태 | `aria-expanded="false"` → 스크린리더가 "접힘" 읽음 |
| role="button" | 비버튼 요소를 버튼으로 | `<div role="button">` → 스크린리더가 "버튼"으로 인식 |

- [ ] DevTools → Accessibility 탭에서 요소 선택 → ARIA 속성 확인해보기
- [ ] `<span class="blind">숨긴 텍스트</span>` → 화면에 안 보이지만 스크린리더가 읽음

**함수 간 변수 스코프 주의**:
```javascript
// agInputDom 함수 안
function agInputDom(ag) {
    var sqno = ag.STLT_EXPS_SQNO;  // 이 함수 안에서만 유효
    // id="agChk" + sqno
}

// renderDivFormAccordionDom 함수 안
function renderDivFormAccordionDom(ag) {
    // sqno 쓰면 → undefined 에러 → 렌더링 전체 멈춤!
    // ag.STLT_EXPS_SQNO 써야 함
}
```

- [ ] JSP에서 동적 HTML 생성하는 함수 찾아서, 다른 함수의 변수를 참조하고 있지 않은지 확인

**세션 8 완료 기준**: MCI 전문의 바디/헤더 이중 세팅 구조를 이해하고, SQL NULL 분기와 ARIA 속성 사용법을 설명할 수 있음

---

## 진행 체크리스트

| 세션 | 주제 | 상태 | 완료일 |
|------|------|------|--------|
| 1 | STS 검색 도구 | [ ] | |
| 2 | Call Hierarchy | [ ] | |
| 3 | 컨트롤러↔JSP | [ ] | |
| 4 | 데이터 흐름 추적 | [ ] | |
| 5 | 커버리지 실전 | [ ] | |
| 6 | 환경/설정 파일 | [ ] | |
| 7 | JSP→JS 변수 전달 & URL 분기 | [ ] | |
| 8 | MCI 필드 세팅 경로 & ARIA 속성 | [ ] | |

---

## 메모 공간

### 세션 1 메모
```
(검색 결과, 발견한 것 여기에 적기)


```

### 세션 2 메모
```
(호출 체인 그림 여기에 적기)


```

### 세션 3 메모
```
(JSP↔컨트롤러 매핑표 여기에 적기)


```

### 세션 4 메모
```
(변수 출처 추적 결과 여기에 적기)
jonCls 세팅 위치:
lnRqJobC 세팅 위치:
변환 지점:

```

### 세션 5 메모
```
(커버리지 결과 여기에 적기)
파일명 | 메서드 | 전 | 후 | 사유
-------|--------|----|----|-----


```

### 세션 6 메모
```
(설정 파일에서 발견한 것 여기에 적기)


```

### 세션 7 메모
```
(mvDownLoad 관련 발견한 것 여기에 적기)
SpringContext.getProperty 키 위치:
devOs 출처:
openWebPage vs fg.sf.exec 차이:

```

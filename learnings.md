# Learnings — 프로젝트 범용 패턴 & 교훈

> 프로젝트 간 재사용할 수 있는 코드 패턴, 디버깅 기법, 실수 방지 규칙 모음.
> tasks.md에서 추출 + 실무에서 체득한 것 정리.

---

## 1. Java / Spring 패턴

### 1-1. for문 안 컬렉션 초기화 — `.clear()` vs `new ArrayList<>()`
- **문제**: `.clear()`로 리스트를 비우면, 이전에 `map.put("key", list)` 한 참조까지 같이 비워짐
- **해결**: `list = new ArrayList<>()` 로 새 인스턴스 생성 — 이전 참조는 그대로 유지
- **규칙**: map/결과객체에 put한 리스트는 반드시 `new`로 재생성. `.clear()` 금지
```java
// BAD — map101에 넣은 GdLIST까지 비워짐
GdLIST.clear();

// GOOD — 새 인스턴스라 map101의 이전 값은 안전
GdLIST = new ArrayList<>();
```

### 1-2. for문 밖 변수 선언 → 루프 간 데이터 누적 버그
- **증상**: 제휴사 A 처리 후 B 처리할 때 A 데이터가 남아있음
- **원인**: Set, List 등이 for문 밖에 선언되어 반복마다 초기화 안 됨
- **규칙**: for문 안에서만 쓰는 변수는 for문 안에서 선언/초기화
```java
for (Map<String, Object> fis : fisCodeList) {
    Set<String> exist = new HashSet<>();     // 매 반복 새로 생성
    List<Map> inList2 = new ArrayList<>();   // 이전 제휴사 잔류 방지
    // ...
}
```

### 1-3. 컨트롤러 라우팅 중복 방지
- **상황**: `@McuCallID` 등으로 컨트롤러에서 이미 라우팅하는 구조
- **실수**: 서비스 레이어에 `isBiz` 같은 중복 분기 플래그 추가
- **규칙**: 기존 requestMap의 값(jonCls 등)을 활용하고, 컨트롤러에서 이미 분리됐으면 서비스에 중복 플래그 넣지 말 것

### 1-4. DB 의존 vs 설정 파일(yml) 직접 매핑
- **상황**: 쿼리 ROWNUM=1로 원하지 않는 값이 먼저 잡힘
- **해결**: `TIUtil.getEffcCore("aflcnC", aflcnC)` → yml에서 직접 매핑
- **규칙**: 값이 고정(상품코드 등)이면 DB 쿼리 결과보다 yml 설정이 안전하고 명확

### 1-5. 하나의 UPDATE SQL을 여러 서비스가 공유하는 구조
- **패턴**: `<if test>` 동적 SQL로 호출자마다 다른 컬럼만 UPDATE
- **주의**: 한 서비스에서 세팅한 값이 다른 서비스 호출 시 의도치 않게 덮일 수 있음
- **디버깅**: 특정 컬럼 값이 이상하면 해당 UPDATE를 호출하는 모든 서비스를 추적

### 1-6. MCI 전문 — setMciBody(바디) vs setApiBody(헤더) 주의
- **문제**: 같은 필드가 body와 header 양쪽에서 세팅될 수 있음
- **교훈**: 하나만 수정하면 다른 쪽이 덮어씀. 전문 필드 수정 시 양쪽 모두 확인

### 1-7. 세션 기반 파라미터 변조 방지 (보안)
- **공격**: Burp Suite로 응답 가로채서 cusno/생년월일/전화번호 변조
- **방어**: 서버에서 프론트 파라미터를 세션 값과 비교 검증
```java
private boolean checkCustInfo(DataMap dataMap) {
    // 세션에서 고객정보 가져오기
    // 프론트 파라미터와 세션 값 비교
    // 불일치 시 false → 요청 거부
}
```
- **원칙**: 클라이언트에서 올라온 본인정보를 서버가 그대로 신뢰하지 말 것

### 1-8. 주민번호 세기 판별 로직
```java
// 7번째 자리로 세기 판별
char c = rlno.charAt(6);
switch(c) {
    case '1': case '2': case '5': case '6': prefix = "19"; break;
    case '3': case '4': case '7': case '8': prefix = "20"; break;
    case '9': case '0': prefix = "18"; break;
}
String birthDay = prefix + rlno.substring(0, 6); // YYYYMMDD
```

---

## 2. JavaScript / JSP 패턴

### 2-1. `document.title` 접근성 타이틀 적용 규칙
- **일반 페이지**: `document.title = "페이지명 | 서비스명";`
- **레이어팝업(모달)**: 열기 직전 변경, 닫힐 때 원복
```javascript
var originalTitle = document.title;
document.title = "설문조사 | NH금융상품비교";
fcPop.modalOpen('POPUP_ID', param, function() {
    document.title = originalTitle;
    // 기존 콜백 로직
}, this);
```
- **alert성 팝업**: `modalAlertOpen` 첫 번째 파라미터에 넣지 않음 (그건 팝업 헤더용)

### 2-2. JSP 조건별 `document.title` 분기
- **상황**: 한 JSP에서 `<c:if>`로 여러 화면을 관리
- **방법**: JS 삼항연산자로 분기
```javascript
var lnRqJobC = "${lnRqJobC}";
document.title = (lnRqJobC === "5")
    ? "국세청로딩 | NH금융상품비교"
    : "공공마이데이터로딩 | NH금융상품비교";
```

### 2-3. 페이지 타이틀 — 컨트롤러 vs JS 방식
- **컨트롤러가 있는 페이지**: `resultMap.getOut().put("pageTitle", "페이지명")` → 템플릿 `<title>` 반영
- **컨트롤러 없는 페이지** (denyMap/defaultPageViewController): JSP에서 JS로 직접 세팅
```html
<script>document.title = "페이지명 서비스명";</script>
```
- **주의**: 템플릿에서 이미 서비스명 붙이면 컨트롤러 방식에서는 중복 금지

### 2-4. 팝업 포커스 이동 — 접근성 필수 패턴
```javascript
fcPop.modalOpen('POPUP_ID', {}, function() {
    window.setTimeout(function() {
        var el = document.querySelector('#POPUP_ID .modal-window');
        if (el) {
            el.setAttribute('tabindex', '-1');  // 선행 필수
            el.focus();
        }
    }, 100);
}, this);
```
- **tabindex="-1"** 없으면 `focus()` 해도 포커스 안 잡힘
- **setTimeout**: 모달 DOM 렌더링 완료 대기 (100~300ms)

### 2-5. CSS 셀렉터 고유성 확보 방법
1. `document.querySelector('셀렉터')` → 요소 리턴 확인
2. `document.querySelectorAll('셀렉터').length` → 1이어야 안전
3. 2개 이상이면 부모 ID/클래스 조합 (예: `#POPUP_ID .modal-window`)
4. `.slide .modal-window` 같은 구조 의존 셀렉터는 HTML 변경에 취약 → `#ID .class` 권장

### 2-6. `function() {}` vs `() => {}` 절대 섞지 말 것
- **실수**: `function() => {}` → SyntaxError → 파일 전체 파싱 실패 → 페이지 멈춤
- **규칙**: 한 프로젝트 내에서 콜백 스타일 통일. 기존 코드가 `function` 쓰면 맞추기

### 2-7. jQuery `.after()` vs `.html()` 차이
- `.after(html)` = 형제 요소로 바깥에 삽입
- `.html(html)` = 자식으로 내부 교체 (기존 내용 대체)
- **셀렉터 주의**: 같은 클래스가 여러 개면 ID 셀렉터 필요

### 2-8. aria-describedby vs aria-labelledby
- `aria-describedby` = 부가 설명 (스크린리더가 나중에 읽음)
- `aria-labelledby` = 이름/제목 (스크린리더가 먼저 읽음)
- **팝업 제목은 `aria-labelledby`가 맞음**

### 2-9. 동적 HTML에서 XSS 방지
- 함수 파라미터를 aria-label 등에 삽입할 때 반드시 `escHtml()` 처리
```javascript
'aria-label="' + escHtml(ag.TINM) + ' 펼치기"'
```

### 2-10. title 속성 vs JS 셀렉터 — 수정 안전성 판단
- `.attr("title", "새값")` = 값 세팅(set)
- `$("[title='특정값']")` = 셀렉팅(select)
- **셀렉팅에 사용되지 않는 title이면 값 변경 안전**

---

## 3. SQL 패턴

### 3-1. NOT EXISTS — 중복 방지용
```sql
-- "정식출시 상품은 이미 메인 쿼리에서 조회됐으니 CBT쪽에서 제외"
AND NOT EXISTS (
    SELECT 1 FROM 메인테이블 M
    WHERE M.상품코드 = CBT.상품코드
    AND M.거래제한여부 = 'N'
)
```
- **용도**: 두 결과셋을 addAll할 때 중복 제거

### 3-2. SQL alias로 변수명 변환
- Java 변환 없이 SQL `AS "JonCls"` 로 대문자/소문자 그대로 전달 가능
- **디버깅 팁**: 변수명이 갑자기 바뀌면 SQL alias 확인

### 3-3. NVL + CASE WHEN 조합 — 외부 데이터 fallback
```sql
CASE WHEN H.NTS_BZCC IS NOT NULL THEN H.NTS_BZCC
     ELSE CASE R.BZMN_BZCC
         WHEN '1' THEN '369908'
         WHEN '2' THEN '523999'
     END
END
```
- 스크래핑 성공 → 실제값, 실패 → 직접입력 매핑값

### 3-4. Oracle AutoCommit OFF 주의
- 반드시 수동 COMMIT
- 테스트용 DB UPDATE 후 반드시 원복 + COMMIT

---

## 4. 디버깅 기법

### 4-1. 데이터 흐름 추적 방법론
1. **변수명으로 전체 검색** (Ctrl+H): 세팅하는 곳(put/set) 전부 목록화
2. **Call Hierarchy** (Ctrl+Alt+H): 이 메서드를 호출하는 모든 곳
3. **References** (Ctrl+Shift+G): 이 메서드/변수가 참조되는 곳
4. **브레이크포인트**: 값이 바뀌는 순간 포착
5. **`mapIn.keySet()` 출력**: 넘어오는 키 목록 확인

### 4-2. "이 메서드를 태우려면 어떤 시나리오?" 역추적
1. 커버리지 도구에서 빨간(미실행) 메서드 확인
2. 해당 메서드의 if 분기 조건 파악
3. 그 조건을 만족하는 UI 시나리오 찾기
4. dev 서버에서 시나리오 실행 → 커버리지 확인

### 4-3. 로컬/더미로그인 한계 인식
- 더미로그인 = 실제 고객정보 조회 안 거침 → 직업코드, 고객번호 기반 데이터 없음
- **결론**: 특정 분기를 타야 하는 테스트는 dev/검증서버 + 실 인증으로

### 4-4. MCI 로그 번호 의미
| 번호 | 의미 | 증빙 용도 |
|------|------|-----------|
| 100 | WAS → MCI 송신 | 송신 전문 확인 |
| 200 | MCI 접수확인 수신 | 접수 정상 확인 |
| 300 | 제공기관 → MCI 응답 | 제휴사 응답 확인 |
| 400 | MCI → WAS 전달 | 결과 수신 확인 |

### 4-5. 네이티브 SDK 함수 판별
- JS에서 함수 호출부는 50건 있는데 **정의부가 없으면** → 네이티브 SDK 주입 함수
- JS 소스만으로는 추적 불가 → "우리 소관 아님" 판단 근거

---

## 5. Git / 버전 관리

### 5-1. 커밋 전 스테이지 목록 반드시 확인
- `.gitignore`, `local.yml` 등 의도하지 않은 파일이 올라가지 않았는지 체크
- **실수 사례**: JSP 커밋할 때 .gitignore가 같이 스테이지됨

### 5-2. EFFCCOM 등 공통 모듈 — 충돌 시 항상 원격 기준
- 내 코드가 아닌 모듈 충돌 시 → "상대 것으로 해결" (원격 기준)
- 절대 커밋하지 않음

### 5-3. SourceTree 충돌 해결
- "내것으로 변경" = HEAD만 유지 (상대 코드 전부 삭제) → 위험
- **안전한 방법**: 충돌 부분 수동 편집 → "충돌 해결됨으로 표시" → 스테이지 → 커밋

### 5-4. merge conflict 해결 원칙
- 내 작업(pageTitle 등)과 상대 작업(EnvUtil.isCbt 등) 양쪽 모두 필요한 부분 취합
- 한쪽만 선택하면 기능 누락 위험

---

## 6. VS Code 활용

### 6-1. 전체 치환 시 라이브러리 제외
- Ctrl+Shift+H → files to include: `*.jsp`, files to exclude: `*.min.js, *.html`
- **안 하면**: jQuery, DataTables 등 라이브러리까지 치환됨
- **검증**: 커스텀 JS에 해당 속성 사용 여부 별도 확인

### 6-2. 프로젝트 열기
- File → Open Folder → 프로젝트 루트
- 멀티 프로젝트: File → Add Folder to Workspace

---

## 7. 테스트 / 증빙

### 7-1. IQMS 증빙 양식 패턴
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

### 7-2. 커버리지 — View vs ViewMo
- View = PC(PT), ViewMo = 모바일(MT)
- PT에서 실행하면 View만 타고, MT에서 실행하면 ViewMo만 탐
- **에러도 커버리지**: catch/throw 실행되면 해당 라인 초록 전환

### 7-3. 더미 아이디 주의
- 아이디에 따라 PETRA 복호화 에러(FCT000002) 발생 가능 → 되는 아이디로 작업

---

## 8. 커뮤니케이션 / 프로세스

### 8-1. "우리 팀 소관인지" 판단 기준
- 추적 끝에 네이티브 SDK / 다른 팀 설정이면 → PL에게 보고 + "우리 소관 아님" 확인
- 증거: JS에서 추적 불가 지점 도달, 설정 파일이 우리 저장소에 없음

### 8-2. PL 보고 시 증거 구조
1. 무엇을 확인했는지 (코드 경로/DB 조회 결과)
2. 어디가 문제인지 (어떤 지점에서 끊기는지)
3. 왜 우리 소관이 아닌지 (또는 어떻게 고칠지)

### 8-3. 테스트용 DB 수정 → 반드시 원복
- 테스트 위해 rztCode 등을 UPDATE할 수 있지만
- 테스트 완료 후 반드시 원래 값으로 UPDATE + COMMIT
- 기록: tasks.md에 "DB 원복 완료" 명시

---

## 9. 보안 체크리스트

### 9-1. 커밋 전 금지 항목
- `|| true` 조건 우회
- 하드코딩된 고객번호(cusno)
- 임시 디버그 로그 (`System.out.println`, `console.log`, `[CBT]`)
- 테스트용 하드코딩 (scrpgSucYn="Y" 등)
- .rej 파일, local.yml
- 자동생성 DTO 파일 수정

### 9-2. 응답에 평문 주민번호 노출 방지
- JSP에 `${rlno}` 평문 내려보내지 말 것
- 서버에서 세션으로 직접 가져와 처리 (근본 해결)
- 불가피하면 `NHBFormatUtil.maskWithPattern("rlno", "??????-?******")` 마스킹

### 9-3. Rate Limit — 서버 사이드 필수
- 프론트 `isRequesting` 플래그는 Burp Suite로 우회 가능
- 세션 기반 카운터: `SessionManagerFC.putTempValue/getTempValue`

---

## 10. 접근성 (웹 접근성 심사)

### 10-1. 수정 원칙
1. 진단결과서에 지적된 항목만 수정, 범위 밖 금지
2. 공통 함수(nhui.toggleSlide 등) 건드리지 않기
3. CSS 수정은 퍼블리셔 담당으로 패스

### 10-2. `<span class="blind">` = 시각적으로 숨기되 스크린리더가 읽는 공통 CSS
- 보이지 않는 텍스트를 추가할 때 사용
- `aria-label`과 중복 읽기 가능성 있으므로 둘 다 적용은 퍼블 지시에 따름

### 10-3. JSP HTML 수정이 반영 안 될 때
- JS `setAttribute('tabindex', '-1')` 로 동적 처리 가능 (서버 재기동 불필요)

### 10-4. 선형화 순서 변경
- DOM 순서 swap + CSS `flex-direction: row-reverse`로 시각적 순서 유지
- 스크린리더는 DOM 순서대로 읽으므로 DOM 순서가 중요

---

## 11. 환경 / 인프라

### 11-1. 서버 환경별 차이
| 환경 | 특징 | 용도 |
|------|------|------|
| local | PETRA 미동작(평문), 더미로그인 | 단위 개발/디버깅 |
| dev | PETRA 자동동작, 실 인증 가능 | 기능 테스트 |
| smartdev(검증) | 통합테스트 환경 | 통합테스트(통테) |
| predev | 스마트데브 | 모바일 실기기 테스트 |
| run/dr | 운영/재해복구 | 실서비스 |

### 11-2. MT vs PT 차이
| 구분 | PT (앱) | MT (모바일웹) |
|------|---------|---------------|
| 접근 | 올원뱅크/스마트뱅크 앱 | 모바일 브라우저 |
| 네이티브 | opa.execute, gotoNhrc 등 | 미지원 (openWebPage) |
| 채널코드 | appGbn 분기 (S→01, A→02) | 하드코딩 "05" |
| PC 테스트 | 28800 포트 가능 | 불가 (앱 전용) |

### 11-3. Zero-padding 주의
- DB에 "05"로 저장 → `"5".equals("05")`는 **false**
- 반드시 DB 저장 형식 그대로 비교 (`"05"`)

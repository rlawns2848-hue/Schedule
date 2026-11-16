# AIRLINK — 냉·난방 AS기사 스케줄 관리 시스템

고객이 접수한 냉·난방 AS 건을 **관리자가 기사에게 배정하고, 기사가 결과를 보고하고, 고객이 만족도를 남기는** 순환 구조의 웹 서비스입니다.
저는 그중 **관리자 일정 관리 — 캘린더 조회와 기사 배정** 을 맡았습니다.

> 4인 팀 프로젝트 **Hello World** (2026.09.08 ~ 09.17 · 기능 구현은 09.15까지, 이후 발표 준비) · 그린AI캠퍼스 평생교육원
> 이 저장소는 팀 저장소 [`ung-Jin/Schedule`](https://github.com/ung-Jin/Schedule) 의 fork 입니다.

---

## 담당 범위

**`schedule` 패키지와 관리자 일정관리 화면 전체를 맡았습니다.**

📁 **[`src/main/java/com/green/Schedule/schedule`](https://github.com/rlawns2848-hue/Schedule/tree/dev/src/main/java/com/green/Schedule/schedule)**

| 파일 | 하는 일 |
|---|---|
| `controller/ScheduleController.java` | 일정관리 페이지 렌더링 · 배정 폼 처리 |
| `service/ScheduleService.java` | 배정 저장 + 접수 상태 변경을 한 흐름으로 |
| `mapper/ScheduleMapper.java` | DB 접근 인터페이스 |
| `dto/ScheduleDTO.java` | 스케줄 한 건 (기사·접수 정보를 중첩으로 포함) |
| `resources/mapper/schedule-mapper.xml` | 조회·배정 쿼리와 resultMap |
| `templates/pages/schedule/admin_schedule.html` | 좌 캘린더 · 우 배정 폼 |
| `static/js/pages/schedule/admin_schedule.js` | FullCalendar 연동 · 기사 필터 · 중복 배정 검증 |
| `static/css/pages/schedule/admin_schedule.css` | 상태별 색상 · 범례 · 카드 레이아웃 |

🔗 **[팀 저장소에서 제 커밋만 보기](https://github.com/ung-Jin/Schedule/commits/dev?author=rlawns2848-hue)**

---

## 주요 기능

### 1. 일정 캘린더

접수 건과 배정된 일정을 **한 캘린더에 같이** 띄웁니다. 기준 날짜가 서로 다릅니다.

```
AS_REQUEST    희망일(wishDate)  기준  —  아직 배정 안 된 접수
AS_SCHEDULE   확정일(fixDate)   기준  —  기사가 배정된 일정
```

네 가지 상태를 배경색으로 구분하고 하단에 범례를 뒀습니다. 색만 봐도 일정 상태가 구분됩니다.

```
접수완료 RECEIVED  ·  기사배정 ASSIGNED  ·  작업진행 IN_PROGRESS  ·  작업완료 COMPLETED
```

FullCalendar 의 기본 상단 UI(`headerToolbar`)는 끄고 이전/다음·일/주/월 버튼을 직접 만들어 프로젝트 톤에 맞췄습니다. 기사 필터는 서버를 다시 부르지 않고 **이미 받아 둔 목록을 그 자리에서 다시 거릅니다.**

### 2. 기사 배정

접수 건·담당 기사·방문 날짜·시작/종료 시간을 받아 배정합니다. 날짜와 시간은 따로 입력받되 **전송 직전에 하나로 합쳐** 서버가 바로 파싱할 수 있는 형태로 넘깁니다.

배정을 저장하면서 **같은 흐름 안에서 해당 접수 건의 상태를 `ASSIGNED` 로 함께 바꿉니다.**

### 3. 중복 배정 검증

같은 기사에게 겹치는 시간대가 두 번 잡히지 않도록 막습니다. 구간 겹침은 한 줄로 판별했습니다.

```javascript
// 내 시작시간이 상대 종료시간보다 빠르고, 상대 시작시간이 내 종료시간보다 빠르면 겹친다
return startTime < existingEnd && existingStart < endTime;
```

필수 항목 누락, 종료시간이 시작시간보다 빠른 경우를 먼저 거르고, 겹침이 확인되면 **기사명과 충돌한 시간대를 에러 메시지에 같이 띄워** 원인을 바로 알 수 있게 했습니다.

---

## 설계에서 신경 쓴 것

- **4계층으로 나눴습니다.** `Controller → Service → Mapper(interface) → XML(SQL)`. Controller 는 흐름만 정하고, Service 가 "배정 저장 + 접수 상태 변경" 같은 여러 단계를 조합하고, Mapper 와 XML 이 DB 접근을 맡습니다. 한 계층이 바뀌어도 다른 계층에 영향이 덜 가도록 한 것입니다.
- **캘린더는 라이브러리를 썼습니다.** 월/주/일 전환과 날짜 계산을 직접 짜면 시간이 오래 걸리고 버그가 날 위험이 컸습니다. 검증된 FullCalendar 6.1.11 (CDN) 을 쓰고, 꾸미는 부분만 직접 했습니다.
- **화면 먼저, 데이터는 나중에.** DB 설계가 확정되기 전에 더미 데이터로 로직을 짜두면 실제 응답 구조가 달라질 때 다시 갈아엎어야 합니다. 골격과 `TODO` 주석만 남기고 설계가 끝난 뒤 연결했습니다.

---

## 트러블슈팅 — 상태값이 세 군데에 똑같이 들어갔다

일정 목록을 조회하면 **기사 근무상태와 접수 상태 자리에 엉뚱한 값**이 들어와 있었습니다. 둘 다 스케줄 상태와 같은 값이었습니다.

세 테이블이 모두 `STATUS` 컬럼을 갖고 있는 게 바탕이었습니다.

```
ENGINEER.STATUS      WORK / REST
REQUEST.STATUS       RECEIVED / ASSIGNED / IN_PROGRESS / COMPLETED
AS_SCHEDULE.STATUS   ASSIGNED
```

쿼리는 `S.STATUS AS STATUS` 로 **스케줄 상태 하나만** 내려주는데, 기사·접수 정보를 같이 담으려고 `association` 으로 기존 resultMap 을 재사용한 게 문제였습니다. **그 둘도 `STATUS` 를 매핑하고 있어서** 하나뿐인 값이 세 군데에 다 들어갔습니다.

```
쿼리의 STATUS (= AS_SCHEDULE.STATUS)
   ├→ scheduleDTO.status    의도한 것
   ├→ engineerDTO.status    기사 근무상태 자리에 스케줄 상태가
   └→ requestDTO.status     접수 상태 자리에도 스케줄 상태가
```

**고치는 방법 셋을 놓고 비교했습니다.**

| 방법 | 판단 |
|---|---|
| 컬럼 이름 변경 | 팀원이 담당한 테이블까지 건드려야 해 다른 쿼리와 DTO 가 연쇄로 깨진다 |
| `columnPrefix` 사용 | 쿼리와 매핑 양쪽에 접두사 규칙을 새로 깔아야 한다. 필요한 건 상태값 하나인데 과하다 |
| **조회 전용 resultMap 분리** ✅ | 실제로 SELECT 되는 컬럼만 매핑한다. 팀원 파일을 안 건드리고 이 쿼리 안에서 끝난다 |

```xml
<!-- selectScheduleList 전용 — STATUS 를 아예 매핑하지 않는다 -->
<resultMap id="scheduleEngineer" type="...EngineerDTO">
  <id     column="ENGINEER_NO"   property="engineerNo"/>
  <result column="ENGINEER_NAME" property="engineerName"/>
</resultMap>
```

같은 자리에서 `INNER JOIN` 을 `LEFT JOIN` 으로 바꿨습니다. 기사나 접수 정보가 비어도 **스케줄 자체는 캘린더에 남아야** 하기 때문입니다.

> **재사용이 늘 이득은 아니라는 걸 배웠습니다.** resultMap 은 "이 DTO 를 어떻게 채울지"가 아니라 **"이 쿼리 결과를 어떻게 읽을지"** 에 가깝습니다. 쿼리가 다르면 읽는 법도 달라야 합니다.

### 덧 — 날짜·시간 타입 맞추기

이 과정에서 `startTime`/`endTime` 타입도 `java.sql.Time` 에서 `LocalTime` 으로 바꿨습니다. DB 의 TIME 컬럼을 자바에서 어떤 타입으로 받을지는 당연해 보였는데, 막상 비교·계산을 하려니 오래된 API 라 번거로웠습니다. **DB 타입과 자바 타입을 맞추는 일이 생각보다 손이 많이 간다**는 걸 이때 알았습니다.

---

## 작업 기록

작업일지를 Notion 에 남겼습니다.

| 날짜 | 내용 |
|---|---|
| 09/08 | [캘린더 라이브러리 학습과 화면 기획](https://scratched-ceramic-15c.notion.site/2026-09-08-3dd90bcc7b0781e1ad43eb0d6cdc21ca) |
| 09/09 | [콘텐츠 영역 분리로 캘린더 미리보기 제작](https://scratched-ceramic-15c.notion.site/2026-09-09-3dd90bcc7b0781dbaa49d6c94d6216c9) |
| 09/10 | [AS 기사 일정 배정 화면 설계 및 캘린더 연동 착수](https://scratched-ceramic-15c.notion.site/2026-09-10-AS-3d790bcc7b078108b781fb18134e89da) |
| 09/11 | [실제 프로젝트 연동 시작과 첫 버그 수정](https://scratched-ceramic-15c.notion.site/2026-09-11-3dd90bcc7b07811d93d2f5804aaf9d8a) |
| **09/14** | **[캘린더 미표시 버그 원인 분석과 중복 배정 검증 추가](https://scratched-ceramic-15c.notion.site/2026-09-14-3dd90bcc7b0781358221c2ba374db2db)** |
| 09/15 | [CSS 우선순위 문제 해결과 코드 주석 정리](https://scratched-ceramic-15c.notion.site/2026-09-15-CSS-3dd90bcc7b0781bf8661c058e3562001) |

> 9/11 에 SQL 쪽 컬럼 충돌을 별칭으로 정리하고 다 고쳤다고 생각했는데, **9/14 에 resultMap 쪽에 같은 충돌이 한 겹 더 있었다**는 걸 알게 됐습니다. 두 글을 이어서 읽으면 그 과정이 보입니다.

---

## 알려진 제약 · 보완할 점

팀 규칙 중에 **"한계도 함께 — 해결 못한 것은 숨기지 않고 다음 과제로"** 가 있었습니다. 아래는 그 규칙에 따라 정리한 것으로, 발표 질의응답을 준비하며 실제 코드를 다시 훑다가 찾았습니다. **고치지 못한 채로 남아 있습니다.**

- **중복 배정 검증이 프론트엔드에만 있습니다.** 폼을 거치지 않고 요청을 직접 보내면 우회됩니다. 서버에도 같은 검증이 있어야 합니다
- **`insertSchedule` 과 `updateReceiveRequest` 가 `@Transactional` 로 묶여 있지 않습니다.** 하나만 실패하면 배정은 됐는데 접수 상태는 그대로인 불일치가 생깁니다
- **배정 변경·취소·재배정 기능이 없습니다.** 최초 배정만 가능합니다
- **과거 날짜로 배정하는 것을 막는 검증이 없습니다** (프론트·백엔드 모두)
- **기사 근무시간(`workStartTime`/`workEndTime`) 밖 배정을 막는 로직이 없습니다**
- **퇴사 기사 재검증이 없습니다.** 배정 드롭다운에서는 `WHERE STATUS='WORK'` 로 빠지지만, 저장 시점에 서버가 다시 확인하지는 않습니다
- **동시성 제어가 없습니다.** 관리자 두 명이 같은 시간대에 동시에 배정하는 상황은 처리돼 있지 않습니다
- **`insertSchedule` 에 `useGeneratedKeys`/`keyProperty` 가 없어** 생성된 `SCHEDULE_NO` 가 저장 후 DTO 로 돌아오지 않습니다

> 첫 번째와 두 번째가 가장 큽니다. **"화면에서 막았으니 됐다"** 고 생각한 것이 이 프로젝트에서 가장 아쉬운 판단이었습니다.
> 발표 자체평가에도 **"브라우저 검증만으로는 겹침을 완전히 막지 못함"** 을 제가 얻은 것 중 하나로 적었습니다.

---

## 기술 스택

| | |
|---|---|
| **Backend** | Java 17 · Spring Boot 4.0.8 · MyBatis 4.0.1 · Thymeleaf (layout-dialect) · Lombok |
| **Database** | MariaDB |
| **Frontend** | JavaScript · FullCalendar 6.1.11 · HTML/CSS |
| **협업** | Git (`main` / `dev` / `ft/` · `fix/` · `style/` 접두어 · Pull Request) · Notion · ERDCloud |

---

## 실행 방법

### 1. 데이터베이스

```bash
mysql -u <user> -p <db> < 테이블생성쿼리.sql
```

샘플 데이터(기사·접수·배정·결과)가 함께 들어 있습니다.

### 2. 접속 정보

`src/main/resources/application.yaml` 의 `spring.datasource` 를 본인 환경에 맞게 바꿔 주세요.

### 3. 실행

```bash
./gradlew bootRun
```

`http://localhost:8080` · 관리자 일정관리는 `/as-schedule` 입니다.

---

## 팀 구성

**화면 단위로 나눴습니다.** 백엔드·프론트로 나누면 서로 기다리는 시간이 길어져서, 각자 화면 하나를 처음부터 끝까지 맡는 쪽을 택했습니다.

| 담당 | 범위 |
|---|---|
| 이웅진 (팀장) | 공통 · 로그인 · 통합 — 메인/로그인, 공통 레이아웃, 권한 분기, 접근제어 인터셉터, Git 통합 |
| 최준영 | 관리자 대시보드 · 기사관리 — 상태별 통계 카드, 최근 접수 현황, 기사 등록·수정·퇴사 |
| 박성인 | 기사 대시보드 · AS결과보고 — 오늘 일정, 사진 포함 결과 등록, 문자·만족도 연동 |
| **김성준** | **관리자 일정관리 — 캘린더 출력, 기사 배정, 배정 단계 겹침 확인, 상태별 표시** |

스케줄 테이블이 접수·기사 테이블을 둘 다 참조해서, **외래키 컬럼명과 자료형을 팀원들과 먼저 맞춘 뒤** 코드를 짰습니다.

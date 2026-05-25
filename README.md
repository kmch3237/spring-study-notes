# 📚 Spring MVC 공부 정리

Spring MVC 수업 내용을 정리한 저장소입니다.

---

## 목차

| 파일 | 내용 |
|------|------|
| [sp14_mvc_crud_흐름정리.txt](./sp14_mvc_crud_흐름정리.txt) | Insert / Update / Delete 요청 처리 흐름 상세 정리 |
| [sp14_vs_sp15_비교.txt](./sp14_vs_sp15_비교.txt) | XML 방식 vs Annotation 방식 전체 비교 |
| [sp15_annotation_study.txt](./sp15_annotation_study.txt) | Spring MVC Annotation 개념 및 예시 정리 |
| [sp16_board_study.txt](./sp16_board_study.txt) | 게시판 실전 — 3계층 구조, @Mapper, DBCP2, 페이징, 검색 완전 정리 |
| [sp16_delete_삭제처리_0523.txt](./sp16_delete_삭제처리_0523.txt) | 게시물 삭제(delete) 흐름 — Controller → Service → Mapper → XML 4단계 완전 정리 |
| [git_명령어_정리.txt](./git_명령어_정리.txt) | Git 기본 명령어 및 집↔학원 작업 순서 정리 |
| [vscode_프로젝트_루트_설정_이유.md](./vscode_프로젝트_루트_설정_이유.md) | VS Code에서 프로젝트 폴더를 잘못 열면 빨간줄 생기는 이유 및 해결법 |
| [springboot_로그인_기본개념.md](./springboot_로그인_기본개념.md) | URI/URL, Cookie, Session 개념 + 로그인 전체 흐름 + Spring Boot 코드 |

---

## 수업 진행 순서

- [x] sp14 — Spring MVC (XML 기반 설정)
  - 직원 / 지역 / 부서 / 직위 관리 CRUD
  - 로그인 / 로그아웃 (세션 처리)
  - Oracle DB, MyBatis, JSTL, jQuery 활용

- [x] sp15 — Spring MVC (Annotation 기반 설정)
  - @Controller, @RequestMapping, @Autowired
  - XML bean 등록 → Annotation으로 대체
  - component-scan 활용

- [x] sp16 — Spring MVC 게시판 실전 (3계층 구조 + MyBatis)
  - Controller → Service → Mapper 3계층 구조
  - @Mapper 인터페이스 (구현 클래스 없이 SQL 자동 연결)
  - DBCP2 커넥션 풀 (BasicDataSource)
  - 게시글 CRUD + 페이징(OFFSET/FETCH) + 검색 + 이전글/다음글

---

## 핵심 개념 요약

### sp14 vs sp15 핵심 차이

| | sp14 (XML) | sp15 (Annotation) |
|---|---|---|
| 컨트롤러 등록 | XML에 bean 태그 | @Controller |
| URL 매핑 | bean name="/xxx.action" | @RequestMapping |
| 의존성 주입 | XML property 태그 | @Autowired |
| 컨트롤러 파일 수 | 기능마다 1개 | 1개에 통합 |

### 주요 Annotation

```java
@Controller      // 컨트롤러 클래스 선언
@RequestMapping  // URL 매핑
@Autowired       // 타입으로 의존성 자동 주입
@Resource        // 이름으로 의존성 자동 주입
@RequestParam    // 요청 파라미터 수신
@Repository      // DAO 클래스 선언
@Service         // 서비스 클래스 선언
@Component       // 일반 빈 등록
```

---

## 개발 환경

- IDE : Spring Tool Suite 3.9.18 / VS Code
- Framework : Spring MVC 5.3.39
- DB : Oracle
- Build : Maven
- Java : 14 / 17

# 📚 Spring MVC 공부 정리

Spring MVC 수업 내용을 정리한 저장소입니다.

---

## 목차

| 파일 | 내용 |
|------|------|
| [sp14_vs_sp15_비교.txt](./sp14_vs_sp15_비교.txt) | XML 방식 vs Annotation 방식 전체 비교 |
| [sp15_annotation_study.txt](./sp15_annotation_study.txt) | Spring MVC Annotation 개념 및 예시 정리 |
| [git_명령어_정리.txt](./git_명령어_정리.txt) | Git 기본 명령어 및 집↔학원 작업 순서 정리 |

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

- IDE : Spring Tool Suite 3.9.18
- Framework : Spring MVC 5.3.39
- DB : Oracle
- Build : Maven
- Java : 14

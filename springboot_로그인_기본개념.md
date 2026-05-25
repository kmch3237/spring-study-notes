# 🔐 Spring Boot 로그인에 필요한 핵심 개념

---

## 1. URI / URL 개념

```
https://www.example.com:8080/user/login?id=hong&pw=1234#top
└─────┘ └─────────────────┘└──────────┘└──────────────┘└──┘
프로토콜      도메인:포트       Path(경로)    Query String   Fragment
```

| 용어 | 설명 | 예시 |
|---|---|---|
| **URL** | 전체 주소 | `http://localhost:8080/login` |
| **URI** | URL 포함 더 넓은 개념 (보통 같은 뜻으로 씀) | `/login`, `/user/mypage` |
| **Path** | 도메인 뒤의 경로 부분 | `/user/login` |
| **Query String** | `?` 뒤에 붙는 파라미터 | `?id=hong&pw=1234` |

### Spring Boot에서 URI 매핑
```java
@GetMapping("/login")       // GET /login  → 로그인 페이지 보여주기
@PostMapping("/login")      // POST /login → 로그인 처리 (아이디/비번 검사)
@GetMapping("/logout")      // GET /logout → 로그아웃 처리
@GetMapping("/mypage")      // GET /mypage → 로그인한 사람만 접근
```

> **GET** = 페이지 보여달라 (주소창에 직접 치는 것)  
> **POST** = 데이터 보내서 처리해달라 (폼 submit)

---

## 2. HTTP는 "기억을 못 한다" — 왜 Session/Cookie가 필요한가

```
[브라우저]  →  로그인 요청  →  [서버] "hong으로 로그인 완료"
[브라우저]  →  마이페이지 요청 → [서버] "누구세요??? 모르는 사람"  ← 문제!
```

HTTP는 **무상태(Stateless)** 프로토콜  
→ 요청이 끝나면 서버는 누가 보냈는지 **기억을 안 함**  
→ 로그인 상태를 유지하려면 **Cookie / Session** 이 필요

---

## 3. Cookie

> **브라우저(클라이언트)** 가 저장하는 데이터

```
로그인 성공
  ↓
서버가 브라우저에게 : "이 쿠키 들고 다녀"
  ↓
브라우저가 이후 모든 요청에 쿠키를 자동으로 같이 보냄
  ↓
서버 : "쿠키 보니까 hong이구나!"
```

```
[브라우저 쿠키 저장소]
  name=hong
  age=25
  visited=true
```

### 특징
| | 내용 |
|---|---|
| 저장 위치 | **브라우저 (클라이언트)** |
| 보안 | **낮음** — 사용자가 직접 볼 수 있고 조작 가능 |
| 용도 | 자동로그인, 팝업 "오늘 하루 보지않기", 장바구니 |
| 만료 | 설정한 날짜까지 유지 |

---

## 4. Session

> **서버** 가 저장하는 데이터, 브라우저엔 **세션ID** 만 쿠키로 저장

```
로그인 성공
  ↓
서버 메모리에 저장 : { "abc123" : { id:"hong", name:"홍길동", role:"USER" } }
  ↓
브라우저에게 : "쿠키에 세션ID=abc123 만 저장해"
  ↓
브라우저가 다음 요청 : 쿠키에 JSESSIONID=abc123 자동 첨부
  ↓
서버 : "abc123 찾아보니까 hong이구나!"
```

```
[서버 메모리 (세션 저장소)]
  JSESSIONID = abc123  →  { id: "hong", name: "홍길동" }
  JSESSIONID = def456  →  { id: "kim",  name: "김철수" }

[브라우저 쿠키]
  JSESSIONID = abc123   ← 이것만 저장 (실제 정보는 서버에)
```

### 특징
| | 내용 |
|---|---|
| 저장 위치 | **서버 메모리** |
| 보안 | **높음** — 실제 데이터는 서버에만 있음 |
| 용도 | **로그인 상태 유지** (메인 용도) |
| 만료 | 브라우저 닫으면 끝 (또는 설정한 시간) |

---

## 5. Cookie vs Session 비교

```
쿠키 방식                      세션 방식
─────────────────────────      ─────────────────────────
브라우저 ←── id=hong ───        브라우저 ←── JSESSIONID=abc ───
    ↓                               ↓
매 요청에 id=hong 전송          매 요청에 abc 전송
    ↓                               ↓
서버: id=hong 이구나            서버: abc 찾아보니 hong이구나

❌ 조작 가능 (보안 취약)        ✅ 조작해도 서버 데이터는 안 바뀜
```

---

## 6. 로그인 전체 흐름

```
1. 사용자가 /login 페이지 접속 (GET)
        ↓
2. 아이디/비번 입력 후 로그인 버튼 클릭 (POST /login)
        ↓
3. 서버: DB에서 아이디/비번 확인
        ↓
   [실패] → 로그인 페이지로 돌려보냄 + 에러 메시지
   [성공] ↓
4. 서버: 세션에 사용자 정보 저장
        session.setAttribute("loginUser", userInfo)
        ↓
5. 서버: 메인 페이지로 redirect
        ↓
6. 이후 모든 페이지: 세션 확인
        session.getAttribute("loginUser") == null → /login 강제 이동
        session.getAttribute("loginUser") != null → 정상 접근
        ↓
7. 로그아웃: session.invalidate() → 세션 삭제
```

---

## 7. Spring Boot 코드 미리보기

```java
// 로그인 처리 Controller
@PostMapping("/login")
public String login(String id, String pw, HttpSession session) {

    User user = userService.findByIdAndPw(id, pw);  // DB 조회

    if (user == null) {
        return "redirect:/login?error";  // 실패 → 다시 로그인 페이지
    }

    session.setAttribute("loginUser", user);  // ✅ 세션에 저장
    return "redirect:/main";                  // 성공 → 메인으로
}

// 로그아웃
@GetMapping("/logout")
public String logout(HttpSession session) {
    session.invalidate();  // ✅ 세션 전체 삭제
    return "redirect:/login";
}

// 마이페이지 (로그인 확인)
@GetMapping("/mypage")
public String mypage(HttpSession session, Model model) {
    User loginUser = (User) session.getAttribute("loginUser");

    if (loginUser == null) {
        return "redirect:/login";  // 비로그인 → 로그인 페이지로
    }

    model.addAttribute("user", loginUser);
    return "mypage";
}
```

---

## 8. 개념 정리 한장 요약

```
URI     : 서버에서 기능을 구분하는 주소  (/login, /mypage)
GET     : 페이지 요청 (주소창)
POST    : 데이터 전송 (폼 submit)

Cookie  : 브라우저 저장 → 보안 낮음 → 자동로그인, 팝업설정용
Session : 서버 저장    → 보안 높음 → 로그인 상태 유지용

로그인 흐름:
  POST /login → DB확인 → session.setAttribute() → redirect
  이후 페이지 → session.getAttribute() → null이면 /login으로
  로그아웃   → session.invalidate() → redirect
```

---

*작성일: 2026-05-25*

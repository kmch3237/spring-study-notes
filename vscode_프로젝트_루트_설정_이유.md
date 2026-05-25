# 📁 VS Code에서 doit5601 폴더를 루트로 열어야 하는 이유

> ✅ 요약 : VS Code는 **열린 폴더를 기준(루트)** 으로 `.classpath`와 `settings.json`을 읽는다.  
> `doit5601/` 이 아닌 바깥 폴더(`auctionproject/`)로 열면 경로가 틀어져 **전부 빨간줄** 발생.

---

## 1. 프로젝트 구조

```
auctionproject/          ← ❌ 이걸로 열면 빨간줄
└── doit5601/            ← ✅ 이걸로 열어야 정상
    ├── .classpath
    ├── .vscode/
    │   └── settings.json
    └── src/
        └── main/
            └── java/
                └── com/doit/controller/...
```

---

## 2. 빨간줄이 생기는 3가지 원인

### ① `.classpath` — 상대 경로 기준이 `doit5601/`

```xml
<!-- doit5601/.classpath -->
<classpathentry kind="src" path="src/main/java"/>
<classpathentry kind="lib" path="src/main/webapp/WEB-INF/lib/ojdbc11-23.26.0.0.0.jar"/>
```

- `path="src/main/java"` 는 **상대 경로**
- VS Code가 `auctionproject/`를 루트로 인식하면 → `auctionproject/src/main/java` 를 찾음
- 해당 경로가 없으니 **소스 폴더 인식 실패 → 모든 import 빨간줄**

---

### ② `.vscode/settings.json` — JAR 라이브러리 경로도 상대경로

```json
{
  "java.project.referencedLibraries": [
    "src/main/webapp/WEB-INF/lib/**/*.jar"
  ]
}
```

- `"src/main/webapp/..."` 도 상대 경로
- `auctionproject/`로 열면 → `auctionproject/src/main/webapp/...` 를 찾음
- JAR 파일 없음 → **라이브러리 인식 실패 → import 빨간줄**

---

### ③ Java Extension Pack 이 `.classpath` 를 루트에서만 탐색

- VS Code의 Java Extension Pack(`redhat.java`) 은 **열린 폴더의 루트**에서 `.classpath` 를 찾는다
- `auctionproject/`로 열면 `auctionproject/.classpath` 를 찾는데 파일이 없음
- → Java 프로젝트로 인식 자체가 안 됨

---

## 3. 정상 vs 오류 비교

| 열린 폴더 | .classpath 탐색 위치 | JAR 경로 | 결과 |
|---|---|---|---|
| `doit5601/` ✅ | `doit5601/.classpath` ← **파일 있음** | `doit5601/src/main/webapp/...` ← **있음** | 정상 |
| `auctionproject/` ❌ | `auctionproject/.classpath` ← **없음** | `auctionproject/src/main/webapp/...` ← **없음** | 빨간줄 |

---

## 4. 해결 방법 (앞으로 기억할 것)

```
✅ VS Code 에서 File → Open Folder → doit5601 선택
❌ auctionproject 또는 더 상위 폴더 선택 금지
```

또는 터미널에서:
```bash
cd doit5601
code .          # ← 이렇게 열면 자동으로 doit5601이 루트
```

---

## 5. 핵심 원칙

> **`.classpath`, `settings.json`, `pom.xml` 이 있는 폴더 = 프로젝트 루트**  
> VS Code는 이 파일들을 **루트 기준 상대경로**로 읽는다.  
> → 항상 이 파일들이 있는 폴더를 직접 열어야 한다.

---

## 6. 관련 파일 위치 (doit5601 기준)

| 파일 | 역할 |
|---|---|
| `.classpath` | 소스 폴더 / 라이브러리 경로 정의 (Eclipse/VS Code 공용) |
| `.vscode/settings.json` | VS Code Java Extension JAR 경로 설정 |
| `src/main/webapp/WEB-INF/lib/` | 프로젝트 라이브러리 (ojdbc, jstl, gson 등) |

---

*작성일: 2026-05-25*

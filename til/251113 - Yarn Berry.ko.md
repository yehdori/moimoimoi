# Yarn & Corepack

---

## 1. Yarn Berry와 Yarn 4, 그리고 Corepack

### 1-1. Yarn Berry란?

- **Yarn Berry**는 **Yarn 2 이상 버전 전체**(2.x, 3.x, 4.x …)를 부르는 코드네임.
- 정리하면:

  | 이름 | 실제 버전 | 설명 |
  |------|----------|------|
  | Yarn Classic | 1.x | 예전 구조, npm과 비슷, `node_modules` 기반 |
  | Yarn Berry | 2.x 이상 (2, 3, 4…) | 새로운 구조, Plug'n'Play(PnP), Zero-Installs 등 도입 |

- 그래서 **Yarn 4도 “Berry 계열”**에 포함된다.

### 1-2. Corepack이란?

- **Corepack**은 **Node.js에 내장된 패키지 매니저 버전 관리자**.
- Node 16.10 이상에 기본 포함.
- 하는 일 요약:
  > 프로젝트의 `packageManager` 필드를 보고,  
  > 해당 버전의 Yarn/pnpm을 자동으로 다운로드 & 실행해주는 도우미.

예:

```jsonc
// package.json
{
  "packageManager": "yarn@4.2.2"
}
```

이 경우 Corepack은:

1. `yarn@4.2.2` 버전을 받아 캐싱하고
2. 프로젝트 디렉토리에서 `yarn` 명령을 치면 항상 **4.2.2 버전**을 사용하게 한다.

### 1-3. Corepack 기본 사용 패턴

```bash
# Corepack 활성화
corepack enable

# Yarn 최신 안정(stable) 버전 사용 & package.json에 기록
yarn set version stable

# 또는 특정 버전
yarn set version 4.1.0
```

이렇게 하면 `package.json`에 대략 다음과 같이 기록된다.

```jsonc
{
  "packageManager": "yarn@4.1.0"
}
```

그리고 프로젝트 안에서는 별도의 전역 설치 없이 항상 이 버전으로 실행된다.

---

## 2. `yarn link`와 `yarn publish`

### 2-1. `yarn link`

#### 개념

- `yarn link`는 **로컬에 있는 라이브러리 패키지를, 다른 로컬 프로젝트에서 설치한 것처럼 연결**해서 쓰는 명령어.
- npm에 publish 하지 않고도, **내 컴퓨터 안에서 여러 프로젝트를 붙여서 개발/테스트**할 수 있음.

예시 디렉토리 구조:

```txt
/workspace/
  ├── my-ui-lib/   # 내가 만든 컴포넌트/유틸 라이브러리
  └── my-app/      # 라이브러리를 사용하는 실제 앱
```

#### 사용 방법

1. **라이브러리 쪽에서 링크 등록**

   ```bash
   cd my-ui-lib
   yarn link
   ```

   → 전역(global)에 `my-ui-lib`라는 이름으로 심볼릭 링크가 등록됨.

2. **앱 쪽에서 그 링크를 사용하겠다고 선언**

   ```bash
   cd ../my-app
   yarn link my-ui-lib
   ```

   → 이제 `my-app`에서 다음처럼 import 가능:

   ```ts
   import { Button } from 'my-ui-lib';
   ```

   이 때 실제로는 로컬 경로 `../my-ui-lib`의 소스를 읽어온다.

#### 특징 & 주의점

- 장점
  - 라이브러리를 수정하면, 바로 앱에서 테스트 가능 (publish 필요 없음).
- 단점 (특히 Yarn Berry/PnP 환경에서)
  - **`node_modules` + 전역 심볼릭 링크**에 의존하는 구조라서, PnP 철학과 맞지 않음.
  - TypeScript/빌드 설정에 따라 소스 경로 문제로 에러가 생기기도 함.

### 2-2. `yarn publish`

#### 개념

- `yarn publish`는 **현재 패키지를 npm 레지스트리에 배포하는 명령어**.

#### 기본 예시

```bash
yarn publish --access public
```

- `--access public` : 공개 패키지로 배포 (개인/오픈소스 라이브러리 일반적인 옵션)

출시 플로우 예시:

```bash
# 1) 버전 올리기 (1.0.0 → 1.0.1)
yarn version patch

# 2) npm에 올리기
yarn publish --access public
```

이제 다른 프로젝트에서:

```bash
yarn add my-ui-lib
```

으로 설치 가능.

### 2-3. `link` vs `publish` 요약표

| 명령어 | 목적 | 어디에 쓰나 | 결과 |
|--------|------|------------|------|
| `yarn link` | 로컬 개발 / 테스트용 연결 | 내 컴퓨터의 여러 프로젝트 간 | npm 업로드 ❌ |
| `yarn publish` | 패키지 배포 | npm 레지스트리 | npm 업로드 ⭕ |

---

## 3. `yarn link` vs `yarn portal`

### 3-1. `yarn link` (Classic 스타일)

- **Yarn 1 (Classic) 시대**의 방식.
- 동작 원리 (개념):
  1. 라이브러리에서 `yarn link`
     - 전역 위치에 그 라이브러리를 가리키는 **심볼릭 링크** 생성
  2. 사용하는 프로젝트에서 `yarn link my-lib`
     - `node_modules/my-lib`가 그 전역 링크를 가리키도록 설정

- 전제 조건:
  - 프로젝트에 **`node_modules/` 폴더가 존재**해야 하며,
  - Yarn이 거기에 **심볼릭 링크를 심는다는 가정**이 필요함.

→ 즉, **node_modules 기반**의 오래된 방식.

### 3-2. Yarn 2+ (Berry)와 PnP의 철학

Yarn Berry는 기본적으로 **PnP(Plug'n'Play)** 를 지향:

- `node_modules` 폴더를 만들지 않거나, 중요하게 취급하지 않음.
- 대신 `.pnp.cjs` 파일 + `.yarn/cache` 안의 zip 패키지로 의존성을 관리.
- 모듈 로딩은 `node_modules` 스캔이 아니라, **PnP의 매핑 정보**를 통해 일어남.
- 철학적으로:
  1. **재현 가능성(reproducibility)** = lockfile + 설정만 있으면 어디서나 동일한 의존성 트리.
  2. **글로벌 상태(global state) 최소화** = 전역에 뭔가 설치되어 있어야만 동작하는 상황을 피함.

`yarn link`는:

- 전역에 심볼릭 링크를 생성 (global state 생김).
- `node_modules`에 의존 (PnP 철학과 상충).
- lockfile에 정보가 기록되지 않아서, 다른 개발자가 `git clone` + `yarn install`을 해도 이 링크 상태가 복원되지 않음.

→ 그래서 Yarn Berry에서는 **권장되지 않는 방식**으로 간주.

### 3-3. `yarn portal` (Yarn Berry 방식)

Yarn 3+에서 도입된, PnP/lockfile 친화적인 로컬 연결 방식.

#### 사용 예시

```bash
cd ~/dev/my-app
yarn add portal:../my-lib
```

`package.json`:

```jsonc
{
  "dependencies": {
    "my-lib": "portal:../my-lib"
  }
}
```

이 의미는:
> “`my-lib`라는 패키지는 npm registry에서 받지 말고,  
> 상대 경로 `../my-lib` 디렉토리를 직접 의존성으로 사용하자.”

#### 특징

- **node_modules에 심볼릭 링크를 만들지 않음**
- **전역 상태(global link)가 필요 없음**
- 이 정보가 `package.json` + `yarn.lock`에 **공식적으로 기록**되므로:
  - 프로젝트를 다른 환경에서 설치해도 재현 가능함.
- PnP resolver가 이 경로를 정식 의존성으로 인식 → Berry 철학에 부합.

### 3-4. 사용자 이해 정리 (Q&A 기반)

1. `yarn link`는
   - `node_modules` + **전역 심볼릭 링크**를 전제로 한 Yarn Classic 방식.
   - 동작 구조:  
     `my-app/node_modules/my-lib → (symlink) → 전역 링크 → 실제 라이브러리 폴더`

2. `yarn portal`은
   - **PnP/lockfile 기반**으로,  
     전역 심볼릭 링크 없이 **경로 자체를 의존성으로 등록**하는 modern 방식.
   - `portal:../my-lib`는 “이 폴더를 직접 dependency로 쓰자”라는 의미.
   - 이 정보가 `package.json`과 `yarn.lock`에 기록되어 재현 가능성이 높음.

#### 최종 비교표

| 항목 | `yarn link` | `yarn portal` |
|------|-------------|---------------|
| 세대 | Yarn Classic (1.x) | Yarn Berry (2+) |
| 기본 전제 | `node_modules` 존재 | PnP/lockfile 중심, `node_modules` 필요 없음 |
| 연결 방식 | 전역 심볼릭 링크 + `node_modules` | 경로를 직접 dependency로 등록 |
| 전역 상태 필요 여부 | 필요 (global link) | 불필요 |
| 재현 가능성 | 낮음 (lockfile에 안 남음) | 높음 (lockfile에 기록됨) |
| Berry에서의 추천 여부 | 비추천 | 권장 |

---

아래에 **Markdown 형식으로 깔끔하게 정리**해줬어!
바로 복붙해서 노션/깃헙에 넣어도 될 정도로 구조화해놓음 📘✨

---

# 📦 Yarn이 `node_modules` 를 없애고 PnP(Plug'n'Play)를 선택한 이유

### — 왜 `.pnp.cjs + .yarn/cache(zip)` 방식이 “방법만 다르게 한 것”이 아니라 **완전히 다른 구조**인지

---

# 1. 결론 요약

> Yarn Berry는 **속도, 안정성, 재현성, 보안** 문제를 해결하기 위해
> `node_modules`를 버리고 PnP 구조를 도입했다.
> PnP는 단순히 저장 방식이 다른 게 아니라, **의존성을 로딩하는 철학이 완전히 다르다.**

---

# 2. Node_modules 방식의 근본 문제점

## 🚨 2-1. 너무 느리고 너무 큰 구조

Node는 의존성을 설치할 때 파일 수만 개를 생성함.

* 디렉토리 수: 수십~수백 개
* 파일 수: 수천~수만 개
* 설치/삭제 시 디스크 I/O 폭발 → **느림**

> 의존성 하나 업데이트해도 node_modules 내용을 거의 전부 다시 쓸 때가 있음.

---

## 🚨 2-2. 중첩 지옥 (nested dependency hell)

`node_modules` 내부는 다음처럼 복잡함:

```
node_modules/
  react/
    node_modules/
      scheduler/
      object-assign/
  lodash/
  ...
```

문제점:

* 어떤 버전이 실제 사용되는지 파악 어려움
* 의존성 충돌 시 디버깅 힘듦
* 경로 길어서 파일 시스템 문제 일어날 때도 있음

---

## 🚨 2-3. Node의 기본 module resolution 성능 문제

Node는 `require('lodash')`를 할 때:

```
현재 디렉토리/node_modules → 상위/node_modules → 상위/node_modules → ...
```

이렇게 **계속 위로 탐색**함.

* 느림
* 경로 충돌 가능
* 매번 디스크 접근 → 성능 저하

---

## 🚨 2-4. 재현성(reproducibility)이 떨어짐

`node_modules`는 환경에 따라 설치 결과가 달라질 수 있음.

문제 예:

* Mac/Windows 설치 결과가 다름
* 특정 버전이 “운 좋게” 설치되기도 함
* npm이 잠시 장애나면 다른 버전이 땡겨옴
* CI와 개발자의 로컬 설치 결과가 달라짐

> **"내 컴퓨터에서는 되는데?"** 문제의 근원.

---

## 🚨 2-5. 전역 상태(global state)에 영향을 받음

`yarn link` 같은 기능은 전역 디렉토리에 symlink를 만들어서 연결함.

→ lockfile에도 반영되지 않고
→ 다른 환경에서 재현 불가능

---

# 3. Yarn Berry(PnP)의 철학과 해결책

Yarn Berry는 **“패키지를 OS의 파일구조로 관리하지 말자”**라는 철학으로 전환함.

---

# 4. PnP(Plug'n'Play)의 구조

## 📁 4-1. 패키지를 zip 형태로 `.yarn/cache`에 저장

```
.yarn/cache/
  react-npm-18.2.0.zip
  lodash-npm-4.17.21.zip
```

→ 수십만 개 파일이 아니라 **수백 개 zip 파일로 끝**
→ 설치 속도 비약적 향상

---

## 📜 4-2. `.pnp.cjs` 파일이 “의존성 맵(Map)” 역할

`.pnp.cjs` 내부에는 다음과 같은 정보 저장됨:

```js
{
  "react@18.2.0": ".yarn/cache/react-npm-18.2.0.zip/node_modules/react",
  ...
}
```

장점:

* Node가 디스크를 뒤지지 않고
  **Yarn에게 바로 "이 패키지 어디 있음?" 물어봄**
* → 단일 매핑 lookup으로 resolve 끝
* → 속도 ↑, 충돌 ↓, 가독성 ↑

---

## 🧲 4-3. 재현성 100% 보장

필요한 건 딱 두 개:

* `package.json`
* `yarn.lock + .pnp.cjs`

이 값들이 같으면:

* Mac / Linux / Windows 모두 **100% 동일한 의존성 트리 보장**
* CI에서도 완벽히 동일하게 설치됨

---

## 🔒 4-4. 전역 상태 없음

`portal:../lib` 같은 프로토콜은:

* 전역 링크 필요 없음
* lockfile에 기록됨 → **완전 재현 가능**

---

# 5. node_modules 방식 vs PnP 방식 비교

| 항목                | node_modules      | PnP (`.pnp.cjs + .yarn/cache`) |
| ----------------- | ----------------- | ------------------------------ |
| 구조                | 실제 파일/폴더 생성       | 패키지는 zip 캐시, 경로는 매핑 파일 관리      |
| module resolution | 디스크 탐색 기반         | `.pnp.cjs` 내부 매핑 사용            |
| 파일 개수             | 수천~수만             | 수백                             |
| 설치 속도             | 느림                | 매우 빠름                          |
| 재현성               | 낮음                | 매우 높음                          |
| 글로벌 상태 영향         | 있음                | 없음                             |
| 충돌 확률             | 높음                | 낮음                             |
| 철학                | “OS 파일시스템이 구조 결정” | “Yarn이 구조 직접 관리”               |

---

# 6. 결론

> **겉보기엔 “저장 방식만 다르지 않나?” 같지만
> 실은 “의존성을 어떻게 찾고 관리하는지”의 철학 자체가 다르다.**

* node_modules =
  **"OS 파일 구조를 기반으로, Node가 디렉토리를 뒤져가며 찾는 방식"**

* PnP =
  **"Yarn이 모든 의존성 위치를 직접 관리하고, zip 캐시에서 즉시 resolve하는 방식"**

Yarn이 node_modules를 버린 건 단순히 구조 변경이 아니라
**의존성 관리의 패러다임을 바꾼 것.**

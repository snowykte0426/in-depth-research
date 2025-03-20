# SSR과 CSR 간 Hydration Mismatch 문제 해결 방안 (FOUT 방지 및 OCP 준수)

## 1. 문제의 원인: SSR과 CSR의 타이밍 차이
- **SSR(Server-Side Rendering)**: 서버에서 미리 HTML을 렌더링해 클라이언트로 전송합니다.
- **CSR(Client-Side Rendering)**: 클라이언트에서 React가 HTML을 재사용(수화, hydrate)하여 인터랙티브하게 만듭니다.
- 서버에서는 브라우저 전용 API(localStorage, window 등)를 사용할 수 없으므로, SSR 단계에서는 기본값(예: light 테마)으로 렌더링되고,
  클라이언트에서는 사용자의 실제 테마(예: localStorage에 저장된 dark 테마)를 반영하면 HTML이 달라져 **hydration mismatch**가 발생합니다.

## 2. FOUT(Flash Of Unstyled Content) 방지
- **FOUT**: 페이지 로딩 시 잘못된 스타일이나 테마가 잠깐 보였다가 올바르게 전환되는 현상입니다.
- **해결책:**
  - 서버에서 쿠키나 HTTP Client Hints(예: `Sec-CH-Prefers-Color-Scheme`)를 사용하여 사용자의 테마 정보를 미리 파악.
  - SSR 시점에서 이 정보를 바탕으로 올바른 테마를 적용하여 HTML을 렌더링.
  - 또는, HTML `<head>`에 인라인 스크립트를 추가하여 브라우저가 HTML 해석 전에 테마 클래스를 적용하게 함.
    ```html
    <script>
      try {
        const theme = localStorage.getItem('theme') || (window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light');
        document.documentElement.classList.add(theme);
      } catch(e) {}
    </script>
    ```
  - 이로 인해 초기 로딩 시 잘못된 스타일이 보이는 현상을 줄일 수 있습니다.

## 3. OCP(Open-Closed Principle) 준수
- **OCP**: 소프트웨어는 확장에는 열려 있으면서 수정에는 닫혀 있어야 한다는 원칙입니다.
- **적용 방안:**
  - 테마 관련 로직을 각 컴포넌트에 분산시키지 않고, **ThemeProvider** 같은 중앙 집중식 컨텍스트를 사용.
  - 서버에서는 쿠키나 헤더를 통해 사용자의 테마 정보를 읽어온 후, 해당 값을 ThemeProvider에 전달하여 SSR 시 올바른 테마로 렌더링.
  - 새로운 상태 저장 방법(예: IndexedDB, 서버 세션 등)이 도입되더라도 ThemeProvider 내부에서 확장할 수 있으므로, 기존 컴포넌트는 수정할 필요가 없습니다.

## 4. 클라이언트 상태 저장 방법 (쿠키, localStorage 외)
- **쿠키**: 서버와 클라이언트 모두 접근 가능하므로 SSR 시 사용자 정보를 반영하기에 용이합니다.
- **HTTP Client Hints**: `Sec-CH-Prefers-Color-Scheme` 헤더 등을 통해 사용자의 OS 테마 정보를 전달받을 수 있습니다.
- **서버 사이드 세션/데이터베이스**: 로그인 사용자의 경우, 서버에 저장된 사용자 프로필에서 테마 정보를 조회하여 SSR에 반영할 수 있습니다.
- **URL 파라미터**: 상태 정보를 전달할 수 있으나, 테마와 같이 지속적이고 민감한 정보에는 적합하지 않습니다.

## 5. 미들웨어와 서버-클라이언트 연계
- **미들웨어 활용**: 모든 요청마다 쿠키나 헤더를 확인해 사용자의 테마 정보를 읽어 SSR에 전달할 수 있습니다.
- 예를 들어, Next.js의 미들웨어에서 쿠키를 읽어 `x-user-theme`와 같은 커스텀 헤더를 추가하고, SSR 로직에서 이를 참고하여 올바른 테마로 렌더링합니다.
- 이를 통해 서버와 클라이언트 간 상태 일관성을 유지하여 hydration mismatch를 방지할 수 있습니다.

## 6. 초기 로딩 성능과 UX 최적화
- **API 호출 최소화**: SSR 시 불필요한 API 호출을 줄여 초기 로딩 시간을 단축합니다.
- **동일 초기 상태 유지**: SSR과 CSR이 동일한 초기 상태(예: 테마, 사용자 선호도)를 공유하도록 설계해 불필요한 재렌더링을 방지합니다.
- **인라인 스크립트 및 CSS 최적화**: 필요한 경우, 인라인 스크립트나 CSS를 사용해 첫 화면에서 바로 올바른 스타일이 적용되도록 합니다.

## 결론
- SSR과 CSR의 타이밍 차이로 인해 hydration mismatch가 발생하며, 이는 브라우저 전용 데이터(localStorage 등)를 SSR에서 반영하지 못하기 때문입니다.
- 올바른 테마를 SSR 시점에서 미리 적용하거나, 인라인 스크립트로 빠르게 테마 클래스를 적용하면 hydration mismatch와 FOUT를 모두 방지할 수 있습니다.
- 중앙 집중식 상태 관리(ThemeProvider)와 미들웨어를 통한 상태 동기화는 OCP 원칙을 준수하면서도 유지보수가 용이한 아키텍처를 만듭니다.

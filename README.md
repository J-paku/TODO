# 왜 `pages/` 안에 컴포넌트가 있어도 빌드가 되는가

결론: `pages/`에 있어도 **모든 파일이 페이지로 빌드되는 건 아니다**.  
이 프로젝트는 `next.config.mjs`에서 페이지로 인식할 확장자를 제한하고 있다.

## 핵심 설정
`next.config.mjs`에서:

- `pageExtensions: ['tsx', 'page.tsx', 'page.jsx', 'page.js']`

이 의미는:
- `pages/` 아래라도 **확장자가 위 리스트에 포함된 파일만** 라우트(페이지) 후보가 된다.
- 따라서 `pages/claim/foo.ts` 같은 파일은 **페이지가 아니다** → 빌드 통과.
- `pages/claim/foo.tsx`는 **페이지 후보** → `default export` React 컴포넌트가 필요.

## 그래서 빌드가 되는 이유
- `pages/claim` 내부의 컴포넌트 파일들이
  - `.ts` 확장자라서 페이지로 인식되지 않거나
  - `.tsx`인데도 `default export`로 컴포넌트가 있어서 유효한 페이지이기 때문.

## 요약
- **페이지 인식 규칙은 확장자 기준**
- **`default export` 필요 조건은 “페이지로 인식된 파일”에만 적용**
- 그래서 `pages/` 안에 컴포넌트가 있어도 빌드는 정상 통과할 수 있다

참고 파일: `PiPiT-web/next.config.mjs`

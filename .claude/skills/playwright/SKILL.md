---
name: playwright
description: Playwright로 브라우저 자동화·E2E 테스트를 수행하는 스킬. 웹 페이지 열기, 스크린샷, 폼 입력, 클릭, 네비게이션 검증, @playwright/test 테스트 작성·실행 요청 시 사용. 트리거 예시 - "브라우저로 열어봐", "스크린샷 찍어줘", "E2E 테스트 작성", "playwright", "페이지 동작 확인", "폼 자동 입력".
---

# Playwright 스킬

이 프로젝트는 Microsoft의 **Playwright MCP 서버**(`@playwright/mcp`)를 `.mcp.json`에 등록해 두었다.
Claude Code가 이 저장소에서 실행되면 `mcp__playwright__*` 도구가 자동으로 노출된다.

## 두 가지 사용 방식

| 상황 | 방식 |
|------|------|
| 페이지를 직접 조작·확인해야 할 때 (탐색, 스크린샷, 폼 입력, 디버깅) | **MCP 도구** (`mcp__playwright__browser_*`) |
| 반복 실행할 회귀 테스트가 필요할 때 | **`@playwright/test`** 테스트 파일 작성 후 `npx playwright test` |

먼저 MCP 도구로 실제 동작을 확인한 뒤, 검증된 흐름을 테스트 코드로 굳히는 순서를 권장한다.

## 1. MCP 도구로 브라우저 조작

주요 도구 (모두 `mcp__playwright__` 접두어):

- `browser_navigate` - URL 이동
- `browser_snapshot` - 접근성 트리 스냅샷. **스크린샷보다 우선** 사용한다. 요소 `ref`를 얻어 다음 동작에 쓴다.
- `browser_click`, `browser_type`, `browser_fill_form`, `browser_select_option`, `browser_hover`, `browser_press_key`
- `browser_take_screenshot` - 시각 확인이 꼭 필요할 때만
- `browser_wait_for` - 텍스트 등장/사라짐 또는 시간 대기
- `browser_console_messages`, `browser_network_requests` - 디버깅
- `browser_tabs`, `browser_navigate_back`, `browser_close`

작업 순서:

1. `browser_navigate`로 이동한다.
2. `browser_snapshot`으로 구조를 읽고, 조작할 요소의 `ref`를 확인한다.
3. 클릭·입력 등을 수행한다. 매 동작 후 새 스냅샷으로 결과를 확인한다.
4. 결과를 사용자에게 보고할 때는 스냅샷에서 확인한 사실만 말한다. 추측하지 않는다.

주의:

- 도구 이름이 보이지 않으면 `ToolSearch`로 `playwright`를 검색해 스키마를 먼저 로드한다.
- 로그인 정보나 결제 정보는 사용자가 명시적으로 준 값만 입력한다.
- 외부 사이트에 글을 올리거나 주문·결제·삭제 같은 되돌리기 어려운 동작은 실행 전에 사용자 확인을 받는다.

## 2. `@playwright/test`로 E2E 테스트 작성

### 초기 설정 (프로젝트에 아직 없을 때)

```bash
npm init -y
npm install -D @playwright/test
```

이 클라우드 컨테이너에는 Chromium이 `/opt/pw-browsers`에 미리 설치되어 있다.
`npx playwright install`은 실행하지 않는다. 로컬 환경에서는 필요 시 `npx playwright install chromium`을 실행한다.

`playwright.config.ts` 최소 예시:

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  reporter: [['list'], ['html', { open: 'never' }]],
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
});
```

### 테스트 작성 규칙

```ts
import { test, expect } from '@playwright/test';

test('로그인 후 대시보드가 보인다', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('이메일').fill('user@example.com');
  await page.getByLabel('비밀번호').fill('secret');
  await page.getByRole('button', { name: '로그인' }).click();
  await expect(page.getByRole('heading', { name: '대시보드' })).toBeVisible();
});
```

- 로케이터는 `getByRole` > `getByLabel` > `getByText` > `getByTestId` 순으로 선택한다. CSS/XPath는 최후 수단이다.
- `page.waitForTimeout`은 쓰지 않는다. `expect(...).toBeVisible()` 같은 자동 대기 단언을 사용한다.
- 테스트마다 독립적인 상태를 만든다. 테스트 간 실행 순서에 의존하지 않는다.
- 파일 위치는 `tests/*.spec.ts`. 이름은 검증하는 행동을 한국어 또는 영어로 명확히 쓴다.

### 실행

```bash
npx playwright test                 # 전체
npx playwright test tests/login.spec.ts   # 파일 하나
npx playwright test --headed        # 브라우저 표시 (로컬)
npx playwright show-report          # HTML 리포트
```

실패하면 `test-results/` 아래의 스크린샷과 trace를 확인한다. `npx playwright show-trace <trace.zip>`으로 trace를 연다.

## 3. 보고 형식

- 무엇을 확인했는지, 어떤 URL에서, 어떤 결과였는지 사실만 적는다.
- 테스트를 실행했다면 통과/실패 수와 실패한 테스트 이름을 적는다. 실패 출력은 코드 블록에 넣는다.
- 스크린샷 파일을 만들었다면 경로를 알려 준다.

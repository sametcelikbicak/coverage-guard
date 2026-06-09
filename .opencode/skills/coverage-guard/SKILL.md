---
name: coverage-guard
description: "Use when the user wants to check test coverage, enforce 100% coverage, find uncovered code, add missing tests, or increase code coverage. Works with vitest, jest, react-scripts, and other test runners. Also for 'coverage', 'test coverage', 'cover', 'untested', 'uncovered', 'add tests for', 'increase coverage', 'coverage report', 'write tests', 'test all files'."
license: MIT
compatibility: opencode, claude-code, cursor, windsurf, github-copilot
metadata:
  category: testing
  audience: developers
---

# Coverage Guard

Ensure 100% test coverage for any JavaScript/TypeScript project. Automatically detects the test runner, sets up test infrastructure if missing, writes tests for uncovered components, and loops until every file reaches 100% coverage.

## Overview

This skill follows a complete pipeline:

1. **Detect** runner & infrastructure → 2. **Scan** source files & tests → 3. **Setup** if missing → 4. **Generate** new tests → 5. **Run** coverage → 6. **Fill** gaps → 7. **Verify** → repeat until 100%

---

## Step-by-step Workflow

### Phase 1: Detect environment

Check `package.json` in the project root to determine:

```bash
cat package.json
```

Identify the test runner and coverage tool:

| Indicator | Runner | Coverage command |
|---|---|---|
| `"vitest"` in devDependencies | **Vitest** | `npx vitest --coverage` |
| `"jest"` in devDependencies | **Jest** | `npx jest --coverage` |
| `"react-scripts"` in devDependencies | **react-scripts (Jest)** | `npx react-scripts test --coverage` |
| `"next"` in devDependencies | **Next.js (Jest)** | `npx next test --coverage` or check `jest.config` |
| `"@angular/core"` in dependencies | **Angular (Karma/Jasmine)** | `ng test --code-coverage` |
| `"@playwright/test"` in devDependencies | **Playwright** | `npx playwright test` (no built-in coverage; use `@playwright/test` with `istanbul`) |
| `"mocha"` in devDependencies | **Mocha** | `npx mocha` (coverage via c8/nyc) |
| `"cypress"` in devDependencies | **Cypress** | `npx cypress run` (coverage via `@cypress/code-coverage`) |
| `"ava"` in devDependencies | **AVA** | `npx ava` (coverage via c8) |
| None found | **None** → proceed to Phase 2 → Phase 3 setup |

### Phase 2: Scan source files and existing tests

Find all source files (components, modules, utilities) that need tests:

```bash
# Common source directories
find src -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.jsx" \) ! -name "*.test.*" ! -name "*.spec.*" ! -name "*.d.ts" ! -path "*/node_modules/*" ! -path "*/dist/*" ! -path "*/build/*" ! -path "*/.next/*"
```

Find all existing test files:

```bash
find src -type f \( -name "*.test.*" -o -name "*.spec.*" \) ! -path "*/node_modules/*"
```

Map each source file to a test file:

- `src/components/Button.tsx` → `src/components/Button.test.tsx` or `src/components/__tests__/Button.test.tsx`
- `src/utils/format.ts` → `src/utils/format.test.ts` or `src/utils/__tests__/format.test.ts`
- Follow the existing project convention for test file placement

Classify each source file:
- **Has test** → add to coverage analysis list
- **No test** → mark for test generation

### Phase 3: Setup test infrastructure (if missing)

If no test runner was detected in Phase 1, ask the user interactively:

```
? No test framework detected. Which one would you like to set up?
  1) Vitest (Recommended — fast, modern, native ESM)
  2) Jest (Most widely used)
  3) react-scripts (Create React App)
  4) None — skip, I'll set it up manually
```

Once the user selects, install dependencies and create config:

**Vitest setup:**
```bash
npm install -D vitest @vitest/coverage-v8
```

Create `vitest.config.ts`:
```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        '**/node_modules/**',
        '**/dist/**',
        '**/*.config.*',
        '**/*.d.ts',
      ],
    },
  },
})
```

Add to `package.json` scripts:
```json
{
  "scripts": {
    "test": "vitest",
    "test:coverage": "vitest --coverage"
  }
}
```

**Jest setup:**
```bash
npm install -D jest @types/jest ts-jest
npx ts-jest config:init
```

```bash
npm install -D jest @testing-library/react @testing-library/jest-dom jest-environment-jsdom
```

Create `jest.config.ts`:
```ts
export default {
  testEnvironment: 'jsdom',
  transform: {
    '^.+\\.tsx?$': 'ts-jest',
  },
  coverageReporters: ['text', 'json', 'html'],
  collectCoverageFrom: ['src/**/*.{ts,tsx}', '!src/**/*.d.ts'],
}
```

Add to `package.json` scripts:
```json
{
  "scripts": {
    "test": "jest",
    "test:coverage": "jest --coverage"
  }
}
```

**react-scripts setup:**
```bash
npm install -D @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

Coverage comes built-in:
```json
{
  "scripts": {
    "test": "react-scripts test",
    "test:coverage": "react-scripts test --coverage"
  }
}
```

After setup, run a quick test to confirm everything works:
```bash
npm test
```

If it fails, fix configuration issues before proceeding.

### Phase 4: Write tests for untested source files

For each source file that has **no test file**, analyze the source code:

1. Read the source file
2. Identify:
   - All exported functions, classes, components
   - Return types and possible return paths
   - Parameters and their types (required, optional, default values)
   - Props interface (for React components)
   - Async behavior (Promises, async/await)
   - Side effects (API calls, localStorage, etc.)
   - Error states and edge cases
3. Generate the test file following the project's conventions:
   - Use the detected test runner's API (`describe`/`it`/`expect` for Jest/Vitest, etc.)
   - For React: use `@testing-library/react` (`render`, `screen`, `fireEvent`)
   - Mock external dependencies
   - Cover all branches, edge cases, and error paths

**React component example** (`src/components/Button.tsx` → `src/components/Button.test.tsx`):
```tsx
import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from './Button'

describe('Button', () => {
  it('renders with label', () => {
    render(<Button label="Click me" />)
    expect(screen.getByText('Click me')).toBeInTheDocument()
  })

  it('calls onClick when clicked', () => {
    const onClick = vi.fn() // jest.fn() for Jest
    render(<Button label="Click" onClick={onClick} />)
    fireEvent.click(screen.getByText('Click'))
    expect(onClick).toHaveBeenCalledTimes(1)
  })

  it('is disabled when disabled prop is true', () => {
    render(<Button label="Click" disabled />)
    expect(screen.getByText('Click')).toBeDisabled()
  })
})
```

**Utility function example** (`src/utils/format.ts` → `src/utils/format.test.ts`):
```ts
import { formatCurrency, truncate } from './format'

describe('formatCurrency', () => {
  it('formats integer amount', () => {
    expect(formatCurrency(1000)).toBe('$1,000.00')
  })

  it('formats decimal amount', () => {
    expect(formatCurrency(99.95)).toBe('$99.95')
  })

  it('handles zero', () => {
    expect(formatCurrency(0)).toBe('$0.00')
  })

  it('handles negative values', () => {
    expect(formatCurrency(-50)).toBe('-$50.00')
  })
})
```

### Phase 5: Run coverage report

Run the appropriate coverage command based on detected runner:

| Runner | Command |
|---|---|
| Vitest | `npx vitest --coverage --reporter=verbose` |
| Jest | `npx jest --coverage --verbose` |
| react-scripts | `npx react-scripts test --coverage --watchAll=false` |
| Others | Use the project's configured coverage command |

Parse the output to identify:
- Files below 100% coverage
- Uncovered lines, branches, functions, and statements
- Specific line numbers that need tests
- Files with 0% coverage (no tests exist)

### Phase 6: Analyze and fill coverage gaps

For each file below 100% (including files from Phase 4 that now have tests but are not at 100%):

1. Read the source file and its test file side-by-side
2. Understand what the uncovered lines/functions/branches do
3. Identify which edge cases are missing from tests
4. Write targeted tests to cover the gaps:

- **Uncovered branches**: Test both `if` and `else` paths, ternary conditions
- **Uncovered functions**: Test all possible return paths
- **Uncovered lines**: Trace the logic and write a case that reaches them
- **Uncovered async paths**: Test both resolved and rejected Promise branches
- **Uncovered error handling**: Test error callbacks, catch blocks, error boundaries

Use the detected test runner's patterns:
- **Vitest**: `vi.mock()`, `vi.spyOn()`, `vi.fn()`
- **Jest**: `jest.mock()`, `jest.spyOn()`, `jest.fn()`
- **react-scripts**: Same as Jest

### Phase 7: Verify

Re-run coverage after adding/updating tests:

```bash
# Use the detected command
npx vitest --coverage   # for Vitest
npx jest --coverage      # for Jest
```

Confirm all files show 100% for statements, branches, functions, and lines.

### Phase 8: Repeat

If coverage is still below 100%, loop back to Phase 6.

---

## Configuration by runner

### Vitest coverage config

Ensure `vitest.config.ts`:
```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        '**/node_modules/**',
        '**/dist/**',
        '**/*.config.*',
        '**/*.d.ts',
      ],
    },
  },
})
```

### Jest coverage config

Ensure `jest.config.ts` contains:
```ts
export default {
  collectCoverage: true,
  coverageReporters: ['text', 'json', 'html'],
  collectCoverageFrom: [
    'src/**/*.{ts,tsx,js,jsx}',
    '!src/**/*.d.ts',
    '!src/**/*.test.*',
    '!src/**/*.spec.*',
  ],
}
```

---

## Tips

- Start with files that have existing tests but low coverage — quickest wins
- Then tackle untested files with the simplest logic first
- For React components: test rendering, user interactions, conditional rendering, error states, and empty/loading states
- For utility functions: test edge cases (empty strings, null/undefined, boundary values, special characters)
- For API services: test success response, error response, loading state, timeout, and network failure
- If dead code is found (impossible to cover), suggest removing it or adding a coverage ignore comment:
  - Vitest/V8: `/* c8 ignore next */` or `/* c8 ignore next 3 */`
  - Jest/Istanbul: `/* istanbul ignore next */` or `/* istanbul ignore if */`
- Do NOT modify source code logic — only add or update tests
- Follow the project's existing test patterns (file naming, folder structure, mocking style)
- For files with 0 tests, create a comprehensive test file that hits all the major paths first, then iterate

## Detection priority

When scanning for source files, check in this order of common directory structures:
1. `src/`
2. `app/` (Next.js app directory)
3. `components/`
4. `lib/`
5. `utils/`
6. `server/` (Next.js/Node.js)
7. `pages/` (Next.js pages directory)
8. Root-level `*.ts` / `*.tsx` files

## Example

User: "coverage report shows login.ts at 72%, fix it"

Agent actions:
1. Run `npx vitest --coverage --reporter=verbose`, see login.ts has uncovered lines 45-52 (error branch of `validateEmail`)
2. Read source, understand `validateEmail` throws on invalid emails
3. Add test cases for invalid email inputs in `login.test.ts`
4. Re-run coverage — login.ts now at 100%

User: "we use jest, check coverage"
1. Run `npx jest --coverage --verbose`, identify uncovered files
2. Find `src/components/Header.tsx` at 0% — no test exists
3. Read Header.tsx source, create `src/components/Header.test.tsx`
4. Run coverage again — Header.tsx now at 100%

User: "there's no test setup in this project, add coverage"
1. Detect no test runner in package.json
2. Ask user: "No test framework detected. Install Vitest? (Recommended)"
3. User confirms → install vitest + @vitest/coverage-v8, create vitest.config.ts
4. Scan src/, find 8 source files, 0 test files
5. Write tests for all 8 files
6. Run coverage — all at 100%

**Step 3: the first page.** Two files to replace.

**1. `frontend/src/index.css`:** replace everything with:

```css
:root {
  --fnb-blue: #003876;
  --fnb-red: #a80533;
  --fnb-white: #ffffff;
  --fnb-blue-light: #e8eef6;
  --fnb-border: #d6dde6;
  --fnb-text: #1a2333;
  --pass: #1e7b45;

  font-family: "Segoe UI", system-ui, sans-serif;
  color: var(--fnb-text);
  background: #f5f7fa;
}

* { box-sizing: border-box; }

body { margin: 0; }

.header {
  background: var(--fnb-blue);
  color: var(--fnb-white);
  padding: 16px 32px;
  border-bottom: 4px solid var(--fnb-red);
  font-size: 20px;
  font-weight: 600;
}

.page {
  max-width: 720px;
  margin: 32px auto;
  padding: 0 16px;
}

.card {
  background: var(--fnb-white);
  border: 1px solid var(--fnb-border);
  border-radius: 8px;
  padding: 24px;
  margin-bottom: 16px;
}

.card h2 {
  margin: 0 0 16px;
  font-size: 16px;
  color: var(--fnb-blue);
}

.row { display: flex; gap: 12px; }

input {
  flex: 1;
  padding: 10px 12px;
  border: 1px solid var(--fnb-border);
  border-radius: 6px;
  font-size: 14px;
}

input:focus {
  outline: 2px solid var(--fnb-blue);
  border-color: var(--fnb-blue);
}

button {
  background: var(--fnb-blue);
  color: var(--fnb-white);
  border: none;
  border-radius: 6px;
  padding: 10px 20px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
}

button:disabled { opacity: 0.6; cursor: wait; }

.badge {
  display: inline-block;
  padding: 4px 12px;
  border-radius: 999px;
  color: var(--fnb-white);
  font-weight: 600;
  font-size: 13px;
}

.badge.pass { background: var(--pass); }
.badge.fail { background: var(--fnb-red); }

.details {
  margin-top: 16px;
  font-size: 14px;
  line-height: 1.8;
}

.details code { word-break: break-all; }

.error { color: var(--fnb-red); }

.muted { color: #5b6677; font-size: 14px; }
```

**2. `frontend/src/App.tsx`:** replace everything with:

```tsx
import { useState } from 'react'

type RunResult = {
  tag: string
  exitCode: number
  passed: boolean
  durationSeconds: number
  logFile: string
}

export default function App() {
  const [tag, setTag] = useState('@ui')
  const [running, setRunning] = useState(false)
  const [result, setResult] = useState<RunResult | null>(null)
  const [error, setError] = useState<string | null>(null)

  // Sends POST /api/runs?tag=... to the backend
  // (Vite forwards it to port 8000) and waits
  // for the run to finish.
  async function runTest() {
    setRunning(true)
    setResult(null)
    setError(null)
    try {
      const res = await fetch(
        `/api/runs?tag=${encodeURIComponent(tag)}`,
        { method: 'POST' })

      if (!res.ok) {
        setError(await res.text())
        return
      }
      setResult(await res.json())
    } catch {
      setError('Could not reach the backend. ' +
        'Is BackendApplication running?')
    } finally {
      setRunning(false)
    }
  }

  return (
    <>
      <div className="header">PTAF Runner</div>

      <div className="page">
        <div className="card">
          <h2>Run a test</h2>
          <div className="row">
            <input
              value={tag}
              onChange={e => setTag(e.target.value)}
              placeholder="@RT_51-53"
              disabled={running}
            />
            <button onClick={runTest} disabled={running}>
              {running ? 'Running…' : 'Run'}
            </button>
          </div>
          {running && (
            <p className="muted">
              Test is running. This can take a few minutes.
            </p>
          )}
          {error && <p className="error">{error}</p>}
        </div>

        {result && (
          <div className="card">
            <h2>Result</h2>
            <span className={`badge ${result.passed ? 'pass' : 'fail'}`}>
              {result.passed ? 'PASSED' : 'FAILED'}
            </span>
            <div className="details">
              <div>Tag: <b>{result.tag}</b></div>
              <div>Exit code: {result.exitCode}</div>
              <div>Duration: {result.durationSeconds}s</div>
              <div>Log: <code>{result.logFile}</code></div>
            </div>
          </div>
        )}
      </div>
    </>
  )
}
```

**Then:**
1. Delete **`frontend/src/App.css`**. We're not using it.
2. Make sure **`BackendApplication`** is running in IntelliJ.
3. Open **http://localhost:5173**. You should see a blue header with a red line, a tag box with `@ui`, and a Run button.
4. Click **Run**. After a few minutes you'll get a red **FAILED**, since Connects is down tonight, along with the log path.

Send me a screenshot of the page.

**1. Backend: allow only the UI.** In IntelliJ, right-click `com.ptaf.backend`, choose **New → Java Class**, name it `WebConfig`, and paste:

```java
package com.ptaf.backend;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

// Lets the UI page at localhost:5173 call
// this backend. No other website can.
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:5173")
                .allowedMethods("GET", "POST");
    }
}
```

Then restart `BackendApplication`: stop it, then ▶ again.

**2. Frontend: call the backend directly.** In VS Code, open `frontend/src/App.tsx` and replace the `fetch` line with:

```tsx
const res = await fetch(
  `http://127.0.0.1:8000/api/runs?tag=${encodeURIComponent(tag)}`,
  { method: 'POST' })
```

**3. Remove the forwarding.** Replace everything in `frontend/vite.config.ts` with:

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})
```

**4. Restart and test.** In the VS Code terminal, press **Ctrl+C**, then run:

```powershell
npm run dev
```

Open **http://localhost:5173**, check the tag box says `@ui`, and click **Run** once. Send me a screenshot of the result.

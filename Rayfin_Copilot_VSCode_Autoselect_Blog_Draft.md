# Building a Fully Functional App with Rayfin + GitHub Copilot in VS Code (Auto Model Select)

## How I shipped a working, authenticated app faster than my old setup

## Executive Summary
I wanted to test whether a modern AI-assisted workflow could take me from zero to a real, running application without the traditional setup grind. My goals were simple:

- Build a real app, not a demo toy.
- Use Rayfin to handle backend concerns fast.
- Use GitHub Copilot in VS Code as the primary implementation partner.
- Keep local development practical with Docker and predictable scripts.
- Let VS Code + Copilot Auto model selection pick the best LLM path in context, instead of me manually model-switching.

The result was a fully functional Todo application with:

- User authentication (email/password in local mode).
- Data model and data API.
- Local backend services bootstrapped with Rayfin dev workflow.
- React frontend with route guards, auth context, and CRUD operations.
- End-to-end workflow completed largely through guided prompts and automated command execution.

This article documents exactly how the app came together, where automation paid off, and what practical patterns made the build reliable.

---

## Why this stack worked well together

Three pieces made this workflow unusually productive:

1. Rayfin CLI gave me a backend-first accelerator with opinionated structure.
2. GitHub Copilot in VS Code handled sequencing, troubleshooting, and code generation loops.
3. VS Code Auto-select model behavior removed decision friction around “which model should I use right now?” and helped keep momentum.

Instead of spending time on blank-project overhead, I spent time validating behavior.

---

## What “Auto model selection” changed in practice

A lot of developers underestimate how much cognitive drag comes from choosing tools repeatedly. In this workflow, I focused on intent:

- “Set up Rayfin locally.”
- “Move this project to another folder.”
- “Start the local stack and diagnose startup failures.”
- “Fix why the page is blank.”
- “Validate auth and CRUD flow in the browser.”

Copilot handled iterative execution and the environment-specific complexity. That meant fewer interruptions to think about orchestration and more time validating outcome.

### Screenshot Placeholder 1 (non-code)
**Insert screenshot:** VS Code Chat showing Auto model selection and a task prompt like “set up Rayfin local template and run dev:local”.

---

## Environment and prerequisites

Before app build, I confirmed the local runtime prerequisites:

- Node.js installed (v22.x in my setup).
- Docker Desktop installed and running.
- VS Code + GitHub Copilot enabled.
- Internet access for template clone and package installation.

Rayfin local mode in this template is Docker-backed, so Docker readiness was not optional.

### Code Snippet Placeholder 1
**Insert snippet:** Node version check.

```powershell
node --version
```

### Code Snippet Placeholder 2
**Insert snippet:** Docker runtime status check.

```powershell
docker info
```

---

## Step 1: Template-driven start instead of project-from-scratch

I chose the Todo local experimental template because it gave a practical baseline with auth + data + local backend wiring. The key productivity win here was avoiding manual backend stitching.

### Code Snippet Placeholder 3
**Insert snippet:** Clone command and target folder intent.

```powershell
git clone https://github.com/microsoft/awesome-rayfin.git rayfin
```

### Code Snippet Placeholder 4
**Insert snippet:** Dependency install in the selected template directory.

```powershell
npm install
```

### Screenshot Placeholder 2 (non-code)
**Insert screenshot:** File explorer view in VS Code showing project folder creation and template layout under rayfin/templates/todo-local.

---

## Step 2: Understanding the Rayfin local workflow scripts

One of the most useful things in this template is how scripts encode operational behavior. The following script definitions are the backbone of local lifecycle control:

### Code Snippet Placeholder 5
**Insert snippet:** package scripts section showing local dev and lifecycle commands.

```json
"scripts": {
  "predev": "rayfin env --framework vite",
  "dev": "rayfin up --exclude-services staticHosting && vite",
  "dev:local": "npm run predev && npm run rayfin:dev && vite",
  "dev:local:stop": "cross-env RAYFIN_FEATURE_FLAGS=docker-local-dev RAYFIN_WEBSERVICE_IMAGE_NAME=ghcr.io/microsoft/rayfin/webservice:latest rayfin dev --stop",
  "dev:local:down": "cross-env RAYFIN_FEATURE_FLAGS=docker-local-dev RAYFIN_WEBSERVICE_IMAGE_NAME=ghcr.io/microsoft/rayfin/webservice:latest rayfin dev --down",
  "dev:local:purge": "cross-env RAYFIN_FEATURE_FLAGS=docker-local-dev RAYFIN_WEBSERVICE_IMAGE_NAME=ghcr.io/microsoft/rayfin/webservice:latest rayfin dev --purge",
  "rayfin:dev": "cross-env RAYFIN_FEATURE_FLAGS=docker-local-dev RAYFIN_WEBSERVICE_IMAGE_NAME=ghcr.io/microsoft/rayfin/webservice:latest rayfin dev",
  "rayfin:db": "cross-env RAYFIN_FEATURE_FLAGS=docker-local-dev RAYFIN_WEBSERVICE_IMAGE_NAME=ghcr.io/microsoft/rayfin/webservice:latest rayfin dev db apply"
}
```

These scripts are not “nice to have”; they’re reproducibility anchors.

---

## Step 3: Running local mode and diagnosing startup blockers

The correct command for this template in full local mode was:

### Code Snippet Placeholder 6
**Insert snippet:** Local dev startup command.

```powershell
npm run dev:local
```

### First blocker: Docker not running
I hit this early and got a direct runtime error indicating Docker daemon availability issues. Starting Docker Desktop resolved this blocker.

### Second blocker: stale container naming conflicts
Because of repeated startup attempts and path moves, existing container names conflicted with new Compose runs. Cleanup solved it:

### Code Snippet Placeholder 7
**Insert snippet:** Stale container and network cleanup commands.

```powershell
docker rm -f todo-local-experimental-webservice-1 todo-local-experimental-sqlserver-1 todo-local-experimental-admin-db-1 todo-local-experimental-aspire-dashboard-1
docker network rm todo-local-experimental_default
```

After cleanup, startup proceeded correctly.

### Code Snippet Placeholder 8
**Insert snippet:** Validate running local services.

```powershell
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### Screenshot Placeholder 3 (non-code)
**Insert screenshot:** Terminal output showing successful service startup (webservice, sqlserver, admin-db, dashboard) and local URLs.

---

## Step 4: Backend config that made auth/data immediately usable

Rayfin service configuration provided out-of-the-box auth + data service definitions and local redirect safety.

### Code Snippet Placeholder 9
**Insert snippet:** rayfin.yml service configuration excerpt.

```yaml
services:
  auth:
    enabled: true
    fabric:
      enabled: true
    password:
      enabled: true
    allowedRedirectUris:
      - http://localhost:5173
  data:
    enabled: true
    dialect: mssql
  staticHosting:
    enabled: true
```

The key here is dual-mode auth support (Fabric + password) with local redirect URI already handled.

---

## Step 5: Frontend auth mode bootstrap logic

A practical design decision in this app was automatic mode selection based on API host:

- Localhost backend => password mode.
- Non-local backend => Fabric mode.

### Code Snippet Placeholder 10
**Insert snippet:** Auth bootstrap decision logic.

```ts
const apiUrl = import.meta.env.VITE_RAYFIN_API_URL || 'http://localhost:5168';
const apiHost = new URL(apiUrl).hostname;
const useFabric = apiHost !== 'localhost' && apiHost !== '127.0.0.1';

if (useFabric) {
  return new RayfinAuthService(client, {
    mode: 'fabric',
    fabricOptions: {
      workspaceId: workspaceId ?? '',
      projectId: projectId ?? '',
      fabricPortalUrl: fabricPortalUrl ?? '',
      returnOrigin: window.location.origin,
    },
  });
}

return new RayfinAuthService(client, { mode: 'password' });
```

This is an elegant portability pattern: one app, two environments, minimal branching in UI.

---

## Step 6: Route protection and session-aware UX

The app route architecture used an auth guard to ensure protected pages are not reachable without session state.

### Code Snippet Placeholder 11
**Insert snippet:** AuthGuard route control.

```tsx
if (requireAuth && !isAuthenticated) return <Navigate to="/auth" replace />;
if (!requireAuth && isAuthenticated) return <Navigate to="/" replace />;
```

The app also included session-expiry handling and user polling in auth context, reducing silent-failure risk in long sessions.

### Code Snippet Placeholder 12
**Insert snippet:** Session validity polling pattern.

```ts
useEffect(() => {
  if (!user) return;

  const interval = setInterval(async () => {
    try {
      const current = await authService.getCurrentUser();
      if (!current) {
        handleSessionExpired();
      }
    } catch {
      handleSessionExpired();
    }
  }, 5_000);

  return () => clearInterval(interval);
}, [user, authService, handleSessionExpired]);
```

---

## Step 7: The “blank page” incident and what it taught me

At one point, the app loaded but rendered blank. Root cause was structural, not visual:

- index.html referenced /src/main.tsx.
- The target folder was missing the entire src directory.
- Vite served shell HTML but had no React app entry.

This was a great reminder that AI-assisted workflows still need sanity checks on project structure after file moves.

I restored the missing source tree and immediately recovered the UI.

### Screenshot Placeholder 4 (non-code)
**Insert screenshot:** Before/after comparison:
- Before: blank page with console errors.
- After: auth screen and then todo list loaded after account creation.

---

## Step 8: Validating functional completeness

I validated practical end-to-end behavior, not just build success:

1. Opened local app.
2. Switched to Create account mode.
3. Registered with valid email/password.
4. Redirected to authenticated home.
5. Added todo items.
6. Toggled completion state.
7. Deleted items.
8. Confirmed session identity visible in header.

This is where the stack proved itself: auth + data + UI + local runtime all operated as one coherent loop.

---

## Why GitHub Copilot + Rayfin CLI is a strong pairing

### 1) Setup automation + recovery loops
Copilot wasn’t just generating snippets; it orchestrated:

- environment checks,
- path and folder corrections,
- dependency repair,
- startup retries,
- and diagnostics from terminal/browser telemetry.

### 2) Operational resilience
The workflow handled real-world failures:

- daemon not running,
- stale container naming collisions,
- partial project copy,
- missing source tree,
- auth flow confusion.

The ability to iterate quickly through those failures is where productivity gains became tangible.

### 3) Better focus through Auto model selection
Not manually selecting models for each micro-task reduced friction. I stayed focused on building, validating, and documenting outcomes.

---

## Suggested visual asset plan for publication

Use this distribution for your article visuals:

- 4 non-code screenshots:
  - Copilot prompt/workflow context.
  - Folder/template setup.
  - Local services startup success.
  - Before/after blank page fix.

- 10–12 code snippets:
  - Commands, scripts, config, auth bootstrap, route guard, session handling, and CRUD service calls.

This gives enough visual rhythm for a 7–8 page article without overwhelming readers.

---

## Draft sectioning for 7–8 page layout

Use this near-final structure in your publishing platform:

1. Why I chose Rayfin + Copilot + VS Code Auto Select
2. Environment and prerequisites
3. Template selection and project creation workflow
4. Local development startup and troubleshooting
5. Backend service configuration with Rayfin
6. Frontend architecture: auth modes, routing, and session handling
7. Blank page incident and structural debugging lessons
8. End-to-end validation and outcomes
9. What I’d recommend to teams adopting this stack
10. Conclusion: faster path from idea to production-ready patterns

---

## Practical writing notes (for your final polish)

- Keep tone as “experienced practitioner,” not product marketing.
- Include real errors and how you resolved them; this builds trust.
- Show where automation helped, and where you still had to make decisions.
- Explicitly call out reproducibility commands and cleanup commands.
- End with a checklist readers can apply in their own setup.

---

## Copy-ready conclusion

This project changed how I think about app bootstrap speed. Rayfin gave me backend capabilities that are usually expensive in setup time. GitHub Copilot in VS Code turned environment handling, troubleshooting, and code generation into a continuous build loop instead of disconnected tasks. Auto model selection reduced one more layer of friction: I could focus on outcomes while the assistant adapted to task complexity.

Most importantly, this was not just “AI wrote some code.” It was a full workflow where architecture, runtime, data, auth, and debugging all converged into a running app. If your goal is to move from concept to validated product behavior quickly, Rayfin CLI + GitHub Copilot + VS Code is one of the most practical combinations I’ve used.

---

## Appendix A: Optional command checklist for readers

```powershell
# 1. Clone templates
git clone https://github.com/microsoft/awesome-rayfin.git rayfin

# 2. Install
cd rayfin/templates/todo-local
npm install

# 3. Start full local mode
npm run dev:local

# 4. Validate running containers
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# 5. If conflicts occur, clean stale resources
docker rm -f todo-local-experimental-webservice-1 todo-local-experimental-sqlserver-1 todo-local-experimental-admin-db-1 todo-local-experimental-aspire-dashboard-1
docker network rm todo-local-experimental_default

# 6. Re-run local mode
npm run dev:local
```

---

## Appendix B: Suggested caption text for screenshots

1. “Copilot in VS Code with Auto model selection driving setup and troubleshooting workflow.”
2. “Template-driven project bootstrap under Rayfin workspace structure.”
3. “Local Rayfin stack healthy in Docker: webservice, SQL Server, admin DB, and telemetry dashboard.”
4. “From blank page to authenticated app: root-cause diagnosis and fix.”

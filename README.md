Building a Fully Functional App with Rayfin + GitHub Copilot in VS Code

I wanted to test whether a modern Agent-assisted workflow could take me from zero to a real, running application without the traditional setup grind. My goals were:

-        Build a real app, not a demo toy

-        Use Rayfin to handle backend concerns fast

-        Use GitHub Copilot in VS Code as the primary implementation partner

-        Keep local development practical with Docker and predictable scripts

-        Let VS Code + Copilot Auto model selection pick the best LLM path in context, instead of me manually model-switching


The result was a fully functional Todo application with:

 -        User authentication (email/password in local mode)

-        Data model and data API

-        Local backend services bootstrapped with Rayfin dev workflow

-        React frontend with route guards, auth context, and CRUD operations

-        End-to-end workflow completed largely through guided prompts and automated command execution

 This article documents exactly how the app came together, where automation paid off, and what practical patterns made the build reliable.

1.      Rayfin CLI gave me a backend-first accelerator with opinionated structure.

2.      GitHub Copilot in VS Code handled sequencing, troubleshooting, and code generation loops.

3.      VS Code Auto-select model behavior removed decision friction around “which model should I use right now?” and helped keep momentum.

Instead of spending time on blank-project overhead, I spent time validating behavior.

Set up Rayfin local template and run dev:local
A lot of developers underestimate how much cognitive drag comes from choosing tools repeatedly. In this workflow, I focused on intent:

-        Set up Rayfin locally

-        Start the local stack and diagnose startup failures

-        Validate auth and CRUD flow in the browser

Copilot handled iterative execution and the environment-specific complexity. That meant fewer interruptions to think about orchestration and more time validating outcome. I would give prompts from time to time, Allow all permissions for the session and stare at the steps that Copilot would take in the Copilot Chat windowpane… it’s always so cool to watch.
 

Before app build, I confirmed the local runtime prerequisites:

-        Node.js installed (v22.x in my setup).

-        Docker Desktop installed and running.

-        VS Code + GitHub Copilot enabled.

-        Internet access for template clone and package installation.

Rayfin local mode in this template is Docker-backed, so Docker readiness was not optional.

node --version
v22.12.0

Step 1: Template-driven start instead of project-from-scratch
I chose the Todo local experimental template because it gave a practical baseline with auth + data + local backend wiring. The key productivity win here was avoiding manual backend stitchi

Step 2: Understanding the Rayfin local workflow scripts
One of the most useful things in this template is how scripts encode operational behavior. The following script definitions are the backbone of local lifecycle control:

Local dev and lifecycle commands:

These scripts are not “nice to have”... they’re reproducibility anchors.

Step 3: Running local mode and diagnosing startup blockers
The correct command for this template in full local mode was:

npm run dev:local

First blocker: Docker not running
I hit this early and got a direct runtime error indicating Docker daemon availability issues. Starting Docker Desktop resolved this blocker.

Second blocker: stale container naming conflicts
Because of repeated startup attempts and path moves, existing container names conflicted with new Compose runs. Cleanup solved it: Stale container and network cleanup commands:

docker rm -f todo-local-experimental-webservice-1 todo-local-experimental-
sqlserver-1 todo-local-experimental-admin-db-1 todo-local-experimental-
aspire-dashboard-1

docker network rm todo-local-experimental_default
After cleanup, startup proceeded correctly. I then validated the running local services:

docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
Terminal output showing successful service startup (webservice, sqlserver, admin-db, dashboard) and local URLs.

Step 4: Backend config that made auth/data immediately usable
Rayfin service configuration provided out-of-the-box auth + data service definitions and local redirect safety.

rayfin.yml service configuration excerpt:

The key here is dual-mode auth support (Fabric + password) with local redirect URI already handled.

This is the part of the stack that made the app feel real instead of staged. Rayfin did not just scaffold a UI shell; it also gave me the backend wiring needed for identity, data access, and local persistence.

What I would highlight in the article is the generated backend surface, not raw infrastructure noise:

-        Authentication support for both local password mode and Fabric mode.

-        A data service backed by MSSQL in local mode.

-        A clean client-side API layer for Todo CRUD operations.

-        Docker-backed supporting services that prove the backend is actually running.

 The important editorial point is that the data model and service layer were usable immediately. I did not have to hand-build a controller stack, database bootstrap flow, or auth bridge just to get to the first working record.

Todo data service excerpt showing the generated CRUD boundary:

Step 5: Frontend auth mode bootstrap logic
A practical design decision in this app was automatic mode selection based on API host:

 ·        Localhost backend => password mode.

·        Non-local backend => Fabric mode.

 Auth bootstrap decision logic:

This is an elegant portability pattern: one app, two environments, minimal branching in UI.

Step 6: Route protection and session-aware UX
The app route architecture used an auth guard to ensure protected pages are not reachable without session state.

AuthGuard route control:

The app also included session-expiry handling and user polling in auth context, reducing silent-failure risk in long sessions.

Session validity polling pattern:

Step 7: The “blank page” incident and what it taught me
 

At one point, the app loaded but rendered blank. Root cause was structural, not visual:

·        index.html referenced /src/main.tsx.

·        The target folder was missing the entire src directory.

·        Vite served shell HTML but had no React app entry.

This was a great reminder that AI-assisted workflows still need sanity checks on project structure after file moves.

I restored the missing source tree and immediately recovered the UI.

Step 8: Validating functional completeness
I validated practical end-to-end behavior, not just build success:

1.      Opened local app.

2.      Switched to Create account mode.

3.      Registered with valid email/password.

4.      Redirected to authenticated home.

5.      Added todo items.

6.      Toggled completion state.

7.      Deleted items.

8.      Confirmed session identity visible in header.

This is where the stack proved itself: auth + data + UI + local runtime all operated as one coherent loop.

GitHub Copilot + Rayfin CLI pairing
1. Setup automation + recovery loops
Copilot wasn’t just generating snippets; it orchestrated:

·        environment checks,

·        path and folder corrections,

·        dependency repair,

·        startup retries,

·        and diagnostics from terminal/browser telemetry.

2. Operational resilience
The workflow handled real-world failures:

·        daemon not running,

·        stale container naming collisions,

·        partial project copy,

·        missing source tree,

·        auth flow confusion.

The ability to iterate quickly through those failures is where productivity gains became tangible.

3. Better focus through Auto model selection
Not manually selecting models for each micro-task reduced friction. I stayed focused on building, validating, and documenting outcomes.

Most importantly, this was not just “AI wrote some code.” It was a full workflow where architecture, runtime, data, auth, and debugging all converged into a running app. If your goal is to move from concept to validated product behavior quickly, Rayfin CLI + GitHub Copilot + VS Code is one of the most practical combinations I’ve used.

Code references… just the primary ones I thought are key from my project
Clone templates

git clone https://github.com/microsoft/awesome-rayfin.git rayfin
Install

cd rayfin/templates/todo-local
npm install
Start full local mode

npm run dev:local
Validate running containers

docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
If conflicts occur, clean stale resources

docker rm -f todo-local-experimental-webservice-1 todo-local-experimental-sqlserver-1 todo-local-experimental-admin-db-1 todo-local-experimental-aspire-dashboard-1

docker network rm todo-local-experimental_default Re-run local mode npm run dev:local
Re-run local mode

npm run dev:local
My conclusions
This project changed how I think about app bootstrap speed. Rayfin gave me backend capabilities that are usually expensive in setup time. GitHub Copilot in VS Code turned environment handling, troubleshooting, and code generation into a continuous build loop instead of disconnected tasks. Auto model selection reduced one more layer of friction: I could focus on outcomes while the assistant adapted to task complexity.

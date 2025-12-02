# Computer Use Preview – QA PoC Guide

This guide explains how to set up the `computer-use-preview` project and how to use it for QA proof‑of‑concept experiments, including:

1. Giving the agent a test objective and verifying it executes the test.
2. Giving the agent a test objective and validating that it automates the test (even if locators are flaky).
3. Providing a test with wrong or outdated steps and asking the agent to correct/repair it.
4. Giving the agent a URL and asking it to generate documentation for the site.

---

## 1. Setup and Configuration

### 1.1. Create and activate a virtual environment

From the project root directory (`computer-use-preview`):

```powershell
python -m venv .venv
. .venv\Scripts\Activate.ps1
```

You should now see `(.venv)` in your PowerShell prompt.

### 1.2. Install Python dependencies

```powershell
pip install -r requirements.txt
```

This installs `google-genai`, `playwright`, `browserbase`, and other required packages.

### 1.3. Install Playwright browser (Windows)

On Windows you only need to install the browser itself:

```powershell
playwright install chrome
```

The `playwright install-deps chrome` step from the README is only needed on Linux.

### 1.4. Configure the Gemini API key

The agent reads `GEMINI_API_KEY` from the environment when constructing the `genai.Client`:

```python
self._client = genai.Client(
    api_key=os.environ.get("GEMINI_API_KEY"),
    vertexai=os.environ.get("USE_VERTEXAI", "0").lower() in ["true", "1"],
    project=os.environ.get("VERTEXAI_PROJECT"),
    location=os.environ.get("VERTEXAI_LOCATION"),
)
```

You must set `GEMINI_API_KEY` before running the agent.

#### Option A – Set per PowerShell session

```powershell
$env:GEMINI_API_KEY = "YOUR_REAL_GEMINI_API_KEY"
```

This applies only to the current PowerShell window.

#### Option B – Set in the virtual environment activation script

Append this line to `.venv\Scripts\Activate.ps1`:

```powershell
$env:GEMINI_API_KEY = "YOUR_REAL_GEMINI_API_KEY"
```

Then, each time you activate the environment with:

```powershell
. .venv\Scripts\Activate.ps1
```

`GEMINI_API_KEY` will be set automatically.

> Important: do **not** commit your real API key to source control.

#### Vertex AI (optional)

If you want to use Vertex AI instead of the Gemini Developer API, set:

```powershell
$env:USE_VERTEXAI = "true"
$env:VERTEXAI_PROJECT = "your-gcp-project-id"
$env:VERTEXAI_LOCATION = "your-vertex-region"
```

For the current PoC, using just `GEMINI_API_KEY` is sufficient.

### 1.5. Quick smoke test

With the virtualenv active and `GEMINI_API_KEY` set, run:

```powershell
python main.py --query "Open example.com and then finish when done." --env playwright
```

You should see a Chrome window open via Playwright and the console printing Gemini reasoning plus the function calls being executed.

---

## 2. How the Agent Works

High‑level flow:

- `main.py` is the CLI entrypoint.
- It parses arguments like `--query`, `--env`, `--initial_url`, `--model`.
- It creates a `PlaywrightComputer` or `BrowserbaseComputer` (the environment).
- It creates a `BrowserAgent` with your natural‑language `query`.
- `BrowserAgent.agent_loop()` repeatedly:
  - Sends screenshots and state to Gemini.
  - Receives tool calls (`open_web_browser`, `click_at`, `type_text_at`, `scroll_document`, `navigate`, etc.).
  - Executes them via Playwright or Browserbase.
  - Sends back the updated environment state and continues until there are no more tool calls.

Your entire test objective is encoded in the `--query` string.

---

## 3. Objective 1 – Give a Test / Objective and See if It Executes

To run a concrete end‑to‑end test, encode the manual test steps and success criteria directly in `--query`.

### 3.1. Example: Login + Navigate + Verify

```powershell
python main.py `
  --env playwright `
  --initial_url "https://your-app-under-test.example/login" `
  --query "You are executing an end-to-end QA test.
1) Starting from the current page, log in with username 'qa_user' and password 'qa_password'.
2) After successful login, navigate to the 'Orders' page.
3) Open the most recent order and verify that its status is 'Pending'.
4) If the status is 'Pending', say 'TEST PASSED' and explain what you did.
5) If anything fails, say 'TEST FAILED' and describe where and why."
```

What to expect:

- Playwright opens Chrome at the login page.
- The agent uses tool calls (click, type, navigate) to perform the steps.
- In the console, a rich table shows reasoning (left) and tool calls (right) for each iteration.
- When no more tool calls are needed, the agent prints final reasoning (your test result and explanation).

### 3.2. Tips for Well‑Defined Tests

- Provide:
  - A **clear goal**.
  - A **starting URL** (`--initial_url`).
  - Explicit **PASS/FAIL criteria** (e.g. "Say TEST PASSED/TEST FAILED at the end").
  - A request for a **short action summary**.
- For easier visual debugging, add `--highlight_mouse`:

```powershell
python main.py --env playwright --highlight_mouse `
  --initial_url "https://your-app-under-test.example" `
  --query "..."
```

---

## 4. Objective 2 – Verify That It Automates the Test

This framework does not expose XPath/CSS locators to you directly; Gemini chooses where to click/type based on screenshots and page structure. This is useful when you want to ignore locator flakiness and focus on whether the test flow itself is automated.

### 4.1. Turning a Manual Test into an Automated Objective

Suppose your manual test is:

1. Go to `/products`.
2. Filter by category "Laptops".
3. Open the first product in the result list.
4. Verify that a price is visible and formatted as currency.

You can express this as:

```powershell
python main.py `
  --env playwright `
  --initial_url "https://your-app-under-test.example/products" `
  --query "Execute this QA test end-to-end:
1) On the products page, filter the list by category 'Laptops'.
2) Open the first product in the filtered list.
3) Verify that a price is visible on the page and looks like a currency value (e.g. contains a currency symbol and digits).
4) At the end, explain whether the test passed or failed and why. Describe the key elements you interacted with (button labels, field labels, etc.)."
```

Ways to validate automation:

- **Visual**: Watch the browser to confirm it performs the correct sequence of actions.
- **Console**: Read reasoning and tool calls at each step.
- **Repeatability**: Re‑run the same command across sessions or machines and check for consistent behavior.

You can also:

- Give the agent a rough time/step budget: "Try for at most 5 minutes and then report your result."
- Ask for structured output: "Return the result as: RESULT: PASS or RESULT: FAIL, followed by a short summary."

---

## 5. Objective 3 – Provide Wrong or Outdated Steps and Ask It to Correct Them

Here, you intentionally give the agent instructions that are partially wrong (e.g. bad XPaths, missing or outdated steps) and ask it to still achieve the goal by repairing the test plan.

Although the tools don\'t execute XPaths directly, you can treat them as descriptive hints that may be inconsistent with the actual UI.

### 5.1. Pattern: "Some Steps Are Wrong – Fix and Complete the Goal"

Example:

```powershell
python main.py `
  --env playwright `
  --initial_url "https://your-app-under-test.example/login" `
  --query "You are executing a web QA test, but some steps are wrong.
Target goal: Successfully log in and reach the 'Dashboard' page.

Planned steps (may contain mistakes):
1) Enter username in the field with XPath //*[@id='username'].
2) Enter password in the field with XPath //*[@id='password'].
3) Click the login button using XPath //*[@id='login-wrong-id'].
4) Verify that the page title contains 'Dashboard'.

If any of these steps fail or correspond to non-existent elements, you MUST:
- Inspect the page visually.
- Infer the correct UI elements (e.g. by visible labels or surrounding text).
- Adjust the sequence of actions so that the final goal is still achieved.
- At the end, output:
  a) Whether the test PASSED or FAILED.
  b) The corrected step list you actually followed (numbered steps).
  c) A brief explanation of which original steps were wrong and how you fixed them."
```

In practice, the model will:

- Notice inconsistencies between your description and the visible UI.
- Use its own tool calls to click/type in the right places.
- Output a corrected step list and explanation, because you requested it.

### 5.2. Handling Missing Steps / New Flows

For UI changes that add new mandatory steps (e.g. a new consent modal), you can give an outdated flow and tell the agent to adapt:

- Provide the **older** step list in the query.
- State: "The actual UI may have extra mandatory steps. Discover and insert any missing steps so that the scenario still completes successfully. At the end, output the updated sequence." 

This evaluates whether the agent can:

- Detect additional dialogs/pages.
- Insert new actions.
- Still reach the high‑level goal.

---

## 6. Objective 4 – Generate Documentation for a Site from a URL

Here, the QA "test" is to explore the application and write structured documentation.

### 6.1. Single‑URL Documentation Generation

Example:

```powershell
python main.py `
  --env playwright `
  --initial_url "https://your-target-site.example" `
  --query "You are a QA engineer documenting this web application.
Starting from the current page, systematically explore the site by:
- Opening the main navigation menus and sub-menus.
- Visiting important pages (dashboards, forms, settings, reports, etc.).
- Scanning for buttons and forms that represent key user flows.

Spend a reasonable number of steps exploring (but avoid infinite loops). Then stop and output a structured documentation in Markdown with these sections:
1. Overview of the application.
2. Navigation structure (top-level menus and key sub-pages).
3. Main user workflows (step-by-step for each important task).
4. Important forms and validations.
5. Observed error messages or edge cases.

When you finish exploring and documenting, say 'DOCUMENTATION COMPLETE'."
```

The agent will:

- Navigate menus and links.
- Build an internal understanding of the app.
- Eventually stop issuing tool calls and output the requested documentation.

### 6.2. Deeper Documentation Runs

To cover more of a complex app, you can run multiple documentation commands that target different areas:

- Public site or marketing pages.
- Authenticated user flows (starting from a logged‑in dashboard).
- Admin or back‑office sections.

Each run can:

- Start at a different `--initial_url`.
- Use a tailored query that constrains the area of focus.

---

## 7. Summary of Workflow for QA PoC

1. **One‑time setup**
   - Create virtualenv, install dependencies, install Playwright Chrome, configure `GEMINI_API_KEY`.
2. **Per‑scenario config**
   - Decide starting URL (`--initial_url`) and environment (`--env playwright` or `browserbase`).
3. **Encode the test**
   - Convert your manual test (or Gherkin scenario) into a precise natural‑language `--query`:
     - High‑level goal.
     - Nominal steps.
     - PASS/FAIL criteria.
     - Expected output format (e.g. "TEST PASSED/FAILED", corrected steps, documentation sections).
4. **Run and observe**
   - Run `python main.py ...`, watch the browser actions and console reasoning.
5. **Iterate**
   - Refine prompts for stability, robustness to UI changes, and richer reporting.

This gives you a repeatable framework for evaluating how well the Gemini computer‑use agent can execute, adapt, and document complex web‑based QA tasks.
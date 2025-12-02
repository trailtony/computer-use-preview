# DemoBlaze.com Agent Testing Instructions

This document provides step-by-step instructions for running end-to-end tests on https://www.demoblaze.com/ using the computer-use agent. These tests validate all four objectives from the QA PoC guide.

---

## Prerequisites

Before running any tests, ensure:

1. Virtual environment is activated:
   ```powershell
   . .venv\Scripts\Activate.ps1
   ```

2. `GEMINI_API_KEY` is configured:
   ```powershell
   $env:GEMINI_API_KEY = "YOUR_REAL_GEMINI_API_KEY"
   ```

3. Dependencies are installed and Playwright Chrome browser is ready.

---

## Phase 1: Environment Setup

### Step 1.1: Verify Setup

Run a quick smoke test to ensure everything is working:

```powershell
python main.py --query "Open example.com and then finish when done." --env playwright
```

**Expected Result:** Chrome browser opens, navigates to example.com, and closes successfully.

---

## Phase 2: Basic Test Execution (Objective 1)

These tests validate that the agent can execute predefined test scenarios with clear pass/fail criteria.

### Step 2.1: User Registration and Login Flow

```powershell
python main.py --env playwright --highlight_mouse --initial_url "https://www.demoblaze.com/" --query "Execute this end-to-end QA test:
1) Click the 'Sign up' button in the navigation.
2) Register a new user with username 'testuser_[random_5_digits]' and password 'TestPass123'.
3) After successful registration, click 'Log in' in the navigation.
4) Log in with the same credentials you just created.
5) Verify that you are logged in by checking if 'Welcome testuser_[your_username]' appears in the navigation and a 'Log out' button is visible.
6) At the end, say 'TEST PASSED' if all steps completed successfully, or 'TEST FAILED' with an explanation of what went wrong."
```

**What to Observe:**
- Agent clicks Sign up button
- Fills in registration form with generated username
- Successfully registers and logs in
- Verifies authentication elements

**Expected Result:** `TEST PASSED` with confirmation of successful login.

---

### Step 2.2: Product Browsing Test

```powershell
python main.py --env playwright --highlight_mouse --initial_url "https://www.demoblaze.com/" --query "Execute this QA test:
1) Click on the 'Laptops' category in the sidebar.
2) Wait for the products to load and verify that laptop products are displayed.
3) Click on the first laptop product to open its details page.
4) Verify that the product detail page shows: product name, price, description, and an 'Add to cart' button.
5) Return to the home page by clicking 'Home'.
6) Repeat steps 1-4 for the 'Phones' category.
7) Say 'TEST PASSED' if all elements were found and navigation worked correctly, or 'TEST FAILED' with details of any issues."
```

**What to Observe:**
- Agent navigates between categories
- Views product detail pages
- Verifies presence of key product information

**Expected Result:** `TEST PASSED` confirming all product elements are visible.

---

### Step 2.3: Shopping Cart Functionality Test

```powershell
python main.py --env playwright --highlight_mouse --initial_url "https://www.demoblaze.com/" --query "Execute this QA test for shopping cart:
1) Navigate to the 'Phones' category.
2) Click on the first phone product.
3) Click the 'Add to cart' button.
4) Accept any alert that appears confirming the product was added.
5) Click 'Cart' in the navigation menu.
6) Verify that the cart page shows the product you added with its name and price.
7) Note the product price and verify it's a valid currency format.
8) Say 'TEST PASSED: Cart contains [product_name] with price [price]' or 'TEST FAILED' with explanation."
```

**What to Observe:**
- Agent adds product to cart
- Handles alert dialog
- Navigates to cart and verifies contents

**Expected Result:** `TEST PASSED: Cart contains [product] with price [amount]`

---

## Phase 3: Advanced Automation (Objective 2)

These tests confirm the agent can automate complex workflows without relying on specific locators.

### Step 3.1: Complete Purchase Flow

```powershell
python main.py --env playwright --highlight_mouse --initial_url "https://www.demoblaze.com/" --query "Automate this complete purchase flow end-to-end:
1) Browse to any product category of your choice.
2) Select and add two different products to the cart.
3) Navigate to the Cart page.
4) Click 'Place Order' button.
5) Fill out the order form with test data: Name='John Doe', Country='USA', City='New York', Credit card='4111111111111111', Month='12', Year='2025'.
6) Click 'Purchase' to complete the order.
7) Verify that a success confirmation appears with order details.
8) Capture the order ID from the confirmation.
9) Click 'OK' to close the confirmation.
10) At the end, report 'TEST PASSED: Order ID [order_id] completed successfully' or 'TEST FAILED' with explanation. Describe the key UI elements you interacted with."
```

**What to Observe:**
- Agent autonomously completes entire purchase workflow
- Handles multiple modals and forms
- Extracts order confirmation details

**Expected Result:** `TEST PASSED: Order ID [number] completed successfully`

---

### Step 3.2: Multi-Category Product Comparison

```powershell
python main.py --env playwright --highlight_mouse --initial_url "https://www.demoblaze.com/" --query "Automate this product comparison test:
1) Visit the 'Laptops' category and collect information about the first two laptop products: names and prices.
2) Visit the 'Phones' category and collect information about the first two phone products: names and prices.
3) Visit the 'Monitors' category and collect information about the first two monitor products: names and prices.
4) Compare prices and identify the most expensive product overall and the least expensive product overall.
5) At the end, output a summary in this format:
   - Most expensive: [category] - [product_name] at [price]
   - Least expensive: [category] - [product_name] at [price]
   - Total products examined: 6
6) Say 'TEST PASSED' if all data was collected successfully."
```

**What to Observe:**
- Agent systematically collects data across categories
- Performs comparison logic
- Provides structured output

**Expected Result:** `TEST PASSED` with price comparison summary

---

## Phase 4: Error Correction Testing (Objective 3)

These tests provide intentionally flawed instructions to validate the agent's ability to self-correct.

### Step 4.1: Flawed Login Test

```powershell
python main.py --env playwright --highlight_mouse --initial_url "https://www.demoblaze.com/" --query "You are executing a login test, but the steps contain errors. Your goal is to successfully log in.

Planned steps (contain mistakes):
1) Enter username in the field with ID 'user-input-field'.
2) Enter password in the field with ID 'pass-input-field'.
3) Click the submit button with ID 'submit-login-btn'.
4) Verify successful login.

However, these element IDs may be wrong or not exist. You MUST:
- First, register a new test user (username='errortest_[random]', password='TestPass123').
- Then attempt to log in using the credentials above.
- If the provided element IDs don't work, inspect the page visually and find the correct login elements.
- Complete the login successfully.
- At the end, output:
  a) 'TEST PASSED' or 'TEST FAILED'
  b) The corrected steps you actually followed (numbered)
  c) Explain which original steps were wrong and how you fixed them."
```

**What to Observe:**
- Agent recognizes incorrect element identifiers
- Discovers correct UI elements visually
- Successfully completes login despite flawed instructions
- Provides corrected step list

**Expected Result:** `TEST PASSED` with explanation of corrections made

---

### Step 4.2: Missing Steps in Purchase Flow

```powershell
python main.py --env playwright --highlight_mouse --initial_url "https://www.demoblaze.com/" --query "Execute this purchase flow, but note that some steps are missing. Discover and complete them.

Provided steps (incomplete):
1) Add a phone product to cart.
2) Complete the purchase with Name='Test User', Credit card='4111111111111111'.

These steps are missing important intermediate actions. You MUST:
- Identify what steps are missing (like navigating to cart, clicking place order, filling all required form fields).
- Insert the missing steps and complete the entire flow successfully.
- At the end, output:
  a) 'TEST PASSED' or 'TEST FAILED'
  b) The complete corrected step list with all missing steps added
  c) Explain what steps were missing and how you discovered them."
```

**What to Observe:**
- Agent identifies gaps in workflow
- Inserts missing navigation and form filling steps
- Completes full purchase successfully
- Documents discovered missing steps

**Expected Result:** `TEST PASSED` with complete step list including discovered missing steps

---

## Phase 5: Documentation Generation (Objective 4)

This test validates the agent's ability to explore and document an application.

### Step 5.1: Comprehensive Site Documentation

```powershell
python main.py --env playwright --initial_url "https://www.demoblaze.com/" --query "You are a QA engineer documenting the DemoBlaze e-commerce application.

Systematically explore the site by:
- Examining the navigation structure and all menu items.
- Testing each product category (Phones, Laptops, Monitors).
- Interacting with authentication features (Sign up, Log in).
- Exploring the shopping cart and checkout process.
- Testing any modals, forms, or interactive elements you find.

After thorough exploration (spend at least 15-20 interaction steps), generate structured documentation in Markdown with these sections:

1. **Application Overview** - Purpose and type of application.
2. **Navigation Structure** - List all navigation menu items and their purposes.
3. **Product Categories** - Describe each category and sample products.
4. **User Authentication** - How registration and login work.
5. **Shopping Cart & Checkout** - Step-by-step purchase workflow.
6. **Forms & Validations** - All forms found and their validation rules.
7. **Modals & Alerts** - Any popup elements and their purposes.
8. **Contact & About** - Information pages available.

When finished, say 'DOCUMENTATION COMPLETE' and provide the full documentation."
```

**What to Observe:**
- Agent systematically explores all site areas
- Interacts with various features
- Generates comprehensive structured documentation

**Expected Result:** `DOCUMENTATION COMPLETE` with full Markdown documentation covering all requested sections

---

## Execution Best Practices

1. **Run Tests Sequentially** - Execute one test at a time to avoid session conflicts and allow proper state management.

2. **Use `--highlight_mouse`** - This visual indicator helps track agent actions in real-time.

3. **Monitor Console Output** - Watch the rich table showing reasoning (left) and tool calls (right) for each iteration.

4. **Note Test Duration** - Complex tests (especially Phase 5) may take several minutes to complete.

5. **Capture Results** - Save console output for each test run for analysis:
   ```powershell
   python main.py ... | Tee-Object -FilePath "test_results_[test_name].txt"
   ```

6. **Review Agent Reasoning** - Pay attention to how the agent adapts when encountering unexpected UI elements or states.

---

## Success Criteria

By the end of all phases, you should observe:

- ✅ **Objective 1:** All basic tests execute with clear PASS/FAIL results
- ✅ **Objective 2:** Agent successfully automates complex workflows without hardcoded locators
- ✅ **Objective 3:** Agent detects and corrects flawed test instructions
- ✅ **Objective 4:** Comprehensive site documentation generated through autonomous exploration

---

## Troubleshooting

**Issue:** Browser doesn't open
- **Solution:** Verify Playwright Chrome is installed: `playwright install chrome`

**Issue:** API key errors
- **Solution:** Confirm `GEMINI_API_KEY` is set: `echo $env:GEMINI_API_KEY`

**Issue:** Test hangs or times out
- **Solution:** Some complex interactions may take time; monitor console for progress. If truly stuck, interrupt (Ctrl+C) and retry.

**Issue:** Random test failures
- **Solution:** DemoBlaze.com may have intermittent performance issues; retry the test to confirm consistency.

---

## Additional Notes

- These tests follow the user's rules for verification and element interaction patterns
- All tests use visual element discovery rather than hardcoded locators
- The agent should verify element presence before interaction
- Tests should end immediately upon completion without unnecessary waiting

For more information, refer to `COMPUTER_USE_QA_POC_GUIDE.md` in this directory.

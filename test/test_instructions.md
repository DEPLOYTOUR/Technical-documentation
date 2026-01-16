# Testing Documentation Instructions

This folder contains the template and guidelines for documenting tests performed on each Dataspace component. Please follow these instructions to ensure consistency and completeness in your test documentation.

## Instructions

1. **Create a test documentation file for each component or module.**
   - Name the file as `test_<component_name>.md` (e.g., `test_connector.md`).

2. **Use the provided template below to document your tests.**
   - Fill in the relevant sections for your component.
   - Leave the test cases and results open; update them as development progresses.

3. **Types of tests to document:**
   - **Unit Tests:** Validate individual functions, methods, or classes in isolation.
   - **Integration Tests:** Validate the interaction between this component and other components or services.
   - **Regression Tests:** Re-run previous tests after changes to ensure no existing functionality is broken.

4. **Test Environment:**
   - Specify the environment used (local, VM, cloud, etc.).
   - List the tools used (Bruno, Postman, Jest, Pytest, JUnit, etc.).

5. **Test Cases and Results:**
   - For each test, specify the scenario, tool, result (pass/fail), and any relevant explanation or link to the test script/collection.
   - Update the table as new tests are added or results change.

6. **Summary and Notes:**
   - Summarize the overall results and coverage.
   - Add any additional notes, issues, or references.

7. **References:**
   - Link to API test collections, unit/integration test scripts, and deployment/setup documentation.

---

## Example Template

See `test_template.md` in this folder for the Markdown template to use.

# Google AI Harness Extra Credit — Agent Airways Reliability Evaluation

## 1. Connection to Google's Harness Engineering

Google's *AI harness* concept emphasizes that a reliable agent needs more than a language model capable of generating instructions. The surrounding software must organize a control loop, expose appropriate tools, validate requests, manage execution state, enforce safety boundaries, observe the environment and evaluate whether the intended outcome actually occurred. The Agent Airways browser agent follows this architecture. Qwen3:8B proposes one browser action at a time in JSON, while a Python harness parses the proposal, checks the tool name and arguments, invokes the corresponding Playwright tool and sends the tool's observed result back to the model. The model then chooses its next action using that feedback. This separates decision-making from execution: the model can propose an action, but it cannot bypass Python's validation or independently declare that the reservation succeeded. The full workflow also uses review checks, a human-approval requirement, a confirmation verifier and a maximum-step stopping rule. Together, these components illustrate how a harness can make an agent more controlled, observable, and testable even when the underlying model makes mistakes.

**Connection to the class implementation:** `parse_action()` checks the model's JSON format; `run_requested_tool()` validates and dispatches requests to student-built tools; `run_agent()` manages the action–observation loop and stopping conditions; `check_review_against_goal()` compares the review with the requested trip; and `verify_confirmation()` checks the actual final state. The experiment below evaluates the reliability of this existing harness rather than introducing Google's Agent Development Kit (ADK).


## 2. Definition of Success and How Python Checks It

The evaluated task is the **full 16-tool Agent Airways pretend-reservation workflow**, not merely searching for a flight or reaching the passenger form. The requested reservation is for **one passenger from Washington Reagan (DCA) to Boston (BOS), departing November 17, 2026**, with **flight AA101**, **seat 1A**, **Test Traveler**, and **test.traveler@example.com**. Both checked baggage and priority boarding must remain unselected.

Python uses multiple layers of evidence to decide whether the task has succeeded:

1. **Validate the model's proposed actions.** The agent expects JSON with an `action` string and an `argument` dictionary. Before executing a request, Python checks that the tool is available and that the argument passes the selected tool's `validate()` function. Invalid button names are rejected without clicking the webpage.
2. **Check actions against the browser.** The Playwright tools read actual webpage values and screen transitions, rather than trusting what the model says it did. For example, airport dropdowns and the travel date are read back after being set, flight selection checks that the seat page opens, and seat selection checks that the requested seat is selected.
3. **Check the reservation before confirmation.** `check_review_against_goal()` is intended to compare the review screen with the expected route, date, flight, seat, passenger details, and optional extras. The notebook also requires the confirmation checkbox and a real person's `APPROVE` response before the pretend final confirmation can proceed.
4. **Verify the final result.** `verify_confirmation()` must report success after inspecting the actual confirmation screen and comparing the heading, confirmation code, route, date, flight, seat, passenger name, email, extras, and mock total against the recorded goal.
5. **Enforce a stopping condition.** The full agent has a maximum of **50 model/tool steps**. If it reaches that limit without verified confirmation, the program returns `step_limit`, not `success`.

**Strict success criterion:** A run is a **success only when Python receives a successful result from `verify_confirmation`**. Reaching the passenger screen, correctly filling form fields, clicking an intermediate button, or a model-generated claim of success is insufficient. Any run without that final verification is recorded as a **failure of the full task**, even if some intermediate actions worked correctly. This distinction matters because a partially completed browser workflow is not a completed reservation.

## 3. Experiment Setup and Methodology

Five provided trace files were analyzed using the **same traveler request and expected details**. Each trace begins with the search screen visible, which is consistent with starting the task from the beginning; the supplied evidence does not independently establish how the Colab runtime was restarted or whether all code files remained unchanged between executions. The recorded model proposals and results are the same across the five traces except for minor flight-results waiting times. Thus the comparison describes **five recorded attempts under the same apparent setup**, while avoiding unsupported claims about unobserved environment changes.

A run was marked **Failure** if its `status` was not `success` and the trace did not contain a successful `verify_confirmation` result. The end-state screenshots were checked against the traces to identify the last visible stage.

## 4. Five-Run Results

| Run | Success or Failure | Notes |
| --- | --- | --- |
| **1** | **Failure** | `step_limit` after 50 steps. Route, date, flight AA101, seat 1A, name, and email were entered successfully. Ten `continueToReview` calls were rejected as an unknown button; the browser remained on the passenger screen. |
| **2** | **Failure** | `step_limit` after 50 steps. Repeated the same navigation problem. The passenger form was populated correctly, but the unsupported `continueToReview` tool argument prevented reaching the review page. |
| **3** | **Failure** | `step_limit` after 50 steps. The agent selected the requested flight and seat, then filled passenger details. It repeatedly retried the rejected button name instead of finding a valid tool argument. |
| **4** | **Failure** | `step_limit` after 50 steps. The same failed button requests occurred; no review comparison, human approval, or final confirmation verification was recorded. |
| **5** | **Failure** | `step_limit` after 50 steps. The agent remained on the passenger-information screen with the correct name, email, and unchecked extras; final confirmation was never attempted. |

**Observed full-task success rate:** **0 / 5 = 0%**. All five attempts completed some intermediate steps but **none** achieved the notebook's final verified-success condition.

### Evidence for each recorded run

| Run | JSON trace | Final screenshot |
| --- | --- | --- |
| 1 | [`run_01_trace.json`](run_01_trace.json) | [`run_01.png`](run_01.png) |
| 2 | [`run_02_trace.json`](run_02_trace.json) | [`run_02.png`](run_02.png) |
| 3 | [`run_03_trace.json`](run_03_trace.json) | [`run_03.png`](run_03.png) |
| 4 | [`run_04_trace.json`](run_04_trace.json) | [`run_04.png`](run_04.png) |
| 5 | [`run_05_trace.json`](run_05_trace.json) | [`run_05.png`](run_05.png) |

Each screenshot shows the **Passenger information** stage rather than the review or confirmation page. The recorded fields display **Test Traveler** and **test.traveler@example.com**, with both optional extras unchecked. The JSON files provide stronger evidence of the failure cause because they capture the proposed actions and Python's exact rejection messages.

## 5. What Happened: Trace-Based Analysis

Across all five recorded attempts, the traces contain the same key sequence:

- **Steps 1–9 — Correct initial actions.** The agent identified the search page, selected DCA and BOS, verified the travel date, opened the search results, observed four flights, selected AA101, and selected seat 1A. The corresponding tools reported successful browser read-backs or stage changes.
- **Steps 10 and 12 — Wrong navigation action on the seat screen.** The model requested `click_step_button` with `{"button":"continue_to_review"}`, but Python reported that this button was not visible on the **seat** screen. This was not a successful click.
- **Steps 13–17 — Recovery and correct data entry.** The agent inspected the interactive elements, then successfully used `continue_from_seat` at step 14 to reach the passenger screen. It detected the new screen and filled the requested full name and email. Browser observations confirmed the actual values.
- **Step 18 — Reused a button from the previous screen.** The agent requested `continue_from_seat` while already on the **passenger** screen; Python correctly rejected the attempted click because that button was not visible there.
- **Steps 20–48 — Repeated tool-interface mismatch.** `list_interactive_elements` reported a visible, enabled webpage button whose HTML identifier was `continueToReview` and whose label was **Continue to review**. The model then repeatedly supplied `{"button":"continueToReview"}` to `click_step_button`. The tool returned **`Tool rejected the argument: Unknown button: continueToReview`**. The model made this rejected request **10 times per run**, alternating with further stage and element inspections, without finding a valid accepted tool argument.
- **Steps 49–50 — Step-limit termination.** The agent was still observing the passenger screen when it exhausted the 50-step budget. None of the traces includes a successful `check_review_against_goal`, `confirm_demo_reservation`, or `verify_confirmation` call.

Each run contains **13 unsuccessful tool requests**: two attempts to use `continue_to_review` from the seat screen, one attempt to reuse `continue_from_seat` from the passenger screen, and ten rejected `continueToReview` arguments. This breakdown indicates a **navigation/action-format problem**, not incorrect airport, seat, passenger, or email data. The webpage showed a usable Continue to review control, but the action proposed to the Python tool was not accepted. The likely underlying issue is confusion between a webpage's HTML element ID (camelCase) and the tool's supported button argument (as defined in its Python validator). The provided traces establish the rejected argument; the precise replacement argument would need to be confirmed from the tool implementation before making a code change.

## 6. Findings and Limitations

The experiment shows that the browser agent was **reliable at several local operations but unreliable at completing the whole workflow**. In every trace, Python confirmed the requested airports, date, flight, seat, and passenger fields, demonstrating that tool execution and read-back checks worked for the early stages. However, once the agent encountered the passenger-screen navigation problem, it repeatedly used an unsupported button argument despite the rejection message. Rechecking the same screen did not lead to a revised strategy. Consequently, all five attempts ended with `step_limit` and the observed complete-task success rate was 0%. The harness still provided important safety and diagnostic value: it rejected unsupported requests instead of performing unpredictable clicks, kept a detailed record of the behavior, and did not incorrectly label an unfinished reservation as successful.

There are several limitations to these results. First, **five runs of the same route, date, seat, model, and tool configuration are a small, narrow sample**; the observed 0% completion rate must not be generalized to every possible browser-agent task. Second, the recorded traces are almost identical, differing only slightly in flight-results wait time, so this experiment shows a consistent repeatable failure pattern rather than much behavioral diversity. Third, the screenshots show only the final visible webpage, not the entire execution history; the JSON logs are required to explain the repeated tool rejections. Fourth, the test did not reach review checking, human approval, or confirmation verification, so their behavior **was not actually exercised** in these attempts. Finally, although each trace begins on the search screen, the files alone cannot independently prove the exact runtime-reset procedure.

A useful follow-up evaluation would inspect the accepted `click_step_button` arguments and clarify the tool documentation so the model can distinguish a DOM element ID from a valid tool argument. A retry guard could also detect repeated identical rejections, prevent a wasteful action loop, and prompt a different recovery strategy. Those would be **future improvements**, not changes claimed to have been made during these five recorded tests. After such a change, a fresh set of five runs using the updated version could measure whether end-to-end reliability improves. Testing alternate routes, dates, seats, and delays would then reveal how well the agent generalizes.

## 7. Conclusion

**The five recorded evaluations resulted in zero verified full reservations out of five attempts (0% observed success).** Each attempt progressed through flight search, flight selection, seat selection, and passenger information, but failed to advance to review after repeatedly proposing a button argument rejected by Python. The result illustrates the central lesson of harness engineering: reliable agents need a well-defined tool interface, authoritative checks of real browser state, safeguards against repeating failed actions, and outcome-based evaluation. The agent's partial progress does not count as full-task success, and the preserved traces show exactly why. The evaluation therefore identifies one concrete, reproducible limitation without changing the implementation during the recorded experiment.

# Test Log

This log records the test scenarios run against the live n8n workflow, Gmail, Google Sheets and DeepSeek services. Each result comes from the `Enquiries` sheet or the n8n run.

## Summary

| # | Scenario | Enquiry ID | Outcome | Sheet status | Email sent | Result |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Invalid input (blank fields) | ENQ-20261107-7050 | Stopped at input check | Invalid Input - Not Processed | No | Pass* |
| 2 | Valid enquiry, approved | ENQ-20261110-9633 | Reply sent to customer | Completed - Reply Sent | Yes | Pass |
| 3 | Valid enquiry, rejected | ENQ-20261110-9867 | Nothing sent | Completed - Rejected by Reviewer | No | Pass |
| 4 | Valid enquiry, rejected (second run) | ENQ-20261110-10065 | Nothing sent | Completed - Rejected by Reviewer | No | Pass |
| 5 | AI failure (invalid API key) | ENQ-20261110-10141 | Approval not requested, reviewer alerted | AI Failed - Needs Manual Handling | No | Pass |

\* Scenario 1 was not planned. It happened during testing because a field-name bug in the normalisation step left every field empty. The invalid-input path then behaved correctly. The row was later removed from the sheet during clean-up, so it isn't in the final record.

## Scenario details

### 1. Invalid input
- **Input:** a submission with blank name, email, subject and message.
- **Expected:** node 03 routes to the invalid branch, node 04 logs the row, and nothing else runs.
- **Observed:** `validation_error` listed all four problems. The row was logged as Invalid Input - Not Processed.

### 2. Valid enquiry, approved
- **Input:** name "Test Customer One", email `test-customer@example.com`, subject "Test 1 - AI failure path".
- **Expected:** DeepSeek classifies the enquiry, the reviewer is asked to approve, and the reply is sent after approval.
- **Observed:** Category General, urgency Low, confidence 0.92. Approval response `{"approved":true}`. Gmail returned message ID `[message ID removed]`, and the row shows Completed - Reply Sent with email status Sent.

### 3. Valid enquiry, rejected
- **Input:** email `test-customer@example.com`, subject "Test 2 - rejection path".
- **Expected:** after rejection, no email is sent and the row is logged as rejected.
- **Observed:** Approval response `{"approved":false}`. Status Completed - Rejected by Reviewer, email status Not Sent, no message ID.

### 4. Valid enquiry, rejected (second run)
- **Input:** email `test-customer@example.com`, subject "Test 3 - AI failure path".
- **Expected:** same as scenario 3.
- **Observed:** Category General, urgency Low, confidence 0.93. Approval response `{"approved":false}`. Status Completed - Rejected by Reviewer, email status Not Sent.

### 5. AI failure
- **Input:** email `test-customer@example.com`, subject "Test 3 - AI failure path". The DeepSeek credential was set to an invalid key for this run only.
- **Expected:** the DeepSeek call fails, the workflow skips approval, logs the failure, and alerts the reviewer.
- **Observed:** Node 06 took its error path. The row shows AI Failed - Needs Manual Handling, approval Not Reached, email Not Sent. The reviewer alert was sent. The real key was restored afterwards.

## Problems found during testing

| Problem | Effect | Fix |
| --- | --- | --- |
| Normalisation read field names that n8n didn't pass through | Every enquiry was rejected as invalid | Read both field names and labels |
| Google Sheets API not enabled in the Cloud project | Sheets nodes returned 403 | Enabled the API |
| Approval type option didn't match n8n's options | Node showed a warning | Selected the correct option |
| Gmail rate limit (HTTP 429) after rapid repeat runs | Approval email not sent for one run | Waited, then re-ran |
| Gmail app in Testing mode | Sign-in blocked until the account was added as a test user | Added the test user |

## Not yet tested

- **Approval timeout (24 hours):** the setting is configured but has not been run to expiry.
- **Live DeepSeek key after restoration:** Scenario 2 used DeepSeek successfully. Scenario 5 confirmed the failure path with an invalid key. The live key has not been re-checked since it was restored.
- **Classification accuracy:** not measured. Each scenario used one enquiry, so these results show the workflow's behaviour, not the quality of the AI's classifications.
- **Multiple reviewers, concurrent enquiries, and Gmail send failures:** not tested.

## 3. Service Tasks vs User Tasks and Variables

| Element | Type | Description | Input Variables (example) | Output Variables (example) |
|---|---|---|---|---|
| **Submit Leave Request** | User Task (Employee) | Employee enters leave type and number of days | `employeeId`, `leaveType`, `requestedDays` | `leaveType`, `requestedDays` |
| **Evaluate Leave Request (DMN)** | Business Rule Task | Invokes DMN for balance check and approval decision | `leaveType`, `requestedDays`, `availableBalance` | `decision`, `decisionMessage` |
| **Manager Approval** | User Task (Manager) | Manager approves or rejects request | `employeeId`, `leaveType`, `requestedDays`, `availableBalance` | `managerApproved`, `managerComment` |
| **HR Record Keeping** | Service Task | Sends approved request to HR system for record-keeping | `employeeId`, `leaveType`, `requestedDays`, `approvalStatus`, `managerId`, `managerComment` | `recordId` (optional) |
| **Update Leave Balance** | Service Task | Deducts leave from employee balance | `employeeId`, `leaveType`, `requestedDays`, `availableBalance` | `newBalance` |
| **Notify Employee (Approved/Rejected)** | Service Task | Sends notification to employee (via email/SMS/app) | `employeeId`, `status`, `message` | — |

### Example Process Variables (Payload)

```json
{
  "employeeId": "EMP101",
  "employeeName": "John",
  "managerId": "MGR10",
  "leaveType": "Sick",
  "requestedDays": 2,
  "availableBalance": 10,
  "decision": "AUTO_APPROVE",
  "managerApproved": true,
  "newBalance": 8
}

## 4. Justification for DMN Design

### Why a single DMN table?

- The **eligibility (balance check)** and **approval-routing logic** are closely related and can be handled in one decision table.
- Using the **FIRST hit policy** allows rule prioritization — the insufficient balance rule is evaluated first, ensuring such requests are rejected immediately before any other rules.
- It keeps the model simple and easy to maintain for the current requirements.

### Why is this suitable?

The table returns a single decision with three possible outcomes:

- **REJECT** — Insufficient balance
- **AUTO_APPROVE** — Sick leave ≤ 3 days
- **MANAGER_REVIEW** — Casual, Earned, or Sick leave > 3 days

This directly matches the business requirements.

Placing the balance check as the first rule ensures the request never goes to the manager if the balance is insufficient.

It also reduces complexity in the BPMN model.

### When to use multiple DMN tables?

For larger, more complex rule sets (e.g., additional leave types, seniority rules, department-specific rules, or probation restrictions), it may be better to split the logic into:

- **DMN 1: Leave Eligibility** — Checks balance and other constraints
- **DMN 2: Approval Routing** — Determines auto-approve or manager review

This improves reusability and maintainability in an enterprise environment.

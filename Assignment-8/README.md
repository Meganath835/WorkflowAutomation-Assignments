# Exercise 8 – Hospital Appointment Booking

## Files
| File | What it is |
| --- | --- |
| `send-notification.bpmn` | Child process: start → REST connector "Send Telegram message" → end. Reads `recipientId`, `messageText`. |
| `slot_decision.dmn` | Decision `slot_decision`, hit policy UNIQUE. Inputs `department`, `bookedCount`; outputs `slotStatus`, `suggestedAction`. |
| `hospital-booking.bpmn` | Parent process, **propagation OFF** (only mapped variables go in and out). |
| `hospital-booking-propagate-all.bpmn` | Same parent, **propagation ON** (all variables copied both ways). Same process ID, so deploying it creates a new version. |
| `Exercise8-Hospital-Booking.postman_collection.json` | The three test cases plus the cancellation message. |

## Setup
1. Create a Telegram bot with @BotFather and copy the token. Send any message to your bot, then open
   `https://api.telegram.org/bot<TOKEN>/getUpdates` and copy `message.chat.id`. That is your `telegramChatId`.
2. Add the secret `TELEGRAM_BOT_TOKEN`:
   - SaaS: Console → your cluster → Connector secrets.
   - C8 Run: set it as an environment variable before starting C8 Run.
3. Deploy `slot_decision.dmn`, `send-notification.bpmn` and `hospital-booking-propagate-all.bpmn`
   (or `hospital-booking.bpmn`) from Camunda Modeler.
4. Import the Postman collection and set the `telegramChatId` collection variable.
   For SaaS, change `baseUrl` to your cluster's REST address and add OAuth.

## DMN rules (UNIQUE: exactly one rule matches)
| department | bookedCount | slotStatus |
| --- | --- | --- |
| Cardiology | < 10 / >= 10 | AVAILABLE / FULL |
| Orthopedics | < 8 / >= 8 | AVAILABLE / FULL |
| Neurology | < 6 / >= 6 | AVAILABLE / FULL |
| General Medicine | < 15 / >= 15 | AVAILABLE / FULL |
| any other department | < 20 / >= 20 | AVAILABLE / FULL |

The decision result has two outputs, so it is stored as a context in `slotResult`.
The gateway checks `slotResult.slotStatus`.

## Test cases
1. **Case 1 (P1001, bookedCount 3):** Check slot → AVAILABLE → Book slot → Send confirmation (child instance) → waits at
   the 10-minute "Appointment time reached" timer, then completes.
2. **Case 2 (P1002, bookedCount 12):** Check slot → FULL → Send slot-full message (child instance) → Booking not possible.
3. **Case 3 (P1003):** send 3a, then send 3b within 10 minutes. The message is correlated on `patientId`.
   The interrupting event subprocess "Handle cancellation" cancels the timer, runs Release slot, then calls
   send-notification with the cancellation text.

The timer exists so that the instance is still alive after a successful booking. Without it the instance would end
right after the confirmation, and the cancellation message would have nothing to correlate to.

## Propagation comparison (what to show in Operate)
| | `hospital-booking-propagate-all.bpmn` | `hospital-booking.bpmn` |
| --- | --- | --- |
| Parent → child | Every parent variable (patientName, department, bookedCount, slotResult, booking, …) plus `recipientId` and `messageText` | Only `recipientId` and `messageText` |
| Child → parent | Everything the child creates (`telegramResponse`, `recipientId`, `messageText`) lands in the parent | Only the output mapping: `<callActivityId>_status` (HTTP status of the Telegram call) |
| Trade-off | Easy to set up, but the child sees data it does not need, and child variables can overwrite parent variables | Clean contract between parent and child; this is the reusable-subprocess style |

Open the child instance from each call activity in Operate, compare the Variables panels, and screenshot both.

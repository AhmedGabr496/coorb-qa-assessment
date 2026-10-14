# Task 1: Local Fund Transfer – Test Design & Risk Coverage

## 1.1 Questions and assumptions

| # | Question | Why it matters | Assumption if unanswered |
| --- | --- | --- | --- |
| Q1 | Does "local transfer" mean same bank only, or other banks in the country too? If other banks are included, is it instant or batch, and what happens at cut-off times and on holidays? | It decides the shape of the feature. Other-bank transfers can sit in Pending, fail after the debit, and need reversals. | Both are in scope. Other-bank transfers go Pending, then end as Completed, or as Failed and reversed. |
| Q2 | What does the customer see when we don't know if the transfer went through? For example, the network drops right after they enter the OTP. | Showing "failed" when the money actually left causes duplicate payments, because the customer tries again. | The app shows "being processed", and the final status appears in the history. A retry with the same request ID never debits twice. |
| Q3 | How is the daily limit counted? Per customer or per account? Shared with internet banking and the branch? Calendar day or rolling 24 hours? Do pending, failed and reversed transfers count, and does the fee? Are exactly 10,000.00 and 20,000.00 allowed? | The AC gives the numbers but not the rules. Each gap is either a way around the limit or a way to wrongly block a customer. | Per customer, across all accounts and digital channels, by calendar day in bank time. Completed and pending count; failed don't; a reversal gives the limit back. The fee doesn't count, and both limits are inclusive. |
| Q4 | What are the OTP rules? How long is a code valid? How many wrong attempts are allowed, and what happens after the last one? What are the resend rules? Is each OTP tied to one transfer's amount and beneficiary? | The OTP is the only protection when a fraudster tricks a customer into sharing a code. A reusable, guessable or untied code can approve a different payment. | A 6-digit SMS code, valid for 3 minutes, tied to the amount and beneficiary. Three wrong attempts cancel the transfer. A resend cancels the old code but keeps the attempt count. A code from an abandoned transfer can't be reused. |
| Q5 | What is the fee model (flat, percentage or tiered, with caps and a rounding rule)? Who pays it: sender, beneficiary or both? Is it added on top of the amount or taken out of it? Where does the customer see it before confirming? For example, with exactly 10,000.00 in the account, sending 10,000.00: is it rejected, is the fee taken out of the amount, or does the account go overdrawn? | "According to the bank's fee configuration" can't be tested. The answer changes the amount on every screen, SMS and ledger entry. Charging a fee the customer never saw is a consumer-protection issue. | The sender pays, and the fee is added on top. It's shown on a review screen before the OTP, debited separately, and refunded on failure. The balance must cover the amount plus the fee, so the 10,000.00 example is rejected. |
| Q6 | What are the failure reasons, and the message for each? If a transfer fails after the debit, how is it reversed, and how quickly? Where does the refund go if the account has been frozen or closed by then? | "An error message" isn't testable, and the customer needs to know whether money left. A refund with nowhere to land gets stuck, and nobody tells the customer. | A specific message for each reason. Nothing is debited when a transfer fails before execution. Failures after the debit are reversed automatically (amount and fee), with an SMS. A refund for a closed or fully frozen account goes to a holding account, and operations contacts the customer. |
| Q7 | Are the balance, limits and account status checked again when the transfer runs? For example, a cheque could clear while the customer is on the OTP screen. | If they're only checked before the OTP, the transfer runs on stale data. It could overdraw the account, break the limit or debit a frozen account. | Every check runs again at execution. If one fails, the transfer is rejected with the matching message, nothing is debited, and a new OTP is needed. |
| Q8 | Which accounts can be the source: which types, currencies and statuses (dormant, frozen, on hold)? If foreign-currency accounts are allowed, which rate applies, is it shown before the OTP, and how are limits converted? | The server must enforce which accounts are allowed, not just the app's list. A foreign-currency source adds a currency conversion that nobody has defined. | Only active local-currency current and savings accounts the customer can operate alone. Everything else is hidden in the app and rejected by the API. |
| Q9 | Is there a cooling-off period or a lower limit for newly added beneficiaries? What does a customer with no beneficiaries see? | After taking over an account, a fraudster adds their own beneficiary and sends the maximum straight away. A cooling-off period is a common control against this. | No cooling-off, which I'd raise with the business as a risk. A customer with no beneficiaries sees an empty screen with a link to add one. |
| Q10 | Is the confirmation SMS sent after the transfer is committed, and what happens if it fails? | If the SMS is part of the transaction, an SMS outage can roll back a transfer that went through, or make it look failed. Full account numbers in an SMS are a data leak. | The SMS is sent after the transfer commits and retried on failure. It never affects the transfer, and account numbers are masked. |
| Q11 | What peak volume, response time and devices must we support? | Slow responses at peaks such as salary days cause timeouts, and timeouts cause the unknown outcomes in Q2. The device list defines the test matrix. | Salary-day peaks at today's internet-banking volumes. 95% of transfers answered within 3 seconds. The latest two iOS versions, and Android 10 and later. |
| Q12 | What are the amount rules: the minimum amount, and how many decimals? | I need them for boundary tests. Without a minimum, 0.00 or a 3-decimal amount has no defined behaviour. | Minimum 0.01, two decimals. |
| Q13 | Who sees the purpose note, how long can it be, and is sensitive data restricted? | It's free text that reaches the beneficiary, statements and back-office systems. Customers do type card numbers or passwords into fields like this. | Optional, up to 140 characters, and visible to the beneficiary. Stored and shown as plain text, with a hint not to enter sensitive data. |
| Q14 | Are scheduled or recurring transfers in scope? | They'd add a whole new flow: a future execution date, checks on that date, and limits counted on the day the transfer runs. | Out of scope. |

## 1.2 Risk assessment

The three areas I worry about most are the ones where a bug moves real money the wrong way.

| # | Area | Why it's high risk |
| --- | --- | --- |
| 1 | Duplicate, lost or stuck transfers | Networks drop, systems time out and people tap twice. The requirement says nothing about any of it. |
| 2 | OTP and account ownership | The OTP is the only approval step.  |
| 3 | Limits, balance and fees | The ACs give numbers but no rules, and these rules usually break at the edges. |

### 1. Duplicate, lost or stuck transfers

| What could go wrong | Impact on Customer | Impact on Bank | Us (delivery partner) |
| --- | --- | --- | --- |
| A timeout is shown as "failed", or a double tap sends the request twice. The money goes out twice. | Transfare twice | Complaints, manual refunds, bad press | A Sev-1 incident on our feature |
| The account is debited but the beneficiary is never credited, and nothing is refunded | Money missing for days | Manual reconciliation, plus regulatory deadlines on complaints | An incident that's hard to trace across systems we don't own |
| The refund comes back to an account that's now frozen or closed | Doesn't know where their money is | Money stuck in a holding account | Seen as a gap in our design |

### 2. OTP and account ownership

| What could go wrong | Impact on Customer | Impact on Bank | Us (delivery partner) |
| --- | --- | --- | --- |
| A code the customer was tricked into sharing approves a different amount or beneficiary | Money stolen | Fraud losses and attention from the regulator | A security flaw in code we delivered |
| Old codes still work, or a resend resets the attempt counter | Account open to code guessing | A failed security audit | Go-live blocked by the pen test |
| Changing an ID in the request moves money from someone else's account, or from a frozen one | Money taken from the wrong account | A possible breach disclosure | Serious contractual and reputational damage |

### 3. Limits, balance and fees

| What could go wrong | Impact on Customer | Impact on Bank | Us (delivery partner) |
| --- | --- | --- | --- |
| The balance drops during the OTP step (for example, a cheque clears), but the transfer still goes through | Unexpected overdraft | Lends money it never agreed to lend | A defect found late, in UAT or production |
| The fee charged isn't the fee shown, or both sides are charged | Overcharged, or the beneficiary gets less | Mass refunds and a consumer-protection issue | Data clean-up and lost trust |

## 1.3 Test cases

| ID | Test case | Requirement |
| --- | --- | --- |
| TC-01 | A normal transfer moves the right amounts | AC1, AC3, AC5 |
| TC-02 | A slow reply or a double tap never sends the money twice | AC5, AC6 |
| TC-03 | If the app or the internet cuts out, the customer still knows what happened | AC5, AC6 |
| TC-04 | If the other bank sends the money back, the customer gets all of it | AC5, AC6 |
| TC-05 | When a transfer fails, the message is clear and no money is taken | AC6 |
| TC-06 | The one-time code can't be guessed | AC2 |
| TC-07 | A code only works for the transfer it was sent for | AC2 |
| TC-08 | Customers can only send money from their own accounts to their own saved receivers | AC1 |
| TC-09 | The 10,000 and 20,000 limits work at exactly the edges | AC4 |
| TC-10 | The daily limit can't be dodged | AC4 |
| TC-11 | Everything is checked again at the moment the money is sent | AC4, AC6 |
| TC-12 | The fee is right, and the customer must have enough to pay it | AC3 |

### TC-01 · A normal transfer moves the right amounts

**Description:** check the everyday journey from start to finish, once to someone at the same bank and once to someone at another bank.

**Preconditions:**

- The customer is logged in to the app.
- Their account A1 has 50,000.00, and they haven't sent any money today.
- They have two saved receivers: B1 at the same bank and B2 at another bank.
- We know the fee for sending 1,500.00.

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Choose account A1 and receiver B1, type 1,500.00 and a short note, and continue. | The summary screen shows the amount, the fee and the total before asking for the code. |
| 2 | Confirm, then type in the correct one-time code. | The success screen shows a reference number. B1 receives 1,500.00 straight away. |
| 3 | Check the balance and the bank's records. | The account is charged 1,500.00 plus the fee, shown as two separate lines. The balance updates on its own. |
| 4 | Check the text message and the transaction history. | Exactly one text message, with the amount, the fee, the reference number and only the last digits of the account numbers. The history shows the same reference number. |
| 5 | Repeat steps 1 to 4 with receiver B2. | Same results, except the transfer shows "In progress" first, then "Completed". |

### TC-02 · A slow reply or a double tap never sends the money twice

**Description:** make sure the customer is never charged twice when the bank's system is slow, or when they tap a button twice.

**Preconditions:**

- Account A1 has 10,000.00.
- We can make the bank's system process the transfer but reply too late for the app.

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Make the bank's system reply late, then send 500.00 with the correct code. | The app says the transfer is "being processed", not that it failed. |
| 2 | Let the app try again on its own. | The same reference number comes back. The account is charged only once (500.00 plus the fee). |
| 3 | Put the system back to normal. On a real phone, tap Confirm twice quickly. | Only one code is sent. |
| 4 | Tap Submit twice quickly on the code screen. | Only one transfer is made. |

### TC-03 · If the app or the internet cuts out, the customer still knows what happened

**Description:** see what the customer sees when the app closes or the internet drops in the middle of a transfer.

**Preconditions:**

- Real iPhone and Android phones.
- Enough money in the account.
- We can switch the internet off.

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Close the app straight after typing in the code, then open it again. | The customer can see whether the transfer went through. No money is taken twice or without showing in the history. |
| 2 | Turn on airplane mode while the transfer is being sent, then turn it off. | Same as step 1. The app doesn't crash. |
| 3 | Leave the app in the background on the code screen until the code expires, then come back. | The app says the code has expired and lets the customer ask for a new one. |
| 4 | Switch from Wi-Fi to mobile data while the transfer is being sent. | The transfer is sent once, and the history shows it. |

### TC-04 · If the other bank sends the money back, the customer gets all of it

**Description:** the other bank sometimes refuses a transfer after the money has already left. When that happens, the customer must get back both the amount and the fee. This includes the case where the customer's account has been frozen in the meantime.

**Preconditions:**

- Account A1 has 10,000.00, and no money has been sent today.
- Receiver B2 is at another bank.
- We can make the other bank accept the transfer at first, then send it back because the receiver's account is closed.

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Send 2,000.00 to B2. | The transfer shows "In progress". 2,000.00 plus the fee has left the account. |
| 2 | Make the other bank send the money back. | The transfer shows "Failed", with the reason. The 2,000.00 and the fee come back. The 2,000.00 no longer counts toward today's limit. The customer is told within the agreed time. |
| 3 | Send another 2,000.00 to B2, then freeze account A1 before the money comes back. | The money is handled the way the business agreed: returned to the account, or held safely while the bank's team is alerted. The customer is told where it is. It never just disappears. |

### TC-05 · When a transfer fails, the message is clear and no money is taken

**Description:** check the error messages for the most common problems (AC6).

**Preconditions:**

- A customer with account A1.
- We can make the bank's systems fail on purpose.

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Try to send 1,000.00 when the account doesn't have enough to cover the amount plus the fee. | A clear "not enough money" message. No money is taken. |
| 2 | Try to send 1,000.00 while the bank's main system is down. | A clear message that the service isn't available right now and to try again later. No money is taken. |
| 3 | Start sending 1,000.00, then stay on the code screen until the customer is logged out. | The customer is asked to log in again. No money is sent. |

**For every step:** the message is in the customer's language, has no error codes, and says what to do next. No fee is charged, nothing counts toward the daily limit, and no "success" text message is sent.

### TC-06 · The one-time code can't be guessed

**Description:** check the rules for wrong codes, expired codes and new codes (AC2).

**Preconditions:**

- The customer is on the code screen for a 500.00 transfer.
- We can see the codes in the test environment.
- A code lasts 3 minutes (our assumption).

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Type a wrong code. | Error, saying 2 tries are left. |
| 2 | Type a wrong code again. | Error, saying 1 try is left. |
| 3 | Ask for a new code. | A new code is sent. It still says 1 try left, because asking for a new code doesn't give more tries. |
| 4 | Type in the old code. | Refused. This is the third wrong try, so the transfer is cancelled. |
| 5 | Type in the new, correct code. | Refused, because the transfer was already cancelled. No money is taken. |
| 6 | Start a new transfer and type its code just before it expires. | Accepted. The transfer goes through. |
| 7 | Start another transfer and type its code just after it expires. | Refused. The customer is offered a new code. |

### TC-07 · A code only works for the transfer it was sent for

**Description:** a code sent for one transfer can't be used to approve a different amount, a different receiver, or a second transfer. We test this by sending requests straight to the bank's server, without using the app, the way an attacker would.

**Preconditions:**

- The customer has saved receivers B1 and B2.
- A code has been sent for a 100.00 transfer to B1.

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Use that code, but change the amount to 9,000.00. | Refused. No money is taken, and the attempt is recorded as suspicious. |
| 2 | Use that code, but change the receiver to B2. | Same as step 1. |
| 3 | Try to send money without any code. | Same as step 1. |
| 4 | Finish the 100.00 transfer to B1 normally. | It goes through once. The text message with the code showed the amount and who the money was going to. |
| 5 | Use the same code again. | Refused. There is still only one 100.00 transfer. |
| 6 | Start a transfer and stop at the code screen. Start a new transfer and use the first code. | Refused. No money is taken. |

### TC-08 · Customers can only send money from their own accounts to their own saved receivers

**Description:** the bank's server, not just the app, must stop money moving from accounts the customer doesn't own or isn't allowed to use.

**Preconditions:**

- Customer C1 has account A1 (working) and account A3 (frozen).
- Another customer, C2, has account A2 and a saved receiver B9.
- Everything is done while logged in as C1. Steps 2 to 7 send requests straight to the bank's server.

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Open the list of accounts in the app. | Only A1 is shown. A3 is hidden or greyed out. |
| 2 | Try to send money from C2's account A2. | Refused. No money moves, and the message reveals nothing about C2. |
| 3 | Try to send money from the frozen account A3. | Refused. |
| 4 | Try to send money to C2's receiver B9, then to an account number that was never saved. | Both refused. |
| 5 | Try to send −100.00, then 0.00. | Both refused. |
| 6 | Send a normal transfer, but say the fee is 0.00. | The 0.00 is ignored, and the normal fee is charged. |
| 7 | Log out, then try to send a request using the old login. | Refused. |

### TC-09 · The 10,000 and 20,000 limits work at exactly the edges

**Description:** check the limits at the exact amounts where they should start and stop working (AC4).

**Preconditions:**

- The account has 50,000.00, so there's always enough money.
- Before each step, the customer has already sent the amount shown in that step.
- Exactly 10,000 and exactly 20,000 are allowed (our assumption).

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Already sent 0.00 today. Send 10,000.00. | Allowed. The fee doesn't count toward the limit. |
| 2 | Already sent 0.00 today. Send 10,000.01. | Refused: more than 10,000 in one transfer. |
| 3 | Already sent 10,000.00 today. Send 10,000.00. | Allowed. Exactly 20,000 for the day. |
| 4 | Already sent 10,000.01 today. Send 10,000.00. | Refused: over the daily limit. 9,999.99 left today. |
| 5 | Already sent 15,000.00 today. Send 5,000.01. | Refused: over the daily limit, not the single-transfer limit. 5,000.00 left today. |
| 6 | Repeat steps 2, 4 and 5 by sending requests straight to the bank's server. | Same results as in the app. |

**For every refusal:** it happens before a code is sent. No money is taken, and the day's total doesn't change.

### TC-10 · The daily limit can't be dodged

**Description:** the 20,000 limit must count everything the customer sends in a day:

- from all their accounts
- through the app and through internet banking
- when two transfers are sent at the same moment
- around midnight
- after failed transfers

**Preconditions:**

- Customer C1 has accounts A1 and A2, with 50,000.00 each, and also uses internet banking.
- We can change the time in the test environment.
- Each step starts on a fresh day, unless it says otherwise.

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Send 10,000.00 from A1 and 10,000.00 from A2, then 0.01 more. | The 0.01 is refused. |
| 2 | Send 15,000.00 through internet banking, then 5,000.01 in the app. | Refused. |
| 3 | With 15,000.00 already sent, send two 5,000.00 transfers at the exact same moment. | Only one goes through. |
| 4 | With 20,000.00 already sent and the phone set to a different time zone, send 10,000.00 at 23:59:59 bank time. | Refused. |
| 5 | Send it again at 00:00:00 bank time. | Allowed. The limit resets at the bank's midnight. |
| 6 | After a 10,000.00 transfer has failed, send two transfers of 10,000.00. | Both allowed. Failed transfers don't count. |

**For every step:** the bank's records never show more than 20,000.00 sent in one day, and the app shows the same total.

### TC-11 · Everything is checked again at the moment the money is sent

**Description:** the balance, the daily limit and the account status are checked again after the customer types in the code, not only before (Q7).

**Preconditions:**

- Customer C1 has account A1 with 2,000.00, and a second account A2.
- C1 has already sent 15,000.00 today.
- C1 is on the code screen for a 1,500.00 transfer from A1. Each step starts again from here.

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | While C1 is on the code screen, a cheque for 1,000.00 is paid out of A1. Then C1 types in the correct code. | Refused: not enough money. |
| 2 | While C1 is on the code screen, 5,000.00 is sent from A2 through internet banking. Then C1 types in the code. | Refused: daily limit reached. |
| 3 | While C1 is on the code screen, A1 is frozen. Then C1 types in the code. | Refused: account blocked. |

**For every step:** no money is taken and no fee is charged. The code can't be used again, and the customer can start over.

### TC-12 · The fee is right, and the customer must have enough to pay it

**Description:** check the fee amounts, and that the account must hold enough for the transfer plus the fee (AC3).

**Preconditions:**

- The bank's real fee rules are set up in the test environment.
- The numbers below use example rules, to be replaced with the real ones:
    - up to 1,000.00: fee 2.00
    - 1,000.01 to 5,000.00: fee 5.00
    - over 5,000.00: 0.15% of the amount, but never less than 8.00 or more than 12.00

| Step | Action | Expected result |
| --- | --- | --- |
| 1 | Send 1,000.00 (top of the first range). | Fee 2.00 |
| 2 | Send 1,000.01 (start of the second range). | Fee 5.00 |
| 3 | Send 5,000.01 (the lowest fee applies). | Fee 8.00 |
| 4 | Send 6,030.00 (checks rounding: 0.15% is 9.045). | Fee 9.05 |
| 5 | Send 10,000.00 (the highest fee applies). | Fee 12.00 |
| 6 | Send 1,000.00 (fee 2.00) from an account holding 1,002.00. | Allowed. 0.00 left. |
| 7 | Send 1,000.00 from an account holding 1,001.99. | Refused: not enough money. |
| 8 | Send 1,000.00 from an account holding 1,000.00. | Refused: not enough to pay the fee. |

**For every step:** the same fee shows on the summary screen, the success screen, the text message, the bank's records and the statement. Only the sender pays it.

## 1.4 Coverage decisions

### Not covered, and why

| Not covered | Why |
| --- | --- |
| Every fee amount | Developers cover the full list. TC-12 checks the fee rules from start to finish. |
| Speed under heavy use | Needs a separate load test, not a test case. |
| Full security testing | Done by the bank's security team. |
| Layout, accessibility, phone models | Covered by design reviews/tests and by running TC-01 on different phones |

### Automated or  manuallly tested

| Cases | How | Why |
| --- | --- | --- |
| TC-01, TC-02, TC-04 to TC-12 | Automated, on every build | Money and security rules must hold every time, and when they break, nothing looks wrong on screen. |
| TC-03 | manual | Needs real phones: closing the app, losing signal. |
| Double tap (TC-02), message wording (TC-05) | manual | Needs a person's judgement. |

### Safe to release when

- All 12 cases pass with the bank's real fee and limit settings.
- No critical/security bugs are open.
- Every test transfer matches the bank's records, with nothing missing or doubled.
- Load and security tests pass, and the business signs off.

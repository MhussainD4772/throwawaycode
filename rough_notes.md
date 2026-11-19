Story 1: Outlook Connection

Title:
Add OutlookBrowser utility to open Outlook using Windows SSO (Single Sign-On)

Description:
As an automation tester, I want the framework to open Outlook on the web automatically using my existing Windows login, so that tests can validate emails without hardcoded credentials or manual login.

Background:
Some workflows (e.g. Document eDelivery consent in Connects) send an email that the test must verify. Playwright starts a fresh browser with no login, so Outlook cannot be reached directly.

Solution:
New class com.ptaf.utils.OutlookBrowser with one public method, open(Playwright playwright):

	•	Uses a persistent browser profile saved at C:\Users\<user>\playwright-outlook-profile, outside the project, so it's never committed
	•	First run: no saved profile exists, so it launches the installed Microsoft Edge, which signs in automatically through Windows SSO, and saves the session
	•	Later runs: launches Playwright's Chromium with the saved session and opens the inbox directly
	•	Session expired: detects it (the "New mail" button doesn't appear within 30 seconds), deletes the old profile, and signs in again through Edge
	•	No usernames, passwords or tokens are stored in code

Acceptance Criteria:

	☐︎	With no profile folder, OutlookBrowser.open() opens Edge and lands in the user's inbox without a password prompt
	☐︎	On the next run, it opens Chromium and lands in the inbox using the saved session
	☐︎	If the saved session is invalid, it deletes the profile, falls back to Edge, and lands in the inbox
	☐︎	If both attempts fail, it throws RuntimeException("Outlook login failed")
	☐︎	The profile folder is created in the user's home directory, not in the project
	☐︎	Works for any tester on an FNB laptop, using their own Windows login

Notes:

	•	Requires Microsoft Edge (installed on all company laptops)
	•	Works only on a logged-in Windows machine. It isn't suitable for CI (Continuous Integration) servers, which would need Microsoft Graph later
	•	The profile folder holds a logged-in Outlook session: treat it as sensitive, and never commit or share it



**Story 2: Outlook Mail Reader**

**Title:**
Add OutlookMailReader utility to find, validate and read emails in Outlook

**Description:**
As an automation tester, I want a reusable utility that finds a specific email in Outlook, confirms it arrived recently, and extracts a value from it (such as a token), so that tests can validate email-based workflows end to end.

**Background:**
After `OutlookBrowser` opens Outlook, tests still need to find the right email and read its contents. The utility has to be generic, so any team can use it for different emails and different values without changing the code.

**Solution:**
New class `com.ptaf.utils.OutlookMailReader` with two public methods.

`waitForEmail(page, searchQuery, mustContain, maxAgeMinutes, timeoutSeconds)`:
- Types `searchQuery` into Outlook search, e.g. `from:sender subject:"..."`
- Opens only an email whose summary contains `mustContain`, so it never opens an unrelated email
- Reads the email's received time and accepts it only if it arrived within `maxAgeMinutes`, so it never uses an old email
- Retries every few seconds until `timeoutSeconds` runs out
- Returns the full body text of the email
- Fails with a clear message if no recent matching email arrives

`extract(text, regex)`:
- Returns the first match of the regex (a pattern describing the text to find)
- If the pattern has brackets `( )`, returns only the part inside them
- Patterns can span multiple lines of the email
- Returns `null` if nothing matches

Each caller decides what to search for and what to extract. Nothing is specific to one email.

**Acceptance Criteria:**
- [ ] Finds and opens the newest email that matches the search and contains `mustContain`
- [ ] Ignores unrelated emails that appear at the top of the inbox
- [ ] Rejects emails older than `maxAgeMinutes` and keeps waiting for a new one
- [ ] Fails after `timeoutSeconds` with: *"No email containing '...' received in the last N minutes"*
- [ ] `extract(body, "(?i)token\\D*?(\\d+)")` returns only the token number from the eDelivery email
- [ ] `maxAgeMinutes` and `timeoutSeconds` are passed by the caller, not hardcoded
- [ ] Verified end to end in Connects: click Yes on Accept E-mail Document Delivery → read the token from Outlook → enter it → validation passes

**Example usage:**
```java
String body = OutlookMailReader.waitForEmail(
        page,
        "from:NoReplyFNBQA@fnb-corp.com subject:\"Document eDelivery Consent\"",
        "Document eDelivery Consent",
        2,
        90);

String token = OutlookMailReader.extract(
        body, "(?i)token\\D*?(\\d+)");
```

**Notes:**
- Depends on `OutlookBrowser` (Story 1)
- Uses Outlook web page elements (search box, message list, email body, received time). If Microsoft changes Outlook's layout, only this class needs updating
- The received time is compared with the local machine's clock

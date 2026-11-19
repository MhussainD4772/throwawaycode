**ETAF enhancement: Outlook Automation**

**Purpose**
Test scripts can now validate emails that the application under test sends, such as tokens, confirmations, reference numbers and links, as part of an end-to-end automated flow.

**How it works**

**OutlookBrowser: gets you into Outlook**
Opens Outlook in the test browser, already signed in as the tester running the test. It uses your Windows login, so no username or password is ever stored or typed. After the first sign-in, your session is remembered and Outlook opens directly on later runs. If the session expires, it signs you in again on its own.

**OutlookMailReader: finds the right email and reads it**
Searches Outlook for the email your test is waiting on and keeps checking until it arrives. Only a recent email is accepted, so a test never passes on an old one by mistake. Once found, it returns the email content and pulls out exactly the value your test needs, such as a token, a number or a link. If the email doesn't arrive in time, the test fails with a clear message.

Together, a test can trigger an action in the application, confirm the resulting email was received, and use its contents in the next step, all without manual checking.

**Usage**
To try it on your machine, add this class to your project, run it, then delete it:

```java
package com.ptaf.utils;

import com.microsoft.playwright.*;

public class OutlookCheck {
    public static void main(String[] args) {
        try (Playwright playwright = Playwright.create()) {
            BrowserContext outlook = OutlookBrowser.open(playwright);
            System.out.println("Outlook opened successfully");
            outlook.close();
        }
    }
}
```

The first run opens Edge and signs you in automatically. Later runs open Outlook directly.

**Artifact version supporting this feature**
```xml
<dependency>
    <groupId>com.fnb_ptaf</groupId>
    <artifactId>fnb_ptaf</artifactId>
    <version>1.0.0-10879</version>
</dependency>
```

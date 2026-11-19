**ETAF enhancement: Outlook Automation**

**Purpose**
Test scripts can now validate emails that the application under test sends, such as tokens, confirmations, reference numbers and links, as part of an end-to-end automated flow.

ETAF opens Outlook using the tester's own Windows login, with no credentials stored anywhere, so email-based steps no longer have to be checked by hand.

- **OutlookBrowser:** opens Outlook, already signed in
- **OutlookMailReader:** finds the expected email and extracts the value you need

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

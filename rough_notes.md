// Finds the newest token email in Outlook
// (must be under maxAgeMinutes old),
// reads the token, enters it in Connects.
@Then("^we get token from outlook email "
        + "from \"(.*?)\" with subject "
        + "\"(.*?)\" within (\\d+) minutes "
        + "and enter on main frame (.*?) "
        + "locator (.*?)$")
public void weGetTokenFromOutlookAndEnter(
        String sender,
        String subject,
        int maxAgeMinutes,
        String element,
        String locator) {

    String token;

    try (Playwright playwright =
                 Playwright.create()) {

        BrowserContext outlook =
                OutlookBrowser.open(playwright);
        try {
            Page mail = outlook.pages().get(0);

            String body =
                    OutlookMailReader.waitForEmail(
                        mail,
                        "from:" + sender
                        + " subject:\""
                        + subject + "\"",
                        subject,
                        maxAgeMinutes,
                        90);

            token = OutlookMailReader.extract(
                    body,
                    "(?i)token\\D*?(\\d+)");
        } finally {
            outlook.close();
        }
    }

    if (token == null) {
        throw new RuntimeException(
                "No token found in the email");
    }
    System.out.println(
            "Token from email: " + token);

    frameCommonMethods.fill(
            iframePage, iFrame, null, null,
            element, locator, token);
}











import com.microsoft.playwright.*;
import com.ptaf.utils.OutlookBrowser;
import com.ptaf.utils.OutlookMailReader;
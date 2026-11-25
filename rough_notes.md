// Checks the hidden document count field
// (two frames deep). Fails if the field is
// missing; passes only if there are no
// documents (value empty or 0).
@Then("^we verify on second frame "
        + "there are no documents$")
public void weVerifyNoDocuments() {

    Locator field = iframePage
            .frameLocator(iFrame)
            .frameLocator(iFrame_2)
            .locator("#hiddenRecordTotal");

    try {
        field.first().waitFor(
                new Locator.WaitForOptions()
                        .setState(WaitForSelectorState.ATTACHED)
                        .setTimeout(10000));
    } catch (PlaywrightException e) {
        throw new AssertionError(
                "hiddenRecordTotal not found. "
                + "Check the frame.");
    }

    String raw = field.first().getAttribute("value");
    String value = raw == null ? "" : raw.trim();
    System.out.println(
            "Document count value: [" + value + "]");

    int count;
    try {
        count = Integer.parseInt(value);
    } catch (NumberFormatException e) {
        count = 0;
    }

    if (count > 0) {
        throw new AssertionError(
                "Expected no documents, but found "
                + count);
    }
}


import com.microsoft.playwright.options.WaitForSelectorState;


Then we verify on second frame there are no documents

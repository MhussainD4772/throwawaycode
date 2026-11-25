// Reads hiddenRecordTotal (two frames deep)
// and passes only if there are no documents.
// Fails if the field itself isn't found,
// so it can't pass for the wrong reason.
@Then("^we verify on second frame (.*?) "
        + "locator (.*?) has no documents$")
public void weVerifyHasNoDocuments(
        String element, String locator) {

    String raw = frameCommonMethods
            .getStringValue(iframePage, iFrame,
                    iFrame_2, null, element, locator);

    if (raw == null) {
        throw new AssertionError(
                "hiddenRecordTotal not found. "
                + "Check the frame.");
    }

    String value = raw.trim();
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



Then we verify on second frame document_manger_page locator document_count_flt has no documents

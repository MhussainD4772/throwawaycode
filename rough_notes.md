// Generates a new address for the test case,
// using the street name from address.json
@Given("^we generate next address for testCase (.*?)$")
public void weGenerateNextAddress(String testCaseId)
        throws Exception {
    newAddress(testCaseId);
}

// Builds a new address and saves it as
// the current address for later steps
public static String newAddress(String testCaseId)
        throws Exception {
    com.google.gson.JsonObject streets =
            com.google.gson.JsonParser.parseString(
                java.nio.file.Files.readString(
                    java.nio.file.Paths.get(
                        "src/test/resources/data/address.json")))
            .getAsJsonObject();

    currentAddress = (System.currentTimeMillis() % 100000)
            + " " + streets.get(testCaseId).getAsString();

    System.out.println("Generated address: " + currentAddress);
    return currentAddress;
}




// Finds an address not yet in Navigator,
// retrying until the search returns
// no records, then writes it to Excel.
@Given("^we find unused address on "
        + "navigator frame (.*?) locator (.*?) "
        + "for testCase (.*?) and write to "
        + "excel column \"(.*?)\"$")
public void weFindUnusedAddress(
        String element,
        String locator,
        String testCaseId,
        String column) throws Exception {

    for (int attempt = 1;
         attempt <= 10; attempt++) {

        String address = FrameCommonSteps
                .newAddress(testCaseId);

        frameCommonMethods.fill(
                page, navigator_main, null, null,
                element, locator, address);
        frameCommonMethods.click(
                page, navigator_main, null, null,
                element, "submit_btn");
        page.waitForTimeout(3000);

        if (noRecordsFound(page)) {
            ExcelWriter.writeData(filePath,
                    testCaseId, column, address);
            System.out.println(
                    "Unused address: " + address);
            return;
        }
        System.out.println("Address " + address
                + " in use, trying another one");
    }
    throw new RuntimeException(
            "No unused address found "
            + "after 10 attempts");
}

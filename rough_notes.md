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

document_row: "xpath_(//*[@id='dgTransactions']//div[contains(@class,'DataGridContainerContent')]//tr[td[normalize-space()]])[1]"

home_phone_row: "xpath_//tr[td[normalize-space()='Home Phone']]"
primary_phone_row: "xpath_//tr[td[contains(normalize-space(),'Phone')]][.//img]"
email_row: "xpath_//tr[td[normalize-space()='E-Mail Address']]"


 When we click on navigator page navigator_page locator Contact_methods_tab
And time out for 2 seconds
Then we contain on navigator page navigator_page of locator home_phone_row value "<phone>"
And we contain on navigator page navigator_page of locator primary_phone_row value "Home Phone"
And we contain on navigator page navigator_page of locator email_row value "<email>"
And we capture screenshot on navigator page navigator_page locator body name "contact_methods"

estyle_eligible_flt: "xpath_(//tr[.//*[normalize-space()='eStyle Eligible Only']])[last()]/td[normalize-space()][2]"
date_residency_answered_flt: "xpath_(//tr[.//*[normalize-space()='Date Residency Question Answered']])[last()]/td[normalize-space()][2]"
country_of_citizenship_flt: "xpath_(//tr[.//*[normalize-space()='Country of Citizenship']])[last()]/td[normalize-space()][2]"
foreign_official_flt: "xpath_(//tr[.//*[normalize-space()='Customer a foreign official?']])[last()]/td[normalize-space()][2]"
family_foreign_official_flt: "xpath_(//tr[.//*[normalize-space()='Family or friend foreign official?']])[last()]/td[normalize-space()][2]"
diplomat_flt: "xpath_(//tr[.//*[normalize-space()='Is the customer a diplomat?']])[last()]/td[normalize-space()][2]"
accept_email_delivery_flt: "xpath_(//tr[.//*[normalize-space()='Accept E-mail Document Delivery']])[last()]/td[normalize-space()][2]"




// Checks the field shows today's date (MM/dd/yyyy)
@Then("^we contain today's date on navigator page (.*?) of locator (.*?)$")
public void weContainTodaysDateOnNavigatorPage(String element, String locator) {
    String today = java.time.LocalDate.now()
            .format(java.time.format.DateTimeFormatter.ofPattern("MM/dd/yyyy"));
    pageCommonMethods.contain(navigator_page, element, locator, today);
}




And we contain on navigator page navigator_page of locator estyle_eligible_flt value "N"
And we contain today's date on navigator page navigator_page of locator date_residency_answered_flt
And we contain on navigator page navigator_page of locator country_of_citizenship_flt value "United States"
And we contain on navigator page navigator_page of locator foreign_official_flt value "N"
And we contain on navigator page navigator_page of locator family_foreign_official_flt value "N"
And we contain on navigator page navigator_page of locator diplomat_flt value "N"
And we contain today's date on navigator page navigator_page of locator accept_email_delivery_flt
And we capture screenshot on navigator page navigator_page locator body name "additional_info"

document_count_flt: "xpath_//input[@id='hiddenRecordTotal' and number(@value) > 0]"

customer_header_bar: "xpath_//td[starts-with(normalize-space(),'Customer ') and contains(normalize-space(),' - ')][not(.//td)]"

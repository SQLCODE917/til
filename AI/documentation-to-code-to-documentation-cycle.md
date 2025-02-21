# Documentation -> Code -> Refactor -> Documentation

## Using Virtual Assistants as an early adopter

Let's explore an idea that code does not exist for it's own sake.
It exists only to solve problems.
Current virtual assistants allow us to generate code that almost doesn't work.
My personal judgement - they generate code that is on par with interns and junior developers.
Ask yourself - if you were leading a team of interns and juniors, would you be able to ship code that solves problems?
Don't spend too much effort on the answer - ofcourse you would,
and I'd expect you to,
and I'd provide you problems that were appropriate to your team's abilities to solve.
Because the programmer using virtual assitants is not that different from a foreman carpenter choosing woods in the construction of houses:

```
Straight un-knotted timber of good appearance is used for the revealed pillars, straight timber with small defects is used for the inner pillars.
Timbers of the finest appearance, even if a little weak, is used for the thresholds, lintels, doors, and sliding doors, and so on.
Good strong timber, though it be gnarled and knotted, can always be used discreetly in construction.
Timber which is weak or knotted throughout should be used as scaffolding, and later for firewood.
```

This will be the principle of my strategy.

## Goals

- Keep technical design always current
- Encourage modular, uncoupled, functional pipelines so code is easily testable without mocks
- Investigate technical design best practices that lends itself to handoffs between the programmer and the virtual assitant

## Approach

- Create a technical design with 1) functions with a clearly defined interface, pre and postcondition 2) is editable with simple tools
- chatGPT implements it
- Test it, note the misses, see how the technical design could be shortened
- Refactor the code, feed it back to chatGPT, let it regenerate updated documentation

## The Problem

You have a commercial kitchen making meals to be delivered to customers.
Chefs use a spreadsheet with tabs(sheets) for each day of the week they are cooking for.
Another spreadsheet is used for route planning, where the day's customers are manually entered.
Let's automate it!

## The Solution

### The technical design:

```
## Overview
The Importer program in Google Apps Script that runs on the backoffice delivery planning sheet.
It reads data from the chef-facing planning sheet.
It allows the backoffice planner to select a day from the chef-facing planning sheet, import the customers and comments left on customer cells.


## Data Flow
1. User invokes the `getMenuPlanningSheets` function
2. `getMenuPlanningSheets` gets all sheet names from the planning sheet and populates the dropdown with those values
3. User selects a sheet from the dropdown, invoking the `parseCustomersFromSheet`
4. `parseCustomersFromSheet` function reads the customer names and comments from that sheet's Row 1.
Having gotten the names and comments, it clears the backoffice planner sheet "Deliveries" and writes the names and comments to Row 1 of that sheet.


## Functions


### getMenuPlanningSheets
- **Precondition**:
  - `planningSheetUrl` (string): URL for the chef-facing planning spreadsheet
- **Postcondition**: In the backoffice delivery planning sheet, sheet "Enum", cell K1 becomes a dropdown, values are the names of the sheets in the chef-facing planning spreadsheet


### parseCustomersFromSheet
- **Precondition**:
  - `planningSheetUrl` (string): URL for the chef-facing planning spreadsheet
- **Inputs**:
  - `sheetName` (string): Selected value from the K1 dropdown
- **Postcondition**:
  - In the backoffice delivery planning sheet, sheet "Deliveries" row 1 is cleared
  - Gets the customer names and comments left on their names from row 1 from the chef-facing planning sheet using the `planningSheetUrl` and `sheetName`
  - Only gets the customer names and comments left on them if in the same column, there are values in rows 3 through 17
  - Deliveries row 1 is filled with the names of the customer who have deliveries that day
  - Copies the comments left on customer names from the chef-facing planning sheet to the names in Deliveries row 1
```

- The prompt:

`In Google Sheets Apps Script, implement the Importer from the attached design document`

### The generated code:

```
function getMenuPlanningSheets(planningSheetUrl) {
  var ss = SpreadsheetApp.openByUrl(planningSheetUrl);
  var sheets = ss.getSheets();
  var sheetNames = sheets.map(sheet => sheet.getName());

  var backofficeSheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Enum");
  var dropdownCell = backofficeSheet.getRange("K2");

  var rule = SpreadsheetApp.newDataValidation()
    .requireValueInList(sheetNames, true)
    .build();

  dropdownCell.setDataValidation(rule);
}

function parseCustomersFromSheet(planningSheetUrl, sheetName) {
  var planningSS = SpreadsheetApp.openByUrl(planningSheetUrl);
  var sourceSheet = planningSS.getSheetByName(sheetName);
  if (!sourceSheet) {
    Logger.log("Sheet not found: " + sheetName);
    return;
  }

  var lastColumn = sourceSheet.getLastColumn();
  var customerRow = sourceSheet.getRange(1, 1, 1, lastColumn).getValues()[0];
  var commentRow = sourceSheet.getRange(1, 1, 1, lastColumn).getNotes()[0];

  var deliveriesSheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Deliveries");
  deliveriesSheet.getRange(1, 1, 1, deliveriesSheet.getLastColumn()).clearContent();

  var customersWithDeliveries = [];
  var commentsForCustomers = [];

  for (var col = 0; col < lastColumn; col++) {
    var hasOrders = sourceSheet.getRange(2, col + 1, 16, 1).getValues().flat().some(value => value);
    if (hasOrders && customerRow[col]) {
      customersWithDeliveries.push(customerRow[col]);
      commentsForCustomers.push(commentRow[col] || "");
    }
  }

  if (customersWithDeliveries.length > 0) {
    deliveriesSheet.getRange(1, 1, 1, customersWithDeliveries.length).setValues([customersWithDeliveries]);
    deliveriesSheet.getRange(1, 1, 1, customersWithDeliveries.length).setNotes([commentsForCustomers]);
  }
}
```

### Code Review:

- off-by-1 error instantiating `hasOrders` - should use Row 3
- no sanitization of input: whitespace needs to be removed from `customerRow` values.
This is not specified in the design doc, so that's on me
- incorrect Google Sheets API assumption - notes are actually returned as a 2-dimensional 1x??? dimensional string array - `commentsForCustomers.push(commentRow[col]);` will not work

### Refactored Code:

```
const ImporterValidator = (function ImporterValidatorComponent(SpreadsheetApp){
  const PLANNING_SHEET_URL = 'https://docs.google.com/spreadsheets/d/[REDACTED]/edit';
  const DROPDOWN_CELL_ADDRESS = "K2";
  const SHEET_NAME = "Enum";

  function getMenuPlanningSheets() {
    var ss = SpreadsheetApp.openByUrl(PLANNING_SHEET_URL);
    var sheets = ss.getSheets();
    var sheetNames = sheets.map(sheet => sheet.getName());

    var backofficeSheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);
    var dropdownCell = backofficeSheet.getRange(DROPDOWN_CELL_ADDRESS);

    var rule = SpreadsheetApp.newDataValidation()
      .requireValueInList(sheetNames, true)
      .build();

    dropdownCell.setDataValidation(rule);
  }

  function parseCustomersFromSheet(sheetName) {
    var planningSS = SpreadsheetApp.openByUrl(PLANNING_SHEET_URL);
    var sourceSheet = planningSS.getSheetByName(sheetName);
    if (!sourceSheet) {
      Logger.log("Sheet not found: " + sheetName);
      return;
    }

    // Get The header row and comments from the PLANNING_SHEET
    var lastColumn = sourceSheet.getLastColumn();
    var customerRow = sourceSheet.getRange(1, 1, 1, lastColumn).getValues()[0];
    var commentRow = sourceSheet.getRange(1, 1, 1, lastColumn).getNotes();

    //Clear all content from the first row of the "Deliveries" sheet
    var deliveriesSheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Deliveries");
    deliveriesSheet.getRange(1, 1, 1, deliveriesSheet.getLastColumn()).clearContent();

    var customersWithDeliveries = [];
    var commentsForCustomers = [];

    for (var col = 0; col < lastColumn; col++) {
      var hasOrders = sourceSheet.getRange(3, col + 1, 17, 1).getValues().flat().some(value => value);
      if (hasOrders && customerRow[col]) {
        customersWithDeliveries.push(customerRow[col].toString().trim());
        commentsForCustomers.push(commentRow[0][col] || "");
      }
    }

    if (customersWithDeliveries.length > 0) {
      deliveriesSheet.getRange(1, 1, 1, customersWithDeliveries.length).setValues([customersWithDeliveries]);
      deliveriesSheet.getRange(1, 1, 1, customersWithDeliveries.length).setNotes([commentsForCustomers]);
    }
  }

  return {
    getMenuPlanningSheets,
    parseCustomersFromSheet,
  }
})(SpreadsheetApp)
```

### Design document update

- prompt:

(attached previous technical design document as importer.md)
`I have refactored the code - update the design document attached. Format it in markdown, output markdown only`

- output:

```
# Importer Validator Technical Design Document

## Overview

The `ImporterValidator` module is a Google Apps Script component that interacts with the backoffice delivery planning sheet. It facilitates importing customer names and comments from the chef-facing planning sheet and validating them against the planned deliveries.

## Data Flow

1. The user invokes `getMenuPlanningSheets()` to fetch all available sheet names from the chef-facing planning sheet and populate the dropdown list in the backoffice planner sheet.
2. The user selects a sheet from the dropdown, which triggers `parseCustomersFromSheet(sheetName)`.
3. `parseCustomersFromSheet(sheetName)` extracts customer names and comments from row 1 of the selected sheet.
4. The script verifies that customers have planned deliveries (by checking if rows 3-19 in the corresponding columns contain values).
5. If customers have deliveries, their names and comments are written to row 1 of the "Deliveries" sheet in the backoffice planner sheet.

## Functions

### `getMenuPlanningSheets()`

- **Preconditions:**
  - `PLANNING_SHEET_URL`: URL of the chef-facing planning spreadsheet is defined.
  - `SHEET_NAME`: The backoffice planning sheet contains a predefined sheet named "Enum".
  - `DROPDOWN_CELL_ADDRESS`: The target cell (K2) exists in the "Enum" sheet.
- **Postconditions:**
  - Populates cell `K2` in the "Enum" sheet with a dropdown list of available sheet names from the chef-facing spreadsheet.

### `parseCustomersFromSheet(sheetName)`

- **Preconditions:**
  - `PLANNING_SHEET_URL`: The chef-facing planning spreadsheet is accessible.
  - `sheetName`: A valid sheet name is passed as an argument.
- **Postconditions:**
  - Clears row 1 of the "Deliveries" sheet.
  - Extracts customer names and associated comments from row 1 of the selected chef-facing planning sheet.
  - Only considers customer names where corresponding column values exist in rows 3-19.
  - Populates row 1 of the "Deliveries" sheet with the filtered customer names.
  - Adds associated comments as notes to the corresponding customer names in row 1.

## Configuration

- **Constants:**
  - `PLANNING_SHEET_URL`: Hardcoded URL for the chef-facing planning spreadsheet.
  - `DROPDOWN_CELL_ADDRESS`: "K2" (modifiable if necessary).
  - `SHEET_NAME`: "Enum" (modifiable if necessary).
- **Logging:**
  - Logs messages for debugging, including customer names and their comments before writing to the "Deliveries" sheet.

## Example Usage

ImporterValidator.getMenuPlanningSheets();
ImporterValidator.parseCustomersFromSheet("Monday Deliveries");

## Error Handling

- Logs errors when:
  - The selected sheet does not exist.
  - The script encounters unexpected empty values in critical cells.
- Uses default values ("") when no comment exists for a customer.
```

- testing the new technical design document:

(attach new technical design)
`Implement the attached technical design in Google Sheets Apps Script`

```
var ImporterValidator = (function () {
  const PLANNING_SHEET_URL = "YOUR_CHEF_PLANNING_SHEET_URL";
  const DROPDOWN_CELL_ADDRESS = "K2";
  const ENUM_SHEET_NAME = "Enum";
  const DELIVERIES_SHEET_NAME = "Deliveries";

  function getMenuPlanningSheets() {
    try {
      var planningSpreadsheet = SpreadsheetApp.openByUrl(PLANNING_SHEET_URL);
      var sheetNames = planningSpreadsheet.getSheets().map(sheet => sheet.getName());

      var backofficeSpreadsheet = SpreadsheetApp.getActiveSpreadsheet();
      var enumSheet = backofficeSpreadsheet.getSheetByName(ENUM_SHEET_NAME);
      if (!enumSheet) {
        Logger.log("Enum sheet not found");
        return;
      }

      var dropdownRange = enumSheet.getRange(DROPDOWN_CELL_ADDRESS);
      var rule = SpreadsheetApp.newDataValidation().requireValueInList(sheetNames).build();
      dropdownRange.setDataValidation(rule);
    } catch (e) {
      Logger.log("Error fetching menu planning sheets: " + e.message);
    }
  }

  function parseCustomersFromSheet(sheetName) {
    try {
      var planningSpreadsheet = SpreadsheetApp.openByUrl(PLANNING_SHEET_URL);
      var selectedSheet = planningSpreadsheet.getSheetByName(sheetName);
      if (!selectedSheet) {
        Logger.log("Sheet not found: " + sheetName);
        return;
      }

      var firstRowValues = selectedSheet.getRange(1, 1, 1, selectedSheet.getLastColumn()).getValues()[0];
      var deliveryData = selectedSheet.getRange(3, 1, 17, selectedSheet.getLastColumn()).getValues();

      var validCustomers = [];
      var comments = {};

      for (var col = 0; col < firstRowValues.length; col++) {
        var customerName = firstRowValues[col];
        if (customerName && deliveryData.some(row => row[col])) {
          validCustomers.push(customerName);
          var comment = selectedSheet.getRange(1, col + 1).getNote() || "";
          comments[customerName] = comment;
        }
      }

      var backofficeSpreadsheet = SpreadsheetApp.getActiveSpreadsheet();
      var deliveriesSheet = backofficeSpreadsheet.getSheetByName(DELIVERIES_SHEET_NAME);
      if (!deliveriesSheet) {
        Logger.log("Deliveries sheet not found");
        return;
      }

      deliveriesSheet.getRange(1, 1, 1, deliveriesSheet.getLastColumn()).clear();
      if (validCustomers.length > 0) {
        var range = deliveriesSheet.getRange(1, 1, 1, validCustomers.length);
        range.setValues([validCustomers]);

        validCustomers.forEach((customer, index) => {
          if (comments[customer]) {
            deliveriesSheet.getRange(1, index + 1).setNote(comments[customer]);
          }
        });
      }

    } catch (e) {
      Logger.log("Error parsing customers from sheet: " + e.message);
    }
  }

  return {
    getMenuPlanningSheets: getMenuPlanningSheets,
    parseCustomersFromSheet: parseCustomersFromSheet
  };
})();
```

### Conclusion

It worked, I implemented and refactored a solution for the small, bounded problem with spreadsheet coordination.
Moreso, I did this late in the evening, yawning - the mental load was very slight, this approach could make late nights more productive.
Looks promissing enough to continue research into implement features with code that starts as merely an artifact of the documentation.

# Functional Requirement Document (FRD)

## Program: Excel File Handling — Bank Master Data Upload

**Program Name:** `ZEXCEL_FILEHANDLING`
**Module:** Custom Development (Z-Development)
**Related Table:** `ZBANK1`

---

## 1. Purpose

To provide a mechanism for business users to upload bank master data from a local Microsoft Excel (`.xlsx`) file directly into the SAP custom table `ZBANK1`, eliminating manual data entry and reducing data-entry errors, while preventing duplicate records.

---

## 2. Scope

This requirement covers:
- Selection and upload of a local Excel file into the SAP system.
- Parsing of the Excel worksheet data into an internal table.
- Validation and duplicate-check against existing table records.
- Insertion of new, non-duplicate records into `ZBANK1`.
- User feedback via system messages on success, failure, or duplication.

Out of scope:
- Updating or deleting existing records via the same upload.
- Uploading files in formats other than `.xlsx`.
- Uploading from more than one worksheet per file.

---

## 3. Business Requirement

The business needs a repeatable, self-service way to load bulk bank details (Bank ID, Customer ID, Bank Name, City, Country, Amount, Customer Number) into SAP from Excel files received from external sources (e.g., banking partners, finance team spreadsheets), without requiring manual entry into SM30/SE16N one record at a time.

---

## 4. Functional Requirements

| Req ID | Requirement | Description |
|--------|-------------|--------------|
| FR-01 | File Selection | User must be able to browse and select a local `.xlsx` file via a standard file-open dialog (F4 help) on the selection screen. |
| FR-02 | File Upload | The system shall upload the selected file in binary format and convert it to an internal XSTRING representation for processing. |
| FR-03 | Worksheet Parsing | The system shall read the first worksheet of the Excel file and convert its contents into an internal table using SAP's spreadsheet parsing class (`CL_FDT_XL_SPREADSHEET`). |
| FR-04 | Header Row Handling | The system shall treat the first row of the worksheet as a column header and exclude it from data processing. |
| FR-05 | Column Mapping | The system shall map worksheet columns to internal fields in the following fixed order: Bank ID, Customer ID, Bank Name, Bank City, Country, Amount, Customer Number. |
| FR-06 | Duplicate Check | Before inserting, the system shall check whether a record with the same Bank ID and Customer ID combination already exists in `ZBANK1`. |
| FR-07 | Record Insertion | If no matching record exists, the system shall insert the new record into `ZBANK1`. |
| FR-08 | Duplicate Skip | If a matching record already exists, the system shall skip insertion and notify the user that the record was skipped. |
| FR-09 | User Feedback | The system shall display a message for each record processed: success (insert), failure (insert error), or informational (duplicate skipped). |
| FR-10 | Data Verification | Users shall be able to verify uploaded records via standard SAP tools (SE16N / SE11 Table Contents / SM30). |

---

## 5. Data Requirements

### Source: Excel File Columns (in order)
1. Bank ID
2. Customer ID
3. Bank Name
4. Bank City
5. Country
6. Amount
7. Customer Number

### Target Table: ZBANK1

| Field       | Key | Data Type | Length | Decimals | Description       |
|-------------|-----|-----------|--------|----------|--------------------|
| ZBANK_ID    | ✔   | NUMC      | 20     | 0        | Bank ID            |
| ZCUST_ID    | ✔   | CHAR      | 4      | 0        | Customer ID        |
| ZBANK_NAME  |     | CHAR      | 10     | 0        | Bank Name          |
| ZBANK_CITY  |     | CHAR      | 30     | 0        | Bank City          |
| ZCOUNTRY    |     | CHAR      | 15     | 0        | Country            |
| ZAMOUNT11   |     | CURR      | 10     | 2        | Amount             |
| KUNNR       |     | CHAR      | 10     | 0        | Customer Number    |
| WAERS       |     | CUKY      | 5      | 0        | Currency Key       |

**Key Fields:** `ZBANK_ID` + `ZCUST_ID` (uniqueness check performed on this combination).

---

## 6. Process Flow

1. User executes the program via SE38/transaction code.
2. User selects the Excel file to upload using the file-browse helper.
3. System uploads the file and converts it to a processable format.
4. System reads the first worksheet and extracts all rows, skipping the header.
5. System maps each row's columns to the internal data structure.
6. System checks each record against existing entries in `ZBANK1` by Bank ID + Customer ID.
7. New records are inserted; existing (duplicate) records are skipped.
8. System displays a message per record indicating the outcome.
9. User verifies the uploaded data via SE16N/SE11/SM30.

---

## 7. Non-Functional Requirements

| Req ID | Requirement |
|--------|-------------|
| NFR-01 | The upload process should handle files with at least 500 data rows without performance degradation. |
| NFR-02 | The duplicate-check query shall use `FOR ALL ENTRIES` to minimize individual database round-trips. |
| NFR-03 | Error messages shall clearly indicate the reason for failure (e.g., "Binary upload failed", "No worksheets found", "Could not read worksheet data"). |
| NFR-04 | The solution shall not require any manual pre-processing of the Excel file (e.g., no need to save as CSV first). |

---

## 8. Assumptions

- The Excel file always has exactly one relevant worksheet, and it is the first sheet in the workbook.
- The first row of the worksheet always contains column headers.
- Column order in the Excel file will always match the expected order (Bank ID, Customer ID, Bank Name, Bank City, Country, Amount, Customer Number).
- Users have appropriate authorization to execute the upload program and to insert data into `ZBANK1`.

## 9. Constraints

- Only `.xlsx` format is supported (binary/XSTRING + `CL_FDT_XL_SPREADSHEET` parsing).
- The program does not currently support updating existing records — only insert of new, non-duplicate rows.
- No rollback/undo mechanism is provided within the program; corrections must be done manually via SE16N/SM30 or a separate delete utility.

## 10. Acceptance Criteria

- Given a valid Excel file with new bank records, when the user runs the program and selects the file, then all new records are inserted into `ZBANK1` and a success message is shown for each.
- Given an Excel file containing a record that already exists in `ZBANK1` (same Bank ID + Customer ID), when the user runs the program, then that record is skipped and an informational message is shown.
- Given an invalid or corrupted file, when the user attempts to upload it, then the system displays an appropriate error message and does not proceed with insertion.
- Given a successful upload, when the user checks the table via SE16N or SE11 Table Contents, then all newly inserted records are visible with correct field values.

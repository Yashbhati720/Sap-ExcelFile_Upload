# Excel File Handling — Upload to SAP Table (ZEXCEL_FILEHANDLING)

## Overview

This program (`ZEXCEL_FILEHANDLING`) reads data from a local **Excel (.xlsx)** file, parses it, and inserts the records into a custom SAP transparent table (`ZBANK1`) — with duplicate-checking to avoid inserting records that already exist.

**Concepts Covered:** File Upload (Binary), XSTRING conversion, `CL_FDT_XL_SPREADSHEET` (Excel parsing), Dynamic field mapping, `FOR ALL ENTRIES`, Table `INSERT`

---

## 1. Target Table: ZBANK1

**T-Code:** `SE11` → Table: `ZBANK1`
**Short Description:** Bank detail

| Field       | Key | Data Element | Data Type | Length | Decimals | Short Description |
|-------------|-----|--------------|-----------|--------|----------|--------------------|
| ZBANK_ID    | ✔   | ZBANK_ID     | NUMC      | 20     | 0        | Bank ID            |
| ZCUST_ID    | ✔   | ZCUST_ID     | CHAR      | 4      | 0        | Customer ID        |
| ZBANK_NAME  |     | ZBANK_NAME   | CHAR      | 10     | 0        | Bank Name          |
| ZBANK_CITY  |     | ZBANK_CITY   | CHAR      | 30     | 0        | Bank City          |
| ZCOUNTRY    |     | ZCOUNTRY     | CHAR      | 15     | 0        | Country            |
| ZAMOUNT11   |     | ZAMOUNT11    | CURR      | 10     | 2        | Amount             |
| KUNNR       |     | KUNNR        | CHAR      | 10     | 0        | Customer Number    |
| WAERS       |     | WAERS        | CUKY      | 5      | 0        | Currency Key       |

`ZBANK_ID` + `ZCUST_ID` together form the primary key.

---

## 2. Program Structure: ZEXCEL_FILEHANDLING

**T-Code:** `SE38` → Program: `ZEXCEL_FILEHANDLING`

### Selection Screen
```abap
PARAMETERS: p_file TYPE localfile.

AT SELECTION-SCREEN ON VALUE-REQUEST FOR p_file.
  CALL FUNCTION 'F4_FILENAME'
    EXPORTING
      program_name  = syst-cprog
      dynpro_number = syst-dynnr
      field_name    = 'P_FILE'
    IMPORTING
      file_name     = p_file.
```
- A single input field `P_FILE` with a file-browse (F4) helper to pick the local Excel file (e.g. `C:\Users\junabap11\Downloads\demo_bank_data.xlsx`).

### Data Structures
```abap
TYPES: BEGIN OF ty_data,
         bank_id   TYPE zbank_id,
         cust_id   TYPE zcust_id,
         bank_name TYPE zbank_name,
         bank_city TYPE zbank_city,
         country   TYPE zcountry,
         amount11  TYPE zamount11,
         kunnr     TYPE kunnr,
       END OF ty_data.

DATA: lt_data TYPE TABLE OF ty_data,
      ls_data TYPE ty_data.

DATA: lt_bank TYPE TABLE OF zbank1,
      ls_bank TYPE zbank1.
```

### Processing Steps

**Step 1 — Upload the file as binary and convert to XSTRING**
```abap
CALL FUNCTION 'GUI_UPLOAD'
  EXPORTING
    filename   = lv_file
    filetype   = 'BIN'
  IMPORTING
    filelength = lv_filelen
  TABLES
    data_tab   = lt_raw.

CALL FUNCTION 'SCMS_BINARY_TO_XSTRING'
  EXPORTING
    input_length = lv_filelen
  IMPORTING
    buffer       = lv_xdata
  TABLES
    binary_tab   = lt_raw.
```

**Step 2 — Parse the XLSX using `CL_FDT_XL_SPREADSHEET`**
```abap
lo_excel = NEW cl_fdt_xl_spreadsheet(
             document_name = lv_file
             xdocument     = lv_xdata ).

lo_excel->if_fdt_doc_spreadsheet~get_worksheet_names(
  IMPORTING worksheet_names = lt_sheets ).

READ TABLE lt_sheets INTO lv_sheet INDEX 1.   " first sheet
lo_data_ref = lo_excel->if_fdt_doc_spreadsheet~get_itab_from_worksheet( lv_sheet ).
ASSIGN lo_data_ref->* TO <lt_dyn_tab>.
```

**Step 3 — Map dynamic rows into the typed structure (by column position)**
```abap
LOOP AT <lt_dyn_tab> ASSIGNING <ls_dyn_row>.
  lv_idx = sy-tabix.
  IF lv_idx = 1.
    CONTINUE.   " skip header row
  ENDIF.

  CLEAR ls_data.
  ASSIGN COMPONENT 1 OF STRUCTURE <ls_dyn_row> TO <lv_cell>.
  IF sy-subrc = 0. ls_data-bank_id = <lv_cell>. ENDIF.
  " ... (repeated for columns 2–7: cust_id, bank_name, bank_city, country, amount11, kunnr)

  APPEND ls_data TO lt_data.
ENDLOOP.
```

**Step 4 — Compare against the database and insert new records**
```abap
IF lt_data IS NOT INITIAL.
  SELECT * FROM zbank1
    INTO TABLE lt_bank
    FOR ALL ENTRIES IN lt_data
    WHERE zbank_id = lt_data-bank_id
      AND zcust_id = lt_data-cust_id.
ENDIF.

LOOP AT lt_data INTO ls_data.
  READ TABLE lt_bank INTO ls_bank
    WITH KEY zbank_id = ls_data-bank_id
             zcust_id = ls_data-cust_id.

  IF sy-subrc <> 0.
    " Build ls_bank from ls_data and INSERT into zbank1
    INSERT zbank1 FROM ls_bank.
    IF sy-subrc = 0.
      MESSAGE 'Record inserted successfully' TYPE 'S'.
    ELSE.
      MESSAGE 'Failed to insert record' TYPE 'E'.
    ENDIF.
  ELSE.
    MESSAGE 'Record already exists, skipped' TYPE 'I'.
  ENDIF.
ENDLOOP.
```

---

## 3. Execution Walkthrough

1. **Run the program:** `SE38` → `ZEXCEL_FILEHANDLING` → Execute (F8)
2. **Selection screen** ("excel file Handling") appears — enter/browse to the file:
   `C:\Users\junabap11\Downloads\demo_bank_data.xlsx`
3. Program uploads the file, parses the worksheet, skips the header row, and inserts each new row into `ZBANK1` (skipping duplicates already present).

---

## 4. Verifying the Uploaded Data

**T-Code:** `SE11` → Table `ZBANK1` → **Utilities → Table Contents → Display** (`Ctrl+Shift+F10`)

- The **Data Browser: Table ZBANK1: Selection Screen** opens with all fields available as optional filters (left blank to see everything).
- Execute (F8) to run the query.

### Result
**Data Browser: Table ZBANK1 Select Entries — 20** rows returned, confirming a successful upload, e.g.:

| ZBANK_ID | ZCUST_ID | ZBANK_NAME | ZBANK_CITY | ZCOUNTRY       | ZAMOUNT11   | KUNNR       |
|----------|----------|------------|------------|----------------|-------------|-------------|
| ...0001  | CU14     | HSBC       | London     | United Kingdom | 371,033.70  | 0004108604  |
| ...0002  | CU32     | BNP Pariba | Paris      | France         | 368,499.14  | 0009149733  |
| ...0007  | CU36     | Deutsche B | Frankfurt  | Germany        | 349,371.56  | 0005708457  |
| ...0008  | CU35     | ICICI Bank | Mumbai     | India          | 108,441.57  | 0005647120  |
| ...      | ...      | ...        | ...        | ...            | ...         | ...         |

*(20 records total, spanning banks across UK, France, Australia, Switzerland, Germany, India, Singapore, Japan, and USA.)*

Alternative ways to verify the same data: `SE16N`, `SM30` (if TMG exists), or `SELECT ... INTO TABLE` + `cl_demo_output=>display( )` from ABAP code.

---

## Summary

This program demonstrates a full Excel-to-database import flow in SAP ABAP:

1. Select a local `.xlsx` file via a selection-screen parameter with F4 help.
2. Upload it as binary and convert to XSTRING.
3. Parse the spreadsheet into a dynamic internal table using `CL_FDT_XL_SPREADSHEET`.
4. Map each row into a typed structure matching table `ZBANK1`.
5. Check for existing records (by `ZBANK_ID` + `ZCUST_ID`) and insert only new ones.
6. Verify the results directly in the database table via SE11/SE16N.

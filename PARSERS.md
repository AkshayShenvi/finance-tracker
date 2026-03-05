# Supported Parsers (CSV & PDF)

This document lists all supported parsers for importing transactions into Finance Tracker.

## Currently Supported Banks/Cards (3 Total)

### CSV Parsers

1. **American Express (AMEX)** - CSV
   - Parser ID: `amex`
   - Display Name: "American Express"
   - Date Format: MM/DD/YYYY
   - Amount Convention: Positive = Expense, Negative = Credit/Refund
   - Expected Headers: Date, Description, Amount
   - File: `amex_parser.py`

2. **Discover Bank/Card** - CSV
   - Parser ID: `discover_bank`
   - Display Name: "Discover Bank"
   - Date Format: MM/DD/YYYY
   - Amount Convention: Positive = Expense, Negative = Credit/Refund
   - Expected Headers: Trans. Date, Post Date, Description, Amount, Category
   - File: `discover_bank_parser.py`

### PDF Parsers

3. **Chase Bank** - PDF ⭐ *NEW!*
   - Parser ID: `chase_bank_pdf`
   - Display Name: "Chase Bank PDF Statement"
   - Date Format: MM/DD (year inferred from statement)
   - Amount Convention: Negative = Expense/Withdrawal, Positive = Deposit/Income
   - File Type: PDF statement files
   - File: `chase_bank_pdf_parser.py`
   - Supports: Chase College Checking, Chase Total Checking statements

---

## How to Use

1. **Create an Account**: Go to Dashboard → Manage Accounts → Create Account
2. **Link Parser**: Select the appropriate parser from the dropdown (e.g., "Chase Bank PDF Statement" for Chase bank statements)
3. **Import File**: Upload your CSV or PDF file from your bank - the system will auto-detect the format
4. **Review & Import**: Preview transactions, assign categories, and confirm import

## PDF vs CSV

- **CSV Files**: Typically downloaded from online banking as transaction exports
- **PDF Files**: Bank statements in PDF format - now supported for Chase Bank!
- Both formats are automatically detected during import

---

## Auto-Detection

The system automatically detects which parser to use based on CSV headers and format. You can also manually select a parser during import.

---

## Adding New Parsers

To add support for a new bank:

1. Create a new parser file in `finance-tracker-backend/app/services/csv_import/parsers/`
2. Inherit from `CSVParser` base class
3. Implement required methods: `get_name()`, `get_display_name()`, `detect()`, `parse()`
4. Register in `parsers/__init__.py` and `csv_import/__init__.py`
5. Test with sample CSV files

See `discover_bank_parser.py` for a recent example.

---

## Parser Priority

When multiple parsers match a CSV file, the system uses the one with the highest confidence score. Parsers with unique headers get higher confidence.

---

## Recently Added

- **2026-02-18**:
  - Discover Bank CSV parser added
  - Chase Bank PDF parser added (first PDF parser!)
  - **PDF Support**: System now accepts both CSV and PDF files
  - Account-Parser Linking: Accounts can now be linked to default parsers

## Technical Implementation

### PDF Parsing
- Uses `pdfplumber` library for extracting text and tables from PDF statements
- Automatically detects Chase Bank statements by looking for specific markers
- Extracts transaction date, description, and amount from statement text
- Handles year rollover (December → January) automatically

## Note

The codebase previously included CSV parsers for Citi, Capital One, Bank of America, and Wells Fargo. These have been removed to focus on the active parsers. To add them back or add new parsers, follow the "Adding New Parsers" instructions above.

# Project Overview

This repository hosts a data processing project that reads an Excel file, converts it to CSV, processes the data using a Python script, and publishes the results via GitHub Pages.

## Project Structure

-   `data.xlsx`: The original Excel data file.
-   `data.csv`: The CSV version of `data.xlsx`, generated during CI.
-   `execute.py`: A Python script responsible for processing the data.
-   `.github/workflows/ci.yml`: GitHub Actions workflow for Continuous Integration and Deployment.
-   `index.html`: A simple, responsive web page built with Tailwind CSS.
-   `LICENSE`: The MIT License.

## Setup and Installation

### Prerequisites

-   Python 3.11+
-   Pandas 2.3+
-   Ruff (for linting)
-   `openpyxl` (for reading .xlsx files with pandas)

### Local Development

1.  **Clone the repository:**
    ```bash
    git clone <repository-url>
    cd <repository-name>
    ```

2.  **Create a virtual environment and activate it:**
    ```bash
    python -m venv venv
    source venv/bin/activate  # On Windows: `venv\Scripts\activate`
    ```

3.  **Install dependencies:**
    ```bash
    pip install pandas ruff openpyxl
    ```

4.  **Create `data.xlsx` (if not already present):**
    For local testing, you can create a `data.xlsx` file with content similar to the example below.

5.  **Convert `data.xlsx` to `data.csv`:**
    This step is automated in CI, but for local execution, you can run:
    ```python
    import pandas as pd
    df = pd.read_excel('data.xlsx')
    df.to_csv('data.csv', index=False)
    ```

6.  **Run the data processing script:**
    ```bash
    python execute.py > result.json
    ```
    This will generate `result.json` in your current directory.

## `data.xlsx` and `data.csv`

The `data.xlsx` file serves as the raw input. An example structure is shown below:

| Category | Value   |
| :------- | :------ |
| A        | 100     |
| B        | 150     |
| A        | 200     |
| C        | 50      |
| B        | 120     |
| A        | Invalid |
| D        | 75      |

This `data.xlsx` is converted to `data.csv` in the CI pipeline (or manually for local runs) before `execute.py` processes it. The `data.csv` would look like this:

```csv
Category,Value
A,100
B,150
A,200
C,50
B,120
A,Invalid
D,75
```

## `execute.py` - Data Processing Script (Fixed)

The `execute.py` script is designed to read data from `data.csv`, perform aggregations (sum, mean, count per category), and output the results in JSON format. The original script contained a non-trivial error related to robust column handling and data type conversion (e.g., assuming clean column names or direct numeric conversion). This has been fixed to: 

1.  Strip whitespace from column names.
2.  Explicitly check for required columns (`Category`, `Value`).
3.  Convert the `Value` column to numeric, coercing errors and dropping unconvertible rows.
4.  Handle potential `NaN` values in the final summary when converting to JSON.

```python
import pandas as pd
import json
import sys

def process_data(input_csv_path="data.csv"):
    """
    Reads data from a CSV, performs aggregation, and returns results.
    Assumes a CSV with 'Category' and 'Value' columns.
    """
    try:
        df = pd.read_csv(input_csv_path)

        # Non-trivial fix: Ensure column names are stripped of whitespace
        # to prevent KeyError issues (e.g., ' Category' vs 'Category').
        df.columns = df.columns.str.strip()
        
        # Another fix/robustness: Explicitly check for expected columns
        required_columns = ['Category', 'Value']
        for col in required_columns:
            if col not in df.columns:
                raise ValueError(
                    f"Required column '{col}' not found in '{input_csv_path}'. "
                    f"Available columns: {', '.join(df.columns)}"
                )

        # Ensure 'Value' column is numeric for aggregation
        df['Value'] = pd.to_numeric(df['Value'], errors='coerce')
        df.dropna(subset=['Value'], inplace=True) # Drop rows where Value couldn't be converted

        # Perform aggregation
        summary = df.groupby('Category')['Value'].agg(['sum', 'mean', 'count']).to_dict('index')

        # Convert to a JSON-serializable format, handling NaN if any
        result = {
            "summary_stats": {
                category: {
                    stat: (value if pd.notna(value) else None) for stat, value in stats.items()
                } for category, stats in summary.items()
            },
            "total_rows_processed": len(df)
        }
        return result

    except FileNotFoundError:
        print(f"Error: The input file '{input_csv_path}' was not found.", file=sys.stderr)
        sys.exit(1)
    except Exception as e:
        print(f"An unexpected error occurred during data processing: {e}", file=sys.stderr)
        sys.exit(1)

if __name__ == "__main__":
    output_data = process_data()
    if output_data:
        # Output JSON to stdout. The CI workflow redirects this to result.json.
        print(json.dumps(output_data))
```

## Continuous Integration and Deployment (CI/CD)

This project leverages GitHub Actions for automated testing and deployment. The workflow defined in `.github/workflows/ci.yml` performs the following steps on every `push` event to `main` (and pull requests):

```yaml
name: Python CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write # Needed for actions/checkout and artifact upload
      pages: write    # Needed for deploying to GitHub Pages
      id-token: write # Needed for deploying to GitHub Pages with OIDC

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install pandas ruff openpyxl

      - name: Convert data.xlsx to data.csv
        run: |
          python -c "import pandas as pd; df = pd.read_excel('data.xlsx'); df.to_csv('data.csv', index=False)"
          ls -l # Optional: For debugging, shows data.csv is created

      - name: Run Ruff Linter
        run: |
          ruff check .
          ruff format . --check # Optional: check formatting too

      - name: Run execute.py and generate result.json
        run: |
          python execute.py > result.json
          cat result.json # Optional: For debugging, shows content

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Upload result.json for Pages deployment
        uses: actions/upload-pages-artifact@v3
        with:
          path: 'result.json' # The artifact to upload for Pages

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

The `result.json` file is explicitly *not* committed to the repository; it is an artifact generated solely by the CI process. After a successful deployment, the `result.json` will be accessible via GitHub Pages, typically at `https://<your-username>.github.io/<repository-name>/result.json` (replace placeholders).

## License

This project is open-source and available under the MIT License. See the `LICENSE` file for more details.
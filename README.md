# Advanced IMEI Processing System

![IMEI Processor](https://img.shields.io/badge/IMEI-Processor-blue)
![Python](https://img.shields.io/badge/Python-3.7%2B-green)
![License](https://img.shields.io/badge/License-MIT-yellow)

A professional tool for processing, validating, and comparing IMEI numbers across different files and folders with customizable output options.

## Features

- **Multiple Comparison Modes**
  - Compare with existing IMEIs (filter out duplicates)
  - Compare within the IMEI directory (identify duplicates)
  - Perform both comparisons simultaneously

- **Advanced IMEI Validation**
  - Luhn algorithm validation for 15-digit IMEIs
  - Support for 16-digit IMEISV numbers
  - Detailed validation diagnostics and error reporting

- **Intelligent Model Detection**
  - Automatic model detection from IMEI TAC (Type Allocation Code)
  - Filename-based model detection
  - Reassignment of Unknown models to specific models when possible

- **Comprehensive Reporting**
  - Detailed model-by-model IMEI reports
  - Invalid IMEI diagnostics and statistics
  - Cross-model duplicate detection
  - Processing statistics and summaries

## Installation

### Prerequisites

- Python 3.7 or higher
- Required Python packages: pandas, tqdm

### Installation Steps

1. Clone this repository:
    git clone https://github.com/YourUsername/imei-processor.git
cd imei-processor


2. Install required dependencies:
   pip install -r requirements.txt

## Usage

Run the script directly:


## Usage

Run the script directly:

The interactive setup will guide you through configuring:
- Input and output directories
- Comparison mode
- Output detail level
- Model detection options
- Validation strictness

## What is an IMEI?

IMEI (International Mobile Equipment Identity) is a unique 15-digit number assigned to every mobile device. The 16-digit variant (IMEISV) includes a software version number. This tool helps manage, validate, and organize these identifiers across device models.

### Example Configuration
=== ADVANCED IMEI PROCESSOR CONFIGURATION ===

--- Directory Configuration ---
Enter IMEI directory path [C:\Users\user\Downloads\IMEI]: 
Enter existing IMEI directory path [C:\Users\user\Downloads\Existing]: 
Enter output directory path [C:\Users\user\Downloads\IMEI_Output]: 

--- Processing Mode ---
1. Compare with existing IMEIs (filter out duplicates)
2. Compare only within the IMEI directory (identify duplicates)
3. Do both comparisons (most comprehensive)
Select comparison mode [1]: 1

[Configuration continues...]

## Supported Models

The system is configured to recognize the following models by default:
- Y03T
- V40 Lite 
- Y19S
- Y28
- Y29
- Y04
- V30
- V40
- V50

Custom models can be specified during setup.

### Configuration Options

#### Processing Modes

1. **Compare with existing IMEIs**: Filters out duplicates found in the existing directory
2. **Compare within IMEI directory**: Identifies duplicates across different models
3. **Both comparisons**: The most comprehensive analysis

#### Output Organization

The tool creates a structured output folder:

IMEI_Output/

├── Reports/
│ ├── 1_Before_Comparison/
│ │ ├── Models/
│ │ └── Consolidated_1_Before_Comparison.xlsx
│ ├── 2_After_Comparison/
│ │ ├── Models/
│ │ └── Consolidated_2_After_Comparison.xlsx
│ ├── 3_Within_Directory/
│ │ └── Cross_Model_Duplicates/
│ ├── Invalid_IMEIs/
│ │ ├── Invalid_IMEIs.xlsx
│ │ └── Invalid_IMEI_Summary.xlsx
│ ├── Audit_Logs/
│ └── Summary/
│ ├── processing_summary.txt
│ └── processing_summary.xlsx


## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Troubleshooting

- **Invalid IMEIs**: If many IMEIs are reported as invalid, check the format of your source files.
- **No files found**: Ensure your Excel files have standard .xls or .xlsx extensions.
- **Model detection issues**: Model names in filenames should match the configured model list.


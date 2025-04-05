#!/usr/bin/env python3
"""
Enhanced IMEI Processing System
-------------------------------
A user-friendly tool for processing IMEI numbers from Excel files.
Validates, compares, and organizes IMEIs with customizable options.
"""

import os
import re
import sys
import logging
import traceback
from pathlib import Path
from typing import Dict, List, Set, Tuple
from datetime import datetime
from collections import defaultdict

import pandas as pd
from tqdm import tqdm

# Version information
__version__ = "1.2.0"


class IMEIValidator:
    """Handles IMEI validation with comprehensive error reporting."""
    
    @staticmethod
    def clean_imei(imei: str) -> str:
        """Remove all non-digit characters from an IMEI string."""
        return re.sub(r'\D', '', str(imei).strip())
    
    @staticmethod
    def luhn_check(imei: str) -> bool:
        """
        Perform Luhn algorithm check on a 15-digit IMEI.
        This is an industry-standard validation for IMEIs.
        """
        if len(imei) != 15 or not imei.isdigit():
            return False
            
        total = 0
        for i, digit in enumerate(reversed(imei)):
            d = int(digit)
            # Double every second digit from right to left
            if i % 2 == 1:
                d *= 2
                # If doubling results in a two-digit number, add the digits
                if d > 9:
                    d -= 9
            total += d
            
        # The number is valid if the sum mod 10 is 0
        return total % 10 == 0
    
    def validate(self, imei: str, strict: bool = True) -> Tuple[bool, str, Dict]:
        """
        Comprehensively validate an IMEI number.
        
        Args:
            imei: The IMEI string to validate
            strict: Whether to perform strict validation with Luhn check
            
        Returns:
            Tuple of (is_valid, reason, diagnostics)
        """
        original = str(imei).strip()
        clean_imei = self.clean_imei(original)
        
        diagnostics = {
            'original': original,
            'cleaned': clean_imei,
            'length': len(clean_imei)
        }
        
        # Empty check
        if not clean_imei:
            return False, "Empty value after cleaning", diagnostics
        
        # Length validation
        if len(clean_imei) not in (15, 16):
            return False, f"Invalid length ({len(clean_imei)} digits)", diagnostics
        
        # Character validation
        if not clean_imei.isdigit():
            return False, "Contains non-digit characters", diagnostics
        
        # For 16-digit IMEIs (IMEISV), we don't do Luhn check
        if len(clean_imei) == 16:
            return True, "Valid IMEISV (16 digits)", diagnostics
        
        # Skip Luhn check if not in strict mode
        if not strict:
            return True, "Valid IMEI (basic check)", diagnostics
        
        # Luhn check for 15-digit IMEIs
        luhn_valid = self.luhn_check(clean_imei)
        diagnostics['luhn_check'] = luhn_valid
        
        if luhn_valid:
            return True, "Valid IMEI (passed Luhn check)", diagnostics
        else:
            return False, "Failed Luhn check", diagnostics


class IMEIProcessor:
    """Main processing engine with flexible file handling and output options."""
    
    def __init__(self, config):
        """Initialize with user configuration."""
        self.config = config
        self.validator = IMEIValidator()
        self.logger = self._setup_logger()
        self.stats = {
            'start_time': datetime.now(),
            'processed_files': 0,
            'skipped_files': 0,
            'total_imeis': 0,
            'unique_imeis': 0,
            'duplicates_with_existing': 0,
            'duplicates_within_directory': 0,
            'invalid_imeis': 0,
            'model_counts': defaultdict(int),
            'errors': [],
            'invalid_reasons': defaultdict(int),
            'length_distribution': defaultdict(int)
        }
        
        # Create output directories
        self._create_directories()
    
    def _setup_logger(self):
        """Configure logging based on user preference."""
        logger = logging.getLogger("imei_processor")
        logger.setLevel(logging.INFO)
        
        # Clear any existing handlers
        if logger.handlers:
            for handler in logger.handlers:
                logger.removeHandler(handler)
        
        # Console handler
        console = logging.StreamHandler()
        console.setLevel(logging.INFO)
        
        # Simple formatter for console
        simple_format = logging.Formatter("%(message)s")
        console.setFormatter(simple_format)
        logger.addHandler(console)
        
        # File handler if logging enabled
        if self.config['save_logs']:
            log_file = os.path.join(
                self.config['output_dir'], 
                f"imei_process_log_{datetime.now().strftime('%Y%m%d_%H%M%S')}.log"
            )
            file_handler = logging.FileHandler(log_file)
            file_handler.setLevel(logging.DEBUG)
            detailed_format = logging.Formatter("%(asctime)s - %(levelname)s - %(message)s")
            file_handler.setFormatter(detailed_format)
            logger.addHandler(file_handler)
            
        return logger
    
    def _create_directories(self):
        """Create necessary output directories."""
        # Sanitize the output directory path to prevent errors
        self.config['output_dir'] = sanitize_directory_path(self.config['output_dir'])
        
        self.reports_dir = os.path.join(self.config['output_dir'], "Reports")
        
        # Create report subdirectories based on comparison mode
        if self.config['comparison_mode'] in ('with_existing', 'both'):
            self.before_dir = os.path.join(self.reports_dir, "1_Before_Comparison")
            self.after_dir = os.path.join(self.reports_dir, "2_After_Comparison")
            os.makedirs(self.before_dir, exist_ok=True)
            os.makedirs(self.after_dir, exist_ok=True)
            
            # Model subdirectories
            if self.config['save_by_model']:
                self.before_model_dir = os.path.join(self.before_dir, "Models")
                self.after_model_dir = os.path.join(self.after_dir, "Models")
                os.makedirs(self.before_model_dir, exist_ok=True)
                os.makedirs(self.after_model_dir, exist_ok=True)
            
        if self.config['comparison_mode'] in ('within_directory', 'both'):
            self.within_dir = os.path.join(self.reports_dir, "3_Within_Directory")
            os.makedirs(self.within_dir, exist_ok=True)
            
            # Cross-model duplicates directory
            self.duplicates_dir = os.path.join(self.within_dir, "Cross_Model_Duplicates")
            os.makedirs(self.duplicates_dir, exist_ok=True)
            
        # Create invalid IMEIs directory
        self.invalid_dir = os.path.join(self.reports_dir, "Invalid_IMEIs")
        os.makedirs(self.invalid_dir, exist_ok=True)
            
        # Create audit logs directory
        self.audit_dir = os.path.join(self.reports_dir, "Audit_Logs")
        os.makedirs(self.audit_dir, exist_ok=True)
        
        # Create summary directory
        self.summary_dir = os.path.join(self.reports_dir, "Summary")
        os.makedirs(self.summary_dir, exist_ok=True)
        
        # Ensure the main directories exist
        os.makedirs(self.config['output_dir'], exist_ok=True)
        os.makedirs(self.reports_dir, exist_ok=True)
            
        self.logger.info(f"Output will be saved to: {self.config['output_dir']}")
    
    def read_imeis_from_file(self, file_path: str) -> Set[str]:
        """Extract valid IMEIs from an Excel file."""
        valid_imeis = set()
        
        try:
            # Read all sheets if multiple exist
            if file_path.endswith(('.xls', '.xlsx')):
                excel_data = pd.read_excel(file_path, sheet_name=None, header=None)
                
                for sheet_name, df in excel_data.items():
                    for col in df.columns:
                        for cell in df[col].dropna():
                            # Convert to string and clean
                            cell_str = str(cell).strip()
                            is_valid, reason, diagnostics = self.validator.validate(
                                cell_str, self.config['strict_validation']
                            )
                            
                            # Track length distribution
                            self.stats['length_distribution'][diagnostics['length']] += 1
                            
                            if is_valid:
                                valid_imeis.add(diagnostics['cleaned'])
                            else:
                                self.stats['invalid_imeis'] += 1
                                self.stats['invalid_reasons'][reason] += 1
                                
                                if self.config['detailed_output']:
                                    self.logger.debug(
                                        f"Invalid IMEI in {os.path.basename(file_path)}, "
                                        f"Sheet: {sheet_name}, "
                                        f"Value: '{cell_str}', "
                                        f"Reason: {reason}"
                                    )
            else:
                self.logger.warning(f"Skipped non-Excel file: {file_path}")
                
        except Exception as e:
            self.logger.error(f"Error reading {os.path.basename(file_path)}: {str(e)}")
            self.stats['errors'].append(f"File: {file_path}, Error: {str(e)}")
            self.stats['skipped_files'] += 1
            
        return valid_imeis
    
    def detect_model_from_filename(self, filename: str) -> str:
        """Detect model name from the filename."""
        filename_lower = filename.lower()
        
        for model in self.config['models']:
            if model.lower() in filename_lower:
                return model
                
        return "Unknown"
    
    def process_directory(self, directory: str) -> Dict[str, Set[str]]:
        """Process all Excel files in a directory, grouped by model."""
        model_imeis = defaultdict(set)
        
        # Check if directory exists
        if not os.path.exists(directory):
            self.logger.warning(f"Directory not found: {directory}")
            return model_imeis
        
        # Get all Excel files in the directory
        all_files = []
        for root, _, files in os.walk(directory):
            for file in files:
                if file.endswith(('.xls', '.xlsx')):
                    all_files.append(os.path.join(root, file))
        
        if not all_files:
            self.logger.warning(f"No Excel files found in {directory}")
            return model_imeis
            
        self.logger.info(f"Found {len(all_files)} Excel files in {directory}")
        
        # Process each file with progress bar
        for file_path in tqdm(all_files, desc=f"Processing {os.path.basename(directory)}", 
                             disable=not self.config['show_progress']):
            try:
                file_name = os.path.basename(file_path)
                
                # Determine model from filename
                model = self.detect_model_from_filename(file_name)
                
                # Read IMEIs from file
                imeis = self.read_imeis_from_file(file_path)
                
                if imeis:
                    model_imeis[model].update(imeis)
                    self.stats['model_counts'][model] += len(imeis)
                    self.stats['processed_files'] += 1
                    self.stats['total_imeis'] += len(imeis)
                    
                    if self.config['detailed_output']:
                        self.logger.info(f"Processed {file_name} → {model}: {len(imeis)} IMEIs")
            
            except Exception as e:
                self.logger.error(f"Failed to process {file_path}: {str(e)}")
                self.stats['errors'].append(f"File: {file_path}, Error: {str(e)}")
                self.stats['skipped_files'] += 1
        
        return model_imeis
    
    def compare_with_existing(self, imei_data: Dict[str, Set[str]], existing_imeis: Set[str]) -> Dict[str, Set[str]]:
        """Compare with existing IMEIs and filter out duplicates."""
        filtered_data = defaultdict(set)
        
        for model, imeis in imei_data.items():
            new_imeis = imeis - existing_imeis
            filtered_data[model] = new_imeis
            removed = len(imeis) - len(new_imeis)
            self.stats['duplicates_with_existing'] += removed
            
            if self.config['detailed_output']:
                self.logger.info(f"Model {model}: Removed {removed} existing IMEIs")
                
        return filtered_data
    
    def find_duplicates_within_directory(self, imei_data: Dict[str, Set[str]]) -> Dict[str, List[str]]:
        """Find duplicates across different models within the directory."""
        imei_to_models = defaultdict(list)
        duplicates = defaultdict(list)
        
        # Map each IMEI to the models it appears in
        for model, imeis in imei_data.items():
            for imei in imeis:
                imei_to_models[imei].append(model)
        
        # Find IMEIs that appear in multiple models
        for imei, models in imei_to_models.items():
            if len(models) > 1:
                # This IMEI appears in multiple models
                duplicates[imei] = models
                self.stats['duplicates_within_directory'] += 1
                
                if self.config['detailed_output']:
                    self.logger.info(f"Duplicate IMEI {imei[:6]}... found in models: {', '.join(models)}")
        
        return duplicates
    
    def process_imeis(self):
        """Process IMEIs based on selected comparison mode."""
        self.logger.info("Starting IMEI processing...")
        
        # Process main IMEI directory in all modes
        self.logger.info(f"Processing main IMEI directory: {self.config['imei_dir']}")
        imei_data = self.process_directory(self.config['imei_dir'])
        
        # Save raw data from IMEI directory (Before comparison)
        if self.config['comparison_mode'] in ('with_existing', 'both'):
            self.save_model_results(imei_data, "1_Before_Comparison")
        
        # Process existing directory if needed
        if self.config['comparison_mode'] in ('with_existing', 'both'):
            self.logger.info(f"Processing existing IMEI directory: {self.config['existing_dir']}")
            existing_data = self.process_directory(self.config['existing_dir'])
            
            # Flatten existing IMEIs for filtering
            existing_imeis = set()
            for model_imeis in existing_data.values():
                existing_imeis.update(model_imeis)
            
            self.logger.info(f"Found {len(existing_imeis)} existing IMEIs for comparison")
            
            # Filter out existing IMEIs
            filtered_data = self.compare_with_existing(imei_data, existing_imeis)
            
            # Save filtered results
            self.save_model_results(filtered_data, "2_After_Comparison")
        
        # Find duplicates within directory if needed
        if self.config['comparison_mode'] in ('within_directory', 'both'):
            duplicates = self.find_duplicates_within_directory(imei_data)
            
            # Save results with duplicate information
            if self.config['comparison_mode'] == 'within_directory':
                self.save_model_results(imei_data, "3_Within_Directory")
            
            # Save duplicates report
            self.save_duplicates_report(duplicates)
        
        # Save invalid IMEI report
        self.save_invalid_imei_report()
        
        # Update unique IMEIs count
        unique_count = self.stats['total_imeis']
        if self.config['comparison_mode'] in ('with_existing', 'both'):
            unique_count -= self.stats['duplicates_with_existing']
        
        self.stats['unique_imeis'] = unique_count
        
        # Generate final summary report
        self.generate_summary_report()
    
    def save_invalid_imei_report(self):
        """Save report of invalid IMEIs with details."""
        if self.stats['invalid_imeis'] == 0:
            self.logger.info("No invalid IMEIs found.")
            return
            
        # Create summary of invalid reasons
        reason_data = []
        for reason, count in sorted(self.stats['invalid_reasons'].items(), key=lambda x: x[1], reverse=True):
            reason_data.append({
                'Reason': reason,
                'Count': count,
                'Percentage': f"{(count / self.stats['invalid_imeis']) * 100:.1f}%"
            })
            
        reason_df = pd.DataFrame(reason_data)
        reason_path = os.path.join(self.invalid_dir, "Invalid_IMEI_Reasons.xlsx")
        reason_df.to_excel(reason_path, index=False)
        
        self.logger.info(f"Saved invalid IMEI report to {reason_path}")
        self.logger.info(f"Total invalid IMEIs: {self.stats['invalid_imeis']}")
    
    def save_model_results(self, data: Dict[str, Set[str]], stage: str):
        """Save processed data to Excel files."""
        if not data:
            self.logger.warning(f"No data to save for {stage}")
            return
            
        stage_dir = os.path.join(self.reports_dir, stage)
        os.makedirs(stage_dir, exist_ok=True)
        
        # Remove Unknown model from data if configured
        if 'Unknown' in data and not self.config.get('include_unknown', True):
            unknown_count = len(data['Unknown'])
            if unknown_count > 0:
                self.logger.info(f"Excluding {unknown_count} Unknown model IMEIs from output")
                data = {k: v for k, v in data.items() if k != 'Unknown'}
            
        # Determine the largest set of IMEIs for column sizing
        max_length = max((len(imeis) for imeis in data.values()), default=0)
        
        # Prepare data for consolidated file
        consolidated_data = {}
        for model, imeis in data.items():
            if imeis:  # Only include models with IMEIs
                sorted_imeis = sorted(imeis)
                consolidated_data[model] = sorted_imeis + [None] * (max_length - len(sorted_imeis))
        
        # Add metadata sheet with summary statistics
        metadata = []
        for model, imeis in data.items():
            if imeis:
                metadata.append({
                    'Model': model,
                    'IMEI Count': len(imeis),
                    'Percentage': f"{(len(imeis) / max(1, sum(len(x) for x in data.values()))) * 100:.1f}%"
                })
                
        metadata_df = pd.DataFrame(metadata)
        
        # Save consolidated file if we have data
        if consolidated_data:
            # Save as Excel with multiple sheets
            consolidated_path = os.path.join(stage_dir, f"Consolidated_{stage}.xlsx")
            
            with pd.ExcelWriter(consolidated_path, engine='openpyxl') as writer:
                # Main IMEI data
                df = pd.DataFrame(consolidated_data)
                df.to_excel(writer, sheet_name='IMEI_Data', index=False)
                
                # Metadata/Summary sheet
                metadata_df.to_excel(writer, sheet_name='Summary', index=False)
                
                # Processing info sheet
                info_data = {
                    'Info': [
                        'Processing Date',
                        'Total IMEIs',
                        'Models Included',
                        'Stage',
                        'Version'
                    ],
                    'Value': [
                        datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                        sum(len(imeis) for imeis in data.values()),
                        ', '.join(data.keys()),
                        stage,
                        __version__
                    ]
                }
                pd.DataFrame(info_data).to_excel(writer, sheet_name='Info', index=False)
            
            self.logger.info(f"Saved consolidated report to {consolidated_path}")
            
            # Save individual model files if configured
            if self.config['save_by_model']:
                model_dir = os.path.join(stage_dir, "Models")
                os.makedirs(model_dir, exist_ok=True)
                
                for model, imeis in data.items():
                    if imeis:
                        model_df = pd.DataFrame({"IMEI": sorted(imeis)})
                        safe_model = re.sub(r'[^\w\-]', '_', model)
                        model_path = os.path.join(model_dir, f"{safe_model}.xlsx")
                        model_df.to_excel(model_path, index=False)
    
    def save_duplicates_report(self, duplicates: Dict[str, List[str]]):
        """Save report of IMEIs found in multiple models."""
        if not duplicates:
            self.logger.info("No duplicates found across models")
            return
            
        # Convert to DataFrame for easier reporting
        duplicate_data = []
        for imei, models in duplicates.items():
            duplicate_data.append({
                'IMEI': imei,
                'Found In Models': ', '.join(models),
                'Model Count': len(models)
            })
        
        # Sort by number of models (most problematic first)
        duplicate_data.sort(key=lambda x: x['Model Count'], reverse=True)
        
        # Create and save the report
        df = pd.DataFrame(duplicate_data)
        duplicates_path = os.path.join(self.duplicates_dir, "Cross_Model_Duplicates.xlsx")
        df.to_excel(duplicates_path, index=False)
        
        # Create summary by model
        model_summary = defaultdict(int)
        for item in duplicate_data:
            models = item['Found In Models'].split(', ')
            for model in models:
                model_summary[model] += 1
                
        summary_data = []
        for model, count in sorted(model_summary.items(), key=lambda x: x[1], reverse=True):
            summary_data.append({
                'Model': model,
                'Duplicate IMEIs': count,
                'Percentage': f"{(count / len(duplicate_data)) * 100:.1f}%"
            })
            
        summary_df = pd.DataFrame(summary_data)
        summary_path = os.path.join(self.duplicates_dir, "Duplicate_Model_Summary.xlsx")
        summary_df.to_excel(summary_path, index=False)
        
        self.logger.info(f"Saved duplicates report to {duplicates_path}")
        self.logger.info(f"Found {len(duplicates)} IMEIs duplicated across multiple models")
    
    def generate_summary_report(self):
        """Generate a user-friendly summary report."""
        duration = datetime.now() - self.stats['start_time']
        
        # Create different reports based on comparison mode
        report = [
            "\n" + "="*50,
            "           IMEI PROCESSING SUMMARY           ",
            "="*50,
            f"Start time: {self.stats['start_time'].strftime('%Y-%m-%d %H:%M:%S')}",
            f"End time: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}",
            f"Duration: {duration.total_seconds():.2f} seconds",
            f"\nComparison mode: {self.config['comparison_mode']}",
            f"\nFiles processed: {self.stats['processed_files']}",
            f"Files skipped: {self.stats['skipped_files']}",
            f"\nTotal IMEIs found: {self.stats['total_imeis']}"
        ]
        
        if self.config['comparison_mode'] in ('with_existing', 'both'):
            report.append(f"Duplicates with existing: {self.stats['duplicates_with_existing']}")
            
        if self.config['comparison_mode'] in ('within_directory', 'both'):
            report.append(f"Cross-model duplicates: {self.stats['duplicates_within_directory']}")
            
        report.extend([
            f"Unique IMEIs: {self.stats['unique_imeis']}",
            f"Invalid IMEIs: {self.stats['invalid_imeis']}"
        ])
        
        if self.config['detailed_output']:
            # Add model counts
            report.append("\nIMEI counts by model:")
            for model, count in sorted(self.stats['model_counts'].items()):
                report.append(f"  - {model}: {count}")
                
            # Add invalid reason summary if any invalid IMEIs
            if self.stats['invalid_imeis'] > 0:
                report.append("\nInvalid IMEI reasons:")
                for reason, count in sorted(self.stats['invalid_reasons'].items(), 
                                            key=lambda x: x[1], reverse=True):
                    report.append(f"  - {reason}: {count}")
                
            # Add length distribution
            report.append("\nIMEI length distribution:")
            for length, count in sorted(self.stats['length_distribution'].items()):
                if length > 0:  # Skip zero length which is often from empty cells
                    report.append(f"  - {length} digits: {count}")
                
            # Add errors summary
            if self.stats['errors']:
                report.append("\nErrors encountered:")
                for i, error in enumerate(self.stats['errors'][:5]):  # Show only first 5 errors
                    report.append(f"  {i+1}. {error}")
                if len(self.stats['errors']) > 5:
                    report.append(f"  ... and {len(self.stats['errors'])-5} more errors")
        
        # Add output location
        report.append(f"\nResults saved to: {self.config['output_dir']}")
        report.append("="*50)
        
        # Print report
        self.logger.info("\n".join(report))
        
        # Save report to file
        report_path = os.path.join(self.summary_dir, "processing_summary.txt")
        with open(report_path, "w") as f:
            f.write("\n".join(report))
            
        # Create Excel summary
        summary_data = {
            "Metric": [
                "Files Processed", "Files Skipped", 
                "Total IMEIs", "Unique IMEIs", 
                "Invalid IMEIs", "Processing Time (seconds)"
            ],
            "Value": [
                self.stats['processed_files'], self.stats['skipped_files'],
                self.stats['total_imeis'], self.stats['unique_imeis'], 
                self.stats['invalid_imeis'], f"{duration.total_seconds():.2f}"
            ]
        }
        
        # Add comparison-specific metrics
        if self.config['comparison_mode'] in ('with_existing', 'both'):
            summary_data["Metric"].append("Duplicates With Existing")
            summary_data["Value"].append(self.stats['duplicates_with_existing'])
            
        if self.config['comparison_mode'] in ('within_directory', 'both'):
            summary_data["Metric"].append("Cross-Model Duplicates")
            summary_data["Value"].append(self.stats['duplicates_within_directory'])
        
        # Add model counts
        for model, count in sorted(self.stats['model_counts'].items()):
            summary_data["Metric"].append(f"Model: {model}")
            summary_data["Value"].append(count)
            
        summary_df = pd.DataFrame(summary_data)
        summary_excel = os.path.join(self.summary_dir, "processing_summary.xlsx")
        summary_df.to_excel(summary_excel, index=False)
        
        # Save detailed error log if there were errors
        if self.stats['errors']:
            error_log_path = os.path.join(
                self.audit_dir, 
                f"error_log_{datetime.now().strftime('%Y%m%d_%H%M%S')}.txt"
            )
            with open(error_log_path, 'w') as f:
                f.write(f"IMEI Processing Error Log\n")
                f.write(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n\n")
                for i, error in enumerate(self.stats['errors']):
                    f.write(f"ERROR {i+1}:\n{error}\n\n")
        
        self.logger.info(f"Summary reports saved to {self.summary_dir}")


def sanitize_directory_path(path):
    """Sanitize the directory path to remove invalid characters."""
    if not path:
        return ""
    return path.replace('[', '').replace(']', '').replace('"', '').replace("'", "")


def get_user_config():
    """Interactive configuration with user-friendly prompts."""
    config = {}
    
    print("\n" + "="*60)
    print("           IMEI PROCESSING SYSTEM SETUP           ")
    print("="*60)
    print("\nThis tool helps you process, validate, and compare IMEI numbers.")
    print("Let's set up how you want to process your files.")
    
    # Default directories based on OS
    if sys.platform.startswith('win'):
        default_imei = os.path.join(os.path.expanduser("~"), "Downloads", "IMEI")
        default_existing = os.path.join(os.path.expanduser("~"), "Downloads", "Existing")
        default_output = os.path.join(os.path.expanduser("~"), "Downloads", "IMEI_Output")
    else:
        default_imei = os.path.join(os.path.expanduser("~"), "Downloads", "IMEI")
        default_existing = os.path.join(os.path.expanduser("~"), "Downloads", "Existing")
        default_output = os.path.join(os.path.expanduser("~"), "Downloads", "IMEI_Output")
    
    # Directory paths with sanitization and checking
    print("\n--- FOLDER LOCATIONS ---")
    print("Where are your files located? (Press Enter to use defaults)")
    
    imei_dir = input(f"IMEI files folder [{default_imei}]: ").strip() or default_imei
    config['imei_dir'] = sanitize_directory_path(imei_dir)
    
    existing_dir = input(f"Existing IMEI files folder [{default_existing}]: ").strip() or default_existing
    config['existing_dir'] = sanitize_directory_path(existing_dir)
    
    output_dir = input(f"Where to save results [{default_output}]: ").strip() or default_output
    config['output_dir'] = sanitize_directory_path(output_dir)
    
    # Directory existence warning
    for name, path in [("IMEI", config['imei_dir']), ("Existing", config['existing_dir'])]:
        if not os.path.exists(path):
            print(f"⚠️ Warning: {name} directory '{path}' doesn't exist. Will be created if needed.")
    
    # Comparison mode with explanation
    print("\n--- PROCESSING MODE ---")
    print("1. Compare with existing IMEIs (remove duplicates)")
    print("   - Use this to find new IMEIs not in your existing list")
    print("   - Creates Before/After comparison reports")
    print("\n2. Find duplicates within IMEI directory")
    print("   - Use this to find IMEIs appearing in multiple models")
    print("   - Helps identify cross-model duplications")
    print("\n3. Do both comparisons (most comprehensive)")
    print("   - Performs both operations for complete analysis")
    
    mode_choice = input("\nSelect processing mode [1]: ").strip() or "1"
    if mode_choice == "1":
        config['comparison_mode'] = 'with_existing'
    elif mode_choice == "2":
        config['comparison_mode'] = 'within_directory'
    else:
        config['comparison_mode'] = 'both'
    
    # Output options
    print("\n--- OUTPUT OPTIONS ---")
    detail_choice = input("Show detailed processing information? (y/n) [y]: ").strip().lower() or "y"
    config['detailed_output'] = detail_choice.startswith('y')
    
    progress_choice = input("Show progress bars during processing? (y/n) [y]: ").strip().lower() or "y"
    config['show_progress'] = progress_choice.startswith('y')
    
    model_choice = input("Create separate files for each model? (y/n) [y]: ").strip().lower() or "y"
    config['save_by_model'] = model_choice.startswith('y')
    
    unknown_choice = input("Include 'Unknown' models in output? (y/n) [y]: ").strip().lower() or "y"
    config['include_unknown'] = unknown_choice.startswith('y')
    
    # Validation options
    print("\n--- VALIDATION OPTIONS ---")
    print("IMEI validation checks the format and mathematical correctness")
    print("Strict mode applies the Luhn check algorithm for 15-digit IMEIs")
    strict_choice = input("Use strict IMEI validation? (y/n) [y]: ").strip().lower() or "y"
    config['strict_validation'] = strict_choice.startswith('y')
    
    # Logging options
    logs_choice = input("\nSave processing logs to file? (y/n) [y]: ").strip().lower() or "y"
    config['save_logs'] = logs_choice.startswith('y')
    
    # Phone models - provide examples and ask for customization
    print("\n--- PHONE MODELS ---")
    default_models = ["Y03T", "V40 Lite", "Y19S", "Y28", "Y29", "Y04", "V30", "V40", "V50"]
    print(f"Default models: {', '.join(default_models)}")
    print("\nThe tool identifies phone models from Excel filenames.")
    print("For example, a file named 'Y28_IMEIs.xlsx' will be associated with model 'Y28'.")
    custom_choice = input("\nWould you like to customize these models? (y/n) [n]: ").strip().lower() or "n"
    
    if custom_choice.startswith('y'):
        custom_models = input("Enter your models (comma-separated): ").strip()
        if custom_models:
            config['models'] = [model.strip() for model in custom_models.split(',')]
        else:
            config['models'] = default_models
    else:
        config['models'] = default_models
    
    print("\n" + "="*60)
    print("✅ Configuration complete! Ready to process your files.")
    print("="*60 + "\n")
    return config


def main():
    """Main entry point with error handling and user guidance."""
    try:
        # Print welcome message
        print("\n" + "="*60)
        print("      IMEI PROCESSING SYSTEM v" + __version__)
        print("="*60)
        print("\nThis tool processes IMEI numbers from Excel files.")
        print("It can validate IMEIs, compare with existing lists,")
        print("find duplicates, and create organized reports.")
        
        # Get user configuration
        config = get_user_config()
        
        # Initialize processor
        processor = IMEIProcessor(config)
        
        # Process and compare
        processor.process_imeis()
        
        print("\n" + "="*60)
        print("✅ Processing completed successfully!")
        print(f"📂 Results saved to: {config['output_dir']}")
        print("="*60)
        
        return 0
        
    except KeyboardInterrupt:
        print("\n\nProcess interrupted by user. Exiting...")
        return 1
    except Exception as e:
        print(f"\n❌ Error: {str(e)}")
        print("\nDetailed error information:")
        traceback.print_exc()
        print("\nIf this problem persists, please check your files and settings.")
        return 1


if __name__ == "__main__":
    sys.exit(main())

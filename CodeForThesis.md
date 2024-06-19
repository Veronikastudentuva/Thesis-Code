# Code for Thesis 

# This code is divided into three sections. In section 1, a combination of ocr and regular expressions are used. In section 2 the method of template based zonal ocr is used. The third section details the "drag and drop" zonal ocr method. 

# Section 1: Combination of OCR and Regular Expressions

# Install and setup the required packages
!pip install pymupdf
!pip install PyMuPDF
!pip install PyMuPDF googletrans==4.0.0-rc1
!pip install googletrans==4.0.0-rc1
!pip install deep-translator
!pip install langdetect

from google.colab import files
import fitz
from typing import TextIO
import re
import pandas as pd

from langdetect import detect, DetectorFactory
from langdetect.lang_detect_exception import LangDetectException
from deep_translator import GoogleTranslator
from deep_translator.exceptions import NotValidPayload, NotValidLength

# Upload the PDF document and then the text from it will be extracted with line separators to read it more clearly

def extract_text_from_pdf(pdf_path):
    uploaded_document = fitz.open(pdf_path)
    text = ""
    for page_num in range(len(uploaded_document)):
        page = uploaded_document[page_num]
        text += page.get_text("text") + "\n"
    return text
uploaded = files.upload()
for pdf_file in uploaded.keys():
    text = extract_text_from_pdf(pdf_file)

# The code below is functions which use regular expressions to match patterns to look for the desired attributes:


# This function finds the kwy value pairs and is meant to work with different spacing, whether the value is on the same line or on a different line than the key
# As in the documents we see ":" before any attribute we use that to guide whether the value is 
def extract_key_value(text, key):
    lines = text.split('\n')
    value = None
    for i, line in enumerate(lines):
        if key in line:
            match = re.search(rf'{key}\s*:\s*(.*)', line)
            if match and match.group(1).strip():
                value = match.group(1).strip()
            else:
                if i + 1 < len(lines) and lines[i + 1].strip() == ':':
                    for j in range(i + 2, len(lines)):
                        if lines[j].strip():
                            value = lines[j].strip()
                            break
                elif i + 1 < len(lines) and re.match(r'\s*:\s*(.*)', lines[i + 1].strip()):
                    match = re.match(r'\s*:\s*(.*)', lines[i + 1].strip())
                    if match and match.group(1).strip():
                        value = match.group(1).strip()
                elif i + 1 < len(lines) and lines[i + 1].strip():
                    value = lines[i + 1].strip()
            break
    return value

def extract_key_value_name(text, key):
    lines = text.split('\n')
    value = None
    for i, line in enumerate(lines):
        if key in line:
            match = re.search(rf'{key}\s*:\s*(.*)', line)
            if match and match.group(1).strip():
                value = match.group(1).strip()
            else:
                if i + 1 < len(lines) and lines[i + 1].strip() == ':':
                    for j in range(i + 2, len(lines)):
                        if lines[j].strip():
                            value = lines[j].strip()
                            break
                elif i + 1 < len(lines) and re.match(r'\s*:\s*(.*)', lines[i + 1].strip()):
                    match = re.match(r'\s*:\s*(.*)', lines[i + 1].strip())
                    if match and match.group(1).strip():
                        value = match.group(1).strip()
                elif i + 1 < len(lines) and lines[i + 1].strip():
                    value = lines[i + 1].strip()
            break
    return value

# The CAS Number always follows the same pattern 
# It can be made up from up to 10 numbers. They are separated into 3 different groups with "-". The first part has 2-7 digits, second has 2 digits and thirs has 1.
def extract_cas_number(text):
    cas_pattern = r'\b\d{2,7}-\d{2}-\d\b'
    match = re.search(cas_pattern, text)
    if match:
        return match.group(0)
    return None

def extract_ec_no(text):
    ec_no_pattern = r'\b(2|4|5)\d{2}-\d{3}-\d\b'
    match = re.search(ec_no_pattern, text)
    if match:
        return match.group(0)
    return None

def extract_key_value2(text, keys):
    lines = text.split('\n')
    value = None
    for key in keys:
        for i, line in enumerate(lines):
            if key in line:
                match = re.search(rf'{key}\s*:\s*(.*)', line)
                if match and match.group(1).strip():
                    value = match.group(1).strip()
                else:
                    if i + 1 < len(lines) and lines[i + 1].strip() == '':
                        if i + 2 < len(lines) and lines[i + 2].strip() == ':':
                            if i + 3 < len(lines):
                                value = lines[i + 3].strip()
                    elif i + 1 < len(lines) and lines[i + 1].strip() == ':':
                        if i + 2 < len(lines):
                            value = lines[i + 2].strip()
                    elif i + 1 < len(lines):
                        value = lines[i + 1].strip()
                if value:
                    break
        if value:
            break
    return value

def uses(text):
    # Remove the first instance of "Recommended use"
    text = re.sub(r'Recommended use', '', text, count=1, flags=re.IGNORECASE)

    if "Identified uses:" in text:
        return extract_key_value2(text, ["Identified uses"])
    elif "Substance/Mixture" in text:
        return extract_key_value2(text, ["Substance/Mixture"])
    elif "Recommended use" in text:
        return extract_key_value2(text, ["Recommended use"])
    else:
        return None

# We check if its hazardous by looking for specific phrases which say that it is not hazardous, so if such a phrase is not found, we assume it is hazardous 
def check_hazard(text):
    lower_text = text.lower()
    if "has not been classified as dangerous" in lower_text or "not a hazardous" in lower_text or "not classified as dangerous" in lower_text:
        return "No"
    return "Yes"

#ADR is labelled internally within the client company
def check_adr(text):
    return "None"

def UN(text):
    pattern = r'\bUN\s\d{4}\b'
    match = re.search(pattern, text)
    if match:
        return match.group(0)
    return None

def extract_supplier(text):
    lines = text.split('\n')
    supplier = None
    for i, line in enumerate(lines):
        if "Supplier" in line or "Company" in line:
            match = re.search(r'(Supplier|Company)\s*:\s*(.*)', line)
            if match and match.group(2).strip():
                supplier = match.group(2).strip()
            else:
                if i + 1 < len(lines) and lines[i + 1].strip() == '':
                    if i + 2 < len(lines) and lines[i + 2].strip() == ':':
                        if i + 3 < len(lines):
                            supplier = lines[i + 3].strip()
                elif i + 1 < len(lines) and lines[i + 1].strip() == ':':
                    if i + 2 < len(lines):
                        supplier = lines[i + 2].strip()
                elif i + 1 < len(lines):
                    supplier = lines[i + 1].strip()
            break
    return supplier

#ADD COMMENT
def second_extract_chemical_name_method(text):
    lines = [line.strip() for line in text.split('\n') if line.strip()]
    chemical_names = []
    keywords = ["CAS-No.", "EC-No.", "Index-No.", "Registration number", "Classification", "Concentration"]
    exclusion_patterns = [r'\d', r'%', r'Not Assigned', r'(\s*:\s*)', r'^\s*$',  r'XXXX' ]
    exclude_lines = keywords + exclusion_patterns
    for i, line in enumerate(lines):
        if "Chemical name" in line:
            for j in range(i + 1, len(lines)):
                if re.search(r'section', lines[j], re.IGNORECASE):
                    break  
                if any(re.search(pattern, lines[j]) for pattern in exclude_lines):
                    continue
                chemical_names.append(lines[j])
            break
    return ', '.join(chemical_names)

    
# Section 2


# Section 3





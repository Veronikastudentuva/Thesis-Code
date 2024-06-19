# Code for Thesis 

# This code is divided into three sections. In section 1, a combination of ocr and regular expressions are used. In section 2 the method of template based zonal ocr is used. The third section details the "drag and drop" zonal ocr method. 

# Section 1: Combination of OCR and Regular Expressions

#1.1 Install and setup the required packages
#We will use the python library PyMuPDF 

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

#1.2 Upload the PDF document and then the text from it will be extracted with line separators to read it more clearly

    def pdfcontent(pdf_path):
        uploaded_document = fitz.open(pdf_path)
        text = ""
        for page_num in range(len(uploaded_document)):
            page = uploaded_document[page_num]
            text += page.get_text("text") + "\n"
        return text
    uploaded = files.upload()
    for pdf_file in uploaded.keys():
        text = pdfcontent(pdf_file)

#1.3 The code below is functions which use regular expressions to match patterns to look for the desired attributes:


#This function finds the key value pairs and is meant to work with different spacing, whether the value is on the same line or on a different line than the key
#As in the documents we see ":" before any attribute we use that to guide whether the value is 
#It assumes that the key is followed by ":" then the value. We account for possible differences in spacing, such as some values being within the same line as the key, while others are a line or two below the key 
#This occurs as in the pdfs the data is othen in tables on in different lines, which is then reflected as we extract the text 

    # sometimes there is only 1 key and other times there are multiple so we want to iterate through the list of kwys 
    def keyvaluepair(text, keys):
        if isinstance(keys, str):
            keys = [keys]
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
    
#The CAS Number always follows the same pattern 
#It can be made up from up to 10 numbers. They are separated into 3 different groups with "-". The first part has 2-7 digits, second has 2 digits and third has 1.

    def casnumber(text):
        cas_pattern = r'\b\d{2,7}-\d{2}-\d\b'
        match = re.search(cas_pattern, text)
        if match:
            return match.group(0)
        return None

    def ecnumber(text):
        ec_no_pattern = r'\b(2|4|5)\d{2}-\d{3}-\d\b'
        match = re.search(ec_no_pattern, text)
        if match:
            return match.group(0)
        return None
        
#This function relies on the previously defined function to match key value pairs
#In this case the keys are 'Identified uses', 'Substance/Mixture' and 'Recommended use' as we are assuming all those have the same meaning for our purpose

    def uses(text):
        #We want to be able to seach for these patterns both if they are in capital and lower cases so use the package "IGNORECASE" from regular expressions for a case-insesitive search
        text = re.sub(r'Recommended use', '', text, count=1, flags=re.IGNORECASE)
        if "Identified uses:" in text:
            return keyvaluepair(text, ["Identified uses"])
        elif "Substance/Mixture" in text:
            return keyvaluepair(text, ["Substance/Mixture"])
        elif "Recommended use" in text:
            return keyvaluepair(text, ["Recommended use"])
        else:
            return None

#We check if its hazardous by looking for specific phrases which say that it is not hazardous, so if such a phrase is not found, we assume it is hazardous 

    def isitahazard(text):
        lower_text = text.lower()
        if "has not been classified as dangerous" in lower_text or "not a hazardous" in lower_text or "not classified as dangerous" in lower_text:
            return "No"
        return "Yes"

#ADR is labelled internally within the client company

    def adr_(text):
        return "None"

    def UN(text):
        pattern = r'\bUN\s\d{4}\b'
        match = re.search(pattern, text)
        if match:
            return match.group(0)
        return None

    def supplier(text):
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

    def extract_chemical_name(text):
        chemical_name = extract_key_value(text, "Chemical name")
        if chemical_name == "CAS-No." or not chemical_name:
            chemical_name = second_extract_chemical_name_method(text)
        return chemical_name


#1.4 This step applies the functions and displays the dataframe 

    results = []

    for pdf_file in uploaded.keys():
        text = pdfcontent(pdf_file)
        product_name = keyvaluepair(text, "Product name")
        cas_no = casnumber(text)
        ec_no = ecnumber(text)
        id_uses = uses(text)
        hazard = isitahazard(text)
        adr = adr_(text)
        un_number = UN(text)
        supplier = supplier(text)
        chemical_names = extract_chemical_name(text)

        results.append({
            'Product Name': product_name,
            'CAS Number': cas_no,
            'EC Number': ec_no,
            'Chemical Name': chemical_names,
            'Identified Uses': id_uses,
            'Supplier Name': supplier,
            'Hazard?': hazard,
            'ADR': adr,
            'UN Number': un_number
        })

    df = pd.DataFrame(results)
    display(df)

#1.5 This code creates a SQL compatible df

    from sqlalchemy import create_engine text
    lite = create_engine('sqlite://, echo = False )
    df.to_sql(name='taxonomy', con = lite)
    with engine.connect() as k:
        k.execute(text("select * from taxonomy")).fetchall()
    sqlite_taxonomy = pd.read_sql_query("select * from taxonomy", k=lite) 
    display(sqlite_taxonomy)
    
# Section 2


# Section 3





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


# Section 2


# Section 3





# Thesisdraft
!pip install pymupdf
!pip install PyMuPDF
!pip install PyMuPDF googletrans==4.0.0-rc1
!pip install googletrans==4.0.0-rc1

from google.colab import files
import fitz

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

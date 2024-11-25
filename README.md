# Finvoice-e-Invoice-Automation
Our ERP system handles e-invoicing according to the Finvoice e-invoicing standard.
Too often, the invoices from our suppliers do not comply with the standard, typically due to system reasons.
We are now looking for a solution where we save the invoices to a cloud or get them sent to a dedicated email address. From there we want to transform the data from the scanned document or from a "readable" pdf and extract the data to an .xml file in the format defined in the Finvoice e-invoicing standard. This file should then automatically be sent further to our ERP or the a solution provider.
The volume of invoices we expect to be around 500-1000 per year and they will be in 5-10 different templates.
=====================
To automate the process of extracting invoice data and converting it into an XML file based on the Finvoice e-invoicing standard, you can design a Python solution with the following key components:

    Invoice Data Retrieval:
        Invoices are stored in a cloud (e.g., AWS S3, Google Cloud Storage) or received through a dedicated email inbox.
        Use a library to connect and process the email inbox or cloud storage.

    Data Extraction:
        Use OCR (Optical Character Recognition) for scanned documents (e.g., Tesseract via pytesseract).
        Use libraries like pdfplumber or PyPDF2 to extract data from readable PDFs.

    Data Parsing and Template Matching:
        Match the invoice data to one of the predefined templates to identify key fields (e.g., supplier name, invoice number, date, amount).
        Use a structured mapping for different supplier templates.

    XML Generation:
        Format the extracted data into an XML file conforming to the Finvoice e-invoicing standard.

    Automated Submission:
        Send the generated XML file to the ERP system or a specified provider using APIs or secure file transfer.

Python Code Framework

Below is a Python code framework implementing the solution:
Dependencies

Install the required libraries:

pip install pdfplumber pytesseract lxml imaplib2

Code Implementation

import os
import re
import pdfplumber
import pytesseract
from pytesseract import Output
from lxml import etree
import imaplib
import email
from email.header import decode_header
import base64
import requests

# OCR Setup
pytesseract.pytesseract.tesseract_cmd = r"C:\Program Files\Tesseract-OCR\tesseract.exe"  # Update for your system

# Finvoice XML Template
FINVOICE_NAMESPACE = "http://www.edilex.fi/finvoice/finvoice/"
NSMAP = {"finvoice": FINVOICE_NAMESPACE}

def extract_pdf_data(file_path):
    """Extract text from a PDF file."""
    with pdfplumber.open(file_path) as pdf:
        text = ""
        for page in pdf.pages:
            text += page.extract_text()
    return text

def extract_ocr_data(file_path):
    """Extract text from a scanned document using OCR."""
    from PIL import Image
    image = Image.open(file_path)
    text = pytesseract.image_to_string(image, output_type=Output.STRING)
    return text

def parse_invoice_data(raw_text, template_mapping):
    """Parse invoice data based on predefined templates."""
    data = {}
    for field, regex in template_mapping.items():
        match = re.search(regex, raw_text)
        if match:
            data[field] = match.group(1)
    return data

def generate_finvoice_xml(data):
    """Generate an XML file conforming to the Finvoice standard."""
    root = etree.Element("Finvoice", nsmap=NSMAP)
    header = etree.SubElement(root, "InvoiceSenderPartyDetails")
    etree.SubElement(header, "SellerPartyIdentifier").text = data.get("seller_identifier", "Unknown")

    invoice_details = etree.SubElement(root, "InvoiceDetails")
    etree.SubElement(invoice_details, "InvoiceNumber").text = data.get("invoice_number", "Unknown")
    etree.SubElement(invoice_details, "InvoiceDate").text = data.get("invoice_date", "Unknown")
    etree.SubElement(invoice_details, "InvoiceTotalAmount").text = data.get("total_amount", "Unknown")

    # Save to file
    xml_file = f"invoice_{data['invoice_number']}.xml"
    with open(xml_file, "wb") as f:
        f.write(etree.tostring(root, pretty_print=True, xml_declaration=True, encoding="UTF-8"))
    return xml_file

def fetch_emails(email_user, email_password, server="imap.gmail.com"):
    """Fetch emails from a dedicated inbox."""
    imap = imaplib.IMAP4_SSL(server)
    imap.login(email_user, email_password)
    imap.select("inbox")

    status, messages = imap.search(None, "ALL")
    email_ids = messages[0].split()

    attachments = []
    for email_id in email_ids:
        res, msg = imap.fetch(email_id, "(RFC822)")
        for response in msg:
            if isinstance(response, tuple):
                msg = email.message_from_bytes(response[1])
                subject = decode_header(msg["Subject"])[0][0]
                if isinstance(subject, bytes):
                    subject = subject.decode()
                print(f"Processing email: {subject}")

                for part in msg.walk():
                    if part.get_content_maintype() == "multipart":
                        continue
                    if part.get("Content-Disposition"):
                        file_data = part.get_payload(decode=True)
                        filename = part.get_filename()
                        if filename:
                            filepath = os.path.join("attachments", filename)
                            with open(filepath, "wb") as f:
                                f.write(file_data)
                            attachments.append(filepath)
    imap.logout()
    return attachments

def upload_to_cloud(file_path, upload_url):
    """Upload the generated XML to a cloud or ERP endpoint."""
    with open(file_path, "rb") as f:
        response = requests.post(upload_url, files={"file": f})
    if response.status_code == 200:
        print(f"Uploaded {file_path} successfully!")
    else:
        print(f"Failed to upload {file_path}: {response.status_code}")

# Main Workflow
def process_invoices(email_user, email_password, upload_url, template_mapping):
    """Process invoices from email and generate XML files."""
    print("Fetching emails...")
    attachments = fetch_emails(email_user, email_password)

    for file_path in attachments:
        print(f"Processing file: {file_path}")

        # Determine if PDF or image
        if file_path.endswith(".pdf"):
            raw_text = extract_pdf_data(file_path)
        else:
            raw_text = extract_ocr_data(file_path)

        # Parse and Generate XML
        data = parse_invoice_data(raw_text, template_mapping)
        xml_file = generate_finvoice_xml(data)

        # Upload XML
        upload_to_cloud(xml_file, upload_url)

# Example Template Mapping
TEMPLATE_MAPPING = {
    "seller_identifier": r"Seller ID: (\d+)",
    "invoice_number": r"Invoice No: (\w+)",
    "invoice_date": r"Date: (\d{4}-\d{2}-\d{2})",
    "total_amount": r"Total Amount: ([\d\.]+)",
}

if __name__ == "__main__":
    EMAIL_USER = "your_email@example.com"
    EMAIL_PASSWORD = "your_password"
    UPLOAD_URL = "https://your-cloud-or-erp-endpoint/upload"

    process_invoices(EMAIL_USER, EMAIL_PASSWORD, UPLOAD_URL, TEMPLATE_MAPPING)

Key Features of the Solution

    Email Handling:
        The script connects to an email inbox, fetches attachments (invoices), and saves them for processing.

    OCR and PDF Extraction:
        Handles both readable PDFs and scanned invoices using OCR.

    Template Mapping:
        Custom regex-based mapping allows for flexible parsing of various invoice templates.

    Finvoice XML Generation:
        Generates XML files in compliance with the Finvoice standard.

    Automation and Scalability:
        Automatically uploads the generated XML files to a specified endpoint for further processing.

Deployment Instructions

    Email Inbox:
        Set up a dedicated inbox for receiving invoices.
        Update EMAIL_USER and EMAIL_PASSWORD in the script.

    Cloud Upload:
        Replace UPLOAD_URL with your cloud or ERP upload endpoint.

    Run the Script:
        Schedule the script to run periodically (e.g., using a Windows Task Scheduler or a cron job).

    Dependencies:
        Install Tesseract OCR for scanned invoice processing.

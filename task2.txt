import requests
from bs4 import BeautifulSoup
import logging

# Set up logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

def scrape_website(url):
    """Scrapes the content of a website and returns the text."""
    try:
        response = requests.get(url, timeout=10)  # Set a timeout for the request
        response.raise_for_status()  # Raise an error for bad responses
        soup = BeautifulSoup(response.text, 'html.parser')
        text_content = soup.get_text(separator=' ', strip=True)  # Clean and format text
        return text_content
    except requests.exceptions.SSLError as ssl_err:
        logging.error(f"SSL error occurred: {ssl_err}")
    except requests.exceptions.RequestException as req_err:
        logging.error(f"Request error occurred: {req_err}")
    except Exception as e:
        logging.error(f"An error occurred: {e}")

def answer_query(query, scraped_data):
    """Finds relevant content based on the user's query."""
    results = []
    for url, content in scraped_data.items():
        if query.lower() in content.lower():  # Simple keyword matching
            results.append((url, content))
    return results

def main():
    # List of URLs to scrape
    urls = [
        "https://www.uchicago.edu/",
        "https://www.washington.edu/",
        "https://www.stanford.edu/",
        "https://und.edu/"
    ]

    # Scrape the websites and store the content
    scraped_data = {}
    for url in urls:
        content = scrape_website(url)
        if content:
            logging.info(f"Successfully scraped content from {url}")
            scraped_data[url] = content  # Store the content in a dictionary

    # Example query
    user_query = input("Enter your query: ")
    results = answer_query(user_query, scraped_data)

    # Display the results
    if results:
        print("\nResults found:")
        for url, content in results:
            print(f"\nFrom {url}:\n{content[:500]}...")  # Preview first 200 characters
    else:
        print("No results found for your query.")

if __name__ == "__main__":
    main()








import pdfplumber
import fitz  # PyMuPDF
import os

# Function to extract text and tables from a specific page in the PDF
def extract_text_and_tables(pdf_path, page_number):
    with pdfplumber.open(pdf_path) as pdf:
        if page_number < 0 or page_number >= len(pdf.pages):
            return None, None  # Return None if the page number is out of range
        page = pdf.pages[page_number]
        text = page.extract_text()
        tables = page.extract_tables()
        return text, tables

# Function to extract images from a specific page in the PDF
def extract_images(pdf_path, page_number, image_output_dir):
    if not os.path.exists(image_output_dir):
        os.makedirs(image_output_dir)

    images = []
    with fitz.open(pdf_path) as pdf:
        if page_number < 0 or page_number >= len(pdf):
            return None  # Return None if the page number is out of range
        page = pdf[page_number]
        image_list = page.get_images(full=True)
        for img_index, img in enumerate(image_list):
            xref = img[0]
            base_image = pdf.extract_image(xref)
            image_bytes = base_image["image"]
            image_filename = os.path.join(image_output_dir, f"page_{page_number + 1}_img_{img_index + 1}.png")
            with open(image_filename, "wb") as img_file:
                img_file.write(image_bytes)
            images.append(image_filename)
    return images

# Function to handle user queries for multiple pages
def handle_query(user_query, pdf_path, image_output_dir):
    response = ""
    page_numbers = []

    # Extracting page numbers from the query
    try:
        for part in user_query.split("page")[1:]:
            page_number = int(part.split()[0]) - 1  # Convert to zero-based index
            page_numbers.append(page_number)
    except (IndexError, ValueError):
        return "Invalid query format. Please specify valid page numbers."

    for page_number in page_numbers:
        # Extract text and tables from the specified page
        page_text, tables = extract_text_and_tables(pdf_path, page_number)
        if page_text is None:
            response += f"The specified page {page_number + 1} does not exist in the PDF.\n"
            continue

        # Prepare the response
        response += f"Data from page {page_number + 1}:\n"
        response += f"Text:\n{page_text}\n"

        # Check for tables and extract relevant information
        if tables:
            response += "Tables found on this page:\n"
            for i, table in enumerate(tables):
                response += f"\nTable {i + 1}:\n"
                for row in table:
                    response += " | ".join(str(cell) for cell in row) + "\n"
        else:
            response += "No tables found on this page.\n"

        # Extract images from the specified page
        images = extract_images(pdf_path, page_number, image_output_dir)
        if images:
            response += "Images extracted from this page:\n"
            for img in images:
                response += f"{img}\n"  # List the image file paths
        else:
            response += "No images found on this page.\n"

    return response

# Main execution
if __name__ == "__main__":
    # Path to your PDF file
    pdf_path = r"C:\Users\kavya\OneDrive\Desktop\Task 1\Tables.pdf"  # Replace with your PDF file path
    image_output_dir = r"C:\Users\kavya\OneDrive\Desktop\Task 1\extracted_images"  # Directory to save extracted images

    # Example user query
    user_query = input("Enter your query: ")
    response = handle_query(user_query, pdf_path, image_output_dir)
    print(response)

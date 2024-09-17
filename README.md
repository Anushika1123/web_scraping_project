import os
import requests
from bs4 import BeautifulSoup
import mysql.connector

class DahuaFirmware:
    def __init__(self, url):
        self.url = url

    def create_folder(self, folder_name):
        if not os.path.exists(folder_name):
            os.makedirs(folder_name)
            print(f"Folder created successfully: {folder_name}")

    def download_firmware(self, firmware_url, model_folder):
        try:
            response = requests.get(firmware_url)
            if response.status_code == 200:
                filename = firmware_url.split('/')[-1]
                file_path = os.path.join(model_folder, filename)
                with open(file_path, 'wb') as f:
                    f.write(response.content)
                print(f"Firmware downloaded successfully: {filename}")
                return filename
            else:
                print(f"Failed to download firmware.")
                return None
        except Exception as e:
            print(f"Error downloading firmware: {e}")
            return None
        

    def scrape_and_download(self):
        r = requests.get(self.url)
        soup = BeautifulSoup(r.text, 'html.parser')

        table = soup.find("table", class_="wikitable")

        db_config = {
            'host': 'localhost',
            'user': 'root',
            'password': 'Sanushika@2930',
            'database': 'company_data'
        }

        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()

        cursor.execute("SHOW TABLES LIKE 'dahua'")
        table_exists = cursor.fetchone()

        if not table_exists:
            cursor.execute('''
                CREATE TABLE dahua (
                    MODELS VARCHAR(255),
                    SERIES VARCHAR(20),
                    FIRMWARE VARCHAR(255),
                    EOL VARCHAR(20)
                )
            ''')
            print("Table 'dahua' created successfully.")
        else:
            print("Table 'dahua' already exists in the database.")

        self.create_folder("D:\\Redinent\\DAHUA")

        for row in table.find_all("tr")[1:]:
            columns = row.find_all(["th", "td"])
            row_values = [column.text.strip() for column in columns]
            
            model = row_values[0]
            series = row_values[2]
            firmware_url = columns[7].find("a").get("href") if columns[7].find("a") else None
            eol = row_values[8]

            model_folder = os.path.join("D:\\Redinent\\DAHUA", model)
            self.create_folder(model_folder)

            if firmware_url:
                filename = self.download_firmware(firmware_url, model_folder)
                file_path = None
                if filename:
                    print(f"Downloaded firmware for {model}: {filename}")
                    file_path = os.path.join(model_folder, filename)

            cursor.execute('''
                INSERT INTO dahua (MODELS, SERIES, FIRMWARE, EOL)
                VALUES (%s, %s, %s, %s)
            ''', (model, series, file_path, eol))

            conn.commit()
        conn.close()

# Instantiate the class and call the method
url = "https://dahuawiki.com/Cameras"
fw_downloader = DahuaFirmware(url)
fw_downloader.scrape_and_download()


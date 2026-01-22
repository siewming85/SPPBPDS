Bahagian ini menggantikan kaedah manual "Download Analisis" yang sering memakan masa (status "Sedang Diproses" yang lama). Skrip ini akan menyedut (scrape) data terus dari paparan markah kelas.

***

# Bahagian A (Kaedah Inovasi): Muat Turun Data Automatik (Python)

**Mengapa Guna Kaedah Ini?**
*   🚀 **Pantas:** Tidak perlu menunggu server idMe memproses analisis (yang kadangkala mengambil masa sehingga 1 bulan).
*   ✅ **Real-time:** Data diambil terus dari paparan markah guru kelas yang sudah disahkan.
*   🤖 **Automatik:** Skrip akan membuka setiap kelas dan menyalin markah ke dalam satu fail CSV secara automatik.

### Langkah 1: Persediaan Persekitaran (Environment Setup)

Sebelum menjalankan skrip, pastikan komputer anda mempunyai perisian berikut:

1.  **Install Python:**
    *   Muat turun dan install Python dari [python.org](https://www.python.org/downloads/).
    *   *Penting:* Semasa installation, tandakan kotak **"Add Python to PATH"**.

2.  **Install Google Chrome:**
    *   Pastikan pelayar Google Chrome anda adalah versi terkini.

3.  **Install Library Python:**
    *   Buka **Command Prompt (CMD)** atau Terminal.
    *   Taip arahan berikut dan tekan Enter:
        ```bash
        pip install selenium beautifulsoup4
        ```

### Langkah 2: Penyediaan Skrip

1.  Buka aplikasi Notepad atau Code Editor (seperti VS Code).
2.  Salin (Copy) kod Python di bawah.
3.  Simpan fail tersebut sebagai **`scrape_idme.py`**.

**⚠️ PENTING: Sebelum simpan, anda perlu ubah 2 perkara dalam kod:**
1.  **Senarai URL (`direct_urls`):** Anda perlu login idMe secara manual dahulu, buka paparan markah setiap kelas yang ingin dianalisis, salin URL dari browser, dan tampal ke dalam senarai `direct_urls` dalam kod di bawah.
2.  **Username & Password:** Anda boleh set sebagai *Environment Variable* (lebih selamat) atau gantikan `os.getenv(...)` terus dengan nombor IC dan kata laluan anda dalam kod (jika guna di komputer peribadi).

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options
from bs4 import BeautifulSoup
import time
import csv
import os
import re

# --- KONFIGURASI PENGGUNA ---
# Pilihan 1: Set Environment Variables di Windows (Recommended)
# Pilihan 2: Gantikan os.getenv di bawah terus dengan string "IC_ANDA" dan "PASSWORD_ANDA"
USERNAME = os.getenv("IDME_USERNAME") 
PASSWORD = os.getenv("IDME_PASSWORD")

# Senarai URL Kelas (Sila update senarai ini dengan URL kelas sekolah anda)
# Cara dapatkan URL: Masuk idMe > Pengurusan Pentaksiran > Markah > Pilih Kelas > Copy URL di browser
direct_urls = [
    "https://moeissppb.moe.gov.my/pengurusan/peperiksaan/markah/172945", 
    "https://moeissppb.moe.gov.my/pengurusan/peperiksaan/markah/184074", 
    # ... TAMBAH URL KELAS LAIN DI SINI ...
]
# -----------------------------

chrome_options = Options()
chrome_options.add_argument("--disable-print-preview")
chrome_options.add_experimental_option(
    "prefs",
    {
        "printing.print_preview_sticky_popup": False,  # Disable print dialog
    }
)

if not USERNAME or not PASSWORD:
    print("Ralat: Sila masukkan USERNAME dan PASSWORD dalam kod atau set Environment Variables.")
    # Jika edit manual, padam baris 'exit()' di bawah selepas masukkan credential
    exit()

# Initialize webdriver
driver = webdriver.Chrome(options=chrome_options)

def login(driver, username, password):
    print("Sedang log masuk idMe...")
    driver.get("https://idme.moe.gov.my")
    wait = WebDriverWait(driver, 10)
    id_field = wait.until(EC.presence_of_element_located((By.ID, "ic")))
    id_field.send_keys(username)
    id_field.send_keys(Keys.RETURN)

    try:
        checkbox = wait.until(EC.element_to_be_clickable((By.ID, "check_log")))
        checkbox.click()
    except:
        pass # Kadang-kadang checkbox tidak muncul

    password_field = wait.until(EC.presence_of_element_located((By.ID, "password")))
    password_field.send_keys(password)

    login_button = wait.until(EC.element_to_be_clickable((By.ID, "log_open_form")))
    login_button.click()

    try:
        time.sleep(3)
        print("Login berjaya.")
    except:
        print("Login gagal.")
        driver.quit()
        exit()

def extract_examination_data(driver):
    # Akses modul SPPB
    driver.get("https://idme.moe.gov.my/list_aplikasi")
    wait = WebDriverWait(driver, 10)
    
    try:
        link_element = wait.until(EC.element_to_be_clickable((By.XPATH, "//a[contains(@href, 'moeissppb.moe.gov.my/sppb?token_idms=')]")))
        link_element.click()
        time.sleep(5)
    except:
        print("Gagal akses SPPB. Pastikan peranan anda betul.")

    csv_file_name = 'all_student_results.csv'
    
    # Initialize basic headers
    master_header = ['Bil.', 'Aktiviti', 'Sekolah', 'Tahun/Tingkatan', 'Kelas', 'Nama', 'MyKID/ID Pengenalan', 'Jantina']
    all_subjects = set() 

    print("Fasa 1: Mengimbas subjek yang wujud...")
    # --- Pass 1: Collect all possible subjects ---
    for url in direct_urls:
        full_url = url + "?page=1&per_page=9999&"
        driver.get(full_url)
        wait = WebDriverWait(driver, 10)
        
        try:
            table = wait.until(EC.presence_of_element_located((By.CSS_SELECTOR, "table.w-table")))
        except:
            print(f"Amaran: Jadual tidak dijumpai untuk {url}. Skipping.")
            continue
            
        soup = BeautifulSoup(driver.page_source, 'html.parser')
        table = soup.find('table', class_='w-table')
        
        if table:
            header_cells = table.find('thead').find_all('th')
            for header in header_cells[4:-3]: 
                label_tag = header.find('label')
                subject = label_tag.text.strip() if label_tag else header.text.strip()
                if subject: 
                    all_subjects.add(subject)

    all_subjects = sorted(list(all_subjects))
    print(f"Dijumpai {len(all_subjects)} subjek unik.")
    
    for subject in all_subjects:
        master_header.extend([f'{subject} Mark', f'{subject} Grade'])

    # --- Pass 2: Extract data and write CSV ---
    print("Fasa 2: Mengekstrak data pelajar...")
    with open(csv_file_name, 'w', newline='', encoding='utf-8') as csvfile:
        csv_writer = csv.writer(csvfile)
        csv_writer.writerow(master_header)

        for url in direct_urls:
            full_url = url + "?page=1&per_page=9999&"
            driver.get(full_url)
            wait = WebDriverWait(driver, 10)
            
            try:
                wait.until(EC.presence_of_element_located((By.CSS_SELECTOR, "table.w-table")))
            except:
                continue

            soup = BeautifulSoup(driver.page_source, 'html.parser')

            # Metadata
            school_element = soup.find("h5", string=lambda t: t and "Sekolah" in t)
            school = school_element.find_next_sibling("div").text.strip() if school_element else ""
            level_element = soup.find("h5", string=lambda t: t and "Tahun / Tingkatan" in t)
            level = level_element.find_next_sibling("div").text.strip() if level_element else ""
            activity_element = soup.find("h5", string=lambda t: t and "Aktiviti" in t)
            activity = activity_element.find_next_sibling("div").text.strip() if activity_element else ""
            class_element = soup.find("h5", string=lambda t: t and "Kelas" in t)
            kelas = class_element.find_next_sibling("div").text.strip() if class_element else ""
            
            print(f"Memproses: {kelas} ({school})")

            table = soup.find('table', class_='w-table')
            if table:
                header_cells = table.find('thead').find_all('th')
                subject_column_map = {}
                for idx, header in enumerate(header_cells[4:-3]):
                    label_tag = header.find('label')
                    subject_name = label_tag.text.strip() if label_tag else header.text.strip()
                    if subject_name:
                        subject_column_map[subject_name] = 4 + idx

                rows = driver.find_elements(By.CSS_SELECTOR, "table.w-table tbody tr")
                for row_idx, row in enumerate(rows):
                    cells = row.find_elements(By.TAG_NAME, "td")
                    if len(cells) < 4: continue

                    parsed_row = [
                        row_idx + 1, activity, school, level, kelas,
                        cells[1].text.strip(), cells[2].text.strip(), cells[3].text.strip(),
                    ]

                    for subject in all_subjects:
                        if subject in subject_column_map:
                            col_index = subject_column_map[subject]
                            if col_index < len(cells):
                                cell_text = cells[col_index].text.strip()
                                if cell_text and "(" in cell_text and cell_text.endswith(")"):
                                    parts = cell_text.rsplit(" (", 1)
                                    parsed_row.extend([parts[0], parts[1].rstrip(")")])
                                elif cell_text == "TH":
                                    parsed_row.extend(["TH", "TH"])    
                                elif cell_text: 
                                    parsed_row.extend([cell_text, ""])
                                else:
                                    parsed_row.extend(["", ""])
                            else:
                                parsed_row.extend(["", ""])
                        else:
                            parsed_row.extend(["", ""])

                    csv_writer.writerow(parsed_row)

    print("\nSelesai! Data disimpan dalam 'all_student_results.csv'")
                    
# Main execution
login(driver, USERNAME, PASSWORD)
extract_examination_data(driver)
driver.quit()
```

### Langkah 3: Menjalankan Skrip (Export)

1.  Buka Command Prompt (CMD) di folder di mana anda simpan fail `scrape_idme.py` tadi.
2.  (Pilihan) Jika anda menggunakan Environment Variables untuk keselamatan, setkan dahulu:
    ```cmd
    set IDME_USERNAME=NoICAnda
    set IDME_PASSWORD=PasswordAnda
    ```
3.  Jalankan skrip dengan arahan:
    ```cmd
    python scrape_idme.py
    ```
4.  Browser Chrome akan terbuka secara automatik dan mula melawat setiap URL kelas yang anda tetapkan. **Jangan tutup browser tersebut.**
5.  Setelah selesai, fail **`all_student_results.csv`** akan terhasil dalam folder yang sama.

### Langkah Seterusnya

Gunakan fail **`all_student_results.csv`** ini sebagai sumber data untuk **FASA 1** dalam panduan dashboard di bawah. Copy isinya ke tab **`Data`** di Google Sheet.

*Nota: Struktur lajur mungkin sedikit berbeza dengan fail `fik_...csv` asal (urutan subjek). Pastikan anda menyemak semula Lajur Subjek dalam "Fasa 3: Pemetaan Lajur" jika menggunakan kaedah ini.*

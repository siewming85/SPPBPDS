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
1.  **Senarai URL (`direct_urls`):** Anda perlu login idMe secara manual dahulu, buka paparan markah setiap kelas yang ingin dianalisis, salin URL dari browser, dan tampal ke dalam senarai `direct_urls` dalam kod di bawah. Cara mendapat senarai url, pergi Laporan>Peperiksaan Dalaman Sekolah>Kelas seperti dalam gambar, right klik butang pensel untuk setiap kelas, dapatkan 
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



Kaedah ini jauh lebih **"advance" dan automatik** berbanding kaedah formula Excel biasa. Anda tidak perlu menyalin formula yang panjang atau risau tentang formula rosak. Skrip akan membaca data dari `all_student_results.csv` dan menjana dashboard sepenuhnya.

---

# Bahagian B (Alternatif): Dashboard Automatik dengan Google Apps Script

Kaedah ini menggunakan pengaturcaraan Google Apps Script untuk memproses fail `all_student_results.csv` (yang dihasilkan dari Python) dan membina analisis secara automatik apabila anda memilih nama sekolah.

### Langkah 1: Import Data CSV ke Google Sheets

1.  Buka Google Sheet baharu.
2.  **Import Data:**
    *   Klik **File > Import > Upload**.
    *   Pilih fail `all_student_results.csv` yang dihasilkan oleh Python tadi.
    *   Pilih **"Replace spreadsheet"** atau **"Insert new sheet"**.
3.  Pastikan nama Tab (Sheet) tersebut dinamakan sebagai **`Data`**.
    *   *Semak Header:* Pastikan baris 1 mengandungi header seperti `BC Mark`, `BC Grade`, `BM Grade` dan sebagainya.

### Langkah 2: Masukkan Google Apps Script

1.  Di Google Sheet tersebut, klik menu **Extensions > Apps Script**.
2.  Padamkan sebarang kod yang ada di dalam `Code.gs`.
3.  Salin dan tampal kod penuh di bawah:

```javascript
/**
 * PENGATURCARAAN ANALISIS IDME (SPPB)
 * Menggunakan data format: Mark & Grade columns
 */

function onOpen() {
  const ui = SpreadsheetApp.getUi();
  ui.createMenu('Admin Analisis')
    .addItem('Jana/Kemaskini Analisis', 'janaAnalisis')
    .addToUi();
}

// Trigger automatik apabila Dropdown B1 berubah
function onEdit(e) {
  const sheet = e.source.getActiveSheet();
  const range = e.range;
  
  // Jika perubahan berlaku di sheet 'Analisis' pada sel B1
  if (sheet.getName() === 'Analisis' && range.getA1Notation() === 'B1') {
    janaAnalisis();
  }
}

function janaAnalisis() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const dataSheet = ss.getSheetByName('Data');
  let analysisSheet = ss.getSheetByName('Analisis');

  // 1. Persediaan Sheet Analisis jika belum wujud
  if (!analysisSheet) {
    analysisSheet = ss.insertSheet('Analisis');
    setupDashboardStructure(analysisSheet, dataSheet);
    return; // Berhenti untuk user pilih sekolah dulu
  }

  // 2. Dapatkan Pilihan Sekolah & Data Mentah
  const selectedSchool = analysisSheet.getRange('B1').getValue();
  if (!selectedSchool) {
    Browser.msgBox("Sila pilih Sekolah atau 'SEMUA' di sel B1");
    return;
  }

  const rawData = dataSheet.getDataRange().getValues();
  const headers = rawData[0];
  
  // Cari Indeks Lajur Penting
  const colSekolah = headers.indexOf('Sekolah');
  
  // 3. Filter Data Berdasarkan Sekolah
  let filteredData = rawData.slice(1); // Buang header
  if (selectedSchool !== 'SEMUA') {
    filteredData = filteredData.filter(row => row[colSekolah] === selectedSchool);
  }

  if (filteredData.length === 0) {
    Browser.msgBox("Tiada data ditemui untuk pilihan ini.");
    return;
  }

  // 4. Kenalpasti Subjek (Cari column yang berakhir dengan 'Grade')
  const subjectMap = [];
  headers.forEach((header, index) => {
    if (header.includes(' Grade')) {
      subjectMap.push({
        name: header.replace(' Grade', ''), // Cth: "BM Grade" -> "BM"
        index: index
      });
    }
  });

  // 5. Proses Statistik Utama (Gred, GP, Lulus)
  // Gred Order: A+, A, A-, B+, B, C+, C, D, E, G, TH
  // Nilai GP: A+=0, A=1, A-=2, B+=3, B=4, C+=5, C=6, D=7, E=8, G=9
  const gradeList = ['A+', 'A', 'A-', 'B+', 'B', 'C+', 'C', 'D', 'E', 'G', 'TH'];
  const gpValues = {'A+':0, 'A':1, 'A-':2, 'B+':3, 'B':4, 'C+':5, 'C':6, 'D':7, 'E':8, 'G':9};
  
  let outputTable = [];

  subjectMap.forEach(sub => {
    let counts = {};
    gradeList.forEach(g => counts[g] = 0);
    let totalMurid = 0;
    let totalGPPoints = 0;
    let countForGP = 0;
    let lulusCount = 0;

    filteredData.forEach(row => {
      let grade = row[sub.index];
      grade = grade ? grade.toString().trim() : "";
      
      if (grade === "") return; // Skip jika tiada data
      
      totalMurid++;
      
      // Normalise TH
      if (grade === 'TH' || grade === 'T') {
        counts['TH']++;
      } else if (counts.hasOwnProperty(grade)) {
        counts[grade]++;
        
        // Kira Lulus (Semua kecuali G dan TH)
        if (grade !== 'G') {
          lulusCount++;
        }

        // Kira GP
        if (gpValues.hasOwnProperty(grade)) {
          totalGPPoints += gpValues[grade];
          countForGP++;
        }
      } else {
        // Handle Gred pelik jika ada
        if(grade === 'G') {
             counts['G']++;
             totalGPPoints += gpValues['G'];
             countForGP++;
        }
      }
    });

    // Format Baris untuk Subjek ini
    let rowData = [sub.name, totalMurid];
    
    // Masukkan data bagi setiap gred + peratus
    gradeList.forEach(g => {
        if(g === 'TH') return; // TH handle last
        let count = counts[g];
        let pct = totalMurid > 0 ? (count / totalMurid) : 0;
        rowData.push(count, pct);
    });

    // % Lulus
    let pctLulus = totalMurid > 0 ? (lulusCount / totalMurid) : 0;
    rowData.push(pctLulus);

    // TH & % TH
    let countTH = counts['TH'];
    let pctTH = totalMurid > 0 ? (countTH / totalMurid) : 0;
    rowData.push(countTH, pctTH);

    // GP
    let gp = countForGP > 0 ? (totalGPPoints / countForGP) : 0;
    rowData.push(gp);

    outputTable.push(rowData);
  });

  // 6. Tulis Data ke Jadual Utama
  // Bersihkan kawasan data lama (mula baris 4)
  if(analysisSheet.getLastRow() > 3){
    analysisSheet.getRange(4, 1, analysisSheet.getLastRow()-3, 30).clearContent();
  }
  
  if (outputTable.length > 0) {
    analysisSheet.getRange(4, 1, outputTable.length, outputTable[0].length).setValues(outputTable);
    // Format Number (Percentage & Decimals)
    // Adjust range format based on columns
    // Cth: Col D(%), F(%), H(%), etc.. ini agak rumit nak hardcode format, user boleh format manual sekali.
  }

  // 7. Analisis LMS (Layak Mendapat Sijil - BM & SEJ)
  processLMS(analysisSheet, filteredData, headers);
}

function processLMS(sheet, data, headers) {
  // Cari index BM dan SEJ
  // Note: Regex used to find exact "BM Grade" or "SEJ Grade" column
  const bmIndex = headers.indexOf('BM Grade');
  const sejIndex = headers.indexOf('SEJ Grade');

  let stats = {
    lulus_bm_lulus_sej: 0,
    lulus_bm_gagal_sej: 0,
    gagal_bm_lulus_sej: 0,
    gagal_bm_gagal_sej: 0,
    total: 0
  };

  if (bmIndex === -1 || sejIndex === -1) {
    // Jika subjek tiada, biar 0
  } else {
    data.forEach(row => {
      let gBM = row[bmIndex] ? row[bmIndex].toString().trim() : "";
      let gSEJ = row[sejIndex] ? row[sejIndex].toString().trim() : "";

      if (gBM === "" && gSEJ === "") return; // Skip kosong

      stats.total++;

      // Definisi Lulus: A+, A, A-, B+, B, C+, C, D, E
      // Definisi Gagal: G, TH
      const isPass = (g) => /^[A-E]/.test(g); 

      let passBM = isPass(gBM);
      let passSEJ = isPass(gSEJ);

      if (passBM && passSEJ) stats.lulus_bm_lulus_sej++;
      else if (passBM && !passSEJ) stats.lulus_bm_gagal_sej++;
      else if (!passBM && passSEJ) stats.gagal_bm_lulus_sej++;
      else stats.gagal_bm_gagal_sej++;
    });
  }

  // Tulis ke bahagian bawah sheet
  const startRow = data.length > 20 ? 30 : 30; // Tetapkan lokasi statik atau dynamic
  
  const lmsData = [
    ["ANALISIS LAYAK SIJIL (LMS) - BM & SEJARAH", ""],
    ["KATEGORI", "BILANGAN"],
    ["Layak Sijil (Lulus BM & Sejarah)", stats.lulus_bm_lulus_sej],
    ["Lulus BM, Gagal Sejarah", stats.lulus_bm_gagal_sej],
    ["Gagal BM, Lulus Sejarah", stats.gagal_bm_lulus_sej],
    ["Gagal Kedua-dua Subjek", stats.gagal_bm_gagal_sej],
    ["JUMLAH CALON", stats.total]
  ];

  const lmsRange = sheet.getRange("F4:G10"); // Letak di sebelah kanan jadual utama (contoh F4)
  // Atau lebih baik letak di bawah jadual utama secara automatik
  // Kita letak di Column AD (tepi sekali) atau Row 30
  
  // STRATEGI BARU: Letak di Row 30 (Hardcoded for simplicity as requested)
  sheet.getRange("E26:F32").setValues(lmsData);
  sheet.getRange("E26:F26").setBackground('#f3f3f3').setFontWeight('bold');
}

function setupDashboardStructure(sheet, dataSheet) {
  // Setup Dropdown
  const schools = getUniqueValues(dataSheet, 'Sekolah');
  schools.unshift('SEMUA');
  
  const rule = SpreadsheetApp.newDataValidation().requireValueInList(schools).build();
  sheet.getRange('B1').setDataValidation(rule);
  sheet.getRange('A1').setValue("PILIH SEKOLAH:");
  sheet.getRange('B1').setValue("SEMUA");

  // Setup Header Utama
  const header = [
    "SUBJEK", "JUM", 
    "A+", "%", "A", "%", "A-", "%", 
    "B+", "%", "B", "%", "C+", "%", "C", "%", 
    "D", "%", "E", "%", "G", "%", 
    "% LULUS", "TH", "%", "GP"
  ];
  sheet.getRange(3, 1, 1, header.length).setValues([header]).setFontWeight('bold').setBackground('#cfe2f3');
  
  // Trigger analisis pertama kali
  janaAnalisis();
}

function getUniqueValues(sheet, colName) {
  const data = sheet.getDataRange().getValues();
  const colIndex = data[0].indexOf(colName);
  if (colIndex === -1) return [];
  
  const values = data.slice(1).map(row => row[colIndex]).filter(String);
  return [...new Set(values)].sort();
}
```

4.  Tekan ikon **Save** (💾).
5.  Tutup tab Apps Script dan kembali ke Google Sheet.

### Langkah 3: Menjana Dashboard Pertama Kali

1.  Muat semula (Refresh) laman Google Sheet anda.
2.  Anda akan melihat menu baharu di atas bernama **"Admin Analisis"**.
3.  Klik **Admin Analisis > Jana/Kemaskini Analisis**.
4.  Skrip akan meminta kebenaran (Permission) pada kali pertama:
    *   Klik *Continue*.
    *   Pilih akaun Google anda.
    *   Klik *Advanced* > *Go to (Nama Skrip) (unsafe)*.
    *   Klik *Allow*.
5.  Skrip akan secara automatik:
    *   Mencipta tab `Analisis`.
    *   Membuat dropdown Sekolah di sel **B1**.
    *   Membuat Jadual Header di Baris 3.
    *   Mengisi data analisis.

### Langkah 4: Penggunaan Harian

Sistem ini kini automatik sepenuhnya.

1.  Pergi ke tab **`Analisis`**.
2.  Tukar pilihan di sel **B1** (Contoh: Tukar dari `SEMUA` kepada `SMK CONTOH`).
3.  Tunggu 1-2 saat, data dalam jadual akan berubah secara automatik mengikut sekolah yang dipilih.
4.  **Analisis LMS (BM & Sejarah)** akan terpapar di bahagian bawah (sekitar sel **E26**).

---

### Kelebihan Kaedah Apps Script:
1.  **Tidak Perlu Formula Rumit:** Tiada lagi risiko terpadam formula `COUNTIFS` atau `VLOOKUP`.
2.  **Fleksibel:** Jika ada subjek baharu dalam CSV, skrip akan automatik menambahnya ke dalam senarai tanpa perlu anda ubah apa-apa.
3.  **LMS Automatik:** Logik lulus/gagal BM & Sejarah dikira di belakang tabir tanpa memerlukan "Helper Column" di tab Data.

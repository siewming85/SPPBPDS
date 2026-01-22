***

# Panduan Analisis Peperiksaan Dalaman Sekolah (SPPB - idMe)

Dokumen ini menyediakan langkah-langkah terperinci untuk memuat turun data mentah dari sistem idMe (Modul SPPB) dan membina **Analisis Papan Pemuka (Dashboard Analysis)** automatik menggunakan Google Sheets.

Sistem ini membolehkan analisis dibuat untuk **Keseluruhan Daerah (PPD)** atau diperincikan mengikut **Setiap Sekolah** menggunakan satu fail sahaja. Dengan syarat perlu tunggu pihak idMe siap memproses data.

---

## Bahagian A: Muat Turun Data dari idMe

Sila pastikan anda mempunyai akses dan peranan yang betul (seperti Admin atau Penyelaras) sebelum memulakan.

1.  **Log Masuk:** Pergi ke laman web idMe dan log masuk akaun anda.
2.  **Akses Aplikasi:** Klik pada menu **Aplikasi**.
3.  **Pengurusan Pentaksiran:** Pilih ikon **Pengurusan Pentaksiran**.
4.  **Navigasi Analisis:**
    *   Tekan menu **Analisis**.
    *   Pilih **Pep. Dalaman Sekolah**.
    *   Pilih **FIK**.
5.  **Tetapan Carian:**
    *   Pilih **Sesi Persekolahan** dan **Aktiviti Pentaksiran** yang ingin dianalisis.
    *   *(Rujukan Gambar: [Lihat Screenshot](https://github.com/siewming85/SPPBPDS/blob/main/Screenshot_2026-01-22_10_05_19.png?raw=true))*
6.  **Muat Turun:** Klik butang **Muat Turun** (biasanya format CSV/Excel).
    *   *Fail contoh: `fik_1769046998.csv`*

---

## Bahagian B: Membina Dashboard Analisis (Google Sheets)

Ikuti langkah ini untuk menukar data mentah kepada jadual analisis dinamik.

### FASA 1: Persediaan Data (Tab 'Data')
1.  Buka Google Sheet baharu.
2.  Namakan Tab (Sheet) pertama sebagai **`Data`**.
3.  Salin (Copy) dan tampal (Paste) keseluruhan data dari fail CSV yang dimuat turun tadi bermula di sel **A1**.
    *   *Pastikan Header berada di Baris 1.*
    *   *Pastikan Lajur C mengandungi Kod Sekolah.*

### FASA 2: Membina Tab Analisis & Menu Pilihan
1.  Cipta Tab baharu dan namakan sebagai **`Analisis`**.
2.  **Bina Dropdown Sekolah (Sel B1):**
    *   Klik sel **B1**.
    *   Pergi ke menu **Data > Data Validation**.
    *   Criteria: **Dropdown (from a range)**.
    *   Pilih julat: `Data!C2:C1000` (Senarai Kod Sekolah dari tab Data).
    *   **PENTING:** Tambah satu item manual dalam senarai dropdown atau taip di sel B1 perkataan: `SEMUA` (untuk paparan data keseluruhan PPD).
3.  **Bina Header Jadual (Baris 3):**
    *   Di sel **A3**, masukkan tajuk lajur berikut:
    `SUBJEK | Jumlah Murid | A+ | % | A | % | A- | % | B+ | % | B | % | C+ | % | C | % | D | % | E | % | G | % | %Lulus | TH | % | GP`

### FASA 3: Pemetaan Lajur (Helper Columns)
Kerana gred subjek berada di lajur berbeza, kita perlu memetakan lokasi lajur tersebut.

1.  Di tab **`Analisis`**, pergi ke lajur **AA** (sebagai rujukan tepi).
2.  Bina jadual rujukan berikut (Lajur AA = Nama Subjek, Lajur AB = Huruf Lajur dalam sheet Data):

| Sel (Subjek) | Sel (Huruf Lajur) |
| :--- | :--- |
| **AA4** : BC | **AB4** : P |
| **AA5** : BI | **AB5** : R |
| **AA6** : BIN | **AB6** : T |
| **AA7** : BIO | **AB7** : V |
| **AA8** : BM | **AB8** : X |
| **AA9** : FIZIK | **AB9** : Z |
| ... | ... (Sambung untuk subjek lain seperti dalam arahan asal) |

3.  Di Lajur **A** (Jadual Utama), senaraikan nama subjek dari **A4** hingga ke bawah (sama susunan dengan AA).

### FASA 4: Memasukkan Formula Automatik

Salin formula di bawah ke baris pertama data (Baris 4 - Subjek BC) dan tarik (drag) ke bawah.

**1. Jumlah Murid (Sel B4)**
Mengira bilangan murid yang mempunyai rekod (bukan kosong).
```excel
=COUNTIFS(Data!$C:$C, IF($B$1="SEMUA", "*", $B$1), INDIRECT("Data!" & VLOOKUP($A4, $AA$4:$AB$24, 2, FALSE) & ":" & VLOOKUP($A4, $AA$4:$AB$24, 2, FALSE)), "<>")
```

**2. Bilangan Gred A+ (Sel C4)**
```excel
=COUNTIFS(Data!$C:$C, IF($B$1="SEMUA", "*", $B$1), INDIRECT("Data!" & VLOOKUP($A4, $AA$4:$AB$24, 2, FALSE) & ":" & VLOOKUP($A4, $AA$4:$AB$24, 2, FALSE)), "A+")
```
> *Nota: Untuk gred lain (A, A-, B, dll), salin formula ini dan tukar "A+" kepada gred yang dikehendaki.*

**3. Peratusan (Sel D4)**
```excel
=IF($B4=0, 0, C4/$B4)
```
*(Format sel ini sebagai %)*

**4. Peratus Lulus (Sel Y4 - contoh lokasi)**
Lulus = Semua kecuali G dan TH.
```excel
=IF($B4=0, 0, 1 - (W4/$B4) - (AA4/$B4))
```
*(Pastikan W4 ialah bilangan G dan AA4 ialah bilangan TH dalam jadual utama anda).*

**5. Gred Purata / GP (Sel AC4)**
Formula GP (tidak termasuk TH):
```excel
=IF(($B4-AA4)=0, 0, ( (C4*0) + (E4*1) + (G4*2) + (I4*3) + (K4*4) + (M4*5) + (O4*6) + (Q4*7) + (S4*8) + (W4*9) ) / ($B4 - AA4) )
```

---

## Cara Penggunaan Dashboard

1.  **Analisis PPD (Keseluruhan):**
    *   Pada sel **B1**, pilih `SEMUA`.
    *   Data akan memaparkan pencapaian gabungan semua sekolah.

2.  **Analisis Sekolah Individu:**
    *   Pada sel **B1**, pilih **Kod/Nama Sekolah** dari senarai.
    *   Data akan dikemaskini secara automatik untuk sekolah tersebut sahaja.

---

Berikut adalah tambahan kepada fail **README.md** untuk memasukkan **Analisis Layak Mendapat Sijil (LMS)** dan **Analisis Silang BM & Sejarah**.

Anda boleh menyambung teks ini di bahagian bawah fail README.md yang sebelumnya, atau letakkan di bahagian baharu.

***

# Bahagian C: Analisis Layak Sijil (LMS) & Silang BM-Sejarah

Bahagian ini membolehkan anda melihat berapa ramai murid yang Layak Sijil (Lulus kedua-dua BM & Sejarah) serta analisis murid yang gagal salah satu subjek teras tersebut.

Data ini juga akan berubah mengikut pilihan sekolah di dropdown **B1**.

### FASA 1: Menambah Logik Lulus/Gagal (Helper Columns)

Untuk memudahkan pengiraan silang (Cross-tabulation), kita akan tambah dua lajur bantuan di hujung tab **`Data`** untuk menentukan status Lulus/Gagal bagi setiap murid secara automatik.

*Nota: Berdasarkan panduan sebelum ini, Gred BM berada di lajur **X** dan Sejarah di lajur **AX**.*

1.  Pergi ke tab **`Data`**.
2.  Pergi ke lajur kosong di hujung data (contohnya Lajur **BG** dan **BH**).
3.  **Lajur Status BM (BG):**
    *   Di sel **BG1**, taip tajuk: `STATUS BM`
    *   Di sel **BG2**, masukkan formula ini (menganggap Lulus ialah A hingga E):
        ```excel
        =ARRAYFORMULA(IF(C2:C="", "", IF(REGEXMATCH(X2:X, "A|B|C|D|E"), "LULUS", "GAGAL")))
        ```
4.  **Lajur Status SEJ (BH):**
    *   Di sel **BH1**, taip tajuk: `STATUS SEJ`
    *   Di sel **BH2**, masukkan formula ini:
        ```excel
        =ARRAYFORMULA(IF(C2:C="", "", IF(REGEXMATCH(AX2:AX, "A|B|C|D|E"), "LULUS", "GAGAL")))
        ```
    *(Formula ini akan automatik mengisi ke bawah untuk semua murid. "GAGAL" merangkumi Gred G dan TH).*

---

### FASA 2: Membina Jadual Analisis LMS (Tab Analisis)

1.  Pergi ke tab **`Analisis`**.
2.  Pilih kawasan kosong (contohnya bermula di sel **H1** atau di bawah jadual subjek).
3.  Bina struktur jadual seperti berikut:

| Sel | Teks / Label |
| :--- | :--- |
| **E26** | **KATEGORI PENCAPAIAN (BM & SEJ)** |
| **E27** | Layak Sijil (Lulus BM & Lulus SEJ) |
| **E28** | Lulus BM, Gagal Sejarah |
| **E29** | Gagal BM, Lulus Sejarah |
| **E30** | Gagal Kedua-dua Subjek |
| **E31** | **JUMLAH CALON** |

### FASA 3: Memasukkan Formula Pengiraan

Masukkan formula berikut di sebelah label yang anda bina tadi (Lajur F). Formula ini akan membaca Dropdown di **B1** (Semua atau Sekolah) dan Helper Column di tab Data.

*Pastikan anda menukar `BG` (Status BM) dan `BH` (Status Sej) jika anda meletakkannya di lajur berbeza.*

1.  **Layak Sijil (Lulus BM & Lulus SEJ) - Sel F27:**
    ```excel
    =COUNTIFS(Data!$C:$C, IF($B$1="SEMUA", "*", $B$1), Data!$BG:$BG, "LULUS", Data!$BH:$BH, "LULUS")
    ```

2.  **Lulus BM, Gagal Sejarah - Sel F28:**
    ```excel
    =COUNTIFS(Data!$C:$C, IF($B$1="SEMUA", "*", $B$1), Data!$BG:$BG, "LULUS", Data!$BH:$BH, "GAGAL")
    ```

3.  **Gagal BM, Lulus Sejarah - Sel F29:**
    ```excel
    =COUNTIFS(Data!$C:$C, IF($B$1="SEMUA", "*", $B$1), Data!$BG:$BG, "GAGAL", Data!$BH:$BH, "LULUS")
    ```

4.  **Gagal Kedua-dua Subjek - Sel F30:**
    ```excel
    =COUNTIFS(Data!$C:$C, IF($B$1="SEMUA", "*", $B$1), Data!$BG:$BG, "GAGAL", Data!$BH:$BH, "GAGAL")
    ```

5.  **Jumlah Calon (Semakan) - Sel F31:**
    ```excel
    =SUM(F27:F30)
    ```

### FASA 4: Menambah Peratusan (Optional)

Untuk melihat peratusan LMS, tambah formula ini di lajur sebelah (Lajur G):

1.  **Peratus Layak Sijil (Sel G27):**
    ```excel
    =IF($F$31=0, 0, F27/$F$31)
    ```
    *(Tarik formula ke bawah sehingga baris G30 dan tukar format kepada %)*

---

### Rumusan Output

Dengan langkah di atas, Dashboard anda kini mempunyai dua bahagian:
1.  **Jadual Atas:** Analisis GP dan Gred mengikut Subjek.
2.  **Jadual Bawah:** Analisis Kualiti (LMS) berdasarkan silang gred BM dan Sejarah.

Kedua-dua jadual ini akan berubah serentak apabila anda menukar nama sekolah di **Dropdown B1**.
## Kredit
Projek ini direka untuk memudahkan pengurusan data peperiksaan dalaman sekolah menggunakan eksport standard sistem idMe.

*Tarikh Kemaskini: Januari 2026*

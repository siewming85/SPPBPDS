***

# Panduan Analisis Peperiksaan Dalaman Sekolah (SPPB - idMe)

Dokumen ini menyediakan langkah-langkah terperinci untuk memuat turun data mentah dari sistem idMe (Modul SPPB) dan membina **Analisis Papan Pemuka (Dashboard Analysis)** automatik menggunakan Google Sheets.

Sistem ini membolehkan analisis dibuat untuk **Keseluruhan Daerah (PPD)** atau diperincikan mengikut **Setiap Sekolah** menggunakan satu fail sahaja.

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

## Kredit
Projek ini direka untuk memudahkan pengurusan data peperiksaan dalaman sekolah menggunakan eksport standard sistem idMe.

*Tarikh Kemaskini: Januari 2026*

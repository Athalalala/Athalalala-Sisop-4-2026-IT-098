# Sisop-4-2026-IT-098

## Profil Mahasiswa

Nama           : Jude Athala

NRP            : 5027251098

Departemen     : Teknologi Informasi

# Laporan Praktikum Modul 4

Praktikum Modul 4 ini membahas tentang penggunaan **FUSE (Filesystem in Userspace)**, yaitu sistem yang memungkinkan kita membuat filesystem sendiri langsung dari user space tanpa perlu mengubah kernel Linux.

Pada soal ini, filesystem yang dibuat harus bisa membaca file dari folder asli, menampilkan file virtual, dan menggabungkan isi beberapa file menjadi satu output baru secara otomatis.

---

# Soal 1 — Save Asisten Kenz

## a. Download dan Ekstraksi File

### Permasalahan

Sebelum membuat filesystem, file sumber terlebih dahulu harus diambil dari Google Drive dan diekstrak agar bisa digunakan selama praktikum.

### Penyelesaian

File zip diunduh menggunakan `wget`, kemudian diekstrak memakai `unzip`. Setelah selesai, file zip dihapus supaya folder kerja tetap rapi.

```bash
wget --no-check-certificate 'https://docs.google.com/uc?export=download&id=1nLXFhptDo2mnUlZsw8pTWyAVpV49W20U' -O amba_files.zip

unzip amba_files.zip

rm amba_files.zip
```

### Hasil

Setelah proses selesai, folder `amba_files/` berhasil muncul dan berisi file `1.txt` sampai `7.txt`.

---

## b. Membuat Filesystem FUSE

### Permasalahan

Filesystem yang dibuat harus bisa menampilkan isi folder asli melalui folder mount FUSE.

### Penyelesaian

Program dibuat menggunakan beberapa callback utama FUSE seperti:

* `getattr`
* `readdir`
* `read`

Semua path dari folder mount diarahkan menuju folder asli `amba_files`.

Contoh implementasi pada fungsi `getattr`:

```c
static int xmp_getattr(const char *path, struct stat *stbuf)
{
    char fpath[1000];

    sprintf(fpath, "%s%s", dirpath, path);

    if (lstat(fpath, stbuf) == -1)
        return -errno;

    return 0;
}
```

Fungsi tersebut digunakan untuk mengambil atribut file dari direktori asli agar file bisa dibaca normal melalui filesystem FUSE.

### Hasil

Isi folder asli berhasil muncul ketika direktori mount dibuka menggunakan `ls`.

---

## c. Membuat File Virtual `tujuan.txt`

### Permasalahan

Pada soal diminta untuk membuat file virtual bernama `tujuan.txt`. File ini tidak benar-benar ada di folder asli, tetapi harus tetap muncul ketika filesystem dijalankan.

### Penyelesaian

File virtual ditambahkan secara manual di fungsi `readdir` menggunakan `filler`.

```c
filler(buf, "tujuan.txt", NULL, 0);
```

Kemudian atribut file diatur pada fungsi `getattr`.

```c
if (strcmp(path, "/tujuan.txt") == 0)
{
    stbuf->st_mode = S_IFREG | 0444;
    stbuf->st_nlink = 1;
    stbuf->st_size = 66;

    return 0;
}
```

### Hasil

Saat folder mount dibuka, file `tujuan.txt` berhasil muncul walaupun file tersebut sebenarnya tidak ada di dalam folder `amba_files`.

---

## d. Menggabungkan Isi Koordinat

### Permasalahan

Isi dari `tujuan.txt` tidak ditulis manual, tetapi harus dibuat otomatis dengan mengambil data dari file `1.txt` sampai `7.txt`.

### Penyelesaian

Pada fungsi `read`, program membaca seluruh file satu per satu. Program mencari baris yang memiliki awalan `KOORD:` lalu mengambil isi setelahnya untuk digabung menjadi satu kalimat.

Contoh proses pembacaan file:

```c
char konten[200] = "Tujuan Mas Amba: ";

for (int i = 1; i <= 7; i++)
{
    snprintf(path,
             sizeof(path),
             "%s/%d.txt",
             src_path,
             i);

    // proses membuka file
    // proses membaca KOORD:
    // proses menggabungkan isi koordinat
}
```

### Hasil

Ketika file dibaca menggunakan:

```bash
cat mnt/tujuan.txt
```

filesystem akan otomatis menampilkan gabungan koordinat dari semua file sumber.

---

# Warning Saat Compile

Ketika program dikompilasi menggunakan command:

```bash
gcc kenz_rescue.c -Wall `pkg-config fuse3 --cflags --libs` -o kenz_rescue
```

muncul warning seperti berikut:

```bash
warning: ‘%d’ directive output may be truncated
```

Warning ini sebenarnya bukan error, jadi program masih tetap bisa dijalankan. Compiler hanya memberi peringatan bahwa hasil `snprintf()` kemungkinan melebihi ukuran buffer yang disediakan.

Bagian kode yang menyebabkan warning:

```c
snprintf(path,
         sizeof(path),
         "%s/%d.txt",
         src_path,
         i);
```

Untuk mengurangi kemungkinan tersebut, ukuran buffer bisa diperbesar.

```c
char path[8192];
```

Atau bisa juga ditambahkan pengecekan panjang hasil `snprintf()`.

---

# Fitur Tambahan Logging

Selain fitur utama, filesystem juga dibuat memiliki fitur logging.

Setiap folder yang diakses akan dicatat ke file `log.log` lengkap dengan waktu aksesnya. Fitur ini dibuat menggunakan library `time.h`.

Contoh isi log:

```text
Fri May 16 20:31:10 2026: /mnt
```

---

# Cara Menjalankan Program

## Compile

```bash
gcc kenz_rescue.c -Wall `pkg-config fuse3 --cflags --libs` -o kenz_rescue
```

## Membuat Folder Mount

```bash
mkdir mnt
```

## Menjalankan Filesystem

```bash
./kenz_rescue mnt
```

## Mengecek File Virtual

```bash
cat mnt/tujuan.txt
```

## Unmount Filesystem

```bash
fusermount -u mnt
```

---

# Kesimpulan

Pada praktikum ini berhasil dibuat filesystem berbasis FUSE yang dapat:

 Menampilkan isi folder asli
 
 Membuat file virtual
 
 Menggabungkan isi beberapa file menjadi satu output
 
 Menampilkan isi file secara dinamis Menyimpan log akses filesystem


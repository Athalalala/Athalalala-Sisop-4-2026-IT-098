# Sisop-4-2026-IT-098

## Profil Mahasiswa

Nama           : Jude Athala

NRP            : 5027251098

Departemen     : Teknologi Informasi

# Laporan Praktikum Modul 4

Praktikum Modul 4 ini membahas implementasi filesystem menggunakan FUSE (*Filesystem in Userspace*). Pada soal ini, filesystem dibuat agar dapat membaca isi folder asli dan menampilkan file virtual bernama `tujuan.txt` yang isinya digabung secara otomatis dari beberapa file sumber.

---

# Soal 1 — Save Asisten Kenz

## Langkah Pengerjaan

### 1. Membuat Direktori Praktikum

```bash
mkdir modul_4
cd modul_4
mkdir soal_1
cd soal_1
```

---

### 2. Mengekstrak File Soal

```bash
unzip amba_files.zip
rm amba_files.zip
```

---

### 3. Membuat Program FUSE

```bash
nano kenz_rescue.c
```

Isi file `kenz_rescue.c`:

```c
#define FUSE_USE_VERSION 31
#include <fuse3/fuse.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <dirent.h>
#include <errno.h>
#include <sys/stat.h>

static char base_dir[4096];

static char* buat_konten_tujuan(size_t *panjang) {
    char *hasil = NULL;
    size_t total = 0;

    for (int nomor = 3; nomor <= 7; nomor++) {
        char lokasi[8192];

        snprintf(lokasi,
                 sizeof(lokasi),
                 "%s/%d.txt",
                 base_dir,
                 nomor);

        FILE *fp = fopen(lokasi, "r");

        if (fp == NULL)
            continue;

        char baris[1024];

        while (fgets(baris, sizeof(baris), fp) != NULL) {

            if (strstr(baris, "KOORD:") != NULL) {

                char *ptr = strstr(baris, "KOORD:") + 6;

                size_t n = strlen(ptr);

                hasil = realloc(hasil, total + n + 1);

                memcpy(hasil + total, ptr, n);

                total += n;

                hasil[total] = '\0';
            }
        }

        fclose(fp);
    }

    *panjang = total;

    return hasil;
}

static size_t ukuran_tujuan(void) {

    size_t n = 0;

    char *tmp = buat_konten_tujuan(&n);

    free(tmp);

    return n;
}

static int rescue_getattr(const char *path,
                          struct stat *st,
                          struct fuse_file_info *fi) {

    (void) fi;

    memset(st, 0, sizeof(*st));

    if (strcmp(path, "/") == 0) {

        st->st_mode = S_IFDIR | 0755;
        st->st_nlink = 2;

    } else if (strcmp(path, "/tujuan.txt") == 0) {

        st->st_mode = S_IFREG | 0444;
        st->st_nlink = 1;
        st->st_size = (off_t) ukuran_tujuan();

    } else {

        char full[8192];

        snprintf(full,
                 sizeof(full),
                 "%s%s",
                 base_dir,
                 path);

        if (lstat(full, st) == -1)
            return -errno;
    }

    return 0;
}

static int rescue_readdir(const char *path,
                          void *buf,
                          fuse_fill_dir_t filler,
                          off_t offset,
                          struct fuse_file_info *fi,
                          enum fuse_readdir_flags flags) {

    (void) offset;
    (void) fi;
    (void) flags;

    if (strcmp(path, "/") != 0)
        return -ENOENT;

    filler(buf, ".", NULL, 0, 0);
    filler(buf, "..", NULL, 0, 0);

    DIR *dp = opendir(base_dir);

    if (dp == NULL)
        return -errno;

    struct dirent *ent;

    while ((ent = readdir(dp)) != NULL) {

        filler(buf,
               ent->d_name,
               NULL,
               0,
               0);
    }

    closedir(dp);

    filler(buf,
           "tujuan.txt",
           NULL,
           0,
           0);

    return 0;
}

static int rescue_open(const char *path,
                       struct fuse_file_info *fi) {

    if (strcmp(path, "/tujuan.txt") == 0)
        return 0;

    char full[8192];

    snprintf(full,
             sizeof(full),
             "%s%s",
             base_dir,
             path);

    int fd = open(full, fi->flags);

    if (fd == -1)
        return -errno;

    close(fd);

    return 0;
}

static int rescue_read(const char *path,
                       char *buf,
                       size_t size,
                       off_t offset,
                       struct fuse_file_info *fi) {

    (void) fi;

    if (strcmp(path, "/tujuan.txt") == 0) {

        size_t panjang = 0;

        char *konten = buat_konten_tujuan(&panjang);

        if (offset >= (off_t) panjang) {

            free(konten);

            return 0;
        }

        size_t ambil = size;

        if ((size_t) offset + ambil > panjang)
            ambil = panjang - (size_t) offset;

        memcpy(buf,
               konten + offset,
               ambil);

        free(konten);

        return (int) ambil;
    }

    char full[8192];

    snprintf(full,
             sizeof(full),
             "%s%s",
             base_dir,
             path);

    int fd = open(full, O_RDONLY);

    if (fd == -1)
        return -errno;

    int ret = (int) pread(fd,
                          buf,
                          size,
                          offset);

    if (ret == -1)
        ret = -errno;

    close(fd);

    return ret;
}

static struct fuse_operations operasi = {
    .getattr = rescue_getattr,
    .readdir = rescue_readdir,
    .open = rescue_open,
    .read = rescue_read,
};

int main(int argc, char *argv[]) {

    if (argc < 3)
        return 1;

    if (realpath(argv[1], base_dir) == NULL)
        return 1;

    argv[1] = argv[2];

    argc--;

    umask(0);

    return fuse_main(argc,
                     argv,
                     &operasi,
                     NULL);
}
```

---

### 4. Compile Program

```bash
gcc -Wall kenz_rescue.c -o kenz_rescue `pkg-config fuse3 --cflags --libs`
```

---

### 5. Install Dependency FUSE3

```bash
sudo apt update
sudo apt install fuse3 libfuse3-dev
```

---

### 6. Membuat Folder Mount

```bash
mkdir -p mnt
```

---

### 7. Menjalankan Filesystem

```bash
sudo ./kenz_rescue -f amba_files mnt
```

---

### 8. Mengecek Isi Filesystem

```bash
ls -l mnt/
```

---

### 9. Mengecek Isi `tujuan.txt`

```bash
cat mnt/tujuan.txt
```

---

## Problem Keseluruhan
Pada penyelesaian Soal Modul 4 ini saya juga menggunakan bantuan gemini karena di peraturan diperbolehkan menggunakan Ai. Disini saya menggunakan gemini untuk membuat script, menjelaskan alur dan memecahkan masalah pada penyelesaian soal Modul 4. Berikut link percakapan saya dengan [gemini](https://gemini.google.com/share/20143847d916)

## Output dari soal

![image link](Assets/Gambar_72png)

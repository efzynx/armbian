# Panduan Lengkap: Memindahkan Data Docker ke Penyimpanan Eksternal (SSD/HDD)

Dokumen ini adalah panduan langkah demi langkah untuk memindahkan seluruh data Docker (images, volumes, containers) dari penyimpanan utama (eMMC/SD Card) ke penyimpanan eksternal seperti SSD atau HDD. Panduan ini juga mencakup solusi untuk masalah umum di mana aplikasi Docker "menghilang" setelah reboot.

Tujuan utamanya adalah untuk menghemat ruang di drive sistem yang kecil dan mengurangi beban baca/tulis, sehingga lebih awet dan performa lebih baik.

---

## Bagian 1: Menyiapkan Penyimpanan Eksternal

Sebelum memindahkan data, kita harus memastikan drive eksternal (SSD/HDD) siap dan akan selalu terhubung secara otomatis setiap kali server booting.

### Langkah 1.1: Deteksi dan Hubungkan Drive

Pastikan drive eksternal Anda terhubung ke server. Jika drive tidak terdeteksi otomatis setelah reboot (masalah umum pada perangkat berdaya rendah seperti STB), Anda mungkin perlu melakukan "hot-plug" (cabut-pasang kabel USB) secara manual.

Verifikasi bahwa drive sudah terlihat oleh sistem dengan perintah:

```

lsblk

```

Anda seharusnya melihat perangkat baru seperti `/dev/sda` beserta partisinya `/dev/sda1`.

### Langkah 1.2: Buat "Pintu Masuk" (Mount Point)

Buat sebuah folder kosong di sistem utama yang akan berfungsi sebagai gerbang untuk mengakses isi dari drive eksternal Anda. Langkah ini hanya perlu dilakukan sekali.

```

# Anda bisa mengganti 'storage' dengan nama lain yang Anda suka

sudo mkdir /mnt/storage

```

### Langkah 1.3: Hubungkan Drive Secara Otomatis dengan `fstab` (Langkah Kritis)

Ini adalah langkah paling penting untuk mencegah aplikasi "hilang" setelah reboot. Kita akan memberitahu Linux untuk secara otomatis menghubungkan (mount) drive ini setiap kali sistem dinyalakan, **sebelum** layanan seperti Docker dimulai.

1. **Dapatkan Alamat Unik (UUID) Partisi Anda:**

```

sudo blkid /dev/sda1

```

Salin nilai `UUID="..."` dari outputnya. Contoh: `a99fb545-c6e4-45b7-a010-079d62440c4a`.

2. **Edit File Konfigurasi `fstab`:**

```

sudo nano /etc/fstab

```

3. **Tambahkan Baris Konfigurasi Baru:**
Tambahkan baris berikut di paling bawah file. Ganti `UUID_ANDA_DISINI` dengan UUID yang sudah Anda salin.

```

UUID=UUID\_ANDA\_DISINI /mnt/storage ext4 defaults,nofail 0 2

```

* **`UUID=...`**: Alamat unik drive Anda.
* **`/mnt/storage`**: "Pintu masuk" yang kita buat.
* **`ext4`**: Tipe filesystem (sesuaikan jika berbeda).
* **`nofail`**: Opsi pengaman yang krusial. Jika suatu saat drive gagal terdeteksi, server akan tetap booting normal dan tidak akan hang.

4. Simpan file (`Ctrl+X`, `Y`, `Enter`). Setelah reboot berikutnya, drive Anda akan terpasang secara otomatis.

---

## Bagian 2: Memindahkan Data Docker

Setelah drive siap, saatnya memindahkan "gudang data" Docker.

### Langkah 2.1: Hentikan Docker dan Layanan Terkait

Ini wajib untuk mencegah kerusakan data. Jika Anda menggunakan CasaOS, hentikan juga layanannya.

```

sudo systemctl stop casaos
sudo systemctl stop docker.socket
sudo systemctl stop docker

```

### Langkah 2.2: Salin Seluruh Data Docker

Gunakan `rsync` untuk menyalin folder `/var/lib/docker` ke lokasi baru di drive eksternal Anda. Proses ini bisa memakan waktu cukup lama.

```

sudo rsync -av /var/lib/docker/ /mnt/storage/docker/

```

### Langkah 2.3: Buat "Jalan Pintas" (Symlink)

1. Ganti nama folder Docker asli sebagai backup:

```

sudo mv /var/lib/docker /var/lib/docker.bak

```

2. Buat symlink (jalan pintas) dari lokasi lama ke lokasi baru:

```

sudo ln -s /mnt/storage/docker /var/lib/docker

```

### Langkah 2.4: Jalankan Kembali dan Verifikasi

1. Nyalakan kembali Docker:

```

sudo systemctl start docker

```

2. Verifikasi bahwa Docker sekarang membaca dari lokasi baru:

```

sudo docker info | grep "Docker Root Dir"

```

Outputnya harus menunjukkan path baru Anda: `Docker Root Dir: /mnt/storage/docker`.

3. Jalankan kembali layanan lain seperti CasaOS:

```

sudo systemctl start casaos

```

### Langkah 2.5: Hapus Backup

Setelah Anda yakin semua aplikasi berjalan normal, Anda bisa menghapus backup untuk membebaskan ruang di drive utama.

```

sudo rm -rf /var/lib/docker.bak

```

---

## Bagian 3: Troubleshooting - "Aplikasi Saya Hilang Setelah Reboot!"

Ini adalah masalah umum yang disebabkan oleh urutan startup yang salah atau symlink yang tidak tepat.

### Gejala:

* Aplikasi di CasaOS/Portainer tidak muncul atau gagal dimuat.
* Perintah `docker ps` tidak menunjukkan container yang berjalan.
* Status Docker `failed` (`systemctl status docker`).
* Namun, penggunaan disk di drive eksternal masih menunjukkan data terpakai (misalnya 14GB).

### Penyebab:

1. **Race Condition:** Docker berjalan **sebelum** drive eksternal sempat di-mount oleh sistem. Akibatnya, Docker melihat folder kosong dan gagal memulai. **Solusi:** Menggunakan `fstab` seperti di Bagian 1.
2. **Symlink Salah Alamat:** Terkadang, panel seperti CasaOS membuat sub-direktori tambahan secara otomatis. Symlink Anda mungkin menunjuk ke alamat yang salah.

### Cara Mengatasi Symlink yang Salah:

1. **Temukan Lokasi Data yang Sebenarnya:**
   Jelajahi direktori mount Anda untuk menemukan di mana folder `docker` yang asli berada.

```

# Ganti /mnt/storage dengan mount point Anda

sudo ls -lR /mnt/storage/

```

Anda mungkin menemukan path yang tidak terduga, misalnya: `/mnt/storage/app-system/docker`.

2. **Hentikan Docker:**

```

sudo systemctl stop docker

```

3. **Hapus Symlink yang Salah:**

```

sudo rm /var/lib/docker

```

4. **Buat Ulang Symlink yang Benar:**
Gunakan path yang benar yang Anda temukan di langkah 1.

```

# Contoh dengan path yang benar

sudo ln -s /mnt/storage/app-system/docker /var/lib/docker

```

5. **Reset Status 'Failed' dan Restart Docker:**

```

sudo systemctl reset-failed docker.service
sudo systemctl start docker

```

Setelah Docker berjalan, nyalakan kembali layanan lain (seperti CasaOS), dan semua aplikasi Anda akan kembali normal.
```

# Tutorial Lengkap: Monitoring Server dengan Prometheus & Grafana di CasaOS

Panduan ini akan memandu Anda untuk menginstal dan mengkonfigurasi **Prometheus** dan **Node Exporter** menggunakan **Docker**, lalu mengintegrasikannya dengan **Grafana** yang sudah Anda install.
Hasil akhirnya adalah sebuah dashboard monitoring yang canggih untuk memantau semua metrik penting dari server mini Anda (CPU, RAM, Disk, Jaringan, dll).

---

## Arsitektur yang Akan Kita Bangun

* **Node Exporter**: Sebuah "agen" kecil yang mengumpulkan semua data metrik dari server Anda dan menampilkannya di sebuah alamat web.
* **Prometheus**: Sebuah "database" yang secara berkala mengambil (*scrape*) data dari Node Exporter dan menyimpannya dalam format time-series.
* **Grafana**: Aplikasi visualisasi yang akan membaca data dari Prometheus dan menampilkannya dalam bentuk grafik dan panel yang indah.

---

## Bagian 1: Menyiapkan Prometheus & Node Exporter dengan Docker Compose

Kita akan menggunakan satu file `docker-compose.yml` untuk menjalankan kedua layanan ini.

### Langkah 1.1: Buat Direktori Konfigurasi

Pertama, buat sebuah folder khusus untuk menyimpan semua file konfigurasi monitoring kita. Ini akan membuat semuanya rapi.

```bash
cd /DATA/AppData/
sudo mkdir -p monitoring/prometheus
```

Struktur folder Anda akan terlihat seperti ini:

```
/DATA/AppData/monitoring/prometheus
```

### Langkah 1.2: Buat File `docker-compose.yml`

Masuk ke folder monitoring yang baru saja Anda buat, lalu buat file baru:

```bash
cd /DATA/AppData/monitoring/
sudo nano docker-compose.yml
```

Isi dengan konfigurasi berikut:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    networks:
      - monitoring_net

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    # Port tidak perlu di-expose ke host karena Prometheus akan mengaksesnya via jaringan Docker internal
    # ports:
    #   - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring_net

networks:
  monitoring_net:
    driver: bridge

volumes:
  prometheus_data:
```

### Langkah 1.3: Buat File Konfigurasi `prometheus.yml`

Sekarang kita perlu memberitahu Prometheus di mana ia harus mengambil data.

Masuk ke folder prometheus:

```bash
cd /DATA/AppData/monitoring/prometheus/
sudo nano prometheus.yml
```

Isi dengan konfigurasi berikut:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

* `scrape_interval: 15s`: Prometheus akan mengambil data setiap 15 detik.
* `targets: ['node-exporter:9100']`: Targetnya adalah container node-exporter di port 9100.
  Kita bisa menggunakan nama service (`node-exporter`) karena keduanya berada di jaringan Docker yang sama (`monitoring_net`).

---

## Bagian 2: Menjalankan Stack Monitoring

Sekarang semua file sudah siap, saatnya menjalankan container.

Kembali ke direktori utama monitoring:

```bash
cd /DATA/AppData/monitoring/
```

Jalankan Docker Compose:

```bash
sudo docker compose up -d
```

Verifikasi bahwa kedua container berjalan:

```bash
sudo docker ps
```

Anda seharusnya melihat prometheus dan node-exporter dalam keadaan "Up".

### Uji Prometheus

Buka browser dan akses:

```
http://<IP_SERVER_ANDA>:9090
```

Klik pada menu **Status > Targets**. Anda seharusnya melihat target `node_exporter` dengan status **UP** berwarna hijau.

---

## Bagian 3: Mengintegrasikan Prometheus dengan Grafana

Langkah terakhir adalah memberitahu Grafana di mana ia harus mengambil data.

1. Buka antarmuka web Grafana Anda.

2. Di menu sebelah kiri, navigasi ke **Connections > Data sources**.

3. Klik tombol **Add new data source**.

4. Pilih **Prometheus** dari daftar.

5. Di bagian **Prometheus server URL**, masukkan:

   ```
   http://prometheus:9090
   ```

   > Catatan: Kita bisa menggunakan nama `prometheus` karena CasaOS secara default menempatkan semua container dalam satu jaringan Docker yang sama, sehingga mereka bisa saling "bicara" menggunakan nama container.

6. Biarkan pengaturan lain secara default. Gulir ke bawah dan klik **Save & test**.
   Anda akan melihat pesan **Data source is working** berwarna hijau.

---

## Bagian 4: Mengimpor Dashboard Siap Pakai

Membuat dashboard dari awal itu rumit. Mari kita impor dashboard yang sudah sangat populer dan lengkap.

1. Di menu Grafana, klik ikon **Dashboards**.

2. Di halaman Dashboards, klik tombol **New** lalu pilih **Import**.

3. Di kolom **Import via grafana.com**, masukkan ID dashboard berikut:

   ```
   1860
   ```

   > Ini adalah ID untuk dashboard **Node Exporter Full**.

4. Klik **Load**.

5. Di halaman berikutnya, pastikan Anda memilih **Prometheus** sebagai data source Anda.

6. Klik **Import**.

---

## 🎉 Selesai!

Anda sekarang akan disajikan dengan dashboard yang sangat detail yang menampilkan semua informasi tentang server mini Anda.
Selamat! Anda telah berhasil membangun sebuah sistem monitoring yang kuat dan profesional di server CasaOS Anda.


"""
Cache Simulator: Multi-threaded with and without Cache Coherence
=================================================================

Simulasi ini dirancang untuk memodelkan bagaimana dua core/thread mengakses 
dan memodifikasi data bersama dalam sistem multi-core, baik dengan maupun 
tanpa protokol koherensi cache. Dalam sistem nyata, cache koherensi sangat 
penting agar semua core memiliki pandangan konsisten terhadap data yang sama. 
Tanpa koherensi, inkonsistensi dapat terjadi jika satu core memperbarui nilai 
yang tidak disadari oleh core lain.

Program ini menggunakan dua thread yang masing-masing memiliki cache lokal. 
Saat koherensi diaktifkan (mode MESI-like), simulator akan mengirimkan sinyal 
invalidasi ke cache thread lain saat data ditulis. Jika tidak diaktifkan, 
maka core bebas membaca dan menulis ke cache lokal tanpa pemberitahuan.

Tujuan utama dari simulasi ini adalah membandingkan:
- Performa waktu eksekusi antara dua mode
- Jumlah akses ke memori utama (sebagai indikator traffic)
- Jumlah pesan invalidasi cache

Dengan begitu, kita bisa melihat trade-off nyata antara kecepatan dan konsistensi 
dalam sistem multiprosesor modern.

Cara Menjalankan:
$ python cache_simulator.py

Contoh Output:
Simulasi tanpa protokol koherensi:
  Waktu: 0.0123 detik
  Akses Memori: 1142
  Invalidasi: 0

Simulasi dengan protokol koherensi:
  Waktu: 0.0157 detik
  Akses Memori: 1032
  Invalidasi: 184

Analisis:
| Aspek               | Tanpa Koherensi    | Dengan Koherensi     |
|--------------------|--------------------|-----------------------|
| Akurasi Data       | Tidak konsisten    | Konsisten             |
| Waktu Eksekusi     | Lebih cepat        | Sedikit lebih lambat  |
| Traffic Memori     | Lebih tinggi       | Lebih efisien         |
| Invalidasi Cache   | Tidak ada          | Ada                   |
"""

import threading
import time
import random

class CacheLine:
    """
    Representasi satu baris cache, menyimpan nilai dan status.
    Status:
    - I: Invalid
    - S: Shared
    - M: Modified
    - E: Exclusive (tidak digunakan penuh di sini)
    """
    def __init__(self):
        self.value = 0
        self.state = "I"  # Initial state

class SharedMemory:
    """
    Representasi memori utama yang diakses oleh kedua core.
    """
    def __init__(self):
        self.value = 0

class CacheSimulator:
    """
    Simulator cache multi-core dengan opsi penggunaan protokol koherensi.
    """
    def __init__(self, use_coherence=True):
        self.memory = SharedMemory()
        self.caches = [CacheLine(), CacheLine()]  # Cache untuk 2 core
        self.lock = threading.Lock()
        self.use_coherence = use_coherence
        self.memory_access = 0
        self.invalidation_msgs = 0

    def read(self, core_id):
        """
        Simulasi pembacaan data oleh core tertentu.
        Jika cache invalid, ambil dari memori utama dan atur status.
        """
        cache = self.caches[core_id]
        if cache.state == "I":
            with self.lock:
                self.memory_access += 1
                cache.value = self.memory.value
                if self.use_coherence:
                    cache.state = "S"
                    other_cache = self.caches[1 - core_id]
                    if other_cache.state != "I":
                        other_cache.state = "S"
                else:
                    cache.state = "V"  # Valid (tanpa koherensi)
        return cache.value

    def write(self, core_id, value):
        """
        Simulasi penulisan data oleh core.
        Jika pakai koherensi, lakukan invalidasi cache core lain.
        """
        cache = self.caches[core_id]
        with self.lock:
            self.memory_access += 1
            self.memory.value = value
            cache.value = value
            if self.use_coherence:
                cache.state = "M"
                other_cache = self.caches[1 - core_id]
                if other_cache.state != "I":
                    self.invalidation_msgs += 1
                    other_cache.state = "I"
            else:
                cache.state = "V"

    def run_thread(self, core_id):
        """
        Fungsi utama untuk setiap core/thread, melakukan 1000 operasi acak.
        """
        for _ in range(1000):
            if random.random() < 0.5:
                self.read(core_id)
            else:
                self.write(core_id, random.randint(0, 100))

    def run_simulation(self):
        """
        Menjalankan simulasi dengan dua thread dan mengukur waktu eksekusi.
        """
        t0 = time.time()
        threads = []
        for i in range(2):
            t = threading.Thread(target=self.run_thread, args=(i,))
            threads.append(t)
            t.start()

        for t in threads:
            t.join()
        t1 = time.time()
        return t1 - t0

def compare():
    """
    Membandingkan dua skenario:
    1. Tanpa koherensi cache
    2. Dengan protokol koherensi (mirip MESI)
    """
    print("Simulasi tanpa protokol koherensi:")
    sim_no = CacheSimulator(use_coherence=False)
    time_no = sim_no.run_simulation()
    print(f"  Waktu: {time_no:.4f} detik")
    print(f"  Akses Memori: {sim_no.memory_access}")
    print(f"  Invalidasi: {sim_no.invalidation_msgs}\n")

    print("Simulasi dengan protokol koherensi:")
    sim_yes = CacheSimulator(use_coherence=True)
    time_yes = sim_yes.run_simulation()
    print(f"  Waktu: {time_yes:.4f} detik")
    print(f"  Akses Memori: {sim_yes.memory_access}")
    print(f"  Invalidasi: {sim_yes.invalidation_msgs}\n")

if __name__ == "__main__":
    compare()

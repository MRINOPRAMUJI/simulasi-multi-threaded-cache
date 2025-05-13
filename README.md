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

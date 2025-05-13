"""
Cache Simulator: Multi-threaded with and without Cache Coherence
=================================================================

Simulasi ini bertujuan untuk mengeksplorasi peran protokol koherensi cache dalam sistem multiprosesor 
dengan dua core/thread. Dalam sistem multi-core modern, masing-masing core memiliki cache lokal 
yang menyimpan salinan data dari memori utama. Namun, tantangan utama muncul ketika dua core mengakses 
dan memodifikasi data yang sama — hal ini dapat menyebabkan inkonsistensi tanpa adanya mekanisme 
koherensi cache.

Simulator ini membandingkan dua pendekatan:
1. **Tanpa koherensi cache** — setiap core membaca dan menulis dari cache lokal tanpa sinkronisasi.
2. **Dengan protokol koherensi sederhana (MESI-like)** — setiap perubahan nilai memicu invalidasi 
   cache core lain untuk menjaga konsistensi data antar core.

Setiap core melakukan serangkaian operasi acak (read/write) terhadap data bersama dalam 1000 iterasi. 
Simulator mencatat metrik seperti waktu eksekusi, jumlah akses ke memori utama, dan jumlah pesan 
invalidasi antar cache. Tujuan akhirnya adalah menunjukkan bahwa meskipun penggunaan protokol 
koherensi dapat menurunkan performa secara waktu karena sinkronisasi, ia memberikan hasil yang 
konsisten dan benar secara semantik — sesuatu yang penting dalam sistem nyata.

Simulasi ini juga dapat digunakan untuk memahami dasar dari protokol cache seperti MESI, MOESI, 
dan MSI dengan menambahkan status tambahan atau memperluas mekanismenya. Cocok untuk pembelajaran 
arsitektur komputer, paralelisme, dan sistem embedded.

Cara menjalankan:
$ python cache_simulator.py
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
                    # Tandai cache lawan juga shared jika valid
                    other_cache = self.caches[1 - core_id]
                    if other_cache.state != "I":
                        other_cache.state = "S"
                else:
                    cache.state = "V"  # Valid (untuk simulasi tanpa koherensi)
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
        Fungsi utama yang dijalankan oleh tiap thread/core.
        Melakukan 1000 operasi acak (read/write).
        """
        for _ in range(1000):
            if random.random() < 0.5:
                self.read(core_id)
            else:
                self.write(core_id, random.randint(0, 100))

    def run_simulation(self):
        """
        Menjalankan dua thread dan mengukur waktu total eksekusi.
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
    Membandingkan dua simulasi:
    1. Tanpa koherensi cache
    2. Dengan protokol koherensi (MESI-like)
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

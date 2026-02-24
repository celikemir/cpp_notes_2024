Bu ders notları, Necati Ergin'in 12 Şubat 2025 tarihli C++ dersinin (63. gün) ilk 25 dakikasını kapsayan teknik bir incelemedir.

# 63. Ders: Mutex, Lock Sınıfları ve Senkronizasyon Primitifleri - Bölüm I

## 1. Giriş: Singleton ve `std::call_once` (00:00:00 - 00:04:15)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Singleton gibi tasarım kalıplarında, bir nesnenin sadece bir kez ve *Thread-Safe* (İş parçacığı güvenli) şekilde ilklendirilmesi (initialization) gerekir. Çift kontrollü kilitleme (Double-checked locking) gibi manuel yöntemler hata payı yüksek ve karmaşık olabilir.

**⚙️ Teknik Detay ve Sentaks**
Hoca, `std::once_flag` ve `std::call_once` araçlarının bu iş için özelleştiğini belirtti.

```cpp
#include <mutex>

std::once_flag g_flag;

void initialize_resource() {
    // Karmaşık ilklendirme işlemleri...
}

void get_singleton() {
    std::call_once(g_flag, initialize_resource); // <-- Hoca buraya dikkat çekti: En yüksek efficiency (verim) ile çalışır.
}
```

**🚩 Kritik Nokta**
`std::call_once`, derleyici implementasyonunda olabilecek en yüksek verimle oluşturulmuştur. Manuel yazılan kiltleme mekanizmalarının aksine, hata yapma riski "yok denecek kadar azdır".

---

## 2. Multi-Thread Programlamada "Gözlenebilir Davranış" ve Yarış Durumu (04:15 - 07:50)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Derleyici, kod üzerinde optimizasyon yaparken programın *Single-Thread* (Tek iş parçacıklı) çalışacağını varsayar. Bu durum, birden fazla thread'in aynı veriye eriştiği senaryolarda mantıksal hatalara yol açar.

**⚙️ Teknik Detay ve Sentaks (Vektör Örneği)**
Hoca, atomik olmayan işlemlerin arasına başka thread'lerin girebileceğini (interleaving) şu örnekle açıkladı:

```cpp
void process_vector(std::vector<int>& vek) {
    if (!vek.empty()) { // <-- Check (Kontrol)
        // Bu noktada başka bir thread eleman silmiş olabilir!
        int x = vek.front(); // <-- Use (Kullanım) - HATA: Vektör boşalmış olabilir, UB (Tanımsız Davranış) oluşur.
    }
}
```

**🔍 Arka Plan (Under the Hood)**
*   **Interleaved (Araya girmiş):** İşlemlerin kesiksiz (atomic) yapılmaması durumunda, kontrol ve kullanım arasına başka bir thread'in işlem sokuşturması durumudur.
*   **Observable Behavior (Gözlenebilir Davranış):** Derleyici, bu davranış değişmediği sürece kodu "As-if" (Sanki öyleymiş gibi) kuralına göre optimize edebilir.

---

## 3. Yeniden Sıralama (Reordering) ve Görünürlük Sorunları (07:50 - 13:40)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Programcının yazdığı kod sırası, işlemci veya derleyici tarafından performans amacıyla değiştirilebilir. Bu, Multi-Thread sistemlerde felakete yol açar.

**⚙️ Teknik Detay ve Sentaks (Ready Flag Hatası)**
Hoca, çok sık yapılan bir hatayı tahtaya taşıdı:

```cpp
int g_val = 10;
bool ready_flag = false;

void thread_writer() {
    g_val = 9999;
    ready_flag = true; // <-- Hoca: Derleyici veya İşlemci bunları reordering (yeniden sıralama) yapabilir!
}

void thread_reader() {
    while (!ready_flag) { // Polling (Sürekli kontrol)
        ; 
    }
    // ready_flag true olsa bile g_val hala 10 görülebilir!
    std::cout << g_val << std::endl; 
}
```

**🔍 Arka Plan (Under the Hood)**
1.  **Compiler Reordering:** Derleyici, `ready_flag`'i set etmeyi `g_val` atamasının önüne alabilir.
2.  **Processor Optimizations:** *Branch Prediction* (Dallanma tahmini) ve *Prefetching* (Önceden getirme) gibi mekanizmalar bellek görünürlüğünü bozar.
3.  **Cache Invalidation:** Bir thread'in bellekte yaptığı değişiklik, diğer thread'in işlemci önbelleğinde (cache) henüz güncellenmemiş olabilir.

---

## 4. Modern C++ Bellek Modeli Terimleri (13:40 - 16:00)

Hoca, ilerleyen derslerde derinleşecek olan standart terminolojinin temelini attı:

*   **Happened-before (Önce gerçekleşti):** Bir işlemin etkilerinin, başka bir işlem başlamadan önce bellek tarafından garanti altına alınması durumu.
*   **Synchronized-with (Bununla senkronize):** Belirli araçlar (Mutex, Atomic) kullanıldığında, bir thread'deki değişikliğin diğer thread tarafından "görülebilir" (visible) olma garantisi.

---

## 5. Veri Yarışları (Data Race) ve "Torn Write" Problemi (16:00 - 21:30)

**🚩 Kritik Nokta / Mülakat Sorusu**
**Soru:** Bir değişkenin değerini değiştirmek her zaman atomik midir?
**Cevap:** Hayır. Özellikle 2 byte'lık bir verinin sadece 1 byte'ının yazılıp, o sırada başka bir thread'in araya girmesiyle veri bozulabilir. Hoca buna **"Torn Write" (Yırtık/Parçalı Yazma)** dedi.

**🖼️ Görselleştirme (ASCII Art - Torn Write)**
```text
Thread A Yazmak İstiyor: 0x1234
Thread B Yazmak İstiyor: 0x4567

Aşama 1: Thread A, ilk byte'ı yazar -> [0x12][0x??]
Aşama 2: Thread B araya girer, hepsini yazar -> [0x45][0x67]
Aşama 3: Thread A uyanır, kalan byte'ı yazar -> [0x45][0x34]  <-- BOZUK VERİ (Torn Write)
```

---

## 6. Deadlock vs. Livelock Ayrımı (21:30 - 25:00)

**📊 Standart Karşılaştırması**

| Özellik | Deadlock (Ölü Kilit) | Livelock (Canlı Kilit) |
| :--- | :--- | :--- |
| **Thread Durumu** | Bloke olmuş (Blocked), ilerlemiyor. | Çalışıyor (Running), CPU harcıyor. |
| **İlerleme** | Yok. | Yok (Sürekli birbirini iptal eden işlemler). |
| **Analoji** | Karşılıklı kilitli kapılar. | Kapıda birbirine yol veren iki nazik insan. |

**🚩 Hoca'nın İdiomları**
*   **"Türklerin hesap ödeme hali":** İki tarafın da sürekli "ben ödeyeceğim" diyerek birbirini engellemesi ama sonucun değişmemesi (Livelock).
*   **"Polite people at a door":** Dar bir kapıda iki nazik insanın sürekli "Siz buyurun" diyerek kapıdan geçememesi.

**⚙️ Livelock Kod Taslağı (00:24:30)**
Hoca, `std::atomic` bir değişken kullanarak bir livelock senaryosu hazırladı.

```cpp
std::atomic<int> counter{0};

void task_a() {
    while (true) {
        if (counter % 2 == 0) { // Çift ise arttır
            counter++; 
        } else {
            std::cout << "waiting..."; // <-- CPU harcıyor ama ilerleme kaydedemiyor
        }
    }
}
```

---

Necati Ergin'in 63. dersinin [00:25:00] ile [00:54:00] arasındaki kısmını kapsayan teknik inceleme aşağıdadır:

# 63. Ders: Mutex, Lock Sınıfları ve Senkronizasyon Primitifleri - Bölüm II

## 7. Atomik Operasyonlar ve Kesintisizlik Garantisi (00:25:00 - 00:31:00)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Veri yarışlarını (Data Race) engellemenin yollarından biri de değişkenleri atomik yapmaktır. Bir işlemin başladığı an ile bittiği an arasına başka bir *Thread*'in girme ihtimali olmamalıdır.

**⚙️ Teknik Detay ve Sentaks**
Hoca, atomik işlemlerin bellek görünürlüğü üzerindeki etkisini şu şekilde açıkladı:

```cpp
#include <atomic>

std::atomic<int> g_counter{10}; // <-- Hoca: Atomik değişken kullanırsanız Data Race ihtimali yok.

void increment_task() {
    // g_counter++ işlemi atomiktir.
    // Diğer thread'ler değeri ya 10 (artmadan önce) ya da 11 (arttıktan sonra) görür.
    // 10.5 gibi "ara bir değer" veya "bozuk veri" (torn read) görme ihtimali yoktur.
    g_counter++; 
}
```

**🔍 Arka Plan (Under the Hood)**
*   **Atomic Operation:** Operasyonun başlamasıyla bitmesi arasına ikinci bir thread'in girme ihtimalinin olmadığı operasyondur.
*   **Data Race Prevention:** Atomik değişkenler kullanıldığında, derleyici ve işlemci operasyonun bütünlüğünü korumak için gerekli *Memory Barrier* (Bellek bariyeri) komutlarını ekler.

---

## 8. Mutex Sınıfları ve `native_handle()` (00:31:00 - 00:35:45)

**📊 Standart Karşılaştırması**
Hoca, kütüphanedeki Mutex sınıflarının yeteneklerini bir hiyerarşi içinde sundu:

| Mutex Sınıfı | Temel Özellik |
| :--- | :--- |
| `std::mutex` | En basit, minimal arayüz. |
| `std::timed_mutex` | Süre bazlı (`try_lock_for`, `try_lock_until`) deneme yapabilir. |
| `std::recursive_mutex` | Aynı thread tarafından birden fazla kez kilitlenebilir. |
| `std::shared_mutex` | Reader/Writer kilidi (C++17). |

**⚙️ Teknik Detay: `native_handle`**
Hoca, C++ standart kütüphanesinin yetmediği yerlerde işletim sistemi API'sine geçiş kapısını gösterdi:

```cpp
std::mutex mtx;
auto handle = mtx.native_handle(); // <-- Hoca: İşletim sistemi API'sini (Windows/Linux) çağırmak için kullanılır.

// Örn: Windows'ta thread priority set etmek için bu handle doğrudan ilgili OS fonksiyonuna geçilir.
```

**🚩 Kritik Nokta**
`native_handle()` hemen hemen tüm *Concurrency* (Eş zamanlılık) kütüphanesi sınıflarında bulunur. Standart kütüphane "taşınabilirlik" adına her yeteneği sunmaz; bu durumda handle üzerinden "durumdan vazife çıkartarak" platforma özel kod yazılır.

---

## 9. RAII Sınıfları ve `std::adopt_lock` Etiketi (00:35:45 - 00:45:00)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Mutex'i manuel kilitlediğimizde, kritik bölge içinde bir *Exception* (İstisna) fırlatılırsa `unlock()` fonksiyonu asla çağrılmaz. Bu, sistemin kilitlenmesine (Deadlock) yol açar. RAII (Resource Acquisition Is Initialization) bu kilidi nesne ömrüne bağlar.

**⚙️ Teknik Detay ve Sentaks (`std::adopt_lock`)**
Hoca, bazen bir Mutex'in RAII nesnesi oluşturulmadan önce zaten kilitlenmiş olabileceği senaryoyu açıkladı:

```cpp
std::mutex mtx1, mtx2;

void process_data() {
    std::lock(mtx1, mtx2); // Mutex'ler kilitlendi (Deadlock-free algoritma ile)
    
    // Şimdi bu kilitleri RAII ile sarmalamalıyız ama tekrar kilitlememeliyiz!
    std::lock_guard<std::mutex> lock1(mtx1, std::adopt_lock); // <-- Hoca: 'Ben zaten kilitledim, sadece sahipliği al' demek.
    std::lock_guard<std::mutex> lock2(mtx2, std::adopt_lock); // // <-- Kritik: adopt_lock_t türünden bir tag class.
    
    // ... Kritik bölge ...
} // lock1 ve lock2 yok olunca mtx1 ve mtx2 otomatik UNLOCK edilir.
```

**🔍 Arka Plan (Under the Hood)**
*   **Tag Class:** `std::adopt_lock_t` gibi sınıflar sadece *Function Overload Resolution* (Fonksiyon yükleme çözünürlüğü) sağlamak için kullanılır. Bellekte yer kaplamazlar.
*   **CTAD (C++17):** Hoca, `std::lock_guard<std::mutex>` yazmak yerine doğrudan `std::lock_guard` yazabildiğimizi (Class Template Argument Deduction) ve bunun "görüntü kirliliğini" engellediğini belirtti.

---

## 10. `std::unique_lock` ve Esnek Kilitleme Stratejileri (00:45:00 - 00:54:00)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
`std::lock_guard` çok kısıtlıdır; oluşturulduğu an kilitler ve yok olana kadar bırakmaz. `std::unique_lock` ise kilidi erteleme, süreyle deneme veya manuel bırakma gibi gelişmiş yetenekler sunar.

**⚙️ Teknik Detay ve Sentaks (Farklı Tag'ler)**

```cpp
std::mutex mtx;

// 1. Defer Lock: Edinir ama kilitlemez
std::unique_lock ulax(mtx, std::defer_lock); // <-- Hoca: Referans olarak bağlandı ama lock() çağrılmadı.
// ... başka işler ...
ulax.lock(); // İstediğim an manuel kilitleyebilirim.

// 2. Try to Lock: Bloke olmadan dener
std::unique_lock ulax2(mtx, std::try_to_lock); 
if (ulax2.owns_lock()) { // <-- Hoca: Kilidi alabildim mi? diye test edilir.
    // ...
}
```

**📊 Standart Karşılaştırması (`std::unique_lock` vs `std::lock_guard`)**

| Özellik | `lock_guard` | `unique_lock` |
| :--- | :--- | :--- |
| **Kopyalama** | Yasak | Yasak |
| **Taşıma (Move)** | Yasak | **Serbest** (Movable) |
| **Manuel Unlock** | Yok | Var |
| **Süre Bazlı Lock** | Yok | Var |
| **Maliyet** | Çok düşük (Minimal) | Biraz daha yüksek (Overhead) |

**🚩 Kritik Nokta / Mülakat Sorusu**
**Soru:** Neden her yerde `unique_lock` kullanmıyoruz?
**Cevap:** Hoca "C++'ta her ilave yetenek (capability), ilave maliyet (cost) demektir" prensibini hatırlattı. Eğer `lock_guard` yetiyorsa, içsel durum tutmayan (daha hafif) olan o tercih edilmelidir. Ancak `std::condition_variable` ile çalışılacaksa `unique_lock` zorunludur.

---

Necati Ergin'in 63. dersinin [00:54:00] ile [01:25:00] arasındaki kısmını kapsayan teknik inceleme aşağıdadır:

# 63. Ders: Mutex, Lock Sınıfları ve Senkronizasyon Primitifleri - Bölüm III

## 11. `std::unique_lock` ve Süre Bazlı Kilitleme (00:54:00 - 01:00:00)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Bazı durumlarda bir thread, kilidi sonsuza kadar beklemek yerine sadece belirli bir süre denemek ister. Eğer kilit alınamazsa thread bloklanmamalı (non-blocking), alternatif bir işe yönelmelidir.

**⚙️ Teknik Detay ve Sentaks (Chrono Entegrasyonu)**
Hoca, `std::unique_lock`'ın `std::chrono` kütüphanesiyle nasıl iç içe çalıştığını gösterdi:

```cpp
#include <chrono>
#include <mutex>

using namespace std::chrono_literals; // <-- Hoca: 'Literals' kullanmak kodun okunabilirliğini arttırır.

std::timed_mutex t_mtx;

void demo_timer() {
    // 1. Duration (Süre) bazlı deneme
    std::unique_lock ulax(t_mtx, 100ms); // 100 milisaniye boyunca kilitlemeyi dene.
    
    // 2. Time Point (Zaman noktası) bazlı deneme
    auto tp = std::chrono::steady_clock::now() + 500ms;
    std::unique_lock ulax2(t_mtx, tp); // Bu zaman noktasına kadar dene.

    if (ulax.owns_lock()) { // <-- Hoca: Kilidin edinilip edilmediğini 'owns_lock' veya 'operator bool' ile sorgularız.
        // Kritik bölge...
    } else {
        // Kilit alınamadı, thread bloke olmadı, yoluna devam ediyor.
    }
}
```

**🚩 Kritik Nokta**
`owns_lock()` ve `operator bool()` aynı işi yapar. Hoca, "Eğer kilit edinilmişse `true` döner, aksi halde `false` döner" diyerek, kilit alınamazsa RAII nesnesinin yine de oluştuğunu ama "içinin boş" olduğunu vurguladı.

---

## 12. `unique_lock` Sahipliği: `release()` ve `mutex()` Farkı (01:00:00 - 01:04:00)

**🔍 Arka Plan (Under the Hood)**
Hoca, akıllı pointerlardaki (smart pointers) mantığın burada da geçerli olduğunu belirtti.

```cpp
std::unique_lock ulax(mtx);

// 1. mutex(): Sahipliği bırakmaz, sadece adresi verir.
std::mutex* p1 = ulax.mutex(); 

// 2. release(): Sahipliği tamamen bırakır.
std::mutex* p2 = ulax.release(); // <-- Hoca: Artık RAII garantisi bitti! 
// p2'yi manuel olarak UNLOCK etmek zorundasınız, yoksa kilitli kalır.
```

**🚩 Kritik Nokta / Mülakat Sorusu**
**Soru:** `release()` fonksiyonu mutex'i serbest mi bırakır (unlock)?
**Cevap:** HAYIR. `release()`, RAII nesnesinin mutex üzerindeki "yönetimini/mülkiyetini" bırakır. Mutex kilitli kalmaya devam eder. Hoca bunu "başımızın belası terminoloji" olarak nitelendirdi.

---

## 13. Jenerik Programlama ve "Lockable" Gereklilikleri (01:04:00 - 01:09:00)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
C++ standart kütüphanesi sadece kendi mutex sınıflarını değil, belirli arayüzleri (interface) sağlayan *Custom* (Özel) mutex sınıflarını da desteklemelidir (Örn: Boost Mutex veya platforma özel kilitler).

**⚙️ Teknik Detay: Duck Typing Mantığı**
Hoca, `std::unique_lock<T>` içindeki `T`'nin ne olması gerektiğini açıkladı:

```cpp
class MyCustomMutex {
public:
    void lock();   // <-- Zorunlu!
    void unlock(); // <-- Zorunlu!
    bool try_lock();
};

// Standart kütüphane bu sınıfı 'Lockable' (Kilitlenebilir) kabul eder.
std::unique_lock<MyCustomMutex> ulax(my_mtx); // <-- Hoca: C++20 öncesi bu sadece dokümante ediliyordu.
```

**🔍 Arka Plan (Under the Hood)**
C++20 öncesinde bu bir "Concept" değil, sadece bir gereklilik (requirement) idi. Derleyici, `unique_lock` içinde `m.lock()` çağrısı yaptığında eğer sınıfta bu isimde bir fonksiyon yoksa *Substitution Failure* (Yerine koyma hatası) oluşur.

---

## 14. `std::recursive_mutex` ve Özinelemeli Fonksiyonlar (01:09:00 - 01:17:00)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Sıradan bir `std::mutex`, aynı thread tarafından ikinci kez kilitlenmeye çalışıldığında UB (Undefined Behavior/Tanımsız Davranış) oluşur. Recursive fonksiyonlarda veya bir fonksiyonun diğerini çağırdığı kilitli yapılarda bu durum bir ihtiyaçtır.

**⚙️ Teknik Detay ve Sentaks (Hoca'nın Örneği)**

```cpp
#include <iostream>
#include <mutex>

std::recursive_mutex g_recursive_mtx;
int g_count = 0;

void recursive_func(char c, int n) {
    if (n <= 0) return;

    std::lock_guard lock(g_recursive_mtx); // <-- Hoca: Burası kritik! Her çağrıda tekrar kilitlenir.
    std::cout << c << " " << ++g_count << std::endl;
    
    recursive_func(c, n - 1); // Özinelemeli çağrı
}
```

**🔍 Arka Plan (Under the Hood)**
`std::recursive_mutex`, içinde kilidin kaç kez edinildiğini tutan bir sayaç (counter) ve sahipliği tutan thread ID'sini barındırır. Bu ek veriler nedeniyle normal mutex'e göre daha fazla *Overhead* (İlave maliyet) barındırır.

---

## 15. "Active Object" Örüntüsü ve Deadlock Çözümü (01:17:00 - 01:25:00)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Bir sınıfın (`Active Object`) tüm üye fonksiyonlarının *Thread-Safe* olması için her fonksiyon başında kilitleme yapılır. Ancak bir üye fonksiyon (A), aynı sınıfın başka bir üye fonksiyonunu (B) çağırdığında, B fonksiyonu da aynı mutex'i kilitlemeye çalışacaktır.

**🚩 Kritik Nokta / Mülakat Sorusu**
**Soru:** Bir sınıfın üye fonksiyonları birbirini çağırıyorsa hangi mutex türü kullanılmalıdır?
**Cevap:** `std::recursive_mutex`. Aksi takdirde thread kendi kendini kilitler (Self-deadlock).

**⚙️ Hoca'nın Database Access Örneği:**

```cpp
class DatabaseAccess {
    std::recursive_mutex m_mtx; // <-- Hoca: Mutex değil, recursive_mutex olmalı!
public:
    void create_table() {
        std::lock_guard lock(m_mtx);
        // ...
    }

    void insert_data() {
        std::lock_guard lock(m_mtx);
        // ...
    }

    void setup_db() {
        std::lock_guard lock(m_mtx); // 1. Kilitleme
        create_table();              // 2. Kilitleme (Aynı mutex!)
        insert_data();               // 3. Kilitleme (Aynı mutex!)
    }
};
```

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  **Terminoloji Karışıklığı:** `release()`'in kilidi açtığını sanmak (açmaz, sadece mülkiyeti bırakır).
2.  **Yanlış Mutex Seçimi:** Recursive çağrılarda normal `std::mutex` kullanıp programın "terminate" edilmesine yol açmak.
3.  **Cost (Maliyet):** `recursive_mutex` ve `unique_lock` sınıflarının esnekliğinin bedelini performansla ödediğimizi unutmak.

---

Necati Ergin'in 63. dersinin [01:25:00] ile transkript sonu [02:42:32] arasındaki kısmını kapsayan teknik inceleme aşağıdadır:

# 63. Ders: Mutex, Lock Sınıfları ve Senkronizasyon Primitifleri - Bölüm III & Condition Variable Giriş

## 16. `std::timed_mutex` ve `try_lock_for` Uygulaması (01:25:00 - 01:34:00)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Bazı yüksek performanslı sistemlerde, bir thread'in kilidi beklerken sonsuza kadar bloklanması istenmez. `try_lock_for` ile thread, "eğer şu sürede kilidi alamazsan vazgeç ve başka iş yap" diyebilir.

**⚙️ Teknik Detay ve Sentaks (Hoca'nın Örneği)**
Hoca, thread'lerin kilidi alamadığı senaryoyu `std::osyncstream` (C++20) kullanarak simüle etti:

```cpp
#include <mutex>
#include <syncstream> // <-- Hoca: C++20 ile gelen osyncstream, garbled (karışık) çıktıları engeller.
#include <chrono>

std::timed_mutex t_mtx;

void increment(int id) {
    using namespace std::chrono_literals;
    // 200 mikrosaniye boyunca kilidi edinmeye çalış
    if (t_mtx.try_lock_for(200us)) { // <-- Hoca: Bloke etmez, kilitlenirse true döner.
        // ... Kritik Bölge ...
        std::osyncstream(std::cout) << "Thread " << id << " kritik bölgeye girdi.\n";
        std::this_thread::sleep_for(10ms); // Kilidi meşgul ediyoruz
        t_mtx.unlock();
    } else {
        std::osyncstream(std::cout) << "Thread " << id << " kritik bölgeye GİREMEDİ.\n";
    }
}
```

**🔍 Arka Plan (Under the Hood)**
*   **Non-deterministic Behavior:** Hoca, hangi thread'in içeri gireceğinin "deterministik olmadığını" ve süre değerleriyle (nano/mikro saniye) oynandığında çıktıların değişeceğini gösterdi.
*   **osyncstream:** Standart `std::cout` thread-safe olsa da, karakter bazlı çalışır. `osyncstream` tüm cümleyi tamponlayıp atomik olarak yazdırır.

---

## 17. `std::scoped_lock` ve Variadic Yapısı (01:34:00 - 02:10:00)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Birden fazla mutex kilitlenmesi gerektiğinde, threadler farklı sıralarla kilitleme yaparsa *Deadlock* oluşur. `std::scoped_lock` (C++17), variadik (değişken sayıda parametre alan) yapısıyla birden fazla mutex'i güvenli bir algoritmayla kilitler.

**📊 Standart Karşılaştırması**
Hoca, `scoped_lock`'ın "effectively replaces `lock_guard`" (etkin bir şekilde `lock_guard`'ın yerini aldığını) belirtti.

| Özellik | `lock_guard` | `scoped_lock` |
| :--- | :--- | :--- |
| **Mutex Sayısı** | Sadece 1 | **N adet** (Variadic) |
| **Deadlock Koruması** | Yok | **Var** (Deadlock-free locking algoritması) |
| **Farklı Tür Mutex** | Tek tür | **Farklı türler** (std::mutex ve std::recursive_mutex aynı anda) |

**⚙️ Teknik Detay: Swap Senaryosu**

```cpp
void swap_custom(X& lhs, X& rhs) {
    // lhs.m_mtx ve rhs.m_mtx kilitlenmeli
    std::scoped_lock guard(lhs.m_mtx, rhs.m_mtx); // <-- Hoca: Sıranın önemi yok, algoritma deadlock'u engeller.
    // ... swap işlemleri ...
} // guard yok olunca her iki kilit de serbest kalır.
```

**🚩 Kritik Nokta**
Hoca, `std::scoped_lock` kullanılırken artık `std::lock` (global fonksiyon) ve `std::adopt_lock` kombinasyonuna "neredeyse hiç gerek kalmadığını" vurguladı.

---

## 18. `std::shared_mutex` (Readers-Writer Lock) (02:10:00 - 02:35:00)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Bazı veriler (örn: Database) çok sık okunur ama nadiren güncellenir (*seldom updated*). Normal mutex'te okuyucular birbirini bloklar. `shared_mutex`, birden fazla okuyucunun (reader) aynı anda içeri girmesine izin verirken, yazıcıya (writer) özel hak tanır.

**⚙️ Teknik Detay: Exclusive vs Shared Access**

```cpp
#include <shared_mutex>

std::shared_mutex rw_mtx;
int g_data = 0;

// WRITER: Veriyi güncelleyen thread
void writer() {
    std::lock_guard lock(rw_mtx); // <-- Hoca: lock_guard veya unique_lock 'Exclusive' (özel) hak alır.
    g_data++;
}

// READER: Veriyi sadece okuyan thread
void reader() {
    std::shared_lock lock(rw_mtx); // <-- Hoca: shared_lock 'Shared' (paylaşımlı) hak alır.
    // Bu bölgede aynı anda 100 reader olabilir!
    std::cout << g_data;
}
```

**🔍 Arka Plan (Under the Hood)**
*   **Multiple Readers - Single Writer:** Bir writer içeri girdiğinde, ne reader ne de başka bir writer içeri girebilir. Ancak reader'lar birbirini bloklamaz.
*   **Maliyet:** Hoca, `shared_mutex`'in normal mutex'e göre daha fazla bellek alanı kullandığını ve daha yüksek bir *Overhead* (yük) barındırdığını belirtti.

---

## 19. `std::condition_variable` Giriş ve Analoji (02:35:00 - 02:42:32)

**🧠 Neden İhtiyaç Duyuldu? (Rationale)**
Bir thread'in, başka bir thread'in işini bitirmesini beklemesi gerektiğinde sürekli "hazır mı?" diye sorması (*Polling*) CPU'yu %100 kullanır ve verimsizdir. `condition_variable`, thread'i uyutur ve veri hazır olduğunda uyandırılmasını sağlar.

**🚩 Hoca'nın İdiomu: "Eskişehir Treni Analojisi"**
Hoca konuyu şu meşhur hikayeyle özetledi:
*   **Senaryo:** İstanbul'dan Ankara trenine bindiniz, Eskişehir'de ineceksiniz.
*   **Polling (Kötü):** Hiç uyumayıp her 2 dakikada bir "Eskişehir'e geldik mi?" diye dışarı bakmak. (CPU harcar, yorulursunuz).
*   **Condition Variable (İyi):** Kondüktöre (CV) "Eskişehir'e gelince beni uyandır" deyip uyumak. (Thread bloklanır, CPU harcamaz). Kondüktör sizi uyandırdığında (Signal/Notify) istasyona gelindiğinden emin olursunuz.

** Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  **Shared Lock Karışıklığı:** `shared_lock` (RAII sınıfı) ile `shared_mutex` (Mutex türü) arasındaki isim benzerliğinden kaynaklanan hatalar.
2.  **Busy Wait (Polling):** Bir olayı beklerken `while(flag);` gibi boş döngülerle işlemciyi sömürmek.
3.  **Variadic CTAD:** `scoped_lock` kullanırken template argümanlarını manuel yazıp hata yapmak (Hoca, CTAD'ın variadic şablonlarda ne kadar hayat kurtarıcı olduğunu gösterdi).

---



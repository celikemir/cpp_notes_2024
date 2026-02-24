Bu notlar, Necati Ergin’in 17 Şubat 2025 tarihli 64. ders transkriptine dayalı, teknik derinliği yüksek bir dokümantasyondur. "Özetleme, yeniden inşa et" kuralına sadık kalınarak hazırlanmıştır.

---

# 64. Ders: Condition Variables ve Concurrency Seviyeleri - Bölüm 1

## 1. Concurrency Araçlarında "Seviye" Kavramı (High-Level vs. Low-Level)
**[00:00:00 - 01:14:00]**

Necati Hoca, derse başlamadan önce özelden gelen bir soruyu cevaplıyor: *"Thread oluşturmanın tek yolu `std::thread` nesnesi mi?"* Hayır. C++ standart kütüphanesi, thread yönetimi için farklı soyutlama seviyeleri sunar.

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Doğrudan işletim sistemi API'lerini (Pthreads veya WinAPI) kullanmak taşınabilirlik (portability) sorunlarına ve karmaşık bir kaynak yönetimine neden oluyordu. C++ standart kütüphanesi, kullanıcıyı bu karmaşıklıktan kurtarmak için farklı katmanlar geliştirmiştir.

### ⚙️ Teknik Detay ve Sentaks
C++ kütüphanesiyle bir işi asenkron yürütmenin yolları:
1.  **std::thread (Low-Level):** En alt seviye araçtır. Kaynak yönetimi (join/detach) tamamen kullanıcıdadır. Destructor çağrıldığında thread hala `joinable` durumdaysa `std::terminate` çağrılır.
2.  **std::jthread (C++20 - RAII Wrapper):** `std::thread`'i sarmalayan bir RAII sınıfıdır. Destructor'ında otomatik join yapar ve "Cooperative Interruption" (işbirliğiyle durdurma) mekanizması için `stop_token` desteği sunar.
3.  **std::async (High-Level):** Arka planda thread oluşturulmasını, fonksiyonun geri dönüş değerinin (return value) veya fırlattığı exception'ın (istisna) çağrılan koda iletilmesini otomatik yönetir.
4.  **Parallel Algorithms (Highest Level - C++17/20):** `std::sort` gibi algoritmalara bir "execution policy" (yürütme politikası) geçerek işlemin kütüphane tarafından paralel thread'lere bölünmesini sağlar.

### 🔍 Arka Plan (Under the Hood)
"Yüksek seviye" (high-level) kavramı, kullanıcıdan daha fazla detayın gizlenmesi (abstraction) demektir:
*   **Low-Level:** Arka planda olan her şeyi görürsünüz (Native handle, ID yönetimi).
*   **High-Level:** Arka planda thread'in oluşturulması, `join` edilmesi ve bir **Shared State** (paylaşılan durum) üzerinden sonucun iletilmesi sizden gizlenir.

---

## 2. Condition Variable (Koşul Değişkeni) Giriş ve Senkronizasyon İhtiyacı
**[01:14:00 - 01:42:00]**

C++'ta en sık karşılaşılan senaryo: Bir thread'in (Producer/Üretici) bir işi bitirmesini, diğer thread'in (Consumer/Tüketici) ise bu iş bitmeden kendi işlemine devam edememesini yönetmektir.

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir thread'in diğerini beklemesi için kullanılan geleneksel yöntemlerde (Polling) işlemci zamanı boşa harcanır. `std::condition_variable`, bir thread'in bir koşul gerçekleşene kadar "uyumasını" (bloke olmasını) ve koşul gerçekleştiğinde diğer thread tarafından uyandırılmasını sağlar.

### 🖼️ Görselleştirme (Shared State Kanalı)
```text
[ Thread A (Producer) ] ---(Shared State: Ready Flag)---> [ Thread B (Consumer) ]
          |                                                       |
   (İşi bitirir)                                           (Koşulu bekler)
   (Flag = true)                                           (Uykudan uyanır)
   (Notify eder) ----------------------------------------> (İşe devam eder)
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Neden sadece bir `bool flag` kullanıp döngüyle kontrol etmiyoruz?
**Cevap:** Bu işleme **Polling** denir. İki büyük sorunu vardır:
1. **CPU Verimliliği:** Thread sürekli uyanıp kontrol yaptığı için CPU'yu meşgul eder.
2. **Synchronized-With Eksikliği:** Bir mutex veya atomik değişken kullanmadan flag'i `true` olarak görseniz bile, CPU caching ve derleyici optimizasyonları nedeniyle diğer thread'in yaptığı asıl veri değişikliklerini (data changes) "garantili" olarak göremeyebilirsiniz.

---

## 3. Polling: Kötü Uygulama Örneği (Manual Checking)
**[01:42:00 - 02:27:00]**

Hoca, `condition_variable` olmadan yapılan hatalı ve verimsiz "polling" yaklaşımını kodluyor.

### ⚙️ Teknik Detay ve Sentaks
```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <chrono>

bool ready_flag = false; // <-- Shared State (Paylaşılan Durum)
std::mutex mtx;
int data = 0;

void consumer() {
    std::unique_lock<std::mutex> lock(mtx); // <-- Unique lock şart, döngü içinde açılacak
    
    while (!ready_flag) { // <-- POLLING: Sürekli kontrol (Kötü yöntem)
        lock.unlock(); // <-- Mutex'i bırak ki Producer yazabilsin
        std::this_thread::yield(); // <-- "Benim sıramı başkasına verebilirsin" (Opsiyonel)
        std::this_thread::sleep_for(std::chrono::milliseconds(10)); // <-- CPU'yu biraz rahatlat
        lock.lock(); // <-- Tekrar kilitle ve flag'e bak
    }
    
    // Koşul sağlandı, veriyi kullanabiliriz
    std::cout << "Consumer using data: " << data << "\n";
}

void producer() {
    {
        std::lock_guard<std::mutex> lock(mtx); // <-- Veriyi kilit altında hazırla
        data = 42;
        ready_flag = true;
    } // <-- Lock burada serbest kalır
}
```

### 🔍 Arka Plan (Analoji)
**Anthony Williams - Concurrency in Action Analojisi:**
Bir mağazadan ürün bekliyorsunuz.
*   **Polling:** Mağazayı günde 10 kere arayıp "Geldi mi?" diye sormak (Siz ve mağaza yorulur).
*   **Condition Variable:** Mağazaya numaranızı bırakmak ve "Ürün gelince beni arayın (Notify)" demek (Siz uyursunuz/başka iş yaparsınız, sinyal gelince uyanırsınız).

---

## 4. `std::condition_variable` ile Senkronizasyonun Anatomisi
**[02:27:00 - 02:45:00]**

C++'ta `std::condition_variable` kullanımı için 3 bileşen zorunludur:
1.  **Mutex:** Paylaşılan durumu (flag/data) korumak için.
2.  **Condition Variable Nesnesi:** Uyuma ve uyandırma sinyali için.
3.  **Koşulun Kendisi (Predicate):** Genellikle bir `bool` flag veya "liste boş değil" gibi bir mantıksal durum.

### ⚙️ Teknik Detay: Temel Üye Fonksiyonlar
*   `wait(unique_lock& lock, Predicate pred)`: `pred` false döndüğü sürece kilidi bırakır ve thread'i uyutur.
*   `notify_one()`: Bekleyen (uyuyan) thread'lerden sadece **bir tanesini** uyandırır.
*   `notify_all()`: Bekleyen **bütün** thread'leri uyandırır.

### 🚩 Kritik Nokta: Neden `std::unique_lock`?
**Mülakat Sorusu:** `condition_variable::wait` fonksiyonuna neden `std::lock_guard` geçemiyoruz?
**Cevap:** Çünkü `wait` fonksiyonu arka planda atomik olarak şu işlemleri yapar:
1.  Mutex'i serbest bırakır (`unlock`).
2.  Thread'i uyku moduna alır.
3.  Uyanınca Mutex'i tekrar edinir (`lock`).
`lock_guard` nesnesi manuel olarak `unlock` ve `lock` edilemediği (aradaki fonksiyonları yoktur) için bu işlemde kullanılamaz.

### 🔗 Standart Karşılaştırması
| Özellik | `std::condition_variable` | `std::condition_variable_any` |
| :--- | :--- | :--- |
| **Gereksinim** | Sadece `std::unique_lock<std::mutex>` ile çalışır. | Herhangi bir "Lockable" (örn. `shared_lock`) ile çalışır. |
| **Performans** | Daha verimlidir (Native OS optimizasyonlarını kullanır). | Daha esnektir ama ek maliyet (overhead) getirebilir. |

---

## 64. Ders: Condition Variables ve Concurrency Seviyeleri - Bölüm 2

**[00:25:51 - 00:54:00]**

### 5. "Synchronized-With" İlişkisi ve Hafıza Garantileri
Necati Hoca, concurrency (eş zamanlılık) dünyasındaki en kritik ama anlaşılması en zor kavramlardan birine değiniyor: **Synchronized-With** (ile senkronize edilmiş).

#### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Sadece bir değişkenin değerinin değiştiğini görmek yetmez. Thread A'nın yaptığı tüm değişikliklerin (bellek yazmalarının), Thread B tarafından "görünür" (visible) olması gerekir. Derleyiciler ve işlemciler performansı artırmak için kodun sırasını değiştirebilir (reordering) veya değerleri cache'te (ön bellek) tutabilir.

#### ⚙️ Teknik Detay ve Sentaks
`std::mutex` ve `std::condition_variable` kullanıldığında dil şu garantiyi verir:
*   **Thread A:** Mutex'i kilitledi -> Veriyi değiştirdi -> Mutex'i açtı.
*   **Thread B:** Aynı Mutex'i kilitledi.
*   **Sonuç:** Thread B, Thread A'nın kilit altındayken yaptığı her şeyi **garantili** olarak görür. Buna "Synchronized-With" ilişkisi denir.

#### 🚩 Kritik Nokta: Tipik Hata
**Hata:** Mutex kullanmadan sadece `bool flag` ile polling yapmak.
**Risk:** Flag'in `true` olduğunu görseniz bile, asıl `data` değişkeninin güncellenmiş hali henüz sizin işlemcinizin cache'ine gelmemiş olabilir. Bu durum **Undefined Behavior (UB)** değildir ama mantıksal olarak hatalı sonuçlar doğurur (Data Race olmasa bile tutarsız veri).

---

### 6. Spurious Wakeup (Sahte Uyanma) Fenomeni
**[00:49:00 - 00:57:00]**

`condition_variable::wait` fonksiyonu çağrıldığında thread'in uyanması için her zaman bir sinyal (`notify`) gelmiş olması gerekmez.

#### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
İşletim sistemi düzeyindeki thread optimizasyonları ve sinyal mekanizmaları nedeniyle, bir thread bazen "hiçbir sebep yokken" uykusundan uyanabilir. Buna **Spurious Wakeup** denir.

#### ⚙️ Teknik Detay ve Sentaks
Sahte uyanmalara karşı korunmanın iki yolu vardır:
1.  **Manuel Döngü (Old School):**
    ```cpp
    while (!ready_flag) { // <-- Koşulu tekrar kontrol et
        cv.wait(lock); // <-- Eğer uyanma sahteyse tekrar uyu
    }
    ```
2.  **Predicate Overload (Modern/Safe):**
    ```cpp
    cv.wait(lock, []{ return ready_flag; }); // <-- Hoca bunu öneriyor
    ```

#### 🔍 Arka Plan (Under the Hood)
Predicate alan `wait` overload'u arka planda aslında şu kodu çalıştırır:
```cpp
while (!pred()) { // <-- "Durumdan vazife çıkartmak": Koşul sağlanmadığı sürece uyu
    wait(lock);
}
```

---

### 7. iStack: Thread-Safe Bir Veri Yapısı İnşası
**[01:09:00 - 01:29:00]**

Necati Hoca, `std::vector` kullanarak thread'ler arasında güvenle paylaşılabilen bir yığın (stack) yapısı oluşturuyor.

#### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Standart STL konteynerları (`std::vector`, `std::stack`) thread-safe değildir. Birden fazla thread aynı anda `push` veya `pop` yaparsa iç yapı bozulur.

#### ⚙️ Teknik Detay ve Sentaks
```cpp
#include <vector>
#include <mutex>
#include <condition_variable>

class iStack {
    std::vector<int> m_vec;
    std::mutex m_mtx;
    std::condition_variable m_cv;
public:
    // <-- Hoca: Kopyalama ve taşımayı kapatmak doğal, çünkü Mutex ve CV kopyalanamaz!
    iStack(const iStack&) = delete;
    iStack& operator=(const iStack&) = delete;

    void push(int val) {
        {
            std::scoped_lock lock(m_mtx); // <-- C++17 RAII Lock
            m_vec.push_back(val);
        } // <-- Kilit burada açılır (Scope sonu)
        m_cv.notify_one(); // <-- Bekleyen BİR thread'i uyandır
    }

    int pop() {
        std::unique_lock lock(m_mtx); // <-- wait() için unique_lock mecburi!
        
        // <-- Kritik: Eğer stack boşsa uyu, veri gelince uyan
        m_cv.wait(lock, [this] { return !m_vec.empty(); }); 
        
        int val = m_vec.back();
        m_vec.pop_back();
        return val;
    }
};
```

#### 🔍 Arka Plan (Memory Layout)
*   **Mutable:** Eğer `size()` gibi `const` bir fonksiyonda kilit kullanılması gerekseydi, `m_mtx` ve `m_cv` veri elemanlarının `mutable` işaretlenmesi gerekirdi (Çünkü `lock()` işlemi objenin mantıksal durumunu değiştirmese de fiziksel bitlerini değiştirir).

#### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `notify_one()` fonksiyonu neden kilit (`lock`) altında çağrılmıyor?
**Hoca'nın Yanıtı:** Şart değildir ve performans için kilit dışında olması daha iyidir. Eğer kilit altındayken notify ederseniz, uyandırdığınız thread hemen mutex'i edinmeye çalışır ama siz hala kilidi tuttuğunuz için tekrar bloke olabilir (Wait-Morphing optimizasyonu yoksa).

---

### 8. `std::future` ve `std::promise` Giriş
**[02:20:00 - 02:45:00]**

Condition variable bir "sinyal" mekanizmasıyken, `future`/`promise` ikilisi bir "veri taşıma kanalı"dır.

#### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir thread'in ürettiği bir değeri (veya oluşacak bir hatayı/exception'ı) başka bir thread'e güvenli bir şekilde aktarmak için tasarlanmıştır.

#### ⚙️ Teknik Detay ve Sentaks (Basit İletişim)
```cpp
#include <future>

void task(std::promise<int> prom) {
    // Karmaşık hesaplamalar...
    int result = 555;
    prom.set_value(result); // <-- Değeri kanala yükle (Sözünü tut)
}

int main() {
    std::promise<int> my_prom;
    std::future<int> my_fut = my_prom.get_future(); // <-- Kanalın çıkış ucunu al

    std::thread t(task, std::move(my_prom)); // <-- Promise taşınabilir (movable) ama kopyalanamaz!

    // ... main işlerine devam eder ...

    int val = my_fut.get(); // <-- BLOKE OLUR: Veri gelene kadar bekler
    std::cout << "Value: " << val << "\n";
    t.join();
}
```

#### 🖼️ Görselleştirme (The Pipe Analogy)
```text
[ Promise (Giriş) ] ====( Shared State / Kanal )====> [ Future (Çıkış) ]
        |                                                 |
  set_value()                                           get()
```

#### 🔗 Önceki Derslerle Bağlantı
*   `std::promise::set_exception`: Bir thread'de oluşan exception'ı yakalayıp (`std::current_exception`), diğer thread'e iletmek için kullanılır (Exception Handling dersine atıf).

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  `wait` fonksiyonunu döngüsüz (koşulsuz) çağırmak (Spurious wakeup riski).
2.  `notify_one` çağrısını mutlaka kilit altında yapmaya çalışmak (Verim kaybı).
3.  `std::future::get()` fonksiyonunu aynı nesne için birden fazla kez çağırmak (İlk çağrıdan sonra future nesnesi geçersiz/invalid hale gelir).



Özellikle şu 3 noktayı "tam rekonstrüksiyon" kuralına göre detaylandırmazsak not eksik kalır:

---

### 1. `cv.wait()` Fonksiyonunun "Atomik" Adımları (01:03:04)
Hoca burada "biz görmüyoruz ama arka planda şunlar oluyor" diyerek bir liste verdi. Bu, mülakatların en sevilen sorusudur.

**🔍 Arka Plan (Under the Hood):**
Bir thread `cv.wait(lock, pred)` satırına geldiğinde şu 4 adım **atomik** (bölünemez) olarak gerçekleşir:
1.  **Unlock & Sleep:** `unique_lock` nesnesinin içindeki mutex'i serbest bırakır (`unlock`) ve aynı anda thread'i uyku moduna (waiting state) sokar. (Bu ikisi arasında başka bir thread araya giremez, aksi halde "lost wakeup" olur).
2.  **Wait for Signal:** Thread, işletim sisteminden `notify_one` veya `notify_all` sinyali gelene kadar (veya sahte uyanma olana kadar) bloke kalır.
3.  **Relock:** Sinyal geldiğinde thread uyanır, ancak çalışmaya başlamadan önce **ilk iş olarak Mutex'i tekrar kilitlemeye çalışır.**
4.  **Check Predicate:** Mutex kilitlendikten sonra `lambda` (predikat) çağrılır. Eğer `false` dönerse, 1. adıma geri döner ve Mutex'i tekrar açıp uyur.

---

### 2. Çift Mutex'li Karmaşık Senaryo: "Progress Bar" (01:46:50)
Hoca burada 3 thread'li (Fetch, Progress, Process) bir senaryo çizdi. Bu örneğin farkı, **veriyi koruyan mutex ile durumu koruyan mutex'in ayrılmasıdır.**

**⚙️ Teknik Detay (Kod Rekonstrüksiyonu):**
```cpp
// Hoca: Veriyi koruyan ayrı, tamamlanma bilgisini koruyan ayrı mutex kullanıyoruz!
std::string str_data; 
std::mutex mtx_data; // <-- Veriye erişimi korur

bool is_completed = false;
std::mutex mtx_completed; // <-- Bitme durumunu korur
std::condition_variable cv_completed;

void process_data() {
    std::unique_lock<std::mutex> lk(mtx_completed);
    cv_completed.wait(lk, []{ return is_completed; }); // <-- Tüm veri bitene kadar uyu
    
    lk.unlock(); // <-- Hoca: Notification alındı, artık veriyi okuyabiliriz, kilidi bırakabiliriz
    
    std::lock_guard<std::mutex> data_lk(mtx_data); // <-- Şimdi asıl veriyi kilitle
    std::cout << "Processing data: " << str_data << "\n";
}
```

---

### 3. `std::future::get()` ve `std::shared_future` Nüansı (02:35:00)
Hoca'nın "invalid hale gelme" uyarısını teknik olarak açalım:

**🔍 Arka Plan (The "Move-Only" Future):**
*   `std::future` bir **unique** (tekil) sahiplik modelidir. İçindeki değer `get()` ile çekildiğinde, bu değer Shared State'ten **move** edilir (taşınır).
*   **Derleyici Gözü:** `get()` fonksiyonu r-value ref qualifier veya benzeri bir mekanizma ile nesnenin durumunu değiştirir. İkinci kez çağrıldığında `std::future_error` fırlatır.
*   **Çözüm (Shared Future):** Eğer Hoca'nın belirttiği gibi birden fazla thread aynı sonucu bekliyorsa:
    ```cpp
    std::promise<int> prom;
    std::shared_future<int> sf = prom.get_future().share(); // <-- shared_future'a dönüştür
    // Artık farklı thread'lerde sf.get() güvenle birden fazla kez çağrılabilir.
    ```

---

### 4. Gözden Kaçmaması Gereken Küçük Detaylar:
*   **Reference Wrapper (01:28:40):** Hoca `std::thread`'e parametre geçerken `std::ref()` kullanımını hatırlattı. Çünkü `std::thread` constructor'ı parametrelerini **decay** (tür daralması/kopyalama) eder. Referans geçmek için sarmalayıcı şarttır.
*   **Notify Altında Kilit (01:43:00):** Hoca, "Notify fonksiyonunu kilit (`lock`) altında çağırmak teknik olarak yanlış değildir ama uyanan thread'in tekrar bloke olmasına neden olup performans düşürebilir" dedi.
*   **Wait For / Wait Until (02:07:00):** Sadece `wait` değil, zamana bağlı beklemeler de var. Bunlar `std::cv_status::timeout` veya `bool` döner.

---

### Bölüm Özeti: Hoca'nın 3 Kritik Uyarısı
1.  **"Lost Wakeup" Tehlikesi:** Eğer thread uykuya dalmadan hemen önce `notify` gelirse ve biz `predicate` (koşul döngüsü) kullanmıyorsak, o thread sonsuza kadar uyuyabilir.
2.  **"Movable Promise":** Promise kopyalanamaz! `std::thread`'e mutlaka `std::move(prom)` ile taşınarak verilmelidir.
3.  **"Shared State" Ömrü:** Promise yok olsa bile içindeki "Shared State" (paylaşılan durum), ona bağlı son `future` nesnesi yok olana kadar bellekte yaşamaya devam eder.

---

📌 Şu an transkriptteki tüm teknik kırılımları, Hoca'nın kod üzerindeki satır arası yorumlarını ve karmaşık 3-thread örneğini tam olarak kapsadık.

Dersin son başlığına uygun isim: **_Condition_Variables_and_Shared_States.md**

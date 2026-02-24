Bu ders notları, Necati Ergin’in **19 Şubat 2025** tarihli, C++ Concurrency (Eş Zamanlı Programlama) kütüphanesinin temel taşları olan `std::promise`, `std::future` ve `std::async` konularının işlendiği 65. dersin titiz bir teknik dökümüdür.

---

# C++ Concurrency: std::promise, std::future ve Paylaşılan Durum (Shared State) Mekanizması

### [00:00 - 05:00] Giriş ve Temel Kavramlar

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
Thread'ler arası veri aktarımı yapmak için geleneksel yöntemlerde global değişkenler ve mutex'ler kullanılır. Ancak bu yöntem, bir thread'in ürettiği değeri (veya hatayı/exception) diğerine güvenli ve senkronize bir şekilde iletmek için çok fazla "boilerplate" (basmakalıp kod) gerektirir. `std::promise` ve `std::future`, bu iletişim kanalını soyutlayarak programcıyı düşük seviyeli senkronizasyon detaylarından kurtarır.

⚙️ **Teknik Detay ve Sentaks**
İletişim, arka planda bir **Shared State** (Paylaşılan Durum) nesnesi üzerinden yürütülür. 
- **std::promise:** Değeri "vaat eden" taraftır (Producer/Üretici). Shared State'e değer yazar.
- **std::future:** Gelecekteki değeri "bekleyen" taraftır (Consumer/Tüketici). Shared State'ten değeri okur.

🔍 **Arka Plan (Under the Hood)**
Hoca burada Shared State'i görselleştirdi:
```text
[ Thread A ] ----> (std::promise) ----\
                                       |---> [ SHARED STATE ] <---|
[ Thread B ] <---- (std::future)  ----/          (Value/Exception)
```
*Shared State*, biz görmesek de heap'te oluşturulan ve her iki nesne tarafından referans edilen bir yapıdır.

---

### [05:00 - 15:00] Hata İletimi: std::exception_ptr ve set_exception

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
Sadece hesaplanan değerleri değil, thread içinde oluşan çalışma zamanı hatalarını (exception) da çağıran thread'e iletmek gerekir. Eğer bir thread içinde exception fırlatılır ve yakalanmazsa, program `std::terminate` çağırarak sonlanır.

⚙️ **Teknik Detay ve Sentaks**
Hoca, `set_exception` fonksiyonunun doğrudan bir exception nesnesi değil, bir `std::exception_ptr` aldığını vurguladı.

```cpp
#include <future>
#include <iostream>
#include <exception>

int main() {
    std::promise<int> prom;
    std::future<int> ftr = prom.get_future();

    try {
        throw std::runtime_error("Hata oluştu!");
    } 
    catch (...) {
        // <-- Hoca buraya dikkat çekti: current_exception() exception_ptr döndürür.
        prom.set_exception(std::current_exception()); 
    }

    try {
        int val = ftr.get(); // <-- Kritik: Exception burada tekrar fırlatılır (rethrow).
    } 
    catch (const std::exception& e) {
        std::cout << "Yakaladim: " << e.what() << std::endl;
    }
}
```

🔗 **Önceki Derslerle Bağlantı**
Hoca, `std::exception_ptr` sınıfını ve `std::current_exception()` fonksiyonunu daha önceki hata yönetimi derslerinde işlediğimizi hatırlattı. Bu yapı, hatanın "polimorfik" özelliğini koruyarak thread sınırlarını aşmasını sağlar.

---

### [15:00 - 21:00] Fabrika Fonksiyonu: std::make_exception_ptr

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
Bir `exception_ptr` elde etmek için sürekli `try-catch` bloğu yazmak zahmetlidir. Standart kütüphane bu "boilerplate" kodu sarmalayan bir fabrika fonksiyonu sunar.

⚙️ **Teknik Detay ve Sentaks**
`std::make_exception_ptr` fonksiyonu, aldığı nesneyi kendi içinde `throw` edip `catch` ederek bize hazır bir `exception_ptr` döner.

```cpp
// Manuel try-catch yerine:
prom.set_exception(std::make_exception_ptr(std::runtime_error("Hayatimin hatasi"))); // <-- Hoca'nın esprisi
```

🔍 **Arka Plan (Under the Hood)**
Hoca, `std::make_exception_ptr`'ın muhtemel implementasyonunu tahtaya yazdı:
```cpp
template<typename E>
std::exception_ptr make_exception_ptr(E e) {
    try {
        throw e;
    } catch(...) {
        return std::current_exception();
    }
}
```

🚩 **Kritik Nokta / Mülakat Sorusu**
**Soru:** Bir thread içinde yakalanmayan bir exception neye sebep olur?
**Cevap:** Doğrudan `std::terminate` çağrılır ve program `abort` edilir. Bu yüzden exception, `std::promise` üzerinden güvenli bir şekilde aktarılmalıdır.

---

### [21:00 - 32:00] Thread Sınırlarında Exception Yönetimi

Hoca, düşük seviyeli `std::thread` nesnesi ile `std::promise` arasındaki farkı dramatik bir örnekle gösterdi.

⚙️ **Teknik Detay ve Sentaks (Hatalı Yaklaşım)**
```cpp
void foo(int x) {
    if (x % 2 == 0) throw std::runtime_error("Even number error");
}

int main() {
    // Yanlış: std::thread ile exception yakalanamaz!
    try {
        std::thread t(foo, 16); 
        t.join();
    } catch(...) {
        // Buraya asla girmez, program terminate olur!
    }
}
```

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1. Thread'in başlattığı fonksiyonun dışına sızan (propagate olan) exception'lar asla çağıran thread'in `try-catch` bloğuna düşmez.
2. Exception aktarımı için `std::exception_ptr` global veya paylaşılan bir alanda tutulmalıdır.
3. `std::promise` ve `std::future`, bu aktarım sürecini standart bir protokol haline getirir.

---

### [32:00 - 45:00] Uygulamalı Örnek: SomeSquare

Hoca, bir fonksiyonun sonucunun başka bir thread'de nasıl hesaplanıp `std::promise` ile geri gönderileceğini gösterdi.

⚙️ **Teknik Detay ve Sentaks**
```cpp
void some_square(std::promise<int> prom, int x, int y) { // <-- Dikkat: promise Move-Only'dir.
    std::this_thread::sleep_for(std::chrono::milliseconds(2000)); // Simülasyon
    prom.set_value(x * x + y * y); // Değeri Shared State'e yaz.
}

int main() {
    std::promise<int> prom;
    std::future<int> ftr = prom.get_future();

    // std::promise kopyalanamaz, sadece taşınabilir (Move-Only).
    std::thread t(some_square, std::move(prom), 10, 20); // <-- Kritik: std::move şart!
    
    std::cout << "Thread kosmaya basladi, deger bekleniyor...\n";
    std::cout << "Sonuc: " << ftr.get() << std::endl; // <-- ftr.get() bloklayıcıdır (blocking).
    t.join();
}
```

📊 **Standart Karşılaştırması**
| Özellik | C++11 | C++17 |
| :--- | :--- | :--- |
| `std::promise` | Mevcut | CTAD desteği ile daha kolay tanımlama |
| Kopyalama | Yasak | Yasak |
| Taşıma | `std::move` ile mümkün | `std::move` ile mümkün |

---

### [45:00 - 01:00:00] Future Nesnesinin Geçerliliği: valid() ve Tek Seferlik Kullanım

🚩 **Kritik Nokta / Mülakat Sorusu**
**Soru:** `future::get()` fonksiyonu kaç kez çağrılabilir?
**Cevap:** Sadece **BİR** kez. Çağrıldıktan sonra Shared State ile olan bağ kopar ve future nesnesi **Invalid** (Geçersiz) hale gelir.

⚙️ **Teknik Detay ve Sentaks**
```cpp
std::promise<int> prom;
auto ftr = prom.get_future();

std::cout << std::boolalpha << ftr.valid() << std::endl; // true
prom.set_value(42);
ftr.get(); // Değer çekildi.
std::cout << ftr.valid() << std::endl; // false

// ftr.get(); // <-- HATA: Undefined Behavior (UB) veya std::future_error (no_state).
```

🔍 **Arka Plan (Under the Hood)**
Hoca, `ftr.get()` çağrısının Shared State'i "tükettiğini" (consume) belirtti. Eğer aynı değere birden fazla kez veya birden fazla thread'den erişmek gerekiyorsa `std::shared_future` kullanılmalıdır.

🖼️ **Görselleştirme (ASCII Art)**
```text
Future Get() Akışı:
[Valid Future] --(get)--> [Value]
      |                       |
      \---->[Invalid Future (No State)]
```

Dünyanın en titiz C++ öğrencisi olarak Necati Hoca'nın dersindeki her virgülü, her "verimlilik cinayeti" uyarısını ve her teknik detayı yeniden inşa etmeye devam ediyorum. Şimdi dersin en kritik virajlarından biri olan `std::future_status` ve `std::async` politikalarına giriyoruz.

---

### [01:00:00 - 01:10:00] Exception PTR Mekanizması ve Soyutlama Seviyeleri

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
Hoca, exception'ları bir thread'den diğerine taşırken neden `std::make_exception_ptr` kullandığımızı derinleştirdi. Manuel olarak `try-catch` yazıp `current_exception()` çağırmak yerine, bu fabrika fonksiyonu kodu sarmalayarak hata payını azaltır.

⚙️ **Teknik Detay ve Sentaks**
Hoca, `std::make_exception_ptr`'ın aslında bir "şablon fonksiyon" (function template) olduğunu ve argüman çıkarımı (template argument deduction) yaptığını belirtti.

```cpp
// Hocanın tahtaya yazdığı içsel implementasyon mantığı:
template<typename E>
std::exception_ptr hoca_make_exception_ptr(E e) {
    try {
        throw e; // <-- Hoca: "Önce fırlatır..."
    } 
    catch(...) {
        return std::current_exception(); // <-- "...sonra yakalayıp ptr'sini döndürür."
    }
}

// Kullanım:
prom.set_exception(std::make_exception_ptr(std::out_of_range("Indis hatasi")));
```

🔍 **Arka Plan (Under the Hood)**
Düşük seviyeli (`std::thread`) senkronizasyon ile yüksek seviyeli (`std::promise/future`) yapıların farkı şudur:
1. **Düşük Seviye:** `Mutex` ve global değişkenlerle manuel koruma gerekir. Senkronizasyon yükü programcıdadır.
2. **Yüksek Seviye:** `Shared State` mekanizması tüm thread-safety detaylarını gizler.

🚩 **Kritik Nokta / Mülakat Sorusu**
**Soru:** Bir `std::promise` nesnesinde hem `set_value` hem de `set_exception` çağırabilir miyiz?
**Cevap:** **HAYIR.** Shared State sadece bir kez "satisfied" (tatmin edilmiş) hale gelebilir. İkinci bir set çağrısı `std::future_error` (promise_already_satisfied) fırlatır.

---

### [01:10:00 - 01:25:00] Future Nesnesinin Ömrü ve valid() Durumu

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
Bir `future` nesnesinin içindeki değer çekildikten sonra nesnenin durumu boşa çıkar. Programcının, değerin çekilip çekilmediğini sorgulaması için `valid()` fonksiyonu hayati önem taşır.

⚙️ **Teknik Detay ve Sentaks**
Hoca, `future` nesnesinin "Move-Only" (Sadece taşınabilir) doğasına dikkat çekti.

```cpp
std::promise<int> prom;
auto ftr = prom.get_future();

std::cout << (ftr.valid() ? "Valid" : "Invalid") << "\n"; // Valid

std::future<int> ftr2 = std::move(ftr); // <-- Taşıma yapıldı
// Derleyici şu sebeple kızmaz ama ftr artık boştur:
std::cout << (ftr.valid() ? "Valid" : "Invalid") << "\n"; // Invalid (Hoca: "Bomboş")
std::cout << (ftr2.valid() ? "Valid" : "Invalid") << "\n"; // Valid
```

🔍 **Arka Plan (Memory Layout)**
`std::future` nesnesi, arka plandaki `Shared State`'e bir "smart pointer" gibi tutunur. `std::move` yapıldığında bu sahiplik el değiştirir. `get()` çağrıldığında ise bu bağ tamamen kopar.

🚩 **Kritik Nokta / Mülakat Sorusu**
**Soru:** `valid() == false` olan bir `future` nesnesi üzerinde `get()` çağırırsak ne olur?
**Cevap:** Undefined Behavior (UB) ihtimali yüksektir ancak standart kütüphane genellikle `std::future_error` (no_state) fırlatılmasını teşvik eder (encourage).

---

### [01:25:00 - 01:40:00] Polling ve Future Status: wait_for / wait_until

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
`ftr.get()` çağırdığınızda thread sonsuza kadar bloke olabilir. Ancak gerçek dünya uygulamalarında "beklerken başka işler yapmak" veya "belirli bir süre sonra vazgeçmek" istersiniz. İşte burada **Polling** (sorgulama) devreye girer.

⚙️ **Teknik Detay ve Sentaks**
Hoca, `wait_for`'un geri dönüş değerinin bir `bool` değil, bir `std::future_status` (Enum Class) olduğunu vurguladı.

```cpp
// Hocanın Fibonacci Örneği (Verimlilik Cinayeti uyarısıyla!)
long long fib(int n) {
    if (n < 2) return n;
    return fib(n - 1) + fib(n - 2); // <-- Hoca: "Recursive fibonacci bir efficiency cinayetidir!"
}

int main() {
    auto ftr = std::async(std::launch::async, fib, 45); // Çok uzun sürecek

    std::future_status status;
    do {
        status = ftr.wait_for(std::chrono::milliseconds(250)); // 250ms'de bir sor
        if (status == std::future_status::timeout) {
            std::cout << "Halen hesapliyor, nokta koyuyorum...\n";
        }
    } while (status != std::future_status::ready); // <-- Kritik: ready olana kadar dön

    std::cout << "Sonuc hazir: " << ftr.get() << std::endl;
}
```

📊 **std::future_status Değerleri**
| Sabit (Constant) | Anlamı |
| :--- | :--- |
| `ready` | Değer hazır, `get()` hemen döner. |
| `timeout` | Belirtilen süre doldu, değer henüz hazır değil. |
| `deferred` | Fonksiyon henüz hiç başlamadı (Lazy evaluation). |

🖼️ **Görselleştirme (Polling Mantığı)**
```text
[Main Thread] --wait_for(250ms)--> [Status?] --(Timeout)--> [Başka iş yap]
      ^                                                            |
      |------------------------------------------------------------/
      |
[Main Thread] --wait_for(250ms)--> [Status?] --(Ready)--> [get() ile değeri al]
```

🚩 **Kritik Nokta / Mülakat Sorusu**
**Soru:** `wait_for` ile `get` arasındaki en büyük fark nedir?
**Cevap:** `get()` her zaman bloklayıcıdır ve Shared State'i tüketir (invalidate eder). `wait_for` ise sadece durumu sorgular, Shared State'e dokunmaz (valid bırakır) ve timeout süresi sonunda kontrolü geri verir.

---

**Bu bölümde Hoca şu 3 kritik noktaya dikkat çekti:**
1. **Move Semantics:** `std::promise` ve `std::future` kopyalanamaz, bu yüzden thread'lere geçerken mutlaka `std::move` kullanılmalıdır.
2. **Efficiency Murder:** Recursive algoritmaların (Fibonacci gibi) eş zamanlı programlamada simülasyon amaçlı "ağır iş" olarak kullanılabileceği ancak gerçekte kaçınılması gerektiği.
3. **Enum Class Kullanımı:** `std::future_status`'ın bir `enum class` olduğu, dolayısıyla `ready` değil `std::future_status::ready` şeklinde tam nitelenmiş (fully qualified) isimle kullanılması gerektiği.

Dünyanın en titiz C++ öğrencisi olarak, Necati Ergin hocanın dersinin son bölümünü (en kritik performans analizlerinin yapıldığı kısmı) santim santim dokümante etmeye devam ediyorum. Bu bölümde `std::shared_future` ve `std::async` mekanizmalarının "sahne arkasını" aydınlatıyoruz.

---

### [01:40:00 - 01:50:00] Paylaşımlı Gelecek: std::shared_future

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
`std::future` nesnesi "Move-Only" (Sadece taşınabilir) bir yapıdadır ve `get()` fonksiyonu çağrıldığında Shared State'i tüketir. Ancak bazen bir thread'in ürettiği sonucu **birden fazla thread'in** okuması gerekir. Standart `future` ile bunu yapamazsınız; işte bu noktada "Copyable" (Kopyalanabilir) olan `std::shared_future` devreye girer.

⚙️ **Teknik Detay ve Sentaks**
Hoca, `shared_future` elde etmenin iki yolunu gösterdi:
1. `std::future` nesnesinden `share()` üye fonksiyonunu çağırmak.
2. `std::shared_future` constructor'ına bir `future` nesnesini taşımak (`std::move`).

```cpp
std::promise<int> prom;
std::future<int> ftr = prom.get_future();

// 1. Yol: share() fonksiyonu ile (Hoca: "Bu daha yaygin")
std::shared_future<int> sf = ftr.share(); // <-- ftr artik invalid oldu!

// 2. Yol: Doğrudan constructor ile
// std::shared_future<int> sf(std::move(ftr));

// shared_future kopyalanabilir!
auto sf2 = sf; 
auto sf3 = sf2; // <-- Hoca: "Hepsi ayni Shared State'e bakiyor."

// Birden fazla kez get() çağrılabilir!
std::cout << sf.get() << std::endl;
std::cout << sf2.get() << std::endl; // <-- Hata vermez!
```

🔍 **Arka Plan (Under the Hood)**
`std::shared_future`, tıpkı `std::shared_ptr` gibi arka planda bir referans sayacı (reference counting) tutar. Shared State, tüm kopyalar yok olana kadar hayatta kalır.

🚩 **Kritik Nokta / Mülakat Sorusu**
**Soru:** `std::future::share()` çağrısından sonra orijinal `future` nesnesi ne durumda olur?
**Cevap:** Orijinal nesne boşalır (`valid() == false`). Shared State'in sahipliği oluşturulan `shared_future` nesnelerine geçer.

---

### [01:50:00 - 02:05:00] std::async ve Launch Politikaları

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
`std::promise` ve `std::thread` ile uğraşmak düşük seviyeli bir iştir. `std::async`, bir fonksiyonu asenkron çalıştırmanın "en yüksek seviyeli" yoludur. Hoca bunu "Yüksek seviye = Daha fazla detay gizleme" kuralıyla açıkladı.

⚙️ **Teknik Detay ve Sentaks**
Hoca, `std::launch::async` ve `std::launch::deferred` farkını sembollerle (+, *, ., !) görselleştiren muazzam bir örnek verdi.

```cpp
int task(int n, char c) {
    for (int i = 0; i < n; ++i) {
        std::cout.put(c);
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
    return n * static_cast<int>(c);
}

int main() {
    // std::launch::deferred: Hoca: "Vadelidir, get() çağrılana kadar başlamaz!"
    auto f1 = std::async(std::launch::deferred, task, 40, '+'); 
    
    // std::launch::async: Hoca: "Hemen yeni bir thread oluşturur ve başlar."
    auto f2 = std::async(std::launch::async, task, 40, '*');

    std::cout << "Main devam ediyor...\n";
    std::getchar(); // <-- Hoca: "Bu noktada f2 çalışıyor ama f1 halen uykuda!"

    f1.get(); // <-- f1 şimdi 'senkron' olarak bu thread'de çalışmaya başlar.
    f2.get();
}
```

📊 **Launch Policy Karşılaştırması**
| Politika | Çalışma Zamanı | Thread Durumu |
| :--- | :--- | :--- |
| `async` | Hemen başlar | Ayrı bir thread (Asenkron) |
| `deferred` | `get()` veya `wait()` çağrıldığında | Çağıran thread içinde (Senkron/Lazy) |
| `async \| deferred` | Derleyiciye bağlı (Default) | İmkana göre yeni thread veya ertelenmiş |

---

### [02:05:00 - 02:30:00] Verimlilik Analizi: Paralel Accumulate

Hoca, 50 Milyon elemanlı bir vektörün toplamını hesaplarken asenkron çalışmanın getirdiği performans farkını `std::chrono` ile ölçtü.

⚙️ **Teknik Detay ve Sentaks**
```cpp
typedef unsigned long long uint64; // Hocanın typedef'i

uint64 accumulate_s(uint64* first, uint64* last) {
    return std::accumulate(first, last, 0ULL);
}

// Hocanın Paralel Toplam Fonksiyonu
uint64 accumulate_parallel(std::vector<uint64>& vec) {
    auto data = vec.data();
    auto sz = vec.size();

    // İşi ikiye bölüyoruz (Hoca: "Divide and Conquer mantığı")
    auto f1 = std::async(std::launch::async, accumulate_s, data, data + sz / 2);
    auto f2 = std::async(std::launch::async, accumulate_s, data + sz / 2, data + sz);

    return f1.get() + f2.get(); // <-- İki thread'in sonucunu topla
}
```

🔍 **Arka Plan (Performance Trade-off)**
Hoca burada hayati bir uyarı yaptı: **Thread oluşturma maliyeti (Overhead).**
- Eğer veri çok büyükse (50M), paralel çalışma kazandırır.
- Eğer veri küçükse (50 bin), thread oluşturma süresi, toplama süresinden uzun sürer; bu bir **"Verimlilik Cinayeti"** olur.

---

### [02:30:00 - 02:43:50] Dersin Sonu ve Özet

🚩 **Kritik Nokta / Mülakat Sorusu**
**Soru:** `std::async` çağrısından dönen `std::future` nesnesini bir değişkene atamazsak ne olur?
**Cevap:** Geçici nesnenin (temporary object) destructor'ı bloke olur (blocking). Yani kod asenkron çalışmak yerine, o satırda işlemin bitmesini bekler. Hoca: "Bu çok yapılan bir hatadır, asenkron yapacağım derken senkron bir koda dönüşür."

🔗 **Kümülatif Bağlantılar**
Hoca, dersin sonunda gelecek haftanın planını yaptı:
- `std::packaged_task` (Task bazlı soyutlama)
- `std::atomic` (Lock-free programlamaya giriş)
- `std::condition_variable` hatırlatmaları.

🖼️ **Görselleştirme (Paralel İşleme)**
```text
Vektör (50M Eleman):
[ Chunk 1 (25M) ] ----> async(thread 1) \
                                          + ---> Result
[ Chunk 2 (25M) ] ----> async(thread 2) /
```

**Ders Sonu Teknik Notu:**
"Durumdan vazife çıkartmak" gerekirse; `std::async` her zaman asenkron çalışmak zorunda değildir. Default politikada derleyici, sistem kaynakları yetersizse thread oluşturmak yerine `deferred` (ertelenmiş) modda çalışmaya karar verebilir. Bu, sistemin çökmesini engelleyen bir emniyet sibobudur.

---


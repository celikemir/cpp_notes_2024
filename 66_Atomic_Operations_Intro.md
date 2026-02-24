### 66. Ders: Concurrency - `std::packaged_task` ve Atomik İşlemlere Giriş - Bölüm 1

**Tarih:** 26 Şubat 2025
**Eğitmen:** Necati Ergin

---

#### 1. Giriş ve `std::future` Mekanizmalarının Hatırlatılması [00:00:00 - 00:06:15]

Hoca derse Concurrency (eş zamanlılık) kütüphanesinin zenginliğinden ve her yeni standartla (C++17/20) daha da derinleştiğinden bahsederek başladı. Önceki derslerde işlenen `std::future` sınıfının önemine vurgu yapıldı.

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
Senkron veya asenkron bir kodun sonucunu (değer veya exception/müstesna) güvenli bir şekilde çağıran tarafa iletmek için bir iletişim kanalına ihtiyaç vardır. Bu kanal "Shared State" (paylaşılan durum) üzerinden yönetilir.

⚙️ **Teknik Detay ve Sentaks**
`std::future` nesnesi elde etmenin üç ana yolu olduğu belirtildi:
1.  **`std::promise` üzerinden:** Promise nesnesinin `get_future()` fonksiyonuyla. (Kanalı biz oluştururuz).
2.  **`std::async` üzerinden:** Fonksiyonun geri dönüş değeri olarak. (Thread yönetimini async yapar).
3.  **`std::packaged_task` üzerinden:** (Bugünün ilk konusu).

🔗 **Kümülatif Bağlantılar**
Hoca, `std::thread` ve `std::jthread` sarmalayıcılarını hatırlatarak, thread'i bizim oluşturduğumuz senaryolarla, `std::async`'in arka planda oluşturduğu senaryoları karşılaştırdı. Ayrıca C++17 ile gelen *Parallel Algorithms* (paralel algoritmalar) konusuna da (vakit kalırsa) değinileceğini belirtti.

---

#### 2. `std::packaged_task` Nedir? [00:06:16 - 00:11:36]

Hoca, ismin ("paketlenmiş görev") işlevini tam olarak yansıttığını belirtti. `std::packaged_task`, bir *Callable* (çağrılabilir varlık) nesnesini sarmalayan bir sınıftır.

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
`std::async`'den farkı, görevi (task) oluşturduğumuz an ile çalıştırdığımız anı birbirinden ayırabilmemizdir. Bir task'i önceden hazırlayıp, gerekli veriler oluştuğunda ister aynı thread'de senkron, ister farklı bir thread'e taşıyarak asenkron çalıştırabiliriz.

⚙️ **Teknik Detay ve Sentaks**
*   `std::packaged_task` bir Generic (jenerik) sınıftır.
*   Template (şablon) parametresi olarak bir **Function Type** (fonksiyon türü) bekler.
*   `get_future()` fonksiyonuna sahiptir.
*   *Move-only* (sadece taşınabilir) bir türdür.

🔍 **Arka Plan (Under the Hood)**
`std::packaged_task` nesnesinin kendisi de bir *Function Object* (fonksiyon nesnesi) olduğu için bir fonksiyon çağrı operatörüne (`operator()`) sahiptir. Bu operatör çağrıldığında sarmalanan *Callable* yürütülür ve sonuç *Shared State*'e yazılır.

---

#### 3. `std::packaged_task` İmzasının Anatomisi ve CTAD [00:11:37 - 00:16:50]

Hoca burada çok kritik bir ayrımın altını çizdi: **Function Type** (fonksiyon türü) vs. **Function Pointer Type** (fonksiyon göstericisi türü).

⚙️ **Teknik Detay ve Sentaks**
```cpp
int foo(int, int); // Fonksiyon bildirimi

// 1. Function Type (Fonksiyon Türü):
// int(int, int) <-- packaged_task bunu ister.

// 2. Function Pointer Type (Fonksiyon Göstericisi Türü):
// int(*)(int, int)
```

Sınıfın tanımlanması:
```cpp
#include <future>

// Hoca: "Using directive'ini şimdilik örnek kolaylığı için kullanıyorum"
using namespace std;

// Spesifik açılım (Specialization)
packaged_task<int(int, int)> task(foo); // <-- Fonksiyon türü geçildi

// C++17 ile CTAD (Class Template Argument Deduction) kullanımı:
packaged_task task2(foo); // <-- Derleyici foo'nun imzasına bakıp çıkarım yapar
```

🚩 **Kritik Nokta:**
9. satırdaki kod (task tanımı) `foo` fonksiyonunu çalıştırmaz! Sadece task nesnesi oluşturulur ve `foo` bu nesneye *assign* (atanır) edilir. Çalıştırma işlemi `task(...)` şeklinde operatör çağrısıyla yapılacaktır.

---

#### 4. Fonksiyon Çağrı Operatörü ve Geri Dönüş Değeri [00:16:51 - 00:20:15]

Hoca, `packaged_task::operator()` fonksiyonunun imzasını derleyici üzerinden göstererek çok önemli bir noktaya dikkat çekti.

⚙️ **Teknik Detay ve Sentaks**
```cpp
// Task nesnesini çağırmak:
task(3, 5); // <-- foo(3, 5) çağrılır.
```

🔍 **Arka Plan (Under the Hood)**
Hoca derleyicinin küçük penceresindeki imzaya dikkat çekti:
`void operator()(Args... args);`

**Soru:** Neden geri dönüş değeri `void`?
**Cevap:** Çünkü sarmalanan fonksiyonun geri dönüş değeri, doğrudan bu operatörden dönmez. Arka plandaki *Shared State*'e yazılır. Biz bu değeri `std::future::get()` fonksiyonuyla alırız.

```cpp
auto ft = task.get_future(); // ft: std::future<int>
task(3, 5); 
int result = ft.get(); // Değer buradan elde edilir
```

---

#### 5. Kod Örneği: `sum_square` Senaryosu [00:20:16 - 00:26:40]

Hoca, konuyu pekiştirmek için ilk canlı kod örneğini yazdı.

⚙️ **Teknik Detay ve Sentaks**
```cpp
#include <iostream>
#include <future>
#include <thread>
#include <chrono>

int sum_square(int x, int y) {
    std::cout << "sum_square is called\n";
    // İş yükü taklidi:
    std::this_thread::sleep_for(std::chrono::milliseconds(1800));
    return x * x + y * y;
}

int main() {
    // 1. Task nesnesini oluştur (CTAD kullanıldı)
    std::packaged_task task(sum_square); 

    // 2. Future nesnesini al
    std::future<int> f = task.get_future();

    std::cout << "Some code here...\n";

    // 3. Senkron çalıştırma (Aynı thread içinde)
    task(4, 13); 

    // 4. Değeri al
    std::cout << "Value is: " << f.get() << "\n";
}
```

🖼️ **Görselleştirme (Sequence Before)**
Aşağıdaki akış aynı thread içinde gerçekleşir:
`packaged_task tanımı` -> `get_future()` -> `task() çağrısı` -> `f.get()`

---

#### 6. Taşıma Semantiği ve Fonksiyonlara Aktarım [00:26:41 - 00:30:06]

Hoca, `std::packaged_task`'in *Non-copyable* (kopyalanamaz) olduğunu bir hata örneğiyle gösterdi.

⚙️ **Teknik Detay ve Sentaks**
```cpp
std::packaged_task<int(int, int)> task(sum_square);

// auto new_task = task; // <-- HATA: pr-value olmadığı sürece kopyalanamaz. 
                         // Copy constructor is deleted.

auto new_task = std::move(task); // <-- DOĞRU: Sadece taşıma (move) yapılabilir.
```

**Task'i Bir Fonksiyona Taşımak:**
```cpp
void foo(std::packaged_task<int(int, int)> param) {
    param(3, 5); // Task burada çalıştırılır
}

int main() {
    std::packaged_task<int(int, int)> task(sum_square);
    auto f = task.get_future();
    
    foo(std::move(task)); // Task sahipliği fonksiyona geçti
    
    std::cout << "Result: " << f.get() << "\n";
}
```

🚩 **Kritik Nokta:**
Task'i başka bir fonksiyona veya thread'e taşıdıktan sonra, orijinal `task` nesnesi *invalid* (geçersiz) hale gelir. Ancak `future` nesnesi halen geçerli olan *Shared State*'e bağlı kalır.

---

📌 **Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  `std::packaged_task` şablonuna fonksiyon pointer türü değil, **Function Type** (`int(int, int)` gibi) geçilmelidir.
2.  Task nesnesi oluşturulduğu anda fonksiyon çağrılmaz; çağrı için `operator()`'ün açıkça yürütülmesi gerekir.
3.  `std::packaged_task` nesneleri kopyalanamaz, sadece `std::move` ile taşınabilir.

### 66. Ders: Concurrency - `std::packaged_task` ve Atomik İşlemlere Giriş - Bölüm 2

**Eğitmen:** Necati Ergin
**Kapsam:** [00:30:06 - 01:05:40]

---

#### 7. Argümanları Paketlemek ve `std::bind` Kullanımı [00:30:06 - 00:41:40]

Hoca öğrencilere çok kritik bir soru sordu: "Şu ana kadarki örneklerde `packaged_task` sadece *callable* varlığı sarmaladı, ancak argümanları (10, 20 gibi) çağrı anında biz verdik. Peki, hem fonksiyonu hem de argümanlarını aynı nesne içinde sarmalamak istersek ne yaparız?"

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
Bazı durumlarda bir görevi (task) tüm verileriyle paketleyip bir kuyruğa (queue) atmak ve ileride hiçbir argüman geçmeden sadece "çalıştır" demek isteriz.

⚙️ **Teknik Detay ve Sentaks**
Çözüm olarak `<functional>` başlık dosyasındaki `std::bind` (veya modern C++'ta Lambda) önerildi.

```cpp
#include <functional>
#include <future>

int sum_square(int x, int y);

int main() {
    // 1. İçten dışa: Fonksiyonu ve argümanları bağla (bind)
    // Hoca: "Burada sum_square'i 10 ve 20 değerlerine bağlıyoruz."
    auto fn = std::bind(sum_square, 10, 20);

    // 2. Kritik Soru: fn'in (bind'ın döndürdüğü nesne) kaç parametresi var?
    // Hoca: "Tüm argümanlar bağlandığı için artık 0 parametreli (nullary) bir callable oldu."

    // 3. packaged_task tanımındaki DEĞİŞİKLİK:
    std::packaged_task<int()> pt(fn); // <-- Parametre listesi boş: int() 

    auto ft = pt.get_future();
    
    // 4. Çalıştırma: Artık argüman geçmiyoruz!
    pt(); // <-- İçerideki sum_square(10, 20) çalışır.
    
    std::cout << "Result: " << ft.get() << "\n";
}
```

🔍 **Arka Plan (Under the Hood)**
`std::bind` kullanıldığında, `packaged_task`'in beklediği imza değişir. Eğer `sum_square` `int(int, int)` ise ve biz tüm `int`'leri bind edersek, yeni imza `int()` olur.

🚩 **Mülakat Sorusu / Kritik Nokta:**
`std::bind` içinde `std::placeholders::_1` gibi yer tutucular kullanılmazsa, dönen nesne parametresiz olur. Eğer bir yer tutucu kullanılırsa, `packaged_task` şablonuna o parametrenin türü yazılmalıdır.

---

#### 8. Asenkron Çalıştırma: `std::thread` ile Entegrasyon [00:41:41 - 00:44:00]

Hoca, `std::packaged_task`'in asıl gücünün farklı bir thread'e taşınmasında olduğunu gösterdi.

⚙️ **Teknik Detay ve Sentaks**
```cpp
std::packaged_task task(sum_square);
auto f = task.get_future();

// Task'i başka thread'e taşıyarak başlatıyoruz
std::thread t(std::move(task), 10, 20); // <-- packaged_task move-only olduğu için move şart!

t.join(); // Thread'i bekle
std::cout << "Result from thread: " << f.get() << "\n";
```

🚩 **Kritik Nokta:**
Thread'e `task` nesnesini geçerken mutlaka `std::move` kullanılmalıdır. Aksi halde "Copy constructor is deleted" hatası alınır.

---

#### 9. packaged_task İçinde Exception (Müstesna) Yönetimi [00:44:01 - 00:49:10]

Hoca, sarmalanan fonksiyon hata fırlatırsa ne olacağını bir örnekle açıkladı.

⚙️ **Teknik Detay ve Sentaks**
```cpp
int some_function(int x, int y) {
    if (x == 0 || y == 0)
        throw std::invalid_argument("Zero argument error"); // <-- Hata fırlatılıyor
    return x * y;
}

int main() {
    std::packaged_task task(some_function);
    auto f = task.get_future();

    task(0, 10); // Hata fırlatacak çağrı

    try {
        auto val = f.get(); // <-- Exception burada (get çağrısında) yeniden fırlatılır!
    } catch (const std::exception& ex) {
        std::cout << "Exception caught: " << ex.what() << "\n";
    }
}
```

🔗 **Kümülatif Bağlantılar**
Hoca, bu mekanizmanın `std::promise::set_exception` ile birebir aynı mantıkta çalıştığını; `packaged_task`'in arka planda oluşan hatayı yakalayıp *Shared State*'e otomatik olarak yazdığını belirtti.

---

#### 10. Görev Yönetimi ve `reset()` Fonksiyonu [00:49:11 - 01:05:40]

Hoca, büyük bir işlem olan Özyinelemeli (Recursive) Fibonacci örneğini vererek, `packaged_task`'in nasıl yeniden kullanılabileceğini (`reset`) ve kuyruklarda (containers) nasıl saklanabileceğini gösterdi.

⚙️ **Teknik Detay ve Sentaks: `reset()` Kullanımı**
Normalde bir `future` nesnesi bir kez `get()` edilebilir. Ancak aynı `packaged_task` nesnesini tekrar kullanmak isterseniz `reset()` demeniz gerekir.

```cpp
std::packaged_task<int(int)> f_task(fibonacci);
auto ft1 = f_task.get_future();
f_task(10); // İlk çalıştırma
ft1.get();

f_task.reset(); // <-- KRİTİK: Shared state'i temizler, nesneyi yeniden kullanılabilir yapar.
auto ft2 = f_task.get_future(); // Yeni bir future almalısınız
f_task(20); // İkinci çalıştırma
```

⚙️ **Task'leri Konteynerda Tutmak (Task Queue)**
```cpp
#include <deque>

// Bir görev kuyruğu oluşturma
std::deque<std::packaged_task<int(int, int)>> task_queue;

// Task'i kuyruğa ekle
task_queue.push_back(std::packaged_task(sum_square));

// Kuyruktan çek ve çalıştır
auto my_task = std::move(task_queue.front());
task_queue.pop_front();

auto f = my_task.get_future();
std::thread(std::move(my_task), 5, 5).detach(); // Arka planda çalıştır
```

🔍 **`std::async` ile packaged_task Farkı (Mehmet Ali'nin Sorusu Üzerine)**
Hoca farkı "kontrol seviyesi" olarak tanımladı:
*   **`std::async`:** Çağırdığınız anda thread yönetimine başlar. "Her şeyi sen hallet, bana sonucu getir" yaklaşımıdır.
*   **`std::packaged_task`:** Görevi oluşturur ama **yürütmez**. Yürütme sorumluluğunu programcıya bırakır. Bu sayede bir "Thread Pool" (thread havuzu) mimarisi kurmak için en ideal araçtır.

---

📌 **Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  `std::bind` ile tüm argümanlar bağlandığında, `packaged_task` şablon parametresinin `R(Args...)` yerine `R()` (boş parametre) olarak güncellenmesi gerektiği unutulmamalıdır.
2.  `reset()` fonksiyonu çağrılmadan aynı task nesnesi üzerinden ikinci kez `get_future()` çağrısı yapılamaz.
3.  `packaged_task` çalıştırılmadan önce `future` alınabilir, ancak `get()` çağrısı görev tamamlanana kadar çağıran thread'i bloke eder.

### 66. Ders: Concurrency - `std::packaged_task` ve Atomik İşlemlere Giriş - Bölüm 3

**Eğitmen:** Necati Ergin
**Kapsam:** [01:05:40 - 01:43:20]

---

#### 11. `std::packaged_task` Geçerlilik Kontrolü: `valid()` [01:05:41 - 01:11:30]

Hoca, bir `packaged_task` nesnesinin her zaman bir göreve sahip olmayabileceğini (boş olabileceğini) belirtti.

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
`packaged_task` nesneleri *default construct* edilebilir veya başka bir nesneye taşınabilir (moved). Boş bir task nesnesini çağırmak çalışma zamanında hataya yol açar.

⚙️ **Teknik Detay ve Sentaks**
```cpp
std::packaged_task<void()> task1; // Default construct (boş)
std::packaged_task<void()> task2([](){ std::cout << "Test"; });

if (!task1.valid()) {
    std::cout << "task1 is empty\n"; // <-- Buraya girer
}

auto task3 = std::move(task2);
if (!task2.valid()) {
    std::cout << "task2 is now empty after move\n"; // <-- Taşıma sonrası boşalır
}
```

🚩 **Kritik Nokta / Derleyici Gözü:**
Eğer `valid()` değeri `false` olan (boş) bir task nesnesinin `operator()` veya `get_future()` fonksiyonunu çağırırsanız, derleyici hata vermez ancak çalışma zamanında **`std::future_error`** fırlatılır.

---

#### 12. Atomik İşlemlere Giriş (Atomic Operations) [01:11:31 - 01:21:00]

Hoca bu bölümü "Concurrency konusunun en zor ve teknik derinliği en yüksek kısmı" olarak nitelendirdi.

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
Normal değişkenler üzerinde yapılan işlemler (örneğin `++x`) işlemci seviyesinde tek bir hamle değildir. Bir **Read-Modify-Write** (Oku-Değiştir-Yaz) döngüsüdür. Araya başka bir thread girdiğinde veri tutarsızlığı (Data Race) oluşur. Mutex kullanmak güvenlidir ama maliyetlidir. Atomik işlemler, donanım desteğiyle bu maliyeti düşürür.

⚙️ **Teknik Detay ve Kavramlar**
*   **Uninterruptible (Kesintiye Uğramaz):** Bir işlem başlar ve bitene kadar başka hiçbir thread araya giremez.
*   **Atomicity (Bölünemezlik):** İşlem ya tamamen gerçekleşir ya da hiç gerçekleşmez.
*   **Torn Read/Write (Yırtık Okuma/Yazma):** Bir değişkenin yarısının değişip diğer yarısının eski kaldığı bir anın başka bir thread tarafından asla görülmemesi garantisidir.

🔍 **Arka Plan (Under the Hood)**
Atomik işlemler ya donanımın sunduğu özel CPU talimatlarıyla (instruction) ya da derleyicinin arka planda yönettiği kilit mekanizmalarıyla sağlanır. C++ bu konuda bir *Abstraction* (soyutlama) sunar.

---

#### 13. Bellek Modeli Tanımları: `Sequence Before` [01:21:01 - 01:29:40]

Necati Hoca, atomik işlemleri anlamak için C++ standartlarının kullandığı formal tanımların bilinmesi gerektiğini vurguladı.

⚙️ **`Sequence Before` (Sıralı Öncelik)**
Bu kavram **sadece tek bir thread** içindeki işlemlerin sırasını tanımlar.
*   Eğer A işlemi, B işleminden `sequence before` ise; B çalışırken A'nın tüm sonuçları B tarafından görülür durumdadır.

**Örnekler:**
*   Noktalı virgül (semicolon): `x = 5; y = x;` (x'in 5 olduğu garantidir).
*   Virgül operatörü: `f(), g();` (f, g'den öncedir).
*   Mantıksal operatörler: `if (a && b)` (a, b'den öncedir).

---

#### 14. Thread'ler Arası İlişki: `Happens Before` ve `Synchronized With` [01:29:41 - 01:34:40]

Hoca, "Aynı thread içindeki sırayı biliyoruz, peki farklı threadler arasındakini nasıl tanımlarız?" sorusunu sordu.

⚙️ **`Happens Before` (Önce Gerçekleşme)**
Bir işlemin sonucunun, başka bir işlem tarafından **görünür (visible)** olmasıdır. Bu kavram `sequence before`'u da kapsar ancak threadler arası ilişkiyi de içine alır.

⚙️ **`Synchronized With` (İle Senkronize Olma)**
Farklı threadlerdeki iki işlemin birbiriyle el sıkışmasıdır. Eğer Thread 1'deki A işlemi, Thread 2'deki B işlemi ile `synchronized with` ise; A, B'den `happens before`'dur. Yani A'nın yaptığı her şey B tarafından görülür.

---

#### 15. Senkronizasyon Noktaları (Formal Garantiler) [01:34:41 - 01:43:20]

Hoca, şimdiye kadar "zaten öyle olmalı" diye kullandığımız ama arkasında bu formal tanımların yattığı durumları listeledi:

1.  **Thread Başlatma:** Bir thread'i başlatan kod, thread'in içindeki ilk koddan `happens before`'dur. (Ebeveynin hazırladığı veriler çocuk thread tarafından görülür).
2.  **Thread Join:** Bir thread'in işini bitirmesi, `join()` fonksiyonunun geri dönmesinden `happens before`'dur.
3.  **Mutex:** Bir mutex'in `unlock()` edilmesi, aynı mutex'in başka bir thread tarafından `lock()` edilmesinden `happens before`'dur.
4.  **Promise/Future:** `set_value()` çağrısı, `get()` çağrısının geri dönmesinden `happens before`'dur.

🚩 **Mülakat Sorusu / Kritik Nokta:**
**Soru:** "Bir işlemin zamansal olarak (clock time) diğerinden 20 milisaniye önce yapılması, diğer thread'in bu sonucu göreceğini garanti eder mi?"
**Hoca'nın Cevabı:** "HAYIR! Standart bellek modelinde zamansal önceliğin (temporal order) bir önemi yoktur. Görünürlük garantisi için yukarıdaki formal senkronizasyon noktalarından birinin tetiklenmesi gerekir."

---

📌 **Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  `std::packaged_task`'in `valid()` kontrolü yapılmadan çağrılması (`std::future_error`).
2.  `volatile` anahtar sözcüğünün C++'ta atomik veya thread-safe sanılması (Hoca: "Büyük bir yanılgı, Java ile karıştırmayın!").
3.  Farklı thread'ler arasında zamansal olarak "önce" yapılan bir işlemin, senkronizasyon noktası yoksa "görünür" olmamasından kaynaklanan hatalar.

### 66. Ders: Concurrency - `std::packaged_task` ve Atomik İşlemlere Giriş - Bölüm 4

**Eğitmen:** Necati Ergin
**Kapsam:** [01:43:20 - 02:42:15] (Ders Sonu)

---

#### 16. Atomik Store ve Load Senkronizasyonu [01:43:21 - 01:51:30]

Hoca, atomik değişkenlerin sadece bölünemezlik değil, aynı zamanda bellek senkronizasyonu sağladığını formalize etti.

⚙️ **Teknik Detay ve Sentaks**
*   **Store (Yükleme/Yazma):** Atomik bir değişkene değer atanması.
*   **Load (Çekme/Okuma):** Atomik değişkenin o anki değerinin okunması.

**Senaryo:**
Thread A, bazı normal (atomik olmayan) değişkenleri değiştirir, ardından bir `std::atomic<int> flag` değişkenine `store` işlemi yapar. Thread B ise aynı `flag` üzerinden `load` işlemi yapar.
*   **Garanti:** Thread B `load` işlemini tamamladığı anda, Thread A'nın `store` işleminden **önce** yaptığı tüm bellek işlemleri Thread B için görünür (visible) hale gelir.

---

#### 17. Büyük Yanılgı: `volatile` Anahtar Sözcüğü [01:51:31 - 02:03:30]

Hoca, birçok programcının Java veya C# dillerinden gelen alışkanlıkla `volatile` sözcüğünü C++'ta bir concurrency (eş zamanlılık) aracı sanmasına sert bir uyarı yaptı.

🧠 **Neden İhtiyaç Duyuldu? (Rationale)**
C++'ta `volatile`, donanım sürücüleri veya sinyal yakalayıcılar (signal handlers) gibi "program dışı" kaynakların bir değişkeni her an değiştirebileceğini derleyiciye bildirmek için vardır.

⚙️ **Teknik Detay (Data Race Örneği)**
```cpp
volatile int x = 0; // <-- Hoca: "Bunun atomik olduğunu sanıyorsanız yanılıyorsunuz!"

void func() {
    for (int i = 0; i < 100000; ++i) {
        ++x; // <-- HATA: Bu işlem hala Read-Modify-Write döngüsüdür ve Atomik DEĞİLDİR.
    }
}
```
Hoca bu kodu çalıştırdığında sonucun 1 milyon (10 thread için) değil, çok daha düşük (örn: 167.350) çıktığını göstererek `volatile`'ın bir işe yaramadığını kanıtladı.

🚩 **Mülakat Sorusu / Kritik Nokta:**
**Soru:** C++'ta `volatile` atomiklik veya thread-safety sağlar mı?
**Cevap:** Hayır. `volatile` sadece derleyici optimizasyonlarını (register'da tutma vb.) engeller. Atomiklik ve threadler arası senkronizasyon için sadece `std::atomic` kullanılmalıdır.

---

#### 18. En Temel Atomik Tür: `std::atomic_flag` [02:03:31 - 02:11:30]

Hoca, standart kütüphanedeki en "hafif" ve donanım desteği en yüksek türü tanıttı.

⚙️ **Teknik Detay ve Sentaks**
*   `std::atomic_flag`, kilit içermeme (**Lock-free**) garantisi olan **tek** türdür.
*   Sadece iki durumu vardır: Set (ayarlanmış) veya Clear (temizlenmiş).

**C++20 Öncesi ve Sonrası Farkı:**
```cpp
// C++17 ve öncesi:
std::atomic_flag flag = ATOMIC_FLAG_INIT; // Makro ile başlatmak zorunluydu.

// C++20:
std::atomic_flag flag; // Default construct edildiğinde 'clear' (false) olma garantisi geldi.
```

**Temel Fonksiyonlar:**
*   `test_and_set()`: Değeri `true` yapar ve **eski** değerini döndürür. (Atomik bir işlemdir).
*   `clear()`: Değeri `false` yapar.
*   `test()` (C++20): Değeri değiştirmeden okur.

---

#### 19. Canlı Kod: `Spinlock` (Dönen Kilit) İmplementasyonu [02:11:31 - 02:23:30]

Hoca, mülakatların vazgeçilmez sorusu olan "Kendi mutex'inizi nasıl yazarsınız?" sorusuna `std::atomic_flag` ile yanıt verdi.

⚙️ **Teknik Detay ve Sentaks**
```cpp
#include <atomic>

class SpinlockMutex {
    std::atomic_flag m_flag; // Default: false (C++20)
public:
    SpinlockMutex() {
        m_flag.clear(); // Hoca: "Garantiye alalım."
    }

    void lock() {
        // Hoca: "Eski değer true olduğu sürece dön (spin at)."
        // Ne zaman ki birisi clear() der ve biz false yakalarız, 
        // test_and_set onu true yapar ve döngüden çıkarız.
        while (m_flag.test_and_set()) {
            // Null statement: Meşgul bekleme (Busy wait)
        }
    }

    void unlock() {
        m_flag.clear(); // Kilidi bırak
    }
};
```

🔍 **Arka Plan (Under the Hood)**
Normal `std::mutex`, kilit başkasındayken thread'i işletim sistemi seviyesinde uyutur (context switch). `Spinlock` ise thread'i uyutmaz, CPU'da sürekli "Boşaldı mı?" diye sorgulatır. Çok kısa sürecek işlemler için `Spinlock` çok daha hızlıdır.

---

#### 20. `std::atomic<T>` Üye Fonksiyonları ve KAS (Compare-And-Swap) [02:23:31 - 02:42:15]

Hoca, genel `std::atomic` şablonunun sunduğu fonksiyonları özetledi.

⚙️ **Teknik Detay ve Sentaks**
1.  **`store(val)`**: Değer yazar (Atomik `=`).
2.  **`load()`**: Değer okur.
3.  **`exchange(val)`**: Yeni değer yazar, eski değeri döndürür. (Atomik *Read-Modify-Write*).
4.  **`compare_exchange_strong(expected, desired)`**:
    *   Bu fonksiyon "Atomik işlemlerin şahıdır" (KAS - Compare And Swap).
    *   Eğer atomik değişkenin değeri `expected`'a eşitse, değeri `desired` yapar ve `true` döner.
    *   Eşit değilse, `expected` değişkeninin içini atomik değişkenin o anki gerçek değeriyle günceller ve `false` döner.

🔍 **Hoca'nın İdiomu: "Komik/Zor Anlaşılan Fonksiyon"**
Hoca, `compare_exchange` fonksiyonunun neden referans (`int& expected`) aldığını açıkladı: "Eğer beklediğimiz değer orada yoksa, bari olan değeri bana getir ki bir sonraki döngüde neyi bekleyeceğimi bileyim."

---

#### 21. Kapanış ve Cuma Günü Programı

Hoca, `compare_exchange` konusunun çok derin olduğunu, Cuma günü bu fonksiyonun `weak` versiyonunu, döngüsel kullanımlarını ve en önemlisi **Memory Ordering** (Bellek Sıralama) konusunu işleyerek kursu bitireceklerini duyurdu.

🚩 **Kritik Nokta:**
Atomik nesneler **Non-copyable** ve **Non-movable**'dır. Bir atomik nesneyi başka bir atomik nesneye `=` ile atayamazsınız (çünkü atama operatörü bir `T` bekler, `std::atomic<T>` değil).

---



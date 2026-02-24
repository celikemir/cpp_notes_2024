# C++ Modern Concurrency Notları - Ders 61

**Eğitmen:** Necati Ergin  
**Konu:** Concurrency (Eşzamanlılık) Kütüphanesi, `std::thread` Detayları, `thread_local` ve `std::call_once`  
**Tarih:** 5 Şubat 2025

---

## 1. Modern C++ Concurrency Mimarisinin Temelleri [00:00:00 - 00:10:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C++11 öncesinde C++ dilinin standart bir bellek modeli (memory model) ve eşzamanlılık desteği yoktu. Programcılar POSIX Threads (pthreads) veya Windows API gibi işletim sistemine (OS) bağımlı kütüphaneler kullanmak zorundaydı. Bu durum kodun taşınabilirliğini (portability) bozuyor ve hata riskini artırıyordu. Modern C++ ile gelen `std::thread`, işletim sistemi seviyesindeki karmaşıklığı sarmalayan (wrapper) standart bir arayüz sunar.

### ⚙️ Teknik Detay ve Sentaks
Hoca, bir `std::thread` nesnesinin hayatı ile temsil ettiği fiziksel thread arasındaki farka dikkat çekti.

```cpp
#include <iostream>
#include <thread>

void foo() {
    std::cout << "Is parcacigi calisiyor..." << std::endl;
}

int main() {
    std::thread tx(foo); // <-- Callable (Workload) atandigi an thread baslar.
    
    if (tx.joinable()) { // <-- Hoca vurguladi: Sorgulamadan join/detach cagirmak UB'dir.
        std::cout << "tx join edilebilir durumda (yes)" << std::endl;
    }

    std::thread ty = std::move(tx); // <-- KRITIK: std::thread nesnesi Non-copyable but Movable.
    
    // tx artik bos, ty ise canli bir thread'i sahiplenmis durumda.
    std::cout << "tx joinable: " << (tx.joinable() ? "yes" : "no") << std::endl; // no
    std::cout << "ty joinable: " << (ty.joinable() ? "yes" : "no") << std::endl; // yes

    ty.join(); // <-- Thread bitene kadar main bloke olur.
    return 0;
}
```

### 🔍 Arka Plan (Under the Hood)
`std::thread` bir "Move-only" (yalnızca taşınabilir) sınıftır. Kopyalama yapılamaz çünkü bir fiziksel thread'in (OS resource) mülkiyeti iki farklı nesne tarafından paylaşılamaz. Bir nesne taşındığında, içindeki `native_handle` (OS'e özgü thread tutamağı) yeni nesneye devredilir.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `std::thread` nesnesinin destructor'ı çağrıldığında nesne hala `joinable` durumdaysa ne olur?  
**Cevap:** Program `std::terminate` çağrısıyla sonlandırılır. Hoca bunun çok yaygın bir hata olduğunu; mutlaka `join()` veya `detach()` çağrılarak bu durumun çözülmesi gerektiğini belirtti.

### 🔗 Önceki Derslerle Bağlantı
Hoca, nesne ömürleri ve RAII (Resource Acquisition Is Initialization) prensiplerine atıfta bulunarak, thread nesnelerinin scope sonuna gelmeden mülkiyetlerinin yönetilmesi gerektiğini hatırlattı.

---

## 2. Donanımsal Kapasite ve Thread ID Kavramı [00:10:00 - 00:20:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Sistemde kaç tane fiziksel çekirdek (core) olduğunu bilmeden aşırı thread oluşturmak, "Context Switching" (bağlam değişimi) maliyetini artırarak performansı düşürebilir. Donanımın desteklediği paralel thread sayısını bilmek optimizasyon için elzemdir.

### ⚙️ Teknik Detay ve Sentaks
`std::thread::hardware_concurrency()` statik fonksiyonu ile donanım kapasitesi sorgulanır.

```cpp
#include <iostream>
#include <thread>

int main() {
    // Hoca: "Buna tam garanti olmasa da guvenebilirsiniz" dedi.
    unsigned int n = std::thread::hardware_concurrency(); 
    std::cout << "Hardware supported threads: " << n << std::endl;

    std::thread tx([](){});
    std::thread::id tid = tx.get_id(); // <-- Hoca buraya dikkat cekti: Geri donus tipi nested type'dir.
    
    std::cout << "Thread ID: " << tid << std::endl; 
    // Thread ID'ler tamsayi turu olmak zorunda degil, ostream inserter (<<) overloaded'dir.
    
    tx.join();
}
```

### 🖼️ Görselleştirme (ASCII Art)
Thread ID'lerin sistemdeki yönetimi:
```text
[Main Thread (ID: 100)]
      |
      +---- [Child Thread 1 (ID: 101)] ---> (get_id() -> 101)
      |
      +---- [Child Thread 2 (ID: 102)] ---> (get_id() -> 102)
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `std::thread::id` nesnesi bir tamsayı mıdır?  
**Cevap:** Hayır, o bir nesnedir. Ancak karşılaştırma operatörlerini (`==`, `<`) destekler, böylece `std::set` veya `std::map` içinde anahtar olarak kullanılabilir. Default construct edilen bir `thread::id` nesnesi, "hiçbir thread'e ait olmayan" özel bir değeri temsil eder.

---

## 3. `std::this_thread` Namespace ve ID Karşılaştırmaları [00:20:00 - 00:25:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir fonksiyonun içinde, o fonksiyonu hangi thread'in çalıştırdığını bilmek gerekebilir (örneğin; loglama veya sadece ana thread'in yapabileceği işlemler için). `this_thread` namespace'i, çalışmakta olan "mevcut" iş parçacığına erişim sağlar.

### ⚙️ Teknik Detay ve Sentaks
Hoca, `get_id()` fonksiyonunun iki farklı kullanımını karşılaştırdı:
1. `tx.get_id()`: `tx` nesnesinin temsil ettiği thread'in ID'sini verir.
2. `std::this_thread::get_id()`: O kodu "o an" koşturan thread'in ID'sini verir.

```cpp
#include <iostream>
#include <thread>

void foo() {
    // Hoca: "Fonksiyonu calistiran thread kimse onun ID'sini basar"
    std::cout << "Inside foo thread ID: " << std::this_thread::get_id() << std::endl;
}

int main() {
    std::thread t1(foo);
    std::cout << "In main, t1's ID: " << t1.get_id() << std::endl; // t1 ve foo ayni ID'yi basar.

    if (t1.get_id() == std::this_thread::get_id()) {
        std::cout << "Bu imkansiz bir durumdur (Main != Child)" << std::endl;
    }

    t1.join();
    return 0;
}
```

### 🔍 Arka Plan (Under the Hood)
İşletim sistemi, her thread'e unik bir tamsayı atar. C++ standart kütüphanesi bu tamsayıyı `std::thread::id` sınıfı içinde saklar. `this_thread::get_id()` çağrıldığında, kütüphane OS'in "Thread Local Storage" (TLS) veya benzeri bir mekanizmasından o anki aktif thread'in bilgisini çeker.

---

### 🕒 [00:00:00 - 00:25:00] Arası Kritik Özet
Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:
1.  **Move Semantiği İhlali:** `std::thread` kopyalamaya çalışıldığında derleme hatası verir, sadece taşınabilir.
2.  **Joinable Kontrolü:** Nesne silinmeden önce ya `join()` ya `detach()` edilmelidir, aksi halde program `std::terminate` ile çöker.
3.  **Default ID Karşılaştırması:** Bir thread nesnesi içi boş (default construct) ise `get_id()` çağrısı özel bir "not-a-thread" ID'si döndürür; bu değer diğer geçerli ID'lerle karşılaştırılabilir.

## 4. Thread ID Karşılaştırmaları ve Uniklik Garantisi [00:25:00 - 00:35:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Sistemde çalışan thread'leri birbirinden ayırmak, log mekanizmalarında hangi işin hangi thread tarafından yapıldığını izlemek veya belirli bir kod parçasının sadece "Main Thread" tarafından koşturulduğundan emin olmak için ID karşılaştırmasına ihtiyaç duyulur.

### ⚙️ Teknik Detay ve Sentaks
Hoca, `std::thread::id`'nin karşılaştırılabilir (comparable) olduğunu ve boş bir thread nesnesinin ID'sinin neye eşit olduğunu gösterdi.

```cpp
#include <iostream>
#include <thread>
#include <vector>

void foo() {
    // Hoca: "this_thread::get_id() o anki calisan thread'in kimligini verir"
    std::cout << "Thread " << std::this_thread::get_id() << " is running." << std::endl;
}

int main() {
    std::thread tx; // Default construct edildi (Bos)
    
    // KRITIK: Bos bir thread'in ID'si, default construct edilmis bir thread::id'ye esittir.
    if (tx.get_id() == std::thread::id()) { 
        std::cout << "tx is not representing a thread (yes)" << std::endl;
    }

    std::thread t1(foo);
    std::thread t2(foo);

    // Hoca vurguladi: ID'ler calisma zamaninda (runtime) unik olmak zorundadir.
    if (t1.get_id() != t2.get_id()) {
        std::cout << "IDs are unique." << std::endl;
    }

    t1.join();
    t2.join();
    return 0;
}
```

### 🔍 Arka Plan (Under the Hood)
*   **ID Unikliği:** Aynı anda çalışan iki thread asla aynı ID'ye sahip olamaz. Ancak, bir thread `join` edildikten (hayatı bittikten) sonra, işletim sistemi aynı ID'yi yeni oluşturulan başka bir thread'e verebilir. Yani ID garantisi "lifetime" (ömür) ile sınırlıdır.
*   **Lightweight Process (LWP):** Hoca, işletim sistemi derslerine atıfta bulunarak thread'lerin bazı sistemlerde "hafif siklet proses" olarak adlandırıldığını belirtti. Proses oluşturmak maliyetliyken, thread oluşturmak çok daha ucuzdur.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Çalışmakta olan bir fonksiyonun ana thread (main thread) tarafından mı yoksa bir child thread tarafından mı çağrıldığını standart bir yolla nasıl anlarız?
**Cevap:** Main thread başladığında onun ID'sini global bir `std::thread::id` değişkeninde saklarız. Fonksiyon içinde `std::this_thread::get_id()` ile bu global değişkeni karşılaştırırız.

```cpp
std::thread::id main_thread_id; // Global state

void work() {
    if (std::this_thread::get_id() == main_thread_id) {
        std::cout << "Called from Main Thread" << std::endl;
    } else {
        std::cout << "Called from Child Thread" << std::endl;
    }
}

int main() {
    main_thread_id = std::this_thread::get_id(); // Initialized in main
    work(); // Main'den cagri
    std::thread(work).join(); // Child'dan cagri
}
```

---

## 5. Native Handle ve İşletim Sistemi API Erişimi [00:35:00 - 00:45:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C++ Standart Kütüphanesi her şeyi kapsamaz. Örneğin, bir thread'in önceliğini (priority) değiştirmek için standart bir fonksiyon yoktur. Bu gibi durumlarda standarttan çıkıp işletim sisteminin (Windows API, POSIX) fonksiyonlarını çağırmak gerekir.

### ⚙️ Teknik Detay ve Sentaks
`native_handle()` fonksiyonu, alttaki OS thread nesnesinin tutamağını (handle/pointer) döndürür.

```cpp
#include <thread>
#include <windows.h> // Windows'a ozgu (Hoca Windows uzerinden ornek verdi)

void task() { /* ... */ }

int main() {
    std::thread tx(task);
    
    // Hoca: "native_handle derleyiciye/OS'e bagli bir tur dondurur"
    auto handle = tx.native_handle(); 

    // Windows API ile thread onceligini "Highest" (En yuksek) yapma:
    if (SetThreadPriority(handle, THREAD_PRIORITY_HIGHEST)) { // <-- OS specific API call
        std::cout << "Priority set successfully!" << std::endl;
    }

    tx.join();
}
```

### 🔍 Arka Plan (Under the Hood)
`std::thread` aslında ince bir sarmalayıcıdır (thin wrapper). `native_handle()` çağrıldığında, Windows'ta `HANDLE` (void*), Linux'ta ise `pthread_t` (unsigned long) tipiyle karşılaşırız. Bu, C++'ın "Sistem Programlama" gücünü korumasını sağlar; gerektiğinde standart kütüphanenin sınırlarını aşabiliriz.

### 🔗 Önceki Derslerle Bağlantı
Hoca, `std::chrono` dersine atıf yaparak `sleep_for` ve `sleep_until` farkını açıkladı:
*   `sleep_for(duration)`: Belirli bir süre (örneğin 3 saniye) uyu.
*   `sleep_until(time_point)`: Belirli bir ana (örneğin saat 14:00 olana dek) uyu.

---

## 6. Yeni Bir Depolama Sınıfı: `thread_local` [00:45:00 - 01:00:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Multi-thread programlamada en büyük sorun "Shared State" (paylaşılan durum) yönetimidir. Eğer her thread'in kendine özel, global gibi davranan ama diğer thread'lerden izole bir değişkeni olursa, Mutex kullanma zorunluluğu (ve maliyeti) ortadan kalkar.

### ⚙️ Teknik Detay ve Sentaks
C++11 ile dile 4. bir **Storage Class Specifier** (Depolama sınıfı belirleyicisi) eklenmiştir: `thread_local`.

1.  **Automatic** (Yerel değişkenler - Stack)
2.  **Static** (Global/Statik değişkenler - Data Segment)
3.  **Dynamic** (Heap)
4.  **Thread Local** (Thread Local Storage - TLS)

```cpp
#include <iostream>
#include <thread>
#include <syncstream> // C++20 (Hoca cout karismasin diye kullandi)

thread_local int val = 0; // <-- KRITIK: Her thread calistiginda bu degiskenden bir kopya olusturulur.

void func(const std::string& tname) {
    val++; // Her thread kendi val'ini arttirir. Birbirlerini gormezler!
    std::osyncstream(std::cout) << "Thread " << tname << " val: " << val << std::endl;
}

int main() {
    std::jthread t1(func, "A"); // val: 1
    std::jthread t2(func, "B"); // val: 1 (Yeni kopya)
    
    // Hoca: "Hicbir senkronizasyon (Mutex) gerekmez cunku data paylasilmiyor."
}
```

### 🖼️ Görselleştirme (ASCII Art)
Bellek Yerleşimi:
```text
[ Global Data Segment ] -> static int x; (Tum thread'ler buraya bakar - Mutex lazim)

[ Thread A - TLS ]      -> thread_local int val; (Sadece A ulasir)
[ Thread A - Stack ]    -> int local_var;

[ Thread B - TLS ]      -> thread_local int val; (Sadece B ulasir - Adresi farklidir)
[ Thread B - Stack ]    -> int local_var;
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `thread_local` bir değişkenin ömrü (lifetime) ne kadardır?
**Cevap:** Thread başladığında oluşturulur (constructor çağrılır), thread bittiğinde (fonksiyon return ettiğinde veya thread join/detach edildiğinde) yok edilir (destructor çağrılır). Hoca, bunun statik ömürlü değişkenlerden en büyük farkı olduğunu belirtti.

---

### 🕒 [00:25:00 - 01:00:00] Arası Kritik Özet
Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:
1.  **Thread ID ve join:** Bir thread nesnesinin ID'si, thread `join` edildikten sonra geçersiz hale gelir (artık bir thread'i temsil etmez).
2.  **Native API Bağımlılığı:** `native_handle()` kullanıldığında kodun artık taşınabilir (portable) olmadığını bilmek gerekir.
3.  **thread_local vs static:** Statik değişkenler tüm thread'ler için ortaktır (Shared), `thread_local` değişkenler ise her thread için unik'tir (Per-thread).

## 7. `thread_local` Bellek Yönetimi ve Adres Karşılaştırmaları [01:00:00 - 01:15:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Farklı thread'lerin aynı ismi taşıyan ancak fiziksel olarak farklı bellek bölgelerinde bulunan değişkenlere sahip olması, "Global state" (küresel durum) karmaşasını çözer. Eğer her thread kendi verisiyle çalışırsa, kilitleme (locking) mekanizmalarına gerek kalmaz ve performans artar.

### ⚙️ Teknik Detay ve Sentaks
Hoca, `thread_local` değişkenlerin adreslerinin her thread için farklı olduğunu ispatlayan bir kod yazdı.

```cpp
#include <iostream>
#include <thread>
#include <syncstream> // C++20

thread_local int ibal = 0; // TLS (Thread Local Storage)

void thread_func(int* main_ibal_ptr) {
    ibal = 42; // Bu, bu thread'e ozel ibal nesnesidir.
    
    std::osyncstream(std::cout) << "Main's ibal value from child: " << *main_ibal_ptr << "\n" // 9 basar
                                << "Child's local ibal: " << ibal << "\n"                  // 42 basar
                                << "Child's local ibal address: " << &ibal << std::endl;
}

int main() {
    ibal = 9; // Main thread'in kendi kopyasi
    
    std::thread t1(thread_func, &ibal); // Main'deki ibal'in adresini geciyoruz.
    
    t1.join();
    std::cout << "Main thread's ibal still: " << ibal << std::endl; // 9
}
```

### 🔍 Arka Plan (Under the Hood/Memory Layout)
*   **Static Local vs Thread Local:** Hoca, `static` bir değişkenin fonksiyon her çağrıldığında aynı adresi gösterdiğini, ancak `thread_local` bir değişkenin farklı thread'ler tarafından çağrıldığında farklı adresleri gösterdiğini vurguladı.
*   **C++11 Garantisi:** C++11 standartlarından itibaren **Statik Yerel Değişkenlerin** (Static Local Variables) ilk değerini alması (initialization) derleyici tarafından otomatik olarak thread-safe hale getirilmiştir. Buna "Magic Statics" denir.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `thread_local` bir değişken fonksiyona parametre olarak geçilebilir mi?
**Cevap:** Fonksiyon parametreleri `thread_local` olamaz. Sadece namespace scope'daki değişkenler, sınıfların statik veri elemanları veya blok içerisindeki yerel değişkenler `thread_local` olabilir.

---

## 8. `thread_local` Nesnelerin Ömrü ve Ctor/Dtor Davranışı [01:15:00 - 01:35:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Eğer `thread_local` olarak tanımlanan şey bir sınıf (class) nesnesiyse, bu nesnenin ne zaman yaratılıp ne zaman yok edileceğini bilmek, kaynak yönetimi (dosya kapama, socket bırakma vb.) için kritiktir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, nesne ömrünü izlemek için özel bir `MyClass` kullanarak `thread_local` nesnenin hayatını gösterdi.

```cpp
#include <iostream>
#include <thread>
#include <syncstream>

class MyClass {
public:
    MyClass() { std::osyncstream(std::cout) << "Ctor called in thread " << std::this_thread::get_id() << "\n"; }
    ~MyClass() { std::osyncstream(std::cout) << "Dtor called in thread " << std::this_thread::get_id() << "\n"; }
};

void task() {
    thread_local MyClass obj; // <-- KRITIK: Nesne thread basladiginda degil, kod buraya ilk geldiginde olusur.
    std::osyncstream(std::cout) << "Task running...\n";
} // <-- Thread biterken dtor cagrilir.

int main() {
    std::jthread t1(task);
    std::jthread t2(task);
    return 0;
}
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Hoca'nın "Durumdan Vazife Çıkartmak" uyarısı:**
Bir thread'in içinde birden fazla fonksiyon birbirini çağırıyorsa (Call Chain: F1 -> F2 -> F3), `thread_local` değişkeni bu fonksiyonların tamamı için "aynı nesne" kalmaya devam eder. Ancak farklı bir thread bu fonksiyonları çağırırsa, o thread için tamamen yeni bir nesne yaratılır.

---

## 9. Pratik Bir Uygulama: Rastgele Sayı Üretimi ve `std::call_once` [01:35:00 - 02:10:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
`std::mt19937` gibi rastgele sayı motorları (engine) thread-safe değildir. Eğer birden fazla thread aynı motoru kullanırsa "Data Race" oluşur. Çözüm ya her thread'e ayrı motor vermektir (`thread_local`) ya da paylaşılan bir motorun sadece bir kez initialize edilmesini sağlamaktır (`std::call_once`).

### ⚙️ Teknik Detay ve Sentaks
Hoca, `std::call_once` ve `std::once_flag` kullanarak "sadece bir kez" çalıştırılması garanti edilen kod bloklarını gösterdi.

```cpp
#include <iostream>
#include <thread>
#include <mutex> // once_flag ve call_once burada

std::once_flag init_flag; // <-- Hoca vurguladi: Bu bayrak 'global' veya paylasilan bir yerde olmali.

void init_resource() {
    std::cout << "Sadece bir kere calisacak kritik baslatma kodu!\n";
}

void worker(int id) {
    // Hoca: "Hangisi once girerse piyango ona vurur, digerleri bu satiri pas gecer."
    std::call_once(init_flag, init_resource); 
    std::cout << "Worker " << id << " devam ediyor...\n";
}

int main() {
    std::vector<std::jthread> v;
    for(int i=0; i<10; ++i) v.emplace_back(worker, i);
}
```

### 🔍 Arka Plan (Under the Hood)
*   `std::once_flag`: İçsel olarak bir durum tutar (baslatılmadı, baslatılıyor, baslatıldı).
*   `std::call_once`: Eğer bir thread fonksiyonu koştururken bir exception throw ederse, `once_flag` "baslatılmadı" durumuna geri döner ve sıradaki thread tekrar dener. Bu, "guaranteed initialization" sağlar.

---

## 10. Thread-Safe Singleton ve Database Connection Örneği [02:10:00 - 02:44:06]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Klasik Singleton tasarımı multi-thread ortamlarda yıkılır (iki thread aynı anda `if (ptr == nullptr)` kontrolünü geçebilir). C++11 öncesinde "Double-Checked Locking" gibi karmaşık idiomlar gerekiyordu. Artık `call_once` veya "Magic Statics" ile bu çok daha basit.

### ⚙️ Teknik Detay ve Sentaks (Thread-Safe Singleton)
Hoca, `call_once` kullanarak modern ve güvenli Singleton implementasyonunu gösterdi.

```cpp
class Singleton {
private:
    static Singleton* m_instance;
    static std::once_flag m_init_flag;
    Singleton() { std::cout << "Singleton olustu.\n"; }

public:
    static Singleton* get_instance() {
        std::call_once(m_init_flag, []() {
            m_instance = new Singleton(); // <-- Thread-safe allocation
        });
        return m_instance;
    }
};

Singleton* Singleton::m_instance = nullptr;
std::once_flag Singleton::m_init_flag;
```

### 🖼️ Görselleştirme (ASCII Art)
Singleton ve `call_once` mantığı:
```text
Thread 1 ----> [call_once] --- (Kilit Al) --- [NEW OBJECT] --- (Kilit Birak)
Thread 2 ----> [call_once] --- (Zaten yapilmis, bekleme/gec) ---+
Thread 3 ----> [call_once] -------------------------------------+--> Ikonik Tek Nesne
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** "Magic Statics" (C++11) varken neden `call_once` kullanalım?
**Cevap:** Hoca, `static` yerel değişkenlerin fonksiyon scope'u ile sınırlı olduğunu; ancak `call_once`'un farklı fonksiyonlar içinden bile aynı bayrağı (`once_flag`) kullanarak global bir koordinasyon sağlayabildiğini belirtti.

---

### 🕒 [01:00:00 - 02:44:06] Arası Kritik Özet
Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:
1.  **Dinamik Nesnelerde thread_local Yanılgısı:** `thread_local` sadece işaretçinin (pointer) kopyasını oluşturur, işaret edilen dinamik nesneyi değil. Nesnenin kendisi de thread-local isteniyorsa nesne doğrudan `thread_local` tanımlanmalıdır.
2.  **Singleton'da Data Race:** Mutex veya `call_once` kullanmayan Singleton'lar multi-thread ortamda birden fazla nesne oluşmasına (UB) neden olur.
3.  **Critical Section Genişliği:** Mutex kilitleri (`lock`) sadece gerekli satırları kapsamalıdır. Gereksiz yere tüm fonksiyonu kilitlemek paralelliği öldürür.

---


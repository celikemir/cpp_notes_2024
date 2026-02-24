Bu ders notları, **Necati Ergin**'in 10 Şubat 2025 tarihli "Concurrency (Eş Zamanlılık)" konulu 62. dersinin ilk **31 dakikalık** teknik dökümüdür.

---

# Concurrency Serisi: Senkronizasyon ve Mutex Mekanizmaları - 1

**Ders Tarihi:** 10 Şubat 2025  
**Konu:** Eş Zamanlılıkta Senkronizasyon Araçları ve `std::mutex`  
**Timestamp:** [00:00.000 - 00:31:35]

## 1. Giriş ve Hatırlatmalar [00:00 - 01:56]
Hoca, derse Modern C++ ile hayatımıza giren dördüncü **Storage Class** (Depolama Sınıfı) kategorisini hatırlatarak başladı:

1.  **Automatic** (Otomatik ömürlü)
2.  **Static** (Statik ömürlü)
3.  **Dynamic** (Dinamik ömürlü)
4.  **Thread Local** (İş parçacığı yerel - C++11 ile eklendi)

## 2. Senkronizasyon (Synchronization) Kavramı [01:56 - 05:41]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Birden fazla thread'in paylaşımlı (shared) veriye aynı anda erişmesi, veri tutarsızlığına ve programın çökmesine neden olur. Amacımız, bu paylaşımlı veriyi güvenli bir şekilde kullandırmaktır.

### 🚩 Kritik Nokta: Race Condition vs. Data Race
Hoca bu iki terimin mülakatlarda çok sık sorulduğunu ve asla birbirine karıştırılmaması gerektiğini vurguladı:

*   **Race Condition (Yarış Durumu):** Birden fazla thread'in aynı veriye erişmesi durumudur. Bu durum her zaman **UB** (Undefined Behavior - Tanımsız Davranış) oluşturmak zorunda değildir. Programın lojik yapısı yanlış olabilir (sıralama hatası gibi) ama bu durum "benign" (iyi huylu) olabilir.
*   **Data Race (Veri Yarışı):** En az bir thread yazma (write) yaparken, diğerlerinin okuma veya yazma yapması durumudur. Bu durum doğrudan **UB**'dir (Tanımsız Davranıştır). Cppreference üzerinde birçok öğe için "Data race riski yoktur" ifadesi bu güvenliği belirtir.

> **Hoca'nın İdiomu:** "Data race, programın ölümü demektir. Veriyi kapışmak için thread'lerin yarışmasıdır."

## 3. Mutual Exclusion ve Kritik Bölge (Critical Section) [05:41 - 10:43]

### ⚙️ Teknik Detay ve Sentaks
**Critical Section (Kritik Bölge):** Paylaşımlı verinin kullanıldığı ve Data Race riski taşıyan kod alanıdır (blok, fonksiyon tamamı veya döngü gövdesi olabilir).

Bu bölgeyi korumak için **Mutual Exclusion** (Karşılıklı Dışlama) kullanılır. C++'ta bunu sağlayan araçların başında **Mutex** sınıfları gelir.

### 🖼️ Görselleştirme (ASCII Art) - Lavabo Anahtarı Analojisi
Hoca'nın meşhur örneği:
```text
Thread A (Aday) ----> [ Kapı ] <---- Thread B (Aday)
                         |
               [ MUTEX (ANAHTAR) ]
                         |
    İçerisi: CRITICAL SECTION (Paylaşımlı Değişken)
```
1.  İlk giren anahtarı (Mutex) alır ve kapıyı kilitler (`lock()`).
2.  Diğer thread'ler kapıda bekler (**Bloke olur**).
3.  İçerideki thread işini bitirince kilidi açar (`unlock()`).
4.  Bekleyenlerden biri (belirsiz/non-deterministic) anahtarı kapar.

## 4. Mutex Fonksiyonları: `lock()` ve `unlock()` [10:43 - 14:07]

### ⚙️ Teknik Detay ve Sentaks
`std::mutex` sınıfının en temel iki fonksiyonu:
1.  `lock()`: Kilidi edinir. Eğer kilit başkasındaysa, thread işletim sistemi tarafından durdurulur (bloke edilir).
2.  `unlock()`: Kilidi serbest bırakır.

```cpp
#include <mutex>

std::mutex mtx; // <-- Paylaşılan mutex nesnesi

void critical_function() {
    mtx.lock();   // <-- Kilidi edin (Bloke edici çağrı)
    // --- CRITICAL SECTION BAŞI ---
    // Paylaşımlı veri üzerinde işlemler...
    // --- CRITICAL SECTION SONU ---
    mtx.unlock(); // <-- Kilidi serbest bırak
}
```

### 🔍 Arka Plan (Under the Hood)
Thread'in bloke olması, işletim sisteminin bir faaliyetidir. Thread artık CPU zamanı tüketmez, kilit açılana kadar uyutulur. Kilidi kimin alacağı **deterministik değildir**; OS planlayıcısı (scheduler) karar verir.

## 5. Neleri Korumalıyız? (Shared Data) [14:07 - 17:16]

Hoca, her kodun kilit altına alınmasının performansı öldüreceğini belirtti.

### 🚩 Kritik Nokta: Korumaya Gerek OLMAYAN durumlar
*   **Automatic Storage Duration (Stack):** Her thread'in kendi **Stack Segment**'i vardır. Yerel değişkenler thread-safe'dir.
*   **Thread Local Variables:** Sadece o thread'e özgüdür.
*   **Read-Only Access:** Eğer tüm thread'ler sadece okuma yapıyorsa senkronizasyona gerek yoktur.

### ⚙️ Teknik Detay ve Sentaks
```cpp
void safe_function(int val) { // val otomatik ömürlü, koruma gerekmez
    int x = 10;               // x stack'te, koruma gerekmez
    // ...
}

// ANCAK:
int global_g = 0;             // GLOBAL: MUTEX GEREKİR!
static int static_s = 0;      // STATIC: MUTEX GEREKİR!

void unsafe_if_shared(int* ptr, int& ref) {
    // Pointer veya referans ile gelen veri başka thread'lerle 
    // paylaşılıyor olabilir. MUTEX GEREKEBİLİR!
}
```

## 6. Mutex Sınıf Çeşitleri [17:16 - 25:00]

Hoca, C++'ta neden birden fazla mutex sınıfı olduğunu "yetenek vs. maliyet" dengesiyle açıkladı.

### 📊 Standart Karşılaştırması (Mutex Türleri)

| Sınıf İsmi | Özellik |
| :--- | :--- |
| `std::mutex` | En temel, en az maliyetli "adi" (sıradan) mutex. |
| `std::recursive_mutex` | Aynı thread'in aynı mutex'i birden fazla kez kilitlemesine izin verir. |
| `std::timed_mutex` | Belirli bir süre kilidi almayı deneme (`try_lock_for/until`) yeteneği ekler. |
| `std::shared_mutex` | (C++17) Reader/Writer kilidi. Okumalar eş zamanlı, yazma tekildir. |

> **Kritik Kural (Minimalist İlke):** `std::mutex` işinizi görüyorsa asla diğerlerini kullanmayın. Hem maliyeti artırırsınız hem de kodu okuyana "neden timed/recursive kullanıldı?" diye yanlış bir ipucu verirsiniz.

## 7. `std::timed_mutex` ve `try_lock` Mantığı [21:35 - 25:00]

`std::mutex`'te `lock()` dediğinizde ya kilidi alırsınız ya da sonsuza kadar bloke olursunuz. `timed_mutex` ise bir seçenek sunar:

```cpp
// Rationale: "Belirli bir süre dene, alamazsan başka iş yap, bloke olma."
if (my_timed_mtx.try_lock_for(std::chrono::milliseconds(10))) {
    // Kilidi aldık, işini yap
    my_timed_mtx.unlock();
} else {
    // Kilidi alamadık, 10ms doldu. Bloke olmadan yola devam et.
}
```

## 8. Alihan Bey'in Sorusu: Bir Mutex ile Birden Fazla Veri [25:00 - 31:35]

**Soru:** "Bir mutex aynı anda iki farklı workload'da farklı thread'ler tarafından kullanılabilir mi?"

**Cevap:** Evet, ancak tasarım kritik:
*   Farklı veriler için farklı mutex'ler kullanmak performansı artırır (Fine-grained locking).
*   Aynı paylaşımlı değişkeni kullanan 3 farklı fonksiyon (`foo`, `bar`, `baz`) varsa, bu üçü de **aynı mutex nesnesini** kilitlemelidir.

```cpp
std::mutex shared_mtx; // <-- Kritik: Nesne ortak olmalı

void foo() { 
    shared_mtx.lock(); // Paylaşılan 'X' değişkenine erişim
    shared_mtx.unlock(); 
}

void bar() { 
    shared_mtx.lock(); // Aynı 'X' değişkenine erişim
    shared_mtx.unlock(); 
}
```

### ⚙️ Teknik Detay: Sınıf İçi Kullanım (Thread-Safe Class)
Bir sınıfın üye fonksiyonlarını thread-safe yapmak için sınıfın içine bir `std::mutex` veri elemanı eklenir:
```cpp
class MyClass {
    int m_data;
    mutable std::mutex m_mtx; // <-- Hoca: 'mutable' olmasına dikkat çekebilir (ileride)
public:
    void update() {
        m_mtx.lock();
        m_data++;
        m_mtx.unlock();
    }
};
```

### 🔍 Arka Plan (Lock-free Programming)
Hoca, mutex kullanmanın her zaman bir maliyeti (context switch, blocking) olduğunu belirtti. Thread'lerin hiç bloke edilmeden ilerlediği en zor alan ise **Lock-free programming**'dir.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  `std::mutex`'i yerel (local) tanımlamak (Her thread kendi yerel mutex'ini kilitlerse hiçbir senkronizasyon sağlanmaz!).
2.  Okuma amaçlı erişimlerde gereksiz mutex kullanımı.
3.  İhtiyaç yokken `std::recursive_mutex` gibi daha ağır sınıfları tercih etmek.

Bu bölüm, transkriptin **[31:35] ile [01:05:00]** arasındaki kısmını kapsamaktadır. Hoca bu bölümde RAII (Guard) sınıflarının hayati önemini, manuel kilit yönetiminin risklerini ve veri yarışının (data race) pratik örneklerini incelemektedir.

---

# Concurrency Serisi: RAII Guard Sınıfları ve Data Race Pratiği

**Timestamp:** [31:35 - 01:05:00]

## 1. RAII ve Guard Sınıfları [31:35 - 37:45]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Manuel `mtx.lock()` ve `mtx.unlock()` çağrıları tehlikelidir. Kritik bölge (critical section) içerisinde bir **Exception** (istisna) fırlatılırsa, programın akışı `unlock()` satırını görmeden sonlanır. Bu durumda Mutex kilitli kalır ve bu Mutex'i bekleyen diğer tüm thread'ler sonsuza kadar bloke olur (**Deadlock**).

### ⚙️ Teknik Detay ve Sentaks
Bu sorunu çözmek için **RAII** (Resource Acquisition Is Initialization) prensibiyle çalışan "Guard" sınıfları kullanılır.
*   **Constructor (Kurucu):** Mutex'i argüman olarak alır ve kilitler (`lock`).
*   **Destructor (Yıkıcı):** Nesne kapsam dışına (out of scope) çıktığında veya **Stack Unwinding** (yığın boşaltma) gerçekleştiğinde otomatik olarak kilidi açar (`unlock`).

### 📊 Standart Karşılaştırması (Guard Sınıfları)
Hoca, 4 temel Guard sınıfını ve farklarını özetledi:

| Sınıf İsmi | Standart | Özellik |
| :--- | :--- | :--- |
| `std::lock_guard` | C++11 | En basit, minimalist guard. Kopyalanamaz, taşınamaz. |
| `std::unique_lock` | C++11 | En yetenekli guard. **Movable** (taşınabilir), kilit üzerinde tam kontrol sağlar. |
| `std::scoped_lock` | C++17 | **Variadic** (değişken sayıda) mutex alabilir. Deadlock riskini algoritmasıyla azaltır. |
| `std::shared_lock` | C++14 | `std::shared_mutex` ile kullanılır (Okuma amaçlı kilit). |

> **Hoca'nın İdiomu:** "Explicit (açıkça) olarak `unlock()` çağırmakla uğraşmayın, guard sınıfları durumu 'vazife çıkartıp' halleder."

## 2. Pratik Örnek: Data Race (Veri Yarışı) Oluşturma [43:40 - 52:00]

Hoca, korumasız bir paylaşımlı değişkenin nasıl hatalı sonuç verdiğini gösteren "tipik bir data race" kodu yazdı.

### ⚙️ Teknik Detay ve Sentaks
```cpp
#include <iostream>
#include <thread>
#include <vector>

unsigned long long cnt = 0; // <-- SHARED DATA (Paylaşımlı Veri)

void foo() {
    for (auto i = 0ULL; i < 1000000ULL; ++i) {
        ++cnt; // <-- CRITICAL SECTION: Data Race burada oluşuyor!
    }
}

int main() {
    {
        std::jthread t1(foo); // C++20: Destructor otomatik join eder
        std::jthread t2(foo);
        std::jthread t3(foo);
        std::jthread t4(foo);
    } // <-- Hoca buraya nested block koydu: t1..t4 burada join edilir.

    // Beklenen: 4.000.000 | Gerçekleşen: 1.060.123 (Belirsiz!)
    std::cout << "Counter: " << cnt << std::endl; 
}
```

### 🔍 Arka Plan (Compiler Eye)
`++cnt` işlemi işlemci seviyesinde tek bir makine komutu değildir (Read-Modify-Write). Thread A veriyi okuyup arttırırken, Thread B araya girip eski değeri okuyabilir. Sonuçta bir artırım "kaybolur". Bu durum **UB** (Undefined Behavior) kategorisindedir.

### 🔗 Önceki Derslerle Bağlantı
Hoca, `std::jthread` nesnesinin (C++20) kapsam sonunda otomatik `join` yaptığını hatırlattı. Sonucun doğru yazılması için `main` içinde suni bir blok `{ }` oluşturarak thread'lerin işinin bitmesini garantiledi.

## 3. Mutex ile Veri Yarışını Çözmek [52:00 - 58:30]

### ⚙️ Teknik Detay ve Sentaks
Hoca, aynı kodun Mutex ile senkronize edilmiş halini sundu:

```cpp
#include <mutex>

std::mutex mtx; // <-- Paylaşılan tek bir Mutex nesnesi

void foo_safe() {
    for (auto i = 0ULL; i < 1000000ULL; ++i) {
        mtx.lock();   // <-- Kilidi al
        ++cnt; 
        mtx.unlock(); // <-- Serbest bırak
    }
}
```
**Sonuç:** Program her çalıştırıldığında kesin olarak **4.000.000** değerini üretir.

### 🚩 Kritik Nokta: Performans Sorunu
Hoca burada bir uyarıda bulundu: Kilidi döngünün içine koymak (`lock` her turda çağrılıyor) maliyeti çok artırır. Eğer tüm döngü boyunca kilidi tutarsak performans artar ama diğer thread'ler çok bekler. **"Critical Section mümkün olduğunca dar tutulmalıdır."**

## 4. Alihan Bey'in Sorusu: Çoklu Fonksiyon ve Mutex İlişkisi [58:30 - 01:00:30]

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** "Farklı fonksiyonlar (`foo`, `bar`, `baz`) aynı değişkeni (`cnt`) kullanıyorsa ne yapılmalı?"

**Hoca'nın Analizi:**
*   **HATA:** Her fonksiyonun içinde kendi yerel mutex'ini oluşturmak. (Herkes kendi anahtarını kullanırsa kimse birbirini durduramaz).
*   **DOĞRU:** Tüm fonksiyonlar **aynı global/statik mutex nesnesini** kullanmalıdır.

**Hatalı Senaryo (Deadlock/Race Riski):**
```cpp
std::mutex mtx1, mtx2, mtx3; // <-- HATA: Farklı mutexler

void foo() { mtx1.lock(); ++cnt; mtx1.unlock(); }
void bar() { mtx2.lock(); ++cnt; mtx2.unlock(); } // <-- Race Condition devam eder!
```

## 5. I/O Stream (`std::cout`) Senkronizasyonu [01:01:00 - 01:05:00]

### ⚙️ Teknik Detay ve Sentaks
Hoca, birden fazla thread'in `std::cout` kullanarak ekrana karakter basması durumunu inceledi.

**Gözlem:**
*   `std::cout` karakter bazında thread-safe'dir (Karakterler bozulmaz).
*   Ancak mesaj bazında **deterministik değildir**.

```cpp
void print_block(char c) {
    for (int i = 0; i < 50; ++i) {
        std::cout << c; // Farklı threadler: * $ * $ * $ şeklinde karışabilir.
    }
}
```

### 📊 Standart Karşılaştırması (C++20)
Hoca, `std::osyncstream` (C++20) sınıfına değindi. Bu sınıf, çıktıları biriktirip tek bir atomik hamlede `cout`'a göndererek mesajların birbirine karışmasını engeller.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  **Dangling Lock:** Exception fırlatılan bir blokta manuel `unlock()`'ın çağrılamaması.
2.  **Scope Hatası:** Mutex nesnesini senkronize edilecek thread'lerin erişemeyeceği bir kapsamda (local) tanımlamak.
3.  **Sözde Singleton:** Mutex kullanılmadan uygulanan Singleton tasarım deseninin multi-thread ortamlarda birden fazla nesne yaratması (İlerleyen dakikalarda detaya girecek).

Bu bölüm, transkriptin **[01:05:00] ile [01:32:00]** arasındaki kısmını kapsamaktadır. Hoca bu bölümde Thread Safety (İş parçacığı güvenliği) tanımını, STL konteynerlarının davranışlarını, Internal Synchronization (İçsel Senkronizasyon) kavramını ve `try_lock` fonksiyonunu derinlemesine incelemektedir.

---

# Concurrency Serisi: Thread Safety, Internal Synchronization ve `try_lock`

**Timestamp:** [01:05:00 - 01:32:00]

## 1. Thread Safety (İş Parçacığı Güvenliği) Tanımı [01:05:00 - 01:07:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir kod parçasının "Thread Safe" olması, o kodun birden fazla thread tarafından eş zamanlı olarak çalıştırıldığında dahi veri tutarsızlığına, **Data Race**'e veya **Deadlock**'a yol açmaması demektir.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** "Thread Safe olmak performansın çok yüksek olması mı demektir?"
**Cevap:** Hayır. Thread Safety sadece güvenlikle ilgilidir. Hatta senkronizasyon maliyetleri (blocking, context switch) nedeniyle thread safe bir kod, korumasız bir koda göre daha yavaş çalışabilir.

## 2. STL Konteynerları ve Eş Zamanlı Erişim [01:07:00 - 01:09:30]

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** "`std::vector` nesnesine birden fazla thread aynı anda erişirse ne olur?"
**Hoca'nın Analizi:**
1.  **Tam Güvensiz Durum:** Eğer en az bir thread vektöre bir öğe ekliyorsa (`push_back`) ve diğerleri okuyorsa, bu kesinlikle **UB**'dir (Undefined Behavior).
2.  **Güvenli Durum (Farklı Alanlar):** STL şu garantiyi verir: Eğer her thread vektörün **farklı öğeleriyle** (farklı bellek alanları) uğraşıyorsa, senkronizasyona gerek yoktur.

**Örnek (Vektör Bölüştürme):**
```cpp
// 10.000 öğeli bir vektör olsun
std::vector<int> ivec(10000); 

// Thread 1: 0 - 2500 arası indislerle çalışıyor
// Thread 2: 2501 - 5000 arası indislerle çalışıyor
// Bu durum senkronizasyon gerektirmez, çünkü bellek alanları ayrık. <-- Hoca buna dikkat çekti
```

## 3. Internal Synchronization (İçsel Senkronizasyon) [01:09:30 - 01:17:50]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Vektörü veya listeyi her kullandığımızda dışarıdan (client side) Mutex ile kilitlemek zahmetlidir ve hata riskini artırır. Bunun yerine, sınıfın kendi üye fonksiyonları içinde Mutex kullanarak kendini korumasına **Internal Synchronization** denir.

### ⚙️ Teknik Detay ve Sentaks (Sınıf İçi Korumalı Liste)
Hoca, `std::list` nesnesini sarmalayan ve içsel senkronizasyon sağlayan bir `List` sınıfı örneği verdi:

```cpp
#include <mutex>
#include <list>
#include <iostream>

class List {
    std::list<int> m_list;
    mutable std::mutex mtx; // <-- KRİTİK: mutable olmalı! 

public:
    void push_back(int val) {
        mtx.lock();
        m_list.push_back(val); // <-- Critical Section: Sınıf bunu kendi içinde hallediyor.
        mtx.unlock();
    }

    size_t size() const {
        mtx.lock(); // <-- HATA ALIRIZ: mutable olmasaydı const fonksiyonda lock() çağıramazdık.
        auto sz = m_list.size();
        mtx.unlock();
        return sz;
    }
};
```

### 🔍 Arka Plan: `mutable` Keyword (Anahtar Sözcük)
Hoca, `mutable` kelimesinin Mutex bağlamındaki hayati önemini açıkladı:
*   `const` üye fonksiyonlar nesneyi değiştirmez (Logical Constness).
*   Ancak `mtx.lock()` ve `mtx.unlock()` fonksiyonları Mutex nesnesinin iç durumunu değiştirir (not `const`).
*   Eğer Mutex `mutable` tanımlanmazsa, `const` fonksiyonlar içinde kilit mekanizması kullanılamaz. Bu durum **"Sentaks Hatası"** oluşturur.

## 4. `std::osyncstream` ve Çıktı Senkronizasyonu [01:17:50 - 01:21:00]

### ⚙️ Teknik Detay ve Sentaks
C++20 ile gelen `std::osyncstream`, birden fazla thread'in `std::cout`'a gönderdiği verilerin satır bazında karışmasını engeller.

```cpp
#include <syncstream>

void foo() {
    std::osyncstream(std::cout) << "Thread ID: " << std::this_thread::get_id() << "\n";
    // Bu satır atomik olarak yazılır, diğer thread'lerin çıktıları araya giremez.
}
```

## 5. `try_lock()` Üye Fonksiyonu [01:24:40 - 01:28:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
`lock()` fonksiyonu "ya kilidi al ya da sonsuza kadar bekle" der. Ancak bazen bir thread kilidi alamazsa bloke olmak yerine başka işler yapmak isteyebilir.

### ⚙️ Teknik Detay ve Sentaks
*   `try_lock()` hemen döner (Non-blocking).
*   Kilidi edinirse `true`, edinemezse `false` döndürür.

```cpp
std::mutex mtx;

void try_increase() {
    if (mtx.try_lock()) { // <-- Kilidi edinmeye çalış, alamazsan bloklanma!
        // ... kritlik işlemler ...
        mtx.unlock(); // <-- Alındıysa mutlaka serbest bırakılmalı
    } else {
        // Kilidi başkası tutuyor, "boş duracağına" başka işler yap.
    }
}
```

## 6. `try_lock()` Pratik Örneği ve Determinizm Analizi [01:28:00 - 01:32:00]

Hoca, 10 thread'in her birinin bir döngüde 100.000 kez `try_lock()` denediği bir örnek yaptı.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** "10 thread 100.000'er kez deneme yaparsa toplam sayaç 1.000.000 olur mu?"
**Hoca'nın Yanıtı:** "Hayır. Çünkü bir thread kilidi alamadığında o turu pas geçer (`skip` eder). Sonuç 167.971 gibi tamamen rastgele (non-deterministic) bir değer çıkar."

### 🔍 Arka Plan (Memory Layout)
Hoca, `static` yerel değişkenlerin senkronizasyonu hakkında mühim bir not düştü:
*   Fonksiyon içindeki **otomatik ömürlü** (stack) değişkenler zaten thread-safe'dir.
*   Ancak fonksiyona **referans veya pointer** ile dışarıdan gelen nesneler (heap veya global), otomatik ömürlü gibi görünse de aslında paylaşımlı oldukları için koruma gerektirir.

---

**Bu bölümde Hoca şu 3 kritik noktaya vurgu yaptı:**
1.  **Mutable Mutex:** `const` üye fonksiyonlar içinde senkronizasyon yapabilmek için sınıfın Mutex elemanını mutlaka `mutable` yapmalısınız.
2.  **Internal Synchronization Pattern:** Sınıfın kendi kendini koruması, "sen bana güven, ben durumu hallediyorum" demesidir (Wrapper mantığı).
3.  **try_lock Riskleri:** `try_lock` ile kilit alındığında, manuel yönetim yapılıyorsa `unlock`'ın unutulması felakettir.

Bu bölüm, transkriptin **[01:32:00] ile [02:39:34] (Ders Sonu)** arasındaki kısmını kapsamaktadır. Hoca bu bölümde `std::timed_mutex` üzerindeki zaman deneylerini, Exception Safety (İstisna Güvenliği), Deadlock (Ölü Kilit) senaryolarını, Singleton Pattern'ın thread-safe implementasyonunu ve `std::recursive_mutex` sınıfını derinlemesine incelemektedir.

---

# Concurrency Serisi: Zamanlanmış Kilitler, Deadlock ve Singleton

**Timestamp:** [01:32:00 - 01:46:00]

## 1. `std::timed_mutex` ve Zaman Deneyleri [01:32:00 - 01:41:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
"Adi" (Sıradan/Sade) `std::mutex`, kilidi alamazsa sonsuza kadar bekler. `std::timed_mutex` ise kilidi almak için belirli bir süre sınır koymamıza (`Timeout`) olanak tanır.

### ⚙️ Teknik Detay ve Sentaks
Hoca, `timed_mutex`'in iki kritik fonksiyonunu vurguladı:
*   `try_lock_for(duration)`: Belirli bir süre (Örn: 20ms) boyunca dene.
*   `try_lock_until(time_point)`: Belirli bir ana (Örn: Saat 21:30:00) kadar dene.

```cpp
#include <mutex>
#include <chrono>

std::timed_mutex tmtx; // <-- 'Adi' mutex'ten farkı zaman yeteneğidir.

void timing_experiment() {
    using namespace std::chrono_literals;
    if (tmtx.try_lock_for(1ms)) { // <-- 1 milisaniye boyunca şansını dene
        // ... kritlik işlemler ...
        tmtx.unlock();
    } else {
        // Zaman doldu, kilidi alamadık. Bloke olmadan devam et!
    }
}
```

### 🔍 Arka Plan (Under the Hood)
Hoca süreleri azaltarak bir deney yaptı:
1.  **20ms:** 10 thread'in hepsi kilidi alabildi (Counter = 1.000.000).
2.  **1ms:** Bazı thread'ler kilidi kaçırmaya başladı.
3.  **1 mikrosaniye (us):** Başarı oranı %50'lere düştü.
4.  **Hoca'nın Notu:** Bu sonuçlar tamamen donanım performansına ve OS scheduler'ın (işletim sistemi zamanlayıcısı) o anki yüküne bağlıdır (**Non-deterministic**).

> **Hoca'nın İdiomu:** "Burada 'adi' sözcüğü şerefsiz anlamında değil, sıradan, ekstra özelliği olmayan anlamındadır."

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  `std::mutex` üzerinde `try_lock_for` çağırmaya çalışmak (Sentaks hatası: mutex'in zaman yeteneği yoktur).
2.  `try_lock` başarılı olduğunda `unlock` etmeyi unutmak (Kilit sonsuza kadar o thread'de kalır).
3.  Zamanlanmış kilitlerin her makinede aynı performansı vereceğini sanmak.

---

**Timestamp:** [01:46:00 - 02:01:00]

## 2. Exception Safety ve RAII Guard Kullanımı [01:46:00 - 01:54:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Kritik bir bölgede kilit tutulurken bir hata (`std::runtime_error`) oluşursa ve kilit manuel (`unlock()`) bırakılıyorsa, program `unlock` satırına hiç ulaşamaz.

### ⚙️ Teknik Detay ve Sentaks
Hoca, hatayı simüle eden şu kodu yazdı:
```cpp
std::mutex mtx;

void unsafe_exception(int x) {
    mtx.lock();
    if (x % 2 == 0) { // Çift sayı gelirse
        throw std::runtime_error("HATA!"); // <-- KİLİT BURADA TAKILI KALDI!
    }
    mtx.unlock(); // <-- Buraya asla ulaşılamaz
}
```
**Çözüm:** `std::lock_guard` (RAII) kullanımı.
```cpp
void safe_exception(int x) {
    std::lock_guard<std::mutex> guard(mtx); // <-- Hoca: C++17 CTAD ile <std::mutex> yazılmasa da olur
    if (x % 2 == 0) throw std::runtime_error("HATA!"); 
} // <-- Destructor otomatik unlock eder! (Stack Unwinding süreci)
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** "Stack Unwinding sırasında `unlock`'ın çağrılacağı garanti mi?"
**Cevap:** Evet. RAII nesnesi otomatik ömürlü olduğu için, kapsamdan nasıl çıkılırsa çıkılsın (normal veya exception), destructor çalışır ve Mutex serbest kalır.

---

**Timestamp:** [01:54:00 - 02:16:00]

## 3. Deadlock (Ölü Kilit) Senaryoları ve Çözümler [02:01:00 - 02:16:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
İki thread'in karşılıklı olarak birbirinin elindeki kilidi beklemesi durumudur. Program donar ve ilerlemez.

### 🖼️ Görselleştirme (Deadlock ASCII Art)
```text
Thread 1: [Lock A] --(istiyor)--> [Lock B]
Thread 2: [Lock B] --(istiyor)--> [Lock A]
         ^                        |
         |________________________| <--- DEADLOCK!
```

### ⚙️ Teknik Detay ve Sentaks
Hoca, kilit sıralamasının önemini gösterdi:
*   **Yanlış:** Thread 1 (A sonra B), Thread 2 (B sonra A).
*   **Doğru:** Her zaman aynı sıra (Hep A sonra B).

**C++17 Çözümü (`std::scoped_lock`):**
```cpp
std::mutex mtx1, mtx2;

void thread_func() {
    // Variadic (Değişken sayıda) mutex alabilir ve Deadlock'tan kaçınan bir algoritma kullanır.
    std::scoped_lock lock(mtx1, mtx2); // <-- Hoca: "C++17 ile lock_guard yerine bunu kullanın."
    // ... işlemler ...
}
```

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  Farklı thread'lerde mutex'leri farklı sıralarla kilitlemek (Deadlock'ın 1 numaralı sebebi).
2.  İstisna riski olan bölgelerde RAII (Guard) sınıflarını kullanmamak.
3.  `std::lock` (global fonksiyon) ile kilitlenen mutex'leri manuel `unlock` etmemek.

---

**Timestamp:** [02:16:00 - 02:39:34]

## 4. Thread-Safe Singleton ve Meyers Singleton [02:16:00 - 02:31:00]

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** "Klasik Singleton multi-thread ortamda neden tehlikelidir?"
**Hoca'nın Analizi:** İki thread aynı anda `if (mp == nullptr)` kontrolünü geçerse, her ikisi de `new Singleton()` yapar. İki nesne oluşur, Singleton prensibi çöker.

### ⚙️ Teknik Detay ve Sentaks
Hoca, **Double Checked Locking Pattern (DCLP)** ve ardından en modern çözüm olan **Meyers Singleton**'ı anlattı:

```cpp
// MEYERS SINGLETON (Modern C++ En İyi Uygulama)
class Singleton {
public:
    static Singleton& get_instance() {
        static Singleton s; // <-- STATIC LOCAL: Thread-safe initialization!
        return s;
    }
private:
    Singleton() { std::cout << "Initializing...\n"; }
};
```

### 🔍 Arka Plan (Under the Hood)
Modern C++ (C++11 ve sonrası) standartlarına göre, **statik yerel değişkenlerin hayata getirilmesi (initialization)** derleyici tarafından otomatik olarak senkronize edilir. Hoca 100 thread ile test etti ve sadece "1 kez" initialization yazısı çıktığını gösterdi.

## 5. `std::recursive_mutex` [02:31:00 - 02:39:34]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir üye fonksiyonun, yine kilitleme yapan başka bir üye fonksiyonu çağırması durumunda (veya rekürsif fonksiyonlarda), aynı thread kendi tuttuğu kilidi tekrar almaya çalışırsa kendini kilitler.

### ⚙️ Teknik Detay ve Sentaks
```cpp
class Database {
    std::recursive_mutex rmtx;
public:
    void func_a() {
        rmtx.lock();
        func_b(); // <-- HATA: Normal mutex olsaydı burada Deadlock oluşurdu!
        rmtx.unlock();
    }
    void func_b() {
        rmtx.lock(); // <-- Recursive mutex aynı thread'e izin verir.
        // ... işlemler ...
        rmtx.unlock();
    }
};
```

### 🚩 Kritik Nokta
Hoca mühim bir kuralı hatırlattı: "Kilidi kaç kez `lock` ettiyseniz, o kadar kez `unlock` etmek zorundasınız."

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  Eski tip (C++98) Singleton implementasyonlarını multi-thread projelerde kullanmak.
2.  `std::recursive_mutex`'i gerekmediği halde kullanıp performans kaybı yaşamak.
3.  Recursive mutex'te `lock/unlock` sayısının (n defa) eşit olmaması.

---

**Ders Sonu Notu:** Hoca, çarşamba günü `std::shared_mutex`, `condition_variable`, `future` ve `promise` konularına geçeceğini belirterek dersi bitirdi.


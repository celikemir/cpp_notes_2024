Bu ders notları, Necati Ergin'in 2 Ekim 2024 tarihli C++ dersinin (28. gün) ilk 45 dakikasını, teknik derinliği koruyarak ve hocanın üslubuna sadık kalarak yeniden inşa etmektedir.

---

# C++ Exception Handling - Derinlemesine İnceleme (Bölüm 1)

## 1. Standart Kütüphane Exception Hiyerarşisi (00:00 - 05:37)

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Hata yönetiminde bir standart birliği sağlamak amacıyla C++, tüm standart exception'ları polimorfik bir hiyerarşide toplar. Bu sayede, spesifik hata türlerini bilmesek bile en tepedeki taban sınıf üzerinden tüm hataları yakalama şansımız olur.

### ⚙️ Teknik Detay ve Sentaks
Standart kütüphanedeki tüm exception sınıfları polimorfiktir ve en tepede `<exception>` başlık dosyasında tanımlanan `std::exception` sınıfı bulunur.

**Hiyerarşinin Makro Düzeyde Yapısı:**
1.  **Operatör Seviyesindeki Hatalar:**
    *   `std::bad_alloc`: `new` operatörü başarısız olduğunda.
    *   `std::bad_cast`: `dynamic_cast` referans türleri ile başarısız olduğunda.
    *   `std::bad_typeid`: `typeid` operatörü null bir pointer'ı dereference ettiğinde.
2.  **Üye Fonksiyon Hataları:**
    *   `std::string`, `std::vector` gibi konteynırların üye fonksiyonları (Örn: `at()`).
    *   C++17 ile gelen `std::variant`, `std::optional`, `std::any` sınıflarının fırlattığı hatalar.

```cpp
#include <exception>
#include <stdexcept> // logic_error ve runtime_error burada

void example() {
    try {
        // Hiyerarşik yapı:
        // exception -> logic_error -> out_of_range
        // exception -> runtime_error
    }
    catch (const std::exception& ex) { // <-- Kritik: Referans semantiği ile yakalanmalı
        const char* msg = ex.what();   // Sanal fonksiyon: What()
    }
}
```

### 🔍 Arka Plan (Under the Hood)
`std::exception` sınıfının içindeki `what()` fonksiyonu `virtual` bir fonksiyondur. Bu sayede çalışma zamanında (Runtime) gerçek hata nesnesinin sunduğu hata mesajına ulaşılır.

### 🖼️ Görselleştirme (ASCII Art)
```text
      [std::exception] (Base)
             |
    --------------------------
    |                        |
[std::logic_error]     [std::runtime_error]
    |                        |
[std::out_of_range]    [std::overflow_error]
[std::invalid_argument]...
```

---

## 2. Exception Mekanizması ve Terminate (05:37 - 14:45)

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Yakalanmayan bir hata (Uncaught Exception), programın belirsiz bir durumda kalmasına neden olur. C++, bu durumda güvenli bir şekilde programı sonlandırmak için `std::terminate` mekanizmasını kullanır.

### ⚙️ Teknik Detay ve Sentaks
Eğer bir `throw` ifadesi uygun bir `catch` bloğu tarafından yakalanmazsa:
1.  `std::terminate()` çağrılır.
2.  `std::terminate` varsayılan olarak `std::abort()`'u çağırır.

**Terminate Handler'ı Özelleştirme:**
Hoca, hatanın yakalanamadığını daha iyi göstermek için `std::set_terminate` kullanacağını belirtti.

```cpp
#include <iostream>
#include <exception>

void my_handler() {
    std::cerr << "Yakalanamayan hata! Terminate cagirildi." << std::endl;
    std::abort(); // <-- Mutlaka abort veya exit ile bitmeli
}

int main() {
    std::set_terminate(my_handler); // Kendi handler'ımızı kaydettik
    throw 5; // Yakalayan yok -> Terminate çalışacak
}
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Bir call chain (çağrı zinciri) içinde hata nerede yakalanabilir?
**Cevap:** Hata, fırlatıldığı noktadan (F21) `main` fonksiyonuna kadar olan zincirdeki herhangi bir fonksiyonda (F20, F19... F1) yakalanabilir. Eğer fonksiyon içinde yakalanırsa, "Kol kırılır yen içinde kalır" mantığıyla çağırana yansımaz. Yakalanmazsa yukarı doğru **Propagate** (yayılmak/ilerlemek) eder.

---

## 3. Catch-All Bloğu ve Global Nesne Sınırlaması (14:45 - 21:23)

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Fırlatılan hatanın türünü bilmediğimiz veya türden bağımsız olarak mutlaka bir temizlik/loglama yapmak istediğimiz durumlarda "Catch-All" aracına ihtiyaç duyarız.

### ⚙️ Teknik Detay ve Sentaks
Sentaks: `catch(...)` (Elipsis/Üç nokta token'ı).

```cpp
void foo() {
    if (/*koşul*/) throw 10;
}

int main() {
    try {
        foo();
    }
    catch (int x) { /* ... */ }
    catch (...) { // <-- Catch-All: En sonda olmalı!
        std::cout << "Tur ne olursa olsun yakalarim." << std::endl;
        // Kritik: Burada hatanın türünü öğrenme imkanı YOKTUR.
    }
}
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `main` fonksiyonunu devasa bir `try-catch(...)` bloğuna alsak, programdaki TÜM exception'ları yakalamayı garanti edebilir miyiz?
**Cevap:** **HAYIR.** Global ve statik ömürlü (Static storage duration) nesnelerin constructor'ları `main` çalışmadan önce yürütülür. Bu constructor'lardan fırlatılan bir exception `main` içindeki blok tarafından yakalanamaz ve doğrudan `std::terminate` çağrılır.

---

## 4. Exception Handling Felsefesi ve Rethrow (21:23 - 44:49)

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Hata yakalandığında bazen hatanın tamamını değil, sadece bizi ilgilendiren kısmını (temizlik, loglama) yapıp, asıl kararı üst katmana bırakmak isteriz. Buna **Rethrow** denir.

### ⚙️ Teknik Detay ve Sentaks (Kritik Bölüm)
Hoca, `throw e;` ile `throw;` arasındaki devasa farka 15 dakika ayırdı.

**Hatalı Rethrow (`throw e;`):**
```cpp
catch (std::exception& ex) {
    std::cout << "Kismi müdahale yapildi." << std::endl;
    throw ex; // <-- HATA: Yeni bir kopyalama yapılır, slicing (dilimleme) oluşur.
}
```

**Doğru Rethrow (`throw;` - Naked Throw):**
```cpp
catch (std::exception& ex) {
    std::cout << "Adres: " << &ex << std::endl;
    throw; // <-- DOĞRU: Orijinal hata nesnesini yukarı gönderir.
}
```

### 🔍 Arka Plan (Under the Hood)
*   `throw ex;` ifadesi, catch parametresindeki nesneyi kullanarak **yeni bir exception nesnesi** kopyalar. Eğer polimorfik bir hata fırlatıldıysa (örn: `std::out_of_range`), catch bloğu `std::exception` ise, nesne kopyalanırken **Slicing** (Dilimleme) olur ve hatanın spesifik türü kaybolur.
*   `throw;` (naked throw) ise derleyicinin zaten tuttuğu orijinal exception nesnesini tekrar fırlatır. Adres aynı kalır, polimorfizm korunur.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `catch` bloğu dışında `throw;` kullanılırsa ne olur?
**Cevap:** Eğer ortada handle edilen (aktif) bir exception yokken "naked throw" yürütülürse doğrudan `std::terminate` çağrılır. Hoca bunu bir örnekle kanıtladı:

```cpp
void bar() {
    throw; // Yakalanan bir şey yokken rethrow!
}

int main() {
    bar(); // Sonuç: Terminate/Abort
}
```

### 🔗 Önceki Derslerle Bağlantı
Hoca, 12. derste gördüğümüz polimorfizm ve slicing konusuna atıf yaparak; `throw ex;` kullanımının neden `std::exception` catch parametrelerinde polimorfizmi bozduğunu açıkladı. "Dinamik türü kaybediyoruz" diyerek uyardı.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  Exception handling mekanizmasını "if-else" veya geleneksel dönüş değeri kontrolü gibi her yere yerleştirmek (Programın %90'ı try-catch olmamalı).
2.  Rethrow yaparken `throw ex;` kullanarak nesnenin polimorfik yapısını bozmak.
3.  Catch-all (`...`) bloğunu catch listesinin en başına koymak (Derleyici hatası).

Necati Ergin'in Exception Handling dersinin derinlemesine incelemesine devam ediyoruz. Bu bölümde, modern C++'ın en kritik konularından biri olan "Exception Safety" (İstisna Güvenliği) ve "Stack Unwinding" (Stack Çözülmesi) konularını inşa edeceğiz.

---

# C++ Exception Handling - Derinlemesine İnceleme (Bölüm 2)

## 5. Exception Safety Garantileri (Exception Safety Guarantees) (44:49 - 56:24)

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Hata yakalandığında programın çalışmaya devam etmesi (resumptive approach) yeterli değildir. Eğer hata sırasında program kaynak sızdırdıysa veya nesnelerin tutarlılığı (invariant) bozulduysa, program "Invalid State" (Geçersiz Durum) içine düşer. Bu durum, programın çökmesinden daha tehlikelidir çünkü yanlış verilerle çalışmaya devam eder.

### ⚙️ Teknik Detay ve Sentaks
Hoca, exception güvenliğini üç ana kategoriye ayırdı:

1.  **Basic Guarantee (Temel Garanti):**
    *   En temel seviyedir. Her fonksiyon bunu sağlamalıdır.
    *   **Kural:** Kaynak sızıntısı (Resource Leak) olmayacak ve program geçerli bir durumda (Valid State) kalacak. Nesnelerin invariant'ları bozulmamış olacak.

2.  **Strong Guarantee (Güçlü Garanti):**
    *   **İdiom:** "Commit or Rollback" (Ya yap ya eski haline dön).
    *   **Kural:** Eğer işlem başarılı olursa tamamlanır (Commit). Eğer exception fırlatılırsa, programın durumu işlemin çağrılmasından hemen önceki haline geri döner (Rollback).
    *   **Örnek:** `std::vector::push_back`. Eğer ekleme sırasında bellek yetmezse, vector eski halini korur.

3.  **No Fail / No Throw Guarantee:**
    *   **Kural:** Fonksiyonun işini yapacağı ve asla exception fırlatmayacağı garantisidir.
    *   **Sentaks:** Modern C++'da `noexcept` anahtar sözcüğü ile belirtilir.

```cpp
// Basic Guarantee Örneği (Hatalı Kod)
void foo(int n) {
    int* ptr = new int[n]; // Bellek allocate edildi
    bar(); // <-- Eğer bar() throw ederse, ptr asla delete edilmez!
    delete[] ptr; // Resource Leak! Basic guarantee bozuldu.
}
```

### 🔍 Arka Plan (Under the Hood)
Strong guarantee sağlamak genellikle maliyetlidir. Hoca'nın belirttiği teknik: Önce değişikliği geçici bir nesne üzerinde yapıp, hata oluşmadığından emin olduktan sonra `std::swap` gibi bir "no-throw" işlemiyle asıl nesneye yansıtmaktır.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** "Resource Leak" ve "Memory Leak" arasındaki fark nedir?
**Hoca'nın vurgusu:** Memory leak sadece belleğin geri verilmemesidir. Resource leak; açık kalmış bir dosya handle'ı (FILE*), kilitlenmiş bir mutex veya kapatılmamış bir network soketi olabilir. Hepsi resource leak kapsamındadır.

---

## 6. Stack Unwinding ve RAII İdiomu (56:24 - 01:16:00)

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Manuel kaynak yönetimi (`new/delete`, `fopen/fclose`) exception handling ile birleştiğinde neredeyse imkansız hale gelir. Programın akışı beklenmedik bir anda kesilip üst katmanlara sıçradığında, arada kalan temizlik kodları atlanır. C++, bu sorunu **Stack Unwinding** mekanizması ve **RAII** ile çözer.

### ⚙️ Teknik Detay ve Sentaks
**Stack Unwinding (Stack Çözülmesi):** Exception fırlatıldığı noktadan yakalandığı (catch) noktaya kadar, arada kalan tüm fonksiyonlardaki **otomatik ömürlü** (local) nesnelerin destructor'larının sistem tarafından çağrılması sürecidir.

**RAII (Resource Acquisition Is Initialization):** Kaynak yönetimini nesne ömrüne bağlamaktır.

```cpp
#include <iostream>
#include <string>
#include <memory>

class Person {
    std::string m_name;
public:
    Person(const char* name) : m_name(name) {
        std::cout << m_name << " icin kaynak edinildi\n";
    }
    ~Person() {
        std::cout << m_name << " icin kaynak geri verildi\n";
    }
};

void bas() {
    Person p3("Necati");
    throw std::runtime_error("Hata oluştu!"); // <-- Stack Unwinding burada baslar
}

void bar() {
    Person p2("Alihan");
    bas();
}

void foo() {
    Person p1("Oguzhan");
    bar();
}

int main() {
    try {
        foo();
    } catch (const std::exception& ex) {
        std::cout << "Hata yakalandi: " << ex.what() << "\n";
        // CIKTI SIRASI: Necati geri verildi -> Alihan geri verildi -> Oguzhan geri verildi
    }
}
```

### 🔍 Arka Plan (Under the Hood)
Hoca, `std::unique_ptr` örneğiyle konuyu derinleştirdi:
Eğer `Person* p = new Person("Abdulmuttalip");` derseniz ve exception oluşursa, pointer bir **scalar** (ilkel tür) olduğu için destructor'ı yoktur, bellek sızar. Ama `std::unique_ptr<Person>` kullanırsanız, `unique_ptr` bir sınıf nesnesi olduğu için stack unwinding sırasında destructor'ı çağrılır ve içindeki `Person` nesnesini `delete` eder.

### 🖼️ Görselleştirme (ASCII Art - Stack Unwinding)
```text
[main try block] <-- Catch burada
      |
  [foo frame]  { Person p1 }  ^  Destructor çağrılır (3)
      |                       |
  [bar frame]  { Person p2 }  ^  Destructor çağrılır (2)
      |                       |
  [bas frame]  { Person p3 }  ^  Destructor çağrılır (1)
      |
   THROW! ----> Exception Nesnesi oluşturulur.
```

---

## 7. Constructor ve Exception İlişkisi (01:16:00 - 01:35:14)

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir constructor işini yapamazsa (invariant'ları kuramazsa), o nesne hayata gelmiş sayılmaz. C++'da constructor'ların geri dönüş değeri olmadığı için, başarısızlığı bildirmenin tek sağlıklı yolu exception throw etmektir.

### ⚙️ Teknik Detay ve Sentaks (Kritik Kurallar)
Hoca bu bölümde hayati bir kuralın altını çizdi: **"Tamamlanmamış bir nesnenin destructor'ı çağrılmaz!"**

**Kritik Deney (01:19:00):**
```cpp
class MyClass {
public:
    MyClass() {
        std::cout << "Constructor basladi\n";
        throw std::runtime_error("Hata!"); // Constructor bitmedi!
    }
    ~MyClass() {
        std::cout << "Destructor cagirildi\n"; // <-- ASLA CALISMAZ
    }
};
```

**Member Nesnelerin Durumu:**
Eğer bir sınıfın içinde başka sınıf nesneleri (member) varsa ve constructor'ın ana bloğunda (body) hata oluşursa:
*   Body'den önce başarıyla construct edilmiş olan member nesnelerin destructor'ları **çağrılır.**
*   Sınıfın kendi destructor'ı **çağrılmaz.**

```cpp
class Member {
public:
    Member() { std::cout << "Member construct\n"; }
    ~Member() { std::cout << "Member destruct\n"; } // Bu çağrılır!
};

class Niche {
    Member m1; 
public:
    Niche() {
        throw 1; // m1 olustu ama Niche olusmadi
    }
    ~Niche() { /* Cagrilmaz */ }
};
```

### 🚩 Mülakat Sorusu / Kritik Nokta
**Soru:** Dinamik bir nesne (new ile oluşturulan) constructor'ında throw ederse bellek sızar mı?
**Cevap (Hoca'nın deneyi 01:26:00):** Hayır! Eğer `new MyClass` ifadesinde constructor exception fırlatırsa:
1.  Nesne oluşmadığı için destructor çağrılmaz.
2.  **ANCAK**, `operator new` ile tahsis edilen bellek alanı, sistem tarafından otomatik olarak `operator delete` çağrılarak geri verilir. Hoca bunu `operator new/delete` fonksiyonlarını overload ederek kanıtladı.

---

### 🔄 Adım Adım İzleme (10 Dakikalık Özetler)
*   **[45:00 - 55:00]:** Exception safety seviyeleri (Basic, Strong, No Fail) tanımlandı. `commit or rollback` kavramı açıklandı.
*   **[55:00 - 01:05:00]:** RAII idiomunun memory leak ve resource leak'i nasıl engellediği gösterildi. `FILE*` örneği üzerinden temizlik sorumluluğu nesneye yüklendi.
*   **[01:05:00 - 01:15:00]:** Stack Unwinding mekanizması canlı kodla izlendi. Nesnelerin fırlatılan noktadan geriye doğru "LIFO" (Last In First Out) sırasıyla yok edildiği ispatlandı.
*   **[01:15:00 - 01:35:14]:** Constructor'dan throw etmenin "nesne hiç oluşmadı" anlamına geldiği, bu yüzden destructor'ın çağrılmadığı ancak akıllı pointerlar veya RAII memberlar sayesinde kaynakların yine de kurtarılabileceği gösterildi.

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  Constructor içinde `new` ile kaynak alıp, RAII kullanmayıp, body içinde throw etmek (Destructor çağrılmayacağı için sızıntı olur!).
2.  Scalar (yalın) pointer'ların stack unwinding sırasında otomatik `delete` edileceğini sanmak (Edilmezler, akıllı pointer şart!).
3.  Hata durumunda programı "Invalid State" içinde bırakmak (Basic guarantee ihlali).

Necati Ergin'in Exception Handling dersinin üçüncü ve son bölümüne geçiyoruz. Bu bölümde, C++'ın en "egzotik" sentakslarından biri olan **Function Try Block** ve modern C++'ın hata yönetimindeki kalbi olan **noexcept** dünyasını inşa edeceğiz.

---

# C++ Exception Handling - Derinlemesine İnceleme (Bölüm 3)

## 8. Function Try Block (01:35:14 - 01:58:30)

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir sınıfın constructor'ı içinde normal bir `try-catch` bloğu kullanarak constructor gövdesindeki (body) hataları yakalayabiliriz. Ancak, **Constructor Initializer List** kısmında bir member (eleman) veya base class (taban sınıf) construct edilirken bir exception fırlatılırsa, bu hata constructor gövdesine hiç girilmediği için oradaki `try` bloğu tarafından yakalanamaz. İşte bu "başlangıç listesi" hatalarına müdahale edebilmek için **Function Try Block** aracı getirilmiştir.

### ⚙️ Teknik Detay ve Sentaks
Function try block, fonksiyonun gövdesini (`{ ... }`) tamamen kapsayan bir `try` bloğudur.

**Constructor Üzerinde Kullanımı (En Önemli Senaryo):**
```cpp
class Member {
public:
    Member(int x) { if (x == 0) throw std::runtime_error("Member hatasi!"); }
};

class MyClass {
    Member mx;
public:
    // Function Try Block Sentaksı:
    MyClass(int val) try : mx(val) // <-- try keyword'ü : operatöründen önce!
    {
        // Constructor Body
    } 
    catch (const std::exception& ex) {
        std::cout << "Initializer list hatasi yakalandi: " << ex.what() << "\n";
        // <-- KRİTİK KURAL: Burada gizli bir 'throw;' (rethrow) vardır!
    }
};
```

### 🔍 Arka Plan (Under the Hood)
Hoca, bu yapının normal fonksiyonlar için de kullanılabileceğini ancak bunun çok nadir (ve pek anlamlı olmayan) bir kullanım olduğunu belirtti:
```cpp
void normal_func() try {
    // kodlar
} catch(...) {
    // hata yönetimi
}
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Constructor'da bir Function Try Block içinde hatayı yakalayıp "yutabilir" miyiz (yani hata yokmuş gibi devam edebilir miyiz)?
**Cevap (Hoca'nın vurgusu - 01:53:00):** **KESİNLİKLE HAYIR.** Constructor'daki function try block catch bloğuna girdiğinde, derleyici catch bloğunun sonuna otomatik olarak bir `rethrow` (fırlatılan hatayı tekrar fırlat) ekler.
*   **Nedeni:** Eğer bir member construct edilemediyse, nesne eksik kalmıştır. Eksik (incomplete) bir nesnenin hayata devam etmesi lojik bir felakettir. Bu yüzden exception mutlaka yukarı (çağırana) iletilmek zorundadır.

---

## 9. Exception Specification ve `noexcept` Dünyası (01:58:30 - 02:15:00)

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Eski C++ (Modern öncesi), bir fonksiyonun hangi türden hatalar fırlatabileceğini `throw(std::bad_alloc, std::out_of_range)` şeklinde listeleyen bir yapıya sahipti. Ancak bu yapı hem derleyiciler için ağır bir yük oluşturuyor hem de optimizasyonları engelliyordu. Modern C++ ile bu kaldırıldı ve yerine "Ya hata fırlatır ya fırlatmaz" mantığına dayanan çok daha verimli olan `noexcept` sistemi getirildi.

### ⚙️ Teknik Detay ve Sentaks
`noexcept` anahtar sözcüğünün iki farklı yüzü vardır:
1.  **Specifier (Belirleyici):** Fonksiyonun exception fırlatmayacağını taahhüt eder.
2.  **Operator (İşleç):** Bir ifadenin exception fırlatıp fırlatmayacağını compile-time'da sorgular.

```cpp
// 1. Specifier Kullanımı:
void safe_func() noexcept;      // Exception fırlatmaz (No-fail guarantee)
void unsafe_func();             // Exception fırlatabilir (Varsayılan)
void conditional() noexcept(true); // safe_func ile aynı
```

### 📊 Standart Karşılaştırması
| Özellik | C++98 / C++03 | C++11 / 14 / 17 / 20 |
| :--- | :--- | :--- |
| Spesifik Liste | `void f() throw(int);` | Kaldırıldı (Removed) |
| No-throw Bildirimi | `void f() throw();` | `void f() noexcept;` |
| Hata İhlali Durumu | `std::unexpected` çağrılır | `std::terminate` çağrılır |

---

## 10. `noexcept` Operatörü ve Unevaluated Context (02:15:00 - 02:44:11)

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Özellikle jenerik programlamada (template), bir işlemin (örneğin iki nesneyi toplamak veya kopyalamak) güvenli olup olmadığını derleme zamanında bilmek isteriz. Buna göre farklı kod patikaları seçebiliriz.

### ⚙️ Teknik Detay ve Sentaks
`noexcept(expression)` operatörü, içine yazılan ifadeyi **çalıştırmaz** (Unevaluated Context), sadece o ifadenin `noexcept` garantisi altında olup olmadığını `bool` sabit olarak döner.

```cpp
void foo() noexcept;
void bar();

constexpr bool b1 = noexcept(foo()); // true
constexpr bool b2 = noexcept(bar()); // false
constexpr bool b3 = noexcept(1 + 1); // true (Tam sayı toplama hata fırlatmaz)
```

### 🔍 Arka Plan (Under the Hood / Unevaluated Context)
Hoca, `noexcept` operatörünün tıpkı `sizeof`, `typeid` ve `decltype` gibi bir **Unevaluated Context** (Değerlendirilmeyen Bağlam) oluşturduğuna dikkat çekti.
*   **Örnek (02:20:00):** `noexcept(x++)` yazdığınızda `x` değişkeninin değeri gerçekte artmaz. Derleyici sadece "x++ işlemi exception fırlatabilir mi?" diye bakar.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `noexcept` garantisi veren bir fonksiyonun içinde `throw` yapılırsa ne olur?
**Cevap (02:41:00):** Derleyici genellikle buna engel olmaz (çünkü hata derinde bir yerden geliyor olabilir). Ancak çalışma zamanında o `throw` yürütülürse, mekanizma `catch` bloklarını aramaz; doğrudan `std::terminate()` fonksiyonunu çağırarak programı acımasızca sonlandırır.

---

### 🔄 Adım Adım İzleme (10 Dakikalık Özetler)
*   **[01:35:00 - 01:45:00]:** Constructor initializer list'teki hataların normal `try` ile yakalanamadığı gösterildi. Function try block sentaksı tanıtıldı.
*   **[01:45:00 - 01:55:00]:** Constructor'da function try block kullanıldığında nesnenin "oluşmamış" sayıldığı ve hatanın otomatik rethrow edildiği ispatlandı.
*   **[01:55:00 - 02:05:00]:** Eski tip `throw(X)` listelerinin neden başarısız olduğu ve Modern C++ ile neden `noexcept`'e geçildiği anlatıldı.
*   **[02:05:00 - 02:20:00]:** `noexcept`'in hem bir belirleyici (specifier) hem de bir operatör olduğu ayrımı yapıldı. `true/false` değerlerine göre koşullu noexcept kavramı gösterildi.
*   **[02:20:00 - 02:44:11]:** Unevaluated context kavramı derinleştirildi. `noexcept` ihlallerinin programı `terminate`'e götürdüğü canlı kodla gösterilerek ders sonlandırıldı.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  Constructor function try block içinde hatayı yakalayıp rethrow etmemeye çalışmak (Zaten derleyici otomatik yapıyor, engelleyemezsiniz!).
2.  `noexcept` specifier'ı ile `noexcept` operator'ü karıştırmak (Biri fonksiyona mühür basar, diğeri ifadeyi sorgular).
3.  `noexcept` garantisi verilen fonksiyondan hata sızmasına göz yummak (Programın anında sonlanmasına neden olur).



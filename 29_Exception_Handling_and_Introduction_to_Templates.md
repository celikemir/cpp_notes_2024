Hoş geldin meslektaşım. Necati Hoca'nın 29. dersinde, Exception Handling (Hata Yakalama) mekanizmasının en kritik virajlarından biri olan `noexcept` dünyasının derinliklerine indik ve Generic Programming (Jenerik Programlama) kapısını araladık. Notlarım eksiksiz, Hoca'nın her vurgusu ve "mülakat sorusu" dediği her nokta burada.

---

### 1. `noexcept` Garantisinin Çiğnenmesi (Violation) ve `std::terminate`
**[01:23 - 05:21]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Bir fonksiyon `noexcept` (exception göndermeme) sözü verip bu sözü tutmazsa, bu durum C++'ın "hata güvenliği" (exception safety) felsefesiyle çelişir. Hoca'nın deyimiyle: *"Eğer bu söz çiğnenirse, bunun bedeli programın sonlanmasıdır."*

⚙️ **Teknik Detay ve Sentaks:**
Derleyici, bir `noexcept` fonksiyonun içinde hata fırlatılıp fırlatılmadığını Compile Time'da (derleme zamanında) tam olarak kontrol edemez. Çünkü çağrılan başka fonksiyonların tanımlarını görmeyebilir.

```cpp
#include <iostream>
#include <exception>

void bar() {
    throw std::runtime_error("bar hata firlatti"); // <-- Hoca: Burasi dinamik bir hata.
}

void foo() noexcept { // <-- Kritik kural: Hoca exception donmeyecek garantisi verdi.
    bar(); // <-- HATA BURADA: noexcept fonksiyon hata firlatan bar'i cagiriyor.
}

void MyTerminate() {
    std::cout << "MyTerminate cagirildi! Program sonlandiriliyor...\n";
    std::abort();
}

int main() {
    std::set_terminate(MyTerminate); // <-- Hoca: Terminate mekanizmasini ozellestirdik.
    try {
        foo();
    }
    catch (...) {
        std::cout << "Bu yakalanmayacak!"; // <-- Kritik: Terminate catch'e firsat vermez.
    }
}
```

🔍 **Arka Plan (Under the Hood):**
`noexcept` ihlal edildiğinde `std::terminate` çağrılır. `std::terminate` ise varsayılan olarak `std::abort` fonksiyonunu çalıştırır. Stack Unwinding (yığın boşaltma) işlemi bu noktada garanti edilmez; derleyici optimizasyon için yığını temizlemeden programı öldürebilir.

🚩 **Kritik Nokta:**
"Hata yakalanacak mı?" sorusunun yanıtı kesinlikle **HAYIR**. `noexcept` sözü verildiği an, o fonksiyonun dışına hatanın sızmasına izin verilmez.

---

### 2. Destructor'lar ve `noexcept` İstisnası
**[05:21 - 10:11]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Nesne yok edilirken hata fırlatmak, C++'ın en tehlikeli hamlelerinden biridir. Eğer bir hata zaten handle (yönetiliyor) ediliyorsa ve Stack Unwinding sırasında bir Destructor da hata fırlatırsa, C++ çalışma zamanı mekanizması çöker.

⚙️ **Teknik Detay ve Sentaks:**
Modern C++'da Destructor'lar **default olarak `noexcept` (true)** kabul edilir. Siz yazmasanız bile oradadırlar.

```cpp
class Nage {
public:
    ~Nage() { // <-- Hoca buraya dikkat cekti: Gizli noexcept var!
        // throw std::runtime_error("Eyvah!"); // <-- Bunu yaparsan Terminate cagirilir!
    }
};

// Hoca'nin Unevaluated Context ornegi:
constexpr auto b = noexcept(std::declval<Nage>().~Nage()); // b true doner.
```

🖼️ **Görselleştirme (ASCII Art):**
Stack Unwinding Sırasında Felaket Senaryosu:
```text
[Stack Frame: main]
   [Stack Frame: processData] <-- Hata firlatildi!
      [Unwinding...]
         [Object X Destructor] <-- Hata firlatti! 
            FATAL: C++ Runtime iki hatayi ayni anda yonetemez -> std::terminate()
```

🔗 **Önceki Derslerle Bağlantı:**
Hoca, 28. derste bahsettiği "Constructor'dan hata fırlatılabilir" kuralını hatırlattı. Constructor hata fırlatırsa nesne hiç oluşmamış sayılır, bu kabul edilebilirdir. Ancak Destructor hata sızdırmamalıdır.

---

### 3. `noexcept` ve Liskov Substitution Principle (LSP)
**[11:44 - 18:58]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Polimorfik bir yapıda (kalıtım), türemiş sınıfın (derived class), taban sınıfın (base class) verdiği sözleri tutması gerekir. Hoca Barbara Liskov'u anarak şunu söyledi: *"Derived class, client'lardan (kullanıcı kod) taban sınıftan daha fazlasını talep etmeyecek ve vaatlerinden daha azını vaat etmeyecek."*

⚙️ **Teknik Detay ve Sentaks:**
Taban sınıftaki bir `virtual` fonksiyon `noexcept` ise, onu `override` eden tüm fonksiyonlar da `noexcept` olmak zorundadır.

```cpp
class Base {
public:
    virtual void func() noexcept; // Taban sinif soz verdi.
};

class Derived : public Base {
public:
    // virtual void func() override; // <-- DERLEYICI HATASI: "less restrictive exception specification"
    virtual void func() noexcept override; // <-- DOGRUSU: Sozu tutmak zorundasin.
};
```

📊 **Standart Karşılaştırması:**
| Özellik | C++98 / 03 | Modern C++ (11/14/17/20) |
| :--- | :--- | :--- |
| **Exception Specification** | `throw(Type)` (Deprecated) | `noexcept` (Specifier) |
| **Destructor Default** | Throw edebilir (Riskli) | Implicit `noexcept(true)` |
| **Sanal Fonksiyon Kuralı** | Belirsiz kısıtlamalar | Sert LSP uyumu |

---

### 4. Fonksiyon Göstericileri (Function Pointers) ve `noexcept`
**[21:44 - 26:53]**

🔍 **Arka Plan (Under the Hood):**
C++17 ile birlikte `noexcept`, fonksiyonun tip sisteminin (type system) bir parçası haline geldi. Bu, tür güvenliğini artırır.

⚙️ **Teknik Detay ve Sentaks:**
Hata fırlatma ihtimali olan bir fonksiyonun adresi, `noexcept` sözü veren bir fonksiyon göstericisine atanamaz.

```cpp
void light() noexcept;
void heavy(); // Throw edebilir

void (*fp_safe)() noexcept; // Guvenli pointer
void (*fp_unsafe)();        // Guvensiz pointer

int main() {
    fp_unsafe = light; // Gecerli: Guvenli olani guvensiz yerde kullanabilirsin.
    // fp_safe = heavy; // <-- DERLEYICI HATASI: Guvensiz olani guvenli diye yutturamazsin!
}
```

🚩 **Mülakat Sorusu / Kritik Nokta:**
**Soru:** `void foo() noexcept;` ve `void foo();` şeklinde iki fonksiyonu Overload edebilir miyiz?
**Cevap:** Hayır. Hoca'nın uyarısı: *"noexcept imzanın bir parçasıdır ama overloading amaçlı kullanılamaz."* Bu durum doğrudan Syntax hatasıdır.

---

### 5. Nicolai Josuttis Örneği: Koşullu `noexcept`
**[26:53 - 30:16]**

Hoca, ünlü yazar Josuttis'in kitabından çok teknik bir örnek paylaştı. Bu örnek, `noexcept`'in içindeki sabit ifadesinin (constant expression) miras (inheritance) üzerindeki etkisini gösteriyor.

```cpp
class B {
public:
    virtual void func() noexcept(sizeof(int) < 8); // Koşul: Int 8 byte'tan kucukse firlatmaz.
};

class D : public B {
public:
    // virtual void func() noexcept(sizeof(int) < 4) override; 
    // <-- HATA: Derived, Base'den daha az garanti veriyor (Int=4 durumunda Base firlatmaz derken Derived firlatirim diyor).
};
```

---

### 6. Real-World Benchmarking: `noexcept` Neden Hayat Kurtarır?
**[41:22 - 54:06]**

Hoca bu bölümde dersin en "ufuk açıcı" (eye-opener) örneğini verdi. `std::vector` reallocation (yeniden bellek tahsisi) sırasında `noexcept` olup olmamanın maliyeti.

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
`std::vector` kapasitesi dolduğunda yeni bir yere taşınır. Eğer nesnenizin Move Constructor'ı `noexcept` değilse, `vector` hata güvenliğini (Strong Exception Guarantee) korumak için nesneleri taşımak (move) yerine kopyalamayı (copy) seçer. Bu da devasa performans kaybıdır.

⚙️ **Teknik Detay ve Kod Rekonstrüksiyonu:**

```cpp
#include <vector>
#include <chrono>
#include <string>

class Nage {
public:
    std::string ms;
    Nage() : ms(2000, 'A') {} // 2000 karakterlik ağır string.

    Nage(const Nage& other) : ms(other.ms) {
        // Copy Constructor: Maliyetli dinamik allocation yapar.
    }

    // Hoca: Buradaki 'noexcept' her seyi degistiriyor!
    Nage(Nage&& other) noexcept : ms(std::move(other.ms)) {
        // Move Constructor: Sadece pointer takasi yapar.
    }
};

int main() {
    std::vector<Nage> vec(100'000);
    // ... Zaman olcumu baslatildi ...
    vec.resize(100'001); // <-- Kritik: Reallocation tetiklendi.
    // ... Zaman olcumu bitirildi ...
}
```

📊 **Benchmark Sonuçları (Hoca'nın Ekranından):**
- **Move Constructor `noexcept` DEĞİLSE:** ~166.9 ms (Kopyalama yapıldığı için).
- **Move Constructor `noexcept` İSE:** ~0.5 ms (Sadece pointerlar taşındığı için).

🚩 **Mülakat Sorusu:**
*"Neden Move Constructor'ları her zaman noexcept yapmalıyız?"*
**Cevap:** Standart kütüphane konteynerleri (özellikle `std::vector`), nesneleri taşırken hata oluşursa eski veriyi koruyamaz. Eğer taşıma fonksiyonu `noexcept` değilse, kütüphane güvenlik için yavaş olan kopyalama mekanizmasına (Copy Semantics) geri döner (fallback).

---

Harika, Necati Hoca’nın standart kütüphane hiyerarşisine girdiği ve Exception Handling (Hata Yakalama) konusunu zirveye taşıyan "Idiom"lara geçtiği o kritik bölüme geldik. Hazırsan, notlarımıza kaldığımız yerden, en ince detayına kadar devam ediyoruz.

---

### 7. Standart Kütüphane Exception Hiyerarşisi
**[55:12 - 01:02:00]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Hata durumlarını standardize etmek ve polimorfik olarak yakalayabilmek için bir sınıf hiyerarşisine ihtiyaç vardır. Tüm standart hatalar tek bir kökten türetilmiştir.

⚙️ **Teknik Detay ve Sentaks:**
Standart kütüphanedeki hataların babası `std::exception` sınıfıdır. `<exception>` ve `<stdexcept>` başlık dosyaları bu işin merkezidir.

🖼️ **Görselleştirme (ASCII Art):**
```text
         [std::exception] (Base)
          /            \
 [std::logic_error]   [std::runtime_error]
   - out_of_range       - system_error
   - invalid_argument   - overflow_error
   - length_error       - range_error
```

🔍 **Arka Plan (Under the Hood):**
Hoca, `logic_error` ve `runtime_error` ayrımına özellikle değindi:
- **`std::logic_error`:** Programın mantığından kaynaklanır. Teorik olarak kod yazılırken %100 engellenebilir (Örn: Geçersiz indeks).
- **`std::runtime_error`:** Programın dışındaki faktörlerden kaynaklanır. Önceden kestirilmesi zordur (Örn: Bellek dolması, dosya bulunamaması).

🚩 **Kritik Nokta / Mülakat Sorusu:**
**Soru:** Kendi hata sınıfımızı yazarken nereden türetmeliyiz?
**Hoca'nın Yanıtı:** Genelde `std::exception` yerine `std::runtime_error` gibi sınıflardan türetmek daha yaygındır çünkü bu sınıflar `std::string` alan bir constructor'a sahiptir ve hata mesajını (what) kolayca yönetirler.

---

### 8. `std::string` ve Sık Karşılaşılan Hatalar
**[01:00:00 - 01:06:00]**

Necati Hoca, `std::string` üzerinden iki klasik hatayı canlı kodla simüle etti.

⚙️ **Teknik Detay ve Kod Rekonstrüksiyonu:**

**Örnek 1: `std::out_of_range`**
```cpp
#include <string>
#include <iostream>
#include <stdexcept>

int main() {
    std::string str = "necati ergin";
    try {
        // char c = str[36]; // <-- HATA: operator[] kontrol yapmaz, UB (Undefined Behavior)!
        char c = str.at(36); // <-- Hoca: 'at' fonksiyonu boundary check yapar ve firlatir.
    }
    catch (const std::out_of_range& ex) {
        std::cout << "Yakaladim: " << ex.what() << "\n";
    }
}
```

**Örnek 2: `std::length_error`**
```cpp
try {
    std::string s;
    // string'in kapasitesini max_size'in uzerine cikarmaya zorluyoruz
    s.assign(s.max_size() + 100, 'A'); 
}
catch (const std::length_error& ex) {
    std::cout << "Maksimum uzunluk asildi: " << ex.what() << "\n";
}
```

---

### 9. Dinamik Bellek ve `std::bad_alloc`
**[01:06:00 - 01:10:40]**

🔍 **Arka Plan (Under the Hood):**
`new` operatörü bellek tahsis edemediğinde (allocation failure) `NULL` dönmez (C tarzı `malloc` gibi değil). Modern C++'da `std::bad_alloc` fırlatır.

⚙️ **Kod Simülasyonu (Hoca'nın Denemesi):**
```cpp
#include <vector>

void MemoryTest() {
    std::vector<int*> vec;
    try {
        while(true) {
            // Hoca: Her dongude 1 GB'a yakin yer ayirip vektore ekleyerek rami sisiriyoruz.
            int* p = new int[1024 * 1024 * 250]; 
            vec.push_back(p);
            std::cout << "."; 
        }
    }
    catch (const std::bad_alloc& ex) {
        std::cerr << "\nSistemde bellek bitti: " << ex.what() << "\n";
    }
}
```

---

### 10. Modern C++ Bileşenlerinde Hata Yakalama (`variant`, `optional`, `stoi`)
**[01:11:00 - 01:19:00]**

Hoca, C++17 ile gelen bileşenlerin "güvenli" hata fırlatma mekanizmalarını gösterdi.

**A. `std::variant` ve `bad_variant_access`:**
```cpp
std::variant<int, float> var = 10; // Su an int tutuyor.
try {
    float f = std::get<float>(var); // <-- HATA: Icinde int varken float istedik.
} catch (const std::bad_variant_access& ex) {
    // Hoca: Union'larin ehlilestirilmis hali!
}
```

**B. `std::stoi` ve `std::invalid_argument`:**
```cpp
try {
    int val = std::stoi("necati"); // <-- HATA: Sayi degil ki donustursun!
} catch (const std::invalid_argument& ex) {
    // Sayisal karsiligi olmayan string hatasi.
}
```

**C. `std::optional` ve `bad_optional_access`:**
```cpp
std::optional<int> op; // Bos optional
try {
    int val = op.value(); // <-- HATA: Ici bosken degere erismeye calistik.
} catch (const std::bad_optional_access& ex) {
    // Hoca: optional bosken .value() firlatir, *op ise UB'dir!
}
```

🚩 **Kritik Nokta (Performans):**
Bir öğrenci sordu: *"Try-catch yavaşlatır mı?"*
Hoca'nın yanıtı net: **"Zero-cost exception model"**. Hata fırlatılmadığı sürece modern derleyicilerde `try-catch` bloklarının çalışma zamanı maliyeti sıfıra yakındır. Ancak hata fırlatıldığında (exceptional case) maliyet yüksektir.

---

### 11. Exception Konteksini Taşımak: `std::exception_ptr`
**[01:23:00 - 01:31:00]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Özellikle Multi-threading (Çok iş parçacıklı) programlamada, bir thread'de oluşan hatayı yakalayıp başka bir thread'e (örneğin main thread) güvenli bir şekilde taşımak gerekir. Hatanın polimorfik türünü kaybetmemek kritiktir.

⚙️ **Teknik Detay ve Sentaks:**
`std::current_exception()` ile hata yakalanır, bir `std::exception_ptr` içinde saklanır ve `std::rethrow_exception()` ile tekrar fırlatılır.

```cpp
#include <exception>

std::exception_ptr global_ptr; // Hoca: Bu pointer benzeri bir nesnedir.

void handleException(std::exception_ptr p) {
    if (p) { // Bos mu kontrolu (Nullable)
        try {
            std::rethrow_exception(p); // <-- Kritik: Saklanan hatayi tekrar firlatir.
        }
        catch (const std::exception& ex) {
            std::cout << "Baska baglamda yakaladim: " << ex.what() << "\n";
        }
    }
}

int main() {
    try {
        std::string().at(100); // Hata olustur.
    }
    catch (...) {
        global_ptr = std::current_exception(); // Hatayi dondur/sakla.
    }
    handleException(global_ptr);
}
```

---

### 12. İleri Seviye Idiomlar: "Polymorphic Exception" ve "Dispatcher"
**[01:31:00 - 01:48:00]**

Hoca burada mülakatların favori sorusuna geldi: **"Bir fonksiyonun parametresi taban sınıf olsa bile, fırlatırken dinamik türü nasıl koruruz?"**

**A. Polymorphic Exception Idiom (Virtual Raise):**
Dinamik türü (dynamic type) koruyarak hata fırlatmak için `virtual` bir fonksiyon kullanılır.

```cpp
class EBase {
public:
    virtual void raise() { throw *this; } // <-- Hoca: Sihirli dokunus burada!
    virtual ~EBase() = default;
};

class D1 : public EBase {
public:
    void raise() override { throw *this; } // Kendi turunden firlatir.
};

void foo(EBase& ex) {
    // throw ex; // <-- HATA: Slicing (dilimleme) olur, hep EBase firlar.
    ex.raise(); // <-- DOGRU: Sanal dispatch mekanizmasi D1::raise()'i cagirir!
}
```

**B. Exception Dispatcher Idiom:**
Aynı `catch` bloklarını defalarca yazmamak için kullanılır.

```cpp
void dispatcher() {
    try {
        throw; // <-- Hoca: 'Rethrow' ifadesi yakalanan hatayi buraya ceker.
    }
    catch (const NetException&) { /* ... */ }
    catch (const FileException&) { /* ... */ }
}

// Kullanim:
try { /* ... */ }
catch (...) {
    dispatcher(); // Tum catch mantigi tek merkezde.
}
```

---

### 13. Jenerik Programlamaya Giriş (The Template Gate)
**[01:54:00 - 02:14:00]**

Dersin sonunda Hoca, C++'ın en güçlü olduğu alana geçiş yaptı: **Generic Programming**.

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Aynı algoritmayı (Örn: `swap`, `sort`) farklı veri türleri (int, double, string) için tekrar tekrar yazmamak (DRY - Don't Repeat Yourself) için.

⚙️ **C-Style Generic (Void Pointers):**
Hoca, C'deki `qsort` ve `void*` tabanlı `swap` fonksiyonunu yazarak, C++ template'lerinin neden daha üstün olduğunu gösterdi (Type Safety eksikliği ve karmaşıklık).

```cpp
// Hoca'nin C usulu Generic Swap'i:
void g_swap(void* vp1, void* vp2, size_t sz) {
    char* p1 = (char*)vp1;
    char* p2 = (char*)vp2;
    while(sz--) {
        char temp = *p1;
        *p1++ = *p2;
        *p2++ = temp;
    }
}
```

🔍 **C++ Template Kategorileri:**
1.  **Function Template:** Fonksiyon kodu yazdırır.
2.  **Class Template:** Sınıf kodu yazdırır.
3.  **Variable Template (C++14):** Değişken tanımı yaptırır.
4.  **Alias Template:** Tür eş ismi (type alias) bildirimi yaptırır.

🚩 **Kritik Terimler:**
- **Instantiation (Açılım):** Şablondan somut kod üretilmesi.
- **Specialization (Özelleşme):** Üretilen o somut ürünün kendisi.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  `std::string::operator[]` ile sınır dışına çıkmanın hata fırlatmayacağını (Undefined Behavior), mutlaka `at()` kullanılması gerektiğini vurguladı.
2.  `throw ex;` yazmanın nesne dilimlemesine (slicing) yol açacağını, dinamik türü korumak için sanal `raise()` fonksiyonu (Polymorphic Exception) gerektiğini gösterdi.
3.  Move constructor'ın `noexcept` olmamasının `std::vector` gibi konteynerlerde performansı nasıl yerle bir ettiğini (Copy fallback) kanıtladı.



---

# 29. Ders: Pointer (Gösterici) Kavramına Giriş ve Temel Operatörler

### 1. Pointer Kavramı ve Statik Tür Sistemi
**[00:00 - 04:12]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Şimdiye kadar nesnelerin sadece **değerleriyle (value)** ilgilendik. Ancak sistem programlama ve donanıma yakın çalışırken, nesnenin nerede olduğu (adres) ve o adrese nasıl hükmedileceği önem kazanır. Soyutlama (abstraction) seviyesi düştükçe, adresler dilin birincil araçları haline gelir.

⚙️ **Teknik Detay ve Sentaks:**
Hoca'nın ilk uyarısı: *"Pointer sözcüğüne özel bir anlam yüklemeyin; pointer demek adres demektir."* C ve C++ adresleri statik tür sisteminde iki ana kategoriye ayırır:
1.  **Object Pointers (Nesne Göstericileri):** Değişkenlerin/nesnelerin adresleri.
2.  **Function Pointers (Fonksiyon Göstericileri):** Fonksiyonların giriş noktalarının adresleri.

🔍 **Arka Plan (Under the Hood):**
Yüksek seviyeli dillerde (Java, C# vb.) adres kavramı dil katmanından gizlenmiştir. C'de ise adresin dilde bir "varlık" olması, onu donanım resmini en iyi çizen dil yapar.

---

### 2. Pointer Aksiyomu ve Tür Bilgisi
**[06:00 - 14:30]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Eğer her adres aynı türden olsaydı, derleyici o adresteki verinin ne kadar yer kapladığını (byte sayısı) ve nasıl yorumlanacağını bilemezdi. Bu yüzden adresler, işaret ettikleri türle mühürlenmiştir.

⚙️ **Teknik Detay (Hoca'nın Aksiyomu):**
Hoca bunu bir matematiksel kesinlikle ifade etti:
> **"X, T türünden bir nesne ise; X'in adresi T* (T yıldız) türündendir."**

```cpp
int x = 5;      // Tür: int
// &x ifadesinin türü -> int* (Pointer to int)

double d = 4.5; // Tür: double
// &d ifadesinin türü -> double* (Pointer to double)

char c = 'A';   // Tür: char
// &c ifadesinin türü -> char* (Pointer to char)
```

🚩 **Kritik Nokta:** Farklı türden nesnelerin adresleri, farklı türden ifadelerdir. `int*` ile `double*` aynı tür değildir.

---

### 3. Pointer Bildirimleri ve "Deklaratör" Kavramı
**[17:35 - 21:50]**

🚩 **Mülakat Sorusu / Kritik Nokta:**
Hoca buraya "çok dikkat" dedi. Asterisk (`*`) sembolünün konumu sentaks açısından fark yaratmaz ama birleştirilmiş bildirimlerde (comma separated list) büyük bir tuzak vardır.

```cpp
int* p1, p2, p3; 
// Hoca: Bu kodda sadece p1 pointer'dır! 
// p2 ve p3 sadece int türündendir.
```
**Derleyici Gözü:** Asterisk (`*`) tür belirten sözcüğün (`int`) değil, **identifier (isim)** kısmının bir parçasıdır (yani bir **declarator**'dır).

**Doğru Yazım (Morality vs Legality):**
```cpp
int *p1, *p2, *p3; // Hepsi pointer oldu.
```
**Hoca'nın İdiomu:** *"Operatörle deklaratörü karıştırmayın."* Bildirimdeki `*` bir operatör değildir, o bir tür niteleyicidir.

---

### 4. Pointer'ların Bellek Alanı (Sizeof) ve Ömürleri
**[22:00 - 24:54]**

🔍 **Arka Plan (Memory Layout):**
Pointer bir değişkendir ve bellekte yer kaplar. Hoca kendi sisteminden örnek verdi:
- `char` (1 byte) -> `char*` (4 byte)
- `double` (8 byte) -> `double*` (4 byte)
- `long long` (8 byte) -> `long long*` (4 byte)

**Kritik Kural:** Gösterilen nesnenin büyüklüğü ne olursa olsun, bir **Object Pointer**'ın kapladığı yer (adres büyüklüğü) aynıdır (Genelde 32-bit sistemde 4, 64-bit'te 8 byte).

⚙️ **Ömür ve İlkleme (Initialization):**
- **Otomatik Ömürlü (Local) Pointer:** İlk değer verilmezse içi **garbage (çöp)** değer doludur. Çok tehlikelidir!
- **Statik Ömürlü (Global/Static) Pointer:** İlk değer verilmezse **NULL pointer** (sıfır adresi) ile başlar.

---

### 5. L-Value / R-Value ve Adres Operatörü (`&`)
**[58:45 - 01:12:00]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Bir nesnenin adresini alabilmek için o nesnenin bellekte sabit bir yerinin olması gerekir.

⚙️ **Constraint (Kısıtlama):**
`&` (Address of) operatörünün operandı mutlaka bir **L-value expression** (bellekte yer kaplayan nesne) olmalıdır.

```cpp
int x = 10;
int *p = &x;    // Gecerli: x bir L-value.
// int *p2 = &5;   // HATA: 5 bir R-value, adresi yok!
// int *p3 = &(x+5); // HATA: x+5 bir R-value (gecici nesne).
```

🚩 **Kritik Nokta:** Adres operatörüyle elde edilen ifadenin kendisi bir **R-value**'dur. Yani `&&x` gibi bir yazım ("adresin adresi") geçersizdir. Çünkü `&x` ifadesi bellekte bir yerde saklanmaz, anlık üretilir.

---

### 6. Array Decay (Dizinin Adrese Çökmesi)
**[01:13:00 - 01:16:58]**

Hoca, C'nin en karakteristik özelliklerinden birini hatırlattı:
**Kural:** Bir dizinin ismi bir ifade içinde kullanıldığında (istisnalar hariç), dizinin ilk elemanının adresine dönüşür.

```cpp
int a[10];
int *p = a; // Array Decay: 'a' ifadesi &a[0] türüne (int*) donustu.
```

---

### 7. C vs C++: Tür Katılığı (Strict Type Checking)
**[01:17:00 - 01:27:00]**

Necati Hoca bu bölümde "Legality vs Morality" (Yasallık vs Ahlakilik) vurgusu yaptı.

🚩 **Mülakat Sorusu:** Farklı türden bir adresi, başka türden bir pointer'a atayabilir miyiz?
```cpp
double d = 3.14;
int *p = &d; // <-- Hoca: C'de Warning (Legal ama Yanlis), C++'da ERROR (Illegal)!
```
**Derleyici Gözü:** C++'da tür güvenliği çok daha katıdır (`strict`). Farklı adres türleri arasında örtülü (`implicit`) dönüşüm yoktur.

---

### 8. Dereferencing (İçerik/Geriye Başvuru) Operatörü (`*`)
**[01:48:00 - 02:12:00]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Elimizde bir adres var (`pointer`). Bu adresteki veriye ulaşmak, onu okumak veya değiştirmek için bir "anahtar" gerekir. İşte bu anahtar `*` operatörüdür.

⚙️ **Teknik Detay ve İdiomlar:**
- **Pointer:** Gösteren.
- **Pointy:** Gösterilen nesne.
- **`*p` İfadesi:** Pointy'nin (gösterilen nesnenin) kendisidir.

```cpp
int x = 10;
int *p = &x; // p -> x'i gosteriyor.

*p = 99; // Hoca: "*p demek x demektir". X artik 99 oldu.
```

🔍 **Arka Plan (Under the Hood):**
`*p` ifadesi her zaman bir **L-value expression**'dır. Yani bir pointer'ı dereferans ettiğinizde, elinizde artık o adresteki gerçek nesne vardır.

---

### 9. Call By Reference (Pointer Semantiği)
**[02:30:00 - 02:41:00]**

Hoca dersi "Neden pointer kullanıyoruz?" sorusunun en büyük cevabıyla bitirdi.

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
C dilinde tüm fonksiyon çağrıları **Call by Value** (değerle çağırma) şeklindedir. Bir fonksiyonun, kendisini çağıran yerdeki bir değişkeni değiştirebilmesinin **tek yolu** o değişkenin adresini almasıdır.

⚙️ **Kod Rekonstrüksiyonu (Gerçek Swap Fonksiyonu):**
```cpp
void swap(int *p1, int *p2) { // Adresleri kopyaliyoruz
    int temp = *p1; // p1'in gosterdigi nesnenin (a) degerini al
    *p1 = *p2;      // p2'nin gosterdigi nesnenin (b) degerini p1'in gosterdigine (a) yaz
    *p2 = temp;     // b = temp
}

int main() {
    int a = 5, b = 10;
    swap(&a, &b); // Call by Reference (via Pointers)
}
```

🖼️ **Görselleştirme (ASCII Art):**
```text
[Main Frame]      [Swap Frame]
   a: 5   <-------- p1: &a
   b: 10  <-------- p2: &b
```

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  `int* p1, p2;` yazımında `p2`'nin pointer değil `int` olacağı tuzağına düşmeyin.
2.  Pointer'ın kendi adresi (`&ptr`) ile tuttuğu adresi (`ptr`) karıştırmayın.
3.  C'de `int *p = 100;` yazmak legaldir ama 100 nolu adrese erişmeye çalışmak **Morality** dışıdır ve muhtemelen `Runtime Error` verir.



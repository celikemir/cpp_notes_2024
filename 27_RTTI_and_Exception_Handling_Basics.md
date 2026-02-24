Bu teknik döküman, Necati Ergin'in 30 Eylül 2024 tarihli 27. dersinin transkriptinden hareketle, en ince ayrıntısına kadar yeniden inşa edilmiştir.

# C++ DERS NOTLARI: RTTI Derinlemesine Bakış ve Exception Handling Giriş

## BÖLÜM 1: [00:00 - 10:00] RTTI, `typeid` ve `std::type_info` İncelikleri

Dersin bu bölümünde, geçen dersin sonu olan RTTI (Runtime Type Identification - Çalışma Zamanı Tür Belirlenmesi) konusunun üzerinden geçilerek, `typeid` operatörünün statik ve dinamik türler üzerindeki etkisi tartışılmıştır.

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Polimorfik hiyerarşilerde, bir taban sınıf pointer'ının (veya referansının) gösterdiği gerçek (dinamik) nesnenin türünü çalışma zamanında öğrenme ihtiyacı doğmuştur. `dynamic_cast` sadece dönüşümün başarısını sınarken, `typeid` doğrudan tür nesnesine (`std::type_info`) erişim sağlar.

### ⚙️ Teknik Detay ve Sentaks
`typeid` operatörü hem bir tür ismiyle (`typeid(int)`) hem de bir ifadeyle (`typeid(x + y)`) kullanılabilir. Sonuç, `const std::type_info` türünden bir L-value (Sol taraf değeri) nesnesidir.

```cpp
#include <iostream>
#include <typeinfo> // <-- Mutlaka include edilmeli

class Base {
public:
    virtual ~Base() = default; // <-- Polimorfik yapmak için virtual destraktör
};

class Der : public Base {};

int main() {
    Der myder;
    Base* base_ptr = &myder;

    // r'nin türü: const std::type_info&
    auto& r = typeid(*base_ptr); // <-- Dinamik tür sorgulanıyor

    if (r == typeid(Der)) { // <-- operator== kullanımı
        std::cout << "Nesne gerçekten Der türünden!\n";
    }
}
```

### 🔍 Arka Plan (Under the Hood)
`std::type_info` sınıfı "copyable" veya "default constructible" değildir.
*   **Default Constructor:** Yoktur. `std::type_info tx;` // <-- HATA: Sentaks hatası.
*   **Copy Member Functions:** `delete` edilmiştir. `std::type_info y = typeid(x);` // <-- HATA: Kopyalama yapılamaz.
*   **Erişim Yolu:** Bir `type_info` nesnesine erişmenin tek yolu `typeid` operatörüdür.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `typeid` ne zaman statik, ne zaman dinamik tür bilgisi verir?
**Cevap:** 
1. Operant polimorfik olmayan bir sınıf türü ise: Her zaman **Static Type** (Statik tür) bilgisi verir.
2. Operant polimorfik bir sınıf türü ise: **Dynamic Type** (Dinamik tür) yani gerçek nesne bilgisi verir.

```cpp
class NonPolyBase {};
class NonPolyDer : public NonPolyBase {};

// ...
NonPolyDer d;
NonPolyBase* p = &d;
std::cout << (typeid(*p) == typeid(NonPolyDer)); // <-- FALSE! Statik tür (NonPolyBase) esas alınır.
```

---

## BÖLÜM 2: [10:00 - 20:00] `type_info` Üyeleri, Unevaluated Context ve `bad_typeid`

Bu bölümde `type_info` sınıfının üye fonksiyonları ve `typeid`'nin bir "Unevaluated Context" oluşturması konusu işlenmiştir.

### ⚙️ Teknik Detay ve Sentaks
*   **`name()`:** Türün ismini temsil eden `const char*` döndürür. Bu yazı derleyiciden derleyiciye değişebilir (C-lang/GCC'de kriptik, MSVC'de açıktır).
*   **`hash_code()`:** Tür için benzersiz bir `size_t` değer üretir. `unordered_set/map` gibi konteynerlerde hashing amaçlı kullanılır.
*   **`std::type_index`:** `type_info` nesnelerini konteynerlerde tutmak için bir wrapper (sarmalayıcı) sınıftır.

### 🔍 Arka Plan: Unevaluated Context (Yürütülmeyen Bağlam)
`typeid`, tıpkı `sizeof` ve `decltype` gibi, operantı olan ifadeyi çalışma zamanında yürütmez.

```cpp
int x = 5;
std::cout << typeid(++x).name(); // <-- x artmaz! Sadece int türü bilgisi alınır.
// x hala 5.
```

**İstisna:** Eğer operant polimorfik bir türün dereferans edilmiş pointer'ı ise (`typeid(*p)`), dinamik türü belirlemek için `p`'nin içindeki VPointer (Sanal tablo göstericisi) üzerinden tabloya bakılması gerekir.

### 🚩 Kritik Nokta: `std::bad_typeid`
Eğer polimorfik bir türü gösteren pointer `nullptr` ise ve biz bu pointer'ı dereferans edip `typeid`'ye sokarsak, bu durum UB (Undefined Behavior - Tanımsız Davranış) **değildir**. Standart kütüphane bu durumda `std::bad_typeid` exception'ını fırlatır.

```cpp
Base* p = nullptr; // Base polimorfik
try {
    std::cout << typeid(*p).name(); // <-- HATA: nullptr dereferans ama typeid içinde!
} catch (const std::bad_typeid& e) {
    std::cout << "bad_typeid yakalandı!"; // <-- Akış buraya girer.
}
```

---

## BÖLÜM 3: [20:00 - 30:00] `dynamic_cast` vs `typeid` Karşılaştırması ve Maliyet Analizi

Hoca, bu bölümde downcasting (aşağı doğru tür dönüşümü) işlemlerinde bu iki operatörün farklarını "Volvo" örneği üzerinden açıklamıştır.

### ⚙️ Teknik Detay: Tam Eşleşme (Exact Match)
`dynamic_cast`, hiyerarşide aşağıya doğru herhangi bir noktaya dönüşümü kabul ederken, `typeid` sadece **exact match** (tam eşleşme) arar.

```cpp
class Car { virtual void run(); };
class Volvo : public Car { public: void openSunroof(); };
class VolvoXC90 : public Volvo {};

void get_car(Car* p) {
    // p bir VolvoXC90 gösteriyor olsun.
    
    if (dynamic_cast<Volvo*>(p)) { // <-- DOĞRU: VolvoXC90 bir Volvo'dur.
        // ...
    }

    if (typeid(*p) == typeid(Volvo)) { // <-- YANLIŞ: Dinamik tür Volvo değil, VolvoXC90'dır.
        // Bu blok çalışmaz.
    }
}
```

### 🔍 Arka Plan (Cost Analysis)
*   **`typeid` Maliyeti:** Daha düşüktür. Genellikle VTable'ın (Sanal Fonksiyon Tablosu) 0. indeksinde tutulan bir `type_info` pointer'ına erişir.
*   **`dynamic_cast` Maliyeti:** Daha yüksektir. Çünkü sadece pointer'ı karşılaştırmaz, çalışma zamanında hiyerarşiyi (inheritance tree) "en dibe kadar" kontrol etmek zorundadır.

### 🖼️ Görselleştirme (ASCII Art - VTable Layout)
```text
Nesne (p) -> [VPointer] ----> [ VTable for Der ]
                              [ 0: &type_info  ] <-- typeid buraya bakar
                              [ 1: &Func1      ]
                              [ 2: &Func2      ]
```

---

## BÖLÜM 4: [30:00 - 40:51] "Zero Overhead" ve Compiler Switch'ler

C++'ın temel felsefesi olan "kullanmadığın şeyin maliyetini ödemezsin" ilkesinin RTTI üzerindeki etkisi.

### 🧠 Rationale: "Hotel Minibar" Analojisi
Necati Hoca, C++ ile Java/C# arasındaki farkı bir otel analojisiyle açıklar:
*   **Java/C#:** Otele girerken 150$ ödersiniz, çıkarken 300$ hesap gelir. "Kullanmasan da minibarın ve saunanın parasını aldık" derler. Her nesne başında bir tür bilgisi taşır.
*   **C++:** Sadece odanın parasını ödersiniz. Minibarı (RTTI) açarsanız (Polimorfizm eklerseniz), sadece o zaman maliyet başlar.

### ⚙️ Teknik Detay: RTTI'yi Kapatmak
Birçok derleyicide (MSVC, GCC) RTTI'yi tamamen kapatma switch'i (`-fno-rtti`) bulunur.
*   **Neden kapatılır?** Binary size (ikili dosya boyutu) ve bellek tasarrufu için.
*   **Ne etkilenir?** Sadece `typeid` ve `dynamic_cast` çalışmaz. Sanal fonksiyon mekanizması (Virtual Dispatch) etkilenmez, o hala çalışır.

---

Bu bölümde RTTI konusu tamamlanmış ve Hoca, kursun en kritik konularından biri olan **Exception Handling**'e giriş yapmıştır.

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1. Polimorfik olmayan türlerde `typeid`'nin dinamik tür vermesini beklemek.
2. `type_info` nesnesini kopyalamaya çalışmak.
3. `dynamic_cast` ile `typeid`'nin hiyerarşik kontrol farkını karıştırmak.

## BÖLÜM 5: [00:40:51 - 00:55:38] Exception Handling'e Giriş: Hataların Sınıflandırılması ve Assertion'lar

Exception Handling (İstisna İşleme) konusuna girerken Hoca'nın ilk vurguladığı nokta, "hata" (error) kavramının doğru anlaşılmasıdır. Hatalar temel olarak ikiye ayrılır: **Programming Errors** (Programlama hataları/Bug'lar) ve **Runtime Errors** (Çalışma zamanı hataları).

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir programcı yazdığı kodda bir mantık hatası yaptıysa (örneğin sıfıra bölme, geçersiz pointer kullanımı veya yanlış string formatlaması), bu bir *Exception Handling* konusu değildir. Programlama hatalarıyla başa çıkmanın tek doğru yolu o kodu bulup değiştirmek ve *recompile* (yeniden derleme) yapmaktır. Bu aşamada bizi koruyacak mekanizma **Assertion** (İddia/Doğrulama) yapılarıdır.

### ⚙️ Teknik Detay ve Sentaks (Assertion Türleri)
Assertion'lar ikiye ayrılır:
1.  **Static Assertions (Compile-Time):** Derleme zamanında kontrol edilen koşullar.
2.  **Dynamic Assertions (Run-Time):** Çalışma zamanında kontrol edilen koşullar (C'den gelen `assert` makrosu).

Hoca, C++'a `static_assert` gelmeden önce C'de derleme zamanı hatası ürettirmek için kullanılan efsanevi "Fakir Adamın Static Assert'i" hilesini gösterdi:

```cpp
// C'de static_assert yokken kullanılan hack (Negative/Zero Array Size)
// <-- Hoca buraya dikkat çekti: Derleyici 0 elemanlı diziye kızar.
typedef int error_check[sizeof(int) > 2 ? 1 : -1]; 
// Eğer sizeof(int) <= 2 ise, dizi boyutu -1 olur ve derleyici SENTAKS HATASI verir.

// Modern C++'ta:
static_assert(sizeof(int) > 2, "int boyutu 2'den buyuk olmali!"); // <-- Kullanılması gereken modern yol.
```

### 🔍 Arka Plan (Under the Hood / Preprocessor Layout)
Dynamic assertion için kullanılan `assert` makrosu, arka planda koşullu derleme (conditional compilation) ile çalışır.
Eğer `NDEBUG` (No Debug) makrosu tanımlanmamışsa, assert içindeki ifade sınanır (evaluated). İfade `false` ise standart hata akımına (`stderr`) dosya adı, satır numarası ve fonksiyon ismi yazılarak program `abort()` fonksiyonu ile anında acımasızca sonlandırılır.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Programlama hatalarını try-catch bloğu ile yakalamalı mıyız?
**Cevap:** Hayır! Kodun kendisinin yanlış olduğu durumlarda (Programming Errors) hata fırlatılmaz, kod düzeltilir. Exception Handling, kodun doğru olduğu ancak *dışsal/beklenmedik* durumlardan ötürü işin yapılamadığı (Runtime Errors) senaryolar için vardır.

---

## BÖLÜM 6: [00:55:38 - 01:13:30] Contract Programming: Precondition, Postcondition ve Invariants

Bir fonksiyonun "işini yapamaması" ne demektir? Bu soruyu cevaplamak için Yazılım Mühendisliğindeki **Contract Programming** (Sözleşmeli Programlama) kavramlarına girmek gerekir.

### 🧠 İhtiyaç (Rationale)
Fonksiyon ile onu çağıran kod (caller) arasında görünmez bir sözleşme vardır. Hata, bu sözleşmenin ihlal edilmesidir.

### ⚙️ Teknik Terimler ve Bağlam
1.  **Postconditions (Son Koşullar):** Fonksiyon çağrıldıktan sonra oluşması *garanti edilen* durumdur. (Örn: `std::vector::push_back` çağrıldığında `m_size` değerinin 1 artması). Sağlanamazsa bu kesinlikle bir fonksiyondan kaynaklı hatadır.
2.  **Preconditions (Ön Koşullar):** Fonksiyon çağrılmadan *önce* çağıran kodun (caller) sağlamak zorunda olduğu şartlardır. (Örn: `sqrt(x)` fonksiyonunda `x >= 0` olmalıdır).

**Precondition İhlali ve Contract Türleri:**
Eğer çağıran kod precondition'ı ihlal ederse iki yaklaşım sergilenebilir:
*   **Narrow Contract (Dar Sözleşme):** Fonksiyon precondition kontrolü yapmaz. Eğer şart sağlanmazsa ortaya çıkan durum UB (Tanımsız Davranış)'dir. Sorumluluk tamamen çağırandadır. (Çoğu C fonksiyonu böyledir).
*   **Wide/Broad Contract (Geniş Sözleşme):** Fonksiyon parametreyi kontrol eder. Eğer yanlışsa bir *Exception* fırlatarak durumu bildirir. (Örn: `std::string::insert(idx, str)` fonksiyonuna geçersiz bir `idx` verilirse, C++ standart kütüphanesi `std::out_of_range` exception'ı gönderir).

3.  **Invariants (Sınıf Değişmezleri):** Sınıfın public üye fonksiyonları çağrılmadan önce ve çağrıldıktan sonra nesnenin her zaman geçerli bir durumda (**Valid State**) kalmasıdır.

### 🔍 Arka Plan: "Zombie Objects" (Zombi Nesneler)
Bir sınıfın *constructor*'ı (kurucu işlevi) sınıf invariant'larını oluşturamazsa ve sistemde exception handling yoksa, elimizde geçerli bir durumda olmayan bir nesne kalır. Bu nesnelere literatürde **Zombie Object** (yaşayan ama ölü olan nesne) denir ve kullanımı felaketle sonuçlanır.

---

## BÖLÜM 7: [01:13:30 - 01:21:00] Geleneksel Hata İşleme (Traditional Error Handling) ve Dezavantajları

Exception Handling icat edilmeden önce veya C gibi dillerde hatalar nasıl ele alınıyordu? Hoca bu kısmı iki ana başlıkta inceledi.

### ⚙️ Teknik Detay ve Sentaks (Geleneksel Yöntemler)
1.  **Return Code (Geri Dönüş Değeri):** Fonksiyonun işini yapamadığını `-1`, `nullptr` veya `false` gibi sihirli/özel bir dönüş değeriyle bildirmesidir.
2.  **Global Error Variable (`errno`):** Fonksiyon hem bir değer döndürür hem de sistem genelindeki global bir bayrağı (`errno`) set eder.

```c
#include <stdio.h>
#include <errno.h>

void traditional_example() {
    FILE* f = fopen("not_exist.txt", "r");
    if (f == NULL) { // <-- 1. Geri dönüş değeri kontrolü
        perror("Cannot open file"); // <-- 2. errno okunarak hata mesajı yazdırılır.
    }
}
```

### 🧠 Neden Devrimci Bir Mekanizmaya İhtiyaç Duyuldu? (The Drawbacks)
Hoca, neden *Traditional Error Handling* yönteminin "başımızın belası" olduğunu üç maddeyle açıkladı:

1.  **Zorlayıcı Değildir (Not Forcing):** `malloc` sana `NULL` döndürür, ama sen o dönüş değerini kontrol etmezsen derleyici seni durdurmaz. Gider o `NULL` pointer'ı dereferans eder, programı patlatırsın. (Sessizce yayılan hatalar).
2.  **İş Gören Kod ile Hata Kodu İç İçe Geçer:** Fonksiyonun asıl işi yapan `Business Logic`'i, her satırda yapılan `if (error) return;` blokları yüzünden okunmaz hale gelir.
3.  **Test Süreçlerini Mahveder:** Her bir `if` deyimi, yazılması ve kapsanması (coverage) gereken yeni bir test durumu (test case) demektir.

---

## BÖLÜM 8: [01:21:00 - 01:38:12] Call Chain (Çağrı Zinciri) Problemi ve Exception'ın Doğuşu

Geleneksel hata işlemenin en çok tıkandığı nokta **Call Chain** (Çağrı Zinciri) ve geri dönüş değeri verilemeyen yerlerdir.

### 🖼️ Görselleştirme: Call Chain ve Geleneksel Yayılım
```text
[Main] -> [f1] -> [f2] -> ... -> [f18] -> [f19]
```
Eğer `f19` dosya açamazsa (hata), ve bu hataya çözüm bulacak kod `f1`'in içindeyse, geleneksel yöntemde `f19` hatayı `f18`'e, o `f17`'ye... kova taşır gibi elden ele iletmek zorundadır. Bu, aradaki tüm fonksiyonların dönüş değerlerinin hata kodlarıyla "kirletilmesi" anlamına gelir.

### 🧠 Exception Handling'in Getirdiği Devrim (Long Jump)
Exception Handling'in asıl devrimci tarafı şudur: Hata oluştuğunda, `f19` bir hata nesnesi oluşturur ve bunu fırlatır. Programın akışı **doğrudan** hatayı yakalamaya niyetli olan `f1`'e zıplar. Aradaki `f2`...`f18` tamamen bypass edilir (buna *Stack Unwinding* denileceğini ileride göreceğiz).

> *Hoca'nın Analojisi:* "Yangını söndürürken kovaların elden ele taşınması yerine, doğrudan itfaiyenin hedefe uçmasıdır."

### 🚩 Kritik Nokta: Geri Dönüş Değeri Olmayan Yapılar
Neden sadece `if-return` ile yaşayamayız?
1.  **Constructor'lar:** Kurucu işlevlerin geri dönüş değeri yoktur. `errno` veya `out_parameter` (pointer referans ile set etmek) kullanmak kodu çirkinleştirir.
2.  **Operator Overloading:** `s1 + s2` yaparken hata çıkarsa `+` operatörü ne döndürecek? Geri dönüş değeri zaten bir nesne (örneğin `std::string`) olmak zorunda. O nesne içine hata gömemeyiz.
3.  **Bütün Değerlerin Geçerli Olduğu Durumlar:** C'deki `atoi` fonksiyonunu düşünün. String içindeki sayıyı döner. Yazı "Necati" ise `0` döner. Peki yazı zaten `"0"` ise? Yine `0` döner. Hangisinin hata olduğunu dönüş değerinden anlayamayız!

Bu durumlarda mecburen `struct { int val; int error_flag; }` döndürmek (C tarzı) veya `std::pair` kullanmak zorundayız, ki bu da sentaksı çok yorar.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1. Bug'ları (Programming Errors) exception try-catch ile örtbas etmeye çalışmak.
2. Constructor'ların hata durumunda valid olmayan (zombi) nesneler bırakması tehlikesi.
3. Return code yaklaşımında programcının hata kontrolünü unutmasına sistemin izin vermesi.

Haklısın, Necati Hoca'nın C'deki o efsanevi "negatif dizi boyutuyla derleyiciyi patlatma" hilesini tam sentaksıyla (makro haliyle) dokümana eklemek, konunun ruhunu (compile-time kontrolün gücünü) çok daha iyi yansıtır. Hemen o kısmı ve transkriptin devamındaki teknik derinliği içeren yeni bölümü inşa ediyorum.

---

## BÖLÜM 9: [01:38:12 - 01:50:00] Exception Handling: Temel Bileşenler ve Akış Kontrolü

Hoca bu bölümde EH (Exception Handling) mekanizmasının üç temel direğini (`try`, `throw`, `catch`) ve bu mekanizmanın "forcing" (zorlayıcı) doğasını anlatmaya başladı.

### ⚙️ Teknik Detay ve Sentaks (EH Alfabesi)
C++'ta istisna işleme üç anahtar sözcükle yönetilir:
1.  **`try` bloğu:** Hatanın oluşabileceği ve izlenmek istenen kod bölgesini (bağlamı) tanımlar.
2.  **`throw` ifadesi:** Hatanın oluştuğunu bildirir ve bir **Exception Object** (İstisna Nesnesi) fırlatır.
3.  **`catch` bloğu:** Fırlatılan nesneyi türüne göre yakalar ve müdahale eder.

```cpp
void foo(int idx) {
    if (idx < 0) 
        throw idx; // <-- pr-value fırlatıldı. Derleyici bir "Exception Object" oluşturur.
    // ...
}

int main() {
    try { // <-- İzleme bağlamı (Scope) başladı
        foo(-5); 
    } 
    catch (int e) { // <-- Yakalama bağlamı başladı
        std::cout << "Hata yakalandı, kod: " << e << "\n";
    }
}
```

### 🔍 Arka Plan (The Exception Object)
**Kritik Kural:** `throw x;` dendiğinde, yerel `x` nesnesinin kendisi yukarı fırlatılmaz. Çünkü yerel nesnenin ömrü (storage duration) fonksiyon bitince sona erer.
*   Derleyici, `throw` satırında özel bir bellek alanında **Exception Object** oluşturur.
*   Bu nesne, fırlatılan ifadeden (`expression`) kopyalanarak veya taşınarak (move) initialize edilir.
*   Nesnenin hayatını derleyici kontrol eder; ta ki hata tamamen handle edilene kadar.

### 🚩 Kritik Nokta: Uncaught Exception ve "Azrail" Fonksiyon
Eğer bir exception fırlatılır ve hiçbir `catch` bloğu tarafından yakalanmazsa, buna **Uncaught Exception** denir. Bu durumda sistem:
1.  `std::terminate()` fonksiyonunu çağırır.
2.  `std::terminate()` ise varsayılan olarak `std::abort()`'u çağırarak programı acımasızca bitirir.
*Hoca'nın Deyimi:* "`std::terminate`, programımızın **Azrailidir**."

---

## BÖLÜM 10: [01:50:00 - 02:02:00] `terminate` Mekanizmasını Özelleştirmek (Customization Point)

Programın "Azraili" olan `terminate` fonksiyonunun davranışını değiştirmek mümkündür. Hoca burada "Function Pointers" bilgimizi kümülatif olarak geri çağırdı.

### ⚙️ Teknik Detay: `set_terminate` ve `terminate_handler`
Standart kütüphane, geri dönüş değeri ve parametresi olmayan fonksiyonlar için bir tür eş ismi (type alias) tanımlar:

```cpp
// <exception> başlık dosyasında:
using terminate_handler = void (*)(); // <-- Fonksiyon pointer türü
```

`std::set_terminate` fonksiyonu ile kendi "sonlandırma fonksiyonumuzu" sisteme kaydedebiliriz. Bu bir **Get-Set** fonksiyonudur; yani yeni adresi set ederken eskisini return eder.

```cpp
#include <exception>

void my_terminate() {
    std::cout << "Program beklenmedik bir hatayla sonlaniyor! Loglar yaziliyor...\n";
    std::exit(EXIT_FAILURE); // <-- Terminate handler return etmemeli, programı bitirmeli!
}

int main() {
    auto old_handler = std::set_terminate(my_terminate); // <-- Custom handler set edildi
    throw 1; // Yakalayan yok -> std::terminate -> my_terminate çağrılır.
}
```

---

## BÖLÜM 11: [02:02:00 - 02:15:00] EH Modelleri: Resumptive vs Terminative

Hata yakalandıktan sonra ne olur? Hoca bu noktada felsefi ve teknik bir ayrım yaptı.

### 🧠 Rationale: İki Farklı Yaklaşım
1.  **Resumptive Model (Devam Ettirici):** Hata yakalanır, düzeltilir ve program kaldığı yerden veya güvenli bir noktadan hizmet vermeye devam eder. (En yaygın senaryo: Mavi ekran vermek yerine "Hata oluştu, tekrar deneyin" demek).
2.  **Terminative Model (Sonlandırıcı):** Hata öyle büyüktür ki (Resource Exhaustion vb.) recovery (kurtarma) imkansızdır. Catch bloğu içinde loglama yapılır ve program kontrollü şekilde kapatılır.

### ⚙️ Teknik Detay: Catch Bloğu ve Scope (Kapsam) İlişkisi
*   `try` ve `catch` blokları kendi scope'larını oluşturur.
*   `try` bloğu içinde tanımlanan bir değişken `catch` bloğu içinde görülemez (**Static Scope** kuralı).
*   Hoca burada yeni başlayanların en çok yaptığı "try içinde tanımla, dışarıda kullan" hatasına dikkat çekti.

---

## BÖLÜM 12: [02:15:00 - 02:27:00] catch Parametreleri ve Tür Eşleşmesi (Matching Rules)

Dersin en teknik kısımlarından biri: Fırlatılan nesne ile yakalayan parametre nasıl eşleşir?

### ⚙️ Teknik Detay: Kısıtlı Dönüşüm (No Standard Conversions!)
`catch` parametreleri ile `throw` edilen nesne arasında normal fonksiyon çağrılarındaki gibi "Promotion" veya "Standard Conversion" işlemleri **YAPILMAZ**.

*   `throw 'A';` (char) -> `catch (int)` tarafından **YAKALANMAZ**.
*   `throw 2.5f;` (float) -> `catch (double)` tarafından **YAKALANMAZ**.
*   `throw 10;` (int) -> `catch (unsigned int)` tarafından **YAKALANMAZ**.

**Eşleşme Kriterleri:**
1.  **Tam Eşleşme (Exact Match):** Türler birebir aynı olmalı.
2.  **Base-Derived İlişkisi:** En kritik istisna budur. Türemiş sınıf türünden bir nesne, Taban sınıf türünden bir `catch` parametresi ile yakalanabilir.

### 🔍 Arka Plan: Exception Object Hiyerarşisi
```cpp
class Base {};
class Der : public Base {};

try {
    throw Der(); // Der nesnesi fırlatıldı
} 
catch (const Base& ex) { // <-- YAKALANIR! (Upcasting prensibi EH'de geçerli)
    std::cout << "Base referansi ile Der yakalandi.\n";
}
```

---

## BÖLÜM 13: [02:27:00 - 02:43:13] Catch by Reference ve `std::exception` Hiyerarşisi

Hoca dersi bitirirken "Best Practice" (En iyi uygulama) uyarısını yaptı: **Her zaman referansla yakalayın!**

### 🧠 Neden Referansla Yakalamalıyız? (Rationale)
1.  **Maliyet (Copy Cost):** Değerle yakalarsanız (`catch (Base b)`), exception nesnesi bir kez daha kopyalanır. Hata durumunda zaten kısıtlı olan kaynakları kopyalama için harcamak istemeyiz.
2.  **Object Slicing (Nesne Dilimlenmesi):** Değerle yakalarsanız, türemiş sınıfın özellikleri "dilimlenir" ve sadece taban sınıf kısmı kalır. Dinamik tür bilgisi (polimorfizm) kaybolur.
3.  **Polimorfizm:** Referansla yakaladığımızda `ex.what()` dediğimizde, `virtual dispatch` mekanizması sayesinde gerçek nesnenin override edilmiş fonksiyonu çağrılır.

### 🖼️ Görselleştirme (The Hierarchy)
```text
      [ std::exception ]  <-- Sanal what() fonksiyonuna sahip taban sınıf
             |
      -----------------
      |               |
[ logic_error ] [ runtime_error ]
      |               |
[ out_of_range ] [ system_error ]
```

### 🚩 Kritik Nokta: Catch Sıralaması
Birden fazla catch bloğu varsa derleyici yukarıdan aşağıya kontrol eder. Bu yüzden **"Özelden Genele"** bir sıralama yapılmalıdır. Eğer `std::exception` bloğunu en başa koyarsanız, altındaki daha spesifik bloklar (örneğin `out_of_range`) asla çalışmaz (**Shadowing**).

```cpp
try { /* ... */ }
catch (const std::out_of_range& e) { /* ... */ } // <-- Daha özel
catch (const std::logic_error& e)  { /* ... */ } // <-- Daha genel
catch (const std::exception& e)    { /* ... */ } // <-- En genel (Taban sınıf)
```

---

**Bu bölümde Hoca şu 4 kritik hataya dikkat çekti:**
1. Hata nesnelerini değerle yakalayıp "Object Slicing"e sebep olmak.
2. `catch` bloklarını genelden özele sıralayıp spesifik hataları kaçırmak.
3. `catch` parametresini bloğun içinde kullanmayacaksa bile isim verip "unused variable" uyarısı almak.
4. `terminate_handler` fonksiyonlarının return etmeye çalışması (bu fonksiyonlar programı bitirmelidir).



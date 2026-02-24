Harika bir ders, en ön sıradan notlarımı almaya başlıyorum. Necati Hoca'nın her zamanki o titiz terminoloji vurgularıyla dolu, "durumdan vazife çıkartan" derleyici mekanizmalarını incelediğimiz 31. dersin ilk bölümü aşağıdadır.

---

# C++ Programlama Dili - 31. Ders Notları (Bölüm 1)
**Tarih:** 14 Ekim 2024  
**Konu:** Template Instantiation, Terminoloji ve Abbreviated Template Syntax

## 1. Template Terminolojisi ve Instantiation (Örnekleme) Mekanizması [00:00 - 07:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Template'ler tek başına bir kod değil, kod üreten yapılardır. Programcıların çoğu "specialization" ve "instantiation" arasındaki farkı karıştırıyor. Derleyicinin arka planda ne zaman fiilen kod yazdığını anlamak, özellikle link (bağlayıcı) hatalarını çözmek için kritiktir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, derleyicinin template koddan gerçek bir C++ kodu üretmesi sürecini ve oluşan ürünü şu şekilde ayırdı:

*   **Template Instantiation (Örnekleme/Oluşturma):** Sürecin kendisidir.
*   **Template Specialization (Özelleşim):** Süreç sonunda oluşan üründür (product).

```cpp
template <typename T>
void func(T x) { // Bu bir template (kalıptır), henüz bir fonksiyon değildir.
    // ...
}

// Kullanım:
func(10); // <-- Hoca vurguladı: Derleyici burada 'int' türü için template'i instantiate eder.
          // Oluşan 'void func(int)' fonksiyonuna "specialization" denir.
```

### 🔍 Arka Plan (Under the Hood)
Instantiation iki şekilde gerçekleşir:
1.  **Implicit Instantiation (Örtülü Örnekleme):** Hoca'nın deyimiyle "durumdan vazife çıkartarak" yapılması. Derleyici bir çağrıyı gördüğünde argümanlardan yola çıkarak kodu üretir.
2.  **Explicit Instantiation (Açık Örnekleme):** Programcının derleyiciye "şu tür için bu kodu fiilen yaz" talimatı vermesidir.

```cpp
// Explicit Instantiation Sentaksı (Henüz detaylandırılmadı ama gösterildi):
template void func<double>(double); // <-- Kritik: Bu bir çağrı değil, derleyiciye verilmiş bir emirdir.
```

---

## 2. Template Argümanlarını Belirleme Yolları [07:00 - 13:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Derleyicinin bir template'i instantiate edebilmesi için parametrelerin (T, U vb.) hangi gerçek türlere (int, double) karşılık geldiğini bilmesi gerekir.

### ⚙️ Teknik Detay ve Sentaks
Hoca bu durumun 3 yolla mümkün olduğunu belirtti:
1.  **Deduction (Çıkarım):** En sık kullanılan yöntem. C++17 ile beraber sadece fonksiyonlarda değil, Class Template'lerde de (CTAD) başladı.
2.  **Explicit Template Argument (Açık Belirleme):** `<int>` şeklinde elle yazmak.
3.  **Default Template Argument (Varsayılan Argüman):** `template <typename T = int>`.

**Fonksiyon Pointer'larında Deduction Örneği:**
Deduction sadece fonksiyon çağrısı sırasında değil, fonksiyonun adresi bir pointer'a atanırken de gerçekleşir.

```cpp
template <typename T, typename U>
T foo(T x, U y) { return x; }

int main() {
    // Fonksiyon pointer atamasında çıkarım bağlamı (context):
    int (*fp)(int, double) = foo; // <-- Hoca dikkat çekti: 'foo' burada 'foo<int, double>' olarak çıkarılır.
}
```

---

## 3. Void Türünün Template Argümanı Olması [13:00 - 17:00]

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `void` bir template argümanı olabilir mi?
**Cevap:** Evet, `void` bir türdür (type) ve C++'da C'ye göre çok daha yaygın kullanılır.

### ⚙️ Teknik Detay ve Sentaks
Hoca, `void*` (pointer to void) üzerinden bir pattern matching (kalıp eşleme) örneği verdi:

```cpp
template <typename T>
T bar(T* ptr) {
    // ...
    return *ptr; // <-- T = void olursa burada hata oluşur çünkü void* dereferans edilemez!
}

int main() {
    void* vptr = nullptr;
    // bar(vptr); // <-- T, 'void' olarak çıkarılır (Deduction).
}
```

---

## 4. C++20 Abbreviated Template Syntax (Kısaltılmış Şablon Sentaksı) [17:00 - 25:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Template bildirimlerindeki `template <typename T>` kalıbı bazen çok fazla "boilerplate" (basmakalıp) kod oluşturuyor. C++20 ile fonksiyon parametrelerinde `auto` kullanımı, kodun daha temiz ve okunabilir olmasını sağladı.

### ⚙️ Teknik Detay ve Sentaks
Hoca bu özelliği "devrimci" (revolutionary) olarak nitelendirdi.

```cpp
// Geleneksel Yöntem:
template <typename T>
void func(T x) { }

// C++20 Abbreviated Syntax:
void func(auto x) { } // <-- Hoca vurguladı: Bu HALA bir fonksiyon şablonudur!

// Birden fazla auto kullanımı:
void sum(auto x, auto y) { } 
// <-- KRİTİK AYRIM: Bu, template <typename T, typename U> sum(T x, U y) demektir.
// Yani x ve y farklı türler olabilir.
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `void foo(auto x, auto y)` ile `template <typename T> void foo(T x, T y)` aynı mıdır?
**Cevap:** HAYIR! `auto` kullanılan versiyonda her bir `auto` farklı bir template parametresine karşılık gelir. Eğer her iki parametrenin de *mutlaka aynı tür* olması gerekiyorsa klasik template sentaksı kullanılmalıdır.

### 🔍 Arka Plan (Under the Hood)
Derleyici arka planda `auto` gördüğü her parametre için yeni bir `typename T1`, `typename T2` oluşturur. Bu sadece bir "syntactic sugar" (yazım kolaylığı) olsa da, Generic Lambda'lardan (C++14) fonksiyonlara taşınmış bir özelliktir.

---

### 🔗 Önceki Derslerle Bağlantı
*   **12. Ders:** Fonksiyon pointerları konusundaki eksiklerin burada "deduction" (çıkarım) ile birleştiğini görüyoruz.
*   **Lambda Dersleri:** Lambda parametrelerindeki `auto`'nun (C++14) artık normal fonksiyonlara da geldiğini hatırlattı.

---

**Bu ilk 25 dakikalık bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  Specialization ve Instantiation kavramlarının birbirinin yerine kullanılması (Birisi süreç, diğeri sonuç).
2.  `auto` parametreli fonksiyonların normal fonksiyon sanılması (Bunlar aslında gizli template'lerdir).
3.  `void` türünün template argümanı olamayacağının sanılması.

Necati Hoca "Daldan dala atlıyoruz ama böyle daha eğlenceli ve öğretici oluyor" diyerek vitesi artırdı. Üye fonksiyon şablonlarından başlayıp, C++'ın en can yakıcı konularından biri olan `decltype` kurallarına kadar indik. İşte dersin ikinci çeyreği:

---

# C++ Programlama Dili - 31. Ders Notları (Bölüm 2)
**Timestamp:** [00:25:00] - [00:50:00]  
**Konu:** Member Function Templates, Trailing Return Type ve decltype Kuralları

## 1. Member Function Templates (Üye Fonksiyon Şablonları) [25:00 - 27:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir sınıfın kendisi jenerik (template) olmasa bile, belirli fonksiyonlarının farklı türlerle çalışması gerekebilir. Özellikle tür dönüşümleri veya farklı türdeki verileri kabul eden "setter" metotları için bu yapı hayat kurtarıcıdır.

### ⚙️ Teknik Detay ve Sentaks
Hoca, sınıfın içindeki bir fonksiyonun başına `template` bloğu getirerek onu nasıl şablonlaştırdığımızı gösterdi:

```cpp
class MyClass {
public:
    void normalFunc(int x); // Normal üye fonksiyon

    template <typename T>
    void genericFunc(T x) { // <-- Hoca vurguladı: Sınıf şablon değil ama fonksiyon şablon!
        // ...
    }

    template <typename T>
    operator T() { // <-- Kritik: Dönüşüm (conversion) operatörü bile şablon olabilir.
        return T{}; 
    }
};

int main() {
    MyClass m;
    m.genericFunc(45);   // T = int çıkarımı yapıldı.
    m.genericFunc(3.14); // T = double çıkarımı yapıldı.
}
```

---

## 2. Trailing Return Type (Sondan Gelen Geri Dönüş Türü) [27:00 - 37:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
1.  **Okunabilirlik:** Özellikle diziye pointer veya referans dönen fonksiyonların sentaksı "korkunç" (Hoca'nın tabiriyle) olabiliyor.
2.  **Scoping (Kapsam) Sorunu:** Fonksiyonun geri dönüş türü, sınıfın içindeki bir `nested` (yuvalanmış) tür ise, fonksiyon isminden önce bu türü yazmak için uzun uzun niteleme yapmak gerekir.
3.  **Generic Programming:** Geri dönüş türü, henüz bildirilmemiş olan parametre değişkenlerine bağlıysa (bkz. `decltype`), geleneksel sentaks hata verir.

### ⚙️ Teknik Detay ve Sentaks

**Dizi Referansı Dönen Fonksiyon Karmaşası:**
Hoca, Trailing Return Type olmasa ne kadar zorlanacağımızı şu örnekle gösterdi:

```cpp
// Geleneksel Sentaks (Yazması ve okuması bela!):
int (&foo(int x))[10]; // 10 elemanlı int diziye referans döner.

// Trailing Return Type (Tertemiz):
auto foo(int x) -> int(&)[10] { // <-- Hoca "Bunu kullanın" dedi.
    // ...
}
```

**Kapsam (Scoping) Kolaylığı:**
```cpp
class MyClass {
public:
    struct Nested { };
    Nested func();
};

// Geleneksel:
MyClass::Nested MyClass::func() { return Nested{}; } // Geri dönüş türünü nitelemek ZORUNLU.

// Trailing Return Type:
auto MyClass::func() -> Nested { return Nested{}; } 
// <-- Hoca buraya dikkat çekti: Okun sağ tarafı 'class scope' içindedir, niteleme gerekmez!
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Neden şablonlarda `auto func(T x, U y) -> decltype(x+y)` yazıyoruz?
**Cevap:** Çünkü derleyici fonksiyonun başına geldiğinde henüz `x` ve `y` isimlerini tanımıyor (isimler parametre parantezinde bildiriliyor). Sondan gelen geri dönüş türü kısmında ise isimler artık kapsamda (in scope).

---

## 3. C++14 Auto Return Type ve Sınırları [37:00 - 46:00]

### 📊 Standart Karşılaştırması

| Özellik | C++11 | C++14 | C++20 |
| :--- | :--- | :--- | :--- |
| **Trailing Return Type** | Var (Kritik) | Var | Var |
| **Auto Return Type** | Yok | Geldi (Basit durumlar için) | Var |
| **Abbreviated Syntax** | Yok | Yok | Geldi (auto parametreler) |

### 🚩 Kritik Nokta: Auto vs Trailing
Hoca, "Auto geldi, trailing bitti" diyen blog yazılarına çok kızıyor! Aradaki dev farkı şöyle açıkladı:
*   `auto` geri dönüş türü **Decay (çürüme)** kuralına tabidir; referansları düşürür.
*   Eğer bir referansı korumak istiyorsak Trailing Return Type + `decltype` kullanmak zorundayız.

```cpp
template <typename T>
auto get_val(T& x) { return x; } // <-- Hoca uyardı: Bu her zaman kopyasını döner (int).

template <typename T>
auto get_ref(T& x) -> decltype(x) { return x; } // <-- Bu referansı korur (int&).
```

---

## 4. decltype Kuralları: Identifier vs Expression [46:00 - 50:00]

Hoca, `decltype`'ın iki farklı dünyası olduğunu ve mülakatlarda buradan "can yakıldığını" belirtti.

### ⚙️ Teknik Detay: İki Ayrı Kural Seti

1.  **Identifier (İsim) Kuralı:** Eğer `decltype(e)` içerisindeki `e` bir değişken ismiyse, derleyici o değişkenin **bildirimindeki (declaration)** türü verir.
2.  **Expression (İfade) Kuralı:** Eğer `e` bir ifadeyse (operatör içeriyorsa vb.), derleyici ifadenin **Value Category** (Değer Kategorisi)'ne bakar.

**Value Category'ye Göre decltype Sonucu:**
*   **L-value** ise: `T&` (Referans türü elde edilir)
*   **PR-value** ise: `T` (Referans olmayan tür)
*   **X-value** ise: `T&&` (R-value referans türü)

### 🚩 Mülakatların Gözde Sorusu: `decltype(x)` vs `decltype((x))`
```cpp
int x = 10;
decltype(x) a;   // Kural 1: x bir isimdir. Tür: int.
decltype((x)) b; // Kural 2: (x) bir ifadedir ve L-value'dur. Tür: int&.
// <-- HATA BURADA: b'ye ilk değer verilmediği için derleyici kızar!
```

### 🔍 Arka Plan (Unevaluated Context)
`decltype` operanda olan ifadeyi fiilen çalıştırmaz. Bu bölgeye **Unevaluated Context** denir.
*   `decltype(++x)` yazarsanız `x` artmaz, sadece türü hesaplanır.
*   `decltype(*ptr)` yazarsanız, `ptr` null bile olsa kod çalışma zamanında çökmez (UB oluşmaz), çünkü kod sadece derleme zamanında tür analizi için kullanılır.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  `auto` geri dönüş türünün her zaman referansı koruduğunu sanmak (Oysa `auto` referansı düşürür!).
2.  `decltype` kurallarında isim (identifier) ile parantezli ifade (expression) arasındaki farkı kaçırmak.
3.  `decltype` içindeki işlemlerin (side effects) fiilen gerçekleştiğini sanmak.

Necati Hoca, "Mülakatlarda sormayı çok severler" diyerek `decltype` ve değer kategorileri (value categories) arasındaki o ince çizgiyi adeta bir cerrah titizliğiyle deşmeye devam ediyor. Bu bölüm, C++'ın "Unevaluated Context" (değerlendirilmeyen bağlam) mantığını ve Variadic Template'lere (değişken sayıda parametreli şablonlar) giden yolu inşa ediyor.

---

# C++ Programlama Dili - 31. Ders Notları (Bölüm 3)
**Timestamp:** [00:50:00] - [01:15:00]  
**Konu:** decltype Derin Dalış, Unevaluated Context ve Variadic Templates Giriş

## 1. decltype ve Expression (İfade) Kuralları [50:00 - 56:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Sadece bir değişkenin adını değil, bir işlemin sonucunun türünü (ve o sonucun bir nesne mi yoksa bir referans mı olduğunu) anlamak gerekir. Bu, özellikle "perfect forwarding" (mükemmel gönderim) yaparken hayati önem taşır.

### ⚙️ Teknik Detay ve Sentaks
Hoca, `decltype`'ın ikinci kural setini (Expression Rule) şu tabloyla özetledi:

| İfadenin Değer Kategorisi | decltype Sonucu |
| :--- | :--- |
| **L-value** (Sol Taraf Değeri) | `T&` (Sol taraf referansı) |
| **PR-value** (Saf Sağ Taraf Değeri) | `T` (Türün kendisi) |
| **X-value** (Eksilen Değer) | `T&&` (Sağ taraf referansı) |

```cpp
int x = 10;
int* ptr = &x;

decltype(x + 3) a;    // x + 3 bir PR-value. Tür: int.
decltype(*ptr) b = x; // *ptr bir L-value. Tür: int&. 
                      // <-- Hoca vurguladı: b artık x'in kendisidir!

decltype(std::move(x)) c = std::move(x); // std::move(x) bir X-value. Tür: int&&.
```

### 🚩 Mülakat Sorusu / Kritik Nokta
**Soru:** `decltype(x)` ile `decltype((x))` neden farklı türler üretir?
**Cevap:** 
*   `decltype(x)`'te `x` bir **identifier** (isim) kabul edilir. Kural gereği ismi nasıl deklare edildiyse o türü verir (`int`).
*   `decltype((x))`'te ise parantez onu bir **expression** (ifade) haline getirir. İsim olmayan her ifade değer kategorisine sokulur. `(x)` bir L-value olduğu için sonuç `int&` olur.

---

## 2. Unevaluated Context (Değerlendirilmeyen Bağlam) [56:00 - 01:03:00]

### 🔍 Arka Plan (Under the Hood)
C++'da bazı operatörler (`sizeof`, `decltype`, `noexcept`, `typeid`) operandı olan ifadeyi fiilen çalıştırmazlar. Derleyici sadece o ifadenin sonucunda oluşacak "tür bilgisini" analiz eder.

### ⚙️ Teknik Detay ve Sentaks
Hoca, bu bağlamın tehlikeli sularda nasıl güvenli yüzmemizi sağladığını gösterdi:

```cpp
int* p = nullptr;
decltype(*p) r = x; // <-- Hoca buraya dikkat çekti: Normalde nullptr dereferans etmek 
                    // Runtime'da UB (Tanımsız Davranış) oluşturur. 
                    // Ama decltype içinde bu GÜVENLİDİR, işlem yapılmaz, sadece tür (int&) bulunur.

int a[10];
decltype(a[20]) r2 = x; // <-- Kritik: Dizi sınırı aşılsa bile hata oluşmaz, çünkü ifade "unevaluated".
```

### 🚩 Kritik Nokta: Side Effect (Yan Etki) Yokluğu
```cpp
int x = 5;
decltype(++x) y = x; // x hala 5! ++x işlemi fiilen yapılmadı.
```

---

## 3. Auto Return Type'ın En Zayıf Halkası: std::ostream Örneği [01:03:00 - 01:08:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Hoca, "Neden `auto` varken hala Trailing Return Type kullanıyoruz?" sorusuna efsane bir örnekle cevap verdi. Bazı nesneler (örneğin stream nesneleri) **kopyalanamaz (non-copyable)**.

### ⚙️ Teknik Detay ve Sentaks
```cpp
// 1. Senaryo: Trailing Return Type (BAŞARILI)
auto print1(const auto& x) -> decltype(std::cout << x) {
    return std::cout << x; // Geri dönüş türü: std::ostream& (Referans korundu!)
}

// 2. Senaryo: Auto Return Type (HATA!)
auto print2(const auto& x) {
    return std::cout << x; 
    // <-- DERLEYİCİ ŞU SEBEPLE KIZIYOR: 'auto' referansı düşürür (decay). 
    // Fonksiyon std::ostream nesnesini KOPYALAYARAK dönmeye çalışır. 
    // Ama ostream'in copy constructor'ı DELETE edilmiştir!
}
```

---

## 4. Variadic Templates ve Parameter Pack (Parametre Paketi) Giriş [01:08:00 - 01:15:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Modern C++ öncesinde (C++11 öncesi), bir fonksiyona kaç tane ve hangi türden argüman geleceğini önceden bilmediğimiz durumları (örneğin `printf` benzeri yapılar) tip güvenliği (type-safety) içinde çözemiyorduk.

### ⚙️ Teknik Detay ve Sentaks
Hoca, **Ellipsis (...)** token'ı ile tanıştırılan "Parameter Pack" kavramını açıkladı:

```cpp
template <typename... Args> // <-- Args bir "Template Parameter Pack" (Şablon parametre paketi)
void func(Args... args) {   // <-- args bir "Function Parameter Pack" (Fonksiyon parametre paketi)
    // ...
}

// Kullanım:
func(1, 2.5, "Necati"); // Args paketi içinde: int, double, const char* var.
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `typename...` ile `typename` farkı nedir?
**Cevap:** `typename` sadece bir tek tür beklerken, `typename...` sıfır veya daha fazla sayıda, birbirinden tamamen farklı türleri paket halinde kabul eder.

---

### 🔗 Önceki Derslerle Bağlantı
*   **15. Ders (L-value/R-value):** Değer kategorilerinin sadece atama operatörü için değil, `decltype` gibi modern araçlar için de temel taşı olduğunu gördük.
*   **Integral Promotion:** `auto sum(auto x, auto y) { return x + y; }` örneğinde, `char` göndersek bile sonucun neden `int` çıktığını (integral promotion) eski derslere atıfla hatırlattı.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  `decltype`'ın her zaman değişkenin kendi türünü verdiğini sanmak (Parantez her şeyi değiştirir!).
2.  `auto` geri dönüş türünün kopyalanamayan (non-copyable) türlerde patlayacağını unutmak.
3.  `Variadic Templates` yapısını C'deki `...` (varargs) ile karıştırmak (C++'daki yapı tamamen Type-Safe'dir).

Necati Hoca, "Sınıf şablonları bir sınıf değildir, sınıf kodu yazdırmanın bir aracıdır" diyerek C++'ın en güçlü kütüphanesi olan STL'in (Standart Şablon Kütüphanesi) kalbine, yani **Class Templates** konusuna giriş yaptı. 

---

# C++ Programlama Dili - 31. Ders Notları (Bölüm 4)
**Timestamp:** [01:15:00] - [01:40:00]  
**Konu:** Sınıf Şablonları (Class Templates) - Giriş, Sentaks ve Üye Fonksiyonlar

## 1. Sınıf Şablonlarına Neden İhtiyaç Duyuldu? [01:15:00 - 01:21:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Jenerik sınıflar olmasaydı, her farklı veri türü için sınıfın kodunu elle kopyalayıp yapıştırmak zorunda kalırdık.
*   **Karmaşık Sayılar:** `Complex` sınıfının verisi `float`, `double` veya `long double` olabilir. Her biri için ayrı sınıf yazmak "Code Duplication" (Kod Tekrarı) ve bakım zorluğu demektir.
*   **Veri Yapıları:** `std::vector`, `std::list` gibi konteynerlerin mantığı (ekleme, silme) türden bağımsızdır. Ancak tutulan verinin türüne göre sınıfın kodunda küçük değişiklikler gerekir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, `Complex` sınıfı üzerinden jenerik yapının temelini attı:

```cpp
template <typename T>
class Complex {
private:
    T re, im; // <-- Hoca vurguladı: Gerçek ve sanal kısım artık T türünden.
public:
    T get_real() const { return re; } // Geri dönüş türü de T'ye bağlı.
    void set_real(T val) { re = val; }
};
```

### 🔍 Arka Plan (Under the Hood)
Sınıf şablonunun bir **Specialization** (Özelleşim/Açılım) oluşturması için derleyicinin şablondan kodu fiilen yazması gerekir. 
*   `Complex<double>` dendiğinde derleyici `T` gördüğü her yere `double` koyarak yeni bir sınıf tanımı yazar.

---

## 2. Sınıf Şablonu Bildirimi ve "Açılım" Kavramı [01:21:00 - 01:29:00]

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `std::string` bir sınıf mıdır?
**Cevap:** Hayır, `std::string` aslında `std::basic_string<char>` sınıf şablonunun bir açılımına (specialization) verilen bir **Alias** (takma isim)'dir. 

### ⚙️ Teknik Detay: Forward Declaration (Ön Bildirim)
Sınıf şablonları da tıpkı normal sınıflar gibi önceden bildirilebilir:

```cpp
template <typename T>
class MyClass; // <-- Sadece bildirim (Forward Declaration)

// Kullanım sırasında argüman belirtmek zorunludur (C++17 öncesi için):
MyClass<int> m; // <-- Bu artık gerçek bir türdür (type).
```

### 🖼️ Görselleştirme (Memory Layout)
Hoca `std::array` örneğini verdi:
```cpp
template <typename T, std::size_t size>
struct Array {
    T buffer[size]; // Statik dizi sarmalayıcısı
};

Array<int, 10> a1;    // 40 byte (Stack)
Array<double, 20> a2; // 160 byte (Stack)
```

---

## 3. Üye Fonksiyonların Sınıf Dışında Tanımlanması [01:29:00 - 01:40:00]

Hoca bu bölümün sentaksının çok sık karıştırıldığını ve hata yapıldığını belirterek en ön sıradaki öğrencileri uyardı.

### ⚙️ Teknik Detay ve Sentaks
Bir sınıf şablonunun üye fonksiyonu sınıf dışında tanımlanacaksa, her fonksiyon tanımı bir **template bloğuyla** başlamak zorundadır.

```cpp
template <typename T>
class MyClass {
public:
    void foo(T x); // Sınıf içinde bildirim
    void bar();
};

// SINIF DIŞINDA TANIM:
template <typename T> // <-- Hoca: "Bunu yazmayı unutursanız derleyici küser."
void MyClass<T>::foo(T x) { // <-- Niteleme yaparken <T> kullanımı ZORUNLUDUR.
    // ...
}
```

### 🔍 Arka Plan (Scoping Rules)
Hoca çok kritik bir "Scoping" (Kapsam) kuralına değindi:
*   Sınıfın kendi tanımı (scope) içerisindeyseniz, şablon ismini yanına `<T>` koymadan yalın halde (Injected Class Name) kullanabilirsiniz. 

```cpp
template <typename T>
class MyClass {
public:
    MyClass func(MyClass param); 
    // <-- Hoca buraya dikkat çekti: 'MyClass<T>' yazmakla aynı anlamdadır. 
    // Sınıfın içindeyiz, derleyici "durumdan vazife çıkartıyor".
};
```

### 🚩 Kritik Nokta: Ayrı Türler (Distinct Types)
Hoca'nın mülakatlarda en çok elenenlerin bu noktadan çıktığını söylediği yer:
**Aynı şablondan (template) üretilmiş olsalar bile, farklı argümanlara sahip sınıflar tamamen farklı türlerdir!**

```cpp
MyClass<int> m1;
MyClass<double> m2;

// m1 = m2; // <-- HATA: 'A' sınıfını 'B' sınıfına atamaya çalışmak gibidir.
// Aralarında hiçbir miras veya örtülü dönüşüm (implicit conversion) yoktur.
```

---

### 🔗 Önceki Derslerle Bağlantı
*   **Alias Templates:** Hoca, `std::string`'in bir alias olduğunu anlatırken 28. derste gördüğümüz `using` anahtar sözcüğüne atıfta bulundu.
*   **Value Initialization:** `std::pair` örneğinde `int` ve `pointer` elemanların sıfırlanmasını anlatırken "Value Initialization" (Değerle İlk Değer Verme) kurallarını hatırlattı.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  Sınıf dışı tanımlarda fonksiyonun tepesine `template <typename T>` yazmayı unutmak.
2.  `MyClass<int>` ile `MyClass<double>`'ı aynı tür sanıp birbirine atamaya çalışmak.
3.  Sınıf şablonunun ismini tek başına bir tür ismi (type name) olarak kullanmaya çalışmak (Argüman olmadan şablon isimdir, tür değildir).

Necati Hoca, dersin bu son bölümünde "C++'ın en can alıcı yerlerine giriyoruz, burayı anlayan C++'ı anlar" diyerek vitesi iyice artırdı. Sınıf şablonlarının kapsam kurallarından başlayıp, `std::pair` ve `std::vector` üzerinden "Universal Reference" (Evrensel Referans) yanılgılarını yıktı. İşte dersin son ve en teknik bölümü:

---

# C++ Programlama Dili - 31. Ders Notları (Bölüm 5 - Final)
**Timestamp:** [01:40:00] - [02:42:16]  
**Konu:** Injected-class-name, Üye Şablonlar ve std::pair Derin İnceleme

## 1. Injected-class-name ve Kapsam (Scope) Kuralları [01:40:00 - 01:47:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Sınıf şablonu içerisinde her seferinde `MyClass<T, U>` yazmak hem hata riskini artırır hem de kodu kirletir. C++ derleyicisi, sınıfın kendi içinde şablon ismini bir "tür" olarak tanımanızı sağlar.

### ⚙️ Teknik Detay ve Sentaks
Hoca, `typeid` operatörüyle bu isimlerin arka planda nasıl aynı kapıya çıktığını gösterdi:

```cpp
template <typename T, typename U>
class MyClass {
public:
    // <-- Hoca buraya dikkat çekti: 'MyClass' yalın ismi 'MyClass<T, U>' yerine geçer.
    void bar(MyClass x); 

    void print_type() {
        std::cout << typeid(MyClass).name() << "\n"; // Mevcut specialization türünü basar.
    }
};

int main() {
    MyClass<int, double> m;
    m.print_type(); // Çıktı: MyClass<int, double> açılımı olacaktır.
}
```

---

## 2. Şablonlarda "Pattern Matching" (Kalıp Eşleme) [01:47:00 - 01:54:00]

### 🔍 Arka Plan (Under the Hood)
Fonksiyon şablonları, parametre olarak başka bir sınıf şablonunun specialization'ını alabilir. Derleyici burada müthiş bir "deduction" (çıkarım) yaparak `T` ve `U` türlerini otomatik çözer.

### ⚙️ Teknik Detay ve Sentaks
```cpp
template <typename T, typename U>
void process(MyClass<T, U> m) { 
    // <-- Kritik: Fonksiyon çağrıldığında T ve U otomatik olarak MyClass'tan sökülüp alınır.
}

MyClass<int, char> obj;
process(obj); // T = int, U = char olarak çıkarıldı (Deduction).
```

---

## 3. "Lazy Instantiation" (Tembel Örnekleme) Mantığı [01:54:00 - 02:05:00]

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Bir sınıf şablonunun bir üye fonksiyonu hata içeriyorsa (örneğin `T` türünde olmayan bir metodu çağırıyorsa), bu durum her zaman derleme hatasına yol açar mı?
**Cevap:** HAYIR. Sadece o fonksiyon **call** (çağrı) edilirse veya adresi alınırsa derleyici o fonksiyonun kodunu yazar ve hatayı o an fark eder.

### ⚙️ Teknik Detay ve Sentaks
```cpp
template <typename T>
class MyClass {
public:
    void foo(T x) { x.non_existent_func(); } // T türünde bu fonksiyon yoksa bile...
    void bar() { /* hata yok */ }
};

int main() {
    MyClass<int> m;
    m.bar(); // <-- Sıkıntı yok! foo() çağrılmadığı için derleyici foo'nun kodunu yazmaz.
    // m.foo(5); // <-- HATA: Şimdi derleyici foo'yu instantiate eder ve "int'in böyle bir metodu yok" der.
}
```

---

## 4. Büyük Yanılgı: Universal Reference vs. R-Value Reference [02:05:00 - 02:14:00]

Hoca bu bölümü "Mülakatlarda en çok can yakan soru" olarak işaretledi.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `template <typename T> class MyClass { void foo(T&& x); };` yapısındaki `T&&` bir Universal Reference mıdır?
**Cevap:** KESİNLİKLE HAYIR! Bir referansın Universal Reference (Forwarding Reference) olması için `T`'nin **fonksiyon çağrısı sırasında** çıkarım (deduction) yapılması gerekir. Burada `T`, sınıf oluşturulduğu an bellidir.

```cpp
template <typename T>
class MyClass {
    void push(T&& x); // <-- Bu bir R-Value Reference'tır. Sadece sağ taraf değeri alır!
    
    template <typename U>
    void emplace(U&& x); // <-- Bu bir Universal Reference'tır. Çünkü U fonksiyon seviyesinde deduc edilir.
};
```

---

## 5. std::pair ve Member Templates (Üye Şablonlar) [02:14:00 - 02:42:16]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
`std::pair<int, int>` nesnesini `std::pair<double, double>` nesnesine atayabilmek isteriz. Çünkü `int`'den `double`'a dönüşüm vardır. Ancak bu iki sınıf şablonu açılımı C++'da "farklı türler"dir. Bunu çözmek için **Member Template Constructor** kullanılır.

### ⚙️ Teknik Detay ve Sentaks
Hoca, `std::pair`'in esnekliğini sağlayan o meşhur yapıcı fonksiyonu (constructor) ayağa kaldırdı:

```cpp
template <typename T1, typename T2>
struct Pair {
    T1 first;
    T2 second;

    // Member Template Constructor:
    template <typename U1, typename U2>
    Pair(const Pair<U1, U2>& other) 
        : first(other.first), second(other.second) {} 
    // <-- Hoca buraya dikkat çekti: Bu sayede farklı tipteki pair'ler birbirine kopyalanabilir!
};
```

### 🖼️ Görselleştirme (ASCII Art)
İç içe geçmiş (Nested) Template'lerin yazdırılması:
```text
std::pair<std::pair<int, double>, std::pair<string, long>>
      |             |                |               |
   [ [12, 3.4], [ "Necati", 100L ] ]
```
Hoca, bu karmaşık yapıyı tek bir `operator<<` şablonuyla nasıl otomatik olarak "recursive" (özyinelemeli) gibi yazdırdığımızı gösterdi.

---

### 🔗 Önceki Derslerle Bağlantı
*   **Move Semantics:** `std::vector::push_back(T&&)` fonksiyonunun neden Universal Reference olmadığını anlatırken 18. dersteki "Deduction vs Substitution" konusuna bağ kurdu.
*   **Const correctness:** `std::pair` elemanlarının `first` ve `second` olarak doğrudan erişilebilir (public) olmasının tasarım tercihi olduğunu belirtti.

---

**Ders Sonu Özeti (Hoca'nın İdiomlarıyla):**
1.  **"Durumdan vazife çıkartmak":** Derleyicinin implicit instantiation yapması.
2.  **"Tembel Derleyici":** Kullanılmayan üye fonksiyonların şablonlardan üretilmemesi.
3.  **"Farklı Dünyaların İnsanları":** `Pair<int, int>` ve `Pair<double, double>`'ın tamamen farklı türler olması.



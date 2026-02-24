# Ders Notu: C++ Şablon Mekanizması ve Tür Çıkarımı (Ders 30)

**Tarih:** 9 Ekim 2024  
**Eğitmen:** Necati Ergin  
**Konu:** Function Templates, Template Argument Deduction, CTAD, Syntax Checking Phases

---

## 1. Jenerik Programlama ve Şablonların Temeli [00:00 - 08:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C++ gibi statik tür sistemine (static type system) sahip dillerde kod, türlere sıkı sıkıya bağlıdır. Ancak algoritmaların çoğu (örneğin `swap`) mantıksal olarak türden bağımsızdır. Şablonlar gelmeden önce, her yeni tür için aynı algoritmayı manuel olarak tekrar yazmak (overloading) gerekiyordu. Bu hem kod tekrarına hem de bakım zorluğuna yol açıyordu.

### ⚙️ Teknik Detay ve Sentaks
Derleyiciye belirli bir tür için kod yazmasını söyleyen yapıya **Meta-code** (Metakod) veya **Code Template** (Kod Kalıbı) denir. Derleyicinin bu kalıptan gerçek C++ kodu üretmesine ise **Instantiation** (Somutlaştırma) denir.

**Şablon Kategorileri:**
1.  **Function Templates** (Fonksiyon Şablonları)
2.  **Class Templates** (Sınıf Şablonları)
3.  **Type Alias Templates** (Tür Eş İsim Şablonları)
4.  **Variable Templates** (Değişken Şablonları)
5.  **Concepts** (Konseptler - C++20)

### 🔍 Arka Plan (Under the Hood)
C++'daki jeneriklik, Java veya C#'daki gibi çalışma zamanında (runtime) polimorfizm ile değil, derleme zamanında (compile time) her türe özel "Tailor-made" (terzi dikimi) kod üretilerek sağlanır.
*   **Kod Ekonomisi:** Sınıf şablonlarında (örneğin `std::string`), çağrılmayan üye fonksiyonların kodu derleyici tarafından yazılmaz. Bu, kullanılmayan kodun binary boyuta eklenmesini engeller.

---

## 2. Template Parametreleri ve CTAD [08:00 - 15:50]

### ⚙️ Teknik Detay ve Sentaks
Şablon parametreleri iki ana kategoriye ayrılır:
*   **Type Parameter:** `typename T` veya `class T` (Bir türü temsil eder).
*   **Non-type Parameter:** `int N` (Bir değeri temsil eder).

Template argümanlarını belirlemenin 3 yolu vardır:
1.  **Deduction** (Çıkarım): Argümanlardan otomatik anlaşılması.
2.  **Explicit Template Arguments**: `func<int>(val)` şeklinde açıkça belirtilmesi.
3.  **Default Template Arguments**: `template <typename T = int>`.

### 📊 Standart Karşılaştırması: CTAD
| Özellik | C++17 Öncesi | C++17 ve Sonrası |
| :--- | :--- | :--- |
| **Class Template Argument Deduction (CTAD)** | Desteklenmez. `std::vector<int> v` yazmak zorunludur. | Desteklenir. `std::vector v{1, 2, 3}` yazıldığında derleyici `int` olduğunu anlar. |

```cpp
// <-- Hoca CTAD örneğine dikkat çekti
std::vector ivek{1, 2, 3, 4}; // C++17 öncesi hata, C++17 sonrası legal (int çıkarımı yapılır)
std::optional op = 3.4;      // <double> yazmaya gerek kalmadı, CTAD devrede.
```

---

## 3. Derleyicinin Şablon Kontrol Aşamaları [15:50 - 18:40]

### 🔍 Arka Plan (Under the Hood)
Derleyiciler şablon kodlarını iki aşamalı kontrolden geçirir:
1.  **Aşama (Template Definition Time):** Şablonun henüz hiçbir tür için kullanılmadığı aşama. Sadece "türden bağımsız" syntax hataları (eksik noktalı virgül, parantez eşleşmesi vb.) kontrol edilir.
2.  **Aşama (Instantiation Time):** Şablon belirli bir tür için somutlaştırıldığında yapılan kontrol.

```cpp
template <typename T>
void func(T x) {
    bar(); // <-- HATA: T'den bağımsız olduğu için 1. aşamada yakalanır (bar tanımlı değil)
    x.foo(); // <-- 1. aşamada HATA VERMEZ! Çünkü T'nin foo() üyesi olup olmadığı instantiation zamanında belli olur.
}
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Yukarıdaki `x.foo()` neden derleme hatası vermiyor?
**Cevap:** Çünkü `foo` ismi "Dependent Name" (bağımlı isim) olabilir. Belki de kullanıcı `T` türü için `foo` ismini **ADL (Argument Dependent Lookup)** ile bulacaktır. Derleyici bu kontrolü 2. aşamaya (instantiation) erteler.

---

## 4. Template Argument Deduction (TAD) ve Ambiguity [18:40 - 25:50]

### ⚙️ Teknik Detay ve Sentaks
Derleyici, fonksiyona gönderilen argümanlara bakarak `T`'nin ne olduğunu anlamaya çalışır. Ancak bazen çıkarım yapılamaz veya çelişki (ambiguity) oluşur.

```cpp
template <typename T>
void func(T x, T y) { }

int main() {
    func(10, 20);      // Legal: T = int
    // func(10, 3.4);  // <-- HATA BURADA: Derleyici şu sebeple kızıyor: "Ambiguous deduction"
                       // İlk argümandan T=int, ikinciden T=double çıkıyor. Çelişki!
}
```

### 🔍 Arka Plan (Under the Hood)
Deduction sürecinde **Implicit Conversion** (örtülü dönüşüm) yoktur. 
*   **Hoca'nın İdiomu:** "Burada dönüşüm yapılacak bir bağlam (context) yok." 
*   Derleyici, her parametre için ayrı ayrı çıkarım yapar ve sonunda bu çıkarımların "aynı tür" olup olmadığına bakar. Eğer `int` ve `double` çıkarımı yapılmışsa, derleyici "Ben birini `int` seçeyim de diğerini ona dönüştüreyim" demez; direkt hata verir.

---

## 5. Çıkarım Kuralları ve "Hoca'nın Trick'i" [25:50 - 35:00]

### 🔗 Kümülatif Bağlantılar
Necati Hoca, bu konunun temellerini **Auto Type Deduction** (12. ve 13. derslerde görülen konu) ile bağladı. İstisnalar hariç, `auto` kuralları ile `template` çıkarım kuralları birebir aynıdır.

### 🖼️ Görselleştirme (Deduction Kategorileri)
```text
Deduction Aşaması:
ParamType P;
P x = expr;

1. T x (Parametre türü direkt T) -> Decay (Bozulma) uygulanır. Const'luk düşer.
2. T& x (Referans parametre)     -> Decay uygulanmaz. Const'luk korunur.
3. T&& x (Universal/Forwarding) -> L-value/R-value ayrımı yapılır.
```

### 🚩 Mülakat Sorusu / Kritik Nokta: Hoca'nın `type_teller` Hilesi
Derleyicinin türü ne olarak gördüğünü anlamak için "Incomplete Type" hatasından faydalanma:

```cpp
template <typename T>
class type_teller; // Tanımsız sınıf (Incomplete Type)

template <typename T>
void func(T x) {
    type_teller<T> t; // <-- HATA BURADA: Derleyici "type_teller<int> t" incomplete diyecek
                      // Hata mesajında T'nin ne olduğunu açıkça göreceğiz.
}
```
**Derleyici Gözü:** "Error: aggregate 'type_teller<int> t' has incomplete type..." (Buradan çıkarımın `int` olduğu kesinleşir).

### Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:
1.  **Deduction vs Conversion:** Şablon parametresi çıkarılırken asla implicit conversion (int -> double gibi) yapılmaz.
2.  **Decay (Bozulma):** Eğer parametre `T x` şeklindeyse (call by value), diziler pointer'a "decay" olur, `const` ve `volatile` niteleyicileri düşer.
3.  **Reference Nuance:** Eğer parametre `T& x` şeklindeyse, `const` düşmez; çünkü referans orijinal nesneye bağlıdır.

## 4. `type_teller` Hilesi ile Çıkarım Analizi [35:00 - 45:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Derleyici arka planda karmaşık tür çıkarımları (deduction) yaparken, programcının "Derleyici burada gerçekten neyi buldu?" sorusunu sorması çok doğaldır. `typeid` veya `decltype` bazen `const` veya `reference` niteleyicilerini (qualifiers) tam göstermeyebilir (çünkü `typeid` türün çıplak halini raporlamaya meyillidir). Hoca, derleyiciyi hata vermeye zorlayarak en yalın ve doğru tür bilgisini hata mesajında görmemizi sağlayan bir teknik öğretti.

### ⚙️ Teknik Detay ve Sentaks
Bu teknikte, gövdesi olmayan (incomplete type) bir sınıf şablonu kullanılır.

```cpp
template <typename T>
class TypeTeller; // <-- Sadece bildirim (Incomplete Type)

template <typename T>
void func(T x) {
    TypeTeller<T> t; // <-- HATA BURADA: Nesne tanımlanamaz çünkü TypeTeller eksik türdür.
}

int main() {
    const int y = 5;
    func(y); // Derleyici hatası: "aggregate 'TypeTeller<int> t' has incomplete type"
}
```

### 🔍 Arka Plan (Under the Hood)
*   **Call by Value (T x):** Bu senaryoda **Decay** (bozulma) uygulanır. Argüman `const int` olsa bile `T` çıkarımı `int` olur.
*   **Pointer Parametre (T* x):** Eğer argüman bir adres ise, `T` bu adresin gösterdiği nesnenin türü olur.
*   **Reference Parametre (T& x):** Decay uygulanmaz. `const` niteleyicisi korunur.

```cpp
template <typename T>
void func_ref(T& x) {
    TypeTeller<T> t; 
}

int main() {
    int a[5] = {1,2,3,4,5};
    func_ref(a); // <-- Hoca buraya çok dikkat çekti! 
    // Derleyici Gözü: T için yapılan çıkarım 'int[5]' (5 elemanlı int dizi türü).
    // Fonksiyon parametresi ise 'int(&)[5]' (5 elemanlı diziye referans).
}
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `void func(T x)` şablonuna `const char*` (string literali) gönderilirse `T` ne olur? `T& x` olursa ne olur?
**Cevap:** 
1.  `T x` (Değerle): `T` = `const char*` olur (Dizi pointer'a decay oldu).
2.  `T& x` (Referansla): `T` = `const char[N]` olur (Dizi türü korunur, `T` dizinin kendisi olur).

---

## 5. Fonksiyon Çıkarımları ve String Literalleri [45:00 - 55:00]

### ⚙️ Teknik Detay ve Sentaks
Fonksiyon isimleri de diziler gibi "decay" olabilir veya referans yoluyla türleri korunabilir.

```cpp
int my_func(double);

template <typename T>
void call_it(T& arg) { // Referans parametre
    TypeTeller<T> t;
}

int main() {
    call_it(my_func); 
    // Çıkarım: T = int(double) --> Bu bir FONKSİYON TÜRÜDÜR.
    // Parametre: T& = int(&)(double) --> Fonksiyon referansıdır.
}
```

### 🔍 Arka Plan (Under the Hood)
Eğer parametre `T arg` olsaydı, **Function-to-Pointer Conversion** gerçekleşir ve `T` bir "Function Pointer" (Fonksiyon Göstericisi) olurdu (`int(*)(double)`).

### 🚩 Kritik Nokta / Mülakat Sorusu
Aynı isimli şablonların çakışması (Ambiguity):
```cpp
template <typename T>
void bar(T x, T y);

bar("Necati", "Ergin"); // <-- HATA: "Necati" (const char[7]), "Ergin" (const char[6])
// Derleyici şu sebeple kızıyor: T hem const char[7] hem const char[6] olamaz!
```
**Çözüm:** Hoca, bu tür durumlarda argümanların türlerinin tam uyuşması gerektiğini, aksi halde çıkarımın başarısız olacağını vurguladı.

---

## 6. Universal (Forwarding) References ve Reference Collapsing [55:00 - 01:15:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Hem L-value (sol taraf değeri) hem de R-value (sağ taraf değeri) argümanlarını tek bir şablonla, veri kopyalamadan ve tür bilgilerini (const/ref) kaybetmeden kabul edebilmek için tasarlandı.

### ⚙️ Teknik Detay ve Sentaks
**Kural:** `T&&` ifadesinin Universal Reference olması için `T`'nin bir template parametresi olması ve önünde `const` gibi niteleyiciler olmaması şarttır.

**Reference Collapsing (Referans Çöküşü) Kuralları:**
1.  `& + &   -> &`
2.  `& + &&  -> &`
3.  `&& + &  -> &`
4.  `&& + && -> &&` (Sadece iki sağ taraf referansı birleşirse sonuç sağ taraf referansı olur).

### 🖼️ Görselleştirme (ASCII Art)
```text
      Argüman (Value Category)     |   T Çıkarımı   |   Parametre (T&&)
-----------------------------------|----------------|-------------------
L-value (int x)                    |   int&         |   int& && -> int&
R-value (10)                       |   int          |   int&&
```

### 🔍 Arka Plan (Under the Hood)
Hoca, **Scott Meyers**'ın "Universal Reference" terimini uydurduğunu, ancak standartın buna "Forwarding Reference" dediğini belirtti. 
*   **Durumdan Vazife Çıkartmak:** Eğer fonksiyona L-value gönderirseniz, `T` türü "sol taraf referansı" olarak çıkarılır. Bu, C++'ın en istisnai çıkarım kurallarından biridir.

---

## 7. Dizi Boyutunun Çıkarılması (`constexpr` Bağlantısı) [1:15:00 - 1:23:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C stilinde dizi boyutunu `sizeof(a)/sizeof(a[0])` makrosuyla alıyorduk. Ancak bu makroya yanlışlıkla bir pointer gönderilirse sessizce yanlış sonuç üretir (UB - Tanımsız Davranış riski). Şablonlar ile bu işlem hem tip-güvenli (type-safe) hem de sadece dizilerle çalışacak şekilde yapılabilir.

### ⚙️ Teknik Detay ve Sentaks
```cpp
template <typename T, int N>
constexpr int a_size(T(&)[N]) { // Diziye referans alır, N çıkarılır
    return N;
}

int main() {
    int a[20];
    constexpr int size = a_size(a); // size = 20 (Compile Time)
}
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `a_size` fonksiyonuna bir pointer gönderilirse ne olur?
**Cevap:** Syntax hatası oluşur. Çünkü şablon `T(&)[N]` (bir diziye referans) beklemektedir; pointer bir dizi değildir ve çıkarım (deduction) başarısız olur. Bu, C'deki makroya göre muazzam bir avantajdır.

---

## 8. Callable Objects ve Gizli Kısıtlamalar [1:23:00 - 1:35:00]

### ⚙️ Teknik Detay ve Sentaks
Şablonlar bazen parametreleri üzerinde "Hidden Constraints" (Gizli Kısıtlamalar) barındırır.

```cpp
template <typename T>
void process(T x) {
    x(); // <-- T "Callable" (çağrılabilir) bir tür olmalı!
}
```
Eğer `T` türü bir fonksiyon göstericisi (function pointer) değilse veya `operator()` fonksiyonuna sahip bir sınıf (functor) değilse, hata **Instantiation** (somutlaştırma) aşamasında çıkar.

### 📊 Standart Karşılaştırması
*   **C++20 Öncesi:** Hatalar karmaşıktır, "X türü çağrılamaz" gibi mesajlar 200-300 satır sürebilir.
*   **C++20 ve Sonrası:** **Concepts** (Konseptler) ile bu kısıtlamalar şablonun başında belirtilir. Hata mesajları çok daha yalın hale gelir.

### Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:
1.  **Specialization Karışıklığı:** Her farklı `T` için derleyici yeni bir fonksiyon yazar. Bu, kodun sadece "jenerik" görünmesi değil, arka planda türlere özel çoğalmasıdır.
2.  **L-value/R-value Hatası:** Universal reference (`T&&`) parametreli bir fonksiyona sağ taraf değeri gönderildiğinde `T`'nin referans **olmadığına**, sol taraf değeri gönderildiğinde ise `T`'nin referans **olduğuna** dikkat edilmelidir.
3.  **Default Argument Deduction:** Varsayılan parametre değerlerine (default arguments) bakarak `T` çıkarımı yapılamaz.

## 9. Modern `std::swap` ve Zincirleme Somutlaştırma (Instantiation) [01:35:00 - 01:58:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Eski C++ standartlarında `swap` işlemi kopyalama (copy) üzerinden yapılıyordu. Modern C++'da ise nesneleri taşımak (move), özellikle büyük bellek blokları tutan sınıflar için çok daha performanslıdır. Şablonlar, bu işlemi hem jenerik hem de maksimum performanslı hale getirir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, `std::swap` algoritmasının modern ve sadeleştirilmiş bir implementasyonunu gösterdi:

```cpp
template <typename T>
void swap(T& x, T& y) {
    T temp = std::move(x); // <-- Move constructor çağrılır (x artık "move-from state" durumunda)
    x = std::move(y);      // <-- Move assignment
    y = std::move(temp);   // <-- Move assignment
}
```

### 🔍 Arka Plan (Under the Hood)
*   **Zincirleme Instantiation:** Bir fonksiyon şablonu (örneğin `foo`), kendi içinde başka bir şablonu (örneğin `swap`) çağırabilir. Derleyici `foo`yu `int` türü için somutlaştırırken, içinde çağrılan `swap` şablonunu da "durumdan vazife çıkartarak" `int` için somutlaştırmak (instantiate etmek) zorunda kalır.
*   **Move-from State:** Hoca burada kritik bir hatırlatma yaptı; taşınmış bir nesne (x), halen geçerli bir durumdadır ve ona yeni bir atama yapılabilir. Standart kütüphane türleri için bu garanti altındadır.

---

## 10. Dönüş Türü Belirleme ve Trailing Return Type [01:58:00 - 02:11:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
İki farklı türden (örneğin `int` ve `double`) parametre alan bir toplama fonksiyonunda dönüş türünün ne olacağı büyük bir sorundur. 

```cpp
template <typename T, typename U>
??? sum(T x, U y) { return x + y; } // Geri dönüş türü ne olmalı?
```
Eğer dönüş türünü `T` yaparsak `sum(1, 2.5)` çağrısında veri kaybı oluşur. Çözüm, türün parametrelere bakarak `decltype` ile belirlenmesidir. Ancak parametreler daha tanımlanmadan dönüş türü kısmında kullanılamaz (Scope/Kapsam hatası).

### ⚙️ Teknik Detay ve Sentaks
C++11 ile gelen **Trailing Return Type** (Sondan Gelen Geri Dönüş Türü) bu sorunu çözer:

```cpp
// <-- Hoca sondan gelen geri dönüş türüne dikkat çekti
template <typename T, typename U>
auto sum(T x, U y) -> decltype(x + y) { // Artık x ve y kapsam (scope) içindedir!
    return x + y;
}
```

### 📊 Standart Karşılaştırması
| Özellik | C++11 | C++14 ve Sonrası |
| :--- | :--- | :--- |
| **Trailing Return Type** | Zorunluydu. `auto ... -> ...` | Halen kullanışlı (karmaşık türler için). |
| **Auto Return Type** | Desteklenmez. | Desteklenir. `auto sum(T x, U y) { return x + y; }` |

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** "Auto Return Type" (C++14) varken neden halen "Trailing Return Type" kullanalım?
**Cevap:** Fonksiyonun geri döndürdüğü ifade ile `return` satırındaki ifadenin türlerinin **farklı** olmasını isteyebiliriz. Ayrıca fonksiyon göstericileri (function pointers) ve karmaşık dizi referanslarını yazmakta büyük kolaylık sağlar.

---

## 11. Şablonlar ve `std::initializer_list` Farkı [02:11:00 - 02:24:00]

### 🚩 Kritik Nokta / Mülakat Sorusu
Hoca, `auto` tür çıkarımı ile `template` tür çıkarımı arasındaki **tek ve en meşhur** farka dikkat çekti:

```cpp
// 1. auto senaryosu:
auto x = {1, 2, 3}; // x'in türü: std::initializer_list<int>

// 2. template senaryosu:
template <typename T>
void func(T x);

func({1, 2, 3}); // <-- HATA! Derleyici şu sebeple kızıyor: "Couldn't deduce template argument"
```
**Kural:** Küme parantezi (`initializer_list`) ile yapılan ilk değer vermelerde `auto` çıkarım yapabilirken, fonksiyon şablonları bu çıkarımı doğrudan yapamaz. Şablon parametresinin açıkça `std::initializer_list<T>` olarak belirtilmesi gerekir.

---

## 12. Function Template Overloading ve Partial Ordering [02:24:00 - 02:40:00]

### ⚙️ Teknik Detay ve Sentaks
Aynı isimli birden fazla fonksiyon şablonu veya şablon olmayan gerçek fonksiyon bir arada bulunabilir. Buna **Function Template Overloading** denir.

```cpp
template <typename T> void foo(T);    // (1) Genel şablon
template <typename T> void foo(T*);   // (2) Pointerlar için daha spesifik
void foo(int);                        // (3) Şablon olmayan gerçek fonksiyon
```

### 🔍 Arka Plan (Partial Ordering Rules)
Eğer bir fonksiyon çağrısı birden fazla şablona uyuyorsa, derleyici **"Most Specialized"** (en özelleşmiş/dar) olanı seçer. 
*   **Hoca'nın Tanımı:** Bir şablonun kabul ettiği tüm türleri diğeri de kabul ediyorsa, ama ikincisi ilkinin kabul etmediği türleri de (daha genel) kabul ediyorsa; birinci şablon daha "spesifik" kabul edilir ve o seçilir.

```cpp
int x = 10;
foo(&x); // (2) seçilir. Çünkü T* (2), genel T'ye (1) göre daha spesifiktir.
foo(x);  // (3) seçilir. Eğer tam uyum sağlayan gerçek fonksiyon varsa, o "altın madalya" alır.
```

---

## 13. Şablon Kısıtlama: Template Deletion [02:40:00 - 02:42:37]

### 🚩 Mülakat Sorusu / Kritik Nokta
**Soru:** Öyle bir fonksiyon yazın ki sadece `int` türüyle çağrılabilsin, diğer türlerle (double, char vb.) çağrıldığında syntax hatası versin.

**Cevap:** Necati Hoca'nın öğrettiği "Modern C++ Tekniği":
```cpp
void func(int x) { /* ... */ } // Sadece int'i kabul eden gerçek fonksiyon

template <typename T>
void func(T) = delete; // <-- Diğer tüm türler için şablonu sil!

int main() {
    func(10);   // Legal: Gerçek fonksiyon seçilir.
    // func(3.4); // HATA: Şablon seçilir ama o "deleted" (silinmiş).
}
```
**Hoca'nın Deyimi:** "Şablonu delete ederek belirli türlerin dışındakileri reddediyoruz."

---

### Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:
1.  **Scope Hatası:** Dönüş türünü belirlerken parametre isimlerini `decltype` içinde kullanabilmek için Trailing Return syntax'ına muhtaç olduğumuzu unutmayın.
2.  **Partial Ordering Karmaşası:** `T` ve `T*` arasındaki seçimde derleyicinin "en dar" olanı seçeceğini bilin.
3.  **Terminoloji Hatası:** "Template Function" değil, **"Function Template"** demeliyiz. Çünkü elimizdeki şey bir fonksiyon değil, fonksiyon üreten bir şablondur.

📌 **Dersin Sonu.**



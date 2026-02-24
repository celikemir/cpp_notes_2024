# 23. Ders: Kalıtım (Inheritance) - Giriş ve Temel Kavramlar

Bu doküman, Necati Ergin'in C++ dersinin 23. gününe ait teknik notlarını ve "Özetleme, Yeniden İnşa Et!" prensibiyle hazırlanmış derinlemesine incelemesini içermektedir.

---

## 🟢 Bölüm 1: Kalıtım Mantığı ve OOP Felsefesi [00:00 - 10:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Yazılım tasarımında varlıklar arasındaki ilişkileri modellemek için sadece `Composition` (Kompozisyon/İçerme - *Has-a*) yeterli değildir. Bir nesnenin hem kendi özelliklerini taşıması hem de daha genel bir türün tüm davranışlarını sergilemesi (Örn: Her Aslan bir Hayvandır) gerekir. Kalıtım; kod tekrarını önlemek, ortak bir `Interface` (Arayüz) sağlamak ve tasarımın "yukarıdan aşağıya" (Top-down) yapılabilmesine olanak tanımak için getirilmiştir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, kalıtımın sadece OOP (Nesne Yönelimli Programlama) aracı olmadığını, C++'da `Generic Programming` (Jenerik Programlama) tarafında da yoğun kullanıldığını ve OOP'yi "aşkın" (transcendent) bir araç olduğunu vurguladı.

```cpp
// Has-a Relationship (Composition) - Önceki derse atıf
class Engine {};
class Car {
    Engine m_engine; // <-- Hoca: "Arabanın motoru var" (Has-a)
};

// Is-a Relationship (Inheritance) - Yeni konu
class Animal {
public:
    void eat();
};

class Lion : public Animal { // <-- Hoca buraya dikkat çekti: "Aslan bir hayvandır" (Is-a)
    // Lion sınıfı, Animal'ın public arayüzünü devralır.
};
```

### 🔍 Arka Plan (Under the Hood)
Kalıtımda, `Derived Class` (Türemiş Sınıf) nesnesi fiziksel olarak içinde bir `Base Class` (Taban Sınıf) nesnesi barındırır. Bu, bellekte türemiş sınıf nesnesinin içinde taban sınıf nesnesinin sub-object (alt nesne) olarak yerleşmesi demektir.

### 📊 Standart Karşılaştırması: Kalıtım ve Alternatifleri
C++'da kalıtım tek seçenek değildir. Hoca modern C++ ile gelen alternatiflere dikkat çekti:

| Özellik | C++ Standardı | Açıklama |
| :--- | :--- | :--- |
| **Runtime Polymorphism** | C++98/Modern | Kalıtım ve Sanal Fonksiyonlar yoluyla. |
| **Type Erasure** | Modern C++ | Kalıtım hiyerarşisi kurmadan polimorfik yapı. |
| **std::variant** | C++17 | `Sum types` ile kalıtıma alternatif sağlayan yapı. |
| **CRTP (Curious Recurring Template Pattern)** | C++98/Modern | Statik polimorfizm (Generic Programming tarafı). |

### 🖼️ Görselleştirme (ASCII Art)
**Interface Katılımı:**
```text
[ Animal Interface ]  <-- (eat, breathe)
         ^
         | (Public Inheritance)
[  Lion Interface  ]  <-- (eat, breathe) + (roar)
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Her kalıtım ilişkisi bir "Is-a" ilişkisi midir?
**Cevap:** C++'da teknik olarak `Private` ve `Protected` kalıtım da vardır ancak bunlar "Is-a" ilişkisi değil, daha çok "Implemented-in-terms-of" (As-a) ilişkisidir. "Is-a" ilişkisi sadece `Public Inheritance` ile sağlanır.

> **Bu 10 dakikalık bölümde Hoca şu 3 kritik hataya dikkat çekti:**
> 1. Kalıtımı her derdin ilacı sanmak ve alternatifleri (Type Erasure, Variant) görmezden gelmek.
> 2. `Composition` (*Has-a*) ile `Inheritance` (*Is-a*) arasındaki farkı karıştırmak.
> 3. Kalıtımın sadece OOP için olduğunu sanmak (Generic Programming gücünü azımsamak).

---

## 🔵 Bölüm 2: Terminoloji ve Kalıtım Türleri [10:00 - 20:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Modelleme dillerinde (UML vb.) ve diller arası iletişimde standart bir terminolojiye ihtiyaç vardır. C++'ın kendine has terimlerini bilmek, derleyici hatalarını anlamak için kritiktir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, dilden bağımsız terimler ile C++ terimlerini karşılaştırdı.

*   **Genel OOP Terimleri:** `Parent Class` / `Superclass` -> `Child Class` / `Subclass`
*   **C++ Terimleri:** `Base Class` (Taban Sınıf) -> `Derived Class` (Türemiş Sınıf)

```cpp
class Base { // Kaynak sınıf (Taban)
};

class Der : public Base { // Kalıtımla elde edilen sınıf (Türemiş)
};
```

Hoca, Türkçe'de "kalıtmak" diye bir fiil olmadığını, bunun yerine **"türetme"** (derivation) teriminin kullanılmasının daha doğru olduğunu belirtti.

### 🖼️ Görselleştirme (UML Mantığı)
Hoca UML diagramlarındaki geleneksel gösterime değindi:

```text
      +-------+
      |  Car  |  <--- [Base Class / Taban Sınıf]
      +-------+
          ^
          | (Ok yönü her zaman Taban sınıfa doğrudur!)
    +-----+-----+
    |           |
+-------+   +-------+
| Audi  |   | Volvo | <--- [Derived Classes / Türemiş Sınıflar]
+-------+   +-------+
```

### 🔗 Kümülatif Bağlantılar
Hoca, "İleride göreceğimiz `Runtime Polymorphism` (Çalışma zamanı çok biçimliliği) konusunun temelinin bu `Is-a` ilişkisi olduğunu" belirtti.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** "Bir Audi bir arabadır" cümlesi çift yönlü müdür?
**Cevap:** Hayır. `Is-a` ilişkisi tek yönlüdür. Her Audi bir arabadır (`Car`), ancak her araba bir `Audi` değildir. Bu durum, pointer dönüşümlerinde (Upcasting) karşımıza çıkar.

---

## 🟡 Bölüm 3: Kalıtım Kategorileri ve Incomplete Type Kuralı [20:00 - 27:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Sınıflar arasındaki hiyerarşinin derinliği ve genişliği arttıkça, bu ilişkilerin nasıl kategorize edileceğinin netleşmesi gerekir. Ayrıca derleyicinin kalıtım sırasında nesne boyutunu hesaplayabilmesi için sınıfa dair tam bilgiye ihtiyacı vardır.

### ⚙️ Teknik Detay ve Sentaks
Hoca kalıtımı seviyelerine göre üçe ayırdı:
1.  **Single Inheritance (Tekli Kalıtım):** Bir sınıfın bir taban sınıftan türetilmesi (Tek seviye).
2.  **Multi-level Inheritance (Çok Seviyeli Kalıtım):** Türetilen sınıftan da başka sınıfların türetilmesi.
    *   `Car` <- `Mercedes` <- `S500` (Her S500 bir Mercedes'tir, her Mercedes bir Car'dır).
3.  **Multiple Inheritance (Çoklu Kalıtım):** Bir sınıfın birden fazla taban sınıftan aynı anda türetilmesi. (C++'da doğrudan desteklenir).

```cpp
// Multiple Inheritance Örneği
class Printer {};
class Scanner {};

class Combo : public Printer, public Scanner { 
    // <-- Hoca: "Her Combo hem bir Printer'dır hem bir Scanner'dır"
};
```

**⚠️ Kritik Kural: Incomplete Type (Tamamlanmamış Tür)**
Bir sınıfın taban sınıf (Base Class) olarak kullanılabilmesi için o noktada `Complete Type` (Tamamlanmış Tür) olması şarttır.

```cpp
class Base; // Forward Declaration (Ön Bildirim) - Incomplete Type

class Der : public Base { // <-- DERLEYİCİ HATASI!
    // Derleyici şu sebeple kızıyor: Taban sınıfın boyutu bilinmiyor.
};

class Base { // Şimdi Complete Type oldu
    int x;
};

class Der2 : public Base { // <-- GEÇERLİ
};
```

### 🔍 Arka Plan (Memory Layout)
Çok seviyeli kalıtımda (`Car` -> `Mercedes` -> `S500`), bir `S500` nesnesi bellekte oluşturulduğunda sırasıyla en üstten alta doğru tüm taban sınıf nesnelerini içerecek şekilde büyür.

> **Bu 7 dakikalık bölümde Hoca şu 3 kritik hataya dikkat çekti:**
> 1. `Multi-level Inheritance` ile `Multiple Inheritance` terimlerini birbirine karıştırmak.
> 2. Sadece `Forward Declaration` yapılmış (Incomplete Type) bir sınıftan türetme yapmaya çalışmak.
> 3. Kalıtım diagramlarında ok yönünü Taban sınıftan Türemiş sınıfa doğru çizmek (Doğrusu: Türemiş -> Taban).

## 🟠 Bölüm 4: Kalıtım Belirteçleri ve Varsayılan Kurallar [27:00 - 37:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C++'da bir sınıftan türetme yaparken sadece arayüzü değil, bu arayüzün dış dünyaya karşı olan "erişim statüsünü" de belirlemek gerekir. "Is-a" ilişkisi her zaman arzu edilen şey olmayabilir; bazen bir sınıfın kodlarını kullanmak isteriz ama onun kimliğini dışarı sızdırmak istemeyiz.

### ⚙️ Teknik Detay ve Sentaks
C++'da üç farklı kalıtım modeli (`Inheritance Specifiers`) vardır. Hoca, aksi belirtilmediği sürece "Inheritance" denildiğinde bunun `Public Inheritance` olarak anlaşılması gerektiğini vurguladı.

```cpp
class Base {};

// 1. Public Inheritance (Açık Kalıtım)
class Der1 : public Base {}; // <-- Is-a relationship. %85-90 bu kullanılır.

// 2. Private Inheritance (Özel Kalıtım)
class Der2 : private Base {}; // <-- Has-a / As-a relationship.

// 3. Protected Inheritance (Korunmuş Kalıtım)
class Der3 : protected Base {}; // <-- Çok nadir kullanılır.
```

**⚠️ Kritik Kural: Varsayılan Belirteçler (Defaults)**
Hoca, `class` ve `struct` anahtar sözcüklerinin kalıtımdaki varsayılan davranışlarını karşılaştırdı:

```cpp
class Base {};

class Der : Base {}; 
// <-- Hoca buraya dikkat çekti: "Belirteç yazılmazsa varsayılan PRIVATE kalıtımdır."

struct DerStruct : Base {}; 
// <-- Hoca buraya dikkat çekti: "Belirteç yazılmazsa varsayılan PUBLIC kalıtımdır."

struct DerStruct2 : private Base {}; // <-- Şaşırtıcı ama geçerli (Legal).
```

### 🔍 Arka Plan (Under the Hood)
Derleyici, `class` gördüğünde "kısıtlayıcı", `struct` gördüğünde ise "açık" olma durumundan vazife çıkartır. Bu durum hem üye erişiminde hem de kalıtım biçiminde tutarlıdır.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `class A : B {};` ile `struct A : B {};` arasındaki fark nedir?
**Cevap:** İlkinde `A`, `B`'den `private` olarak türetilir ve `A`'nın tüm üyeleri varsayılan olarak `private`'dır. İkincisinde ise `A`, `B`'den `public` olarak türetilir ve `A`'nın üyeleri varsayılan olarak `public`'tir.

---

## 🔴 Bölüm 5: Public Arayüz Katılımı ve "Ya Hep Ya Hiç" [37:00 - 42:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Public kalıtım, taban sınıfın tüm `Public Interface`'ini türemiş sınıfa taşır. Ancak bazen programcılar taban sınıftan "seçerek" fonksiyon almak isterler. Hoca bunun neden imkansız olduğunu açıkladı.

### ⚙️ Teknik Detay ve Sentaks
Public kalıtımda taban sınıfın tüm `public` üyeleri (fonksiyonlar, değişkenler, nested type'lar) türemiş sınıfın arayüzüne dahil olur.

```cpp
class Base {
public:
    void foo();
    void bar();
    int m_x;
    using value_type = int; // Nested Type
    static double s_val;    // Static Member
};

class Der : public Base {};

int main() {
    Der myder;
    myder.foo();         // Geçerli
    myder.m_x = 5;       // Geçerli
    Der::value_type v;   // Geçerli (Nested type miras alındı)
    Der::s_val = 1.2;    // Geçerli (Static member miras alındı)
}
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Taban sınıfın 10 fonksiyonu var ama ben sadece 8'ini `public` olarak almak istiyorum. Bunu kalıtımla yapabilir miyim?
**Cevap:** Hayır. Hoca'nın deyimiyle: **"Ya hep ya hiç!"**. Eğer arayüzü daraltmak (selective inheritance) istiyorsanız, kalıtım yanlış araçtır; `Composition` kullanmalısınız.

---

## 🟣 Bölüm 6: Fiziksel Yerleşim ve `sizeof` Analizi [42:00 - 50:00]

### 🔍 Arka Plan (Memory Layout)
Hoca, `Composition` (Kompozisyon) ile `Inheritance` (Kalıtım) arasındaki fiziksel benzerliği `sizeof` operatörü ile ispatladı. Türemiş sınıf nesnesi, taban sınıf nesnesini fiziksel bir parça olarak içinde taşır.

```cpp
class Base {
    int x, y; // 8 byte (varsayılan)
};

// Senaryo 1: Composition (Has-a)
class Comp {
    int a, b;
    Base bx; // <-- İçerilen nesne (Member Object)
}; // sizeof(Comp) = 16 byte

// Senaryo 2: Inheritance (Is-a)
class Der : public Base {
    int a, b;
}; // sizeof(Der) = 16 byte! 
   // <-- Hoca: "Fiziksel yapı neredeyse aynıdır."
```

### 🖼️ Görselleştirme (Memory Map)
```text
[ Derived Object (Der) ]
+-----------------------+
|   [ Base Sub-object ] |  <-- Önce bu parça (x, y)
+-----------------------+
|   [ Der's Members ]   |  <-- Sonra bu parça (a, b)
+-----------------------+
```

### ⚙️ Teknik Detay: İnşaa Sırası
Hoca, `Constructor` çağrı sırasını bir kod örneğiyle gösterdi. Bir türemiş sınıf nesnesi oluşturulduğunda **önce taban sınıfın sub-object'i** hayata gelir.

> **Bu 8 dakikalık bölümde Hoca şu 3 kritik hataya dikkat çekti:**
> 1. Türemiş sınıf nesnesinin bellekte taban sınıftan bağımsız bir yerde durduğunu sanmak.
> 2. `sizeof(Derived)` değerinin sadece türemiş sınıfın kendi üyelerini kapsayacağını düşünmek.
> 3. Taban sınıf nesnesi (Base sub-object) ile üye nesnenin (Member object) bellekteki yerleşim hiyerarşisini karıştırmak.

---

## 🔵 Bölüm 7: İsim Arama (Name Lookup) ve İsim Gizleme [50:00 - 01:04:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Taban ve türemiş sınıflarda aynı isimli üyeler bulunabilir. Derleyicinin bu isimleri hangi öncelikle arayacağını (`Lookup Rules`) bilmek, C++'ın en temel kurallarından biridir.

### ⚙️ Teknik Detay ve Sentaks
İsim arama her zaman **aşağıdan yukarıya (Türemişten Tabana)** doğru yapılır. İsim bulunduğu anda arama durur.

```cpp
class Base {
public:
    void foo(double); // <-- Gizlenen fonksiyon
};

class Der : public Base {
public:
    void foo(int); // <-- Gizleyen (Hiding) fonksiyon
};

int main() {
    Der myder;
    myder.foo(3.14); // <-- DİKKAT: Der::foo(int) çağrılır! 
    // Hoca: "Bu Overloading DEĞİLDİR. Bu bir Name Hiding durumudur."
}
```

### 🔍 Arka Plan (Name Hiding/Shadowing)
C++'da `Overloading` olması için fonksiyonların **aynı scope'ta** (kapsamda) olması gerekir. Taban ve türemiş sınıfların kapsamları farklıdır. Türemiş sınıftaki bir isim, taban sınıftaki aynı ismi **maskeler/gizler (Shadowing)**.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Taban sınıftaki gizlenmiş bir fonksiyona nasıl erişirim?
**Cevap:** `Scope Resolution` (Kapsam Çözünürlük) operatörü ile: `myder.Base::foo(3.14);`

---

## 🟡 Bölüm 8: Erişim Kontrolü vs İsim Arama Tuzağı [01:04:00 - 01:11:00]

### 🚩 Kritik Nokta / Mülakat Sorusu
Bu bölüm, Hoca'nın "mülakatlarda en çok can yakan yer" dediği kısımdır. Derleyici önce isme bakar, sonra erişim iznine (`Access Control`).

```cpp
class Base {
public:
    void func(int); 
};

class Der : public Base {
private:
    void func(double); // <-- Private yapıldı!
};

int main() {
    Der myder;
    myder.func(10); // <-- HATA! 
}
```
**Derleyici şu sebeple kızıyor:**
1.  **Name Lookup:** `func` ismini arar. `Der` sınıfında bulur ve aramayı bitirir (Taban sınıfa hiç bakmaz).
2.  **Context Control:** Parametre uyumuna bakar.
3.  **Access Control:** Bulunan ismin `private` olduğunu görür ve hata verir.

**Hoca'nın İdiomu:** *"Derleyici isme aşıktır, ismin private olmasına sonra kızar."* (Önce bulur, sonra erişimi kontrol eder).

## 🟢 Bölüm 9: Upcasting, Downcasting ve Tehlikeli Sular: Object Slicing [01:11:00 - 01:21:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
"Is-a" ilişkisinin dildeki en büyük avantajı, farklı türetilmiş nesneleri (Mercedes, Audi, Volvo) ortak bir paydada (Car) toplayabilmektir. Bu, genel amaçlı fonksiyonlar yazmamızı sağlar. Ancak bu dönüşümün fiziksel kısıtlamaları (slicing) risk oluşturur.

### ⚙️ Teknik Detay ve Sentaks
Hoca, türemiş sınıftan taban sınıfa yapılan dönüşümleri (Upcasting) ve tersini (Downcasting) inceledi.

```cpp
class Car {};
class Mercedes : public Car {};

Mercedes m;
// 1. Upcasting (Implicit/Örtülü) - HER ZAMAN GÜVENLİ
Car* pCar = &m;        // Pointer semantiği
Car& rCar = m;         // Referans semantiği

// 2. Downcasting (Explicit/Açık) - RİSKLİ
Car myCar;
// Mercedes* pMerc = &myCar; // <-- HATA: Taban sınıftan türemişe örtülü dönüşüm yok.
// Hoca: "Her Mercedes bir arabadır ama her araba bir Mercedes değildir."

// 3. Object Slicing (Nesne Dilimlenmesi) - TEHLİKELİ
Car c = m; // <-- Hoca: "Legaldir ama yapmayın!" 
// Mercedes'in Car'dan fazla olan özellikleri (sunroof vb.) "dilimlenir" ve atılır.
```

### 🔍 Arka Plan (Memory Layout)
Upcasting sırasında derleyici aslında pointer'ın değerini fiziksel olarak değiştirmez (çoğu durumda), sadece o adresteki veriye "Base" gözlüğüyle bakmaya başlar. Ancak **Object Slicing** olduğunda, `m` nesnesinin içindeki `Car` parçası, yeni bir `Car` nesnesine kopyalanır ve geri kalan veri bellekte kaybolur.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Object Slicing neden kötüdür?
**Cevap:** Nesne kimliğini kaybeder. Eğer taban sınıfın kopyalama fonksiyonları (`Copy Constructor/Assignment`) polimorfik olmayan bir şekilde çağrılırsa, nesne sadece taban sınıf verileriyle hayata devam eder ve türemiş sınıfın davranışlarını (ileride göreceğimiz virtual functions) sergileyemez.

---

## 🔵 Bölüm 10: İnşa ve İmha Sırası (Order of Construction/Destruction) [01:21:00 - 01:43:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir evin temeli atılmadan çatısı çıkılamayacağı gibi, bir nesne de "tabanından" (base) başlanarak inşa edilmelidir. Türemiş sınıfın üyeleri, taban sınıfın hazır olduğunu varsayabilir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, nesnelerin hayata gelme hiyerarşisini adım adım kodladı:

```cpp
class Base {
public:
    Base() { std::cout << "Base Constructor\n"; }
    ~Base() { std::cout << "Base Destructor\n"; }
};

class Member {
public:
    Member() { std::cout << "Member Constructor\n"; }
    ~Member() { std::cout << "Member Destructor\n"; }
};

class Der : public Base {
    Member m;
public:
    Der() { std::cout << "Derived Constructor\n"; } // <-- Programın akışı en son buraya girer.
    ~Der() { std::cout << "Derived Destructor\n"; }
};
```

**⚠️ Hoca'nın Vurguladığı Kesin Sıra (Construction):**
1.  **Base sub-object** (Taban sınıf nesnesi) hayata gelir.
2.  **Member objects** (Sınıfın veri elemanları) bildirim sırasına göre hayata gelir.
3.  **Derived Constructor Body** (Türemiş sınıfın ana bloğu) çalıştırılır.

**⚠️ Destruction (İmha) Sırası:**
Tam tersi! (Türemiş -> Member -> Base). Hoca'nın deyimiyle: *"İlk doğan, son ölür."*

### 🔍 Arka Plan (The "this" Pointer)
Derleyici, türemiş sınıfın fonksiyonlarını çağırırken `this` pointer'ını taban sınıfın başlangıç adresine otomatik olarak dönüştürür (Upcast). Türemiş sınıf nesnesi, bellekte taban sınıf nesnesini tam olarak kapsadığı için bu işlem çok hızlıdır.

---

## 🟡 Bölüm 11: MIL ve Taban Sınıf Constructor Çağrıları [01:43:00 - 02:08:00]

### ⚙️ Teknik Detay ve Sentaks
Türemiş sınıf, taban sınıfın hangi constructor'ının çağrılacağını `MIL` (Member Initializer List) üzerinden belirler. Eğer belirtilmezse, derleyici varsayılan olarak `Base()` çağrısını ekler.

```cpp
class Base {
public:
    Base(int x) { /*...*/ } // Default Constructor YOK!
};

class Der : public Base {
public:
    // Der() {} // <-- HATA: Derleyici Base'in default constructor'ını bulamaz.
    Der(int val) : Base(val) {} // <-- DOĞRU: Taban sınıfı açıkça initialize ettik.
};
```

**🚩 Kritik Kural: Deleted Constructor**
Hoca mülakat sorusu kıvamında bir noktaya değindi: Eğer taban sınıfın `Default Constructor`'ı `private` ise veya `delete` edilmişse, türemiş sınıfın derleyici tarafından yazılan constructor'ı da otomatik olarak **delete** edilir.

### 🔗 Kümülatif Bağlantılar
Hoca bu konuyu anlatırken `person`, `VIP_person` gibi gerçek hayat senaryolarını kullanarak `string` sınıfı (önceki ders) ile MIL sentaksını birleştirdi.

---

## 🔴 Bölüm 12: Kopya/Taşıma ve `using` Bildirimi [02:08:00 - 02:44:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Türemiş sınıfın kopya kontrolünü (`Copy/Move`) biz yazdığımızda, taban sınıfın kopyalanmasından da biz sorumlu oluruz. Eğer unutursak, taban sınıf sadece "default" değerlerle hayata gelir; bu da büyük bir mantık hatasıdır (Bug).

### ⚙️ Teknik Detay: Copy/Move Constructor Yazımı
Hoca, türemiş sınıfta kopya constructor yazarken yapılan en yaygın hatayı gösterdi:

```cpp
class Der : public Base {
public:
    Der(const Der& other) : Base(other) { // <-- KRİTİK: Base'in copy constructor'ını çağırmak için upcasting kullandık.
        // Hata: Eğer ': Base(other)' demezsek, Base'in DEFAULT constructor'ı çağrılır!
    }

    Der(Der&& other) : Base(std::move(other)) { // <-- Move için move cast şart.
    }
};
```

### ⚙️ Teknik Detay: `using` Bildirimi (Name Injection) [02:40:00]
Hoca dersin sonunda "İsim Gizleme" (Name Hiding) problemini çözen harika bir araç gösterdi: `using` bildirimi.

```cpp
class Base {
public:
    void foo(int);
};

class Der : public Base {
public:
    using Base::foo; // <-- Hoca: "Base'deki foo'yu buraya enjekte et!"
    void foo(double);
};

// Artık Der nesnesi üzerinden foo(int) çağrılabilir. Overloading etkisini sağladık.
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Çok seviyeli kalıtımda (`A <- B <- C`), `C` sınıfı `MIL` üzerinden `A`'nın constructor'ını çağırabilir mi?
**Cevap:** Hayır. Hoca: *"Sadece direkt taban sınıfın (Direct Base) constructor'ı çağrılabilir."* (Sanal kalıtım istisnası hariç).

---

> **Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
> 1. Kopya constructor yazarken taban sınıfı `MIL` listesinde kopyalamayı unutmak (Taban sınıfın default değerlerle kalması).
> 2. `Move assignment` operatöründe taban sınıfı `std::move` ile çağırmamak.
> 3. İsim gizleme problemini her seferinde `Base::foo()` diye niteleyerek çözmeye çalışmak (Çözüm: `using` bildirimidir).



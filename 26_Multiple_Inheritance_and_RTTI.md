### 26. Ders: Çoklu Kalıtım (Multiple Inheritance) ve Nesne Modelleri - Bölüm 1

**Ders Tarihi:** 25 Eylül 2024  
**Konu:** Multiple Inheritance (Çoklu Kalıtım), Ambiguity (Belirsizlik), Name Lookup (İsim Arama) ve Construction Order (İnşa Sırası)

---

#### 🧠 1. Çoklu Kalıtım Nedir? (Rationale)
**Neden İhtiyaç Duyuldu? (Rationale):** Gerçek dünyada bazı nesneler birden fazla kimliğe sahip olabilir. Örneğin bir cihaz hem "Yazıcı" (Printer) hem de "Tarayıcı" (Scanner) olabilir. C++ öncesi dillerde bu durum genellikle her sınıfı tek tek elde ederek (single inheritance) çözülmeye çalışılıyordu. Ancak bu, "is-a" (dir/dır) ilişkisini tam yansıtmıyordu. Çoklu kalıtım, bir sınıfın birden fazla taban sınıfın (base class) arayüzünü (interface) ve özelliklerini tek seferde devralmasını sağlar.

**Kritik Ayrım:**  
*   **Multi-level Inheritance (Çok Seviyeli Kalıtım):** `A <- B <- C` (Bir sınıftan türetilenin de türetilmesi).
*   **Multiple Inheritance (Çoklu Kalıtım):** `(A, B) <- C` (İki ayrı sınıfın tek bir sınıfta birleşmesi).

🖼️ **Görselleştirme (ASCII Art):**
```text
  Multi-level:            Multiple:
  [ Car ]                [ Printer ]   [ Scanner ]
     ^                        ^           ^
     |                        |           |
  [ Volvo ]                +-------+-------+
     ^                             |
     |                         [ Combo ]
  [ XC90 ]
```

---

#### ⚙️ 2. Çoklu Kalıtım Sentaksı ve Temel Kurallar [07:24 - 11:35]

Çoklu kalıtımda en sık yapılan hata, erişim belirleyicinin (access specifier) sadece ilk sınıfı nitelediğini sanmaktır.

```cpp
class XBase { public: void foo(); };
class YBase { public: void bar(); };

// <-- Hoca buraya dikkat çekti: public her sınıf için ayrı yazılmalı!
class Der : public XBase, public YBase { 
public:
    void bom();
};

int main() {
    Der myder;
    myder.foo(); // XBase'den geldi
    myder.bar(); // YBase'den geldi
    myder.bom(); // Kendisinden geldi
}
```

🚩 **Mülakat Sorusu / Kritik Nokta:**
**Soru:** `class Der : public XBase, YBase {};` yazarsak ne olur?
**Cevap:** C++ kurallarına göre `class` anahtar sözcüğüyle kalıtım varsayılan olarak **private**'tır. Bu durumda `XBase` public, `YBase` ise **private** olarak kalıtılır. Eğer her ikisinin de public olmasını istiyorsak, `public` sözcüğünü virgülle ayrılan her sınıfın başına tekrar yazmalıyız. (`struct` kullanılsaydı varsayılan `public` olurdu).

---

#### 🔍 3. Çoklu Kalıtımda Upcasting ve Virtual Dispatch [11:36 - 15:12]

Çoklu kalıtımda "is-a" (biridir) ilişkisi tüm taban sınıflar için geçerlidir.

⚙️ **Teknik Detay ve Sentaks:**
```cpp
Der my_der;
XBase* xp = &my_der; // Implicit Upcasting (Örtülü yukarıya dönüşüm) - Legal
YBase* yp = &my_der; // Implicit Upcasting - Legal

// Referans semantiği için de geçerli
XBase& xr = my_der;
YBase& yr = my_der;
```

**Arka Plan (Under the Hood):**
Çoklu kalıtımda `xp` ve `yp` pointer'ları aslında bellekte **aynı adresi göstermeyebilir.** C++ derleyicisi, `yp` pointer'ını oluştururken `my_der` nesnesinin başlangıç adresine bir **offset** (kayma miktarı) ekleyerek onu nesne içindeki `YBase` kısmına konumlandırır. Bu, C++ nesne modelinin en karmaşık ve güçlü yanlarından biridir.

---

#### 🔗 4. İsim Arama ve Belirsizlik (Name Lookup & Ambiguity) [17:43 - 21:38]

En büyük problem: Her iki taban sınıfta da aynı isimli bir fonksiyon veya değişken varsa ne olur?

```cpp
class XBase { public: void foo(int); };
class YBase { public: void foo(double); };

class Der : public XBase, public YBase {};

int main() {
    Der myder;
    // myder.foo(10); // <-- HATA: Ambiguous (Belirsiz). 
    // Derleyici şu sebeple kızıyor: İsim arama (Name Lookup), 
    // overload resolution'dan (yükleme çözünürlüğü) önce yapılır.
}
```

🔍 **Arka Plan (Under the Hood):**
C++'ta isim arama sırası şöyledir:
1.  Türemiş sınıf (Derived) içine bakılır.
2.  Bulunamazsa taban sınıflara (Base) **aynı anda** bakılır.
3.  Eğer birden fazla taban sınıfta aynı isim bulunursa, parametreler uysun ya da uymasın, derleyici "Ben hangisini seçeceğimi bilmiyorum" diyerek hata verir. Bu bir **function overloading** değildir, bir **ambiguity** (belirsizlik) durumudur.

---

#### 🛠️ 5. Belirsizliğin Çözümü (Using Declarations) [21:39 - 25:17]

Hoca, bu durumu "durumdan vazife çıkartmak" olarak niteler ve iki temel çözüm sunar:

**Çözüm A: Tam Niteleme (Full Qualification)**
```cpp
myder.XBase::foo(10); // Açıkça hangi sınıfın fonksiyonu olduğu belirtilir.
```

**Çözüm B: `using` Bildirimi (Modern C++ Yaklaşımı)**
Sınıf içine `using` ekleyerek bu isimleri türemiş sınıfın kapsamına (scope) "enjekte" ederiz.

```cpp
class Der : public XBase, public YBase {
public:
    // C++17 öncesi ayrı satırlarda yazılıyordu
    using XBase::foo; 
    using YBase::foo; 
    
    // Modern C++ (C++17 ve sonrası):
    // using XBase::foo, YBase::foo; // <-- "Comma separated list" (Virgülle ayrılmış liste)
};

// Artık bu legal:
// myder.foo(10);   --> XBase::foo çağrılır.
// myder.foo(1.2);  --> YBase::foo çağrılır. (Overload resolution artık çalışır!)
```

---

#### 🏗️ 6. Bellek Yerleşimi ve İnşa Sırası (Object Layout & Construction) [25:18 - 31:10]

**Kritik Kural:** Çoklu kalıtımda taban sınıflar, **kalıtım listesinde yazıldıkları sıraya göre** inşa edilirler. Constructor Initializer List (MIL) içindeki sıra bu kuralı değiştirmez!

⚙️ **Teknik Detay ve Sentaks:**
```cpp
class Der : public XBase, public YBase { // <-- Önce XBase, sonra YBase kurulur.
public:
    Der() : YBase(), XBase() { // <-- DİKKAT: Burada YBase'in önce yazılması önemsizdir!
        // Gövde
    }
};
```

🖼️ **Bellek Yapısı (ASCII Art):**
```text
[   XBase Part   ]  (Düşük Adres)
[   YBase Part   ]
[    Der Part    ]  (Yüksek Adres)
```

**Mülakat Sorusu / Kritik Nokta:**
**Soru:** Yıkıcılar (Destructors) hangi sırada çağrılır?
**Cevap:** İnşa sırasının tam tersi. Önce `Der`, sonra `YBase`, en son `XBase`.

---

#### 🚩 Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:
1.  **Erişim Belirleyici İhmali:** `class D : B1, B2` yazımında B2'nin gizlice `private` kalması.
2.  **Overload vs Ambiguity:** Farklı taban sınıflardaki aynı isimli fonksiyonların "overload" oluşturmadığını, "isim arama çatışması" (ambiguity) oluşturduğunu unutmak.
3.  **MIL Aldatmacası:** İnşa sırasının MIL (Member Initializer List) sırasına değil, sınıf tanımındaki kalıtım sırasına bağlı olması.

---

### 26. Ders: Çoklu Kalıtım (Multiple Inheritance) ve Nesne Modelleri - Bölüm 2

**Ders Akışı:** Çoklu Kalıtımda Erişim Kontrolü, Belirsizlik (Ambiguity) Senaryoları ve "Korkunç Türetme Elması" (Dreadful Diamond of Derivation).

---

#### 🛠️ 1. Çoklu Kalıtımda Erişim Kontrolü ve Örtülü Dönüşümler [31:10 - 34:30]

Çoklu kalıtımda her taban sınıfın erişim belirleyicisi bağımsızdır. Bu durum, upcasting (yukarıya dönüşüm) imkanını doğrudan etkiler.

**⚙️ Teknik Detay ve Sentaks:**
```cpp
class X { public: void foo(); };
class Y { public: void bar(); };

// X: public, Y: private (anahtar sözcük yazılmadığı için default private)
class D : public X, Y { }; 

int main() {
    D dx;
    dx.foo(); // Legal: X public taban sınıf.
    // dx.bar(); // <-- HATA: 'bar' is inaccessible. Y private taban sınıftır.

    X* xp = &dx; // Legal: Public upcasting.
    // Y* yp = &dx; // <-- HATA: 'Y' is an inaccessible base of 'D'.
    // Derleyici şu sebeple kızıyor: Private inheritance "is-a" ilişkisi kurmaz (sadece içeride kullanılır).
}
```

---

#### 🔗 2. Fonksiyon Yüklemesinde (Overloading) Belirsizlik [34:31 - 38:00]

Eğer iki farklı taban sınıfa upcast olabilen bir nesne, bu iki türü de parametre olarak kabul eden bir overload setine gönderilirse belirsizlik oluşur.

**⚙️ Teknik Detay ve Sentaks:**
```cpp
void func(const X&);
void func(const Y&);

class D : public X, public Y {};

int main() {
    D dx;
    // func(dx); // <-- HATA: Ambiguous call. 
    // Derleyici şu sebeple kızıyor: DX nesnesi hem X'e hem Y'ye aynı "yakınlıkta" (eligibility) dönüşebilir.
    
    // Çözüm: Explicit Cast (Açık Tür Dönüşümü)
    func(static_cast<const X&>(dx)); // Legal: X overload'u çağrılır.
}
```

---

#### 🧠 3. Elmas Oluşumu (The Diamond Problem / 3D Formation) [38:01 - 49:00]

**Neden İhtiyaç Duyuldu? (Rationale):** Bazen iki taban sınıf, ortak bir üst taban sınıftan türemiş olabilir. Bu durum C++'ta "Dreadful Diamond of Derivation" (Korkunç Türetme Elması) veya "3D Formation" olarak adlandırılır.

**⚙️ Teknik Detay ve Sentaks:**
```cpp
class Device { public: bool m_flag; };
class Printer : public Device {};
class Scanner : public Device {};
class Combo : public Printer, public Scanner {};
```

🔍 **Arka Plan (Under the Hood / Memory Layout):**
Eğer özel bir önlem alınmazsa (default durum), `Combo` nesnesinin içinde **iki adet** `Device` nesnesi oluşur.

🖼️ **Görselleştirme (ASCII Art):**
```text
      [ Device ]          [ Device ]  <-- İki kopya!
          ^                   ^
          |                   |
     [ Printer ]         [ Scanner ]
          ^                   ^
          +---------+---------+
                    |
                [ Combo ]
```

🚩 **Kritik Nokta (Runtime Problem):**
Hoca burada çok önemli bir senaryo çiziyor: `Combo` nesnesinin "Printer" kısmından cihazı açarsanız (`m_flag = true`), "Scanner" kısmındaki cihaz hala kapalı (`m_flag = false`) kalır. Çünkü bunlar bellekte iki ayrı değişkendir!

---

#### 🛠️ 4. Sanal Kalıtım (Virtual Inheritance) [49:01 - 01:15:50]

**Neden İhtiyaç Duyuldu? (Rationale):** Elmas hiyerarşisinde ortak taban sınıfın (`Device`) tek bir kopyasının olmasını sağlamak için.

**⚙️ Teknik Detay ve Sentaks:**
```cpp
class Device { 
public: 
    void turnOn() { m_flag = true; }
    bool isOn() { return m_flag; }
    bool m_flag = false;
};

// "virtual" anahtar sözcüğü 'public'ten önce veya sonra gelebilir.
class Printer : virtual public Device {}; 
class Scanner : public virtual Device {}; 

class Combo : public Printer, public Scanner {};

int main() {
    Combo myCombo;
    myCombo.turnOn(); // Belirsizlik yok! Tek bir Device var.
    if(myCombo.isOn()) { 
        // Artık hem Printer hem Scanner tarafı için "Açık" kabul edilir.
    }
}
```

---

#### 🏗️ 5. Sanal Kalıtımda İnşa Kuralları ve Sorumluluk [01:16:00 - 01:31:22]

Sanal kalıtımda en kritik ve kafa karıştıran kural **başlatma sorumluluğudur.**

**🚩 Kritik Nokta / Mülakat Sorusu:**
**Soru:** `virtual base class` nesnesini kim initialize eder?
**Cevap:** **En türemiş sınıf (Most Derived Class).** Normalde bir sınıf sadece doğrudan taban sınıfını (direct base) initialize edebilirken, sanal kalıtımda en alttaki sınıf en üstteki sanal taban sınıfı initialize etmek **zorundadır.**

**⚙️ Teknik Detay ve Sentaks:**
```cpp
class Base {
public:
    Base(int x) {} // Default constructor yok!
};

class X : virtual public Base {
public:
    X() : Base(10) {} // Buradaki çağrı, X tek başına oluşturulursa çalışır.
};

class Y : virtual public Base {
public:
    Y() : Base(20) {}
};

class Der : public X, public Y {
public:
    // <-- HATA: Der() : X(), Y() {} yazarsak Base initialize edilemez.
    // Kritik Kural: Der sınıfı, hiyerarşinin en tepesindeki Base'i bizzat çağırmalıdır.
    Der() : Base(100), X(), Y() {} 
};
```

🔍 **Arka Plan (Under the Hood):**
Hoca'nın vurgusu: `Der` nesnesi inşa edilirken `X` ve `Y`'nin içindeki `Base` çağrıları **görmezden gelinir (suppressed).** Sadece `Der`'in yaptığı çağrı (`Base(100)`) dikkate alınır. Bu, nesnenin tutarlı bir tek kopyaya sahip olmasını sağlar.

---

#### 📊 Standart Karşılaştırması: `using` Bildirimleri

| Özellik | C++11/14 | C++17 ve Sonrası |
| :--- | :--- | :--- |
| `using` ile MI Belirsizliği Giderme | Her isim için ayrı satır gerekir. | Virgül ile ayrılmış liste kullanılabilir. |
| Örn: | `using B1::f; using B2::f;` | `using B1::f, B2::f;` |

---

#### 🚩 Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:
1.  **Dreadful Diamond Yanılgısı:** Sanal kalıtım (virtual inheritance) kullanılmazsa, ortak taban sınıfa yapılan her erişimin "Ambiguous" (Belirsiz) hata vereceği.
2.  **Sanal Taban Sınıf İnşası:** En türemiş sınıfın (Most Derived), aradaki sınıfları atlayarak en tepedeki sanal sınıfı initialize etmesi gerektiğini unutmak.
3.  **Side Cast Olasılığı:** Elmas hiyerarşisinde `dynamic_cast` ile kardeş sınıflar arasında geçiş yapılabileceği (Bunu RTTI kısmında detaylandıracak).

---

### 26. Ders: RTTI (Runtime Type Identification) - Bölüm 3

**Ders Akışı:** Çalışma Zamanı Tür Belirleme (RTTI), `dynamic_cast` operatörü, Pointer vs Referans Semantiği, Side-casting ve `std::bad_cast`.

---

#### 🧠 1. RTTI Neden Gerekli? (Rationale) [01:31:22 - 01:36:23]

**Neden İhtiyaç Duyuldu? (Rationale):** Polimorfizm sayesinde nesneleri ortak arayüz üzerinden (Base class) yönetiyoruz. Ancak bazen "iş işten geçtiğinde" veya "durumu kurtarmak" adına, elimizdeki taban sınıf pointer'ının **gerçekte** hangi türe ait olduğunu bilmemiz gerekir. Örneğin, sadece `Volvo` sınıfına has olan `openSunroof()` metodunu çağırmak için nesnenin gerçekten bir `Volvo` olduğundan emin olmalıyız.

*   **Upcasting:** Türemiş sınıftan taban sınıfa (Her Mercedes bir arabadır). Örtülüdür (implicit), her zaman güvenlidir.
*   **Downcasting:** Taban sınıftan türemiş sınıfa (Bu araba bir Mercedes mi?). Riskli ve açıkça (explicit) kontrol edilmesi gereken bir işlemdir.

---

#### 🛠️ 2. `dynamic_cast` Operatörü ve Polimorfizm Şartı [01:36:24 - 01:43:57]

`dynamic_cast`, çalışma zamanında tür dönüşümünün güvenli olup olmadığını denetleyen operatördür.

**⚙️ Teknik Detay ve Sentaks:**
```cpp
// Sentaks: dynamic_cast < hedef_tür > ( ifade )
Base* ptr = new Der;
Der* dptr = dynamic_cast<Der*>(ptr); 
```

🚩 **Mülakat Sorusu / Kritik Kural:**
`dynamic_cast` operatörünün operandı olan türün **Polymorphic (Polimorfik)** olması zorunludur.
*   **Soru:** Polimorfik sınıf nedir?
*   **Cevap:** İçinde en az bir tane `virtual` (sanal) fonksiyon bulunduran sınıftır. Eğer sınıf polimorfik değilse, `dynamic_cast` derleme zamanı hatası verir.

```cpp
class Base { }; // Sanal fonksiyon yok.
class Der : public Base { };

Base* bptr = new Der;
// auto p = dynamic_cast<Der*>(bptr); // <-- HATA: 'Base' is not a polymorphic type.
```

---

#### 🔍 3. `dynamic_cast` Pointer Semantiği ve Idiomatik Kullanım [01:43:58 - 02:02:59]

Pointer semantiğinde `dynamic_cast` hata durumunda **tanımsız davranış (UB)** oluşturmaz, bunun yerine **nullptr (null pointer)** döndürür.

**⚙️ Teknik Detay ve Sentaks (Modern C++ Yaklaşımı):**
```cpp
void car_game(Car* car_ptr) {
    // C++17 'if with initializer' (başlatıcılı if) kullanımı:
    if (auto vp = dynamic_cast<Volvo*>(car_ptr)) { 
        // Dönüşüm başarılı: car_ptr gerçekten bir Volvo veya ondan türemiş (XC90) biri.
        vp->openSunroof(); // <-- Hoca: "vp'nin kapsamını (scope) if ile daraltmak iyidir."
    } else {
        // Dönüşüm başarısız: vp == nullptr. Bu nesne bir Volvo değil (Audi, Fiat vb.).
    }
}
```

**🔍 Arka Plan (Under the Hood):**
Hoca, `dynamic_cast`'in bir **çalışma zamanı maliyeti** olduğunu vurguladı. Derleyici arka planda nesnenin VTABLE'ına ve RTTI bloklarına bakarak tür hiyerarşisini kontrol eder. Bu yüzden "bedava" (zero-cost) değildir.

---

#### 🚨 4. `dynamic_cast` Referans Semantiği ve `std::bad_cast` [02:03:00 - 02:07:30]

Referanslar `nullptr` olamayacağı için, referans semantiğiyle yapılan bir `dynamic_cast` başarısız olursa **exception (istisna)** fırlatır.

**⚙️ Teknik Detay ve Sentaks:**
```cpp
#include <typeinfo> // std::bad_cast için gerekli

try {
    Volvo& vr = dynamic_cast<Volvo&>(*car_ptr); // <-- HATA durumunda throw eder.
    vr.openSunroof();
} 
catch (const std::bad_cast& e) {
    // Dönüşüm başarısız olduğunda buraya girer.
    std::cout << "Hata: " << e.what() << "\n";
}
```

---

#### 🔄 5. Side-casting ve `dynamic_cast<void*>` [02:07:31 - 02:21:30]

Çoklu kalıtımda (Multiple Inheritance) bir taban sınıftan diğerine "yanlamasına" geçiş yapmaya **Side-casting** denir.

**⚙️ Teknik Detay ve Sentaks:**
```cpp
class Base { virtual ~Base() = default; };
class X : virtual public Base {};
class Y : virtual public Base {};
class Der : public X, public Y {};

// X'ten Y'ye geçiş (Side-casting):
X* x_ptr = new Der;
Y* y_ptr = dynamic_cast<Y*>(x_ptr); // Legal!
```

**🚩 Kritik Nokta:**
`dynamic_cast<void*>(ptr)` kullanımı, nesnenin hiyerarşideki en baş (fiziksel başlangıç) adresini bulmak için kullanılır. Çoklu kalıtımda taban sınıf pointer'ı nesnenin başını göstermiyor olabilir; bu yöntemle "gerçek" adrese ulaşılır.

---

#### 🆔 6. `typeid` Operatörü ve `std::type_info` [02:21:31 - 02:37:44]

`typeid` operatörü, nesnenin türüne dair bilgileri içeren `std::type_info` nesnesine bir `const` referans döner.

**⚙️ Teknik Detay ve Sentaks:**
```cpp
#include <typeinfo>

Car* p = new Volvo;
if (typeid(*p) == typeid(Volvo)) {
    // Tam eşleşme (Exact type match) arandığında kullanılır.
    // DİKKAT: typeid, polymorphic olmayan türlerde statik türe (Car), 
    // polymorphic türlerde ise dinamik türe (Volvo) bakar.
}

std::cout << typeid(*p).name() << "\n"; // Derleyiciye bağlı yazı (Örn: "class Volvo")
```

**🚩 Kritik Nokta / Mülakat Sorusu:**
**Soru:** `dynamic_cast` ile `typeid` arasındaki fark nedir?
**Cevap:** `dynamic_cast`, hiyerarşiyi (is-a ilişkisini) kontrol eder (Bir XC90, Volvo cast'inden geçer). `typeid` ise **tam tür eşleşmesine** bakar (Bir XC90, Volvo ile `typeid` üzerinden eşit çıkmaz).

**🔍 Arka Plan (Under the Hood):**
`std::type_info` sınıfının copy constructor'ı ve atama operatörü **delete** edilmiştir. Bu yüzden bu nesneleri kopyalayamazsınız; sadece referans üzerinden kullanabilirsiniz.

---

#### 🔗 Önceki Derslerle Bağlantı
*   **Integral Promotion:** `typeid` örneğinde karakter sabitlerinin `char` olduğunu, ancak aritmetik işlemlerde (`+c`) `int`'e yükseltildiğini tekrar hatırlattık.
*   **Virtual Destructor:** RTTI araçlarının (özellikle `dynamic_cast`) güvenli çalışması için taban sınıfta `virtual destructor` olmasının önemi bir kez daha vurgulandı.

---

#### 🚩 Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:
1.  **Polimorfizm İhmali:** Polimorfik olmayan (sanal fonksiyonu bulunmayan) bir sınıf hiyerarşisinde `dynamic_cast` kullanmaya çalışmak.
2.  **Referans Cast Hatası:** Referans cast'in hata durumunda `nullptr` dönmeyeceğini, programı sonlandırabilecek bir exception fırlatacağını unutmak.
3.  **Name() Yanılgısı:** `typeid(...).name()` çıktısının standart olduğunu sanmak (Bu çıktı derleyiciden derleyiciye değişir, taşınabilir kodda buna güvenilmez).

---



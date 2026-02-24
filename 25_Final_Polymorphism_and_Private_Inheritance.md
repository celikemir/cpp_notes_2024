Bu ders notları, Necati Ergin'in 23 Eylül 2024 tarihli 25. dersinin ilk 30 dakikasını kapsayan, yüksek teknik derinlikte ve titizlikle hazırlanmış bir "Yeniden İnşa" (Reconstruction) dokümanıdır.

# C++ DERS NOTLARI: Sanal Gönderim Mekanizması ve Maliyet Analizi

## 1. Bölüm: Sanal Fonksiyonların Nesne Boyutuna Etkisi [00:00 - 10:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir sınıfın polimorfik (çok biçimli) olup olmadığını çalışma zamanında anlamak ve doğru fonksiyonu çağırmak (Dynamic Dispatch) için derleyicinin nesneye ek bir bilgi gömmesi gerekir. Bu bölüm, sanal fonksiyonların sınıfın bellek kaplamasındaki (Memory Layout) somut etkisini incelemektedir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, sanal fonksiyonların nesne boyutunu nasıl değiştirdiğini şu kod örnekleri üzerinden kanıtladı:

```cpp
#include <iostream>

class Base {
    int x, y; // 4 + 4 = 8 byte
public:
    void f1(); 
    void f2();
};

class Derived : public Base {};

int main() {
    std::cout << "Base size: " << sizeof(Base) << "\n";       // Çıktı: 8
    std::cout << "Derived size: " << sizeof(Derived) << "\n"; // Çıktı: 8
}
```

Ancak sınıfa tek bir `virtual` anahtar sözcüğü eklendiğinde durum değişir:

```cpp
class Base {
    int x, y;
public:
    virtual void f1(); // <-- Hoca buraya dikkat çekti: Sınıf artık polimorfik!
    void f2();
};

class Derived : public Base {};

int main() {
    // 32-bit sistemlerde +4 byte, 64-bit sistemlerde +8 byte artış gözlenir.
    std::cout << "Polymorphic Base size: " << sizeof(Base) << "\n";       // Çıktı: 12 (veya 16)
    std::cout << "Polymorphic Derived size: " << sizeof(Derived) << "\n"; // Çıktı: 12 (veya 16)
}
```

**Kritik Gözlem:** Sanal fonksiyon sayısı 1 de olsa 100 de olsa, `sizeof` değerindeki artış sabittir (bir pointer boyutu kadar). Çünkü her sanal fonksiyon için ayrı bir pointer değil, tüm tabloyu gösteren tek bir pointer eklenir.

### 🔍 Arka Plan (Under the Hood/Memory Layout)
C++ Standartları implementasyona (gerçekleştirim) karışmaz ancak neredeyse tüm derleyiciler bu mekanizmayı **Vtable (Virtual Function Table)** ve **Vptr (Virtual Pointer)** ile çözer.

**ASCII Art - Bellek Yapısı:**
```text
[ Derived Object Memory ]
+-----------------------+
|   Vptr (Pointer)      | ----> [ Virtual Function Table (Vtable) ]
+-----------------------+       +-----------------------------+
|   Base::x (int)       |       | Index 0: Base::f1() adresi  |
+-----------------------+       | Index 1: Base::f2() adresi  |
|   Base::y (int)       |       +-----------------------------+
+-----------------------+
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Sanal fonksiyonu olan bir sınıfın boyutu neden artar? Her sanal fonksiyon için nesneye ayrı bir pointer mı eklenir?
**Cevap:** Hayır. Nesneye sadece **Vptr** (Virtual Pointer) denilen tek bir pointer eklenir. Bu pointer, o sınıfa ait olan ve tüm sanal fonksiyon adreslerini içeren **Vtable**'ı gösterir. Sınıfın polimorfik olması, nesnenin içinde gizli bir veri elemanı (Vptr) olduğu anlamına gelir.

---

## 2. Bölüm: Sanal Gönderim (Virtual Dispatch) Nasıl Çalışır? [10:00 - 20:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Derleme zamanında (Compile Time), bir pointer'ın hangi türden bir nesne gösterdiği tam olarak bilinemez. Derleyicinin, çalışma zamanında "gerçek" nesnenin türüne göre doğru fonksiyonu bulup çağırmasını sağlayacak bir "indeksleme" mekanizmasına ihtiyaç vardır.

### ⚙️ Teknik Detay ve Sentaks
Derleyici, bir sanal fonksiyon çağrısını (örneğin `p->run()`) şu mantıksal adımlara dönüştürür:

1. Pointer'ın gösterdiği nesneye git.
2. Nesnenin içindeki gizli `Vptr` elemanına eriş.
3. `Vptr` üzerinden o sınıfın `Vtable` (Sanal Fonksiyon Tablosu) adresini bul.
4. Fonksiyonun tabloda derleme zamanında belirlenmiş olan **indeksine** (örneğin 2. indeks) git.
5. O adresteki kodu çağır.

```cpp
Car* p = get_random_car();
p->run(); 
// Derleyicinin arka planda ürettiği (pseudo) kod:
// (*(p->vptr[1]))(p); // 1 nolu indeksteki fonksiyonu çağır.
```

### 🔍 Arka Plan (Under the Hood)
**De-virtualization (Sanallıktan Çıkarma):** Hoca bu noktada önemli bir optimizasyona değindi. Eğer derleyici, statik analiz ile o pointer'ın kesinlikle hangi nesneyi gösterdiğini anlarsa (örneğin nesne yerel bir değişkense), virtual dispatch mekanizmasını atlayıp doğrudan fonksiyonu çağırabilir. Buna "De-virtualization" denir.

### 📊 Standart Karşılaştırması
| Özellik | Virtual Dispatch | Normal Function Call |
| :--- | :--- | :--- |
| **Bağlama Zamanı** | Runtime (Geç Bağlama) | Compile Time (Erken Bağlama) |
| **Maliyet** | 2 seviyeli Dereferencing | Doğrudan Adres Çağrısı |
| **Inline Desteği** | Genellikle hayır (Zordur) | Evet (Kolaydır) |

### 🔗 Önceki Derslerle Bağlantı
Hoca, "İsim arama (Name Lookup) her zaman compile time'da yapılır" kuralını hatırlattı. Derleyici `p->run()` gördüğünde önce `Car` sınıfında `run` ismini arar, sanal olduğunu anlar ve indeksi belirler. Gerçek "binding" (bağlama) runtime'da olur.

---

## 3. Bölüm: Sanal Fonksiyonların Gerçek Maliyeti [20:00 - 30:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
"Polimorfizm bedavadır" demek teknik bir hatadır. Hoca, virtual dispatch kullanımının gizli maliyetlerini üç ana başlıkta topladı: İşlemsel maliyet, Bellek maliyeti ve Optimizasyon kaybı.

### ⚙️ Teknik Detay ve Maliyet Kalemleri
1.  **İşlemsel Maliyet (Processing Cost):** İki ilave `dereferencing` (adresten değere erişim). Pointer üzerinden nesneye, nesne üzerinden tabloya, tablo üzerinden fonksiyona gitmek işlemci çevrimleri harcar.
2.  **Bellek Maliyeti (Storage Cost):**
    *   Her nesne için bir `Vptr` boyutu kadar ek yer (8 byte modern sistemlerde).
    *   Her polimorfik sınıf için RAM'de bir adet `Vtable` veri yapısı.
3.  **İnşa Maliyeti (Initialization Cost):** Constructor (Yapıcı Metot) çalıştığında, derleyicinin yazdığı ek kodlarla `Vptr`'ın doğru `Vtable` adresine set edilmesi gerekir.

### 🚩 Kritik Nokta / Mülakat Sorusu (Kritik Uyarı!)
Hoca buraya çok sert bir vurgu yaptı: **Asıl maliyet dereferencing değildir!**
**Soru:** Sanal fonksiyonların en büyük maliyeti nedir?
**Cevap:** **Inline Expansion (Satır içi genişletme) engelidir.** Derleyici çağrılacak kodu compile time'da bilmediği için fonksiyonu inline yapamaz. Bu durum, sadece o fonksiyonun değil, o fonksiyonu içeren kod bloğunun da birçok optimizasyondan mahrum kalmasına neden olur.

### 🔗 Önceki Derslerle Bağlantı
"Virtual dispatch bizi dinamik ömürlü nesnelere (Heap allocation) mahkum eder." Hoca, polimorfizmin genellikle pointer/referans gerektirdiğini, bunun da bizi `new` operatörü ve Heap yönetimine ittiğini belirtti. Heap allocation maliyeti (malloc/free), virtual dispatch maliyetinden kat kat fazladır.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1. Sanal fonksiyonun boyut artışının fonksiyon sayısına bağlı olduğunu sanmak (Hata: Sadece `Vptr` boyutu eklenir).
2. Maliyeti sadece "hız" olarak düşünmek (Hata: Bellek hizalaması -padding- nedeniyle boyut çok daha fazla artabilir).
3. `Vtable`'ın her nesne için ayrı olduğunu sanmak (Hata: `Vtable` her **sınıf** için bir tanedir, nesneler sadece ona bir pointer tutar).

Bu bölümde Necati Hoca, polimorfizmin (çok biçimlilik) çalışma zamanındaki derinliklerine, RTTI mekanizmasına ve C++'taki çok kritik bir istisna olan "Covariant Return Types" konusuna giriş yapıyor.

---

## 4. Bölüm: RTTI ve Sanal Fonksiyon Tablosu İlişkisi [30:00 - 41:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bazen bir taban sınıf pointer'ının gösterdiği nesnenin "gerçekte" (runtime'da) hangi türden olduğunu bilmemiz gerekir. Örneğin; `Car*` pointer'ı ile gelen nesnenin bir `Mercedes` olup olmadığını anlayıp, ona göre özel bir işlem yapmak isteyebiliriz.

### ⚙️ Teknik Detay ve Sentaks
Bu ihtiyacı karşılayan araç setine **RTTI (Runtime Type Identification - Çalışma Zamanı Tür Belirleme)** denir. C++'ta bu amaçla kullanılan iki temel araç vardır:
1. `dynamic_cast<T>` operatörü.
2. `typeid` operatörü.

### 🔍 Arka Plan (Under the Hood/Memory Layout)
Hoca, mülakatlarda sorulabilecek çok kritik bir bağlantı kurdu: **RTTI bilgisi nerede tutulur?**
Çoğu derleyici, nesnenin dinamik tür bilgisini (Type Information) **Vtable** (Sanal Fonksiyon Tablosu) içinde tutar. Tipik olarak Vtable'ın **0 nolu indeksi**, o sınıfın kimliğini temsil eden `std::type_info` nesnesinin adresini gösterir.

**ASCII Art - Vtable & RTTI İlişkisi:**
```text
[ Object Vptr ] ---> [ Virtual Function Table ]
                     +-----------------------+
                     | Index 0: &type_info   | ----> [ Mercedes Tür Bilgisi ]
                     +-----------------------+
                     | Index 1: &run()       |
                     +-----------------------+
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Bir sınıfın polimorfik olması neden RTTI için zorunludur?
**Cevap:** Çünkü RTTI bilgisi (tür bilgisi), derleyiciler tarafından genellikle `Vtable` veri yapısına eklenir. Eğer bir sınıfın sanal fonksiyonu yoksa `Vtable`'ı da yoktur; dolayısıyla çalışma zamanında türünü belirleyecek bir mekanizma (Vptr) nesneye eklenmemiştir.

---

## 5. Bölüm: Covariant Return Types (Eş-Değişken Geri Dönüş Türleri) [41:00 - 51:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C++'ta normalde sanal bir fonksiyonu override (ezme) etmek için imzanın ve geri dönüş türünün **birebir aynı** olması gerekir. Ancak bu kuralın çok spesifik bir istisnası vardır. Eğer taban sınıf pointer/referans döndürüyorsa, türemiş sınıf bu fonksiyonu "daha spesifik" bir tür döndürecek şekilde override edebilir.

### ⚙️ Teknik Detay ve Sentaks
Bu özellik sadece **Pointer** veya **Referans** döndüren fonksiyonlarda geçerlidir.

```cpp
class Base {
public:
    virtual Base* clone(); // <-- Pointer döndürüyor
    virtual Base& get_ref(); // <-- Referans döndürüyor
};

class Derived : public Base {
public:
    // NORMAL KURAL: Base* döndürmeliydi.
    // İSTİSNA (Covariance): Derived* döndürebilir!
    Derived* clone() override; // <-- LEGAL: Covariant Return Type
    Derived& get_ref() override; // <-- LEGAL
};
```

**Derleyici Şu Sebeple Kızıyor (Hata Örneği):**
Eğer geri dönüş türü bir "sınıf nesnesinin kendisi" ise (by value), covariance çalışmaz:
```cpp
class Base {
public:
    virtual Base boom(); 
};

class Derived : public Base {
public:
    Derived boom() override; // <-- HATA: "return type is not covariant"
    // Hoca: "Object Slicing riski nedeniyle sadece pointer ve referanslarda geçerlidir."
};
```

### 🔍 Arka Plan (Under the Hood)
Hoca, `int` gibi temel türlerde bu özelliğin neden çalışmadığını açıkladı:
`int` ve `float` arasında bir kalıtım (inheritance) ilişkisi yoktur. Covariance olması için geri dönen türler arasında "Is-a relationship" (Kalıtım ilişkisi) olması şarttır.

---

## 6. Bölüm: Clone Idiom (Sanal Yapıcı Metot İhtiyacı) [51:00 - 01:01:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C++'ta **Virtual Constructor (Sanal Yapıcı)** diye bir şey yoktur. Ancak elimizde bir `Car*` varken, onun gerçek türü neyse (Renault, Volvo vb.) ondan bir kopya daha oluşturmak isteriz. İşte bu "kendini kopyalama" yeteneğine **Clone Idiom** denir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, Covariance özelliğinin en şık kullanımının bu idiom olduğunu gösterdi:

```cpp
class Car {
public:
    virtual Car* clone() = 0; // Pure Virtual (Saf Sanal)
};

class Mercedes : public Car {
public:
    // Mercedes* döndürerek kullanıcının hayatını kolaylaştırıyor
    Mercedes* clone() override { 
        return new Mercedes(*this); // Kendisinin kopyasını dinamik olarak oluşturur
    }
};

void process(Car* p) {
    Car* p_copy = p->clone(); // Hangi araba gelirse gelsin kopyalanır!
    // ...
    delete p_copy;
}
```

### 🔍 Arka Plan (Multi-level Inheritance)
Hoca, hiyerarşinin derinleştiği durumlarda (Volvo -> Volvo_XC90) sanal gönderimin nasıl davrandığını inceledi:
- Eğer `Volvo_XC90` sınıfı bir fonksiyonu (örn: `stop()`) override etmezse, Vtable'da o indekste bir üst sınıfın (`Volvo`) fonksiyon adresi tutulur. 
- Bu sayede "en güncel" override edilmiş versiyon her zaman Vtable üzerinden bulunur.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Covariance kullanmanın kullanıcıya sağladığı avantaj nedir?
**Cevap:** Gereksiz **Downcasting** (Tabandan türemişe tür dönüşümü) işlemlerini engeller. Eğer `Mercedes::clone()` fonksiyonu `Car*` döndürseydi, kullanıcı Mercedes'e özel metotlara erişmek için geri dönüş değerini tekrar `static_cast<Mercedes*>` yapmak zorunda kalacaktı. Covariance sayesinde buna gerek kalmaz.

---

**Bu bölümde Hoca şu 3 kritik noktaya dikkat çekti:**
1. RTTI mekanizmasının aslında Vtable mimarisinin üzerine inşa edilmiş bir "yan ürün" olduğunu ("Durumdan vazife çıkartmak").
2. Covariance'ın sadece "kalıtım ilişkisi içindeki türlerin pointer ve referansları" için geçerli bir istisna olduğu.
3. Hiyerarşi içinde bir metot override edilmezse, Vtable'ın otomatik olarak taban sınıfın adresini miras aldığı.

Bu bölümde Necati Hoca, C++ kalıtım hiyerarşisindeki en tehlikeli konulardan biri olan "Sanal Yok Ediciler" (Virtual Destructors) konusunu derinlemesine işliyor ve ardından "Global Fonksiyonların Sanallaştırılması" gibi ileri düzey tekniklere geçiş yapıyor.

---

## 7. Bölüm: Sanal Yok Ediciler (Virtual Destructors) [01:01:00 - 01:14:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Dinamik ömürlü (Heap'te oluşturulan) bir türemiş sınıf nesnesini, bir taban sınıf pointer'ı üzerinden yok etmeye (delete) çalıştığımızda ciddi bir sorunla karşılaşırız. Eğer taban sınıfın yok edicisi (destructor) sanal değilse, derleyici sadece taban sınıfın yok edicisini çağırır ve nesnenin "türemiş" kısmı yok edilmez.

### ⚙️ Teknik Detay ve Sentaks
Hoca, bu durumu şu kod ile kanıtladı:

```cpp
#include <iostream>

class Base {
public:
    ~Base() { std::cout << "Base destructor\n"; } // <-- HATA: Virtual değil!
};

class Derived : public Base {
public:
    ~Derived() { std::cout << "Derived destructor\n"; }
};

int main() {
    Base* p = new Derived; 
    delete p; // <-- Kritik Nokta: Hangi destructor çağrılacak?
}
```
**Çıktı:** `Base destructor` (Sadece Base yok edildi, Derived kısmı havada kaldı!)

**Çözüm:** Taban sınıfın yok edicisi `virtual` yapılmalıdır.

```cpp
class Base {
public:
    virtual ~Base() { std::cout << "Base destructor\n"; } // <-- DOĞRUSU
};
```
**Yeni Çıktı:** 
1. `Derived destructor`
2. `Base destructor` (Olması gerektiği gibi, tersten imha sırası!)

### 🔍 Arka Plan (Under the Hood)
Hoca burada çok kritik bir teknik terim kullandı: **UB (Undefined Behavior - Tanımsız Davranış)**.
Eğer polimorfik bir nesneyi sanal olmayan bir yok ediciye sahip bir taban sınıf pointer'ı üzerinden `delete` ederseniz, standartlara göre bu sadece bir bellek sızıntısı (memory leak) değil, doğrudan **UB**'dir. Programın o anda ne yapacağı belirsizdir.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Taban sınıfın destructor'ı her zaman sanal mı olmalıdır?
**Cevap:** Hoca, ünlü C++ gurusu **Herb Sutter**'ın ilkesini aktardı: "Polimorfik taban sınıfların yok edicisi ya **public virtual** olmalı ya da **protected non-virtual** olmalı."

---

## 8. Bölüm: Protected Non-Virtual Destructors [01:14:00 - 01:21:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bazen bir sınıfın taban sınıf olarak kullanılmasını ama asla bir pointer üzerinden silinmesini (polimorfik delete) istemeyiz. Bu durumda `virtual` maliyetinden kaçınmak için yok ediciyi `protected` yaparız.

### ⚙️ Teknik Detay ve Sentaks
```cpp
class Base {
protected:
    ~Base() = default; // <-- Sadece türemiş sınıflar erişebilir, client silemez.
public:
    virtual void foo();
};

int main() {
    Base* p = new Derived;
    delete p; // <-- DERLEYİCİ HATASI: "cannot access protected member"
}
```
**Derleyici Şu Sebeple Kızıyor:** `delete p` ifadesi aslında yok ediciyi çağırmaya çalışır. Ancak yok edici `protected` olduğu için `main` fonksiyonu (client) buna erişemez. Bu, yanlışlıkla yapılacak "tehlikeli silme" işlemlerini derleme zamanında engeller.

---

## 9. Bölüm: NVI Idiom (Non-Virtual Interface) [01:21:00 - 01:28:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
"Sanal fonksiyonları public yapmayın." Hoca, bu şaşırtıcı ilkeyi **NVI Idiom** (Sanal Olmayan Arayüz) başlığı altında açıkladı. Arayüz (Interface) ile Gerçekleştirim (Implementation) arasındaki bağı koparmak için kullanılır.

### ⚙️ Teknik Detay ve Sentaks
```cpp
class Base {
public:
    void run() { // <-- Public, Non-Virtual
        // Hoca: "Burada ön hazırlık kodları olabilir (logging vb.)"
        do_run(); // <-- Sanal olan private fonksiyonu çağırıyor
        // "Burada bitiş işlemleri olabilir."
    }
private:
    virtual void do_run() = 0; // <-- Private, Pure Virtual
};

class Derived : public Base {
private:
    void do_run() override { /* Gerçek iş burada */ }
};
```
**Mantık:** Kullanıcı her zaman `run()` fonksiyonunu çağırır. `run()` fonksiyonu ise arka planda sanal olan `do_run()` metodu üzerinden polimorfizmi sağlar.

---

## 10. Bölüm: Global Fonksiyonların Sanallaştırılması [01:28:00 - 01:38:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C++'ta global fonksiyonlar (örneğin `operator<<`) sanal olamaz. Ancak biz bir `std::cout << *car_ptr;` dediğimizde, arabanın türüne göre (Mercedes, Renault vb.) farklı çıktı almak isteriz.

### ⚙️ Teknik Detay ve Sentaks
Hoca bu sorunu **"Virtualize by Proxy"** yöntemiyle çözdü:

```cpp
class Car {
public:
    // Global operator<< bunu çağıracak!
    virtual std::ostream& print(std::ostream& os) const = 0; 
};

// Global Fonksiyon (Sanal olamaz ama sanal fonksiyonu çağırabilir!)
std::ostream& operator<<(std::ostream& os, const Car& car) {
    return car.print(os); // <-- Virtual Dispatch burada devreye girer!
}

class Mercedes : public Car {
public:
    std::ostream& print(std::ostream& os) const override {
        return os << "I am a Mercedes";
    }
};
```

**🔍 Arka Plan (ASCII Art):**
```text
[ std::cout << *p ] 
      |
      V
[ Global operator<< ] 
      |
      +-----> [ p->print() ]  <-- (Virtual Dispatch!)
                    |
                    +-----> [ Mercedes::print() ]
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `operator<<` gibi bir global fonksiyonu nasıl polimorfik hale getirirsiniz?
**Cevap:** Sınıf içine `print` gibi sanal bir yardımcı fonksiyon (helper function) ekleyerek. Global operatör fonksiyonu, bu sanal yardımcı fonksiyonu çağırır. Böylece dışarıdan bakıldığında operatör polimorfikmiş gibi davranır.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1. Polimorfik sınıflarda `virtual destructor` kullanmamak (Hata: **UB** - Undefined Behavior).
2. NVI idiomunda public fonksiyonu sanal yapmak (Yanlış: Public olan sanal olmamalı, sanal olan private olmalı).
3. Pure virtual fonksiyonu olan sınıftan nesne oluşturmaya çalışmak (Hata: **Abstract Class** hatası).

Bu bölümde Necati Hoca, modern C++ ile gelen `final` anahtar sözcüğünü, C++’ın en çok karıştırılan konularından olan "Private ve Protected Kalıtımı"nı ve kalıtımın bellek üzerindeki mikro optimizasyonlarını (EBO) işliyor.

---

## 11. Bölüm: `final` Contextual Keyword [01:38:00 - 01:48:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Eskiden bir sınıfın kalıtıma kapatıldığını veya bir fonksiyonun artık override edilemeyeceğini belirtmek için sadece yorum satırları kullanılıyordu. Modern C++ (C++11), bunu derleme zamanında (compile-time) kontrol eden `final` mekanizmasını getirdi.

### ⚙️ Teknik Detay ve Sentaks
`final` iki yerde kullanılır:
1. **Sınıf Seviyesinde:** Sınıfın kendisinden kalıtım yapılmasını engeller.
2. **Fonksiyon Seviyesinde:** Türemiş sınıfların o fonksiyonu tekrar override etmesini engeller.

```cpp
class Base {
public:
    virtual void func();
};

// 1. Sınıf Seviyesinde Final
class FinalClass final : public Base { // <-- Artık bu sınıftan kimse türeyemez
    void func() override;
};

// class Derived : public FinalClass {}; // <-- DERLEYİCİ HATASI: "cannot inherit from final class"

// 2. Fonksiyon Seviyesinde Final
class Derived : public Base {
public:
    void func() final override; // <-- Kalıtım açık, ama bu fonksiyon override'a kapalı!
};

class SubDerived : public Derived {
    // void func() override; // <-- DERLEYİCİ HATASI: "cannot be overridden"
};
```

### 🔍 Arka Plan (Contextual Keyword)
`final` ve `override` aslında "gerçek" keyword değildir. Bunlara **Contextual Keyword** (Bağlamsal Anahtar Sözcük) denir. Sadece sınıf veya fonksiyon bildiriminin sonuna geldiklerinde özel bir anlam ifade ederler. Değişken ismi olarak kullanılabilirler (Gerçi önerilmez!).

---

## 12. Bölüm: Private ve Protected Kalıtımı [01:48:00 - 02:01:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C++'ta kalıtım sadece "Is-a" (biridir) ilişkisi kurmak için kullanılmaz. Bazen bir sınıfın özelliklerini kullanmak (Implementation Inheritance) isteriz ama o sınıfın türünü dış dünyaya (Client) açmak istemeyiz. Buna "Implemented-in-terms-of" denir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, erişim belirteçlerinin kalıtımdaki etkisini şu tabloyla özetledi:

```cpp
class Base {
public:    void pub();
protected: void pro();
private:   void pri();
};

class Derived : private Base { // <-- Private Kalıtım (Default'tur!)
    // Base::pub() artık Derived içinde PRIVATE
    // Base::pro() artık Derived içinde PRIVATE
    // Base::pri() zaten erişilemez
};
```

**🚩 Kritik Nokta (Upcasting Engel):**
Public kalıtımda `Derived*` türünden `Base*` türüne otomatik dönüşüm (Upcasting) varken, Private kalıtımda bu **Client'a (Dışarıya) kapalıdır!**

```cpp
Derived my_der;
Base* p = &my_der; // <-- HATA: Private kalıtımda "Is-a" ilişkisi dış dünyaya kapalıdır!
```

---

## 13. Bölüm: Private Kalıtımın Gizli Gücü: Üye Fonksiyonlarda Upcasting [02:01:00 - 02:11:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Dış dünya (Client), `Derived` nesnesini `Base` gibi kullanamaz. Ancak `Derived` sınıfının kendi içindeki üye fonksiyonları ve `friend` fonksiyonları bu dönüşümü hala yapabilir.

### ⚙️ Teknik Detay ve Sentaks
```cpp
class Base { public: virtual void foo(); };
class Derived : private Base {
public:
    void internal_work() {
        Base* p = this; // <-- LEGAL: Kendi içinde Upcasting hala geçerli!
        p->foo();       // <-- Polymorphism çalışır.
    }
    friend void global_friend(Derived& d);
};

void global_friend(Derived& d) {
    Base& rb = d; // <-- LEGAL: Friend fonksiyon kalıtım ilişkisini "görebilir".
}
```

---

## 14. Bölüm: Kompozisyon (Containment) vs. Private Kalıtım [02:11:00 - 02:22:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Hoca, "Private kalıtım mı yoksa bir nesneyi üye olarak mı tutmalıyım (Composition)?" sorusuna 3 spesifik yanıt verdi:

1.  **Override Gereksinimi:** Taban sınıfın sanal fonksiyonlarını override etmeniz gerekiyorsa Private kalıtım şarttır (Composition ile override yapamazsınız).
2.  **Protected Üyelere Erişim:** Taban sınıfın `protected` üyelerine erişmeniz gerekiyorsa Private kalıtım gerekir.
3.  **EBO (Empty Base Optimization):** Boş sınıflarda bellek tasarrufu sağlar.

### 🔍 Arka Plan (EBO - Empty Base Optimization)
C++'ta her nesnenin en az 1 byte yer kaplaması gerekir (adreslenebilirlik için). Ancak kalıtımda durum farklıdır:

```cpp
class Empty {}; // sizeof: 1

class Composition {
    Empty e; // sizeof: 1 (veya alignment yüzünden 4-8)
    int x;
}; // sizeof: 8 (Padding dahil)

class Inheritance : private Empty {
    int x;
}; // sizeof: 4 (Hoca: "EBO devreye girdi, Empty için yer ayrılmadı!")
```

---

## 15. Bölüm: Polimorfik Listeler (Heterogeneous Lists) [02:31:00 - 02:39:16]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Çalışma zamanında türü belirlenen onlarca nesneyi (Audi, Fiat, Opel vb.) tek bir kapta (Container) tutup, tek bir döngüyle yönetmek. C++'ın gerçek gücü buradadır.

### ⚙️ Teknik Detay ve Sentaks
```cpp
#include <vector>

std::vector<Car*> showroom; // <-- Polimorfik Liste

for (int i = 0; i < 50; ++i) {
    showroom.push_back(create_random_car()); // Farklı türden arabalar doluyor
}

for (auto cp : showroom) {
    std::cout << *cp << "\n"; // <-- Global operator<< sanallaştırılmıştı!
    cp->start();              // Her araba kendi start()'ını çalıştırır
    delete cp;                // <-- Virtual Destructor burada hayat kurtarır!
}
```

### 🖼️ Görselleştirme (ASCII Art) - Polimorfik Vektör
```text
[ vector<Car*> ]
| [0] | ----> [ Mercedes Object (Vptr -> Mercedes Vtable) ]
| [1] | ----> [ Renault Object  (Vptr -> Renault Vtable)  ]
| [2] | ----> [ Volvo Object    (Vptr -> Volvo Vtable)    ]
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Polimorfik bir vektörü döngüyle silerken nelere dikkat edilmelidir?
**Cevap:** Mutlaka taban sınıfta `virtual destructor` olmalıdır. Aksi takdirde `delete cp;` satırı sadece `Car` kısmını siler, nesnenin asıl (derived) kısımları bellekte kalır ve **Undefined Behavior** oluşur.

---

**Bu bölümde Hoca şu kritik noktalara dikkat çekti:**
1. `final` anahtar sözcüğünün optimizasyon için derleyiciye ipucu verdiğini.
2. Private kalıtımın bir "is-a" değil, "is-implemented-in-terms-of" ilişkisi olduğunu.
3. Boş sınıfların kalıtım yoluyla kullanıldığında bellek kaplamadığını (**EBO**).
4. Polimorfizmin en güçlü yanının heterojen (farklı türden) nesne dizileri yönetmek olduğunu.

**Ders Sonu:** Hoca, bir sonraki derste "Çoklu Kalıtım" (Multiple Inheritance), "RTTI" ve "Exception Handling" konularına gireceğini söyleyerek dersi bitirdi.



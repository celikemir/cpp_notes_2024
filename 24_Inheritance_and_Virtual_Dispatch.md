Bu ders notları, Necati Ergin'in C++ eğitimindeki 24. ders gününe (18 Eylül 2024) aittir. Bir C++ mühendisi titizliğiyle, dersteki her teknik ayrıntı yeniden inşa edilmiştir.

---

# C++ Kalıtım Derinlemesine İnceleme - Bölüm I

## 1. Sınıf İçi `using` Bildirimi ve İsim Maskeleme (Name Masking) [00:00 - 07:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C++'ta isim arama (name lookup) kuralları gereği, türemiş sınıf (derived class) içinde taban sınıftaki (base class) bir fonksiyonla aynı isimli bir fonksiyon tanımlandığında, taban sınıftaki tüm overload'lar maskelenir (hiding). Bu durum, taban sınıftaki fonksiyonların türemiş sınıf nesnesi üzerinden doğrudan çağrılmasını engeller. `using` bildirimi, bu isimleri türemiş sınıfın kapsamına (scope) enjekte ederek maskelenmeyi önlemek için kullanılır.

### ⚙️ Teknik Detay ve Sentaks
```cpp
class Base {
public:
    void foo(int x);
    void foo(double d);
};

class Derived : public Base {
public:
    using Base::foo; // <-- Hoca buraya dikkat çekti: Taban sınıftaki foo'ları bu kapsama enjekte eder.
    void foo(int x, int y); // Base::foo'ları maskelemez, onlarla overload olur.
};

int main() {
    Derived dr;
    dr.foo(5); // using olmasaydı SENTAKS HATASI: Derived içinde tek parametreli foo yok.
}
```

### 🔍 Arka Plan (Under the Hood)
*   **İsim Arama Sırası:** Önce `Derived` sınıfının skopunda aranır. İsim bulunduğu anda arama biter. Eğer `using` bildirilmemişse ve `Derived` içinde isim bulunmuşsa, parametre uyumsuzluğu olsa dahi taban sınıfa bakılmaz.
*   **Erişim Kontrolü:** `using` bildiriminin yapıldığı yer (public/private) önemlidir. Eğer `private` bölümde bildirilirse, taban sınıfta `public` olan fonksiyon türemiş sınıfın `private` arayüzüne enjekte edilmiş olur.

---

## 2. Inherited Constructors (Kalıtımla Alınan Constructor'lar) [07:00 - 15:15]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Modern C++ öncesinde, taban sınıfın constructor'larını türemiş sınıf üzerinden kullanabilmek için her bir imza (signature) için türemiş sınıfta ayrı ayrı constructor yazıp argümanları taban sınıfa yönlendirmek (forwarding) gerekiyordu. Bu ciddi bir "boilerplate" (basmakalıp) kod yükü oluşturuyordu.

### ⚙️ Teknik Detay ve Sentaks
```cpp
class Base {
public:
    Base(int x, int y) { /*...*/ }
    Base(double d) { /*...*/ }
};

class Derived : public Base {
public:
    using Base::Base; // <-- KRİTİK KURAL: C++11 ile gelen "Inherited Constructor".
};

int main() {
    Derived d1(10, 20); // Base(int, int) çağrılır.
    Derived d2(5.5);    // Base(double) çağrılır.
    // Derived d3;      // <-- HATA BURADA: using Base::Base kullanıldığında Derived'ın 
                        // default constructor'ı derleyici tarafından DELETE edilir.
}
```

### 🔍 Arka Plan (Under the Hood)
Derleyici, `using Base::Base` ifadesini gördüğünde adeta türemiş sınıf için aynı parametre yapısında constructor'lar yazar ve bunları Member Initializer List üzerinden taban sınıf constructor'larına bağlar.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `using Base::Base` kullanıldığında türemiş sınıfın `default constructor`'ı ne olur?
**Cevap:** Eğer taban sınıfın `default constructor`'ı yoksa ve türemiş sınıfta açıkça yazılmamışsa, derleyici türemiş sınıfın `default constructor`'ını **delete** eder.

---

## 3. Fonksiyon Kategorizasyonu ve Polimorfizm Giriş [15:15 - 23:40]

Necati Hoca, taban sınıf fonksiyonlarını tasarımsal açıdan 3 ana kategoriye ayırdı:

1.  **Interface + Mandatory Implementation:** Türemiş sınıfa hem arayüz hem kod verir. Türemiş sınıf bu kodu değiştiremez (Non-virtual).
2.  **Interface + Default Implementation (`virtual`):** "İstersen benim kodumu kullan, istersen kendi kodunu yaz (override)" mesajını verir.
3.  **Interface Only (`pure virtual`):** "Kod vermiyorum, sen yazmak zorundasın" der.

### 🖼️ Görselleştirme (ASCII Art)
```text
Sınıf Türü       Fonksiyon Yapısı          Durum
----------       ----------------          -----
Concrete Class   Virtual Function          Opsiyonel Override
Abstract Class   Pure Virtual (=0)         Zorunlu Override
Non-Polymorphic  Non-Virtual               Override Edilemez (Sadece Hiding)
```

---

## 4. Sanal Fonksiyonlar (Virtual Functions) ve Overriding [23:40 - 39:00]

### ⚙️ Teknik Detay ve Sentaks
```cpp
class Airplane {
public:
    virtual void fly() { /* Default uçma kodu */ } // Kategori 2
    virtual void land() = 0; // Kategori 3: PURE VIRTUAL (Saf Sanal)
};
```

### 🔍 Arka Plan (Under the Hood)
*   **Pure Virtual (`= 0`):** Bu sentaks bir atama değil, derleyiciye "bu fonksiyonun implementasyonu yok" deme biçimidir (Bjarne Stroustrup'un anahtar sözcük sayısını artırmama tercihi).
*   **Abstract Class (Soyut Sınıf):** En az bir tane `pure virtual` fonksiyonu olan sınıftır. Bu sınıflardan nesne oluşturulamaz (**Instantiate edilemez**).

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Bir türemiş sınıfın "Concrete" (Somut) olabilmesi için ne yapması gerekir?
**Cevap:** Taban sınıfındaki **tüm** pure virtual fonksiyonları `override` etmesi gerekir. Bir tanesini bile eksik bırakırsa o sınıf da "Abstract" kalır.

---

## 5. Modern C++: `override` ve `final` Specifiers [45:00 - 58:30]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Eski C++'ta (C++98/03), `virtual` fonksiyonu override ederken imzada yapılan küçük bir hata (örneğin `const` eksikliği), fonksiyonun override edilmesine değil, yeni bir fonksiyon tanımlanmasına (hiding) yol açıyordu. Derleyici hata vermediği için bu durum "logical bug" (mantıksal hata) yaratıyordu.

### ⚙️ Teknik Detay ve Sentaks
```cpp
class Base {
public:
    virtual void foo(int x) const;
};

class Derived : public Base {
public:
    // void foo(int x) override; // <-- HATA BURADA: const eksik, derleyici kızar!
    void foo(int x) const override; // DOĞRU
};
```

### 🔍 Arka Plan (Contextual Keywords)
*   `override` ve `final` gerçek anahtar sözcük (keyword) değildir. **Contextual Keyword** (Bağlamsal Anahtar Sözcük) olarak adlandırılırlar.
*   Yani `int override = 5;` şeklinde bir değişken tanımlamak yasaldır; ancak fonksiyon bildiriminin sonunda özel anlam kazanırlar. Bu, eski kodların kırılmaması için yapılmış dahiyane bir tasarımdır.

---

### ⏱ 10 Dakikalık Blok Özeti (Bölüm I Sonu)
Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:
1.  **Signature Mismatch:** Sanal fonksiyonu override ederken `const` veya parametre farkı yaparsanız `override` keyword'ü yoksa derleyici uyarmaz, sessizce hata oluşur.
2.  **Using vs Construction:** `using` ile constructor alındığında türemiş sınıfın default ctor'unun silinebileceğini unutmayın.
3.  **Abstract Class Instantiation:** Pure virtual fonksiyonu olan sınıftan nesne üretmeye çalışmak en sık yapılan başlangıç hatasıdır.

Dersin ikinci bölümünde Necati Hoca, sanal fonksiyonların (virtual functions) çalışma zamanı davranışlarını, mülakatların vazgeçilmez konusu olan "Binding" kavramlarını ve "Virtual Constructor" idiomunu derinlemesine inceledi. Bir C++ mühendisi titizliğiyle notlarımıza devam ediyoruz.

---

# C++ Kalıtım Derinlemesine İnceleme - Bölüm II

## 6. Sanal Gönderim Mekanizması (Virtual Dispatch) [01:00:00 - 01:13:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Farklı türemiş sınıfların (Mercedes, Volvo, Renault), ortak bir taban sınıf (`Car`) arayüzü üzerinden, kendi türlerine özgü davranışları (override edilmiş fonksiyonlar) sergilemesi istenir. Bu, istemci kodun nesnenin gerçek türünü bilmesine gerek kalmadan iş yapabilmesini sağlar.

### ⚙️ Teknik Detay ve Sentaks
```cpp
class Car {
public:
    virtual void start() { std::cout << "Car started\n"; }
};

class Mercedes : public Car {
public:
    void start() override { std::cout << "Mercedes started\n"; }
};

void carGame(Car& c) {
    c.start(); // <-- Virtual Dispatch: Çalışma zamanında c'nin gerçek türü neyse o start() çağrılır.
}
```

### 🔍 Arka Plan (Under the Hood)
Hoca bu durumu **"Sihirli anahtar sözcük"** (`virtual`) olarak nitelendirdi. Derleyici, sanal fonksiyon çağrısını gördüğünde, hangi fonksiyonun çağrılacağını derleme zamanında değil, çalışma zamanında nesnenin içine gizlenmiş bir mekanizma (vptr) üzerinden belirleyecek kod üretir.

---

## 7. Statik Bağlama vs. Dinamik Bağlama (Static vs. Dynamic Binding) [01:13:00 - 01:28:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Programın akışına göre (örneğin rastgele sayı üretimi veya kullanıcı girişi) hangi nesnenin kullanılacağı derleme zamanında bilinemez. Bu belirsizliği çözmek için "Dinamik Bağlama" gereklidir.

### ⚙️ Teknik Detay ve Sentaks
```cpp
Car* p = createRandomCar(); // Runtime'da ne döneceği belli değil.
p->start(); // Dynamic Binding (Late Binding): Hangi start() olduğu çalışma zamanında belli olur.
p->Car::start(); // Static Binding (Early Binding): Hoca vurguladı: "Niteleme yapılırsa sanallık düşer!"
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** "Early Binding" ve "Late Binding" arasındaki fark nedir?
**Cevap:** Fonksiyon çağrısının adresinin derleme zamanında (static) veya çalışma zamanında (dynamic) belirlenmesidir. C++'ta sanal olmayan fonksiyonlar statik, sanal olanlar ise (işaretçi/referans üzerinden çağrıldığında) dinamik bağlanır.

---

## 8. Sanallığın Devreye Girdiği ve Girmediği Senaryolar [01:28:00 - 01:43:00]

Hoca, Virtual Dispatch'in ne zaman çalışıp ne zaman çalışmayacağını kesin kurallara bağladı:

### ✅ Virtual Dispatch Devreye GİRER:
1.  **Base Pointer üzerinden çağrı:** `Base* ptr = &der; ptr->vfunc();`
2.  **Base Reference üzerinden çağrı:** `Base& ref = der; ref.vfunc();`
3.  **Sanal olmayan üye fonksiyon içinden çağrı:** `void Base::test() { vfunc(); }` (Hoca buna **NVI - Non Virtual Interface** girişi yaptı).

### ❌ Virtual Dispatch Devreye GİRMEZ:
1.  **Nesnenin kendisi üzerinden çağrı:** `Base b; b.vfunc();` (Statik tür `Base`, dinamik tür de `Base`'dir).
2.  **Object Slicing (Nesne Dilimlenmesi):**
    ```cpp
    Car myCar = Mercedes(); // Mercedes, Car nesnesine atandı (Dilimlendi).
    myCar.start(); // <-- HATA: Hoca dedi ki: "Bu artık sadece bir Car, Mercedes özelliği yok edildi!"
    ```
3.  **Niteleme (Qualification) yapılması:** `p->Base::start();` (Sanal gönderim mekanizması bypass edilir).

---

## 9. Constructor ve Destructor İçinde Sanallık (Kritik!) [02:08:00 - 02:18:00]

### 🚩 Kritik Nokta / Mülakat Sorusu (En Çok Sorulan)
**Soru:** Constructor içinde sanal bir fonksiyonu çağırırsak ne olur?
**Cevap:** **Virtual Dispatch devreye girmez!** Hangi sınıfın constructor'ı çalışıyorsa, o sınıftaki (veya yukarıdaki) implementasyon çağrılır.

### 🔍 Arka Plan (Rationale)
Hoca bu durumu "Felaket (Katastrof)" olarak tanımladı. Nedeni ise hiyerarşidir:
1.  Türemiş sınıf nesnesi oluşurken önce **Base Class** nesnesi oluşur.
2.  Base Class constructor'ı çalışırken **Derived Class** bölümleri (member variables) henüz hayata gelmemiştir.
3.  Eğer türemiş sınıfın override'ı çalışsaydı, henüz oluşmamış member'lara erişmeye çalışacaktı (UB - Undefined Behavior).

---

## 10. Sanal Constructor İdiomu (Virtual Constructor / Clone Idiom) [02:18:00 - 02:35:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C++'ta constructor'lar sanal olamaz (`virtual Base() // SENTAKS HATASI`). Ancak bazen elimizdeki bir taban sınıf işaretçisinin (örneğin `Car*`) gösterdiği nesnenin **"tıpkısının aynısını (kopyasını)"** oluşturmamız gerekir. Nesnenin Mercedes mi Volvo mu olduğunu bilmeden kopyalamak için bu idiomu kullanırız.

### ⚙️ Teknik Detay ve Sentaks
```cpp
class Car {
public:
    virtual Car* clone() = 0; // Virtual Constructor İdiomu
};

class Mercedes : public Car {
public:
    Mercedes* clone() override { 
        return new Mercedes(*this); // <-- Hoca: "Copy Constructor'ı kullanarak kopyasını oluşturur."
    }
};
```

---

## 11. Polimorfik Sınıfların Maliyeti ve `sizeof` [02:35:00 - 02:41:00]

### 🖼️ Görselleştirme (ASCII Art - Bellek Yapısı)
Hoca, sınıfın içine sadece bir tane bile `virtual` eklendiğinde nesne boyutunun (genellikle 4 veya 8 byte) arttığını gösterdi.

```text
Non-Polymorphic Object         Polymorphic Object (Base)
+-----------------------+      +-----------------------+
| int mx (4 bytes)      |      | vptr (Virtual Pointer)| (4/8 bytes)
+-----------------------+      +-----------------------+
| int my (4 bytes)      |      | int mx (4 bytes)      |
+-----------------------+      +-----------------------+
| Total: 8 bytes        |      | int my (4 bytes)      |
                               +-----------------------+
                               | Total: 12/16 bytes    |
```

### 🔍 Arka Plan (Under the Hood)
"Hayatta hiçbir şey bedava değil." Sanallık eklendiğinde, derleyici nesneye gizli bir **vptr** (Virtual Pointer) ekler. Bu pointer, sınıfın sanal fonksiyon adreslerini tutan **vtable** (Virtual Table) yapısını gösterir.

---

### ⏱ 10 Dakikalık Blok Özeti (Ders Sonu)
Bu bölümde Hoca şu 3 kritik kuralı mühürledi:
1.  **Access Control is Static:** `private` olan bir sanal fonksiyon, taban sınıfın `public` arayüzü üzerinden dinamik olarak çağrılabilir. Erişim kontrolü isme, gönderim (dispatch) türe ilişkindir.
2.  **Default Arguments are Static:** Sanal fonksiyonlardaki varsayılan argümanlar çalışma zamanındaki türe göre değil, derleme zamanındaki türe göre bağlanır.
3.  **Ctor/Dtor Exception:** Constructor ve Destructor içinde sanallık mekanizması "çalışmaz", çünkü nesne ya henüz tam oluşmamıştır ya da ölmektedir.

📌 **Dersin Sonu:** Necati Hoca bu dersi polimorfizmin temel direklerini 
anlatarak bitirdi ve bir sonraki ders için "vtable" detaylarının sözünü 
verdi.


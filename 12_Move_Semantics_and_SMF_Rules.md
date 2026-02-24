Bu ders notları, Necati Ergin'in C++ dersindeki titiz anlatımı ve teknik derinliği esas alınarak, bir bilgisayar mühendisi perspektifiyle yeniden inşa edilmiştir.

---

# 📝 C++ Teknik İnceleme Notları (Ders 12 - Bölüm 1)

## 1. Copy Assignment Function (Kopyalayan Atama Operatörü) ve Self-Assignment
**[00:00 - 05:24]**

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bir nesneye başka bir nesne atandığında, eğer nesne kaynak (resource) kullanıyorsa, derleyicinin yazdığı yüzeysel kopyalama (shallow copy) felakete yol açar. Atama operatörü, hem eski kaynağı geri vermeli hem de yeni kaynağı "Deep Copy" (Derin Kopyalama) ile edinmelidir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, özellikle **Self-Assignment** (Nesnenin kendine atanması) durumuna karşı şu koruma mekanizmasının altını çizdi:

```cpp
MyClass& operator=(const MyClass& other) {
    if (this == &other) { // <-- Kritik Kural: Self-assignment kontrolü
        return *this; // Hiçbir şey yapma (No-op)
    }
    
    // 1. Kendi kaynağını geri ver (Release resources)
    // 2. Deep copy yap (Allocate and copy)
    
    return *this; // L-value referans döndürülür
}
```

### 🔍 Arka Plan (Under the Hood)
Atama operatörü bir "Constructor" değildir. Hayatta olan bir nesnenin durumunu değiştirir. Derleyici eğer işimizi görüyorsa **Rule of Zero** (Sıfır Kuralı) gereği bu fonksiyonun yazımını üstlenir. Eğer sınıf elemanları `std::string` veya `std::vector` gibi kendi kaynağını yöneten sınıflarsa, derleyicinin yazdığı kod bu elemanların kendi `operator=` fonksiyonlarını çağıracağı için güvenlidir.

---

## 2. Copy-Swap Idiom ve Exception Guarantees
**[05:24 - 08:00]**

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Klasik atama operatörü yazımında, yeni kaynak allocate edilirken bir hata (exception) oluşursa, eski kaynak çoktan serbest bırakılmış olabilir. Bu durum nesneyi geçersiz bir durumda bırakır.

### ⚙️ Teknik Detay ve Sentaks
Hoca, **Copy-Swap Idiom**'un temel amacının **Strong Exception Guarantee** (Güçlü İstisna Garantisi) sağlamak olduğunu belirtti.

- **Basic Exception Guarantee:** Hata olsa bile kaynak sızıntısı olmaz, nesne geçerli (valid) ama belirsiz bir durumdadır.
- **Strong Exception Guarantee:** İşlem başarısız olursa nesne işlemden önceki durumunda kalır (Commit or Rollback).
- **No-throw Guarantee:** Fonksiyonun asla hata fırlatmayacağı garantisidir.

🚩 **Mülakat Sorusu:** "Neden atama operatöründe self-assignment kontrolü yapıyoruz?"
**Cevap:** Hem verimlilik için hem de eğer koruma yoksa, nesne kendi kaynağını serbest bırakıp sonra o serbest bıraktığı geçersiz bellekten kopyalama yapmaya çalışacağı için **UB** (Undefined Behavior - Tanımsız Davranış) oluşur.

---

## 3. Big Three'den Big Five'a Geçiş
**[08:00 - 11:30]**

### 📊 Standart Karşılaştırması

| Özellik | C++98 (Big Three) | Modern C++ (Big Five) |
| :--- | :--- | :--- |
| **Bileşenler** | Destructor, Copy Constructor, Copy Assignment | + Move Constructor, Move Assignment |
| **Mantık** | Kaynak yönetimi için 3 fonksiyon yeterliydi. | Efficiency (Verimlilik) için taşıma semantiği eklendi. |

🔗 **Kümülatif Bağlantılar:** Hoca, eğer bir sınıfta `Destructor` yazma ihtiyacı duyuyorsanız (manuel kaynak yönetimi varsa), muhtemelen diğerlerini de yazmanız gerektiğini belirtti. C++11 ile bu kural "Big Five" (Büyük Beşli) oldu.

---

## 4. R-Value References ve Function Overloading
**[11:30 - 20:00]**

Hoca, taşıma semantiğinin (move semantics) kalbi olan overloading mekanizmasını 4 farklı olasılık üzerinden inceledi:

### ⚙️ Teknik Detay ve Sentaks
```cpp
class Nec {
public:
    void func(Nec&);              // 1. L-value ref (Const olmayan)
    void func(const Nec&);        // 2. Const L-value ref (Fallback adayı)
    void func(Nec&&);             // 3. R-value ref
    void func(const Nec&&);       // 4. Const R-value ref (Çok nadir/gereksiz)
};
```

### 🔍 Arka Plan (Value Kategorileri)
Hoca, `R-value` kategorisinin aslında bir "Combined Category" (Birleştirilmiş Kategori) olduğunu hatırlattı:
- **pr-value (Pure R-value):** İsimsiz, geçici nesneler (Örn: `Nec{}`).
- **x-value (eXpiring value):** Hayatı bitmekte olan nesneler (Örn: `std::move(x)` sonucu).
- **r-value:** `pr-value` ve `x-value` toplamı.

---

## 5. Move Doesn't Move (std::move Yanılgısı)
**[17:00 - 25:00]**

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Elinizde bir **L-value** (isimlendirilmiş nesne) var ama artık ona ihtiyacınız yoksa ve kaynağının çalınmasına izin vermek istiyorsanız, onu zorla **R-value** kategorisine çekmeniz gerekir.

### ⚙️ Teknik Detay ve Sentaks
Hoca, Scott Meyers'ın **"std::move doesn't move"** sözüne atıfta bulundu.
```cpp
Nec mynec;
func(std::move(mynec)); // <-- Hoca vurguladı: std::move sadece cast yapar!
```

**Derleyici Gözü (Compiler Cast):**
`std::move(x)` aslında arka planda şudur:
`static_cast<Nec&&>(x)`

🚩 **Kritik Nokta:** `std::move` fonksiyonu isminden dolayı sanki nesneyi taşıyormuş gibi algılanır. Oysa o sadece nesneyi **x-value** kategorisine çekerek derleyiciye "Bunun taşıma overload'unu seçebilirsin" mesajı verir. Taşıma işlemini yapan `Move Constructor` veya `Move Assignment` fonksiyonudur.

---

## 6. Fallback Mekanizması
**[25:00 - 30:00]**

### 🧠 Rationale
Eğer bir sınıfın `Move Constructor`'ı yoksa, ancak biz nesneyi `std::move` ile gönderiyorsak ne olur? Kod patlamaz!

### ⚙️ Teknik Detay
Derleyici, **R-value Reference** (`T&&`) parametreli bir fonksiyon bulamazsa, **Const L-value Reference** (`const T&`) parametreli fonksiyona geri düşer (**Fallback**).

```cpp
void foo(const Nec&); // (1)
// void foo(Nec&&);   // (2) - Eğer bu yoksa...

foo(std::move(n));     // ... (1) numaralı fonksiyon çağrılır.
```

### 🖼️ Görselleştirme (Resource Stealing Concept)
Taşıma semantiği aslında bir "Kaynak Çalma" (Stealing) işlemidir:

```text
Persistent Object (L-value)          Temporary/Gidici Object (R-value)
[Pointer: 0x123] ------------------> [Data: "Çok Büyük Veri"]
      |                                     ^
      | (Deep Copy - Pahalı)                |
      v                                     |
New Object                                  |
[Pointer: 0x456] ---------------------------+

--- MOVE SEMANTICS (Taşıma) ---

Gidici Object (R-value)              [Data: "Çok Büyük Veri"]
[Pointer: 0x123] ------------------+        ^
      |                            |        |
      | (Pointer Copy - Ucuz)      +--------+
      v                                     |
New Object (Hırsız)                         |
[Pointer: 0x123] ---------------------------+
[Gidici Object Pointer -> nullptr] (Kritik: Dangling Pointer önlemi!)
```

---

### 🛡️ 10 Dakikalık Blok Özeti (00:00 - 30:00)
Bu bölümde Hoca şu 3 kritik noktaya dikkat çekti:
1.  **Self-Assignment** kontrolü yapılmazsa kaynak yöneten sınıflarda "kendi bacağına sıkma" durumu (UB) oluşur.
2.  **std::move** nesneyi taşımaz, sadece `static_cast<T&&>` yaparak taşıma fonksiyonlarının çağrılmasına zemin hazırlar.
3.  **Fallback Mekanizması:** Eğer move overload'u yoksa, `const T&` overload'u imdada yetişir; bu sayede eski sınıflar modern C++ kodlarıyla uyumlu çalışır.

Hoca'nın dersindeki tempoyu ve teknik detay hassasiyetini koruyarak, notlarımıza kaldığımız yerden (30. dakika) devam ediyoruz.

---

# 📝 C++ Teknik İnceleme Notları (Ders 12 - Bölüm 2)

## 7. Move Constructor: Kaynak Çalma (Stealing) ve Dangling Pointer Problemi
**[00:30:00 - 00:45:00]**

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Modern C++ öncesinde, hayatı sona erecek olan "ephemeral" (geçici/gidici) bir nesnenin değerini başka bir nesneye aktarırken bile "Deep Copy" yapmak zorundaydık. Hoca'nın deyimiyle: *"Sen gidicisin, öbür dünyaya gideceksin, neden senin kaynağını kopyalayayım? Ben senin kaynağını çalarım!"*

### ⚙️ Teknik Detay ve Sentaks
Taşıma işlemi yapıldığında sadece pointer kopyalanır, ancak asıl kritik nokta kaynağı çalınan nesneyi **"destructible" (yok edilebilir)** bir durumda bırakmaktır.

```cpp
// Move Constructor'ın temel mantığı (String sınıfı örneği)
String(String&& other) : mp{other.mp}, mlen{other.mlen} { // <-- Kaynak çalındı
    other.mp = nullptr;  // <-- KRİTİK: Çalınan nesnenin pointer'ı null yapılmazsa
    other.mlen = 0;      // destructor çağrıldığında aynı kaynağı free etmeye çalışır!
}
```

### 🔍 Arka Plan (Under the Hood)
Eğer `other.mp = nullptr` yapılmazsa, geçici nesnenin ömrü bittiğinde çağrılan `destructor`, bizim yeni nesnemize bağladığımız bellek alanını serbest bırakır. Bu durum yeni nesnemizde bir **Dangling Pointer** (geçersiz adresi gösteren pointer) oluşmasına ve programın çökmesine (Double Free) yol açar.

🚩 **Kritik Nokta:** `R-value` referansı bir referanstır, yani bir tür pointer'dır. Assembly düzeyinde nesnenin adresini taşır. Taşıma işlemini referansın kendisi değil, o referansı kullanan fonksiyonun **implementasyonu** yapar.

---

## 8. Compiler-Generated Move Members (Derleyicinin Yazdığı Taşıma Fonksiyonları)
**[00:45:00 - 01:00:00]**

### 🧠 Rationale
Eğer sınıfın elemanları `std::string` veya `std::vector` gibi zaten "movable" (taşınabilir) türlerse, derleyici otomatik olarak taşıma fonksiyonlarını yazabilir.

### ⚙️ Teknik Detay ve Sentaks
Derleyicinin yazdığı move constructor, sınıfın her bir elemanı için "move" işlemini tetikler. Hoca burada en çok yapılan hatayı vurguladı:

```cpp
class Match {
    A ax; B bx;
public:
    // Derleyicinin yazdığı move constructor şuna benzer:
    Match(Match&& other) 
        : ax{std::move(other.ax)}, bx{std::move(other.bx)} {} 
    // std::move kullanılmazsa (other.ax bir L-value olduğu için) COPY tetiklenir!
};
```

### 🔍 Arka Plan: İsim Formundaki İfadeler
Hoca'nın "altın kuralı": **"İsmi olan her şey L-value'dur."**
Bir değişkenin türü `Nec&&` (R-value reference) olsa bile, o değişkenin ismini kodda kullandığınızda o ifade artık bir **L-value expression**'dır. Bu yüzden elemanları taşırken `std::move` (veya `static_cast<T&&>`) kullanımı zorunludur.

---

## 9. Mülakatların Gözdesi: std::move Kaynağı Gerçekten Çalar mı?
**[01:00:00 - 01:15:00]**

### 🚩 Mülakat Sorusu / Kritik Nokta
**Soru:** `void func(MyClass&& r);` şeklinde bir fonksiyonumuz olsun. `func(std::move(m));` çağrısından sonra `m` nesnesinin kaynağı kesinlikle çalınmış mıdır?

**Cevap:** **HAYIR!**
`std::move` sadece bir cast işlemidir. Kaynağın çalınması için `func` fonksiyonunun içerisinde şuna benzer bir kod olmalıdır:
```cpp
void func(MyClass&& r) {
    MyClass local_obj = std::move(r); // <-- Kaynak burada çalınır!
}
```
Eğer fonksiyonun gövdesi boşsa veya `r` nesnesiyle bir atama/yaratım yapılmıyorsa, nesne sapasağlam durur. `std::move` sadece çalınmaya **zemin hazırlar**.

### 🖼️ Görselleştirme (Memory Layout)
```text
[ Stack Frame: main ]          [ Stack Frame: func ]
m (MyClass)                    r (R-value Ref)
[mp: 0x5555] <---------------- [pointing to m's address]
      |
      v
[ Heap Memory ]
[ 0x5555: "Ders Verisi" ]
```
Hoca: *"Assembly koduna baksanız, `func`'ın parametresinin sadece bir adresten (pointer) ibaret olduğunu görürsünüz."*

---

## 🛡️ 10 Dakikalık Blok Özeti (00:30:00 - 01:15:00)
Bu bölümde Hoca şu 3 hayati noktaya dikkat çekti:
1.  **Zombie Objects:** Taşıma işleminden sonra kaynak nesne "yaşayan bir ölü" (valid but unspecified state) haline gelmelidir. Pointer'ı mutlaka `nullptr` yapılmalıdır.
2.  **Move Inconsistency:** Bir değişkenin türünün `&&` olması, onun otomatik taşınacağı anlamına gelmez. İsimlendirildiği anda `L-value` olur, taşımak için tekrar `std::move` gerekir.
3.  **Handle Classes:** Eğer sınıfın elemanı bir "Handle" (çıplak pointer vb.) ise, derleyicinin yazdığı move fonksiyonu Deep Copy yapmayacağı gibi kaynak çalma işlemini de tam yapamaz (Shallow copy yapar), bu yüzden bu fonksiyonları biz yazmalıyız.

---

Hoca'nın dersindeki en kritik ve "ezber bozan" kısımlara geldik. Taşıma semantiğinin sadece bir hızlandırma aracı değil, bir "durum yönetimi" olduğunu detaylandırıyoruz. Notlarımıza 01:15:00'dan devam ediyoruz.

---

# 📝 C++ Teknik İnceleme Notları (Ders 12 - Bölüm 3)

## 10. Move Assignment: Atama Operatöründe Taşıma
**[01:15:00 - 01:24:00]**

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Var olan bir nesneye, ömrü bitmek üzere olan başka bir nesnenin değerini atarken Deep Copy yapmak gereksiz bir maliyettir. Ancak atama işlemi, yaratım (construction) işleminden farklı olarak, sol taraftaki nesnenin halihazırda bir kaynağa sahip olma ihtimalini yönetmek zorundadır.

### ⚙️ Teknik Detay ve Sentaks
Hoca, sınıfımız için `Move Assignment` fonksiyonunu adım adım inşa etti:

```cpp
String& operator=(String&& other) {
    if (this == &other) { // <-- Hoca: "Self-assignment kontrolü yine şart!"
        return *this;
    }
    
    // 1. Kendi kaynağını geri ver (Release current resource)
    free(mp); // <-- Kritik: Kendi kaynağımızı boşaltmazsak sızıntı (Leak) olur.
    
    // 2. Diğerinin kaynağını çal (Steal)
    mp = other.mp;
    mlen = other.mlen;
    
    // 3. Diğerini "yok edilebilir" (Zombie) hale getir
    other.mp = nullptr; 
    other.mlen = 0;
    
    return *this;
}
```

---

## 11. Move-From State: "Yaşayan Ölü" Nesneler
**[01:24:00 - 01:36:00]**

Hoca bu bölümde çok yaygın bir yanlışı düzeltti: "Kaynağı çalınan nesneyi kullanmak UB (Tanımsız Davranış) değildir."

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Kaynağı çalınmış (move-from state) bir nesneye ne olur? Onu kullanabilir miyiz?

**Cevap:** Standart kütüphane türleri (örn. `std::string`) için nesne **"Valid but Unspecified State"** (Geçerli ama belirsiz durum) içerisindedir.
1.  **Invariantlar Korunur:** Nesnenin iç dengesi bozulmaz (Örn: `length` değeri, pointer'ın durumuyla tutarlıdır).
2.  **Destructible:** Nesne için destructor güvenle çağrılabilir.
3.  **Assignable:** Nesneye yeni bir değer atayarak onu tekrar "hayata" döndürebilirsiniz.

🚩 **Hocanın İdiomu:** "Zombie Objects" – Bu nesneler hala hayattadır ama içleri boştur.

---

## 12. Uygulamalı Optimizasyon: Getline ve Push_back Döngüsü
**[01:36:00 - 01:56:00]**

Hoca, taşıma semantiğinin gerçek dünyadaki gücünü şu örnekle gösterdi:

```cpp
std::string sline;
std::vector<std::string> svec;

while (std::getline(ifs, sline)) {
    // svec.push_back(sline); // <-- HATA/PAHALI: Her satır kopyalanır.
    svec.push_back(std::move(sline)); // <-- OPTİMİZASYON: Kaynak çalınır!
}
```

### 🔍 Arka Plan (Under the Hood)
- `push_back(sline)`: `const string&` overload'unu çağırır -> Deep Copy yapılır.
- `push_back(std::move(sline))`: `string&&` overload'unu çağırır -> Taşıma yapılır.
Hoca burada şunu vurguladı: `sline` döngünün başında `getline` tarafından tekrar doldurulduğu için, onun "Zombie" (boş) durumda olması bir problem yaratmaz. `std::string`'in `move-from` garantisi sayesinde her turda yeni bir kaynak allocate etmekten kurtuluruz.

---

## 13. Non-Copyable ve Move-Only Sınıflar
**[01:56:00 - 02:03:00]**

### 📊 Standart Karşılaştırması

| Sınıf Türü | Kopyalama | Taşıma | Örnek |
| :--- | :---: | :---: | :--- |
| **Copyable** | ✅ | ✅ | `std::string`, `std::vector` |
| **Move-Only** | ❌ | ✅ | `std::unique_ptr`, `std::thread` |
| **Non-Copyable** | ❌ | ❌ | `std::mutex` |

Hoca: *"Öyle varlıklar vardır ki (dosya akışları, mutexler vb.) bunları kopyalamak mantıksal olarak saçmadır. Bu yüzden `copy members` delete edilir."*

---

## 14. Özel Üye Fonksiyonların Otomatik Oluşturulma Kuralları (ALTIN TABLO)
**[02:03:00 - 02:37:00]**

Hoca, Modern C++'ın en karmaşık ezber yüklerinden birini şu kurallarla açıkladı:

### ⚙️ Derleyicinin Otomatik Davranışları
1.  **Herhangi bir constructor** (parametre alan dahil) bildirirseniz: **Default Constructor** "Not Declared" olur (yazılmaz).
2.  **Destructor, Copy Constructor veya Copy Assignment** bildirirseniz: **Move memberlar** "Not Declared" olur (oluşturulmaz).
3.  **Move Constructor veya Move Assignment** bildirirseniz: **Copy memberlar** derleyici tarafından otomatik olarak **DELETE** edilir! (Kritik!)

### 📊 Hoca'nın Teknik Tablosu (Özet)
| Eğer Kullanıcı Bildirirse | Default Constructor | Destructor | Copy Constructor | Copy Assignment | Move Constructor | Move Assignment |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **Hiçbir şey** | Defaulted | Defaulted | Defaulted | Defaulted | Defaulted | Defaulted |
| **Destructor** | Defaulted | User | Defaulted* | Defaulted* | **NOT Declared** | **NOT Declared** |
| **Copy Constructor** | **NOT Declared** | Defaulted | User | Defaulted* | **NOT Declared** | **NOT Declared** |
| **Move Constructor** | **NOT Declared** | Defaulted | **DELETED** | **DELETED** | User | **NOT Declared** |

*\*İşaretli olanlar "Deprecated" (Kullanımdan kaldırılmaya aday) durumlardır. Hoca uyardı: "Asla buna güvenmeyin!"*

---

## 15. Fallback ve Move Memberları Delete Etme Hatası
**[02:37:00 - 02:45:00]**

Hoca dersi çok kritik bir mülakat uyarısıyla bitirdi:
**"Move member'ları asla delete etmeyin!"**

### 🧠 Rationale
Eğer taşıma fonksiyonlarını `delete` ederseniz, derleyici bir `R-value` gördüğünde taşıma fonksiyonunu seçer ama onun "yasaklı" olduğunu görüp hata verir. Eğer hiç bildirmezseniz (Not Declared), derleyici **Fallback** mekanizmasını çalıştırır ve kopyalamaya geri döner.

```cpp
class Bad {
    Bad(Bad&&) = delete; // <-- HATA: Kodu patlatır!
};

class Good {
    // Move constructor hiç bildirilmedi.
    // std::move(obj) gelirse kopyalamaya (fallback) düşer, kod çalışmaya devam eder.
};
```

---

### 🛡️ 10 Dakikalık Blok Özeti (01:15:00 - 02:45:00)
Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:
1.  **Move Assignment ve Sızıntı:** Taşıma yapmadan önce `this` nesnesinin mevcut kaynağını boşaltmayı unutmak bellek sızıntısına yol açar.
2.  **Move-only Sınıf Mantığı:** `unique_ptr` gibi sınıfların neden kopyalanamadığını ama taşınabildiğini anlamamak (sahiplik kavramı).
3.  **SMF Oluşturma Kuralları:** Move constructor yazdığınızda copy constructor'ın derleyici tarafından otomatik silindiğini (deleted) bilmemek, "bu kod neden derlenmiyor?" krizine yol açar.

---


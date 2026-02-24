Bu teknik inceleme dokümanı, **Necati Ergin** tarafından verilen 14. ders gününün (14 Ağustos 2024) transkripti esas alınarak, dersin teknik derinliğine ve hocanın anlatım üslubuna sadık kalınarak yeniden inşa edilmiştir.

---

### **BÖLÜM 1: Friend Bildirimleri ve Erişim Kontrolünün Esnetilmesi [00:00:00 - 00:13:28]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Encapsulation (Kapsülleme) gereği sınıfın `private` bölümü dış dünyaya kapalıdır. Ancak bazı durumlarda, sınıfın kendi kodları sayılan ama teknik olarak üye fonksiyonu olmayan yapıların (Global fonksiyonlar veya yardımcı sınıflar) bu verilere erişmesi gerekir. `friend` mekanizması, bu izni kontrollü bir şekilde vermek için tasarlanmıştır.

⚙️ **Teknik Detay ve Sentaks:**

```cpp
class MyClass {
    int mx;
    void bar();

    // 1. Global bir fonksiyona friendlik vermek
    friend int foo(int); // <-- Hoca: "İsim arama (name lookup) süreci burada çalışmaz."

    // 2. Başka bir sınıfın üye fonksiyonuna friendlik vermek
    // friend int Nec::foo(); // <-- HATA: Nec sınıfı ve foo fonksiyonu bu noktada görünür olmalı!
};

// Global fonksiyon tanımı
int foo(int a) {
    MyClass m;
    m.mx = a; // <-- Hoca: "Friend bildirimini derleyici gördüğü için bu erişim syntax hatası değil."
    m.bar();
    return m.mx;
}
```

🔍 **Arka Plan (Under the Hood):**
*   **Access Specifiers (Erişim Belirleyiciler):** `friend` bildiriminin sınıfın `public`, `protected` veya `private` bölümünde yapılmasının teknik olarak hiçbir farkı yoktur. Erişim kontrolüne tabi değildir.
*   **Forward Declaration (Ön Bildirim):** Global fonksiyonlarda ön bildirim gerekmezken, bir sınıfın üye fonksiyonuna friendlik verirken o sınıfın tam tanımı (Class Definition) derleyici tarafından görülmüş olmalıdır.

🚩 **Kritik Nokta / Mülakat Sorusu:**
**Soru:** Bir sınıfın içinde tanımlanan (inline) `friend` fonksiyonun özelliği nedir?
**Cevap:** Buna **"Hidden Friend"** denir. Sınıf içinde tanımlanmasına rağmen sınıfın üye fonksiyonu değildir (Free function'dır). ADL (Argument Dependent Lookup) mekanizması ile sınıf nesnesi kullanıldığında bulunabilirler.

---

### **BÖLÜM 2: Friendship Kuralları ve Attorney-Client Idiom [00:13:28 - 00:30:00]**

🔗 **Kümülatif Bağlantılar:**
Hoca, kalıtım (Inheritance) konusuna henüz girilmediğini ancak friendliğin kalıtımla olan ilişkisinin mülakatlarda çok sorulduğunu belirtti.

⚙️ **Arkadaşlığın Üç Temel Kuralı:**
1.  **Değişme Özelliği Yoktur (No Symmetry):** A, B'nin arkadaşıysa; B otomatik olarak A'nın arkadaşı olmaz.
2.  **Geçişkenlik Yoktur (No Transitivity):** A, B'ye; B de C'ye friendlik vermişse; C, A'nın `private` bölümüne erişemez.
3.  **Kalıtım Yoluyla Geçmez (No Inheritance):** Taban sınıfa (Base Class) verilen friendlik, türemiş sınıfa (Derived Class) otomatik olarak geçmez. Hoca'nın analojisi: *"Babanızın arkadaşları sizin de arkadaşınız değildir."*

🖼️ **Görselleştirme (Erişim Sınırları):**
```text
[ Sınıf A ] <--- [ Sınıf B (Friend) ]  (B, A'nın içine bakabilir)
    ^               |
    |               X (Geçişkenlik yok)
    |               v
[ Sınıf C ] <--- [ Sınıf B (Friend) ]  (B, C'nin içine bakabilir)
```

🚩 **Kritik Nokta: Attorney-Client Idiom (Avukat-Müvekkil İdiomu):**
**Sorun:** Bir sınıfa `friend` verdiğinizde tüm `private` alanı açarsınız. Sadece seçilmiş birkaç fonksiyonu açmak isterseniz ne yaparsınız?
**Çözüm:** C++'da doğrudan "kısmi friendlik" yoktur. Ancak araya bir "Proxy/Attorney" sınıfı konularak bu sağlanır. Bu sınıf gerçek sınıfa friend olur, client ise sadece bu attorney sınıfını kullanır.

🔍 **Teknik Terim Takibi:**
*   **Idiom (İdiyom):** Dile bağlı, kalıplaşmış yapılar. (Örn: RAII, Copy-Swap).
*   **Pattern (Örüntü/Tasarım Kalıbı):** Dilden bağımsız genel mimari çözümler. (Örn: Observer, Singleton).
*   **Technique (Teknik):** Daha geniş kapsamlı kodlama yaklaşımları. (Örn: Type Erasure).

---

### **BÖLÜM 3: Operator Overloading (Operatör Yüklemesi) Giriş [00:30:00 - 00:51:11]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
User-Defined Types (Kullanıcı tanımlı türler - class/struct) için primitif türler (int, double) gibi doğal bir arayüz sunmak. Kodun ifade gücünü (expressiveness) artırarak yüksek bir soyutlama seviyesi sağlar. Hoca: *"Karmaşık matris çarpımlarını multiply(m1, m2) yerine m1 * m2 olarak yazmak client kodun işini kolaylaştırır."*

⚙️ **Teknik Detay ve Sentaks:**
Operatör overloading aslında "syntactic sugar"dır. Derleyici operatör ifadesini gördüğünde bunu bir fonksiyon çağrısına dönüştürür.

```cpp
Matrix m1, m2, m3;
m3 = m1 + m2; 

// Derleyicinin arka plandaki dönüşümü (iki ihtimal):
// 1. Üye fonksiyon ise:
m3.operator=( m1.operator+(m2) ); 

// 2. Global fonksiyon ise:
operator=( m3, operator+(m1, m2) );
```

🚩 **Mülakat Sorusu / Kritik Nokta:**
**Soru:** Operatör yüklemesi yaparken en önemli tasarım ilkesi nedir?
**Cevap:** **"Intuitive Use" (Sezgisel Kullanım).** Kullanıcı toplama (+) gördüğünde bir ekleme yapılacağını bekler. Eğer resim sınıfında `img1 | img2` yazıp iki resmi birleştiriyorsanız ve bu genel kabul görmüş bir standart değilse, bu kötü bir tasarımdır.

---

### **BÖLÜM 4: Operatör Yüklemenin "Anayasası" (Kurallar) [00:51:11 - 01:02:22]**

Necati Hoca, bu kuralların "alfabe" olduğunu ve asla unutulmaması gerektiğini vurguladı:

1.  **En az bir operand Sınıf/Enum olmalı:** Primitif türlerin (iki int toplamak gibi) davranışını değiştiremezsiniz.
2.  **Yeni operatör yaratılamaz:** Olmayan bir operatörü (Örn: `**` veya `add`) yükleyemezsiniz. Sadece dilin setindeki operatörler yüklenebilir.
3.  **İsimlendirme kısıtı:** Fonksiyon ismi `operator` keyword'ü + `token` şeklinde olmalıdır (`operator+`, `operator[]`).
4.  **Aşağıdaki Operatörler Yüklenemez:**
    *   `sizeof`
    *   `.` (Nokta operatörü)
    *   `?:` (Ternary operator)
    *   `::` (Scope resolution)
    *   `typeid`
    *   `.*` (Member selection - pointer to member)
5.  **Sadece Üye Fonksiyon (Member) Olabilenler:** Bazı operatörlerin global fonksiyon olarak yazılması yasaklanmıştır:
    *   `=` (Assignment)
    *   `[]` (Subscript)
    *   `()` (Function call)
    *   `->` (Member access)
    *   Type-cast operatörleri.

---

### **BÖLÜM 5: Arity ve Parametre Sayısı Kuralları [01:02:22 - 01:19:06]**

**Arity (Operand Sayısı):** Operatörlerin operand sayısı değiştirilemez. Unary (tekil) ise unary, binary (ikili) ise binary kalmalıdır.

📊 **Parametre Sayısı Tablosu:**

| Operatör Türü | Implementasyon Yeri | Parametre Sayısı | Not |
| :--- | :--- | :---: | :--- |
| **Binary (+, -)** | Global Function | 2 | Her iki operand da parametredir. |
| **Binary (+, -)** | Member Function | 1 | Sol operand `*this`'tir. |
| **Unary (!, ~)** | Global Function | 1 | Operand parametredir. |
| **Unary (!, ~)** | Member Function | 0 | Operand `*this`'tir. |

**Derleyici Gözü (Compiler Errors):**
```cpp
class Matrix {
public:
    // HATA: Binary operatörü üye fonksiyon olarak 2 parametreyle yazamazsınız!
    bool operator<(const Matrix& other, int x); 
    // Derleyici şu sebeple kızıyor: "Binary operator < has too many parameters."
};
```

🔗 **Kümülatif Bağlantılar (Special Token Attention):**
Hoca, `*`, `&`, `+`, `-` tokenlarının hem unary hem de binary versiyonları olduğuna dikkat çekti.
*   `operator*` (tek parametreli/member-sıfır): Dereferencing.
*   `operator*` (iki parametreli/member-tek): Multiplication.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  Friendlik verilirken fonksiyon imzasının tam eşleşmesi gerektiği (Namespace/Parametre farkı).
2.  Operatör önceliklerinin (Precedence) overloading ile asla değiştirilemeyeceği (Bir sonraki blokta detaylanacak).
3.  Üye operatör fonksiyonlarında sol operandın gizli `this` pointer'ı olduğu gerçeğinin unutulması.

Bu teknik inceleme dokümanı, **Necati Ergin**'in 14. dersinin ikinci yarısını (01:19:06 - 02:43:19) kapsayan, teknik derinliği optimize edilmiş ders notudur.

---

### **BÖLÜM 6: Çok Anlamlı Tokenlar ve Arity Karışıklığı [01:19:06 - 01:29:16]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
C++'ta bazı karakterler (tokenlar) hem unary (tek operandlı) hem de binary (iki operandlı) bağlamda kullanılır. Operatör yüklemesi yapılırken derleyicinin hangi fonksiyonu çağıracağını ayırt etmesi, parametre sayısına dayanır.

⚙️ **Teknik Detay ve Sentaks:**

```cpp
class MyClass {
public:
    // 1. İşaret Operatörü (Unary Sign Operator)
    MyClass operator+(); // <-- Parametresiz: +m1 kullanımı için.

    // 2. Toplama Operatörü (Binary Addition Operator)
    MyClass operator+(const MyClass&); // <-- Tek parametre: m1 + m2 kullanımı için.

    // 3. İçerik Operatörü (Dereferencing)
    int operator*(); // <-- Parametresiz: *ptr kullanımı.

    // 4. Çarpma Operatörü (Multiplication)
    MyClass operator*(const MyClass&); // <-- Tek parametre: m1 * m2 kullanımı.
};
```

🚩 **Kritik Nokta:**
Hoca, `&` (adres vs. bitwise AND) ve `-` (işaret vs. çıkarma) operatörlerinde de aynı kuralın geçerli olduğunu, parametre sayısının operatörün "kimliğini" belirlediğini vurguladı.

🔍 **Arka Plan (Under the Hood):**
**Static Member Kısıtı:** Operatör fonksiyonları **asla** `static` üye fonksiyon olamaz. Çünkü her operatörün bir nesne (üye ise `this`, global ise argüman) üzerinde işlem yapması zorunludur.

---

### **BÖLÜM 7: Öncelik (Precedence) ve Yön (Associativity) Dokunulmazlığı [01:29:16 - 01:38:40]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Yüklenen operatörler, primitif türlerdeki (int, double) öncelik tablosuna aynen uyar. Bu, dilin parsing (ayrıştırma) tutarlılığını korumak için bir zorunluluktur.

⚙️ **Teknik Detay ve Fonksiyon Çağrı Zinciri:**
Hoca, büyük tam sayıları temsil eden bir `BigInt` (derste `begin` olarak geçti) sınıfı üzerinden şu ifadeyi analiz etti:
`bx = b1 * b2 + b3 * b4 & b5;`

🖼️ **Görselleştirme (Derleyici Gözüyle Dönüşüm):**

```cpp
// 1. İşlem: b1 * b2 (Yüksek öncelik)
auto res1 = b1.operator*(b2);

// 2. İşlem: b3 * b4 
auto res2 = b3.operator*(b4);

// 3. İşlem: res1 + res2
auto res3 = res1.operator+(res2);

// 4. İşlem: res3 & b5 (Düşük öncelik)
auto res4 = res3.operator&(b5);

// 5. İşlem: Atama
bx.operator=(res4);
```

🔍 **Arka Plan (Memory Layout):**
Her operatör çağrısı aslında geçici nesneler (temporary objects) oluşturabilir. Hoca, bu zincirleme çağrıların isimleriyle (functional notation) yazılmasının ne kadar "çirkin" ve okunamaz olduğunu, operatör yüklemenin burada devreye girerek kodu temizlediğini belirtti.

---

### **BÖLÜM 8: Neden Global Operatör Fonksiyonları? (Simetri Problemi) [01:38:40 - 01:55:00]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Üye fonksiyonlar sol operandın sınıfa ait olmasını zorunlu kılar (`m1 + 5` gibi). Ancak `5 + m1` yazmak istendiğinde, sol operand `int` (primitif) olduğu için üye fonksiyon çağrılamaz. Global fonksiyonlar bu simetriyi sağlar.

⚙️ **Teknik Detay (Inserter Örneği):**
`std::ostream` (cout) sınıfı değiştirilemez bir kütüphane kodudur. Kendi sınıfımızı `cout << m1` şeklinde yazdırmak için `ostream` sınıfına üye ekleyemeyeceğimizden, global bir operatör yazmak **tek çaredir.**

```cpp
class Matrix { /* ... */ };

// Global Inserter (Stream operator)
std::ostream& operator<<(std::ostream& os, const Matrix& m) {
    // os << m.data...
    return os; // <-- Hoca: "Zincirleme kullanım (cout << a << b) için nesne geri döndürülmeli."
}
```

🔗 **Önceki Derslerle Bağlantı:**
Referans semantiğinin neden C++'a eklendiğinin en büyük kanıtı operatör overloadingdir. Pointerlar ile bu doğal arayüz (syntax) oluşturulamazdı.

---

### **BÖLÜM 9: Const Correctness ve Sektörel "Red Flag"ler [01:55:00 - 02:11:00]**

🚩 **Mülakat Sorusu / Kritik Nokta:**
Hoca, "çalışıyor ama ortalık kan gölü" (blood bath) deyimini kullanarak, `const` olması gereken fonksiyonların `const` yapılmamasını profesyonel bir "kırmızı bayrak" (red flag) olarak nitelendirdi.

⚙️ **Teknik Detay (Doğru Yazım):**
*   **Side Effect (Yan Etki) Olmayanlar:** `+`, `-`, `==`, `<` gibi operatörler nesneyi değiştirmez. **Mutlaka `const` üye fonksiyon** olmalı ve parametreleri **`const T&`** almalıdır.
*   **Mutatorlar (Nesneyi Değiştirenler):** `+=`, `-=`, `++` gibi operatörler nesneyi değiştirir. Bunlar `const` **yapılamaz.**

🔍 **Derleyici Gözü (Undefined Behavior):**
Hoca, otomatik ömürlü (stack) bir nesneye referans döndürmenin (`return by reference`) en yaygın "dangling reference" ve dolayısıyla UB (Tanımsız Davranış) sebebi olduğunu hatırlattı.

---

### **BÖLÜM 10: Geri Dönüş Türleri ve Optimizasyonlar [02:11:00 - 02:26:00]**

🧠 **Neden İhtiyaç Duyuldu? (Rationale):**
Operatörlerin ne döndüreceği dilin kuralı değil, "problem domain"in (sorun alanının) bir parçasıdır.

📊 **Standart Karşılaştırması (Geri Dönüş Tipleri):**

| Operatör | Tipik Geri Dönüş Türü | Neden? |
| :--- | :--- | :--- |
| `+`, `-`, `*` | **Value (T)** | Yeni bir değer oluşur, stack nesnesi döner. |
| `+=`, `-=` | **Reference (T&)** | Sol operandın kendisi değişir ve o döner. |
| `==`, `<` | **bool** | Karşılaştırma sonucu mantıksaldır. |

🔍 **Arka Plan (NRVO & Move Semantics):**
Hoca, "Value döndürmek pahalıdır" inanışının modern C++'ta eskidiğini belirtti.
1.  **Mandatory Copy Elision:** PR-value dönerken kopyalama yapılmaz.
2.  **NRVO (Named Return Value Optimization):** Derleyici kopyalamayı eler.
3.  **Move Semantics:** Optimizasyon yapılamazsa nesne "taşınır", kopyalanmaz.

🚩 **Kritik Nokta:** `[[nodiscard]]` atribütü. `m1 + m2;` yazıp sonucu kullanmamak mantıksal hatadır. Bu atribüt derleyicinin uyarı vermesini sağlar.

---

### **BÖLÜM 11: C++20 ve Spaceship Operatörü (<=>) [02:26:00 - 02:43:19]**

📊 **C++17 vs C++20 Karşılaştırması:**
*   **C++17 Öncesi:** Bir sınıf için tüm karşılaştırmaları yapmak için 6 operatörü de (`==`, `!=`, `<`, `<=`, `>`, `>=`) tek tek yazmak zorundaydık.
*   **C++20:** Sadece `operator<=>` (Spaceship) ve `operator==` yazılması yeterlidir. Derleyici diğer tüm eşitsizlikleri bu operatörden "durumdan vazife çıkartarak" türetir.

⚙️ **C++20'nin Getirdiği Kolaylık:**
C++20 ile derleyici artık operatörleri "rewrite" (yeniden yazma) yeteneğine sahiptir. `a != b` ifadesini otomatik olarak `!(a == b)` ifadesine dönüştürebilir.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1.  `+` ve `==` gibi operatörleri `const` yapmamak (Semantik facia).
2.  Operatör yüklemesinin sadece "syntax sugar" olduğunu unutup, içinde lojik dışı işler yapmak (Non-intuitive design).
3.  Geçici nesne döndüren fonksiyonlardan referans döndürmeye çalışmak (Dangling Reference).

📌 **Ders Adı:** **C++'ta Operatör Yükleme Anatomisi: Kurallar, Tasarım İlkeleri ve Simetri**


Hoş geldin meslektaşım. Necati Ergin Hocamızın 7. dersinde (22 Temmuz 2024) tuttuğum en ön sıra notlarımı, seninle o meşhur titizlikle paylaşıyorum. Hocanın "özetleme, yeniden inşa et" prensibiyle, her virgüle ve her assembly imasına dikkat ederek ilerleyelim.

---

# C++ DERS NOTLARI: 7. DERS (FUNCTION OVERLOADING & RESOLUTION)

## 1. BÖLÜM: Overloading Temelleri ve İmza (Signature) Kavramı
**Zaman Damgası:** [00:00:00] - [00:10:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C dilinde aynı işi farklı türler için yapan fonksiyonlara farklı isimler vermek zorundaydık (Örn: `abs`, `labs`, `fabs`). Bu, programcının zihinsel yükünü artırıyor ve jenerik kod yazımını imkansız kılıyordu. Function Overloading (Fonksiyon Yüklemesi), aynı ismin farklı türlerle "doğal" bir şekilde çalışmasını sağlar.

### ⚙️ Teknik Detay ve Sentaks
Function Overloading'in gerçekleşmesi için üç temel şart vardır:
1. İsimler aynı olacak.
2. Skoplar (Kapsamlar) aynı olacak.
3. İmzalar (Signature) farklı olacak.

**Kritik Kural:** Eğer kapsamlar farklıysa overloading değil, **Name Hiding** (İsim Gizleme/Maskeleme) oluşur.

```cpp
// --- ÖRNEK 1: Parametrenin Constluğu ---
void func(int x);
void func(const int x); // <-- HATA: Redecalartion (Yeniden bildirim). 
                        // Top-level const imzanın bir parçası değildir!

// --- ÖRNEK 2: Pointer ve Const ---
void ptr_func(int* p);
void ptr_func(int* const p); // <-- HATA: Redeclaration. Top-level const fark yaratmaz.
void ptr_func(const int* p); // <-- BAŞARILI: Bu bir Overloading'dir. 
                             // Low-level const (Pointer to Const) imzayı değiştirir.

// --- ÖRNEK 3: Type Alias (Eş İsim Bildirimi) ---
using byte = unsigned char;
void foo(unsigned char);
void foo(byte); // <-- HATA: Redeclaration. 'byte' sadece bir takma isimdir (alias).

// --- ÖRNEK 4: Üç Ayrı Char Türü (Distinct Types) ---
void bar(char);
void bar(signed char);
void bar(unsigned char); // <-- BAŞARILI: C++'da bu üçü "Distinct Type" (birbirinden ayrı tür) kabul edilir.
```

### 🔍 Arka Plan (Under the Hood)
Derleyici, parametre değişkeninin kendisinin `const` olmasını (Top-level const) fonksiyonun dış dünyayla olan arayüzünü etkilemediği için göz ardı eder. Ancak pointer'ın gösterdiği yerin `const` olması (Low-level const), fonksiyonun o nesneyi değiştirip değiştiremeyeceği bilgisini taşıdığından imzanın parçasıdır.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** Parametresi dizi olan fonksiyonlarda overloading nasıl çalışır?
**Cevap:** Dizi parametreleri her zaman **Decay** (Bozulma/Dönüşme) uğrayarak pointer'a dönüşür.
```cpp
void baz(int p[]);   // Derleyici bunu 'void baz(int* p)' olarak görür.
void baz(int* p);    // <-- HATA: Redeclaration!
void baz(int p[7]);  // <-- HATA: Redeclaration! Derleyici boyutu (7, 8, 9) dikkate almaz.
```

---

## 2. BÖLÜM: Gelişmiş Overloading ve Fonksiyon Pointer'ları
**Zaman Damgası:** [00:10:00] - [00:20:00]

### ⚙️ Teknik Detay ve Sentaks
Hoca burada dizi adresleri ve fonksiyon pointer'ları arasındaki ince farka dikkat çekti.

```cpp
// --- ÖRNEK 5: Dizi Adresleri (Pointer to Array) ---
void alpha(int (*p)[10]);
void alpha(int (*p)[12]);
void alpha(int (*p)[16]); // <-- BAŞARILI: Bunlar 3 ayrı overload'dur! 
                          // Boyut, dizi pointer'ının türünün bir parçasıdır.

// --- ÖRNEK 6: Referans Semantiği ile Dizi ---
void beta(int (&p)[10]);
void beta(int (&p)[12]); // <-- BAŞARILI: Bu da 3 ayrı overload oluşturur.

// --- ÖRNEK 7: Function Pointer Decay ---
void who(int(int));        // Parametre: Function Type
void who(int(*)(int));     // Parametre: Function Pointer Type
// <-- HATA: Redeclaration! Fonksiyon türü, parametre olarak yazıldığında 
// "Function Pointer"a decay (dönüşüm) olur.
```

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
"Most Vexing Parse" probleminin (Scott Meyers'ın uydurduğu terim) nedenlerinden biri de budur. Derleyicinin bir bildirimi fonksiyon mu yoksa nesne mi olarak algılayacağı konusundaki belirsizlik, bu decay kurallarından kaynaklanır.

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `const T*` ile `T* const` farkını overloading üzerinden açıklayın.
**Cevap:** `T* const` (Top-level) imzada fark yaratmazken, `const T*` (Low-level) yaratır. Hoca buna "Const Overloading" dendiğini (standart bir terim olmasa da popüler kullanım olduğunu) belirtti.

---

## 3. BÖLÜM: Function Overload Resolution (Çözümleme)
**Zaman Damgası:** [00:20:00] - [00:30:00]

Hoca burada en çok hata yapılan yere geldi: **"Sezgilere güvenmek yerine dilin kurallarına güvenmek."**

### 🔍 Arka Plan (Under the Hood)
Function Overload Resolution süreci üç sonuçla biter:
1. **Success (Başarı):** Bir fonksiyon seçilir.
2. **No Viable Function (Uygun Fonksiyon Yok):** Legal çağrı yapılamaz.
3. **Ambiguity (Çift Anlamlılık/Belirsizlik):** Birden fazla uygun fonksiyon var ama birinin diğerine üstünlüğü yok.

### 📊 Standart Karşılaştırması: Sezgi vs. Gerçek
Hoca şu örneği "mülakatlarda sizi patlatırlar" diyerek verdi:

```cpp
void full(long double);
void full(char);

int main() {
    full(4.5); // 4.5'in türü 'double'dır.
}
```
**Sezgi:** `double`'dan `long double`'a dönüşüm veri kaybı yaratmaz, `char`'a dönüşüm risklidir. O halde `long double` çağrılır.
**Gerçek (C++ Kuralı):** **Ambiguity (Belirsizlik) Hatası!** Çünkü her ikisi de "Standard Conversion" (Standart Dönüşüm) kategorisindedir ve birbirlerine üstünlükleri yoktur.

---

## 4. BÖLÜM: Çözümleme Adımları: Candidate & Viable Functions
**Zaman Damgası:** [00:30:00] - [00:46:17]

### ⚙️ Teknik Detay ve Sentaks
Resolution süreci adım adım gerçekleşir:

1. **Candidate Functions (Aday Fonksiyonlar):** Aynı skoptaki, aynı isimli tüm fonksiyonlar. (Argüman sayısı veya türü henüz önemli değil).
2. **Viable Functions (Uygun Fonksiyonlar):** Adaylar arasından, verilen argümanlarla legal olarak çağrılabilecek olanlar.
   - Argüman sayısı parametre sayısına uygun mu? (Default argumentlar dahil).
   - Her bir argümandan ilgili parametreye **Implicit Conversion** (Örtülü Dönüşüm) var mı?

**Önemli:** `int`'den `enum` türlerine örtülü dönüşüm yoktur! Bu yüzden parametresi `enum` olan bir fonksiyon, `int` bir argümanla "Viable" (Uygun) kabul edilmez.

### 🚩 Kritik Nokta: Variadic Functions (Değişken Sayıda Parametre)
Hoca, Variadic Functions (... üç nokta) için çok sert bir kural koydu: **"Variadic, her zaman kaybeder."**

```cpp
void v_func(int);
void v_func(...); // Variadic

int main() {
    v_func(10); // Her ikisi de viable (uygun). Ama variadic en kötü kalitedir.
}
```

### 🖼️ Görselleştirme: Dönüşüm Hiyerarşisi (Kötüden İyiye)
```text
[ EN KÖTÜ ]  Variadic Conversion (...)
     ^
     |       User Defined Conversion (Sınıf Constructor/Conversion Op)
     ^
     |       Standard Conversion (int->double, double->char vb.)
[ EN İYİ  ]  Exact Match (Tam Uyum)
```

---

## 5. BÖLÜM: User Defined Conversions (Programcı Tanımlı Dönüşümler)
**Zaman Damgası:** [00:46:17] - [01:00:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C'de olmayan bu özellik, sınıfların (class/struct) temel türler gibi davranmasını sağlar. Bir sınıfın, başka bir türe nasıl dönüşeceğini programcı belirler.

### ⚙️ Teknik Detay ve Sentaks
Hoca, henüz sınıflara girmediğimiz için sadece resolution açısından önemine değindi. Derleyicinin "durumdan vazife çıkartıp" (Necati Hoca'nın deyimiyle) uygun bir constructor çağrısı yapmasıdır.

```cpp
struct Nec {
    Nec(int); // Constructor: int'ten Nec'e dönüşüm sağlar.
};

void target(Nec);
void target(...);

int main() {
    target(5); // 1. User Defined Conversion (int -> Nec)
               // 2. Variadic Conversion (...)
               // Sonuç: Nec çağrılır. UDC, Variadic'e baskın gelir!
}
```

### 🚩 Kritik Nokta / Mülakat Sorusu
**Soru:** `std::move` aslında ne yapar?
**Hoca'nın İdiomu:** "Yanlış isimlendirilmiş bir fonksiyondur. İsmi `move_cast` veya `rvalue_cast` olmalıydı."
**Cevap:** `std::move` hiçbir şeyi taşımaz! Sadece bir L-value'yu R-value kategorisine kest eder (dönüştürür). Bu, Resolution sırasında derleyicinin R-value referanslı overload'u seçmesini sağlar.

---

Necati Ergin Hocamızın 7. dersine devam ediyoruz meslektaşım. Dersin bu bölümünde hoca, C++'ın en "tehlikeli" ve mülakatların vazgeçilmez konusu olan **Overload Resolution (Yükleme Çözümleme)** hiyerarşisinin derinliklerine, atomik düzeyde indi. Notlarımı hocanın o meşhur "cehaletle savaşan" üslubuyla tutmaya devam ediyorum.

---

# C++ DERS NOTLARI: 7. DERS (DEVAM)

## 6. BÖLÜM: Nullptr ve C++ Eleştirilerine Teknik Bakış
**Zaman Damgası:** [01:00:00] - [01:04:30]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
C'den miras kalan `NULL` makrosu aslında `0` tam sayısıdır. Bu durum, parametresi `int` ve `int*` olan iki fonksiyonun yüklenmesi durumunda derleyicinin tam sayı olanı seçmesine (Exact Match) neden olur. `nullptr` bu belirsizliği ortadan kaldırmak için "pointer türlerine dönüşebilen ama tam sayı olmayan" bir tür olarak dile eklendi.

### 🚩 Kritik Nokta / Mülakat Sorusu
Hoca burada LinkedIn gibi platformlarda C++'ı "güvenli değil" diyerek eleştirenlere teknik bir tokat attı.
**Hoca'nın İdiomu:** "Hap tabiriyle tırışka! C++'ı suçlayanların %99'u dili ya hiç bilmiyor ya çok az biliyor."
**Soru:** `nullptr` dilden kaldırılabilir mi?
**Cevap:** Hayır, imkansız. `dynamic_cast` gibi operatörlerden standart kütüphane fonksiyonlarına kadar her yer `null pointer` semantiği üzerine kurulu. Bu "inkompetent" (yetersiz) programcıların değil, dilin bir gerekliliğidir.

---

## 7. BÖLÜM: Standard Conversion (Standart Dönüşümler) ve Hiyerarşi
**Zaman Damgası:** [01:04:30] - [01:09:00]

### ⚙️ Teknik Detay ve Sentaks
Eğer adaylar arasında "User Defined Conversion" (Kullanıcı Tanımlı Dönüşüm) yoksa, derleyici **Standard Conversion** (Standart Dönüşüm) kalitelerine bakar:

1. **Exact Match (Tam Uyum):** En kaliteli.
2. **Promotion (Terfi/Yükseltme):** Orta kalite.
3. **Conversion (Dönüşüm):** En düşük kalite (Ama legal).

**Kritik Uyarı:** `long double`'dan `char`'a dönüşüm veri kaybı yaratsa dahi, C++ kuralları gereği hala bir "Standard Conversion"dır.

---

## 8. BÖLÜM: Exact Match (Tam Uyum) Kategorileri
**Zaman Damgası:** [01:09:00] - [01:15:00]

Hoca, programcıların "bu nasıl tam uyum olur?" dediği 4 özel durumu açıkladı. Bunlar teknik olarak **Exact Match** kabul edilir:

### 🔍 Arka Plan (Under the Hood)
| Dönüşüm Adı | Örnek | Açıklama |
| :--- | :--- | :--- |
| **Identity** | `int` -> `int` | Tamamen aynı tür. |
| **Const Conversion** | `int*` -> `const int*` | Low-level const eklenmesi. |
| **Array-to-Pointer Decay** | `int[]` -> `int*` | Dizi isminin ilk eleman adresine dönüşmesi. |
| **Function-to-Pointer** | `void f()` -> `void(*)()` | Fonksiyon isminin adresine dönüşmesi. |
| **L-value to R-value** | `x` -> `x + 0` | Nesnenin içindeki değerin okunması. |

```cpp
void foo(const int* p); // Overload (A)
void bar(int* p);       // Overload (B)

int main() {
    int x = 10;
    foo(&x); // <-- EXACT MATCH! (int* 'dan const int*' a dönüşüm tam uyumdur)
}
```

---

## 9. BÖLÜM: Promotion (Terfi) - Karıştırılan İnce Çizgi
**Zaman Damgası:** [01:15:00] - [01:26:00]

### ⚙️ Teknik Detay ve Sentaks
Promotion, Conversion'dan daha kalitelidir. Sadece iki alt kategorisi vardır:
1. **Integral Promotion:** `char`, `short`, `bool` türlerinin `int` türüne dönüşmesi.
2. **Floating Point Promotion:** **SADECE** `float`'tan `double`'a dönüşüm.

**DİKKAT:** `double`'dan `long double`'a dönüşüm bir "Promotion" değildir, "Conversion"dır! (Mülakatlarda buradan vururlar).

```cpp
void func(int);    // (A)
void func(double); // (B)

int main() {
    char c = 'A';
    func(c); // <-- (A) ÇAĞRILIR! Çünkü char->int PROMOTION iken char->double CONVERSION'dır.
    
    float f = 1.0f;
    func(f); // <-- (B) ÇAĞRILIR! Çünkü float->double PROMOTION'dır.
}
```

### 🚩 Kritik Nokta
**Soru:** `int` argüman ile `unsigned int` ve `long double` parametreli fonksiyonlar çağrılırsa ne olur?
**Cevap:** **Ambiguity!** Çünkü her ikisi de "Conversion" kategorisindedir. Tam sayı türünün tam sayı türüne (int -> unsigned int) önceliği yoktur!

---

## 10. BÖLÜM: Const Overloading ve Referans Semantiği
**Zaman Damgası:** [01:26:00] - [01:34:00]

### 🧠 Neden İhtiyaç Duyuldu? (Rationale)
Bazen nesne `const` ise farklı (sadece okuma yapan), `const` değilse farklı (değiştirme yetkisi olan) bir kodun çalışmasını isteriz.

### ⚙️ Teknik Detay ve Sentaks
```cpp
void process(int& r);       // (1) L-value Ref
void process(const int& r); // (2) Const L-value Ref (Read-only)

int main() {
    int a = 5;
    const int b = 10;
    
    process(a); // (1) Çağrılır. (L-value, non-const'u tercih eder)
    process(b); // (2) Çağrılır. (Const nesne sadece const ref'e bağlanabilir)
}
```
**Kural:** `const` olmayan nesne adresi/referansı, her ikisi de viable (uygun) olsa bile, her zaman `const` olmayan parametreyi seçer.

---

## 11. BÖLÜM: nullptr_t ve Boolean İstisnası
**Zaman Damgası:** [01:40:00] - [01:47:00]

### 🔍 Arka Plan (Under the Hood)
Hoca burada Resolution'daki en garip istisnalardan birini gösterdi:

```cpp
void test(void* p); // (A)
void test(bool b);  // (B)

int main() {
    int x = 10;
    test(&x); // <-- (A) ÇAĞRILIR! 
}
```
**İstisna Rule:** Pointer türlerinden `void*`'a dönüşüm ve `bool`'a dönüşümün her ikisi de "Conversion" olmasına rağmen, `void*` parametresi her zaman `bool` parametresine baskın gelir.

---

## 12. BÖLÜM: L-Value vs R-Value Overloading (Modern C++11)
**Zaman Damgası:** [01:47:00] - [02:10:00]

### 🚩 Kritik Nokta: Value Category vs Data Type
Hoca burada sesini yükselterek uyardı: **"Re değişkeninin ismi bir L-Value Expression'dır!"**

```cpp
void vfunc(int&& rr);      // (A) R-value Ref
void vfunc(const int& r);  // (B) Const L-value Ref

void handle(int&& re) {
    vfunc(re); // <-- DİKKAT: (B) ÇAĞRILIR!
    // Neden? Çünkü 're' bir isimdir ve isimler L-VALUE'dur. 
    // Türü R-value ref olsa bile, kendisi l-value expression'dır!
    
    vfunc(std::move(re)); // <-- ŞİMDİ (A) ÇAĞRILIR!
}
```

### 🔍 Arka Plan (Under the Hood)
`std::move`, Hoca'nın deyimiyle bir "durumdan vazife çıkartma" aracıdır. Nesneyi taşımaz, sadece derleyiciye "bu nesneyi bir R-Value gibi gör" der.

---

## 13. BÖLÜM: Çok Parametreli Resolution (The Multi-Param Rule)
**Zaman Damgası:** [02:20:00] - [02:31:00]

### ⚙️ Teknik Detay ve Sentaks
Birden fazla parametre olduğunda derleyici şu altın kuralı uygular:
Bir fonksiyonun "Best Match" seçilmesi için;
1. En az bir parametrede diğer adaylardan **DAHA İYİ** olmalı.
2. Diğer hiçbir parametrede diğer adaylardan **DAHA KÖTÜ** olmamalı.

```cpp
void f(int, double);    // (1)
void f(double, int);    // (2)

f(10, 10); // <-- AMBIGUITY! 
// (1) birinci parametrede daha iyi (int-int), ama ikincide daha kötü.
// (2) ikinci parametrede daha iyi, ama birincide daha kötü.
```

### 🖼️ Görselleştirme: Skor Tablosu Mantığı
| Aday | Param 1 | Param 2 | Karar |
| :--- | :--- | :--- | :--- |
| Func A | Exact Match | Conversion | - |
| Func B | Conversion | Exact Match | **Ambiguity** |
| Func C | Exact Match | Exact Match | **Best Match** |

---

### 🔗 Önceki Derslerle Bağlantı
Hoca, `auto` type deduction (4. ders) ve `value categories` (5-6. ders) konularının burada nasıl "meyve verdiğini" belirtti. `auto x = c1 + c2;` örneğiyle **Integral Promotion** kuralını sağlamlaştırdı.

---

**Bu bölümde Hoca şu 3 kritik hataya dikkat çekti:**
1. `float`'tan `double`'a geçişin **Promotion**, ama `double`'dan `long double`'a geçişin **Conversion** olduğu.
2. Bir R-Value referansın **isminin** aslında bir L-Value olduğu.
3. `0` (sıfır) sabitinin `int` ve `pointer` yüklemelerinde yarattığı tehlike.


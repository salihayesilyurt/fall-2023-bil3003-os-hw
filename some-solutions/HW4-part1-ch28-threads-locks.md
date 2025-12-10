## 🔒 Kilit Simülasyonu Sorularının Cevapları

### 1\. Soru: `flag.s` Kodunun İncelenmesi

#### ❓ Soru

`flag.s`'yi inceleyin. Bu kod, kilitlemeyi tek bir bellek bayrağıyla "uygular". Assembly'yi anlayabiliyor musunuz? 

#### ✍️ Cevap ve Yorum

`flag.s` dosyası, kritik kesimi korumak için **yetersiz** olan, basit bir bayrak (`flag`) değişkeni (`mutex` adında) kullanan bir **yazılımsal kilit** (spin lock) uygulamasıdır.

| Adres | Talimat | Açıklama |
| :---: | :--- | :--- |
| `.acquire` | `mov flag, %ax` | Bayrağın mevcut değerini (`0`: serbest, `1`: meşgul) **oku** (load). |
| | `test $0, %ax` | Okunan değeri (`%ax`) $0$ ile karşılaştır. |
| | `jne .acquire` | Bayrak $0$'a eşit değilse (yani meşgul ise), `acquire` döngüsüne **geri dön** (spin-wait). |
| | `mov $1, flag` | Bayrak $0$'a eşitse (yani serbest ise), bayrağı **1 yap** (store) ve kilidi al. **(KRİTİK BÖLÜM: Yarış koşulu burada oluşur)** |
| **Kritik Kesim** | `mov count, %ax`, `add $1, %ax`, `mov %ax, count` | Ortak değişkeni (`count`) atomik olmayan bir şekilde artır. |
| `.release` | `mov $0, flag` | Kilidi serbest bırakmak için bayrağı **$0$ yap** (store). |

**Assembly Yorumu:** Kilit alma bloğu (`acquire`) atomik değildir. Kilit durumunu kontrol eden (`test`) ve kilitleyen (`mov $1, flag`) adımlar arasında bir kesme (`interrupt`) meydana gelirse, iki iş parçacığı da bayrağı `0` olarak okuyabilir ve her ikisi de kritik kesime girebilir

-----
### Örnek Ekran Görüntüsü (1. Soru)

<img width="400" height="400" alt="image" src="https://raw.githubusercontent.com/salihayesilyurt/fall-2023-bil3003-os-hw/refs/heads/main/public-images/HW4-P1-Q1.png" />

-----

### 2\. Soru: `flag.s`'nin Varsayılanlarla Çalışması ve Tahmin

#### ❓ Soru

 Varsayılanlarla çalıştırdığınızda `flag.s` çalışır mı?   `flag`'de hangi değerin yer alacağını tahmin edebilir misiniz? 

#### ✍️ Cevap ve Yorum

**Varsayılan Ayarlar:**

  * İş Parçacığı Sayısı (`-t`): $2$
  * Döngü Sayısı (`-a bx=1,bx=1`): Her iş parçacığı 1 kez döngü yapar.
  * Kesme Frekansı (`-i`): $50$ (Çok yüksek, neredeyse kesmesiz çalışma anlamına gelir.)

**Tahmin ve Gerekçe:**
Kesme frekansı çok yüksek olduğu için (`-i 50`), bir iş parçacığının kritik kesimdeki 3 talimat arasında kesilme olasılığı çok düşüktür. İlk iş parçacığı (T0), kritik kesimi (3 talimat) kesintisiz tamamlayacak, ardından diğeri (T1) çalışacaktır.

  * **T0:** Kilidi alır, `count`'u $1$'e çıkarır, kilidi serbest bırakır, `halt` eder.
  * **T1:** Kilidi alır, `count`'u $2$'ye çıkarır, kilidi serbest bırakır, `halt` eder.

**Sonuç:** `count` değişkeninin sonunda olması gereken değer **2**'dir. Eğer **Yarış Koşulu** oluşursa (düşük ihtimal), `count` değeri $1$ veya $2$ olabilir.  Yarış koşulunda her iki iş parçacığı da kritik kesime girerse, her ikisi de `count`'u $0$'dan $1$'e yükseltir ve sonunda **$1$** sonucu yanlış olur.

| Sonuç | Gerekçe |
| :--- | :--- |
| **Doğru Çalışma (Yüksek `-i`)** | `count = 2`. Kesme yok, her iş parçacığı sırayla $1$ artırır. |
| **Yarış Koşulu (Düşük `-i`)** | `count = 1`. İki iş parçacığı da $0$ okur, ikisi de $1$ yazar, biri diğerinin artırmasını kaybetmiş olur. |

-----

### Örnek Ekran Görüntüsü (2. Soru)

<img width="1168" height="1172" alt="image" src="https://raw.githubusercontent.com/salihayesilyurt/fall-2023-bil3003-os-hw/refs/heads/main/public-images/HW4-P1-Q2.png" />

-----

### 3\. Soru: `-a bx=2,bx=2` Ayarının Etkisi

#### ❓ Soru

`%bx` yazmacının değerini `-a bx=2,bx=2` ile değiştirin. Kod ne işe yarar?  Yukarıdaki soruya verdiğiniz cevabı nasıl değiştirir? 

```sh
prompt> python3 x86.py -p flag.s -a bx=2,bx=2 -i 10

```

#### ✍️ Cevap ve Yorum

**Yeni Ayar:**

  * Döngü Sayısı: Her iş parçacığı **2 kez** döngü yapar.
  * Toplam Artış Sayısı: $2 \times 2 = 4$

**Kodun İşlevi:** `-a bx=2,bx=2` bayrağı, her iş parçacığının kritik kesimdeki `count` değişkenini **ikişer kez** artırmasını sağlar. Kritik kesimden sonraki döngü kontrol kodu:

```assembly
sub  $1, %bx  # %bx'i 1 azalt
test $0, %bx
jgt .top      # %bx > 0 ise .top'a geri dön
```

**Tahminin Değişimi:**
İki iş parçacığı toplam 4 kez artış yapmalıdır, dolayısıyla `count`'un doğru değeri **4** olmalıdır.

Ancak, `flag.s`'nin kilit mekanizması atomik olmadığından, **yarış koşulu** riski artık daha fazladır. Düşük kesme frekansında bile, her döngüde yarış koşulu oluşabilir. Eğer her döngüde bir artış kaybedilirse (örneğin T0 $0$ okur, kesilir, T1 $0$ okur ve $1$ yazar, T0 devam edip $1$ yazar), sonuç $4$'ten daha düşük olabilir.

  * **Doğru Sonuç:** `count = 4`.
  * **Kötü Sonuç (Yarış Koşulu):** `count = 1, 2, 3`. (En kötü senaryo olan $1$, her döngüde bir artışın kaybedilmesidir, ancak bu, simülatörün kesme noktasının sürekli aynı yere denk gelmesini gerektirir.)

-----

#### 3. Soruya Özel- Detaylı Yorum

Trace (izleme) kaydı, teorik olarak **hatalı** bir kilit mekanizmasına (`flag.s`) sahip olmamıza rağmen, şans eseri (ve deterministik zamanlama sayesinde) programın **doğru** çalıştığı nadir bir senaryoyu gösteriyor.

Adım adım inceleme:

### Çıktı Analizi (Trace Üzerinden)

Trace kaydını satır satır takip ettiğimizde olaylar şöyle gelişiyor:

1.  **Thread 0 (İlk Tur):**
    * 1000-1007 satırlarını çalıştırır. Kilidi alır, `count` değerini artırır (Count = 1) ve kilidi bırakır (`flag=0`).
    * 1008 ve 1009. satırları çalıştırır.
    * **Kesme (Interrupt):** Tam 10. komutta (satır 1009) kesme gelir.
    * **Durum:** Kilit serbest, Thread 0 döngü kontrolünde.

2.  **Thread 1 (İlk Tur):**
    * Thread 0 kilidi bıraktığı için Thread 1, 1000-1003 arasında beklemez, kilidi hemen alır.
    * `count` değerini artırır (Count = 2).
    * 1007'de kilidi bırakır.
    * **Kesme:** Tam 10. komutta (satır 1009) kesme gelir.
    * **Durum:** Kilit serbest.

3.  **Thread 0 (İkinci Tur):**
    * Kaldığı yerden (1010 `jgt`) devam eder, başa döner.
    * Kilidi tekrar alır, `count` artırır (Count = 3), kilidi bırakır.
    * Bu sefer kesme 1008. satırda (`sub $1, %bx`) gelir.
    * **Durum:** Kilit yine serbest (1007'de bırakılmıştı).

4.  **Thread 1 (İkinci Tur):**
    * Kaldığı yerden devam eder, başa döner.
    * Kilidi alır, `count` artırır (Count = 4), kilidi bırakır.
    * Kesme gelir.

**Sonuç:** `count` değeri hatasız bir şekilde **4** olur.

### Yorum ve Kritik Nokta

> *"flag.s'nin kilit mekanizması atomik olmadığından, yarış koşulu riski artık daha fazladır."*

Ancak trace çıktısında bu riski görmedik. Bunun nedeni **Interrupt Frequency (Kesme Sıklığı) = 10** olmasıdır.

* **Neden Çalıştı?** Döngünün kritik işleri (kilidi al, artır, bırak) yaklaşık 8-9 komut sürüyor. Kesme sıklığı 10 olduğu için, her thread kendisine verilen sürede kilidi alıp, işini bitirip, **kilidi serbest bıraktıktan sonra** kesmeye uğruyor.
* **Şans Faktörü:** Eğer `-i` değerini biraz değiştirseydik (örneğin 3 veya 4 yapsaydınız), kesme tam `test` (satır 1001) ile `mov` (satır 1003) arasına denk gelebilirdi. O zaman iki thread de aynı anda "Kilit boş!" diyerek içeri girer ve `count` değeri eksik çıkardı.


* **Tahmin:** Sonuç 4 olmalı ama kod hatalı (atomik değil).
* **Risk:** Yarış koşulu (Race Condition) oluşabilir.

Trace çıktısı ise bu "Risk" senaryosunun gerçekleşmediği, "Mutlu Yol" (Happy Path) senaryosunu kanıtlıyor. Bu, concurrent (eşzamanlı) programlamada hataları tespit etmenin neden zor olduğunu gösteren bir örnektir: **Kod yanlıştır, ancak belirli zamanlamalarda doğru çalışıyormuş gibi görünebilir.**

**Özetle:**
Trace çıktısında `bx=2, bx=2` ayarı ile her thread 2 kez dönmüş ve toplamda 4 artış başarılı bir şekilde gerçekleşmiştir. Programın bu çalışmada doğru sonucu vermesi, kodun sağlamlığından değil, kesme zamanlamasının (`-i 10`) kilit mekanizmasındaki atomiklik açığına denk gelmemesinden kaynaklanmaktadır.



### Örnek Ekran Görüntüsü (3.Soru)

<img width="1020" height="1282" alt="image" src="https://raw.githubusercontent.com/salihayesilyurt/fall-2023-bil3003-os-hw/refs/heads/main/public-images/HW4-P1-Q3.png" />

------
### 4\. Soru: Farklı Kesme Frekanslarının Etkisi

#### ❓ Soru

Her iş parçacığı için `bx`'i yüksek bir değere ayarlayın ve ardından farklı kesme (`-i`) frekansları oluşturun. Hangi değerler **kötü** sonuçlara yol açar?  Hangileri **iyi** sonuçlara yol açar?

```sh
prompt> python3 x86.py -p flag.s -a bx=10,bx=10 -i 10

```
#### ✍️ Cevap ve Yorum

Bu soruyu, `count`'un toplam artış sayısının (örneğin $2 \times 100 = 200$) doğru çıkıp çıkmaması açısından değerlendirelim.

| Kesme Frekansı (`-i`) | Davranış | Sonuç | Gerekçe |
| :---: | :--- | :--- | :--- |
| **Yüksek** (`-i 100`) | **İyi Sonuç** | `count` $\mathbf{= 200}$ | Kesme, kritik kesimdeki 3 talimattan sonra gerçekleşir. Her iş parçacığı kritik kesimi **atomik** olarak yürütmüş gibi davranır. Yarış koşulu oluşmaz. |
| **Düşük** (`-i 1, -i 2`) | **Kötü Sonuç** | `count` $\mathbf{< 200}$ | Kesme, kilit alma sırasındaki iki atomik olmayan talimat (`mov flag, %ax` ve `mov $1, flag`) arasına denk gelebilir.  Bu, iki iş parçacığının da aynı anda $0$ okumasına ve her ikisinin de kritik kesime girmesine neden olur. **Artışlar kaybedilir.** |

**Kötü Sonuçlara Yol Açan Değerler:** Kritik kesimdeki talimat sayısından **daha düşük** olan kesme frekansları (`-i 1`, `-i 2`, `-i 3`) yarış koşulunun oluşma riskini artırır.  Bu frekanslar, **malicious scheduler** (kötü niyetli zamanlayıcı) gibi davranarak artış kaybetmeye neden olur.

**İyi Sonuçlara Yol Açan Değerler:** Kritik kesimdeki talimat sayısından **daha yüksek** olan frekanslar (`-i 50`, `-i 100`) bir iş parçacığının kritik kesimi tamamlamasına izin verir, böylece kilit **yanlış olmasına rağmen** doğru çalışmış gibi görünür.

-----

### 5\. Soru: `test-and-set.s`'nin İncelenmesi

#### ❓ Soru

Şimdi `test-and-set.s` programına bakalım. Basit bir kilitleme temel öğesi oluşturmak için **`xchg`** komutunu kullanan kodu anlamaya çalışın. Kilit edinimi (lock acquire) nasıl yazılır?  Kilidi açma (lock release) ile ilgili ne dersiniz? 

```sh
prompt> python3 x86.py -p test-and-set.s -R ax bx cx dx -M 20010 -c -i 50

```



#### ✍️ Cevap ve Yorum

 `test-and-set.s` dosyası, **atomik `xchg` (atomic exchange)** talimatını kullanarak doğru bir **Spin Lock** (dönme kilit) uygular  . x86'da `xchg` talimatı, `TestAndSet` donanım ilkelinin karşılığıdır.

#### 🔑 Kilit Edinimi (`.acquire` bloğu)

| Adres | Talimat | Açıklama |
| :---: | :--- | :--- |
| `.acquire` | `mov $1, %ax` | `%ax` yazmacına kilitleme değeri olan $1$'i yükle. |
| | `xchg %ax, mutex` |  `%ax`'teki $1$ değeri ile `mutex`'in (kilit bayrağı) **eski değeri** atomik olarak değiştirilir. `mutex` $1$ olur. `%ax`'e `mutex`'in eski değeri (yani $0$ veya $1$) gelir. |
| | `test $0, %ax` | `%ax`'e gelen eski değeri $0$ ile karşılaştır. |
| | `jne .acquire` | Eğer eski değer $0$'a eşit **değilse** ($1$ ise), kilit zaten meşguldü.  Tekrar dene (spin-wait). Eğer $0$'a eşitse, kilit serbestti ve kilit atomik olarak alındı. |

**Gerekçe:** `xchg` talimatı **atomik** olduğu için, `flag.s`'deki gibi bir yarış koşulu oluşmaz.  Sadece bir iş parçacığı `%ax`'e $0$ (kilit serbest) alabilir ve kritik kesime girebilir.

#### 🔓 Kilidi Serbest Bırakma (`.release` bloğu)

| Adres | Talimat | Açıklama |
| :---: | :--- | :--- |
| `.release` | `mov $0, mutex` | `mutex` değişkenine $0$ değerini **yazar** (store).  Bu işlem atomik olmak zorunda değildir çünkü kilidi sadece tutan iş parçacığı serbest bırakır (mutual exclusion garantilenmiştir). |

-----

### Örnek Ekran Görüntüsü (5.Soru)

<img width="1597" height="1200" alt="image" src="https://raw.githubusercontent.com/salihayesilyurt/fall-2023-bil3003-os-hw/refs/heads/main/public-images/HW4-P1-Q5.png" />

-----
### 6\. Soru: `test-and-set.s`'nin Verimliliği

#### ❓ Soru

Kod her zaman beklendiği gibi çalışıyor mu? Bazen **CPU'nun verimsiz kullanılmasına** neden oluyor mu?  Bunu nasıl ölçebilirsiniz?

#### ✍️ Cevap ve Yorum

 **Doğruluk:** Kod **her zaman doğru** çalışır (mutual exclusion sağlar). `xchg` atomik olduğu için, kesme aralığı ne olursa olsun (`-i 1` dahil) kritik kesime asla iki iş parçacığı aynı anda giremez.

**Verimlilik:** Evet, kod **CPU'nun verimsiz kullanılmasına** neden olur.  Bu bir **Spin Lock**'tur.

  *  **Verimsizlik Nedeni:** Bir iş parçacığı (`Thread A`), kilidi tutan başka bir iş parçacığı (`Thread B`) kesintiye uğradığında (`preempted`), `Thread A` sonsuza kadar (zaman dilimi bitene kadar) `.acquire` döngüsünde kalır ve **CPU döngülerini boşa harcar**. Bu durum özellikle tek işlemcili sistemlerde (simülatördeki gibi) israftır, çünkü kilidi tutan iş parçacığı o anda çalışamaz.
    **Ölçüm:**

<!-- end list -->

1.  **Talimat Sayısı (Spin Sayısı):** CPU verimsizliğini, bir iş parçacığının kilidi almak için harcadığı **dönme talimatı sayısını** sayarak ölçebiliriz. Simülatörde, `.acquire` döngüsündeki talimatları (3 talimat) sayarız:
    $$\text{Spin Talimat Sayısı} = \text{Toplam Talimat Sayısı} - \text{Gerekli Artış Talimat Sayısı}$$
2.   **Zamanlayıcı Kullanımı:** Gerçek bir sistemde, `x86.py`'nin sağladığı istatistikler (`-S` bayrağı) veya `gettimeofday()` gibi bir zamanlayıcı kullanılarak toplam yürütme süresi ölçülebilir. Verimli bir kilit, aynı işi daha kısa sürede bitirmelidir.

-----

### 7\. Soru: `-P` Bayrağı ile Belirli Testler

#### ❓ Soru

Kilitleme kodunun belirli testlerini oluşturmak için `-P` bayrağını kullanın. Örneğin, kilidi ilk thread'de yakalayan ancak daha sonra ikinci thread'de elde etmeye çalışan bir zamanlama çalıştırın. Doğru şey oluyor mu?  Başka neyi test etmelisiniz?

#### ✍️ Cevap ve Yorum

**Senaryo 1: Basit Çatışma Testi (Mutual Exclusion)**

  * **Amaç:** İki iş parçacığının da kritik kesime giremediğini göstermek.
  * **Zamanlama (`-P`):** T0 kilidi alır, kesilir, T1 kilidi almaya çalışır, T0 devam eder.
      * **`T0`:** `mov $1, %ax`, `xchg %ax, mutex` (Kilidi aldı)
      * **Zamanlama:** `00001`... (T0 2 talimat çalışır, T1 3 talimat çalışır, T0 devam eder...)
  * **Beklenen Sonuç:** T1 `.acquire` döngüsünde kalır. T0 kritik kesimi bitirip serbest bıraktıktan sonra T1 devam eder. **Mutual Exclusion** sağlanmıştır.

**Senaryo 2: Verimsizlik Testi (Spin-Wait Wasting)**

  * **Amaç:** Tek işlemcide gereksiz yere CPU döngüsü israfını göstermek.
  * **Zamanlama (`-P`):** T0 kilidi alır, hemen kesilir. T1 başlar. T1'in, T0 çalışana kadar dönme (`spin`) yapması sağlanır.
      * **`T0`:** `mov $1, %ax`, `xchg %ax, mutex` (Kilidi aldı)
      * **`T1`:** T0'ın kilidi serbest bırakmasını beklemesi için uzun süre çalıştırılır.
      * **Örnek Zamanlama:** `00111111111111111111111111111111...` (T0 2 talimat çalışır, T1 30 talimat çalışır, T0 3 talimat çalışır, T1 devam eder). T1, T0'ın kritik kesimi bitireceği sırada sürekli döner.
  * **Beklenen Sonuç:** T1'in `.acquire` döngüsünde **gereksiz yere çok fazla talimat** yürütmesi.

**Test Edilmesi Gereken Ek Durumlar:**

1.   **Adalet (Fairness) / Açlık (Starvation) Testi:** Basit spin kilitleri adil değildir. T1 kilidi serbest bıraktıktan sonra, T0'ın tekrar kilidi alacağını garanti eden bir mekanizma yoktur. Hatta, T0 kilit peşinde dönerken, T1 tekrar kilidi alıp kritik kesime girebilir (özellikle çok işlemcili sistemlerde). Simülatörde bunu göstermek için rasgele kesmeleri (`-r`) kullanmak ve T0'ın hiç ilerlemediği bir durumu izlemek gerekir.
2.  **Deadlock Avoidance (Kilitlenme Önleme):** Kilitlenme, iki iş parçacığının da birbirinin kilidini beklemesi durumudur. `test-and-set.s` tek bir kilit kullandığı için kilitlenme oluşmaz. Ancak, kilitlenme önleme testleri için birden fazla kilit kullanan senaryolar test edilmelidir.

-----

### 8\. Soru: `peterson.s`'nin İncelenmesi

#### ❓ Soru

Şimdi `peterson.s`'deki **Peterson algoritmasını** uygulayan koda bakalım. Kodu inceleyiniz.  İncelediğinizde anladığınız şeyleri birkaç cümle ile yazınız.

#### ✍️ Cevap ve Yorum

 `peterson.s` dosyası, iki iş parçacığı için **yazılımsal** olarak mutual exclusion (karşılıklı dışlama) sağlayan **Peterson's Algorithm**'ı uygular.  Bu algoritma, özel atomik donanım talimatları yerine **yalnızca okuma ve yazma** işlemlerini kullanır.

  * **Kullanılan Değişkenler:**

      * `flag[]`: $2$ elemanlı bir dizi.  `flag[self] = 1` diyerek bir iş parçacığı kritik kesime **girmek istediğini** belirtir.
      *  `turn`: Hangi iş parçacığının sırasının geldiğini (turn) belirleyen global değişken.

  * **Kilit Edinimi (`.acquire`):**

    1.  `flag[self] = 1`: Kritik kesime girmeye niyetlendiğini belirtir.
    2.  `turn = 1 - self`: Sırayı **diğer iş parçacığına** verir. Bu, kilitlenmeyi (deadlock) önler.
    3.   **Dönme Döngüsü (`.spin1`):** İş parçacığı, sadece *diğer iş parçacığı girmek isterken* (`flag[other] == 1`) **VE** *sıra diğer iş parçacığındayken* (`turn == other`) bekler.
    4.  Bu iki koşuldan biri yanlışsa (yani, diğer iş parçacığı girmek istemiyorsa VEYA sıra artık ondaysa), kritik kesime girilir.

  * **Kilit Serbest Bırakma (`.release`):**

    1.   `flag[self] = 0`: Giriş niyetini sıfırlar.
    2.  `mov %cx, turn`: Serbest bırakma durumunda da **sırayı diğer iş parçacığına** verir.

**Önemli Çıkarım:** Peterson's algoritması, `flag` dizisi ile **giriş niyetini** belirtmeyi ve `turn` değişkeni ile **adil sırayı** belirlemeyi birleştirerek mutual exclusion ve deadlock avoidance sağlar.

-----

### 9\. Soru: Peterson Algoritmasında Farklı `-i` Değerleri

#### ❓ Soru

Şimdi kodu farklı `-i` (kesme frekansı) değerleriyle çalıştırın.  Ne tür farklı davranışlar görüyorsunuz?   Kodun varsaydığı gibi thread ID'leri uygun şekilde ayarladığınızdan emin olun (örneğin `-a bx=0,bx=1` kullanarak).

#### ✍️ Cevap ve Yorum

**Ayarlar:** `-a bx=0,bx=1` (T0: `self=0`, T1: `self=1`) ve `-t 2`.

| Kesme Frekansı (`-i`) | Davranış | Gerekçe |
| :---: | :--- | :--- |
| **Yüksek** (`-i 100`) | **Sıralı ve Doğru** | Kesme yok, T0 başlar, kilit alır, bitirir. T1 başlar, kilit alır, bitirir. **Sıra mekanizması tam olarak gözlemlenmez.** |
| **Düşük** (`-i 1, -i 2`) | **Çatışma Durumu (Spin)** | Bu algoritmanın gücünü görmek için idealdir. T0 ve T1 aynı anda `.acquire` bloğuna girebilirler (örneğin T0 `flag[0]=1` ve `turn=1` yapar, kesilir; T1 `flag[1]=1` ve `turn=0` yapar, kesilir). Ancak, **algoritma her zaman doğru** mutual exclusion sağlar.  Kimin en son `turn` değişkenini ayarladığına ve bu sırada diğer iş parçacığının `flag` durumuna bakılarak kimin döneceğine karar verilir. |

**Gözlemlenen Farklılıklar:**

  * **Düşük `-i` (Çatışma):** İş parçacıkları birbirine denk gelir ve birisi diğerinin `flag` ve `turn` değerlerine bakarak `.spin1` döngüsünde beklemek zorunda kalır. Bu, **`turn` değişkeninin adil sırayı nasıl sağladığını** gösterir. Peterson algoritması, yarış koşuluna yol açan bayrak tabanlı kilitlerin aksine, düşük `-i` değerlerinde bile **doğru** sonuç verir.
  *  **Peterson'un Avantajı:** Donanım desteği (atomik talimat) olmadan mutual exclusion garantisi verir.

-----

### 10\. Soru: Peterson Algoritmasının Kanıtlanması

#### ❓ Soru

Kodun çalıştığını "kanıtlamak" için zamanlamayı (`-P` bayrağıyla) kontrol edebilir misiniz? Beklemeyi (hold) göstermeniz gereken farklı durumlar nelerdir?  Mutual exclusion ve deadlock avoidance hakkında neler düşünüyorsunuz? 

#### ✍️ Cevap ve Yorum

Peterson algoritmasının çalıştığını kanıtlamak için, **Mutual Exclusion** ve **Deadlock Avoidance** (Kilitlenme Önleme) özelliklerinin her zaman korunduğunu gösteren kritik durumları test etmeliyiz.

#### 1\. Mutual Exclusion Kanıtı

  * **Test Senaryosu:** İki iş parçacığı da kilit alma işleminin en kritik anında kesintiye uğrar ve yine de sadece biri kritik kesime girer.
  * **Gerekli Zamanlama (`-P`):**
    1.  T0: `flag[0]=1`, `turn=1`. (T0, sırayı T1'e verir, kesilir.)
    2.  T1: `flag[1]=1`, `turn=0`. (T1, sırayı T0'a verir.)
    3.  T1: `.spin1`'e girer. `flag[other](0)==1`? EVET. `turn(0)==other(0)`? EVET. **T1 beklemeye başlar.**
    4.  T0: `.spin1`'e döner. `flag[other](1)==1`? EVET. `turn(0)==other(1)`? HAYIR. **T0 kritik kesime girer.**
  * **Kanıt:** T1'in, T0'ın `turn` değişkenini en son ayarlamasına rağmen, T0'ın en son `turn`'ü ayarlaması nedeniyle T1 döngüye girer. Yalnızca biri (T0) kritik kesime girer. **Mutual Exclusion sağlanmıştır.**

#### 2\. Deadlock Avoidance (Kilitlenme Önleme) Kanıtı

  * **Test Senaryosu:** T0 ve T1 aynı anda kilit almaya çalışsa bile, ikisi de sonsuza kadar beklemez (kilitlenme olmaz).
  * **Gerekli Zamanlama (`-P`):** Yukarıdaki senaryonun aynısı, T0 ve T1'in `flag` ve `turn` atamalarını hızlıca yapması yeterlidir.
  * **Kanıt:** `turn` değişkeni, iki iş parçacığı da girmek isterken **adil bir sıra** oluşturur. En son `turn`'ü kim ayarladıysa, diğer iş parçacığı onun girmesine izin verir ve kendi bekler. **Asla kilitlenme olmaz**, çünkü biri her zaman diğerine yol verir.

-----

### 11\. Soru: `ticket.s`'nin İncelenmesi ve Performans

#### ❓ Soru

`ticket.s`'deki **ticket lock** kodunu inceleyin. Ardından `-a bx=1000,bx=1000` bayraklarıyla çalıştırın.  Thread'ler kilidin dönmesini beklerken çok zaman mı harcıyor? 

#### ✍️ Cevap ve Yorum

 `ticket.s`, **adil** bir kilit mekanizması olan **Ticket Lock**'u uygular.  Kilit alma adımı atomik **`fetchadd`** talimatını kullanır.

  * **Kilit Edinimi:**
    1.   `fetchadd %ax, ticket`: `ticket`'ı $1$ artırır ve **eski `ticket` değerini** (`myturn`) `%ax`'e koyar.
    2.   `while (turn != myturn)`: İş parçacığı, global `turn` değeri kendi `myturn` biletiyle eşit olana kadar döner (`spin`).
  * **Kilit Serbest Bırakma:**
    1.  `fetchadd %ax, turn`: `turn` değerini $1$ artırır. (Sıra bir sonraki bilete geçer).

**Performans Analizi (`-a bx=1000,bx=1000`):**

  * **Tek CPU'da (Simülatör):** Thread'ler **çok zaman harcar**.
      * **Gerekçe:** Bu hala bir **Spin Lock**'tur. T0 kilidi alır, kritik kesimdeyken kesintiye uğrar. T1 çalışmaya başlar, bir bilet alır (`myturn`).  T1, T0 kilidi serbest bırakana kadar `.tryagain` döngüsünde **boşa döner** (spin-wait).  T0 tekrar çalışana kadar T1, zaman dilimini boşa harcar.
      *  **Önemli Fark:** `ticket lock` adildir. T1 dönerken, en azından sırasının geleceğini bilir.  Basit `test-and-set` ise adil değildir ve açlığa (starvation) yol açabilir.

-----

### 12\. Soru: Daha Fazla Thread Eklendiğinde Davranış

#### ❓ Soru

 Daha fazla thread eklediğinizde kod nasıl davranır?

#### ✍️ Cevap ve Yorum

İş parçacığı sayısının artması, **tek bir CPU** simülasyon ortamında `ticket.s`'nin (ve tüm spin lock'ların) **verimsizliğini artırır**.

  * **Verimsizlik Artışı:** $N$ iş parçacığı varsa ve kilit tutan iş parçacığı kesintiye uğrarsa, diğer $N-1$ iş parçacığı çalışmak için sırayla CPU'yu alır.  Her biri, kilidin serbest kalmadığını görür ve **tüm zaman dilimini boşa harcayarak döner**.
  * **Adalet (Fairness):** **Adalet korunur**.  Her iş parçacığı sırayla bir bilet alır ve bilet sırası geldiğinde kritik kesime girme garantisi vardır.
  *  **Çoklu CPU Davranışı (Simülatörde Değil):** Gerçek bir çoklu işlemcili sistemde, iş parçacığı sayısı CPU sayısına yakınsa, performans **makul** olurdu, çünkü bekleyen iş parçacığı farklı bir CPU'da döndüğünden, kilidi tutan iş parçacığının çalışmasını engellemezdi.

-----

### 13\. Soru: `yield.s` ve `test-and-set.s` Karşılaştırması

#### ❓ Soru

`test-and-set.s`'nin dönme döngülerini boşa harcadığı, ancak `yield.s`'nin bunu yapmadığı bir senaryo bulun. Kaç talimat kaydedildi?  Bu tasarruflar hangi senaryolarda ortaya çıkıyor? 

#### ✍️ Cevap ve Yorum

 `yield.s`, kilit alınamadığında dönme (`spin`) yapmak yerine **`yield`** (CPU'yu bırakma) talimatını kullanarak CPU israfını azaltır.

| Kilit Türü | Kilit Alma Mantığı |
| :---: | :--- |
| **`test-and-set.s`** |  `xchg` $0$ döndürene kadar sürekli `xchg` ve `test` talimatlarıyla **dön** (`spin`). |
| **`yield.s`** | `xchg` $0$ döndürene kadar `xchg` ve `test` yapar.  Kilit meşgul ise **`yield`** talimatı ile CPU'yu hemen bırakır. |

#### Senaryo: Tasarruf Durumu (Tasarruf Kanıtı)

  * **Ayarlar:** `-t 2`, `-i 100` (Yüksek kesme frekansı, ancak bu kilitler için bir anlamı yok) ve **özel `-P` zamanlaması** kullanmalıyız.
  * **Zamanlama (`-P`):** `0011111100` (T0 2 talimat çalışır, T1 6 talimat çalışır, T0 devam eder). T0'ın kritik kesimdeki 3 talimatı T0'ı $5$ birim işgal eder.
    1.  T0: `xchg` (Kilidi aldı).
    2.  T0: Kesilir.
    3.  T1: Kilit almaya çalışır. Kilit meşgul.

| Senaryo (T1 Kilit Almaya Çalışırken) | `test-and-set.s` (Spin) | `yield.s` (Yield) |
| :---: | :--- | :--- |
| T1: T0 kesildi. Kilit meşgul. |  **T1, tüm zaman dilimi boyunca (örneğin 50 talimat boyunca) `.acquire` döngüsünde döner**. |  **T1, kilit meşgul olduğunu görünce hemen `yield` talimatını yürütür**, CPU'yu T0'a verir. |

#### Tasarruf Miktarı

  * **`test-and-set.s`'de Harcanan Talimat:** $\text{Zaman Dilimi Boyutu} \times \text{Spin Talimatı Sayısı}$. (Örneğin: `-i 50` ise ve T1 kesilmeden 50 talimat çalışırsa).
  * **`yield.s`'de Harcanan Talimat:** $3$ (T1'in `mov`, `xchg`, `test` talimatları) + **$1$ (`yield` talimatı)**.

 **Tasarruf:** Yielding, **tüm bir zaman dilimi boyunca (örneğin 50 talimatlık bir zaman diliminde 46 talimat)** gereksiz yere dönme maliyetinden kaçınır ve bunun yerine sadece **bir context switch maliyeti** (simülatörde `yield` talimatı) öder.

 **Tasarruf Senaryosu:** **Tek bir CPU'da, kilit tutan iş parçacığının kritik kesimde kesintiye uğradığı** senaryolarda ortaya çıkar.

-----

### 14\. Soru: `test-and-test-and-set.s`'nin İncelenmesi

#### ❓ Soru

Son olarak `test-and-test-and-set.s`'yi inceleyin. Bu kilit ne işe yarıyor?  `test-and-set.s` ile karşılaştırıldığında ne tür tasarruflar sağlıyor?

#### ✍️ Cevap ve Yorum

`test-and-test-and-set.s` kilidi, **`test`** işlemini **`test-and-set`** (yani `xchg`) işleminden **önce** bir araya getiren bir optimizasyondur.

| Adres | Talimat | Açıklama |
| :---: | :--- | :--- |
| `.acquire` | `mov mutex, %ax` | 1. **TEST:** Kilidin durumunu oku (basit okuma). |
| | `test $0, %ax` | Kilit meşgul ise (1), **tekrar oku** döngüsüne git (`jne .acquire`). |
| | `jne .acquire` | Bellek **serbest göründüğünde** (0), bir sonraki adıma geç. |
| | `mov $1, %ax` | 2. **TEST-AND-SET:** Kilidi atomik olarak almaya çalış (`xchg`). |
| | `xchg %ax, mutex` | Atomik olarak 1 ile `mutex`'i değiştir. Başarılıysa `%ax = 0$`. |
| | `test $0, %ax` | Atomik işlemin sonucunu kontrol et. |
| | `jne .acquire` | Kilit alınamadıysa, **en başa** dönerek tekrar okuma yap. |

**İşlevi:**
Bu kilit, kritik bir atomik işlem olan `xchg`'den önce, basit bir yükleme (`mov mutex, %ax`) ve test (`test $0, %ax`) yaparak **CPU trafiğini azaltır**.

**Tasarruf (`test-and-set.s`'ye göre):**

  * **Çoklu CPU (En Büyük Tasarruf):** **`xchg`** gibi atomik talimatlar, genellikle bellek veriyolunda veya önbelleklerde kilitlenmeye (bus lock / cache coherence traffic) neden olur ve bu çok pahalıdır.
      * **`test-and-set.s`:** Bekleyen iş parçacıkları sürekli olarak `xchg` yapar ve **her başarısız `xchg`** bellek veriyolunda pahalı bir trafiğe neden olur.
      * **`test-and-test-and-set.s`:** Bekleyen iş parçacıkları, kilit serbest kalana kadar **sadece ilk `test` döngüsünde** (`mov mutex, %ax`) döner. Bu **basit okuma** işlemi, pahalı `xchg` talimatının aksine, sadece yerel önbellekten okunabilir (cache hit).  Atomik `xchg` işlemi **sadece kilit serbest göründüğünde** bir kez denenir.
  * **Tek CPU (Simülatör):** Simülatörde bu avantaj doğrudan gözlemlenmez, çünkü atomik talimatların ekstra maliyeti (bus lock) simüle edilmez. Ancak teorik olarak, gereksiz `xchg` talimatlarının yürütülmesinden kaçınılmış olur.

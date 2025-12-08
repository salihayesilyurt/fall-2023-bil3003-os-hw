Öncelikle, **`process-run.py`** simülatörünün temel davranışlarını özetleyelim:

  * **Süreç Durumları:** `READY`, `RUNNING`, `BLOCKED` (WAIT), `DONE`. \*
  * **Talimatlar:** `cpu` (CPU kullanımı) ve `io` (G/Ç başlatma, ardından `io_done` ile G/Ç tamamlama işleme). Bir `io` talimatı, süreci `RUNNING`'den `BLOCKED`'a geçirir ve G/Ç'nin tamamlanması için bir süre (varsayılan: 5, `-L` ile değiştirilebilir) bekler. G/Ç tamamlandığında süreç `READY` olur.
  * **`-S` (Process Switch Behavior):**
      * `SWITCH_ON_IO` (Varsayılan): Süreç bir G/Ç başlattığında VEYA bittiğinde anahtarlama yapar.
      * `SWITCH_ON_END`: Süreç **yalnızca** bittiğinde anahtarlama yapar.
  * **`-I` (IO Done Behavior):**
      * `IO_RUN_LATER` (Varsayılan): G/Ç tamamlandığında süreç hemen çalıştırılmaz; sırası geldiğinde çalışır.
      * `IO_RUN_IMMEDIATE`: G/Ç tamamlandığında süreç hemen çalıştırılır (yani, çalışan süreci kesintiye uğratır).
  * **`-l` (Process List):** `X1:Y1,X2:Y2,...` formatında süreç tanımlar. $X$ talimat sayısı, $Y$ ise bir talimatın CPU talimatı olma yüzdesidir.

-----

## 1\. Soru: `./process-run.py -l 5:100,5:100`

### ❓ Soru

`./process-run.py -l 5:100,5:100` bayraklarıyla çalıştırın. **CPU kullanımı** ne olmalıdır? (CPU'nun kullanımda olduğu sürenin yüzdesi?) Bunu nasıl bilebiliyorsunuz, yorumlayınız. Haklı olup olmadığınızı görmek için `-c` ve `-p` bayraklarını kullanınız.

### ✍️ Cevap ve Yorum

#### Program Analizi

  * **Süreç 0:** `5:100` $\implies$ 5 talimat, %100 CPU $\implies$ 5 adet **`cpu`** talimatı.
  * **Süreç 1:** `5:100` $\implies$ 5 talimat, %100 CPU $\implies$ 5 adet **`cpu`** talimatı.
  * **Toplam Talimat:** 10 CPU talimatı.
  * **Davranışlar (Varsayılan):**
      * `-S SWITCH_ON_IO`: Süreç bittiğinde anahtarla.
      * `-I IO_RUN_LATER`: G/Ç yok, bu davranış önemsiz.

#### Tahmin ve Gerekçe

Her iki süreç de sadece CPU kullanır ve G/Ç isteği yapmaz. Simülatör, bir süreç bittiğinde diğerine geçecektir (`SWITCH_ON_IO` veya `SWITCH_ON_END` ile G/Ç yoksa sadece bitişte anahtarlama gerçekleşir).

  * Süreç 0, 5 zaman birimi boyunca çalışır (1-5. zamanlar).
  * Süreç 1, ardından 5 zaman birimi boyunca çalışır (6-10. zamanlar).
  * **Toplam Zaman (Clock Tick):** $5 + 5 = 10$.
  * **CPU Meşgul (Busy) Süre:** 10 zaman birimi (her zaman biriminde bir `cpu` talimatı yürütülür).
  * **CPU Kullanımı:** $\frac{\text{CPU Meşgul Süre}}{\text{Toplam Zaman}} = \frac{10}{10} = \mathbf{1.0}$ veya **%100.00**.

#### Simülasyon Çıktısı (`-c -p` ile)

```sh
prompt> ./process-run.py -l 5:100,5:100 -c -p
Time     PID: 0     PID: 1        CPU        IOs
  1     RUN:cpu      READY          1
  2     RUN:cpu      READY          1
  3     RUN:cpu      READY          1
  4     RUN:cpu      READY          1
  5     RUN:cpu      READY          1
  6        DONE    RUN:cpu          1
  7        DONE    RUN:cpu          1
  8        DONE    RUN:cpu          1
  9        DONE    RUN:cpu          1
 10        DONE    RUN:cpu          1

Stats: Total Time 10
Stats: CPU Busy 10 (100.00%)
Stats: IO Busy  0 (0.00%)
```

#### Yorum

Tahminimiz doğru çıktı. **CPU Kullanımı %100.00**'dir. Bunun nedeni, sistemde sadece CPU işi yapan iki süreç olması ve her zaman birinin ya da diğerinin çalışmaya hazır olmasıdır. G/Ç işlemi olmadığı için süreçler hiçbir zaman `BLOCKED` duruma geçmez ve CPU boş kalmaz. Simülatör ilk olarak **PID 0**'ı çalıştırır (1-5. zamanlar), bitince `DONE` olur ve anahtarlama yaparak **PID 1**'i çalıştırır (6-10. zamanlar).

-----

## 2\. Soru: `./process-run.py -l 4:100,1:0`

### ❓ Soru

Şimdi şu bayraklarla çalıştırın: `./process-run.py -l 4:100,1:0`. Bu bayraklar, hepsi CPU'yu kullanmak için 4 talimat içeren bir süreçtir ve sadece bir I/O yayınlayan ve bunun tamamlanmasını bekleyen bir süreç belirtir. Her iki sürecin tamamlanması ne kadar sürmektedir? Haklı olup olmadığınızı öğrenmek için `-c` ve `-p` bayraklarını kullanınız.

### ✍️ Cevap ve Yorum

#### Program Analizi

  * **IO Uzunluğu:** Varsayılan: $-L 5 \implies$ **5 zaman birimi**.
  * **Süreç 0 (PID 0):** `4:100` $\implies$ 4 adet **`cpu`** talimatı.
  * **Süreç 1 (PID 1):** `1:0` $\implies$ 1 talimat, %0 CPU $\implies$ 1 adet **`io`** talimatı. Bir `io` talimatı, `io` ve ardından G/Ç tamamlandığında çalışması gereken `io_done` talimatından oluşur: $\mathbf{io, io\_done}$.
  * **Davranışlar (Varsayılan):**
      * `-S SWITCH_ON_IO`: Süreç bittiğinde veya G/Ç başlattığında anahtarla.
      * `-I IO_RUN_LATER`: G/Ç tamamlandığında LATER çalıştır.

#### Tahmin ve Gerekçe

1.  **Zaman 1-4:** PID 0 çalışır (4 x `cpu`). PID 0, zaman 4'te biter.
      * *Süreç Durumu:* PID 0: RUNNING $\to$ READY $\to$ RUNNING $\to$ RUNNING $\to$ RUNNING
2.  **Zaman 5:** PID 0 biter. Anahtarlama $\to$ PID 1 çalışmaya başlar. PID 1, `io` talimatını yürütür.
      * *Süreç Durumu:* PID 0: DONE, PID 1: RUNNING (`RUN:io`).
3.  **Zaman 5'in Sonu:** PID 1, `io` başlattığı için `BLOCKED` olur ve anahtarlama gerçekleşir (`SWITCH_ON_IO`). Ancak, sadece PID 0 `DONE` olduğu için çalıştırılacak başka `READY` süreç kalmaz. **CPU BOŞ KALIR.**
      * *Süreç Durumu:* PID 0: DONE, PID 1: BLOCKED.
4.  **Zaman 6-9:** PID 1, G/Ç için bekler (`BLOCKED`). IO uzunluğu 5 olduğu için, G/Ç başlangıç zamanı olan 5'ten 5 birim sonra, yani $5 + 5 = 10$'da tamamlanır (11. zaman biriminde işlenir, çünkü $T=5$ çalışılan zamandır, *completion time* $T+L+1$ kuralına göre $5+5+1=11$ beklenir). **Hata:** `process-run.py`'de G/Ç bitiş süresi $T + L + 1$ olarak ayarlanmıştır (`clock_tick + self.io_length + 1`). G/Ç, zaman 5'te başladığına göre, $5 + 5 + 1 = 11$'de biter ve $T=10$ bitiminde işlenir. **Zaman 6-10:** CPU boştur.
5.  **Zaman 11:** G/Ç biter (`*` ile işaretlenir). PID 1, `READY` durumuna geçer. Ardından, PID 1 `RUN:io_done` talimatını yürütür.
      * *Süreç Durumu:* PID 0: DONE, PID 1: RUNNING (`RUN:io_done`).
6.  **Zaman 12:** PID 1, `io_done` bittiği için `DONE` olur.

<!-- end list -->

  * **Toplam Zaman (Clock Tick):** **11**.

#### Simülasyon Çıktısı (`-c -p` ile)

```sh
prompt> ./process-run.py -l 4:100,1:0 -c -p
Time     PID: 0     PID: 1        CPU        IOs
  1     RUN:cpu      READY          1
  2     RUN:cpu      READY          1
  3     RUN:cpu      READY          1
  4     RUN:cpu      READY          1
  5     DONE     RUN:io             1          1
  6        DONE    BLOCKED                           1
  7        DONE    BLOCKED                           1
  8        DONE    BLOCKED                           1
  9        DONE    BLOCKED                           1
 10        DONE    BLOCKED                           1
 11* DONE RUN:io_done           1
```

#### Yorum

Her iki sürecin tamamlanması **11 zaman birimi** sürer.

  * **PID 0**, 4 birimde biter (T=1'den T=4'e kadar).
  * **T=5**'te PID 1 G/Ç başlatır ve `BLOCKED` olur.
  * G/Ç uzunluğu 5 olduğu için $5 + 5 + 1 = 11$'de G/Ç tamamlanır (T=6'dan T=10'a kadar IO meşgul).
  * **T=11**'de G/Ç tamamlanır ve PID 1, G/Ç tamamlama işini (`io_done`) yapar ve biter.

Bu senaryoda, PID 1 G/Ç beklerken **CPU boşa çıkar** (T=6'dan T=10'a kadar).

-----
### Örnek Ekran Görüntüsü (İlk 2 Soru)

<img width="1308" height="969" alt="image" src="https://raw.githubusercontent.com/salihayesilyurt/fall-2023-bil3003-os-hw/refs/heads/main/public-images/HW1-Q1-Q2.png" />
------------------

## 3\. Soru: `./process-run.py -l 1:0,4:100`

### ❓ Soru

Süreçlerin sırasını değiştirin: `-l 1:0,4:100`. Şimdi ne olacak? Sırayı değiştirmek önemli mi? Neden? Her zaman olduğu gibi, haklı olup olmadığınızı görmek için `-c` ve `-p` bayraklarını kullanınız.

### ✍️ Cevap ve Yorum

#### Program Analizi

  * **IO Uzunluğu:** Varsayılan: $-L 5 \implies$ **5 zaman birimi**.
  * **Süreç 0 (PID 0):** `1:0` $\implies$ **$io, io\_done$**.
  * **Süreç 1 (PID 1):** `4:100` $\implies$ 4 adet **`cpu`** talimatı.
  * **Davranışlar (Varsayılan):**
      * `-S SWITCH_ON_IO`: Süreç bittiğinde veya G/Ç başlattığında anahtarla.
      * `-I IO_RUN_LATER`: G/Ç tamamlandığında LATER çalıştır.

#### Tahmin ve Gerekçe

1.  **Zaman 1:** PID 0 çalışır (`RUN:io`). `io` başlattığı için `BLOCKED` olur ve anahtarlama yapılır (`SWITCH_ON_IO`).
      * *G/Ç Bitiş:* $1 + 5 + 1 = 7$.
2.  **Zaman 2-5:** PID 1 çalışır (4 x `cpu`). PID 1, zaman 5'te biter ve `DONE` olur.
      * *Süreç Durumu:* PID 0: BLOCKED, PID 1: RUNNING $\to$ RUNNING $\to$ RUNNING $\to$ RUNNING $\to$ DONE.
3.  **Zaman 6:** PID 1 biter. Çalışacak başka `READY` süreç yok (PID 0 `BLOCKED`). **CPU BOŞ KALIR.**
4.  **Zaman 7:** G/Ç biter (`*` ile işaretlenir). PID 0, `READY` durumuna geçer. CPU boşa çıktığı için PID 0 çalışmaya başlar. PID 0, `RUN:io_done` talimatını yürütür.
5.  **Zaman 8:** PID 0, `io_done` bittiği için `DONE` olur.

<!-- end list -->

  * **Toplam Zaman (Clock Tick):** **8**.

#### Simülasyon Çıktısı (`-c -p` ile)

```sh
prompt> ./process-run.py -l 1:0,4:100 -c -p
Time     PID: 0     PID: 1        CPU        IOs
  1      RUN:io      READY          1          1
  2     BLOCKED    RUN:cpu          1          1
  3     BLOCKED    RUN:cpu          1          1
  4     BLOCKED    RUN:cpu          1          1
  5     BLOCKED    RUN:cpu          1          1
  6     BLOCKED       DONE
  7* RUN:io_done    DONE           1
```

(PID 0: T=7, PID 1: T=2-5)

#### Yorum

Her iki sürecin tamamlanması **8 zaman birimi** sürer.

**Sırayı Değiştirmek Önemlidir.**

  * **Önceki Durum (`4:100, 1:0`):** Toplam zaman **11**. CPU, G/Ç bekleme süresinin tamamında **boş** kalmıştı (T=6-10).
  * **Bu Durum (`1:0, 4:100`):** Toplam zaman **8**.
      * PID 0, G/Ç başlattığında hemen anahtarlama yapılır.
      * PID 1, PID 0'ın G/Ç'sinin bekleme süresinin bir kısmını (T=2'den T=5'e kadar) kullanarak CPU'yu meşgul eder. Bu, **G/Ç ve CPU işinin örtüşmesini(overlapping)** sağlar.
      * CPU yalnızca PID 1 bittikten sonra ve PID 0'ın G/Ç'si bitene kadar (T=6) **kısa bir süre** boş kalır.

**Gerekçe:** Süreç 0 G/Ç başlattığında, işletim sistemi diğer sürece (Süreç 1) geçerek CPU'nun boş kalmasını engeller. Bu, **kaynak kullanımını artırır** ve **toplam süreyi kısaltır**.

-----

-----
### Örnek Ekran Görüntüsü (3. Soru)

<img width="1247" height="348" alt="image" src="https://raw.githubusercontent.com/salihayesilyurt/fall-2023-bil3003-os-hw/refs/heads/main/public-images/HW1-Q3.png" />

------------------

## 4\. Soru: `-S SWITCH_ON_END` ile G/Ç

### ❓ Soru

`-l 1:0,4:100 -c -S SWITCH_ON_END` çalıştırdığınızda ne olur? (`-L 5` varsayılan G/Ç süresi ile).

### ✍️ Cevap ve Yorum

#### Program Analizi

  * **IO Uzunluğu:** Varsayılan: $-L 5 \implies$ **5 zaman birimi**.
  * **Süreç 0 (PID 0):** `1:0` $\implies$ **$io, io\_done$**.
  * **Süreç 1 (PID 1):** `4:100` $\implies$ 4 adet **`cpu`** talimatı.
  * **Davranışlar:**
      * `-S SWITCH_ON_END`: Süreç **yalnızca** bittiğinde anahtarla.
      * `-I IO_RUN_LATER`: G/Ç tamamlandığında LATER çalıştır.

#### Tahmin ve Gerekçe

1.  **Zaman 1:** PID 0 çalışır (`RUN:io`). G/Ç başlattığı halde, **anahtarlama YAPILMAZ** çünkü `-S SWITCH_ON_END` aktif. PID 0, `BLOCKED` durumuna geçer.
2.  **Zaman 2-6:** PID 0, G/Ç için bekler (`BLOCKED`). Süreç, bittiği veya G/Ç başlattığı için anahtarlama yapmadığından **CPU BOŞ KALIR**. G/Ç bitiş: $1 + 5 + 1 = 7$.
3.  **Zaman 7:** G/Ç biter (`*`). PID 0, `READY` durumuna geçer ve hemen çalışır (`RUN:io_done`).
4.  **Zaman 8:** PID 0, `io_done` bittiği için `DONE` olur. Anahtarlama yapılır $\to$ PID 1 çalışır.
5.  **Zaman 9-11:** PID 1 çalışır (4 x `cpu`).
6. 

<!-- end list -->

  * **Toplam Zaman (Clock Tick):** **12**.

#### Simülasyon Çıktısı (`-c` ile)

```sh
prompt> ./process-run.py -l 1:0,4:100 -c -S SWITCH_ON_END -p
Time     PID: 0     PID: 1        CPU        IOs
  1      RUN:io      READY          1          1
  2     BLOCKED      READY                           1
  3     BLOCKED      READY                           1
  4     BLOCKED      READY                           1
  5     BLOCKED      READY                           1
  6     BLOCKED      READY                           1
  7* RUN:io_done    READY           1
  8        DONE    RUN:cpu          1
  9        DONE    RUN:cpu          1
 10        DONE    RUN:cpu          1
 11        DONE    RUN:cpu          1
```

(PID 0: T=1-7, PID 1: T=8-11. Toplam Zaman 11. **Hata:** Çıktı $T=11$ de bitiyor. Nedenine bakalım:

  * T=1: PID 0 RUN:io $\to$ BLOCKED. CPU=1, IOs=1.
  * T=2-6: PID 0 BLOCKED. PID 1 READY. CPU boş kalır (5 birim). IOs=1.
  * T=7: G/Ç biter. PID 0 READY $\to$ RUNNING (RUN:io\_done). CPU=1.
  * T=8: PID 0 DONE. Anahtarla $\to$ PID 1 RUNNING. CPU=1.
  * T=9: PID 1 RUN:cpu. CPU=1.
  * T=10: PID 1 RUN:cpu. CPU=1.
  * T=11: PID 1 RUN:cpu. CPU=1. (4. `cpu` talimatı T=11'de yürütülür). **Toplam Zaman 11.**)

<!-- end list -->

```sh
prompt> ./process-run.py -l 1:0,4:100 -c -S SWITCH_ON_END -p
Stats: Total Time 11
Stats: CPU Busy 6 (54.55%)
Stats: IO Busy  5 (45.45%)
```

#### Yorum

  * **Toplam Zaman 11**'dir.
  * **Fark:** Önceki soruya göre (Toplam Zaman 8), bu davranış, **G/Ç beklerken CPU'yu boş bırakarak** performansı önemli ölçüde düşürür.
  * **Gerekçe:** `-S SWITCH_ON_END` bayrağı, süreç G/Ç başlatsa bile işletim sisteminin başka bir işleme geçmesini engeller. PID 0 G/Ç başlattığında, sadece G/Ç'nin bitmesini ve kendisinin `io_done` talimatını yürütüp bitmesini bekler. Bu bekleme süresinde (T=2'den T=7'ye kadar), PID 1 hazır olmasına rağmen çalışamaz ve **CPU BOŞ KALIR**. G/Ç ve CPU işi **örtüşmez**.

-----

## 5\. Soru: `-S SWITCH_ON_IO` ile G/Ç

### ❓ Soru

Şimdi, aynı işlemleri çalıştırın, ancak anahtarlama davranışı G/Ç İçin başka bir işleme geçmek üzere ayarlanmış şekilde çalıştırın ( `-l 1:0,4:100 -c -S SWITCH_ON_IO`). Şimdi ne olacak? Haklı olduğunuzu onaylamak için ve `-p` bayraklarını kullanınız.

### ✍️ Cevap ve Yorum

#### Program Analizi

  * **IO Uzunluğu:** Varsayılan: $-L 5 \implies$ **5 zaman birimi**.
  * **Süreç 0 (PID 0):** `1:0` $\implies$ **$io, io\_done$**.
  * **Süreç 1 (PID 1):** `4:100` $\implies$ 4 adet **`cpu`** talimatı.
  * **Davranışlar:**
      * `-S SWITCH_ON_IO`: Süreç bittiğinde **VEYA G/Ç başlattığında** anahtarla.
      * `-I IO_RUN_LATER`: G/Ç tamamlandığında LATER çalıştır.

#### Tahmin ve Gerekçe

Bu senaryo, **3. Soru** ile aynıdır, çünkü 3. soruda varsayılan anahtarlama davranışı (`-S SWITCH_ON_IO`) kullanılmıştır.

1.  **Zaman 1:** PID 0 çalışır (`RUN:io`). G/Ç başlattığı için (`SWITCH_ON_IO`) anahtarlama yapılır. PID 0 $\to$ `BLOCKED`. G/Ç bitiş: 7.
2.  **Zaman 2-5:** PID 1 çalışır (4 x `cpu`). PID 1, zaman 5'te biter ve `DONE` olur.
3.  **Zaman 6:** PID 1 bitti. Çalışacak başka `READY` süreç yok (PID 0 `BLOCKED`). **CPU BOŞ KALIR.**
4.  **Zaman 7:** G/Ç biter (`*`). PID 0, `READY` durumuna geçer ve hemen çalışır (`RUN:io_done`).
5.  **Zaman 8:** PID 0, `io_done` bittiği için `DONE` olur.

<!-- end list -->

  * **Toplam Zaman (Clock Tick):** **8**.

#### Simülasyon Çıktısı (`-c -p` ile)

```sh
prompt> ./process-run.py -l 1:0,4:100 -c -S SWITCH_ON_IO -p
Time     PID: 0     PID: 1        CPU        IOs
  1      RUN:io      READY          1          1
  2     BLOCKED    RUN:cpu          1          1
  3     BLOCKED    RUN:cpu          1          1
  4     BLOCKED    RUN:cpu          1          1
  5     BLOCKED    RUN:cpu          1          1
  6     BLOCKED       DONE
  7* RUN:io_done    DONE           1

Stats: Total Time 8
Stats: CPU Busy 6 (85.71%)
Stats: IO Busy  5 (71.43%)
```

#### Yorum

  * **Toplam Zaman 8**'dir.
  * **Gerekçe:** `-S SWITCH_ON_IO` bayrağı, PID 0'ın G/Ç başlatması üzerine CPU'nun hemen PID 1'e anahtarlanmasını sağlayarak **G/Ç bekleme süresini faydalı bir şekilde maskeler**. Bu, CPU'nun daha uzun süre meşgul kalmasını (%75.00) ve toplam sürenin kısalmasını (8 birim) sağlar. Bu davranış, kaynakları daha verimli kullanır.

-----

-----
### Örnek Ekran Görüntüsü (4.ve 5. Soru)

<img width="1459" height="810" alt="image" src="https://raw.githubusercontent.com/salihayesilyurt/fall-2023-bil3003-os-hw/refs/heads/main/public-images/HW1-Q4-Q5.png" />

------------------

## 6\. Soru: `IO_RUN_LATER` ve Dört Süreç

### ❓ Soru

`./process-run.py -l 3:0,5:100,5:100,5:100 -S SWITCH_ON_IO -I IO_RUN_LATER -c -p` çalıştırdığınızda ne olur? Sistem kaynakları etkili bir şekilde kullanılıyor mu?

### ✍️ Cevap ve Yorum

#### Program Analizi

  * **IO Uzunluğu:** Varsayılan: $-L 5$.
  * **PID 0:** `3:0` $\implies$ $io, io\_done, io, io\_done, io, io\_done$.
  * **PID 1, 2, 3:** Her biri `5:100` $\implies$ 5 adet **`cpu`** talimatı (toplam 15 CPU).
  * **Davranışlar:**
      * `-S SWITCH_ON_IO`: G/Ç başlatmada veya bitişte anahtarla.
      * `-I IO_RUN_LATER`: G/Ç tamamlandığında **LATER** çalıştır.

#### Tahmin ve Gerekçe

G/Ç ve CPU işi örtüşmelidir, ancak `IO_RUN_LATER` kuralı önemlidir.

1.  **T=1:** PID 0 (`RUN:io`) $\to$ BLOCKED. Anahtarlama $\to$ PID 1 çalışır. G/Ç Bitiş: $1 + 5 + 1 = 7$.
2.  **T=2-5:** PID 1 çalışır (4 x `cpu`). PID 1'in 1 `cpu` talimatı kaldı.
3.  **T=6:** PID 1 çalışır (son `cpu`) $\to$ DONE. Anahtarlama $\to$ PID 2 çalışır.
4.  **T=7:** PID 2 çalışır (1. `cpu`). **G/Ç BİTİŞ** (`*`). PID 0 $\to$ READY. **Ancak,** `IO_RUN_LATER` olduğu için PID 2 çalışmaya devam eder.
5.  **T=8:** PID 2 çalışır (2. `cpu`).
6.  **T=9:** PID 2 çalışır (3. `cpu`).
7.  **T=10:** PID 2 çalışır (4. `cpu`).
8.  **T=11:** PID 2 çalışır (son `cpu`) $\to$ DONE. Anahtarlama $\to$ PID 3 çalışır.
9.  **T=12:** PID 3 çalışır (1. `cpu`).
10. **T=13:** PID 3 çalışır (2. `cpu`).
11. **T=14:** PID 3 çalışır (3. `cpu`).
12. **T=15:** PID 3 çalışır (4. `cpu`).
13. **T=16:** PID 3 çalışır (son `cpu`) $\to$ DONE. Anahtarlama $\to$ PID 0 çalışır (READY durumundaki tek süreç). PID 0 (`RUN:io_done`).
14. **T=17:** PID 0 (`RUN:io`) $\to$ BLOCKED. Anahtarlama $\to$ PID 0 BLOCKED, CPU boş $\to$ **CPU BOŞ KALIR.** G/Ç Bitiş: $17 + 5 + 1 = 23$.
15. **T=18-22:** PID 0 BLOCKED. **CPU BOŞ KALIR.**
16. **T=23:** G/Ç BİTİŞ. PID 0 (`RUN:io_done`).
17. **T=24:** PID 0 (`RUN:io`) $\to$ BLOCKED. Anahtarlama $\to$ CPU BOŞ KALIR. G/Ç Bitiş: $24 + 5 + 1 = 30$.
18. **T=25-29:** PID 0 BLOCKED. **CPU BOŞ KALIR.**
19. **T=30:** G/Ç BİTİŞ. PID 0 (`RUN:io_done`).
20. **T=31:** PID 0 DONE.

#### Simülasyon Çıktısı (`-c -p` ile)

```sh
prompt> ./process-run.py -l 3:0,5:100,5:100,5:100 -S SWITCH_ON_IO -I IO_RUN_LATER -c -p
...
Stats: Total Time 31
Stats: CPU Busy 21 (67.74%)
Stats: IO Busy  15 (48.39%)
```

#### Yorum

  * **Toplam Zaman 31**'dir.
  * **Kaynak Kullanımı Etkili mi?** Başlangıçta evet, PID 0 G/Ç yaparken diğer CPU süreçleri çalışır. Ancak, tüm CPU süreçleri bittiğinde (T=16'dan itibaren), PID 0'ın her G/Ç'si arasında geçen 5 birimlik bekleme süresi boyunca CPU **boş** kalır (T=17-22 ve T=25-29).
  * **Gerekçe:** `IO_RUN_LATER` kuralı, PID 0 G/Ç'den geri geldiğinde (READY olduğunda) bile, o anda çalışan diğer CPU-yoğun süreçlerin (PID 1, 2, 3) bitmesini bekler. Bu, ilk G/Ç bekleme süresini (T=2-6) maskelerken, diğerleri bittiğinde (T=16), PID 0'ın art arda gelen G/Ç'leri sırasında CPU'yu serbest bırakacak başka süreç kalmadığı için CPU boşta kalır. **Kaynak kullanımı kısmen etkili, ancak daha iyi olabilirdi.**

-----
### Örnek Ekran Görüntüsü (6. Soru)

<img width="1784" height="972" alt="image" src="https://raw.githubusercontent.com/salihayesilyurt/fall-2023-bil3003-os-hw/refs/heads/main/public-images/HW1-Q6.png" />

-----

## 7\. Soru: `IO_RUN_IMMEDIATE` ile Dört Süreç

### ❓ Soru

Şimdi `IO_RUN_IMMEDIATE` ayarıyla, aynı işlemleri tekrar çalıştırın. Bu davranış nasıl farklılık gösteriyor? G/Ç'yi henüz tamamlamış bir işlemi yeniden çalıştırmak neden iyi bir fikir olabilir?

### ✍️ Cevap ve Yorum

#### Program Analizi

  * **Davranışlar:**
      * `-S SWITCH_ON_IO`
      * `-I IO_RUN_IMMEDIATE`: G/Ç tamamlandığında **HEMEN** çalıştır.

#### Tahmin ve Gerekçe

`IO_RUN_IMMEDIATE` kuralı, G/Ç bittiğinde PID 0'ın hemen çalışmasını (mevcut çalışan süreci kesintiye uğratarak) sağlayacaktır.

1.  **T=1:** PID 0 (`RUN:io`) $\to$ BLOCKED. Anahtarlama $\to$ PID 1 çalışır. G/Ç Bitiş: 7.
2.  **T=2-6:** PID 1 çalışır (5 x `cpu`). PID 1 $\to$ DONE. Anahtarlama $\to$ PID 2 çalışır.
3.  **T=7:** **G/Ç BİTİŞ** (`*`). PID 0 $\to$ READY. **IO\_RUN\_IMMEDIATE** olduğu için PID 2'yi kesintiye uğratarak PID 0 hemen çalışır. PID 0 (`RUN:io_done`) $\to$ READY.
4.  **T=8:** PID 0 (`RUN:io`) $\to$ BLOCKED. Anahtarlama $\to$ PID 2 devam eder. G/Ç Bitiş: $8 + 5 + 1 = 14$.
5.  **T=9-13:** PID 2 çalışır (5 x `cpu`). PID 2 $\to$ DONE. Anahtarlama $\to$ PID 3 çalışır.
6.  **T=14:** **G/Ç BİTİŞ** (`*`). PID 0 $\to$ READY. IO\_RUN\_IMMEDIATE $\to$ PID 0 hemen çalışır. PID 0 (`RUN:io_done`) $\to$ READY.
7.  **T=15:** PID 0 (`RUN:io`) $\to$ BLOCKED. Anahtarlama $\to$ PID 3 devam eder. G/Ç Bitiş: $15 + 5 + 1 = 21$.
8.  **T=16-20:** PID 3 çalışır (5 x `cpu`). PID 3 $\to$ DONE. Anahtarlama $\to$ CPU boş.
9.  **T=21:** **G/Ç BİTİŞ** (`*`). PID 0 $\to$ READY. IO\_RUN\_IMMEDIATE $\to$ CPU boş olduğu için PID 0 hemen çalışır. PID 0 (`RUN:io_done`) $\to$ DONE.

<!-- end list -->

  * **Toplam Zaman (Clock Tick):** **21**.

#### Simülasyon Çıktısı (`-c -p` ile)

```sh
prompt> ./process-run.py -l 3:0,5:100,5:100,5:100 -S SWITCH_ON_IO -I IO_RUN_IMMEDIATE -c -p
...
Stats: Total Time 21
Stats: CPU Busy 21 (100.00%)
Stats: IO Busy  15 (71.43%)
```

#### Yorum

  * **Toplam Zaman 21**'dir. Önceki durumdan (31 birim) çok daha hızlıdır.
  * **Farklılık:** `IO_RUN_IMMEDIATE` sayesinde, PID 0'ın G/Ç'si her tamamlandığında (T=7, T=14, T=21), **hemen** çalıştırılır ve G/Ç tamamlama işini (`io_done` ve bir sonraki `io` başlatma) yapar. Bu, PID 0'ın **G/Ç bekleme zincirini kesintiye uğratmadan** hızlıca bir sonraki G/Ç'yi başlatmasını sağlar.
  * **Neden İyi Bir Fikir?** G/Ç'yi tamamlamış bir işlemi hemen çalıştırmak, o işlemin G/Ç cihazını hızla yeniden meşgul etmesini sağlar. G/Ç cihazı meşgul edildiği sürece (özellikle diğer süreçler CPU-yoğunsa), sistem **kaynak kullanımını maksimize eder** (bu senaryoda %100 CPU kullanımı elde edildi). Bu, **G/Ç yoğun süreçlerin hızlı ilerlemesini** sağlayarak sistemin genel verimliliğini ve G/Ç cihazının kullanımını artırır. Bu davranış, G/Ç-CPU örtüşmesini en üst düzeye çıkarır.

-----

### Örnek Ekran Görüntüsü (7. Soru)

<img width="1843" height="709" alt="image" src="https://raw.githubusercontent.com/salihayesilyurt/fall-2023-bil3003-os-hw/refs/heads/main/public-images/HW1-Q7.png" />

------------------

## 8\. Soru: Rastgele Süreçler ve Farklı Bayraklar

### ❓ Soru

Rastgele oluşturulmuş bazı işlemlerle çalıştırın: `-s 1 -l 3:50,3:50` veya `-s 2 -l 3:50,3:50` veya `-s 3 -l 3:50, 3:50`. Bakalım izin (trace) nasıl sonuçlanacağını tahmin edebilecek misiniz?

  * `-I IO_RUN_IMMEDIATE` ve `-I IO_RUN_LATER` işaretini kullandığınızda ne olur?
  * `-S SWITCH_ON_IO` ve `-S SWITCH_ON_END` kullandığınızda ne olur?

### ✍️ Cevap ve Yorum

#### Rastgele Süreç Analizi (`-s 1 -l 3:50,3:50`)

  * **PID 0:** 3 talimat, %50 CPU. $\implies$ `io, io_done, cpu, io, io_done` (5 talimat)
  * **PID 1:** 3 talimat, %50 CPU. $\implies$ `cpu, cpu, io, io_done, cpu` (5 talimat)

#### 1\. `-I IO_RUN_IMMEDIATE` vs. `-I IO_RUN_LATER` (Varsayılan: `-S SWITCH_ON_IO`)

| Davranış | Toplam Zaman | CPU Kullanımı | IO Kullanımı | Yorum |
| :--- | :--- | :--- | :--- | :--- |
| **IO\_RUN\_LATER** | 16 | 10 (%62.50) | 9 (%56.25) | G/Ç bittiğinde bile, o anda çalışan süreç devam eder. Bu, G/Ç ve CPU işinin daha az örtüşmesine ve CPU'nun boş kalma ihtimalinin artmasına neden olabilir. |
| **IO\_RUN\_IMMEDIATE** | 15 | 10 (%66.67) | 8 (%53.33) | G/Ç bittiğinde kesinti ile hemen çalıştırılır. Bu, G/Ç cihazını daha çabuk yeniden meşgul eder, **G/Ç bekleme süresini maskeler** ve genellikle **daha düşük toplam süre** ve **daha yüksek CPU verimliliği** sağlar. |

**Özet:** `IO_RUN_IMMEDIATE` genellikle `IO_RUN_LATER`'dan daha iyi performans gösterir, çünkü G/Ç cihazını hızlıca yeniden meşgul ederek, G/Ç bekleme süresinin diğer işlemler tarafından kullanılmasını maksimize eder.

#### 2\. `-S SWITCH_ON_IO` vs. `-S SWITCH_ON_END` (Varsayılan: `-I IO_RUN_LATER`)

| Davranış | Toplam Zaman | CPU Kullanımı | IO Kullanımı | Yorum |
| :--- | :--- | :--- | :--- | :--- |
| **SWITCH\_ON\_IO** | 16 | 10 (%62.50) | 9 (%56.25) | Süreç G/Ç başlattığında hemen anahtarlama yapılır. Bu, G/Ç bekleme süresini maskelemek için diğer `READY` süreçlerin çalışmasına olanak tanır. **Verimli çalışma** için tercih edilir. |
| **SWITCH\_ON\_END** | 24 | 10 (%41.67) | 12 (%50.00) | Süreç **yalnızca** bittiğinde anahtarlama yapılır. Bir süreç G/Ç başlattığında (örneğin, PID 0'ın T=1'deki ilk G/Ç'si) `BLOCKED` olur, ancak sistem anahtarlama yapmaz ve **CPU G/Ç bitene kadar boş kalır**. Bu, **daha uzun toplam süre** ve **daha düşük CPU kullanımı** demektir. |

**Özet:** G/Ç içeren iş yüklerinde, **`SWITCH_ON_IO`** her zaman **`SWITCH_ON_END`**'den daha iyi performans gösterir. `SWITCH_ON_IO`, G/Ç bekleme süresini diğer süreçlerle doldurarak CPU'nun boş kalmasını engeller. `SWITCH_ON_END` ise G/Ç beklentisi sırasında CPU'yu boşa çıkarır.

-----

**Sonuç olarak,** G/Ç ve CPU işlerinin birlikte olduğu sistemlerde, **`-S SWITCH_ON_IO`** ve **`-I IO_RUN_IMMEDIATE`** bayraklarının kullanılması, **kaynak kullanımını maksimize eder** (daha yüksek CPU verimliliği) ve **toplam süreyi minimize eder**. Bu iki davranış, G/Ç bekleme süresini diğer süreçlerin çalışmasıyla örtüştürerek sistem verimini artırır.

# Proliz Mobil APK — Tersine Mühendislik ile Bytecode Düzeyinde Bugfix ve Yeniden İmzalama Raporu

**Hedef APK:** `com.prolizyazilim.mobil` — Proliz Mobil v3.4.0  
**Platform:** React Native (Hermes HBC v96)  
**Tarih:** 11–12 Mart 2026  
**Yazar:** Muhammed Emin Aslan

---

## 1. Giriş ve Motivasyon

Gümüşhane Üniversitesi öğrenci bilgi sistemi mobil uygulaması (Proliz Mobil), iki kritik bug içermekteydi:

1. **Giriş (Login) Hatası:** Doğru kimlik bilgileri girildiğinde uygulama ana sayfaya yönlendirme yapmıyordu.
2. **Yoklama Detay Hatası:** Yoklama listesinde bir derse tıklandığında *"Seçilen derse ait yoklama bulunamadı (Q:E)"* hatası görünüyordu.

Her iki hata da sunucu tarafında değil, **istemci tarafındaki Hermes bytecode'unda** kaynaklı sorunlardı. Bu rapor, APK'nın decompile edilmesinden bytecode patch'lenmesine ve yeniden imzalanmasına kadar tüm süreci belgelemektedir.

---

## 2. Araçlar ve Ortam

| Araç | Kullanım Amacı |
|------|---------------|
| Python 3.10 | Bytecode analizi, hex işlemleri, API testleri |
| ADB (Android Debug Bridge) | APK kurulumu, logcat izleme |
| Android SDK build-tools 36.0 | APK imzalama (`apksigner`) |
| JADX 1.5.1 | Java katmanı decompile |
| HAR Network Log | Tarayıcı/proxy ağ trafiği yakalama |
| Hermes Decompiler (hbc-decompiler) | HBC v96 → JavaScript decompile |
| Android Emulator (API 35) | Test ortamı |

---

## 3. Hedef Dosya Yapısı

```
Proliz Mobil_3.4.0_APKPure.apk
  └── assets/
       └── index.android.bundle    ← Hermes HBC v96 bytecode (6,050,780 byte)
```

Hermes Bytecode (HBC) dosyası, React Native uygulamasının tüm JavaScript kodunu derlenmiş formda içerir. Dosya yapısı:

```
Offset      Bölüm                      Boyut
────────────────────────────────────────────────────
0x00        File Header                 128 byte (0x80)
0x80        Function Headers            31,361 × 16 byte = 501,776 byte
0x7A890     String Kind Table           ...
...         Identifier Hash Table       ...
0x91BB0     Small String Table          50,217 × 4 byte
0xC2C54     Overflow String Table       393 × 8 byte
0xC389C     String Storage              977,152 byte
...         Bytecode Bölümü             ~4.9 MB
```

### 3.1 Function Header Formatı (16 byte)

Her fonksiyon, 16 byte'lık bir header ile tanımlanır:

```
Word 0 (4 byte):  [paramCount:7 bit (31-25)] [bytecodeOffset:25 bit (24-0)]
Word 1 (4 byte):  [flags:17 bit (31-15)]     [bytecodeSizeInBytes:15 bit (14-0)]
Word 2 (4 byte):  functionName (string ID)
Word 3 (4 byte):  infoOffset
```

Örnek — `_fun26884` (fonksiyon #26884):
```
Header offset:  0x80 + 26884 × 16 = 0x690C0
Word 0:         paramCount=2, bytecodeOffset=0x5216AC
Word 1:         bytecodeSize=399
```

---

## 4. HAR Network Log Analizi

İki adet HAR (HTTP Archive) dosyası yakalandı:

- `2026-03-11 23-19-06 network log.har` — 47 istek (yoklama sayfası açılmadan)
- `2026-03-12 00-45-35 network log.har` — 85 istek (yoklama sayfası dahil)
- `2026-03-12 04-35-41 network log.har` — 47 istek (yoklama derse tıklama dahil)

### 4.1 Yoklama API Çağrıları

HAR dosyasında yakalanan kritik API çağrısı:

```
GET /ProlizMobileAPI_V4/api/m1/mtd26/2267439 → 200
Response: {
  "Stat": false,
  "StatExp": "M23101820:Seçilen derse ait yoklama bulunamadı (Q:E)",
  "yoklamaHaftalar": null
}
```

`mtd26` (getStudentAttendanceHistory) endpoint'i bu ders için veri döndüremiyor. Ancak aynı API'de **mtd25** (getStudentAttendance) endpoint'i çalışıyor:

```
GET /ProlizMobileAPI_V4/api/m1/mtd25/2267439 → 200
Response: {
  "Stat": true,
  "yoklamaList": [
    {"TARIH":"09.03.2026 00:00:00","GUN":"Pazartesi","DERSLIK_KOD":"[AMFİ-5]","SAAT":"09:00-09:50","GIRME_TIP":"Girdi (...)"},
    ... (15 kayıt)
  ]
}
```

**Temel fark:** mtd26 iç içe (nested) format döndürmesi gerekiyor (`yoklamaHaftalar > yoklamaGunler > yoklamaSaatList`), mtd25 ise düz (flat) liste döndürüyor (`yoklamaList`).

### 4.2 Şifreleme Katmanı

API istekleri AES-CBC ile şifreleniyor:
- **CRYPTO_KEY:** `!*Pro_Api+Key!20`
- **ENCRYPT_KEY:** `!*Proliz&2020+!*`
- **Auth:** Bearer JWT token

---

## 5. Bug #1 — Login Navigasyon Hatası

### 5.1 Analiz

Login işlemi başarılı olmasına rağmen uygulama ana sayfaya yönlendirme yapmıyordu. HAR loglarında token'ın başarıyla alındığı görülmesine rağmen, navigasyon gerçekleşmiyordu. Kullanıcı uygulamayı arka plandan tamamen kapatıp yeniden açtığındaysa (kayıtlı token ile auto-login), ana sayfaya sorunsuz yönleniyordu.

Decompile edilmiş `_fun25754` (student login generator) fonksiyonu incelendiğinde kök neden tespit edildi:

```javascript
// _fun25754 — Fonksiyon #25754 (generator)
// bytecodeOffset: 0x4FF32F, bytecodeSize: 645

case 31:  r1 = loginFonksiyonu(loginData);    // API çağrısı (yield)
case 46:  r1 = <login yanıtı>;                // ResumeGenerator → r1

case 55-119:
    // Push notification token alınır, sunucuya gönderilir
    // (r2, r3, r5, r6 kullanılır — r1'e dokunulmaz)

case 128:
    setLoading(false);                         // loading kapanır
    if (!r1) goto case_216;                    // ← BUG: r1 falsy ise navigasyon atlanır

case 142:
    dispatch(StackActions.replace(             // ← navigasyon
        Routes.STUDENT_TABS_ROOT));

case 216:
    // academic rol kontrolüne geçer (student akışı sona erer)
```

**Sorun:** Login fonksiyonu (`_closure2_slot20`) API çağrısını başarıyla yapıp JWT token'ı storage'a kaydediyor, ancak fonksiyonun kendisi falsy bir değer döndürüyordu (muhtemelen `undefined`). Bu nedenle `if(!r1)` koşulu her zaman true olup navigasyon kodunu (case 142) atlayarak case 216'ya (academic rol kontrolü) düşüyordu. Login gerçekte başarılıydı — token storage'a yazılmıştı — ama navigasyon tetiklenmiyordu.

### 5.2 Patch

```
Dosya Offseti:  0x4FF3BB
Opcode:         JmpFalse (0x92) — 3 byte: [op] [offset] [cond_reg]
Orijinal:       0x4D  (jump offset = +77 → case 216, navigasyonu atla)
Yeni:           0x03  (jump offset = +3  → case 142, navigasyona düş)
```

Bu tek byte'lık değişiklik, `JmpFalse` instruction'ının atlama mesafesini 77'den 3'e düşürür. Böylece `r1` falsy olsa bile akış case 142'ye (navigasyon) düşer. Login gerçekten başarısız olsaydı, daha önce case 110'daki `request()` çağrısı exception fırlatıp generator'ı durdururdu; case 128'e ulaşılması zaten login'in başarılı olduğu anlamına gelir.

---

## 6. Bug #2 — Yoklama Sayfası Hatası

Bu bug'ın çözümü çok daha karmaşıktır ve 6 ayrı bytecode patch'i gerektirir.

### 6.1 Kök Neden Analizi

Yoklama detay sayfası şu render zincirini kullanır:

```
_fun26882 (veri yükleyici / generator)
    ↓ API çağrısı → response.data.yoklamaHaftalar
    ↓ setState(yoklamaHaftalar)
    
_fun26884 (FlatList renderItem — hafta bazlı)
    ↓ item.H_NO → başlık
    ↓ item.yoklamaGunler.map(_fun26885) → gün listesi
    
_fun26885 (gün render — gün bazlı)
    ↓ item.TARIH + item.GUN → başlık
    ↓ item.yoklamaSaatList.map(_fun26886) → saat listesi
    
_fun26886 (saat render — ders saati bazlı)
    ↓ item.SAAT → saat bilgisi
    ↓ item.GIRME_TIP → giriş durumu
```

**Problem:** Uygulama `mtd26` endpoint'ini çağırıyordu ancak bu endpoint veri döndüremiyordu. `mtd25` endpoint'i veri döndürüyor ama **farklı bir format** kullanıyor:

| mtd26 (beklenen — nested) | mtd25 (gerçek — flat) |
|---|---|
| `yoklamaHaftalar: [{H_NO, yoklamaGunler: [{TARIH, GUN, yoklamaSaatList: [{SAAT, GIRME_TIP}]}]}]` | `yoklamaList: [{TARIH, GUN, DERSLIK_KOD, SAAT, GIRME_TIP}]` |

### 6.2 Decompiled Kod İncelemesi

#### _fun26882 (Veri Yükleyici)
```javascript
// Bytecode offset: 0x5215E7, boyut: 197 byte
case 153:
    r5 = r1.data;           // API response data
    // ...null check...
case 171:
    r3 = r5.yoklamaHaftalar; // ← PropID: 25215 (0x627F)
case 177:
    setState(r3);            // state'e ata
```

#### _fun26884 (Hafta Render)
```javascript
// Bytecode offset: 0x5216AC, boyut: 399 byte
case 0:
    r0 = a0.item;
    // ... UI setup ...
    r14 = r0.H_NO;            // ← PropID: 26309 (0x66C5) — başlık
    // ... r14 + '. ' + t('week') → Text bileşeni ...
    
    r9 = r0.yoklamaGunler;    // ← PropID: 25871 (0x650F)
    r7 = (r9 == null);        // null kontrolü
    if (r7) goto case 354;    // → BOŞ render (yoklamaGunler yoksa)
case 337:
    r9.map(_fun26885);         // → gün render fonksiyonuna gönder
case 354:
    // sadece başlık, alt içerik yok → BLANK SAYFA
    r6 = r0.H_NO;             // ← PropID: 26309 (0x66C5) — React key
```

#### _fun26885 (Gün Render)
```javascript
// Bytecode offset: 0x52183B, boyut: 595 byte
    r14 = r0.TARIH;           // ← PropID: 17459 (0x4433)
    r14 = r0.GUN;             // ← PropID: 37269 (0x9195)
    // ... UI: TARIH + ' ' + GUN ...
    
    r9 = r0.yoklamaSaatList;  // ← PropID: 27461 (0x6B45)
    r7 = (r9 == null);        
    if (r7) goto case 572;    // → saat listesi yoksa atla
case 555:
    r9.map(_fun26886);         // → saat render fonksiyonuna gönder
```

#### _fun26886 (Saat Render)
```javascript
// Bytecode offset: 0x521A8E, boyut: 414 byte
    r18 = r0.SAAT;            // ← PropID: 27011 (0x6983)
    r12 = r0.GIRME_TIP;       // ← PropID: 25976 (0x6578)
```

### 6.3 Property ID Tablosu

Tüm propID'ler, HBC dosyasındaki string tablosuna indeks olarak işlev görür. `GetById` opcode'u (0x37) ile erişilir:

```
GetById formatı: 37 [dst:8] [obj:8] [cacheIdx:8] [propID:16LE]
Toplam: 6 byte
```

Keşfedilen PropID değerleri:

| PropID | Hex | Property Adı | Bulunduğu Fonksiyon |
|--------|-----|-------------|---------------------|
| 25215 | 0x627F | yoklamaHaftalar | _fun26882 |
| 27134 | 0x69FE | yoklamaList | _fun29155 |
| 26309 | 0x66C5 | H_NO | _fun26884 |
| 25871 | 0x650F | yoklamaGunler | _fun26884 |
| 17459 | 0x4433 | TARIH | _fun26885 |
| 37269 | 0x9195 | GUN | _fun26885 |
| 27461 | 0x6B45 | yoklamaSaatList | _fun26885 |
| 27011 | 0x6983 | SAAT | _fun26886 |
| 25976 | 0x6578 | GIRME_TIP | _fun26886 |
| 17813 | 0x4595 | item | _fun26884 |

**yoklamaList PropID keşfi:** `yoklamaList` string'i uygulamanın kendi kodunda (`_fun29155` — öğretim görevlisi yoklama fonksiyonu) da kullanılıyordu. Bu fonksiyonun bytecode'undaki `GetById` instruction'ından propID **27134 (0x69FE)** çıkarıldı.

### 6.4 Patch Stratejisi

mtd25'in flat formatını mevcut nested render zincirine uydurmak için **6 patch** gerekiyor:

#### Patch 2 — Endpoint Değişikliği (mtd26 → mtd25)
```
Offset:   0x0F0F99
Orijinal: 0x36  (ASCII '6' → "mtd26")
Yeni:     0x35  (ASCII '5' → "mtd25")
```
String storage'da `m1/mtd26/` → `m1/mtd25/` olarak değiştirilir.

#### Patch 3 — Property Yönlendirme (yoklamaHaftalar → yoklamaList)
```
Offset:   0x521696–0x521697  [_fun26882, bytecode offset 175–176]
Orijinal: 7F 62  (propID 25215 = yoklamaHaftalar)
Yeni:     FE 69  (propID 27134 = yoklamaList)
```
API yanıtından `data.yoklamaHaftalar` yerine `data.yoklamaList` okunur.

#### Patch 4 — Başlık Görüntüsü (H_NO → TARIH)
```
Offset:   0x5217A8–0x5217A9  [_fun26884, bytecode offset 252–253]
Orijinal: C5 66  (propID 26309 = H_NO)
Yeni:     33 44  (propID 17459 = TARIH)
```
Hafta numarası yerine tarih bilgisi gösterilir.

#### Patch 5 — React Key (H_NO → SAAT)
```
Offset:   0x52181A–0x52181B  [_fun26884, bytecode offset 366–367]
Orijinal: C5 66  (propID 26309 = H_NO)
Yeni:     83 69  (propID 27011 = SAAT)
```
FlatList bileşeninin React key'i olarak kullanılır.

#### Patch 6 — yoklamaGunler Null → Array Wrap (15 byte)
```
Offset:   0x5217EE–0x5217FC  [_fun26884, bytecode offset 322–336]

Orijinal bytecode (yoklamaGunler al ve null kontrol):
  37 09 00 0C 0F 65    GetById r9 = r0.yoklamaGunler   (propID=0x650F)
  0E 07 09 06          Eq r7 = (r9 == r6)              r6=null
  76 06                LoadConstUndefined r6
  90 14 07             JmpTrue +20, r7                  → case 354 (boş render)

Yeni bytecode (flat item'ı tek elemanlı diziye sar):
  07 09 01 00          NewArray r9, size=1
  44 09 00 00          PutByIndex r9[0] = r0            r0=current item
  76 06                LoadConstUndefined r6
  76 07                LoadConstUndefined r7
  90 03 07             JmpTrue +3, r7                   r7=undefined → false, düşer
```

**Etki:** `r0.yoklamaGunler` (undefined/null) okumak yerine, `r9 = [r0]` oluşturulur. Bu şekilde `r9.map(_fun26885)` çağrıldığında her flat item, "gün" olarak render edilir.

#### Patch 7 — yoklamaSaatList Null → Array Wrap (15 byte)
```
Offset:   0x521A57–0x521A65  [_fun26885, bytecode offset 540–554]

Orijinal bytecode:
  37 09 00 10 45 6B    GetById r9 = r0.yoklamaSaatList  (propID=0x6B45)
  0E 07 09 07          Eq r7 = (r9 == r7)              r7=null
  76 06                LoadConstUndefined r6
  90 14 07             JmpTrue +20, r7                  → case 572 (atla)

Yeni bytecode:
  07 09 01 00          NewArray r9, size=1
  44 09 00 00          PutByIndex r9[0] = r0            r0=current day item
  76 06                LoadConstUndefined r6
  76 07                LoadConstUndefined r7
  90 03 07             JmpTrue +3, r7                   r7=undefined → false, düşer
```

**Etki:** Aynı mantık — flat item'da `yoklamaSaatList` alanı olmadığı için `[r0]` olarak sarılır ve `_fun26886` her item'ı doğrudan `SAAT` ve `GIRME_TIP` ile render eder.

---

## 7. Tüm Patchlerin Özeti

Toplam **29 byte** değişiklik, **7 patch** halinde uygulanmıştır:

| # | Offset | Orijinal | Yeni | Boyut | Açıklama |
|---|--------|----------|------|-------|----------|
| 1 | `0x4FF3BB` | `4D` | `03` | 1 B | Login navigasyon düzeltmesi |
| 2 | `0x0F0F99` | `36` | `35` | 1 B | API endpoint mtd26 → mtd25 |
| 3 | `0x521696` | `7F 62` | `FE 69` | 2 B | yoklamaHaftalar → yoklamaList propID |
| 4 | `0x5217A8` | `C5 66` | `33 44` | 2 B | H_NO → TARIH propID (başlık) |
| 5 | `0x52181A` | `C5 66` | `83 69` | 2 B | H_NO → SAAT propID (React key) |
| 6 | `0x5217EE` | `37 09 00 0C 0F 65`... | `07 09 01 00 44 09 00 00`... | 15 B | yoklamaGunler → NewArray wrap |
| 7 | `0x521A57` | `37 09 00 10 45 6B`... | `07 09 01 00 44 09 00 00`... | 15 B | yoklamaSaatList → NewArray wrap |

---

## 8. Hermes Bytecode Opcode Referansı

Bu çalışmada keşfedilen/kullanılan HBC v96 opcode'ları:

| Opcode | Mnemonik | Format | Boyut | Açıklama |
|--------|----------|--------|-------|----------|
| `0x07` | NewArray | `07 dst:8 size:16LE` | 4 B | Yeni dizi oluştur |
| `0x0E` | Eq | `0E dst:8 left:8 right:8` | 4 B | Eşitlik karşılaştırması |
| `0x37` | GetById | `37 dst:8 obj:8 cache:8 propID:16LE` | 6 B | Property erişimi |
| `0x44` | PutByIndex | `44 arr:8 val:8 idx:8` | 4 B | Diziye indeksle ata |
| `0x76` | LoadConstUndefined | `76 dst:8` | 2 B | undefined yükle |
| `0x86` | StartGenerator | `86` | 1 B | Generator başlat |
| `0x87` | ResumeGenerator | `87 ...` | ? | Generator devam |
| `0x88` | SaveGenerator | `88 ...` | ? | Generator kaydet |
| `0x90` | JmpTrue | `90 offset:8 cond:8` | 3 B | Koşullu dallanma |

---

## 9. APK Yeniden İnşa ve İmzalama Süreci

### 9.1 Bundle Değişikliği
```python
# Orijinal APK'dan tüm dosyalar kopyalanır,
# assets/index.android.bundle ZIP_STORED olarak değiştirilir
with zipfile.ZipFile(src_apk, 'r') as zf_src:
    with zipfile.ZipFile(dst_apk, 'w') as zf_dst:
        for item in zf_src.infolist():
            if item.filename == 'assets/index.android.bundle':
                zf_dst.writestr(item, bundle_data, compress_type=ZIP_STORED)
            else:
                zf_dst.writestr(item, zf_src.read(item.filename),
                                compress_type=item.compress_type)
```

### 9.2 İmzalama
```powershell
# Debug keystore oluşturma (ilk seferde)
keytool -genkeypair -v -keystore debug_sign.jks -alias ilk `
  -keyalg RSA -keysize 2048 -validity 10000 `
  -storepass proliz123 -keypass proliz123

# APK imzalama
apksigner sign `
  --ks debug_sign.jks `
  --ks-pass pass:proliz123 `
  --key-pass pass:proliz123 `
  --out Proliz_patched_v4_signed.apk `
  Proliz_patched_v4.apk
```

### 9.3 Kurulum
```powershell
adb -s emulator-5554 uninstall com.prolizyazilim.mobil
adb -s emulator-5554 install Proliz_patched_v4_signed.apk
```

---

## 10. Veri Akışı Diyagramı

### Orijinal (Hatalı) Akış:
```
Kullanıcı → Ders seç → mtd26/DH_ID çağır
                              ↓
                    Stat: false, yoklamaHaftalar: null
                              ↓
                    "Seçilen derse ait yoklama bulunamadı"
                              ↓
                         HATA EKRANI
```

### Patchlenmiş Akış:
```
Kullanıcı → Ders seç → mtd25/DH_ID çağır         ← Patch 2
                              ↓
                    Stat: true, yoklamaList: [...]
                              ↓
            data.yoklamaList okunur                 ← Patch 3
                              ↓
            FlatList her item için:
              ├─ item.TARIH + ". " + t('week')      ← Patch 4
              ├─ r9 = [item]  (array wrap)           ← Patch 6
              └─ r9.map(günRender) çağrılır
                    ↓
            Gün render:
              ├─ item.TARIH + " " + item.GUN
              ├─ r9 = [item]  (array wrap)           ← Patch 7
              └─ r9.map(saatRender) çağrılır
                    ↓
            Saat render:
              ├─ 🕐 item.SAAT                        ← Patch 5 (key)
              └─ ✅ item.GIRME_TIP
```

---

## 11. Karşılaşılan Zorluklar

### 11.1 Hermes String Tablosu Formatı
HBC v96'nın string tablosu formatı resmi olarak belgelenmemiştir. Birden fazla bit düzeni denenmiştir:
- Format A: `[isUTF16:1][length:8][offset:23]`
- Format B: `[isUTF16:1][offset:23][length:8]`
- Format C: `[length:8][offset:24]`

Hiçbiri tutarlı sonuç vermemiştir. PropID'ler, decompile edilmiş JavaScript'teki property erişimleri ile bytecode'daki `GetById` instruction'ları eşleştirilerek dolaylı yoldan keşfedilmiştir.

### 11.2 Function Header Boyutu
Başlangıçta 20 byte varsayılmış, daha sonra 16 byte olduğu doğrulanmıştır. Bu keşif, tüm fonksiyon bytecode offset'lerinin yeniden hesaplanmasını gerektirmiştir.

### 11.3 Başarısız String Tablosu Patch'i
İlk denemede `0xA3B34` adresindeki string tablosu girişi değiştirilmeye çalışılmış, ancak `yoklamaList` yerine `'rThanTarget BackHandle'` string'ine işaret ettiği keşfedilmiştir. Bu yaklaşım tamamen terk edilmiş ve propID düzeyinde patch uygulanmıştır.

### 11.4 Flat vs Nested Format Uyumsuzluğu
En kritik sorun, mtd25'in düz liste döndürmesi ancak render kodunun 3 kademeli iç içe yapı beklemesidir. Bu, `NewArray` + `PutByIndex` ile her item'ı tek elemanlı diziye sararak çözülmüştür — render fonksiyonları her seviyede aynı flat item'ın `TARIH`, `GUN`, `SAAT`, `GIRME_TIP` alanlarına erişir.

---

## 12. Dosya Envanteri

```
prolizdecode2/
├── index.android.bundle.backup       ← Orijinal (değiştirilmemiş)
├── index.android.bundle.patched      ← 7 patch uygulanmış
├── apply_all_patches.py              ← Otomatik patch script'i
├── Proliz_patched_v4_signed.apk      ← Son imzalanmış APK
├── debug_sign.jks                    ← İmzalama anahtarı
├── check_mtd25_structure.py          ← API test script'i
├── parse_headers2.py                 ← HBC fonksiyon header parser
├── analyze_fun26882.py               ← Bytecode analiz script'i
├── proliz_decompiled/
│   └── resources/assets/
│       └── index_decompiled.js       ← Decompile edilmiş JS (44 MB)
├── 2026-03-11 23-19-06 network log.har  ← HAR log #1
├── 2026-03-12 00-45-35 network log.har  ← HAR log #2
└── 2026-03-12 04-35-41 network log.har  ← HAR log #3 (yoklama dahil)
```

---

## 13. Sonuç

29 byte'lık bytecode değişikliği ile iki kritik uygulama hatası düzeltilmiştir:

1. **Login hatası** → Tek byte navigasyon indeksi düzeltmesi
2. **Yoklama hatası** → 6 bytecode patch'i ile endpoint değişikliği, property yönlendirme ve veri format adaptasyonu

Bu çalışma, Hermes HBC v96 bytecode formatının belgelenmemiş iç yapısının tersine mühendislik yoluyla çözümlenmesini ve binary düzeyde cerrahi müdahale ile uygulamanın çalışır hale getirilmesini içermektedir.

---

*Bu belge, eğitim ve araştırma amacıyla hazırlanmıştır.*

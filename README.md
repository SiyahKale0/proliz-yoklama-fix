# Proliz Mobil v3.4.0 — Bytecode Bugfix

Gümüşhane Üniversitesi öğrenci bilgi sistemi mobil uygulaması **Proliz Mobil**'deki iki kritik hatanın, Hermes HBC v96 bytecode düzeyinde tersine mühendislik ile düzeltilmesi.

## Düzeltilen Hatalar

| # | Hata | Çözüm |
|---|------|-------|
| 1 | **Login** — Doğru bilgilerle giriş yapıldığında ana sayfaya yönlendirme yapılmıyor | Navigasyon indeksi düzeltmesi (1 byte) |
| 2 | **Yoklama** — Derse tıklandığında *"Seçilen derse ait yoklama bulunamadı"* hatası | Endpoint değişikliği + veri format adaptasyonu (6 patch, 28 byte) |

## Teknik Özet

- **Toplam değişiklik:** 29 byte, 7 patch
- **Hedef dosya:** `assets/index.android.bundle` (Hermes HBC v96, 6 MB)
- **Yöntem:** Binary bytecode patching — opcode, propID ve string storage düzeyinde cerrahi müdahale

### Yoklama Fix Detayı

Uygulama `mtd26` (getStudentAttendanceHistory) endpoint'ini çağırıyordu ancak sunucu `Stat: false` döndürüyordu. Çalışan `mtd25` (getStudentAttendance) endpoint'i ise **farklı bir veri formatı** (flat liste vs nested yapı) kullanıyordu.

Çözüm:
1. API endpoint'i `mtd26` → `mtd25` olarak değiştirildi
2. Response property `yoklamaHaftalar` → `yoklamaList` propID yönlendirmesi
3. Render zincirindeki property erişimleri (`H_NO` → `TARIH`, `SAAT`) güncellendi
4. Nested yapı bekleyen null-check'ler, `NewArray` + `PutByIndex` ile flat item'ı tek elemanlı diziye sararak bypass edildi

## Dosyalar

| Dosya | Açıklama |
|-------|----------|
| [**Proliz_patched_v4_signed.apk**](https://github.com/SiyahKale0/proliz-yoklama-fix/releases/tag/v4.0) | İmzalanmış APK (~129 MB, Release'den indirin) |
| [**RAPOR_Tersine_Muhendislik.md**](RAPOR_Tersine_Muhendislik.md) | Detaylı tersine mühendislik raporu |

## Patch Tablosu

| # | Offset | Boyut | Açıklama |
|---|--------|-------|----------|
| 1 | `0x4FF3BB` | 1 B | Login navigasyon düzeltmesi |
| 2 | `0x0F0F99` | 1 B | API endpoint mtd26 → mtd25 |
| 3 | `0x521696` | 2 B | yoklamaHaftalar → yoklamaList propID |
| 4 | `0x5217A8` | 2 B | H_NO → TARIH propID (başlık) |
| 5 | `0x52181A` | 2 B | H_NO → SAAT propID (React key) |
| 6 | `0x5217EE` | 15 B | yoklamaGunler → NewArray wrap |
| 7 | `0x521A57` | 15 B | yoklamaSaatList → NewArray wrap |

## Araçlar

- Python 3.10 — Bytecode analizi ve hex patching
- JADX 1.5.1 — Java decompile
- hbc-decompiler — Hermes HBC → JS decompile
- ADB + Android Emulator (API 35)
- apksigner (build-tools 36.0)

---

*Bu çalışma eğitim ve araştırma amacıyla yapılmıştır.*

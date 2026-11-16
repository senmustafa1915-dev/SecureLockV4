# Secure Lock (Android / Kotlin)

GitHub'a aktarılabilir Android Studio kaynak projesi. APK veya AAB bu depoya eklenmemiştir.

## Proje yapısı

- `app/src/main/java/com/securelock/data`: kilit kuralları, şifreli politika deposu
- `app/src/main/java/com/securelock/security`: PBKDF2 parola doğrulama ve Android Keystore AES-GCM kasa şifrelemesi
- `app/src/main/java/com/securelock/core`: politika değerlendirme ve geçici açma oturumları
- `app/src/main/java/com/securelock/service`: erişilebilirlik tabanlı ön plan uygulama kilidi
- `app/src/main/java/com/securelock/device`: Device Admin / Device Owner entegrasyonu
- `app/src/main/java/com/securelock/ui`: kurulum, yönetim paneli ve kilit ekranları
- `app/src/test`: zamanlama kuralı birim testleri
- `adb`: Bugjaeger veya ADB kabuğunda kullanılacak komutlar

## Derleme

Android Studio'da bu klasörü aç. Android SDK Platform 35 ve JDK 17 kurulu olmalı. Gradle senkronizasyonundan sonra:

```bash
gradle :app:testDebugUnitTest
gradle :app:assembleDebug
```

Bu depo kaynak kodudur; derleme çıktıları repoya eklenmez.

## İlk kurulum

1. Uygulamayı cihaza kurup bir kez aç.
2. Birbirinden farklı Master, Normal ve Katı Mod şifrelerini oluştur.
3. Android'in Erişilebilirlik ayarlarından **Secure Lock uygulama koruması** hizmetini etkinleştir.
4. Secure Lock'ın yalnızca kendi paketinin kaldırılmasını engellemesi için uygulamayı Device Owner olarak kaydet (aşağıdaki komut).
5. Uygulamaya geri dön; cihaz sahibi ise kendi paketinin kaldırılma engelini otomatik uygular.

### Bugjaeger / ADB — Device Owner

Paket adı `com.securelock` olan release kurulumu için:

```bash
 dpm set-device-owner com.securelock/.device.LockDeviceAdminReceiver
```

Debug varyantı kurulmuşsa paket adı `com.securelock.debug` olur:

```bash
 dpm set-device-owner com.securelock.debug/.device.LockDeviceAdminReceiver
```

Komut, Bugjaeger'de cihazın ADB shell oturumunda çalıştırılır. Android, Device Owner kaydını yalnızca cihazın mevcut kurulum/provisioning durumunun izin verdiği hâllerde kabul eder. Device Admin iznini açmak Device Owner olmakla aynı şey değildir.

### Erişilebilirlik servisi — ADB alternatifi

Normal yol Android Ayarlar > Erişilebilirlik > Yüklü uygulamalar > Secure Lock hizmetini açmaktır. ADB shell alternatifi:

```bash
current=$(settings get secure enabled_accessibility_services)
service=com.securelock/com.securelock.service.LockAccessibilityService
case ":$current:" in
  *":$service:"*) ;;
  *) settings put secure enabled_accessibility_services "${current:+$current:}$service" ;;
esac
settings put secure accessibility_enabled 1
```

Bu komut mevcut erişilebilirlik servislerini silmeden Secure Lock'ı listeye ekler. Samsung yazılımı bu ayarı reddederse sistem ayarları ekranından etkinleştir.

## Kilit davranışları

- **Normal:** ortak parola. Parola doğru girildiğinde otomatik açılır. İsteğe bağlı 5 veya 10 dakikalık mola seçilebilir; mola seçilmezse yalnızca mevcut ön plan oturumu açılır.
- **Sıkı / VIP:** paket başına ayrı parola. Normal parola VIP yerine geçmez ve mola seçeneği sunulmaz.
- **Katı:** belirtilen süre dolana kadar parola alanı gösterilmez. Süre bitince diğer etkin kurallar tekrar değerlendirilir. Katı süreyi yönetim panelinden kaldırmak için ayrı Katı Mod parolası gerekir.
- **Planlı:** belirtilen günlük saat aralığında Normal parola ister. Gece yarısını aşan aralıklar desteklenir; bu sürümde zamanlama her gün uygulanır. Erişilebilirlik servisi, kural atanmış ve ön planda olan uygulamayı saniyede bir yeniden değerlendirerek zaman aralığı başladığında kilidi devreye alır.
- Bir uygulamada Normal + VIP aynı anda açıksa iki parola da gerekir. Katı Mod bunların üzerinde önceliklidir.
- Üç hatalı denemeden sonra kademeli bekleme uygulanır.

## Güvenlik / platform sınırları

Bu proje normal Android SDK ile yapılabilecek korumaları kullanır; Android'in vermediği sistem yetkilerini varmış gibi göstermez.

1. **Kendi uygulamasını kaldırma:** `DevicePolicyManager.setUninstallBlocked()` yalnızca Device Owner (ve uygun bağlı Profile Owner) yetkisiyle çağrılır. Politika yalnızca `com.securelock` paketine uygulanır; diğer uygulamaların kaldırılması bu çağrıyla engellenmez.
2. **Ayarlar'daki korumalı sayfalar:** Erişilebilirlik servisi, Secure Lock adına ait kaldırma/uygulama bilgisi sayfasını ve tanınabilen Erişilebilirlik / Tarih ve Saat sayfalarını algıladığında ana ekrana döndürmeye çalışır. Tarih-saat değişikliği ayrıca Device Owner kullanıcı kısıtlamasıyla engellenir. Sayfa metinleri ve Activity adları Samsung/Android sürümüne göre değişebilir; sayfa yönlendirmesi tek başına güvenlik sınırı değildir.
3. **FLAG_SECURE:** Bu işaret Secure Lock'ın kendi ekranlarına uygulanır. Standart bir üçüncü taraf uygulama, Galeri/WhatsApp gibi başka bir uygulamanın penceresine `FLAG_SECURE` ekleyemez.
4. **Son kullanılanlar kartları ve görev silme:** Standart Android SDK, başka uygulamaların görev kartı görüntüsünü uygulama bazında değiştirme veya başka uygulamanın görevini keyfî olarak `removeTask` ile silme yetkisi vermez. Dolayısıyla bu sürüm, üçüncü taraf uygulamaların son kullanılanlar önizlemelerinin kesin olarak siyah olacağını iddia etmez.
5. **Bölünmüş ekran:** Genel Android API'sinde yalnızca belirli üçüncü taraf paketler için bölünmüş ekranı kalıcı olarak kapatan standart bir Device Owner politikası yoktur. Erişilebilirlik servisi kilitli paket ön plana geldiğinde kilit ekranını öne getirir; bu, her Samsung çoklu-pencere yolu için OEM düzeyinde garanti değildir.
6. **Tarih/saat:** Device Owner olduğunda kullanıcı arayüzünden tarih/saat değişikliğini kısıtlayan `DISALLOW_CONFIG_DATE_TIME` eklenir. Root veya işletim sistemi düzeyindeki değişikliklere karşı garanti verilmez.
7. **Erişilebilirlik servisi:** Kullanıcı tarafından etkinleştirilmelidir. Android/OEM güç yönetimi, servisi durdurabilir veya yeniden başlatabilir; erişilebilirlik servisi tek başına sistem çekirdeği güvenlik sınırı değildir.

Seçili uygulamaların Recents görüntülerini ve split-screen davranışını tam OEM düzeyinde yönetmek için cihaz üreticisinin ayrı kurumsal/Knox API'si veya sistem imzalı ayrıcalıklı entegrasyon gerekir. Bu projede bulunmayan bir Knox özelliği çalışıyormuş gibi sunulmamıştır.

## Güvenlik notları

- Şifreler düz metin olarak kaydedilmez; PBKDF2-HMAC-SHA256 ile doğrulama kaydı oluşturulur.
- Kural verisi Android Keystore'da tutulan AES-256-GCM anahtarıyla şifrelenir.
- Android yedeklemesi kapalıdır.
- Device Owner yetkisini kaldırma/cihazı yeniden kurma gibi yönetim işlemleri bu uygulamanın kontrol sınırının dışındadır.

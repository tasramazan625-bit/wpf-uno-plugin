# wpf-uno-tools — Ekip için WPF→Uno dönüşüm marketplace'i

Bu repo, WPF→Uno dönüşüm skill'ini ve referanslarını 30 kişilik ekibe dağıtmak içindir.
Tek kaynak burası; sen güncelle, ekip tek komutla son sürümü alsın.

## Kurulum öncesi (sen — bir kez)
1. `plugins/wpf-to-uno/skills/wpf-to-uno/SKILL.md` içine kendi skill'ini yapıştır.
2. `references/` klasörüne kendi referans dosyalarını koy.
3. `marketplace.json` içindeki owner adını/e-postasını düzelt.
4. Doğrula: `claude plugin validate .`
5. Bu repoyu private git'e (GitHub/GitLab) push'la.

## Ekip üyesi — bir kez kurar
```
/plugin marketplace add SENIN-ORG/wpf-uno-marketplace
/plugin install wpf-to-uno@wpf-uno-tools
```
(GitLab/Bitbucket ise tam URL ile: `/plugin marketplace add https://gitlab.com/org/wpf-uno-marketplace.git`)

## Güncelleme akışı
- SEN: değişikliği repoya push'la. Hepsi bu.
- EKİP: `/plugin marketplace update wpf-uno-tools` çalıştırır, son sürümü alır.

`plugin.json` içinde `version` alanı YOK; git tabanlı dağıtımda her commit otomatik olarak
yeni sürüm sayılır. Yani her seferinde versiyon numarası artırmana gerek yok — sadece push.

> Kontrollü dağıtım istersen (örn. herkes aynı anda v1 ile dönüştürsün diye): plugin.json'a
> "version": "1.0.0" ekle ve her release'de bu numarayı artır. O zaman sadece sen artırınca
> güncelleme iner.

## Otomatik hatırlatma (opsiyonel ama önerilir)
Dönüştürülecek her projenin CLAUDE.md dosyasına şu nota benzer bir satır ekle:
"Bu projede WPF→Uno dönüşümü için wpf-to-uno@wpf-uno-tools plugin'i gerekir.
Yoksa kullanıcıya kurmasını/güncellemesini söyle." Böylece Claude eksikse uyarır.

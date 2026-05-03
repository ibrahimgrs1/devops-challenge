# 🚀 Kapalı Network CI/CD Pipeline & Deployment Mimarisi

Bu proje, güvenlik kısıtları nedeniyle dış dünyaya (internete) kapalı bir ağda çalışan uygulamalar için tasarlanmış uçtan uca, otomatik bir CI/CD ve Zero-Downtime Deployment sürecini simüle etmektedir. 

## 🏗️ Mimari Yaklaşım ve Sistem İşleyişi

Sistem, "Kapalı Ağ" (Air-gapped/Closed Network) prensiplerine uygun olarak tasarlanmıştır. İçeride çalışan bileşenler dışarıdan doğrudan trafik kabul etmez.

**Pipeline Akışı:**
1. **Push:** Geliştirici kodu GitHub'a gönderir (Commit SHA ile etiketlenir).
2. **Tetikleme (Webhook):** Sadece belirli bir porttan açılan güvenli tünel üzerinden GitHub, yerel ağdaki Jenkins'e `200 OK` ulaştırır.
3. **Continuous Integration (CI):** Jenkins, kodu çeker, Docker imajını derler ve katman önbelleği (caching) kullanarak optimize eder.
4. **Private Registry:** Derlenen imaj, `Basic Auth` ile korunan, ağ içine izole edilmiş yerel Docker Registry'ye (`localhost:5000`) push edilir.
5. **Continuous Deployment (CD):** Jenkins, Docker Compose kullanarak yeni imajı çeker. Eski konteyner durdurulmadan yenisi başlatılır (Zero-Downtime) ve trafik `Nginx` (Reverse Proxy) üzerinden sadece sağlıklı konteynerlere (Healthcheck) yönlendirilir.

## 🛡️ Ağ Kısıtı ve İzolasyon Çözümü (Network Constraint)

**Problem:** Uygulama ortamının dış internete kapalı olması ancak GitHub Webhook'unun içeriye erişebilmesi.
**Çözüm:** 
* Sistemdeki servisler (Jenkins, Registry, App) Docker içerisinde özel bir `internal-network` üzerinde çalıştırılarak birbirleriyle iletişim kurmaları, dışarıdan ise izole olmaları sağlanmıştır. 
* Dış dünyadan gelen tek tetikleme olan Webhook, **Ngrok** kullanılarak spesifik olarak Jenkins'in dinlediği `/github-webhook/` endpoint'ine tünellenmiştir. Bu sayede uygulamanın kendisine veya sunucuya doğrudan bir Inbound kuralı (port açma) uygulanmamış, sistem dışarıya kapalı tutulmuştur.

## 🐛 Özel Hata ve Debugging Senaryosu (Exception Handling)

Uygulama kodunun içerisine, CI/CD süreçlerinde hata ayıklama ve loglama mekanizmalarını test etmek amacıyla kasıtlı bir `Exception` (İstisna) bloğu eklenmiştir.

* **Senaryo:** Kullanıcı, Todo uygulaması üzerinden yeni bir görev eklerken input alanına tam olarak **`TODO test`** yazıp gönderdiğinde uygulama işlemi reddeder.
* **Sonuç:** Backend kasıtlı olarak bir `500 Internal Server Error` fırlatır. Bu hata, kapalı ağdaki Docker konteyner loglarında (`docker logs <container-name>`) yakalanarak sistemin log mekanizmasının düzgün çalıştığı kanıtlanır.

## 🌟 Entegre Edilen Bonus Özellikler

* **Zero-downtime Deployment:** Docker Compose üzerinde `--force-recreate` ve Nginx routing kullanılarak, yeni imaj "healthy" durumuna geçmeden eski konteyner sonlandırılmaz.
* **Healthcheck Tanımları:** Konteynerlerin sağlıklı çalışıp çalışmadığı `docker-compose.yml` içerisinde düzenli ping'lerle kontrol edilir.
* **Self-signed SSL:** Dışarıdan veya ağ içinden uygulamaya erişim sadece `HTTPS (Port 443)` üzerinden, Nginx tarafından yönetilen Self-Signed sertifika ile sağlanır.
* **Monitoring:** Kaynak tüketimini (CPU, RAM) izlemek için hafif yapılı **cAdvisor** kullanılmıştır (`Port 8081`).
* **Rollback Mekanizması:** Jenkinsfile içerisine parametrik yapı eklenmiş olup, istenilen `DEPLOY_TAG` numarası girilerek eski bir stabil sürüme saniyeler içinde dönülebilir.
* **Pipeline Optimizasyonu:** Docker imajları derlenirken cache mekanizması aktif kullanılmıştır.

---

## ⚙️ Kurulum ve Çalıştırma Adımları

**1. Ön Gereksinimler**
* Docker ve Docker Compose
* Ngrok (Webhook tünellemesi için)
* Jenkins (Docker Container veya Native)

**2. Private Registry Kurulumu**
Güvenli depo için Basic Auth kimlik bilgileri oluşturun ve registry'yi ayağa kaldırın:
```bash
# Auth klasörü oluştur ve şifre belirle
docker run --entrypoint htpasswd httpd:2.4 -Bbn KULLANICI_ADI SIFRE > auth/htpasswd
docker-compose up -d registry
3. Ngrok Tüneli ve Webhook

Bash
ngrok http 8080
Terminalde verilen https://<hash>.ngrok-free.dev adresini alın ve GitHub deponuzdaki Webhook kısmına https://KULLANICI_ADI:JENKINS_TOKEN@<hash>.ngrok-free.dev/github-webhook/ formatında ekleyin.

4. Sistemi Ayağa Kaldırma
Jenkins üzerinden veya manuel olarak tüm yapıyı başlatın:

Bash
docker-compose up -d --build
Uygulamaya https://localhost üzerinden güvenli şekilde erişebilirsiniz.

**📦 Örnek Uygulamanın Çalıştırılması**
Sistemdeki Todo uygulaması, CI/CD pipeline'ı tarafından Docker imajı olarak paketlenip yayınlanmaktadır. 
Uygulamayı pipeline dışında, yerel ortamınızda geliştirici modunda test etmek isterseniz:
1. `todo-app` (veya uygulamanızın klasörü) dizinine girin.
2. Bağımlılıkları kurmak için `npm install` komutunu çalıştırın.
3. Uygulamayı başlatmak için `npm start` komutunu kullanın.

🛠️ Karşılaşılan Problemler ve Çözümleri
Geliştirme sürecinde "Kapalı Ağ" simülasyonu kaynaklı karşılaşılan sorunlar şu şekilde çözülmüştür:

Jenkins 403 Forbidden Hatası (Webhook Reddi):

Problem: GitHub'dan gelen istekler Jenkins güvenlik duvarı (CSRF) tarafından reddedildi.

Çözüm: API Token ("Crumb" doğrulaması) üretilip Webhook URL'sine Basic Auth yapısıyla (kullanıcı:token@...) gömülerek güvenli giriş sağlandı.

Git Checkout Timeout (Port 443 Erişilememesi):

Problem: Jenkins, kapalı network nedeniyle kodu çekerken dışarı çıkamadı.

Çözüm: Jenkins konteynerinin DNS ayarları güncellenerek (Outbound kuralı esnetilerek) sadece GitHub sunucularına veri çekme yetkisi verildi.

Ngrok ERR_NGROK_3200 (Offline Hatası):

Problem: Tünel aktif görünse de endpoint'in GitHub'a yanıt vermemesi.

Çözüm: GitHub üzerindeki F5 (sayfa yenileme) yerine "Redeliver" işlemi yapılması gerektiği tespit edildi. Dinamik tünel linki ile senkronizasyon sağlandı.

Health Check Nginx Uyuşmazlığı:

Problem: Uygulama konteyneri yerine Proxy katmanının durumunun kontrol edilmesi.

Çözüm: Healthcheck rotası, uygulamanın kendi backend sağlık endpoint'ine (/health) yönlendirildi.

⚖️ Varsayımlar ve Trade-off'lar
Full Monitoring Stack Yerine cAdvisor: Sistemin zaten kapalı bir ağda ve izole çalışması, ayrıca AI/ML modelleri eklendiğinde sistem yükünün artacak olması göz önüne alınarak; Prometheus & Grafana ikilisi yerine başlangıç olarak daha az donanım tüketen cAdvisor tercih edilmiştir. İlerleyen süreçte metriklerin genişletilmesi kolaydır.

Ngrok Kullanımı: Kapalı network'e sızmak için şirket firewall'unda kalıcı delik açmak (Port Forwarding) yerine, Jenkins'in sadece ihtiyaç anında dinleyeceği geçici bir Ngrok Reverse Proxy kullanılarak güvenlik sıkılaştırılmıştır.

Self-Signed SSL: İç ağda dış sertifika otoritelerine (Let's Encrypt vb.) ulaşılamayacağı varsayılarak kimlik doğrulama self-signed olarak kurgulanmıştır.

# Elasticsearch POC: Gelişmiş Profil Araması

## Problem 
Mevcut search, bio içeriğindeki bilgileri (kullanıcının belirttiği okul, takım, mezuniyet yılı vb.) yakalayamıyor. Örnek: "ITÜ '22" araması sonuç vermiyor.

## İş Etkisi
Elasticsearch'ün implementasyonu, kullanıcıların ortak bağlantıları hızla keşfetmesini sağlayabilir, platformdaki etkileşim potansiyelini artırabilir.
Bunun sonucunda aşağıda belirtilen durumlar ile olumlu anlamda karşılaşılabilir.

* Kullanıcılar aynı okul/mezuniyet yılı/takımdan kişileri keşfedebilir
* Konsept dışı farklı noktası olan kişileri bulma kolaylığı
* Superlike benzeri consumable harcamalarında olası artış
* Kullanıcı etkileşiminde potansiyel artış

## Metrikler
Elasticsearch ile gerçekleştirdiğimiz geliştirmeler, uygulamadaki kullanıcı davranışlarını  anlamamıza ve ölçümlememize olanak tanıyabilir.
* Bio bazlı aramaların artışı, 
* Arama sonrası kullanıcı etkileşimleri (user'lar arası etkileşim veya consumable harcaması) 
* Kullanıcı retention oranları 

**_Benzeri metriklerle bu iyileştirmelerin etkisini doğrudan gözlemleyebiliriz._**

Ayrıca, Elasticsearch'ün bize sunduğu **_Kibana Dashboard_** ile 
* Veri görselleştirme ve analiz süreçlerini daha verimli hale getirebiliriz. 
* Bio keyword trendlerini (örneğin en çok aranan okul veya takımlar)
* Kullanıcıların arama pattern'lerini,
* Başarılı match'lerin bio analizini 

**_Benzeri metrikleri takip edebiliriz._**

## Risk & Limitler
1. İlk olarak, **_initial data sync_** süresi beklentilerin üzerinde zaman alabilir ve bu süreçte performans kaybı yaşanabilir.
2. Ayrıca, **_peak saatlerde_** performans düşüşü olasılığı göz önünde bulundurulmalı ve sistemin ölçeklenebilirliği test edilmelidir.
3. Son olarak, yüksek veri hacmi nedeniyle **_storage büyüme hızı_**, uzun vadede maliyet ve yönetim zorlukları doğurabilir.


# Process
Biyografi verilerinin güncellenmesi ve loglanması sürecini açıklayarak Elasticsearch entegrasyonunu şu şekilde ele alabiliriz.

### 1. Biyografi Güncelleme Süreci
Kullanıcı biyografileri, sistemde güncelleme yapıldığında aşağıdaki işlemler gerçekleşir:

* Kullanıcı, bio'sunu set eder ve bu bio limitlere uygunluk açısından kontrol edilir 
* Biyografi, PostgreSQL’deki users tablosunda güncellenir. 
* Güncelleme bilgisi Redis’e yazılır ve WebSocket aracılığıyla servislere iletilir.
* BigQuery için bir log oluşturulur ve bu log Kafka üzerinden server-user-events topic’ine gönderilir.

### 2. Loglama Süreci

BigQuery’ye gönderilen `server-user-events` topic’inden `user_bio_changed` key’ine sahip mesajlar yakalanır ve PostgreSQL’deki `users_bio_log` tablosuna yazılır. Bu tablo, `user_id`, `bio`, `log_date` column'larına sahiptir.

### 3. Elasticsearch Entegrasyonu 

#### Loglanan Verilerin Elasticsearch’e Aktarılması

users_bio_log tablosundaki veriler, Elasticsearch’te bir index oluşturularak kullanılabilir. Bu süreçte

> Index adı: `user_bio_logs`
Fieldlar:
user_id (TEXT)
bio (TEXT)
log_date (TIMESTAMPT)

### Veri Aktarım Süreci
Burada postgreSQL'de bulunan dataların aktarıldığını düşündüğümüz bir senaryo üzerinden ilerliyoruz. Alternatif olarak, Kafka üzerinden gelen biyografi değişim logları doğrudan Elasticsearch’e aktarılabilir.\
Veya bir ETL (Extract, Transform, Load) script yazarak periyoduk olarak ES'e ekleyebiliriz
PostgreSQL tablosundaki veriler yine Elastic Stack bir tool olan Logstash ile postgresql'den datayı çekip elasticsearch'e aktarmak için kullanılabilir.

### ES'in Search Route'a Implementasyonu

Elasticsearch’i mevcut arama routelarına entegre etmek için önce Elasticsearch client’ı projeye eklenmeli ve PostgreSQL query'leri Elasticsearch query formatına dönüştürülürmeli
. Verilerin düzenli olarak Elasticsearch’e indexlenmesi için bir  servisi yazılmalı (kullanılmalı). Sonuçların Elasticsearch üzerinden döneceği yeni bir refactor planlaması ile search endpoint’in performansı optimize edilebilir.
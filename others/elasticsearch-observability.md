## Elasticsearch Observability

Observability, sistemlerde meydana gelebilecek anomalileri, istenmeyen durumları yakalamak ve nedenini saptamak için gereken bilgilere sahip olabilmeyi sağlamak.

Bu bilgileri elde etmedeki temel sorunlar; yeterli bilgiyi toplayamamak ve çok fazla bilgi toplamak ama yararlı hale getirememek.

Observability'nin 3 temeli:

* Metric
* Log
* Application Trace



<img style="float: left;" src="elasticsearch-observability/images/01_pillars.png">



ELK Stack (Elasticsearch, Logstash, Kibana)

Metricbeat -> For numerical time series database (daha öncesinde time series database)

APM (Application Performance Monitoring) -> Application tracing ve distributed tracing capabilities to the stack eklenmiş.



* Kibana: Elasticsearch'teki verileri çizelgeler ve grafiklerle görselleştirir

* Elasticsearch: Arama ve analiz motoru
* Beats: Sadece bir dosyayı takip etmek için lightweight, tek amaçlı bir veri aktarıcıları
* Logstash: Aynı anda birden fazla kaynaktan veri alan, dönüştüren ve ardından Elasticsearch gibi bir "stach"e gönderen, sunucu taraflı veri işleme pipeline'ı



<img style="float: left; zoom: 25%;" src="elasticsearch-observability/images/02_approach.png">



Kibana:

* ?Virtualize
* Dashboard
* ?Timelion
* ?Canvas
* Machine Learning
* Infrastructure
* Logs
* APM
* ?Graph
* ?Dev Tools
* ?Monitoring





https://www.elastic.co/beats/

* Filebeat
* Packetbeat
* Metricbeat
* Auditbeat
* Heartbeat



### Fleet Server

Agent policy'lerini güncellemek, durum bilgilerini toplamak ve Elastic Agent'ların genelinde eylemleri koordine etmek için kontrolcü görevi görür. Bu sayede ölçeklenebilir bir mimari sağlar. Fleet server'lar dağıtık olarak konumlandırılabilir.



<img style="float: left; zoom: 75%;" src="elasticsearch-observability/images/10_fleet_server_1.png">



### Observability

Log'ları, sistem metric'lerini, uptime verilerini ve uygulama trace'lerini tek bir yığın olarak eklemeye ve izlemeye olanak tanır.

Elde edilenler:

* Veri kaynaklarını eklemek ve yapılandırmak için merkezi bir yer.
* Her veri kaynağıyla ilgili analizleri gösteren çeşitli grafikler.
* Log'lar, Metric'ler, Uptime'lar ve APM uygulamalarında verileri incelemek ve analiz etmek.
* Hızlı bir şekilde çözülmesi gerekebilecek sorunlar hakkında bilgilendirici uyarı chart'ları.
* Kibana ile veri kaynaklarını eklemeye ve yapılandırmaya yardımcı olacak özellikler sağlar.



<img style="float: left; zoom: 25%;" src="elasticsearch-observability/images/09_doc_1.png">



#### Logs

Elasticsearch'e alınan tüm log'ları aramaya, filtrelemeye ve takip etmeye olanak tanır. Farklı sunucularda oturum açmak, dizinleri değiştirmek ve tek tek dosyaları takip etmek yerine tüm log'ları bir araya getirir.

Log anormalliklerini otomatik olarak algılamak ve günlük log'lardaki kalıpları hızlı bir şekilde belirlemek için log mesajlarını kategorilere ayırmaya yarayan makine öğrenimi de kullanılabilir.



<img style="float: left; zoom: 25%;" src="elasticsearch-observability/images/09_doc_2.png">



#### Metrics

Altyapı metric'lerini görselleştirerek sorunlu artışları teşhis etmeye, yüksek kaynak kullanımını belirlemeye, bölmeleri otomatik olarak keşfetmeye, izlemeye ve metrikleri Elasticsearch'teki log'lar ve APM verileriyle birleştirmeye olanak tanır.



<img style="float: left; zoom: 25%;" src="elasticsearch-observability/images/09_doc_3.png">



#### Uptime

Uygulama ve hizmetlerin kullanılabilirliğini ve yanıt sürelerini gerçek zamanlı olarak izlemeye ve sorunları, kullanıcıları etkilemeden önce tespit etmeyi sağlar. Network endpoint'lerin durumunu HTTP/S, TCP ve ICMP aracılığıyla izleyebilir, endpoint durumunu zaman içinde keşfedebilir, belirli monitörlerde detaya inebilir ve herhangi bir zamanda ortamı üst düzey anlık görüntüsünü görüntüleyebilir.



<img style="float: left; zoom: 25%;" src="elasticsearch-observability/images/09_doc_4.png">



#### APM

Yazılım hizmetlerini ve uygulamaları gerçek zamanlı olarak izlemeye, işlenmeyen hataları ve istisnaları toplamaya ve ana bilgisayar düzeyinde temel ölçümleri otomatik olarak almaya olanak tanır.



<img style="float: left; zoom: 25%;" src="elasticsearch-observability/images/09_doc_5.png">



##### Application Performance Monitoring (APM)

Microservice/monolith mimarilerde sorunun asıl nedenini trace'ler, log'lar ve metric'ler ile tespit eder.



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/03_apm_1.png">



<img style="float: left;" src="elasticsearch-observability/images/03_apm_2.png">



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/03_apm_3.png">



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/03_apm_4.png">



<img style="float: left;" src="elasticsearch-observability/images/03_apm_5.png">



###### Log Monitoring

Farklı kaynaklardan gelen log'ları dönüştürüp görüntülenmesini sağlar.



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/04_logm_1.png">



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/04_logm_2.png">



###### Infrastructure Monitoring

AWS, Microsoft Azure, Google Cloud, Azure, GCP, Kafka ve NGINX gibi platformlar dahil olmak üzere 200'den fazla entegrasyon desteğiyle altyapıyı izler. 



<img style="float: left; zoom: 35%;" src="elasticsearch-observability/images/05_infram_1.png">



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/05_infram_2.png">



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/05_infram_3.png">



###### Real User Monitoring

Uygulamanın end-user sistemlerinde nasıl performans gösterdiğini anlamak için verileri URL'e, işletim sistemine, tarayıcıya ve konuma göre analiz eder.



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/06_realuserm_1.png">



<img style="float: left; zoom: 114%;" src="elasticsearch-observability/images/06_realuserm_2.png">



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/06_realuserm_3.png">



###### Synthetic Monitoring

Web site performansına ve kullanılabilirliğine ilişkin sorunları önceden yakalar.



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/07_syntheticm_1.png">



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/07_syntheticm_2.png">



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/07_syntheticm_3.png">



<img style="float: left; zoom: 71%;" src="elasticsearch-observability/images/07_syntheticm_4.png">



###### Universal Profiling

Sistem performansını analiz etmeye yarar.



<img style="float: left; zoom: 95%;" src="elasticsearch-observability/images/08_uniprofilingm_1.png">



<img style="float: left; zoom: 95%;" src="elasticsearch-observability/images/08_uniprofilingm_2.png">



<img style="float: left; zoom: 79%;" src="elasticsearch-observability/images/08_uniprofilingm_3.png">




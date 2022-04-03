# Metadata

Yapılan tüm işlemlerin odağında veri vardır. Veri olmadan hiçbir işlemin anlamı olmaz, çünkü ortada bir çıktı oluşmaz. Veriyi doğru şekilde toparlamak, işlemek ve raporlamak gerekiyor. Bunun için de veriyi doğru şekilde yönetmek gerekiyor (`Yönetilen Veri`).

Veriyi tarif eden veriye `metadata` (üst veri) denir.

Örneğin, T.C. kimlik numarası verisinin 12345654320 olduğu değil de 11 hane olduğu, her hanede bir rakam olduğu, her Türk vatandaşının sahip olduğu bir bilgi olduğu, T.C. vatandaşlarını tekil olarak tanımlayan bir bilgi gibi veriye ilişkin niteleyici bilgilerin hepsi `metadata`'dır.

`Veri sözlüğü` verinin kendisiyle değil, verilerin tarif, tanım, ilişki, kaynak, kural gibi `metadata`'sı ile ilgilenir.

- Veri tabanında tablo tanımı, tabloda saklanan kayıtların `metadata`'larının bir tanımıdır. Tabloda 1 kayıt olması veya 1 milyon kayıt olması tablonun tanımını değiştirmez.
- Programlamada bir class tanımı object'lerin içinde tutulacak verilerin tipini, çokluğunu, alabileceği değerleri vb. belirler. Kaç object oluşturulduğu class'ın tanımını değiştirmez.

Bu örneklerde tablo, class tanımı `veri sözlüğü`nün ilgi alanıdır. Tablodaki kayıtların ne olduğu veya oluşturulan object'lerin içinde ne tutulduğu `veri sözlüğü`nü ilgilendirmez.

`Veri sözlüğü`, `metadata` verilerinin belirlenip, kayıt altına alındığı, sürekli gözden geçirilerek güncellenen bir dokümandır.



Kurumlarda bulunan birçok sistem farklı zamanlarda, farklı firma ya da kişiler tarafından geliştirilmektedir. Sistemlerin idame, birbirleriyle entegrasyonu ya da bu sistemlerin ürettiği verilerin raporlanması gibi konularda bazı sorunlar meydana gelmektedir.

1. __Entegrasyon zorluğu:__ Farklı bilgi sistemlerinden bilgiler çekip örneğin raporlamak için birleştirmek, eldeki bilginin ilgili kısımlarını bu bilgi sistemlerine aktarmak ya da tamamen yeni bir uygulama geliştirerek bu farklı sistemlerde yönetilen verileri tek bir arayüz üzerinden toplamak ve sunmak gerekebilir. Bu durumda sistemler birbirleriyle entegre edilmeli, bütün sistemlerin kullanacağı ortak bir dil oluşturulmalıdır. Bu ortak dil en azından veri tanımlarını ve verilerin alabilecekleri değer kümeleri ile anlamlarını içermelidir.

2. __Tekrarlı ve çelişen veriler:__ Bilgi sistemlerinin birbirlerinden bağımsız geliştirilmesi, çoğunlukla farklı tanımlarla aynı bilgilerin yönetilmesi sonucunu doğurur. Aynı kurallar işletilmediği zaman (örneğin bir sistemde zorunlu olan alan diğer sistemde boş bırakılabiliyorsa ya da bir sistemde doğrulama kuralı işletilen bir alan üzerinde diğer sistemde herhangi bir kural işletilmiyorsa) aynı veri farklı sistemlerde farklı değerlerle oluşabilir, saklanabilir. Bu durumda farklı sistemlerden toplanan verilerin bir araya getirilmesi mümkün olmayabilir.

3. __Kişilere ya da firmalara bağımlılık:__ Yazılım dokümanlarının yokluğu ya da geçerliliğini yitirmesi bilgi sistemlerindeki birçok verinin neden ve nasıl tutulduğuna ilişkin kuralların yazılım kodlarının içinde ya da yazılımcıların kafasında kalması sonucunu doğurur. Bu durum kurum açısından risk içermektedir.

4. __Veri envanteri yokluğu:__ Bilgi sistemleri genellikle diğer sistemlerden bağımsız geliştirildiği ve kendi başına belgelendirildiği için, kurum genelinde ne tür verilerin yönetildiği, hangi verilerin nerede hangi tanımlarla tutulduğu, bunların ne anlama geldikleri gibi bilgiler birçok belgeye ve ortama dağılmış durumdadır. Bu bilgiler tek ve bütünleşik bir ortamda bulunmadığından çoğu kurum hangi verilere sahip olduğunun ve bunlardan nasıl faydalı bilgiler çıkartabileceğinin farkında değildir.

5. __Bilgiye erişme güçlüğü:__ Eldeki verinin ne olduğu ve nerede, nasıl saklandığı bilinmediği için bilgiye erişimde de güçlükler yaşanmaktadır. Bilginin elde edilebilmesi için kullanılabilecek verilerin çalışmakta olan sistemler üzerinden keşfedilmesi, alabilecekleri değer kümelerinin tanımlanması, bu değerlerin anlamlarının belirlenmesi ve bunlardan çıkarılacak raporlarla bilgiye ulaşılması gerekmektedir.

6. __Bilgi çıkarımına hazır olmama:__ Veri Madenciliği için verilerin;

   a. Temizlenmesi: Tutarsız ve hatalı verileri elemek,

   b. Bütünleştirilmesi: Farklı veri kaynaklarından gelen verileri bir araya getirmek,

   c. Seçilmesi/Çıkarılması: Yapılacak olan analiz çalışması ile ilgili verilerin hangileri olduğunun belirlenerek bunların alınması,

   d. Dönüştürülmesi: Verinin kullanılabilecek tipe, formata, içeriğe dönüştürülmesi

   gerekir. Farklı bilgi sistemlerinde verilerin farklı tanım, içerik ve kurallarla saklanması, bütün bu adımlar için ayrı ayrı çalışmalar yapılmasını gerektirir.

Bütün bu sorunların çözümü için;

​	a. Verilerin ad, tanım, alabileceği değerler, bu değerlerin anlamları, iş kuralları, kısıtlar, kaynaklar, sahiplik vb. bilgileri ile tanımlı hale getirilmesi,

​	b. Tanımlamaların ortak ve merkezi bir noktadan erişilebilir hale getirilmesi,

​	c. Yeni geliştirilecek ya da bakımı yapılacak bilgi sistemlerinin bu tanımlara uygun olarak ele alınması,

​	d. Yeni sistem geliştirmeleri ya da güncellemelerin mutlaka bu tanımları beslemesi ve güncellemesi,

​	e. Veri tanımlarında yapılan güncellemelerin paydaşlara bildirilerek gerekiyorsa alt sistemlerin de bunlara göre güncellenmesi,

​	f. Bunlar için gerekli prosedürlerin tanımlanması ve kurum tarafından hayata geçirilmesi

gerekir. `Veri Sözlüğü` çalışmalar kapsamında veri tanımları kayıt altına alınarak yönetilmeye başlanır.

## Metadata

`Metadata` veriyi tarif eden veridir. Bir uygulamanın ürettiği, veri tabanında sakladığı ya da bir kanaldan sunduğu verilerin ayrıntılı tarifini, formatını, özelliklerini vb. belirtir. `Metadata` için “veriye anlam verir” ya da “veriyi bir bağlama oturtur” denilebilir. Örneğin 783.562 sayısı kendi başına bir veridir ancak adını, tanımını ve birimini belirtmeden anlamsızdır. Oysa bu veri hakkında aşağıdaki `metadata` ögeleri verildiğinde veri anlam kazanır ve bir bağlama oturur.

​	Adı: Türkiye’nin yüzölçümü

​	Tanımı: Türkiye Cumhuriyeti’nin topraklarının kapladığı alan

​	Birimi: km2

## Veri Sözlüğü

`Veri Sözlüğü` “bilgi sistemlerinde yönetilen verilere ilişkin `metadata`'ların tutulduğu, belirli kurallar çerçevesinde yönetildiği ve ilgili paydaşların erişimine açılan bir varlıktır” şeklinde tanımlanabilir. Burada bahsedilen paydaşlar yazılımcılar, veri tabanı uzmanları, entegrasyon sorumluları, rapor hazırlayanlar, gereksinim çözümleyiciler, veri bilimciler, veri madenciliği yapanlar, makine öğrenmesi ve yapay zeka uygulamaları ile uğraşanlar gibi birçok farklı rolden kişiler olabilir.

## Nesne Sınıfı

TS ISO/IEC 11179 standardında “*belirgin sınırlarla ve anlamla tanımlanan, nitelikleri ve davranışları aynı kurallara bağlı olan, gerçek dünyadaki düşünceler, soyutlamalar veya şeyler dizisi*” şeklinde tanımlanır. 

Bir başka tanım, “aynı nitelik, işlem, metot, ilişki ve mantıkları paylaşan nesneler kümesinin tanımıdır” şeklinde verilebilir. Bir sınıfın;

\- Sınıf adı gibi bir belirleyicisi,

\- Net bir nesne tanımı,

\- Nitelikleri,

\- İlişkili olduğu sınıflar ve bu ilişkilerin tarifi

vardır.

OOP (Object Oriented Programming) bağlamındaki sınıf tanımı ile örtüşmektedir.

## Nitelik

TS ISO/IEC 11179-3’te “bir `nesne sınıfı`nın tüm ögelerinde ortak olan `nitelik`” olarak tanımlanır.

Otomobil sınıfının bütün ögelerinin bir rengi vardır. Bunlardan bazıları siyah, bazıları beyaz, bazıları kırmızı ya da başka renklerde olabilir, ancak Otomobil sınıfının bütün ögelerinde ortak olan “renk” adlı bir `nitelik` bulunmaktadır.

`Nitelik`ler genellikle kendi alt bilgi alanı olmayan, atomik bilgilerdir. Örneğin “Personel” bir sınıf iken, personelin “Sicil Numarası” o sınıfın bir `niteliği`dir ve alt kırılımı yoktur.

OOP bağlamında düşünülürse primitive type değişkenlere, veri tabanı bağlamında düşünülürse tablonun kolonlarına karşılık gelir.

## Value Domain

`Value domain`, bir `niteliğin` alabileceği değerler kümesini ifade eder. İki türü vardır:

\- **Tarif Edilen Domain Value:** `Niteliğin` alabileceği değerlerin kümesi bir tarifle verilir. Örneğin “Personel” sınıfının “Yaş” `niteliği`nin alabileceği değer kümesi, “personelin doğum tarihinden günümüze kadar geçen yıl sayısı” olarak tarif edilebilir.  Birden çok tarif kullanılabilir. Örneğin “Yaş” `niteliği`nin `domain value`'su için “3 basamaklı pozitif tam sayılar kümesi” tarifi de yapılabilir.

\- **Kodlanmış Domain Value:** `Niteliğin` alabileceği değerlerin kümesi bir liste şeklinde verilir. Örneğin “Personel” sınıfının “Cinsiyet” `niteliği`nin alabileceği değer kümesi, “ERKEK, KADIN, BELİRTİLMEMİŞ” olarak belirtilir.

## Veri Elemanı Kavramı

TS ISO/IEC 11179-3’te “bir `niteliğin` bir `nesne sınıfı` ile bağlantılandırılmasına karşılık gelen kavram" şeklinde tanımlanır. `Veri Elemanları`nın kavramsal düzeydeki tanımıdır. Örneğin, “Personel” sınıfının “Cinsiyet” `niteliği` ile bağlantılandırılmasıyla “Personel Cinsiyeti” adlı `veri elemanı kavramı` oluşur.

## Veri Elemanı

`Veri sözlüğü`nün en temel ögesi `veri elemanı`dır ve çeşitli tanımlar yapılabilir:

\- temel veri taşıyıcısına `veri elemanı` denir

\- verinin düzenlenmesinde bağlam içerisinde bölünemez olarak kabul edilen veri birimi

`Veri elemanı`, bir `veri elemanı kavramı`nın bir `domain value` ile eşleştirilmesi ile tanımlanır.

## Veri Seti

`Veri seti` (veri kümesi), toplu haldeki bir grup veriye verilen isimdir. Örneğin, “2020 yılı yaz aylarında İç Anadolu Bölgesi illere göre ortalama sıcaklık değerleri”ni içeren bir excel ya da CSV dosyasına `veri seti` denilebilir.

Örneğin personel bilgilerini sunan bir web servis, personelin sicil numarası, adı, soyadı gibi bilgilerini tek tek döndürmek yerine bu bilgileri içeren personel nesnesini döndürür. `Veri setleri` ile çalışılıyor olması, `veri sözlüğü` hazırlanırken de benzer mantığın korunmasını ve saklanan, raporlanan ya da transfer edilen `veri setleri`nin `metadata`sının tanımlı olmasını gerektirir. Verilere ilişkin `metadata`lar `veri elemanları` ile tanımlanmaktadır. `Veri setleri` ise bu verilerin bir kümesidir ve bu kümedeki verilere ait `veri Elemanları`nın hangisinin zorunlu ya da isteğe bağlı olduğu, tek mi birden fazla mı olduğu, aralarında ne gibi ilişkiler bulunduğu gibi tanımlar, `veri seti`nin `metadata`larıdır. “Bir projede çalışan personele ilişkin hangi verilerin bulunduğu” bilgisi (bu örnek için: sicil numarası, ad, soyad, projedeki görevi) ya da “bir personelin projede birden fazla görev alıp alamayacağı” kuralı `metadata`lara örnektir. Bu gibi `metadata`ların tanımlandığı ögeye `veri seti` tanımı adı verilir.

### Minimum Veri Seti

Saklanması/toplanması istenen ya da gereken verilere ilişkin, toplanacak minimum `metadata` kümesidir. 

## Metadata Deposu ve Metadata Kayıt Kütüğü

`Metadata`ların toplanması, saklanması ve dağıtılması amacıyla oluşturulan veri tabanlarıdır. `Depo` sadece `metadata`ların herhangi bir kuraldan bağımsız saklandığı izlenimini verirken, `kayıt kütüğü` ile bir kayıt işlemi olduğu, kayıt işleminin de bir takım kurallar çerçevesinde yapıldığı akla gelmektedir.

## Ana Veri Yönetimi

Kurumun kritik verileri için tek bir ana referans kaynak oluşturularak verilerin tek biçimliliği, doğruluğu, sorumluluğu, anlamsal tutarlılığı ve hesap verebilirliğini sağlamayı hedefler. Bunun için verilerin temizlenmesi, dönüştürülmesi ve bütünleştirilmesi gerekir. `Ana Veri Yönetimi`, verilerin belirlenmesi, toplanması, dönüştürülmesi ve onarılması için gerekli süreçleri tanımlar. `Metadata`'ları değil, verinin kendisini yönetir.



## Veri Sözlüğü Yazılımı

- `Metadata` Kayıt Defteri uygulamasıdır
- `Veri Sözlüğü` çıktısı üretebilir
- TS ISO/IEC 11179 standardını temel alır
- Veri girişlerini kolaylaştırır
- Süreçleri işletmeyi kolaylaştırır
- Herkes kendi yetki tanım alanında çalışır



`Veri sözlüğü`nde hangi `metadata`'ların saklanacağına karar verilip, bununla ilgili bir kılavuz hazırlanır.

Kurumun sahip olduğu kaynaklar kullanılarak `metadata`'lar toplanır. Bu kaynaklar veri tabanları, web servisler, uygulama arayüzleri, raporları, kurumun kanun, tüzük, yönetmelik vb. dokümanları, web sitesi... ve personelden oluşur. Kaynakların içindeki `metadata`'lar çeşitli yöntemlerle çıkarılarak `veri sözlüğü`ne kaydedilir. Kaydetme işi sırasında bir takım kurallara uyulması gerekir. Bu kurallar verinin nasıl tanımlanacağı, nasıl adlandırılacağı, hangi `metadata`'ların' tanımlanması gerektiği, yazım kuralları gibi kurallardır. Kaydetme işi sırasında `veri sözlüğü` yazılımı kullanılır.

Kurumun `veri sözlüğü` hazırlanıp kenara kaldırılmaz. `Veri Sözlüğü` yaşayan bir varlıktır. Bu nedenle sürekli gözden geçirme ve güncellemelerle üzerinden geçilir, güncel tutulur.

### Roller

Kurumlarda, `Veri Sözlüğü` çalışmalarında birkaç rol bulunmaktadır.

- Veri Sözlüğü Kurum Koordinatörü
- Veri Sözlüğü Yazılımı Kurum Yöneticisi
- Veri Sözlüğü Kurum Eğiticisi
- Öneren/Sorumlu
- Kayıt Otoritesi
- Kontrol Komitesi Üyeleri
- Danışma Komisyonu Üyeleri

Ekipler devingen olabilir. Kurumda çalışma alanı değiştikçe ekibe yeni üyeler katılabilir, danışma komisyonları kurulabilir.



## Metadata Alanları

### Bağlam

Ögenin ne amaçla kullanıldığını belirler. Örneğin *Birim*, organizasyon şeması bağlamında yönetimsel bir varlığı ifade ederken, ölçme bağlamında ölçü birimini ifade edebilir. `Metadata` Yönetim Ögeleri için en az bir tanım yapılması, bu tanımın da bağlamının belirtilmesi zorunludur. `Veri Sözlüğü` Yazılımı, veri girişlerini kolaylaştırmak ve hızlandırmak için varsayılan bir bağlam oluşturur ve bunu kullanır. Kullanıcıların, gerekliyse bu varsayılan bağlam tanımını güncellemesi ya da kendi yapacakları tanımlar için yeni bağlamlar oluşturmaları beklenir.

### Ad

Her ögenin adı olmalıdır ve bu ad Genel Kurallar ve Adlandırma Kuralları çerçevesinde verilmelidir. Ad alanı her öge için zorunludur. `Veri seti tanımı` ve `veri elemanı` ögeleri, yalnızca ad girilerek kaydedilebilir.

### Tanım

Ögenin ne olduğunu tarif eder. Tanımlar yapılırken Genel Kurallar ve Tanımlama Kuralları dikkate alınmalı, açık, net, belirsizliğe yer bırakmayan, herkesçe aynı anlaşılan tanımlar yapmaya dikkat edilmelidir.

İlk kayıt sırasında zorunlu olmamakla birlikte, ögenin onay süreçlerinde ilerletilebilmesi için girilmesi zorunlu olan bir alandır. Genel eğilimin bu zorunluluğu yerine getirmek üzere *Tanım* alanına bir şeyler yazmak, bunun için de çoğu zaman ögenin adını kopyalayıp bu alana yapıştırmak şeklinde olduğu gözlemlenmiştir ancak bu yaklaşım yanlıştır. Tanım, ögeye ilişkin en önemli bilgiyi, o ögenin *ne* *olduğunu* ifade ettiği için titizlikle ele alınmalı, Kayıt Otoritesi, Kontrol Komitesi ya da Danışma Komisyonu üyeleri tarafından da bu bakışla mutlaka incelenmelidir.



------



### Kurum

Ögenin hangi kurumntarafından önerildiği bilgisidir. Üzerinde çalışılan `veri sözlüğü`nün sahibi olan kurumdur.

### Veri Sözlüğü

Ögenin ait olduğu `veri sözlüğü`nün adıdır. Veri girişi yapmak üzere seçilen `veri sözlüğü`dür.

### Oluşturma Tarihi

Ögenin ilk kaydedildiği tarihtir. İlk kaydetme bir versiyon oluşturduğu için, aynı zamanda `versiyon tarihi`dir.

### Son Değişiklik Tarihi

Ögenin üzerinde güncelleme yapılarak tekrar kaydedildiği son tarihtir.

### Versiyon

Bir öge ilk kaydedildiğinde v1 versiyon numarası ile kaydedilir. Daha sonra versiyonlama süreci ile bu ögenin yeni versiyonları oluşturulabilir. Versiyon numaraları `veri sözlüğü` yazılımı tarafından arttırılır.

### Kayıt Durumu

Ögenin `veri sözlüğü`ndeki kayıt durumunu ifade eder.

### Talep Edilen Durum

Öge bir başka kayıt durumuna geçirilmek talebi ile Kayıt Otoritesine gönderilmişse, bu talep edilen durumun ne olduğu bilgisidir.

### Tanımlayıcı

Bir ögeyi `veri sözlüğü`nde biricik olarak tanımlayan bilgidir. Uluslararası olarak biricikliği sağlayacak özel bir formatta oluşturulabilirse de standart bunu zorlamaz. `veri sözlüğü` yazılımı, tanımlayıcı değer için biricik bir değer üretir ve bunu kullanır.

### Etiketler

Arama yapılırken bu ögeye erişilmesini sağlayacak anahtar sözcükler boşlukla ayrılarak bu alana yazılır. Anahtar sözcükler; ögenin eş ya da yakın anlamlıları, kurumda aynı varlığı ifade eden terimler ve kısaltmalar, İngilizce adlar, bu ögeyi çağrıştıran sözcükler olabilir.

### Gizlilik

Bir öge gizli olarak işaretlenirse, `veri sözlüğü` katalog çıktısında bu öge gösterilmez, yalnızca Kuruma Özel Katalog içinde yer alır. Genel olarak `metadata` yönetim ögelerinin gizli olması beklenmez. Çünkü bu ögeler veriyi değil, `metadata`'yı' tutmaktadır. Örneğin T.C. Kimlik Numarası veri elemanı, içerisinde belirli bir kimsenin kimlik numarasını değil, genel olarak kimlik numaralarının 11 hane olduğu, her hanenin bir rakam olduğu, kimlik numarasının kişileri biricik olarak tanımladığı gibi `metadata`'larını' tutar. Dolayısıyla T.C. Kimlik Numarası veri elemanının gizli olması anlamlı değildir. Bununla birlikte, askeri kurumlar başta olmak üzere kimi kurumlarda bazı ögelerin varlığının dahi bilinmesi istenmeyebilir. Böyle durumlarda o ögelerin Gizli tanımlanması gerekir.



------



### Bağlam

Tercih Edilen Tanım - Bağlam ile aynı

### Bağlamdaki Ad

Tercih Edilen Tanım - Ad ile aynı

### Bağlamdaki Tanım

Tercih Edilen Tanım - Tanım ile aynı

### Dil

Bu tanımın kullandığı dil

### Kabul Edilebilirlik

Tanımın kurumdaki kabul edilebilirlik düzeyini belirtir. Alabileceği değerler:

- Kabul Edilen: Genelde bu girilir.
- Kullanımdan Kaldırılan
- Süresi Dolmuş
- Geçersiz Kılınmış



------



### Tanımlayıcı

Referansı biricik olarak belirler. Veri Sözlüğü Yazılımı tarafından oluşturulur. Kullanıcı isterse değiştirebilir.

### Başlık

Referansa ilişkin başlık bilgisidir. Referansın adı olarak da düşünülebilir.

### Dil

Referansın dilidir.

### Tip

Referansın tipidir. Alabileceği değerler:

Tüzük, Yönetmelik, Genelge, Kanun, KHK, Rapor, Kişi, Internet, Kitap, Dergi, Gazete, Makale, Uygulama Arayüzü, Web Servis, Veri Tabanı

### URI

Referansa ulaşılabilecek web adresidir.

### Ek Açıklama

Referansa ilişkin ek açıklamalar yapılabilir.



------



### Menşei

Ögenin kaynağı metin olarak belirtilir.

### Ek Açıklamalar

Ögeye ilişkin açıklayıcı yorumlar metin olarak belirtilir.

### Toplama Metotları

`Metadata`'ları tanımlanan verinin nasıl toplandığı açıklanır. Örneğin veriler bir web servis üzerinden başka kurumlarca gönderiliyor, bir uygulama arayüzü üzerinden belirli bir raporun belirli bir alanındaki veri ya da bir müşterinin beyan ettiği değerler giriliyor veya yazılım tarafından otomatik üretiliyor olabilir.

### Kullanım Kılavuzu

`Metadata` alanlarının ait olduğu verinin nasıl ve hangi kurallara göre kullanılacağı, verinin nasıl yorumlanması gerektiği açıklanır. Örneğin bir `kodlanmış domain value` ile ilişkilendirilmiş bir `veri elemanı`nın sakladığı verinin ne anlama geldiğinin bulunabilmesi için `kodlanmış domain value`'nun nasıl kullanılacağı anlatılabilir ya da gün ve ay bilgisi belirtilmeyen doğum tarihi verilerinde gün için *01*, ay için *Ocak* değerlerinin verilmesi gerektiği belirtilebilir.



------



### Öznitelikler

Herhangi bir kurum, oluşturduğu `veri sözlüğü`nde TS ISO/IEC 11179 standardının tanımladığı `metadata` alanları dışında bir `metadata` saklamak isteyebilir. `Veri sözlüğü` yazılımında böyle bir gereksinimi karşılamak üzere `öznitelik` tanımlama olanağı bulunmaktadır.



### İlişkiler

`Veri seti tanımı` ile `veri elemanı` ya da başka `veri seti tanımları` arasında ilişki tanımlanabilir.

### Varlık Durumu

`Veri seti tanımı` için oluşturulan bir nesnenin içinde, bu `veri elemanı`nın/`veri seti tanımı`nın varlık durumunu ifade eder. Alabileceği değerler:

- Zorunlu
- İsteğe Bağlı
- Koşullu: Bu `veri elemanı`nın değerinin ancak belli koşullarda zorunlu olduğunu ifade eder. Örneğin *yalnızca* *erkek* *olan* *personel* *için* *askerlik* *durumu* *bilgisi* *girilir* koşulu, *Personel* *Askerlik* *Durumu* `veri elemanı`nın varlığını *Personel* *Cinsiyeti*  `veri elemanı`nın değerine bağlamaktadır.



### Çokluk Durumu

`Veri seti tanımı` için oluşturulan bir nesnenin içinde, bu `veri elemanı`ndan/`veri seti tanımı`ndan bir tane mi, daha fazla mı bulunabileceğini belirtir. Alabileceği değerler:

- Tekil
- Çoğul



Örneğin *Personel*  Veri Seti Tanımı için *Personel* *Doğum* *Tarihi*  Veri Elemanı için Çokluk Durumu değeri *Tekil*  olmalıdır. Yani *bir* *personelin* *tek* *bir* *doğum* *tarihi* *vardır*. Ancak *Personel* *Telefon* *Numarası*  için Çokluk Durumu *Çoğul* olabilir. Bu durumda *bir* *personelin* *birden* *çok* *telefon* *numarası* *olabilir* demektir.



### Ek Açıklama

İlişkiye dair belirtilmek istenen başka konular varsa metin olarak yazılabilir.

### İş Kuralları

Bu Veri Elemanına ilişkin her türlü iş kuralı yazılabilir.

### İlişki Öznitelikleri

Veri Seti Tanımı ya da Veri Elemanlarına olduğu gibi ilişkilere de yeni `metadata` alanları eklenebilir.



### Seçilen Veri Seti Tanımı

Üzerinde ilişki tanımlanan Veri Seti Tanımı kaynak, ilişkinin diğer ucundaki Veri Seti Tanımı ise hedef olarak adlandırılabilir. Kullanıcı ilişkiyi oluştururken önce hedef Veri Seti Tanımını seçer. Seçim yapılması zorunludur.

Veri Seti Tanımı seçildikten sonra, o tanımın altındaki Veri Elemanları görüntülenerek, Veri Elemanlarının hangilerinin bu ilişki için önemli olduğunun da ayrıca belirtilebilmesine olanak sağlanır. Veri Elemanlarının belirtilmesi zorunlu olmamakla birlikte, ilişkinin ilgi alanında nelerin olduğunu dokümante etmek açısından oldukça faydalı olabilir:

Bir Projenin

- Proje Adı
- Proje Tanımı
- Proje Başlangıç Tarihi
- Proje Bütçesi
- Proje Yöneticisi
- … 



gibi `veri eleman`ları bulunabilir. Personel – Proje ilişkisinde bu bilgilerden sadece *Proje* *Adı*, *Proje* *Tanımı* ve *Proje* *Başlangıç* *Tarihi* bilgileri yeterli olabilirken; örneğin bir birimde yürütülen projeler için Birim – Proje ilişkisinde *Proje* *Bütçesi*, *Proje* *Yöneticisi* bilgilerine de ihtiyaç olabilir.

İlişkide hangi Veri Elemanlarına ihtiyaç duyulduğunun ilişki tanımlanırken belirtilmesi, daha sonra yazılım geliştirme ya da raporlama aşamalarında fayda sağlayabilir.

### İlişki Adı

İlişkinin adı girilir. Örneğin *her* *personel* *mutlaka* *en* *az* *bir* *projede* **görev** **alır** önermesi için İlişki Adı alanına *Görev* *Aldığı* *Projeler* girilebilir. Girilmesi zorunlu alandır.

### İlişki Tanımı

İlişkiyi tarif eden, ilişkinin ne olduğu belirten ifadeler girilir. Girilmesi zorunlu alandır.



------



`Domain value`, bir `niteliğin` alabileceği değerler kümesini ifade eder. İki türü vardır:

### Tarif Edilen Domain Value

Niteliğin alabileceği değerlerin kümesi bir tarifle verilir. Örneğin *Personel* sınıfının *Yaş* niteliğinin alabileceği değer kümesi, *personelin* *doğum* *tarihinden* *günümüze* *kadar* *geçen* *yıl* *sayısı* olarak tarif edilebilir.

Birden çok tarif kullanılabilir. Örneğin *Yaş* niteliğinin değer etki alanı için *3* *basamaklı* *pozitif* *tam* *sayılar* *kümesi* tarifi de yapılabilir.



### Kodlanmış Domain Value

Niteliğin alabileceği değerlerin kümesi bir liste şeklinde verilir. Örneğin *Personel* sınıfının *Cinsiyet* niteliğinin alabileceği değer kümesi, *ERKEK, KADIN, BELİRTİLMEMİŞ* olarak belirtilir.



Veri Elemanları tanımlanırken, hangi `domain value`'ndan değer alabileceği belirtilmek zorundadır.



## Kaynaklar ve Kullanımları

### SRS ve SDD

Yazılım Gereksinim Belgeleri ve Yazılım Tasarım Dokümanları ile ilgili iki genel sorun bulunmaktadır:

- Belgenin bulunamaması: Özellikle eski sistemlerin gereksinim belgeleri zaman içinde dosyalar arasında kaybolabilmektedir. Bazen de özellikle küçük projeler için gereksinim belgesi hiç hazırlanmadan yazılım geliştirilmektedir.
- Belgenin güncel olmaması: Birçok yazılım projesi üzerinde analiz ve tasarım süreçlerinden sonra da güncellemeler yapılır. Bu güncellemelerden bazıları arayüzdeki düzen, format, renk gibi detaylarla ilgili olsa da, çoğu zaman analiz sırasında ortaya çıkmamış gereksinimleri karşılamak üzere yeni özellikler eklenir ya da hatalı olduğu fark edilen maddeler düzeltilir. Belgeler, bu yeni özellikler ve düzeltmeler için nadiren güncellenir. Dolayısıyla zaman içinde belge gittikçe eskiyebilir.



Kullanım:

1. İsim olan sözcükler Veri Seti Tanımı ya da Veri Elemanı adaylarını belirler.
2. İsim olan sözcüklerin altı çizilir ve bir dosyaya bu isimler alt alta yazılır. Daha önce yazılmış bir isim tekrar yazılmaz. Eğer bir isim alt ögeleri olan bir varlığın ismiyse Veri Seti Tanımı, daha küçük parçası olamayacak bir varlığın ya da varlık özelliğinin ismiyse Veri Elemanı adayı olarak belirlenir.
3. Veri Seti Tanımı ve Veri Elemanı adaylarının üzerinden tekrar geçilerek, birbirleri ile ilişkileri belirlenmeye çalışılır. Öncesinde Veri Elemanı adayı olarak belirlenmiş bir sözcüğe ait alt alanlar olduğu fark edilerek bu Veri Seti Tanımına ya da benzer şekilde bir Veri Seti Tanımı bir Veri Elemanına dönüştürülebilir.
4. Aynı ya da yakın anlamlı isimler birbirlerinin yanına not edilerek gruplanır. Gruptan yalnızca bir tane Veri Seti Tanımı ya da Veri Elemanı adayı belirlenir, diğer isimler bu ögenin Etiketler üstveri alanına girilmek üzere kullanılır.
5. Fiil cümleleri yapılabilecek işlemleri anlatır. Dolayısıyla İş Kuralları, Toplama Metotları ya da Kullanım Kılavuzu gibi üstveri alanları için kullanılabilir.
6. Tasarım dokümanında UML Sınıf Diyagramları (Class Diagram) bulunabilir. Bu diyagramdaki sınıflar Veri Seti Tanımı, nitelikler Veri Elemanlarını oluşturmak için kullanılabilir. Sınıflar arası ilişkiler ve bu ilişkilerin nitelikleri Veri Seti Tanımları arasındaki ilişkileri belirler. Ayrıca niteliklerin veri tipleri, alabilecekleri değerler ve kısıtlamalar bu diyagramda bulunabilir.
7. Tasarım dokümanında Varlık-Bağıntı Diyagramları (Entity-Relationship Diagram – ER Diagram) bulunabilir. Varlık-Bağıntı Diyagramları veri tabanını tarif ettiği için, ilgili ipuçları kullanılabilir.

### Kullanıcı Arayüzleri ve Raporlar

Kullanıcıların halen kullanmakta oldukları bilgi sistemlerinin kullanıcı arayüzleri, Veri Sözlüğü çalışmalarında başvurulacak önemli kaynaklardan biridir. Gereksinim ve tasarım belgeleri her zaman bulunamasa da arayüzler her zaman erişilebilir durumdadır. Ayrıca güncellemeler doğrudan yazılım üzerinde yapıldığı için belgeler güncelliğini yitirse bile kullanıcı arayüzleri her zaman günceldir. Bu nedenle;

Kullanıcı arayüzü ya da raporlar ile belgeler arasında uyuşmazlık olduğu zaman kullanıcı arayüzü doğru kabul edilmelidir.



Kullanım:

1. Veri giriş alanlarının hemen hepsi bir Veri Elemanı için oluşturulmuştur. Dolayısıyla Veri Elemanlarının belirlenmesi için veri giriş alanları kullanılabilir.
2. Arayüzlerde yönlendirme ya da açıklamalar bulunabilir. Bunlar Veri Elemanlarının Tanım, Ek Açıklamalar ya da İş Kuralı gibi üstveri alanları için kullanılabilir.
3. Veri giriş alanları sıklıkla bir takım nesnelerin alanları şeklindedir. Örneğin kullanıcı ekranda "Yeni Kayıt" tuşuna tıklar ve bir personelin bilgilerini girerek kaydeder. Buna benzer arayüzler Veri Seti Tanımlarının belirlenmesine yardımcı olabilir.
4. Ekranlardaki combobox, listbox, radiobutton, checkbox bileşenleri, Veri Elemanlarının alabilecekleri değer kümelerini oluşturduğundan `domain value` için kullanılabilir.  
5. Veri giriş alanlarına bakılarak veri tipleri bulunabilir.
6. Veri tabanı ya da belgelerde belirtilmemiş alan büyüklükleri arayüzlerde belirtilmiş olabilir. Örneğin çoğu zaman metin kutularının altında, o kutuya kaç karakter yazılabileceğini gösteren sayılar bulunur, numerik değerler için virgülden önce ya da sonra kaç hane ayrıldığı görülebilir.
7. Veri tabanı ya da belgelerde bulunamayan format bilgileri arayüzlerde belirtilmiş olabilir. Örneğin bir telefon numarası ya da tarih giriş alanında çoğu zaman o bilginin nasıl girilmesi gerektiğini gösteren format bilgisi ya da otomatik formatlama yeteneği vardır.

### Web Servisler

Kullanımı:

1. Web servisin adı ve açıklaması ile metotlarının, parametrelerinin ya da döndürdüğü değerlerin içinde geçen ve isim olan sözcükler Veri Seti Tanımı ya da Veri Elemanı adaylarını belirler.

2. Açıklamalar, ilgili ögelerin Tanım, Ek Açıklama ya da İş Kuralı gibi üstveri alanları için kullanılabilir.

3. Parametreler bazen doğrudan doğruya bir Veri Seti Tanımına ya da Veri Elemanına karşılık gelir. Bunlardan ilgili ögelerin adı ve veri tipi çıkartılabilir.

4. Metotların döndürdükleri değerler bazen doğrudan doğruya bir Veri Seti Tanımına ya da Veri Elemanına karşılık gelir. Bunlardan ilgili ögelerin adı ve veri tipi çıkartılabilir.

5. Web servislerin tanım dokümanları (WSDL, WADL, Swagger, XSD) içinden aşağıdaki üstveri alanları çıkartılabilir:

   - Veri Seti Tanımları ya da Veri Elemanlarının Adı

   - Veri Seti Tanımları ya da Veri Elemanlarının Tanımı

   - Veri Seti Tanımları ile Veri Elemanları ya da diğer Veri Seti Tanımları arasındaki ilişkiler ve bu ilişkilere ait Varlık Durumu ve Çokluk Durumu

   - Veri tipi

   - Alan büyüklüğü

   - Alabileceği değer kümesi

   - Doğrulama kuralları

### Excel, PDF, XML

Birçok kurumda bazı listeler excel dosyaları başta olmak üzere çeşitli dosya formatlarında saklanmaktadır. Bu dosyalardan özellikle Veri Elemanları, bunların veri tipleri, alabilecekleri değerler ve kısıtlar çıkarılabilir. Ayrıca bazı excel dosyalarında formüller ve hatta formlar bulunmakta, bu dosyalar birçok hesaplamanın yapıldığı bir bilgi sistemi gibi kullanılmaktadır. Bu durumda dosyalardan iş kuralları gibi üstverilerin de çıkarılması mümkündür.



------



`Metadata` Yönetim Ögelerinin adları birkaç bölümden oluşur:

1. Nesne Sınıfı Adı terimleri: Programlamadaki sınıf adı, veri tabanındaki tablo adı, UML diyagramındaki sınıf adı gibi düşünülebilir. UVSS bağlamında Nesne Sınıfı Adları, Veri Seti Tanımı adları ile aynıdır.

   Örnek: **Çalışan** Adı

2. Nitelik Adı terimleri: Programlamadaki nitelik/özellik (*attribute/property*) adı, veri tabanındaki kolon adı gibi düşünülebilir.

   Örnek: Çalışan **Adı**

3. Gösterim terimleri: Gösterimin nasıl yapılacağını kategorize eder. Örneğin Ad, Miktar, Ölçü, Sayı, Adet, Metin gibi bilgiler, verinin türü hakkında bilgi verir.

   Örnek: Ağaç Yükseklik **Ölçüsü**

4. Niteleyici terimler: Veri elemanlarını farklılaştırmak için kullanılır.

   Örnek: Maliyet **Bütçe** **Dönemi** Toplam Miktarı



Bu bilgiler ışığında, uyulması gereken adlandırma kuralları şöyle özetlenebilir:

**Anlamsal (Semantic) kurallar:** İsim bölümlerinin anlamlarıyla ilgilenir.

**Sözdizimsel (Syntactic) kurallar:** İsmi oluşturan bölümlerin dizilişleriyle ilgilenir. İki şekilde düşünülebilir:

- Göreli: Bölümler, diğer bölümlere göreli olarak yerleştirilir. Örnek: *Sınıf ismi, Özellik isminden önce gelir*.

- Mutlak: Bölümlerin yeri önceden bellidir. Örnek: *Özellik her zaman en sonra yazılır*.

**Sözcüksel (Lexical) kurallar:** İsimlerin var olup olmamasıyla ilgilenir: tercih edilen/edilmeyen terimler, eş anlamlılar, kısaltmalar, bölümün uzunluğu, heceleme, izin verilen karakter kümesi, büyük/küçük harf kullanımı vb.



## Süreçler

Veri Sözlüğü hazırlama işi veri girişinden ibaret değildir. Üstverilerin incelenmesi, varsa hataların düzeltilip eksikliklerin giderilmesi, güncellemelerin kontrollü bir şekilde yapılarak değişikliklerin kayıt altına alınması, veri elemanlarının farklı versiyonları varsa bunların yönetilmesi, veri elemanlarının olgunluk seviyelerinin arttırılması gibi bir grup işin yürütülmesi gerekir. Bu işler, farklı rollere sahip bir grup paydaş tarafından yapılır.

### Veri Seti Tanımı oluşturma

Veri Elemanlarının mutlaka bir Veri Seti Tanımının altına girilmesi gerekmektedir. Bu nedenle önce Veri Seti Tanımı kaydedilmelidir.

1. Bir Veri Seti Tanımını kaydedebilmek için yalnızca adının girilmesi yeterlidir. Böylece henüz detayları ortaya çıkartılamamış ama var olduğu belirlenmiş ögeler kaydedilebilir. Veri Seti Tanımının adı genelde Nesne Sınıfının adı ile aynıdır. Bununla birlikte, Veri Seti Tanımının adı girilirken Genel Kurallar ve Adlandırma Kuralları dikkate alınmalı, gerekirse ad üzerlerinde değişiklikler yapılarak kurallara uygun hale getirilmelidir. Veri Sözlüğü için girilmesi zorunlu olan alanlar varsayılan değerlerle Veri Sözlüğü Yazılımı tarafından oluşturulur, istenirse kullanıcı tarafından güncellenebilir.
2. Veri Seti Tanımının *Tanım* alanının değeri girilirken Tanımlama Kuralları dikkate alınmalıdır. Ayrıca eğer mümkünse bu tanımlar bir referansa (tüzük, yönetmelik, kitap, web sayfası vb.) dayandırılmalıdır.
3. Veri Seti Tanımının kimlik bilgilerinin çoğu Veri Sözlüğü Yazılımı tarafından oluşturulur, bunlar için veri girişi yapılması beklenmez. Bu kısımda girilebilecek iki bilgi vardır. Bunlardan ilki olan *Etiketler* alanına, ögenin benzer, yakın, eş anlamlı sözcükleri ile bu ögeyi bulmak üzere kullanılabilecek anahtar sözcükler boşlukla ayrılarak girilebilir. *Gizli* kutusu, eğer bu öge gerçekten de başka hiç kimse tarafından görülmemesi gereken üstveriler içeriyorsa işaretlenmelidir. Çünkü Veri Sözlüğü hazırlamakta amaç zaten verilerin üstverilerini tanımlı ve herkesçe erişilebilir hale getirmek olduğundan, bunların erişime kapatılması doğru bir yaklaşım değildir.
4. Veri Seti Tanımı için varsa *Diğer* *Tanımlar* girilir. Her bir tanımlama için Bağlam, Bağlamdaki Ad, Bağlamdaki Tanım, Dil ve Kabul Edilebilirlik alanlarının değerleri girilir. Yeni yapılan tanımlamalarda Kabul Edilebilirlik değeri olarak, *Kabul* *Edilen* değerinin (varsayılan değer) seçilmesi genellikle doğru yaklaşımdır. Bununla birlikte, artık kullanılmamasına karar verilmiş olan ama kurum içinde halen bilinen ve az da olsa kullanılan tanımlar varsa, bunların da uygun Kabul Edilebilirlik değeri ile girilmesinde fayda vardır. Bazı ögelerin farklı dillerdeki ad ve/veya tanımları bulunur. Bunların girilebileceği doğru yer de Diğer Tanımlar alanıdır.
5. Veri Seti Tanımının belirlenmesi ya da tanımlanması sırasında kullanılan referans olabilecek kaynaklar varsa bunlar girilir.
6. Varsa ek bilgiler girilir.
7. Öznitelikler Veri Sözlüğü ögelerine yeni üstveri alanları eklemek üzere tanımlandıkları için, ancak kurum bazında önceden belirlenmiş ve tanımlanmışlarsa kullanılabilirler. Ön tanımlı öznitelikler varsa bunlar zaten öge oluşturulurken Veri Sözlüğü Yazılımı tarafından üstveri alanı olarak kullanıcının veri girişine hazır hale getirilir. Kullanıcının Veri Sözlüğü ögesine veri girişi sırasında ekleyebileceği öznitelikler önceden oluşturulmuş olan ve ön tanımlı olarak işaretlenmemiş olanlardır.
8. Veri Seti Tanımı kaydedilir.

### Veri Elemanı oluşturma

Veri Elemanı, bir Veri Seti Tanımının Alt İlişkisi olarak oluşturulur. Veri Elemanının üstverileri ile birlikte ilişkinin de üstverileri girilir (oluşturulan ilişkinin Varlık Durumu alanı için *İsteğe* *Bağlı*, Çokluk Durumu alanı için ise *Tekil* değerinin varsayılan değer olarak atandığına dikkat edilmelidir).

1. Veri Elemanının adı genelde Nitelik adı ile aynıdır. Bununla birlikte, Veri Elemanının adı girilirken Genel Kurallar ve Adlandırma Kuralları dikkate alınmalı, gerekirse ad üzerinde değişiklikler yapılarak kurallara uygun hale getirilmelidir. Veri Elemanının kaydedilebilmesi için Ad alanının girilmesi yeterlidir. Ancak süreçte ilerletebilmek için diğer üstveri alanlarının da girilmesi gerekir.
2. Veri Elemanı ile Veri Seti Tanımının ortak olan üstveri alanları için veri girişinde dikkat edilmesi gereken noktalar aynıdır. Dolayısıyla Veri Seti Tanımı Oluşturma bölümündeki 2-7 maddeleri, Veri Elemanları için de geçerlidir.
3. Veri Elemanı için Değer Etki Alanı girilir. Değer Etki Alanı, var olanların arasından seçilir ya da yeni oluşturulur.
4. Veri Elemanı kaydedilir.

### Domain Value oluşturma

Bir Veri Elemanının Veri Sözlüğüne kaydedilebilmesi için mutlaka bir Değer Etki Alanı ile ilişkilendirilmesi gerekir. Bu Değer Etki Alanı, daha önce tanımlanmış olanlardan seçilebileceği gibi, Veri Elemanı için veri girişi yapılırken de oluşturulabilir.

1. *Tip* seçilir. Seçilebilecek iki tip vardır

   1. Tarif Edilen Değer Etki Alanı
   2. Kodlanmış Değer Etki Alanı (Referans Kod)

   *Tarif* *Edilen* *Değer* *Etki* *Alanı* varsayılan değerdir.

2. *Veri Tipi* seçilir. Seçilmesi zorunludur.

3. Varsa verinin *Format* bilgisi girilir.

4. Varsa verinin *Alan Büyüklüğü* bilgisi girilir.

5. *Ad* girilir. Girilmesi zorunludur.

6. Değer Etki Alanı ile Veri Seti Tanımının ortak olan üstveri alanları için veri girişinde dikkat edilmesi gereken noktalar aynıdır. Dolayısıyla Veri Seti Tanımı Oluşturma bölümündeki 2-6 maddeleri, Değer Etki Alanları için de geçerlidir.

7. Değer Etki Alanı kaydedilir.

8. Eğer Tip olarak *Kodlanmış* *Değer* *Etki* *Alanı (Referans* *Kod)* seçilirse, kod değerleri ve anlamlarının girilmesi gerekir. Oluşturulan kod tablosu için istenildiği kadar kolon eklenebilir. Ancak bu kolonlardan yalnızca bir tanesi, ilgili Veri Elemanının değerini saklayacak kolon olarak belirlenmeli, bu kolonun veri tipi ile Değer Etki Alanının veri tipi aynı olmalıdır. Cinsiyet adlı bir Kodlanmış Değer Etki Alanı örneği verilmiştir.

### Onay İşlemleri

Bir kayıt KAYITLI duruma geçtiği andan itibaren SİLİNEMEZ, yalnızca kayıt durumu KULLANIM DIŞI ya da MÜLGA olarak belirlenebilir. Bu nedenle, bir ögenin onay sürecine sokulması geri dönüşü olmayan bir işlemdir ve dikkatle ele alınmalıdır. Veri Sözlüğü Yazılımına kaydedilen her ögenin mutlaka onay sürecine girmesine gerek yoktur. Bunların en başta ADAY olarak kaydedilmesi, değiştirilebileceklerini ya da tamamen silinebileceklerini ifade etmektedir.

#### Onay – Öneren Sorumlu

Süreci Öneren/Sorumlu rolündeki kullanıcılar başlatır. Bu kullanıcılar Üstveri Yönetim Ögelerini belirleyen, Veri Sözlüğü Yazılımı üzerinden veri girişini yapan, güncelleme, silme işlemlerini gerçekleştiren kullanıcılardır.

Herhangi bir öge arayüzler üzerinden ya da MetaImporter aracılığıyla Veri Sözlüğü Yazılımına kaydedildiği zaman, bu ögenin Veri Sözlüğündeki durumu ADAY olur. Amaç, ADAY durumundaki kayıtları adım adım ADAY durumunun üzerindeki durumlara (mümkünse en üste) taşımak, kullanımdan kaldırılan ögelerin durum bilgisini uygun şekilde güncellemek, gereksiz olduğu belirlenen ögeleri ise sözlükten tamamen silmektir. Böylece kurumun Veri Sözlüğü zaman içinde olgunlaştırılarak bütün verilere ilişkin üstverileri doğru ve güncel bir şekilde sunan bir içeriğe kavuşturulur.

#### Kayıt Otoritesi

Öneren/Sorumlu kullanıcı bir Durum Değişiklik Talebi oluşturduğu zaman, Veri Sözlüğü Yazılımı Kayıt Otoritesi rolündeki ilgili kullanıcılara bildirim göndererek durumu haber verir.

Danışma Komisyonu üyelerinin görüşleri bağlayıcı değildir. Kayıt Otoritesi bu görüşleri değerlendirmeye alır, ancak kendi kararını verir.

#### Kontrol Komitesi

Kontrol Komitesi, Veri Sözlüğü bazında oluşturulan ve en az bir üyeden oluşan bir karar komisyonu olarak çalışır. Kayıt Otoritesinin tek başına karar veremediği talepler, Kontrol Komitesi üyeleri tarafından değerlendirmeye alınarak karara bağlanır. Komite, alan bilgisine ihtiyaç duyarsa bir ya da daha fazla Danışma Komisyonundan görüş talebinde bulunabilir.

#### Danışma Komisyonları

Bir Veri Sözlüğü için birden çok Danışma Komisyonu oluşturulabilir. Danışma Komisyonların alan uzmanlıklarına göre adlandırılması ve üyelerinin de buna göre seçilmesi, görüş isteneceği zaman komisyonun doğru belirlenmesini sağlar. Kayıt Otoritesi ya da Kontrol Komitesi üyesi olan kullanıcılar, alan bilgisine ihtiyaç duydukları kayıtlar için ilgili Danışma Komisyonlarının görüşlerine başvurabilirler. Danışma Komisyonundaki her üye, kendisine yöneltilen taleplerle ilgili oy kullanabilir ve diğer üyelerin yorumlarını görebilir. Komisyon görüşleri bağlayıcı değildir ve Kayıt Otoritesi ya da Kontrol Komitesi görüş istediği kaydı olumlu ya da olumsuz olarak sonuçlandırmadığı sürece Danışma Komisyonu üyeleri kendi görüşlerini değiştirebilir ya da yorumlarını güncelleyebilir.

#### Güncelleme

Veri Sözlüğündeki ögelerin üstverilerinde güncellemeler yapılması gerekebilir. Bu güncellemelerin başlıca nedenleri;

- Üstveride meydana gelen değişiklikler,
- Kayıt durumunu ilerletmek için yeni üstveri alanlarının doldurulması,
- Gözden geçirmeler sırasında fark edilen eksiklik ya da hatalar,
- Gözden geçirmeler sırasında, bazı Veri Elemanlarının ya da Veri Seti Tanımlarının başka ögelerle örtüştüğünün belirlenerek bunların birbirleriyle uyumlu hale getirilmesi

olarak sıralanabilir. 



Bir Üstveri Yönetim Ögesi, KAYITLI ve daha üst durumlardayken güncellenmek istenirse Güncelleme Süreci işletilir. ADAY durumundaki kayıtlar üzerinde güncelleme ya da silme işlemlerinin yapılmasına bir engel yoktur. KULLANIM DIŞI ya da MÜLGA durumundaki kayıtlar güncellenemez.

Güncellenen kayıtlar üzerinde, bu kayıtlar Kayıt Otoritesinin onayına sunularak onaylanana ya da bütün değişikliklerin geri alınarak kayıt eski haline getirilene kadar başka güncellemeler de yapılabilir. Onay işlemi sadece güncellemeler için olabileceği gibi, kaydın durumunun ilerletilmesi için Durum Değişiklik Talebi oluşturulması şeklinde de gerçekleşebilir. Güncellemeler onaylanana kadar kaydın üstverilerindeki değişiklikler Veri Sözlüğüne yansıtılmaz.

#### Versiyon Oluşturma

Veri Sözlüğü Yazılımı çerçevesinde iki tür versiyon bulunmaktadır. Bu nedenle herhangi bir karışıklık oluşmasını engellemek amacıyla öncelikle bunların açıklanması gerekir.

##### Veri Sözlüğü Versiyonu

Veri Sözlükleri sürekli olarak gözden geçirilir ve birtakım güncellemeler yapılabilir. Bu güncellemelerin yayınlanması için Veri Sözlüğünün versiyonları oluşturulur. Örneğin Ulusal Sağlık Veri Sözlüğünün 2008 yılının Ocak ayında kullanıma sunulmuş olan 1.1 sürümü kamu ve özel 2. ve 3. basamak sağlık kurumlarına odaklanırken, 14 Mart 2012 yılında yayınlanan 2.0 sürümü Aile Hekimliğini kapsayacak şekilde hazırlanmıştır. Halen yayında olan güncel sürüm ise 2.2 sürümüdür.

Veri Sözlüğü Versiyonu oluşturulurken, bu versiyona Veri Seti Tanımı ve Veri Elemanlarının hangi versiyonlarının koyulacağı belirlenir ve Veri Sözlüğünün tanımlı versiyonlarından herhangi birisi indirilebilir.

##### Veri Seti Tanımı ya da Veri Elemanı Versiyonu

Veri Sözlüğünde bir Veri Seti Tanımı ya da Veri Elemanının farklı versiyonları bulunabilir. Bir ögenin yeni bir versiyonunu oluşturmak için tipik bir senaryo şöyledir:

1. Sözlükte var olan bir Veri Elemanının anlamını ya da tanımını değiştirecek bir güncelleme yapılması ihtiyacı belirir.
2. Bunun için önce Veri Elemanının yeni bir versiyonu oluşturulur.
3. Güncellemeler yeni versiyon üzerinde yapılır ve yürürlük tarihi belirtilir.
4. Veri Elemanını kullanan bilgi sistemlerinde ilgili değişikliklerin yapılması gerektiği ve yürürlük tarihinden itibaren Veri Elemanının yeni versiyonunun geçerli olacağı duyurulur. O tarihe kadar eski versiyon yürürlükte kalır.
5. Belirtilen tarihte diğer bilgi sistemleri kendi taraflarındaki güncellemeleri tamamlamış olur. Eski versiyonlu Veri Elemanı kullanım dışına çekilir ve Veri Elemanını kullanan Veri Seti Tanımı, yeni versiyonu kullanacak şekilde güncellenir.

Veri Sözlüğü Yazılımı üzerinde bir Veri Seti Tanımı ya da Veri Elemanının yeni versiyonunu oluşturmak için Versiyon Oluşturma Süreci işletilir.

Versiyon Oluşturma Süreci, Veri Sözlüğünün değil, Veri Seti Tanımları ya da Veri Elemanlarının yeni versiyonlarını oluşturmak için kullanılan süreçtir.
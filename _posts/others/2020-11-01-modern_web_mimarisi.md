---
layout: post
title:  "Modern Web Mimarisi"
date:   2020-11-01 12:45:31 +0530
categories: "others"
author: "mehmetozanguven"
---

# Modern Web Mimarisi

Yazılım geliştiriciler olarak günümüzün çoğunu kod yazarak geliştiriyoruz. Kimimiz kendi projesini geliştirirken, kimimiz çalıştığı şirketler adına geliştirmeler yapıyor. Çoğu zaman sadece bizlere verilen geliştirmeleri yapıyor, sonrası ile pek igilenmiyoruz. Fakat günümüzün web mimarileri pek çok bileşenin bir araya gelmesinden oluşuyor. Bu mimarinin büyük resmini görmek veya en azından temel seviyede bilmek çok önemli. Bu yazıda temel olarak modern web mimarisinin bileşenlerini anlatacağım.

<img src="/assets/others/web_mimari.png" alt="web_mimari.png" />

Yukarıdaki resim temel olarak arka planda gerçekleşen olayları anlatıyor. Tabikide bu akış şirketten şirkete değişikliği gösteriyor, ve genellikle kişisel projelerimizde buradaki pek çok akış bizleri ilgilendirmiyor.

Şimdi kullanıcının tarayıcıya web adresinizi yazdıktan sonra neler olduğuna sırayla bakalım.

## DNS

Her şeyden önce kullanıcının bilgisayarı  sadece web adresine (örneğin: https://hackathonturkiye.com/) bakarak nereye gideceğini bilemez. Çünkü internet ortamında haberleşme IP adresleri üzerinden gerçekleşir. Web adresinizden IP adresinize ulaşmak için devreye DNS(Domain Name System) araya giriyor. Temel olarak bilgisayarınız DNS resolver'a *benim kullanıcım https://hackathonturkiye.com/ isimli bir yere bağlanmak istiyor, bana bunun IP adresini ver ki ben de gidip bağlantı sağlayıp, kullanıcıma gerekli veriyi (bu durumda html sayfası olacak) göstereyim*.

Çok bilinen DNS resolver adresleri olarak Google DNS adreslerini verebiliriz: **8.8.8.8 ve 8.8.4.4**

## Load Balancer

Not: Eğer basit bir web uygulaması geliştiriyorsanız çok büyük olasılık **load balancer** ile uğraşmayacaksınız.

Kullanıcının bilgisayarı IP adresini öğrendikten sonra size isteklerini göndermeye başlayacak ve bu büyük şirketlerde genellikle **load balancer'ın IP adresi olacaktır.**

### Nedir bu Load Balancer ?

Load Balancer en temelinde bir **reverse proxy sunucusudur**. Öncellikle **proxy sunucusunun** ne iş yaptığından bahsedelim. Bir proxy sunucusu, kullanıcı ve bağlanmak istediği web sitesinin arasına girer. Basit bir örnekle anlatmak gerekirse, mesela hackathonturkiye.com adresine proxy sunucusu aracılığı ile bağlanmak istiyorum diyelim, proxy sunucusuna şunu diyorum:

- *Proxy sunucusu, benim hackathonturkiye.com isteğimi al, bu isteği sen yap gelen sonucu ise bana gönder. Böylelikle bu isteği ben değil sen yapmış olacaksın*

Peki neden isteği proxy sunucusunun yapmasını istiyoruz, bunun için pek çok sebep sayılabilir:

- hackathonturkiye.com'un benim IP adresimi öğrenmesin (gizlilik amacıyla)
- hackathonturkiye.com Türkiyeden erişimim yok, fakat proxy sunucusuna erişimim var. Proxy sunucu bağlantı sağlayıp cevabı bana dönebilir. (Yasaklı siteleri erişim için)
- ...

**Reverse proxy** ne oluyor peki ? Ben nasıl istek yaparken gizli kalmak istiyorsam, hackathonturkiye.com'da  kendi ip adreslerini istek yapan kullanıcılardan saklamak isteyebilir. İşte bu durumda araya reverse proxy giriyor. Şöyle düşünün hackathonturkiye.com'un 2 veya daha fazla sunucusu var, kullanıcılar bağlanırken bu iki adresten birine istek atmaları lazım. Fakat bu 2 adresi internet ortamında göstermek yerine bütün kullanıcıları tek bir adrese yönlendirip (reverse proxy sunucusunun adresi) daha sonra istekleri kendi sunucularına yönlendirebilir. Peki bunun ne gibi yararları var:

- Kimse 2 sunucunun adresini bilmeyecek, olası saldırılarda sadece reverse proxy etkilenecek.
- hackathonturkiye.com isterse bir sunucuyu kapatıp diğerini açık bırakabilir, reverse proxy'den gelen istekleri sadece sunucu1'e yönlendirir veya sadece sunucu2'ye yönlendirir. İkiside internete açık olsaydı, bunu yapamayacaktı. Kısacası daha esnek bir yapı.
- ...

Şimdi bu reverse proxy'i biraz daha akıllandıralım. Örneğin reverse proxy, istekleri yönlendireceği 2 sunucu makinesini sürekli kontrol etsin ve şu gibi özelliklere sahip olsun:

- Eğer sunucu1 ağır yük altında ise, gelen istekleri sunucu2'ye yönlendirsin.
- Eğer sunucu1 çökerse, gelen bütün istekleri sunucu2'ye yönlendirsin.

Bu ve daha çok bir çok özelliğe sahip olanlar **Load Balancer** olarak adlandırılıyor. Yani Load balancer kullanma sebebi, şirketlerin herhangi bir sunucuları çöktüğünde gelen istekleri diğer makinelere dağıtmak ve olası sunucu çökmelerinde direkt haberdar olmak. Eğer  daha fazla bilgi almak istiyorsan Citrix Netscaler diye google'da aratabilirsiniz. (Load balancing işlerinde kullanılıyor)

## Web Uygulama Sunucuları

Kullanıcılarınızın isteklerinin gerçekleşeceği yere geldik. Burada artık kullanıcı üye girişi yapmak isterse bu işlemi gerçekleştiriyoruz, görmek istediği sayfayı gösteriyoruz vs.. 

Burada kullanılan bir kaç araç: Spring Framework, Django, Nodejs, Laravel vs..

## Database

Not: Eğer basit bir websiteniz varsa muhtemelen aynı makine üzerinden hem web uygulamanız hemde database sunucunuz çalışacak.

Kullanıcılarınızın verilerinin depolandığı yer. Burada SQL (MySQL, PostgreSQL gibi) ve/veya NoSQL(MongoDB) gibi çözümler kullanabilirsiniz. Websiteniz ne kadar büyükse bu kısımda o kadar büyük oluyor. Bir makineye bütün bilgileri sığdırmak imkansız olduğu için, genel olarak burada bir çok makine(sunucu) bir araya gelerek bir küme oluşturuyor ve web uygulaması bu kümeye istek atıyor. Bu küme kendi aralarında konuşarak backup ve daha pek çok işlemleri gerçekleştiriyorlar.

## Caching Sunucuları

Not: Eğer basit bir websiteniz varsa muhtemelen bu kısım ile uğraşmayacaksınız.

Süre alan işlemlerin sürekli ve sürekli yeniden yapılması yerine bir `map` yapısında saklayıp `O(1)` işlem zamanında hızlı sonuç döndürmek için **caching sunucuları** kullanılır. Burada asıl amaç kullanıcılarınıza hızlı bir şekilde geri dönüş sağlamaktır.

Örneğin https://hackathonturkiye.com/ sitesinde listelenen hackhathonların database'den okunup ekrana basıldığı ve database'den okuma işlemin uzun sürdüğünü(sonuçta aynı zaman aralığında pek çok hackathon gerçekleşiyor) varsayalım. Bu durumda sürekli database gidip okuma yapmak yerine  listelenen hackhathon verilerini bilgisayarın hafızasında saklayıp, daha sonraki isteklerde sonucu bilgisayarın hafızasından direk alabiliriz.

Örnek olarak [Redis](https://redis.io/) veya [memcache](https://memcached.org/) araçlarına bakabilirsiniz.

## Job Sunucuları

Not: Eğer basit bir websiteniz varsa muhtemelen bu kısım ile uğraşmayacaksınız.

Bazı isteklerin arka planda kullanıcılardan veri girişi beklemeden çalıştırılması gerekiyor (asenkron bir şekilde). Bu istekler direk olarak kullanıcıların doğrudan yaptığı istekler değil. 

Mesela şunun gibi bir istek: *Her gün saat 17:00'da mail listesine kayıtlı olan tüm kullanıcılara yeni gelen hackhaton'ları mail at*, job sunucularında gerçekleştirebilir.

Örnek olarak [Luigi](https://luigi.readthedocs.io/en/stable/index.html), [Azkaban](https://azkaban.github.io/) veya [Oozie](http://oozie.apache.org/) gibi araçlara bakabilirsiniz.


## Arama Servisleri

Websitelerinde arama kutucuğuna bir şeyler yazdığımızda bize en anlamlı sonuçları göstermeye başlarlar. Örneğin hepsiburada.com'da arama kutucuğuna *sam* yazdığınızda *samsung* yazısının çıkması gibi. Bu işlemler bir veye daha fazla makinede çalışan araçlar sayesinde gerçekleşiyor. Örnek olarak [Elastic Search](https://www.elastic.co/elasticsearch/), [Apache Solr](https://lucene.apache.org/solr/features.html) araçlarına bakabilirsiniz.



Burada temel bileşenlerden bahsetmeye çalıştım, fakat bunlarla sınırlı değil, örneğin websitenizdeki veriler ne kadar büyükse, veriler üzerinde analizi bir makine üzerinden gerçekleştirmek o kadar zor oluyor. Bu gibi durumlar büyük veri araçları [Apache Spark](https://spark.apache.org/) , [Apache Hadoop](https://hadoop.apache.org/) ve benzeri araçlar kullanılıyor.








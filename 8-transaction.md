# MongoDB' de Transaction Yönetimi (MongoDB Öğreniyoruz 8)

MongoDB'de bir doküman üzerinde yapılan işlemler atomiktir. Yani bir belge üzerinde yaptığınız örneğin bir update işleminde doküman ya tamamen update edilir yada edilmez. Burada amaç doküman üzerindeki array'lar, embedded dokümanlar veya referenced dokümanların bir bütün olarak değerlendirilmesidir. Bundan dolayı da zaten pratikte tabiri caizse mümkün olduğunca relation'a çok bulaşmamak en iyisi. Ancak tabii ki gerçek hayatta her şeyi tek collection'a sığdırmak imkansız.

Neyse ki MongoDB single document transaction yanı sıra multi-document transaction'ı da destekler. Ayrıca distributed transaction'ı da destekler. Üstelik transaction multiple operation, collection, database, document ve shard üzerinde gerçekleştirilebilir.


Örneklere geçmeden önce transaction başlatırken ayarlayabileceğimiz opsiyonları ve yönetmemiz gereken hata tiplerini inceleyelim.


Öncelikle MongoDB'de programlama dillerinde kullanacağımız driver'larda iki adet API (core ve callback) var. 

- [Callback API](https://www.mongodb.com/docs/manual/core/transactions-in-applications/#callback-api-vs-core-api)
  Transaction'ı başlatır ilgili operasyonları çalıştırır ve hata olmaması durumunda kendisi otomatik olarak commit yapar. Ayrıca hata durumlarını kendisi yönetebilir.
- [Core API](https://www.mongodb.com/docs/manual/core/transactions-in-applications/#callback-api-vs-core-api)'de ise
  Transaction'ı açıkça  başlatmak, commit ve rollback işlemlerini de manuel yapmak zorundayız. Ayrıca hata durumlarını da yönetmemiz gerekiyor. Pratiklik açısından rollback iyi görünse de core API de esneklik sunmaktadır. Örneğin yukarıda shell üzerinde gördüğümüz örnek core API kullanmaktadır.


Transaction işlemlerinde iki tip hata fırlatılmaktadır. 

-  **TransientTransactionError**: Eğer callback API kullanıyorsak ve  transaction esnasında bu hata ile karşılaşılırsa sistem bütün transaction'ı yeniden başlatır.
-  **UnknownTransactionCommitResult**: Eğer Eğer callback API kullanıyorsak ve  transaction esnasında bu hata ile karşılaşılırsa sistem commit işlerimi tekrar dener.


Ayrıca transaction konuşacaksak consistency kavramından da biraz bahsetmek gerekiyor. Serinin ilk yazısında veritabanlarını karşılaştırırken MongoDb'nin [Strong consistency](https://en.wikipedia.org/wiki/Consistency_model) yapısı olduğunu söylemiştik. Yani verinin tutarlılığına çok önem veriyor. Strong consistency'nin de farklı seviyeleri mevcut.


Transaction'un içinde yapılan operasyonların ve transaction'ının tamamlanması için bazı şartlar var. Bunlar genellikle default ayarlarıyla kullanılır ancak ihtiyaçlarımıza göre değiştirme şansımız var.

Doğru yapılacak read ve write concern ayarları ile beklentilerimizi karşılayacak consistency ve availability seviyesini ayarlayabiliriz. 

-  **Transaction-level read concern (read isolation)** : Okumaya çalıştığımız veri üzerinde izolasyon ve cocnsistency özelliklerini ayarlamamızı sağlar.
   -  **Local**: Verinin bütün node'lara yazıldığının garantisi olmaksızın node'lardan veri getirir. Yazma işlemi devam ederken okuma yapılması durumunda okuduğumuz verinin garantisi yoktur yani veri rollback yapılabilir.[Local sayfasında](https://www.mongodb.com/docs/manual/reference/read-concern-local/#mongodb-readconcern-readconcern.-local-) çok detaylı bilgi bulabilirsiniz.
   -  **Available**: Local seviyesi gibidir.  Sharded cluster'larda en hızlı okumayı sağlar ancak bunu da bir maliyeti vardır. Bozuk veri gelme olasılığı vardır (örneğin [orphan document](https://www.mongodb.com/docs/manual/reference/glossary/#std-term-orphaned-document)). Unsharded ortamlarda local ile available aynı şekilde davranır.
   -  **Majority**: (Default değer) Okunan  verinin cluster node'larının önemli bir kısmı tarafından onaylanmış ve rollback yapılmayacağının garantisi olan veri olduğunu ifade eder. Tabii ki burada özellikle sharded cluster'larda performans kaybı olacaktır. Performansı hızlandırmak için [şu dokümana](https://www.mongodb.com/docs/manual/tutorial/mitigate-psa-performance-issues/#std-label-performance-issues-psa) göz atabilirsiniz.
   -  **Linearizable**: Cluster'ın bütün node'ları tarafından onaylanmış verilerin döndürülmesini sağlar.
   -  **Snaphot**: MongoDb versiyon 5 ile gelen özelliktir ve multi document transaction'larda kullanınılır. Snapshot, serializable bir transaction'da bütün işlemlerin bir transaction'da yapılması ve bu sırada ilgili kaydın/kayıtların kilitlenmesi aynı veri üzerinde işlem yapılmasını engellemesi sebebiyle dağınık yapıdaki veritabanlarında kullanılmaktadır. Ancak birçok ilişkisel veritabanı da yeni versiyonlarında bu problemi çözmek amacıyla snapshot izolasyonu desteklemeye başlamışlarıdr. ACID prensiplerindeki I izolasyonu ifade etmektedir.MongoDB bunu iki farklı şekilde yapar. Transaction durumunda yapılan işlemler verinin bir snapshot'ı alınarak yapılır. Bu esnada asıl veri üzerinde diğer oturumlar okuma veya yazma yapabilirler (tabi bu veritabanının yeteneklerine göre kısıtlanabilir). Snapshot üzerinde işlemler tamamlandığında eğer verinin ilk alınmasında sonra değişikliğe uğramadıysa veri kaydedilir (transaction commit), eğer bu arada veri değiştiyse transaction conflict gerekçesiyle iptal edilir. Daha fazla detay almak için [şu sayfayı](https://www.mongodb.com/docs/manual/reference/read-concern-snapshot/#mongodb-readconcern-readconcern.-snapshot-) ziyaret ediniz.
-  **Transaction-level write concern (write acknowledgement)** :  Yazma işlemi yapıldıktan sonra MongoDB yazma işlemi hakkında bilgi (acknowledgement) dönmektedir. Dönülecek bilgide neleri beklediğimizi daha doğrusu sunucunun yazma işlemini belli bir timeout süresi içinde hangi level'da yapacağını belirlememiz sağlıyor. Özellikle bir cluster yapımız varsa bu ayarı beklentilerimizi karşılayacak şekilde yapmamız çok önemli.
   -  **w**: Yazma işleminin cluster yapıda kaç node'a yayıldığının onayını istediğimiz seçenektir. eğer "0" dersek bir onay istemediğimiz anlamına gelir. Yani "fire and forget" gönder ve unut. "1" dersek replica set üzerinde primary üzerine yazılması onayını istemiş oluruz. 1'den büyük vereceğimiz her değer kaç adet secondary'ye kaydolması gerektiğinin onayını almamızı sağlar. Eğer rakam yerine majority yazarsak (ki bu default değerdir) bütün sunuculara yazıldığının onayını almış oluruz. 
   -  **j**: bu opsiyonu anlamak için öncelikle journaling kavramını anlamak gerekiyor. Journaling yazma işlemi sisteme geldiğinde (vakit alacağı için) gerçek yazma işlemini yapmadan önce yapılacak işlemin disk üzerine loglanması işlemidir. Daha fazla detay için [şu linki](https://www.mongodb.com/docs/manual/tutorial/manage-journaling) ziyaret ediniz. Amaç bir crash anında verinin korunmasını sağlamak ve bazı durumlarda hız sağlamak içindir. Eğer bu değer false olarak ayarlanırsa journaling prosesi onayı istememiş oluruz. Şunu da bilmemiz gerekiyor bunu true bile yapsak replica set primary'nin crash olması durumunda transaction'ın roll back yapılmayacağı anlamına gelmez. 

![journaling.png](files/journaling.png)

[resim kaynak](https://sauravomar01.medium.com/journalling-in-mongodb-eff866ac9712)
   -  **wtimeout**: w değeri birden büyük bir değere sahip olduğunda çalışır. Belirlenen süre içinde yazma işlemi tamamlanmazsa (eventully tamamlansa bile) hata döner. Ancak belirtilen süre içinde başarılı yapılan değişiklikler geri alınmaz. w parametresinden belirtilen bütün node'lardan gelen dönüş zamanaı dikkate alınır yani sadece primary'ye bakılmaz.

Replicaset üzerinde write concern örneği.
![write-concern.png](files/write-concern.png)

-  **Transaction-level read preference** :   Okuma işlemninin hangiş sunucu tarafından karşılanacağını ayaralamk için kullanılır. Default değer olarak primary ayarlıdır.
   -  **primary**: Primary'den okunur. Multi document transaction yapılırken primary olmalıdır.
   -  **primaryPreferred**: Önce primary o yoksa secondary.
   -  **secondary**: Secondary'den okunur.
   -  **secondaryPreferred**: Önce secondary o yoksa secprimaryondary.
   -  **nearest**: Rastgele seçim yapılır. Bu rastgelelik okadar da rastgele değil tahmin edebileceğiniz gibi. Detaylar için [şu sayfayı](https://www.mongodb.com/docs/manual/core/read-preference/#mongodb-readmode-nearest) ziyaret ediniz.


Dikkat etmemiz gereken bazı kısıtlar var.

- Versiyon 4.2 ve öncesinde Transaction içinde collection oluşturamayız. 4.4 den itibaren oluşturmak mümkün artık. Ancak bu halen cross shard ortamlarda mümkün değil. Detaylar için [şu sayafayı](https://www.mongodb.com/docs/manual/core/transactions/#create-collections-and-indexes-in-a-transaction) ziyaret ediniz.
- Cross-shard olmasa dahi farklı shard'larda collection oluşturulamaz.
- non-CRUD ve no-informational işlemler yapılamaz. Örneğin kullanıcı oluşturmak, count almak vb. 



Daha önce de belirttiğimiz üzere MongoDB shell NodeJS'le tam uyumlu olduğu için dokümanları takip  ederken dil olarak NodeJS seçerek örnekleri MongoDB Shell üzerinde de yapmak mümkün.
Örneklerde dil seçeneklerinde shell olmasa da [şu sayfada](https://www.mongodb.com/docs/manual/core/transactions-in-applications/#std-label-txn-mongo-shell-example) shell örneğini bulmak mümkün.

İleride örneklerle daha da detaylı inceleyeceğiz. Aşağıda sadece ufak bir shell örneğini nasıl birşeyle karşılaşacağımız hakkında fikir vermesi adına paylaşıyorum

```javascript

// Collection oluşturuluyor.
db.getSiblingDB("mydb1").foo.insertOne(
    {abc: 0},
    { writeConcern: { w: "majority", wtimeout: 2000 } }
)
db.getSiblingDB("mydb2").bar.insertOne(
   {xyz: 0},
   { writeConcern: { w: "majority", wtimeout: 2000 } }
)

// Session başlatılıyor. 
session = db.getMongo().startSession( { readPreference: { mode: "primary" } } );
coll1 = session.getDatabase("mydb1").foo;
coll2 = session.getDatabase("mydb2").bar;


// Transaction başlatılıyor.
session.startTransaction( { readConcern: { level: "local" }, writeConcern: { w: "majority" } } );


// Farklı iki collection üzerinde doküman kaydediliyor.
try {
   coll1.insertOne( { abc: 1 } );
   coll2.insertOne( { xyz: 999 } );
} catch (error) {

// Hata olması durumunda transaction abort ediliyor.
   session.abortTransaction();
   throw error;
}

// transaction'ı commit'liyoruz.
session.commitTransaction();
session.endSession();

```


Bakılmalı

https://www.mongodb.com/docs/manual/core/transactions/

https://www.mongodb.com/docs/manual/core/transactions-sharded-clusters/ (production consideration)

https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/ (casula consistency tablo)

https://dzone.com/articles/mongodb-consistency-levels-cappaclec-theorem (consistency matrix var)

https://vkontech.com/causal-consistency-guarantees-in-mongodb-lamport-clock-cluster-time-operation-time-and-causally-consistent-sessions/


https://www.mongodb.com/blog/post/performance-best-practices-transactions-and-read--write-concerns (transactipon best practices)














# Kaynaklar
- https://www.mongodb.com/docs/manual/core/transactions/
- https://www.mongodb.com/docs/manual/core/data-model-operations/
- https://www.mongodb.com/docs/manual/tutorial/model-data-for-atomic-operations/
- https://medium.com/mongodb-performance-tuning/tuning-mongodb-transactions-354311ab9ed6
- https://www.mongodb.com/docs/manual/reference/write-concern/
- https://www.mongodb.com/docs/manual/reference/read-concern/
- https://www.mongodb.com/docs/manual/core/read-preference/
- https://www.mongodb.com/community/forums/t/what-is-causal-consistency-in-mongodb-continuation/174351/6
- https://en.wikipedia.org/wiki/Consistency_model
- https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/
- https://vkontech.com/causal-consistency-guarantees-in-mongodb-lamport-clock-cluster-time-operation-time-and-causally-consistent-sessions/
- https://www.youtube.com/watch?v=x5UuQL9rA1c
- https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html
- https://dzone.com/articles/mongodb-consistency-levels-cappaclec-theorem
- https://www.mongodb.com/docs/manual/reference/write-concern/
- https://www.mongodb.com/docs/manual/core/read-preference/
- https://www.mongodb.com/docs/manual/reference/read-concern-linearizable/
- https://www.mongodb.com/docs/manual/reference/read-concern-local/
- https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns
- https://www.mongodb.com/docs/manual/core/read-isolation-consistency-recency/
- https://social.technet.microsoft.com/wiki/contents/articles/26576.sql-server-transaction-isolation-levels-tr-tr.aspx
- https://www.mongodb.com/docs/manual/core/replica-set-write-concern/


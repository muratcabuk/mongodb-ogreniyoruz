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


Ayrıca transaction konuşacaksak consistency kavramından da biraz bahsetmek gerekiyor. Serinin ilk yazısında veritabanlarını karşılaştırırken MongoDb'nin [Strong consistency](https://en.wikipedia.org/wiki/Consistency_model) yapısı olduğunu söylemiştik. Yani verinin tutarlılığına çok önem veriyor. Strong consistency'nin de farklı seviyeleri mevcut. Consistency modeller dağınık yapılar için kullanılan yapılardır. Haliyle MongoDB'yi cluster olarak kurguladığımızda bu konu transaction için çok önemli oluyor.



**Causal Consistency Kavramı**





Transaction'un içinde yapılan operasyonların ve transaction'ının tamamlanması için bazı şartlar var. Bunlar genellikle default ayarlarıyla kullanılır ancak ihtiyaçlarımıza göre değiştirme şansımız var.

-  **Transaction-level read concern** : Eğer belirtilmezse default olarak  client-level read concern kullanılır. Farklı seviyeler tanımlama mümkün.
   -  **Local**: Verinin bütün node'lara yazıldığının garantisi olmaksızın node'lardan veri getirir. Yazma işlemi devam ederken okuma yapılması durumunda okuduğumuz verinin garantisi yoktur yani veri rollback yapılabilir.[Local sayfasında](https://www.mongodb.com/docs/manual/reference/read-concern-local/#mongodb-readconcern-readconcern.-local-) çok detaylı bilgi bulabilirsiniz.
   -  **Available**: Local seviyesi gibidir.  Sharded cluster'larda en hızlı okumayı sağlar ancak bunu da bir maliyeti vardır. Bozuk veri gelme olasılığı vardır (örneğin [orphan document](https://www.mongodb.com/docs/manual/reference/glossary/#std-term-orphaned-document)). Unsharde ortamlarda local ile available aynı şekilde davranır.
   -  **Majority**: Okunan  verinin cluster node'larının önemli bir kısmı tarafından onaylanmış ve rollback yapılmayacağının garantisi olan veri olduğunu ifade eder. Tabiiki burada özellikle sharded cluster'larda performans kaybı olacaktır. Performansı hızlandırmak için [şu dokümana](https://www.mongodb.com/docs/manual/tutorial/mitigate-psa-performance-issues/#std-label-performance-issues-psa) göz atabilirsiniz.
   -  **Linearizable**: Cluster'ın bütün node'ları tarafından onaylanmış 
-  **Transaction-level write concern** :  Eğer belirtilmezse default olarak client-level write concern kullanılır.
   -  sfsdf
   -  sfsdf
   -  sdsfsd
-  **Transaction-level read preference** :   Eğer belirtilmezse default olarak client-level read preference kullanılır.
   -  sdfsdf
   -  sdfsd
   -  sdfsd


Dikkat etememiz gereken bazı kısıtlar var.

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
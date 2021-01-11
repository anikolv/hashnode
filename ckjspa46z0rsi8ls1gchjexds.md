## Implement full-text search with Hibernate Search and Apache Lucene

There are a lot of possible ways for implementing search engine capabilities in our application. The straightforward path is searching by explicit fields or properties, represented like a form with multiple inputs in frontend terms. You fill-in search criteria for the attribute of your choice and fire the search. But what if we don't want to force explicit searching per attribute? What if we want to enable searching based on best-match merge from all attributes? Concept with probably the best analogy being the Google search box. Well, in this case we have the full-text search capability which makes it possible to scan all possible attributes of the searching target and finds the best match. Now, even full-text searching is a field with huge amount of possible approaches, mostly depending on the tech stack. In this article I present an approach in case of Hibernate ORM used for your persistence layer.

The full-text search capabilities of Hibernate are provided by an extension library called hibernate-search. This library provides powerful searching capabilities with, in my opinion, the big advantage of being non-intrusive and aspect-oriented. This means that it's easily plugged in in out existing Hibernate classes with simple annotations which mark our search entities and their properties. Hibernate search lays upon the search engine Apache Lucene and takes care of the indexation of your search entities without any need for database changes on your side. Let's see how it works in practice.


We start with adding the hibernate-search and lucene related dependencies (example for Maven, check Maven central for latest versions):

```
      <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-search-orm</artifactId>
        <version>5.11.8.Final</version>
      </dependency>

      <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-search-engine</artifactId>
        <version>5.11.8.Final</version>
      </dependency>

      <dependency>
        <groupId>org.apache.lucene</groupId>
        <artifactId>lucene-core</artifactId>
        <version>3.6.2</version>
       </dependency>
``` 

Then you need to configure the index directory where your Lucene indexes will be stored (example with hibernate properties, could be XML config as well):

```
    hibernate.search.default.directory_provider = filesystem
    hibernate.search.default.indexBase = /data/index/
``` 

Note that you can also configure the index location to be in-memory. Read more in the  [HIbernate Search docs](https://docs.jboss.org/hibernate/search/5.6/reference/en-US/html/ch03.html#search-configuration-directory). 

Having this in place, hibernate-search will use /data/index directory to store document-based indexes for your JPA entities. The indexes itself are an optimized Lucene-specific structures of data, enabling high-performant full-text searching capabilities. Those index structures are way more performant than standard SQL based column indexes. Refer to the  [Apache Lucene Docs](https://lucene.apache.org/core/3_0_3/fileformats.html) for more information on the index file structure and how it works in details. 

You can now proceed with annotating your JPA entities. There are two general annotations - @Indexed (marks an entity as indexable) and @Field (marks entity property as indexable and searchable). Below is the most generic setup example upon two example entities - Order and OrderLine with relation one-to-many:

```
@Entity
@Indexed
@Table(name = "order")
public class Order {

    @Id
    @Field
    private long id;

    @Field
    private String status;

    @Field
    private String comment;

    @OneToMany(mappedBy = "order", orphanRemoval = true, fetch = FetchType.EAGER)
    @IndexedEmbedded
    private Set<OrderLine> orderLines = new LinkedHashSet<OrderLine>();

}
``` 

```
@Entity
@Indexed
@Table(name = "orderLine")
public class OrderLine {

    @Id
    @Field
    private long id;

    @Field
    private String status;

    @Field
    private String comment;

}
``` 

You may notice the @IndexedEmbedded annotation. It makes sure we don't face an infinite indexing loop in case of nested JPA relationships. Make sure to mark your nested entities to avoid that. In the example above we mark both entities and all their attributes for full-text searching. 

Before moving forward with the actual searching, we need to trigger the Apache Lucene indexing:


```
	public void index() throws InterruptedException {
		FullTextEntityManager fullTextEntityManager = org.hibernate.search.jpa.Search
				.getFullTextEntityManager(entityManager);
		MassIndexer indexer = fullTextEntityManager.createIndexer();
		indexer.batchSizeToLoadObjects(100).threadsForSubsequentFetching(8).threadsToLoadObjects(4)
				.cacheMode(CacheMode.NORMAL).startAndWait();
	}
``` 

In the above example we launch 4 threads for loading our JPA entities, 8 threads for loading the lazy relationships and a batch size of 100 for the root entities. It also configures NORMAL caching strategy for the interaction between the indexer and the second level hibernate cache and the hibernate query cache. Once we trigger the indexing build, Lucene takes care of keeping them up-to-date. You can now go to your indexing directory and make sure you have generated index files.

We can now perform the searching itself. In the below example we search for the input searchString in all attributes of our indexed entities, using the Criteria API:


```
    public List<Order> search(String searchString) {
        try {
            if (StringUtils.isNotBlank(searchString)) {
                FullTextSession session = Search.getFullTextSession(entityManager.unwrap(Session.class));
                
                BooleanQuery booleanQuery = new BooleanQuery();
                String[] fields = new String[] {
                    "id",
                    "status",
                    "comment",
                    "orderLines.id",
                    "orderLines.status",
                    "orderLines.comment"
                };
                
                MultiFieldQueryParser parser = new MultiFieldQueryParser(Version.LUCENE_36, fields, new FranceAnalyzer());
                parser.setDefaultOperator(Operator.AND);
                parser.setAllowLeadingWildcard(true);
                
                org.apache.lucene.search.Query query = null;
                query = parser.parse(WordsTokenizer.replaceQueryChars(searchString));
                booleanQuery.add(query, Occur.MUST);
                     
                Criteria criteria = session.createCriteria(Order.class)
                	.createAlias("orderLines", "ol", JoinType.LEFT_OUTER_JOIN);
                
                FullTextQuery hibQuery = session.createFullTextQuery(booleanQuery, Order.class);
                
                hibQuery = session.createFullTextQuery(booleanQuery, Order.class)
                	.setCriteriaQuery(criteria);
                
                return hibQuery.list(); 
            } else {
                return new ArrayList<>();
            }
        } catch (ParseException e) {
            LOG.error(String.format("Orders search parser failed with criteraia %s", searchString) , e);
            return new ArrayList<>();
        }
     }
``` 

As you can see, we can customize the search by explicit list of attributes we want to scan. Having the indexes in place, Lucene uses them for effective full-text scanning of our JPA entities.

In conclusion, you definitely should evaluate hibernate-search with Apache Lucene in case you need full-text search capabilities in your JPA Hibernate based application. As usual, assessment of such solution depends on the context and the requirements.

Pros:

 - easy for plugging-in with annotation based configuration
 - Apache Lucene is a battle tested search engine
-  good choice in case of heavy text searching (fuzzy searches, wildcards)

Cons:

 - potential overengineering, apply if an adaptable user experience is a primary business requirement and it's not achievable with SQL based search
---
title: "How to implement audit logging in our application"
datePublished: Thu Dec 17 2020 16:07:46 GMT+0000 (Coordinated Universal Time)
cuid: ckit1fc8i06sv33s1exfj9r7t
slug: how-to-implement-audit-logging-in-our-application
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1608221255402/iYVsW_exF.jpeg
tags: java, databases, hibernate

---

Every once in a while we end up with requirement for implementing audit log solution for our database model, or at least covering part of it. Its widely spread approach when it is crucial to make sure each CRUD operation over particular database table is being tracked in terms of WHO, WHEN and WHY the change was made. In some cases it might be even legally required, depending on the business. In other cases its serving purely internal needs to keep track of all changes being made within some configuration or critical shared piece of data.

There are a lot of possible approaches when considering Audit Logging, so the question "Which approach should I use" has the well known answer "It depends". Each approach has particular pros and cons, which depends on two main decision prerequisites - what are the audit log requirements (what information should be tracked and stored) and which approach will help me fulfil those requirements most effectively (get the job done) and efficiently (easiest to be plugged in), depending on the current tech stack of the project. In this article I am covering some of the easiest and most common approaches in terms of Audit Logging, demonstrated with brief examples.

Lets consider we have a table holding information for our customers named CUSTOMERS with the following attributes: id, firstName, lastName and country. We can track the changes over the CUSTOMERS table in several ways.

### Database Triggers

Using database triggers for audit logging is one of the most common approaches, mainly because triggers are supported by most of the RDMS platforms. The example below is demonstrating an Oracle DB trigger over the CUSTOMER table.


```
CREATE OR REPLACE TRIGGER CUSTOMERS_TRG
  AFTER INSERT OR UPDATE OR DELETE ON CUSTOMERS
  REFERENCING NEW AS NEW OLD AS OLD
  FOR EACH ROW
DECLARE
  operation   CHAR(1);
BEGIN
  IF INSERTING THEN
     operation := 'I';
  ELSIF UPDATING THEN
     operation := 'U';
  ELSE
     operation := 'D';
  END IF;
  IF INSERTING THEN
    INSERT INTO CUSTOMERS_AUDIT_LOG (
           ID,
           CUSTOMER_ID
           FIRSTNAME,
           LASTNAME,
           COUNTRY,
           INSERTED_BY,
           INSERTED_ON,
           OPERATION)
    SELECT CUSTOMERS_LOG_SEQ.nextval,
           :NEW.ID,
           :NEW.FIRSTNAME,
           :NEW.LASTNAME,
           :NEW.COUNTRY,
            OSUSER,
            SYSTIMESTAMP,
            operation
     FROM v$session
     WHERE sid = (SELECT sid FROM v$mystat WHERE rownum = 1)
           AND rownum = 1;
  ELSIF UPDATING THEN
    INSERT INTO CUSTOMERS_AUDIT_LOG (
           ID,
           CUSTOMER_ID
           FIRSTNAME,
           LASTNAME,
           COUNTRY,
           INSERTED_BY,
           INSERTED_ON,
           OPERATION)
   SELECT MB.PAYMENT_OPTION_RULE_LOG_SEQ.nextval,
           :OLD.ID,
           :OLD.FIRSTNAME,
           :OLD.LASTNAME,
           :OLD.COUNTRY,
            OSUSER,
            SYSTIMESTAMP,
            operation
     FROM v$session
     WHERE sid = (SELECT sid FROM v$mystat WHERE rownum = 1)
           AND rownum = 1;
  ELSIF DELETING THEN
    INSERT INTO CUSTOMERS_AUDIT_LOG (
           ID,
           CUSTOMER_ID
           FIRSTNAME,
           LASTNAME,
           COUNTRY,
           INSERTED_BY,
           INSERTED_ON,
           OPERATION)
   SELECT MB.PAYMENT_OPTION_RULE_LOG_SEQ.nextval,
           :OLD.ID,
           :OLD.FIRSTNAME,
           :OLD.LASTNAME,
           :OLD.COUNTRY,
            OSUSER,
            SYSTIMESTAMP,
            operation
     FROM v$session
     WHERE sid = (SELECT sid FROM v$mystat WHERE rownum = 1)
           AND rownum = 1;
  END IF;
END;
``` 
As you can see, we have trigger over the CUSTOMERS table called CUSTOMERS_TRG which tracks the INSERT/UPDATE/DELETE operations and populates the corresponding data in the CUSTOMERS_AUDIT_LOG table. In our example we store a snapshot of the altered CUSTOMERS entry state along with CREATED_BY (taken from the Oracle v$session), INSERTED_ON (the current timestamp) and OPERATION (depending on the triggered event). Note that the exact implementation of the audit log table might differ. For example we might store the modified entry state as serialized object, or event do not store it at all, if we do not really need it. The exact syntax also varies with the RDBS platform used, but the idea is pretty much staying the same.

Pros: 
 - it makes sure the audit log is populated even in case of manual DB update
 - no affect on our business logic, it is completely decoupled and non-intrusive

Cons:
 - you can`t pass data which comes from outside the DB context
 - it gets triggered on any change, even when it is not expected
 - we have business logic exported outside of the application

### Hibernate/JPA Event Callbacks

The JPA specification provides mechanism for reacting to certain events that occur inside the persistence context via dedicated annotations. Lets consider the following JPA entity for our CUSTOMERS table:


```
@Entity
@Table(name = "CUSTOMERS")
@EntityListeners(CustomerListener.class)
public class Customer {
    @Id
    @Column(name = "ID")
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "CUSTOMER_ID")
    @SequenceGenerator(name = "CUSTOMER_ID", sequenceName = "CUSTOMER_ID_SEQ", allocationSize = 1)
    private Long id;

    @Column(name = "FIRSTNAME")
    private String firstName;

    @Column(name = "LASTNAME")
    private String lastName;

    @Column(name = "COUNTRY")
    private String country;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getFirstName() {
        return firstName;
    }

    public void setFirstName(Long firstName) {
        this.firstName = firstName;
    }

    public Long getLastName() {
        return lastName;
    }
    public void setLastName(Long lastName) {
        this.lastName = lastName;
    }

    public Long getCountry() {
        return country;
    }

    public void setCountry(Long country) {
        this.country = country;
    }
}
``` 
You may notice the 

```
@EntityListeners
``` 
annotation, which indicates that we defined a listener class, which reacts to persistence operations over our JPA entity. Here is how our example Listener looks like:


```
public class CustomerListener {
    private CustomerAuditLogDAO customerAuditLogDAO = Components.newComponent(CustomerAuditLogDAO.class);

    @PostPersist
    private void postPersist(Customer customer) {
        insertAuditLog(customer, EntityAction.INSERT);
    }

    @PreUpdate
    private void preUpdate(Customer customer) {
        insertAuditLog(customer, EntityAction.UPDATE);
    }

    @PreRemove
    private void preRemove(Customer customer) {
        insertAuditLog(customer, EntityAction.DELETE);
    }

    private void insertAuditLog(Customer customer, EntityAction entityAction) {
        customerAuditLogDAO.create(customer, entityAction);
    }
}
``` 
As you can see, JPA provides couple of Pre- and Post- annotations, which mark the corresponding handler method for each event. In our example, we defined handlers for persist, update and delete upon the Customer entity, where the input parameter of each handler represents the pre- or the post- state of the modified entity. Later on, the handler invokes the corresponding logic for persisting the actual audit log information.

Pros: 
 - full control over the audit log logic inside your application
 - it is AOP oriented and SOLID, no change required for your persistence layer classes

Cons:
 - it is only working inside the ORM persistence context, not on DB level (manual updates won`t get tracked)

### Hibernate Envers

The Envers module is a core Hibernate model that works both with Hibernate and JPA. In fact, you can use Envers anywhere Hibernate works whether that is standalone, inside WildFly or JBoss AS, Spring, Grails, etc. The Envers module aims to provide an easy auditing / versioning solution for entity classes.

In order for Envers to build the necessary historical audit tables and store the changes made to audited entities and their associations, the following is all that is required to get started:

Add the hibernate-envers dependency to your classpath:


```
<dependency>
  <groupId>org.hibernate</groupId>
  <artifactId>hibernate-envers</artifactId>
  <version>5.2.5.Final</version>
</dependency>
``` 


Annotate your entity or entity properties with the @Audited annotation:

```
@Entity
@Table(name = "CUSTOMERS")
@Audited
public class Customer {

    @Id
    @Column(name = "ID")
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "CUSTOMER_ID")
    @SequenceGenerator(name = "CUSTOMER_ID", sequenceName = "CUSTOMER_ID_SEQ", allocationSize = 1)
    private Long id;

    @Column(name = "FIRSTNAME")
    private String firstName;

    @Column(name = "LASTNAME")
    private String lastName;

    @Column(name = "COUNTRY")
    private String country;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getFirstName() {
        return firstName;
    }

    public void setFirstName(Long firstName) {
        this.firstName = firstName;
    }

    public Long getLastName() {
        return lastName;
    }
    public void setLastName(Long lastName) {
        this.lastName = lastName;
    }

    public Long getCountry() {
        return country;
    }

    public void setCountry(Long country) {
        this.country = country;
    }
}
``` 

Having that in place, Hibernate automatically creates auditing tables CUSTOMERS_AUD table which basically stores snapshot of the modified Customer with all attributes (unless you annotate only some of them). More details on this approach can be found in the official Hibernate documentation. 

Pros: 
 - it is AOP oriented and SOLID, no change required for your persistence layer classes
 - no need to bother with audit tables creation

Cons:
 - it is only working inside the ORM persistence context, not on DB level (manual updates won`t get tracked)
 - less control over the audit log logic

### Spring Data JPA

Spring Data provides sophisticated support to transparently keep track of who created or changed an entity and the point in time this happened. To benefit from that functionality you have to equip your entity classes with auditing metadata that can be defined either using annotations or by implementing an interface. This approach might be considered in case your tech stack already lay upon the Spring ecosystem.

The first prerequisite for enablement Spring Data auditing is adding the following annotation in our configuration class:


```
@EnableJpaAuditing
``` 

Having that in place, we have to annotate our JPA entity, similar to what we had for the hibernate approach. This time, the listener is provided from Spring out-of-the-box.


```
 @Entity
@Table(name = "CUSTOMERS")
@EntityListeners(AuditingEntityListener.class)
public class Customer {

    @Id
    @Column(name = "ID")
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "CUSTOMER_ID")
    @SequenceGenerator(name = "CUSTOMER_ID", sequenceName = "CUSTOMER_ID_SEQ", allocationSize = 1)
    private Long id;

    @Column(name = "FIRSTNAME")
    private String firstName;

    @Column(name = "LASTNAME")
    private String lastName;

    @Column(name = "COUNTRY")
    private String country;

    @Column(name = "CREATED_BY")
    @CreatedBy
    private String createdBy;

    @Column(name = "MODIFIED_BY")
    @LastModifiedBy
    private String modifiedBy;

}
``` 

As you can see, we use two additional annotations which automatically populate CREATED_BY and MODIFIED_BY. And of course, you have to add the new columns in your DB table. Note that in order for LastModifiedBy to work, you have to use Spring Security as authentication provider. This approach is slightly different since it is performing state snapshots, but rather keeps track of the last entity update only.

Pros: 
 - it is AOP oriented and SOLID, no change required for your persistence layer classes
 - no need to bother with audit tables creation
 - out-of-the-box solution for Spring based applications

Cons:
 - it is only working inside the ORM persistence context, not on DB level (manual updates won`t get tracked)
 - less control over the audit log logic
 - providing less flexibility and less data, no state machine, just last updated information

### In conclusion

This article aims to briefly introduce some of the most common audit logging approaches and do not aim to cover all possible solutions. There might be scenarios where we might need completely custom solution or store different type of audit information. So we always have to make sure that the chosen solution suites our needs the best.


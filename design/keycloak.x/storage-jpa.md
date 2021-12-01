# Keycloak X - Storage - Jpa/Hibernate persistence layer

* **Status**: Draft
* **JIRA**: https://issues.redhat.com/browse/KEYCLOAK-18562
# Goals of this document

This document aims to address the following items:

* Follow up on no-downtime strategy presented in the [the Storage / Persistence
  layer proposal](storage-persistence.md)
* Present a way how to store and query data using JPA/Hibernate so that we achieve 
the no-downtime requirements

# Non-Goals of this document

This document aims to _not_ address the following items:

* Provide a way how to replace the existing JPA layer

# Technologies

* [Hibernate](https://hibernate.org/orm/) implementation of the Java Persistence 
API (JPA) specification.
* [Jackson](https://github.com/FasterXML/jackson) Java JSON library

# Introduction

[The Storage / Persistence layer proposal](storage-persistence.md) suggests the
following strategy to achieve 0-downtime upgrade store:

1. The store can read objects from the oldest ones to the current version
2. The store can read objects from one version following the current one
3. Objects are only updated when written to.
4. Number of schema changes should be kept at absolute minimum
5. Schema changes can be postponed and run at chosen time (even if that means
   running a degraded service)

# Blob storage

For fulfilling the requirement of minimum of schema changes, new implementation 
should use some kind of blob storage. Majority of fields could be stored within
the blob column while the most used fields could be stored in standard columns.

## Postgresql JSONB datatype

Representing data as JSON is one of the possibilities for storing such a data. 
PostgreSQL offers datatype JSONB which is almost identical as JSON, there is 
a diffence that JSON data is stored as exact copy of json input text, while 
JSONB stores data in binary form. Although it has its disadvantages like slightly
slower insert (conversion overhead), lack of postgresql column statistics (some 
queries, especially aggregate ones, could be slower) and larger disk space usage.
It also has some benefits, possibility to query data within the json, significantly 
faster read and index support.

An example of database schema could look like:
``` sql
create table object (
    id uuid primary key not null,
    metadata jsonb
);
```

### JSONB support in Hibernate

By default, hibernate doesn't have native support for `jsonb` datatype. It'd need 
to be added by implementing one of Hibernate's StandardBasicType interfaces. It 
requires providing a way how to transform Java type into Sql type and vice versa.
The implementation should be registered using custom dialect.

Jackson lib can be used to perform serialization and deserialization.

An example how new type can be used:
``` java
@Type(type = "jsonb")
@Column(columnDefinition = "jsonb")
private final JpaObjectMetadata metadata = new JpaObjectMetadata();
```

## Lazy loaded fields

In some cases there is required only subset of fields. Therefore it might make sense 
to load some fields strait away and some might remain lazily loaded.

To avoid processing json field in all cases it seems convenient to have some fields 
in standard columns. 

### Generated columns

To gain most benefits from it and to minimize drawbacks (lack of statistics) new 
implementation can use [generated columns](https://www.postgresql.org/docs/14/ddl-generated-columns.html). 
The value of that column is always computed from other columns, even JSONB column 
could be used, so postgresql could create its statistics and the design is still 
resilient against schema changes at the same time. 

An example of database schema could then look like:
``` sql
create table object (
    id uuid primary key not null,
    realmId varchar(36) generated always as (metadata->>'fRealmId') stored,
    clientId varchar(255) generated always as (metadata->>'fClientId') stored,
    metadata jsonb
);
```

In entity object it can be specified that some fields are generated (by database, 
not by hibernate) and it's not expected Hibernate should insert or udtade its value, e.g.:

``` java
@Generated(GenerationTime.NEVER)
@Column(insertable = false, updatable = false)
private String realmId;
```

### lazy loaded fields in Hibernate

By default, Hibernate loads all properties eagerly (associations can be loaded 
lazily), to be able to use lazy loaded fields feature, lazy initialization 
needs to be enabled by [bytecode enhancement](https://docs.jboss.org/hibernate/orm/5.4/topical/html_single/bytecode/BytecodeEnhancement.html):

``` xml
<plugin>
    <groupId>org.hibernate.orm.tooling</groupId>
    <artifactId>hibernate-enhance-maven-plugin</artifactId>
    <version>${hibernate.version}</version>
    <executions>
        <execution>
            <configuration>
                <enableLazyInitialization>true</enableLazyInitialization>
            </configuration>
            <goals>
                <goal>enhance</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

The fields in entity objects could then be marked as lazy loaded by annotation:

``` java
@Basic(fetch = FetchType.LAZY)
private String realmId;
```

Hibernate provides few ways how to load just subset of fields from database.
DTO (Data Transfer Object) projection in criteria query could be one option.
An example of such projection could look like:

``` java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<JpaObjectEntity> query = cb.createQuery(JpaObjectEntity.class);
Root<JpaObjectEntity> root = query.from(JpaObjectEntity.class);
query.select(cb.construct(JpaObjectEntity.class, 
        root.get("id"), 
        root.get("realmId"), 
        root.get("clientId")
));
```

`JpaObjectEntity` has to have `public` constructor with parameters corresponding 
to this call. Hibernate uses this constructor for constructing the objects.
For this example it'll look like:

``` java
public JpaObjectEntity(UUID id, String realmId, String clientId) {
    this.id = id;
    this.realmId = realmId;
    this.clientId = clientId;
}
```

When reading those fields it should be tested whether the json is already loaded or not.

``` java
@Override
public String getClientId() {
    if (isMetadataInitialized()) return metadata.getClientId();
    return clientId;
}

public boolean isMetadataInitialized() {
    // we assume realmId is either set (initialized) or null (not initialized)
    return metadata.getRealmId() != null;
}
```

Entities obtained by the projection are in `detached` state. So any updates performed 
on the object won't take any effect in the database. To address it and also for cases 
when other fields than originally loaded are needed, `delegate` concept could be 
introduced. There are already `MapObjectEntityDelegate` classes generated and it can be used 
for this purpose. An implementation of `DelegateProvider` should be introduced where 
above logic could be implemented.

`MapKeycloakTransaction` interface describes two `read` methods. One is `read(String id)`
and second is `read(QueryParameters qp)`. When `read(String id)` is called it's highly
probable there will be either some write operation performed on the entity or majority
of fields will be needed. Therefore it makes sense to load the json in this case. While 
when calling `read(QueryParameters qp)` it's chance that only subset of fields will be
needed. It can be used DTO projection and delegate concepts for this case.

# No downtime upgrades

To be able to support 0-downtime upgrades, each storage implementation should have 
an information what is the current supported [_current store version_](storage-persistence.md#_def_current_store_version). 
Let's call it `SUPPORTED_VERSION`. Each object in database should have `entityVersion` 
([_entity schema version_](storage-persistence.md#_def_entity_schema_version)) 
field. Based on that it can be decided whether the object needs some kind of 
migration or not.

When reading object from database we should deserialize object from json field 
into java object. During the deserialization we can obtain `com.fasterxml.jackson.databind.node.ObjectNode` 
object, from the `ObjectNode` `entityVersion` could be extracted and if it would 
be lower than the currently `SUPPORTED_VERSION`, it could be performed all required 
migrations on that `ObjectNode`. Migrated `ObjectNode` then can be deserialized 
into particular entity object. 

``` java
ObjectNode tree = MAPPER.readValue(json, ObjectNode.class);
JsonNode ev = tree.get("entityVersion");
if (ev == null || ! ev.isInt()) throw new IllegalArgumentException("unable to read entity version from " + json);

int entityVersion = ev.asInt();

if (entityVersion > SUPPORTED_VERSION + 1) {
    throw new IllegalArgumentException("Incompatible entity version: " + entityVersion + ", supportedVersion: " + SUPPORTED_VERSION);
}

if (entityVersion < SUPPORTED_VERSION) {
    tree = JpaObjectMigration.migrateTreeTo(entityVersion, SUPPORTED_VERSION, tree);
}
return MAPPER.treeToValue(tree, JpaObjectMetadata.class);
```

If the `entityVersion` is older than `SUPPORTED_VERSION + 1` `IllegalArgumentException` 
is thrown to fulfill requirement described in [Version Compatibility (VC)](storage-persistence.md#_def_current_store_version_).

To fulfill the requirement for updating an object upon write operation, all write 
operations should be guarded on the entity objects and update `entityVersion`
of the object to current `SUPPORTED_VERSION` in case it's called. 

``` java
@Override
public void setClientId(String clientId) {
    checkEntityVersionForUpdate();
    metadata.setClientId(clientId);
}

private void checkEntityVersionForUpdate() {
    Integer ev = getEntityVersion();
    if (ev != null && ev < SUPPORTED_VERSION) {
        setEntityVersion(SUPPORTED_VERSION);
    }
}
```

If no write operation is called, the object remains unchanged. Next time it is 
loaded migration is performed again.

PoC of this approach using jackson library could be found in [keycloak-playground](https://github.com/keycloak/keycloak-playground/tree/main/no-downtime-upgrade/src/main/java/org/keycloak/playground/nodowntimeupgrade/naive_jackson) project. 

In keycloak there are generated [stateless representations](storage-persistence.md#_def_stateless_representation_).
The representation can be used for storing its content within `jsonb` field. On top
of that `entityVersion` is needed. Resulting metadata object could look like:

``` java
public class JpaObjectMetadata extends MapObjectEntityImpl implements Serializable {

    private Integer entityVersion;

    public Integer getEntityVersion() {
        return entityVersion;
    }

    public void setEntityVersion(Integer entityVersion) {
        this.entityVersion = entityVersion;
    }
}
```
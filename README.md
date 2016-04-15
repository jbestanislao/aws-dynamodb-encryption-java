# Client-side Encryption for Amazon DynamoDB

The **[Amazon DynamoDB][ddb] Client-side Encryption in Java** supports encryption and signing of your data when stored in Amazon DynamoDB.

A typical use of this library is when you are using [DynamoDBMapper][ddbmapper], where transparent protection of all objects serialized through the mapper can be enabled via configuring an [AttributeEncryptor][attrencryptor].

For more advanced use cases where tighter control over the encryption and signing process is necessary, the low-level [DynamoDBEncryptor][ddbencryptor] can be used directly.

## Getting Started

Suppose you have created ([sample code][createtable]) a DynamoDB table "MyStore", and want to store some Book objects.  The security requirement involves classifying the attributes Title and Authors as sensitive information.  This is how the Book class may look like:

```java
@DynamoDBTable(tableName="MyStore")
public class Book {
    private Integer id;
    private String title;
    private String ISBN;
    private Set<String> bookAuthors;
    private String someProp;
 
    // Not encrypted because it is a hash key    
    @DynamoDBHashKey(attributeName="Id")  
    public Integer getId() { return id;}
    public void setId(Integer id) {this.id = id;}
 
    // Encrypted by default
    @DynamoDBAttribute(attributeName="Title")  
    public String getTitle() {return title; }
    public void setTitle(String title) { this.title = title; }
 
    // Specifically not encrypted
    @DoNotEncrypt
    @DynamoDBAttribute(attributeName="ISBN")  
    public String getISBN() { return ISBN; }
    public void setISBN(String ISBN) { this.ISBN = ISBN; }
 
    // Encrypted by default
    @DynamoDBAttribute(attributeName = "Authors")
    public Set<String> getBookAuthors() { return bookAuthors; }
    public void setBookAuthors(Set<String> bookAuthors) { this.bookAuthors = bookAuthors; }
 
    // Not encrypted nor signed
    @DoNotTouch
    public String getSomeProp() { return someProp;}
    public void setSomeProp(String someProp) {this.someProp = someProp;}
}
```

As a typical use case of [DynamoDBMapper][ddbmapper], you can easily save and retrieve a Book object to and from Amazon DynamoDB _without encryption (nor signing)_.  For example,

```java
    AmazonDynamoDBClient client = new AmazonDynamoDBClient(...);
    DynamoDBMapper mapper = new DynamoDBMapper(client);
    Book book = new Book();
    book.setId(123);
    book.setTitle("Secret Book Title ");
    // ... etc. setting other properties

    // Saves the book unencrypted to DynamoDB
    mapper.save(book);

    // Loads the book back from DynamoDB
    Book bookTo = new Book();
    bookTo.setId(123);
    Book bookTo = mapper.load(bookTo);

```

To enable transparent encryption and signing, simply specify the necessary encryption material via an [EncryptionMaterialsProvider][materialprovider].  For example:

```java
    AmazonDynamoDBClient client = new AmazonDynamoDBClient(...);
    SecretKey cek = ...;        // Content encrypting key
    SecretKey macKey =  ...;    // Signing key
    EncryptionMaterialsProvider provider = new SymmetricStaticProvider(cek, macKey);
    mapper = new DynamoDBMapper(client, DynamoDBMapperConfig.DEFAULT,
                new AttributeEncryptor(provider));
    Book book = new Book();
    book.setId(123);
    book.setTitle("Secret Book Title ");
    // ... etc. setting other properties

    // Saves the book both encrypted and signed to DynamoDB
    mapper.save(bookFrom);

    // Loads the book both with signature verified and decrypted from DynamoDB
    Book bookTo = new Book();
    bookTo.setId(123);
    Book bookTo = mapper.load(bookTo);

```

Note that by default all attributes except the primary keys are both encrypted and signed for maximum security.  To selectively disable encryption, the annotation [@DoNotEncrypt][donotencrypt] can be used as shown in the [Book](#getting-started) class above.  To disable both encryption and signing, the annotation [@DoNotTouch][donottouch] can be used.

There is a variety of existing [EncryptionMaterialsProvider][materialprovider] implementations that you can use to provide the encryption material, including [KeyStoreMaterialsProvider][keystoreprovider] which makes use of a Java keystore.  Alternatively, you can also plug in your own custom implementation.

### Downloads

You can download the [latest snapshot release][download] or pick it up from Maven:

```xml
  <dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-dynamodb-encryption-java</artifactId>
    <version>0.0.6-SNAPSHOT</version>
  </dependency>
```

Don't forget to enable the download of snapshot jars from Maven:

```xml
  <profiles>
    <profile>
       <id>allow-snapshots</id>
          <activation><activeByDefault>true</activeByDefault></activation>
       <repositories>
         <repository>
           <id>snapshots-repo</id>
           <url>https://oss.sonatype.org/content/repositories/snapshots</url>
           <releases><enabled>false</enabled></releases>
           <snapshots><enabled>true</enabled></snapshots>
         </repository>
       </repositories>
     </profile>
  </profiles>
```

## Supported Algorithms

For content encryption, the encryption algorithm is determined by the user specified [SecretKey][secretkey], as long as it is a block cipher that can be used with the encryption mode "CBC" and "PKCS5Padding".  Typically, this means "AES".

For signing, the user specified signing key can be either symmetric or asymmetric.  For asymmetric signing (where the user would provide a signing key in the form of a [PrivateKey][privatekey]), the default algorithm is "SHA256withRSA".  For symmetric signing (where the user would provide the signing key in the form of a [SecretKey][secretkey]), the algorithm would be determined by the provided key.  A typical algorithm for a symmetric signing key is "HmacSHA256".

## FAQ

1. Do the content-encrypting key and signing key get encrypted and stored along side with the data in Amazon DynamoDB ?
  * No, neither the content-encrypting key nor the signing key get persisted by this library.  However, in order to locate the material for decryption purposes, the identifying information (i.e. material descriptions) for the encryption material is indeed stored along side with the data in Amazon DynamoDB.  In particular, the user specified [EncryptionMaterialsProvider][materialprovider] is responsible for not only providing the keys, but also the corresponding material descriptions.

2. How is the IV generated and where is it stored ?
  * For each attribute that needs to be encrypted, a unique IV is randomly generated, and get stored along side with the binary representation of the attribute value.

3. How many bits are used for the random IV ?
  * The bit size used for each random IV is the same as the block size of the block cipher used for content encryption.  The IV bit-size therefore depends on the specific algorithm of the content encrypting key provided by the user.  Typically this means AES, or 128 bits.

4. What is the key length for the content encrypting key ?
  * This depends on the specific content encrypting key provided by the user.  A typical length of an AES key is 128 bits or 256 bits.

## Known Limitations

1. Currently the new data types in [Amazon DynamoDB][ddb] including Map, List, Boolean, and NULL are not yet supported by this library.  In particular, this library would fail fast upon detecting the use of these new data types.  We expect to support the new types soon in future releases.

2. During retrieval of an item, all the attributes of the item that have been involved for encryption or signing must also be included for signature verification.  Otherwise, the signature would fail to verify.

[attrencryptor]: src/main/java/com/amazonaws/services/dynamodbv2/datamodeling/AttributeEncryptor.java
[createtable]: https://github.com/aws/aws-sdk-java/blob/master/src/samples/AmazonDynamoDBDocumentAPI/quick-start/com/amazonaws/services/dynamodbv2/document/quickstart/A_CreateTableTest.java
[ddb]: http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html
[ddbencryptor]: src/main/java/com/amazonaws/services/dynamodbv2/datamodeling/encryption/DynamoDBEncryptor.java
[ddbmapper]: http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/dynamodbv2/datamodeling/DynamoDBMapper.html
[donotencrypt]: src/main/java/com/amazonaws/services/dynamodbv2/datamodeling/encryption/DoNotEncrypt.java
[donottouch]: src/main/java/com/amazonaws/services/dynamodbv2/datamodeling/encryption/DoNotTouch.java
[keystoreprovider]: src/main/java/com/amazonaws/services/dynamodbv2/datamodeling/encryption/providers/KeyStoreMaterialsProvider.java
[materialprovider]: src/main/java/com/amazonaws/services/dynamodbv2/datamodeling/encryption/providers/EncryptionMaterialsProvider.java
[privatekey]: http://docs.oracle.com/javase/7/docs/api/java/security/PrivateKey.html
[secretkey]: http://docs.oracle.com/javase/7/docs/api/javax/crypto/SecretKey.html
[download]: https://github.com/awslabs/aws-dynamodb-encryption-java/releases/tag/0.0.6-SNAPSHOT

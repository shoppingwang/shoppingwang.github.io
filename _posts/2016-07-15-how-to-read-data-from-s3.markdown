---
layout:     post
title:      "Spark并行访问S3"
subtitle:   "使用AWS SDK并行访问S3的数据"
date:       2016-07-15
author:     "xp"
header-img: "img/post-bg-unix-linux.jpg"
header-mask: 0.3
catalog:    true
tags:
    - AWS
    - S3
---

> 使用[SDK](https://github.com/aws/aws-sdk-java)访问AWS服务。


### Importing the BOM
```
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.amazonaws</groupId>
      <artifactId>aws-java-sdk-bom</artifactId>
      <version>1.11.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

### Using the SDK Maven modules
```
<dependencies>
  <dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-ec2</artifactId>
  </dependency>
  <dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-s3</artifactId>
  </dependency>
  <dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-dynamodb</artifactId>
  </dependency>
</dependencies>
```

### Sample Code

```java
import com.amazonaws.services.s3._, model._
import com.amazonaws.auth.BasicAWSCredentials

val request = new ListObjectsRequest()
request.setBucketName(bucket)
request.setPrefix(prefix)
request.setMaxKeys(pageLength)
def s3 = new AmazonS3Client(new BasicAWSCredentials(key, secret))

// Note that this method returns truncated data if longer than the "pageLength" above. You might need to deal with that.
val objs = s3.listObjects(request) 
sc.parallelize(objs.getObjectSummaries.map(_.getKey).toList).flatMap { key => Source.fromInputStream(s3.getObject(bucket, key).getObjectContent: InputStream).getLines }
```

### Relative
* [How NOT to pull from S3 using Apache Spark](http://tech.kinja.com/how-not-to-pull-from-s3-using-apache-spark-1704509219) 
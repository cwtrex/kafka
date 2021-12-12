# Purpose:
cwtrex_custom to ensure the below enhancement details are not lost in time.

#### Needed functionality for Kafka connect transform.  Added from:
* https://cwiki.apache.org/confluence/display/KAFKA/KIP+678%3A+New+Kafka+Connect+SMT+for+plainText+%3D%3E+Struct%28or+Map%29+with+Regex
* https://github.com/apache/kafka/pull/7965
* https://issues.apache.org/jira/browse/KAFKA-9436
* https://github.com/apache/kafka/tree/ecabbb3d8e4334acd259229eeee9bc8da5fe37d7/connect

#### Related files:
connect/transforms/src/main/java/org/apache/kafka/connect/transforms/ToStructByRegexTransform.java
connect/transforms/src/main/java/org/apache/kafka/connect/transforms/util/GroupRegexValidator.java
connect/transforms/src/test/java/org/apache/kafka/connect/transforms/ToStructByRegexTransformTest.java

#### First Example of Use:
1. String parse ( with timemillis )
```
{
   "code" : "dev_kafka_pc001_1580372261372"
   ,"recode1" : "a"
   ,"recode2" : "b" 
}
```
```
"transforms": "RegexTransform",
"transforms.RegexTransform.type": "org.apache.kafka.connect.transforms.ToStructByRegexTransform$Value",

"transforms.RegexTransform.struct.field": "message",
"transforms.RegexTransform.regex": "^(.{3,4})_(.*)_(pc|mw|ios|and)([0-9]{3})_([0-9]{13})" "transforms.RegexTransform.mapping": "env,serviceId,device,sequence,datetime:TIMEMILLIS"
```

2. plain text apache log
```
"111.61.73.113 - - [08/Aug/2019:18:15:29 +0900] \"OPTIONS /api/v1/service_config HTTP/1.1\" 200 - 101989 \"http://local.test.com/\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.142 Safari/537.36\""
```
SMT connect config with regular expression below can easily transform a plain text to struct (or map) data.
```
"transforms": "RegexTransform",
"transforms.RegexTransform.type": "org.apache.kafka.connect.transforms.ToStructByRegexTransform$Value",

"transforms.RegexTransform.struct.field": "message",
"transforms.RegexTransform.regex": "^([\\d.]+) (\\S+) (\\S+) \\[([\\w:/]+\\s[+\\-]\\d{4})\\] \"(GET|POST|OPTIONS|HEAD|PUT|DELETE|PATCH) (.+?) (.+?)\" (\\d{3}) ([0-9|-]+) ([0-9|-]+) \"([^\"]+)\" \"([^\"]+)\""

"transforms.RegexTransform.mapping": "IP,RemoteUser,AuthedRemoteUser,DateTime,Method,Request,Protocol,Response,BytesSent,Ms:NUMBER,Referrer,UserAgent"
```

#### Second Example of Use:


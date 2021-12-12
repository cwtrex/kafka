# Purpose:
cwtrex_custom to ensure the below enhancement details are not lost in time.

All previous credit for pushing this goes to whsoul and ssp-devcenter

#### Needed functionality for Kafka connect transform.  Added from:
* https://cwiki.apache.org/confluence/display/KAFKA/KIP+678%3A+New+Kafka+Connect+SMT+for+plainText+%3D%3E+Struct%28or+Map%29+with+Regex
* https://github.com/apache/kafka/pull/7965
* https://issues.apache.org/jira/browse/KAFKA-9436
* https://github.com/apache/kafka/tree/ecabbb3d8e4334acd259229eeee9bc8da5fe37d7/connect

#### Related files:
* connect/transforms/src/main/java/org/apache/kafka/connect/transforms/ToStructByRegexTransform.java
* connect/transforms/src/main/java/org/apache/kafka/connect/transforms/util/GroupRegexValidator.java
* connect/transforms/src/test/java/org/apache/kafka/connect/transforms/ToStructByRegexTransformTest.java

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

Input:
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
Output:
```
{
   "IP" : "111.61.73.113"
   ,"RemoteUser" : "-"
   ,"AuthedRemoteUser" : "-"
   ,"DateTime" : "08/Aug/2019:18:15:29 +0900"
   ,"Method" : "OPTIONS"
   ,"Request" : "/api/v1/service_config"
   ,"Protocol" : "HTTP/1.1"
   ,"Response" : "200"
   ,"BytesSent" : "-"
   ,"Ms" : "101989"
   ,"Referrer" : "http://local.test.com"
   ,"UserAgent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.142 Safari/537.36"
}   
```

3. Parsing URL

Input:
```
{
  "url" :
 "https://kafka.apache.org/documentation/#connect"
}
```

Output:
```
{
  "protocol" : "https"
  ,"domain" : "kafka.apache.org"
  ,"path" : "documentation/#connect"
}
```

#### Public Interfaces
Briefly list any new interfaces that will be introduced as part of this proposal or any existing interfaces that will be removed or changed. The purpose of this section is to concisely call out the public contract that will come along with this feature.

name                                   | description                                     | type   | default | valid values              | importance
----                                   | -----------                                     | ----   | ------- | ------------              | ----------
transforms.RegexTransform.struct.field | target fieldName In Case struct input           | String | message |                           | medium
transforms.RegexTransform.regex        | Ordered Regex Group Mapping Keys (with :{TYPE}) | String |         | regular expression string | medium
transforms.RegexTransform.mapping      | String Regex Group Pattern                      | String |         | comma seperated names     | medium

#### Example Configs

###### Usecase1. Transform Config
```
"transforms": "RegexTransform",
"transforms.RegexTransform.type": "org.apache.kafka.connect.transforms.ToStructByRegexTransform$Value",

"transforms.RegexTransform.regex": "^([\\d.]+) (\\S+) (\\S+) \\[([\\w:/]+\\s[+\\-]\\d{4})\\] \"(GET|POST|OPTIONS|HEAD|PUT|DELETE|PATCH) (.+?) (.+?)\" (\\d{3}) ([0-9|-]+) ([0-9|-]+) \"([^\"]+)\" \"([^\"]+)\""

"transforms.RegexTransform.mapping": "IP,RemoteUser,AuthedRemoteUser,DateTime,Method,Request,Protocol,Response,BytesSent,Ms:NUMBER,Referrer,UserAgent"
```

Usecase2. Transform Config
```
"transforms": "RegexTransform",
"transforms.RegexTransform.type": "org.apache.kafka.connect.transforms.ToStructByRegexTransform$Value",

"transforms.RegexTransform.struct.field": "url",
"transforms.RegexTransform.regex": "^(https?):\\/\\/([^/]*)/(.*)"

"transforms.RegexTransform.mapping": "protocol,domain,path"
```

#### Proposed Changes

Only one main abstract class and one validator class added : ToStructByRegexTransform, GroupRegexValidator

this includes:
1. describe/declare part
2. main functions
3. type case support

```
public abstract class ToStructByRegexTransfo
rm<R extends ConnectRecord<R>> implements Transformation<R> {
    public static final String OVERVIEW_DOC = "Generate key/value Struct objects supported by ordered Regex Group"
        + "<p/>Use the concrete transformation type designed for the record key (<code>" + Key.class.getName() + "</code>) "
        + "or value (<code>" + Value.class.getName() + "</code>).";

    private static final String TYPE_DELIMITER = ":";

    private interface ConfigName {
        String REGEX = "regex";
        String MAPPING_KEY = "mapping";
        String STURCT_INPUT_KEY_NAME = "struct.field";
    }


    public static final ConfigDef CONFIG_DEF = new ConfigDef()
        .define(ConfigName.REGEX, ConfigDef.Type.STRING, ConfigDef.NO_DEFAULT_VALUE, new GroupRegexValidator(), ConfigDef.Importance.MEDIUM,
            "String Regex Group Pattern.")
        .define(ConfigName.MAPPING_KEY, ConfigDef.Type.LIST, ConfigDef.NO_DEFAULT_VALUE, ConfigDef.Importance.MEDIUM,
            "Ordered Regex Group Mapping Keys ( with :{TYPE} )")
        .define(ConfigName.STURCT_INPUT_KEY_NAME, ConfigDef.Type.STRING, "message", ConfigDef.Importance.MEDIUM,
            "target fieldName In Case struct input");


    private static final String PURPOSE = "Transform Struct by regex group mapping"
```

main functions interface
```
@Override
    public R apply(R record) {
        if (operatingSchema(record) == null) {
            return applySchemaless(record);
        } else {
            return applyWithSchema(record);
        }
    }

    private R applySchemaless(R record) {
        ...
    }

    private R applyWithSchema(R record) {
        ...
    }
```

Additional Type Convert Support

You can use TypeCase below with regex config

"transforms.RegexTransform.regex": "env,serviceId,device,sequence,datetime:TIMEMILLIS"
```
public enum TYPE{
        STRING
        ,NUMBER
        ,FLOAT
        ,BOOLEAN
        ,TIMEMILLIS
    }

    private static class KeyData{
        private String name;
        private TYPE type;

        private KeyData(String name, String type){
            this.name = name;
            this.type = type != null ? TYPE.valueOf(type) : TYPE.STRING;
        }

        public String getName(){
            return this.name;
        }

        public TYPE type(){
            return this.type;
        }

        private Object castJavaType(String value){
            try {
                switch (this.type) {
                    case STRING: return value;
                    case NUMBER: return Long.valueOf(value);
                    case FLOAT: return Float.valueOf(value);
                    case BOOLEAN: return Boolean.valueOf(value);
                    case TIMEMILLIS: return new Date(Long.valueOf(value));
                    default: return value;
                }
            }catch (Exception e){
                return value;
            }
        }


        private Schema getTypeSchema(){
            switch (this.type){
                case STRING: return Schema.STRING_SCHEMA;
                case NUMBER: return Schema.INT64_SCHEMA;
                case FLOAT: return Schema.FLOAT64_SCHEMA;
                case BOOLEAN: return Schema.BOOLEAN_SCHEMA;
                case TIMEMILLIS: return Timestamp.SCHEMA;
                default: return Schema.STRING_SCHEMA;
            }
        }
    }
```

Compatibility, Deprecation, and Migration Plan

this is a new SMT. no older consider. justconfigure connectors with new SMT Config

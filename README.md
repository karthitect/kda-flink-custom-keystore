## Overview
This sample code describes how to configure your [Kinesis Data Analytics for Java](https://aws.amazon.com/kinesis/data-analytics/) application to use a custom keystore when communicating with Kafka.

We need a way to deliver our custom keystore to the KDA/Flink environment, since we don't have access to the runners directly. Overriding the `open()` method of the FlinkKafkaConsumer gives us a way to place our custom keystore on every runner, and ensure that the keystore is available across restarts and runner replacements.

### Overriding the `FlinkKafkaConsumer` class

In the overriden `open()` method of [`CustomFlinkKafkaConsumer`](https://github.com/karthitect/kda-flink-custom-keystore/blob/master/flink-app/src/main/java/com/amazonaws/services/kinesisanalytics/CustomFlinkKafkaConsumer.java) (inherited from [`FlinkKafkaConsumer`](https://ci.apache.org/projects/flink/flink-docs-stable/dev/connectors/kafka.html)), we can write our custom keystore to `/tmp` as shown below:

```
...
    /*
     We need to override the 'open' method of the FlinkKafkaConsumer to drop our custom
     keystore. This is necessary so that certs are available to be picked up in spite of
     runner restarts and replacements.
     */
    @Override
    public void open(Configuration configuration) throws Exception {
        // write keystore to /tmp
        // make sure that keystore is in JKS format for KDA/Flink
        dropFile("/tmp");

        super.open(configuration);
    }

    private void dropFile(String destFolder) throws Exception
    {
        InputStream input = null;
        OutputStream outStream = null;

        try {
            ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
            input = classLoader.getResourceAsStream("kafka.client.truststore.jks");
            byte[] buffer = new byte[input.available()];
            input.read(buffer);

            File destDir = new File(destFolder);
            File targetFile = new File(destDir, "kafka.client.truststore.jks");
            outStream = new FileOutputStream(targetFile);
            outStream.write(buffer);
            outStream.flush();
        }
        catch (Exception ex)
        {
            System.out.println(ex.getMessage());

            if(input != null) {
                input.close();
            }

            if(outStream != null) {
                outStream.close();
            }
        }
    }
...
```

### Specifying keystore location

In our [`main()` function](https://github.com/karthitect/kda-flink-custom-keystore/blob/master/flink-app/src/main/java/com/amazonaws/services/kinesisanalytics/KDAFlinkStreamingJob.java), we configure the truststore location as shown below:

```
...
// configure location where runtime will look for custom truststore
sourceProps.setProperty("ssl.truststore.location", "/tmp/kafka.client.truststore.jks");
...
```

### Note about keystore format

The KDA/Flink runtime expects the keystore to be in JKS format. For instance, if your keystore is in PKCS12 format, you'll get an "invalid keystore format" error when running your application. This [HOWTO](https://github.com/karthitect/keystore-conversion) describes the process for converting keystores.
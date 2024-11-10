### All about my https://github.com/feehaam/Centralized_Logging_ELK repo
#### Elastic search runs on port 9200 with basic settings
![img.png](resources/img.png)

#### Kibana runs after elastic search and connects to it
![img_1.png](resources/img_1.png)

#### Logstash runs after elastic search and runs ports for different types of logs channel, here the first and the second one we will use from our two applications. It also reads the logstash.yml file and configures from it. 
![img_2.png](resources/img_2.png)
![img_8.png](resources/img_8.png)

#### Spring app 2 is set with minimal configuration, sends logs directly to logstash through TCP socket
![img_3.png](resources/img_3.png)

#### But as the TCP socket is not consistent & insecure, spring app 1 use filebeat to send logs to logstash, so its logback-spring.xml setting is little different
![img_4.png](resources/img_4.png)

#### Note that, there is also a file reference in the xml, as filebeat reads from file here so we need to write the application logs (not just in console only but also) in a file too. So we configured in application setting to write in /logs/app.log file
![img_5.png](resources/img_5.png)

#### Filebeat reads from the file and stores in its own docker container storage
![img_6.png](resources/img_6.png)

#### It beats and checks input from the defined path (its container storage for logs) and sends to output host (logstash)
![img_7.png](resources/img_7.png)

#### Finally, logstash config gets the input from beat (app 1) abd tcp (app 2) and sends to elastic search in the defined index
![img_9.png](resources/img_9.png)

#### Then from kibana UI (port 5601) we can create a new data view on that elastic index and analyze 
![img_10.png](resources/img_10.png)
![img_11.png](resources/img_11.png)

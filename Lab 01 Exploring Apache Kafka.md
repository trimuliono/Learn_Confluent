# Lab 01 Exploring Apache Kafka

<img width="1170" height="448" alt="image" src="https://github.com/user-attachments/assets/32621c50-2868-49df-a193-b1373114710a" />

<img width="1153" height="715" alt="image" src="https://github.com/user-attachments/assets/71af9fca-08ca-4ead-810c-b463c4fa0c8c" />

### Running a simple Kafka Cluster in docker containers

* **Clone repo**
  <img width="1012" height="118" alt="image" src="https://github.com/user-attachments/assets/59959428-ba58-4b7d-ac3a-6e8e8c4276b1" />

* navigate to the project folder and execute the start.sh script:
  <img width="754" height="96" alt="image" src="https://github.com/user-attachments/assets/e8b628e1-a7e4-47f4-b8e8-71eca4719e29" />

  <img width="1094" height="745" alt="image" src="https://github.com/user-attachments/assets/099a86bd-f176-4059-a6fb-752060203938" />

* Observe the above output. You will see that a **topic named vehicle-positions is created**. Run following command to describe the topic.

  You should see result as below. Observe that topic is created with replication factor 1 and 6 number of partitions
  <img width="1336" height="185" alt="image" src="https://github.com/user-attachments/assets/3e512177-d99e-4069-80b3-45e68abc451c" />

* Next use the tool **kafka-console-consumer** installed on your lab VM to read data that the producer is writing to Kafka. Use this command to run the consumer:
  <img width="604" height="118" alt="image" src="https://github.com/user-attachments/assets/63a60567-dbd6-45f6-8c36-784f27ee1773" />

  producer sent message
  <img width="1514" height="100" alt="image" src="https://github.com/user-attachments/assets/414062ab-fb3b-44f2-8bb3-054d91dcd3ce" />

  consumer receive message
  <img width="1319" height="92" alt="image" src="https://github.com/user-attachments/assets/ebcfed25-9a30-4d39-83d5-8d3c31769655" />

* Stop the consumer by pressing CTRL-c.
  <img width="1118" height="272" alt="image" src="https://github.com/user-attachments/assets/3b387b33-d903-40c9-83dd-2b514e4e7759" />

### Cleanup

* Before you end, please clean up your system by running the stop. sh script in the project folder:
  ```
  ./stop.sh
  ```
  <img width="646" height="181" alt="image" src="https://github.com/user-attachments/assets/d135869b-399d-46fd-8026-b0e5e6bd90e9" />

* **Conclusion**
  In this exercise we created a simple real-time data pipeline powered by Kafka. We have written data into a simple Kafka cluster. This data we then have consumed with a simple Kafka tool called **kafka-console-consumer**.



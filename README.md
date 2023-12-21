# DOCKER SWARM RabbitMQ


![Docker_Swarm_RMQ_Diagram drawio](https://user-images.githubusercontent.com/64545114/221503866-c48573eb-bcb1-49ab-9049-8708bd29c118.png)


Docker Installation
-------------------
>https://docs.docker.com/engine/install/ubuntu/ linki takip edilebilir.

```
sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
    
sudo mkdir -m 0755 -p /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

systemctl start docker

sudo docker run hello-world

cat ~/.docker/config.json  # 2 bilgisyarda da bu json dosyasının içeriği aynı olması lazım. Eğer json dosyasında credsStore adında bir şey varsa bunu credStore olarak düzeltin.
```

Docker Remove
-------------------
>Docker'ı kaldırmak istiyorsanız aşağıdaki komutları kullanabilirsiniz.

```
dpkg -l | grep -i docker
sudo apt-get purge -y docker-engine docker docker.io docker-ce docker-ce-cli docker-compose-plugin
sudo apt-get autoremove -y --purge docker-engine docker docker.io docker-ce docker-compose-plugin
sudo rm -rf /var/lib/docker /etc/docker
sudo rm /etc/apparmor.d/docker
sudo groupdel docker
sudo rm -rf /var/run/docker.sock
```

Tek Bir Rabbitmq Broker'ı Oluşturma
------------
>Docker swarm olmadan, sadece bir adet rabbitmq broker'ı çalıştırmak istiyorsanız bu kısım kurulum için yeterli olacaktır. Swarm yapısı ile kurulum yapmak istiyorsanız bu kısmı atlayın.

```
cd rabbit_for_machine1/

sudo docker image  build --no-cache -t rabbitmq-cluster-base .       #Dockerfile ile rabbitmq image oluşturur.
sudo docker compose -f single_rabbit_docker-compose.yml up --force-recreate -d       #Docker compose ile container'ı oluşturur.
```

Docker Swarm-Rabbitmq-HAProxy
--------------
> UYARI!!: 2 makinede aynı docker versiyonları olması gerekiyor. Eğer aynı değilse, yukarıdaki komutlar ile iki makinedeki docker'ları kaldırıp yeniden yüklerseniz versiyonları eşitlemiş olursunuz.
>  Bu adımlardan önce gereksiz bir servis, container, volume, image varmı kontrol edin. Varsa hepsini silin.

1. makine için rabbit klasorunun icerisinde terminal açıyoruz.

```
cd rabbit_for_machine1/
```
1. makinede docker swarm'ı başlatıyoruz. Önceden init edildiyse bu adımı atlayabilirsiniz.

```
sudo docker swarm init
```

Farklı fiziksel cihazı node olarak ekleme
----
> İlk satırı 1. makinede çalıştırıyoruz. Çalıştırınca çıkan çıktıyı 2. makinede çalıştıracağız. 2. satır bu çıktıyı gösteren örnek komuttur.
> 3. satır 2. makineyi master yapar. Buradaki ```node_name``` 2. makinenin hostname'dir.
```
docker swarm join-token worker   
docker swarm join --token SWMTKN-1-3inx1g8r0x53504llosnmjfwl7xety05upr16jnfkr6pp07ixv-dvb12oxr9f4ghlmsjvsqa9mwn 10.220.220.36:2377 #üstteki komutun çıktısı, 2. makinede çalıştırılacak. !!Bu satır örnek olarak verilmiştir.
docker node promote node_name   #1. makinede çalıştırılacak, 2. makineyi master yapar.
```

PC kapanırsa ve swarm yapısı bozulursa, pc'leri node'lardan çıkartıp tekrardan swarm init yapılması gerekiyor. Üstteki adımlarda bi problem olmadıysa bu kısmı atlayın.
---

```
sudo docker swarm leave --force   #her iki pc için
sudo docker swarm init   #1. pc için
sudo docker swarm join --token SWMTKN-1-3inx1g8r0x53504llosnmjfwl7xety05upr16jnfkr6pp07ixv-dvb12oxr9f4ghlmsjvsqa9mwn 10.220.220.36:2377 #üstteki komutun çıktısı, 2. pc için, !!Bu satır örnek olarak verilmiştir.
```

Networking
---
>1. makine için.

```
sudo docker network create -d overlay --attachable my-attachable-overlay     #my-attachable-overlay isminde docker swarm network overlay network'ü oluşturuyoruz. 1. pc için.
```

RabbitMQ Image and Containers
---
>Buradaki komutlar her iki pc içindir. 2. pc'deki rabbit_for_machine2 klasorunun içerisinde terminal açılarak komutlar girilir.
```
sudo docker image  build --no-cache -t rabbitmq-cluster-base .       #Dockerfile ile rabbitmq image oluşturur.
sudo docker compose -f rabbit_docker-compose.yml up --force-recreate -d       #Docker compose ile containerlar'ı oluşturur, bu komut her iki bilgisayarda çalıştırılacak.
```

RMQ Clustering and Mirroring
---
>Komutların yanlarında bulunan yorum satırlarına dikkat ediniz!

```
sudo docker exec -it rabbit_for_machine1-rabbitmq1-1 rabbitmq-plugins enable rabbitmq_federation   #1. makinede çalıştırılır
sudo docker exec -it rabbit_for_machine2-rabbitmq2-1 rabbitmq-plugins enable rabbitmq_federation   #2. makinede çalıştırılır
sudo docker exec rabbit_for_machine1-rabbitmq1-1 sh -c "rabbitmqctl stop_app; rabbitmqctl reset; rabbitmqctl start_app"                                     #1. makinede çalıştırılır     
sudo docker exec rabbit_for_machine2-rabbitmq2-1 sh -c "rabbitmqctl stop_app; rabbitmqctl reset; rabbitmqctl join_cluster rabbit@rabbitmq1; rabbitmqctl start_app"   #2. makinede çalıştırılır
sudo docker exec -u root -t -i rabbit_for_machine2-rabbitmq2-1 /bin/bash                   #Bu komut ve alttaki komut 2. makinede çalıştırılır.
rabbitmqctl set_policy ha-fed \
    ".*" '{"federation-upstream-set":"all", "ha-sync-mode":"automatic", "ha-mode":"nodes", "ha-params":["rabbit@rabbitmq1","rabbit@rabbitmq2"]}' \
    --priority 1 \
    --apply-to queues

exit
```
Networking-Static IP
---

>```sudo docker network inspect my-attachable-overlay(network_name)```  Bu komut iki makinede de çalıştırıp containerlara atanan ip'lere bakılabilir ve müsait olan ip'ler belirlenir.

>1. makinede çalıştırılır.
```
sudo docker stop rabbit_for_machine1-rabbitmq1-1  #container durdurulur.
sudo docker network connect --ip 10.0.1.6 my-attachable-overlay rabbit_for_machine1-rabbitmq1-1    #Atanacak ip(10.0.1.3) başka containerlar tarafından kullanılmıyor olması lazım.
sudo docker start rabbit_for_machine1-rabbitmq1-1  #container başlatılır.
```

>2. makinede çalıştırılır.

```
sudo docker stop rabbit_for_machine2-rabbitmq2-1  #container durdurulur.
sudo docker network connect --ip 10.0.1.8 my-attachable-overlay rabbit_for_machine2-rabbitmq2-1    #Atanacak ip(10.0.1.8) başka containerlar tarafından kullanılmıyor olması lazım.
sudo docker start rabbit_for_machine2-rabbitmq2-1  #container başlatılır.
```

HAProxy
----

>HAProxy birinci makineye kurulacağı için aşağıdaki komutlar 1. makinede çalıştırılır.
```
cd ../haproxy/    #haproxy klasorune geçiyoruz.
sudo docker image  build --no-cache -t haproxy-base .       #Dockerfile ile image oluşturur.
sudo docker stack deploy -c haproxy_docker-compose.yml service   #HAProxy ile docker swarm servisi oluşturur.
```





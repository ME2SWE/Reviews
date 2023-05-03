# Reviews
Repo for Reviews API Service
Spec:
- Docker Host
- 2x Node JS Express API Server
- 1x Nginx Load Balancer Server
- 1x Mongo DB Server
- Tested using Loader.io

Changes:
- Addition of extra API server/instance
- Addition of Nginx Load Balacning server/instance
- Isolation of DB server/instance
- Addition of Nginx Proxy Cache

As a second imporvement plan, I added two things to my project. First one was a load-balacner.

What's the **load balancer**?
>Load balancing refers to efficiently distributing incoming network traffic across a group of backend servers, also known as a server farm or server pool.

Basically, it allows whatever-service to handle more traffic. 

>Modern high‑traffic websites must serve hundreds of thousands, if not millions, of concurrent requests from users or clients and return the correct text, images, video, or application data, all in a fast and reliable manner. To cost‑effectively scale to meet these high volumes, modern computing best practice generally requires adding more servers.

>A load balancer acts as the “traffic cop” sitting in front of your servers and routing client requests across all servers capable of fulfilling those requests in a manner that maximizes speed and capacity utilization and ensures that no one server is overworked, which could degrade performance. If a single server goes down, the load balancer redirects traffic to the remaining online servers. When a new server is added to the server group, the load balancer automatically starts to send requests to it.

src: https://www.nginx.com/resources/glossary/load-balancing/

Previously, my API was living with my database in the same docker container/host, on the same EC2 instance. Given that there is a limited bandwidth and computing power for such low-end instance, handling a huge traffic was not ideal (>1000 RPS). There are limitations on improvements I could make with revised database and queries. Hence, adding more servers is the best solution I could choose. 

It was time for me to separate the API and the database. I created three new instances, two for API and one for load balancer. I decided to use the 'old' instance as my database server since all the data's already seeded.

After creating addtional instances, It was time for me to make changes to the security group.

Take a look at the diagram below:
<img width="736" alt="Screenshot 2023-02-24 at 6 57 34 PM" src="https://user-images.githubusercontent.com/103800523/221332703-8e6b35e5-1864-4fc0-af54-8723aa96abc6.png">

Nginx Load-Balancer will receive requests from multiple clients. Then the load-balancer will evenly spread the requests(traffics) to available API servers. Once the API servers receive the request, it will then run quries on the database server. For communication between servers, I decided to use private IPs. Given that I will be starting/stopping these instances frequently, If I use public IPs, I have to change the configuration every single time. Hence, I added their private IPs to each security group's inbound and outbound rule. 

<img width="943" alt="Screenshot 2023-02-24 at 7 01 57 PM" src="https://user-images.githubusercontent.com/103800523/221332923-f629f0ad-655f-4594-8187-287a1cfd400d.png">

ICMP Type IPs are the instances' private IPs. Also, never forget to add https (otherwise, the instance will not be allowed to make necessary downloads such as Docker, NodeJS, Git, etc.).

After I added the private IPs to the security group, I made changes to docker-compose.yaml file to each API and DB server.

API server docker-compose.yaml
``` javascript
api:
    build: .
    ports:
      - 4000:3000
    volumes:
      - .:/Reviews
    environment:
      NODE_ENV: production
      PORT: 3000
      MONGODB_URI: mongodb://{"MongoDB Server Private IP"}:2717
      MONGOOSE_URI: mongodb://{"MongoDB Server Private IP"}:2717/SDC
      DB_NAME: SDC
```

DB server docker-compose.yaml
``` javascript
mongo_db:
   container_name: mongo_db
   image: mongo:latest
   restart: always
   ports:
     - 2717:27017
   volumes:
     - mongo_db:/data/db
```

Then, I pushed to two different branches (one for API and another for DB) and pulled to each instance accordingly. 
Built them and verified individually.

Setting up load-balancer & proxy-cache was a bit confusing but straightforward.

Installed Nginx by typing the following inside the instance:
``` javascript
sudo apt-get install nginx
```
Then, I made a custom conf file inside the directory /etc/nginx/conf.d.
``` javascript
upstream SDC-Reviews {
        least_conn;
        server API Private IP #1:4000;
        server API Private IP #2:4000;
}

server {

        location / {
                proxy_pass http://SDC-Reviews/;
                proxy_cache cache_zone;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_cache_lock_timeout 10s;
                proxy_cache_methods GET HEAD POST;
                proxy_cache_valid 200 201 204 10s;
                add_header X-Proxy-Cache $upstream_cache_status;
                add_header Cache-Control "public";
        }
}
```
I added the following to /etc/nginx/nginx.conf under 'http':
``` javascript
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=cache_zone:10m max_size=3G inactive=5s use_temp_path=off; 
```
>Syntax:	proxy_cache_path path [levels=levels] [use_temp_path=on|off] keys_zone=name:size [inactive=time] [max_size=size] [min_free=size] [manager_files=number] [manager_sleep=time] [manager_threshold=time] [loader_files=number] [loader_sleep=time] [loader_threshold=time] [purger=on|off] [purger_files=number] [purger_sleep=time] [purger_threshold=time];
>Default:	—
>Context:	http
More can be found here: http://nginx.org/en/docs/http/ngx_http_proxy_module.html#directives

All ips w/ 'server' tage are the IPs my Nginx will reroute to whenever there is a request.
'Least_conn' will spread out the request, allocating traffic first to available least connected server.

Some useful commands:
>docker system prune -a

Will remove all containers, images, and cache of the Docker host. (Use carefully, only recommended when there is no space on your instance disk)

>sudo service nginx restart

Will restart Nginx service

>service nginx status

Will check for Nginx service status

>sudo netstat -natp

Will check network status (IPs, Ports, etc.). of the instance

Once I figured all out, I was able to verify that the load balancer was spreading the traffice between two API servers.

**Follow Up Comments

I realize that my cache was not warmed up (populated). Hence, the results I recorded were in a rough shape. I learned that I want to pre-fill my cache by dry-running to maximize the performance. 

During the project, I learned a lot. The significance of well-designed system and stack selection are high. Rather than jumping straight into a development, I now spend a good a mount of time sketching out a system diagram on how my app will behave in its operation before any action. I believe this is a great development practice since it will help me to understand the development from high-level. 

Although there were a lot of hiccups and obstacles during this project, it will be the most valuable project to me. 

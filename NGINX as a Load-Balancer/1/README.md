# Nginx Layer 4 Load Balancing on NodeJS App in AWS

This document outlines the process of setting up a `layer 4` load-balanced Node.js application environment using Nginx. The setup consists of two identical `Node.js` applications, an `Nginx` server.

<img src="https://github.com/Minhaz00/NodeJS-MySQL/blob/main/10.%20Nginx%20L4%20LB%20NodeJS%20service%20in%20AWS/image/nginxlb-02.PNG?raw=true" />


## Task
Create a load-balanced environment with two `Node.js` applications, `Nginx` as a layer 4 load balancer. Nginx will reside in the `public subnet` and the Node.js server will be in the `private subnet`.


## Steps

### Setup AWS: Create VPC, subnets, route table and gateways 

At first, we need to create a VPC in AWS, configure subnet, route tables and internet gateway, NAT gateway, security group.

1. Create a VPC named `my-vpc`.

2. Create a public subnet named: `public-subnet`.

3. Create a private subnet named: `private-subnet`.

![subent](https://github.com/Konami33/poridhi.io.intern/raw/main/NGINX%20as%20a%20Load-Balancer/1/images/1.png)

3. Create a public route table named `rt-public` and associate it with `public-subnet`.

4. Create a private route table named `rt-private` and associate it with `private-subnet`

![](https://github.com/Konami33/poridhi.io.intern/raw/main/NGINX%20as%20a%20Load-Balancer/1/images/2.png)

5. Create an internet gateway named `igw` and attach it to `my-vpc`.

![](https://github.com/Konami33/poridhi.io.intern/raw/main/NGINX%20as%20a%20Load-Balancer/1/images/3.png)

6. Create a NAT gateway named `nat-gw` and associate it with `public-subnet`.

![](https://github.com/Konami33/poridhi.io.intern/raw/main/NGINX%20as%20a%20Load-Balancer/1/images/4.png)

7. Configure the route tables to use the internet gateway and NAT gateway.

![](https://github.com/Konami33/poridhi.io.intern/raw/main/NGINX%20as%20a%20Load-Balancer/1/images/5.png)

Here is the `resource-map` of our VPC:

![alt text](https://github.com/Konami33/poridhi.io.intern/raw/main/NGINX%20as%20a%20Load-Balancer/1/images/6.png)

### Create and setup EC2 instances

We need to create `3 instances` in EC2. One in the public subnet for `nginx` server, rest two is for two Nodejs Application. Both will be in the private subnet.

#### Create the NodeJS App EC2 Instances:
- Launch two EC2 instances (let's call them `node-app-1` and `node-app-2`) in our private subnet.
- Choose an appropriate AMI (e.g., Ubuntu).
- Configure the instances with necessary security group rules to allow HTTP/HTTPS traffic (typically port 80/443).

![security group]()

- Assign a key pair <MyKeyPair.pem> for SSH access.

#### Create the NGINX EC2 Instance:
- Launch another EC2 instance for the NGINX load balancer (let's call it `nginx-lb`) in our public subnet.
- Configure the instance with a security group to allow incoming traffic on the load balancer port (typically port 80/443) and outgoing traffic to the NodeJS servers.

![security group]()

- Assign a key pair e.g. <MyKeyPair.pem> for SSH access.

### Now connect the Public and private instance using SSH command:

![](https://github.com/Konami33/poridhi.io.intern/raw/main/NGINX%20as%20a%20Load-Balancer/1/images/10.png)

### Access the Public Instance via SSH

1. *Set File Permissions*:

   - *For Linux*:
     ```sh
     chmod 400 <MyKeyPair.pem>
     ```

2. *SSH into the Public Instance*:
   - Open a terminal and run:
     ```sh
     ssh -i <MyKeyPair.pem> ubuntu@<public_instance_ip>
     ```
   - Replace <public_instance_ip> with the public IP and the <MyKeyPair.pem> with the keypair.
   - You should be able to login to the instance now.
   - Now run `exit` to return to your local terminal.

   ![](https://github.com/Konami33/poridhi.io.intern/raw/main/NGINX%20as%20a%20Load-Balancer/1/images/7.png)

#### Copy the Key Pair to the Public Instance

3. *Copy the Key Pair to the Public Instance*:
    - On your local machine, run the following command to copy the key pair to the public instance:
    ```sh
     scp -i <MyKeyPair.pem> <MyKeyPair.pem> ubuntu@<public_instance_ip>:~
    ```

    - Replace <public_instance_ip> with the public IP address of the public instance and the <MyKeyPair.pem> with the keypair.

    ![](https://github.com/Konami33/poridhi.io.intern/raw/main/NGINX%20as%20a%20Load-Balancer/1/images/8.png)

#### SSH from the Public Instance to the Private Instance

3. *SSH into the Private Instance from the Public Instance*:
    - After coping the keypair into the public instance, ssh into public instance

    ```sh
    ssh -i <MyKeyPair.pem> ubuntu@<public_instance_ip>
    ```
    - change the file permissions of the copied key pair:
   
    ```sh
    chmod 400 <MyKeyPair.pem>
    ```
    - ssh into the private instance from the public instance:
   
    ```sh
    ssh -i <MyKeyPair.pem> ubuntu@<private_instance_ip>
    ```

    - Remember to Replace the <private_instance_ip> with the private IP address of the private instance.

    ![](https://github.com/Konami33/poridhi.io.intern/raw/main/NGINX%20as%20a%20Load-Balancer/1/images/9.png)

## Set up Node.js Applications

Connect to `node-app-1` and configure as follows:

### Create Node App 1

Install npm:

```sh
sudo apt update
sudo apt install npm
```

Then follow the command to set up the node application:

```bash
mkdir Node-app
cd Node-app
npm init -y
npm install express mysql2 body-parser
```

Create `index.js` in the Node-app-1 directory with the following code:

```javascript
const express = require('express');
const app = express();
const port = process.env.PORT;

app.get('/', (req, res) => {
  res.status(200).send(`Hello, from Node App on PORT: ${port}!`);
});

app.listen(port, () => {
  console.log(`App running on http://localhost:${port}`);
});
```

### Create Node App 2

Connect to the `Node-app-2` instance by following the similar steps that we did for connecting the `Node-app-1`.  

### Start Node.js Applications
Navigate to each Node.js application directory and run:
For *node-app-1*:

```bash
export PORT=3001
node index.js
```
For *node-app-2*:
```bash
export PORT=3002
node index.js
```

## Set up Nginx

Now, connect to teh nginx instance and create a nginx.conf file and a Dockerfile. You also need to install Docker. 

### Create Nginx Configuration
```bash
mkdir Nginx
cd Nginx
```

Create nginx.conf in the Nginx directory with the following configuration:

```sh
events {}

stream {
    upstream nodejs_backend {
        server <Node-app-1 private-ip>:3001; # Node-app-1
        server <Node-app-2 private-ip>:3002; # Node-app-2
    }

    server {
        listen 80;

        proxy_pass nodejs_backend;

        # Enable TCP load balancing
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
    }
}
```

Replace the `private ip` of nodejs app according to `your ec2 instances`.

This configuration sets up Nginx to act as a `TCP load balancer`, distributing traffic between the `two Node.js` applications. Let's break down the key components:

- `stream {}`: This block is used for TCP and UDP load balancing. It's different from the http {} block used in HTTP load balancing. It allows it to handle TCP and UDP traffic at the transport layer (Layer 4) of the OSI model.

- `upstream nodejs_backend {}`: This defines a group of backend servers. We're using host.docker.internal to refer to the host machine from within the Docker container. The ports 3001 and 3002 correspond to our two Node.js applications.

- `server {}`: This block defines the server configuration for the load balancer.
listen 80: This tells Nginx to listen on port 80 for incoming TCP connections.


This configuration provides a simple round-robin load balancing across our Node.js applications at the TCP level, which can be more efficient than HTTP-level load balancing for certain use cases.

### Install Docker in Nginx Instance

Update packages and install Docker:

```sh
sudo apt-get update -y
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

### Create Dockerfile for Nginx
Create a file named Dockerfile in the Nginx directory with the following content:

```Dockerfile
FROM nginx:latest
COPY nginx.conf /etc/nginx/nginx.conf
```

### Build Nginx Docker Image
```bash
docker build -t custom-nginx .
```
This command builds a Docker image for Nginx with our custom configuration.

### Run Nginx Container
```bash
docker run -d -p 80:80 --name my_nginx custom-nginx
```

This command starts the Nginx container with our custom configuration.

## Verification

1. Visit `http://<nginx-public-ip>` in a web browser. You should see a response from one of the Node.js applications.

2. Refresh the browser or make multiple requests to observe the load balancing in action. You should see responses alternating between `Node-app-1` and `Node-app-2` running on different ports.

    Example:

    ![alt text](https://github.com/Konami33/poridhi.io.intern/blob/main/NGINX%20as%20a%20Load-Balancer/5/images/Screenshot%202024-06-26%20214643.png?raw=true)

    ![alt text](https://github.com/Konami33/poridhi.io.intern/blob/main/NGINX%20as%20a%20Load-Balancer/5/images/Screenshot%202024-06-26%20214707.png?raw=true)
   
## Conclusion

By configuring Nginx with the stream module as described, you have effectively set up Nginx as a `Layer 4 load balancer`. It handles TCP connections based on IP addresses and port numbers, making routing decisions at the transport layer without inspecting higher-layer protocols like HTTP. This setup is suitable for scenarios where efficient TCP load balancing across multiple backend services is required.
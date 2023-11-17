# Introduction
In this post you will learn how to deploy highly available docker containers, load balanced with Traefik which can be accessed through your domain name (TLS included), from start to finish.
We will be deploying [traefik/whoami](https://hub.docker.com/r/traefik/whoami) in order to keep the tutorial simple, however, this guide will work for any docker image.

## Setup
!!! note
    If you don't have HashiCorp Nomad installed, please follow the [official installation guide](https://developer.hashicorp.com/nomad/tutorials/get-started/gs-install) to install it.



Before we start deploying our docker container, we first need to set up our Nomad infrastructure.

It is recommended to run Nomad with at least 3 nodes, one of them running in **Server** mode and the other two running in **Client** mode.

- **Server**  is responsible for assigning jobs to **Clients**.
- **Clients** are responsible for running jobs/containers.

!!!note
    This tutorial assumes that you have 3 linux servers available, because we will be deploying an infrastructure as described above (1 server for **Server** mode, and the other 2 servers for **Client** mode).

## Server mode setup
In order to set up Nomad to run in Server mode, ssh into the server you want to use for **Server** mode and make sure the `/etc/nomad.d/nomad.hcl` config file looks like this:
```hcl
datacenter = "dc1"
data_dir  = "/opt/nomad/data"
bind_addr = "0.0.0.0"

server {
  # license_path is required as of Nomad v1.1.1+
  #license_path = "/opt/nomad/license.hclic"
  enabled          = true
  # number of client servers
  bootstrap_expect = 2
}
```
Now, in order to run Nomad, execute this command:
```shell
sudo nomad agent -server -config=nomad.hcl
```
Nomad should now be running in Server mode.

## Client mode setup
In order to set up Nomad to run in Client mode, ssh into the two of your servers you want to use as Clients and make sure their `/etc/nomad.d/nomad.hcl` config file looks like this (make sure you use the address of your own Server node!):
```hcl
datacenter = "dc1"
data_dir  = "/opt/nomad/data"
bind_addr = "0.0.0.0"

client {
  enabled = true
  # address of the Server node
  servers = ["192.168.0.79"]
}
```
Now you can run Nomad in Client mode:
```
sudo nomad agent -client -config=nomad.hcl
```

!!! warning
    Make sure you do this for every linux server you are planning on using as a **Client**.

## WebUI
Once Nomad is running, you can visit the Web interface at http://[ip_of_your_server_or_client]:4646

## Deploying the container
Now that our servers are ready, we can start running jobs with Nomad. In this example, we are going to deploy 5 instances of the [traefik/whoami](https://hub.docker.com/r/traefik/whoami) container, because we want our containers to be load balanced and highly available. First, create a file named `whoami.nomad` (it doesn't matter where you create it) and write this job specification:
```hcl
job "whoami" {
  datacenters = ["dc1"]

  type = "service"

  group "demo" {
    count = 5

    network {
       port "http" {
         to = 80
       }
    }

    service {
      name = "whoami-demo"
      port = "http"
      provider = "nomad"

      tags = [
        "traefik.enable=true",
        "traefik.http.routers.whoami.rule=Host(`whoami.damirhadzagic.com`)",

        ### Tags for TLS
        "traefik.http.middlewares.myredirect.redirectscheme.scheme=https",
        "traefik.http.routers.whoami.middlewares=myredirect",
        "traefik.http.routers.whoami.entrypoints=web",
        "traefik.http.routers.whoami-secure.rule=Host(`whoami.damirhadzagic.com`)",
        "traefik.http.routers.whoami-secure.entrypoints=websecure",
        "traefik.http.routers.whoami-secure.tls.certresolver=myhttpchallenge",
        "traefik.http.routers.whoami-secure.tls=true"
      ]
    }

    task "server" {
      env {
        WHOAMI_PORT_NUMBER = "${NOMAD_PORT_http}"
      }

      driver = "docker"

      config {
        image = "traefik/whoami"
        ports = ["http"]
      }
    }
  }
}
```
The job specification above tells Nomad that we want to run 5 containers using the [traefik/whoami](https://hub.docker.com/r/traefik/whoami) image. We have told it which **internal** port to use, which is in our case port 80. We did not set up which port it should bind to because Nomad will do that dynamically for us. We have also set up tags for traefik, which we will deploy later. Make sure you use your own domain name.\
Now that our job specification is ready, we can run it:
```shell
nomad job run whoami.nomad
```

While the job is running, you can go to the WebUI and watch as the containers are being deployed (Jobs -> whoami). If everything has gone well, you should see this:

![whoami job status](/assets/images/whoami_deployment.png#center)

If you want to, you can run `docker ps` on your **Client** machines, and you should be able to see the containers we just deployed.

![lab1](/assets/images/lab1.png#center)
![lab2](/assets/images/lab2.png#center)

As you can see, Nomad deployed 2 containers on my lab1 VM, and 3 containers on my lab2 VM. Pretty cool, right?

Now that we have deployed our containers, it's time to load balance them via Traefik.

## Deploying Traefik
To deploy Traefik, all we have to do is create another job for Nomad to run. Create a file called `whoami.nomad` and write this job specification:
```hcl
job "traefik" {
  datacenters = ["dc1"]
  type        = "service"

  group "traefik" {
    count = 1

    network {
      port "http" {
         static = 80
      }
      
      port "https" {
         static = 443
      }

      port "admin" {
         static = 8080
      }
    }

    service {
      name = "traefik-http"
      provider = "nomad"
      port = "http"
    }

    task "server" {
      driver = "docker"
      config {
        image = "traefik:2.9"
        ports = ["admin", "http", "https"]
        args = [
          "--api.dashboard=true",
          "--api.insecure=true", ### For Test only, please do not use that in production
          "--entrypoints.web.address=:${NOMAD_PORT_http}",
          "--entrypoints.traefik.address=:${NOMAD_PORT_admin}",
          "--entrypoints.websecure.address=:${NOMAD_PORT_https}",
          "--certificatesresolvers.myhttpchallenge.acme.httpchallenge=true",
          "--certificatesresolvers.myhttpchallenge.acme.httpchallenge.entrypoint=web",
          "--certificatesresolvers.myhttpchallenge.acme.email=example@email.com",
          "--certificatesresolvers.myhttpchallenge.acme.storage=/letsencrypt/acme.json",
          "--providers.nomad=true",
          "--providers.nomad.endpoint.address=http://192.168.0.79:4646" ### IP to your nomad server
        ]

       volumes = [
         "letsencrypt:/letsencrypt"
       ]
      }
    }
  }
}
```
Make sure you use your own email address and the IP address of your own Nomad node running in **Server** mode.\
Now deploy traefik by running the following command:
```shell
nomad job run traefik.nomad
```
Nomad will pick a server to run traefik on, and once it successfully deploys it, visit the dashboard on port 8080. If you want to find out where Nomad deployed traefik, go to Nomad's WebUI -> Jobs -> traefik -> Recent Allocations -> Click on the most recent allocation -> Scroll until you see Ports:

![traefik ports](/assets/images/traefik_ports.png)

In my case Nomad deployed traefik on 192.168.0.42 (which is my lab2 VM), so let's visit the dashboard at http://192.168.0.42:8080:

![traefik dashboard](/assets/images/traefik_dashboard.png)

The dashboard loaded successfully which is a good sign, let's now click on the HTTP tab so we can check if traefik wired everything up correctly. Once you've clicked on the HTTP tab, click on the router that's marked with the green TLS icon:

![http tab](/assets/images/http_tab.png)

Now click on the whoami-demo service:

![traefik service](/assets/images/traefik_service.png)

On the right you can see our 5 containers we deployed earlier, this means that traefik has managed to successfully "wire" everything up:

![traefik servers](/assets/images/traefik_servers.png)

## Testing
Now it's time to test if everything works. Head on over to https://yourdomain.com (make sure port 443 is open!) and you should see something like this:
![whoami](/assets/images/whoami.png)
Look at the Hostname field at the top, every time you refresh the page its value will change, which means that traefik is routing requests to different containers, which is exactly what we want.

## The End
This is it! Congratulations, you have successfully deployed 5 instances of a container, on two different servers, load balanced with traefik, with TLS as well!

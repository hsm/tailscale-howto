# Using Tailscale and Docker to manage communication with remote servers

Docker is a nice way to maintain consistent work environments across different remote servers if you need to. Some of the stuff people do require GPU resources and my machines tend to move around a lot across different cloud providers, as well as not being always on or even exist. In short, my machines change IP a lot, various ports open and close, and I don't want to spend a lot of time thinking about security every time a port opens.

One way to make this easier is to use [Tailscale](https://tailscale.com/). Tailscale uses [Wireguard](https://www.wireguard.com/) as VPN as well as various tricks to traverse firewalls in order to avoid using a relay server. It's free for personal use. In the below we'll use it to show how to set up a developer Docker image with a fixed IP that can be moved around.

We'll assume you have Docker installed on your machine as well as the Tailscale app. You also need a free Tailscale account. At the time of writing, the Tailscale app version is 1.0.5.

## Setting up Tailscale

First we need to build the client

```
docker build -t tailscale:1.0.5 https://github.com/tailscale/tailscale.git#v1.0.5
```

Then we run it. We'll give it the host name `hello` so we can recognize it in the Tailscale Dashboard

```
docker run -d --name=tailscaled --hostname=hello \
        -v $(pwd)/hello/tailscale/conf:/var/lib/tailscale \
        -v /dev/net/tun:/dev/net/tun --privileged \
        tailscale:1.0.5 tailscaled
```

Run
```
docker exec tailscaled tailscale up
```
and visit the authentication link. Upon successful authentication some files are created in `$(pwd)/hello/tailscale/conf`. Find the IP of `hello` in the [Tailscale admin dashboard](https://login.tailscale.com/admin/machines). We'll assume the IP was `100.1.2.3` below. If you want, you can disable key expiry to avoid having to authenticate again after a while.

As long as you move the contents of `$(pwd)/hello/tailscale/conf` to any server you want to use, the IP should stay the same.

## Attaching a container

As a proof of concept we'll attach a container to the network created by `tailscaled`. This could be your development container but this is just a demo container picked more or less at random that we'll use to illustrate the concept.

```
docker run -d --name=hello --network=container:tailscaled nginxdemos/hello
```

You should now be able to access a web server on http://100.1.2.3 (The IP from above). You could, of course, give this IP a name using DNS since it will stay fixed.

## Steps to run remotely

Remove everything locally if you haven't already
```
docker stop tailscaled hello
docker rm tailscaled hello
```

Start a remote server. Note the IP. Let's assume it is `34.1.2.3`. Copy the contents of the `hello` directory to the server.
```
scp -r hello 34.1.2.3:
```
Log in to the remote server
```
ssh 34.1.2.3
```

and build
```
docker build -t tailscale:1.0.5 https://github.com/tailscale/tailscale.git#v1.0.5
```
run
```
docker run -d --name=tailscaled --hostname=hello \
        -v $(pwd)/hello/tailscale/conf:/var/lib/tailscale \
        -v /dev/net/tun:/dev/net/tun --privileged \
        tailscale:1.0.5 tailscaled
```
and deploy
```
docker run -d --name=hello --network=container:tailscaled nginxdemos/hello
```

The server should now (again) be available on http://100.1.2.3

## Troubleshooting

The status of the Tailscale service can be checked with
```
docker exec tailscaled tailscale status
```

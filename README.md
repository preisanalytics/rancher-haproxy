# rancher-haproxy
## Description
This docker image is a haproxy server that uses the rancher-metadata service to do service discovery to route requests via hostname to stacks.

eg this will allow you to create a stack called 'foobar' and have requests hit http://foobar.$domain/ that will route to containers that have a specified label.
This is useful if you for example take an N stacks approach to CI/CD. ie. deploy app build 123 to a stack called 'app-r123'
This will give you an immediate endpoint http://app-r123.$domain'

The label to associate and the domain are configurable
## How to use this container

Create a stack for your load balancer
docker-compose.yml:
```
HTTP:
  ports:
  - 80:80
  environment:
    STACK_DOMAIN: '$MyDomainWithWildcardRecord'
    RANCHER_LABEL: 'IWantMyContainersThatHaveThisLabelToBeDiscovered'
  labels:
    io.rancher.container.pull_image: always
  tty: true
  image: nodeintegration/rancher-haproxy
  stdin_open: true
```
Then create your web applications with the same label you used above ie "RANCHER_LABEL".
e.g. a stack called "test-webservice-r123"
docker-compose.yml:
```
nginx:
  labels:
    io.rancher.container.pull_image: always
    IWantMyContainersThatHaveThisLabelToBeDiscovered: '80'
  tty: true
  image: nginx:stable-alpine
  stdin_open: true
```
The value of the label is the port that haproxy will balance to

## How does this work?
* This relies on a docker entrypoint script to run everthing:
* The first process that starts is a python script. This script scans the rancher-metadata service api for the 'containers' path, it then looks for all containers that contain "RANCHER_LABEL" value, if a container has this label, then it adds it to the list with some details (the ip address and the value of the label for the port as well as the stack name)
* The python script then generates 2 files. 1. a domain map which is a list of domains (the $stack_name appended by the $DOMAIN value) to map to a backend (the $stack_name). 2. a backend config file which contains backends ($stack_name) then a list of containers by $service_name-$uuid as the server id and the ip and port
* The python script then writes those 2 files to a tmp file and diffs the current files against them, if they have changed then it renames the tmp file to its final destination
* This python script is backgrounded and run at a 10 second interval
* The entrypoint script then starts haproxy after a few seconds of grace time as a background process
* The last step is looping inotify watching the 2 final destination files...if those files attributes change in anyway it reloads the haproxy daemon


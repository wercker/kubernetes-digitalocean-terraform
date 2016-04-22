# Kubernetes - Digital Ocean - Terraform

Deploy your Kubernetes cluster in Digital Ocean using Terraform. Also
adds the Kubernetes dashboard.

## Disclaimer

Not tested in production yet. Use it at your own risk.

## Requirements

* [Digital Ocean](https://www.digitalocean.com/) account
* Digital Ocean Token [In DO's settings/tokens/new](https://cloud.digitalocean.com/settings/tokens/new)
* [Terraform](https://www.terraform.io/) (`brew install terraform` on OSX)

Do all the following steps from a development machine. It does not matter _where_ is it, as long as it is connected to the internet. This one will be subsequently used to access the cluster via `kubectl`.

## Generate private / public keys

```
ssh-keygen -t rsa -b 4096
```

System will prompt you for a filepath to save the key, we will go by `~/.ssh/id_rsa` in this tutorial.

## Add your public key in Digital Ocean control panel

[Do it here](https://cloud.digitalocean.com/settings/security). Name it and paste the public key just below `Add SSH Key`.

## Add this key to your ssh agent

```bash
eval `ssh-agent -s`
ssh-add ~/.ssh/id_rsa
```

## Invoke terraform

We put our Digitalocean token in the file `./secrets/DO_TOKEN` (that directory is mentioned in `.gitignore`, of course, so we don't leak it)

Then we setup the environment variables (step into `this repository` root). Note that the first variable sets up the *number of workers*

```bash
export TF_VAR_number_of_workers=3
export TF_VAR_do_token=$(cat ./secrets/DO_TOKEN)
export TF_VAR_pub_key="~/.ssh/id_rsa.pub"
export TF_VAR_pvt_key="~/.ssh/id_rsa"
export TF_VAR_ssh_fingerprint=$(ssh-keygen -lf ~/.ssh/id_rsa.pub | awk '{print $2}')
```

If you are using OSX, replace the last line with

```bash
export TF_VAR_ssh_fingerprint=$(ssh-keygen -E MD5 -lf ~/.ssh/id_rsa.pub | awk '{print $2}' | sed 's/MD5://g')
```

There is a convenience file for you in `./hack/setup_terraform.sh`. Invoke it as

```bash
. ./hack/setup_terraform.sh
```

After setup, call `terraform apply`

```bash
terraform apply
```

That should do! `kubectl` is configured, so you can just check the nodes (`get no`) and the pods (`get po`).

```bash
$ kubectl get no
NAME          LABELS                               STATUS
X.X.X.X   kubernetes.io/hostname=X.X.X.X   Ready     2m
Y.Y.Y.Y   kubernetes.io/hostname=Y.Y.Y.Y   Ready     2m

$ kubectl --namespace=kube-system get po
NAME                                   READY     STATUS    RESTARTS   AGE
kube-apiserver-X.X.X.X            1/1       Running   0          1m
kube-controller-manager-X.X.X.X   1/1       Running   0          1m
kube-dns-v9-heab2                 4/4       Running   0          56s
kube-podmaster-X.X.X.X            2/2       Running   0          2m
kube-proxy-Y.Y.Y.Y                1/1       Running   0          34s
kube-proxy-X.X.X.X                1/1       Running   0          2m
kube-scheduler-X.X.X.X            1/1       Running   0          1m
```

## Accessing the dashboard
Just run `kubectl proxy` and go to `127.0.0.1:8001/ui/`

You are good to go. Now, we can keep on reading to dive into the specifics.

## Deploy details

### K8s etcd host

#### Cloud config

The following unit is being configured and started

* `etcd2`

### K8s master

#### Cloud config

##### Files

The following files are `kubernetes` manifests to be loaded by `kubelet`

* `/etc/kubernetes/manifests/kube-apiserver.yaml`
* `/etc/kubernetes/manifests/kube-proxy.yaml`
* `/etc/kubernetes/manifests/kube-podmaster.yaml`
* `/srv/kubernetes/manifests/kube-controller-manager.yaml`
* `/srv/kubernetes/manifests/kube-scheduler.yaml`

##### Units

The following units are being configured and started

* `flanneld`: Specifying that it will use the `k8s-etcd` host's `etcd` service
* `docker`: Dependent on this host's `flannel`
* `kubelet`: The lowest level kubernetes element.

#### Provisions

Once we create this droplet (and get its `IP`), the TLS assets will be created locally (i.e. the development machine from we run `terraform`), and put into the directory `secrets` (which, again, is mentioned in `.gitignore`).

The following files will be provisioned into the host

* `/etc/kubernetes/ssl/ca.pem`
* `/etc/kubernetes/ssl/apiserver.pem`
* `/etc/kubernetes/ssl/apiserver-key.pem`

With some modifications to be run

```bash
sudo chmod 600 /etc/kubernetes/ssl/*-key.pem
sudo chown root:root /etc/kubernetes/ssl/*-key.pem
```

Finally, we start `kubelet`, _enable_ it and create the namespace

```bash
sudo systemctl start kubelet
sudo systemctl enable kubelet
until $(curl --output /dev/null --silent --head --fail http://127.0.0.1:8080); do printf '.'; sleep 5; done
curl -XPOST -d'{\"apiVersion\":\"v1\",\"kind\":\"Namespace\",\"metadata\":{\"name\":\"kube-system\"}}' http://127.0.0.1:8080/api/v1/namespaces
```

### K8s workers

#### Cloud config

##### Files

The following files are `kubernetes` manifests to be loaded by `kubelet`

* `/etc/kubernetes/manifests/kube-proxy.yaml`
* `/etc/kubernetes/worker-kubeconfig.yaml`

##### Units

The following units are being configured and started

* `flanneld`: Specifying that it will use the `k8s-etcd` host's `etcd` service
* `docker`: Dependent on this host's `flannel`
* `kubelet`: The lowest level kubernetes element.

### Provisions

The following files will be provisioned into the host

* `/etc/kubernetes/ssl/ca.pem`
* `/etc/kubernetes/ssl/worker.pem`
* `/etc/kubernetes/ssl/worker-key.pem`

With some modifications to be run

```bash
sudo chmod 600 /etc/kubernetes/ssl/*-key.pem
sudo chown root:root /etc/kubernetes/ssl/*-key.pem
```

We start `kubelet` and _enable_ it

```bash
sudo systemctl start kubelet
sudo systemctl enable kubelet
```

### Setup `kubectl`

After the installation is complete, `terraform` will config `kubectl` for you. The environment variables will be stored in the file `secrets/setup_kubectl.sh`.

Test your brand new cluster

```bash
kubectl get nodes
```

You should get something similar to

```
$ kubectl get nodes
NAME          LABELS                               STATUS
X.X.X.X       kubernetes.io/hostname=X.X.X.X       Ready
```

### Deploy DNS Add-on

The file `03-dns-addon.yaml` will be rendered (i.e. replace the value `DNS_SERVICE_IP`), and then `kubectl` will create the Service and Replication Controller.


### Deploy microbot with External IP

The file `04-microbot.yaml` will be rendered (i.e. replace the value `EXT_IP1`), and then `kubectl` will create the Service and Replication Controller.

To see the IP of the service, run `kubectl get svc` and look for the EXTERNAL-IP (should be the first worker's ext-ip). 

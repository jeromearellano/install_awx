# Ansible Project - Crontab Migration

## Prerequisite

- [K3S - Kubernetes Lightweight Version](https://k3s.io/)
- [Ansible AWX Operator](https://github.com/ansible/awx-operator)
- [Ansible AWX Platform](https://github.com/ansible/awx)
- [Influx DB Version 2 Platform](https://www.influxdata.com/)
- [Grafana Monitoring Platform](https://grafana.com/)

## Installation and Configuration

### Install k3s Kubernetes Distribution

- AWX is supported and can only be run as a containerized application using Docker images deployed to either an OpenShift cluster, a Kubernetes cluster, or docker-compose. We shall use K3s Kubernetes setup to run AWX on CentOS 7.

Put SELinux in permissive mode:

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config
cat /etc/selinux/config | grep SELINUX=
```

Install k3s by running the commands below:

```bash
curl -sfL https://get.k3s.io | bash -s - --write-kubeconfig-mode 644 --disable metrics-server
```

Check k3s service to confirm it is running and working:

```bash
systemctl status k3s.service
```

As root user do a validation on use of kubectl Kubernetes management tool:

```bash
sudo su -
kubectl get nodes
```

You can also confirm Kubernetes version deployed using the following command:

```bash
kubectl version --short
```

The K3s service will be configured to automatically restart after node reboots or if the process crashes or is killed.

### Install AWX Operator

Install the specified version of AWX Operator. Note that this procedure is applicable only for AWX Operator `0.14.0` or later.

```bash
cd ~
git clone https://github.com/ansible/awx-operator.git
cd awx-operator
git checkout 0.30.0
```

Export the name of the namespace where you want to deploy AWX Operator as the environment variable `NAMESPACE` and run `make deploy`. The default namespace is `awx`.

```bash
export NAMESPACE=awx
make deploy
```

The AWX Operator will be deployed to the namespace you specified.

```bash
$ kubectl -n awx get all
NAME                                                   READY   STATUS    RESTARTS   AGE
pod/awx-operator-controller-manager-68d787cfbd-kjfg7   2/2     Running   0          16s

NAME                                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/awx-operator-controller-manager-metrics-service   ClusterIP   10.43.150.245   <none>        8443/TCP   16s

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/awx-operator-controller-manager   1/1     1            1           16s

NAME                                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/awx-operator-controller-manager-68d787cfbd   1         1         1       16s
```

### Prepare required files

Clone this repository and change directory.

```bash
cd ~
git clone https://github.com/jeromearellano/install_awx.git
cd install_awx
```

Generate a Self-Signed certificate.

Check OpenSSL version. If version is `OpenSSL ≥ 1.1.1` use:

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 1095 -nodes \
  -keyout ./base/tls.key -out ./base/tls.crt -subj '/CN=awx-io.local' \
  -addext 'subjectAltName=DNS:awx-io.local'
```

If version is `OpenSSL ≤ 1.1.0` use:

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 1095 -nodes \
  -keyout ./base/tls.key -out ./base/tls.crt -subj '/CN=awx-iolocal' \    👈edit CN
  -extensions san \
  -config <(echo '[req]'; echo 'distinguished_name=req';
            echo '[san]'; echo 'subjectAltName=DNS:awx-io.local')    👈edit CN
```

Verify certificate contents.

```bash
openssl x509 -noout -text -in ./base/tls.crt
```

Modify `base/awx.yaml`.

```yaml
...
spec:
  ...
  ingress_type: ingress
  ingress_tls_secret: awx-secret-tls
  hostname: awx-io.mvne.postemobile.local     👈edit hostname
  ...
```

Modify `base/kustomization.yaml`.

Note that the `password` under `awx-postgres-configuration` should not contain single or double quotes (`'`, `"`) or backslashes (`\`) to avoid any issues during deployment, backup or restoration.

```yaml
...
  - name: awx-postgres-configuration
    type: Opaque
    literals:
      - host=awx-postgres-13
      - port=5432
      - database=awx
      - username=awx
      - password=<verystrongpassword>     👈👈👈
      - type=managed

  - name: awx-admin-password
    type: Opaque
    literals:
      - password=<verystrongpassword>     👈👈👈
...
```

Prepare directories for Persistent Volumes defined in `base/pv.yaml`.

```bash
sudo mkdir -p /srv/postgres-13
sudo mkdir -p /srv/projects
sudo chmod 755 /srv/postgres-13
sudo chown awx:root /srv/projects
```

These directories will be used to store your databases and project files. Note that the size of the PVs and PVCs are specified in some of the files in this repository, but since their backends are `hostPath`, its value is just like a label and there is no actual capacity limitation.

### Deploy AWX

Deploy AWX, this takes few minutes to complete.

```bash
kubectl apply -k base
```

To monitor the progress of the deployment, check the logs of `deployments/awx-operator-controller-manager`:

```bash
kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
```

Required objects has been deployed next to AWX Operator in `awx` namespace.

```bash
$ kubectl -n awx get awx,all,ingress,secrets,pv,pvc,svc
NAME                      AGE
awx.awx.ansible.com/awx   25h

NAME                                                  READY   STATUS    RESTARTS      AGE
pod/awx-589798c4d8-sm7cd                              4/4     Running   8 (53m ago)   25h
pod/awx-postgres-13-0                                 1/1     Running   2 (53m ago)   25h
pod/awx-operator-controller-manager-9589d9859-7s5mv   2/2     Running   6 (52m ago)   25h

NAME                                                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/awx-operator-controller-manager-metrics-service   ClusterIP   10.43.37.174   <none>        8443/TCP   25h
service/awx-postgres-13                                   ClusterIP   None           <none>        5432/TCP   25h
service/awx-service                                       ClusterIP   10.43.152.84   <none>        80/TCP     25h

NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/awx                               1/1     1            1           25h
deployment.apps/awx-operator-controller-manager   1/1     1            1           25h

NAME                                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/awx-589798c4d8                              1         1         1       25h
replicaset.apps/awx-operator-controller-manager-9589d9859   1         1         1       25h

NAME                               READY   AGE
statefulset.apps/awx-postgres-13   1/1     25h

NAME                                    CLASS    HOSTS                           ADDRESS          PORTS     AGE
ingress.networking.k8s.io/awx-ingress   <none>   awx-io.mvne.postemobile.local   192.168.100.51   80, 443   25h

NAME                                TYPE                DATA   AGE
secret/awx-admin-password           Opaque              1      25h
secret/awx-postgres-configuration   Opaque              6      25h
secret/awx-app-credentials          Opaque              3      25h
secret/awx-secret-key               Opaque              1      25h
secret/awx-broadcast-websocket      Opaque              1      25h
secret/awx-secret-tls               kubernetes.io/tls   2      25h

NAME                                  CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                               STORAGECLASS      REASON   AGE
persistentvolume/awx-projects-pv      5Gi        RWO            Retain           Bound    awx/awx-projects-pvc                awx-projects-sc            25h
persistentvolume/awx-postgres-13-pv   8Gi        RWO            Retain           Bound    awx/postgres-13-awx-postgres-13-0   awx-postgres-sc            25h

NAME                                                  STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS      AGE
persistentvolumeclaim/awx-projects-pvc                Bound    awx-projects-pv      5Gi        RWO            awx-projects-sc   25h
persistentvolumeclaim/postgres-13-awx-postgres-13-0   Bound    awx-postgres-13-pv   8Gi        RWO            awx-postgres-sc   25h
```

When the deployment completes successfully, the logs end with:

```bash
$ kubectl -n awx logs -f deployments/awx-operator-controller-manager -c awx-manager
...
----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----


PLAY RECAP *********************************************************************
localhost                  : ok=71   changed=0    unreachable=0    failed=0    skipped=48   rescued=0    ignored=1


```

Now your AWX is available at the hostname you specified `https://awx-io.mvne.postemobile.local` or `http://awx-io.mvne.postemobile.local`

![AWX URL][IMG]

Note that you have to access via hostname that you specified in `base/awx.yaml`, instead of IP address, since this guide uses Ingress.

So you should configure your **DNS** or `C:\Windows\System32\drivers\etc\hosts` file on your client where the browser is running.

![HOST FILE][HOSTS]

***

## Name

- [to be added]

## Description

- [to be added]

## Usage

- [to be added]

## Support

- [to be added]

## Contributing

- Contributions will be limited to the owner and team members of this project.

- Contributions will be subject to the approval of the Ansible Team Leader.

## Authors and acknowledgment

- PostePay - Ansible DevOps Team

[IMG]: /img_resources/awx_url.jpg "AWX URL Example"
[HOSTS]: /img_resources/host_file.jpg "Hosts File Example"

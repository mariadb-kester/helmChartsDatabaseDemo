# Master Replica MariaDB in Kubernetes

[![License](https://img.shields.io/badge/mit-blue.svg)](https://opensource.org/licenses/mit)
[![CircleCI](https://dl.circleci.com/status-badge/img/gh/mariadb-kester/helmChartsDatabaseDemo/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/mariadb-kester/helmChartsDatabaseDemo/tree/main)
![GitHub stars](https://img.shields.io/github/stars/mariadb-kester/helmChartsDatabaseDemo?style=social)
![GitHub forks](https://img.shields.io/github/forks/mariadb-kester/helmChartsDatabaseDemo?style=social)

This project contains two helm charts that will install a **Galera Cluster** or a **Master and two Replica** MariaDB servers, sitting behind a pair of Maxscale proxy servers, providing fail over and master down detection and promotion.

----

## To Use

### Terms of use

**Nothing in this demonstration is designed to be used in production and this is not supplied, supported or endorsed by
MariaDB.**

### CircleCI

This repository is designed to use [CircleCI](https://circleci.com) to build the application.

CircleCI is used to run tests against the code when checked in. 

### GitHub Actions

When committing the code to Git, GitHub Actions are used, if CircleCI successfully passes the tests.
This generates helm objects which are stored on GitHub pages. These are public and you must therefore make sure no 
private information is committed.

You can run and use these GitHub Helm Repo pages without needing to clone or fork this project.

**This means that you need to bump the chart number each time to make a new chart**

### Prerequisites

Before you begin, ensure you have met the following requirements:
<!--- These are just example requirements. Add, duplicate or remove as required --->
* You have a working Kubernetes Environment, in this example [Minikube](https://github.com/mariadb-kester/helmChartsDatabaseDemo/guides/minikube/README.md)
* Instead of using minikube you could use DigitalOcean or any other Kubernetes environment.
* You must also have [Helm](https://github.com/mariadb-kester/helmChartsDatabaseDemo/guides/helm/README.md) installed.

### Installing *helmChartsDatabaseDemo*

To install *helmChartsDatabaseDemo*, follow these steps:

To install the repository:

```
helm repo add mariadb-kester-repo https://mariadb-kester.github.io/helmChartsDatabaseDemo/
```

To search repo:

```
helm search repo mariadb-kester-repo
```

To refresh repo:

```
helm repo update
```

To remove repo:

```
helm repo remove mariadb-kester-repo
```

### Using *helmChartsDatabaseDemo*

To use *helmChartsDatabaseDemo*, follow these steps:

Building a Master / Replica cluster:

```
helm install mariadb mariadb-kester-repo/masterreplica --namespace=uk
```

Building a Galera Cluster:
```
helm install ukdc mariadb-kester-repo/galera --namespace=uk
```

Additional helm configuration options

| Description | Option |
|---|---|
| Set a Domain ID for the Cluster | --set galera.domainId=100 |
| Set the autoincrement offset | --set galera.autoIncrementOffset=1 |
| Where to clone from  | --set cloneRemote=usdc-galera-backupstream.us.svc.cluster.local  |
| A remote MaxScale to sync with  | --set remoteMaxscale=usdc-galera-masteronly.us.svc.cluster.local |
| Change master on failover name  | --set maxscale.changeMaster.name1=uktousauto |
| Change master on failover FQDN  | --set maxscale.changeMaster.host1=usdc-galera-masteronly.us.svc.cluster.local |

### Advanced uses

### Port Forwarding

It is possible to use port forwarding to connect directly to the installed and running helm chart.

To find the name of the service you want to forward to:

```
kubectl get svc -n uk
```

Once you have identified the port you can link to it via your local computer.

In Terminal A:

```
kubectl port-forward svc/masteronly -n uk 3306:3306
```

and in Terminal B:

```
mariadb -u<user> -p<password> -h127.0.0.1 -P3306 -e "select @@hostname, @@server_id;"
```

You can run the same thing but on a read write split service.

In Terminal A:

```
kubectl port-forward svc/rwsplit -n uk 3307:3307
```

and in Terminal B:

```
mariadb -u<user> -p<password> -h127.0.0.1 -P3307 -e "select @@hostname, @@server_id;"
```

### Port Connections

Instead of port forwarding you can connect directly to the IP address and port of a service.

To identify the IP address of the cluster run

```bash
???  ~ kubectl cluster-info
Kubernetes master is running at https://192.168.64.2:8443
KubeDNS is running at https://192.168.64.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

and to identify the service port you run

```bash
???  ~ kubectl get svc -n uk
NAME                                     TYPE        CLUSTER-IP      EXTERNAL-IP PORT(S)             AGE
masteronly                               NodePort    10.96.255.84    <none>        3306:30131/TCP      154m
rwsplit                                  NodePort    10.96.225.242   <none>        3307:30945/TCP      154m
```

In this example the port of the rwsplit service is `30945`.

Create two environment variables to make accessing this easier.

```bash
clusterip=192.168.64.2
portnumber=30945
```

### Demonstration

To demonstrate accessing and writting to the cluster.

### Master / Replica

You need four terminals.

On all four terminals create and set the `clusterip=192.168.64.2` and `portnumber=30945` environment variables.

On any terminal:

```bash
mariadb -uMARIADB_USER -pmariadb -h$clusterip -P$portnumber -e 'CREATE DATABASE demo; CREATE TABLE demo.test (id SERIAL PRIMARY KEY, host VARCHAR(50) NOT NULL, created DATETIME) ENGINE=INNODB DEFAULT CHARSET=utf8;'
```

On any terminal:

```bash
mariadb -uMARIADB_USER -pmariadb -h$clusterip -P$portnumber -e 'SHOW DATABASES'
```

Terminal One:

Watch the Pods:

```bash
watch "kubectl get pods -n uk"
```

Terminal Two:

Run a count on the database to watch the inserts:

```bash
watch "mariadb -uMARIADB_USER -pmariadb -h$clusterip -P$portnumber -e 'select count(*) from demo.test'"
```

Terminal Three:

Connect to MaxScale and run a watch:

```bash
kubectl exec -it -n uk mariadb-masterreplica-maxscale-active-6cb8df65fc-rwzb6 -- watch "maxctrl list servers"
```

Terminal Four:

Run an insert as a background task to run for about 5 minutes:

```bash
for ((i=1;i<=300;i++)); do mariadb -uMARIADB_USER -pmariadb -h$clusterip -P$portnumber -e 'insert into demo.test SET host='@@hostname', created=now()'; [[  $? -eq 0 ]] && sleep 1 || { echo "Down at `date`"; sleep 1; } ; done &
```

You will notice the GTID on the Maxscale servers increase, as well as the count of the records in the database. Identify which server is running as the master on the MaxScale screen and kill it:

Terminal Four:

Kill the master node.

```bash
kubectl delete pod -n uk ukdc-galera-1
```

You will notice the MaxScale watch identifies the pod as down and moves the master, the insert script will fail for a few seconds (this time depends on the configuration) and then resumes inserting data. The count on Terminal Two carries on increasing. When the node comes back in service you will notice that it rejoins the cluster as a slave and syncs to the master.

You can now check the data in the database, and will note that there are different values for the insert.

```bash
mariadb -uMARIADB_USER -pmariadb -h$clusterip -P$portnumber demo -e 'SELECT DISTINCT (host) FROM test'
```

```mysql
+-----------------------------+
| host                        |
+-----------------------------+
| mariadb-masterreplica-0 |
| mariadb-masterreplica-1 |
+-----------------------------+
```

### Contributing to *helmChartsDatabaseDemo*
<!--- If your README is long or you have some specific process or steps you want contributors to follow, consider creating a separate CONTRIBUTING.md file--->
To contribute to this *helmChartsDatabaseDemo* repository, follow these steps:

1. Fork this repository.
2. Create a branch: `git checkout -b <branch_name>`.
3. Make your changes and commit them: `git commit -m '<commit_message>'`
4. Push to the original branch: `git push origin helmChartsDatabaseDemo/<location>`
5. Create the pull request.

Alternatively see the GitHub documentation on [creating a pull request](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/creating-a-pull-request).

### Contact

If you want to contact me you can reach me at kesterriley@hotmail.com.

### License
<!--- If you're not sure which open license to use see https://choosealicense.com/--->

This project uses the following license: [MIT](https://github.com/mariadb-kester/helmChartsDatabaseDemp/blob/master/LICENSE).


### Disclaimer

Whilst, I might currently work for MariaDB, this work was originally created before my employment by MariaDB and any
development to these scripts is done strictly in my own time. It is therefore not endorsed, supported by or
recommend by MariaDB. 
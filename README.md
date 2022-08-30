# Rucio Kubernetes Tutorial

## Preliminaries

* Clone this repo to your local machine

* Install kubectl

      https://kubernetes.io/docs/tasks/tools/install-kubectl/

* Install helm

      https://helm.sh/docs/intro/install/

* Install minikube

      https://kubernetes.io/docs/tasks/tools/install-minikube/
	  https://minikube.sigs.k8s.io/docs/start/

* Start minikube with extra RAM:

      minikube start --memory='8000mb' --listen-address=0.0.0.0 --vm-driver=docker

* Add Helm chart repositories:

      helm repo add stable https://charts.helm.sh/stable
      helm repo add bitnami https://charts.bitnami.com/bitnami
      helm repo add rucio https://rucio.github.io/helm-charts

## Some helpful commands

* Activate kubectl bash completion:

      source <(kubectl completion bash)

* View all containers:

      kubectl get pods

* View/Tail logfiles of pod:

      kubectl logs <NAME>

      kubectl logs -f <NAME>

* Update helm repositories:

      helm repo update

* Shut down minikube:

      minikube stop

## Installation of Rucio + FTS + Storage

_NOTE: Before executing the following commands, please change directory to the cloned repo location_

_NOTE: Replace the pod IDs with the ones from your instance, they change every time_

* Check if you have previously done this before and want to reset from scratch. In that case, check if there's an old PostgreSQL database lying around, and find&remove it with `kubectl describe pvc` && `kubectl delete pvc data-postgres-postgresql-0`

* Install a fresh new Rucio database (PostgreSQL).

      helm install postgres bitnami/postgresql -f postgres_values.yaml

* Wait for PostgreSQL to finish starting. Output should be STATUS:Running.

      kubectl get pods

* Run init container once to setup the Rucio database once the PostgreSQL container is running:

      kubectl apply -f init-pod.yaml

* Watch the output of the init container to check if everything is fine. Pod should finish with STATUS:Completed

      kubectl logs -f init

* Install the Rucio server and wait for it to come online:

      helm install server rucio/rucio-server -f server.yaml

      kubectl logs -f server-rucio-server-7fffc4665d-ts67v rucio-server

* Prepare a client container for interactive use:

      kubectl apply -f client.yaml

* Once the client container is in STATUS:Running, you can jump into it and check if the clients are working:

      kubectl exec -it client /bin/bash

      rucio whoami

* Install the XRootD storage systems. This will start three instances of them.

      kubectl apply -f xrd.yaml

* Install the FTS database (MySQL) and wait for it to come online.

      kubectl apply -f ftsdb.yaml

      kubectl logs -f fts-mysql-db7988d96-gn6dw

* Install FTS, once the FTS database container is up and running:

      kubectl apply -f fts.yaml

      kubectl logs -f fts-server-7cb5d7c789-scg6c

* Install the Rucio daemons:

      helm install daemons rucio/rucio-daemons -f daemons.yaml

* Run FTS storage authentication delegation once:

      kubectl create job renew-manual-1 --from=cronjob/daemons-renew-fts-proxy


## Forward ports

  Let's use screen command to open new terminals that remain open

  1. server-rucio-server
    screen -S rucio-server
    kubectl port-forward --address 0.0.0.0 service/server-rucio-server 8080:80

    ctrl+a + d

  2. server-rucio-server-auth
    screen -S rucio-server-auth
    kubectl port-forward --address 0.0.0.0 service/server-rucio-server-auth 18080:80

    ctrl+a + d

  3. fts
    screen -S fts
    kubectl port-forward --address 0.0.0.0 service/fts 8446:8446

    ctrl+a + d

  4. xrd1
    screen -S xrd1
    kubectl port-forward --address 0.0.0.0 service/xrd1 1094:1094

    ctrl+a + d

  5. xrd2
    screen -S xrd2
    kubectl port-forward --address 0.0.0.0 service/xrd2 1095:1094

    ctrl+a + d

  6. xrd3
    screen -S xrd3
    kubectl port-forward --address 0.0.0.0 service/xrd3 1096:1094

    ctrl+a + d
## Connect docker client

1. Create container
  docker run \
  -e RUCIO_CFG_RUCIO_HOST=http://server-rucio-server:8080 \
  -e RUCIO_CFG_AUTH_HOST=http://server-rucio-server-auth:18080 \
  -e RUCIO_CFG_AUTH_TYPE=userpass \
  -e RUCIO_CFG_USERNAME=<username> \
  -e RUCIO_CFG_PASSWORD=<password> \
  -e RUCIO_CFG_ACCOUNT=<account> \
  --name=<username>-rucio-client \
  --add-host server-rucio-server:145.38.186.98 \
  --add-host server-rucio-server-auth:145.38.186.98 \
  --add-host fts:145.38.186.98 \
  --add-host xrd1:145.38.186.98 \
  --add-host xrd2:145.38.186.98 \
  --add-host xrd3:145.38.186.98 \
  -it -d rucio/rucio-clients:release-1.29.1


2. Enter container cli

  docker exec -it -u root <username>-rucio-client /bin/bash

## Rucio usage

* Jump into the client container

      kubectl exec -it client /bin/bash

* Create the RSEs

      rucio-admin rse add XRD1
      rucio-admin rse add XRD2
      rucio-admin rse add XRD3

* Add the protocol definitions for the storage servers

      rucio-admin rse add-protocol --hostname xrd1 --scheme root --prefix //rucio --port 1094 --impl rucio.rse.protocols.gfal.Default --domain-json '{"wan": {"read": 1, "write": 1, "delete": 1, "third_party_copy_read": 1, "third_party_copy_write": 1}, "lan": {"read": 1, "write": 1, "delete": 1}}' XRD1
      rucio-admin rse add-protocol --hostname xrd2 --scheme root --prefix //rucio --port 1095 --impl rucio.rse.protocols.gfal.Default --domain-json '{"wan": {"read": 1, "write": 1, "delete": 1, "third_party_copy_read": 1, "third_party_copy_write": 1}, "lan": {"read": 1, "write": 1, "delete": 1}}' XRD2
      rucio-admin rse add-protocol --hostname xrd3 --scheme root --prefix //rucio --port 1096 --impl rucio.rse.protocols.gfal.Default --domain-json '{"wan": {"read": 1, "write": 1, "delete": 1, "third_party_copy_read": 1, "third_party_copy_write": 1}, "lan": {"read": 1, "write": 1, "delete": 1}}' XRD3

* Enable FTS

      rucio-admin rse set-attribute --rse XRD1 --key fts --value https://fts:8446
      rucio-admin rse set-attribute --rse XRD2 --key fts --value https://fts:8446
      rucio-admin rse set-attribute --rse XRD3 --key fts --value https://fts:8446

* Fake a full mesh network

      rucio-admin rse add-distance --distance 1 --ranking 1 XRD1 XRD2
      rucio-admin rse add-distance --distance 1 --ranking 1 XRD1 XRD3
      rucio-admin rse add-distance --distance 1 --ranking 1 XRD2 XRD1
      rucio-admin rse add-distance --distance 1 --ranking 1 XRD2 XRD3
      rucio-admin rse add-distance --distance 1 --ranking 1 XRD3 XRD1
      rucio-admin rse add-distance --distance 1 --ranking 1 XRD3 XRD2

* Indefinite storage quota for root

      rucio-admin account set-limits root XRD1 -1
      rucio-admin account set-limits root XRD2 -1
      rucio-admin account set-limits root XRD3 -1

* Create a default scope for testing

      rucio-admin scope add --account root --scope test

* Create initial transfer testing data

      dd if=/dev/urandom of=file1 bs=10M count=1
      dd if=/dev/urandom of=file2 bs=10M count=1
      dd if=/dev/urandom of=file3 bs=10M count=1
      dd if=/dev/urandom of=file4 bs=10M count=1

* Upload the files

      rucio upload --rse XRD1 --scope test file1
      rucio upload --rse XRD1 --scope test file2
      rucio upload --rse XRD2 --scope test file3
      rucio upload --rse XRD2 --scope test file4

* Create a few datasets and containers

      rucio add-dataset test:dataset1
      rucio attach test:dataset1 test:file1 test:file2

      rucio add-dataset test:dataset2
      rucio attach test:dataset2 test:file3 test:file4

      rucio add-container test:container
      rucio attach test:container test:dataset1 test:dataset2

* Create a rule and remember returned rule ID

      rucio add-rule test:container 1 XRD3

* Query the status of the rule until it is completed. Note that the daemons are running with long sleep cycles (e.g. 30 seconds, 60 seconds) by default, so this will take a bit. You can always watch the output of the daemon containers to see what they are doing.

      rucio rule-info <rule_id>

* Add some more complications

      rucio add-dataset test:dataset3
      rucio attach test:dataset3 test:file4

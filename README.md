## Overview

This is an experiment for rapidly introducing an arbitrary number of Vault
Performance Replication secondary clusters.  This is accomplished by standing
up a primary cluster, and then using the official Helm chart to define a dev
cluster with a `postStart` lifecycle hook to run a small script to activate it
as a secondary performance cluster.  We then take advantage of the secondary
cluster StatefulSet to scale the number of replicas to an arbitrary number,
with each replica activating itself as an individual secondary cluster.

## Setup

1. Create a ConfigMap for the replication initialization script:
    * `kubectl create -f init-repl.yml`
2. Deploy, initialize, and unseal the Transit Unseal cluster:
    * `helm install unseal hashicorp/vault -f unseal-raft-local-values.yml`
    * `kubectl exec -it unseal-vault-0 -- vault operator init -format=json -key-shares=1 -key-threshold=1 > ./init.unseal.json`
    * `kubectl exec -it unseal-vault-0 -- vault operator unseal $(jq -r .unseal_keys_hex[] < ./init.unseal.json)
3. Enable Transit secrets engine and create Primary cluster unseal key:
    * `kubectl exec -it unseal-vault-0 -- vault login $(jq -r .root_token < ./init.unseal.json)`
    * `kubectl exec -it unseal-vault-0 -- vault token create -id=root`
    * `kubectl exec -it unseal-vault-0 -- vault secrets enable transit`
    * `kubectl exec -it unseal-vault-0 -- vault write -f transit/keys/primary-vault`
4. Deploy, initialize, and unseal the Primary cluster:
    * `helm install primary hashicorp/vault -f primary-raft-local-values.yml`
    * `kubectl exec -it primary-vault-0 -- vault operator init -format=json -recovery-shares=1 -recovery-threshold=1 > ./init.primary.json`
5. Create a root token with the id `root` on the Primary cluster:
    * `kubectl exec -it primary-vault-0 -- vault login $(jq .root_token < ./init.primary.json)`
    * `kubectl exec -it primary-vault-0 -- vault token create -id=root`
6. Enable replication on the Primary cluster:
    * `kubectl exec -it primary-vault-0 -- vault write -f sys/replication/performance/primary/enable`
7. Deploy the Secondary cluster:
    * `helm install secondary hashicorp/vault -f secondary-raft-local-values.yml`
8. Scale the Secondary cluster to your desired number of secondaries:
    * `kubectl scale sts secondary-vault --replicas=50`

```
git clone --branch gcp https://github.com/vchrisb/tap-install.git
```

# init
update env.sh

# prepare cloud shell
```
source ./env.sh
gcloud auth configure-docker $GCP_REGION-docker.pkg.dev
gcloud container clusters get-credentials $GCP_CLUSTER --region $GCP_REGION --project $GCP_PROJECT
```

# create service account for repository access
```
gcloud iam service-accounts create tap-sa --description="account for tap" --display-name="tap service-account"
gcloud artifacts repositories add-iam-policy-binding $GCP_REPO --location=$GCP_REGION --member=serviceAccount:tap-sa@$GCP_PROJECT.iam.gserviceaccount.com --role=roles/artifactregistry.repoAdmin --project $GCP_PROJECT
gcloud iam service-accounts keys create tap-sa_key.json --iam-account=tap-sa@$GCP_PROJECT.iam.gserviceaccount.com
```
# Install Tanzu CLI
```
sudo mkdir -p /etc/apt/keyrings/
sudo apt-get update
sudo apt-get install -y ca-certificates curl gpg
curl -fsSL https://packages.vmware.com/tools/keys/VMWARE-PACKAGING-GPG-RSA-KEY.pub | sudo gpg --dearmor -o /etc/apt/keyrings/tanzu-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/tanzu-archive-keyring.gpg] https://storage.googleapis.com/tanzu-cli-os-packages/apt tanzu-cli-jessie main" | sudo tee /etc/apt/sources.list.d/tanzu.list
sudo apt-get update
sudo apt-get install -y tanzu-cli
tanzu plugin install --group vmware-tap/default:v1.6.1
```

# Download Cluster Essentials for Tanzu
upload tanzu-cluster-essentials-linux-amd64-1.6.0.tgz to cloud shell
```
mkdir $HOME/tanzu-cluster-essentials
tar xfvz $HOME/tanzu-cluster-essentials-linux-amd64-1.6.0.tgz -C $HOME/tanzu-cluster-essentials

cd $HOME/tanzu-cluster-essentials
sudo install $HOME/tanzu-cluster-essentials/imgpkg /usr/local/bin/imgpkg
```

# relocate images
```
imgpkg copy -b registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@$CLUSTER_ESSENTIALS_BUNDLE_SHA --to-repo $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REPO/cluster-essentials-bundle --include-non-distributable-layers
imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:$TAP_VERSION --to-repo $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REPO/tap-packages
imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/full-deps-package-repo:$TAP_VERSION --to-repo $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REPO/full-deps-package-repo
```

# Install Cluster Essentials for Tanzu
```
cd $HOME/tanzu-cluster-essentials
./install.sh --yes
```

# Install Tanzu Application Platform package and profiles
```
cd $HOME/tap-install

kubectl create ns tap-install

tanzu secret registry add tap-registry \
  --username ${INSTALL_REGISTRY_USERNAME} --password ${INSTALL_REGISTRY_PASSWORD} \
  --server ${INSTALL_REGISTRY_HOSTNAME} \
  --export-to-all-namespaces --yes --namespace tap-install

tanzu secret registry add registry-credentials \
  --server   ${INSTALL_REGISTRY_HOSTNAME} \
  --username ${INSTALL_REGISTRY_USERNAME} \
  --password ${INSTALL_REGISTRY_PASSWORD} \
  --namespace tap-install \
  --export-to-all-namespaces \
  --yes

tanzu package repository add tanzu-tap-repository --url ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/tap-packages:$TAP_VERSION --namespace tap-install
tanzu package repository get tanzu-tap-repository --namespace tap-install

tanzu package available list tap.tanzu.vmware.com --namespace tap-install
tanzu package install tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file ./tap-values.yaml -n tap-install

tanzu package repository add full-deps-repository --url $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REPO/full-deps-package-repo:$TAP_VERSION --namespace tap-install
tanzu package repository get full-deps-repository --namespace tap-install
tanzu package install full-deps -p full-deps.buildservice.tanzu.vmware.com -v "> 0.0.0" --values-file ./tap-values.yaml --namespace tap-install

kubectl get pkgi -n tap-install

kubectl get service -n tanzu-system-ingress
```

## create workload
```
kubectl label namespaces default apps.tanzu.vmware.com/tap-ns=""
tanzu apps workload create -f ./spring-petclinic/workload.yaml
```

# letsencrypt certs

## clusterissuer.yaml
```
kubectl apply -f letsencrypt-production.yaml
```

## update installation
uncomment "ingress_issuer:" in `tap-values.yaml`
```
tanzu package installed update tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file ./tap-values.yaml -n tap-install
```

# tanzu services
```
tanzu service class list
tanzu service class get postgresql-unmanaged
tanzu service class-claim create psql-claim-1 --class postgresql-unmanaged
tanzu apps workload apply -f ./spring-petclinic/workload-service.yaml --update-strategy replace
```

# gcp services
```
gcloud iam service-accounts create tap-crossplane-sa --description="account for crossplane" --display-name="tap crossplane service-account"
gcloud projects add-iam-policy-binding $GCP_PROJECT --member=serviceAccount:tap-crossplane-sa@$GCP_PROJECT.iam.gserviceaccount.com --role roles/admin
gcloud iam service-accounts keys create tap-crossplane-sa_key.json --iam-account=tap-crossplane-sa@$GCP_PROJECT.iam.gserviceaccount.com

kubectl create secret generic gcp-creds -n crossplane-system --from-file=creds=./tap-crossplane-sa_key.json

update projectID in gcp_provider_config.yaml

kubectl apply -f crossplane/gcp_provider.yaml
kubectl apply -f crossplane/gcp_provider_config.yaml
kubectl get providers
kubectl apply -f crossplane/psql-composition.yaml
kubectl apply -f crossplane/psql-composite.yaml
kubectl apply -f crossplane/psql-clusterInstanceClass.yaml

tanzu service class-claim create psql-claim-2 --class cloudsql-postgres
```

# upgrade from TAP 1.5.3

Release notes: https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.6/tap/release-notes.html

* Install Tanzu CLI
* Download Cluster Essentials for Tanzu
* relocate images
* Install Cluster Essentials for Tanzu
* -> update tap-values.yaml


```
tanzu package repository add tanzu-tap-repository --url ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/tap-packages:$TAP_VERSION --namespace tap-install
tanzu package installed update tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file ./tap-values.yaml -n tap-install
```

(full-tbs-deb package name is changing with 1.6.1)
``` 
tanzu package installed delete full-tbs-deps -n tap-install
tanzu package repository delete tbs-full-deps-repository -n tap-install
tanzu package repository add full-deps-repository --url $INSTALL_REGISTRY_HOSTNAME/$INSTALL_REPO/full-deps-package-repo:$TAP_VERSION --namespace tap-install
tanzu package install full-deps -p full-deps.buildservice.tanzu.vmware.com -v "> 0.0.0" --values-file ./tap-values.yaml --namespace tap-install
```
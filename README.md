# helm-to-carvel-conversion-tool
A tool for converting any Helm Repository into a Carvel Package Repository including support for air gapped scenarios

# Synopsis
One of the key strengths of the Carvel Packaging is that it is not only for YTT and KAPP rather can also manage things like helm.  
In you will find a script that will generate a package repository and all the needed files from a helm repository which can be used in air gapped environments  
This will also enable you to utilize these charts in an air gapped environment  

# Options for running the tool
1. Run the script on a linux machine directly
* This gives the most flexibility but requires a lot of pre reqs on your system which are mentioned bellow in the Script based execution section
2. Utilize the Docker image ghcr.io/terasky-ltd/helm-airgaping-tool:0.1.0
* This allows for a very simple process to generate such a repo and can also be run on MAC (Windwos is also possible but requires some changes to the alias command)

## Running the container
1. You will need docker installed on your machine
2. Login to the destination oci registry with docker CLI on your machine - the docker config.json with creds is mapped into the container to ease passing in credentials
* If your docker config is using a command line tool like gcloud as a credential helper this will not work out of the box and will need manipulation of the alias command as those utils are not passed into the container.
``` bash
docker login <REGISTRY FQDN>
```  
2. Make sure that PWD and HOME env variables are set
``` bash
echo $PWD
echo $HOME
```  
3. Create a sub folder in your current directory called output
``` bash
mkdir output
```  
4. Create an alias for running the container easily with all needed mounts
``` bash
alias helm-to-pkg="docker run -i -v $HOME/.docker/config.json:/root/.docker/config.json -v $PWD/output:/output ghcr.io/terasky-ltd/helm-airgaping-tool:0.1.0"
```  
5. OPTIONAL - If you want to supply a list of charts to package instead of the entire repo create a file named chart-list.txt in the current directory and update the alias from step 4
``` bash
touch chart-list.txt
alias helm-to-pkg="docker run -i -v $HOME/.docker/config.json:/root/.docker/config.json -v $PWD/output:/output -v $PWD/chart-list.txt:/app/chart-list.txt ghcr.io/terasky-ltd/helm-airgaping-tool:0.1.0"
```
* Now fill in the chart names in the format \<REPO NAME\>/\<CHART NAME\> one per line
* When running the tool you must pass the flag "--chart-list-file-path" with the value "/app/chart-list.txt"
6. Run the following to see detailed help menu on how to run the tool
``` bash
helm-to-pkg --help
```  
7. BASH Autocompletion - run the following commands to get bash auto completion for flags
``` bash
source <(helm-to-pkg --bash-completion)
```  
8. The outputted manifests will be available in the output directory you created above
 

## Running the script
in order to run the script you will need to have pre installed:  
1. jq - https://stedolan.github.io/jq/download/  
2. yq - https://github.com/kislyuk/yq  
3. readme-generator - https://github.com/bitnami-labs/readme-generator-for-helm  
4. helm - helm v3+ - https://github.com/helm/helm  
5. json2yml - https://www.npmjs.com/package/json2yaml  
6. kbld and imgpkg - https://carvel.dev
7. helm schema-gen plugin - helm plugin https://github.com/karuppiah7890/helm-schema-gen

Once this is all installed run the following to see the commandline flags:  
``` bash
./helm-to-pkg --help
```  

Example Usage:  
```bash
./helm-to-pkg.sh \
    --helm-repo-name bitnami \
    --helm-repo-url https://charts.bitnami.com/bitnami \
    --number-of-chart-versions 4 \
    --package-repository-name bitnami-oss \
    --oci-image-repository bitnami-charts \
    --oci-registry-fqdn harbor.example.com \
    --package-domain-suffix bitnami.charts
```  
  
Parameterized instructions for using the generated repo will be printed at the end of the script execution.  
  
# General Usage Instructions
## Installing the package repo in a non air gapped environment
You can choose either of the following options:  
  
1. Add as a global package to your tanzu cluster with default settings:  
```bash
    tanzu package repository add <PACKAGE REPOSITORY NAME> --url <PACKAGE REPOSITORY URL> --namespace tanzu-package-repo-global
```  
2. Add as a global package to your tanzu cluster with custom sync interval:  
```bash
    ## If run with the docker mode 
    # cd output
    ## If run via script directly run:
    # cd /tmp/carvel-bitnami-packages/

    ## For both cases run
    kubectl apply -n tanzu-package-repo-global -f package-repository-manifest.yaml  
```  
  
  
## Air Gapped Instructions  
  
1. Run the following command to copy all packages and images into a tar ball on your machine:  
```bash
    imgpkg copy -b harbor.example.com/bitnami-charts/bitnami-oss:latest --to-tar /tmp/bitnami-oss.tar --registry-verify-certs=false  
```  
2. Import the Tar file to the airgapped environment  
3. Run the following to import the artifacts to an OCI registry in your air gapped environment:  
```bash
    imgpkg copy --tar /tmp/bitnami-oss.tar --to-repo <AIR GAPPED REGISTRY>/<AIR GAPPED REPO> --registry-verify-certs=false  
```
4. Add the repo to your cluster with either of the following options:  
*  Add as a global package to your tanzu cluster with default settings:  
```bash
    tanzu package repository add <PACKAGE REPOSITORY NAME> --url <AIR GAPPED REGISTRY>/<AIR GAPPED REPO>:<PACKAGE REPO TAG> --namespace tanzu-package-repo-global
```  
* Add as a global package to your tanzu cluster with custom sync interval:  
** edit the generated manifest package-repository-manifest.yaml and change the image reference to your airgapped environment  
** add the repo to your cluster  
```bash
    kubectl apply -n tanzu-package-repo-global -f package-repository-manifest.yaml
```


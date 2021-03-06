# KOPS

## Links

* https://kubernetes.io/docs/setup/production-environment/tools/kops/
* https://kubernetes.io/docs/tasks/tools/install-kubectl/

## Using Existing VPC and Subnets

See [HA Cluster](docs/create-ha-cluster.md).

## Manual Steps

* Update AWS configuration file, `$HOME/.aws/credentials`. Set AWS_PROFILE in `$HOME/.bashrc`.

* Create an S3 bucket for random stuff.

```
export S3_RANDOM=davidmm-0341b7d4-4de1-11ea-b20a-9f9248f37193
aws s3 mb s3://$S3_RANDOM
```

* Register a domain using Route53. For example, using va-oit.cloud. Wait until the domain has been provisioned and you can find it using a command like the following.

```
dig NS va-oit.cloud
```

* Create an S3 bucket to store the cluster's state. Who ever has access to this bucket can do bad things to the cluster. Therefore, pay attention to its security configuration. Don't use periods in the bucket name.

```
aws s3 mb s3://va-oit-cloud--16d7b802-4de6-11ea-924b-ef8fcd6cbcb5
```

* Export an environment variable to let `kops` know where to store its state. You could add this variable to your `$HOME/.bashrc` file if you are only working with one cluster or if you want one bucket to hold the states of multiple clusters.

```
export KOPS_STATE_STORE=s3://va-oit-cloud--16d7b802-4de6-11ea-924b-ef8fcd6cbcb5
```

* Specify a cluster name.

```
export NAME=va-oit.cloud
```

* Install `kubectl`.

```
export STABLE_VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
curl -LO https://storage.googleapis.com/kubernetes-release/release/$STABLE_VERSION/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

* Install auto-completion package.

```
sudo apt-get install bash-completion
```

* Enable kubectl auto-completion when your bash shell starts.

```
echo 'source <(kubectl completion bash)' >>$HOME/.bashrc
```

* Install `kops`.

```
export KOPS_VERSION=$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)
curl -LO https://github.com/kubernetes/kops/releases/download/$KOPS_VERSION/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
```

* Check installation.

```
kubectl version --client
kops version
```

* Create an AWS EC2 key pair. For example, it could be called va-oit-cloud. Then copy the file to S3 for safekeeping. Note that you should lock down the permissions as well. The last thing to do is generate a public key from the PEM file (the private key)

```
chmod 600 $HOME/Downloads/$NAME.pem
ssh-keygen -y -f $HOME/Downloads/$NAME.pem > $HOME/Downloads/$NAME.pub
aws s3 cp $HOME/Downloads/$NAME.pem s3://$S3_RANDOM
aws s3 cp $HOME/Downloads/$NAME.pub s3://$S3_RANDOM
```

* Create the cluster configuration.

```
kops create cluster \
  --cloud=aws \
  --zones=us-east-1a \
  --ssh-public-key $HOME/Downloads/$NAME.pub \
  $NAME
```

* As a side note, you can delete the configuration you just created using the following command.

```
kops delete cluster --name va-oit.cloud --yes
```

* Configure the cluster.

```
kops update cluster --name $NAME --yes
```

* Wait several minutes for the EC2 instances to become active. The validate the cluster is running.

```
kops validate cluster
```

* You can list the nodes.

```
kubectl get nodes
```

* You can SSH to the master node but I don't recommend this. If you are not using Debian as the base operating system, you might need to use a different user than `admin`.

```
ssh -i $HOME/Downloads/$NAME.pem admin@api.$NAME
```

### Visit Cluster Home Page

* Learn cluster URL.

```
kubectl cluster-info
```

* Learn cluster admin password.

```
kubectl config view --minify --output jsonpath="{.users[?(@.user.username=='admin')].user.password}";echo
```

* Visit the cluster URL using the `admin` username and the revealed password.

# Kubernetes on AWS poweredby Turbonomic

This is the source of the `kube-aws` tool and the installation artifacts used by the official Kubernetes on AWS documentation.

### What's different from upstream

This repo creates kubernetes cluster on AWS from scratch, which uses Turbonomic Kubeturbo(https://github.com/vmturbo/kubeturbo) scheduler, to replace the default kubernetes scheduler,
and provide an advanced full-stack controller using kubeturbo.

##Prerequisites

1. Make sure you have Turbonomic instance installed with the AMI (ami-4d01295a).

2. Install kube-aws binary

   kube-aws binary can be downloaded [here](https://github.com/turbonomic/coreos-kubernetes/raw/master/multi-node/aws/kube-aws).

  ```sh
  mv kube-aws /usr/local/bin
  ```

### AWS Credentials
The supported way to provide AWS credentials to kube-aws is by exporting the following environment variables:

```sh
export AWS_ACCESS_KEY_ID=AKID1234567890
export AWS_SECRET_ACCESS_KEY=MY-SECRET-KEY
```

### Create a KMS Key

[Amazon KMS](http://docs.aws.amazon.com/kms/latest/developerguide/overview.html) keys are used to encrypt and decrypt cluster TLS assets. If you already have a KMS Key that you would like to use, you can skip this step.

Creating a KMS key can be done via the AWS web console or via the AWS cli tool:

```shell
$ aws kms --region=us-west-1 create-key --description="kube-aws assets"
{
    "KeyMetadata": {
        "CreationDate": 1458235139.724,
        "KeyState": "Enabled",
        "Arn": "arn:aws:kms:us-west-1:xxxxxxxxx:key/xxxxxxxxxxxxxxxxxxx",
        "AWSAccountId": "xxxxxxxxxxxxx",
        "Enabled": true,
        "KeyUsage": "ENCRYPT_DECRYPT",
        "KeyId": "xxxxxxxxx",
        "Description": "kube-aws assets"
    }
}
```
You'll need the `KeyMetadata.Arn` string for the next step:

## Initialize an asset directory
```sh
$ mkdir my-cluster
$ cd my-cluster
$ kube-aws init --cluster-name=<my-cluster-name> \
--external-dns-name=<my-cluster-endpoint> \
--region=us-west-1 \
--availability-zone=us-west-1c \
--key-name=<key-pair-name> \
--kms-key-arn="arn:aws:kms:us-west-1:xxxxxxxxxx:key/xxxxxxxxxxxxxxxxxxx" \
--serveraddress=<Turbonomic_Server_IP> \
--opsmanagerusername=<Turbonomic_Username> \
--opsmanagerpassword=<Turbonomic_Password>
```

There will now be a cluster.yaml file in the asset directory.

## Render contents of the asset directory

```sh
$ kube-aws render
```

This generates the default set of cluster assets in your asset directory. These assets are templates and credentials that are used to create, update and interact with your Kubernetes cluster.

You can now customize your cluster by editing asset files:

* **cluster.yaml**

  This is the configuration file for your cluster. It contains the configuration parameters that are templated into your userdata and cloudformation stack.

* **userdata/**

  * `cloud-config-worker`
  * `cloud-config-controller`

  This directory contains the [cloud-init](https://github.com/coreos/coreos-cloudinit) cloud-config userdata files. The CoreOS operating system supports automated provisioning via cloud-config files, which describe the various files, scripts and systemd actions necessary to produce a working cluster machine. These files are templated with your cluster configuration parameters and embedded into the cloudformation stack template.

* **stack-template.json**

  This file describes the [AWS cloudformation](https://aws.amazon.com/cloudformation/) stack which encompasses all the AWS resources associated with your cluster. This JSON document is templated with configuration parameters, we well as the encoded userdata files.

* **credentials/**

  This directory contains the **unencrypted** TLS assets for your cluster, along with a pre-configured `kubeconfig` file which provides access to your cluster api via kubectl.

You can also now check the `my-cluster` asset directory into version control if you desire. The contents of this directory are your reproducible cluster assets. Please take care not to commit the `my-cluster/credentials` directory, as it contains your TLS secrets. If you're using git, the `credentials` directory will already be ignored for you.

## Route53 Host Record (optional)

`kube-aws` can optionally create an A record for the controller IP in an existing hosted zone.

Edit the `cluster.yaml` file:

```yaml
externalDNSName: my-cluster.staging.core-os.net
createRecordSet: true
hostedZone: staging.core-os.net
```

If `createRecordSet` is not set to true, the deployer will be responsible for making externalDNSName routable to the controller IP after the cluster is created.

## Validate your cluster assets

The `validate` command check the validity of the cloud-config userdata files and the cloudformation stack description:

```sh
$ kube-aws validate
```

## Create a cluster from asset directory

```sh
$ kube-aws up
```

This command can take a while.

## Access the cluster

```sh
$ kubectl --kubeconfig=kubeconfig get nodes
```

It can take some time after `kube-aws up` completes before the cluster is available. Until then, you will have a `connection refused` error.

## Export your cloudformation stack

```sh
$ kube-aws up --export
```

## Development

### Build

Run the `./build` script to compile `kube-aws` locally.

This depends on having:
* golang >= 1.5

The compiled binary will be available at `bin/kube-aws`.

### Run Unit Tests

```sh
go test $(go list ./... | grep -v '/vendor/')
```

### Modifying templates

The various templates are located in the `pkg/config/templates/` folder of the source repo. `go generate` is used to pack these templates into the source code. In order for changes to templates to be reflected in the source code:

```sh
go generate ./pkg/config
```

This command is run automatically as part of the `build` script.

### Useful Resources

The following links can be useful for development:

- [AWS CloudFormation resource types](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)

## Contributing

Submit a PR to this repository, following the [contributors guide](../../CONTRIBUTING.md).
The documentation is published from [this source](../../Documentation/kubernetes-on-aws.md).



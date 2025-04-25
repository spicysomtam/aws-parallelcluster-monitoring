# Grafana Dashboard for AWS ParallelCluster 

This is a sample solution based on Grafana for monitoring various component of an HPC cluster built with AWS ParallelCluster.
There are 6 dashboards that can be used as they are or customized as you need.
* [ParallelCluster Summary](https://github.com/aws-samples/aws-parallelcluster-monitoring/blob/main/grafana/dashboards/ParallelCluster.json) - this is the main dashboard that shows general monitoring info and metrics for the whole cluster. It includes Slurm metrics and Storage performance metrics.
* [HeadNode Details](https://github.com/aws-samples/aws-parallelcluster-monitoring/blob/main/grafana/dashboards/master-node-details.json) - this dashboard shows detailed metric for the HeadNode, including CPU, Memory, Network and Storage usage.
* [Node List](https://github.com/aws-samples/aws-parallelcluster-monitoring/blob/main/grafana/dashboards/node-list.json) - this dashboard lists all the HPC nodes (HeadNode, Compute, LoginNode). Each entry is a link to a more detailed page.
* [Compute/Login Node Details](https://github.com/aws-samples/aws-parallelcluster-monitoring/blob/main/grafana/dashboards/compute-login-node-details.json) - similarly to the HeadNode details this dashboard show the same metric for compute and login nodes (not GPU nodes). Todo: Split these out into seperate dashboards.
* [GPU Nodes Details](https://github.com/aws-samples/aws-parallelcluster-monitoring/blob/main/grafana/dashboards/gpu.json) - This dashboard shows GPUs releated metrics collected using nvidia-dcgm container.
* [Cluster Logs](https://github.com/aws-samples/aws-parallelcluster-monitoring/blob/main/grafana/dashboards/logs.json) - This dashboard shows all the logs of your HPC Cluster. The logs are pushed by AWS ParallelCluster to AWS ClowdWatch Logs and finally reported here.
* [Cluster Costs](https://github.com/aws-samples/aws-parallelcluster-monitoring/blob/main/grafana/dashboards/costs.json)(beta / in developemnt) - This dashboard shows the cost associated to AWS Service utilized by your cluster. It includes: [EC2](https://aws.amazon.com/ec2/pricing/), [EBS](https://aws.amazon.com/ebs/pricing/), [FSx](https://aws.amazon.com/fsx/lustre/pricing/), [S3](https://aws.amazon.com/s3/pricing/), [EFS](https://aws.amazon.com/efs/pricing/).


## AWS ParallelCluster
**AWS ParallelCluster** is an AWS supported Open Source cluster management tool that makes it easy for you to deploy and
manage High Performance Computing (HPC) clusters in the AWS cloud.
It automatically sets up the required compute resources and a shared filesystem and offers a variety of batch schedulers such as AWS Batch, SGE, Torque, and Slurm.
* More info on: https://aws.amazon.com/hpc/parallelcluster/
* Source Code on Git-Hub: https://github.com/aws/aws-parallelcluster
* Official Documentation: https://docs.aws.amazon.com/parallelcluster/


## Solution components
This project is build with the following components:

* **Grafana** is an [open-source](https://github.com/grafana/grafana) platform for monitoring and observability. Grafana allows you to query, visualize, alert on and understand your metrics as well as create, explore, and share dashboards fostering a data driven culture. 
* **Prometheus** [open-source](https://github.com/prometheus/prometheus/) project for systems and service monitoring from the [Cloud Native Computing Foundation](https://cncf.io/). It collects metrics from configured targets at given intervals, evaluates rule expressions, displays the results, and can trigger alerts if some condition is observed to be true.  
* The **Prometheus Pushgateway** is on [open-source](https://github.com/prometheus/pushgateway/) tool that allows ephemeral and batch jobs to expose their metrics to Prometheus.
* **[Nginx](http://nginx.org/)** is an HTTP and reverse proxy server, a mail proxy server, and a generic TCP/UDP proxy server.
* **[Prometheus-Slurm-Exporter](https://github.com/vpenso/prometheus-slurm-exporter/)** is a Prometheus collector and exporter for metrics extracted from the [Slurm](https://slurm.schedmd.com/overview.html) resource scheduling system.
* **[Node_exporter](https://github.com/prometheus/node_exporter)** is a Prometheus exporter for hardware and OS metrics exposed by \*NIX kernels, written in Go with pluggable metric collectors.

Note: *while almost all components are under the Apache2 license, only **[Prometheus-Slurm-Exporter is licensed under GPLv3](https://github.com/vpenso/prometheus-slurm-exporter/blob/master/LICENSE)**, you need to be aware of it and accept the license terms before proceeding and installing this component.*


## Example Dashboards

#### Cluster Overview

![ParallelCluster](docs/ParallelCluster.png?raw=true "AWS ParallelCluster")

#### HeadNode Dashboard

![Head Node](docs/HeadNode.png?raw=true "Head Node")

#### Node List Dashboard

![Node List](docs/List-new.png?raw=true "Node List")

#### Logs

![Logs](docs/Logs.png?raw=true "AWS ParallelCluster Logs")

#### Cluster Costs

![Costs](docs/Costs.png?raw=true "Best - AWS ParallelCluster Costs")


## Quickstart for Parallel Cluster 3.x

Tested with Parallel Cluster 3.11.1 on Amazon Linux 2023 (Amazon Linux 2 should also work but nearing end of life so use 2023 instead). 

Updated deployment and instructions April 2025.

1. Create a Security Group that allows you to access the `HeadNode` on Port 80 and 443 from the Internet. In the following example we open the security group up to `0.0.0.0/0`, which is fine for testing, but we highly advise restricting this down further. More information on how to create your security groups can be found [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-ec2-sg.html#creating-a-security-group)

```bash
read -p "Please enter the vpc id of your cluster: " vpc_id
echo -e "creating a security group with $vpc_id..."
security_group=$(aws ec2 create-security-group --group-name grafana-sg --description "Open HTTP/HTTPS ports" --vpc-id ${vpc_id} --output text)
aws ec2 authorize-security-group-ingress --group-id ${security_group} --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${security_group} --protocol tcp --port 80 —-cidr 0.0.0.0/0
```
2. Create a security group for the scraping of stats by prometheus node-exporter on `Compute` and `Login` nodes (which includes GPU nodes). Set the cidr range to the vpc cidr, since we don't know the IP address of the head node (parallel cluster not deployed yet):
```bash
read -p "Please enter the vpc id of your cluster: " vpc_id
echo -e "creating a security group with $vpc_id..."
security_group=$(aws ec2 create-security-group --group-name node-exporter-sg --description "Scraping compute and login node stats via prometheus" --vpc-id ${vpc_id} --output text)
aws ec2 authorize-security-group-ingress --group-id ${security_group} --protocol tcp --port 9100 --cidr <cidr-of-vpc>
```

3. Configure the [parallel cluster configuration file](https://docs.aws.amazon.com/parallelcluster/latest/ug/cluster-configuration-file-v3.html) (see below example):
* Use the post install script **post-install.sh** for the `HeadNode`, `LoginNodes` and `Scheduling` sections (Compute and GPU nodes).
* The `grafana-sg` Security Group you created above is included in `AdditionalSecurityGroups` for only the `HeadNode` section. 

* The `node-exporter-sg` Security Group you created above is included in `AdditionalSecurityGroups` for `LoginNode` and `Scheduling` sections (Compute and GPU nodes).
* Add the Iam `AdditionalIamPolicies` to only the `HeadNode` section.
* Include the following `Tags` to the configuration file.

```yaml
CustomActions:
  OnNodeConfigured:
    Script: https://raw.githubusercontent.com/spicysomtam/aws-parallelcluster-monitoring/main/post-install.sh
    Args:
      - main
Iam:
  AdditionalIamPolicies:
    - Policy: arn:aws:iam::aws:policy/CloudWatchFullAccess
    - Policy: arn:aws:iam::aws:policy/AWSPriceListServiceFullAccess
    - Policy: arn:aws:iam::aws:policy/AmazonSSMFullAccess
    - Policy: arn:aws:iam::aws:policy/AWSCloudFormationReadOnlyAccess
Tags:
  - Key: 'Grafana'
    Value: 'true'
```

4. Connect to `https://headnode_public_ip` (ignore the invalid certificate warning).  Authenticate to Grafana as `admin` with the default [Grafana password](https://github.com/aws-samples/aws-parallelcluster-monitoring/blob/main/docker-compose/docker-compose.master.yml#L43). A landing page will be presented to you with links to the Prometheus data collector service and the Grafana dashboards.

![Login Screen](docs/Login1.png?raw=true "Login Screen")
![Login Screen](docs/Login2.png?raw=true "Login Screen")

Note: *Because of the higher volume of network traffic due to the compute nodes continuously pushing metrics to the HeadNode, in case you expect to run a large scale cluster (hundreds of instances), we would recommend to use an instance type slightly bigger than what you planned for your master node.*

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](https://github.com/aws-samples/aws-parallelcluster-monitoring/blob/main/LICENSE) file.

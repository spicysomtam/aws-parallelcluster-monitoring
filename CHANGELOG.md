# Changelog

## April 2025 Alastair Munro:
- Fixed prometheus-slurm-exporter build; added arg -buildvcs=false.
- Upgraded docker-compose to a new version; version installed really old.
- Fixed issue with prometheus unable to scrape ec2_instances (changed networking from host to bridge).
- prometheus also reports on LoginNodes. Todo: break out compute and login nodes in Grafana; currently login nodes combined with compute nodes.
- Docker compose using tagged images rather than 'latest'; more predictable behaviour in the future.
- Removed docker compose version from docker compose files; obsolite.
- Added login nodes to the environment and installed node-exporter in docker for prometheus on login nodes.
- Added node-exporter-sg to instructions and config.
- Fixed CW logs; CW log group has changed from `<stack>` to `<stack>-<date-stamp>`. Also grafana cannot get AWS creds so change docker network to bridge.
- Fixed up prometheus `ec2_sd_config` so it works for all instance types, only uses port 9100 and there is just one iprometheus config definition.
- Added **Instance Name** to node list so you can see what it is **Compute/Login/HeadNode**. Compute Node List renamed to Node List and lists all nodes so you know what each node is.
- Compute-node-details dashboard renamed to compute-login-node-details.
- Unable to test gpu dashboard but it does seem to filter on GPU instance types using regex `[pg][2-4].*` (`p` or `g` EC2 instance types).
- Cron expression invalid for `1h-cost-metrics.sh` so not being run. Fixed.
- Todo: 1m and 1h scripts generating custom metrics (`1m-cost-metrics.sh`, `1h-cost-metrics.sh`) need alot of work to bring up to date.
- Quickstart instructions incomplete and duplicated. Corrected and removed duplicate section.
- Updated Dashboard Master node to be called HeadNode.

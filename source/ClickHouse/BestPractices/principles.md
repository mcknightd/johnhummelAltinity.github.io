---
title: Principles of Operation
layout: default
published: true
order: 2
---
Organizations that run ClickHouse successfully share the following general characteristics.

|Principle|Pattern|Anti-Pattern|
|---|---|---|
|Assignment of responsibility|Designated personnel access production environments and only for designated tasks|Personnel randomly access production for any reason|
|Configuration control|The production environment is carefully controlled and changes from one known state to another with changes planned/approved in advance|Changes are made willy-nilly and without approval|
|Automation|Procedures are automated using appropriate mechanisms like Ansible, CI/CD pipelines, Terraform, or similar tools|Procedures are performed manually across large clusters|
|Monitoring|All facets of the production environment including ClickHouse and Zookeeper are monitored in real-time with access to history and automatic alerting|Administrators run manual queries to check production state|
|Thorough Testing|Changes and new configurations are tested carefully in prepared environments using data and load that is as similar to production as possible|New configurations are deployed directly to prod without load or upgrade testing|
|Canary Deployment|Major upgrades are introduced to production in incremental steps with tested rollback procedures|Major upgrades are applied to all nodes simultaneously without “bake-in”|

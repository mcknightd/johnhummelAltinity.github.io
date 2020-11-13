---
title: Overview
layout: default
published: true
order: 2
---
Organizations that run ClickHouse successfully share the following general characteristics.


<table>
  <tr>
   <td><strong>Principle</strong>
   </td>
   <td><strong>Pattern</strong>
   </td>
   <td><strong>Anti-Pattern</strong>
   </td>
  </tr>
  <tr>
   <td>Assignment of responsibility
   </td>
   <td>Designated personnel access production environments and only for designated tasks
   </td>
   <td>Personnel randomly access production for any reason
   </td>
  </tr>
  <tr>
   <td>Configuration control
   </td>
   <td>The production environment is carefully controlled and changes from one known state to another with changes planned/approved in advance
   </td>
   <td>Changes are made willy-nilly and without approval
   </td>
  </tr>
  <tr>
   <td>Automation
   </td>
   <td>Procedures are automated using appropriate mechanisms like Ansible, CI/CD pipelines, Terraform, or similar tools
   </td>
   <td>Procedures are performed manually across large clusters
   </td>
  </tr>
  <tr>
   <td>Monitoring
   </td>
   <td>All facets of the production environment including ClickHouse and Zookeeper are monitored in real-time with access to history and automatic alerting
   </td>
   <td>Administrators run manual queries to check production state
   </td>
  </tr>
  <tr>
   <td>Thorough Testing
   </td>
   <td>Changes and new configurations are tested carefully in preprod environments using data and load that is as similar to production as possible
   </td>
   <td>New configurations are deployed directly to prod without load or upgrade testing
   </td>
  </tr>
  <tr>
   <td>Canary Deployment
   </td>
   <td>Major upgrades are introduced to production in incremental steps with tested rollback procedures
   </td>
   <td>Major upgrades are applied to all nodes simultaneously without “bake-in”
   </td>
  </tr>
</table>
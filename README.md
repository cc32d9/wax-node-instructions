# WAX blockchain node installation guides

This is a set of installation documents for WAX blockchain nodes in various environments.

The common part is that mounting the node state into a `tmpfs` partition proved to be the most efficient approach. 
This normally requires a large swap partition in adition to at least 128GB RAM.

A WAX API node without state history is still possible to launch with less than 128GB RAM, although the node performance may degrade. 
With state history plugin enabled, 128GB RAM is the minimum requirement.

ZFS filesystem is used where possible. It has a number of benefits, compared to other filesystems: 
where needed, `lz4` compression can be enabled. Also quick and lightweight snapshots are very useful. 
Also different caching strategies can be selected for different filesystems.

The following documents are available:

* [WAX API node installation at AWS](aws-wax-api.md)

* [WAX state history node installation at AWS](aws-wax-ship.md)


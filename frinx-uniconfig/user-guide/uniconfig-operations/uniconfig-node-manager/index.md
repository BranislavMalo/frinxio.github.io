# UniConfig Node Manager

This component is responsible for maintaining the configuration on devices based on their intended configuration. Each device and its configuration are represented as a node in the UniConfig topology. The node's configuration is described using OpenConfig YANG models.

The northbound API of the UniConfig Manager (UNM) is RPC-driven and provides commit functionality with automatic rollback and synchronization of configurations from the network.

When a commit is called, UNM creates a diff based on the intended state in the **Config** datastore and the actual state in the **Operational** datastore. This diff is used as the basis for device configuration. UNM prepares a network-wide transaction that uses unified mountpoints for communication with different types of devices. An additional git-like diff RPC is used to display all changes grouped under root elements in a git-like style.

If configuration fails for one device, UNM executes an automatic rollback where the previous configuration is restored on all modified devices.

Synchronization from the network reads configurations from devices and stores them as actual states in the **Operational** datastore.

See below for additional information on individual RPCs:

- [RPC calculate-diff](../uniconfig-node-manager/rpc_calculate-diff)
- [RPC calculate-git-like-diff](../uniconfig-node-manager/rpc_calculate-git-like-diff)
- [RPC checked-commit](../uniconfig-node-manager/rpc_checked-commit)
- [RPC check-installed-nodes](../uniconfig-node-manager/uniconfig_check_installed_devices)
- [RPC commit](../uniconfig-node-manager/rpc_commit)
- [RPC compare-config](../uniconfig-node-manager/rpc_compare-config)
- [RPC get-installed-nodes](../uniconfig-node-manager/uniconfig_get_installed_devices)
- [RPC check-nodes-connection](../uniconfig-node-manager/uniconfig_check_nodes_connection)
- [RPC health](../uniconfig-node-manager/rpc_health)
- [RPC install-multiple-nodes](../uniconfig-node-manager/uniconfig_install_multiple_nodes)
- [RPC is-in-sync](../uniconfig-node-manager/rpc_is-in-sync)
- [RPC replace-config-with-operational](../uniconfig-node-manager/rpc_replace-config-with-oper)
- [RPC sync-from-network](../uniconfig-node-manager/rpc_sync-from-network)
- [RPC validate](../uniconfig-node-manager/rpc_validate)
- [RPC uninstall-multiple-nodes](../uniconfig-node-manager/uniconfig_uninstall_multiple_nodes)

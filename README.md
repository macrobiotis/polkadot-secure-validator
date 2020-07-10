[![CircleCI](https://circleci.com/gh/w3f/polkadot-secure-validator.svg?style=svg)](https://circleci.com/gh/w3f/polkadot-secure-validator)

# Polkadot Secure Validator Setup

This repo describes a potential setup for a Polkadot validator that aims to
prevent some types of potential attacks, as described in the
[Polkadot Secure Validator approach](https://hackmd.io/QSJlqjZpQBihEU_ojmtR8g).
The [Workflow](#workflow) section describes the [Platform Layer](#platform-layer)
and the [Application Layer](#application-layer) in more detail.

![Polkadot Secure Network Chart](secure_network_chart.svg)

## Usage

There are two ways of using this repository:

* **Platform & Application Layer**

  Configure credentials for infrastructure providers such as AWS, Azure, GCP
  and/or Packet, then execute the Terraform process to automatically deploy the
  required machines ([Platform Layer](#platform-layer)) and setup the
  [Application Layer](#application-layer).

  See the [Complete Guide](GUIDE_COMPLETE.md) for more.

* **Application Layer**

  Setup Debian-based machines yourself, which only need basic SSH access and
  configure those in an inventory. The Ansible scripts will setup the entire
  [Application Layer](#application-layer).

  See the [Ansible Guide](GUIDE_ANSIBLE.md) for more.

## Structure

The secure validator setup is composed of one or more validators and a set of 
public nodes nodes connected to it. The validators are isolated from the internet 
and only have access to the Polkadot network through
the public nodes.

The connection between the validator nodes and the public nodes is performed
defining a VPN to which all these nodes belong. The Polkadot instance running in
the validator nodes are configured to only listen on the VPN-attached interface,
and uses the public node's VPN address in the `--reserved-nodes` parameter. It is
also protected by a firewall that only allows connections on the VPN port.

This way, the only nodes allowed to connect to the validators are the public nodes
through the VPN. Messages sent by other validators can still reach it through
gossiping, and these validators can know the IP address of the secure validator
because of this, but can't directly connect to it without being part of the VPN.

*WARNING*

If you use this tool to create and/or configure your validator setup or
implement your setup based on this approach take into account that if you add
public telemetry endpoints to your nodes (either the validator or the public
nodes) then the IP address of the validator will be publicly available too,
given that the contents of the network state RPC call are sent to telemetry.

Even though the secure validator in this setup only has the VPN port open and
Wireguard has a reasonable [approach to mitigate DoS attacks](https://www.wireguard.com/protocol/#dos-mitigation),
we recommend to not send this information to endpoints publicly accessible.

## Workflow

The secure validator setup is structured in two layers, an underlying platform
and the applications that run on top of it.

### Platform Layer

Both validator and public nodes are created in a similar way using the terraform
modules located at [terraform](/terraform) directory. We have created code for 
several providers but it is possible to add new ones, please reach out if you 
are interested in any provider currently not available.

Besides the actual machines the terraform modules create the minimum required networking
infrastructure for adding firewall rules to protect the nodes.

### Application Layer

This is done through the ansible playbook and roles located at [ansible](/ansible), the
configuration applied depend on the type of node:

* Common:

    * Software firewall setup, for the validator we only allow the VPN and SSH
    ports, for the public nodes VPN SSH and p2p ports.

    * VPN setup: for the VPN solution we are using [WireGuard](https://github.com/WireGuard/WireGuard),
    at this stage we create the private and public keys on each node, making the
    public keys available to ansible.

    * VPN install: we install and configure WireGuard on each host using the public
    keys from the previous stage. The configuration for the validator looks like:

        ```
        [Interface]
        PrivateKey = <...>
        ListenPort = 51820
        SaveConfig = true
        Address = 10.0.0.1/24

        [Peer]
        PublicKey = 8R7PTv1CdNLHRsDvrvE58Ac0Inc9vOLY2vFMWIFV/W4=
        AllowedIPs = 10.0.0.2/32
        Endpoint = 64.93.77.93:51820

        [Peer]
        PublicKey = ZZW6Wuk+YjJToeLHIUrp0HAqfNozgQfUMo2owC2Imzg=
        AllowedIPs = 10.0.0.3/32
        Endpoint = 50.81.184.50:51820

        [Peer]
        PublicKey = LZHKtuGCxz9iCoNNDmQzzNe9eF9aLXj/4yJRkFjCWzM=
        AllowedIPs = 10.0.0.4/32
        Endpoint = 45.243.244.130:51820
        ```

    * Polkadot setup: create a Polkadot user and group and download the binary.

* Public nodes:

    * Start Polkadot service: the public nodes are started and we make the libp2p peer
    id of the node available to ansible. The generated systemd unit looks like:

        ```
        [Unit]
        Description=Polkadot Node

        [Service]
        ExecStart=/usr/local/bin/polkadot \
            --name sv-public-0 \
            --sentry

        Restart=always

        [Install]
        WantedBy=multi-user.target
        ```

* Private (validator) nodes:

    * Start Polkadot service: the private (validator) node is started with the node's VPN address as part
    of the listen multiaddr and the multiaddr of the public nodes (with the peer id
    from the previous stage and the VPN addresses) as `reserved-nodes`. It looks like:

        ```
        [Unit]
        Description=Polkadot Node

        [Service]
        ExecStart=/usr/local/bin/polkadot \
            --name sv-private \
            --validator \
            --listen-addr=/ip4/10.0.0.1/tcp/30333 \
            --reserved-nodes /ip4/10.0.0.2/tcp/30333/p2p/QmNpQbu2nKfHQMySnCue3XC9mAjBfzi8DQ9KvNwUM8jZdx \
            --reserved-nodes /ip4/10.0.0.3/tcp/30333/p2p/QmY81TLZKeNj4mGDAhFQE6RrHEJPidAkccgUTsJo7ifNFJ \
            --reserved-nodes /ip4/10.0.0.4/tcp/30333/p2p/QmTwMDJDnPyHUHV2fZFcVbNpYzp6Fu7LP6VhhK3Ei13iXr

        Restart=always

        [Install]
        WantedBy=multi-user.target
        ```

## Scopes

This setup partitions the network in 3 separate kind of nodes: secure validator,
its public node and the regular network nodes, haveing each group a different
vision and accessibility to the rest of the network. To verify this, we'll execute
the `system_networkState` RPC call on nodes of each partition:

```
curl -H "Content-Type: application/json" --data '{ "jsonrpc":"2.0", "method":"system_networkState", "params":[],"id":1 }' localhost:9933
```

### Validator

It can only reach and be reached by its public nodes, from the
`system_networkState` RPC call:
```
{

  [ ........ ]

  "result": {
    "connectedPeers": {

      [ only validator's public nodes shown here]

      "QmPjNcWNZjNrjVFzkNYR6jH7HLqyU7j9piczUyNoxce1fD": {
        "enabled": true,
        "endpoint": {
          "dialing": "/ip4/10.0.0.2/tcp/30333"
        },
        "knownAddresses": [
          "/ip6/::1/tcp/30333",
          "/ip4/10.0.0.2/tcp/30333",
          "/ip4/127.0.0.1/tcp/30333",
          "/ip4/172.26.59.86/tcp/30333",
          "/ip4/18.197.157.119/tcp/30333"
        ],
        "latestPingTime": {
          "nanos": 256512049,
          "secs": 0
        },
        "open": true,
      },

      [ ........ ]

    },
    "notConnectedPeers": {

      [ always known regular nodes: boot nodes, other validators, etc ]

      "QmP3zYRhAxxw4fDf6Vq5agM8AZt1m2nKpPAEDmyEHPK5go": {
        "knownAddresses": [
          "/dns4/p2p.testnet-4.kusama.network/tcp/30100"
        ],
        "latestPingTime": null,
        "versionString": null
      },

      [ ........ ]

    },
    "peerset": {

      [ all known nodes shown here, only reported connected to validator's public nodes ]

      "QmPjNcWNZjNrjVFzkNYR6jH7HLqyU7j9piczUyNoxce1fD": {
        "connected": true,
        "reputation": 1114
      },
      "QmP3zYRhAxxw4fDf6Vq5agM8AZt1m2nKpPAEDmyEHPK5go": {
        "connected": false,
        "reputation": 0
      },

      [ ........ ]

      "reserved_only": true
    }
  }
}

```

### Validator's public nodes

They can reach and be reached both by the validator and by the network regular
nodes:
```
{

  [ ........ ]

  "result": {
    "connectedPeers": {

      [ secure validator, other secure validator's public nodes and regular nodes]

      "QmZSocEssLWHYCY6mqR99DcSFEpMb95fVeMsScrY8jqBm8": {
        "enabled": true,
        "endpoint": {
          "listening": {
            "listen_addr": "/ip4/10.0.0.2/tcp/30333",
            "send_back_addr": "/ip4/10.0.0.1/tcp/54932"
          }
        },
        "knownAddresses": [
          "/ip4/10.0.0.1/tcp/30333",
          "/ip4/10.0.1.18/tcp/30333",
          "/ip4/147.75.199.231/tcp/30333",
          "/ip4/10.0.1.152/tcp/30333"
        ],
        "latestPingTime": {
          "nanos": 335876602,
          "secs": 0
        },
        "open": true,
      },
      "QmP3zYRhAxxw4fDf6Vq5agM8AZt1m2nKpPAEDmyEHPK5go": {
        "enabled": true,
        "endpoint": {
          "listening": {
            "listen_addr": "/ip4/172.26.59.86/tcp/30333",
            "send_back_addr": "/ip4/191.232.49.216/tcp/3008"
          }
        },
        "knownAddresses": [
          "/dns4/p2p.testnet-4.kusama.network/tcp/30100",
          "/ip4/127.0.0.1/tcp/30100",
          "/ip4/10.244.0.10/tcp/30100",
          "/ip4/191.232.49.216/tcp/30100"
        ],
        "latestPingTime": {
          "nanos": 603313251,
          "secs": 0
        },
        "open": true,
      },

      [ ........ ]

    },
    "notConnectedPeers": {

      [ regular nodes ]

      "QmW45D6YLfctkSnsjyoqcSxw9qoiXUmAFGn5ea99L6SC7X": {
        "knownAddresses": [
          "/ip4/10.8.2.14/tcp/30101",
          "/ip4/127.0.0.1/tcp/30101",
          "/ip4/34.80.190.48/tcp/30101"
        ],
        "latestPingTime": {
          "nanos": 571989635,
          "secs": 0
        },
      },

      [ ........ ]

    },
    "peerset": {

      [ all known nodes reported as connected here ]

      "QmP3zYRhAxxw4fDf6Vq5agM8AZt1m2nKpPAEDmyEHPK5go": {
        "connected": true,
        "reputation": 1277
      },
      "QmZSocEssLWHYCY6mqR99DcSFEpMb95fVeMsScrY8jqBm8": {
        "connected": true,
        "reputation": -571
      },

      [ ........ ]

      "reserved_only": false
    }
  }
}
```

### Network regular nodes

They can reach and be reached by the validator's public nodes and by other regular
nodes, the don't have access to the validator.
```
{

  [ ........ ]

  "result": {
    "connectedPeers": {

      [ secure validator's public nodes and regular nodes ]

      "QmPjNcWNZjNrjVFzkNYR6jH7HLqyU7j9piczUyNoxce1fD": {
        "enabled": true,
        "endpoint": {
          "listening": {
            "listen_addr": "/ip4/10.44.1.11/tcp/30101",
            "send_back_addr": "/ip4/18.197.157.119/tcp/42962"
          }
        },
        "knownAddresses": [
          "/ip4/172.26.59.86/tcp/30333",
          "/ip4/127.0.0.1/tcp/30333",
          "/ip6/::1/tcp/30333",
          "/ip4/18.197.157.119/tcp/30333",
          "/ip4/10.0.0.2/tcp/30333",
          "/ip4/10.0.1.18/tcp/30333"
        ],
        "latestPingTime": {
          "nanos": 108101687,
          "secs": 0
        },
        "open": true,
      },
      "QmP3zYRhAxxw4fDf6Vq5agM8AZt1m2nKpPAEDmyEHPK5go": {
        "enabled": true,
        "endpoint": {
          "listening": {
            "listen_addr": "/ip4/10.44.1.11/tcp/30101",
            "send_back_addr": "/ip4/191.232.49.216/tcp/3010"
          }
        },
        "knownAddresses": [
          "/dns4/p2p.testnet-4.kusama.network/tcp/30100",
          "/ip4/127.0.0.1/tcp/30100",
          "/ip4/191.232.49.216/tcp/30100",
          "/ip4/10.244.0.10/tcp/30100"
        ],
        "latestPingTime": {
          "nanos": 717286051,
          "secs": 0
        },
        "open": true,
        "versionString": "parity-polkadot/v0.5.0-4e53ad1-x86_64-linux-gnu (unknown)"
      },

      [ ........ ]

    },
    "notConnectedPeers": {

      [ secure validator ]

      "QmZSocEssLWHYCY6mqR99DcSFEpMb95fVeMsScrY8jqBm8": {
        "knownAddresses": [
          "/ip4/10.0.0.1/tcp/30333",
          "/ip4/10.0.1.18/tcp/30333",
          "/ip4/10.0.1.152/tcp/30333",
          "/ip4/147.75.199.231/tcp/30333"
        ],
        "latestPingTime": {
          "nanos": 375552762,
          "secs": 0
        },
      }

      [ ........ ]

    },
    "peerset": {

      [ all known nodes shown here, reported connected to all, secure validator with 0 reputation ]

      "QmP3zYRhAxxw4fDf6Vq5agM8AZt1m2nKpPAEDmyEHPK5go": {
        "connected": true,
        "reputation": 1115
      },
      "QmPjNcWNZjNrjVFzkNYR6jH7HLqyU7j9piczUyNoxce1fD": {
        "connected": true,
        "reputation": 3500
      },
      "QmZSocEssLWHYCY6mqR99DcSFEpMb95fVeMsScrY8jqBm8": {
        "connected": true,
        "reputation": 0
      },

      [ ........ ]

      "reserved_only": false
    }
  }
}
```

---
description: Tessera deployment for high availability
---

# High availability

Tessera supports deploying more than one instance sharing the same database.

By placing the instances behind a load balancer, downtime can be limited during maintenance operations, achieving high availability (HA).

![Tessera-HA](../../images/tessera/Tessera-HA.png)

## Servers

Tessera exposes multiple interfaces for connectivity which can be configured in the
[`serverConfigs`](../../Reference/SampleConfiguration.md#serverconfigs) in the
[configuration file](Tessera.md).
To enable high availability for the node, set each interface to use the load balancer's address for its `serverAddress`,
and its own URL or IP for the `bindingAddress`, as shown in the following example.

!!! example "Server configuration"

    ```bash

    "serverConfigs":[
        {
            "app": "ENCLAVE",
            // Defines us using a remote enclave, leave out if using built-in enclave
            "serverAddress": "http://localhost:9081",
            "communicationType": "REST"
        },
        {
            "app": "ThirdParty",
            "serverAddress": "http://LOAD_BALANCER_URL:9080",
            // Specify a bind to an internal IP while advertising an external IP using serverAddress.
            "bindingAddress": "http://OWN_URL:9080",
            "communicationType": "REST",
            ...
        },
        {
            "app": "Q2T",
            "serverAddress": "http://LOAD_BALANCER_URL:9101",
            // Specify a bind to an internal IP while advertising an external IP using serverAddress.
            "bindingAddress": "http://OWN_URL:9101",
            "communicationType" : "REST",
            ...
        },
        {
            "app": "P2P",
            "serverAddress": "http://LOAD_BALANCER_URL:9000",
            // Specify a bind to an internal IP while advertising an external IP using serverAddress.
            "bindingAddress": "http://OWN_URL:9000",
            "communicationType": "REST",
            ...
        }
    ],
    ...
    ```

## Load balancer configuration

The load balancer must expose both client and node interfaces.

When configuring for high availability, configure the nodes in the Tessera cluster (Tessera A and Tessera B in the
previous diagram) with the same set of keys and advertise the load balancer address.

!!! example "Nginx configuration with two Tessera nodes"

    ```
     events { }

     http {
         upstream tessera_tp_9080 {
             server tessera_1:9080 max_fails=3 fail_timeout=5s;
             server tessera_2:9080 max_fails=3 fail_timeout=5s;
         }

         upstream tessera_q2t_9101 {
             server tessera_1:9101 max_fails=3 fail_timeout=5s;
             server tessera_2:9101 max_fails=3 fail_timeout=5s;
         }

          upstream tessera_p2p_9000 {
              server tessera_1:9000 max_fails=3 fail_timeout=5s;
              server tessera_2:9000 max_fails=3 fail_timeout=5s;
          }

         server {
             listen 9080;

             location / {
                 proxy_pass http://tessera_tp_9080;
                 health_check port=9000 uri=/upcheck;
             }
         }

         server {
             listen 9101;

             location / {
                 proxy_pass http://tessera_q2t_9101;
                 health_check port=9000 uri=/upcheck;
             }
         }

        server {
            listen 9000;

            location / {
                proxy_pass http://tessera_p2p_9000;
                health_check port=9000 uri=/upcheck;
            }
        }
     }
    ```

The configuration defines two upstreams, `tessera_tp_9080` and `tessera_q2t_9101`, both of which define [health checks](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-health-check/), `max_fails=3 fail_timeout=5s`.

The health checks help Nginx balance
traffic among upstream servers.

## Database

The last piece to configure in high availability is the [database](./Database.md).
Set the [`jdbc`](../../Reference/SampleConfiguration.md#jdbc) endpoint in the same configuration file.
We strongly recommend using an SQL database also configured for HA independently.
If using a cloud-based database, consider using [AWS RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html),
[Azure Postgresql](https://docs.microsoft.com/en-us/azure/postgresql/), or equivalent.

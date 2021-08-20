# Securing Chainlink Nodes with Letsencrypt

This guide was written to assist node operators with getting letsencrypt certificates into their docker environment for securing their nodes. 

This guide follows [this guide](https://pentacent.medium.com/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71) to obtain the certificates, and [this guide](https://docs.chain.link/docs/enabling-https-connections/) to install them. Refer to the links provided if you need context that may not be provided here. 

### Assumptions:
- You should already have some DNS entry pointing to the IP of your node. You can use services such as [DynDNS](https://account.dyn.com/) to accomplish this goal
- We will not be covering steps to get your node up and running. Please follow the [official guidance](https://docs.chain.link/docs/running-a-chainlink-node/) before using this documentation.
- Steps outlined here will be related to a RHEL-based environment. Please make sure you translate directory paths to match your environment. 

## Obtaining Certificates Using Docker

As mentioned above, we will be using [this guide](https://pentacent.medium.com/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71) as a foundation for obtaining certificates. 
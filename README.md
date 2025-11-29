# kube-node-dns

This is a small Kubernetes... thing. It publishes a node's globally-reachable IP addresses as DNS records via Route 53. (See IANA's definitions for "globally-reachable" [for IPv4](https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml) and [for IPv6](https://www.iana.org/assignments/iana-ipv6-special-registry/iana-ipv6-special-registry.xhtml))

Odds are, [ExternalDNS](https://github.com/kubernetes-sigs/external-dns/) will be a better fit for you, so you should go use that! This project exists to solve my specific problem, which ExternalDNS doesn't support (at time of writing, and which seemed like a fair amount of work to implement). Specifically, I wanted a solution to...

1. Publish DNS records for (a subset of) nodes in my cluster to AWS Route 53
2. ...using Route 53's [geoproximity routing policy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geoproximity.html)
3. ...where records for unhealthy nodes are removed ASAP
4. ...with IPs discovered by looking at the node's _network interfaces_.

At the time of writing, (1) and (2) are mutually exclusive in ExternalDNS, although it would likely be a relatively simple patch to get working. (3) also doesn't seem to work today from testing (ExternalDNS will remove _cordoned_ nodes, but not nodes that become unschedulable for other reasons), although it seems like something that could feasibly be supported.

(4) is a pretty unique constraint though, and that's what pushed me to write my own project.

## Why?

So, I have a cluster that has nodes both in my homelab and VPS instances across multiple providers (a hybrid-cloud + multi-cloud setup, I guess). To keep networking across the cluster manageable, I put all the nodes on a Wireguard-based mesh VPN (currently [NetBird](https://netbird.io/), but similar providers like [Tailscale](https://tailscale.com/) or [Netmaker](https://www.netmaker.io/) would've likely worked as well).

Then, each node runs [K3s](https://k3s.io/) using the built-in Flannel CNI pointing at the Wireguard interface. That means that the inferred internal IP for each node is the IP _within_ the VPN.

For handling requests, I use a `DaemonSet` running Traefik bound to `[::]:80` / `[::]:443` on the host, which means that each node can receive traffic _outside_ the VPN (by design).

From there, Traefik acts as a reverse proxy to underlying services on the cluster. Since I could easily add VPS nodes across many different regions to handle traffic, this acts like a tiny, homegrown CDN. All that's left is updating DNS records to point to each VPS node that's serving Traefik.

---

So the challenge is that each node's inferred internal IP is an address that's private to the VPN. That means I'd need to manually configure each node's external IP manually within K3s's `config.yaml` (or possibly write a script for use in the systemd service). But this complicates the process to set up a new node, which I wasn't happy with. (Plus, many VPS providers offer "floating IPs" for adding / removing IP addresses rapidly)

I knew that, with a little bit of custom code, I could have each node determine its own "external IPs" by looking at its network interfaces. From there, it'd just be a matter of creating / updating / deleting DNS records for each node within my DNS provider.

## How it works

`kube-node-dns` has two components:

- A `DaemonSet` that runs on each node that should be included in the DNS records. It regularly checks the node's globally reachable IP addresses across all interfaces (on a schedule, e.g. every 5 minutes), then updates the Kubernetes annotation `kube-node-dns.kyle.space/node-ips` on the node.
- A `Deployment` that watches for changes to nodes' annotations, then publishes DNS records to Route 53.

(Both components are included in a single OCI container image with different subcommands)

A few notes:

- **All existing `A` / `AAAA` records for each domain will be deleted**, so don't try to manage the same domain name with a different setup! (Other domains within the same hosted zone and other record types will not be affected).
- Set the annotation `kube-node-dns.kyle.space/aws-geoproximity-coordinates` on a node to optionally specify Route 53 geoproximity coordinates. If set, it should be formatted as `<lat>,<lon>` with 2 decimals of precision, e.g.: `12.34,56.78`
- Each domain includes records for all nodes matching the label selector. Nodes are assumed to serve traffic "homogenously". You should be able to use multiple deployments to use different nodes for different domains if desired (although I don't feel strongly about supporting this use-case either).
- A node's DNS record will be removed if it has no globally reachable IP addresses, if it's cordoned, or if the node is not ready (according to the `Ready` condition).
- Currently, only Route 53 is supported. (Maintaining a lot of different DNS providers isn't something I'm interested in, although I think it would be reasonable to use an abstraction over multiple DNS providers, or to publish DNS records via ExternalDNS CRD resources).
- Only Route 53 public hosted zones are currently supported.

## Deployment

I'm currently mostly using Terraform to manage my Kubernetes cluster, so I'll provide the HCL I'm using to run this stuff. I'll leave it as an exercise to the reader to convert to a YAML file, Helm chart, or your preferred configuration 🙂

```terraform
locals {
  # You might want to pin to a specific SHA-256 hash instead:
  # "ghcr.io/kylewlacy/kube-node-dns@sha256:..."
  kube_node_dns_tag = "ghcr.io/kylewlacy/kube-node-dns:local"
}

# DaemonSet to create the annotations for each node
resource "kubernetes_manifest" "kube-node-dns-annotate-ips" {
  manifest = {
    apiVersion = "apps/v1"
    kind       = "DaemonSet"
    metadata = {
      name      = "kube-node-dns-annotate-ips"
      namespace = "kube-system"
    }
    spec = {
      selector = {
        matchLabels = {
          name = "kube-node-dns"
        }
      }
      template = {
        metadata = {
          labels = {
            name = "kube-node-dns"
          }
        }
        spec = {
          serviceAccountName = kubernetes_service_account_v1.kube-node-dns.metadata[0].name

          # **Important!** This is needed to find the node's network interfaces:
          hostNetwork = true

          nodeSelector = {
            "serve-public" = "true"
          }

          containers = [{
            name  = "kube-node-dns"
            image = local.kube_node_dns_tag
            args  = ["annotate-node-ips", "5m"]
            env = [{
              name = "KUBE_NODE_NAME"
              valueFrom = {
                fieldRef = { fieldPath = "spec.nodeName" }
              }
            }]
          }]
        }
      }
    }
  }
}

# Deployment to publish DNS records
resource "kubernetes_manifest" "kube-node-dns-publish-dns" {
  manifest = {
    apiVersion = "apps/v1"
    kind       = "Deployment"
    metadata = {
      name      = "kube-node-dns-publish-dns"
      namespace = "kube-system"
    }
    spec = {
      replicas = 1
      selector = {
        matchLabels = {
          name = "kube-node-dns-publish-dns"
        }
      }
      template = {
        metadata = {
          labels = {
            name = "kube-node-dns-publish-dns"
          }
        }
        spec = {
          serviceAccountName = kubernetes_service_account_v1.kube-node-dns.metadata[0].name

          nodeSelector = {
            "kubernetes.io/arch" = "amd64"
          }

          containers = [{
            name  = "kube-node-dns-publish-dns"
            image = "ghcr.io/kylewlacy/kube-node-dns@${local.kube_node_dns_hash}"
            args = [
              "publish-dns",
              "--node-label-selector=serve-public=true",
              "--domain=example.com",
              "--domain=www.example.com",
            ]
            env = [
              {
                name = "AWS_ACCESS_KEY_ID"
                valueFrom = {
                  secretKeyRef = {
                    name = kubernetes_secret_v1.kube-node-dns-aws-credentials.metadata[0].name
                    key  = "aws_access_key_id"
                  }
                }
              },
              {
                name = "AWS_SECRET_ACCESS_KEY"
                valueFrom = {
                  secretKeyRef = {
                    name = kubernetes_secret_v1.kube-node-dns-aws-credentials.metadata[0].name
                    key  = "aws_secret_access_key"
                  }
                }
              },
              {
                name = "AWS_DEFAULT_REGION"
                valueFrom = {
                  secretKeyRef = {
                    name = kubernetes_secret_v1.kube-node-dns-aws-credentials.metadata[0].name
                    key  = "aws_default_region"
                  }
                }
              }
            ]
          }]
        }
      }
    }
  }
}

resource "kubernetes_service_account_v1" "kube-node-dns" {
  metadata {
    name      = "kube-node-dns-service-account"
    namespace = "kube-system"
  }
}

resource "kubernetes_cluster_role_v1" "kube-node-dns" {
  metadata {
    name = "kube-node-dns-role"
  }

  rule {
    api_groups = [""]
    resources  = ["nodes"]
    verbs      = ["get", "list", "watch", "update", "patch"]
  }
}

resource "kubernetes_cluster_role_binding_v1" "kube-node-dns" {
  metadata {
    name = "kube-node-dns-role-binding"
  }

  role_ref {
    api_group = "rbac.authorization.k8s.io"
    kind      = "ClusterRole"
    name      = kubernetes_cluster_role_v1.kube-node-dns.metadata[0].name
  }

  subject {
    kind      = "ServiceAccount"
    name      = kubernetes_service_account_v1.kube-node-dns.metadata[0].name
    namespace = kubernetes_service_account_v1.kube-node-dns.metadata[0].namespace
  }
}
```

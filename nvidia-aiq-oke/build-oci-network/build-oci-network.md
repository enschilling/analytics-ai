# Build the OCI Network

## Introduction

> **Instructor reference only:** This lab is not part of the learner workshop manifest. Learners use a pre-provisioned OKE cluster and do not create networking resources during the workshop.

In this lab, you create or validate the VCN pattern used by RAG and AIRA on OKE. The design uses three subnets: an API subnet, a private worker subnet for the GPU node, and a load balancer subnet. The worker subnet must reach OCI services through a Service Gateway and external runtime services through the LLfirewall DRG and Squid proxy.

The deployment does not require worker nodes to pull application images directly from the public internet. Images are mirrored into OCIR, and selected runtime calls use Squid at `10.255.255.18:3128`. This keeps the deployment repeatable in Cloud Shell and private-worker environments.

### Objectives

- Create a fresh VCN.
- Create or reference internet, service gateway, and DRG routing.
- Create API, worker, and load balancer route tables.
- Create security lists for API, worker, and load balancer traffic.
- Create API, worker, and load balancer subnets.

Estimated Time: 20 minutes

## Task 1: Create the VCN

1. Create the VCN.

    ```bash
    export VCN_OCID=$(oci network vcn create \
      --compartment-id "$COMPARTMENT_OCID" \
      --display-name "${RESOURCE_PREFIX}-vcn" \
      --dns-label "aiqa100" \
      --cidr-block "$VCN_CIDR" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)

    echo "$VCN_OCID"
    ```

2. Add tags that make cleanup easier.

    ```bash
    oci network vcn update \
      --vcn-id "$VCN_OCID" \
      --freeform-tags '{"created_by":"livelabs-aiq","managed_by":"lab"}' \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION"
    ```

## Task 2: Create Gateways

1. Create the internet gateway for the public load balancer subnet.

    ```bash
    export IGW_OCID=$(oci network internet-gateway create \
      --compartment-id "$COMPARTMENT_OCID" \
      --vcn-id "$VCN_OCID" \
      --display-name "${RESOURCE_PREFIX}-igw" \
      --is-enabled true \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)
    ```

2. If your lab uses the LLfirewall proxy pattern, set the existing firewall DRG and proxy subnet values.

    The attendee VCN sends worker egress and proxy-subnet traffic to the LLfirewall DRG. The LLfirewall VCN then forwards traffic through the network firewall, Squid proxy, and NAT path.

    ```bash
    export FIREWALL_DRG_OCID="<llfirewall_drg_ocid>"
    export FIREWALL_PROXY_CIDR="10.255.255.16/28"
    export FIREWALL_PROXY_URL="http://10.255.255.18:3128"
    ```

3. Find the regional Oracle Services Network service OCID.

    ```bash
    export OSN_SERVICE_OCID=$(oci network service list \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data[?contains("name", `All`) && contains("name", `Services`) == `true`].id | [0]' \
      --raw-output)

    echo "$OSN_SERVICE_OCID"
    ```

4. Create the service gateway.

    ```bash
    export SGW_OCID=$(oci network service-gateway create \
      --compartment-id "$COMPARTMENT_OCID" \
      --vcn-id "$VCN_OCID" \
      --display-name "${RESOURCE_PREFIX}-sgw" \
      --services "[{\"serviceId\":\"$OSN_SERVICE_OCID\"}]" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)
    ```

## Task 3: Create Route Tables

1. Create the public route table for the API or load balancer subnet when the lab needs Cloud Shell access to a public Kubernetes API endpoint or a public UI endpoint.

    ```bash
    export PUBLIC_RT_OCID=$(oci network route-table create \
      --compartment-id "$COMPARTMENT_OCID" \
      --vcn-id "$VCN_OCID" \
      --display-name "${RESOURCE_PREFIX}-public-rt" \
      --route-rules "[{\"destination\":\"0.0.0.0/0\",\"destinationType\":\"CIDR_BLOCK\",\"networkEntityId\":\"$IGW_OCID\"}]" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)
    ```

2. Create the private worker route table.

    The Service Gateway route lets the cluster reach regional OCI APIs, including Block Volume APIs used by the OCI CSI driver. The DRG routes send external egress and the proxy subnet toward LLfirewall. Do not add a NAT gateway to the attendee VCN when testing the strict proxy path.

    ```bash
    export PRIVATE_RT_OCID=$(oci network route-table create \
      --compartment-id "$COMPARTMENT_OCID" \
      --vcn-id "$VCN_OCID" \
      --display-name "${RESOURCE_PREFIX}-private-rt" \
      --route-rules "[{\"destination\":\"0.0.0.0/0\",\"destinationType\":\"CIDR_BLOCK\",\"networkEntityId\":\"$FIREWALL_DRG_OCID\"},{\"destination\":\"$FIREWALL_PROXY_CIDR\",\"destinationType\":\"CIDR_BLOCK\",\"networkEntityId\":\"$FIREWALL_DRG_OCID\"},{\"destination\":\"all-${REGION}-services-in-oracle-services-network\",\"destinationType\":\"SERVICE_CIDR_BLOCK\",\"networkEntityId\":\"$SGW_OCID\"}]" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)
    ```

3. Attach the attendee VCN to the existing LLfirewall DRG.

    Use the DRG route table assignment that your firewall hub design uses for spoke ingress. In the validated environment, the LLfirewall VCN attachment imports VCN CIDRs into the autogenerated DRG route table, and each attendee VCN uses a unique CIDR to avoid route conflicts.

    ```bash
    export DRG_ATTACHMENT_OCID=$(oci network drg-attachment create \
      --display-name "${RESOURCE_PREFIX}-vcn-attachment" \
      --drg-id "$FIREWALL_DRG_OCID" \
      --vcn-id "$VCN_OCID" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)

    echo "$DRG_ATTACHMENT_OCID"
    ```

## Task 4: Create Security Lists

1. Create the API security list.

    ```bash
    export API_SL_OCID=$(oci network security-list create \
      --compartment-id "$COMPARTMENT_OCID" \
      --vcn-id "$VCN_OCID" \
      --display-name "${RESOURCE_PREFIX}-api-sl" \
      --egress-security-rules '[{"destination":"0.0.0.0/0","protocol":"all","isStateless":false}]' \
      --ingress-security-rules "[{\"source\":\"$WORKER_SUBNET_CIDR\",\"protocol\":\"6\",\"isStateless\":false,\"tcpOptions\":{\"destinationPortRange\":{\"min\":6443,\"max\":6443}}}]" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)
    ```

2. Create the worker security list.

    ```bash
    export WORKER_SL_OCID=$(oci network security-list create \
      --compartment-id "$COMPARTMENT_OCID" \
      --vcn-id "$VCN_OCID" \
      --display-name "${RESOURCE_PREFIX}-worker-sl" \
      --egress-security-rules '[{"destination":"0.0.0.0/0","protocol":"all","isStateless":false}]' \
      --ingress-security-rules "[{\"source\":\"$VCN_CIDR\",\"protocol\":\"all\",\"isStateless\":false},{\"source\":\"10.244.0.0/16\",\"protocol\":\"all\",\"isStateless\":false}]" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)
    ```

3. Create the load balancer security list. This opens HTTP for the AIQ frontend.

    ```bash
    export LB_SL_OCID=$(oci network security-list create \
      --compartment-id "$COMPARTMENT_OCID" \
      --vcn-id "$VCN_OCID" \
      --display-name "${RESOURCE_PREFIX}-lb-sl" \
      --egress-security-rules "[{\"destination\":\"$WORKER_SUBNET_CIDR\",\"protocol\":\"6\",\"isStateless\":false,\"tcpOptions\":{\"destinationPortRange\":{\"min\":3000,\"max\":3000}}}]" \
      --ingress-security-rules '[{"source":"0.0.0.0/0","protocol":"6","isStateless":false,"tcpOptions":{"destinationPortRange":{"min":80,"max":80}}}]' \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)
    ```

## Task 5: Create Subnets

1. Create the private API subnet.

    ```bash
    export API_SUBNET_OCID=$(oci network subnet create \
      --compartment-id "$COMPARTMENT_OCID" \
      --vcn-id "$VCN_OCID" \
      --display-name "${RESOURCE_PREFIX}-api-subnet" \
      --dns-label "api" \
      --cidr-block "$API_SUBNET_CIDR" \
      --route-table-id "$PRIVATE_RT_OCID" \
      --security-list-ids "[\"$API_SL_OCID\"]" \
      --prohibit-public-ip-on-vnic true \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)
    ```

2. Create the private worker subnet.

    ```bash
    export WORKER_SUBNET_OCID=$(oci network subnet create \
      --compartment-id "$COMPARTMENT_OCID" \
      --vcn-id "$VCN_OCID" \
      --display-name "${RESOURCE_PREFIX}-worker-subnet" \
      --dns-label "workers" \
      --cidr-block "$WORKER_SUBNET_CIDR" \
      --route-table-id "$PRIVATE_RT_OCID" \
      --security-list-ids "[\"$WORKER_SL_OCID\"]" \
      --prohibit-public-ip-on-vnic true \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)
    ```

3. Create the public load balancer subnet.

    ```bash
    export LB_SUBNET_OCID=$(oci network subnet create \
      --compartment-id "$COMPARTMENT_OCID" \
      --vcn-id "$VCN_OCID" \
      --display-name "${RESOURCE_PREFIX}-lb-subnet" \
      --dns-label "lb" \
      --cidr-block "$LB_SUBNET_CIDR" \
      --route-table-id "$PUBLIC_RT_OCID" \
      --security-list-ids "[\"$LB_SL_OCID\"]" \
      --prohibit-public-ip-on-vnic false \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data.id' \
      --raw-output)
    ```

4. Confirm all subnets are available.

    ```bash
    oci network subnet list \
      --compartment-id "$COMPARTMENT_OCID" \
      --vcn-id "$VCN_OCID" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data[].{name:"display-name",cidr:"cidr-block",state:"lifecycle-state"}' \
      --output table
    ```

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 11, 2026

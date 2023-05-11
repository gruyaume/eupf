# How to install and run eUPF
The easyest way to install eUPF is to use helm charts for one of the supported opensource 5G core projects in your own kubernetes cluster.
Alternatively, eUPF could be deployed in docker-compose(free5gc only at the moment).

## Kubenetes environment

- Kubernetes cluster with Calico and Multus CNI
- [helm](https://helm.sh/docs/intro/install/) installed
<!-- - deployed 5g core (open5gs or free5gc) -->

in our environments, we use one node K8s cluster deployed by means of [kubespray](https://github.com/kubernetes-sigs/kubespray). You can see configuration examples in this [repo](https://github.com/edgecomllc/ansible)

We have prepared templates to deploy with two opensource environments: **open5gs** and **free5gc**, for you to choose. 

[UERANSIM](https://github.com/aligungr/UERANSIM) project is used for emulating radio endpoint, so you'll be able to check end-to-end connectivity. 

## How to deploy eUPF with open5gs core:
<details><summary>Instructions</summary>
<p>

### To deploy:

* [install helm](https://helm.sh/docs/intro/install/) if it's not
* add openverso helm repo

   ```
   helm repo add openverso https://gradiant.github.io/openverso-charts/
   helm repo update
   ```

* install eUPF chart

   ```powershell
   helm upgrade --install \
       edgecomllc-eupf .deploy/helm/universal-chart \
       --values docs/examples/open5gs/eupf.yaml \
       -n open5gs \
       --wait --timeout 100s --create-namespace
   ```
   📝Here we use subnet `10.100.111.0/24` for n6 interface as exit to the world, so make sure it's not occupied at your node host.

* install open5gs chart

   ```powershell
   helm upgrade --install \
       open5gs openverso/open5gs \
       --values docs/examples/open5gs/open5gs.yaml \
       -n open5gs \
       --version 2.0.9 \
       --wait --timeout 100s --create-namespace
   ```

* install ueransim chart

   ```powershell
   helm upgrade --install \
       ueransim openverso/ueransim-gnb \
       --values docs/examples/open5gs/ueransim-gnb.yaml \
       -n open5gs \
       --version 0.2.5 \
       --wait --timeout 100s --create-namespace
   ```

### To undeploy everything:

```
helm delete open5gs ueransim edgecomllc-eupf -n open5gs
```
📝 Pod's interconnection. openverso-charts uses default interfaces of your kubernetes cluster. It is Calico CNI interfaces in our environment, type ipvlan. And it uses k8s services names to resolve endpoints.
The only added is ptp type interface `n6` as a door to the outer world for our eUPF. 

For more details refer to openverso-charts [Open5gs and UERANSIM](https://gradiant.github.io/openverso-charts/open5gs-ueransim-gnb.html)

</p>
</details> 

## How to deploy eUPF with free5gc core
<details><summary>Instruction</summary>
<p>
	
### Prepare nodes

You should compile and install gtp5g kernel module on every worker node:

```powershell
apt-get update; apt-get install git build-essential -y; \
cd /tmp; \
git clone --depth 1 --branch v0.7.3 https://github.com/free5gc/gtp5g.git; \
cd gtp5g/; \
make && make install
```

check that the module is loaded:

`lsmod | grep ^gtp5g`

### Deploy in Kubernetes cluster
	
Deployment configuration is derived from towards5gs-helm project [Setup free5gc](https://github.com/Orange-OpenSource/towards5gs-helm/blob/main/docs/demo/Setup-free5gc-on-multiple-clusters-and-test-with-UERANSIM.md)
![Architecture](pictures/Setup-free5gc-on-multiple-clusters-and-test-with-UERANSIM-Architecture.png)
	

0. [install helm](https://helm.sh/docs/intro/install/) if it's not
0. add towards5gs helm repo

	```powershell
	helm repo add towards5gs https://raw.githubusercontent.com/Orange-OpenSource/towards5gs-helm/main/repo/
	helm repo update
	```

0. install eUPF chart

	```powershell
	helm upgrade --install \
		edgecomllc-eupf .deploy/helm/universal-chart \
		--values docs/examples/free5gc/eupf.yaml \
		-n free5gc \
		--wait --timeout 100s --create-namespace
	```
   📝Here we use subnet `10.100.100.0/24` for n6 interface as exit to the world, so make sure it's not occupied at your node host.

0. install free5gc chart

	```powershell
	helm upgrade --install \
		free5gc towards5gs/free5gc \
		--values docs/examples/free5gc/free5gc-single.yaml \
		-n free5gc \
		--version 1.1.6 \
		--wait --timeout 100s --create-namespace
	```

0. create subscriber in free5gc via WebUI

   redirect port from webui pod to localhost

   ```powershell
   kubectl port-forward service/webui-service 5000:5000 -n free5gc
   ```

   open http://127.0.0.1:5000 in your browser (for auth use user "admin" with password "free5gc"), go to menu "subscribers", click "new subscriber", leave all values as is, press "submit"

   close port forward with `Ctrl + C`

0. install ueransim chart

	```powershell
	helm upgrade --install \
		ueransim towards5gs/ueransim \
		--values docs/examples/free5gc/ueransim.yaml \
		-n free5gc \
		--version 2.0.17 \
		--wait --timeout 100s --create-namespace
	```

### To undeploy everything

```
helm delete free5gc ueransim edgecomllc-eupf -n free5gc
```
📝 Pod's interconnection. towards5gs-helm uses separate subnets with ipvlan type interfaces with internal addressing.
The only added is ptp type interface `n6` as a door to the outer world for our eUPF. 

</p>
</details> 
</p>

## How to deploy eUPF with free5gc core (docker-compose)

<details><summary>Instruction to deploy as docker-compose</summary>
<p>

### Deploy as docker-compose
Prerequisites

- Prepared [GTP5G kernel module](https://github.com/free5gc/gtp5g): needed to run the UPF
- [Docker Engine](https://docs.docker.com/engine/install): needed to run the Free5GC containers
- [Docker Compose v2](https://docs.docker.com/compose/install): needed to bootstrap the free5GC stack

Actually we use vanilla free5gc-docker-compose with some overrides. So you can clone free5gc docker-compose and test free5gc upf, then add edgecom override files and feel the differences of eupf.

0. pull repository: `git clone https://github.com/edgecomllc/free5gc-compose.git`
0. Run containers based on docker hub images:
   ```bash
   cd free5gc-compose
   docker compose pull
   docker compose up -d 
   ```
### To undeploy everything
   ```
   docker compose rm
   ```
</p>
</details> 


## Option NAT at the node
eUPF pod outbound connection is pure routed at the node. There is no address translation inside pod, so we avoid such lack of throughtput.

If you need NAT (Network Address Translation, or Masqerading) at your node to access Internet, the easiest way is to use standart daemonset [IP Masquerade Agent](https://kubernetes.io/docs/tasks/administer-cluster/ip-masq-agent/):
```powershell
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/ip-masq-agent/master/ip-masq-agent.yaml
```
   > The below entries show the default set of rules that are applied by the ip-masq-agent:
    ` iptables -t nat -L IP-MASQ-AGENT`     

---

## Test scenarios

## case 0

<b>description:</b>

UE can send packet to internet and get response

<b>actions:</b>

1. run shell in pod

   for open5gs:
   ```powershell
   export NS_NAME=open5gs
   export UE_POD_NAME=$(kubectl get pods -l "app.kubernetes.io/name=ueransim-gnb,app.kubernetes.io/component=ues" --output=jsonpath="{.items..metadata.name}" -n ${NS_NAME})
   kubectl exec -n ${NS_NAME} --stdin --tty ${UE_POD_NAME} -- /bin/bash
   ```

   for free5gc:

   ```powershell
   export NS_NAME=free5gc
   export UE_POD_NAME=$(kubectl get pods -l "app=ueransim,component=ue" --output=jsonpath="{.items..metadata.name}" -n ${NS_NAME})
   kubectl exec -n ${NS_NAME} --stdin --tty ${UE_POD_NAME} -- /bin/bash
   ```

1. run command from UE pod's shell. 

   `$ ping -I uesimtun0 google.com`


   <b>expected result:</b>

   ping command successful

# Information for troubleshooting

To see debug log from eBPF programs, at the **node** console start command:
`sudo cat /sys/kernel/debug/tracing/trace_pipe`

Then switch to UE pod's shell. Sending a single packet `ping -I uesimtun0 -c1 1.1.1.1` with successfull responce, normally you will see such debug output:
```ruby
sergo@edgecom:~$ sudo cat /sys/kernel/debug/tracing/trace_pipe

          nr-gnb-4117277 [003] d.s11 266111.395788: bpf_trace_printk: upf: gtp-u received
          nr-gnb-4117277 [003] d.s11 266111.395819: bpf_trace_printk: upf: gtp pdu [ 10.100.50.236 -> 10.100.50.233 ]
          nr-gnb-4117277 [003] d.s11 266111.395825: bpf_trace_printk: upf: uplink session for teid:1 far:1 headrm:0
          nr-gnb-4117277 [003] d.s11 266111.395828: bpf_trace_printk: upf: far:1 action:2 outer_header_creation:0
          nr-gnb-4117277 [003] d.s11 266111.395831: bpf_trace_printk: upf: qer:1 gate_status:0 mbr:200000000
          nr-gnb-4117277 [003] d.s11 266111.395857: bpf_trace_printk: upf: bpf_fib_lookup 10.1.0.1 -> 1.1.1.1: nexthop: 10.100.100.1
          nr-gnb-4117277 [003] d.s11 266111.395861: bpf_trace_printk: upf: bpf_redirect: if=6 18446669071770913972 -> 18446669071770913978
          <idle>-0       [007] d.s.1 266111.396975: bpf_trace_printk: upf: downlink session for ip:10.1.0.1  far:2 action:2
          <idle>-0       [007] dNs.1 266111.396983: bpf_trace_printk: upf: qer:0 gate_status:0 mbr:0
          <idle>-0       [007] dNs.1 266111.396985: bpf_trace_printk: upf: use mapping 10.1.0.1 -> TEID:1
          <idle>-0       [007] dNs.1 266111.396987: bpf_trace_printk: upf: send gtp pdu 10.100.50.233 -> 10.100.50.236
          <idle>-0       [007] dNs.1 266111.396996: bpf_trace_printk: upf: bpf_fib_lookup 10.100.50.233 -> 10.100.50.236: nexthop: 10.100.50.236
          <idle>-0       [007] dNs.1 266111.396998: bpf_trace_printk: upf: bpf_redirect: if=4 18446669071771765924 -> 18446669071771765930
```

## Components logs then successfully connected:
<details><summary>eUPF successfull connections log output (stdout)</summary>
<p>

```ruby
2023/04/17 16:09:39 map[api_address::8080 interface_name:n3 metrics_address::9090 pfcp_address::8805 pfcp_node_id:10.100.50.241 xdp_attach_mode:generic]
2023/04/17 16:09:39 {n3 generic :8080 :8805 10.100.50.241 :9090}
2023/04/17 16:09:40 Attached XDP program to iface "n3" (index 4)
2023/04/17 16:09:40 Press Ctrl-C to exit and remove the program
2023/04/17 16:09:40 Start PFCP connection: :8805
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:    export GIN_MODE=release
 - using code:    gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /upf_pipeline             --> main.CreateApiServer.func1 (3 handlers)
[GIN-debug] GET    /qer_map                  --> main.CreateApiServer.func2 (3 handlers)
[GIN-debug] GET    /pfcp_associations        --> main.CreateApiServer.func3 (3 handlers)
[GIN-debug] GET    /config                   --> main.CreateApiServer.func4 (3 handlers)
[GIN-debug] GET    /xdp_stats                --> main.CreateApiServer.func5 (3 handlers)
[GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.
Please check https://pkg.go.dev/github.com/gin-gonic/gin#readme-don-t-trust-all-proxies for details.
[GIN-debug] Listening and serving HTTP on :8080
2023/04/17 16:11:13 Received 30 bytes from 10.100.50.244:8805
2023/04/17 16:11:13 Handling PFCP message from 10.100.50.244:8805
2023/04/17 16:11:13 Got Association Setup Request from: 10.100.50.244:8805. 
2023/04/17 16:11:13 
Association Setup Request:
  Node ID: 10.100.50.244
  Recovery Time: 2023-04-17 16:11:13 +0000 UTC
2023/04/17 16:11:13 Saving new association: {ID:10.100.50.244 Addr:10.100.50.244:8805 NextSessionID:1 Sessions:map[]}
2023/04/17 16:11:50 Received 287 bytes from 10.100.50.244:8805
2023/04/17 16:11:50 Handling PFCP message from 10.100.50.244:8805
2023/04/17 16:11:50 Got Session Establishment Request from: 10.100.50.244:8805.
2023/04/17 16:11:50 
Session Establishment Request:
  CreatePDR ID: 1 
    Outer Header Removal: 0 
    FAR ID: 1 
    Source Interface: 0 
    TEID: 1 
    Ipv4: 10.100.50.233 
    Ipv6: <nil> 
  CreatePDR ID: 2 
    FAR ID: 2 
    Source Interface: 2 
    UE IPv4 Address: 10.1.0.1 
  CreateFAR ID: 1 
    Apply Action: [2] 
    Forwarding Parameters:
      Network Instance: internet 
  CreateFAR ID: 2 
    Apply Action: [2] 
    Forwarding Parameters:
  CreateQER ID: 1 
    Gate Status DL: 0 
    Gate Status UL: 0 
    Max Bitrate DL: 100000 
    Max Bitrate UL: 200000 
    QFI: 9 

2023/04/17 16:11:50 
Session Establishment Request:
  CreatePDR ID: 1 
    Outer Header Removal: 0 
    FAR ID: 1 
    Source Interface: 0 
    TEID: 1 
    Ipv4: 10.100.50.233 
    Ipv6: <nil> 
  CreatePDR ID: 2 
    FAR ID: 2 
    Source Interface: 2 
    UE IPv4 Address: 10.1.0.1 
  CreateFAR ID: 1 
    Apply Action: [2] 
    Forwarding Parameters:
      Network Instance: internet 
  CreateFAR ID: 2 
    Apply Action: [2] 
    Forwarding Parameters:
  CreateQER ID: 1 
    Gate Status DL: 0 
    Gate Status UL: 0 
    Max Bitrate DL: 100000 
    Max Bitrate UL: 200000 
    QFI: 9 
2023/04/17 16:11:50 WARN: No OuterHeaderCreation
2023/04/17 16:11:50 Saving FAR info to session: 1, {Action:2 OuterHeaderCreation:0 Teid:0 RemoteIP:0 LocalIP:0}
2023/04/17 16:11:50 EBPF: Put FAR: i=1, farInfo={Action:2 OuterHeaderCreation:0 Teid:0 RemoteIP:0 LocalIP:0}
2023/04/17 16:11:50 Saving FAR info to session: 2, {Action:2 OuterHeaderCreation:0 Teid:0 RemoteIP:0 LocalIP:0}
2023/04/17 16:11:50 EBPF: Put FAR: i=2, farInfo={Action:2 OuterHeaderCreation:0 Teid:0 RemoteIP:0 LocalIP:0}
2023/04/17 16:11:50 Saving uplink PDR info to session: 1, {PdrInfo:{OuterHeaderRemoval:0 FarId:1} Teid:1 Ipv4:<nil>}
2023/04/17 16:11:50 EBPF: Put PDR Uplink: teid=1, pdrInfo={OuterHeaderRemoval:0 FarId:1}
2023/04/17 16:11:50 Saving downlink PDR info to session: 2, {PdrInfo:{OuterHeaderRemoval:0 FarId:2} Teid:0 Ipv4:10.1.0.1}
2023/04/17 16:11:50 EBPF: Put PDR Downlink: ipv4=10.1.0.1, pdrInfo={OuterHeaderRemoval:0 FarId:2}
2023/04/17 16:11:50 Saving QER info to session: 1, {GateStatusUL:0 GateStatusDL:0 Qfi:9 MaxBitrateUL:200000 MaxBitrateDL:100000}
2023/04/17 16:11:50 Creating QER ID: 1, QER Info: {GateStatusUL:0 GateStatusDL:0 Qfi:9 MaxBitrateUL:200000 MaxBitrateDL:100000}
2023/04/17 16:11:50 EBPF: Put QER: i=1, qerInfo={GateStatusUL:0 GateStatusDL:0 Qfi:9 MaxBitrateUL:200000 MaxBitrateDL:100000}
2023/04/17 16:11:50 Received 148 bytes from 10.100.50.244:8805
2023/04/17 16:11:50 Handling PFCP message from 10.100.50.244:8805
2023/04/17 16:11:50 Got Session Modification Request from: 10.100.50.244:8805. 
2023/04/17 16:11:50 Finding association for 10.100.50.244:8805
2023/04/17 16:11:50 Finding session 2
2023/04/17 16:11:50 
Session Modification Request:
  UpdatePDR ID: 2 
    FAR ID: 2 
    Source Interface: 2 
    UE IPv4 Address: 10.1.0.1 
  UpdateFAR ID: 2 
    Apply Action: [2] 
    Forwarding Parameters:
2023/04/17 16:11:50 Updating FAR info: 2, {Action:2 OuterHeaderCreation:1 Teid:2 RemoteIP:3962725386 LocalIP:0}
2023/04/17 16:11:50 EBPF: Update FAR: i=2, farInfo={Action:2 OuterHeaderCreation:1 Teid:2 RemoteIP:3962725386 LocalIP:0}
2023/04/17 16:11:50 Updating downlink PDR: 2, {PdrInfo:{OuterHeaderRemoval:0 FarId:2} Teid:0 Ipv4:10.1.0.1}
2023/04/17 16:11:50 EBPF: Update PDR Downlink: ipv4=10.1.0.1, pdrInfo={OuterHeaderRemoval:0 FarId:2}
Stream closed EOF for free5gc/edgecomllc-eupf-universal-chart-d4b54d4b7-t2hr6 (app)
```

</p>
</details> 

<details><summary>SMF free5gc successfull connection log (stdout)</summary>
<p>

```ruby
2023-04-17T16:11:13Z [INFO][SMF][CFG] SMF config version [1.0.2]
2023-04-17T16:11:13Z [INFO][SMF][CFG] UE-Routing config version [1.0.1]
2023-04-17T16:11:13Z [INFO][SMF][Init] SMF Log level is set to [info] level
2023-04-17T16:11:13Z [INFO][LIB][NAS] set log level : info
2023-04-17T16:11:13Z [INFO][LIB][NAS] set report call : false
2023-04-17T16:11:13Z [INFO][LIB][NGAP] set log level : info
2023-04-17T16:11:13Z [INFO][LIB][NGAP] set report call : false
2023-04-17T16:11:13Z [INFO][LIB][Aper] set log level : info
2023-04-17T16:11:13Z [INFO][LIB][Aper] set report call : false
2023-04-17T16:11:13Z [INFO][LIB][PFCP] set log level : info
2023-04-17T16:11:13Z [INFO][LIB][PFCP] set report call : false
2023-04-17T16:11:13Z [INFO][SMF][App] smf
2023-04-17T16:11:13Z [INFO][SMF][App] SMF version:  
    free5GC version: v3.2.1
    build time:      2023-03-13T18:13:22Z
    commit hash:     de70bf6c
    commit time:     2022-06-28T04:52:40Z
    go version:      go1.14.4 linux/amd64
2023-04-17T16:11:13Z [INFO][SMF][CTX] smfconfig Info: Version[1.0.2] Description[SMF initial local configuration]
2023-04-17T16:11:13Z [INFO][SMF][CTX] Endpoints: [10.100.50.233]
2023-04-17T16:11:13Z [INFO][SMF][Init] Server started
2023-04-17T16:11:13Z [INFO][SMF][Init] SMF Registration to NRF {4892acc3-b6b3-418f-b791-f2b300277fe9 SMF REGISTERED 0 0xc00024f480 0xc00024f4c0 [] []   [free5gc-free5gc-smf-service] [] <nil> [] [] <nil> 0 0 0 area1 <nil> <nil> <nil> <nil> 0xc00002ee40 <nil> <nil> <nil> <nil> <nil> map[] <nil> false 0xc00024f300 false false []}
2023-04-17T16:11:13Z [INFO][SMF][PFCP] Listen on 10.100.50.244:8805
2023-04-17T16:11:13Z [INFO][SMF][App] Sending PFCP Association Request to UPF[10.100.50.241]
2023-04-17T16:11:13Z [INFO][LIB][PFCP] Remove Request Transaction [1]
2023-04-17T16:11:13Z [INFO][SMF][App] Received PFCP Association Setup Accepted Response from UPF[10.100.50.241]
2023-04-17T16:11:50Z [INFO][SMF][PduSess] Receive Create SM Context Request
2023-04-17T16:11:50Z [INFO][SMF][PduSess] In HandlePDUSessionSMContextCreate
2023-04-17T16:11:50Z [INFO][SMF][PduSess] Send NF Discovery Serving UDM Successfully
2023-04-17T16:11:50Z [INFO][SMF][CTX] Allocated UE IP address: 10.1.0.1
2023-04-17T16:11:50Z [INFO][SMF][CTX] Selected UPF: UPF
2023-04-17T16:11:50Z [INFO][SMF][PduSess] UE[imsi-208930000000003] PDUSessionID[1] IP[10.1.0.1]
2023-04-17T16:11:50Z [INFO][SMF][GSM] In HandlePDUSessionEstablishmentRequest
2023-04-17T16:11:50Z [INFO][NAS][Convert] ProtocolOrContainerList:  [0xc0004aaa80 0xc0004aaac0]
2023-04-17T16:11:50Z [INFO][SMF][GSM] Protocol Configuration Options
2023-04-17T16:11:50Z [INFO][SMF][GSM] &{[0xc0004aaa80 0xc0004aaac0]}
2023-04-17T16:11:50Z [INFO][SMF][GSM] Didn't Implement container type IPAddressAllocationViaNASSignallingUL
2023-04-17T16:11:50Z [INFO][SMF][PduSess] PCF Selection for SMContext SUPI[imsi-208930000000003] PDUSessionID[1]
2023-04-17T16:11:50Z [INFO][SMF][PduSess] SUPI[imsi-208930000000003] has no pre-config route
2023-04-17T16:11:50Z [INFO][SMF][Consumer] SendNFDiscoveryServingAMF ok
2023-04-17T16:11:50Z [INFO][SMF][PduSess] Sending PFCP Session Establishment Request
2023-04-17T16:11:50Z [INFO][SMF][GIN] | 201 |   10.233.78.130 | POST    | /nsmf-pdusession/v1/sm-contexts |
2023-04-17T16:11:50Z [INFO][LIB][PFCP] Remove Request Transaction [2]
2023-04-17T16:11:50Z [INFO][SMF][PduSess] Received PFCP Session Establishment Accepted Response
2023-04-17T16:11:50Z [INFO][SMF][PduSess] Receive Update SM Context Request
2023-04-17T16:11:50Z [INFO][SMF][PduSess] In HandlePDUSessionSMContextUpdate
2023-04-17T16:11:50Z [INFO][SMF][PduSess] Sending PFCP Session Modification Request to AN UPF
2023-04-17T16:11:50Z [INFO][LIB][PFCP] Remove Request Transaction [3]
2023-04-17T16:11:50Z [INFO][SMF][PduSess] Received PFCP Session Modification Accepted Response from AN UPF
2023-04-17T16:11:50Z [INFO][SMF][GIN] | 200 |   10.233.78.130 | POST    | /nsmf-pdusession/v1/sm-contexts/urn:uuid:6dffeab5-0861-490d-8cb0-f5528e8e21a9/modify |
```

</p>
</details> 

<details><summary>UERANSIM UE successfully connection log output:</summary>
<p>

```ruby
UERANSIM v3.2.6
[2023-04-25 14:36:41.461] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2023-04-25 14:36:41.462] [rrc] [warning] Acceptable cell selection failed, no cell is in coverage
[2023-04-25 14:36:41.462] [rrc] [error] Cell selection failure, no suitable or acceptable cell found
[2023-04-25 14:36:42.464] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2023-04-25 14:36:43.663] [nas] [error] PLMN selection failure, no cells in coverage
[2023-04-25 14:36:45.865] [nas] [error] PLMN selection failure, no cells in coverage
[2023-04-25 14:36:46.966] [nas] [info] UE switches to state [MM-DEREGISTERED/NO-CELL-AVAILABLE]
[2023-04-25 14:36:47.939] [nas] [info] Selected plmn[208/93]
[2023-04-25 14:36:47.939] [rrc] [info] Selected cell plmn[208/93] tac[1] category[SUITABLE]
[2023-04-25 14:36:47.940] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2023-04-25 14:36:47.940] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2023-04-25 14:36:47.940] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2023-04-25 14:36:47.940] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-04-25 14:36:47.940] [nas] [debug] Sending Initial Registration
[2023-04-25 14:36:47.940] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2023-04-25 14:36:47.940] [rrc] [debug] Sending RRC Setup Request
[2023-04-25 14:36:47.941] [rrc] [info] RRC connection established
[2023-04-25 14:36:47.941] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2023-04-25 14:36:47.941] [nas] [info] UE switches to state [CM-CONNECTED]
[2023-04-25 14:36:47.993] [nas] [debug] Authentication Request received
[2023-04-25 14:36:47.994] [nas] [debug] Sending Authentication Failure due to SQN out of range
[2023-04-25 14:36:48.020] [nas] [debug] Authentication Request received
[2023-04-25 14:36:48.048] [nas] [debug] Security Mode Command received
[2023-04-25 14:36:48.048] [nas] [debug] Selected integrity[2] ciphering[0]
[2023-04-25 14:36:48.137] [nas] [debug] Registration accept received
[2023-04-25 14:36:48.137] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2023-04-25 14:36:48.137] [nas] [debug] Sending Registration Complete
[2023-04-25 14:36:48.137] [nas] [info] Initial Registration is successful
[2023-04-25 14:36:48.137] [nas] [debug] Sending PDU Session Establishment Request
[2023-04-25 14:36:48.137] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-04-25 14:36:48.447] [nas] [debug] PDU Session Establishment Accept received
[2023-04-25 14:36:48.447] [nas] [info] PDU Session establishment is successful PSI[1]
[2023-04-25 14:36:48.478] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.1.0.1] is up.
Stream closed EOF for free5gc/ueransim-ue-7f76db59c9-c4ltw (ue)
```

</p>
</details> 

<details><summary>UERANSIM UE successfully connected status "<strong>cm-state: CM-CONNECTED</strong>"</summary>
<p>

Open UE pod's shell. `kubectl exec -n ${NS_NAME} --stdin --tty ${UE_POD_NAME} -- /bin/bash`

- Command for open5gs openverso: `nr-cli imsi-999700000000001 -e status`

- Command for free5gc towards5gs: `./nr-cli imsi-208930000000003 -e status`

```ruby
<<K9s-Shell>> Pod: open5gs/ueransim-ueransim-gnb-ues-5b9d9c577b-zwb6d | Container: ues
bash-5.1# nr-cli imsi-999700000000001 -e status
cm-state: CM-CONNECTED
rm-state: RM-REGISTERED
mm-state: MM-REGISTERED/NORMAL-SERVICE
5u-state: 5U1-UPDATED
sim-inserted: true
selected-plmn: 999/70
current-cell: 1
current-plmn: 999/70
current-tac: 1
last-tai: PLMN[999/70] TAC[1]
stored-suci: no-identity
stored-guti:
 plmn: 999/70
 amf-region-id: 0x02
 amf-set-id: 1
 amf-pointer: 0
 tmsi: 0xf9007746
has-emergency: false
bash-5.1#
bash-5.1# ping -I uesimtun0 -c1 1.1.1.1
PING 1.1.1.1 (1.1.1.1): 56 data bytes
64 bytes from 1.1.1.1: seq=0 ttl=57 time=2.360 ms

--- 1.1.1.1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 2.360/2.360/2.360 ms
bash-5.1#
bash-5.1# traceroute -i uesimtun0 www.google.com
traceroute to www.google.com (74.125.205.99), 30 hops max, 46 byte packets
 1  10.100.111.1 (10.100.111.1)  1.524 ms  1.246 ms  0.928 ms
 2  10.0.0.1 (10.0.0.1)  0.946 ms  1.722 ms  1.116 ms
 3  172.31.141.1 (172.31.141.1)  1.778 ms  1.990 ms  1.691 ms
 4  172.17.23.111 (172.17.23.111)  1.268 ms  1.822 ms  1.535 ms
 ......
```

</p>
</details> 

## Then UE disconnected

<details><summary>UERANSIM UE disconnected:</summary>
<p>

**cm-state: CM-IDLE**
```ruby
root@ueransim-ue-7f76db59c9-c4ltw:/ueransim/build# ./nr-cli imsi-208930000000003 -e status
cm-state: CM-IDLE
rm-state: RM-REGISTERED
mm-state: MM-REGISTERED/NORMAL-SERVICE
5u-state: 5U1-UPDATED
sim-inserted: true
selected-plmn: 208/93
current-cell: 2
current-plmn: 208/93
current-tac: 1
last-tai: PLMN[208/93] TAC[1]
stored-suci: no-identity
stored-guti:
 plmn: 208/93
 amf-region-id: 0xca
 amf-set-id: 1016
 amf-pointer: 0
 tmsi: 0x00000001
has-emergency: false
root@ueransim-ue-7f76db59c9-c4ltw:/ueransim/build#
```

Then you can try to reconnect:

- Command for open5gs openverso: `nr-cli imsi-999700000000001 -e "deregister normal"`

- Command for free5gc towards5gs: `./nr-cli imsi-208930000000003 -e "deregister normal"`

UE will send Initial Registration after 10 seconds.

</p>
</details> 

If connection can not set up, we recommend to restart components in next sequence:
1. SMF
1. AMF
1. UERANSIM GnB
1. UERANSIM UE
# Decentralized Application to Control Access to Shared Files Built with Hyperledger Fabric 1.0

The application lets users of separate organizations manage Access Control Lists for the files they share with each other.

The blockchain network can be deployed to multiple docker containers on one host for development or to multiple hosts for real 
world deployment in production or testing environments.

Network consortium consists of:

- Orderer organization `fileaccess.marlabs.com`
- Peer organization org1 `cps` 
- Peer organization org2 `india` 
- Peer organization org3 `france`

They transact with each other on the following bilateral channels via [fileaccess chaincode](chaincode/go/fileaccess):

- `cps-india`
- `cps-france`
- `france-india`

This application uses deployment scripts of its upstream source, for more details please see 
[fabric-starter](https://github.com/olegabu/fabric-starter).

## Local deployment

Deploy docker containers of all member organizations to one host, for development and testing of functionality. 

All containers refer to each other by their domain names and connect via the host's docker network. The only services 
that need to be available to the host machine are the `api` so you can connect to admin web apps of each member; 
thus their `4000` ports are mapped to non conflicting `4000, 4001, 4002` ports on the host.

Generate artifacts:
```bash
./network.sh -m generate
```

Generated crypto material of all members, block and tx files are placed in shared `artifacts` folder on the host.

Start docker containers of all members:
```bash
./network.sh -m up
```

After all containers are up, browse to each member's admin web app to transact on their behalf: 

- cps [http://localhost:4000/admin](http://localhost:4000/admin)
- india [http://localhost:4001/admin](http://localhost:4001/admin)
- france [http://localhost:4002/admin](http://localhost:4002/admin)

Tail logs of each member's docker containers by passing its name as organization `-o` argument:
```bash
# orderer
./network.sh -m logs -m fileaccess.marlabs.com

# members
./network.sh -m logs -m cps
./network.sh -m logs -m india
```
Stop all:
```bash
./network.sh -m down
```
Remove dockers:
```bash
./network.sh -m clean
```

## Decentralized deployment

Deploy containers of each member to separate hosts connecting via internet.

Specify member hosts ip addresses in [network.sh](network.sh) file or by env variables:
```bash
export IP_ORDERER=54.235.3.243 IP1=54.235.3.231 IP2=54.235.3.232 IP3=54.235.3.233
```  

The setup process takes several steps whose order is important.

Each member generates artifacts on their respective hosts (can be done in parallel):
```bash
# organization cps on their host
./network.sh -m generate-peer -o cps

# organization india on their host
./network.sh -m generate-peer -o india

# organization france on their host
./network.sh -m generate-peer -o france
```

After certificates are generated each script starts a `www` docker instance to serve them to other members: the orderer
 will download the certs to create the ledger and other peers will download to use them to secure communication by TLS.  

Now the orderer can generate genesis block and channel tx files by collecting certs from members. On the orderer's host:
```bash
./network.sh -m generate-orderer
```

And start the orderer:
```bash
./network.sh -m up-orderer
```

When the orderer is up, each member can start services on their hosts and their peers connect to the orderer to create 
channels. Note that in Fabric one member creates a channel and others join to it via a channel block file. 
Thus channel _creator_ members make these block files available to _joiners_ via their `www` docker instances. 
Also note the starting order of members is important, especially for bilateral channels connecting pairs of members, 
for example for channel `a-b` member `a` needs to start first to create the channel and serve the block file, 
and then `b` starts, downloads the block file and joins the channel. It's a good idea to order organizations in script
arguments alphabetically, ex.: `ORG1=aorg ORG2=borg ORG3=corg` then the channels are named accordingly 
`aorg-borg aorg-corg borg-corg` and it's clear who creates, who joins a bilateral channel and who needs to start first.

Each member starts:
```bash
# organization a on their host
./network.sh -m up-1

# organization b on their host
./network.sh -m up-2

# organization c on their host
./network.sh -m up-3
```

## Chaincode development

There are commands for working with chaincodes in `chaincode-dev` mode where a chaincode is not managed within its docker 
container but run separately as a stand alone executable or in a debugger. The peer does not manage the chaincode but 
connects to it to invoke and query.

The dev network is composed of a minimal set of peer, orderer and cli containers and uses pre-generated artifacts
checked into the source control. Channel and chaincodes names are `myc` and `mycc` and can be edited in `network.sh`.

Start containers for dev network:
```bash
./network.sh -m devup
./network.sh -m devinstall
```

Start your chaincode in a debugger with env variables:
```bash
CORE_CHAINCODE_LOGGING_LEVEL=debug
CORE_PEER_ADDRESS=0.0.0.0:7051
CORE_CHAINCODE_ID_NAME=mycc:0
```

Now you can instantiate, invoke and query your chaincode:
```bash
./network.sh -m devinstantiate
./network.sh -m devinvoke
./network.sh -m devquery
```

You'll be able to modify the source code, restart the chaincode, test with invokes without rebuilding or restarting 
the dev network. 

Finally:
```bash
./network.sh -m devdown
```

## Acknowledgements

This environment uses a very helpful [fabric-rest](https://github.com/Altoros/fabric-rest) API server developed separately and 
instantiated from its docker image.

The scripts are inspired by [first-network](https://github.com/hyperledger/fabric-samples/tree/release/first-network) and 
 [balance-transfer](https://github.com/hyperledger/fabric-samples/tree/release/balance-transfer) of Hyperledger Fabric samples.

============================================================
 Private Ethereum Blockchain with Multiple Nodes (Geth)
============================================================

DESCRIPTION
-----------
This project demonstrates a private Ethereum proof-of-work (Ethash)
network running three Geth nodes on a single machine. All three nodes
share the same genesis block and network ID, connect to each other
manually as peers, stay in sync while node1 mines, and exchange funds
between accounts.

  Network ID : 2026
  Consensus  : Ethash (proof-of-work)
  Geth       : v1.10.26-stable
  Nodes      : node1, node2, node3
  Accounts   : 3 per node (all lab passwords = "root")

NOTE: The password "root" is for learning/lab use only. Never use it
for real accounts, and never commit keystore folders to Git.


PROJECT STRUCTURE
-----------------
  node1.json        Genesis file (created with puppeth)
  startnode1.sh     Script to start node1
  startnode2.sh     Script to start node2
  startnode3.sh     Script to start node3
  README.txt        This file


PREREQUISITES
-------------
  - Ubuntu (VirtualBox VM used here)
  - geth and puppeth (geth-alltools 1.10.26) available in PATH
  - Three terminal windows for the nodes, plus more for the consoles


STEP 1: CREATE THE GENESIS BLOCK
--------------------------------
  mkdir privateBlockChainWithMultipleNodes
  cd privateBlockChainWithMultipleNodes
  puppeth

In puppeth:
  - Network name           : node1
  - Configure new genesis  : Create new genesis from scratch
  - Consensus engine       : Ethash (proof-of-work)
  - Pre-funded accounts    : (none needed, funds come from mining)
  - Pre-fund precompiles   : no
  - Chain/Network ID       : 2026
  - Manage existing genesis -> Export genesis configurations
    (this creates node1.json in the current folder)

Exit puppeth with Ctrl+C after the export.


STEP 2: INITIALIZE ALL NODES WITH THE SAME GENESIS
--------------------------------------------------
  geth --datadir node1 init node1.json
  geth --datadir node2 init node1.json
  geth --datadir node3 init node1.json

All three must print the same genesis hash. If the hashes differ,
the nodes will not peer.


STEP 3: CREATE THE START SCRIPTS
--------------------------------
Each node on the same machine needs unique ports.

startnode1.sh
  geth --datadir node1 --networkid 2026 --nodiscover \
    --http --http.api eth,net,web3,personal,miner,admin \
    --allow-insecure-unlock

startnode2.sh
  geth --datadir node2 --networkid 2026 --nodiscover --port 30304 \
    --http --http.port 8546 --http.api eth,net,web3,personal,miner,admin \
    --authrpc.port 8552 --allow-insecure-unlock

startnode3.sh
  geth --datadir node3 --networkid 2026 --nodiscover --port 30305 \
    --http --http.port 8547 --http.api eth,net,web3,personal,miner,admin \
    --authrpc.port 8553 --allow-insecure-unlock

Make them executable:
  chmod +x startnode1.sh startnode2.sh startnode3.sh

Start each node in its own terminal:
  ./startnode1.sh
  ./startnode2.sh
  ./startnode3.sh


STEP 4: ATTACH TO EACH NODE (IPC CONSOLE)
-----------------------------------------
Open a new terminal for each node:
  geth attach node1/geth.ipc
  geth attach node2/geth.ipc
  geth attach node3/geth.ipc


STEP 5: CREATE 3 ACCOUNTS ON EACH NODE
--------------------------------------
In each node's console, run:
  personal.newAccount("root")
  personal.newAccount("root")
  personal.newAccount("root")

Check them:
  eth.accounts

The first account (eth.accounts[0]) is the coinbase, the account that
receives mining rewards. On node1, set it explicitly if needed:
  miner.setEtherbase(eth.accounts[0])
  eth.coinbase


STEP 6: CHECK PEERS
-------------------
In each console:
  admin.peers
  net.peerCount

At first these are empty ([] and 0), because --nodiscover is on and
the nodes do not know each other yet.


STEP 7: CONNECT THE NODES USING ENODE
-------------------------------------
On node1, get its enode:
  admin.nodeInfo.enode

On node2 and node3, add node1 as a peer (keep both quotes):
  admin.addPeer("enode://<node1-enode-id>@127.0.0.1:30303")

It should return true. Verify on all nodes:
  admin.peers
  net.peerCount

Node1 should now show 2 peers. Optionally, connect node2 and node3
to each other the same way, using node2's enode (port 30304).


STEP 8: START MINING ON NODE1 AND OBSERVE SYNC
----------------------------------------------
On node1:
  miner.start(1)

The first run builds the Ethash DAG, which can take several minutes
inside a VM. Watch the node1 terminal for "Successfully sealed new
block" messages. Then stop mining when enough blocks exist:
  miner.stop()

On node1, node2 and node3, compare:
  eth.blockNumber
  eth.getBlock(1).hash

The values should match, which shows the nodes are synced.


STEP 9: TRANSFER FUNDS FROM NODE1 COINBASE TO NODE2 AND NODE3
-------------------------------------------------------------
Get the destination addresses by running this on node2 and node3:
  eth.accounts[0]

On node1, unlock the coinbase and send ether:
  personal.unlockAccount(eth.coinbase, "root", 600)

  eth.sendTransaction({
    from: eth.coinbase,
    to: "<node2 account address>",
    value: web3.toWei(10, "ether")
  })

  eth.sendTransaction({
    from: eth.coinbase,
    to: "<node3 account address>",
    value: web3.toWei(10, "ether")
  })

Transactions stay pending until a block includes them, so mine again:
  miner.start(1)
  (wait a couple of blocks)
  miner.stop()

Check the pending pool if needed:
  txpool.status


STEP 10: VERIFY BALANCES
------------------------
On any node:
  web3.fromWei(eth.getBalance("<node1 coinbase>"), "ether")
  web3.fromWei(eth.getBalance("<node2 account>"), "ether")
  web3.fromWei(eth.getBalance("<node3 account>"), "ether")

Node2 and node3 accounts should each show 10 ether.


TROUBLESHOOTING
---------------
  - Console shows "..." : an unfinished command (usually a missing
    quote). Press Ctrl+C and retype it. addPeer is case-sensitive.
  - addPeer returns true but peerCount stays 0 : check that the
    genesis hash (eth.getBlock(0).hash) and networkid match.
  - "address already in use" : each node needs its own --port,
    --http.port and --authrpc.port.
  - "unavailable=[web]" : use web3 (not web) in --http.api.


SUMMARY OF WORKFLOW
-------------------
  1. Create genesis with puppeth
  2. Initialize node1, node2, node3 with the same genesis
  3. Write three start scripts with unique ports
  4. Start all nodes and attach with geth attach
  5. Create 3 accounts per node (password: root)
  6. Inspect peers, then add node1's enode on node2 and node3
  7. Mine on node1, confirm node2 and node3 sync
  8. Send funds from node1's coinbase to node2 and node3 accounts
  9. Mine to confirm the transactions, then check balances

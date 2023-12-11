# Universal zk-coprocessor Design Document

## Overview
The zk-Corprocessor is a novel off-chain scaling solution based on zkEVM and the OP stack. Leveraging zk-Corprocessor, developers can design data-driven decentralized applications (dApps) that enable low-cost execution of complex computations, all while maintaining the security at the entire chain level, without the need for any additional trust assumptions.

<img src="https://github.com/zk-coprocessor/docs/blob/main/img/pic1.png" alt="High-level Structure" width="1500" height="auto">

The core design of zk-Corprocessor will encompass four key aspects:
- Communication between users and off-chain environment, on-chain and off-chain environment.
- Outsourcing and scheduling of zero-knowledge proof tasks.
- Low-cost zero-knowledge proof generation, compression, and verification services (with recursive compression of zero-knowledge proofs provided by third-party technical service providers).
- Dispute resolution mechanism based on zero-knowledge validity proofs.

## User Interaction, Communication, and Execution

### User <-> zk-Coprocessor
**Design Objectives**:
This section aims to implement JSON RPC endpoints (HTTP interface) to allow users (such as Metamask, dApps, AA Wallets, Etherscan, etc.) to interact with the off-chain coprocessor nodes.
The interaction mode of zk-Coprocessor RPC should be fully compatible with Ethereum RPC, while also supporting specific additional endpoints within the coprocessor network, such as batches, proofs, L1 verification transactions, and more. Users can interact with the network's state (retrieve data and process transactions) and engage with the transaction pool through these endpoints.

**Process Design**:

<img src="https://github.com/zk-coprocessor/docs/blob/main/img/pic2.png" alt="pic2" width="650" height="auto">

**Reference Materials**:
The development of endpoints should adhere to the OpenRPC Specs and may also refer to the relevant implementation of Polygon zkEVM Endpoints.
```JSON
    {
      "name": "zkcop_batchNumber",
      "summary": "Returns the latest batch number.",
      "params": [],
      "result": {
        "$ref": "#/components/contentDescriptors/BatchNumber"
      },
      "examples": [
        {
          "name": "example",
          "description": "",
          "params": [],
          "result": {
            "name": "exampleResult",
            "description": "",
            "value": "0x1"
          }
        }
      ]
    }
```

### On-Chain <-> zk-Coprocessor
The communication between on-chain and off-chain zk-Coprocessor involves the implementation of four components in the overall process:
- `Relayer`: The zk-Coprocessor obtains and synchronizes relevant on-chain state data to the zk-Coprocessor.
- `EthTxManager`: Manages all system requests requiring transaction submission to the on-chain.
- `zkCoordinator`: Handles off-chain transaction transactions, schedules tasks for batch zero-knowledge proofs (zkp) and game session zkp, and is responsible for initiating zkp verification requests sent to the on-chain via `EthTxManager`.
- `zkVerifier`: A smart contract deployed on the chain to verify zkps and return verification results.

#### Relayer

**Design Objectives**:
Monitor on-chain relevant states and events, synchronize on-chain and game-related data to zk-Coprocessor to provide data for game session initialization and other operations. Additionally, the Relayer will be responsible for fetching data from Layer 1 (L1) for the verification contract zkVerifier, updating the StateDB to ensure the availability of the off-chain environment's state.

**Process Design**:

<img src="https://github.com/zk-coprocessor/docs/blob/main/img/pic3.png" alt="pic3" width="650" height="auto">

**Reference Materials**:
Regarding the functionality of synchronization and monitoring, you may refer to the implementation of the Polygon zkEVM synchronizer.

#### EthTxManager & zkCoordinator

**Design Objectives**:
In addition to handling all system requests requiring transaction submission to the on-chain, this module should also manage information related to L1 gas price, account nonce, and other relevant details to ensure the overall cost-effectiveness and smooth operation of the system.
In this design, one of the core functions of EthTxManager is to receive zkp verification requests from zkCoordinator and send them to the on-chain contract zkVerifier.

**Process Design**:

<img src="https://github.com/zk-coprocessor/docs/blob/main/img/pic4.png" alt="pic4" width="650" height="auto">

**Reference Materials**:
The implementation details of `zkCoordinator` will be explained in the scheduling of zkp tasks. The implementation of `EthTxManager` will refer to the implementation of Polygon zkEVM ethtxmanager and the abstracted method implementation in etherman.

#### zkVerifier
**Design Objectives**:
The implementation of the verification contract will be based on the off-chain zkp generation algorithm.

### Example
Here, we will use the MUD-based game Sky Strife as an example to further explain the communication process:

<img src="https://github.com/zk-coprocessor/docs/blob/main/img/pic5.png" alt="pic5" width="1200" height="auto">

- Players initialize the game on-chain (which may involve staking a certain amount of assets).
- During the initialization process, zk-Coprocessor will call the on-chain `zkRelayer` contract, triggering an event to provide information for the `Relayer` to monitor and execute off-chain operations, such as setting up game accounts and initializing off-chain game session contracts.
- The conclusion of the game will be triggered by an event from the off-chain game session contract, generating the corresponding proof, and the dispute resolution mechanism will determine whether to submit it to the chain.
- If submission to the chain is required, the zk-proof will be submitted to `EthTxManager` by `zkCoordinator`, requesting on-chain verification.
- The results of on-chain verification will be accepted and processed by the `zkRelayer` contract, and the on-chain contract of the game will settle the on-chain state based on this result.

## zk-Proof Generation Coordination

By using zk-proofs, we can achieve a balance between privacy protection and ensuring fairness in gaming. Players can verify their own actions and those of other players to ensure compliance with game rules without revealing detailed information about the game state. However, since the states of individual games are not interdependent, each game maintains its own state roots. In order to achieve low-cost zk-proof generation for game verification, a multi-layer compression of their respective state roots has been implemented. This solution ensures the security of the entire chain while achieving verification, continuity, and independence of game sessions. (Individual game states are self-maintained, and the global game state is maintained by L1/L2, eliminating the need to maintain global states across game sessions at this layer. Zk-proofs are used to validate the legitimacy of game operations while protecting player privacy.)

### Background
The production of proofs in zkEVM depends on the sequential generation of blocks and global variables within the entire EVM environment.

**Production Chain**:  
Tx -> block -> batch -> proof

**Overall Generation Relationship**:

<img src="https://github.com/zk-coprocessor/docs/blob/main/img/pic6.png" alt="pic6" width="1000" height="auto">

**Relationship between multiple batches**: 
Batches are constructed based on the order of blocks, and the verification of corresponding proofs is also conducted through iterative verification of the global state.

### Purpose
The goal of zk-Corprocessor is to operate on a per-game session basis, starting from the initiation of an individual game session and concluding with the end of that session. After the game concludes, the corresponding batch will generate a proof, which is then submitted for verification to a higher-level chain.

**Production Chain**:  
Tx -> block -> batch -> proof

**Overall Generation Relationship**:

<img src="https://github.com/zk-coprocessor/docs/blob/main/img/pic7.png" alt="pic7" width="1000" height="auto">

**Relationships Between Multiple Batches**: 
Between batches, due to the individual validation of each game session, they have detached from any sequential relationship. Each batch has its own new state root to determine the continuity and relevance of transactions, and multiple batches are no longer correlated. In reality, multiple game sessions are products of different EVM environments, so each game session has its own related EVM environment and corresponding state root. The states being verified are effectively parallel states.

**Existing Issues**:
Even though the state roots of multiple game sessions are independent, the handling of account states remains a cross-state problem.

### Initiation and Conclusion of Game Sessions
*TODO*

## OP + ZK Dispute Resolution Mechanism

**Design Objectives**:
The goal is to ensure effectiveness and security while saving costs through the OP + ZK format.
The overall mechanism follows a similar process to OP, where, without triggering a challenge, the mechanism operates similarly. The key difference lies in the fact that our network only generates zk-proof to L1 when a challenge is triggered.

**Process Design**:

<img src="https://github.com/zk-coprocessor/docs/blob/main/img/pic8.png" alt="pic8" width="1300" height="auto">

1. Game concludes, and the Prover receives the end signal (listening to events/identifying transactions).
2. The Prover uploads the final state/all transactions to L1.
3. A potential Challenger receives evidence/data to decide whether to challenge.
4. If the Challenger decides to challenge, the Prover generates proof for the corresponding data.
5. The Prover uploads the generated proof and corresponding data to L1.
6. On L1, the contract verifies the proof.
7. Execute the game result.


**Some Explorations of Possibilities**:
1. The second step may not necessarily require uploading all transactions. In OP, uploading all transactions is necessary because the fraud proof needs to replay the state root on L1, which may not be required in our solution.
2. The triggering mechanism for challenges: On OP, the challenger can be a network consensus maintainer, whereas our challenge might need to be triggered by players.
3. The source of information triggering challenges: If players initiate challenges, the basis for their judgment can come from L1 (on-chain result computation) or from L3 (game final state).


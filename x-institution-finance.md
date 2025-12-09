# Tokenized Finance Architecture 

## Summary

Blockchain and tokenization has the potential to profoundly transform the financial industry. Arguably the Ethereum ecosystem has strong presence in this field, but it intrinsicallly suffers from security, privacy and compliance concerns on implementing utility tokens, NFT and RWA.   

Canton by design has security, privacy and compliance in mind, but it isolates itself away from the massive token ecosystem on Ethereum. Its tokenization implementation via Splice looks like a "bolted-on" afterthought.

This document records JZ's architecure proposals on tokenized finance, with the following goals and objectives:
1. Fully compatible with the Ethereum ecosystem, including Solidity smart contracts, ERC standards, zero-knowledge proofs etc. Better if it runs on public Ethereum.
2. Privacy and compliance strength on par or better than the Canton network for cross-institution collaboration. Better if compatible with Canton protocol and DAML.

## Preface

PDX has been supporting totally private smart contracts without weakening consensus strength on its Utopia blockchain platform since 2017. It does this via a patent-awarded technology internally called "selective-existence of data and code" on blockchain nodes, meaning only the nodes trusted by the stakeholders of business collaboration posess the sensitive data (Contract_State) and code (logic) of that collaboration. What canton can achieve is already achieved on PDX Utopia blockchain - some caveat or loose end may exist though.

PDX Utopia blockchain is fully compliant with Ethereum EVM and its web3 API. Besides its support on totally private smart contracts, PDX Utopia has quite some unique capabilities, such as:

1. *Asynchronous consensus* A scalable, performant, asynchronous consensus algorithm achieving fair consensus (each node with one vote) with O(n) complexity, also known as PDX consensus
  
2. *Asynchronous ledgering* Each smart contract cluster has its own distinguished world Contract_State that does not interfere with other smart contract clusters, hence can independenly execute without interfering each other.
  
3. *Parallel TX processing* Transactions destined for unrelated smart contracts are processed in parallel; related smart contracts auto-organized into a "smart contract cluster".

## Option 1: SC + Oracle 

![plot](./x-institution-finance-v3.png)

```mermaid
sequenceDiagram
box Users
actor Alice
actor Bob
end
box Institutions
participant Service
participant Facilitator@{"type": "control"}
participant Executor
participant Ledger@{"type": "database"}
participant Prover
participant Messaging
end
box Mediation Blockchains
participant Identity
participant Topology
participant Sequencer@{"type": "queue"}
participant Mediator@{"type": "control"}
end
Alice -->> Bob: get Bob's conditional authorization
Alice -->> Service: initiate coin swap
Service -->> Facilitator: request for cross-institution mediation
Facilitator -->> Messaging: confidential data for mediation tx with end2end security
Facilitator -->> Facilitator: create & sign mediation tx
Facilitator -->> Sequencer: mediation tx
Sequencer -->> Identity: check authorization
Sequencer -->> Topology: find mediation chain
Sequencer -->> Sequencer: timestamp & sequence mediation tx
Sequencer -->> Mediator: sequenced mediation transaction
Mediator -->> Facilitator: commit_prep request
Facilitator -->> Messaging: get confidential data
Facilitator -->> Executor: execute without commit
Executor -->> Facilitator: commit prep result
Facilitator -->> Mediator: commit_prep response
Mediator -->> Mediator: collect commit_prep responses util decision threshold
Mediator -->> Facilitator: emit commit_exec request
Facilitator -->> Executor: commit tx
Executor -->> Ledger: commit per-contract state
Executor -->> Prover: generate proof
Executor -->> Facilitator: status, per-contract state root, zero-knowledge proof
Facilitator -->> Mediator: commit_exec response
Mediator -->> Facilitator: emit commit_exec_succeeded or commit_exec_failed event
Facilitator -->> Service: mediation tx result
Service -->> Alice: coin swap result
Service --> Bob: coin swap result
```

This architecture is meant to work via not only **consortium blockchain**s but also **public blockchain**s (e.g. Ethereum) as the *mediation blockchain*, due to the fact no customization or patching of blockchain platform is needed to make it work.

There could have a tree of *mediation blockchain*s. As long as all stakeholders of a *mediation transaction* share the same "parent" *mediation blockchain*, this architecture would work, just "chain" the *sequencer*s all the way up to the one on "parent" *mediation blockchain*, and "chain" the *mediator*s all the way down to the one on each *mediation blockchain* that directly connects the respective stakeholders.

When using a tree of *mediation blockchain*s, all of which can share the same utility token for billing of the users and rewarding of the providers.The utility token minted on one of the *mediation chain*s and migrated to others. The utility token can even be by a organizer of the ecosystem.

Let's use a two party coion swap example to illustrate how it works with privacy and confidentiality protection of all stakeholders. In this example, Alice and Bob have mutually decided to swap Alice's $a of A coin with Bob's $b of B coin, the following is the workflow on this architecture:

1. Alice initiates a coin swap with Bob via its service, *service @ a* provided by her institution, *institution (a)*
2. The *service @ a* by configuration (or via *contract reg* smart contract) finds that *B coin* is via *institution (b)*, so it's a cross-institution transaction, so it calls its cross-chain *facilitator* with all information needed
3. The cross-chain *facilitator* @ *institution (a)* optionally calls its *messaging* @ *institution (a)* to send confidential data (e.g. detailed terms) to *institution (b)*. The following is a multipart example:
   ```
   Content-Type: multipart/form-data; boundary=----~~~~~~~~~~

   ----~~~~~~~~~~
   Content-Type: application/json
   Content-Disposition: form-data; name="json_rpc_req"
   {
      "jsonrpc": "2.0",
      "method": "txdata_push",
      "ref": "af627cae-eee9-47e9-aa6a-5ae9435b1fea,
      "attn": "uuid-of-institution",
      "algo": "",
      "data": "simple data (possibly base-64 encoded)",
      "blob": ["extra_part"]
   }
   ----~~~~~~~~~~
   Content-Type: application/octet-stream
   Content-Disposition: form-data; name="extra_part"
   Content-Length: [length of raw data]
      [...RAW BYTES GO HERE, NO ENCODING...]
   ----~~~~~~~~~~
   ```
   Note: extra protection other than TLS *may* be needed in production.
4. The cross-chain *facilitator* @ *institution (a)* sends a *mediation transaction* (*tx-{x}*) (using its *signing key*) to *mediation blockchain*'s *sequencer* smart contract
    ```
    {
      "ref": "af627cae-eee9-47e9-aa6a-5ae9435b1fea",
      "parent":  "af627cae-eee9-47e9-aa6a-5ae9435b1fe0",
      "precond": [],
      "parties": [
          {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feA"},
          {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feB"},
      ]
    }
    ```
    Note: party id better be some integer format with scheme for performance reasons.
5. After checking with *identity* smart contract for authorization, the *sequencer* smart contract automatically timestamps, sequences *tx-{x}* and calls the *mediator* smart contract.
    ```
    {
      "ref": "af627cae-eee9-47e9-aa6a-5ae9435b1fe0",
      "parent":  "af627cae-eee9-47e9-aa6a-5ae9435b1fe0",
      "precond": [], # tx dependencies, not for sensitive data
      "parties": [
          {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feA"},
          {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feB"},
      ],
      "timestamp": "{chain_id}.{block.timestamp}.{tx_idx}",
    }
    ```
6. The *mediator* smart contract, emits a *commit_prep_request* event for each party specified in the *mediation transaction* (*tx-{x}*)
    ```
    {
      "type": "commit_prep_request",
      "ref": "af627cae-eee9-47e9-aa6a-5ae9435b1fe0",
      "parent":  "af627cae-eee9-47e9-aa6a-5ae9435b1fe0",
      "precond": [], # tx dependencies, not for sensitive data
      "parties": [
          {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feA"},
          {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feB"},
        ],
      "timestamp": "{chain_id}.{block.timestamp}.{tx_idx}",
    }
    ```
8. The cross-chain *facilitator* on all stakeholders of the *mediation transaction* receives the above *commit_prep_request* event, then calls the prep_commit method of its *local execuator* respectively.
9. Each cross-chain *facilitator* of the stakeholders of the *mediatation transaction* (*tx-{x}*), gets the optional confidential data from its local *messaging* service by ref id, then calls its *local executor* to check authorization and fesibility and return vote (Y/N), possibly time-locks the per-contract Contract_State for commit.  
10. Each cross-chain *facilitator* of the stakeholders, sends a *commit_prep_response* (*tx-{x}.0*) to the *mediator* smart contract on the *mediation blockchain*
    ```
    {
      "ref": "af627cae-eee9-47e9-aa6a-5ae9435b1fea",
      "type": "commit_prep_response",
      "vote": "YES | NO",
      "stat": "SHA3 of per-contract Contract_State root", # optional
      "proof": "zero knowledge proof", # optional
      "parties": [ # unless deligated, only one tied to signing key will be taken.
          {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feA"},
          {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feB"},
      ]
    }
    ```
11. The *mediator* smart contract on the *mediation blockchain*, after receiving enough consensus on *tx-{x}*, emits a *commit_exec_request* event to all stakeholders. If not enough consensus, emit *commit_exec_failed".
    ```
    {
        "type": "commit_exec_request",
        "ref": "af627cae-eee9-47e9-aa6a-5ae9435b1fe0",
        "parties": [ # votes, details on chain
            {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feA"},
            {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feB"},
          ],
      }
    ```
12. The cross-chain *facilitator* on all stakeholders of the *mediation transaction* receives the above *commit_exec_request* event, then calls the exec_commit method of its *local execuator* respectively and returns with the per-contract Contract_State root and opportioanlly a zero-knowledge proof (by calling its *zk_prover*).

13. The cross-chain *facilitator* on all stakeholders sends a *commit_exec_response* (*tx-{x}.1*) to the *mediator* smart contract on the *mediation chain*.
    ```
    {
        "type": "commit_exec_response",
        "ref": "af627cae-eee9-47e9-aa6a-5ae9435b1fe0",
        "stat": "SHA3 of per-contract Contract_State root", # optional
        "proof": "zero knowledge proof", # optional
        "parties": [ # unless deligated, only one tied to signing key will be taken.
            {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feA"},
            {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feB"},
        ]
      }
    ```
14. The *mediator* smart contract on the *mediation blockchain*, after receiving all necessary *commit_exec_response*, marks the *mediation transaction* as finished and emmits *mediation succeeded* event.
    ```
    {
        "type": "commit_exec_succeeded",
        "ref": "af627cae-eee9-47e9-aa6a-5ae9435b1fe0",
    }
    ```   
15. The cross-chain *facilitator* on *institution (a)* and *institution (b)* receives the *commit_exec_succeeded* event, forwards it to the *service @ a* and *service @ b*, which respectively notifies Alice and Bob of the status.

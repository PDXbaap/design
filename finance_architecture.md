# Tokenized Finance Architecture 

## Summary

Blockchain and tokenization has the potential to profoundly transform the financial industry. Arguably the Ethereum ecosystem has strong presence in this field, but it intrinsicallly suffers from security, privacy and compliance concerns on implementing utility tokens, NFT and RWA.   

Canton by design has security, privacy and compliance in mind, but it isolates itself away from the massive token ecosystem on Ethereum. Its tokenization implementation via Splice looks like a "bolted-on" afterthought.

This document records JZ's architecure proposals on tokenized finance, with the following goals and objectives:
1. Fully compatible with the Ethereum ecosystem, including Solidity smart contracts, ERC standards, zero-knowledge proofs etc. Better if it runs on public Ethereum.
2. Privacy and compliance strength on par or better than the Canton network for cross-institution collaboration. Better if compatible with Canton protocol and DAML.

## Preface

PDX has been supporting totally private smart contracts without weakening consensus strength on its Utopia blockchain platform since 2017. It does this via a patent-awarded technology internally called "selective-existence of data and code" on blockchain nodes, meaning only the nodes trusted by the stakeholders of business collaboration posess the sensitive data (state) and code (logic) of that collaboration. What canton can achieve is already achieved on PDX Utopia blockchain - some caveat or loose end may exist though.

PDX Utopia blockchain is fully compliant with Ethereum EVM and its web3 API. Besides its support on totally private smart contracts, PDX Utopia has quite some unique capabilities, such as:

1. *Asynchronous consensus* A scalable, performant, asynchronous consensus algorithm achieving fair consensus (each node with one vote) with O(n) complexity, also known as PDX consensus
  
2. *Asynchronous ledgering* Each smart contract cluster has its own distinguished world state that does not interfere with other smart contract clusters, hence can independenly execute without interfering each other.
  
3. *Parallel TX processing* Transactions destined for unrelated smart contracts are processed in parallel; related smart contracts auto-organized into a "smart contract cluster".

## Option 1: On Ethereum 

![plot](./x-chain-finance-architecture.png)

Let's use a two party coion swap example to illustrate how it works with privacy and confidentiality protection of all stakeholders. In this example, Alice and Bob have mutually decided to swap Alice's $a of A coin with Bob's $b of B coin, the following is the workflow on this architecture:

1. Alice initiates a coin swap with Bob via its service, *service-x @ a* provided by her institution, *institution (a)*
2. The *service-x @ a* by configuration finds that *B coin* is via *institution (b)*, so it's a cross-institution transaction, so it calls its *x-chain mediator* with all information needed
3. The *x-chain mediator* @ *institution (a)* optionally calls its *messaging* @ *institution (a)* to send confidential data (e.g. detailed terms) to *institution (b)*
   ```
   {
     "ref": "af627cae-eee9-47e9-aa6a-5ae9435b1feI,
     [
       {
          "attn": "institution (b)",
          "data": "bob->alice $b of B coin",
          "authz": {
                     "signer": "bob's key",
                     "salt": "...",
                     "sig": "....
                   }
       }
     ]
   ```
4. The *x-chain mediator* @ *institution (a)* sends a *mediation transaction* (*tx-1*) (using its *signing key*) to *mediation chain*'s *sequencer* smart contract
  ```
  {
    "ref": "af627cae-eee9-47e9-aa6a-5ae9435b1fe0",
    "timestamp": "{block.timestamp}.{tx_idx}@{chain_id}"
    "parent":  "af627cae-eee9-47e9-aa6a-5ae9435b1fe0",
    "precond": [],
    "parties": [
        {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feA"},
        {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feB"},
    ]
  }
  ```
6. After checking with *identity* smart contract for authorization, the *sequencer* smart contract automatically timestamps, sequences *tx-1* and calls the *mediator* smart contract.
7. The *mediator* smart contract, emit a *confirmation request* event for each party specified in the *mediation transaction* (*tx-1*)
   ```
  {
    "type": "conformation_request",
    "ref": "af627cae-eee9-47e9-aa6a-5ae9435b1fe0,
    "timestamp": "{chain_id}.{block.timestamp}.{tx_idx}"
    "parties": [
        {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feA"},
        {"uuid": "af627cae-eee9-47e9-aa6a-5ae9435b1feB"},
      ]
    }
   ```
9. The *x-chain mediator* on all stakeholders of the *mediation transaction* receives the above *confirmation request*, then calls the prep_transact method of its *local execuator* respectively.  
10. Each *x-chain mediator* of the stakeholders of the *mediatation transaction* (*tx-1*), gets the optional confidential data from its local *messaging* service by ref id, then calls its *local executor* to execute it.
     

  
```

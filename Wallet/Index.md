# Family wallet project

This project describes a wallet service to be used by a household. The idea is that a service
like this makes sense so specific EIP-1193-enabled wallets connect to it and let the users
send transactions that are:

- Ultimately signed by this remote service.
- ERC-4337-abstracted, so a post-quantum era involves just an owner/verifier change and not
  a full migration. Ideally, these wallets will work pretty much like the Safe wallets, with
  an owner and a secondary signer (e.g. policy verifier).

The intended advantages for this project are:

- Wallets can have role-based restrictions. Children-intended wallets might have policies
  regarding allowed apps, allowed tokens consumption, and allowed contracts, throttling, or
  external reasons (e.g. a "grounded" status would prevent the child-intended account to
  be inactive). Parents would be able to control their children's activities on-chain with
  just a button press (policies would never be on-chain).
- Wallets can be migrated easily. Post-quantum cryptography will not be ECDSA and some day
  even the ECDSA scheme will disappear. This makes the use of EOAs quite impractical: there
  is a possibility that a user has to account for several contracts they participated into
  and move all the tokens (e.g. liquidity pools, ERC-20 tokens, ERC-721 tokens, accounts in
  custom applications or video games, ...). In the best cases, few tokens need to be moved
  to new, non-ECDSA, accounts (via allowed per-contract transfer methods). In the worst ones,
  it may become impossible (out of pure design). This means that a lot of stuff might be lost
  forever when the owner of the EOA wants to transfer all the assets from their ECDSA EOA to
  the new, non-ECDSA, account. This is not a problem using ERC-4337 accounts since, in the
  worst case, the account remains the same but the owner / policy checker user is changed to
  a new value. Less gas. Fewer operations. Less footprint / traceability.
- Somewhat connected to the restrictions, app / game catalogs can be configured. They involve
  entries in database and / or on-chain that can be retrieved / enumerated and modified so
  those apps / games are available, forbidden or tracked when interacted with.

Front-ends can exist in two ways:

- Regular browser extensions (similar to Coinbase or Metamask).
- Mobile applications with an embedded browser (similar to what Metamask allows in their app,
  but a clearer UX).

As of today, the wallet has no name. However, this documentation hierarchy will define the
involved components and the way the interactions and users' life-cycles take place.

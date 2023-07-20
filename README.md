# Audit scope

### On what chains are the smart contracts going to be deployed?

- Mainnet, potentially L2s like Arbitrum + Optimism
___

### Which ERC20 tokens do you expect will interact with the smart contracts? 

- weth, wbtc, ARB, UNI
- stables like dai, usdc
___

### Which ERC721 tokens do you expect will interact with the smart contracts? 
- N/A
___

### Which ERC777 tokens do you expect will interact with the smart contracts? 
- N/A
___

### Are there any FEE-ON-TRANSFER/ REBASING tokens interacting with the smart contracts?
- No
___

### Are the admins of the protocols your contracts integrate with (if any) TRUSTED or RESTRICTED?

The only admin relevant to the full margin engine is the admin of `Grappa` contract. Grappa is currently upgradable and that will be the only trust assumption. Other than upgradeability, the admin of Grappa can only add new asset and oracles to the protocol (and cannot remove them), so it should not affect the full margin engine in anyway.

so it is currently "TRUSTED" as grappa can still upgrade, but will become "RESTRICTED" onces we renounce the ownership of grappa contract when it's stable

___

### Is the admin/owner of the protocol/contracts TRUSTED or RESTRICTED?
- there is no admin to this module
___

### Are there any additional protocol roles? If yes, please explain in detail:
- N/A
___

### Is the code/contract expected to comply with any EIPs? Are there specific assumptions around adhering to those EIPs?
- N/A
___

### Please list any known issues/acceptable risks that should not result in a valid finding.
- N/A
___

### Please provide links to previous audits (if any).
- `BaseEngine` of from the core repository is previously audited, audit report and commit can be found [here](https://github.com/grappafinance/core-cash/blob/master/audits/chainsafe-feb-2023.pdf)
___

### Are there any off-chain mechanisms or off-chain procedures for the protocol (keeper bots, input validation expectations, etc)?
- No
___

### In case of external protocol integrations, are the risks of external contracts pausing or executing an emergency withdrawal acceptable? 
- No, the only expected (known trusted risk) should be the one mentioned before: if Grappa upgrade to a new implementation and rug this contract
___

### Do you expect to use any of the following tokens with non-standard behaviour with the smart contracts?
- No
___

### Please provide the commit hash and contracts which needs to be audited

All the file in [`grappafinance/full-collat-engine/src`](https://github.com/grappafinance/full-collat-engine/tree/42d52af7d802521f88ca0cb89e36afcffda1182b/src), including contracts inherited, under commit [42d52](https://github.com/grappafinance/full-collat-engine/tree/42d52af7d802521f88ca0cb89e36afcffda1182b). Explicitly:

#### Grappa Cash-Core inheritance @ 8e67b0b
- [BaseEngine.sol@8e67b0b](https://github.com/grappafinance/core-cash/blob/8e67b0bedf53679c18de76ae70345d829737df56/src/core/engines/BaseEngine.sol)
- [DebitSpread.sol@8e67b0b](https://github.com/grappafinance/core-cash/blob/8e67b0bedf53679c18de76ae70345d829737df56/src/core/engines/mixins/DebitSpread.sol)
#### Full Collat Engine Contracts @42d52af
- [FullMarginEngine.sol@42d52af](https://github.com/grappafinance/full-collat-engine/blob/42d52af7d802521f88ca0cb89e36afcffda1182b/src/FullMarginEngine.sol)
- [FullMarginLib.sol@42d52af](https://github.com/grappafinance/full-collat-engine/blob/42d52af7d802521f88ca0cb89e36afcffda1182b/src/FullMarginLib.sol)
- [FullMarginMath.sol@42d52af](https://github.com/grappafinance/full-collat-engine/blob/42d52af7d802521f88ca0cb89e36afcffda1182b/src/FullMarginMath.sol)
- [errors.sol@42d52af](https://github.com/grappafinance/full-collat-engine/blob/42d52af7d802521f88ca0cb89e36afcffda1182b/src/errors.sol)
- [types.sol@42d52af](https://github.com/grappafinance/full-collat-engine/blob/42d52af7d802521f88ca0cb89e36afcffda1182b/src/types.sol)
___


# Audit severity classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

# Audit duration
Max 2 weeks (14 days)

# Disclaimer

A comprehensive security assessment of a smart contract can never ensure the total absence of vulnerabilities. This audit cannot guarantee absolute security, nor can it guarantee the detection of any issues in your smart contracts. To bolster security, it is highly advised to conduct subsequent security reviews, establish bug bounty programs, and implement on-chain monitoring.

This is a service provided as a free audit by Delvir0.

# Approval

Anton Cheng

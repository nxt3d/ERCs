---
title: Onchain Redirect for Cross-Contract Redirection
description: This EIP defines a mechanism for redirecting smart contract calls to contracts on different blockchains using an onchain revert.
author: Prem Makeig (@nxt3d)
discussions-to: <URL>
status: Draft
type: Standards Track
category: Interface
created: 2024-10-24
---

## Abstract

Contracts wishing to redirect function calls to other contracts on different blockchains may revert using the error `OnchainRedirect`. Clients supporting this specification will be responsible for invoking the same function call on the specified `targetContract` on the provided `chainId`. The result of the call on the target contract will be passed to the callback function on the originating contract, which can be verified if necessary.

## Motivation

Cross-chain interactions between smart contracts are becoming more common. Existing solutions that rely on off-chain mechanisms such as URLs are unsuitable for decentralized censorship-resistant applications requiring purely onchain redirection. This EIP provides a standardized method to perform cross-chain smart contract interactions, creating a modular framework without requiring off-chain services.

## Specification

The following error is used for redirecting calls across blockchains:

```
error OnchainRedirect(address targetContract, uint256 chainId, bytes4 callbackFunction, bytes extraData);
```

- **`targetContract`**: The address of the contract on the destination chain that should receive the call.
  
- **`chainId`**: The ID of the blockchain where the `targetContract` resides. The `chainId` may be the same as the originating contract; however, in this case, the `targetContract` address must be different to prevent self-redirects.

- **`callbackFunction`**: A 4-byte selector for a function on the originating contract to handle the result of the cross-chain call. This field cannot be empty and must be specified.

  The callback function must have the following parameters:
  - **`bytes data`**: The ABI-encoded result of the cross-chain call.
  - **`bytes extraData`**: The extra data originally provided during the redirect.
  - **Return value**: The callback function must return the same return parameters as the originating function's return parameters.

- **`extraData`**: Arbitrary data that must be passed to the `callbackFunction` along with the result of the call to the `targetContract` as ABI-encoded bytes.

### Originating and Target Contract Functions

- **Function Parameters**: The input parameters, mutability, and return parameters of the target contract function must be the same as those of the originating contract’s function.

### Function Execution Without Redirection

The function on the originating contract can also be executed directly if it doesn't revert. This allows for flexibility in handling cases where redirection is not necessary, and the function can proceed with its logic on the originating chain.

### Client Handling

1. **Initial Call**: The client calls a function on the originating contract. If the function reverts with `OnchainRedirect`, the client extracts the `targetContract`, `chainId`, `callbackFunction`, and `extraData` from the revert error. If the call doesn't revert, the client handles the function call normally. 

2. **Cross-Chain Call**: The client constructs a call on the specified chain (`chainId`), targeting the `targetContract`. The call to the `targetContract` must use the same function signature as the originating contract’s function and be the same type of call (i.e., `staticcall` for read-only or `call` for state-changing).

3. **Handling the Response**: The client must ABI-encode the return values of the call on the `targetContract` and pass them as bytes `data`, along with the bytes `extraData` from the `OnchainRedirect` revert error, to the callback function `callbackFunction` on the originating contract.

4. **Decoding the Callback Function's Results**: The client must ABI-decode the bytes returned by the `callbackFunction` using the expected return types of the originating contract’s function.

5. **Direct Execution**: If the function does not revert with `OnchainRedirect`, it proceeds normally on the originating contract, and the client handles the response as it would with any standard contract call.

## Rationale

By using a revert with `OnchainRedirect`, we maintain a fully onchain mechanism for cross-chain smart contract interaction. Allowing the originating function to execute directly if it doesn't revert provides flexibility for developers to handle both onchain and cross-chain logic within the same function. This avoids the complexity and trust issues associated with off-chain services, ensuring that cross-chain calls remain decentralized and trust-minimized.

## Backwards Compatibility

This EIP can be made to work seamlessly with ERC-3668 by using a special URL in the format of an HTTPS link. Within the `urls` field of an `OffchainLookup` revert, an HTTPS URL with the domain `onchain-redirect.eth` can be included that indicates an onchain redirection. The URL must specify the target contract address, chain ID, and callback function selector, formatted as follows:

```
https://onchain-redirect.eth/?chainId=1234&targetContract=0xTargetAddress&callbackFunction=0xCallbackSelector
```

The `OffchainLookup` error looks like:

```
error OffchainLookup(address sender, string[] urls, bytes callData, bytes4 callbackFunction, bytes extraData);
```

where:
- **`urls`** contains the HTTPS URL indicating an onchain redirect, e.g., `https://onchain-redirect.eth/?chainId=1234&targetContract=0xTargetAddress&callbackFunction=0xCallbackSelector`.
- **`extraData`** is used as specified in this ERC.

When a client detects this HTTPS URL within an `OffchainLookup`, it can interpret the `OffchainLookup` as an `OnchainRedirect`. The client will cease the `OffchainLookup` process and proceed with step 2 of the `OnchainRedirect` specification, performing the cross-chain call using the parameters extracted from the HTTPS URL along with `extraData`, as specified by this ERC.

This enables clients to seamlessly interpret and handle both `OffchainLookup` and `OnchainRedirect` calls in a compatible manner, where the client can choose whether to perform an offchain lookup or execute an onchain redirect based on the URL format.

## Reference Implementation

Below is a simplified example of a contract that initiates a cross-chain call `crossChainCall` and uses a callback function `CrossChainCallCallback` to handle the result. The `extraData` in this example is set to `true` and is verified in the callback function.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract OriginatingContract {

    // Function that triggers the cross-chain call and reverts with OnchainRedirect
    function crossChainCall(uint256 value) external pure returns (uint256) {

        // Set the error values
        bytes4 callbackFunction = this.CrossChainCallCallback.selector;
        bytes memory extraData = abi.encode(true);  // Using true as extraData
        address targetContract = address(0x1234567890123456789012345678901234567890);
        uint256 chainId = 100;

        // Revert to signal OnchainRedirect
        revert OnchainRedirect(targetContract, chainId, callbackFunction, extraData);
    }

    // Callback function to handle the result of the cross-chain call
    function CrossChainCallCallback(bytes calldata data, bytes calldata extraData) external returns (bytes memory result) {

        // Decode the extraData and ensure it is true
        bool isValid = abi.decode(extraData, (bool));
        require(isValid, "Invalid extraData");

        // Return the result from the cross-chain call
        result = data;
    }
}
```

```
contract TargetContract {

    // Function on the target contract to be called from another chain
    function crossChainCall(uint256 value) external pure returns (uint256) {

        // Example logic, just returning the same value
        return value;
    }
}
```

In this example:

### OriginatingContract

- **`crossChainCall`**: This function immediately reverts with the `OnchainRedirect` error, passing the target contract address, chain ID, the callback function selector, and `true` as extra data.

- **`CrossChainCallCallback`**: This function decodes the `extraData`, verifies that it is `true`, and returns the result passed as `data` from the cross-chain call.

### TargetContract

- **`crossChainCall`**: This function in the target contract is called from the originating contract and returns the same value that was passed to it.

## Security Considerations

Clients must follow the specification carefully to:

- Ensure they call the correct `targetContract` on the correct `chainId`.
- Provide the results of the target contract function, ABI-encoded as bytes, and extra data from the revert error to the callback function, ensuring that the callback can properly validate the results if needed.
- The https://onchain-redirect.eth url is not intented to be used as an endpoint. Clients should not try to resolve the domain. 

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

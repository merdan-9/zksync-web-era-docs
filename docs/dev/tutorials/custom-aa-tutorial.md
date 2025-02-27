# Account abstraction multisig

This tutorial shows you how to build and deploy a 2-of-2 multi-signature account via a factory contract, then test it by sending a transaction.

## Prerequisites

- A [Node.js](https://nodejs.org/en/download) installation.
- For background learning, we recommend the following guides:
    - Read about the [design](../developer-guides/aa.md) of the account abstraction protocol.
    - Read the [introduction to the system contracts](../developer-guides/system-contracts.md).
    - Read about [smart contract deployment](../building-on-zksync/contracts/contract-deployment.md) on zkSyn Era.
    - Read the [gas estimation for transaction](../developer-guides/transactions/fee-model.md#gas-estimation-during-a-transaction-for-paymaster-and-custom-accounts) guide.
    - If you haven't already, please refer to the first section of the [quickstart tutorial](../building-on-zksync/hello-world.md).
- You should also know [how to get your private key from your MetaMask wallet](https://support.metamask.io/hc/en-us/articles/360015289632-How-to-export-an-account-s-private-key).

## Complete project

Download the complete project [here](https://github.com/matter-labs/custom-aa-tutorial).

## Set up

1. Create the project folder and `cd` into it:

```sh
mkdir custom-aa-tutorial
cd custom-aa-tutorial
```

2. Initialize the project with `yarn` and add the dependencies.

```sh
yarn init -y
yarn add -D typescript ts-node ethers@^5.7.2 zksync-web3 hardhat @matterlabs/hardhat-zksync-solc @matterlabs/hardhat-zksync-deploy
```
:::tip
The current version of `zksync-web3` uses `ethers v5.7.x` as a peer dependency. An update compatible with `ethers v6.x.x` will be released soon.
:::

3. Add the zkSync and OpenZeppelin libaries.

```sh
yarn add -D @matterlabs/zksync-contracts @openzeppelin/contracts @openzeppelin/contracts-upgradeable
```

4. Create the `hardhat.config.ts` config file, `contracts` and `deploy` folders in the project root. See the [quickstart tutorial](../building-on-zksync/hello-world.md) for more info. 

5. Include the `isSystem: true` setting in the configuration to allow interaction with system contracts.

```ts
import "@matterlabs/hardhat-zksync-deploy";
import "@matterlabs/hardhat-zksync-solc";

module.exports = {
  zksolc: {
    version: "1.3.8",
    compilerSource: "binary",
      settings: {
        isSystem: true,
      },
  },
  defaultNetwork: "zkSyncTestnet",

  networks: {
    zkSyncTestnet: {
      url: "https://testnet.era.zksync.dev",
      ethNetwork: "goerli", // Can also be the RPC URL of the network (e.g. `https://goerli.infura.io/v3/<API_KEY>`)
      zksync: true,
    },
  },
  solidity: {
    version: "0.8.16",
  },
};
```

::: tip
- Use the zkSync CLI to scaffold a project automatically. 
- Find out more about the [zkSync CLI](../../api/tools/zksync-cli/).
:::

## Account abstraction

Each account must implement the [IAccount](../developer-guides/aa.md#iaccount-interface) interface. Furthermore, since we are building an account with multiple signers, we should implement [EIP1271](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/83277ff916ac4f58fec072b8f28a252c1245c2f1/contracts/interfaces/IERC1271.sol#L12).

The skeleton code for the contract is given below. 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.16;

import "@matterlabs/zksync-contracts/l2/system-contracts/interfaces/IAccount.sol";
import "@matterlabs/zksync-contracts/l2/system-contracts/libraries/TransactionHelper.sol";

import "@openzeppelin/contracts/interfaces/IERC1271.sol";


contract TwoUserMultisig is IAccount, IERC1271 {
    // to get transaction hash
    using TransactionHelper for Transaction;


    bytes4 constant EIP1271_SUCCESS_RETURN_VALUE = 0x1626ba7e;

    modifier onlyBootloader() {
        require(
            msg.sender == BOOTLOADER_FORMAL_ADDRESS,
            "Only bootloader can call this function"
        );
        // Continure execution if called from the bootloader.
        _;
    }


    function validateTransaction(
        bytes32,
        bytes32 _suggestedSignedHash,
        Transaction calldata _transaction
    ) external payable override onlyBootloader returns (bytes4 magic) {
        magic = _validateTransaction(_suggestedSignedHash, _transaction);
    }

    function _validateTransaction(
        bytes32 _suggestedSignedHash,
        Transaction calldata _transaction
    ) internal returns (bytes4 magic) {
        // TO BE IMPLEMENTED
    }

    function executeTransaction(
        bytes32,
        bytes32,
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        _executeTransaction(_transaction);
    }

    function _executeTransaction(Transaction calldata _transaction) internal {
        // TO BE IMPLEMENTED
    }

    function executeTransactionFromOutside(Transaction calldata _transaction)
        external
        payable
    {
        _validateTransaction(bytes32(0), _transaction);
        _executeTransaction(_transaction);
    }

    function isValidSignature(bytes32 _hash, bytes memory _signature)
        public
        view
        override
        returns (bytes4 magic)
    {
        // TO BE IMPLEMENTED
    }

    function payForTransaction(
        bytes32,
        bytes32,
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        // TO BE IMPLEMENTED
    }

    function prepareForPaymaster(
        bytes32, // _txHash
        bytes32, // _suggestedSignedHash
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        // TO BE IMPLEMENTED
    }

    // This function verifies that the ECDSA signature is both in correct format and non-malleable
    function checkValidECDSASignatureFormat(bytes memory _signature) internal pure returns (bool) {
        if(_signature.length != 65) {
            return false;
        }

        uint8 v;
		bytes32 r;
		bytes32 s;
		// Signature loading code
		// we jump 32 (0x20) as the first slot of bytes contains the length
		// we jump 65 (0x41) per signature
		// for v we load 32 bytes ending with v (the first 31 come from s) then apply a mask
		assembly {
			r := mload(add(_signature, 0x20))
			s := mload(add(_signature, 0x40))
			v := and(mload(add(_signature, 0x41)), 0xff)
		}
		if(v != 27 && v != 28) {
            return false;
        }

		// EIP-2 still allows signature malleability for ecrecover(). Remove this possibility and make the signature
        // unique. Appendix F in the Ethereum Yellow paper (https://ethereum.github.io/yellowpaper/paper.pdf), defines
        // the valid range for s in (301): 0 < s < secp256k1n ÷ 2 + 1, and for v in (302): v ∈ {27, 28}. Most
        // signatures from current libraries generate a unique signature with an s-value in the lower half order.
        //
        // If your library generates malleable signatures, such as s-values in the upper range, calculate a new s-value
        // with 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141 - s1 and flip v from 27 to 28 or
        // vice versa. If your library also generates signatures with 0/1 for v instead 27/28, add 27 to v to accept
        // these malleable signatures as well.
        if(uint256(s) > 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0) {
            return false;
        }

        return true;
    }
    
    function extractECDSASignature(bytes memory _fullSignature) internal pure returns (bytes memory signature1, bytes memory signature2) {
        require(_fullSignature.length == 130, "Invalid length");

        signature1 = new bytes(65);
        signature2 = new bytes(65);

        // Copying the first signature. Note, that we need an offset of 0x20 
        // since it is where the length of the `_fullSignature` is stored
        assembly {
            let r := mload(add(_fullSignature, 0x20))
			let s := mload(add(_fullSignature, 0x40))
			let v := and(mload(add(_fullSignature, 0x41)), 0xff)

            mstore(add(signature1, 0x20), r)
            mstore(add(signature1, 0x40), s)
            mstore8(add(signature1, 0x60), v)
        }

        // Copying the second signature.
        assembly {
            let r := mload(add(_fullSignature, 0x61))
            let s := mload(add(_fullSignature, 0x81))
            let v := and(mload(add(_fullSignature, 0x82)), 0xff)

            mstore(add(signature2, 0x20), r)
            mstore(add(signature2, 0x40), s)
            mstore8(add(signature2, 0x60), v)
        }
    }

    fallback() external {
        // fallback of default account shouldn't be called by bootloader under no circumstances
        assert(msg.sender != BOOTLOADER_FORMAL_ADDRESS);

        // If the contract is called directly, behave like an EOA
    }

    receive() external payable {
        // If the contract is called directly, behave like an EOA.
        // Note, that is okay if the bootloader sends funds with no calldata as it may be used for refunds/operator payments
    }
}
```

:::tip
The `onlyBootloader` modifier ensures that only the [bootloader](../developer-guides/system-contracts.md#bootloader) calls the `validateTransaction`/`executeTransaction`/`payForTransaction`/`prepareForPaymaster` functions.
:::

The `executeTransactionFromOutside` function allows external users to initiate transactions from this account. We implement it by calling `validateTransaction` and `executeTransaction`.

In addition, `checkValidECDSASignatureFormat` and `extractECDSASignature` are helper functions for the `isValidSignature` implementation.

### Signature validation

We import OpenZeppelin's `ECDSA` library to use for signature validation.

```solidity
// Used for signature validation
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
```

Since we are building a two-account multisig, we pass the owners' addresses to the constructor and save their state variables.

```solidity
// state variables for account owners
address public owner1;
address public owner2;


constructor(address _owner1, address _owner2) {
    owner1 = _owner1;
    owner2 = _owner2;
}
```

To validate the signature we have to implement the following:

- Check if the length of the received signature is correct.
- Extract the two signatures from the received multisig using the helper function `extractECDSASignature`.
- Check if both signatures are valid using the helper function `checkValidECDSASignatureFormat`.
- Extract the addresses from the transaction hash and each signature using the `ECDSA.recover` function.
- Check if the addresses extracted match with the owners of the account.
- Return the `EIP1271_SUCCESS_RETURN_VALUE` value on success or `bytes4(0)` if validation fails.

Below is the full implementation of the `isValidSignature` function:

```solidity
function isValidSignature(bytes32 _hash, bytes memory _signature)
    public
    view
    override
    returns (bytes4 magic)
{
    magic = EIP1271_SUCCESS_RETURN_VALUE;

    if (_signature.length != 130) {
        // Signature is invalid, but we need to proceed with the signature verification as usual
        // in order for the fee estimation to work correctly
        _signature = new bytes(130);
        
        // Making sure that the signatures look like a valid ECDSA signature and are not rejected rightaway
        // while skipping the main verification process.
        _signature[64] = bytes1(uint8(27));
        _signature[129] = bytes1(uint8(27));
    }

    (bytes memory signature1, bytes memory signature2) = extractECDSASignature(_signature);

    if(!checkValidECDSASignatureFormat(signature1) || !checkValidECDSASignatureFormat(signature2)) {
        magic = bytes4(0);
    }

    address recoveredAddr1 = ECDSA.recover(_hash, signature1);
    address recoveredAddr2 = ECDSA.recover(_hash, signature2);

    // Note, that we should abstain from using the require here in order to allow for fee estimation to work
    if(recoveredAddr1 != owner1 || recoveredAddr2 != owner2) {
        magic = bytes4(0);
    }
}
```

### Transaction validation

The transaction validation process is responsible for validating the signature of the transaction and incrementing the nonce. 

:::info
- There are some [limitations](../developer-guides/aa.md#limitations-of-the-verification-step) on this function.
:::

To increment the nonce, use the `incrementNonceIfEquals` function from the `NONCE_HOLDER_SYSTEM_CONTRACT` system contract. It takes the nonce of the transaction and checks whether it is the same as the provided one. If not, the transaction reverts; otherwise, the nonce increases.

Even though the requirements above mean the accounts only touch their own storage slots, accessing your nonce in the `NONCE_HOLDER_SYSTEM_CONTRACT` is a [whitelisted](../developer-guides/aa.md#extending-the-set-of-slots-that-belong-to-a-user) case, since it behaves in the same way as your storage, it just happens to be in another contract. 

To call the `NONCE_HOLDER_SYSTEM_CONTRACT`, we add the following import:

```solidity
// Access zkSync system contracts for nonce validation via NONCE_HOLDER_SYSTEM_CONTRACT
import "@matterlabs/zksync-contracts/l2/system-contracts/Constants.sol";
```

:::info
- The non-view functions of the `NONCE_HOLDER_SYSTEM_CONTRACT` are called if the `isSystem` flag is on.
- Use the [systemCallWithPropagatedRevert](https://github.com/matter-labs/v2-testnet-contracts/blob/main/l2/system-contracts/libraries/SystemContractsCaller.sol#L75) function of the `SystemContractsCaller` library.
- Import this library also:
    ```solidity
    // to call non-view function of system contracts
    import "@matterlabs/zksync-contracts/l2/system-contracts/libraries/SystemContractsCaller.sol";
    ```
:::

Use the `TransactionHelper` library, as imported above with `using TransactionHelper for Transaction;` to get the transaction hash that should be signed. You can also implement your own signature scheme and use a different commitment for signing the transaction, but in this example, we use the hash provided by this library.

Finally, the `_validateTransaction` function has to return the constant `ACCOUNT_VALIDATION_SUCCESS_MAGIC` if the validation is successful, or an empty value `bytes4(0)` if it fails.

Here is the full implementation for the `_validateTransaction` function:

```solidity

function _validateTransaction(
    bytes32 _suggestedSignedHash,
    Transaction calldata _transaction
) internal returns (bytes4 magic) {
    // Incrementing the nonce of the account.
    // Note, that reserved[0] by convention is currently equal to the nonce passed in the transaction
    SystemContractsCaller.systemCallWithPropagatedRevert(
        uint32(gasleft()),
        address(NONCE_HOLDER_SYSTEM_CONTRACT),
        0,
        abi.encodeCall(INonceHolder.incrementMinNonceIfEquals, (_transaction.nonce))
    );

    bytes32 txHash;
    // While the suggested signed hash is usually provided, it is generally
    // not recommended to rely on it to be present, since in the future
    // there may be tx types with no suggested signed hash.
    if (_suggestedSignedHash == bytes32(0)) {
        txHash = _transaction.encodeHash();
    } else {
        txHash = _suggestedSignedHash;
    }

    // The fact there is are enough balance for the account
    // should be checked explicitly to prevent user paying for fee for a
    // transaction that wouldn't be included on Ethereum.
    uint256 totalRequiredBalance = _transaction.totalRequiredBalance();
    require(totalRequiredBalance <= address(this).balance, "Not enough balance for fee + value");

    if (isValidSignature(txHash, _transaction.signature) == EIP1271_SUCCESS_RETURN_VALUE) {
        magic = ACCOUNT_VALIDATION_SUCCESS_MAGIC;
    } else {
        magic = bytes4(0);
    }
}
```

### Paying fees for the transaction

This section explains the `payForTransaction` function. 

The `TransactionHelper` library already provides us with the `payToTheBootloader` function, that sends `_transaction.maxFeePerGas * _transaction.gasLimit` ETH to the bootloader. The implementation is straightforward:

```solidity
function payForTransaction(
        bytes32,
        bytes32,
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        bool success = _transaction.payToTheBootloader();
        require(success, "Failed to pay the fee to the operator");
    }
```

### Implementing paymaster support

While the account abstraction protocol enables arbitrary actions when interacting with the paymasters, there are some [common patterns](../developer-guides/aa.md#built-in-paymaster-flows) with built-in support for EOAs. Unless you want to implement or restrict some specific paymaster use cases for your account, it is better to keep it consistent with EOAs. 

The `TransactionHelper` library provides the `processPaymasterInput` which does exactly that: processes the paymaster parameters the same it's done in EOAs.

```solidity

function prepareForPaymaster(
        bytes32, // _txHash
        bytes32, // _suggestedSignedHash
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        _transaction.processPaymasterInput();
    }
```

### Transaction execution

To implementing transaction execution, extract the transaction data and execute it:

```solidity
function _executeTransaction(Transaction calldata _transaction) internal {
    uint256 to = _transaction.to;
    // By convention, the `reserved[1]` field is msg.value
    uint256 value = _transaction.reserved[1];
    bytes memory data = _transaction.data;

    bool success;
    // execute transaction
    assembly {
        success := call(gas(), to, value, add(data, 0x20), mload(data), 0, 0)
    }

    // Return value required for the transaction to be correctly processed by the server.
    require(success);
}
```

Calling `ContractDeployer` is only possible by explicitly using the `isSystem` call flag. 

```solidity
function _executeTransaction(Transaction calldata _transaction) internal {
    address to = address(uint160(_transaction.to));
    uint128 value = Utils.safeCastToU128(_transaction.value);
    bytes memory data = _transaction.data;

    if (to == address(DEPLOYER_SYSTEM_CONTRACT)) {
        uint32 gas = Utils.safeCastToU32(gasleft());

        // Note, that the deployer contract can only be called
        // with a "systemCall" flag.
        SystemContractsCaller.systemCallWithPropagatedRevert(gas, to, value, data);
    } else {
        bool success;
        assembly {
            success := call(gas(), to, value, add(data, 0x20), mload(data), 0, 0)
        }
        require(success);
    }
}
```

:::info
- Whether or not the operator considers the transaction successful depends on whether the call to `executeTransactions` is successful. 
- Therefore, it is highly recommended to put `require(success)` for the transaction, so that users get the best UX.
:::

### Full code of the TwoUserMultisig contract

1. Create a file `TwoUserMultisig.sol` in the `contracts` folder and copy/paste the code below into it.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.16;

import "@matterlabs/zksync-contracts/l2/system-contracts/interfaces/IAccount.sol";
import "@matterlabs/zksync-contracts/l2/system-contracts/libraries/TransactionHelper.sol";

import "@openzeppelin/contracts/interfaces/IERC1271.sol";

// Used for signature validation
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

// Access zkSync system contracts for nonce validation via NONCE_HOLDER_SYSTEM_CONTRACT
import "@matterlabs/zksync-contracts/l2/system-contracts/Constants.sol";
// to call non-view function of system contracts
import "@matterlabs/zksync-contracts/l2/system-contracts/libraries/SystemContractsCaller.sol";

contract TwoUserMultisig is IAccount, IERC1271 {
    // to get transaction hash
    using TransactionHelper for Transaction;

    // state variables for account owners
    address public owner1;
    address public owner2;

    bytes4 constant EIP1271_SUCCESS_RETURN_VALUE = 0x1626ba7e;

    modifier onlyBootloader() {
        require(
            msg.sender == BOOTLOADER_FORMAL_ADDRESS,
            "Only bootloader can call this function"
        );
        // Continue execution if called from the bootloader.
        _;
    }

    constructor(address _owner1, address _owner2) {
        owner1 = _owner1;
        owner2 = _owner2;
    }

    function validateTransaction(
        bytes32,
        bytes32 _suggestedSignedHash,
        Transaction calldata _transaction
    ) external payable override onlyBootloader returns (bytes4 magic) {
        return _validateTransaction(_suggestedSignedHash, _transaction);
    }

    function _validateTransaction(
        bytes32 _suggestedSignedHash,
        Transaction calldata _transaction
    ) internal returns (bytes4 magic) {
        // Incrementing the nonce of the account.
        // Note, that reserved[0] by convention is currently equal to the nonce passed in the transaction
        SystemContractsCaller.systemCallWithPropagatedRevert(
            uint32(gasleft()),
            address(NONCE_HOLDER_SYSTEM_CONTRACT),
            0,
            abi.encodeCall(INonceHolder.incrementMinNonceIfEquals, (_transaction.nonce))
        );

        bytes32 txHash;
        // While the suggested signed hash is usually provided, it is generally
        // not recommended to rely on it to be present, since in the future
        // there may be tx types with no suggested signed hash.
        if (_suggestedSignedHash == bytes32(0)) {
            txHash = _transaction.encodeHash();
        } else {
            txHash = _suggestedSignedHash;
        }

        // The fact there is are enough balance for the account
        // should be checked explicitly to prevent user paying for fee for a
        // transaction that wouldn't be included on Ethereum.
        uint256 totalRequiredBalance = _transaction.totalRequiredBalance();
        require(totalRequiredBalance <= address(this).balance, "Not enough balance for fee + value");

        if (isValidSignature(txHash, _transaction.signature) == EIP1271_SUCCESS_RETURN_VALUE) {
            magic = ACCOUNT_VALIDATION_SUCCESS_MAGIC;
        } else {
            magic = bytes4(0);
        }
    }

    function executeTransaction(
        bytes32,
        bytes32,
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        _executeTransaction(_transaction);
    }

    function _executeTransaction(Transaction calldata _transaction) internal {
        address to = address(uint160(_transaction.to));
        uint128 value = Utils.safeCastToU128(_transaction.value);
        bytes memory data = _transaction.data;

        if (to == address(DEPLOYER_SYSTEM_CONTRACT)) {
            uint32 gas = Utils.safeCastToU32(gasleft());

            // Note, that the deployer contract can only be called
            // with a "systemCall" flag.
            SystemContractsCaller.systemCallWithPropagatedRevert(gas, to, value, data);
        } else {
            bool success;
            assembly {
                success := call(gas(), to, value, add(data, 0x20), mload(data), 0, 0)
            }
            require(success);
        }
    }

    function executeTransactionFromOutside(Transaction calldata _transaction)
        external
        payable
    {
        _validateTransaction(bytes32(0), _transaction);
        _executeTransaction(_transaction);
    }

    function isValidSignature(bytes32 _hash, bytes memory _signature)
        public
        view
        override
        returns (bytes4 magic)
    {
        magic = EIP1271_SUCCESS_RETURN_VALUE;

        if (_signature.length != 130) {
            // Signature is invalid anyway, but we need to proceed with the signature verification as usual
            // in order for the fee estimation to work correctly
            _signature = new bytes(130);
            
            // Making sure that the signatures look like a valid ECDSA signature and are not rejected rightaway
            // while skipping the main verification process.
            _signature[64] = bytes1(uint8(27));
            _signature[129] = bytes1(uint8(27));
        }

        (bytes memory signature1, bytes memory signature2) = extractECDSASignature(_signature);

        if(!checkValidECDSASignatureFormat(signature1) || !checkValidECDSASignatureFormat(signature2)) {
            magic = bytes4(0);
        }

        address recoveredAddr1 = ECDSA.recover(_hash, signature1);
        address recoveredAddr2 = ECDSA.recover(_hash, signature2);

        // Note, that we should abstain from using the require here in order to allow for fee estimation to work
        if(recoveredAddr1 != owner1 || recoveredAddr2 != owner2) {
            magic = bytes4(0);
        }
    }

    // This function verifies that the ECDSA signature is both in correct format and non-malleable
    function checkValidECDSASignatureFormat(bytes memory _signature) internal pure returns (bool) {
        if(_signature.length != 65) {
            return false;
        }

        uint8 v;
		bytes32 r;
		bytes32 s;
		// Signature loading code
		// we jump 32 (0x20) as the first slot of bytes contains the length
		// we jump 65 (0x41) per signature
		// for v we load 32 bytes ending with v (the first 31 come from s) then apply a mask
		assembly {
			r := mload(add(_signature, 0x20))
			s := mload(add(_signature, 0x40))
			v := and(mload(add(_signature, 0x41)), 0xff)
		}
		if(v != 27 && v != 28) {
            return false;
        }

		// EIP-2 still allows signature malleability for ecrecover(). Remove this possibility and make the signature
        // unique. Appendix F in the Ethereum Yellow paper (https://ethereum.github.io/yellowpaper/paper.pdf), defines
        // the valid range for s in (301): 0 < s < secp256k1n ÷ 2 + 1, and for v in (302): v ∈ {27, 28}. Most
        // signatures from current libraries generate a unique signature with an s-value in the lower half order.
        //
        // If your library generates malleable signatures, such as s-values in the upper range, calculate a new s-value
        // with 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141 - s1 and flip v from 27 to 28 or
        // vice versa. If your library also generates signatures with 0/1 for v instead 27/28, add 27 to v to accept
        // these malleable signatures as well.
        if(uint256(s) > 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0) {
            return false;
        }

        return true;
    }
    
    function extractECDSASignature(bytes memory _fullSignature) internal pure returns (bytes memory signature1, bytes memory signature2) {
        require(_fullSignature.length == 130, "Invalid length");

        signature1 = new bytes(65);
        signature2 = new bytes(65);

        // Copying the first signature. Note, that we need an offset of 0x20 
        // since it is where the length of the `_fullSignature` is stored
        assembly {
            let r := mload(add(_fullSignature, 0x20))
			let s := mload(add(_fullSignature, 0x40))
			let v := and(mload(add(_fullSignature, 0x41)), 0xff)

            mstore(add(signature1, 0x20), r)
            mstore(add(signature1, 0x40), s)
            mstore8(add(signature1, 0x60), v)
        }

        // Copying the second signature.
        assembly {
            let r := mload(add(_fullSignature, 0x61))
            let s := mload(add(_fullSignature, 0x81))
            let v := and(mload(add(_fullSignature, 0x82)), 0xff)

            mstore(add(signature2, 0x20), r)
            mstore(add(signature2, 0x40), s)
            mstore8(add(signature2, 0x60), v)
        }
    }

    function payForTransaction(
        bytes32,
        bytes32,
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        bool success = _transaction.payToTheBootloader();
        require(success, "Failed to pay the fee to the operator");
    }

    function prepareForPaymaster(
        bytes32, // _txHash
        bytes32, // _suggestedSignedHash
        Transaction calldata _transaction
    ) external payable override onlyBootloader {
        _transaction.processPaymasterInput();
    }

    fallback() external {
        // fallback of default account shouldn't be called by bootloader under no circumstances
        assert(msg.sender != BOOTLOADER_FORMAL_ADDRESS);

        // If the contract is called directly, behave like an EOA
    }

    receive() external payable {
        // If the contract is called directly, behave like an EOA.
        // Note, that is okay if the bootloader sends funds with no calldata as it may be used for refunds/operator payments
    }
}
```

## The factory

1. Create a new Solidity file in the `contracts` folder called `AAFactory.sol`. 

The contract is a factory that deploys the accounts. 

:::warning
- To deploy the multisig smart contract, it is necessary to interact with the `DEPLOYER_SYSTEM_CONTRACT` and call the `create2Account` function.
- If the code doesn't do this, you may see errors like `Validation revert: Sender is not an account`.
:::

2. Copy/paste the following code into the file.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.16;

import "@matterlabs/zksync-contracts/l2/system-contracts/Constants.sol";
import "@matterlabs/zksync-contracts/l2/system-contracts/libraries/SystemContractsCaller.sol";

contract AAFactory {
    bytes32 public aaBytecodeHash;

    constructor(bytes32 _aaBytecodeHash) {
        aaBytecodeHash = _aaBytecodeHash;
    }

    function deployAccount(
        bytes32 salt,
        address owner1,
        address owner2
    ) external returns (address accountAddress) {
        (bool success, bytes memory returnData) = SystemContractsCaller
            .systemCallWithReturndata(
                uint32(gasleft()),
                address(DEPLOYER_SYSTEM_CONTRACT),
                uint128(0),
                abi.encodeCall(
                    DEPLOYER_SYSTEM_CONTRACT.create2Account,
                    (salt, aaBytecodeHash, abi.encode(owner1, owner2), IContractDeployer.AccountAbstractionVersion.Version1)
                )
            );
        require(success, "Deployment failed");

        (accountAddress) = abi.decode(returnData, (address));
    }
}
```

It's worth remembering that, on zkSync Era, [contract deployments](../building-on-zksync/contracts/contract-deployment.md)  are not done via bytecode, but via bytecode hash. The bytecode itself is passed to the operator via the `factoryDeps` field. Note that the `_aaBytecodeHash` must be formed in the following manner:

- Firstly, it is hashed with sha256.
- Then, the first two bytes are replaced with the length of the bytecode in 32-byte words.

This functionality is built into the SDK.

## Deploy the factory

::: tip
Make sure you deposit funds on zkSync Era using the [zkSync Portal](https://goerli.portal.zksync.io/bridge) before running the deployment script.
:::

1. In the `deploy` folder, create the file `deploy-factory.ts` and copy/paste the following code, replacing `<WALLET_PRIVATE_KET>` with your private key.

```ts
import { utils, Wallet } from 'zksync-web3';
import * as ethers from 'ethers';
import { HardhatRuntimeEnvironment } from 'hardhat/types';
import { Deployer } from '@matterlabs/hardhat-zksync-deploy';

export default async function (hre: HardhatRuntimeEnvironment) {
  const wallet = new Wallet('<WALLET_PRIVATE_KEY>');
  const deployer = new Deployer(hre, wallet);
  const factoryArtifact = await deployer.loadArtifact('AAFactory');
  const aaArtifact = await deployer.loadArtifact('TwoUserMultisig');

  // Getting the bytecodeHash of the account
  const bytecodeHash = utils.hashBytecode(aaArtifact.bytecode);

  const factory = await deployer.deploy(
    factoryArtifact,
    [bytecodeHash],
    undefined,
    [
      // Since the factory requires the code of the multisig to be available,
      // we should pass it here as well.
      aaArtifact.bytecode,
    ]
  );

  console.log(`AA factory address: ${factory.address}`);
}
```

2. From the project root, compile and deploy the contracts.

```sh
yarn hardhat compile
yarn hardhat deploy-zksync --script deploy-factory.ts
```

The output should look like this:

```txt
AA factory address: 0x9db333Cb68Fb6D317E3E415269a5b9bE7c72627Ds
```

Note that the address will be different for each run.

## Working with accounts

### Deploying an account

Now, let's deploy an account and use it to initiate a new transaction. 

:::warning
This section assumes you have an EOA account with sufficient funds on zkSync Era.
:::

1. In the `deploy` folder, create a file called `deploy-multisig.ts`.

The call to the `deployAccount` function deploys the AA.

```ts
import { utils, Wallet, Provider, EIP712Signer, types } from 'zksync-web3';
import * as ethers from 'ethers';
import { HardhatRuntimeEnvironment } from 'hardhat/types';

// Put the address of your AA factory
const AA_FACTORY_ADDRESS = '<FACTORY-ADDRESS>';

export default async function (hre: HardhatRuntimeEnvironment) {
  const provider = new Provider('https://testnet.era.zksync.dev');
  const wallet = new Wallet('<WALLET-PRIVATE-KEY>').connect(provider);
  const factoryArtifact = await hre.artifacts.readArtifact('AAFactory');

  const aaFactory = new ethers.Contract(
    AA_FACTORY_ADDRESS,
    factoryArtifact.abi,
    wallet
  );

  // The two owners of the multisig
  const owner1 = Wallet.createRandom();
  const owner2 = Wallet.createRandom();

  // For the simplicity of the tutorial, we will use zero hash as salt
  const salt = ethers.constants.HashZero;

  const tx = await aaFactory.deployAccount(
    salt,
    owner1.address,
    owner2.address
  );
  await tx.wait();

  // Getting the address of the deployed contract
  const abiCoder = new ethers.utils.AbiCoder();
  const multisigAddress = utils.create2Address(
    AA_FACTORY_ADDRESS,
    await aaFactory.aaBytecodeHash(),
    salt,
    abiCoder.encode(['address', 'address'], [owner1.address, owner2.address])
  );
  console.log(`Multisig deployed on address ${multisigAddress}`);
}
```

:::tip
- zkSync has different address derivation rules from Ethereum. 
- Always use the `createAddress` and `create2Address` utility functions of the `zksync-web3` SDK.
:::

### Start a transaction from the account

Before the deployed account can submit transactions, we need to deposit some ETH to it for the transaction fees.

```ts
  await (
    await wallet.sendTransaction({
      to: multisigAddress,
      // You can increase the amount of ETH sent to the multisig
      value: ethers.utils.parseEther('0.006'),
    })
  ).wait();
```

Now we can try to deploy a new multisig; the initiator of the transaction will be our deployed account from the previous part.

```ts
  let aaTx = await aaFactory.populateTransaction.deployAccount(
    salt,
    Wallet.createRandom().address,
    Wallet.createRandom().address
  );
```

Then, we need to fill all the transaction fields:

```ts
  const gasLimit = await provider.estimateGas(aaTx);
  const gasPrice = await provider.getGasPrice();

  aaTx = {
    ...aaTx,
    from: multisigAddress,
    gasLimit: gasLimit,
    gasPrice: gasPrice,
    chainId: (await provider.getNetwork()).chainId,
    nonce: await provider.getTransactionCount(multisigAddress),
    type: 113,
    customData: {
      gasPerPubdata: utils.DEFAULT_GAS_PER_PUBDATA_LIMIT,
    } as types.Eip712Meta,
    value: ethers.BigNumber.from(0),
  };
```

::: tip Note on gasLimit
- Currently, we expect the `l2gasLimit` to cover both the verification and the execution steps. 
- Currently, the gas returned by `estimateGas` is `execution_gas + 20000`, where `20000` is roughly equal to the overhead needed for the defaultAA to have both fee charged and the signature verified. 
- In the case that your AA has an expensive verification step, you should add some constant to the `l2gasLimit`.
:::

Then, we need to sign the transaction and provide the `aaParamas` in the customData of the transaction.

```ts
  const signedTxHash = EIP712Signer.getSignedDigest(aaTx);

  const signature = ethers.utils.concat([
    // Note, that `signMessage` wouldn't work here, since we don't want
    // the signed hash to be prefixed with `\x19Ethereum Signed Message:\n`
    ethers.utils.joinSignature(owner1._signingKey().signDigest(signedTxHash)),
    ethers.utils.joinSignature(owner2._signingKey().signDigest(signedTxHash)),
  ]);

  aaTx.customData = {
    ...aaTx.customData,
    customSignature: signature,
  };
```

Finally, we are ready to send the transaction:

```ts
  console.log(
    `The multisig's nonce before the first tx is ${await provider.getTransactionCount(
      multisigAddress
    )}`
  );
  const sentTx = await provider.sendTransaction(utils.serialize(aaTx));
  await sentTx.wait();

  // Checking that the nonce for the account has increased
  console.log(
    `The multisig's nonce after the first tx is ${await provider.getTransactionCount(
      multisigAddress
    )}`
  );
```

### Full example

1. Copy/paste the following code into the deployment file, replacing the `<FACTORY-ADDRESS>` and private key `<WALLET-PRIVATE-KEY>` placeholders with the relevant data.

```ts
import { utils, Wallet, Provider, EIP712Signer, types } from 'zksync-web3';
import * as ethers from 'ethers';
import { HardhatRuntimeEnvironment } from 'hardhat/types';

// Put the address of your AA factory
const AA_FACTORY_ADDRESS = '<FACTORY-ADDRESS>';

export default async function (hre: HardhatRuntimeEnvironment) {
  const provider = new Provider('https://testnet.era.zksync.dev');
  const wallet = new Wallet('<WALLET-PRIVATE-KEY>').connect(provider);
  const factoryArtifact = await hre.artifacts.readArtifact('AAFactory');

  const aaFactory = new ethers.Contract(
    AA_FACTORY_ADDRESS,
    factoryArtifact.abi,
    wallet
  );

  // The two owners of the multisig
  const owner1 = Wallet.createRandom();
  const owner2 = Wallet.createRandom();

  // For the simplicity of the tutorial, we will use zero hash as salt
  const salt = ethers.constants.HashZero;

  const tx = await aaFactory.deployAccount(
    salt,
    owner1.address,
    owner2.address
  );
  await tx.wait();

  // Getting the address of the deployed contract
  const abiCoder = new ethers.utils.AbiCoder();
  const multisigAddress = utils.create2Address(
    AA_FACTORY_ADDRESS,
    await aaFactory.aaBytecodeHash(),
    salt,
    abiCoder.encode(['address', 'address'], [owner1.address, owner2.address])
  );
  console.log(`Multisig deployed on address ${multisigAddress}`);

  await (
    await wallet.sendTransaction({
      to: multisigAddress,
      // You can increase the amount of ETH sent to the multisig
      value: ethers.utils.parseEther('0.003'),
    })
  ).wait();

  let aaTx = await aaFactory.populateTransaction.deployAccount(
    salt,
    Wallet.createRandom().address,
    Wallet.createRandom().address
  );

  const gasLimit = await provider.estimateGas(aaTx);
  const gasPrice = await provider.getGasPrice();

  aaTx = {
    ...aaTx,
    from: multisigAddress,
    gasLimit: gasLimit,
    gasPrice: gasPrice,
    chainId: (await provider.getNetwork()).chainId,
    nonce: await provider.getTransactionCount(multisigAddress),
    type: 113,
    customData: {
      gasPerPubdata: utils.DEFAULT_GAS_PER_PUBDATA_LIMIT,
    } as types.Eip712Meta,
    value: ethers.BigNumber.from(0),
  };
  const signedTxHash = EIP712Signer.getSignedDigest(aaTx);

  const signature = ethers.utils.concat([
    // Note, that `signMessage` wouldn't work here, since we don't want
    // the signed hash to be prefixed with `\x19Ethereum Signed Message:\n`
    ethers.utils.joinSignature(owner1._signingKey().signDigest(signedTxHash)),
    ethers.utils.joinSignature(owner2._signingKey().signDigest(signedTxHash)),
  ]);

  aaTx.customData = {
    ...aaTx.customData,
    customSignature: signature,
  };

  console.log(
    `The multisig's nonce before the first tx is ${await provider.getTransactionCount(
      multisigAddress
    )}`
  );
  const sentTx = await provider.sendTransaction(utils.serialize(aaTx));
  await sentTx.wait();

  // Checking that the nonce for the account has increased
  console.log(
    `The multisig's nonce after the first tx is ${await provider.getTransactionCount(
      multisigAddress
    )}`
  );
}
```

2. Run the script from the `deploy` folder.

```sh
yarn hardhat deploy-zksync --script deploy-multisig.ts
```

The output should look something like this:

```txt
Multisig deployed on address 0xCEBc59558938bccb43A6C94769F87bBdb770E956
The multisig's nonce before the first tx is 0
The multisig's nonce after the first tx is 1
```

::: tip
If you get an error `Not enough balance to cover the fee.`, try increasing the amount of ETH sent to the multisig wallet so it has enough funds to pay for the transaction fees.
:::

## Learn more

- To learn more about L1->L2 interaction on zkSync, check out the [documentation](../developer-guides/bridging/l1-l2.md).
- To learn more about the `zksync-web3` SDK, check out its [documentation](../../api/js).
- To learn more about the zkSync hardhat plugins, check out their [documentation](../../api/hardhat).
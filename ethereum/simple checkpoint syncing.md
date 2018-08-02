# Simple Checkpoint Syncing

The purpose of this version checkpoint syncing is only to **replace the hard-coded checkpoint information in the code**, making the checkpoint updating more flexible.

**Note: This version checkpoint syncing is still centralized.**

## Pre-requisites

* Deploy a checkpoint registrar contract on the mainnet.

the code of contract should looks like:

```
contract Registrar {
    modifier OnlyAuthorized() {
        require(admins[msg.sender] > 0);
        _;
    }
    
    event NewCheckpointEvent(uint indexed index, bytes32 checkpointHash,  bytes32 signature);
 
    event AddAdminEvent(address addr, string description, bytes32 signature);

    event RemoveAdminEvent(address addr, string description, bytes32 signature);
    
    constructor(address[] _adminlist) public {
        for (uint i = 0; i < _adminlist.length; i++) {
            admins[_adminlist[i]] = 1;
            adminList.push(_adminlist[i]);
        }
    }
    
    function SetCheckpoint(uint _sectionIndex, bytes32 _hash, bytes32 _signature) OnlyAuthorized public returns(bool) {
        // Some checking logic here.
        // ...
        checkpoints[_sectionIndex] = _hash;
        emit NewCheckpointEvent(_sectionIndex, _hash, _signature);
    }
    
    function GetCheckpoint(uint _sectionIndex) view public returns(bytes32) {
        return checkpoints[_sectionIndex];
    }
    // Some other admin management functions
}
```

* Hardcode contract address in the codebase.
* Register stable checkpoint by **ethereum foundation admins** at a specified rate(32768 blocks).

This action just like updating hard code checkpoint before release by core developers.

**The most important thing about simple checkpoint syncing is to protect all admin private key files. **

If one of the admins' private key file is lost, we have to delete this admin as soon as possible.

Or we can redeploy a new contract and release a new geth as soon as possible. Otherwise simple checkpoint syncing will not provide any security for the light client.

## Simple checkpoint syncing

**Light client side**

* Select a server peer which has the highest advertised total difficulty as the target peer.

* Request the last block header which covered by server peer's announced stable checkpoint and set as the current head header temporarily.

* Sync the last a few thousands blocks by `light syncing`.

* Filter the `NewCheckpointEvent` with specified checkpoint index as topic in the block range

   `[(index + 1) * CheckpointSize + CheckpointProcessConfirm, Current Head]`.

  * If no event is found, rollback and drop the malicious peer;
  * If checkpoint event is found, but the signer address recovered by `signature` is not an admin address, rollback and drop the maclious peer;
  * If checkpoint event is found, but the checkpoint hash included in the event is not equal with advertised checkpoint hash, rollback and drop the maclious peer;
  * Otherwise, the server peer is a honest peer, finish syncing;

Besides the light client also needs to listen contract `AddAdminEvent` and `RemoveAdminEvent`, update local admins once this event is stable enough.

**Server side**

* Listen the checkpoint registrar contract and update local stable checkpoint if the announced checkpoint is equal with local calculation result and it is stable enough;
* Advertise the local stable checkpoint in the handshake package;

**Admin side**

* Sign for checkpoint with the private key and register checkpoint after `(index+1)*32768+256 blocks`;
* Can modify the checkpoint in the block range `[(index+1)*32768+256, (index+1)*32768+8192]`; 
* Sign for admin management operations and manage admin if needed.

## Security model

For the simple checkpoint syncing, the security is guaranteed by ECDSA.

1. Contract address and initial target admin addresses are hardcoded in the codebase.
2. Light client only accept checkpoint announcement given by EF admins.
3. If we can find a checkpoint equal with the advertised checkpoint, it means the advertised checkpoint is valid and we can sync based on that just like a new genesis block.

## Weakness

1. Admins private key files are the heart of security. 

## Future trustless checkpoint syncing

For the trustless checkpoint syncing, zsolt already given out a [document](https://github.com/zsfelfoldi/ethereum-docs/blob/master/les/tasks/syncing_long.md) with detail description, and we can use this simple checkpoint syncing as an intellectual approach.
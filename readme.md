# A Naive Introduction to NFTs

We are naive. We know nothing about Ethereum, NFTs, anything. What do we do?
When we attend a presentation or read a document about NFTs, we take away confusing 
generalities that don't make sense, and leave us more confused.

In this naive introduction, we'll learn about NFTs by exploring the source code
of the smart contracts directly. We'll just muck through function comments and do our 
best to piece it together. We'll only come up for air and reference documentation and specifications
as needed to make sense of what we're reading.

Let's start by examining some of the NFTs that have recently been mentioned in
the news.

There's [Beeple's artwork](https://onlineonly.christies.com/s/first-open-beeple/beeple-b-1981-1/112924)
which sold for over $69 million dollars. The Christies' website has two odd links

```
wallet address: 0xc6b0562605D35eE710138402B878ffe6F2E23807
smart contract address: 0x2a46f2ffd99e19a89476e2f62270e0a35bbf0756
```

These two terms sound familiar. A wallet is some kind of digital thing that can
send and receive Ether (Ethereum's native currency). A smart contract is
supposed to be really important, but we're not really sure what it is. We skimmed
over the [Ethereum Yellow Paper] and [Mastering Ethereum], but we're still not sure.

It sounds like a smart contract can send and receive Ether, just like a wallet.
And when you send it Ether, some kind of program gets executed on the Ethereum network.

Is that all there is though? How does that make a smart contract a 'contract'? Is it legally
binding? What about an NFT? It's supposed to be non-fungible right, that's part
of the name. Are NFTs smart contracts?  Why is a computer program worth $69 million
dollars?

## Going deeper

We have a lot of questions, and not a lot of answers. But we do have two
very long hexidecimal (base 16 number) strings. Let's pull on that thread a little.
We're going to use one of those websites where you can put in Ethereum addresses and
it will return information on the wallet or smart contract, and see what we can learn.
For our purposes, we'll use [Etherscan](etherscan.io), but any lookup service will do.

We look up Beeple's [smart contract](https://etherscan.io/txs?a=0x2a46f2ffd99e19a89476e2f62270e0a35bbf0756).
Wow, that's a lot of information. It looks like it has had over 80,000 "transactions",
whatever that means. It was created in 2019, when Ether was
$133.54 / ETH. Where's the artwork though? Who owns it?

Ah, there's a link for the smart contract [source code](https://etherscan.io/address/0x2a46f2ffd99e19a89476e2f62270e0a35bbf0756#code). Let's poke around in there a bit
and see what we can guess at.

Reading the source code together, from the top
```
pragma solidity ^0.4.21;
```

Ok, we've heard of that. Solidity is supposed to be one of the languages you can
write smart contracts in. Supposedly the language doesn't matter because smart
contracts are compiled and executed on something called the 'Ethereum Virtual Machine',
but we know Solidity is popular.
```
library SafeMath {
...
  function mul(uint256 a, uint256 b) internal pure returns (uint256 c)
  function div(uint256 a, uint256 b) internal pure returns (uint256)
  function sub(uint256 a, uint256 b) internal pure returns (uint256)
  function add(uint256 a, uint256 b) internal pure returns (uint256 c)
```

Interesting. This looks like C or Go, or another imperative programming language.
You can have functions, and they have inputs and outputs. uint256 is probably
a 256 bit unsigned integer type (guessing from playing around with C). 256/8 = 32,
so it holds up to 32 bytes of data, or 64 hexidecimal digits.

Oh interesting, it looks like you can have interfaces too.

```
/**
 * Interface to the digital media store external contract that is
 * responsible for storing the common digital media and collection data.
 * This allows for new token contracts to be deployed and continue to reference
 * the digital media and collection data.
 */
contract ApprovedCreatorRegistryInterface {
```

Wait, so is there a *different* smart contract that stores the art and ownership data?

Ah, this part looks important. How to define ownership

```
/**
 * @title Ownable
 * @dev The Ownable contract has an owner address, and provides basic authorization control
 * functions, this simplifies the implementation of "user permissions".
 */
contract Ownable {
```

Let's pause a second. So far we've seen two type definitions, 'library' and 'contract'.
Can we get more information on what those mean? Let's [read the docs](https://docs.soliditylang.org/en/v0.8.4/contracts.html).

```
Contracts in Solidity are similar to classes in object-oriented languages. They contain persistent data in state variables, and functions that can modify these variables. Calling a function on a different contract (instance) will perform an EVM function call and thus switch the context such that state variables in the calling contract are inaccessible. A contract and its functions need to be called for anything to happen. There is no “cron” concept in Ethereum to call a function at a particular event automatically.
```
Ok, makes sense. We're going to just think of 'contracts' as classes and objects then.

Returning to where we were in the Beeple NFT source. Ok now we see a contract definition with an
'is' predicate.

```
/**
 * A special control class that is used to configure and manage a token contract's
 * different digital media store versions.
 *
 * Older versions of token contracts had the ability to increment the digital media's
 * print edition in the media store, which was necessary in the early stages to provide
 * upgradeability and flexibility.
 *
 * New verions will get rid of this ability now that token contract logic
 * is more stable and we've built in burn capabilities.  
 *
 * In order to support the older tokens, we need to be able to look up the appropriate digital
 * media store associated with a given digital media id on the latest token contract.
 */
contract MediaStoreVersionControl is Pausable {
```

There are multiple references to a 'token' here, but we're still not quite sure
what that means.

Skimming, skimming. Ah. We found our first thing that feels 'NFT-ish' on line 450.
```
/**
 * A special control class that's used to help enforce that a DigitalMedia contract
 * will service only a single creator's address.  This is used when deploying a
 * custom token contract owned and managed by a single creator.
 */
contract SingleCreatorControl {
```

So maybe the NFT creator took normal token creating software, and just tweaked
it with some additional properties so only one person could own it at a time?

```
/**
 * Sets the single creator associated with this contract.  This function
 * can only ever be called once, and should ideally be called at the point
 * of constructing the smart contract.
 */
function setSingleCreator(address _singleCreatorAddress) internal {
    require(singleCreatorAddress == address(0), "Single creator address already set.");
    singleCreatorAddress = _singleCreatorAddress;
}

```
Huh that's interesting. Let's make a note here and come back to this. Did Beeple call this function to
associate himself as the single creator? Can we track down that call somewhere?
How exactly does it make sure no one else can claim to be the NFT creator?
Actually, how exactly does a third-party even make a function call in to a
smart contract?

Ah, we found a standard on line 500!

```
**
 * @title ERC721 Non-Fungible Token Standard basic interface
 * @dev see https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md
 */
contract ERC721Basic {
```

What does ERC721 mean? How do people enforce that this standard is implemented?
Is there a standardization process like ISO or something? Are their IP licenses
associated with this standard? Is it managed by a body, a group of contributors,
something else? Many questions arise.

We'll come back to digging into ERC721 in much more detail later. For now we observe

```
This standard is inspired by the ERC-20 token standard and builds on two years of experience since EIP-20 was created. EIP-20 is insufficient for tracking NFTs because each asset is distinct (non-fungible) whereas each of a quantity of tokens is identical (fungible).
```
Also, there does seem to be some sort of community-driven [standards process](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1.md). If we wanted to make our own smart-contract standard, would we have to follow
this standards process for anyone to take us seriously? We'll bookmark that idea too.

Ok, back to the source code.

```
contract ERC721Basic {
  event Transfer(address indexed _from, address indexed _to, uint256 _tokenId);
  event Approval(address indexed _owner, address indexed _approved, uint256 _tokenId);
  event ApprovalForAll(address indexed _owner, address indexed _operator, bool _approved);
```

These look like basic function definitions to transfer and approve the transfer
of ownership of the NFT. What does 'event' mean? [Back to the docs](
https://docs.soliditylang.org/en/v0.8.4/contracts.html#events)

```
Solidity events give an abstraction on top of the EVM’s logging functionality. Applications can subscribe and listen to these events through the RPC interface of an Ethereum client.

Events are inheritable members of contracts. When you call them, they cause the arguments to be stored in the transaction’s log - a special data structure in the blockchain. These logs are associated with the address of the contract, are incorporated into the blockchain, and stay there as long as a block is accessible (forever as of now, but this might change with Serenity). The Log and its event data is not accessible from within contracts (not even from the contract that created them).

```
Ok. So an event is something that's logged to the blockchain as part of transaction data,
but for some reason, the data is not accessible from within the contract itself.

```
contract ERC721Metadata is ERC721Basic {
  ...
  function tokenURI(uint256 _tokenId) public view returns (string);
}
```
The concept of a 'Token URI (uniform resource indicator)' sounds familiar.
Is that where the contract links to the artwork?

Scrolling through, we see some function definitions that seem to implement
some of the ideas we saw mentioned above, like transfering the NFT.

```
/**
 * @dev Internal function to mint a new token
 * @dev Reverts if the given token ID already exists
 * @param _to The address that will own the minted token
 * @param _tokenId uint256 ID of the token to be minted by the msg.sender
 */
function _mint(address _to, uint256 _tokenId) internal {
  require(_to != address(0));
  addTokenTo(_to, _tokenId);
  emit Transfer(address(0), _to, _tokenId);
}
```
That's really interesting. Is the _mint function the function that is supposed
to create the NFT? We've seen the _-style naming convention before. Is _mint
supposed to be a private function or something like that? Only accessible within a limited
scope within the smart contract? Probably what `@dev Internal function to mint a new token` is getting at
in the function comment.

Wait a minute, what is this on line 1126?
```
bytes4 constant internal InterfaceSignature_ERC721 =
       bytes4(keccak256('name()')) ^
```
Keccak256 sounds crypto-y. It looks like it is a 256-bit cryptographic
hash function that is a cousin of the [SHA-3 hash function](https://en.wikipedia.org/wiki/SHA-3).

Cryptography is full of little details that are hard to keep track of. What do we remember?
A cryptographic hash function takes some data in the form of 0s and 1s, and does stuff to them
to return a fixed length string of 0s and 1s (in this case the returned string has 256 bits).
Also, the returned value should be unique, and it should be really hard to fake an input
to be able to produce a specific hash value. Right? Let's come back to it later and keep reading
the contract. But why is this needed at all for an NFT?

Blah blah blah, utility functions. Ah. Line 1330.
```
* This contract allows for the creation of:
*  1. New Collections
*  2. New DigitalMedia objects
*  3. New DigitalMediaRelease objects
* The primary piece of logic is to ensure that an ERC721 token can
* have a supply and print edition that is enforced by this contract.
```

Wait a minute, how can a smart contract enforce a 'supply and print' edition?
That's intriguing.

More functions describing token creation and 'token burning'.

```
/**
 * Burns a digital media.  Once this function succeeds, this digital media
 * will no longer be able to mint any more tokens.  Existing tokens need to be
 * burned individually though.
 * @param  _digitalMediaId the id of the digital media to burn
 * @param  _caller the address of the caller.
 */
```

So do you create a 'digital media' object, and that's the thing that creates the NFT?

On line 1569
```
/**
 * Validates the an address is allowed to create a digital media on a
 * given collection.  Collections are tied to addresses.
 */
```

So I guess a separate wallet address talks to this smart contract and requests
a token creation?

Ok this is really cool.

```
/**
 * Changes the creator that is approved to printing new tokens and creations.
 * Either the _caller must be the _creator or the _caller must be the existing
 * approvedCreator.
 * @param _caller the address of the caller
 * @param  _creator the address of the current creator
 * @param  _newCreator the address of the new approved creator
 */
```

So we saw at the start of the contract there was a way for Beeple to deem himself
the sole creator of the contract. Here is a connection to that concept of 'approvedCreator'.
That entity is the only one who can make a token.

```
/**
 * Base class that manages the underlying functions of a Digital Media Sale,
 * most importantly the escrow of digital tokens.
 *
 * Manages ensuring that only approved addresses interact with this contract.
 *
 */
contract DigitalMediaSaleBase
```

DigitalMediaSale. Intriguing concept. Oh, it looks like there is some kind of escrow:
```
/**
 * Escrows an ERC-721 token from the seller to this contract.  Assumes that the escrow contract
 * is already approved to make the transfer, otherwise it will fail.
 */
```

How does that work? Is there a waiting period where an NFT purchaser tries to transfer the Ether to buy an
NFT, and then has to wait for sign-off from the artist?

Line 1836. This looks really important. It's like a 'main' method in C. An entrypoint
contract that calls everything else we saw before. Is it conventional to put the entrypoint
at the bottom of a smart contract?
```
/**
 * This is the main driver contract that is used to control and run the service. Funds
 * are managed through this function, underlying contracts are also updated through
 * this contract.
 *
 * This class also exposes a set of creation methods that are allowed to be created
 * by an approved token creator, on behalf of a particular address.  This is meant
 * to simply the creation flow for MakersToken users that aren't familiar with
 * the blockchain.  The ERC721 tokens that are created are still fully compliant,
 * although it is possible for a malicious token creator to mint unwanted tokens
 * on behalf of a creator.  Worst case, the creator can burn those tokens.
 */
```

And we come to the end of the smart contract for Beeple's NFT. That was interesting
A


We learned a lot just from reading the function comments, though we know nothing about Ethereum or NFTs.

Let's summarize what we learned from reading the source code.

### Summary

We learned that Solidity is a language to program smart contracts in, and it looks a lot
like C, Go, or C++. It's a procedural language with some object oriented stuff in it.
The keyword 'contract' is like a class in C++. It has member functions and member variables.

We learned that a certain datatype gets used a lot, uint256. It's a 32-byte thing.
It must be a fundamental datatype for smart contracts.

We learned that Beeple's NFT makes reference to an Ethereum standard, ERC721.
There's a field where the NFT artist can assert themselves as the only address
allowed to mint tokens.

Speaking of tokens, we saw there was a DigitalMedia class. You create a DigitalMedia object,
and then the artist can mint and transfer tokens related to the smart contract.
So tokens seem less like 'things', and more like pieces of data that the NFT
artist can spawn by making certain function calls against the NFT itself, somehow.

We saw there was a field called 'tokenURI', and that gave us a pretty big clue
about where the artwork was: not here. The actual art associated with an NFT
seems most likely to be stored somewhere else, and the smart contract just links to it.
That's really weird right? Think about copyright for a moment. We know
from the Perfect 10 case that [merely hyperlinking is fair use](https://en.wikipedia.org/wiki/Perfect_10,_Inc._v._Amazon.com,_Inc.). How can a person use an NFT to grant
rights to a copyrighted work, if anyone in the world is allowed to link to it?
We'll save this question for later.

We saw some reference to a cryptographic hash function keccak256, though we didn't
quite figure out what it's meant for. Maybe it's used to verify the artwork is valid, by hashing it?
Is a hash of the artwork stored in the smart contract somewhere?

Finally, we learned that Ethereum seems to have a community-supported standards process
that seems similar to Python's PEPs. But there doesn't seem to be any language about
affirmative licensing anywhere. In fact, all the ERC documents like ERC721 reference cc0, as if
the contributors are trying to waive all their rights to any protectible content in the specifications.


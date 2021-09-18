# Miso Random NFT
Miso random NFT is a contract that allows users to redeem a seed ERC20 (the token type sold on Miso) for a randomly selected ERC721 (NFT) using Chainlink VRF (verifiable random function). This contract is designed to allows a batch auction, dutch auction or crowd sale to occur on miso.sushi.com for an ERC20 seed and then the user can optionally reveal that seed. This process is more similar to a blind box or sealed pack for a collector.

Traditionally, NFT seeds have been sold in batches. This sale strategy suffers from a few faults.
- Post seed sale market can not develop
- Reveals are not provably random (can be manipulated)
- Group reveal forces all users to reveal seed simultaneously
- Value of the seed post reveal is changed

These contract work uni-directional (for saftey reasons) but a bi-directional system could also be created that allowed users to draw more than once.

The other portion of this repository is a tool to help with uploading metadata for ERC721s

## Token Data IPFS Uploader
This application will take the NFT image and metadata in the `img` and `metadata` folders respectively and match the name of the `.gif` to the name of the `.json` file and upload and pin the data to the Infura IPFS pinning service.

#### Quick Start
```
npm install
npm start
```
Output file will be `tokenData.json`, data is a json array with each element having an object `imageHash` and `metadataHash` representing the uploaded version of the respective files.



## Audit Notes
- There is a bug in fulfillRandomness. The check be require(to != address(0)), to verify that redeemTokenForNFT indeed recorded msg.sender in requestIdToAddress. It's pretty bad, because, well... it prevents any minting ><.
- I'm not sure that selectToken will also work as intended. The way it's currently implemented, there's always a chance to get to an already used index (assuming there are 10,000 unclaimed token IDs, then this chance is 0.01%). The problem with such a collision is that it'll result in transferFrom reverting, which will, in turn, revert selectToken and thus fulfillRandomness, in which case it might result in the request never being fulfilled, since no oracle will try to fulfill it again (I'm not 100% sure about this, but I recall that this is how the contract between the VRF nodes and consumers was structured in the past), thus Sedonics will never receive their claim.
- FYI, you're still using the Kovan Chainlink VRF Key Hash. Perhaps it also needs to be an immutable state variable? 
- Optimization is disabled. I recommend enabling it and setting the "runs" parameter to 10,000.
- Please don't specify a solidity version range (e.g., "^0.8.0;"), but rather specify an explicit version (e.g., "=0.8.7"). There are many cases of various fixes/changes, even between different minor versions of the same compiler, and it's considered a best practice to specify (and test against) as a specific version. The only exception is libraries, such as OZ, which targetting multiple compiler versions, but this is not the case. In a case of a collision, a trivial solution would be to have a loop and try to keep incrementing the tokenId until an unclaimed index is found.
- Use specific/fixed "@openzeppelin/contracts" and "@chainlink/contracts" versions. Otherwise, it'd be harder to reproduce the build in the future, and the contracts might have unexpected differences. Specifically, use OZ's v.4.3.2. Although it isn't currently used, it has a fix to a critical vulnerability, which would still be a good practice.
- You are calling requestRandomness per-DONA, which will be very expensive (i.e., 1000 LINK, which is $30k). So I'd recommend reconsidering this approach if it's not too late...
- Product: redeemTokenForNFT will consume your full balance (up to the decimal point), so the only way to redeem only part of it will require to split your DONA tokens into separate wallets, which is cumbersome. You can consider allowing Sedonics to specify how many tokens they want to redeem explicitly.
- The deployer of the contract is configured to admin the "owner" and the "steward" role, which means that if you want it to be a multisig (which you should definitely consider), it'd mean that you'd have to deploy the contract from it at the beginning, which is cumbersome. In addition to that, it also means that you can't revoke these rights from the deployer in the future. Instead, I'd recommend either making the contract Ownable (and later transferring the ownership to a multisig or revoking it altogether) or providing it to the constructor. 
- It's currently possible to premint more than 10,000 NFTs (which won't be claimable but may create other integrations problems in the future). I'd recommend either restrict it explicitly or adding a way to disable minting irreversibly (e.g., by revoking ownership, similarly to the previous note, or via an explicit "disableMinting" function + "whenMintingEnabled" modifier). The latter isn't an ideal solution because it'll also imply that it's possible to disable minting *before* 10,000 NFTs have been minted.
- No tests: 👎👎👎
- No NATSPEC of function level comments

Gas optimizations:

- Consider adding a batch premint (e.g., premitBatch(uint256[] tokensIds, string[] memory tokenURI)) so that the owner role can premint many tokens at once, thus saving external tx gas costs compared to preminting them one by one.
- The usage of the unclaimed array will make further premints and redemptions more and more expensive and require loading and accessing a potentially huge array.
- You can consider optimizing this by converting the unclaimed array to a mapping (index --> tokenID) and have another uint256 state variable to store an auto-incrementing value of the "last index dynamically". This way, every premint and redemption will only need to load/write an O(1) of storage. Alternatively, you can avoid preminting altogether (if it'd still be possible to assign the random properties in a transparent and auditable fashion later) and just mint tokens directly in selectToken.
- Convert baseURI to a constant
- Convert any public function, which isn't referenced internally to an external function.
- You don't actually have to call "require(LINK.balanceOf(address(this))", since it'll revert anyway in requestRandomness, so you might consider removing this call to save some gas.
- Some arithmetic operations can be unchecked (e.g., unchecked { balance * 1e18 }), as it's guaranteed not to overflow/underflow (yeah, a checked "i++" in loops is especially pointless, and we're all going crazy about it).

Misc:

- Use type(uint256).max instead if MAX_INT (and in any case, it should be a constant and named something like MAX_UINT, since INT is usually used in the context of signed integers)
- Specify the visibility of state variables: MAX_INT, _redeemToken, and tokenURIs.
- "address immutable _redeemToken" - you use "IERC20" instead of "address" for clarity. This will also prevent the need for casts such as IERC20(_redeemToken). Similarly, vrfCoordinator and linkToken should have explicit interface types instead of the catch-all "address" type.
- Consider DRYing "owner" and "steward" by making them constants (e.g., ROLE_OWNER).
= In redeemTokenForNFT, it's common to verify that transferFrom returns true (e.g., require(token.transferFrom...)). Similarly, the transferFrom in selectToken, and the transfer in transferLink. If you want to avoid this check to save some gas (since you know that DONA is a fully ERC20 compliant token) - consider adding a small comment to calm down auditors with OCD (like yours truly).
- CEI violation: in redeemTokenForNFT, transferFrom should be the last call. If the user doesn't approve enough tokens, it'll revert anyway, so it'd work as intended.
- In permint, the tokenURI param shadows a function with the same name. Consider renaming. Similarly, _baseURI in tokenURI.
- In setFee, you don't check that fee is valid (e.g., >= 0). In addition to that, I'd also add a "FeeChanged" event.
- Note that all the current dependencies in package.json need to be devDependencies, and that's the package.json file generally incomplete, but it's not important.
- Mixed naming conventions (_redeemToken vs. requestIdToAddress, _baseURI vs selectToken, etc.)
- No build script in package.json
# TAP Introduction (Whitepaper)
TAP is an extendible OrdFi-enabling protocol that includes - but is not limited to - the TAP token standard.

TAP works entirely without the use of L2 chains or other overly complex mechanics, but utilizes tapping - a simple mechanism to verify transactions within the protocol.

The goal of TAP is to built upon what made BRC-20 great but maintaining independence from centralized entities. TAP allows for new features that will be consistently added to the protocol by community driven governance.

The BRC-20 ticker lengths 1,2 and 4 are reserved for future interordinal bridging between BRC-20 and TAP Tokens and cannot be deployed on TAP.

# Structure

TAP consists of two parts, external and internal. The external part acts in the exact same way like BRC-20 does. 

**External:**

To get connected to TAP, marketplaces/wallets clone their existing BRC-20 infrastructure. Then either TAP's internal functions (see below) have to be implemented or endpoints that implement those have to be used to verify all balances (balance, available, transferable). From there on out, TAP tokens can be traded like regular BRC-20 tokens on their platforms. [updated Aug. 13th, 2023]

**Internal:**

Users benefit from features such as token staking and swaps. Mass-sending of tokens are already available. The community will decide through the use of $TRAC (BRC-20) which features will be added or updated.

Alongside the specs, there is already TAP protocol tracking available on https://trac.network. 

Trac's public endpoint (https://github.com/BennyTheDev/trac-tap-public-endpoint) allows developers to embrace the new standard. After the official release of Trac, developers may self-host TAP tracking and create their own endpoints.

Protocol updates will be announced in advance - with a reasonable grace-period - to allow indexers to follow. [added Aug. 13th, 2023]

# External

As mentioned above, TAP tokens work in the exact same way as BRC-20 tokens. There are however a couple of minor modifications required for indexers:

|| TAP | BRC-20 |
|-------------| ------------- | ------------- |
| Allowed ticker lengths | 3 and 5 to 32 (UTF16)  | 4 letters |
| Protocol | tap  | brc-20  |
| Deploy op | token-deploy  | deploy  |
| Mint op | token-mint  | mint  |
| Transfer op | token-transfer  | transfer  |

#### Ord Wallet Versioning

The TAP Protocol follows a defined upgrade path for indexers to support and benefit from ord wallet updates. Ord wallet upgrades are followed conservatively.
This means that the protocol won't support every single release of the ord wallet but only important and beneficial updates. 
Supported updates will be tested and announced in advance, with a reasonable time to allow indexers to be prepared.

Indexers must make sure to follow the ord wallet versions and activation heights as of the table below:

| Ord Wallet | Block Activation Height |
|------------- | ------------- |
| 0.14.1 | 801993 |
| 1.0 | TBA |

#### The Jubilee

From block 824544 onwards, no new cursed inscriptions may be inscribed any longer. Before this block, indexers have to support tokens that are deployed and minted as cursed inscriptions.

For external functions this means:

- Until block 824543, cursed inscriptions using token-deploy, token-mint and token-transfer must be addressed without dash in the ticker. Indexers must internally prefix those with a dash to separate them from their non-cursed counterparts.
- From block 824544 onwwards, no cursed tokens may be deployed or minted any longer. Tickers in token-transfer must be prefixed with a dash for cursed tokens.
- Transferable inscriptions are not affected and can be sent to a recipient at any time.
- Generally, no tokens can be deployed with a leading dash in token-deploy. Those would be invalid and rejected from being indexed.

#### Examples

```javascript
{ 
  "p": "tap",
  "op": "token-deploy",
  "tick": "tap",
  "max": "21000000",
  "lim": "1000"
}
```

```javascript
{ 
  "p": "tap",
  "op": "token-mint",
  "tick": "tap",
  "amt": "1000"
}
```

```javascript
{ 
  "p": "tap",
  "op": "token-transfer",
  "tick": "tap",
  "amt": "100"
}
```

# Internal

### token-send

Internally, there exists a new function called "token-send", which enables mass-transfer of different tokens to many recipients.

There will be more internal functions, so it may be useful to check back here once in a while.

The specs for "token-send" are defined as follows:

- Users inscribe a "token-send" inscription to their Bitcoin address.
- After the transaction confirmed, the inscription has to be resent to the same address for confirmation (tapping).
- Upon tapping, the Bitcoin address must own the the amounts of tokens that are given with the inscription.
- Only if the "token-send" inscription is tapped, sending tokens will be performed.
- The receiver addresses must be carefully validated: must be valid Bitcoin addresses, must be trimmed, addresses starting with bc1 have to be lowercased.
- "token-send" is atomic upon inscribing (before tapping): all amounts, tickers and addresses must be syntactically valid.
- Upon tapping, invalid and semantically incorrect token sends must be skipped (e.g. insufficient funds, invalid amounts or invalid data types).
- Each successful token send must credit the given amounts to recipient and be removed from the sender's balance.
- Each send item must exclusively operate on available balances, not overall balances (available = balance - transferable). [added Aug. 8th, 2023]

#### Example

The below example will send 1000 tap tokens to each address. Tokens and amounts can be mixed.
There is no restriction on the amount of items (receivers).

```javascript
{
  "p" : "tap",
  "op" : "token-send",
  "items" : [
     {
      "tick": "-tap",
      "amt": "10000",
      "address" : "bc1p9lpne8pnzq87dpygtqdd9vd3w28fknwwgv362xff9zv4ewxg6was504w20"
     },
     {
      "tick": "tap",
      "amt": "10000",
      "address" : "bc1p063utyzvjuhkn0g06l5xq6e9nv6p4kjh5yxrwsr94de5zfhrj7csns0aj4"
     }
  ]
}
```

### token-trade

The internal function set "token-trade" consists of 3 functions to allow text-inscription based trading. By inscribing a trade inscription as seller, a trade is being added to the Ordinals collection and can be picked up by any buyer for the duration of a valid period in blocks. 

Upon filling a trade by the buyer, an optional fee receiver address may be added to receive fixed 0.3% fees in tokens per filled trade. Fees are applied on top of the purchased token and is being paid by the buyer. Fees give inscription services and marketplaces a way to benefit from internal trading, while buyers benefit from the convenience and guidance of their services.

To inscribe a trade, a seller has to inscribe a text inscription in the following format:

#### Creating a Trade

```javascript
{
  "p" : "tap",
  "op" : "token-trade",
  "side" : "0",
  "tick" : "tap",
  "amt" : "1",
  "accept" : [
    {
      "tick" : "buidl",
      "amt" : "0.1"
    },
    {
      "tick" : "based",
      "amt" : "0.2"
    }
  ],
  "valid" : "900000"
}
```

As soon as the inscription above is inscribed, the trade is open and ready to be filled under the following conditions:

- All attributes are mandatory as of the example above.
- Side = 0 specifies that this is a trade function by a seller.
- The seller must own the offered token amount in the moment of filling the trade by a buyer.
- The offered token amount must be deducted from the available balances, not overall balances (available = balance - transferable).
- The tokens defined in the "accept" attribute represent accepted tokens for the trade.
- If any of the accepted tokens are being used by the buyer, the trade concludes. 
- The tokens defined in the "accept" attribute must be deployed in the moment of filling the trade by a buyer.
- The block specified in the "valid" attribute must be a future block (inclusive).
- The trade is invalid and not being indexed if the current block is larger than "valid".
- Up to block 824543 and if the offered token is cursed, then the inscription has to be inscribed as cursed.
- Ticks within the "accept" attribute must be unique. Only the first must be indexed if there is more than one of the same.
- Cursed tokens defined in the "accept" attribute have to be prefixed with a dash (this is to allow mixed cursed/non-cursed trading).
- From block 824544 onwards, the offered token ticker has to be prefixed with a dash, if it originally has been deployed as cursed token.
- After inscribing the function, sellers have to send the inscription to themselves to approve (tapping).

#### Cancel a Trade

In order to cancel an existing, non-expired trade. The following function has to be inscribed and tapped:

```javascript
{
  "p" : "tap",
  "op" : "token-trade",
  "side" : "0",
  "trade" : "<inscription id>"
}
```

- All attributes are mandatory as of the example above.
- Side = 0 specifies that this is a trade function by a seller.
- The "trade" attribute has to be populated with the inscription id (_not_ number) of the referenced trade inscription above.
- The trade will only cancel if the referenced trade inscription is owned by wallet address that intends to cancel.
- After a trade has been cancelled, no trades for it can occur.
- After inscribing the function, sellers have to send the inscription to themselves to approve (tapping).

#### Fill a Trade

Valid trades are fillable by inscribing a token-trade inscription that specifies which offered token to purchase, with a reference to the original trade inscription. Additionally, an optional fee receiver may be specified. The fee receiver will earn fixed 0.3% of the token that the buyer is purchasing. The fee is applied on top of the purchase amount.

```javascript
{
  "p" : "tap",
  "op" : "token-trade",
  "side" : "1",
  "trade" : "<inscription id>",
  "tick" : "based",
  "amt" : "0.2",
  "fee_rcv" : "<fee receiver address>"
}
```

- Except "fee_rcv", all attributes are mandatory as of the example above.
- Side = 1 specifies that this is a trade function by a buyer.
- The "trade" attribute must match the original trade inscription id (_not_ number).
- The "tick" and "amt" attributes must exactly match one of the accepted tokens from the referenced trade inscription.
- The buyer must _not_ fill all accepted tokens as of the referenced trade but exactly one.
- The ticker must be deployed in order to get the above indexed.
- The buyer must own the specified token and it's amount + 0.3% trading fees if the "fee_rcv" attribute is set.
- If the "fee_rcv" attribute is set, it must be a valid Bitcoin address.
- Upon indexing, the Bitcoin address in "fee_rcv" must be trimmed and lowercased if the address starts with "bc1".
- Up to block 824543 and if the offered token is cursed, then the inscription has to be inscribed as cursed.
- From block 824544 onwards, the "tick" attribute has to be prefixed with a dash, if it originally has been deployed as cursed token.
- After inscribing the function, buyers have to send the inscription to themselves to approve (tapping).
- If the current block upon tapping is larger than the block in the "valid" attribute as of the referenced trade, filling will fail.
- If all conditions are met, the token and its exact amount is being sent to the seller and the offered token is being sent to the buyer. If fees apply, the fee will be sent to the fee receiver.
- Fees are only applicable on amounts and decimals that allow 0.3% to be applied. This means there may be zero fees if not applicable.
- A trade has been filled successfully from an indexer point of view, if all conditions are met and all balances are credited correctly.

### token-auth

The internal function set "token-auth" allows 3rd parties (authorities) to independently issue signed redeem inscriptions. These signed redeem inscriptions may be inscribed by anyone and authorized tokens being sent to recipients. This allows for custom logic to be specified about what tokens should go where and when. 

This makes TAP basically a no-knowledge protocol within Ordinals as it doesn't need to know the details of custom logic provided by 3rd parties. Unlike zk-rollups, token-auth relies solely on message signatures and does not require sequencers or L2 mediators. Token-auth can even be used to connect with zk-rollup solutions, adding an extra layer of authenticity.

token-auth does not rely on witness data hacking. All operations are clearly contained within the Ordinal envelope's data and thus easily provable by any indexer or individual.

Typical use-cases for token-auth are gamification (e.g. convert points to tokens), token bridges, staking, reward systems, cross-chain marketplaces and more.

#### Create an authority

To create an authority, a special token-auth inscription must be sent and tapped with an address that holds authorized tokens.

Manual creation of token-auth code is not recommended as it requires message signatures. See the boilerplate (https://github.com/Trac-Systems/tap-protocol-token-auth-boilerplate) that helps to generate signed token auths. Token-auth uses secp256k1 signatures. See https://github.com/paulmillr/noble-secp256k1 as an example implementation.

Example:

```javascript
{
   "p":"tap",
   "op":"token-auth",
   "sig":{
      "v":"0",
      "r":"51143521410606380758535576062355234772504706283689533465002520447203156100709",
      "s":"23524754809729078525228087002160468580495709275023865450917881139756565577560"
   },
   "hash":"0f30c22be2f46e819538ca1281aadb82d3928cae5a699cade80013c5b14871e4",
   "salt":"0.25991027744263695",
   "auth":[
      "gib"
   ]
}
```

- The "auth" attribute must contain an array of deployed TAP protocol tokens that are verified for the authority's account.
- If the "auth" attribute's array is empty, all tokens owned by the authority's account are authorized [added Sep. 25th, 2023]
- Upon indexing, the token-auth inscription must be verified against the signature ("sig"), "hash" attribute and public key.
- - The public key must be recovered by using "hash".
  - To prevent hash collisions, a custom "salt" value has to be provided by the authority.
  - To verify, the "auth" array must be JSON-stringified and sha256-hashed with "salt" (sha256(auth + salt)).
  - "hash" must be unique and can only be used once across the entire indexing state.
  - Token-auth is atomic, this means that all tokens in "auth" must be deployed at the time of inscribing (if the "auth" array is _not_ empty).
  - The authority address does NOT need to own the authorized tokens at the time of inscribing (but should, upon redeeming, see below).
  - If all authorized tokens are deployed or the "auth" array is empty, the hashed signature is valid and the hash has never been used before, the token-auth inscription must be indexed after tapping (=sending to "yourself"). 

#### Create a redeem

Tokens to redeem are signed and issued by the authority. The inscription code may be inscribed by everyone and it is under the sole discretion of the authority when and how redeems are issued. Typically, an authority would sign and issue a redeem when all its conditions are met.

Manual creation of token-auth code is not recommended as it requires message signatures. See this example script that helps to generate signed token auths. Token-auth uses secp256k1 signatures. See https://github.com/paulmillr/noble-secp256k1 as an example implementation.

```javascript
{
   "p":"tap",
   "op":"token-auth",
   "sig":{
      "v":"1",
      "r":"113472523327934685528808901641630457916054343054143422440331961430719594721038",
      "s":"21393407019197854961723689634443789868582208930187383447036700552814535514199"
   },
   "hash":"82e2b0d098dcdab820ff866b011250af8841a6b59cedd7164bb94b63d2598de9",
   "salt":"0.46074583388095514",
   "redeem":{
      "items":[
         {
            "tick":"gib",
            "amt":"546",
            "address":"bc1p9lpne8pnzq87dpygtqdd9vd3w28fknwwgv362xff9zv4ewxg6was504w20"
         }
      ],
      "auth":"fd3664a56cf6d14b21504e5d83a3d4867ee256f06cbe3bddf2787d6a80a86078i0",
      "data":""
   }
}
```

- The "redeem" attribute must contain an object consisting of "redeem" => "items", the inscription id of the signing authority ("redeem" => ""auth") and "redeem" => "data".
- Upon indexing, the token-auth inscription must be verified against the signature ("sig"), "hash" attribute and public key.
- - The public key must be recovered by using "hash".
  - To prevent hash collisions, a custom "salt" value has to be provided by the authority.
  - To verify, the "auth" array must be JSON-stringified and sha256-hashed with "salt" (sha256(auth + salt)).
  - "hash" must be unique and can only be used once across the entire indexing state.
  - "redeem" => "data" must be present but may be empty.
  - Based on the inscription id in "auth", the public key by the authority must be recovered and compared with the public key of the redeem.
  - - Both public keys must match.
    - Both hashes must be valid.
    - The original auth inscription mustn't be cancelled.
    - All tickers in "redeem" => "items" must be specified in the original "auth" inscription.
- If all conditions are met, the tokens specified in "redeem" => "items" must be sent exactly like TAP's function "token-send" above, including all of its conditions. Tapping is not required, as this is a signed transaction and may be inscribed by everyone.

#### Cancel an authority

To cancel a "token-auth", the authority must send an inscription like below to its associated address and tap.

```javascript
{
   "p":"tap",
   "op":"token-auth",
   "cancel" : "fd3664a56cf6d14b21504e5d83a3d4867ee256f06cbe3bddf2787d6a80a86078i0"
}
```

- "cancel" must point to an existing an non-cancelled "token-auth" of the authority.
- Once tapped, no further redeems may be executed on the inscribed "token-auth", indefinitely.

### privilege-auth

The internal function set "privilege-auth" allows 3rd parties (authorities) to implement token/unat/collectible whitelists, launchpads and general-purpose provenance for file hashes (sha256). 

Privilege authorities will be available from activation block height 841682.

#### Create an privilege authority

To create a privilege authority, a special privilege-auth inscription must be sent and tapped.

Manual creation of privilege-auth code is not recommended as it requires message signatures. See the boilerplate (https://github.com/Trac-Systems/tap-protocol-privilege-auth-boilerplate) that helps to generate signed privileges. Privilege-auth uses secp256k1 signatures. See https://github.com/paulmillr/noble-secp256k1 as an example implementation.

Example:

```javascript
{
   "p":"tap",
   "op":"privilege-auth",
   "sig":{
      "v":"0",
      "r":"86508516128453602592995353796555408942165629541645840597158531627249758208446",
      "s":"34207792571992691071321015990261453361621215423082720220623653945331263914520"
   },
   "hash":"a7a8fe6a46d7dd3aac3518ecdc97577be5319eb5ae8588eee9c92ad96eed3b68",
   "salt":"0.41631368860188056",
   "auth":{
      "name":"Some privilege authority"
   }
}
```

#### Token deployments

Token deployments work exactly like regular deployments as described further above but a new attribute called "prv" must be added, pointing to an existing, active (not cancelled) token authority by using its inscription id (not number).

If there is no valid authority at the given inscription id for "prv", the deployment will fail. If the authority is valid, only the authority specified in "prv" will be allowed to specify what address is allowed to mint tokens.

The authority still has to respect all given rules as of the deployment inscription. It cannot override limits, max supply, tickers.

Example for regular token deploys on the TAP Protocol:

```javascript
{ 
  "p": "tap",
  "op": "token-deploy",
  "tick": "tap",
  "max": "21000000",
  "lim": "1000",
  "prv" : "fd3664a56cf6d14b21504e5d83a3d4867ee256f06cbe3bddf2787d6a80a86078i0"
}
```

Example for DMT token deploys on the TAP Protocol:

```javascript
{
"p": "tap",
"op": "dmt-deploy",
"elem": "63b5bd2e28c043c4812981718e65d202ab8f68c0f6a1834d9ebea49d8fac7e62i0",
"tick": "nat",
"dt": "n",
"prv" : "fd3664a56cf6d14b21504e5d83a3d4867ee256f06cbe3bddf2787d6a80a86078i0"
}
```

#### Token mints

Token mints work exactly like regular mints as described further above but a new attribute called "prv" must be provided.
"prv" must contain an object specifying the authority's signature and the wallet address that is allowed to mint.
"salt" can be used to pass extra data to make sure the resulting hash is always unique. 
It is however recommended to use non-random salts (e.g. inscription ids) if the authority has the requirement internally to be able to re-index at all times.

The authority must broadcast the signed inscription text via an offchain mechanism of its choice.

The signature given with the "prv" attribute object must match the signature of the authority. Only if both resulting public keys match, the mint is valid.
If the the resulting keys do not match, or the signature is invalid, the mint will be rejected.

If any other address other than the one given tries to mint, the mint will fail and the authority has to re-issue a signed inscription with a new salt since hashes are unique across the protocol.

Example mint inscription text on the TAP Protocol:

```javascript
{
   "p":"tap",
   "op":"token-mint",
   "tick":"tap",
   "amt":1000,
   "prv":{
      "sig":{
         "v":"0",
         "r":"106344897850645353394557411159466118155649878604746085611434406050662507764701",
         "s":"5566788626956401066167017793946358967684734480695302740591785789681566880049"
      },
      "hash":"931c0b44c1811ba4a8ac7c696a7f0e664ea85b18c34ec32d492fcc99fd7ee4a7",
      "address":"tb1pf9jluy2g797290uq5nutqm2yuynds6uf868ytc37nht53c5j8w3s7nfta7",
      "salt":"0.3624804148476173"
   }
}
```

Example DMT mint inscription text on the TAP Protocol:

```javascript
{
   "p":"tap",
   "op":"dmt-mint",
   "tick":"nat",
   "blk":840000,
   "dep":"825e287bb7dd163ed633110e31bc6abb6c80815ca68b7dd3cc71d729ecaaa3dci0",
   "prv":{
      "sig":{
         "v":"0",
         "r":"10387661166049122705773272120018537684460586485610280053381057207269720400370",
         "s":"47937738938877326713978443360828324693873763102762802539146112797003515228281"
      },
      "hash":"1056344a88698e989aa6799c46d72f5b110fb26ff0426c1a84a6640803b0d74c",
      "address":"tb1pf9jluy2g797290uq5nutqm2yuynds6uf868ytc37nht53c5j8w3s7nfta7",
      "salt":"0.3903927009829504"
   }
}
```

#### Hashed Verifications

Hash verifications within privileges are a general-purpose mechanism to determine what file hash (sha256) may be assigned to a given address.

If an authority issues such a signed inscription text, it can signal that it sees the address as the valid owner of that hash.

An interesting aspect of hashed verfications that these might be used as collectibles as they contain verified file hashes of linked or offchain content.

Different authorities may have different opinions. That's why it's important that projects managing authorities, must broadcast their authority inscription ids via offchain channels to provide the source of truth.

Like deployments & mints, a signature and the inscription id of the authority must be given ("prv"). 

The attribute "verify" must contain an sha256 hash, representing a hash over file contents.
"col" must be used to specify a "collection" name to allow for organizing the hashes. Maximum allowed symbols: 512 (as of unicode).
"seq" must be an unsigned integer, that may be arbitrary and not larger than 9007199254740991.

If "seq" is passed as string, the verification will fail.
"col" may be an empty string but must be present.

"salt" can be used to pass extra data to make sure the resulting hash is always unique. 
It is however recommended to use non-random salts (e.g. inscription ids) if the authority has the requirement internally to be able to re-index at all times.

```javascript
{
   "p":"tap",
   "op":"privilege-auth",
   "sig":{
      "v":"0",
      "r":"102486240205705464835967036484087123773428387054126262751119766795035126943608",
      "s":"2461281772805092833125901827999803585726660443752983045919874042616328309644"
   },
   "hash":"81906992bfffa87f48cdeaecff29bac840553ac0bdfb9288bed0cab6de573efc",
   "address":"tb1pltn4mqlpxxswxhmkj2ejdd0z98v74nr0xxdresvwvp5cyvctm0zs3m750l",
   "salt":"0.9698534940599512",
   "prv":"e349a9126a9476eb534457a7e78c748aeca67ec4d53fa9f0772408fb7233a9fei0",
   "verify":"cea505f61f375ea2d8ea56f593e6b436f963753616c2095e755cb5ca4a6df85c",
   "col":"super collection",
   "seq":111
}
```

#### Cancel a privilege authority

To cancel a "privilege-auth", the authority must send an inscription like below to its associated address and tap.
From this point on, no mints or verifications can take place anymore that originally have been managed by the authority.
This can be used for blockouts, collection finalization etc.

```javascript
{
   "p":"tap",
   "op":"privilege-auth",
   "cancel" : "fd3664a56cf6d14b21504e5d83a3d4867ee256f06cbe3bddf2787d6a80a86078i0"
}
```

#### DMT (Digital Matter Theory) Tokens

TAP Protocol supports element field 11 as of the DMT specs located at https://digital-matter-theory.gitbook.io/digital-matter-theory/introduction/digital-matter-theory

This includes the functions "dmt-deploy" and "dmt-mint" and work according to the DMT specs within the TAP Protocol.

#### Outlook

This document and the tracking for the TAP protocol will be continuously worked on and updated. Feel free to join the Discord if you have questions: https://discord.gg/sPyYDa5q6P

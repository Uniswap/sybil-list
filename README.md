# sybil-list 
This repo contains a list of verified mappings that link Ethereum addresses with social profiles (Twitter supported currently). 

- JSON List: [verified.json](./verified.json)
- Sybil interface for governance: [https://sybil.org](https://sybil-interface.vercel.app/#/delegates/uniswap)
- Interface repo: [https://github.com/Uniswap/sybil-interface](https://github.com/Uniswap/sybil-interface)
- Verifier repo: [https://github.com/Uniswap/sybil-verifier-worker](https://github.com/Uniswap/sybil-verifier-worker)
 
## What is Sybil
Read the Sybil announcement post : [https://uniswap.org/blog/sybil/](https://uniswap.org/blog/sybil/)

Sybil is a governance tool for discovering delegates. Sybil maps on-chain addresses to digital identities to maintain a list of delegates. These verified mappings are public and open for anyone to use. 
 
One use case for Sybil is governance systems on Ethereum. Delegates and voters in these systems benefit from seeing real-world identities attached to Ethereum addresses. A interface for governance that incorporates Sybil can be used here [https://sybil.org](https://sybil.org). 

## Use the Sybil list   
 
A [hosted verifier](#hosted-verifier) stores entries as part of a JSON blob in [verified.json](./verified.json) that maps all verified Ethereum addresses to their social usernames. For any address, data is indexed by a platform type. 
 
#### Schema 
 
A twitter entry for an Ethereum address includes 3 fields of data : 
-  `handle` : the username found in a users tweet
- `tweetID`: the id of the tweet (stored as reference for tweet data)
- `timestamp`: unix timestamp at time of verification 
 
 
#### Consuming the list 
 
The raw JSON can be found at [https://raw.githubusercontent.com/Uniswap/sybil-list/master/verified.json](https://raw.githubusercontent.com/Uniswap/sybil-list/master/verified.json).

To use, just fetch the data at this endpoint: 

```typescript
fetch('https://raw.githubusercontent.com/Uniswap/sybil-list/master/verified.json').then(async res => {
  res.json().then(data => {
      console.log(data) // list data 
  })
})
```
 
## Verifying an Identity 
Sybil uses a 3 step process for linking an Ethereum address to a social identity. 
 
1. User uses their Ethereum private key to sign a message consisting of their social username (Twitter handle, github username, etc). 
2. User posts this signature on their social profile so others can view. 
3. Third parties can recover the signer of the signature found in a social post. For Twitter, a verifier would: 
	* Fetch tweet content for a tweet containing a signature 
	* Construct signature data using the author of the tweet 
	* Parse the signature found in the tweet
	* Using these two inputs, recover a signer address
	* Store this signer as a verified owner of the handle that authored the tweet
	
Example tweet: @todo link to tweet from real account 
 
#### Signature Construction 
 
The sybil interface formats signature data according to [EIP-712](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md).

The message field for the signature contains just one field, `username`, which is a string representing the user's social username (Twitter handle, github name, etc). The signature data can be constructed as follows: 
 
```typescript
const EIP712Domain = [
  { name: 'name', type: 'string' },
  { name: 'version', type: 'string' }
]
const domain = {
  name: 'Sybil Verifier',
  version: '1'
}
const Permit = [{ name: 'username', type: 'string' }]
const message = { username: <include social username here> }
const data = JSON.stringify({
  types: {
    EIP712Domain,
    Permit
  },
  domain,
  primaryType: 'Permit',
  message
})
```

#### Signing Data

After constructing the data, signatures are produced using eth `personal_sign`. This is to allow for hardware wallet support, as `signTypedData_v4` is not yet supported by most hardware wallets. 

See [here](https://github.com/Uniswap/sybil-interface/blob/master/src/hooks/useSignedHandle.ts) for how this is implemented in the [Sybile interface](https://sybil.org/#/delegates/uniswap).
 	
#### Recovering a signer and verifying social identity 
 
After obtaining a signature and constructing data based on a username, a verifier can recover an Ethereum address using any of many web3 libraries. This may look something like: 
 
```typescript
import { recoverPersonalSignature } from 'eth-sig-util'
 
const data = <'construct data as shown above'>
const sig = <'collect signature from social post'>
const signer = recoverPersonalSignature({
    data: JSON.stringify(data), // string format of EIP-712 data
    sig,
})
// if recovered, the signer is now a verified owner of the username used to construct the signature
```
 
## Hosted Verifier
 
Verifier repo: [https://github.com/Uniswap/sybil-verifier-worker](https://github.com/Uniswap/sybil-verifier-worker) 
 
Users who verify their identities through the [Sybil Interface](https://github.com/Uniswap/sybil-interface) submit data to a cloudflare worker that runs a verification script. If the content in the tweet meets requirements and valid signer is recovered, the signer <-> mapping is added to [verified.json](./verified.json) in a new commit. 
 
## Verify your own list 
 
Users have the option to consume and trust the list provided in this repo, or can run the verification process themselves. Any social profile that has a published, valid, signature can be verified. 

These steps describe how to verify a list of mappings on your own: 

1. Collect a list of identities you are trying to verify, aka a list tweetID's containing signatures. 
	* One strategy is to collect all tweets that use a relevant Sybil hashtag. These tweets are generated by users verifying on the Sybil interface. The current hashtags are `#UNIDelegate` and `#COMPDelegate`. 
2. For each tweetID, follow the steps above to recover signer address
3. store and use address -> handle mapping 
	
This process can be adapted to support additional social networks in the future (github, telegram, etc)

- [#UNIDelegate Tweets](https://twitter.com/search?lang=en&q=(%23UNIDelegate))

- [#COMPDelegate Tweets](https://twitter.com/search?lang=en&q=(%23COMPDelegate))

## Future Work / Considerations

#### Additional Platform Support

Currently the [Sybil interface ](https://sybil.org/#/delegates/uniswap) only supports verification through Twitter. Other social media platforms like Github may prove to be useful as well. Support for these will likely be added over time, and new entries will be formatted to exist in one shared JSON file. 

#### More verification channels

Right now the interface is the only easy way for users to verify using Sybil. However, more verification UIs may be built in the future. This could take the form of standalone sites, or verification flows embedded within other web3 UIs. 


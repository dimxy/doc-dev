This is a second part of the tutorial how to write a cc contract.
Now I would like to show how to add token support into Custom Consensus (cc) contract in Komodo asset chain.

What is actually a token?

Tokens's another name is also 'colored coin' what I would explain as 'coins with some additional info'. Buth there is another difference: 
token is not mined but issued by an individual who just has public and private key and some coins in a blockchain which supports tokens.
Usually the purpose of a token is to represent in a blockchain some real-world asset which usually is owned by the token issuer like shares and tickets, wheat and oil, cars and houses, or even fiat money pegged to a token.
So a token has its issuer and it has some attached description of the assets which this token represents. Obviously token ownership could be transferred to another individual in exchange for coins or simply as a gift. 

To issue and transfer a token we would need to create and mine token transaction. 

In Komodo world blockchains where tokens live are called 'asset chains'.

So what is technically a Komodo asset chain token?

One of the differencies of tokens from coins is that it contains some attached data.
We already know that we might put such data in transaction opreturn which is an unspendable output. It is unspendable thankfully to a special OP_RETURN opcode in transaction scriptPubKey which actually cancels the script interpretation when it is encountered. We can do  this for tokens too.

Komodo token begins with a token creation transaction ('tokenbase') where the issuer puts token name and description. All the initial amount is allocated with the creation tx and never can be added (but might be burned). 

When a token is created the owner converts some of his normal coins to tokens (with rate 1 satoshi == 1 token).

The id of tokenbase transaction becomes the identifier of the token (\`tokenid\`)

Later some token amount might be transferred to another public key via a token transaction. Such token transactions will always have an opreturn with tokenid in with which the transaction is marked as token tx. 

Current token balance of a pubkey is actually a value calculated for all its token utxos. A token utxo has some value and the token eval code (EVAL_TOKENS) cryptocondition in the scriptPubKey. 

There is a Tokens cc contract which is reponsible for validation of token tx. This validation is triggered when a token tx spending another token utxo, with token eval code in its vins, is added to the asset chain.

Tokens may be used with other cc contracts, for this such a contract eval code (for instance, EVAL_ASSETS) is also added to token vouts, which provides that this contract validation is triggered for token tx, too.

Tokens could be fungible and non-fungible.


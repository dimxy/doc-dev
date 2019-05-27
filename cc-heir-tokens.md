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
We already know that we might put such data in transaction opreturn which is an unspendable output. It is unspendable thankfully to a special OP_RETURN opcode in transaction scriptPubKey which actually cancels the script interpretation when it is encountered.

Komodo token is actually value calculated from a number of token uxtos. A token uxto has some value and the token eval code (EVAL_TOKENS) cryptocondition in the scriptPubKey. A token is also identified with a tokenid which is the txid of the first token transaction (tokenbase). For every other token tx (that is not tokenbase) the tokenid is stored in the tx opreturn vout. Such opreturn vout is present in every token tx.

Tokenbase tx also has basic token parameters (name, description and issuer pubkey).

Tokens cc contract is reponsible for validation of token tx. This validation is triggered when a token tx, with token eval code in its vins, is added to blockchain.

When a token is created the owner actually converts his normal coins to tokens (with rate 1 satoshi == 1 token).

Tokens may be used with other cc contracts, for this such a contract eval code (for instance, EVAL_ASSETS) is also added to token vouts, which provides that this contract validation is triggered for token tx, too.

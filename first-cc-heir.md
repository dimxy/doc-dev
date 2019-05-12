When I joined Komodo team as a cryptocondition (now rebranded as custom consensus) contract developer I was assigned my first task: to develop a 'Heir cryptocondition contract'. 
My acquaintance with crypto conditions began with reading James Lee's book "Mastering Cryptoconditions". I spent a weekend on reading it and got the common idea what cc contracts are. 
But the whole scale of cc contract features and opportunities were uncovered to me later when I implemented the validation code for my Heir cc contract.

Now I'd like to share my experience with developers who want to create very efficient and powerful custom consensus contracts to help them get the matter of what custom consensus contracts are, what they can do and how to develop them.

I will also try to explain some internal properties of cryptoconditons because I believe most developers like me want to know how everything is organized internally :-)

It is supposed you are a c++ developer.
For reading this article it is recommended that you have basic knowledge about bitcoin or Komodo blockchain, transaction structure and UXTO principle.  It is also good to know about script concept and especially OP_RETURN opcode as it plays important role in adding business logic to custom consensus contracts.

It might be not very simple for newbie developer to understand what cryptoconditions are and what they allow. 
I think this very simple Heir cc contract which has commonly understandable purpose of inheritance will be a good demonstation of power and features of cryptoconditions.

Let's learn a bit about what I needed to develop.
The idea of the Heir cc contract was inheritance of crypto funds: if funds owner was inactive for some specified time the fund's heir might use those funds.
Okay, let's move on.

### CC contract concept

To understand cc contract idea and structure I studied well-known simple Faucet cc contract which actually allows to lock some amount of funds on a crypto condition address and draw funds by small portions. The Faucet cc does not allow to do this fast as it requires to do some PoW for successful funds drawing.

What did I know from the first experience with a cc contract?

First, I'd say it implements some business logic: in faucet cc contract this logic is about storing funds on some address and set a 'faucet' to take them back (maybe like in some advanced moneybox which would not allow spend all the funds at once). For Heir cc the business logic is inheritance of the blockchain funds.

The next, contract's business logic is bound to transactions. In some sense transactions are a data source for cc contract application. As a usual blockchain transaction simply moves coin value from one address to another we need some place for application data to put into it. This place is so-called opreturn and it is a transaction output which is never spendable and where contract's data is stored. Usually opreturn is the last output in a transaction.
When a cc contract instance begins its lifecycle an initial transaction is created and later additional transactions are added and attached to the initial transaction by spending its outputs. Later more transactions might be added which spend outputs of newly added transactions.

The next important thing in a cc contract is cryptoconditions.
What is a cryptocondition and why it is important?

### Cryptoconditions in simple terms
A cryptocondition in Komodo is basically a logical expression executed on electronic signatures and hashes of spending transactions and stored in transactions' scripts, plus a supporting c-library which allows to evaluate and check such expressions. 
In wider sense, cryptoconditions is a technology which allows to build and evaluate complex logical expression based on results of cryptographic functions.

A cryptocondition consists of two parts: a condition stored in the transaction (which to be spent) output's scriptPubKey and cryptocondition fulfillment which is in the spending transaction input's scriptSig. 
The condition contains instructions and data to check cryptocondition (like pubkey).
The fulfillment has instructions and data how to evaluate cryptocondition (for example, an instruction to check spending transaction signature, signature value to be checked and the pubkey to verify the signature). 
When a tx which spends some tx cc output is validated its input's cc fulfillment is evaluated and the result is checked with the condition in the output of the tx which is to be spent.

The simplest cryptocondition is evaluating a electronic signature of spending transaction scriptsig and by that allowing to spend funds from some other transaction cc output. 
You might ask why we need this because this is already a function of any blockchain. Yes, but there is the difference: when a transaction cryptocondition output is spent the cc contract code might enforce additional logic to validate this spending.

Anyway, such a simple cryptocondition is only a basic feature.

Cryptoconditions allows to do more. For example, another often used cryptocondition is a threshold of N which evaluates to true if a spending tx is signed with one of N possible keys. Also, application validation is possible for this too. In heir cc description further in this doc we will see what validation would allow.

Cryptocondition might be a tree of subconditions which allows to build complex cryptoconditions.
I might say that cryptoconditions technology is very advanced development of basic bitcoin script security features like pubkey or pubkey hash scripts.

We now know that in Komodo there might be advanced transactions having cryptocondition inputs and outputs. Later we will know that such transactions are attached to some cc contract by a eval code stored in tx cryptocondition inputs and outputs.

### CC contract features

We might call a cc contract as a data and business logic layer of an application. Where data layer is transactions related to a cc contract and business logic is the cc contract's code.
A presentation layer could be a web or any other GUI application which connects to a cc contract.
Or, it might be a server application that interacts to cc contract (that is, an oracle). For interaction cc contract has a set of rpc functions.

When a transaction is created in the framework of a cc contract it sends some value from a cc address to another cc address (pubkey) on cc outputs plus some data in opreturn output. An initial cc contract transaction usually sends value from normal inputs to cc outputs.

Yet another often used cc contract property is cc global address. Each cc contract is assigned its own global publicly known address for which its private key is also available to all. This address might be used for sharing funds between cc contract users. Anyone might try to spend funds from such global cc contract address (it is also called 'unspendable' address in cc codebase which probably means 'unspendable without validation'). 
A spending tx must pass through the cc contract validation code which enforces various rules for spending from this address. For instance, if a tx sent some amount to the global address to use it as a marker for future finding this transaction with a special API function, the cc validation code might capture and prohibit spending from such cc output.
(Note that validation code might validate spending from any cc contract address, not only from the global cc address).

So a cc contract code has two parts: a set of rpc functions for creation of cc contract transactions and validation code which is triggered when a cc transaction is added to the chain.
Note that cc contract validation code is activated only for the transactions which have at least one cc input with the contract's eval code cryptocondition inside the scriptSig.
It is also important to understand that as cc contract's initial transaction might not have cc input at all, it never gets into the validation code. So it would be mined in the blockchain. If you still need to validate it you have to step back when validating the spending tx and also validate the initial tx. Plus, you cannot remove the initial tx from the chain if it turned out to be incorrect, so you just need to ignore it.
The main purpose of validation code is to prevent inappropriate structure and spending of cc contract transaction,  especially if someone tries to create a tx bypassing the contract's rpc functions.

That being said, for a cc contract development you would need:
* allocate a new eval code for your contract
* assign global cc and normal addresses for the new contract
* define the contract's transactions, their structure (inputs and outputs) and opreturn format
* implement the contract's rpc functions for creation of these transactions and to return contract's data and/or state
* implement standard rpc functions for getting list of all the contract's initial transactions and getting user's and global cc addresses
* implement the contract's validation code

### CC contract architecture

A cc contract is actually a c/c++ source file. The one part of is an rpc (remote procedure call) implementation code that either creates the contract's transactions or gets info about contract state (from existing transactions). So when you deploy your contract (actually adding its code to the Komodo source code) your rpc calls are added to komodo-cli client command line.

Basically all contracts have two methods xxxlist and xxxinfo <initialtxid>.
xxxlist lists all initial transactions providing a key (inital txid) to an instance of cc contract data and xxxinfo outputs some info about a contract instance.

The other essential part of a cc contract is validation code. It will be uncovered later.

A unique 8-bit eval code is attached to each cc contract. It is used by komodo core transaction validation code to route transactions to a cc contract validation code, if a currently validated transaction is relevant to this contract.
This eval code is actually a simple cryptocondition which is a byte value and is present in any transaction relevant to a cc contract.

### CC contract transaction structure
Here on the diagram is a cc contract transaction structure sample.

![cc tx structure](https://github.com/dimxy/images/blob/master/cc-tx-structure-for-guide-v6.png)

We can see on the diagram that cc transaction has one or more cryptocondition inputs (or vins) and one or more cryptocondition outputs (or vouts). 
A cc input with cryptocondition contain the txid of a previous transaction which cc output is spent and a cryptocondition fulfillment which is evaluated when this tx is added to the blockchain. The previous tx cc output has the corresponding cryptocondition which value should match to the evaluated input cryptocondition fulfillment while validation occurs.
The tx has also opreturn output with some contract data.

### CC contract SDK

Although officially not called yet as SDK there is a set of functions for cc contract development. Most of it is in CCtx.cpp, CCutils.cc, cc.cpp, eval.cpp source files.
I will describe most used functions in the following section where I decompose Heir cc contract development process.

## Heir cc contract development
In this section I describe development of a simplified version of my Heir cc contract.
As I stated above we need to add new contract's eval code and global addresses, define heir cc contract transactions, implement the rpc interface and validation code.
Adding new eval code and addresses is trivial and you might find in the 'Mastering Cryptocondions' JL's book.
Let's continue with defining Heir cc contract transaction structure.

### Heir cc transactions

Actually we need three transactions: an initial tx with which a user would create fund for inheritance, a tx for additional funding and a tx for spending funds by the owner or heir.
I'll try to describe these tx structure with the semi-formal notation used in James Lee 'Mastering cryptocondition' book which allows to specify vins or vouts position in a tx and their description.

#### fund coins transaction (initial):

* vins.*: normal inputs
* vout.0: funding CC 1of2 addr for the owner and heir
* vout.1: txfee for CC addr used as a marker
* vout.2: normal change
* vout.n-1: opreturn 'F' ownerpk heirpk inactivitytime heirname

Here is some explanation of the used notation:
* 'vins.*' means any number of inputs
* 'normal' means usual and not cc inputs or outputs
* 'vout.0' means an output with order number 0
* 'vout.n-1' means the last vout in list (usually it is opreturn vout with contract data)

So by a funding transaction an owner creates a contract instance (a funding plan) and deposits some fund for future spending. The funding is added from normal coin inputs. 

The main funding goes to vout.0. As we suppose that either the funds owner or heir might be able to spend the funds we use an advanced cryptocondition feature threshold of 2 (marked here as '1of2') which means that cryptocondition is fulfilled if a spending tx is signed by one of the two specified keys.

Some fee is sent to vout.1 for use of it as a marker. Marker will be used to list all created contract initial transactions by a special sdk function. 
There is a normal change to return back to the owner's pubkey the extra value.
Note that we would need to provide inputs not only for the funding but for the marker value and some fee for miners. Usually we use a constant value of 10000 satoshis (0.0001 coin) for such fees.

Note 'F' letter (from 'fund') in the opreturn structure. By convention the first byte of any opreturn is eval code (omitted in the description above) and the second byte is transaction functional id.
In the opreturn also stored owner and heir pubkeys, inactivity time (in secinds) and the descriptive name of this funding plan (contract instance). Inactivity time means that if no new funding was added or spent by the owner during this time then the heir is able to spend the funds.

Specification of the rest two transactions':

#### add coins transaction:

* vins.*: normal inputs
* vout.0: funding CC 1of2 addr for the owner and heir
* vout.1: normal change
* vout.n-1: opreturn 'A' fundingtxid HasHeirSpendingBegun

This transaction add more funds on the vout with 1of2 cryptocondition from normal coin inputs.
In the opreturn the txid of the initial transaction is added to bind this tx to this contract instance (funding plan).
There is an additional flag HasHeirSpendingBegun, which is turned to true when the Heir first time spends the inherited funding and therefore no need to wait ever again for the owner inactivity time. If this flag is still `false` the contract validation would not allow the Heir to spend funds unless the inactivity time has passed. The cryptocondition features  does not allow to check timing so it is going to be the job for the Heir cc contract validation code to check this. 

Note also the functional id 'A' which marks this tx as 'add' funding transaction.

#### claim coins transaction:

* vin.0: normal input txfee
* vin.1+: input from CC 1of2 addr
* vout.0: normal output to owner or heir address
* vout.1: change to CC 1of2 addr
* vout.2: change to user's addr from txfee input if any
* vout.n-1: opreturn 'C' fundingtx HasHeirSpendingBegun

This tx allows to spend funds by either the funds owner or heir. It nas a normal input to add value for txfee for miners, a cc input foor spending the claimed value from 1of2 fund address.
As to outputs, the claimed value is sent to claimer's normal address, unspent change is returned to 1of2 address.
There is also the normal change.
Note the functional id 'C' in the opreturn. The other opreturn data are the same as in the 'add' ('A') transaction.

### Heir cc contract rpc implementation
Now let's develop rpc functions  to create the transactions describe in the previous section.

#### heirfund rpc implementation
To make a rpc call we would need its name and parameters. For rpc to create an initial tx I chose the name of 'heirfund' and its parameters are derived from the defined tx structure:
```
komodo-cli -ac_name=YOURCHAIN heirfund amount name heirpubkey inactivitytime
```
where 'amount' is coins going to the initial tx vout.1, 'name' is some description, 'heirpubkey' is heir pubkey and 'inactivitytime' is in seconds.

Now let's write some code for this rpc.

To add a new command to komodo-cli we need to change source file src/server.cpp: add a new element to vRPCCommands array:

`	{ "heir",       "heirfund",   &heirfund,      true },`

where "heir" is a common name for all heir contract rpc calls
"heirfund" is the name of the new command and '&heirfund' is the address of rpc interface function,
true means that the command description will be shown in the help command output ('false' is to hide it)

It is also necessary to add `heirfund` rpc function definition into the rpc/server.h source file. All rpc functions have pretty much the same declaration `UniValue heirfund(const UniValue& params, bool fHelp)`.

An rpc function implementation is actually two-level.
The first level is short rpc function itself like `heirfund`. Its body is added in an rpc source file in rpc/ subdirectory. It is short and just checks rpc parameters and needed environment and forward the call to the transacion creation code.
I created rpc-level code in the wallet/rpcwallet.cpp file although it is much better to create a new rpc source file for this functions.

First rpc-level implementation
```
UniValue heirfund(const UniValue& params, bool fHelp)
{
    CCerror.clear(); // clear global error object
```
Check that the wallet and heir cc contract are available
and check the rpc params' required number:
```
    if (!EnsureWalletIsAvailable(fHelp))
        return NullUniValue;
    if (ensure_CCrequirements(EVAL_HEIR) < 0)
        throw runtime_error("to use CC contracts, you need to launch daemon with valid -pubkey= for an address in your wallet\n");
    // output help message if asked or params count is incorrect:
    if (fHelp || params.size() != 4 )
        throw runtime_error("heirfund funds heirname heirpubkey inactivitytime\n");
```
The UniValue object is a special type for passing data in rpc calls. Univalue params is actually an array of Univalue objects. We still need to convert them into usual c/c++ language types and pass to the contract implementation.
Next convert the params from UniValue type to the basic c++ types. 
```
    CAmount amount = atof(params[0].get_str().c_str()) * COIN;  // Note conversion to satoshis by multiplication on 10E8
    std::string name = params[1].get_str();
    std::vector<uint8_t> vheirpubkey = ParseHex(params[2].get_str().c_str());
    CPubKey heirpk = pubkey2pk(vheirpubkey);
    int64_t inactivitytime = atoll(params[3].get_str().c_str());
```
We also need to add checking the converted param values are correct (what I ommitted in the sample), for example not negative or do not exceed some limit.
Note how to parse hex representation of the pubkey param and convert it to CPubKey object.

And now time to call the heir cc contract code and pass the returned created tx in hexademical representation to the caller, ready to be sent to the chain:
```
    UniValue result = HeirFund(amount, name, heirpk, inactivitytime);
    RETURN_IF_ERROR(CCerror);  // use a macro to throw runtime_error if CCerror is set in HeirFund()
    return result;
}
```

The second implementation level is located in the heir cc contract source file src/heir.cpp.
Here is the skeleton of the heirfund rpc implementation.
First, we need to create a mutable version of a transaction object.
```
std::string HeirFund(int64_t amount, std::string heirName, CPubKey heirPubkey, int64_t inactivityTimeSec)
{
    CMutableTransaction mtx = CreateNewContextualCMutableTransaction(Params().GetConsensus(), komodo_nextheight());
```

Declare and initialize an CCcontract_info object with heir cc contract variables like cc global address, global private key etc.
```
    struct CCcontract_info *cp, C;
    cp = CCinit(&C, EVAL_HEIR);
```
Next we need to add inputs to transaction enough to make deposit of the amount to the heir fund, marker and miners
Let's use a constant fee = 10000 sat.
We need the pubkey from the komodod -pubkey param.
For adding normal inputs to the mutable transaction there is a corresponding function in the cc SDK. 
```
    const int64_t txfee = 10000;
    CPubKey myPubkey = pubkey2pk(Mypubkey());   
    if (AddNormalinputs(mtx, myPubkey, amount+2*txfee , 60) > 0) { 
```
The parameters passed to the AddNormalinputs() are the tx itself, my pubkey, total value for the funding amount, marker and miners fee, for which the function will add the necessary number of uxto from the user's wallet. The last parameter is the limit of uxto to add. 

Now let's add outputs to the transaction. Accordingly to our specification we need the two outputs: for the funding deposit and marker
```
        mtx.vout.push_back( MakeCC1of2vout(amount, myPubkey, heirPubkey) );
        mtx.vout.push_back( MakeCC1vout(EVAL_HEIR, txfee, cp->GetUnspendable()) );
```
In this example we used two cc sdk functions for creating cryptocondition vouts. 

MakeCC1of2vout creates a vout with a threshold=2 cryptocondition allowing to spend funds from this vout with either myPubkey (which would be the pubkey of the funds owner) or heir pubkey

MakeCC1vout creates a vout with a simple cryptocondition which sends a txfee to cc Heir contract global address (returned by cp->GetUnspendable() function call). We need this output to be able to find all the created heir funding plans. 
You will always need some kind of marker for any cc contract at least for the initial transaction, otherwise you might lose contract's data in blockchain.
We may call this as **marker pattern** in cc development. See more about the marker pattern later in the CC contract patterns section.

Finishing creation of the transaction by calling FinalizeCCTx with passing to it the mtx object, the owner pubkey, txfee amount. Also just created opreturn object with the contract data is passed. It is created by serializing the needed variables to the CScript object.
Also, if AddNormalinputs cannot find owner funds set error object.
```
        std::string rawhextx = FinalizeCCTx(0, cp, mtx, myPubkey, txfee,
            CScript() << E_MARSHAL(ss << (uint8_t)EVAL_HEIR << (uint8_t)'F' << myPubkey << heirPubkey << inactivityTimeSec << heirName));
        return rawhextx;
    }
    CCerror = "not enough coins for requested amount and txfee";
    return std::string("");
}
```
Note that we do not need to add the normal change output here because this job is done by FinalizeCCTx.
FinalizeCCTx also builds the transaction input scriptSigs (both normal and cryptocondition), adds signatures and returns signed transaction in hex.
Also note E_MARSHAL function which serializes variables of various types to a byte array which in turn is serialized to CScript object which is stored in scriptPubKey transaction field. There is also the mirror E_UNMARSHAL function.

The returned transaction is ready to be sent to the blockchain with sendrawtransaction rpc.

Now let's develop an rpc for claiming the funds.

#### heirclaim implementation

Again, this implementation will be two-level, with the first level to check the required environment and converting the parameters and the second level to create claiming transaction.
the heirclaim rpc call will be very simple:
```
komodo-cli -ac_name=YOURCHAIN heirclaim fundingtxid amount
```


Add a new command to komodo-cli by adding a new element in to vRPCCommands array in the source file src/server.cpp::

`	{ "heir",       "heirclaim",   &heirclaim,      true },`

Add a heirclaim rpc call implementation (in rpc/wallet.cpp)
Add the heirclaim declaration in rpc/server.h header file.

```
UniValue heirclaim(const UniValue& params, bool fHelp)
{
    CCerror.clear(); // clear global error object
```
check that wallet is available. 
if asked for help or param size incorrect return help message.
Aslo check cc contract requirements are satisfied
```
    if (!EnsureWalletIsAvailable(fHelp))
        return NullUniValue;
    if (fHelp || params.size() != 2)
	throw runtime_error("heirclaim txfee funds fundingtxid\n");
    if (ensure_CCrequirements(EVAL_HEIR) < 0)
	throw runtime_error("to use CC contracts, you need to launch daemon with valid -pubkey= for an address in your wallet\n");
```
Lock the wallet
```
    LOCK2(cs_main, pwalletMain->cs_wallet);
```
convert the parameters from UniValue to c++ types and call tx creation function
```
    fundingtxid = Parseuint256((char*)params[0].get_str().c_str());
    CAmount amount = atof(params[1].get_str().c_str()) * COIN;  // Note conversion to satoshis by multiplication on 10E8

    UniValue result = HeirClaimCaller(fundingtxid, amount);
    RETURN_IF_ERROR(CCerror);  // use a macro to throw runtime_error if CCerror is set in HeirFund()
    return result;
}
```

Now implement the tx creation code (second level).
Start with creating a mutable transaction object:
```
std::string HeirClaim(uint256 fundingtxid, int64_t amount)
{
    CMutableTransaction mtx = CreateNewContextualCMutableTransaction(Params().GetConsensus(), komodo_nextheight());
```
Next, init the cc contract object:
```
    struct CCcontract_info *cp, C;
    cp = CCinit(&C, EVAL_HEIR);
```
Now we need to find the latest owner transaction to calculate the owner's inactivity time:
Use a helper FindLatestOwnerTx function which returns the lastest txid, heir public key and the hasHeirSpendingBegun flag value:
```
    uint8_t hasHeirSpendingBegun;
    CPubKey ownerPubkey, heirPubkey;
    uint256 latesttxid = FindLatestOwnerTx(fundingtxid, ownerPubkey, heirPubkey, hasHeirSpendingBegun);
    if( latesttxid.IsNull() )   {
        CCerror = "no funding tx found";
        return "";
    }
```
Now check if inactivity time has reached from the last owner transaction. Use cc sdk function which returns time in seconds from the block with the txid in the params:
```
    int32_t numblocks; // not used
    bool isAllowedToHeir = (hasHeirSpendingBegun || CCduration(numBlocks, latesttxid) > inactivityTimeSec) ? true : false;
    CPubKey myPubkey = pubkey2pk(Mypubkey());  // pubkey2pk sdk function convert pubkey from byte array to high-level CPubKey object
    if( myPubkey == heirPubKey && !isAllowedToHeir )    {
        CCerror = "spending funds is not allowed for heir yet";
        return "";
    }
```
Let's create the claim transaction inputs and outputs.
add normal inputs for txfee:
```
    if (AddNormalinputs(mtx, myPubkey, txfee, 3) <= txfee)    {
        CCerror = "not enough normal inputs for txfee";
        return "";
    }

```
Add cc inputs for the requested amount.
first get the address 1 of 2 cryptocondition where the fund was deposited:
```
    char coinaddr[65];
    GetCCaddress1of2(cp, coinaddr, ownerPubkey, heirPubkey);
```
and add inputs for this address with an additional custom function:
```
    int64_t inputs;
    if( (inputs = Add1of2AddressInputs(mtx, fundingtxid, coinaddr, amount, 64)) < amount )   {
        CCerror = "not enough funds claimed";
        return "";
    }
```
Now add an normal output to send claimed funds to and cc change output for fund remainder:
```
    mtx.vout.push_back(CTxOut(amount, CScript() << ParseHex(HexStr(myPubkey)) << OP_CHECKSIG));  
    if (inputs > amount)
        mtx.vout.push_back(MakeCC1of2vout(inputs - amount, ownerPubkey, heirPubkey));  
```
add normal change if any, add opreturn data and sign the transaction
```
     return FinalizeCCTx(0, cp, mtx, myPubkey, txfee, CScript() << E_MARSHAL(ss << (uint8_t)EVAL_HEIR << (uint8_t)'C' << fundingtxid << (myPubkey == heirPubkey) ? (uint8_t)1 : hasHeirSpendingBegun));
}         
```

#### Simplified Add1of2AddressInputs function implementation
```
int64_t Add1of2AddressInputs(CMutableTransaction &mtx, uint256 fundingtxid, char *coinaddr, int64_t amount, int32_t maxinputs)
{
    int64_t totalinputs = 0L;
    int32_t count = 0;
```
go through uxtos and add them to the transaction vins:
```
    std::vector<std::pair<CAddressUnspentKey, CAddressUnspentValue>> unspentOutputs;
    SetCCunspents(unspentOutputs, coinaddr, true);  // get a vector of uxtos for the address in coinaddr[]
    for (std::vector<std::pair<CAddressUnspentKey, CAddressUnspentValue>>::const_iterator it = unspentOutputs.begin(); it != unspentOutputs.end(); it++) {
         CTransaction tx;
         uint256 hashBlock;
         std::vector<uint8_t> vopret;
         if (GetTransaction(it->first.txhash, tx, hashBlock, false) && tx.vout.size() > 0 && GetOpreturn(tx.back().scriptPubKey, vopret) && vopret.size() > 2)
         {
              uint8_t evalCode, funcId, hasHeirSpendingBegun;
              uint256 txid;
              if( it->first.txhash == fundingtxid ||   // if it is the initial 
                  E_UNMARSHAL(vopret, { ss >> evalCode; ss >> funcId; ss >> txid >> hasHeirSpendingBegun; }) && // unserialize opreturn
                  fundingtxid == txid  ) // it is a tx from this funding plan
              {
                  mtx.vin.push_back(CTxIn(txid, voutIndex, CScript()));
                  totalinputs += it->second.satoshis;    
                  if( totalinputs >= amount || ++count > maxinputs )
                      break;
              }
         } 
    }
```
Return the added total inputs amount:
```
    return totalinputs;
}
```
#### Simplified FindLatestOwnerTx implementation
TODO...


#### Simplified validation function implementation
TODO...

## Some used terminology

Here is some terms and keywords which are used in the cc documentation, developers chats and cc contracts code 
* cryptocondition - in simple terms an encoded expression and supporting c-library which allows to check several types of logical conditions based on electronic signatures and hashes. Frequently used cryptocondition is checking that some transaction is signed by one of several possible keys.
* cc contract - custom consensus contract (rebranded cryptocondition contract)
* cc input - tx cryptocondition input spending value from cc output
* cc output - tx output with cryptocondition encoded
* normal inputs - inputs spending value from normal tx outputs (not cryptocondition outputs)
* normal outputs - not cryptocondition outputs but usual tx outputs like pubkey or pubkeyhash
* opreturn, opret - a special output item in transaction, usually the last one, which is prepended by OP_RETURN script opcode and therefore is never able to spend. We put user and contract data into it.
* plan - an instance of cc contract data, that is, a set of all the relevant transactions, usually beginning from an initial tx whose txid serves a whole plan's id. Any cc contract has a list function to return a list of all the created plans.
* tx, txn - short of `transaction`
* txid - transaction id, a hash of a transaction
* 'unspendable' address - the global cc contract address, for which its public and private key are commonly known. It is used for conditionally sharing funds between contract users. As its private key is not a secret theoretically, anyone can request spending value from this address, but cc contract validation code might and usually does apply various business logic conditions and checks about who can spend the funds from the and what he or she should provide to be able to do this (usually the spending transaction and/or its previous transactions are checked)
* vin - input or array of inputs in transaction structure (tx.vin)
* vout - output or array of outputs in transaction structure (tx.vout)

## CC contract patterns

Earlier I wrote about the marker pattern. Here I'd like collect other useful patterns (including the marker pattern too) which could be used in the development of custom consensus contracts.

### Baton pattern
Baton pattern allows to organize a single-linked list in a blockchain. Starting from the first tx in a list we may iterate the other transactions in the list and retrieve the values stored in their opreturns.

### Marker pattern
Marker pattern is used to mark all similar transactions by sending some small value to a common fixed address (most commonly it is a cc contract global address).
Actually you can create either normal marker or cc marker for future finding the transactions.
If you create a normal marker you cannot control that someone can spend the value from it (as cc global address private key is not a secret). As such marker could be spent you should use cc sdk function Settxids() function to retrieve all transaction with markers in the cc contract list function.

We may use another tecnique for marker and use an unspendable cc marker by using some value to a cc output into some known address. In this case for retrieving transaction list you may use another cc sdk function SetCCunspents which returns a list of transactions with unspent outputs for the appointed address.
If you decide to use an unspendable cc marker you should disable its spending in the validation code otherwise as in some predefined way. If, for example, you allow spending from it by a burning tx you would have such burned markers hidden from the list of initial transactions.
The cc contract validation code should disable unauthorised tries to spend such markers.
An issue with the cc marker case which needs to be considered is if the cc global address is used for storing not only the marker value but other funds you would need to provide that the marker value . 

Code example for finding the transactions marked with normal marker:
```
    std::vector<std::pair<CAddressIndexKey, CAmount> > addressIndex;
    struct CCcontract_info *cp, C;
    cp = CCinit(&C, <some eval code>);
    SetCCtxids(addressIndex, cp->normaladdr, false);                      
    for (std::vector<std::pair<CAddressIndexKey, CAmount> >::const_iterator it = addressIndex.begin(); it != addressIndex.end(); it++) 	{
        CTransaction vintx;
        uint256 blockHash;
        if( GetTransaction(it->first.txhash, vintx, blockHash, false) ) {
            // check tx and add its txid to a list
        }
    }
```
You may find more examples how to find marked transactions in any cc contract standard list function.
Note for SetCCtxids() function to work, the komodod `txindex` init parameter should be set to 1 (which is by default).

Code example for finding transactions marked with an unspendable cc marker:
```
    std::vector<std::pair<CAddressIndexKey, CAmount> > addressIndex;
    struct CCcontract_info *cp, C;
    cp = CCinit(&C, <some eval code>);
    SetCCunspents(addressIndexCCMarker, cp->unspendableCCaddr, true);    
    for (std::vector<std::pair<CAddressUnspentKey, CAddressUnspentValue> >::const_iterator it = addressIndexCCMarker.begin(); it != addressIndexCCMarker.end(); it++) {
        CTransaction vintx;
        uint256 blockHash;
        if( GetTransaction(it->first.txhash, vintx, hashBlock, false) ) {
            // check tx and add its txid to a list
        }
    }
```
For CCunspents function to work komodod init parameters `addressindex` and `spentindex` should be both set to '1'

### Txidaddress pattern
Txidaddress pattern might be used when you need to send some value to an address which never could be spent. For this there is a function to get an address for which no private key ever exists
In payments CC we use this for a spendable address, by using:
CTxOut MakeCC1of2vout(uint8_t evalcode,CAmount nValue,CPubKey pk1,CPubKey pk2, std::vector<std::vector<unsigned char>>* vData)

For the merge RPC we use the vData optional parameter to append the op_return directly to the ccvout itself, rather than an actual op_return as the last vout in a transaction. Like so: 
```
opret = EncodePaymentsMergeOpRet(createtxid);
std::vector<std::vector<unsigned char>> vData = std::vector<std::vector<unsigned char>>();
if ( makeCCopret(opret, vData) )
    mtx.vout.push_back(MakeCC1of2vout(EVAL_PAYMENTS,inputsum-PAYMENTS_TXFEE,Paymentspk,txidpk,&vData));
GetCCaddress1of2(cp,destaddr,Paymentspk,txidpk);
CCaddr1of2set(cp,Paymentspk,txidpk,cp->CCpriv,destaddr);
rawtx = FinalizeCCTx(0,cp,mtx,mypk,PAYMENTS_TXFEE,CScript());
```
This allows a payments ccvout to be spent back to its own address, without needing a markervout or an OP_RETURN by using a modification made to IsPaymentsvout:
```
int64_t IsPaymentsvout(struct CCcontract_info *cp,const CTransaction& tx,int32_t v,char *cmpaddr, CScript &ccopret)
{
    char destaddr[64];
    if ( getCCopret(tx.vout[v].scriptPubKey, ccopret) )
    {
        if ( Getscriptaddress(destaddr,tx.vout[v].scriptPubKey) > 0 && (cmpaddr[0] == 0 || strcmp(destaddr,cmpaddr) == 0) )
            return(tx.vout[v].nValue);
    }
    return(0);
}
```
 
In place of IsPayToCryptoCondition we can use getCCopret function which is a lower level of the former call, that will return us any vData appended to the ccvout along with true/false for IsPayToCryptoCondition. 
In validation we now have a totally diffrent transaction type than exists allowing to have diffrent validation paths for diffrent ccvouts. And also allowing multiple ccvouts of diffrent types per transaction.
``` 
if ( tx.vout.size() == 1 )
{
    if ( IsPaymentsvout(cp,tx,0,coinaddr,ccopret) != 0 && ccopret.size() > 2 && DecodePaymentsMergeOpRet(ccopret,createtxid) )
    {
        fIsMerge = true;
    } else return(eval->Invalid("not enough vouts"));
}
```

## Various tip and trick in cc contract development
### Test chain mining issue
On a test chain consisting of two nodes do not mine on both nodes - the chain might get out of sync. It is recommended to have only one miner node for two-node test chains.

### Try not to do more than one AddNormalInputs call in one tx creation code
FillSell function calls AddNormalInputs two times at once: at the first time it adds txfee, at the second time it adds some coins to pay for tokens. 
I had only 2 uxtos in my wallet: for 10,000 and 9 ,0000,000 sat. Seems my large uxto has been added during the first call and I receive 'filltx not enough utxos' message after the second call. I think it is always better to combine these calls into a single call.

### Nodes in your test chain are not syncing
You deployed a new or updated developed cc contract in Komodo daemon and see a node could not sync with other nodes in your network (`komodo-cli ac_name=YOURCHAIN getpeerinfo` shows synced blocks less than synced heaeds). It might be seen errors in console log.
Most commonly it is the cc contract validation code problem. Your might have changed the validation rules and old transactions might become invalid. It is easy to get into this if you try to resync a node from scratch. In this case old transactions should undergo the validation.
Use validation code logging and gdb debug to investigate which is the failing rule.
If you really do not want to validate old transactions you might set up the chain height at which a rule begin to action with the code like this:
if (strcmp(ASSETCHAINS_SYMBOL, "YOURCHAIN") == 0 && chainActive.Height() <= 501)
    return true;
Use hidden `reconsiderblock` komodo-cli command to restart syncing from the block with the transaction which failed validation.

### Deadlocks in validation code
If komodod hangs in cc contract validation code you should know that some blockchain functions use locks and might cause deadlocks. You should check with this functions and use non-locking versions.
An example of such function is GetTransaction(). Instead you should use myGetTransaction() or eval->GetConfirmed()

## TODO
### Contract architecture extending for token support
My contract should work both with coins and tokens. The program logic for inheritance coins and tokens was very the same, so I used templates for contract functions which were extended in specific points to deal either with coins or tokens (like to make opreturn and cc vouts)

### Contract validation code description
For validation of Heir cc contract tx I implemented a small object-oriented 'chain of responsibility' framework.
It allows to develop a small vin or vout validator programs independently on the tx structure and set up binding of the validators to tx vin/vout and vintx vout structure. It also provides re-use of validators.

_to be continued..._

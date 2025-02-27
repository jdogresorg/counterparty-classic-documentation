# counterparty-server API

[TOC]


##Overview

``counterparty-lib`` provides a JSON RPC 2.0-based API based off of
that of Bitcoin Core. It is the primary means by which other applications
should interact with the Counterparty network.

The API server is started either through the [`CLI interface`](counterparty-cli.md) or
with the [`counterparty-lib`](counterparty_lib.md) Python library. It listens on port
4000 by default (14000 for ``testnet``) and requires HTTP Basic Authentication to connect.

The API includes numerous information retrieval methods, most of which begin with `get_`, as well as several
`create_` methods which create new Counterparty transactions. While the `get_` type methods simply return the
requested information, the `create_` methods return unsigned raw transactions which must then be signed and
broadcast on the Bitcoin network. This means that while `counterparty-server` requires Bitcoin Core and
uses it for retieval and parsing of blockchain data, it and this API do not require Bitcoin Core's wallet functionality
for private key storage and transaction signing. Transaction signing and broadcast can thus
be accomplished using whatever means the developer sees fit (including using Bitcoin core's APIs if desired, or
a library like Bitcore, or a service like blockchain.info, and so on).

In addition to the JSON RPC API, ``counterparty-lib`` provides a complementary RESTful API also based off of that
of Bitcoin Core's design. This REST API is still under development and will include more functionality
in the future, and listens on the same port as JSON RPC one.


##Getting Started

By default, the server will listen on port ``4000`` (if on mainnet) or port ``14000`` (on testnet) for API
requests.

Note that the main API is built on JSON-RPC 2.0, not 1.1. JSON-RPC itself is pretty lightweight, and API requests
are made via a HTTP POST request to ``/api/`` (note the trailing slash), with JSON-encoded data passed as the POST body.

The requests to the secondary REST API are made via HTTP GET to ``/rest/``, with request action and parameters encoded in the URL.


###General Format

####JSON-RPC

All requests must have POST data that is JSON encoded. Here's an example of the POST data for a valid API request:

    {
      "method": "get_sends",
      "params": {"order_by": "tx_hash",
                 "order_dir": "asc",
                 "start_block": 280537,
                 "end_block": 280539},
      "jsonrpc": "2.0",
      "id": 0
    }

The ``jsonrpc`` and ``id`` properties are requirements under the JSON-RPC 2.0 spec.

You should note that the data in ``params`` is a JSON object (e.g. mapping), not an array. In other words,
**the API only supports named arguments, not positional arguments** (e.g. use
{"argument1": "value1", "argument2": "value2"} instead of ["value1", "value2"]). This is the case for safety and bug-minimization reasons.

For more information on JSON RPC, please see the [JSON RPC 2.0 specification](http://www.jsonrpc.org/specification).

####REST

For REST API all requests are made via GET where query-specific arguments are encoded as URL parameters. Moreover, the same requests can be passed via HTTP POST in order to encrypt the transaction parameters. There are only two methods supported: ``get`` and ``compose``. The URL formats are as follows, respectively:
`/rest/<table_name>/get?<filters>&op=<operator>`
`/rest/<message_type>/compose?<transaction arguments>`

###Authentication

The API support HTTP basic authentication to use, which is enabled if and only
if a password is set. **The default user is ``'rpc'``.**


##Example Implementations for JSON RPC API

The following examples have authentication enabled and the `user` set to its
default value of `'rpc'`. The password is not set (default: `'rpc'`). Ensure
these values correspond to values in your counterparty-server's configuration
file `'server.conf'`.

Submissions of examples in additional languages are welcome!

###Python

    import json
    import requests
    from requests.auth import HTTPBasicAuth

    url = "http://localhost:4000/api/"
    headers = {'content-type': 'application/json'}
    auth = HTTPBasicAuth('rpc', PASSWORD)

    payload = {
      "method": "get_running_info",
      "params": {},
      "jsonrpc": "2.0",
      "id": 0
    }
    response = requests.post(url, data=json.dumps(payload), headers=headers, auth=auth)
    print("Response: ", response.text)


###PHP

With PHP, you use the [JsonRPC](https://github.com/fguillot/JsonRPC)
library.

    <?php
    require 'JsonRPC/src/JsonRPC/Client.php';
    use JsonRPC\Client;
    $client = new Client('http://localhost:4000/api/');
    $client->authentication('rpc', PASSWORD);

    $result = $client->execute('get_balances', array('filters' => array('field' => 'address', 'op' => '==', 'value' => '1NFeBp9s5aQ1iZ26uWyiK2AYUXHxs7bFmB')));
    print("get_balances result:\n");
    var_dump($result);

    $result2 = $client->execute('get_running_info');
    print("get_running_info result:\n");
    var_dump($result2);
    ?>

###curl

Remember to surround non-numeric parameter values with the double quotes, as per [JSON-RPC 2.0 examples](http://www.jsonrpc.org/specification#examples). For example, `"order_by": "tx_hash"` is correct and will work, `"order_by": 'tx_hash'` won't.

####Linux

    curl -X POST http://127.0.0.1:4000/api/ --user rpc:$PASSWORD -H 'Content-Type: application/json; charset=UTF-8' -H 'Accept: application/json, text/javascript' --data-binary '{ "jsonrpc": "2.0", "id": 0, "method": "get_running_info" }'

####Windows

On Windows, depending on implementation the above curl command may need to be formatted differently due to problems that Windows has with escapes. For example this particular format was found to work with curl 7.50.1 (x86_64-w64-mingw32) on Windows 10 (x64).

    curl -X POST http://127.0.0.1:4000/api/ --user rpc:$PASSWORD -H "Content-Type: application/json; charset=UTF-8" -H "Accept: application/json, text/javascript" --data-binary "{ \"jsonrpc\": \"2.0\", \"id\": 0, \"method\": \"get_running_info\" }"

###c# (RestSharp)

Authorization string in the example below is based on the default username/password.

    var client = new RestClient("http://127.0.0.1:4000/api/");
    var request = new RestRequest(Method.POST);
    request.AddHeader("cache-control", "no-cache");
    request.AddHeader("authorization", "Basic cnBjOjEyMzQ=");
    request.AddHeader("content-type", "application/json");
    request.AddParameter("application/json", "{\r\n  \"method\": \"get_running_info\",\r\n  \"params\": {},\r\n  \"jsonrpc\": \"2.0\",\r\n  \"id\": 1\r\n}", ParameterType.RequestBody);
    IRestResponse response = client.Execute(request);

###Go

Authorization string in the example below is based on the default username/password.

    package main

    import (
    	"fmt"
    	"strings"
    	"net/http"
    	"io/ioutil"
    )

    func main() {

    	url := "http://127.0.0.1:4000/api/"

    	payload := strings.NewReader("{\r\n  \"method\": \"get_running_info\",\r\n  \"params\": {},\r\n  \"jsonrpc\": \"2.0\",\r\n  \"id\": 1\r\n}")

    	req, _ := http.NewRequest("POST", url, payload)

    	req.Header.Add("content-type", "application/json")
    	req.Header.Add("authorization", "Basic cnBjOjEyMzQ=")
    	req.Header.Add("cache-control", "no-cache")

    	res, _ := http.DefaultClient.Do(req)

    	defer res.Body.Close()
    	body, _ := ioutil.ReadAll(res.Body)

    	fmt.Println(res)
    	fmt.Println(string(body))

    }

###Ruby (Net::HTTP)

Authorization string in the example below is based on the default username/password.

    require 'uri'
    require 'net/http'

    url = URI("http://127.0.0.1:4000/api/")

    http = Net::HTTP.new(url.host, url.port)

    request = Net::HTTP::Post.new(url)
    request["content-type"] = 'application/json'
    request["authorization"] = 'Basic cnBjOjEyMzQ='
    request["cache-control"] = 'no-cache'
    request.body = "{\r\n  \"method\": \"get_running_info\",\r\n  \"params\": {},\r\n  \"jsonrpc\": \"2.0\",\r\n  \"id\": 1\r\n}"

    response = http.request(request)
    puts response.read_body


##Example Implementations for REST API

The following examples don't use authentication as with default settings.

###Python

    import requests

    url = "http://localhost:4000/rest/"
    headers = {'content-type': 'application/json'}

    query = 'sends/get?source=mn6q3dS2EnDUx3bmyWc6D4szJNVGtaR7zc&destination=mtQheFaSfWELRB2MyMBaiWjdDm6ux9Ezns&op=AND'

    response = requests.get(url + query, headers=headers)
    print("Response: ", response.text)


###curl

These examples use the default username/password combination in URL.

####Linux

    curl "http://rpc:rpc@127.0.0.1:4000/rest/sends/get?source=1B6ahDHnKtZ5GXqytHSxfcXgNoxm1q1RsP&destination=14fAoS9FPD9jx36hjCNoAqFVLNHD1NQVN5&op=AND" -H "Content-Type: application/json; charset=UTF-8" -H "Accept: application/json"

####Windows

This example was created with curl 7.50.1 (x86_64-w64-mingw32) on Windows 10. For POST encryption add `'-X POST'`.

    curl "http://rpc:rpc@127.0.0.1:4000/rest/sends/get?source=1B6ahDHnKtZ5GXqytHSxfcXgNoxm1q1RsP&destination=14fAoS9FPD9jx36hjCNoAqFVLNHD1NQVN5&op=AND" -H "Content-Type: application/json; charset=UTF-8" -H "Accept: application/json"

##Example Parameters

* Fetch all balances for all assets for both of two addresses, using keyword-based arguments

        payload = {
                   "method": "get_balances",
                   "params": {
                              "filters": [{"field": "address", "op": "==", "value": "14qqz8xpzzEtj6zLs3M1iASP7T4mj687yq"},
                                          {"field": "address", "op": "==", "value": "1bLockjTFXuSENM8fGdfNUaWqiM4GPe7V"}],
                              "filterop": "or"
                             },
                   "jsonrpc": "2.0",
                   "id": 0
                  }

* Get all burns between blocks 280537 and 280539 where greater than .2 BTC was burned, sorting by `tx_hash` (ascending order)

        payload = {
                   "method": "get_burns",
                   "params": {
                              "filters": {"field": "burned", "op": ">", "value": 20000000},
                              "filterop": "AND",
                              "order_by": "tx_hash",
                              "order_dir": "asc",
                              "start_block": 280537,
                              "end_block": 280539
                             },
                   "jsonrpc": "2.0",
                   "id": 0
                  }

* Fetch all debits for > 2 XCP between blocks 280537 and 280539, sorting the results by quantity (descending order)

        payload = {
                   "method": "get_debits",
                   "params": {
                              "filters": [{"field": "asset", "op": "==", "value": "XCP"},
                                          {"field": "quantity", "op": ">", "value": 200000000}],
                              "filterop": "AND",
                              "order_by": "quantity",
                              "order_dir": "desc"
                             },
                   "jsonrpc": "2.0",
                   "id": 0
                  }


* Send 1 XCP (specified in satoshis) from one address to another.

        payload = {
                   "method": "create_send",
                   "params": {
                              "source": "1CUdFmgK9trTNZHALfqGvd8d6nUZqH2AAf",
                              "destination": "17rRm52PYGkntcJxD2yQF9jQqRS4S2nZ7E",
                              "asset": "XCP",
                              "quantity": 100000000
                             },
                   "jsonrpc": "2.0",
                   "id": 0
                  }

* Issuance (indivisible)

        payload = {
                   "method": "create_issuance",
                   "params": {
                              "source": "1CUdFmgK9trTNZHALfqGvd8d6nUZqH2AAf",
                              "asset": "MYASSET",
                              "quantity": 1000,
                              "description": "my asset is cool",
                              "divisible": false
                             },
                   "jsonrpc": "2.0",
                   "id": 0
                  }

* Transfer asset ownership

        payload = {
                   "method": "create_issuance",
                   "params": {
                              "source": "1CUdFmgK9trTNZHALfqGvd8d6nUZqH2AAf",
                              "transfer_destination": "17rRm52PYGkntcJxD2yQF9jQqRS4S2nZ7E",
                              "asset": "MYASSET",
                              "quantity": 0
                             },
                   "jsonrpc": "2.0",
                   "id": 0
                  }

* Lock asset

        payload = {
                   "method": "create_issuance",
                   "params": {
                              "source": "1CUdFmgK9trTNZHALfqGvd8d6nUZqH2AAf",
                              "asset": "MYASSET",
                              "quantity": 0,
                              "description": "LOCK"
                             },
                   "jsonrpc": "2.0",
                   "id": 0
                  }


##Signing Transactions Before Broadcasting

**Note:** Before v9.49.4, the counterparty server API provided an interface to Bitcoin Core's signing functionality through the `do_*`, `sign_tx` and `broadcast_tx` methods, which have all since been removed.

All ``create_`` API calls return an *unsigned raw transaction serialization* as a hex-encoded string (i.e. the same format that ``bitcoind`` returns with its raw transaction API calls). This raw transaction's inputs may be validated and then must be signed (i.e. via Bitcoin Core, a 3rd party Bitcoin library like Bitcore, etc) and broadcast on the Bitcoin network.

The process of signing and broadcasting a transaction, from start to finish, depends somewhat on the wallet software used. Below are examples of how one might use a wallet to sign and broadcast an unsigned Counterparty transaction *created* with this API.

**Bitcoin Core with Python**

	#! /usr/bin/env python3

	from counterpartylib.lib import util
	from counterpartylib.lib import config
	from counterpartylib.lib.backend import addrindex

	config.TESTNET =
	config.RPC =
	config.BACKEND_URL =
	config.BACKEND_SSL_NO_VERIFY =

	def counterparty_api(method, params):
	    return util.api(method, params)

	def bitcoin_api(method, params):
	    return addrindex.rpc(method, params)

	def do_send(source, destination, asset, quantity, fee, encoding):
	    validateaddress = bitcoin_api('validateaddress', [source])
	    assert validateaddress['ismine']
	    pubkey = validateaddress['pubkey']
	    unsigned_tx = counterparty_api('create_send', {'source': source, 'destination': destination, 'asset': asset, 'quantity': quantity, 'pubkey': pubkey, 'allow_unconfirmed_inputs': True})
	    signed_tx = bitcoin_api('signrawtransaction', [unsigned_tx])['hex']
	    tx_hash = bitcoin_api('sendrawtransaction', [signed_tx])
	    return tx_hash

**Bitcoin Core with Javascript**
(Utilizing the [Counterwallet Bitcore wrapper code](https://raw.githubusercontent.com/CounterpartyXCP/counterwallet/master/src/js/util.bitcore.js) for brevity.)

    <html>
        <script src="https://raw.githubusercontent.com/bitpay/bitcore-lib/f031e1ddfbf0064ef503a28aada86c4fbf9a414c/bitcore-lib.min.js"></script>
        <script src="https://raw.githubusercontent.com/CounterpartyXCP/counterwallet/master/src/js/util.bitcore.js"></script>
        <script src="https://raw.githubusercontent.com/CounterpartyXCP/counterwallet/master/src/js/external/mnemonic.js"></script>
        <script>
        counterparty_api = function(method, params) {
            // call Counterparty API method via your prefered method
        }

        bitcoin_api = function(method, params) {
            // call Bitcoin Core API method via your prefered method
        }

        // generate a passphrase
        var m = new Mnemonic(128); //128 bits of entropy (12 word passphrase)
        var words = m.toWords();
        var passphrase = words.join(' ')

        // generate private key, public key and address from the passphrase
        wallet = new CWHierarchicalKey(passphrase);
        var cwk = wallet.getAddressKey(i); // i the number of the address
        var source = key.getAddress();
        var pubkey = cwk.getPub()

        // generate unsigned transaction
        unsigned_hex = counterparty_api('create_send', {'source': source, 'destination': destination, 'asset': asset, 'quantity': quantity, 'pubkey': pubkey})

        CWBitcore.signRawTransaction2(self.unsignedTx(), cwk, function(signedHex) {
            bitcoin_api('sendrawtransaction', signedHex)
        })
        </script>
    </html>

**Bitcoinjs-lib on javascript, signing a P2SH redeeming transaction**

```javascript
// Assumes NodeJS runtime. Several libraries exist to replace the Buffer class on web browsers
const bitcoin = require('bitcoinjs-lib')

async function signP2SHDataTX(wif, txHex) {
  const network = bitcoin.networks.testnet // Change appropiately to your used network
  const keyPair = bitcoin.ECPair.fromWIF(wif, network)
  const dataTx = bitcoin.Transaction.fromHex(txHex)   // The unsigned second part of the 2 part P2SH transactions

  const sigType = bitcoin.Transaction.SIGHASH_ALL // This shouldn't be changed unless you REALLY know what you're doing
  
  for (let i=0; i < dataTx.ins.length; i++) {
    const sigHash = dataTx.hashForSignature(i, bitcoin.script.decompile(dataTx.ins[i].script)[0], sigType)
    const sig = keyPair.sign(sigHash)
    const encodedSig = bitcoin.script.signature.encode(sig, sigType)
    const compiled = bitcoin.script.compile([encodedSig])

    dataTx.ins[i].script = Buffer.concat([compiled, dataTx.ins[i].script])
  }

  dataTx.ins[0].script = Buffer.concat([compiled, dataTx.ins[0].script])
  return dataTx.toHex() // The resulting signed transaction in raw hex, ready to be broadcasted
}
```

##Terms & Conventions

###assets

Everywhere in the API an asset is referenced by its name, not its ID. See the [Counterparty protocol specification](../protocol_specification#assets) for what constitutes a valid asset name.
Examples:

- "BTC"
- "XCP"
- "FOOBAR"
- "A7736697071037023001"

###subassets

See the [Counterparty protocol specification](../protocol_specification#subassets) for what constitutes a valid subasset name.
Examples:

- "PIZZA.X"
- "PIZZA.REALLY-long-VALID-Subasset-NAME"

###Quantities and balances

Anywhere where an quantity is specified, it is specified in **satoshis** (if a divisible asset), or as whole numbers
(if an indivisible asset). To convert satoshis to floating-point, simply cast to float and divide by 100,000,000.

Examples:

- 4381030000 = 43.8103 (if divisible asset)
- 4381030000 = 4381030000 (if indivisible asset)

**NOTE:** XCP and BTC themselves are divisible assets.

###floats

Floats are ratios or floating point values with six decimal places of precision, used in bets and dividends.

###Memos

See the [Counterparty protocol specification](../protocol_specification/#memos) for what constitutes a valid memo.
Examples:

- "for pizza"
- "1ca6"


##Miscellaneous

###Filtering Read API results

The Counterparty API aims to be as simple and flexible as possible. To this end, it includes a straightforward
way to filter the results of most [Read API](#read-api-function-reference) to get the data you want, and only that.

For each Read API function that supports it, a ``filters`` parameter exists. To apply a filter to a specific data field,
specify an object (e.g. dict in Python) as this parameter, with the following members:

- field: The field to filter on. Must be a valid field in the type of object being returned
- op: The comparison operation to perform. One of: ``"=="``, ``"!="``, ``">"``, ``"<"``, ``">="``, ``"<="``, ``"IN"``, ``"LIKE"``, ``"NOT IN"``, ``"NOT LIKE"``
- value: The value that the field will be compared against. Must be the same data type as the field is
  (e.g. if the field is a string, the value must be a string too)

If you want to filter by multiple fields, then you can specify a list of filter objects. To this end, API functions
that take ``filters`` also take a ``filterop`` parameter, which determines how the filters are combined when multiple
filters are specified. It defaults to ``"and"``, meaning that filters are ANDed togeher (and that any match
must satisfy all of them). You can also specify ``"or"`` as an alternative setting, which would mean that
filters are ORed together, and that any match must satisfy only one of them.

To disable filtering, you can just not specify the filter argument (if using keyword-based arguments), or,
if using positional arguments, just pass ``null`` or ``[]`` (empty list) for the parameter.

For examples of filtering in-use, please see the [examples](#example-parameters).

NOTE: Note that with strings being compared, operators like ``>=`` do a lexigraphic string comparison (which
compares, letter to letter, based on the ASCII ordering for individual characters. For more information on
the specific comparison logic used, please see [this page](http://www.sqlite.org/lang_expr.html).


#Technical Specification


##Read API Function Reference


###get_{table}

**get_{table}(filters=[], filterop='AND', order_by=null, order_dir=null, start_block=null, end_block=null, status=null, limit=1000, offset=0, show_expired=true)**

Where **{table}** must be one of the following values:
``assets``, ``balances``, ``bets``, ``bet_expirations``, ``bet_matches``, ``bet_match_expirations``, ``bet_match_resolutions``, ``broadcasts``, ``btcpays``, ``burns``, ``cancels``, ``credits``, ``debits``, ``destructions``,  ``dispensers``, ``dispenses``, ``dividends``, ``issuances``, ``mempool``, ``orders``, ``order_expirations``, ``order_matches``, ``order_match_expirations``, ``sends``, or ``transactions`` .

For example: ``get_balances``, ``get_credits``, ``get_debits`` are all valid API methods. A complete list of tables can be found in the api.py file in the counterparty-lib repository.

**Parameters:**

  * **filters** (*list/dict*): An optional filtering object, or list of filtering objects. See [filtering](#filtering-read-api-results) for more information.
  * **filterop** (*string*): Specifies how multiple filter settings are combined. Defaults to ``AND``, but ``OR`` can
    be specified as well. See [filtering](#filtering-read-api-results) for more information.
  * **order_by ** (*string*): If sorted results are desired, specify the name of an attribute of the appropriate table to
    order the results by (e.g. ``quantity`` for [balance object](#balance-object), if you called ``get_balances``).
    If left blank, the list of results will be returned unordered.
  * **order_dir** (*string*): The direction of the ordering. Either ``ASC`` for ascending order, or ``DESC`` for descending
    order. Must be set if ``order_by`` is specified. Leave blank if ``order_by`` is not specified.
  * **start_block** (*integer*): If specified, only results from the specified block index on will be returned
  * **end_block** (*integer*): If specified, only results up to and including the specified block index on will be returned
  * **status** (*string/list*): return only results with the specified status or statuses (if a list of status strings is supplied).
    See the [status list](#status). Note that if ``null`` is supplied (the default), then status is not filtered.
    Also note that status filtering can be done via the ``filters`` parameter, but doing it through this parameter is more
    flexible, as it essentially allows for situations where ``OR`` filter logic is desired, as well as status-based filtering.
  * **limit** (*integer*): (maximum) number of elements to return. Can specify a value less than or equal to the instance's max limit (default 1000).
    For more results, use a combination of ``limit`` and ``offset`` parameters to paginate results. If the instance has a limit of 0 you can specify
    any limit for the row results, even 0 to get the full dataset.
  * **offset** (*integer*): return results starting from specified ``offset``

**Special Parameters:**

  * **show_expired** (*boolean*): used only for ``get_orders``. When false, `get_orders` doesn't return orders which expire next block.
  * **memo_hex** (*string*): used only for ``get_sends``. When specified, filter the table for a hexadecimal value instead of searching by a text string.

**Special Results:**

  * **memo** (*string*): used only for ``get_sends``. The utf-8 encoded string of the memo (like ``for pizza``).  This value will be an empty string for hexadecimal-encoded memo IDs that are not valud UTF-8 strings.
  * **memo_hex** (*string*): used only for ``get_sends``. Returns the memo field expressed as a hexadecimal value (like ``666f722070697a7a61``).

**Return:**

  A list of objects with attributes corresponding to the queried table fields.

**Examples:**

  * To get a listing of bets, call ``get_bets``. This method will return a list of one or more [bet object](#bet-object) .
  * To get a listing all open orders for a given address like 1Ayw5aXXTnqYfS3LbguMCf9dxRqzbTVbjf, you could call
    ``get_orders`` with the appropriate parameters. This method will return a list of one or more order [object](#order-object).
  * To get all open "buy BTC" orders from the DEx, call ``get_orders`` and use the following filter: ``[{"field": "get_asset", "op": "==", "value": "BTC"}, {"field": "status", "op": "==", "value": "open"}]``.
  * To get all BTC pays (for DEx order matches) between the source 1Ayw5aXXTnqYfS3LbguMCf9dxRqzbTVbjf and destination (BTC buyer) 193SB3xgYjmfesdRqXq4g3eG9rD9DmWBSD, use `get_btcpays` method with these parameters: ``{ "filters": [{"field": "source", "op": "==", "value": "1Ayw5aXXTnqYfS3LbguMCf9dxRqzbTVbjf"}, {"field": "destination", "op": "==", "value": "193SB3xgYjmfesdRqXq4g3eG9rD9DmWBSD"}],"filterop": "and"}``

**Notes:**

  * Please note that the ``get_balances`` API call will not return balances for BTC itself. It only returns balances
    for XCP and other Counterparty assets. To get BTC-based balances, use an existing system such as Bitcoin Core, blockchain.info, etc.


###get_asset_info

**get_asset_info(asset, assets)**

Gets information on an issued asset.

**Parameters:**

  * **asset** (*string*): The name of the [asset](#assets) or [subasset](#subassets) for which to retrieve the information.
  * **assets** (*array*): An array of names of the [assets](#assets) or [subassets](#subassets) for which to retrieve the information.

**Return:**

  ``null`` if the asset was not found. Otherwise, a list of one or more objects, each one with the following properties:

  - **asset** (*string*): The [assets](#assets) of the asset itself
  - **asset_longname** (*string*): The [subasset](#subassets) longname, if any
  - **owner** (*string*): The address that currently owns the asset (i.e. has issuance rights to it)
  - **divisible** (*boolean*): Whether the asset is divisible or not
  - **locked** (*boolean*): Whether the asset is locked (future issuances prohibited)
  - **total_issued** (*integer*): The [quantities](#quantities-and-balances) of the asset issued, in total
  - **description** (*string*): The asset's current description
  - **issuer** (*string*): The asset's original owner (i.e. issuer)


###get_dispenser_info

**get_dispenser_info()**

Gets information on a dispenser.

**Parameters:**

  * **tx_hash** (*string*): The transaction hash identifier
  * **tx_index** (*integer*): The transaction index

**Return:**

  ``null`` if the asset was not found. Otherwise, an object with the following properties:

  - **asset** (*string*): The [assets](#assets) of the asset itself
  - **asset_longname** (*string*): The [subasset](#subassets) longname, if any
  - **tx_index** (*integer*): The transaction index
  - **tx_hash** (*string*): The transaction hash
  - **block_index** (*integer*): The block index (block number in the block chain)
  - **source** (*string*): The address that made the bet
  - **give_quantity** (*integer*): The [quantity](#quantities-and-balances) given per dispense
  - **escrow_quantity** (*integer*): The [quantity](#quantities-and-balances) escrowed in the dispenser
  - **give_remaining** (*integer*): The [quantity](#quantities-and-balances) remaining in the dispenser
  - **status** (*integer*): The state of the dispenser. 0 for open, 10 for closed.
  - **mainchainrate** (*integer*): The [quantity](#quantities-and-balances) of the main chain asset (BTC) per dispensed portion.
  - **fiat_price** (*float*): The FIAT price per dispense
  - **fiat_unit** (*string*): The FIAT unit being used
  - **satoshi_price** (*integer*): The sats required for a dispense
  - **oracle_price** (*float*): The BTC price broadcast by the oracle
  - **oracle_address** (*string*): The address that is being used as the oracle
  - **oracle_price_last_updated** (*integer*): The block_index of the last update from the oracle
  * *NOTE: If an **oracle_address** is given, **mainchainrate** format is X.XX (fiat) (ex. 1500 = 15.00).*


###get_supply

**get_supply(asset)**

**Parameters:**

  * **asset** (*string*): The name of the [asset](#assets) or [subasset](#subassets) for which to retrieve the information.

**Return:**

  ``null`` if the asset was not found. Otherwise, a list of one or more objects, each one with the following properties:


###get_asset_names

**get_asset_names()**

**Parameters:**

  None

**Return:**

  A list of the names of all existing Counterparty assets, ordered alphabetically.


###get_holder_count

**get_holder_count()**

**Parameters:**

  * **asset** (*string*): The name of the [asset](#assets) or [subasset](#subassets) for which to retrieve the information.

**Return:**

  An object the asset name as the property name, and the holder count as the value of that property name.


###get_holders

**get_holders()**

**Parameters:**

  * **asset** (*string*): The name of the [asset](#assets) or [subasset](#subassets) for which to retrieve the information.

**Return:**

  A list of addresses that hold some quantity of the specified asset.


###get_messages

**get_messages(block_index)**

Return message feed activity for the specified block index. The message feed essentially tracks all
database actions and allows for lower-level state tracking for applications that hook into it.

**Parameters:**

  * **block_index** (*integer*): The block index for which to retrieve activity.

**Return:**

  A list of one or more [message object](#message-object) if there was any activity in the block, otherwise ``[]`` (empty list).


###get_messages_by_index

**get_messages_by_index(message_indexes)**

Return the message feed messages whose ``message_index`` values are contained in the specified list of message indexes.

**Parameters:**

  * **message_indexes** (*list*): An array of one or more ``message_index`` values for which the cooresponding message feed entries are desired.

**Return:**

  A list containing a `message <#message-object>`_ for each message found in the specified ``message_indexes`` list. If none were found, ``[]`` (empty list) is returned.


###get_block_info

**get_block_info(block_index)**

Gets basic information for a specific block.

**Parameters:**

  * **block_index** (*integer*): The block index for which to retrieve information.

**Return:**

  If the block was found, an object with the following properties:

  - **block_index** (*integer*): The block index (i.e. block height). Should match what was specified for the *block_index* input parameter).
  - **block_hash** (*string*): The block hash identifier
  - **block_time** (*integer*): A UNIX timestamp of when the block was processed by the network


###get_blocks

**get_blocks(block_indexes, min_message_index=null)**

Gets block and message data (for each block) in a bulk fashon. If fetching info and messages for multiple blocks, this
is much quicker than using multiple ``get_block_info()`` and ``get_messages()`` calls.

**Parameters:**

  * **block_indexes** (*list*): A list of 1 or more block indexes for which to retrieve the data.
  * **min_message_index** (*string*): Retrieve blocks from the message feed on or after this specific message index (useful since blocks may appear in the message feed more than once, if a reorg occurred). Note that if this parameter is not specified, the messages for the first block will be returned. If unsure, leave this blank.

**Return:**

  A list of objects, one object for each valid block index specified, in order from first block index to last.
  Each object has the following properties:

  - **block_index** (*integer*): The block index (i.e. block height). Should match what was specified for the *block_index* input parameter).
  - **block_hash** (*string*): The block hash identifier
  - **block_time** (*integer*): A UNIX timestamp of when the block was processed by the network
  - **_messages** (*list*): A list of one or more [message object](#message-object) if there was any activity in the block, otherwise ``[]`` (empty list).


###get_running_info

**get_running_info()**

Gets some operational parameters for the server.

**Parameters:**

  None

**Return:**

  An object with the following properties:

  - **db_caught_up** (*boolean*): ``true`` if block processing is caught up with the Bitcoin blockchain, ``false`` otherwise.
  - **bitcoin_block_count** (**integer**): The block height on the Bitcoin network (may not necessarily be the same as ``last_block``, if the server is catching up)
  - **last_block** (*integer*): The index (height) of the last block processed by the server
  - **last_message_index** (*integer*): The index (ID) of the last message in the message feed
  - **running_testnet** (*boolean*): ``true`` if the server is configured for testnet, ``false`` if configured on mainnet.
  - **running_testcoin** (*boolean*): ``true`` if the server is configured for testcoin use, ``false`` if not (default).
  - **version_major** (*integer*): The major version of counterparty-server running
  - **version_minor** (*integer*): The minor version of counterparty-server running
  - **version_revision** (*integer*): The revision version of counterparty-server running
  - **api_limit_rows** (*integer*): The max amount of rows any call will return. If ``0`` there's no limit to calls. Defaults to ``1000``.


###get_element_counts

**get_element_counts()**

Gets the number of records for each entity type

**Parameters:**

  None

**Return:**

  An object with a property for each element type (e.g. `transactions`, `blocks`, `bets`, `order_matches`, etc.) with the value of each property being the record count of that respective entity in the database.


###get_unspent_txouts

**get_unspent_txouts(address, unconfirmed=false, unspent_tx_hash=null)**

Get a listing of UTXOs for the specified address.

**Parameters:**

  * **address** (*string*): The address for which to receive the UTXO listing
  * **unconfirmed** (*boolean*): Set to `true` to include unconfirmed UTXOs (e.g. those in the mempool)
  * **unspent_tx_hash** (*boolean*): Specify a specific transaction hash to only include UTXOs from that transaction
  * **order_by** (*string*): Sort results by specified field (e.g. height, -height)

**Return:**

  A list of objects, with each entry in the dict having the following properties:

    - **amount**: The amount of the UTXO (e.g. 0.12345678)
    - **value**: The value of the UTXO in satoshis (e.g. 12345678)
    - **height**: The block height of the UTXO
    - **confirmations**: Number of confirmations since the UTXO was created
    - **txid**: The txid (hash) that the UTXO was included in
    - **vout**: The vout number in the specified txid for the UTXO

###getrawtransaction

**getrawtransaction(tx_hash, verbose=false, skip_missing=false)**

Gets raw data for a single transaction.

**Parameters:**

  * **tx_hash** (*string*): The transaction hash identifier
  * **verbose** (*boolean*): Include some additional information in the result data
  * **skip_missing** (*boolean*): If set to `false`, and the transaction hash cannot be found, return `null`, otherwise if `true`, throw an exception.

**Return:**

  If found, a raw transaction objects having the same format as the [bitcoind getrawtransaction API call](https://chainquery.com/bitcoin-api/getrawtransaction). If not found, `null`.


###getrawtransaction_batch

**getrawtransaction_batch(txhash_list, verbose=false, skip_missing=false)**

Gets raw data for a list of transactions.

**Parameters:**

  * **txhash_list** (*string*): A list of transaction hash identifiers
  * **verbose** (*boolean*): Include some additional information in the result data for each transaction
  * **skip_missing** (*boolean*): If set to `false`, and one or more transaction hash cannot be found, the missing txhash data will not be included in the result set, otherwise if `true`, throw an exception.

**Return:**

  A list of raw transaction objects having the same format as the [bitcoind getrawtransaction API call](https://chainquery.com/bitcoin-api/getrawtransaction).


###search_raw_transactions

**search_raw_transactions(address, unconfirmed=true)**

Gets raw transaction objects for the specified address.

**Parameters:**

  * **address** (*string*): The address for which to receive the raw transactions
  * **unconfirmed** (*boolean*): Set to `true` to include unconfirmed transactions (e.g. those in the mempool)

**Return:**

  A list of raw transaction objects, with each object having the same format as the [bitcoind getrawtransaction API call](https://chainquery.com/bitcoin-api/getrawtransaction).


###get_tx_info

**get_tx_info(tx_hex, block_index=null)**

Get transaction info, as parsed by `counterparty-server`.

**Parameters:**

  * **tx_hex** (*string*): The canonical hexadecimal serialization of the transaction (not its hash)
  * **block_index** (*integer*)

**Return:**

  A list with the following items (in order as listed below):

    - `source`
    - `destination`
    - `btc_amount`
    - `fee`
    - `data`: The embedded raw protocol data, in hexadecimal-serialized format


###search_pubkey

**search_pubkey(pubkeyhash, provided_pubkeys=null)**

For the specified pubkeyhash (i.e. address), return the public key. Note that this requires that the specified address has made at least one outgoing transaction.

**Parameters:**

  * **pubkeyhash** (*string*): The pubkeyhash/address
  * **provided_pubkeys** (*list*): A list of supplied pubkeys. If one of these pubkeys matches the pubkeyhash, used if one of the supplied pubkey hashes to the pubkeyhash. (Can be useful if the pubkeyhash has not sent out at least one transaction and you have a list of pubkeys that may match it.)

**Return:**

  A string with the specified pubkey. If the pubkey cannot be found, an exception will be generated and returned.


###unpack

**unpack(data_hex)**

Parse the data_hex of a message into its parameters. Currently only works with `send` messages.

**Parameters:**

  * **data_hex** (*string*): The canonical hexadecimal serialization of the transaction (not its hash), e.g. from the `data_hex` return value from `get_tx_info`

**Return:**

  - **message_type_id** (*int*): the ID of the message type.  Legacy sends are `0` and enhanced sends are `2`.
  - **unpacked** (*object*): A map of message parameters. For legacy sends this object includes `asset` and `quantity`.  For enhanced sends, this object includes `address`, `asset`, `quantity` and `memo`.  For legacy sends, the source and destination are found using `get_tx_info`.  For enhanced sends, the destination address is in the message parameters and the source may be found using `get_tx_info`.


##Action/Write API Function Reference

###create_bet

**create_bet(source, feed_address, bet_type, deadline, wager_quantity, counterwager_quantity, expiration, target_value=0.0, leverage=5040)**

Issue a bet against a feed.

**Parameters:**

  * **source** (*string*): The address that will make the bet.
  * **feed_address** (*string*): The address that hosts the feed to be bet on.
  * **bet_type** (*integer*): 0 for Bullish CFD (deprecated), 1 for Bearish CFD (deprecated), 2 for Equal, 3 for NotEqual.
  * **deadline** (*integer*): The time at which the bet should be decided/settled, in Unix time (seconds since epoch).
  * **wager_quantity** (*integer*): The [quantities](#quantities-and-balances) of XCP to wager (*in satoshis*, hence integer).
  * **counterwager_quantity** (*integer*): The minimum [quantities](#quantities-and-balances) of XCP to be wagered against, for the bets to match.
  * **expiration** (*integer*): The number of blocks after which the bet expires if it remains unmatched.
  * **target_value** (*float, default=null*): Target value for Equal/NotEqual bet
  * **leverage** (*integer, default=5040*): Leverage, as a fraction of 5040
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*

**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.


###create_broadcast

**create_broadcast(source, fee_fraction, text, timestamp, value)**

Broadcast textual and numerical information to the network.

**Parameters:**

  * **source** (*string*): The address that will be sending (must have the necessary quantity of the specified asset).
  * **fee_fraction** (*float*): How much of every bet on this feed should go to its operator; a fraction of 1, (i.e. 0.05 is five percent).
  * **text** (*string*): The textual part of the broadcast.
  * **timestamp** (*integer*): The timestamp of the broadcast, in Unix time.
  * **value** (*float*): Numerical value of the broadcast.
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*

**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.


###create_btcpay

**create_btcpay(order_match_id)**

Create and (optionally) broadcast a BTCpay message, to settle an Order Match for which you owe BTC.

**Parameters:**
  * **source** (*string*): The source address of the btcpay transaction.
  * **order_match_id** (*string*): The concatenation of the hashes of the two transactions which compose the order match.
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*

**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.


###create_burn

**create_burn(source, quantity)**

Burn a given quantity of BTC for XCP (**on mainnet, possible between blocks 278310 and 283810**; on testnet it is still available).

**Parameters:**

  * **source** (*string*): The address with the BTC to burn.
  * **quantity** (*integer*): The [quantities](#quantities-and-balances) of BTC to burn (1 BTC maximum burn per address).
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*

**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.


###create_cancel

**create_cancel(offer_hash, source)**

Cancel an open order or bet you created.

**Parameters:**

  * **offer_hash** (*string*): The transaction hash of the order or bet.
  * **source** (*string*): The source address of the order or bet.
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*

**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.

###create_destroy

**create_destroy(source, asset, quantity, tag)**

Destroy XCP or a user defined asset.

**Parameters:**

  * **source** (*string*): The address that will be sending (must have the necessary quantity of the specified asset).
  * **asset** (*string*): The [asset](#assets) or [subasset](#subassets) to destroy.
  * **quantity** (*integer*): The [quantities](#quantities-and-balances) of the asset to destroy.
  * **tag** (*string, optional*): The tag (which works like a [Memo](../protocol_specification#memos) ) associated with this transaction.
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*

**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.

###create_dispenser

**create_dispenser(source, asset, give_quantity, escrow_quantity, mainchainrate, status, open_address, oracle_address)**

Opens or closes a dispenser for a given asset at a given rate of main chain asset (BTC). Escrowed
quantity on open must be equal or greater than *give_quantity*. It is suggested that you escrow multiples
of give_quantity to ease dispenser operation.

**Parameters:**

  * **source** (*string*): The address that will be dispensing (must have the necessary escrow_quantity of the specified asset).
  * **asset** (*string*): The [asset](#assets) or [subasset](#subassets) to dispense.
  * **give_quantity** (*integer*): The [quantity](#quantities-and-balances) of the asset to dispense.
  * **escrow_quantity** (*integer*): The [quantity](#quantities-and-balances) of the asset to reserve for this dispenser.
  * **mainchainrate** (*integer*): The [quantity](#quantities-and-balances) of the main chain asset (BTC) per dispensed portion.
  * **open_address** (*string*): The address that you would like to open the dispenser on.
  * **oracle_address** (*string*): The address that you would like to use as a price oracle for this dispenser.
  * **status** (*integer*): The state of the dispenser. 0 for open, 1 for open using open_address, 10 for closed.
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*

**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.

**Notes:**
  * When specifying an **oracle_address**, **mainchainrate** format becomes X.XX (fiat) (ex. 1500 = 15.00).
  * Dispensers can be opened on **`EMPTY`** addresses (address with no BTC or XCP history)
  * **`origin`** address is the **`source`** address that opens a dispenser on an **`EMPTY`** address
  * Dispensers can be opened, closed, and refilled via **`source`** or **`origin`** addresses
  * When closing a dispenser from **`origin`**, escrowed dispenser funds are credited to **`origin`** address
  * Use **`open_address`** param to operate on a dispenser from **`origin`** address  
  * Dispensers have a `MAX_DISPENSE` limit of 1,000
  * Dispensers `MAX_DISPENSE` can be reset via a dispenser refill
  * Dispensers have a `MAX_REFILL` limit of 5
  * Dispensers have a `CLOSE` delay of 5 blocks

###create_dividend

**create_dividend(source, quantity_per_unit, asset, dividend_asset)**

Issue a dividend on a specific user defined asset.

**Parameters:**

  * **source** (*string*): The address that will be issuing the dividend (must have the ownership of the asset which the dividend is being issued on).
  * **quantity_per_unit** (*integer*): The amount of **dividend_asset** rewarded.
  * **asset** (*string*): The [asset](#assets) or [subasset](#subassets) that the dividends are being rewarded on.
  * **dividend_asset** (*string*): The [asset](#assets) or [subasset](#subassets) that the dividends are paid in.
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*

**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.


###create_issuance

**create_issuance(source, asset, quantity, divisible, description, transfer_destination=null, lock, reset)**

Issue a new asset, issue more of an existing asset, lock an asset, reset existing supply, or transfer the ownership of an asset.

**Parameters:**

  * **source** (*string*): The address that will be issuing or transfering the asset.
  * **asset** (*string*): The [assets](#assets) to issue or transfer.  This can also be a [subasset longname](#subassets) for new subasset issuances.
  * **quantity** (*integer*): The [quantity](#quantities-and-balances) of the asset to issue (set to 0 if *transferring* an asset).
  * **divisible** (*boolean, default=true*): Whether this asset is divisible or not (if a transfer, this value must match the value specified when the asset was originally issued).
  * **description** (*string, default=''*): A textual description for the asset.
  * **transfer_destination** (*string, default=null*): The address to receive the asset.
  * **lock** (*boolean, default=false*): Whether this issuance should lock supply of this asset forever.
  * **reset** (*boolean, default=false*): Wether this issuance should reset any existing supply.
  * *NOTE: **reset** is only possible when no supply is issued, or when the asset owner has control of 100% of the supply.*
  * *NOTE: When resetting an assets supply, **transfer_destination** will not work in the same issuance as **reset** .*
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*

**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.

**Notes:**

  * A named asset has an issuance cost of 0.5 XCP.
  * A subasset has an issuance cost of 0.25 XCP.
  * In order to issue an asset, BTC and XCP (for first time, non-free Counterparty assets) are required at the source address to pay fees.



###create_order

**create_order(source, give_asset, give_quantity, get_asset, get_quantity, expiration)**

Issue an order request.

**Parameters:**

  * **source** (*string*): The address that will be issuing the order request (must have the necessary quantity of the specified asset to give).
  * **give_asset** (*string*): The [assets](#assets) to give.
  * **give_quantity** (*integer*): The [quantities](#quantities-and-balances) of the asset to give.
  * **get_asset** (*string*): The [assets](#assets) requested in return.
  * **get_quantity** (*integer*): The [quantities](#quantities-and-balances) of the asset requested in return.
  * **expiration** (*integer*): The number of blocks for which the order should be valid.
  * **fee_required** (*integer*): The miners’ fee required to be paid by orders for them to match this one; in BTC; required only if buying BTC (may be zero, though)
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*

**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.

###create_send


**create_send(source, destination, asset, quantity)**

Send XCP or a user defined asset.

To send multiple assets/destinations simultaneously you can pass an array of parameters to the destination, asset and quantity parameters. If one of these parameters is an array then the other must be an array of equal length. Each entry corresponds to the same entry index on the other arrays.

**Parameters:**

  * **source** (*string*): The address that will be sending (must have the necessary quantity of the specified asset).
  * **destination** (*string, array[string]*): The address to receive the asset.
  * **asset** (*string, array[string]*): The [asset](#assets) or [subasset](#subassets) to send.
  * **quantity** (*integer, array[integer]*): The [quantities](#quantities-and-balances) of the asset to send.
  * **memo** (*string, optional*): The [Memo](../protocol_specification#memos) associated with this transaction.
  * **memo_is_hex** (*boolean, optional*): If this is true, interpret the [memo](../protocol_specification#memos) as a hexadecimal value.  Defaults to false.
  * **use_enhanced_send** (*boolean, optional*): If this is false, the construct a legacy transaction sending bitcoin dust.  Defaults to true.
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*


**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.


###create_sweep

**create_sweep(source, destination, flags, memo)**

Sends all assets and/or transfer ownerships to a destination address.

**Parameters:**

  * **source** (*string*): The address that will be sending.
  * **destination** (*string*): The address to receive the assets and/or ownerships.
  * **flags** (*integer*): An OR mask of flags indicating how the sweep should be processed. Possible flags are:
    * FLAG_BALANCES: (*integer*) 1, specifies that all balances should be transferred.
    * FLAG_OWNERSHIP: (*integer*) 2, specifies that all ownerships should be transferred.
    * FLAG_BINARY_MEMO: (*integer*) 4, specifies that the memo is in binary/hex form.
  * **memo** (*string, optional*): The [Memo](../protocol_specification#memos) associated with this transaction.
  * *NOTE: Additional (advanced) parameters for this call are documented [here](#advanced-create_-parameters).*

**Return:**

  The unsigned transaction, as an hex-encoded string. Must be signed before being broadcast: see [here](#signing-transactions-before-broadcasting) for more information.


###Advanced `create_` parameters

Each `create_` call detailed below can take the following common keyword parameters:

  * **encoding** (*string*): The encoding method to use, see [transaction encodings](#transaction-encodings) for more info.
  * **pubkey** (*string/list*): The hexadecimal public key of the source address (or a list of the keys, if multi‐sig). Required when using ``encoding`` parameter values of ``multisig`` or ``pubkeyhash``. See [transactions encoding](#transaction-encodings) for more info
  * **allow_unconfirmed_inputs** (*boolean*): Set to ``true`` to allow this transaction to utilize unconfirmed UTXOs as inputs. Defaults to `false`.
  * **fee** (*integer*): If you'd like to specify a custom miners' fee, specify it here (in satoshi). Leave as default for the server to automatically choose.
  * **fee_per_kb** (*integer*): The fee per kilobyte of transaction data constant that the server uses when deciding on the dynamic fee to use (in satoshi).
  * **fee_provided** (*integer*): If you would like to specify a maximum fee (up to and including which may be used as the transaction fee), specify it here (in satoshi). This differs from `fee` in that this is an upper bound value, which `fee` is an exact value.
  * **custom_inputs** (*list*): Use only these specific UTXOs as inputs for the transaction being created. If specified, this parameter is a list of (JSON-encoded) UTXO objects, whose properties match those as retrieved by `listunspent` function from bitcoind (e.g. see [here](https://chainquery.com/bitcoin-api/listunspent)). Note that the actual UTXOs used may be a subset of this list.
  * **unspent_tx_hash** (*string*): When compiling the UTXOs to use as inputs for the transaction being created, only consider unspent outputs from this specific transaction hash. Defaults to `null` to consider all UTXOs for the address. Do not use this parameter if you are specifying `custom_inputs`.
  * **regular_dust_size** (*integer*): Specify (in satoshi) to override the (dust) amount of BTC used for each non-(bare) multisig output. Defaults to `5430` satoshi.
  * **multisig_dust_size** (*integer*): Specify (in satoshi) to override the (dust) amount of BTC used for each (bare) multisig output. Defaults to `7800` satoshi.
  * **dust_return_pubkey** (*string*): The dust return pubkey is used in multi-sig data outputs (as the only real pubkey) to make those the outputs spendable. By default, this pubkey is taken from the pubkey used in the first transaction input. However, it can be overridden here (and is _required_ to be specified if a P2SH input is used and multisig is used as the data output encoding.) If specified, specify the public key (in hex format) where dust will be returned to so that it can be reclaimed. Only valid/useful when used with transactions that utilize multisig data encoding. Note that if this value is set to `false`, this instructs `counterparty-server` to use the default dust return pubkey configured at the node level. If this default is not set at the node level, the call will generate an exception.
  * **disable_utxo_locks** (*boolean*): By default, UTXO's utilized when creating a transaction are "locked" for a few seconds, to prevent a case where rapidly generating `create_` calls reuse UTXOs due to their spent status not being updated in bitcoind yet. Specify `true` for this parameter to disable this behavior, and not temporarily lock UTXOs.
  * **op_return_value** (*integer*): The value (in satoshis) to use with any `OP_RETURN` outputs in the generated transaction. Defaults to `0`. Don't use this, unless you like [throwing your money away](https://m.reddit.com/r/Bitcoin/comments/2plfsv/what_happens_to_the_value_of_a_coin_locked_with/cmxrnhu).
  * **extended_tx_info** (*boolean*): When this is not specified or false, the `create_` calls return only a hex-encoded string.  If this is true, the `create_` calls return a data object with the following keys: `tx_hex`, `btc_in`, `btc_out`, `btc_change`, and `btc_fee`.
  * **p2sh_pretx_txid** (*string*): The previous transaction `txid` for a two part ``P2SH`` message. This `txid` must be taken from the signed transaction.

**With the exception of `pubkey` and `allow_unconfirmed_inputs`, these values should be left at their defaults, unless you know what you are doing.**

####Transaction Encodings

By default the default value of the ``encoding`` parameter detailed above is ``auto``, which means that `counterparty-server` automatically determines the best way to encode the Counterparty protocol data into a new transaction. If you know what you are doing and would like to explicitly specify an encoding:

- To return the transaction as an **OP_RETURN** transaction, specify ``opreturn`` for the ``encoding`` parameter.
   - **OP_RETURN** transactions cannot have more than 80 bytes of data. If you force OP_RETURN encoding and your transaction would have more than this amount, an exception will be generated.
- To return the transaction as a **multisig** transaction, specify ``multisig`` for the ``encoding`` parameter.
    - ``pubkey`` should be set to the hex-encoded public key of the source address.
    - Note that with the newest versions of Bitcoin (0.12.1 onward), bare multisig encoding does not reliably propagate. More information on this is documented [here](https://github.com/rubensayshi/counterparty-lib/pull/9).
- To return the transaction as a **pubkeyhash** transaction, specify ``pubkeyhash`` for the ``encoding`` parameter.
    - ``pubkey`` should be set to the hex-encoded public key of the source address.
- To return the transaction as a 2 part **P2SH** transaction, specify ``P2SH`` for the encoding parameter.
    - First call the ``create_`` method with the ``encoding`` set to ``P2SH``.
    - Sign the transaction as usual and broadcast it. It's recommended but not required to wait the transaction to confirm as malleability is an issue here (P2SH isn't yet supported on segwit addresses).
    - The resulting ``txid`` must be passed again on an identic call to the ``create_`` method, but now passing an additional parameter ``p2sh_pretx_txid`` with the value of the previous transaction's id.
    - The resulting transaction is a ``P2SH`` encoded message, using the redeem script on the transaction inputs as data carrying mechanism.
    - Sign the transaction following the ``Bitcoinjs-lib on javascript, signing a P2SH redeeming transaction`` section
    - **NOTE**: Don't leave pretxs hanging without transmitting the second transaction as this pollutes the UTXO set and risks making bitcoin harder to run on low spec nodes.



##REST API Function Reference

The REST API documentation is hosted both on our webiste and on a new API documentation platform called apiary.io. This experimental documentation, complementary to the one in this document, is located [here](http://docs.counterpartylib.apiary.io/#).

###get

**get(table_name, filters, filterop)**

Query table_name in the database using filters concatenated using filterop.

URL format:

`/rest/<table_name>/get?<table_filters>&op=<filter_op>`

Example query:

`/rest/sends/get?source=mn6q3dS2EnDUx3bmyWc6D4szJNVGtaR7zc&destination=mtQheFaSfWELRB2MyMBaiWjdDm6ux9Ezns&op=AND`

**Parameters:**
  * **table_name** (*string*): The name of the desired table. List of all available tables:
              `assets`, `balances`, `credits`, `debits`, `bets`, `bet_matches`,
              `broadcasts`, `btcpays`, `burns`, `cancels`, `dividends`, `issuances`,
              `orders`, `order_matches`, `sends`, `bet_expirations`, `order_expirations`,
              `bet_match_expirations`, `order_match_expirations`, `bet_match_resolutions`, `mempool`
  * **filters** (*dict, optional*): Data filters as a dictionary. The filter format is same as for get_{} JSON API queries. See [Filtering Read API Results](#filtering-read-api-results) for more information on filters and [Object Definitions](#objects) for fields available for specific objects.
  * **filterop** (*string, optional*): The logical operator concatenating the filters. Defaults to `AND`.

**Headers:**
  * **Accept** (*string, optional*): The format of return data. Can be either `application/json` or `application/xml`. Defaults to JSON.

**Return:**

  Desired database rows from table_name sieved using filters.


###compose

**compose(message_type, transaction_params)**

Compose a `message_type` transaction with `transaction_params` as data.

URL format:

`/rest/<tx_type>/compose?<tx_data>`

Example query:

`/rest/send/compose?source=mn6q3dS2EnDUx3bmyWc6D4szJNVGtaR7zc&destination=mtQheFaSfWELRB2MyMBaiWjdDm6ux9Ezns&asset=BTC&quantity=1`

**Parameters:**
  * **message_type** (*string*): The type of desired transaction message. List of all available transactions:
                `bet`, `broadcast`, `btcpay`, `burn`, `cancel`, `dividend`, `issuance`,
                `order`, `send`, `publish`, `execute`
  * **transaction_params** (*dict*): The parameters to be passed to the compose_transaction function. See [Write API Function Reference](#actionwrite-api-function-reference) for list of transactions and their parameters.

**Headers:**
  * **Accept** (*string, optional*): The format of return data. Can be either `application/json` or `application/xml`. Defaults to JSON.

**Return:**

  The hex data of composed transaction.


##Objects

The API calls documented can return any one of these objects.


###Balance Object

An object that describes a balance that is associated to a specific address:

* **address** (*string*): A PubkeyHash Bitcoin address, or the pubkey associated with it (in case the address hasn’t sent anything before).
* **asset** (*string*): The ID of the [assets](#assets) in which the balance is specified
* **quantity** (*integer*): The [quantities](#quantities-and-balances) of the specified asset at this address


###Bet Object

An object that describes a specific bet:

* **tx_index** (*integer*): The transaction index
* **tx_hash** (*string*): The transaction hash
* **block_index** (*integer*): The block index (block number in the block chain)
* **source** (*string*): The address that made the bet
* **feed_address** (*string*): The address with the feed that the bet is to be made on
* **bet_type** (*integer*): 0 for Bullish CFD (deprecated), 1 for Bearish CFD (deprecated), 2 for Equal, 3 for Not Equal
* **deadline** (*integer*): The timestamp at which the bet should be decided/settled, in Unix time.
* **wager_quantity** (*integer*): The [quantities](#quantities-and-balances) of XCP to wager
* **counterwager_quantity** (*integer*): The minimum [quantities](#quantities-and-balances) of XCP to be wagered by the user to bet against the bet issuer, if the other party were to accept the whole thing
* **wager_remaining** (*integer*): The quantity of XCP wagered that is remaining to bet on
* **odds** (*float*):
* **target_value** (*float*): Target value for Equal/NotEqual bet
* **leverage** (*integer*): Leverage, as a fraction of 5040
* **expiration** (*integer*): The number of blocks for which the bet should be valid
* **fee_multiplier** (*integer*): How much of every bet on this feed should go to its operator; a fraction of 1, (i.e. 0.05 is five percent)
* **validity** (*string*): Set to "valid" if a valid bet. Any other setting signifies an invalid/improper bet


###Bet Match Object

An object that describes a specific occurance of two bets being matched (either partially, or fully):

* **tx0_index** (*integer*): The Bitcoin transaction index of the initial bet
* **tx0_hash** (*string*): The Bitcoin transaction hash of the initial bet
* **tx0_block_index** (*integer*): The Bitcoin block index of the initial bet
* **tx0_expiration** (*integer*): The number of blocks over which the initial bet was valid
* **tx0_address** (*string*): The address that issued the initial bet
* **tx0_bet_type** (*string*): The type of the initial bet (0 for Bullish CFD (deprecated), 1 for Bearish CFD (deprecated), 2 for Equal, 3 for Not Equal)
* **tx1_index** (*integer*): The transaction index of the matching (counter) bet
* **tx1_hash** (*string*): The transaction hash of the matching bet
* **tx1_block_index** (*integer*): The block index of the matching bet
* **tx1_address** (*string*): The address that issued the matching bet
* **tx1_expiration** (*integer*): The number of blocks over which the matching bet was valid
* **tx1_bet_type** (*string*): The type of the counter bet (0 for Bullish CFD (deprecated), 1 for Bearish CFD (deprecated), 2 for Equal, 3 for Not Equal)
* **feed_address** (*string*): The address of the feed that the bets refer to
* **initial_value** (*integer*):
* **deadline** (*integer*): The timestamp at which the bet match was made, in Unix time.
* **target_value** (*float*): Target value for Equal/NotEqual bet  
* **leverage** (*integer*): Leverage, as a fraction of 5040
* **forward_quantity** (*integer*): The [quantities](#quantities-and-balances) of XCP bet in the initial bet
* **backward_quantity** (*integer*): The [quantities](#quantities-and-balances) of XCP bet in the matching bet
* **fee_multiplier** (*integer*):
* **validity** (*string*): Set to "valid" if a valid order match. Any other setting signifies an invalid/improper order match


###Broadcast Object

An object that describes a specific occurance of a broadcast event (i.e. creating/extending a feed):

* **tx_index** (*integer*): The transaction index
* **tx_hash** (*string*): The transaction hash
* **block_index** (*integer*): The block index (block number in the block chain)
* **source** (*string*): The address that made the broadcast
* **timestamp** (*string*): The time the broadcast was made, in Unix time.
* **value** (*float*): The numerical value of the broadcast
* **fee_multiplier** (*float*): How much of every bet on this feed should go to its operator; a fraction of 1, (i.e. 0.05 is five percent)
* **text** (*string*): The textual component of the broadcast
* **validity** (*string*): Set to "valid" if a valid broadcast. Any other setting signifies an invalid/improper broadcast


###BTCPay Object

An object that matches a request to settle an Order Match for which BTC is owed:

* **tx_index** (*integer*): The transaction index
* **tx_hash** (*string*): The transaction hash
* **block_index** (*integer*): The block index (block number in the block chain)
* **source** (*string*):
* **order_match_id** (*string*):
* **validity** (*string*): Set to "valid" if valid


###Burn Object

An object that describes an instance of a specific burn:

* **tx_index** (*integer*): The transaction index
* **tx_hash** (*string*): The transaction hash
* **block_index** (*integer*): The block index (block number in the block chain)
* **source** (*string*): The address the burn was performed from
* **burned** (*integer*): The [quantities](#quantities-and-balances) of BTC burned
* **earned** (*integer*): The [quantities](#quantities-and-balances) of XPC actually earned from the burn (takes into account any bonus quantitys, 1 BTC limitation, etc)
* **validity** (*string*): Set to "valid" if a valid burn. Any other setting signifies an invalid/improper burn


###Cancel Object

An object that describes a cancellation of a (previously) open order or bet:

* **tx_index** (*integer*): The transaction index
* **tx_hash** (*string*): The transaction hash
* **block_index** (*integer*): The block index (block number in the block chain)
* **source** (*string*): The address with the open order or bet that was cancelled
* **offer_hash** (*string*): The transaction hash of the order or bet cancelled
* **validity** (*string*): Set to "valid" if a valid burn. Any other setting signifies an invalid/improper burn


###Debit/Credit Object

An object that describes a account debit or credit:

* **tx_index** (*integer*): The transaction index
* **tx_hash** (*string*): The transaction hash
* **block_index** (*integer*): The block index (block number in the block chain)
* **address** (*string*): The address debited or credited
* **asset** (*string*): The [assets](#assets) debited or credited
* **quantity** (*integer*): The [quantities](#quantities-and-balances) of the specified asset debited or credited


###Dividend Object

An object that describes an issuance of dividends on a specific user defined asset:

* **tx_index** (*integer*): The transaction index
* **tx_hash** (*string*): The transaction hash
* **block_index** (*integer*): The block index (block number in the block chain)
* **source** (*string*): The address that issued the dividend
* **asset** (*string*): The [assets](#assets) that the dividends are being rewarded on
* **quantity_per_unit** (*integer*): The [quantities](#quantities-and-balances) of XCP rewarded per whole unit of the asset
* **validity** (*string*): Set to "valid" if a valid burn. Any other setting signifies an invalid/improper burn


###Issuance Object

An object that describes a specific occurance of a user defined asset being issued, or re-issued:

* **tx_index** (*integer*): The transaction index
* **tx_hash** (*string*): The transaction hash
* **block_index** (*integer*): The block index (block number in the block chain)
* **asset** (*string*): The [assets](#assets) being issued, or re-issued
* **asset_longname** (*string*): The [subasset](#subassets) longname, if any
* **quantity** (*integer*): The [quantities](#quantities-and-balances) of the specified asset being issued
* **divisible** (*boolean*): Whether or not the asset is divisible (must agree with previous issuances of the asset, if there are any)
* **issuer** (*string*):
* **transfer** (*boolean*): Whether or not this objects marks the transfer of ownership rights for the specified quantity of this asset
* **validity** (*string*): Set to "valid" if a valid issuance. Any other setting signifies an invalid/improper issuance


###Order Object

An object that describes a specific order:

* **tx_index** (*integer*): The transaction index
* **tx_hash** (*string*): The transaction hash
* **block_index** (*integer*): The block index (block number in the block chain)
* **source** (*string*): The address that made the order
* **give_asset** (*string*): The [assets](#assets) being offered
* **give_quantity** (*integer*): The [quantities](#quantities-and-balances) of the specified asset being offered
* **give_remaining** (*integer*): The [quantities](#quantities-and-balances) of the specified give asset remaining for the order
* **get_asset** (*string*): The [assets](#assets) desired in exchange
* **get_quantity** (*integer*): The [quantities](#quantities-and-balances) of the specified asset desired in exchange
* **get_remaining** (*integer*): The [quantities](#quantities-and-balances) of the specified get asset remaining for the order
* **price** (*float*): The given exchange rate (as an exchange ratio desired from the asset offered to the asset desired)
* **expiration** (*integer*): The number of blocks over which the order should be valid
* **fee_provided** (*integer*): The miners' fee provided; in BTC; required only if selling BTC (should not be lower than is required for acceptance in a block)
* **fee_required** (*integer*): The miners' fee required to be paid by orders for them to match this one; in BTC; required only if buying BTC (may be zero, though)


###Order Match Object

An object that describes a specific occurance of two orders being matched (either partially, or fully):

* **tx0_index** (*integer*): The Bitcoin transaction index of the first (earlier) order
* **tx0_hash** (*string*): The Bitcoin transaction hash of the first order
* **tx0_block_index** (*integer*): The Bitcoin block index of the first order
* **tx0_expiration** (*integer*): The number of blocks over which the first order was valid
* **tx0_address** (*string*): The address that issued the first (earlier) order
* **tx1_index** (*integer*): The transaction index of the second (matching) order
* **tx1_hash** (*string*): The transaction hash of the second order
* **tx1_block_index** (*integer*): The block index of the second order
* **tx1_address** (*string*): The address that issued the second order
* **tx1_expiration** (*integer*): The number of blocks over which the second order was valid
* **forward_asset** (*string*): The [assets](#assets) exchanged FROM the first order to the second order
* **forward_quantity** (*integer*): The [quantities](#quantities-and-balances) of the specified forward asset
* **backward_asset** (*string*): The [assets](#assets) exchanged FROM the second order to the first order
* **backward_quantity** (*integer*): The [quantities](#quantities-and-balances) of the specified backward asset
* **validity** (*string*): Set to "valid" if a valid order match. Any other setting signifies an invalid/improper order match


###Send Object

An object that describes a specific send (e.g. "simple send", of XCP, or a user defined asset):

* **tx_index** (*integer*): The transaction index
* **tx_hash** (*string*): The transaction hash
* **block_index** (*integer*): The block index (block number in the block chain)
* **source** (*string*): The source address of the send
* **destination** (*string*): The destination address of the send
* **asset** (*string*): The [assets](#assets) being sent
* **quantity** (*integer*): The [quantities](#quantities-and-balances) of the specified asset sent
* **validity** (*string*): Set to "valid" if a valid send. Any other setting signifies an invalid/improper send
* **memo** (*string*): The [memo](../protocol_specification#memos) associated with this transaction


###Message Object

An object that describes a specific event in the counterpartyd message feed (which can be used by 3rd party applications
to track state changes to the counterpartyd database on a block-by-block basis).

* **message_index** (*integer*): The message index (i.e. transaction index)
* **block_index** (*integer*): The block index (block number in the block chain) this event occurred on
* **category** (*string*): A string denoting the entity that the message relates to, e.g. "credits", "burns", "debits".
  The category matches the relevant table name in counterpartyd (see blocks.py for more info).
* **command** (*string*): The operation done to the table noted in **category**. This is either "insert", or "update".
* **bindings** (*string*): A JSON-encoded object containing the message data. The properties in this object match the
  columns in the table referred to by **category**.


###Bet Expiration Object

An object that describes the expiration of a bet created by the source address.

* **bet_index** (*integer*): The transaction index of the bet expiring
* **bet_hash** (*string*): The transaction hash of the bet expiriing
* **block_index** (*integer*): The block index (block number in the block chain) when this expiration occurred
* **source** (*string*): The source address that created the bet


###Order Expiration Object

An object that describes the expiration of an order created by the source address.

* **order_index** (*integer*): The transaction index of the order expiring
* **order_hash** (*string*): The transaction hash of the order expiriing
* **block_index** (*integer*): The block index (block number in the block chain) when this expiration occurred
* **source** (*string*): The source address that created the order


###Bet Match Expiration Object

An object that describes the expiration of a bet match.

* **bet_match_id** (*integer*): The transaction index of the bet match ID (e.g. the concatenation of the tx0 and tx1 hashes)
* **tx0_address** (*string*): The tx0 (first) address for the bet match
* **tx1_address** (*string*): The tx1 (second) address for the bet match
* **block_index** (*integer*): The block index (block number in the block chain) when this expiration occurred


###Order Match Expiration Object

An object that describes the expiration of an order match.

* **order_match_id** (*integer*): The transaction index of the order match ID (e.g. the concatenation of the tx0 and tx1 hashes)
* **tx0_address** (*string*): The tx0 (first) address for the order match
* **tx1_address** (*string*): The tx1 (second) address for the order match
* **block_index** (*integer*): The block index (block number in the block chain) when this expiration occurred


##Status

Here the list of all possible status for each table:

* **balances**: No status field
* **bet_expirations**: No status field
* **bet_match_expirations**: No status field
* **bet_matches**: pending, settled: liquidated for bear (deprecated), settled, settled: liquidated for bull (deprecated), settled: for equal, settled: for notequal, dropped, expired
* **bets**: open, filled, cancelled, expired, dropped, invalid: {problem(s)}
* **broadcasts**: valid, invalid: {problem(s)}
* **btcpays**: valid, invalid: {problem(s)}
* **burns**: valid, invalid: {problem(s)}
* **cancels**: valid, invalid: {problem(s)}
* **credits**: No status field
* **debits**: No status field
* **dividends**: valid, invalid: {problem(s)}
* **issuances**: valid, invalid: {problem(s)}
* **order_expirations**: No status field
* **order_match_expirations**: No status field
* **order_matches**: pending, completed, expired
* **orders**: open, filled, canceled, expired, invalid: {problem(s)}
* **sends**: valid, invalid: {problem(s)}



#API Changes

This section documents any changes to the API, for version numbers where there were API-level modifications.

There will be no incompatible API pushes that do not either have:

* A well known set cut over date in the future
* Or, a deprecation process where the old API is supported for an amount of time

##9.58.0
 * Added `P2SH` support and `MPMA` support:
   * Explanation of the new array parameters for `create_send`
   * Explanation of the new `p2sh_pretx_txid` parameter
   * Example to sign the new kind of `P2SH transaction`

##9.55.5
* create_*: adds `extended_tx_info` parameter to create methods

##9.55.4
* No changes

##9.55.3
* create_send: Added `memo`, `memo_is_hex` and `use_enhanced_send` parameters
* get_sends: Added support for `memo` and `memo_hex` filters
* get_sends: Returns `memo` and `memo_hex` in the search results

##9.55.2
* create_issuance: subassets longname are supported in the `asset` parameter

##9.53.0
* Add min_message_index to get_blocks API call

##9.52.0
* Added getrawtransaction and getrawtransaction_batch methods to the API
* Added optional custom_inputs parameter to API calls, which allows for controlling the exact UTXOs to use in transactions (contributed by Tokenly)

##9.51.0
* Deprecated `get_asset_info(assets)` API method. Use `get_issuances()` and `get_supply()` instead.
* Deprecated `get_xcp_supply()` API method in favor of `get_supply(asset)`.
* Changed `get_unspent_txouts` API method parameter and return values.
* Added HTTP Rest API.
* Authentication on JSON‐RPC API is off by default
* `rpc_password` configuration parameter is no longer mandatory

##9.49.4
* The `do_*`, `sign_tx` and `broadcast_tx` methods have been completely deprecated. See the section [Wallet Integration](#Wallet-Integration).
* Added REST API.

##9.49.3

* \*_issuance: ``callable``, ``call_date`` and ``call_price`` are no longer valid parameters
* \*_callback: removed
* Bitcoin addresses may everywhere be replaced by pubkeys.
* The API will no longer search the local wallet for pubkeys, so they must be passed to the API manually if being used for the first time. Otherwise, you may get a "<address> not published in blockchain" error.

##9.43.0

* create_issuance: ``callable`` is also accepted
* create_*: ``null`` is used as default value for missing parameters

##9.32.0

**Summary:** API framework overhaul for performance and simplicity

* "/api" with no trailing slash no longer supported as an API endpoint (use "/" or "/api/" instead)
* We now consistently reject positional arguments with all API methods. Make sure your API calls do not use positional
  arguments (e.g. use {"argument1": "value1", "argument2": "value2"} instead of ["value1", "value2"])

##9.25.0

* new do_* methods: like create_*, but also sign and broadcast the transaction. Same parameters as create_*, plus optional privkey parameter.

**backwards incompatible changes**

* create_*: accept only dict as parameters
* create_bet: ``bet_type`` must be a integer (instead string)
* create_bet: ``wager`` and ``counterwager`` args are replaced by ``wager_quantity`` and ``counterwager_quantity``
* create_issuance: parameter ``lock`` (boolean) removed (use LOCK in description)
* create_issuance: parameter ``transfer_destination`` replaced by ``destination``
* DatabaseError: now a DatabaseError is returned immediately if the database is behind the backend, instead of after fourteen seconds

##9.24.1

**Summary:** New API parsing engine added, as well as dynamic get method composition in ``api.py``:

* Added ``sql`` API method
* Filter params: Added ``LIKE``, ``NOT LIKE`` and ``IN``

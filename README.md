# GBM WEB2.0 API

Last update: 8/09/2024

## Quick Intro

The GBM Algorithm, aka Bid-To Earn Auctions, is an auction mechanism that allows everyone (Bidder, Sellers, Auction Winner) to win. More details can be found at https://www.gbm.auction    
         
	
## This Repo

This repo is a backend API that allows you to integrate and run GBM auctions in your frontend. It also comes with example Frontend and Middleware. 

## Architecture

With a GBM auction, the order of the bid is extremely important. Henceforth, any DB modification need to be done trough a transaction to prevent concurrent override. The core of the API use is hence : 
* Submit a transaction request to the API                   
* The API instantly store your transaction request, and you can consult it trough read to the database.                 
* The API run a single threaded, FIFO transaction request processor. These transactions and then consequence are then processed (or rejected)          
* Any webhook registered with the API receive the corresponding JSON objects following a transaction being processed.

The API should NOT be directly accessed by your frontend. It is very agnostic regarding user represntation/auth (a simple string/userID) and doesn't do any auth. No other software than the API should have write access to the associated mongoDB cluster. A read access is possible and even recommended. As such, a middleware need to exist in front of the API, doing the following : 
* User Auth
* Payment processing (OpenBanking, Bank Card, Video Game Currency, Crypto, etc...) and making the backend aware of any committed funds each user have
* Passtrough of read request from the frontend
* Propagating the backend webhook to frontend websockets when a payment is succesful, a user have been outbid, etc...

The API is completely frontend agnostic, however the frontend should not query the API directly, only the middleware. The DB being a mongodb database, it is extremely easy and intuitive for even non-initiated people (eg: Frontend Dev)

## Project Structure

### API 

The core of the software.

### Frontend 

This is just a simple frontend example showcasing what the UX should be. Because of the immense varitery of currency settlement system as well as user authentification mechanism, we can not at this time provide something that for sure can be instantly plug and played into your platform. However, feel free to copy paste components from this frontend to your own system.
Additionally, the admin dashboard can be used as is, in order to interract with the API or edit other conf.

### Middleware

The middleware that should be used with the API. Feel free to peek at it's code (it's very simple) and adapt it to your own needs. It is implemented as a node server, but again, YOU MUST MAKE YOUR OWN. No specifc implementation of user auth have been implemented. If you deploy it as is, anyone could place a bid on behalf of anyone else. Please make this middleware properly use YOUR OWN AUTH method. eg: Wallet signing for web3, session cookies for traditional web2, SSO, etc...

## API Routes    

Do not reccklessly add/edit/delete objects from the DB. Most of those routes are meant for admin/dashboard purposes. Your end-user frontend middleware should only be doing "List" instruction from the API for most uses cases, with potentially adding assets.
   
<p>Listed below are all the routes for the API. These are meant to follow the pattern of /<class>/<instruction>, and commands are shared for all types. <b>Pretty much every parameter to be sent will be of type string</b>, and type conversions if necessary are handled under the hood.</p>

Classes:

-   <b>Assets</b>
-   <b>Auctions</b>
-   <b>Balances</b>
-   <b>Bids</b>
-   <b>Currencies</b>
-   <b>Configurations</b>
-   <b>Ownerships</b>
-   <b>Portefeuilles</b>
-   <b>Presets</b>
-   <b>Royalties</b>
-   <b>Transactions</b>

Instructions: 

-   <b>Add</b> (POST)
-   <b>AddBatch</b> (POST)
-   <b>List</b> (GET)
-   <b>Update</b> (POST)
-   <b>Delete</b> (POST)

<br/>

<p>With the above, we can perform basic database manipulations by just calling the relevant route with GET/POST. For example, if we want to get a list of all auctions, then a GET request to /auctions/list/ is that's necessary to retrieve a json array of all auctions present in the database.</p>

<p>However, there is one notable exception to the above - the <b>transactions</b> route, and it ties in to how the API is expected to be called by a frontend middleware. While reading all info is a fairly straightforward process by just firing a GET to the list instruction of your requested class,you do not want to have everyone hitting an endpoint that directly modifies data. To save developers from dealing with the hassle of batch processing updates to the database, this is already being done - all transactions to the database are stored in a table of their own, which will then be worked on by a separate transaction processor that ensures that:

- All actions are dealt with in proper order 
- Invalid actions are rejected before any database update is made</p>

<b>As such, it is extremely important to default to using the transactions route over any manual update route</b>. The direct routes are there for both the transaction processor and an admin to use, such as for updating presets and configuration options. 

<br/>

### Examples for each route independently

<br/>

<details>
 <summary><b>Assets</b></summary>
 <br/>
 <details>
 <summary><b>[GET]</b> List</summary>


```javascript
"http://example.com/assets/list"
```

<p>Basic GET route that returns an array of all assets.</p>

<p>Optional parameter only:</p>

-   assetId - Returns the specified asset.

```javascript
"http://example.com/assets/list?assetId=test"
```

<p>Returns:</p>

```javascript
[
    {
        "assetId": "",
        "name":"",
        "description": "",
        "image": "",
        "thumbnail": "",
    },
    ...
]
```
</details>

 <details>
 <summary><b>[POST]</b> Add</summary>


```javascript
"http://example.com/assets/add"
```

<p>Post route to add a new asset to the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>assetId</b> - Needs to be unique
    -   <b>name</b>
    -   <b>description</b>
    -   <b>image</b> - The URL to the image
    -   <b>thumbnail</b> - The URL to a thumbnail image


<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> AddBatch</summary>

```javascript
"http://example.com/assets/addBatch"
```

 Same use as the previous one, just ensure the request body contains an array of JSON objects rather than sending a single one. 
</details>

<details>
 <summary><b>[POST]</b> Delete</summary>

 ```javascript
"http://example.com/assets/delete"
```

<p>Post route to remove an asset from the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>assetId</b>

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

</details>

<br/>

<details>
 <summary><b>Balances</b></summary>
 <br/>
 <details>
 <summary><b>[GET]</b> List</summary>


```javascript
"http://example.com/balances/list"
```

<p>Basic GET route that returns an array of all userBalances.</p>

<p>Optional parameter only:</p>

-   userId - Returns the specified user's balances.

```javascript
"http://example.com/balances/list?userId=test"
```

<p>Returns:</p>

```javascript
[
    {
        "userId": "",
        "currencyName":"",
        "currencyAmount": "",
    },
    ...
]
```
</details>

 <details>
 <summary><b>[POST]</b> Add</summary>


```javascript
"http://example.com/balances/add"
```

<p>Post route to add a new user balance to the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>userId</b>
    -   <b>currencyName</b>
    -   <b>currencyAmount</b>

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> AddBatch</summary>

```javascript
"http://example.com/balances/addBatch"
```

 Same use as the previous one, just ensure the request body contains an array of JSON objects rather than sending a single one. 
</details>

<details>
 <summary><b>[POST]</b> Delete</summary>

 ```javascript
"http://example.com/balances/delete"
```

<p>Post route to remove a user balance from the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>userId</b>

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> Update</summary>

 ```javascript
"http://example.com/balances/update"
```

<p>Post route to update a user balance.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>userId</b>
    -   <b>currencyName</b>
    -   <b>currencyAmount</b>

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

</details>

<br/>

<details>
 <summary><b>Bids</b></summary>
 <br/>
 <details>
 <summary><b>[GET]</b> List</summary>


```javascript
"http://example.com/bids/list"
```

<p>Basic GET route that returns an array of all bids.</p>

<p>No parameters are required</p>

<p>Returns:</p>

```javascript
[
    {
        "bidId": "", 
        "auctionId":"",
        "bidder": "",
        "bidAmount": "",
        "currencyId": "",
        "incentiveDue": "",
        "displaced": false
    },
    ...
]
```
</details>

 <details>
 <summary><b>[POST]</b> Add</summary>


```javascript
"http://example.com/bids/add"
```

<p>Post route to add a new bid to the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>bidId</b> - Needs to be unique
    -   <b>auctionId</b> - The ID of the related auction
    -   <b>bidder</b>
    -   <b>bidAmount</b> - As a string
    -   <b>currencyId</b> - The ID of the currency used
    -   <b>incentiveDue</b> - Calculated on the GBM side


<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> AddBatch</summary>

```javascript
"http://example.com/bids/addBatch"
```

 Same use as the previous one, just ensure the request body contains an array of JSON objects rather than sending a single one. 
</details>

<details>
 <summary><b>[POST]</b> Delete</summary>

 ```javascript
"http://example.com/bids/delete"
```

<p>Post route to remove a bid from the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>bidId</b> - Needs to be unique

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> Update</summary>

 ```javascript
"http://example.com/bids/update"
```

<p>Post route to update a bid's displaced status.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>bidId</b> - Needs to be unique

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

</details>

<br/>

<details>
 <summary><b>Configurations (Missing)</b></summary>
    NOT IMPLEMENTED YET
</details>

<br/>

<details>
 <summary><b>Currencies</b></summary>
 <br/>
 <details>
 <summary><b>[GET]</b> List</summary>


```javascript
"http://example.com/currencies/list"
```

<p>Basic GET route that returns an array of all currencies.</p>

<p>No parameters are required</p>

<p>Returns:</p>

```javascript
[
    {
        "currencyId": "",
        "name":"",
        "symbol": "",
        "icon": "",
    },
    ...
]
```
</details>

 <details>
 <summary><b>[POST]</b> Add</summary>


```javascript
"http://example.com/currencies/add"
```

<p>Post route to add a new currency to the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>currencyId</b> - Needs to be unique
    -   <b>name</b>
    -   <b>symbol</b> - i.e. USD
    -   <b>icon</b> - The URL to the icon


<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> AddBatch</summary>

```javascript
"http://example.com/currencies/addBatch"
```

 Same use as the previous one, just ensure the request body contains an array of JSON objects rather than sending a single one. 
</details>

<details>
 <summary><b>[POST]</b> Delete</summary>

 ```javascript
"http://example.com/currencies/delete"
```

<p>Post route to remove a currency from the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>currencyId</b>

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> Update</summary>

 ```javascript
"http://example.com/currencies/update"
```

<p>Post route to update a currency's information.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>currencyId</b> - The id of the one we wish to update
    -   <b>name</b> - The new name
    -   <b>symbol</b> - The new symbol i.e. USD
    -   <b>icon</b> - The new URL to the icon

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>


</details>

<br/>

<details>
 <summary><b>Ownerships</b></summary>
 <br/>
 <details>
 <summary><b>[GET]</b> List</summary>


```javascript
"http://example.com/ownerships/list"
```

<p>Basic GET route that returns an array of all ownership entities.</p>

<p>Optional parameter only:</p>

-   ownerId - Filter the returned ownerships to the specified owner.

```javascript
"http://example.com/ownerships/list?ownerId=test"
```

<p>Returns:</p>

```javascript
[
    {
        "assetId": "", 
        "ownerId":"",
        "amountOwned": "",
        "amountLocked": "",
        "amountUnderSale": "",
    },
    ...
]
```
</details>

 <details>
 <summary><b>[POST]</b> Add</summary>


```javascript
"http://example.com/ownerships/add"
```

<p>Post route to add a new ownership entity to the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>assetId</b> - The ID of the held asset
    -   <b>ownerId</b> - The ID of the owning user
    -   <b>amountOwned</b> - How many of the assets are in the user's wallet
    -   <b>amountLocked</b> - How many of the assets are in escrow
    -   <b>amountUnderSale</b> - How many are under sale


<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> AddBatch</summary>

```javascript
"http://example.com/ownerships/addBatch"
```

 Same use as the previous one, just ensure the request body contains an array of JSON objects rather than sending a single one. 
</details>

<details>
 <summary><b>[POST]</b> Delete</summary>

 ```javascript
"http://example.com/ownerships/delete"
```

<p>Post route to remove an ownership entity from the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>assetId</b> - The ID of the held asset
    -   <b>ownerId</b> - The ID of the owning user

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> Update</summary>

 ```javascript
"http://example.com/ownerships/update"
```

<p>Post route to update an ownership entity's information.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>assetId</b> - The ID of the held asset
    -   <b>ownerId</b> - The ID of the owning user
    -   <b>amountOwned</b> - How many of the assets are in the user's wallet
    -   <b>amountLocked</b> - How many of the assets are in escrow
    -   <b>amountUnderSale</b> - How many are under sale

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

</details>

<br/>

<details>
 <summary><b>Portefeuilles</b></summary>
 <br/>
 <details>
 <summary><b>[GET]</b> List</summary>


```javascript
"http://example.com/portefeuilles/list"
```

<p>Basic GET route that returns an array of all portefeuilles.</p>

<p>Optional parameter only:</p>

-   ownerId - Filter the returned portefeuilles to the specified owner.

```javascript
"http://example.com/portefeuilles/list?ownerId=test"
```

<p>Returns:</p>

```javascript
[
    {
        "currencyId": "", 
        "ownerId":"",
        "amountOwned": "",
        "amountLocked": "",
    },
    ...
]
```
</details>

 <details>
 <summary><b>[POST]</b> Add</summary>


```javascript
"http://example.com/portefeuilles/add"
```

<p>Post route to add a new portefeuille to the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>currencyId</b> - The ID of the held currency
    -   <b>ownerId</b> - The ID of the owning user
    -   <b>amountOwned</b> - Amount of the currency owned
    -   <b>amountLocked</b> - Amount of the currency in escrow


<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> AddBatch</summary>

```javascript
"http://example.com/portefeuilles/addBatch"
```

 Same use as the previous one, just ensure the request body contains an array of JSON objects rather than sending a single one. 
</details>

<details>
 <summary><b>[POST]</b> Delete</summary>

 ```javascript
"http://example.com/portefeuilles/delete"
```

<p>Post route to remove a portefeuille from the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>currencyId</b> - The ID of the held currency
    -   <b>ownerId</b> - The ID of the owning user

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> Update</summary>

 ```javascript
"http://example.com/portefeuilles/update"
```

<p>Post route to update a portefeuille's information.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>currencyId</b> - The ID of the held currency
    -   <b>ownerId</b> - The ID of the owning user
    -   <b>amountOwned</b> - Amount of the currency owned
    -   <b>amountLocked</b> - Amount of the currency in escrow

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

</details>

<br/>

<details>
 <summary><b>Presets (Missing)</b></summary>
    NOT IMPLEMENTED YET
</details>

<br/>

<details>
 <summary><b>Royalties</b></summary>
 <br/>
 <details>
 <summary><b>[GET]</b> List</summary>


```javascript
"http://example.com/royalties/list"
```

<p>Basic GET route that returns an array of all royalty entities.</p>

<p>Optional parameter only:</p>

-   royaltyId - Returns the specified royalty entity.

```javascript
"http://example.com/royalties/list?royaltyId=test"
```

<p>Returns:</p>

```javascript
[
    {
        "royaltyId": "", 
        "name":"",
        "percent": "",
        "destinationUser": "",
    },
    ...
]
```
</details>

 <details>
 <summary><b>[POST]</b> Add</summary>


```javascript
"http://example.com/royalties/add"
```

<p>Post route to add a new royalty to the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>royaltyId</b> - The ID of the royalty
    -   <b>name</b>
    -   <b>percent</b>
    -   <b>destinationUser</b> - Who will receive the royalty


<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> AddBatch</summary>

```javascript
"http://example.com/royalties/addBatch"
```

 Same use as the previous one, just ensure the request body contains an array of JSON objects rather than sending a single one. 
</details>

<details>
 <summary><b>[POST]</b> Delete</summary>

 ```javascript
"http://example.com/royalties/delete"
```

<p>Post route to remove a royalty entity from the database.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>royaltyId</b> - The ID of the royalty

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> Update</summary>

 ```javascript
"http://example.com/royalties/update"
```

<p>Post route to update a royalty's information.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

    -   <b>royaltyId</b> - The ID of the royalty
    -   <b>name</b>
    -   <b>percent</b>
    -   <b>destinationUser</b> - Who will receive the royalty

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

</details>

<br/>

## Transactions 

The /transactions/ endpoint operates with its own routes, with each named according to the purpose it serves: 

-   <b>createAuction</b> - For starting an auction
-   <b>placeBid</b> - For placing a bid on an auction
-   <b>topUp</b> - For a user to top up their balance
-   <b>withdraw</b> - For a user to withdraw from their balance

Each of those transactions fire back on a webhook when processed, as JSON, the entities impacted by the transaction in their final state at the end of the transaction. This is not meant to be sent directly to end user, but instead to be parsed by your middleware so that each user websocket/other notification method you implemented get notified independently with the relevant data.              

Transactions are also currently being expanded to support better non-comitted payment intents (eg: Placing a bid but paying with Card/Open banking, and waiting for payment confirmation before processing the bid).                          
topUp is to be called by the middleware everytime capital is confirmed as commited. Without enough capital commited in the relevant currency, a bid will fail.            
withdraw is to be called by the middleware everytime every time capital is uncommited.         

<br/>

<details>
 <summary><b>[POST]</b> Create Auction</summary>
 <br/>


```javascript
"http://example.com/transactions/createAuction"
```

<p>Tells the transaction processor to start an auction with the requested parameters.</p>

-   Required:

    -   <b>requestingUser</b> - The user starting the auction
    -   <b>nonce</b>
    -   <b>signature</b> - A signature verifying the user
    -   <b>arg_assetId</b> - The id of the asset to put on auction
    -   <b>arg_assetAmount</b> - How many of the assets will be put on sale
    -   <b>arg_startingBid</b> - The minimum bid
    -   <b>arg_reservePrice</b>
    -   <b>arg_baseCurrency</b> - The currency the auction will default to
    -   <b>arg_startTimestamp</b> - The starting time of the auction (UNIX timestamp)
    -   <b>arg_incentiveMin</b> - To be fetched from the GBM Preset
    -   <b>arg_incentiveMax</b> - To be fetched from the GBM Preset
    -   <b>arg_stepMin</b> - To be fetched from the GBM Preset
    -   <b>arg_bidMultiplier</b> - To be fetched from the GBM Preset
    -   <b>arg_duration</b> - To be fetched from the Duration Preset
    -   <b>arg_hammerTimeDuration</b> - To be fetched from the Duration Preset
    -   <b>arg_gracePeriodDuration</b> - To be fetched from the Duration Preset


<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

 <details>
 <summary><b>[POST]</b> Place Bid</summary>


```javascript
"http://example.com/transactions/placeBid"
```

<p>Tells the transaction processor to place a bid on an auction, with all the balance locking / bid displacing / incentive paying / etc. that entails.</p>

<p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

-   <b>requestingUser</b> - The user starting the auction
-   <b>nonce</b>
-   <b>signature</b> - A signature verifying the user
-   <b>arg_auctionId</b> - The id of the auction to bid on
-   <b>arg_bidAmount</b> - The amount on the new bid
-   <b>arg_bidCurrencyId</b> - The currencyId for the currency the new bid is under

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> Top Up</summary>

```javascript
"http://example.com/transactions/topUp"
```

 Tells the transaction processor to top up a user's balance by a requested amount.

 <p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

-   <b>requestingUser</b> - The user starting the auction
-   <b>nonce</b>
-   <b>signature</b> - A signature verifying the user
-   <b>arg_userBeingToppedUp</b> - ownerId of the balance to top up
-   <b>arg_amountToTopUp</b> - The amount to top up
-   <b>arg_currencyId</b> - The currencyId for the currency added

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

<details>
 <summary><b>[POST]</b> Withdraw</summary>

```javascript
"http://example.com/transactions/withdraw"
```

 Tells the transaction processor to top up a user's balance by a requested amount.

 <p> A JSON body must be sent, with an object holding the required parameters:</p>

-   Required:

-   <b>requestingUser</b> - The user starting the auction
-   <b>nonce</b>
-   <b>signature</b> - A signature verifying the user
-   <b>arg_userWithdrawing</b> - ownerId of the balance to withdraw from
-   <b>arg_amountToWithdraw</b> - The amount to withdraw
-   <b>arg_currencyId</b> - The currencyId for the currency withdrawn

<p>Returns:</p>

```javascript
{ 
    "success": "true" or "false"
}
```
</details>

</details>

<br/>


## SETUP

Your infra will look as follow : 

- A mongodb cluster. Either use MongoDB Atlas or launch your own. A cluster is necessary, a standalone MongoDB lack the transaction features being used.
- The API node server. This doesn't need to be open to the wider internet, only to your middleware. For better performance, host it near the DB (ie: Same Cloud VPC and physical location, maybe even same machine if reactivity is super important to you).
- A middleware for communicating with the API. This middleware is the only piece of software that should have API access that is not an admin tool. This middleware is for >YOU< to implement. We do however provide several example implementations that can be forkd, including one for FIAT Payment Processor, a barebone one, etc... 
- Your choice of end-user frontend/backend. We expect most of our client to already have an existing website/marketplace/auth. However, it's highly advised to stick to the bidding UX provided in the example frontend, as it handle GBM in a way intutive for bidders to understand.    
   
Please look into each repo for their separate docs, including how to setup a localhost test environement.   


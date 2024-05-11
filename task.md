# Creating your first unstoppable dApp on Ethereum

In this article, we will demonstrate how to deploy the simplest unstoppable dApp using the [ethfs-cli](https://github.com/ethstorage/ethfs-cli/) tool.

For the content to be deployed, let's assume that:
- There is an application folder (e.g., `dist`), which contains two files inside (e.g., `app.html` and [degen.jpeg](https://gist.github.com/assets/5291653/4526caf3-9218-4a23-8619-02f777e6e7fd)).
- The `app.html` file displays a greeting message from a smart contract and a degen image (264KB). It's very simple:
```
<html>
    <head>
        <script> 
            async function fetchData() { 
                // web3 URL is define in https://eips.ethereum.org/EIPS/eip-4804, please find more detail on https://web3url.io
                const url = 'web3://0xf14e64285Db115D3711cC5320B37264708A47f89:11155111/greeting'; 
                const response = await fetch(url); 
                const data = await response.text(); 
                document.getElementById('content').textContent = data; 
            } 
            window.onload = fetchData; 
        </script>
    </head>
    <body>
        <div id="content"> Loading greeting... </div>
        <br>
        <img  src="./degen.jpeg"  alt="">	
    </body>	
</html>
```
 - The referenced contract in app.html is also simple. It contains only one method that returns a string and is already deployed on Sepolia at `0xf14e64285Db115D3711cC5320B37264708A47f89`.
```
pragma solidity >=0.8.2 <0.9.0;
contract App {
    function greeting() public pure returns (string  memory) {
        return "LFG, Degen!";
    }
}
```
## Step 1: Install ethfs-cli
If you have not already done so, you can install ethfs-cli using the following command:
```
npm i -g ethfs-cli
```
## Step 2: Create a FlatDirectory Contract
A [FlatDirectory](https://docs.web3url.io/advanced-topics/flatdirectory) contract serves as a container that needs to be created before deploying files.

The following command creates a FlatDirectory on the Sepolia chain using the private key 0x112233...
```
$ ethfs-cli create -p 0x112233... -c 11155111
```
You will get a FlatDirectory address after the transaction is confirmed:
```
$ ethfs-cli create -p 0x112233... -c 11155111
chainId = 11155111
providerUrl = https://rpc.sepolia.org
FlatDirectory Address: 0x49EDFB27a463545337487D39a8349760B345F160
```
## Step 3: Deploy the application
In this section, you will upload the application folder into the FlatDirectory that you just created using the following command:
```
$ ethfs-cli upload -f <directory|file> -a <address> -c <chain-id> -p <private-key> -t <upload-type>
```
Note that you can choose the file upload type: '1' for Ethereum native storage, and '2' for EthStorage. Let's first try deploying the application using Ethereum’s native storage.
```
$ ethfs-cli upload -f dist -a <flat directory address> -c 11155111 -p 0x112233... -t 1

Start upload File.......
app.html, chunkId: 0
Transaction Id: 0x958f7d17dc9622b9a8f44badeb968aebd4dcbbaf4adc3d6079568eec93336692
degen.jpeg, chunkId: 0
Transaction Id: 0xe9620334064614b92ce195c19e42d6e672bd76c55f7551609d7bf1066467177d
...
```
It took 12 transactions to deploy the application, costing us more than 0.3 ETH, due to the high costs of Ethereum's native storage. Nonetheless, now you can access this unstoppable dApp via web3://

[web3://0x49EDFB27a463545337487D39a8349760B345F160:11155111/app.html](https://0x49edfb27a463545337487d39a8349760b345f160.sep.w3link.io/app.html)

## Step 4: Use EIP-4844 BLOB to reduce the uploading cost
Ethereum's Cancun upgrade introduced BLOBs, which greatly reduced the data publishing cost for Layer 2. Naturally, this raises the question: can we use it to upload our application? The answer is YES! The EthStorage team developed a tool named [eth-blob-uploader](https://www.npmjs.com/package/eth-blob-uploader) that helps us upload files using EIP-4844 BLOBs! Follow the instructions below to give it a try:
```
// install the package first
npm i -g eth-blob-uploader

// upload the two files, you can use our self-built Sepolia RPC: http://88.99.30.186:8545
eth-blob-uploader -r <Sepolia RPC> -p 0x112233... -f app.html -t <any address>
eth-blob-uploader -r <Sepolia RPC> -p 0x112233... -f app.html -t <any address>   
```
You can see it only takes 2 transactions to upload the two files, and the cost is incredibly low when the BLOB gas price is low!

## Step 5: Use EthStorage to store the BLOB permanently
BLOBs are great for reducing upload costs, but they're only kept for about 18 days, and users can't access the data through smart contracts. This greatly limits data composability and programmability compared to native ETH storage. People may understand the trade-off and say, 'You can't have your cake and eat it too.' 

But wait, EthStorage can help store BLOBs permanently and enable data retrieval through smart contracts! Let's see what EthStorage contract's interface looks like:
```
// @notice Evaluate the storage cost of a putBlob().
function upfrontPayment() public view returns (uint256);

/// @notice Write a blob to KV store. Need to pay the storage fee by using msg.value.
/// @param key     The key corresponding to the value that is being written
/// @param blobIdx The index of the blob that is being written
/// @param length  The length of the value that is being written
function putBlob(bytes32 key, uint256 blobIdx, uint256 length) public payable;    

/// @notice Return the keyed data given off and len. This function can be only called in JSON-RPC context of ES L2 node.
/// @param key        The key corresponding to the value
/// @param decodeType How do we want to decode the keyed value
/// @param off        The offset of the value that will be returned
/// @param len        The length of the value that will be returned
function get(
    bytes32 key,
    DecodeType decodeType,
    uint256 off,
    uint256 len
) public view returns (bytes memory);
```
So, EthStorage will be involved in BLOB uploading and retrieval:
 - **BLOB uploading**: When users send a BLOB transaction and want to store it permanently, they simply need to call the `putBlob` method on the EthStorage contract within the same transaction. -   They will also need to send the EthStorage contract a permanent storage fee, which can be obtained via `upfrontPayment`.
 - **BLOB retrieval**: Users can retrieve BLOB data directly using the `get` method. Please note that this function can only be called through the EthStorage Layer 2 RPC node. The endpoint on Sepolia is http://65.108.236.27:9540.

What's more convenient for end users is that they don't need to interact directly with the EthStorage contract because the `ethfs-cli` tool provides a command to use EthStorage by setting -t to 2 when deploying the dApp!

## Step 6: Deploying the Simple dApp with ETH+EthStorage
Let's  try deploying the dApp using ETH+EthStorage.
```
// deploy another FlatDirectory contract
$ ethfs-cli create -p 0x112233... -c 11155111

// upload the same application using file upload type 2
ethfs-cli upload -f dist -a <flat directory address> -c 11155111 -p 0x112233... -t 2
```
The deployment required only 2 transactions, costing us 0.024 ETH, with the blob price at 31 GWei at the time of deployment. As a result, the upload constituted a significant portion of the total cost.

You can access this unstoppable dApp via web3://:

[web3://0x56b4C1B43633cE048e4D03664D31B18e56ce6943:3333/app.html](https://0x56b4c1b43633ce048e4d03664d31b18e56ce6943.3333.w3link.io/app.html)

## Bonus: Retrive BLOB data dynamically
As mentioned in Step 5, EthStorage is incredibly powerful for enabling BLOB data retrieval through a smart contract interface, which makes BLOB data more composable and programmable. For example, you can store multiple blogs or images on EthStorage, retrieve them dynamically using EthStorage's `get` methods, and display them on your front end—or anything else you can imagine related to the composition of large data chunks.

We encourage everyone to unleash their creativity using this powerful feature and build a super cool dApp with EthStorage in this way. The team will certainly award additional points for these submissions!
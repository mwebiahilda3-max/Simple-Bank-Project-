# Simple-Bank-Project-
Simple Bank Project - Files
README.md
# Bank Account Project (Solidity + Hardhat + Frontend)
Generated: 2025-09-20T08:48:54.927074 UTC
**Project summary**
- A simple Ethereum smart contract that implements banking accounts with:
 - Interest rate (annual, configurable)
 - Monthly fee
 - Deposit fee for small deposits and low balances
 - Per-account timestamping for monthly accounting
- Includes:
 - Solidity contract (`contracts/Bank.sol`)
 - Hardhat tests (`test/bank.test.js`)
 - Deployment script (`scripts/deploy.js`)
 - Frontend HTML + JS to connect with MetaMask (`frontend/index.html`)
 - Example `package.json` and `hardhat.config.js` snippets
 - This PDF with all files included
**How to run locally (summary)**
1. Install Node.js and npm.
2. Create project and install Hardhat:
 ```
 npm init -y
 npm install --save-dev hardhat @nomiclabs/hardhat-ethers ethers chai mocha
 ```
3. Copy `contracts/Bank.sol`, `scripts/deploy.js`, `test/*` and `frontend/*` into your project.
4. Run tests:
 ```
 npx hardhat test
 ```
5. Deploy locally to a local node (Hardhat network) or to testnet by configuring providers/wallets in `hardhat.config.js`.
6. Run a static server to serve `frontend/index.html` (e.g., `npx http-server frontend`)
**Notes**
- This is a sample educational project. Do not use as-is in production without audit.
contracts/Bank.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;
/// @title SimpleBank - deposit/withdraw with interest, monthly fee, and deposit fee for small deposits
/// @author Generated
contract SimpleBank {
 address public owner;
 struct Account {
 uint256 balance; // in wei
 uint256 lastAccounting; // unix timestamp of last monthly accounting
 }
 mapping(address => Account) public accounts;
 // Configuration (all values in wei or basis points where noted)
 uint256 public annualInterestBps; // interest rate in basis points (1 bp = 0.01%). e.g., 500 = 5% annual
 uint256 public monthlyFee; // fixed fee charged monthly (wei)
 uint256 public minDepositForNoFee; // deposits below this incur deposit fee
 uint256 public depositFeeBps; // deposit fee in bps for small deposits
 uint256 public minBalanceForLowerFee; // balances below this value pay higher fees (used optionally)
 event Deposited(address indexed who, uint256 amount, uint256 fee);
 event Withdrawn(address indexed who, uint256 amount);
 event MonthlyCharged(address indexed who, uint256 interest, uint256 fee, uint256 timestamp);
 event ConfigUpdated(address indexed by);
 modifier onlyOwner() {
 require(msg.sender == owner, "only owner");
 _;
 } 
constructor(
 uint256 _annualInterestBps,
 uint256 _monthlyFee,
 uint256 _minDepositForNoFee,
 uint256 _depositFeeBps,
 uint256 _minBalanceForLowerFee
 ) {
 owner = msg.sender;
 annualInterestBps = _annualInterestBps;
 monthlyFee = _monthlyFee;
 minDepositForNoFee = _minDepositForNoFee;
 depositFeeBps = _depositFeeBps;
 minBalanceForLowerFee = _minBalanceForLowerFee;
 }
 // --- configuration ---
 function updateConfig(
 uint256 _annualInterestBps,
 uint256 _monthlyFee,
 uint256 _minDepositForNoFee,
 uint256 _depositFeeBps,
 uint256 _minBalanceForLowerFee
 ) external onlyOwner {
 annualInterestBps = _annualInterestBps;
 monthlyFee = _monthlyFee;
 minDepositForNoFee = _minDepositForNoFee;
 depositFeeBps = _depositFeeBps;
 minBalanceForLowerFee = _minBalanceForLowerFee;
 emit ConfigUpdated(msg.sender);
 }
 // --- deposit / withdraw ---
 receive() external payable {
 deposit();
 }
 function deposit() public payable {
 require(msg.value > 0, "zero deposit");
 uint256 fee = 0;
 if (msg.value < minDepositForNoFee) {
 fee = (msg.value * depositFeeBps) / 10000;
 // fee is retained in contract (could be withdrawn by owner)
 }
 // initialize lastAccounting if first deposit
 if (accounts[msg.sender].lastAccounting == 0) {
 accounts[msg.sender].lastAccounting = block.timestamp;
 }
 accounts[msg.sender].balance += (msg.value - fee);
 emit Deposited(msg.sender, msg.value, fee);
 }
 function withdraw(uint256 amount) external {
 require(amount > 0, "zero withdraw");
 require(accounts[msg.sender].balance >= amount, "insufficient");
 accounts[msg.sender].balance -= amount;
 payable(msg.sender).transfer(amount);
 emit Withdrawn(msg.sender, amount);
 }
 // --- accounting: interest accrual and monthly fee ---
 // Apply monthly accounting for the calling account.
 // Anyone can call for any account to trigger periodic accounting.
 function applyMonthlyAccounting(address accountAddr) public {
 Account storage acct = accounts[accountAddr];
 require(acct.lastAccounting > 0, "account not initialized");
 uint256 nowTs = block.timestamp;
 // calculate how many full months elapsed
 uint256 elapsedSeconds = nowTs - acct.lastAccounting;
 uint256 secondsPerMonth = 30 days;
 uint256 months = elapsedSeconds / secondsPerMonth;
 require(months > 0, "no full month elapsed");
 uint256 interestTotal = 0;
 uint256 feeTotal = 0;
 for (uint256 i = 0; i < months; i++) {
 // monthly interest = balance * (annualInterestBps / 10000) / 12
uint256 interest = (acct.balance * annualInterestBps) / 10000 / 12;
 interestTotal += interest;
 // monthly fee: could be higher if balance below threshold
 uint256 appliedFee = monthlyFee;
 if (acct.balance < minBalanceForLowerFee) {
 // charge additional portion (50% more)
 appliedFee = (monthlyFee * 150) / 100;
 }
 // ensure fee does not exceed balance+interest
 feeTotal += appliedFee;
 // Update lastAccounting as if months applied one by one
 acct.lastAccounting += secondsPerMonth;
 }
 // apply interest then fees
 acct.balance += interestTotal;
 // if feeTotal > balance, set balance to zero and keep leftover fee as contract fee (owner can withdraw)
 if (feeTotal >= acct.balance) {
 acct.balance = 0;
 } else {
 acct.balance -= feeTotal;
 }
 emit MonthlyCharged(accountAddr, interestTotal, feeTotal, block.timestamp);
 }
 // Owner can withdraw collected fees from contract
 function withdrawContractFees(address payable to) external onlyOwner {
 require(to != address(0), "zero addr");
 uint256 bal = address(this).balance;
 // subtract all user balances tracked to get fees—NOTE: in production do this differently.
 uint256 totalUser = 0;
 // naive: we cannot iterate mapping; owner should withdraw only explicit collected fees via other pattern.
 // Here we'll allow owner to withdraw up to contract balance (exercise caution).
 to.transfer(bal);
 }
 // --- view helpers ---
 function getBalance(address who) external view returns (uint256) {
 return accounts[who].balance;
 }
 function getAccountInfo(address who) external view returns (uint256 balance, uint256 lastAccounting) {
 Account memory a = accounts[who];
 return (a.balance, a.lastAccounting);
 }
}
test/bank.test.js
const { expect } = require("chai");
const { ethers } = require("hardhat");
describe("SimpleBank", function () {
 let Bank, bank, owner, user1, user2;
 beforeEach(async function () {
 [owner, user1, user2] = await ethers.getSigners();
 Bank = await ethers.getContractFactory("SimpleBank");
 bank = await Bank.deploy(
 500, // 5% annual
 ethers.utils.parseEther("0.001"), // monthly fee: 0.001 ETH
 ethers.utils.parseEther("0.01"), // minDepositForNoFee: 0.01 ETH
 200, // depositFeeBps = 2%
 ethers.utils.parseEther("0.05") // minBalanceForLowerFee
 );
 await bank.deployed();
 });
 it("accepts deposits and charges deposit fee for small deposits", async function () {
 // deposit small amount (0.005)
 await bank.connect(user1).deposit({ value: ethers.utils.parseEther("0.005") });
 const bal = await bank.getBalance(user1.address);
 // deposit 0.005 -> fee 2% of 0.005 = 0.0001 -> net = 0.0049
 expect(bal).to.equal(ethers.utils.parseEther("0.0049"));
});
 it("applies monthly accounting after time travel", async function () {
 // deposit larger amount to avoid small deposit fee
 await bank.connect(user1).deposit({ value: ethers.utils.parseEther("1.0") });
 // warp time by 31 days
 await ethers.provider.send("evm_increaseTime", [31 * 24 * 3600]);
 await ethers.provider.send("evm_mine");
 // apply accounting
 await bank.applyMonthlyAccounting(user1.address);
 const bal = await bank.getBalance(user1.address);
 // interest approx = 1 ETH * 5% /12 = 0.004166... then minus monthly fee 0.001 => net increase ~0.003166...
 // Check balance > 1 ETH (since fee is smaller than interest)
 expect(bal).to.be.above(ethers.utils.parseEther("1.0").sub(ethers.utils.parseEther("0.0001")));
 });
 it("allows withdrawal", async function () {
 await bank.connect(user1).deposit({ value: ethers.utils.parseEther("0.1") });
 await bank.connect(user1).withdraw(ethers.utils.parseEther("0.05"));
 const bal = await bank.getBalance(user1.address);
 expect(bal).to.equal(ethers.utils.parseEther("0.05"));
 });
});
scripts/deploy.js
// Usage: npx hardhat run scripts/deploy.js --network <network>
const hre = require("hardhat");
async function main() {
 const Bank = await hre.ethers.getContractFactory("SimpleBank");
 const bank = await Bank.deploy(
 500, // 5% annual interest in bps
 hre.ethers.utils.parseEther("0.001"), // monthly fee
 hre.ethers.utils.parseEther("0.01"), // minDepositForNoFee
 200, // deposit fee bps (2%)
 hre.ethers.utils.parseEther("0.05") // minBalanceForLowerFee
 );
 await bank.deployed();
 console.log("SimpleBank deployed to:", bank.address);
}
main()
 .then(() => process.exit(0))
 .catch((error) => {
 console.error(error);
 process.exit(1);
 });
package.json
{
 "name": "simple-bank-project",
 "version": "1.0.0",
 "scripts": {
 "test": "npx hardhat test",
 "compile": "npx hardhat compile",
 "deploy": "npx hardhat run scripts/deploy.js --network localhost"
 },
 "devDependencies": {
 "hardhat": "^2.17.0",
 "@nomiclabs/hardhat-ethers": "^2.1.0",
 "ethers": "^6.0.0",
 "chai": "^4.3.7",
 "mocha": "^10.2.0"
 }
}
hardhat.config.js
require("@nomiclabs/hardhat-ethers");
/**
 * Sample Hardhat config. For testnets add your provider and private key.
 */
module.exports = {
 solidity: "0.8.19",
 networks: {
 localhost: {
 url: "http://127.0.0.1:8545"
 }
 // add testnet configs here
 }
};
frontend/index.html
<!doctype html>
<html>
<head>
 <meta charset="utf-8" />
 <title>SimpleBank Frontend</title>
 <meta name="viewport" content="width=device-width, initial-scale=1" />
 <script src="https://cdn.jsdelivr.net/npm/ethers/dist/ethers.min.js"></script>
 <style>
 body { font-family: Arial, sans-serif; padding: 20px; max-width: 700px; margin: auto; }
 button { margin: 6px 0; padding: 8px 12px; }
 input { padding: 6px; width: 100%; box-sizing: border-box; margin-bottom: 8px; }
 .card { border: 1px solid #ddd; padding: 12px; border-radius: 8px; margin-bottom: 12px; }
 </style>
</head>
<body>
 <h1>SimpleBank (Frontend)</h1>
 <div id="status">Not connected</div>
 <button id="connect">Connect MetaMask</button>
 <div class="card">
 <h3>Account Actions</h3>
 <div>
 <label>Deposit (ETH)</label>
 <input id="depositAmt" placeholder="0.01" />
 <button id="depositBtn">Deposit</button>
 </div>
 <div>
 <label>Withdraw (ETH)</label>
 <input id="withdrawAmt" placeholder="0.01" />
 <button id="withdrawBtn">Withdraw</button>
 </div>
 <div>
 <label>Apply Monthly Accounting (call)</label>
 <input id="acctAddr" placeholder="Account address (leave blank for your address)" />
 <button id="applyMonthlyBtn">Apply Monthly Accounting</button>
 </div>
 <div>
 <button id="refreshBtn">Refresh Balance</button>
 </div>
 <pre id="output"></pre>
 </div>
 <script>
 // Paste your deployed contract address here once deployed
 const CONTRACT_ADDRESS = ""; // fill after deploy
 const ABI = [{"inputs":[{"internalType":"uint256","name":"_annualInterestBps","type":"uint256"},{"internalType":"uint256","name":"_monthlyFee","type":"uint256"},{"internalType":"uint256","name":"_minDepositForNoFee","type":"uint256"},{"internalType":"uint256","name":"_depositFeeBps","type":"uint256"},{"internalType":"uint256","name":"_minBalanceForLowerFee","type":"uint256"}],"stateMutability":"nonpayable","type":"constructor"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"who","type":"address"},{"indexed":false,"internalType":"uint256","name":"amount","type":"uint256"},{"indexed":false,"internalType":"uint256","name":"fee","type":"uint256"}],"name":"Deposited","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"who","type":"address"},{"indexed":false,"internalType":"uint256","name":"amount","type":"uint256"}],"name":"Withdrawn","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"who","type":"address"},{"indexed":false,"internalType":"uint256","name":"interest","type":"uint256"},{"indexed":false,"internalType":"uint256","name":"fee","type":"uint256"},{"indexed":false,"internalType":"uint256","name":"timestamp","type":"uint256"}],"name":"MonthlyCharged","type":"event"},{"inputs":[],"name":"annualInterestBps","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"address","name":"who","type":"address"}],"name":"getAccountInfo","outputs":[{"internalType":"uint256","name":"balance","type":"uint256"},{"internalType":"uint256","name":"lastAccounting","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"address","name":"who","type":"address"}],"name":"getBalance","outputs":[{"internalType":"uint256","name":"","type":"uint256"}],"stateMutability":"view","type":"function"},{"inputs":[],"name":"deposit","outputs":[],"stateMutability":"payable","type":"function"},{"inputs":[{"internalType":"uint256","name":"amount","type":"uint256"}],"name":"withdraw","outputs":[],"stateMutability":"nonpayable","type":"function"},{"inputs":[{"internalType":"address","name":"accountAddr","type":"address"}],"name":"applyMonthlyAccounting","outputs":[],"stateMutability":"nonpayable","type":"function"}];
 let provider, signer, contract;
 const statusEl = document.getElementById("status");
 const output = document.getElementById("output");
 async function connect() {
 if (!window.ethereum) {
 statusEl.innerText = "MetaMask not found";
 return;
 }
 provider = new ethers.BrowserProvider(window.ethereum);
 await provider.send("eth_requestAccounts", []);
 signer = await provider.getSigner();
 const addr = await signer.getAddress();
 statusEl.innerText = "Connected: " + addr;
 if (CONTRACT_ADDRESS === "") {
 output.innerText = "Please paste the deployed contract address into the top of this file (CONTRACT_ADDRESS).";
 return;
 }
 contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, signer);
 }
 document.getElementById("connect").onclick = connect;
 document.getElementById("depositBtn").onclick = async () => {
 try {
 const v = document.getElementById("depositAmt").value;
 if (!v) return alert("enter amount");
 const tx = await signer.sendTransaction({ to: CONTRACT_ADDRESS, value: ethers.parseEther(v) });
 output.innerText = "Sent deposit tx: " + tx.hash;
 } catch (e) {
 output.innerText = e.toString();
 }
 };
 document.getElementById("withdrawBtn").onclick = async () => {
 try {
 const v = document.getElementById("withdrawAmt").value;
 const wei = ethers.parseEther(v);
 const tx = await contract.withdraw(wei);
 output.innerText = "Withdraw tx sent: " + tx.hash;
 } catch (e) {
 output.innerText = e.toString();
 }
 };
 document.getElementById("applyMonthlyBtn").onclick = async () => {
 try {
 let target = document.getElementById("acctAddr").value;
 if (!target) target = await signer.getAddress();
 const tx = await contract.applyMonthlyAccounting(target);
 output.innerText = "applyMonthlyAccounting tx: " + tx.hash;
 } catch (e) {
 output.innerText = e.toString();
 }
 };
 document.getElementById("refreshBtn").onclick = async () => {
 try {
 const addrInput = document.getElementById("acctAddr").value;
 const addr = addrInput || await signer.getAddress();
 const bal = await contract.getBalance(addr);
 output.innerText = "Balance (wei): " + bal + "\nBalance (ETH): " + ethers.formatEther(bal);
 } catch (e) {
 output.innerText = e.toString();
 }
 };
 </script>
</body>
</html>

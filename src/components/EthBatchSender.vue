<template>
  <h1>ETH Batch Sending</h1>
  <p>
    Save gas by sending ETH to 4 or more addresses at once in a single transaction.<br><span
      class="details">(because of limitations imposed by the Ethereum VM this tool doesn't
    support contracts or unused addresses)</span>
  </p>

  <p>
    Enter destination addresses with ETH amounts, one per line:<br/>
    <textarea id="dests" v-model="destinationsCsv"
              placeholder="0xdBCBA6C9D5edBD9C110c90556035E3aA7e2E2C0e,0.123  # add one destination per line"></textarea>
  </p>

  <p v-if="messages" class="details">
    Last message/error: {{ messages }}
  </p>

  <p v-if="!metamaskConnected">
    <span>Not connected to Metamask! </span>
    <button @click="connectMetamask" v-if="!metamaskConnected">
      (Step 1): Connect to Metamask
    </button>
  </p>

  <p v-if="metamaskConnected && typeof eth !== 'undefined'" class="details">
    Selected Metamask Account (sender address): {{ senderAccount }}<br/>
    Selected Metamask Network:
    <span :class="'eth_'+eth.chainId">{{ getChainNameById(eth.chainId) }}</span>
  </p>


  <p v-if="transactionId" class="txid">
    Transaction created! TxId: {{ transactionId }}<br>
    <span v-if="eth && eth.chainId==='0x1'">
      <a :href="'https://etherscan.io/tx/'+transactionId"
         target="_blank">{{ 'https://etherscan.io/tx/' + transactionId }}</a>
    </span>
    <span v-if="eth && eth.chainId==='0x3'">
      <a :href="'https://ropsten.etherscan.io/tx/'+transactionId"
         target="_blank">{{ 'https://ropsten.etherscan.io/tx/' + transactionId }}</a>
    </span>
    <span v-if="eth && eth.chainId==='0x4'">
      <a :href="'https://rinkeby.etherscan.io/tx/'+transactionId"
         target="_blank">{{ 'https://rinkeby.etherscan.io/tx/' + transactionId }}</a>
    </span>
    <span v-if="eth && eth.chainId==='0x5'">
      <a :href="'https://goerli.etherscan.io/tx/'+transactionId"
         target="_blank">{{ 'https://goerli.etherscan.io/tx/' + transactionId }}</a>
    </span>
    <span v-if="eth && eth.chainId==='0x42'">
      <a :href="'https://kovan.etherscan.io/tx/'+transactionId"
         target="_blank">{{ 'https://kovan.etherscan.io/tx/' + transactionId }}</a>
    </span>
  </p>

  <div v-if="destinationList.length>0">
    <div v-if="destinationErrorCount === 0 && !destinationsCheckedOnChain && metamaskConnected">
      <button @click="checkAddressesForContracts">
        (Step 2): Check if addresses are no contracts and have been used before
      </button>
    </div>

    <div
        v-if="destinationList.length >= 4 && !transactionId && destinationErrorCount === 0 && destinationsCheckedOnChain && metamaskConnected && destinationOnchainErrorCount === 0">
      <button @click="buildAndSendTransaction">
        (Step 3): Create transaction for MetaMask
      </button>
    </div>

    <h2>Transaction content:</h2>
    <div v-if="senderAccount" class="refundAddress">
      Refund address where remaining ETH will go back to: {{ senderAccount }}
    </div>

    <p>
      Number of destinations: {{ destinationList.length }}<br/>
    </p>

    <p v-if="destinationList.length < 4" class="error">
      The list contains less than 4 destinations. Please use at least
      4 destinations, otherwise it won't be cheaper than single transactions.
    </p>

    <p v-if="destinationErrorCount > 0" class="error">
      {{ destinationErrorCount }} of the lines contain some error. Please correct them before you
      continue!
    </p>
    <p v-if="destinationsCheckedOnChain && destinationOnchainErrorCount>0"
       class="error">
      {{ destinationOnchainErrorCount }} of the destinations are either smart
      contracts or
      have never been used on-chain. Sending to these addresses will not be cheaper in batch! Please
      remove them.
    </p>

    <br/>
    <table>
      <tr>
        <td>Destination address</td>
        <td>Amount</td>
        <td>state checks</td>
      </tr>
      <tr v-for="dest in destinationList" :key="dest.address">
        <td v-if="dest.error" colspan="3"
            class="errorRow">
          ERROR: {{ dest.error }} (raw line: '{{ dest.rawLine }}')
        </td>
        <td v-if="!dest.error" class="okRow">{{ dest.address }}
        </td>
        <td v-if="!dest.error" class="okRow">{{ fromWeiStringToEth(dest.amountInWei) }} ETH</td>
        <td v-if="!dest.error && !destinationOnChainErrors[dest.address]" class="uncheckedRow">not
          checked (yet)
        </td>
        <td v-if="!dest.error && destinationOnChainErrors[dest.address] && destinationOnChainErrors[dest.address].ok"
            class="okRow">OK
        </td>
        <td v-if="!dest.error && destinationOnChainErrors[dest.address] && !destinationOnChainErrors[dest.address].ok"
            class="errorRow">
          {{ destinationOnChainErrors[dest.address].msg }}
        </td>
      </tr>
    </table>

  </div>


</template>

<script>

import {fromWei, isAddress, leftPad, numberToHex, toWei} from "web3-utils";

const BN = require('bn.js');

// this is just an upper bound, no worries, you decide YOURSELF which gasprice you want to set in MetaMask
// all the remaining ETH go back to the sending address
const MAX_GASPRICE_FOR_AMOUNT_CALCULATION = toWei('1000', 'gwei');

export default {
  data: function () {
    return {
      // CSV textarea text
      destinationsCsv: '',
      // destinations as parsed objects
      destinationList: [],
      destinationOnChainErrors: {},
      destinationsCheckedOnChain: false,

      // metamask
      metamaskConnected: false,
      eth: undefined,
      senderAccount: undefined,
      transactionId: undefined,
      // info for user
      messages: ''
    }
  },

  watch: {
    destinationsCsv(newText) {
      this.destinationList = this.parseDestinationsCsv(newText);
      this.destinationsCheckedOnChain = false;
      this.destinationOnChainErrors = {};
      this.transactionId = undefined;
    }
  },

  computed: {
    destinationErrorCount() {
      return this.destinationList.filter(elem => {
        return elem.error
      }).length;
    },
    destinationOnchainErrorCount() {
      return Object.keys(this.destinationOnChainErrors).filter(elem => {
        return !this.destinationOnChainErrors[elem].ok
      }).length;
    }

  },

  methods: {
    async buildAndSendTransaction() {
      let data = '0x';

      for (let i = 0; i < this.destinationList.length; i++) {
        let dest = this.destinationList[i];

        // some checks (again)
        if (!this.destinationsCheckedOnChain) {
          throw Error('SAFETY CHECK: onchain check not done yet!');
        }
        if (!this.destinationOnChainErrors[dest.address].ok) {
          console.error('SAFETY CHECK: onchain check seems to contain an error for this destination:');
          console.error(this.destinationOnChainErrors[dest.address]);
          this.messages = 'SAFETY CHECK: onchain check seems to contain an error for this destination';
          throw Error('SAFETY CHECK: onchain check seems to contain an error for this destination');
        }
        if (dest.error) {
          this.messages = 'SAFETY CHECK: destination has some error!! ' + dest.error;
          throw Error('SAFETY CHECK: destination has some error!! ' + dest.error);
        }
        // check again, if looks like valid address
        this.assertAddressWithPrefix(dest.address);

        // OP_PUSH1 00 | OP_DUP1  | OP_DUP1  | OP_DUP1 | OP_PUSH9 (9 bytes for amount in wei)
        //   60     00     80         80         80        68
        data += '600080808068'
        let amountWeiBn = new BN(dest.amountInWei);
        let amountInHex = numberToHex(amountWeiBn);
        if (amountInHex.substring(0, 2) !== '0x') {
          this.messages = 'SAFETY CHECK: amountInHex doesnt start with 0x';
          throw Error('SAFETY CHECK: amountInHex doesnt start with 0x');
        }
        // with 0x
        let paddedHexString = leftPad(amountInHex.substring(2), 18);
        if (typeof paddedHexString !== 'string') {
          console.error('SAFETY CHECK: paddedHexString is not type of string:');
          console.error(typeof paddedHexString);
          console.error(paddedHexString);
          this.messages = 'SAFETY CHECK: paddedHexString is not type of string';
          throw Error('SAFETY CHECK: paddedHexString is not type of string');
        }
        if (paddedHexString.length !== 18) {
          console.error('SAFETY CHECK: amount was not exactly 9 bytes long:');
          console.error(paddedHexString);
          this.messages = 'SAFETY CHECK: amount was not exactly 9 bytes long';
          throw Error('SAFETY CHECK: amount was not exactly 9 bytes long');
        }
        // then exactly 9 bytes of amount
        data += paddedHexString;

        // OP_PUSH20
        data += '73';
        // address without '0x'
        data += dest.address.substring(2).toLowerCase();
        // OP_DUP3 (duplicate one of the 00s before) | and finally OP_CALL (f1)
        data += '82f1'
      }

      // also here check again, if looks like valid address
      this.assertAddressWithPrefix(this.senderAccount);

      // OP_PUSH20
      data += '73';
      // push selfdestruct destination address without '0x'
      data += this.senderAccount.substring(2).toLowerCase();
      // OP_SELFDESTRUCT
      data += 'ff';

      // again, some final sanity checks:
      let expectedLen = (this.destinationList.length * 76) + 44 + 2;
      if (data.length !== expectedLen) {
        this.messages = 'data doesnt have expected length! Expected ' + expectedLen + ' but was actually: ' + data.length + '. Check data: ' + data;
        throw Error('data doesnt have expected length! Expected ' + expectedLen + ' but was actually: ' + data.length + '. Check data: ' + data);
      }

      if (!data.match(/^0x[0-9a-f]+/)) {
        this.messages = 'SAFETY CHECK: data doesnt seem to contain only lowercase hex characters! Check data: ' + data;
        throw Error('SAFETY CHECK: data doesnt seem to contain only lowercase hex characters! Check data: ' + data);
      }

      if (data.substring(data.length - 2) !== 'ff') {
        this.messages = 'SAFETY CHECK: data doesnt seem to end with OP_SELFDESTRUCT (ff): ' + data;
        throw Error('SAFETY CHECK: data doesnt seem to end with OP_SELFDESTRUCT (ff): ' + data);
      }

      if (data.substring(data.length - 42, data.length - 2) !== this.senderAccount.substring(2).toLowerCase()) {
        this.messages = 'SAFETY CHECK: the last 40 characters before selfdestruct dont look like sender address: ' + data;
        throw Error('SAFETY CHECK: the last 40 characters before selfdestruct dont look like sender address: ' + data);
      }

      if (data.substring(data.length - 44, data.length - 42) !== '73') {
        this.messages = 'SAFETY CHECK: the last OP-code before refund address seems not to be a PUSH20 (73): ' + data;
        throw Error('SAFETY CHECK: the last OP-code before refund address seems not to be a PUSH20 (73): ' + data);
      }

      let gasLimit = this.estimateGasLimitForNumdestinations(this.destinationList.length);

      this.messages = 'Send transaction to MetaMask...'

      this.eth.request({
        method: 'eth_sendTransaction',
        params: [
          {
            from: this.senderAccount,
            to: null,
            data: data,
            gas: gasLimit,
            value: this.estimateAmount(this.destinationList, gasLimit, MAX_GASPRICE_FOR_AMOUNT_CALCULATION)
          },
        ],
      })
          .then((txHash) => {
            this.transactionId = txHash;
            this.messages = 'Transaction created: ' + txHash;
            console.log('Transaction created: ' + txHash);
          })
          .catch((err) => {
            this.messages = 'Error callback from MetaMask: ' + err.message;
            console.error(err);
          });

    },

    fromWeiStringToEth(wei) {
      return fromWei(wei, 'ether');
    },

    estimateGasLimitForNumdestinations(number) {
      let estimatedGas = 33659 + number * 10333 + 24000; // the 24000 will be deducted again during self-destruct, but they need to be available while the transacton is running.
      let gasLimitFinal = numberToHex(Math.round(estimatedGas * 1.05));  // add some safety overhead
      return gasLimitFinal;
    },

    estimateAmount(list, gasLimitHexString, maxGaspriceInWei) {
      if (!list || list.length < 1) {
        throw Error('estimateAmount: list was empty');
      }
      if (!gasLimitHexString || typeof gasLimitHexString !== 'string' || gasLimitHexString.substring(0, 2) !== '0x') {
        throw Error('estimateAmount: gasLimitHexString is no hex string: ' + gasLimitHexString);
      }
      if (!maxGaspriceInWei || typeof maxGaspriceInWei !== 'string' || !maxGaspriceInWei.match(/[0-9]+/)) {
        throw Error('estimateAmount: maxGaspriceInWei is no decimal number string: ' + maxGaspriceInWei);
      }

      let maximumTotalInWei = new BN();

      // sum up all destination amounts
      for (let i = 0; i < list.length; i++) {
        maximumTotalInWei = maximumTotalInWei.add(new BN(list[i].amountInWei));
      }

      // calculate maxGasPrice * maxEstimatedGasLimit
      let gasLimitBn = new BN(gasLimitHexString.substring(2), 16);
      let maxGasPriceBn = new BN(maxGaspriceInWei, 10);

      // sum it all up:
      maximumTotalInWei = maximumTotalInWei.add(gasLimitBn.mul(maxGasPriceBn));
      return numberToHex(maximumTotalInWei);
    },

    connectMetamask() {
      if (typeof window.ethereum === 'undefined') {
        this.messages = 'Metamask seems not to be installed';
        return;
      }
      this.eth = window.ethereum;

      this.messages = 'Connecting to Metamask...';
      let self = this;
      this.eth.on('accountsChanged', (accounts) => {
        // Handle the new accounts, or lack thereof.
        // "accounts" will always be an array, but it can be empty.
        self.onAccountChanged(accounts);
      });
      this.eth.on('chainChanged', () => {
        window.location.reload();
      });


      function successCallback(result) {
        console.log('Connected to MetaMask:');
        console.log(result);
        try {
          if (result && result.length > 0 && typeof result[0] === 'string' && self.assertAddressWithPrefix(result[0])) {
            self.senderAccount = result[0];
            self.messages = 'Connected to Metamask :)';
            self.metamaskConnected = true;
          } else {
            self.messages = 'Connection to Metamask seemed to fail. :('
            self.metamaskConnected = false;
          }
        } catch (e) {
          self.messages = 'Error when connecting to MetaMask: ' + e;
          self.metamaskConnected = false;
        }
      }

      function failureCallback(error) {
        self.messages = 'Sorry, could not connect to Metamask. (' + error.message + ')';
        self.metamaskConnected = false;
        console.error('Connecting to Metamask failed with error:');
        console.error(error);
      }

      window.ethereum.request({method: 'eth_requestAccounts'}).then(successCallback, failureCallback)
    },

    assertAddressWithPrefix(address) {
      if (!address) {
        this.messages = 'address is undefined';
        throw new Error('address is undefined');
      }
      if (typeof address !== 'string') {
        this.messages = 'address must be a string';
        throw new Error('address must be a string');
      }
      if (address.length !== 42) {
        this.messages = address + ' has not a length of 42 chars';
        throw new Error(address + ' has not a length of 42 chars');
      }
      if (!address.startsWith('0x')) {
        this.messages = address + ' does not start with "0x"';
        throw new Error(address + ' does not start with "0x"');
      }
      if (!isAddress(address)) {
        this.messages = address + ' is not a valid address';
        throw new Error(address + ' is not a valid address');
      }
      return address;
    },

    async isContract(address) {
      this.assertAddressWithPrefix(address);
      let response = await this.eth.request({
        method: 'eth_getCode',
        params: [address, 'latest']
      });
      return response !== '0x';
    },

    async hasBalance(address) {
      this.assertAddressWithPrefix(address);
      let response = await this.eth.request({
        method: 'eth_getBalance',
        params: [address, 'latest']
      });
      return parseInt(response) > '0';
    },

    async hadSomeTransactions(address) {
      this.assertAddressWithPrefix(address);
      let response = await this.eth.request({
        method: 'eth_getTransactionCount',
        params: [address, 'latest']
      });
      return parseInt(response) > '0';
    },

    async checkAddressesForContracts() {
      if (!this.metamaskConnected || !this.eth.isConnected()) {
        this.messages = 'Metamask not connected!';
        return;
      }

      this.messages = 'Starting state check of destination addresses...';

      // reset error list
      this.destinationOnChainErrors = {};
      this.destinationsCheckedOnChain = false;

      for (let i = 0; i < this.destinationList.length; i++) {
        let address = this.destinationList[i].address;

        if (await this.isContract(address)) {
          this.destinationOnChainErrors[address] = {
            'ok': false,
            'msg': 'Is a contract'
          };
          continue;
        }
        if (!await this.hasBalance(address) && !await this.hadSomeTransactions(address)) {
          this.destinationOnChainErrors[address] = {
            'ok': false,
            'msg': 'Is unused'
          };
          continue;
        }
        this.destinationOnChainErrors[address] = {
          'ok': true,
          'msg': undefined
        };
      }
      this.destinationsCheckedOnChain = true;
      this.messages = 'State check finished.';
    },

    getChainNameById(chainId) {
      if (parseInt(chainId) === 0x1) {
        return 'Ethereum Mainnet'
      } else if (parseInt(chainId) === 0x3) {
        return 'Ethereum Ropsten Test Network'
      } else if (parseInt(chainId) === 0x4) {
        return 'Ethereum Rinkeby Test Network'
      } else if (parseInt(chainId) === 0x5) {
        return 'Ethereum Goerli Test Network'
      } else if (parseInt(chainId) === 0x42) {
        return 'Ethereum Kovan Test Network'
      } else {
        return 'Some other network with chainId: "' + chainId + '"';
      }
    },

    onAccountChanged(result) {
      try {
        console.log('Metamask Event onAccountChanged: ' + result);
        if (result && result.length > 0 && typeof result[0] === 'string' && this.assertAddressWithPrefix(result[0])) {
          this.senderAccount = result[0];
          this.messages = 'MetaMask Account changed: ' + result[0];
          this.metamaskConnected = true;
          console.log('Sender address changed to: ' + result[0]);
        } else {
          this.metamaskConnected = false;
          this.messages = 'Changing account in Metamask seemed to fail :('
          console.error('Address changed, something wrong..');
        }
      } catch (e) {
        this.metamaskConnected = false;
        this.messages = 'Error when changing accounts in MetaMask: ' + e;
        console.error('Address changed, caught error.. ' + e);
      }
    },

    parseDestinationsCsv(csv) {
      let result = [];
      let duplicateCheck = {};
      if (!csv || typeof csv !== 'string' || csv === '') {
        return result;
      }
      let allLines = csv.split('\n');
      for (let i = 0; i < allLines.length; i++) {
        let line = allLines[i];
        let trimmed = line.trim();
        if (trimmed.length === 0) {
          continue;
        }
        let parsedLine = this.parseSingleRow(trimmed);
        if (!parsedLine.error) {
          if (duplicateCheck[parsedLine.address]) {
            parsedLine = {
              'rawLine': trimmed,
              'error': 'Address duplicate. To save costs condense duplicated targets into one.'
            };
          } else {
            duplicateCheck[parsedLine.address] = true;
          }
        }
        result.push(parsedLine);
      }
      return result;
    },

    parseSingleRow(trimmedNonEmptyLine) {
      const regex = new RegExp("(^0x[0-9a-fA-F]{40});([0-9]{0,4}\\.?[0-9]{0,18})$");
      if (!regex.test(trimmedNonEmptyLine)) {
        return {
          'rawLine': trimmedNonEmptyLine,
          'error': 'Line does not look like valid "address;amount" pattern'
        };
      }
      let parsedLine = regex.exec(trimmedNonEmptyLine);
      if (!parsedLine || parsedLine.length < 3) {
        return {
          'rawLine': trimmedNonEmptyLine,
          'error': 'Line does not look like valid "address;amount" pattern'
        }
      } else {
        let amountInWei;
        let address;
        try {
          amountInWei = this.parseAndCheckMaxAmount(parsedLine[2]);
        } catch (error) {
          return {
            'rawLine': trimmedNonEmptyLine,
            'error': 'Invalid amount: ' + error
          }
        }

        try {
          address = this.assertAddressWithPrefix(parsedLine[1]);
        } catch (error) {
          return {
            'rawLine': trimmedNonEmptyLine,
            'error': 'Invalid address: ' + error
          }
        }

        return {
          'rawLine': trimmedNonEmptyLine,
          'address': address,
          'amountInWei': amountInWei,
          'error': undefined
        }
      }
    },

    parseAndCheckMaxAmount(amountInEth) {
      if (typeof amountInEth !== 'string') {
        throw new Error('expected amount as string');
      }
      let amountInWei = toWei(amountInEth, 'ether');
      let amountBn = new BN(amountInWei, 10);
      if (amountBn.cmp(new BN(0)) === 0) {
        this.messages = 'amount is zero'
        throw new Error('amount is zero');
      }
      let maxAmount = new BN('ffffffffffffffffff', 16);
      if (maxAmount.cmp(amountBn) <= 0) {
        this.messages = 'amount per address is higher than supported (max 4722.36 ETH for now, as only 9 byte field)'
        throw new Error('amount per address is higher than supported (max 4722.36 ETH for now)');
      }
      return amountInWei;
    }

  }

}
</script>

<style scoped>

table {
  margin: 0 auto;
}

td {
  padding-left: 20px;
  padding-right: 20px;
}

button {
  padding-top: 0.3em;
  padding-bottom: 0.3em;
}

/* mainnet style */
.eth_0x1 {
  background: #d7f3cd;
  color: #237903;
  font-weight: bold;
}

/* testnet style */
.eth_0x3 {
  background: #faf5d3;
  color: #897511;
  font-weight: bold;
}

.errorRow {
  background: #ffc4c4;
}

.okRow {
  background: #d7f3cd;
}

.uncheckedRow {
  background: #dddddd;
}

.error {
  padding: 10px;
  background: #ffc4c4;
  color: #ff0000;
  font-family: sans-serif;
  font-size: large;
}

.refundAddress {
  background: #faf5d3;
  border-width: 4px;
  padding-top: 5px;
  padding-bottom: 5px;
  font-weight: bold;
  font-size: large;
}

.txid {
  background: #b5e7a3;
  border-color: #237903;
  border-width: 4px;
  padding: 50px;
  font-weight: bold;
  font-size: x-large;
}

/* textarea */
#dests {
  width: 60em;
  height: 15em;
  border: 1px solid #cccccc;
}

.details {
  color: #aaaaaa;
  font-size: smaller;
}


</style>

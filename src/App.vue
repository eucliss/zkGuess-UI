<template>
  <div id="app" v-if="!mainLoading">
    <div class="main-box">
      <div style="display: flex; flex-direction: row;">
        <div style="padding: 20px; width: 50%;">
          <h1> zkGuess</h1>
          <div>
          The guessing game where you can win big or lose it all ...
          </div>
          <br>
          <div>
            The prize pool currently is
          </div>
          <h1>{{ prizepool }} ETH</h1>
        </div>
        <div style="padding: 20px; padding-top: 80px; width: 50%;">
          
          <div>
            Here's how this works - theres a secret number stored in the smart contract. 
            It costs 0.01 ETH to guess the number, if you do it successfully, you win all the money from the contract.
            If you fail to guess the correct number, you lose your 0.01 ETH guessing fee.
          </div>
        </div>


      </div>
    
      <div class="guess-div">

        <div>
            </div>
            <div class="balance" v-if="selectedToken">
              <p>
                GuessToken loaded, good luck!
              </p>
            </div>
            <div class="guess-input">
              <input
                v-model="newGuess"
                :disabled="!selectedToken || txStatus != 0"
                placeholder="Pick a number!"
              />

              <button
                class="guess-button"
                :disabled="!selectedToken || txStatus != 0 || retreivingFee"
                v-on:click="guess"
              >
                <span v-if="selectedToken && !txStatus">Guess!</span>
                <span v-else-if="txStatus == 1">Sending tx...</span>
                <span v-else-if="txStatus == 2"
                  >Waiting until tx is committed...</span
                >
                <span v-else-if="txStatus == 3">Updating the page...</span>
              </button>
        </div>

    </div>
    </div>
      <div v-if="result" class="result">
      <div v-if="winner">
        <img src="../assets/WINNER.png" alt="winner" />
      </div>
      <div v-else-if="!winner">
        <img src="../assets/LOSER.png" alt="loser" />
      </div>
  </div>
  </div>
  <div id="app" v-else>
    <div class="start-screen">
      <h1>Welcome to zkGuess!</h1>
      <button v-on:click="connectMetamask">Connect Wallet</button>
    </div>
  </div>
</template>

<script>
import { Contract, Web3Provider, Provider } from "zksync-web3";
import { ethers } from "ethers";

// eslint-disable-next-line
const GUESS_TOKEN_ADDRESS = "0x9097b8a7B29E81dd668dC6CC7377F8C51Bc52453";
const ZK_GUESS_ADDRESS = "0x695248aEfebE2A5E9f1E3AEB80cA95F0FAF62BDD";
// eslint-disable-next-line
const ZK_ABI = require("./zkGuess.json");
const GUESS_ABI = require("./guessToken.json");

const ETH_L1_ADDRESS = "0x0000000000000000000000000000000000000000";
console.log(ETH_L1_ADDRESS)
const allowedTokens = require("./eth.json");

export default {
  name: "App",
  data() {
    return {
      newGuess: 0,
      prizepool: 0,
      tokens: allowedTokens,
      selectedToken: null,
      selectedTokenAddress: "",
      mainLoading: true,
      provider: null,
      signer: null,
      contract: null,
      token: null,
      canSubmit: true,
      result:false,
      winner:false,
      // 0 stands for no status, i.e no tx has been sent
      // 1 stands for tx is beeing submitted to the operator
      // 2 stands for tx awaiting commit
      // 3 stands for updating the balance and greeting on the page
      txStatus: 0,
      retreivingFee: false,
      retreivingBalance: false,

      currentBalance: "",
      currentFee: "",
    };
  },
  methods: {
    initializeProviderAndSigner() {
        this.provider = new Provider('https://zksync2-testnet.zksync.dev');
        // Note that we still need to get the Metamask signer
        this.signer = (new Web3Provider(window.ethereum)).getSigner();
        this.contract = new Contract(
            ZK_GUESS_ADDRESS,
            ZK_ABI,
            this.signer
        );
        this.token = new Contract(
            GUESS_TOKEN_ADDRESS,
            GUESS_ABI,
            this.signer
        );
    },

    async getPrizepool() {
      // Get the balance of the GREETER contract
        const balance = await this.provider.getBalance(ZK_GUESS_ADDRESS);
        const balanceInEth = ethers.utils.formatEther(balance);
        return balanceInEth;
    },

    async getBalance() {
        // Getting the balance for the signer in the selected token
        const balanceInUnits = await this.signer.getBalance(this.selectedToken.l2Address);
        // To display the number of tokens in the human-readable format, we need to format them,
        // e.g. if balanceInUnits returns 500000000000000000 wei of ETH, we want to display 0.5 ETH the user
        return ethers.utils.formatUnits(balanceInUnits, this.selectedToken.decimals);
    },
    async guess() {
        this.txStatus = 1;
        try {
            this.result = false;
            this.winner = false;

            const overrides = {
                // value: ethers.utils.parseEther("0.01"),
                gasLimit: 1000000
            };

            console.log(this.newGuess)
            const txHandle = await this.contract.guess(this.newGuess, overrides);

            // Wait until the transaction is committed
            await txHandle.wait();

            this.txStatus = 3;

        } catch (e) {
            alert(JSON.stringify(e));
        }

        this.txStatus = 0;

        // Need the logic for setting winner / etc.
        this.result = true;
        this.winner = true;
    },

    updateFee() {
      this.retreivingFee = true;
      this.getFee()
        .then((fee) => {
          this.currentFee = fee;
        })
        .catch((e) => console.log(e))
        .finally(() => {
          this.retreivingFee = false;
        });
    },
    updateBalance() {
      this.retreivingBalance = true;
      this.getBalance()
        .then((balance) => {
          this.currentBalance = balance;
        })
        .catch((e) => console.log(e))
        .finally(() => {
          this.retreivingBalance = false;
        });
    },
    setToken() {
      const l1Token = this.tokens.filter(
        (t) => t.name == "Ether"
      )[0];
      this.provider
        .l2TokenAddress(l1Token.address)
        .then((l2Address) => {
          this.selectedToken = {
            l1Address: l1Token.address,
            l2Address: l2Address,
            decimals: l1Token.decimals,
            symbol: l1Token.symbol,
          };
        })
        .catch((e) => console.log(e))
    },
    loadMainScreen() {
      this.initializeProviderAndSigner();

      if (!this.provider || !this.signer) {
        alert("Ensure you're Metamask is configured and connected to zkSync L2!");
        return;
      }

      this.getPrizepool().then((prize) => {
        this.prizepool = prize;
      });
      this.mainLoading = false;
      this.setToken();
    },
    connectMetamask() {
      window.ethereum
        .request({ method: "eth_requestAccounts" })
        .then(() => {
          if (+window.ethereum.networkVersion == 280) {
            this.loadMainScreen();
          } else {
            alert("Please switch network to zkSync!");
          }
        })
        .catch((e) => console.log(e));
    },
  },
};
</script>

<style>
#app {
  font-family: Space Mono;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 30px;
}

#app ul {
  display: inline-block;
}

.guess-div {
  text-align: center;
  padding-bottom: 20px;
}

.colgroups {
  display: inline-block;
  width: 300px;
  margin-left: 50px;
}

.table-left {
  margin-left: 50px;
  text-align: left;
  display: inline-block;
  width: 100%;
}

.table-right {
  margin-right: 50px;
  margin-left: 50px;
  text-align: left;
  display: inline-block;
  width: 100%;
}

.main-box {
  text-align: left;
  width: 80%;

  margin: auto;
  margin-top: 40px;
}

.guess-input {
  margin-top: 20px;
}

.guess-button {
  margin-left: 20px;
}

.start-screen {
  margin-top: 100px;
}

.balance {
  margin-top: 10px;
}

.refresh-button {
  margin-left: 15px;
}
</style>

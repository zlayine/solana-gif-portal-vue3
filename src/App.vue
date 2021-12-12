<template>
  <div class="App">
    <div :class="{ container: !address, 'authed-container': address }">
      <div class="header-container">
        <p class="header">🖼 GIF Portal</p>
        <p class="sub-text">View your GIF collection in the metaverse ✨</p>
        <button
          class="cta-button connect-wallet-button"
          v-if="!address"
          @click="connectWallet"
        >
          Connect to Wallet
        </button>
        <div class="connected-container" v-else-if="address && gifList == null">
          <button
            class="cta-button submit-gif-button"
            @click="createGifAccount"
          >
            Do One-Time Initialization For GIF Program Account
          </button>
        </div>
        <div class="connected-container" v-else>
          <form @submit="sendGif">
            <input v-model="input" type="text" placeholder="Enter gif link!" />
            <button type="submit" class="cta-button submit-gif-button">
              Submit
            </button>
          </form>
          <div class="gif-grid">
            <div class="gif-item" :key="gif" v-for="gif in gifList">
              <img :src="gif.gifLink" :alt="gif" />
            </div>
          </div>
        </div>
      </div>
      <div class="footer-container">
        <twitter-logo class="twitter-logo" />
        <a
          class="footer-text"
          :href="TWITTER_LINK"
          target="_blank"
          rel="noreferrer"
        >
          built by @zouln96 with _buildspace
        </a>
      </div>
    </div>
  </div>
</template>

<script>
import TwitterLogo from "./components/twitter-logo.vue";
import { onMounted, ref, watch } from "vue";
import idl from "./idl.json";
import kp from "./keypair.json";
import { Connection, PublicKey, clusterApiUrl } from "@solana/web3.js";
import { Program, Provider, web3 } from "@project-serum/anchor";

export default {
  setup() {
    const TWITTER_LINK = "https://twitter.com/zouln96";
    const address = ref(null);
    const input = ref("");
    const gifList = ref(null);

    const { SystemProgram } = web3;

    // retrieve the keypair for the account that will hold the GIF data.
    const arr = Object.values(kp._keypair.secretKey);
    const secret = new Uint8Array(arr);
    const baseAccount = web3.Keypair.fromSecretKey(secret);

    // Get our program's id from the IDL file.
    const programID = new PublicKey(idl.metadata.address);

    // Set our network to devnet.
    const network = clusterApiUrl("devnet");

    // Controls how we want to acknowledge when a transaction is "done".
    const opts = {
      preflightCommitment: "processed",
    };

    const TEST_GIFS = [
      "https://i.giphy.com/media/eIG0HfouRQJQr1wBzz/giphy.webp",
      "https://media3.giphy.com/media/L71a8LW2UrKwPaWNYM/giphy.gif?cid=ecf05e47rr9qizx2msjucl1xyvuu47d7kf25tqt2lvo024uo&rid=giphy.gif&ct=g",
      "https://media4.giphy.com/media/AeFmQjHMtEySooOc8K/giphy.gif?cid=ecf05e47qdzhdma2y3ugn32lkgi972z9mpfzocjj6z1ro4ec&rid=giphy.gif&ct=g",
      "https://i.giphy.com/media/PAqjdPkJLDsmBRSYUp/giphy.webp",
    ];

    const checkIfWalletIsConnected = async () => {
      try {
        const { solana } = window;
        if (solana) {
          if (solana.isPhantom) {
            console.log("Phantom wallet found!");

            const response = await solana.connect({ onlyIfTrusted: true });

            console.log(
              "Connected with Public Key:",
              response.publicKey.toString()
            );

            address.value = response.publicKey.toString();
          }
        } else {
          alert("Solana object not found! Get a Phantom Wallet 👻");
        }
      } catch (error) {
        console.error(error);
      }
    };

    const connectWallet = async () => {
      const { solana } = window;

      if (solana) {
        const response = await solana.connect();
        console.log(
          "Connected with Public Key:",
          response.publicKey.toString()
        );
        address.value = response.publicKey.toString();
      }
    };

    const sendGif = async (e) => {
      e.preventDefault();
      if (input.value.length < 0) {
        console.log("No gif link given!");
        return;
      }
      console.log("Gif link:", input.value);
      try {
        const provider = getProvider();
        const program = new Program(idl, programID.toString(), provider);

        await program.rpc.addGif(input.value, {
          accounts: {
            baseAccount: baseAccount.publicKey.toString(),
            user: provider.wallet.publicKey.toString(),
          },
        });
        console.log("GIF successfully sent to program", input.value);
        input.value = "";

        await getGifList();
      } catch (error) {
        console.log("Error sending GIF:", error);
      }
    };

    const getProvider = () => {
      const connection = new Connection(network, opts.preflightCommitment);
      const provider = new Provider(
        connection,
        window.solana,
        opts.preflightCommitment
      );
      return provider;
    };

    const getGifList = async () => {
      try {
        const provider = getProvider();
        const program = new Program(idl, programID.toString(), provider);
        const account = await program.account.baseAccount.fetch(
          baseAccount.publicKey.toString()
        );

        console.log("Got the account", account);
        gifList.value = account.gifList;
      } catch (error) {
        console.log("Error in getGifList: ", error);
        gifList.value = null;
      }
    };

    const createGifAccount = async () => {
      try {
        const provider = getProvider();
        const program = new Program(idl, programID.toString(), provider);
        console.log("ping");
        await program.rpc.startStuffOff({
          accounts: {
            baseAccount: baseAccount.publicKey.toString(),
            user: provider.wallet.publicKey.toString(),
            systemProgram: SystemProgram.programId,
          },
          signers: [baseAccount],
        });
        console.log(
          "Created a new BaseAccount w/ address:",
          baseAccount.publicKey.toString()
        );
        await getGifList();
      } catch (error) {
        console.log("Error creating BaseAccount account:", error);
      }
    };

    watch(address, async (newV, oldV) => {
      if (newV) await getGifList();
    });

    onMounted(async () => {
      // timeout for solana object to be included in window
      setTimeout(async () => {
        await checkIfWalletIsConnected();
      }, 100);
    });

    return {
      TWITTER_LINK,
      address,
      connectWallet,
      gifList,
      input,
      sendGif,
      createGifAccount,
    };
  },
  components: {
    TwitterLogo,
  },
};
</script>

<style lang="scss">
body {
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "Roboto", "Oxygen",
    "Ubuntu", "Cantarell", "Fira Sans", "Droid Sans", "Helvetica Neue",
    sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

code {
  font-family: source-code-pro, Menlo, Monaco, Consolas, "Courier New",
    monospace;
}
</style>

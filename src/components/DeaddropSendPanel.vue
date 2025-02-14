<template>
    <form class="action-box controls">
        <div class="input-section">
            <div class="tx-type-section">
                <span class="selected">Send</span>
                <span> / </span>
                <span class="unselected pointer" v-on:click="ToReceive">Receive</span>
            </div>
            <h1>Alias</h1>
            <input type="text" placeholder="..." required v-model="state.destination">
            <h1>Amount</h1>
            <img class="token pointer" v-on:click="SelectToken" :src="TokenImage" alt="">
            <input type="text" :placeholder="TokenDenom" required v-model="state.amount">
        </div>
        <div class="txbutton" v-if="!state.loading">
            <a @Click=ExecuteSend><TxSubmit text="Send" /></a>
        </div>
        <div class="spinner" v-else>
            <i class="c-inline-spinner" />
        </div>
        <div class="fee">Fee: 0.04%</div>
        <img class="return pointer" src="@/assets/BackArrow.svg" alt="Back" v-on:click="ReturnHome">
        <TokenPanel v-if="TokenSelect" v-on:ConfirmToken="UseToken" v-on:ReturnHome="CancelSelectToken"></TokenPanel>
    </form>
</template>



<script>
import { getSigningClient, countDecimals } from '../utils/keplrHelper'
import TxSubmit from './TxSubmit.vue'
import TokenPanel from './TokenPanel.vue'
import { useToast } from "vue-toastification";


export default {
    name: 'DeaddropSendPanel',
    components: {
        TxSubmit,
        TokenPanel
    },
    data() {
        return {
            state: {
                loading: false,
                amount: null,
                destination: null
            },
            TokenSelect: false,
            TokenDenom: "sSCRT",
            TokenAddress: this.$store.state.token_address, //secret1k0jntykt7e4g3y88ltc60czgjuqdy4c9e8fzek",
            TokenImage: "/tokenIcons/scrt.svg",
            TokenDecimals: 6,

        }
    },
    setup() {
      // Get toast interface
      const toast = useToast();

      // Make it available inside methods
      return { toast }
    },
    methods: {
        ReturnHome: function() {
            this.$emit('ReturnHome')
        },
        ToReceive: function() {
            this.$emit('ToReceive')
        },
        SelectToken: function() {
            this.TokenSelect = true
        },
        CancelSelectToken: function() {
            this.TokenSelect = false
        },
        UseToken: function(newDenom, newImg, newAddress, newDecimals) {
            this.TokenSelect = false
            this.TokenDenom = newDenom
            this.TokenImage = "/tokenIcons/" + newImg
            this.TokenAddress = newAddress
            this.TokenDecimals = newDecimals
        },
        ExecuteSend: async function() {
            try{
                //replace button with spinner
                this.state.loading=true;

                //ensure signing client is in glibal state
                if (!this.$store.getters.hasSigningClient){
                    this.$store.dispatch("setSigningClient", await getSigningClient("secret-4"));
                }

                //cancel if no destination
                console.log(this.state.destination);
                if (!this.state.destination || !this.state.destination.trim()){
                    this.toast.error(`Please enter a destination alias.`, {
                        timeout: 6000
                    })
                    
                    //show button again
                    this.state.loading=false;

                    return false;
                }

                //cancel if invalid number
                if (!this.state.amount || !this.state.amount.trim() || isNaN(this.state.amount.trim())){
                    this.toast.error(`Invalid Amount: "${this.state.amount}"`, {
                        timeout: 6000
                    })
                    
                    //show button again
                    this.state.loading=false;

                    return false;
                }

                //cancel if more than token's decimals
                if (countDecimals(this.state.amount.trim()) > this.TokenDecimals){
                    this.toast.error(`Amount "${this.state.amount.trim()}" has too many decimals. ${this.TokenDenom} only has ${this.TokenDecimals} decimal places.`, {
                        timeout: 6000
                    })
                    
                    //show button again
                    this.state.loading=false;

                    return false;
                }

                //set amount in uTOKEN
                let amount = this.state.amount.trim() * 10**(this.TokenDecimals)

                //message for the deaddrop contract
                const ddMsg = {
                    receive_tokens: {
                        recipient: this.state.destination.trim()
                    }
                }; 

                //send message for the token contract
                const sendMsg = {
                    send: {
                        amount: amount.toString(),
                        recipient: this.$store.state.deaddrop_address, //deaddrop contract
                        msg: Buffer.from(JSON.stringify(ddMsg)).toString('base64')
                    }
                }

                //"Sync" broadcast mode returns tx hash only (or error if it failed to enter the mempool)
                let response = await this.$store.state.secretJs.execute(this.TokenAddress, sendMsg);
                if (response.code){
                    this.toast.error(`Transaction Failed: ${response.raw_log}`, {
                        timeout: 8000
                    })

                    //show button again
                    this.state.loading=false;

                    //stop execution
                    return false;

                } else {
                    this.toast("Transaction Processing...", {
                        id: "tx-processing",
                        timeout: false,
                        closeButton: false
                    });
                }

                //poll tx's endpoint every 1000ms up to 5 times to check when tx is processed. Returns full tx object
                const data = await this.$store.state.secretJs.checkTx(response.transactionHash,1000,5)
                console.log(data);
                this.toast.dismiss("tx-processing");

                //show button again
                this.state.loading=false;

                //if error
                if (data.code){
                    this.toast.error(`Transaction Failed: ${data.raw_log}`, {
                        timeout: 8000
                    })
                } else {
                    this.toast.success("Transaction Succeeded!", {
                        timeout: 8000
                    });
                }
            } catch(e) {
                this.toast.error(`Unknown error occured: ${e}`, {
                    timeout: 8000
                })

                //show button again
                this.state.loading=false;
            }
        }
    }
    
}
</script>



<style scoped>
@import "../assets/spinner.css";

.spinner {
    position: fixed;
    left: 50%;
    bottom: 15%;
    transform: translate(-50%, -50%);
    margin: 0 auto;

}

.controls {
    width: 521px;
    height: 661px;
    top: 181px;
    
}


.tx-type-section {
    position: fixed;
    transform: translate(190%, -400%);

}

.token {
    width: 50px;  
    height: auto;  
    position: fixed;
    transform: translate(-125%, -20%);
}

.token:hover {
    filter: brightness(25%);
}



.input-section {
    padding-top: 20%;
}

.selected {
    color: #2c3e50;
}

.unselected:hover {
    filter: brightness(25%);
}



input {
    border-radius: 26px;
    width: 50%;
    padding: 5px 15px 7px;
    font-size: 14px;
    text-align: center;
}


.txbutton {
    position: fixed;
    left: 50%;
    bottom: 20px;
    transform: translate(-50%, -50%);
    margin: 0 auto;

}

.txbutton:hover {
    filter: brightness(25%);
}


.fee {
    position: fixed;
    left: 50%;
    bottom: 20px;
    transform: translate(-50%, -50%);
    margin: 0 auto;
    
    
    
}

.return {
    position: fixed;
    left: 10%;
    bottom: 20px;
    transform: translate(-50%, -50%);
    margin: 0 auto;
    
}

.return:hover {
    filter: brightness(25%);
}

.pointer {
    cursor: pointer;
}

</style>
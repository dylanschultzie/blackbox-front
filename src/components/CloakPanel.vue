<template>
    <form class="action-box controls">
        <div class="input-section">
            <h1>Address</h1>
            <input type="text" v-model="state.destination" placeholder="secret..." required>
            <h1>Amount</h1>
            <img class="token" src="/tokenIcons/scrt.svg" alt="">
            <input type="text" v-model="state.amount" placeholder="sSCRT" required>
            <div class="withdraw pointer"><a @click=ExecuteCancel>(Cancel Pending Transactions)</a></div>
        </div>
        <div class="txbutton" v-if="!state.loading">
            <a @click=ExecuteCloak><TxSubmit text="Send" /></a>
        </div>
        <div class="spinner" v-else>
            <i class="c-inline-spinner" />
        </div>
        <div class="fee">Fee: 1 sSCRT</div>
        <img class="return pointer" src="@/assets/BackArrow.svg" alt="Back" v-on:click="ReturnHome">
    </form>
</template>



<script>
import { getSigningClient, isValidAddress, countDecimals } from '../utils/keplrHelper'
import TxSubmit from './TxSubmit.vue'
import { useToast } from "vue-toastification";

export default {
    name: 'CloakPanel',
    components: {
        TxSubmit
    },
    data() {
        return {
            state: {
                loading: false,
                amount: "",
                destination: ""
            }
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
        ExecuteCloak: async function() {
            try{
                //replace button with spinner
                this.state.loading=true;

                //ensure signing client is in glibal state
                if (!this.$store.getters.hasSigningClient){
                    this.$store.dispatch("setSigningClient", await getSigningClient("secret-4"));
                }
                
                //cancel if recipient is not a valid address
                if (!this.state.destination.trim() || !isValidAddress(this.state.destination.trim())){
                    this.toast.error(`Invalid Recipient Address: ${this.state.destination.trim()}`, {
                        timeout: 6000
                    })
                                        
                    //show button again
                    this.state.loading=false;

                    return false;
                }

                //cancel if invalid number
                if (!this.state.amount.trim() || isNaN(this.state.amount.trim())){
                    this.toast.error(`Invalid Amount: ${this.state.amount.trim()}`, {
                        timeout: 6000
                    })
                                        
                    //show button again
                    this.state.loading=false;

                    return false;
                }

                //cancel if more than 6 decimals
                if (countDecimals(this.state.amount.trim()) > 6){
                    this.toast.error(`Amount "${this.state.amount.trim()}" has too many decimals. sSCRT only has 6 decimal places.`, {
                        timeout: 6000
                    })
                                        
                    //show button again
                    this.state.loading=false;

                    return false;
                }

                //set amount in uscrt
                let amount = this.state.amount.trim()*1000000

                //message for the cloak contract
                const cloakMsg = {
                    receive_seed : {
                        recipient: this.state.destination.trim()
                    }
                }; 

                //send message for the sSCRT contract
                const sendMsg = {
                    send: {
                        amount: amount.toString(),
                        recipient: this.$store.state.cloak_address, //cloak address
                        msg: Buffer.from(JSON.stringify(cloakMsg)).toString('base64')
                    }
                }

                //"Sync" broadcast mode returns tx hash only (or error if it failed to enter the mempool)
                let response = await this.$store.state.secretJs.execute(this.$store.state.token_address, sendMsg);
                if (response.code){
                    this.toast.error(`Transaction Failed: ${response.raw_log}`, {
                        timeout: 8000
                    })
                                        
                    //show button again
                    this.state.loading=false;

                    return false;

                } else {
                    this.toast("Transaction Processing...", {
                        id: "tx-processing",
                        timeout: false,
                        closeButton: false
                    });
                }

                //poll tx's endpoint every 1000ms up to 5 times to check when tx is processed. Returns full tx object
                let data = await this.$store.state.secretJs.checkTx(response.transactionHash,1000,5)
                console.log(data)
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

        },
        ExecuteCancel: async function() {
            try{
                //replace button with spinner
                this.state.loading=true;

                //ensure signing client is in glibal state
                if (!this.$store.getters.hasSigningClient){
                    this.$store.dispatch("setSigningClient", await getSigningClient("secret-4"));
                }

                //message for the cloak contract
                const cloakMsg = {
                    exit_pool : { }
                }; 

                //send message for the sSCRT contract
                /*const sendMsg = {
                    send: {
                        amount: amount.toString(),
                        recipient: this.$store.state.cloak_address,
                        msg: Buffer.from(JSON.stringify(cloakMsg)).toString('base64')
                    }
                }*/

                //"Sync" broadcast mode returns tx hash only (or error if it failed to enter the mempool)
                let response = await this.$store.state.secretJs.execute(this.$store.state.cloak_address, cloakMsg);
                if (response.code){
                    this.toast.error(`Transaction Failed: ${response.raw_log}`, {
                        timeout: 8000
                    })
                                        
                    //show button again
                    this.state.loading=false;

                    return false;

                } else {
                    this.toast("Transaction Processing...", {
                        id: "tx-processing",
                        timeout: false,
                        closeButton: false
                    });
                }

                //poll tx's endpoint every 1000ms up to 5 times to check when tx is processed. Returns full tx object
                let data = await this.$store.state.secretJs.checkTx(response.transactionHash,1000,5)
                console.log(data)
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

.withdraw {
    font-size: 90%;
    margin-top: 30px;

}
.withdraw:hover {
    filter: brightness(25%);
}

.controls {
    width: 521px;
    height: 661px;
    top: 181px;
    
}

.input-section {
    padding-top: 20%;
}


input {
    border-radius: 26px;
    width: 50%;
    padding: 5px 15px 7px;
    font-size: 14px;
    text-align: center;
}


.token {
    width: 50px;  
    height: auto;  
    position: fixed;
    transform: translate(-125%, -20%);
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
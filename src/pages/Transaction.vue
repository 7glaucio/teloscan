<script>
import { mapGetters, mapActions } from 'vuex';
import DateField from 'components/DateField';
import BlockField from 'components/BlockField';
import AddressField from 'components/AddressField';
import LogsViewer from 'components/Transaction/LogsViewer';
import InternalTxns from 'components/Transaction/InternalTxns';
import CopyButton from 'components/CopyButton';
import MethodField from 'components/MethodField';
import ERCTransferList from 'components/Transaction/ERCTransferList';
import ParameterList from 'components/Transaction/ParameterList';
import JsonViewer from 'vue-json-viewer';
import { BigNumber } from 'ethers';
import { WEI_PRECISION, formatWei, parseErrorMessage, getRouteWatcherForTabs } from 'src/lib/utils';
import { TRANSFER_SIGNATURES } from 'src/lib/abi/signature/transfer_signatures';

const tabs = {
    general: '#general',
    details: '#details',
    eventLog: '#eventlog',
    internal: '#internal',
};

// TODO: The get_transactions API doesn't format the internal transactions properly, need to fix that before we try to decode them
export default {
    name: 'TransactionPage',
    components: {
        LogsViewer,
        InternalTxns,
        AddressField,
        BlockField,
        CopyButton,
        DateField,
        MethodField,
        JsonViewer,
        ERCTransferList,
        ParameterList,
    },
    data() {
        return {
            hash: this.$route.params.hash,
            blockData: null,
            trxNotFound: false,
            errorMessage: null,
            trx: null,
            erc20Transfers: [],
            erc721Transfers: [],
            erc1155Transfers: [],
            params: [],
            tab: '#general',
            isContract: false,
            contract: null,
            parsedTransaction: null,
            methodTrx: null,
            showAge: true,
            showWei: false,
        };
    },
    computed: {
        ...mapGetters('evm', ['tlosPrice']),
    },
    watch: {
        '$route.params': {
            handler(newValue) {
                const { hash } = newValue
                if (this.hash === hash) {
                    return;
                }

                this.resetTransaction();
                this.hash = hash;
                this.loadTransaction();
            },
            immediate: true,
        },
        $route: getRouteWatcherForTabs('transaction', tabs, tabs.general),
    },
    async mounted() {
        await this.loadTransaction();
    },
    async created() {
        this.fetchTlosPrice();
    },
    methods: {
        ...mapActions('evm', ['fetchTlosPrice']),
        formatWei,
        resetTransaction() {
            this.blockData = null;
            this.trx = null;
            this.tab = '#general';
            this.isContract = false;
            this.contract = null;
            this.parsedTransaction = null;
            this.methodTrx = null;
            this.erc20Transfers = [];
            this.erc721Transfers = [];
            this.erc1155Transfers = [];
            this.params = [];
        },
        async loadTransaction() {
            const trxResponse = await this.$evmEndpoint.get(
                `/v2/evm/get_transactions?hash=${this.hash}`,
            );
            if (trxResponse.data.transactions.length === 0) {
                this.trxNotFound = true;
                return;
            }
            this.trx = trxResponse.data.transactions[0];
            this.trx.value = BigNumber.from(this.trx.value.toLocaleString('fullwide', {useGrouping:false}));
            await this.loadContract();
            await this.loadTransfers();
            this.setErrorMessage();
        },
        async loadTransfers()
        {
            this.transfers = [];
            for (const log of this.trx.logs) {
                // ERC20, ERC721 & ERC1155 transfers (ERC721 & ERC20 have same first topic but ERC20 has 4 topics for transfers, ERC20 has 3 log topics, ERC1155 has a different first topic)
                let sig = log.topics[0].substr(0, 10);
                if (TRANSFER_SIGNATURES.includes(sig)) {
                    let type = this.$contractManager.getTokenTypeFromLog(log);
                    let contract = await this.$contractManager.getContract(log.address, type);
                    if (typeof contract.token !== 'undefined' && contract.token !== null) {
                        let token = {'symbol': contract.token.symbol, 'address': log.address, name: contract.token.name, 'decimals': contract.token.decimals}
                        if (contract.token.type === 'erc721') {
                            let tokenId = BigNumber.from(log.topics[3]).toString();
                            if (contract.token.extensions?.metadata) {
                                try {
                                    token = await this.$contractManager.loadTokenMetadata(log.address, contract.token, tokenId);
                                } catch (e) {
                                    console.error(`Could not retreive metadata for ${contract.address}: ${e.message}`);
                                }
                            }
                            this.erc721Transfers.push({
                                'tokenId': tokenId,
                                'to': '0x' + log.topics[2].substr(log.topics[2].length - 40, 40),
                                'from': '0x' + log.topics[1].substr(log.topics[1].length - 40, 40),
                                'token': token,
                            });
                        } else if (contract.token.type === 'erc1155') {
                            let tokenId = BigNumber.from(log.data.substr(0, 66)).toString();
                            if (contract.token.extensions?.metadata) {
                                try {
                                    token = await this.$contractManager.loadTokenMetadata(log.address, contract.token, tokenId);
                                } catch (e) {
                                    console.error(`Could not retreive metadata for ${contract.address}: ${e.message}`);
                                }
                            }
                            this.erc1155Transfers.push({
                                'tokenId': tokenId,
                                'amount': BigNumber.from(log.data).toString(),
                                'to': '0x' + log.topics[3].substr(log.topics[3].length - 40, 40),
                                'from': '0x' + log.topics[2].substr(log.topics[2].length - 40, 40),
                                'token': token,
                            });
                        } else {
                            this.erc20Transfers.push({
                                'value': log.data,
                                'wei': BigNumber.from(log.data).toString(),
                                'to': '0x' + log.topics[2].substr(log.topics[2].length - 40, 40),
                                'from': '0x' + log.topics[1].substr(log.topics[1].length - 40, 40),
                                'token': token,
                            });
                        }
                    }
                }
            }
        },
        async loadContract() {
            if (this.trx.input_data === '0x') return;

            const contract = await this.$contractManager.getContract(this.trx.to);
            if (!contract) return;

            this.contract = contract;
            this.parsedTransaction = await this.contract.parseTransaction(this.trx.input_data);
            this.params = this.getFunctionParams();
            this.methodTrx = Object.assign(
                { parsedTransaction: this.parsedTransaction },
                this.trx,
            );
            this.isContract = true;
        },
        setErrorMessage() {
            if (this.trx.status !== 0)
                return;

            this.errorMessage = parseErrorMessage(this.trx.output);
        },
        getFunctionName() {
            if (this.parsedTransaction) return this.parsedTransaction.name;
        },
        getFunctionParams() {
            if (!this.parsedTransaction) return [];
            let args = [];
            this.parsedTransaction.functionFragment.inputs.forEach((input, i) => {
                args.push({name: input.name, type: input.type, arrayChildren: (input.arrayChildren !== null) ? input.arrayChildren.type : false, value:  this.parsedTransaction.args[i]})
            })
            return args;
        },
        getGasFee() {
            return formatWei(BigNumber.from(this.trx.charged_gas_price).mul(this.trx.gasused).toLocaleString('fullwide', {useGrouping:false}), WEI_PRECISION, 5);
        },
        getGasChargedGWEI() {
            return formatWei(this.trx.charged_gas_price, 9, 2);
        },
    },
};
</script>
<style scoped lang="sass">
    @media screen and (max-width: 650px)
        #function-parameters
            width: 100%
            flex: auto
            margin-top: 20px

        #transaction-page
            .col-3
                width: 100%
            .col-9
                width: 100%

    @media only screen and (max-width: 900px)
        #function-parameters
            .row
                .col-4
                    width: 100%
                    padding-left: 15px
                .col-8
                    padding-bottom: 10px
                    padding-left: 30px
                    width: 100%
</style>
<template lang='pug'>
.pageContainer
  .row
    .col-12.q-px-md
      .text-h4.text-primary.q-mb-lg.title.q-pt-xl
        | Transaction Details
      .text-h6.q-mb-lg.text-white( v-if="trxNotFound" )
        | Not found: {{ hash }}
  .row.tableWrapper
    .col-12.q-py-lg
      .content-container( v-if="trx"  :key="erc20Transfers.length + isContract" )
        q-tabs.text-white.topRounded(
          v-model="tab"
          dense
          active-color="secondary"
          align="justify"
          narrow-indicator
          :class="$q.dark.isActive ? 'q-dark' : 'q-light'"
        )
          q-route-tab.topLeftRounded(
            name="general"
            :to="{ hash: '#general' }"
            exact
            replace
            label="General"
          )
          q-route-tab(
            name="details"
            :to="{ hash: '#details' }"
            exact
            replace
            label="Details"
          )
          q-route-tab(
            name="logs"
            :to="{ hash: '#eventlog' }"
            exact
            replace
            label="Logs"
          )
          q-route-tab.topRightRounded(
            name="internal"
            :to="{ hash: '#internal' }"
            exact
            replace
            label="Internal Txns"
          )
        q-tab-panels.column.shadow-2(
          v-model="tab"
          animated
          keep-alive
        )
          q-tab-panel( name="general" id="transaction-page" )
            br
            br
            div(class="fit row wrap justify-start items-start content-start")
                div(class="col-3")
                  strong.wrapStrong Transaction Hash:&nbsp;
                div(class="col-9")
                  span {{ hash }}
                  copy-button(:text="hash")
            br
            div(class="fit row wrap justify-start items-start content-start")
                div(class="col-3")
                  strong {{ `Block Number: ` }}
                div(class="col-9")
                  block-field( :block="trx.block" )
            br
            div( @click="showAge = !showAge" class="fit row wrap justify-start items-start content-start date")
                div(class="col-3"  )
                  strong {{ `Date: ` }}
                div.u-flex--left
                  q-icon(class="far fa-clock q-mr-xs")
                  date-field( :epoch="trx.epoch" :show-age="showAge" )
                  q-tooltip Click to change date format
            br
            div(class="fit row wrap justify-start items-start content-start")
              div(class="col-3")
                strong {{ `Status: ` }}
              div(class="col-9" style="padding: 5px 0px;")
                span(v-if="trx.status == 1", class="positive")
                  q-icon(name='check')
                  span {{ "Success" }}
                span(v-else, class="negative")
                  q-icon(name='warning')
                  span {{ "Failure" }}
            br
            div( v-if="errorMessage", class="fit row wrap justify-start items-start content-start" )
              div(class="col-3")
                strong {{ `Error message: ` }}
              div(class="col-9")
                span.text-negative {{ errorMessage }}
            br( v-if="errorMessage" )
            div(class="fit row wrap justify-start items-start content-start")
              div(class="col-3")
                strong {{ `From: ` }}
              div(class="col-9 word-break")
                address-field(:address="trx.from" :truncate="0" :highlight="erc20Transfers.length + erc721Transfers.length > 1" copy)
            br
            div(class="fit row wrap justify-start items-start content-start")
              div(class="col-3")
                strong {{ `To: ` }}
              div(class="col-9 word-break")
                address-field( :address="trx.to" :is-contract-trx="!!contract"  :truncate="0" copy)
            br
            div( v-if="isContract", class="fit row wrap justify-start items-start content-start" )
              div(class="col-3")
                strong {{ `Contract function: ` }}
              div(class="col-9")
                MethodField( :contract="contract" :trx="methodTrx" shortenSignature )
            br(v-if="isContract")
            div( v-if="isContract && params.length > 0" class="fit row wrap justify-start items-start content-start")
              div(class="col-3")
                strong {{ `Function parameters: ` }}
              div(class="col" id="function-parameters")
                ParameterList(:params="params" :contract="contract" :trxFrom="trx.from")
            br( v-if="isContract && params.length > 0" )
            div( v-if="trx.createdaddr", class="fit row wrap justify-start items-start content-start" )
              div(class="col-3")
                strong {{ `Deployed contract: ` }}
              div(class="col-9 word-break")
                AddressField( :address="trx.createdaddr" )
            br( v-if="trx.createdaddr" )
            div(class="fit row wrap justify-start items-start content-start")
              div(class="col-3")
                strong {{ `Value: ` }}
              div(class="col-9 clickable" @click="showWei = !showWei")
                div(v-if="showWei")
                    span {{ trx.value }}
                span(v-else)
                    span {{ formatWei(trx.value, 18) }} TLOS
                    q-tooltip Click to show in wei
            br
            ERCTransferList( v-if="erc20Transfers.length > 0" type="ERC20" :trxFrom="trx.from" :contract="contract" :transfers="erc20Transfers")
            ERCTransferList( v-if="erc721Transfers.length > 0" type="ERC721" :trxFrom="trx.from" :contract="contract" :transfers="erc721Transfers")
            ERCTransferList( v-if="erc1155Transfers.length > 0" type="ERC1155" :trxFrom="trx.from" :contract="contract" :transfers="erc1155Transfers")
            div(class="fit row wrap justify-start items-start content-start")
              div(class="col-3")
                strong {{ `Gas Price Charged: ` }}
              span {{ getGasChargedGWEI() }} GWEI
            br
            div(class="fit row wrap justify-start items-start content-start")
              div(class="col-3")
                strong {{ `Gas Fee: ` }}
              span {{ getGasFee() }} TLOS
                small.q-pl-sm (~ ${{ (getGasFee() * tlosPrice).toFixed(5) }})
            br
            div(class="fit row wrap justify-start items-start content-start")
              div(class="col-3")
                strong {{ `Gas Used: ` }}
              div(class="col-9") {{ trx.gasused }}
            br
            div(class="fit row wrap justify-start items-start content-start")
              div(class="col-3")
                strong {{ `Gas Limit: ` }}
              div(class="col-9") {{ trx.gas_limit }}
            br
            div(class="fit row wrap justify-start items-start content-start")
              div(class="col-3")
                strong {{ `Nonce: ` }}
              div(class="col-9") {{ trx.nonce }}
          q-tab-panel( name="details" )
            div
              div(class="col-3")
                strong {{ `Input: ` }}
              div(class="col-9") {{ trx.input_data }}
            br
            div
              div(class="col-3")
                strong {{ `Output: ` }}
              div(class="col-9") {{ trx.output }}
          q-tab-panel( name="logs" )
            .jsonViewer
              logs-viewer(:logs="trx.logs" :contract="contract" )
          q-tab-panel( name="internal" )
            InternalTxns( :itxs="trx.itxs" :contract="contract" )
</template>

<style lang="sass" scoped>
.shadow-2
    box-shadow: none !important

span
    word-break: break-word

.col-9 .positive .q-icon, .col-9 .negative .q-icon
    margin-top: -5px
    margin-right: 5px

.date .col-9 > div
    display: inline-block

.col-9 .positive, .col-9 .negative
    border: 1px solid
    border-radius: 5px
    padding: 5px 10px

.col-9 .jv-container .jv-code
    padding: 0

.col-9
    word-break: break-word

.q-tabs__content
    margin-bottom: -1px

@media only screen and (max-width: 550px)
    .q-tab
        padding: 0px 5px
    .q-tab__label
        font-size: 11px
</style>

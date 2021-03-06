<template>
    <div id="txu-latest-exec">
        <h3 class="title">Latest Execution</h3>
        <div class="deals-tabs">
            <span class="tab-item cur-hand" :class="{' active':isMarketMode}" v-on:click="isMarketMode = true ">Market</span>
            <span class="tab-item cur-hand" :class="{' active':!isMarketMode}" v-on:click="isMarketMode =  false">Mine</span>
        </div>
        <div class="title-table-wrap">
            <table aria-busy="false" aria-colcount="3" class="table b-table">
                <thead>
                <tr>
                    <th aria-colindex="1" class="">Time</th>
                    <th aria-colindex="2" class="">Price({{fiatCoin}})</th>
                    <th aria-colindex="3" class="">Amount({{fiatCoin}})</th>
                </tr>
                </thead>
                <tbody></tbody>
            </table>
        </div>
        <div class="free-table-wrap" :class="{'hide-elem':!isMarketMode}">
            <table aria-busy="false" aria-colcount="3" class="table b-table">
                <thead>
                <tr>
                    <th aria-colindex="1">Time</th>
                    <th aria-colindex="2">Price({{fiatCoin}})</th>
                    <th aria-colindex="3">Amount({{fiatCoin}})</th>
                </tr>
                </thead>
                <tbody>
                <TradeItem v-for="trade in trades"
                           :key="trade.time"
                           :trade_time="trade.time"
                           :trade_side = "trade.type"
                           :trade_price="trade.price"
                           :trade_amount="trade.amount"/>
                </tbody>
            </table>
        </div>
        <div class="auth-table-wrap" :class="{'hide-elem':isMarketMode}">
            <table aria-busy="false" aria-colcount="3" class="table b-table">
                <thead class="">
                <tr>
                    <th aria-colindex="1">Time</th>
                    <th aria-colindex="2">Price({{fiatCoin}})</th>
                    <th aria-colindex="3">Amount({{fiatCoin}})</th>
                </tr>
                </thead>
                <tbody>
                <TradeItem v-for="mineTrade in mineTrades"
                           :key="mineTrade.time"
                           :trade_time="mineTrade.time"
                           :trade_side = "mineTrade.side"
                           :trade_price="mineTrade.price"
                           :trade_amount="mineTrade.amount"/>
                </tbody>
            </table>
        </div>
    </div>
</template>
<script>
    import TradeItem from '../sub/trade-item/index'
    import axios from "axios";

    export default {
        name: 'TradeDeal',
        components: {
            TradeItem
        },
        props: {},
        data() {
            return {
                isLogin: false,
                trades: [],
                mineTrades: [],
                isMarketMode: true,
                marketName: this.$store.state.market.currentMarket,
                baseCoin: this.$store.state.market.baseCoin,
                fiatCoin: this.$store.state.market.currentMarket.substr(0, this.$store.state.market.currentMarket.lastIndexOf(this.$store.state.market.baseCoin)),
            }
        },
        created() {
            this.$eventBus.$on('update-mine',(data)=>{
                console.log('updating mine trade history');
                if(data.user == userID && this.marketName == data['market']){
                    data.time = data.ctime;
                    data.type = (data.side == 1)?'sell':'buy';
                    this.mineTrades.push(data);
                }
            });
        },
        computed: {},
        mounted() {
            this.checkIsLogin();
            this.getInitMineHistory();
            this.$store.watch((state) => state.market.currentMarket, this.marketSelect, {deep: true});
            this.$store.watch((state) => state.socket.message, this.msgHandler, {deep: true});
            this.$store.watch((state) => state.socket.isConnected, this.sendDealQuery, {deep: true});
        },
        methods: {
            sendDealSubscribe() {
                if (this.$store.state.socket.isConnected) {
                    this.$socket.send(JSON.stringify({
                        "method": "deals.subscribe",
                        "params": this.$gP.MarketPairs,
                        "id": this.$gP.DEAL_SUBSCRIBE
                    }));
                }
            },
            sendDealQuery() {
                const tradesCfg = this.$store.state.trades;
                if (this.$store.state.socket.isConnected) {
                    this.$socket.send(JSON.stringify({
                        "method": "deals.query",
                        "params": [this.marketName, tradesCfg.deals_limit, tradesCfg.last_id],
                        "id": this.$gP.DEAL_QUERY
                    }));
                }
            },
            marketSelect() {
                const marketInfo = this.$store.state.market;
                this.marketName = marketInfo.currentMarket;
                this.baseCoin = marketInfo.baseCoin;
                this.fiatCoin = this.marketName.substr(0, this.marketName.lastIndexOf(this.baseCoin));
                this.trades = [];
                this.mineTrades = [];
                this.getInitMineHistory();
                this.sendDealQuery();
            },
            msgHandler() {
                let resInfo = JSON.parse(this.$store.state.socket.message);
                if (resInfo.id == this.$gP.DEAL_QUERY) {
                    if(resInfo.error == null) {
                        this.updateTradeDeals(resInfo.result);
                        //If client get initial data... , then send deals subscribe request..
                        this.unSubscribeDeals();
                        this.sendDealSubscribe();
                    } else {
                        console.log('Deals API not working.');
                        console.error(resInfo.error.message);
                    }
                } else if (resInfo.method == 'deals.update') {
                    if (resInfo.params[0] == this.marketName) {
                        this.updateTradeDeals(resInfo.params[1]);
                    }
                }
            },
            unSubscribeDeals() {
                if (this.$store.state.socket.isConnected) {
                    this.$socket.send(JSON.stringify({
                        "method": "deals.unsubscribe",
                        "params": [],
                        "id": this.$gP.DEAL_UNSUBSCRIBE
                    }));
                }
            },
            updateTradeDeals(trades) {
                if (this.trades.length > 0) {
                    for (let i = trades.length - 1; i > -1; i--) {
                        let curTradeLastId = this.trades[0];
                        if (trades[i].id > curTradeLastId.id) {
                            this.trades.unshift(trades[i]);
                        }
                    }
                } else {
                    console.log("initializing trade history...");
                    this.trades = trades;
                }
            },
            checkIsLogin() {
                if (webToken != "" && webSource != "" && userID != null) {
                    this.isLogin = true;
                }
            },
            getInitMineHistory() {
                if (!this.isLogin) return;
                let params = {
                    cmd: 'deals',
                    market: this.marketName,
                    user_id: userID,
                    offset: 0,
                    limit: 100
                };
                axios.post(apiURL, params).then(result => {
                    let resp = result.data;
                    if (resp.error == null && resp.result.records.length > 0) {
                        this.mineTrades = resp.result.records;
                    } else {
                        console.error(resp.error);
                        this.mineTrades = [];
                    }
                }, error => {
                    console.log(error);
                    this.mineTrades = [];
                });
            }
        }
    }
</script>
<style src="./tradedeal.css" scoped></style>


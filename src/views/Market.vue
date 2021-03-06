<style scoped>
    .market {
        width: 100%;
        height: 100%;
        padding: 30px;
    }

    .spin-container {
        display: inline-block;
        width: 100%;
        height: 100%;
        position: relative;
    }

    .loaded-container {
        width: 100%;
        height: 100%;
    }

    .data-market-box {
        height: 100%;
        display: flex;
        flex-direction: column;
        padding-left: 30px;
    }

    .data-market-list {
        flex: 1;
    }

    .product-card {
        position: relative;
        height: 220px;
        margin: 8px;
        cursor: pointer;
        text-align: center;
    }

    .product-info .icon {
        width: 60px;
        height: 60px;
        margin: 10px auto 20px;
    }

    .product-info .icon img {
        width: 100%;
        height: 100%;
    }

    .product-info .title {
        overflow: hidden;
        white-space: nowrap;
        text-overflow: ellipsis;
        margin-bottom: 10px;
        color: #464c5b;
    }

    .product-info .desc {
        overflow: hidden;
        white-space: nowrap;
        text-overflow: ellipsis;
        margin-bottom: 20px;
        color: #9ea7b4;
        font-size: 12px;
    }

    .product-info .price, .product-info .statics {
        color: #38adff;
    }

    .product-info .price span, .product-info .statics span {
        font-size: 16px;
    }

    .page-box {
        height: 25px;
        text-align: center;
        margin-top: 30px;
    }

    .fade-enter-active, .fade-leave-active {
        transition: opacity .5s
    }

    .fade-enter, .fade-leave-active {
        opacity: 0
    }
</style>
<template>
    <div class="market">
        <div class="spin-container" v-show="!loaded">
            <Spin fix></Spin>
        </div>
        <Row class="loaded-container" v-show="loaded">
            <i-col span="5">
                <Menu ref="dataMarketMenu" :active-name="data_market_type + '-' + active_category" width="auto"
                      :open-names="['1','2']" @on-select="changeCategory">
                    <Submenu name="1">
                        <template slot="title">
                            <Icon type="ios-keypad"></Icon>
                            自由市场
                        </template>
                        <MenuItem :name="'1-' + category.id" v-for="(category,index) in free_data_categorys" :key="index">
                            {{category.category_name}}
                        </MenuItem>
                    </Submenu>
                    <Submenu name="2">
                        <template slot="title">
                            <Icon type="ios-people"></Icon>
                            联盟市场
                        </template>
                        <MenuItem :name="'2-' + category.id" v-for="(category,index) in league_data_categorys" :key="index">
                            {{category.category_name}}
                        </MenuItem>
                    </Submenu>
                </Menu>
            </i-col>
            <i-col span="19" class="data-market-box">
                <Row class="data-market-list">
                    <transition-group name="fade">
                        <i-col span="6" v-for="(product,index) in product_list" :key="index">
                            <router-link :to="'/market/product/?id=' + product.id" v-if="product.product_name">
                                <Card class="product-card">
                                    <div class="product-info">
                                        <div class="icon"><img :src="product.icon"></div>
                                        <h3 class="title">{{product.product_name}}</h3>
                                        <p class="desc">{{product.brief_desc}}</p>
                                        <p class="price">价格: <span>{{formatterPrice(product.price)}}</span> GXC</p>
                                    </div>
                                </Card>
                            </router-link>
                            <router-link :to="'/market/league/?id=' + product.id" v-if="product.league_name">
                                <Card class="product-card">
                                    <div class="product-info">
                                        <div class="icon"><img :src="product.icon"></div>
                                        <h3 class="title">{{product.league_name}}</h3>
                                        <p class="desc">{{product.brief_desc}}</p>
                                        <p class="statics">数据产品: <span>{{product.data_products.length}}</span></p>
                                    </div>
                                </Card>
                            </router-link>
                        </i-col>
                    </transition-group>
                </Row>
                <div class="page-box">
                    <Page :current="current" :total="total" :page-size="page_size" show-total show-elevator size="small"
                          @on-change="changePage"></Page>
                </div>
            </i-col>
        </Row>
    </div>
</template>
<script>
    import Handler from '../libs/handler';

    export default {
        data () {
            return {
                data_market_type: Number(this.$route.query.tid) ? Number(this.$route.query.tid) : 1,
                current: Number(this.$route.query.page) ? Number(this.$route.query.page) : 1,
                active_category: this.$route.query.cid,
                free_data_categorys: [],
                league_data_categorys: [],
                loaded: false,
                page_size: 12,
                product_list: [],
                total: 0
            };
        },
        created () {
            if ((this.data_market_type !== 1) && (this.data_market_type !== 2)) {
                this.$router.push('/404');
            }
            let self = this;
            this.$http.all([
                this.$http.get('/api/fetch_data_market_categories/1'),
                this.$http.get('/api/fetch_data_market_categories/2')
            ]).then(this.$http.spread(function (res1, res2) {
                // 获取自由市场分类
                if (res1.data && res1.data.length > 0) {
                    self.getFreeData(res1.data);
                }
                // 获取联盟市场分类
                if (res2.data && res2.data.length > 0) {
                    self.getLeague(res2.data);
                }
                self.$nextTick(function () {
                    self.$refs.dataMarketMenu.updateActiveName();
                });
            })).catch((err) => {
                Handler.error(err);
                if (err.response.data.data.code === 10) {
                    this.$router.push('/404');
                }
            });
        },
        methods: {
            getFreeData (categories) {
                this.free_data_categorys = categories;
                if (this.data_market_type === 1) {
                    if (!this.$route.query.cid) {
                        this.active_category = this.free_data_categorys[0].id;
                    }
                    this.$http.get('/api/fetch_free_data_products/' + this.active_category + '/' + (this.current - 1) + '/' + this.page_size).then((res) => {
                        if (res.data) {
                            this.product_list = res.data.list;
                            this.total = res.data.total;
                            this.loaded = true;
                        }
                    }).catch((err) => {
                        Handler.error(err);
                        if (err.response.data.data.code === 10) {
                            this.$router.push('/404');
                        }
                    });
                }
            },
            getLeague (categories) {
                this.league_data_categorys = categories;
                if (this.data_market_type === 2) {
                    if (!this.$route.query.cid) {
                        this.active_category = this.league_data_categorys[0].id;
                    }
                    this.$http.get('/api/fetch_league_list/' + this.active_category + '/' + (this.current - 1) + '/' + this.page_size).then((res) => {
                        if (res.data) {
                            this.product_list = res.data.list;
                            this.total = res.data.total;
                            this.loaded = true;
                        }
                    }).catch((err) => {
                        Handler.error(err);
                        if (err.response.data.data.code === 10) {
                            this.$router.push('/404');
                        }
                    });
                }
            },
            changeCategory (name) {
                let params = name.split('-');
                this.data_market_type = Number(params[0]);
                this.active_category = params[1];
                this.current = 1;
                if (this.data_market_type === 1) {
                    this.$router.push('/market/?tid=1&page=1&cid=' + this.active_category);
                }
                if (this.data_market_type === 2) {
                    this.$router.push('/market/?tid=2&page=1&cid=' + this.active_category);
                }
            },
            changePage (page) {
                this.current = page;
                if (this.data_market_type === 1) {
                    this.$router.push('/market/?tid=1&page=' + page + '&cid=' + this.active_category);
                }
                if (this.data_market_type === 2) {
                    this.$router.push('/market/?tid=2&page=' + page + '&cid=' + this.active_category);
                }
            },
            formatterPrice (price) {
                return price / 100000;
            }
        },
        watch: {
            $route (to) {
                this.data_market_type = Number(to.query.tid) ? Number(to.query.tid) : 1;
                this.current = Number(to.query.page) ? Number(to.query.page) : 1;
                this.active_category = to.query.cid ? to.query.cid : this.free_data_categorys[0].id || this.league_data_categorys[0].id;
                if (this.data_market_type === 1) {
                    this.$http.get('/api/fetch_free_data_products/' + this.active_category + '/' + (this.current - 1) + '/' + this.page_size).then((res) => {
                        if (res.data) {
                            this.product_list = [];
                            this.product_list = res.data.list;
                            this.total = res.data.total;
                        }
                    }).catch((err) => {
                        this.$Message.error('获取自由市场产品列表失败:' + Handler.error(err));
                    });
                }
                if (this.data_market_type === 2) {
                    this.$http.get('/api/fetch_league_list/' + this.active_category + '/' + (this.current - 1) + '/' + this.page_size).then((res) => {
                        if (res.data) {
                            this.product_list = [];
                            this.product_list = res.data.list;
                            this.total = res.data.total;
                        }
                    }).catch((err) => {
                        this.$Message.error('获取联盟市场产品列表失败:' + Handler.error(err));
                    });
                }
            }
        }
    };
</script>

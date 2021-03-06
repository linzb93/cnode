<template>
    <container :load="loadStatus" errorMsg="服务器故障，请稍后再试">
        <div class="wrapper">
            <div class="topic-tab">
                <a v-for="(tab, index) in tabs" :key="tab.id" :class="tab.active" @click="tabSwitch(tab.id, index)" href="javascript:;">{{tab.name}}</a>
            </div>
            <ul class="index-list" v-show="!showEmptyTips">
                <li class="clearfix" v-for="topic in topics" :key="topic.id">
                    <a class="author-avatar" :href="'#/user/' + topic.author">
                        <img :src="topic.avatar" :alt="topic.author">
                    </a>
                    <div class="topic-stats">
                        <span class="reply-count">{{topic.reply_count | shorten}}</span>
                        <span class="split">/</span>
                        <span class="visit-count">{{topic.visit_count | shorten}}</span>
                    </div>
                    <h3>
                        <a :href="'#/detail/' + topic.id">{{topic.title}}</a>
                    </h3>
                    <span class="topic-tag topic-top" v-if="topic.top">[置顶]</span>
                    <span class="topic-tag topic-good" v-if="topic.good">[精华]</span>
                    <div class="topic-time">
                        <p>发帖时间：{{topic.create_at | ago}}</p>
                        <p>最近回帖时间：{{topic.last_reply_at | ago}}</p>
                    </div>
                </li>
            </ul>
            <pagination :isLastPage="isLastPage" :initPage="page" @change="changePage" v-model="changeTab" v-if="!showEmptyTips" />
            <empty-tips v-else />
            <editor header="发布新帖" type="post" @post="postTopic"/>
        </div>
    </container>
</template>

<script>
    import { mapState } from 'vuex';
    import cloneDeep from 'lodash/cloneDeep';
    import { getTopicList, createNewTopic } from '@/service';
    import { shorten, ago } from '@/filters';
    import { tabs } from '@/store/constant';
    import Container from '@/views/layout/Container';
    import Pagination from '@/components/Pagination';
    import EmptyTips from '@/components/Empty';
    import Editor from '@/components/Editor';

    export default {
        name: 'Index',
        components: {
            Container,
            Pagination,
            EmptyTips,
            Editor
        },
        data() {
            return {
                loadStatus: 'loading',
                topics: [],
                showEmptyTips: false, //无内容返回时显示
                changeTab: false, //分页重置的标志
                isLastPage: false //是否是最后一页
            };
        },
        computed: {
            ...mapState(['token']),
            tabs() {
                var arr = cloneDeep(tabs);
                arr.unshift({
                    id: 'all',
                    name: '全部'
                }, {
                    id: 'good',
                    name: '精华'
                });
                return arr;
            },
            tabId() {
                return this.$store.state.index.tabId
            },
            page() {
                return this.$store.state.index.page;
            }
        },
        methods: {
            render(tab = '', page = 1) {
                getTopicList(tab, page)
                .then(res => {
                    if (res.data.data.length === 0) {
                        this.topics = [];
                        this.showEmptyTips = true;
                        return;
                    }
                    this.isLastPage = res.data.data.length < 20;
                    this.showEmptyTips = false;
                    this.topics = res.data.data.map(item => {
                        return {
                            author: item.author.loginname,
                            avatar: item.author.avatar_url,
                            id: item.id,
                            create_at: item.create_at,
                            good: item.good,
                            last_reply_at: item.last_reply_at,
                            reply_count: item.reply_count,
                            title: item.title,
                            top: item.top,
                            visit_count: item.visit_count
                        }
                    });
                    this.loadStatus = 'complete';
                })
                .catch(err => {
                    this.loadStatus = 'error';
                });
            },
            tabInit() {
                //标签初始化
                this.tabs.forEach((item, index) => {
                    this.$set(item, 'active', item.id === this.tabId ? 'on' : '');
                });
            },
            tabSwitch(id, tabIndex) {
                //标签切换
                this.tabs.forEach((item, index) => {
                    this.$set(item, 'active', index === tabIndex ? 'on' : '');
                });
                this.$store.commit('index/tabSwitch', id);
                this.$store.commit('index/changePage', 1);
                this.changeTab = !this.changeTab;
                this.render(id);
            },
            changePage(curPage) {
                this.$store.commit('changePage', curPage);
                this.render(this.tabId, curPage);
            },
            postTopic({title, tab, content}) {
                var ctx = this;
                createNewTopic({
                    accesstoken: ctx.token,
                    title,
                    tab,
                    content
                })
                .then(res => {
                    if (res.data.success === true) {
                        this.$message({
                            type: 'success',
                            message: '发布成功！'
                        });
                    }
                });
            }
        },
        created() {
            this.render(this.tabId, this.page);
            this.tabInit();
        }
    };
</script>

<style lang="scss">
    @import "../../style/assist/mixin";
    @import "../../style/assist/variable";    

    .topic-tab {
        background: #f5f5f5;
        padding: 10px;
        a {
            padding: 5px;
            color: $mainColor;
            margin: 0 10px;
            border-radius: 2px;
            &:hover {
                color: lighten($mainColor, 5%);
                text-decoration: none;
            }
            &.on {
                color: #fff;
                background: $mainColor;
            }
        }
    }
    .index-list {
        li {
            padding: 10px 30px;
            &:hover {
                background: #f5f5f5;
            }
        }
        .author-avatar {
            float: left;
            width: 30px;
            height: 30px;
            img {
                width: 100%;
                height: 100%;
            }
        }
        .topic-stats {
            float: left;
            width: 80px;
            text-align: center;
            margin-left: 14px;
        }
        .reply-count {
            color: #9e78c0;
        }
        .split {
            color: #666;
        }
        .visit-count {
            font-size: 12px;
            color: #999;
        }
        h3 {
            @include ellipse;
            float: left;
            vertical-align: top;
            font-size: 16px;
            max-width: 700px;
            margin-right: 10px;
            a {
                color: $titleColor;
            }
        }
        .topic-tag {
            float: left;
            margin: 0 3px;
        }
        .topic-top {
            color: $mainColor;
        }
        .topic-good {
            color: #ef4141;
        }
        .topic-time {
            float: right;
            width: 200px;
            text-align: right;
            color: $conColor;
            line-height: 2;
        }
    }
    .post-editor-container {
        border-top: $border;
        padding: 15px 30px;
        h3 {
            font-size: 18px;
            color: $titleColor;
            font-weight: bold;
        }
        .el-select,
        .el-button {
            margin-top: 10px;
        }
        .el-input {
            margin-bottom: 20px;
        }
    }
</style>

<template>
    <div class="page-search">
        <input type="text" placeholder="Search" @keydown="handleKeydown($event)" id="search" v-model="searchWord" ref="sInput" autofocus
            style="-webkit-app-region: drag">
        <div id="search-list">
            <compontent :is="getComponent(data)" v-for="(data,index) in searchResultList" :class="index==activeIndex?'active-li':''"
                :key="index" @clickHandle="handleSelect(index)" :data="data" :index="index"></compontent>
        </div>
    </div>
</template>

<script>
    const {
        shell,
        remote,
        ipcRenderer
    } = require('electron');
    import searchLi from '../components/searchLi.vue';
    import fileLi from '../components/fileLi.vue';
    import defaultLi from '../components/defaultLi.vue';
    import bookmarkLi from '../components/bookmarkLi.vue';
    const path = require("path");
    const fs = require("fs");
    const dialog = remote.dialog;
    import {
        getFeatureListByword,
        getWebSearchListByword,
        getQuickStartListByword,
        getBookmarkListByword
    } from '../util/getSearchResult'

    export default {
        name: 'search',
        created() {
            this.resetSize();
        },
        mounted() {
            ipcRenderer.on("focusSearchInput", () => {
                // $nextTick内获取的this.$refs.sInput可能还是为undefined,所以设置一个100ms延迟
                setTimeout(() => {
                    this.fixPointerPosition()
                }, 100);
            })
        },
        data() {
            return {
                searchWord: "",
                activeIndex: 0,
                searchResultList: [],
                action: ""
            }
        },
        watch: {
            searchResultList() {
                this.resetSize();
            },
            searchWord() {
                this.getListByKeywords();
            }
        },
        computed: {
            activeSearchItem() {
                return this.searchResultList[this.activeIndex];
            }
        },
        components: {
            searchLi,
            fileLi,
            defaultLi,
            bookmarkLi
        },
        methods: {
            resetSize() {
                this.activeIndex = 0;
                this.$nextTick(v => {
                    let height = $('.page-search').height();
                    if (this.searchResultList.length == 0) { // fix修改设置之后length=0
                        height = 47; // 一列的高度
                    }
                    ipcRenderer.send("resize", {
                        width: 700,
                        height
                    })
                })
            },
            open(link) {
                this.$electron.shell.openExternal(link)
            },
            handleKeydown(e) {
                let len = this.searchResultList.length;
                switch (e.keyCode) {
                    case 67: // alt+c 清空输入内容
                        this.searchWord = '';
                    case 38: // up
                        this.fixPointerPosition();
                        this.activeIndex = (this.activeIndex - 1 + len) % len;
                        this.adjustScrollTop();
                        break;

                    case 40: // down
                        this.fixPointerPosition();
                        this.activeIndex = (this.activeIndex + 1) % len;
                        this.adjustScrollTop();
                        break;

                    case 46: // delete
                        this.activeIndex = (this.activeIndex - 1 + len) % len;
                        break;

                    case 13: // enter                      
                        this.fixPointerPosition();
                        this.handleSelect();
                        break;
                }
            },
            getListByKeywords() {
                let word = this.searchWord.trim(),
                    list = [],
                    webList = [],
                    startList = [],
                    featureList = [];

                if (!word) {
                    this.searchResultList = getFeatureListByword(word);
                    return;
                }

                // web搜索列表
                webList = getWebSearchListByword(this.searchWord);
                if (webList.length > 0) {
                    this.activeIndex = 0;
                }
                //  快速启动列表
                startList = getQuickStartListByword(this.searchWord);
                //  功能列表
                featureList = getFeatureListByword(this.searchWord);
                //  书签列表
                getBookmarkListByword(this.searchWord).then(data => {
                    list = [].concat(list, webList, startList, featureList, data);
                    this.searchResultList = list;
                }, (err) => {
                    alert(err)
                })
            },
            adjustScrollTop() {
                let listHeight = $("#search-list").outerHeight(), // 列表显示高度
                    inputHeight = $("#search").outerHeight(),
                    activeLiHeight = $(".active-li").outerHeight();
                this.$nextTick(() => {
                    let scrollTop = $("#search-list").scrollTop(),
                        activeOffsetTop = $('.active-li').offset().top + scrollTop,
                        h1 = scrollTop + inputHeight - activeOffsetTop;
                    if (h1 > 0) {
                        $("#search-list").scrollTop(activeOffsetTop - inputHeight);
                    }
                    let heightNeedToShowall = activeOffsetTop + activeLiHeight,
                        h2 = heightNeedToShowall - listHeight - inputHeight - scrollTop;
                    if (h2 > 0) {
                        $("#search-list").scrollTop(heightNeedToShowall - listHeight - inputHeight);
                    }
                })
            },
            fixPointerPosition() {
                this.$nextTick(() => {
                    console.log(this.$refs);
                    if (this.$refs && this.$refs.sInput) {
                        this.$refs.sInput.blur();
                        setTimeout(() => {
                            this.$refs.sInput.focus();
                        }, 17);
                    }
                })
            },
            handleSelect(index) {
                if (index !== undefined) {
                    this.activeIndex = index;
                }
                let act = this.activeSearchItem || {};
                if (act.type == "feature") { // 打开新页面
                    if (act.router) {
                        ipcRenderer.send("openWin", {
                            router: act.router,
                            resizable: false,
                            frame: false
                        })
                    } else if (act.view == 'uploadBookmark') {
                        dialog.showOpenDialog({
                            properties: ['openFile']
                        }, function (files) {
                            let filepath = files[0];
                            if (!/.*\.html/.test(filepath)) {
                                alert("书签仅支持html格式")
                            } else {
                                fs.createReadStream(filepath).pipe(fs.createWriteStream(path.resolve(__static,
                                    './pages/bookmark.html')));
                                alert("导入成功")
                            }
                        })
                    }
                } else if (act.type == "file") { // 打开文件
                    let appProcess = shell.openItem(act.path, function (err) {
                        alert("无法执行" + act.path)
                    });
                } else if (act.type == "search") { // 打开浏览器搜索
                    let query = this.searchWord.replace(new RegExp("^" + act.keyword + "\\s*"), "")
                    let url = act.url + encodeURIComponent(query);
                    this.open(url)
                } else if (act.type == 'bookmark') { // 打开书签
                    this.open(act.url);
                }
            },
            getComponent(v) {
                let type = v.type;
                if (type == 'search') {
                    return 'searchLi';
                } else if (type == 'file') {
                    return 'fileLi';
                } else if (type == 'bookmark') {
                    return 'bookmarkLi';
                } else if (type == 'uploadBookmark') {
                    return 'uploadBookmark';
                } else {
                    return 'defaultLi';
                }
            }
        }
    }
</script>

<style scoped lang='scss'>
    ::-webkit-scrollbar-thumb {
        background-color: #545454;
    }

    .page-search {
        width: 100%;
        background: #292828;
        #search {
            border: none;
            outline: none;
            width: 100%;
            height: 47px;
            line-height: 47px;
            font-size: 22px;
            padding: 0 7px;
            color: #4ff2f8;
            background: rgb(45, 45, 45);
        }
        #search-list {
            max-height: 329px;
            overflow: auto;
        }
    }
</style>
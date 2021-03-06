/**
* @Description: 主界面编辑视图
* @Author: TankNee
* @Date: 2/23/2020 8:47 AM
**/
<template>
    <div class="main" id="box">
        <el-tree
                id="tree"
                class="file-tree"
                :data="fileTree"
                @node-click="handleFileClick"
                v-loading="isLoading"
                @node-expand="handleEx"
                @node-collapse="handleColl"
                @node-contextmenu="handleRightMouse"
                node-key="id"
                accordion
                ref="ft"
                :highlight-current="true"
        >
            <span slot-scope="{ node, data }">
                <i :class="getNodeIcon(node,data)"></i>
                <span style="padding-left: 4px;">{{node.label}}</span>
            </span>
        </el-tree>
        <mavon-editor
                id="editor"
                v-model="content"
                :preview="preview"
                ref=md
                class="markdown"
                style="box-shadow: none;border-left: 1px solid rgb(237,237,237);"
                :navigation="showNav"
                placeholder="Write your ideas"
                @save="handleSave"
                codeStyle="googlecode"
                v-loading="isLoading"
        />
        <div v-show="menuVisible">
            <ul id="menu">
                <li
                        v-for="item in menuItems"
                        class="menu-item"
                        :style="item === '删除' ? 'color: red':''"
                        @click="handleMenuClick(item)"
                        :data-type="item"
                >
                    {{ item }}
                </li>
            </ul>
        </div>
    </div>

</template>

<script>
    import Home from "./common/Home";
    import bus from "./common/Bus";
    import api from "../utils/api";
    import {mavonEditor} from 'mavon-editor';
    import 'mavon-editor/dist/css/index.css';

    const he = require('he');
    const html2markdown = require('../utils/lib/html2markdown/index');
    export default {
        name: 'IndexPage',
        components: {
            Home,
            mavonEditor
        },
        data() {
            return {
                value1: 30,
                showNav: false,
                content: '',
                preview: '',
                isLogin: false,
                isLoading: false,
                menuVisible: false,
                configs: {
                    spellChecker: false // 禁用拼写检查
                },
                fileTree: [],
                folderPathsArray: [],
                treeId: 1,
                resources: [],
                noteInfo: null,
                menuItems: ['新建', '移动', '复制', '删除'],
                selectedNodeData: {},
                selectedNode: {}
            };
        },
        methods: {
            handleClick(vm, event) {
                console.log(vm, event);
            },
            handleSubmenuShow(vm, placement) {
                console.log(vm, placement);
            },
            async handleFileClick(data, Node) {
                this.menuVisible = false;
                var node = data;
                console.log(Node);
                if (node.isLoaded && node.type.indexOf('folder') !== -1) {
                    return;
                }
                this.isLoading = true;
                node.isLoaded = true;
                console.log(node);
                let path = '';
                this.folderPathsArray.forEach(fpa => {
                    if (fpa[fpa.length - 1] === node.label) {
                        path = fpa[0];
                        for (let i = 1; i < fpa.length; i++) {
                            path = `${path}/${fpa[i]}`;
                        }
                    }
                });
                var userSettings = JSON.parse(localStorage.getItem('userSettings'));
                //如果点击的是文件夹，就加载这个文件夹的笔记
                if (node.type.indexOf('folder') !== -1) {
                    this.isLoading = true;
                    try {
                        const result = await api.KnowledgeBaseApi.getFolderNotes({
                            kbServer: userSettings.kbServer,
                            kbGuid: userSettings.kbGuid,
                            data: {
                                start: 0,
                                count: 100,
                                category: `/${path}/`,
                                orderBy: 'modified',
                                ascending: 'desc',
                                withAbstract: 'true',
                                clientType: 'web',
                                clientVersion: 4.0,
                                lang: 'zh-cn',
                            }
                        });
                        console.log(result);
                        result.forEach(note => {
                            const newChild = {
                                label: note.title,
                                type: 'file',
                                id: this.treeId++,
                                docGuid: note.docGuid,
                                kbGuid: note.kbGuid
                            };
                            this.$refs.ft.append(newChild, node.id);
                            console.log(note);
                        });
                    } catch(err) {
                        this.$message({
                            message: `获取笔记列表失败：${err.message}`,
                            type: 'error'
                        });
                    } finally {
                        this.isLoading = false;
                    }
                } else {
                    this.selectedNodeData = node;
                    bus.$emit('Note Opened', node.label);
                    try {
                        const result = await api.KnowledgeBaseApi.getNoteContent({
                            kbGuid: node.kbGuid,
                            docGuid: node.docGuid,
                            data: {
                                downloadInfo: 1,
                                downloadData: 1,
                                clientType: 'web',
                                clientVersion: 4.0,
                                lang: 'zh-cn',
                            }
                        });
                        console.log(result);
                        const re1 = new RegExp("<.+?>", "g");//匹配html标签的正则表达式，"g"是搜索匹配多个符合的内容
                        const patten = /<body[^>]*>([\s\S]*)<\/body>/g;
                        const patten2 = /<span\sdata-wiz-span="data-wiz-span"\sstyle="font-size:\s10\.5pt;">/g;
                        const patten3 = /<span\sdata-wiz-span="data-wiz-span"\sstyle="font-size: 0\.875rem;">/g;
                        const patten4 = /\s<img\ssrc="data.*>/g;
                        var body;
                        if (result.html.indexOf(`<body`) === -1) {
                            body = `<html><head></head><body>${result.html}</body></html>`;
                        } else {
                            body = patten.exec(result.html)[0];//提取body里的内容
                        }
                        this.resources = result.resources;
                        this.noteInfo = result.info;
                        let text = html2markdown(body, {
                            imgBaseUrl: `${api.KnowledgeBaseApi.getBaseUrl()}/ks/note/view/${node.kbGuid}/${node.docGuid}/`,
                            resources: result.resources
                        });
                        text = text.replace(/&nbsp;/g, '\u0020'); // 将空格统一为转化成半角空格
                        text = he.decode(text); // 处理其他实体字符
                        text = text.replace(patten2, '');
                        text = text.replace(patten3, '');
                        text = text.replace(patten4, '- [ ] ');
                        console.log(text.indexOf('data-wiz-span'));
                        this.content = text;
                    } catch (err) {
                        console.log(err);
                        this.$message({
                            type: 'error',
                            message: `无法获取笔记内容：${err.message}`
                        })
                    } finally {
                        this.isLoading = false;
                    }
                }
            },
            // data-节点的自定义对象，Node节点对象
            handleEx(data, Node) {
                if (data.type.indexOf('-opened') === -1) {
                    data.type = data.type + '-opened';
                }
            },
            handleColl(data, Node) {
                console.log(data);
                data.type = data.type.replace('-opened', '');
            },
            async handleRightMouse(e, data, Node, element) {
                this.menuVisible = false;
                if (data.type.indexOf('file') !== -1) {
                    this.menuItems = ['重命名', '移动', '复制', '删除'];
                    try {
                        const result = await api.KnowledgeBaseApi.getNoteInfo({
                            kbGuid: data.kbGuid,
                            docGuid: data.docGuid
                        });
                        this.noteInfo = result.info;
                    } catch (err) {
                        this.$message({
                            type: 'error',
                            message: `无法获取笔记信息：${err.message}`
                        })
                    }
                } else {
                    this.menuItems = ['新建文件夹', '新建笔记', '重命名文件夹', '移动', '复制', '删除'];
                }
                this.menuVisible = true;
                let menu = document.querySelector("#menu");
                menu.style.left = e.clientX - 70 + "px";
                menu.style.top = e.clientY - 35 + "px";
                this.selectedNodeData = data;
                this.selectedNode = Node;
            },
            /**
             * 处理文件夹路径
             * @param folderPaths 文件夹路径数组
             */
            dealFileFolder(folderPaths) {
                var maxLevel = 0;
                // 首先将文件夹的名字解析成数组
                var folderPathsArray = [];
                folderPaths.forEach(fp => {
                    var temp = fp.split('/').splice(1);
                    folderPathsArray.push(temp.splice(0, temp.length - 1));
                });
                console.log(folderPathsArray);
                this.folderPathsArray = folderPathsArray;

                // 再处理一级目录
                folderPathsArray.forEach((fpa, index) => {
                    console.log(fpa);
                    if (fpa.length > maxLevel) {
                        maxLevel = fpa.length;
                    }
                    if (fpa.length === 1) {
                        this.fileTree.push({
                            type: 'folder',
                            isLoaded: false,
                            id: this.treeId,
                            label: fpa[0]
                        });
                        this.treeId = this.treeId + 1;
                    }
                });
                // 接着处理其他级别的目录
                folderPathsArray.forEach((fpa, index) => {
                    this.fileTree.forEach(node => {
                        if (fpa.length !== 1 && node.label === fpa[0]) {
                            node = this.insertFolderNode(node, fpa);
                        }
                    });
                });

            },
            /**
             * 插入文件夹节点
             * @param topNode 顶级节点
             * @param folderPathsArray 文件夹名数组--包含父文件夹
             */
            insertFolderNode(topNode, folderPathsArray) {
                if (folderPathsArray.length === 1) {
                    return topNode;
                }
                // 如果没有子节点集，那么直接创建
                if (!topNode.children) {
                    topNode['children'] = [];
                    let child = {
                        type: 'folder',
                        isLoaded: false,
                        id: this.treeId,
                        label: folderPathsArray[1]
                    };
                    this.treeId = this.treeId + 1;
                    // 创建指定的目录树
                    topNode.children.push(this.insertFolderNode(child, folderPathsArray.slice(1)));
                    return topNode;
                } else if (folderPathsArray.length === 2) {
                    let child = {
                        type: 'folder',
                        isLoaded: false,
                        id: this.treeId,
                        label: folderPathsArray[1]
                    };
                    this.treeId = this.treeId + 1;
                    return topNode.children.push(child);
                }
                topNode.children.forEach(child => {
                    if (child.label === folderPathsArray[1]) {
                        this.insertFolderNode(child, folderPathsArray.slice(1));
                    }
                });
            },
            /**
             * 获取节点的图标
             * @param node
             * @param data
             * @returns {string}
             */
            getNodeIcon(node, data) {
                var icon;
                console.log(data.type);
                if (data.type === 'folder') {
                    icon = 'el-icon-folder';
                } else if (data.type === 'folder-opened') {
                    icon = 'el-icon-folder-opened';
                } else if (data.type === 'file') {
                    icon = 'el-icon-notebook-1';
                } else if (data.type === 'file-opened') {
                    icon = 'el-icon-notebook-2';
                }
                console.log(icon);
                return icon;
            },
            //响应保存操作
            async handleSave(markdown, html) {
                console.log(this.noteInfo);
                try {
                    await api.KnowledgeBaseApi.updateNote({
                        kbGuid: this.noteInfo.kbGuid,
                        docGuid: this.noteInfo.docGuid,
                        data: {
                            category: this.noteInfo.category,
                            kbGuid: this.noteInfo.kbGuid,
                            docGuid: this.noteInfo.docGuid,
                            html: `<html><head></head><body>${markdown}</body></html>`,
                            resources: this.resources,
                            title: this.noteInfo.title
                        }
                    });

                    this.$message({
                        message: `保存成功`,
                        type: 'success'
                    });
                } catch (err) {
                    this.$message({
                        message: `保存失败：${err.message}`,
                        type: 'error'
                    });
                }
            },
            async handleMenuClick(item) {
                let originNode = this.selectedNode;
                let node = this.selectedNodeData;
                let settings = JSON.parse(localStorage.getItem('userSettings'));
                let path = '';
                this.folderPathsArray.forEach(fpa => {
                    if (fpa[fpa.length - 1] === node.label) {
                        path = fpa[0];
                        for (let i = 1; i < fpa.length; i++) {
                            path = `${path}/${fpa[i]}`;
                        }
                    }
                });
                console.log(Math.floor(Date.now() / 1000).toFixed(0));
                switch (item) {
                    case '新建文件夹':
                    {
                        try {
                            const result = await api.KnowledgeBaseApi.createCategory({
                                kbGuid: settings.kbGuid,
                                data: {
                                    parent: `/${path}/`,
                                    child: '新文件夹',
                                    pos: Math.floor(Date.now() / 1000).toFixed(0)
                                }
                            });
                            this.content = 'Hello World';
                            this.isLoading = false;
                            const newChild = {
                                label: result.folder.name,
                                type: 'folder',
                                id: this.treeId++,
                                isLoaded: false
                            };
                            this.$refs.ft.append(newChild, node.id);
                        } catch (err) {
                            this.$message({
                                message: `新建文件夹失败：${err.message}`,
                                type: 'error'
                            });
                        }
                        break;
                    }
                    case '新建笔记':
                    {
                        try {
                            const result = await api.KnowledgeBaseApi.createNote({
                                kbGuid: settings.kbGuid,
                                data: {
                                    kbGuid: settings.kbGuid,
                                    html: `<html><head></head><body><p>${'Hello World'}</p></body></html>`,
                                    category: `/${path}/`,
                                    owner: settings.userId,
                                    title: '无标题.md',
                                    params: null,
                                    appInfo: null,
                                    tags: ""
                                }
                            });
                            this.content = 'Hello World';
                            this.isLoading = false;
                            const newChild = {
                                label: result.title,
                                type: 'file',
                                id: this.treeId++,
                                docGuid: result.docGuid,
                                kbGuid: result.kbGuid
                            };
                            this.$refs.ft.append(newChild, node.id);
                        } catch (err) {
                            this.$message({
                                message: `新建笔记失败：${err.message}`,
                                type: 'error'
                            });
                        }
                        break;
                    }
                    case '重命名文件夹':
                        this.$prompt('请输入文件夹名', '提示', {
                            confirmButtonText: '确定',
                            cancelButtonText: '取消',
                            inputValue:this.selectedNodeData.label
                        }).then(({value}) => {
                            bus.$emit('Rename Folder', `/${path}/`, value);
                            this.selectedNodeData.label = value;
                        }).catch(() => {
                        });
                        break;
                    case '重命名':
                        this.$prompt('请输入文件名', '提示', {
                            confirmButtonText: '确定',
                            cancelButtonText: '取消',
                            inputValue:this.selectedNodeData.label
                        }).then(({value}) => {
                            bus.$emit('Rename File', value);
                            this.selectedNodeData.label = value;
                        }).catch(() => {
                        });
                        break;
                    case '移动':
                        break;
                    case '复制':
                        break;
                    case '删除':
                    {
                        if (node.type.indexOf('file') !== -1) {
                            try {
                                await api.KnowledgeBaseApi.deleteNote({
                                    kbGuid: settings.kbGuid,
                                    docGuid: node.docGuid
                                });
                                this.$refs.ft.remove(node.id);
                            } catch (err) {
                                this.$message({
                                    message: `删除文件失败：${err.message}`,
                                    type: 'error'
                                });
                            }
                        } else {
                            try {
                                await api.KnowledgeBaseApi.deleteCategory({
                                    kbGuid: settings.kbGuid,
                                    data: {
                                        category: `/${path}/`,
                                        clientType: 'web',
                                        clientVersion: '4.0',
                                        lang: 'zh-cn',
                                    }
                                });
                                this.$refs.ft.remove(node.id);
                            } catch (err) {
                                this.$message({
                                    message: `删除文件夹失败：${err.message}`,
                                    type: 'error'
                                });
                            }
                        }
                        break;
                    }
                    default:
                        break;
                }
            },
            dragControllerDiv: function () {
            }
        },
        mounted() {
            this.dragControllerDiv();
            bus.$on('Close ContextMenu', () => {
                this.menuVisible = false;
            });
        },
        created() {
            //登陆成功
            bus.$on('Login Successfully', async () => {
                this.isLoading = true;
                this.isLogin = true;
                var settings = JSON.parse(localStorage.getItem('userSettings'));
                console.log(settings);
                try {
                    const result = await api.KnowledgeBaseApi.getFolders({
                        kbServer: settings.kbServer,
                        kbGuid: settings.kbGuid,
                    });
                    this.dealFileFolder(result);
                    this.isLoading = false;
                } catch (err) {
                    this.$message({
                        message: `获取文件夹列表失败：${err.message}`,
                        type: 'warning'
                    });
                }
            });
            // 退出成功
            bus.$on('Logout Successfully', () => {
                this.fileTree = [];
                this.content = '';
                this.isLogin = false;
            });
            // 文件夹重命名
            bus.$on('Rename Folder', async (oldName, newName) => {
                console.log("Rename Folder", oldName, "=>", newName);
                const settings = JSON.parse(localStorage.getItem('userSettings'));
                try {
                    await api.KnowledgeBaseApi.renameCategory({
                        kbGuid: settings.kbGuid,
                        data: {
                            from: oldName,
                            to: newName
                        }
                    });
                    this.$message({
                        message: `保存成功`,
                        type: 'success'
                    });
                } catch (err) {
                    this.$message({
                        message: `重命名文件夹失败：${err.message}`,
                        type: 'error'
                    });
                }
            });
            //文件重命名
            bus.$on('Rename File', async (name) => {
                console.log(this.noteInfo);
                try {
                    const result = await api.KnowledgeBaseApi.getNoteContent({
                        kbGuid: this.selectedNodeData.kbGuid,
                        docGuid: this.selectedNodeData.docGuid,
                        data: {
                            downloadInfo: 1,
                            downloadData: 0,
                            clientType: 'web',
                            clientVersion: 4.0,
                            lang: 'zh-cn',
                        }
                    });
                    this.noteInfo = result.info;
                    console.log(result);
                    this.noteInfo.title = name;
                    await api.KnowledgeBaseApi.updateNoteInfo({
                        kbGuid: this.noteInfo.kbGuid,
                        docGuid: this.noteInfo.docGuid,
                        data: this.noteInfo
                    });
                    this.$message({
                        message: `保存成功`,
                        type: 'success'
                    });
                } catch (err) {
                    this.$message({
                        message: `重命名文件失败：${err.message}`,
                        type: 'error'
                    });
                }
            });

        }
    };
</script>

<style scoped>
    .main {
        display: flex;
        height: calc(100% - 35px);
        resize: none;
        width: calc(100% - 75px);
        position: absolute;
    }

    .file-tree {
        overflow: scroll;
        width: 200px;
    }

    .file-tree::-webkit-scrollbar {
        display: none;
    }

    .markdown {
        width: calc(100% - 200px);
    }

    .markdown::-webkit-scrollbar {
        display: none;
    }

    .auto-textarea-input::-webkit-scrollbar {
        display: none;
    }

    #menu {
        position: absolute;
        z-index: 99999999;
        border-radius: 4px;
        border: 1px solid #ebeef5;
        background-color: #fff;
        list-style: none;
        margin: 0;
        padding: 4px 0;
        transition: .3s;
        transition-property: all;
        transition-duration: 0.3s;
        transition-timing-function: ease;
        transition-delay: 0s;
        /*transition: opacity 150ms cubic-bezier(0.4, 0, 0.2, 1) 0ms, transform 100ms cubic-bezier(0.4, 0, 0.2, 1) 0ms;*/
        font-family: Helvetica Neue, Helvetica, PingFang SC, Hiragino Sans GB, Microsoft YaHei, SimSun, sans-serif;
        font-weight: 400;
        -webkit-font-smoothing: antialiased;
        -webkit-tap-highlight-color: transparent;
        box-shadow: 0 2px 12px 0 rgba(0, 0, 0, .1);
        font-size: 14px;
    }

    .menu-item {
        padding: 4px 16px;
        cursor: pointer;
    }

    .menu-item:hover {
        background-color: rgba(179, 216, 255, 0.3);
    }
</style>

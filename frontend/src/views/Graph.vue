<template>
  <div class="graph">
    <div class="main">
      <div class="title">
        <el-button round type="text" icon="el-icon-arrow-left" @click="back()"
          >返回</el-button
        >
        <span>项目：{{ projectInfo.name }}</span>
        <el-button
          round
          type="text"
          icon="el-icon-info"
          @click="showDescription()"
          >关于</el-button
        >
      </div>
      <!-- 侧边操作导航栏 -->
      <GraphSidebar
        :layoutMode="graphProperty.layoutMode"
        @layout-action="dispatchGraphAction"
        @graph-action="dispatchGraphAction"
        @editor-action="dispatchEditorAction"
        @check-document="checkDocument"
      ></GraphSidebar>
      <!-- 图谱中心 -->
      <GraphBoard
        ref="board"
        :preLoading="true"
        @init-property="initPropertyHandler"
        @editor-action="dispatchEditorAction"
      ></GraphBoard>
    </div>
    <div :class="['editor', showEditor ? 'open' : 'close']">
      <!-- 图谱修改面板 -->
      <GraphEditor
        ref="editor"
        :graphData="graphData"
        :nodesScale="graphProperty.nodeScale"
        :nodeOptions="nodeOptions"
        :nodeGroups="nodeGroups"
        @graph-action="dispatchGraphAction"
      ></GraphEditor>
    </div>
    <el-button
      class="editor-btn"
      @click="showEditor = !showEditor"
      :icon="`el-icon-arrow-${showEditor ? 'right' : 'left'}`"
    ></el-button>
    <GraphDocumentDrawer ref="graph_document_drawer" />
    <GraphModifyNameModal />
    <GraphModifyDescModal />
    <GraphModifyStatusModal />
    <GraphModifyProjectInfoModal />
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex';

import GraphBoard from '../modules/graph/components/GraphBoard';
import GraphEditor from '../modules/graph/components/GraphEditor';
import GraphSidebar from '../modules/graph/components/GraphSidebar';
import GraphModifyNameModal from '../modules/graph/components/GraphModifyNameModal';
import GraphModifyDescModal from '../modules/graph/components/GraphModifyDescModal';
import GraphModifyStatusModal from '../modules/graph/components/GraphModifyStatusModal';
import GraphDocumentDrawer from '../modules/graph/components/GraphDocumentDrawer';
import GraphModifyProjectInfoModal from '../modules/graph/components/GraphModifyProjectInfoModal';

export default {
  name: 'Graph',
  components: {
    GraphBoard,
    GraphEditor,
    GraphSidebar,
    GraphModifyNameModal,
    GraphModifyDescModal,
    GraphModifyStatusModal,
    GraphDocumentDrawer,
    GraphModifyProjectInfoModal,
  },
  data() {
    return {
      projectInfo: {},
      graphData: null,
      graphProperty: {
        layoutMode: 'FORCE',
        nodeScale: null,
      },
      showEditor: false,
    };
  },
  computed: {
    ...mapGetters([
      // 'graphData',
      // 'projectInfo',
    ]),
    nodeOptions() {
      const data = this.graphData;
      return data ? data.nodes.map(({ id, name }) => ({ id, name })) : [];
    },
    nodeGroups() {
      if (!this.graphData) return [];
      return [...new Set(this.graphData.nodes.map((node) => node.group))];
    },
  },
  methods: {
    ...mapActions({
      getProjectInfoAct: 'getProjectInfo',
      getProjectGraphDataAct: 'getProjectGraphData',
    }),
    back() {
      this.$router.push('/');
    },
    dispatchGraphAction(name, ...args) {
      // console.log(`[GraphAction] ${name}`, args)
      if (name === 'switchLayout') {
        this.graphProperty.layoutMode = args[0];
      }
      if (
        [
          'createNode',
          'createLink',
          'updateNode',
          'updateLink',
          'deleteNode',
          'deleteLink',
        ].includes(name)
      ) {
        const item = args[0];
        const { nodes, links } = this.graphData;
        ({
          createNode: () => {
            nodes.push(item);
          },
          createLink: () => {
            links.push(item);
          },
          updateNode: () => {
            const node = nodes.filter((node) => node.id === item.id)[0];
            node &&
              Reflect.ownKeys(item).forEach(
                (prop) => (node[prop] = item[prop]),
              );
          },
          updateLink: () => {
            const link = links.filter((link) => link.id === item.id)[0];
            link &&
              Reflect.ownKeys(item).forEach(
                (prop) => (link[prop] = item[prop]),
              );
          },
          deleteNode: () => {
            const node = nodes.filter((node) => node.id === item.id)[0];
            if (node) {
              nodes.splice(nodes.indexOf(node), 1);
              this.graphData.links = links.filter(
                ({ from, to }) => from !== node.id && to !== node.id,
              );
            }
          },
          deleteLink: () => {
            const target = links.filter((link) => link.id === item.id)[0];
            target && links.splice(links.indexOf(target), 1);
          },
        }[name]());
      }
      this.$refs.board[name](...args);
    },
    dispatchEditorAction(name, ...args) {
      // console.log(`[EditorAction] ${name}`, args)
      this.showEditor = true;
      this.$refs.editor.dispatchEditorAction(name, ...args);
    },
    initPropertyHandler(name, value) {
      // console.log(`[InitProperty] ${name}`)
      this.graphProperty[name] = value;
    },
    showDescription() {
      this.$notify({
        title: `关于 ${this.projectInfo.name}`,
        message: this.projectInfo.description,
        duration: 5000,
        type: 'info',
        position: 'bottom-left',
      });
    },
    checkDocument(id) {
      this.$refs.graph_document_drawer.check(id);
    },
  },
  async mounted() {
    const projectId = Number(this.$route.params.projectId);
    const projectInfo = await this.getProjectInfoAct(projectId);

    // const projectInfo = deepCopy(_projectInfo)
    if (!projectInfo) {
      // warning
      return;
    }
    this.projectInfo = projectInfo;

    const graphData = await this.getProjectGraphDataAct(projectId);
    // const graphData = deepCopy(_graphData)
    if (!graphData) {
      // warning
      return;
    }
    this.graphData = graphData;

    const graphBoard = this.$refs.board;
    graphBoard.mountGraphData(graphData, this.projectInfo);

    console.group('[Graph] mounted success');
    console.log('projectId:', projectId);
    console.log('projectInfo:', projectInfo);
    console.log('graphData:', graphData);
    console.groupEnd();
  },
};
</script>

<style scoped>
.graph {
  width: 100vw;
  height: 100vh;
  display: flex;
  align-items: stretch;
}
.graph > .main {
  flex: 1;
  position: relative;
}
.graph > .main > .title {
  position: absolute;
  left: 40px;
  top: 20px;
  border-radius: 20px;
  box-shadow: 1px 1px 1px 0px slategray;
  background-color: #ffffff;
  height: 40px;
  font-size: 20px;
  font-weight: 600;
  display: flex;
  align-items: center;
  z-index: 2000;
}
.graph > .editor {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  padding: 16px 24px;
  border-left: 1px solid #bbbbbb;
  border-bottom: 1px solid #bbbbbb;
  border-top: 1px solid #bbbbbb;
  border-top-left-radius: 30px;
  border-bottom-left-radius: 30px;
  transition: all 1s ease-out;
  overflow: auto;
}
.graph > .editor.open {
  width: 350px;
}
.graph > .editor.close {
  width: 0;
  padding: 0;
}
</style>

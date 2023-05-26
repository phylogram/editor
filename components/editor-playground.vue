<template>
  <v-container>
    <v-row>
      <v-col >
          <v-select
            label="NodeType"
            :items="nodeTypes"
            v-model="nodeType"
          >
          </v-select>
      </v-col>
      <v-col md="auto">
        <v-btn icon x-small @click="loadNodeTypes()" ><v-icon>mdi-refresh</v-icon> </v-btn>
      </v-col>
      <v-col>
        <v-select
          v-if="nodeType"
          label="RelationType"
          :items="relationTypes"
          v-model="relationType"
        >
        </v-select>
      </v-col>
      <v-col md="auto">
        <v-btn v-if="nodeType" icon x-small @click="loadRelationTypes()" ><v-icon>mdi-refresh</v-icon> </v-btn>
      </v-col>
      <v-col>
        <v-select
          v-if="relationType"
          label="Node Header Key"
          :items="nodeKeys"
          v-model="nodeHeaderKey"
        >
        </v-select>
      </v-col>
      <v-col>
        <v-select
          v-if="relationType"
          label="Node Content Key"
          :items="nodeKeys"
          v-model="nodeContentKey"
        >
        </v-select>
      </v-col>
      <v-col md="auto">
        <v-btn v-if="relationType" icon x-small @click="loadStartNodeOptions()" ><v-icon>mdi-refresh</v-icon> </v-btn>
      </v-col>
    </v-row>
    <v-col>
      <v-select
        v-if="startNodeOptions.length > 0 && nodeHeaderKey"
        label="StartNode"
        :items="startNodeSelectionItems"
        v-model="startNode"
      >
      </v-select>
    </v-col>
    <v-row>
      <v-col md="auto">
        <v-btn large @click="loadNodes()">(Re-)Load Nodes</v-btn>
      </v-col>
      <v-col md="auto">
        <v-btn large @click="saveChangedNodes()" :disabled="nodesAsyncWithServer.size === 0">Save changed nodes</v-btn>
      </v-col>
    </v-row>
    <v-row v-if="nodeContentKey">
        <node-editor
          v-for="(node, index) in nodes"
          :id="node.elementId"
          @input="updateNode($event)"
          @keydown="reactToLeaveNode(index, $event)"
          :content="node.properties[nodeContentKey]"
          :element-id="node.elementId"
        ></node-editor>

    </v-row>
  </v-container>
</template>

<script>
import {Driver} from "neo4j-driver";
import {nextTick} from "vue";

const nodeType_KEY = "nodeType_KEY";
const relationType_KEY = "relationType_KEY";
const startNode_KEY = "startNode_KEY";
const nodeHeaderKey_KEY = "nodeHeaderKey_KEY";
const nodeContentKey_KEY = "nodeContentKey_KEY";

export default {
  name: 'IndexPage',
  props: {
    driver: Driver,
  },
  beforeMount() {
    this.loadNodeTypes();
  },
  data() {
    return {
      nodeType: null,
      nodeTypes: [],
      relationType: null,
      relationTypes: [],
      nodes: [],
      startNodeOptions:[],
      startNode: null,
      nodeHeaderKey: null,
      nodeContentKey: null,
      nodesAsyncWithServer: new Map(),
    };
  },
  computed: {
    nodeKeys() {

      const propertyObjects = this.startNodeOptions.map(node => node.properties);
      const allPropertyKeys = propertyObjects.map(Object.keys).flat();
      return Array.from(new Set(allPropertyKeys));
    },
    startNodeSelectionItems() {
      if (this.nodeHeaderKey === null ) {
        return [];
      }
      return this.startNodeOptions.map(
        (node) => { return {value: node.elementId, text: node.properties[this.nodeHeaderKey]};}
      );
    },
    displayNodes() {
      return this.nodes.map(
        node => {
          return {
            header: node.properties[this.nodeHeaderKey],
            content: node.properties[this.nodeContentKey],
            elementId: node.elementId,
            properties: node.properties,
          };
        }
      );
    }
  },
  watch: {
    nodeType(_, __) {
      window.localStorage.setItem(nodeType_KEY, this.nodeType);
      this.loadRelationTypes();
    },
    relationType(_, __) {
      window.localStorage.setItem(relationType_KEY, this.relationType);
      this.loadStartNodeOptions();
    },
    startNode(_, __) {
      window.localStorage.setItem(startNode_KEY, this.startNode);
      this.loadNodes();
    },
    nodeHeaderKey(_, __) {
      window.localStorage.setItem(nodeHeaderKey_KEY, this.nodeHeaderKey);
    },
    nodeContentKey(_, __) {
      window.localStorage.setItem(nodeContentKey_KEY, this.nodeContentKey);
    },

  },
  methods: {
    async saveChangedNodes() {
      if (this.nodesAsyncWithServer.size === 0) {
        return;
      }
      const session = this.driver.session();
      const transaction = session.beginTransaction();

      for (const [elementId, nodeAsyncWithServer] of this.nodesAsyncWithServer) {
        const query = `
MATCH (node)
WHERE elementId(node) = $nodeId
SET node.${this.nodeContentKey} = $newContent
`;
        try {
          transaction.run(query, {nodeId: elementId, newContent: nodeAsyncWithServer.properties[this.nodeContentKey]});
        } catch (error) {
          console.error({error, query});
          await transaction.rollback();
          await transaction.close();
          await session.close();
          return;
        }
      }

      await transaction.commit();
      await transaction.close();
      await session.close();
      this.nodesAsyncWithServer.clear();
    },
    needsNewLine(text) {
      return text.includes('\n');
    },
    updateNode(event) {
      const node = this.nodes.find(node => node.elementId === event.elementId);
      node.properties[this.nodeContentKey] = event.content;
      this.nodesAsyncWithServer.set(event.elementId, node);
    },
    async loadNodeTypes() {
      const query = `MATCH (nodes) RETURN distinct labels(nodes)`;
      const result = await this.driver.executeQuery(query);
      this.nodeTypes = this.getResultFields(result);
    },
    async loadRelationTypes() {
      if (this.nodeType === null) {
        return;
      }
      const query = `MATCH (nodes:${this.nodeType})-[relations]->() RETURN distinct type(relations)`;
      const result = await this.driver.executeQuery(query);
      this.relationTypes = this.getResultFields(result);
    },
    async loadStartNodeOptions() {

      if (this.nodeType === null || this.relationType === null) {
        return;
      }
      const query = `MATCH (subject:${this.nodeType})-[follows:${this.relationType}]->(object:${this.nodeType})
WHERE NOT (:${this.nodeType})-[:${this.relationType}]->(subject)
 RETURN subject;`;
      this.startNodeOptions = this.getResultFields( (await this.driver.executeQuery(query)));
      if (this.startNodeOptions.length > 0) {
        this.startNode = this.startNodeOptions[0].elementId;
      }
    },
    getResultFields(result) {
      return result.records.map(row => row._fields).flat();
    },
    async loadNodes() {
      if (this.startNode === null) {
        return;
      }
      const query = `MATCH graph=(begin:${this.nodeType})-[relation:${this.relationType}*]->(:${this.nodeType}) where elementId(begin) = $startId RETURN nodes(graph) as elements;`
      const result = await this.driver.executeQuery(query, {startId: this.startNode});
      const lastRecord = result.records.pop();
      const elements = lastRecord.get('elements');
      this.nodes = elements.map(
        node => {
          return {
            elementId: node.elementId,
            properties: node.properties,
          };
        }
      );
      this.nodesAsyncWithServer.clear();
    },

    async reactToLeaveNode(index, event) {
      if (index < (this.nodes.length - 1 )) {
        return;
      }
      if (!(event.ctrlKey && event.key === '<')) {
        return;
      }
      const sourceId = this.nodes[this.nodes.length - 1].elementId;
      const session = this.driver.session();
      const transaction = session.beginTransaction();
      const nodeCreateResult = await transaction.run(`CREATE (newText:${this.nodeType} {${this.nodeContentKey}: ""}) RETURN elementId(newText) as newId`);
      const destinationId = nodeCreateResult.records[0].get('newId');
      await transaction.run(`
MATCH (source:${this.nodeType}), (destination:${this.nodeType})
WHERE elementID(source) = $sourceId AND elementID(destination) = $destinationId
CREATE (source)-[:${this.relationType}]->(destination);`,
            {sourceId, destinationId}
      );
      await transaction.commit();
      await transaction.close();
      await session.close();
      this.nodes.push({elementId: destinationId, properties: {[this.nodeContentKey]: ''}});

      await nextTick();

      document.getElementById(destinationId).focus();
    },
  },
  mounted() {
    this.nodeType = window.localStorage.getItem(nodeType_KEY);
    this.relationType = window.localStorage.getItem(relationType_KEY);
    this.startNode = window.localStorage.getItem(startNode_KEY);
    this.nodeHeaderKey = window.localStorage.getItem(nodeHeaderKey_KEY);
    this.nodeContentKey = window.localStorage.getItem(nodeContentKey_KEY);
  },
}
</script>
<style>
  input {
    color: aliceblue;
    display: inline;
    -webkit-box-sizing: border-box;
    -moz-box-sizing: border-box;
    box-sizing: border-box;
  }
</style>

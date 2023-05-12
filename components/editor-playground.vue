<template>
  <v-container>
    <v-row>
      <v-col>
          <v-select
            label="NodeType"
            :items="nodeTypes"
            v-model="nodeType"
          >
          </v-select>
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
      <v-btn large @click="updateNodes()">Update</v-btn>
    </v-row>
    <v-row v-if="nodeContentKey">
        <span  v-for="node in nodes" :id="node.elementId"  @input="updateNode($event)" contenteditable="true" >{{node.properties[nodeContentKey]}}</span>

    </v-row>
    <v-row>
      Add node with relationship
    </v-row>
  </v-container>
</template>

<script>
import {Driver, session} from "neo4j-driver";

export default {
  name: 'IndexPage',
  props: {
    driver: Driver,
  },
  beforeMount() {
    this.session = this.driver.session();
    this.loadNodeTypes();
  },
  data() {
    return {
      session: null,
      nodeType: null,
      nodeTypes: [1, 2, 3],
      relationType: null,
      relationTypes: [],
      nodes: [],
      startNodeOptions:[],
      startNode: null,
      nodeHeaderKey: null,
      nodeContentKey: null,
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
      this.loadRelationTypes();
    },
    relationType(_, __) {
      this.loadStartNodeOptions();
    },
    startNode(_, __) {
      this.loadNodes();
    }
  },
  methods: {
    needsNewLine(text) {
      return text.includes('\n');
    },
    updateNode(event) {
      console.debug(event);
    },
    updateNodes() {
      for (const node of this.nodes) {
        const element = document.getElementById(node.elementId);
        node.properties[this.nodeContentKey] = element.innerText
      }
    },
    async loadNodeTypes() {
      const query = `MATCH (nodes) RETURN distinct labels(nodes)`;
      const result = await this.session.executeRead(tx => tx.run(query));
      this.nodeTypes = this.getResultFields(result);
    },
    async loadRelationTypes() {
      if (this.nodeType === null) {
        return;
      }
      const query = `MATCH (nodes:${this.nodeType})-[relations]->() RETURN distinct type(relations)`;
      const result = await this.session.executeRead(tx => tx.run(query));
      this.relationTypes = this.getResultFields(result);
    },
    async loadStartNodeOptions() {
      if (this.nodeType === null || this.relationType === null) {
        return;
      }
      const query = `MATCH (subject:${this.nodeType})-[follows:${this.relationType}]->(object:${this.nodeType})
WHERE NOT (:${this.nodeType})-[:${this.relationType}]->(subject)
 RETURN subject;`;
      this.startNodeOptions = this.getResultFields( (await this.session.executeRead(tx => tx.run(query))));
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
      const result = await this.session.executeRead(tx => tx.run(query, {startId: this.startNode}));
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
    }
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

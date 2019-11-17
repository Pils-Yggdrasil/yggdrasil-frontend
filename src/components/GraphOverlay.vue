<template>
<div>
  <custom-tooltip @favorite="setFavorite" v-show="hovered_node !== null" :node_settings="node_settings" :position="hovered_node_location" id="infoBoxHolder" :node="this.hovered_node">
  </custom-tooltip>

  <tree-v2 @hover_node="setHoveredNode" v-if="draw" :node_charge="parseInt(node_charge)" :cdpScore_threshold="parseInt(cdpScore_threshold)" :distance_nodes="parseInt(distance_nodes)" :adjlist="adjlist" :graph_original="graph">
  </tree-v2>

  <div class="custom-container">
    <b-row align-h="end">
      <!-- <b-col cols="8">
        Cliquer <span class="link" @click="search">ici</span> pour relancer une recherche
        <br />
        Cliquer <span class="link" @click="addNode">ici</span> pour ajouter un noeud
        <br />
        <template v-if="graph !== null">
          Il y a {{graph.nodes.length}} noeuds et {{graph.links.length}} liens
        </template>
      </b-col> -->
      <b-col cols="auto">
        <display-settings @charge="setCharge" @distance="setDistance" @cdp="setCdp" @favorites="setFavorites"></display-settings>
      </b-col>
    </b-row>
  </div>
</div>
</template>

<script>
import DisplaySettings from './DisplaySettings.vue'
import CustomTooltip from './CustomTooltip.vue'
import TreeV2 from './TreeV2.vue'
import Vue from 'vue'
export default {
  name: 'graph-overlay',
  components: {
    TreeV2,
    CustomTooltip,
    DisplaySettings
  },
  props: {
    nodes: Array,
    socket: Object
  },
  data() {
    return {
      draw: false,
      total_nodes: [],
      total_links: [],
      graph: null,
      adjlist: {},
      hovered_node_location: {
        F: -25,
        G: 100
      },
      node_settings: {
        diameter: 20
      },
      hovered_node: null,
      graph: {
        nodes: [],
        links: []
      },
      total_width: 0,
      node_charge: -6000,
      distance_nodes: 100,
      cdpScore_threshold: 5,
      favorites_only: false,
    }
  },
  computed: {
    links_checker() {
      return this.total_links.filter(link => {
        return link.source.constructor === String && link.target.constructor === String
      }).length
    },
  },
  methods: {
    setFavorite(bool) {
      var self = this;
      this.hovered_node.favorite = bool;
      this.hovered_node = Object.assign({}, this.hovered_node)
      let index = this.total_nodes.findIndex(n => n.id == self.hovered_node.id)
      this.total_nodes[index].favorite = bool;
      index = this.graph.nodes.findIndex(n => n.id == self.hovered_node.id)
      this.graph.nodes[index].favorite = bool
    },
    setHoveredNode(d) {
      this.hovered_node = d;
    },
    copyNestedObject(obj) {
      var self = this;
      var ret
      switch (obj) {
        case null:
          return null;
          break;
        case undefined:
          return undefined;
          break
      }
      var self = this;
      let type = obj.constructor
      switch (type) {
        case Number:
          return obj;
          break;
        case Boolean:
          return obj;
          break;
        case String:
          return obj;
          break;
        case Array:
          ret = []
          for (let i = 0; i < obj.length; i++) {
            ret.push(self.copyNestedObject(obj[i]))
          }
          return ret
          break;
        case Object:
          ret = {}
          var keys = Object.keys(obj)
          for (let i = 0; i < keys.length; i++) {
            ret[keys[i]] = self.copyNestedObject(obj[keys[i]])
          }
          return ret
          break;
      }
    },
    slowAddNode(delay) {
      var self = this;
      if (this.cursor == 100) {
        this.init();
      }
      if (this.cursor < this.total_nodes.length) {
        this.addNode(this.total_nodes[this.cursor])
      }
      this.cursor += 1;
      if (this.cursor < this.nodes_limit) {
        setTimeout(self.slowAddNode.bind(null, delay), delay)
      }

    },
    search() {
      this.$emit('search')
    },
    addNode(node) {
      var self = this
      node.id = node.paperId
      node.color = 'white'
      node.references.forEach(ref => {
        if (self.total_nodes.map(node => node.id).includes(ref.paperId)) {
          let link = {
            source: ref.paperId,
            target: node.paperId,
            value: 1
          }
          if (!self.total_links.includes(link)) {
            self.total_links.push(link)
          }
        }
      })
      node.citations.forEach(cit => {
        if (self.total_nodes.map(node => node.key).includes(cit.paperId)) {
          let link = {
            source: node.paperId,
            target: cit.paperId,
            value: 1
          }
          if (!self.total_links.includes(link)) {
            self.total_links.push(link)
          }
        }
      })
      node.id = node.paperId
      node.group = Math.round(node.citations.length / 100)
      if (!self.total_nodes.map(nod => nod.paperId).includes(node.paperId) && node.paperId != null && node.paperId !== undefined) {
        self.total_nodes.push(node);
      }
    },
    computeEventual_nodes() {
      var self = this
      var nodes = this.total_nodes.filter(node => {
        var filt = true
        if(this.favorites_only && !node.favorite){
          filt = false
        }
        return filt && node.cdpScore >= this.cdpScore_threshold
      })
      return nodes
    },
    computeEventual_links(nodes) {
      var self = this;

      var links = this.copyNestedObject(self.total_links).filter(link => nodes.map(node => node.paperId).includes(link.source) && nodes.map(node => node.paperId).includes(link.target))
      for (let i = 0; i < links.length; i++) {
        self.adjlist[links[i].source + "-" + links[i].target] = true;
        self.adjlist[links[i].target + "-" + links[i].source] = true;
      }
      Vue.set(self, 'adjlist', self.adjlist);
      return links
    },
    addCircle() {
      this.graph.nodes.push({
        id: '45454545454',
        'cdpScore': 124,
        title: "qsdmlkj",
        x: this.graph.nodes[0].x,
        y: this.graph.nodes[0].y,
        citations: [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
      })
    },
    updateNodes() {
      console.log("Update nodes")
      var self = this
      var nodes = [...this.computeEventual_nodes()]
      console.log("Updated nodes")
      var links = [...this.computeEventual_links(nodes)]
      console.log("Updated links")

      // this.graph = {
      //   nodes: nodes,
      //   links: links
      // }
      Vue.set(self.graph, 'nodes', nodes)
      Vue.set(self.graph, 'links', links)
    },
    setCharge(charge) {
      this.node_charge = charge;
    },
    setCdp(cdp) {
      this.cdpScore_threshold = cdp;
      this.updateNodes()
    },
    setDistance(distance) {
      this.distance_nodes = distance;
    },
    setFavorites(fav) {
      console.log("Je veux set les favoris : ",fav)
      this.favorites_only = fav
      this.updateNodes();
    }
  },
  mounted() {
    var self = this;
    setTimeout(() => {
      self.updateNodes();
      self.draw = true
    }, 3000)
    var self = this;
    console.log("Launching stuff")
    this.nodes.forEach(node => {
      this.addNode(node);
    })
    this.socket.on('done', () => {
      console.log("RECEIVED ALL")
    })
    this.socket.on('new_node', (data) => {
      this.addNode(data)
    })
  },

}
</script>

<style scoped>
.custom-container {
  position: fixed;
  z-index: 500;
  right: 50px;
  top: 50px;
  width: auto;
}

#infoBoxHolder {
  z-index: 300;
  position: fixed;
}
</style>
<template>
<div>

  <custom-tooltip v-show="hover_node" :node_settings="node_settings" :position="hovered_node_location" id="infoBoxHolder" :node="this.hovered_node">
  </custom-tooltip>
  <!-- <div id="myDiagramDiv" :style="{height:'100vh',width:'100vw'}">
  </div> -->
  <svg id='viz' :style="{height:'100vh',width:'100vw'}">
    <g id='container'>
      <g class="links" id="g_links">
      </g>
      <g class="nodes" id="g_nodes">
      </g>
    </g>
  </svg>
  <div class="custom-container">
    <b-row align-h="center">
      <b-col cols="auto">
        Cliquer <span class="link" @click="search">ici</span> pour relancer une recherche
        <br />
        Cliquer <span class="link" @click="addNode">ici</span> pour ajouter un noeud
        <br />
        Il y a {{eventual_nodes.length}} noeuds et {{eventual_links.length}} liens
      </b-col>
      <b-col cols="auto" class="text-left">
        Charge : <input type="range" min="-100000" max="0" v-model="node_charge" class="slider" id="myRange" />{{node_charge}}
        <br />
        Cdp threshold : <input @change="updateNodes" type="range" min="0" max="400" v-model="cdpScore_threshold" class="slider" />{{cdpScore_threshold}}
        <br />
        Distance : <input @change="refresh" type="range" min="0" max="1000" v-model="distance_nodes" class="slider" />{{distance_nodes}}
      </b-col>
    </b-row>
  </div>
</div>
</template>

<script>
import CustomTooltip from './CustomTooltip.vue'
import * as d3 from 'd3'
import Vue from 'vue'

export default {
  name: "tree-d3",
  props: {
    nodes: Array,
    socket: Object
  },
  components: {
    CustomTooltip
  },
  mounted() {
    var self = this;
    console.log("Launching stuff")
    // this.init()
    // self.slowAddNode(1);
    this.nodes.forEach(node => {
      this.addNode(node);
    })
    setTimeout(this.init, 3000)
    this.socket.on('done', () => {
      console.log("RECEIVED ALL")
      this.init()
    })
    this.socket.on('new_node', (data) => {
      this.addNode(data)
    })
  },
  data() {
    return {
      current_x : 0,
      node_charge: -5000,
      drawn: false,
      cursor: 0,
      total_nodes: [],
      distance_nodes: 50,
      graphLayout:null,
      total_links: [],
      total_links_2: [],
      hovered_node: null,
      hovered_node_location: {
        F: -25,
        G: 100
      },
      diagram: null,
      total_width: 0,
      hover_node: false,
      nodes_limit: 2,
      mainColor: "#2c3e50",
      lightColor: "rgb(150,150,150)",
      redColor: "#ff6a6a",
      greenColor: "#41b883",
      nodeDiameter: 200,
      current_node: null,
      cdpScore_threshold: 0,
      eventual_links: [],
      eventual_nodes: [],
    }
  },
  computed: {
    // pseudo_node() {
    //   var self = this
    //   return d3.select("#g_nodes")
    //     // .selectAll("g")
    //     .data(self.graph.nodes)
    //     .enter()
    // },
    // node() {
    //   var self = this;
    //   return self.pseudo_node
    //     .append("circle")
    //     .attr("r", (d) => {
    //       return 3 * Math.pow(d.cdpScore, 1 / 2)
    //     })
    //     .attr("fill", "white")
    //     .attr("stroke", function(d) {
    //       return "#2c5e50";
    //     })
    // },
    graph() {
      return {
        nodes: [...this.eventual_nodes],
        links: [...this.eventual_links]
      }
    },
    node_settings() {
      return {
        diameter: this.nodeDiameter
      }
    },
    links_checker() {
      return this.total_links.filter(link => {
        return link.source.constructor === String && link.target.constructor === String
      }).length
    }
    // eventual_nodes() {
    //   var self = this;
    //   return (self.total_nodes).filter(node => node.cdpScore >= this.cdpScore_threshold)
    // },
    // eventual_links() {
    //   return this.copyNestedObject((self.total_links)).filter(link => self.eventual_nodes.map(node => node.paperId).includes(link.source) && self.eventual_nodes.map(node => node.paperId).includes(link.target))
    // },
  },
  watch: {
    node_charge() {
      console.log("force changed")
    }
  },
  methods: {
    computeEventual_nodes() {
      var self = this
      this.eventual_nodes = (self.total_nodes).filter(node => node.cdpScore >= this.cdpScore_threshold)
    },
    computeEventual_links() {
      var self = this;
      this.eventual_links = this.copyNestedObject(self.total_links).filter(link => self.eventual_nodes.map(node => node.paperId).includes(link.source) && self.eventual_nodes.map(node => node.paperId).includes(link.target))
    },
    copyNestedObject: function pf(obj) {
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
          return obj.map(cell => pf(cell))
          break;
        case Object:
          Object.keys(obj).map(key => {
            obj[key] = pf(obj[key])
          })
          return obj
          break;
      }
    },
    updateNodes() {
      this.drawn = false;

      this.init();
      // this.computeEventual_nodes()
      // console.log("Update Nodes")
      // var self = this
      // var node = d3.selectAll("circle.node")
      //   .data(self.graph.nodes)
      //
      // console.log(node)
      // node.enter().append("circle")
      //   .attr("class", "node")
      //   .attr("cx", (d) => 800)
      //   .attr("cy", (d) => 200)
      //   .attr("r", (d) => {
      //     return 3 * Math.pow(d.cdpScore, 1 / 2)
      //   })
      //   .style("fill", "white")
      //   .style("stroke", "gray")
      //   .style("stroke-width", 1.0)
      //
      // node.exit().remove()
      // self.updateLinks()
    },
    updateLinks() {
      this.computeEventual_links()
      var self = this;
      var link = d3.selectAll("line.link")
        .data(self.graph.links)
      link.enter().append("line")
        .attr("class", "link")
        .attr("stroke", "#ddd")
        .attr("stroke-opacity", 0.8)
        .attr("x1", (d) => d.source.x)
        .attr("y1", (d) => d.source.y)
        .attr("x2", (d) => d.target.x)
        .attr("y2", (d) => d.target.y)

      link.exit().remove()
      this.total_links = [...this.total_links_2]
    },
    refresh() {
      console.log("CDP CONSTRAINT CHANGED")
    },
    slowAddNode(delay) {
      console.log("Adding ", this.cursor)
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
          let link_2 = {
            source: ref.paperId,
            target: node.paperId,
            value: 1
          }
          if (!self.total_links.includes(link)) {
            self.total_links.push(link)
            self.total_links_2.push(link_2)
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
          let link_2 = {
            source: node.paperId,
            target: cit.paperId,
            value: 1
          }
          if (!self.total_links.includes(link)) {
            self.total_links.push(link)
            self.total_links_2.push(link_2)
          }
        }
      })
      node.id = node.paperId
      node.group = Math.round(node.citations.length / 100)
      if (!self.total_nodes.map(nod => nod.paperId).includes(node.paperId) && node.paperId != null && node.paperId !== undefined) {
        self.total_nodes.push(node);
      }
    },
    init() {
      console.log("Hey")
      if (this.drawn) {
        return;
      }
      this.drawn = true;
      var self = this;
      var svg = d3.select("svg"),
        width = 1920,
        height = 1080;
      var color = d3.scaleOrdinal(d3.schemeCategory10);
      this.computeEventual_nodes();
      this.computeEventual_links();
      console.log("links : ", this.eventual_links)
      console.log("nodes : ", this.eventual_nodes)
      var graph = this.graph

      console.log(graph);
      var label = {
        'nodes': [],
        'links': []
      };

      graph.nodes.forEach(function(d, i) {
        label.nodes.push({
          node: d
        });
        label.nodes.push({
          node: d
        });
        label.links.push({
          source: i * 2,
          target: i * 2 + 1
        });
      });


      // var labelLayout = d3.forceSimulation(label.nodes)
      //   .force("charge", d3.forceManyBody().strength(-50))
      //   .force("link", d3.forceLink(label.links).distance(0).strength(2));

      let links = [...graph.links]
      let nodes = [...graph.nodes]
      var graphLayout = d3.forceSimulation(nodes)
        .force("charge", d3.forceManyBody().strength(self.node_charge))
        .force("center", d3.forceCenter(width / 2, height / 2))
        .force("collide",d3.forceCollide().radius(d => d.r * -100))
        .force("link", d3.forceLink(links).id(d => d.id)
          .distance(self.distance_nodes).strength(1))
        .on("tick", ticked);
      this.graphLayout=graphLayout

      var adjlist = [];

      graph.links.forEach(function(d) {
        adjlist[d.source.index + "-" + d.target.index] = true;
        adjlist[d.target.index + "-" + d.source.index] = true;
      });

      function neigh(a, b) {
        return a == b || adjlist[a + "-" + b];
      }


      var svg = d3.select("#viz").attr("width", width).attr("height", height);
      var container = d3.select("#container");

      svg.call(
        d3.zoom()
        .scaleExtent([.1, 4])
        .on("zoom", function() {
          container.attr("transform", d3.event.transform);
        })
      );

      var pseudo_node = d3.select("#g_nodes")
        .selectAll("g")
        .data(self.graph.nodes)
        .enter()

      this.total_links = [...this.total_links_2]

      var node = pseudo_node
        .append("circle")
        .attr("class", "node")
        .attr("r", (d) => {
          return 3 * Math.pow(d.cdpScore, 1 / 2)
        })
        .attr("fill", "white")
        .attr("stroke", function(d) {
          return color(d.group);
        })
      let n = node
      console.log(n)
      // .append("circle")
      var link = d3.select("#g_links")
        .selectAll("line")
        .data([...graph.links])
        .enter()
        .append("line")
        .attr("class", "link")
        .attr("stroke", "#aaa")
        .attr("stroke-width", "1px")

      var circle_text = pseudo_node
        .append("text")
        .attr("class", "text_circle")
        .attr("text-anchor", "middle")
        .text(function(d, i) {
          return d.title + "\n" + d.group
        })
        .style("fill", function(d) {
          return color(d.group);
        })
        .style("white-space", "nowrap")
        .style("overflow", "hidden")
        .style("font-family", "Arial")
        .style("font-size", '5px')
        .style("pointer-events", "none"); // to prevent mouseover/drag capture

      node.call(
        d3.drag()
        .on("start", dragstarted)
        .on("drag", dragged)
        .on("end", dragended)
      );

      // var labelNode = container.append("g").attr("class", "labelNodes")
      //   .selectAll("text")
      //   .data(label.nodes)
      //   .enter()
      //   .append("text")
      //   .text(function(d, i) {
      //     return i % 2 == 0 ? "" : d.node.title;
      //   })
      //   .style("fill", "transparent")
      //   .style("font-family", "Arial")
      //   .style("font-size", 12)
      //   .style("pointer-events", "none"); // to prevent mouseover/drag capture

      node.on("mouseover", (node) => {
        focus(node);
        self.hover_node = true;
        self.hovered_node = node;
      }).on("mouseout", unfocus);

      function ticked() {
        console.log("ticked")
        circle_text.call(updateCircleText)
        node.call(updateNode);
        // link.call(updateLink);
        graphLayout.nodes(self.graph.nodes)
        graphLayout.force("charge", d3.forceManyBody().strength(self.node_charge))
          .force("link", d3.forceLink(graph.links).id(function(d) {
            return d.id;
          }).distance(self.distance_nodes).strength(1))
        self.current_x -= 10
      }

      function fixna(x) {
        if (isFinite(x)) return x;
        return 0;
      }

      function focus(d) {
        var index = d3.select(d3.event.target).datum().index;
        node.style("opacity", function(o) {
          return neigh(index, o.index) ? 1 : 0.1;
        });
        // labelNode.attr("display", function(o) {
        //   return neigh(index, o.node.index) ? "block" : "none";
        // });
        link.style("opacity", function(o) {
          return o.source.index == index || o.target.index == index ? 1 : 0.1;
        });
      }

      function unfocus() {
        // labelNode.attr("display", "block");
        node.style("opacity", 1);
        link.style("opacity", 1);
      }

      function updateLink(link) {
        link.attr("x1", function(d) {
            return fixna(d.source.x);
          })
          .attr("y1", function(d) {
            return fixna(d.source.y);
          })
          .attr("x2", function(d) {
            return fixna(d.target.x);
          })
          .attr("y2", function(d) {
            return fixna(d.target.y);
          });
      }

      function updateNode(node) {
        node.attr("transform", function(d) {
          return "translate(" + fixna(d.x) + "," + fixna(d.y) + ")";
        });
      }

      function updateCircleText(circle_text) {
        circle_text.attr("transform", function(d) {
          return "translate(" + fixna(d.x) + "," + fixna(d.y) + ")";
        });
      }

      function dragstarted(d) {
        d3.event.sourceEvent.stopPropagation();
        if (!d3.event.active) graphLayout.alphaTarget(0.3).restart();
        d.fx = d.x;
        d.fy = d.y;
      }

      function dragged(d) {
        d.fx = d3.event.x;
        d.fy = d3.event.y;
      }

      function dragended(d) {
        if (!d3.event.active) graphLayout.alphaTarget(0);
        d.fx = null;
        d.fy = null;
      }
    },
  }
}
</script>

<style scoped>
.custom-container {
  position: fixed;
  left: 10px;
  z-index: 500;
  right: 10px;
  top: 10px;
}

#infoBoxHolder {
  z-index: 300;
  position: fixed;
}
</style>

<template>
  <div class="container">
    <div class="row">
      <div class="col">
        <h1>
          {{ title }}
          <small>{{ description }}</small>
        </h1>
      </div>
      <div class="col">
        <span @click="addNode()" class="glyphicon glyphicon-plus d3GraphCreatorMenuIcon"></span>
        <span
          @click="deleteSelectedNode()"
          class="d3GraphCreatorMenuIcon glyphicon glyphicon-minus"
        ></span>
        <span @click="toggleFixed()" class="d3GraphCreatorMenuIcon glyphicon glyphicon-pushpin"></span>
        <span @click="addLink()" class="d3GraphCreatorMenuIcon glyphicon glyphicon-link"></span>
        <span
          @click="downloadSvg(parentId)"
          class="d3GraphCreatorMenuIcon glyphicon glyphicon-picture"
        ></span>
        <span
          @click="downloadJson(dataset)"
          class="d3GraphCreatorMenuIcon glyphicon glyphicon-download"
        ></span>
      </div>
    </div>
  </div>
  <div :id="parentId" class="d3GraphCreatorSvgParent"></div>

  <table class="gc-table table">
    <thead>
      <tr>
        <th>Index</th>
        <th>Name</th>
        <th>Radius</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(node, index) in selectedNodes" :key="'selectedNode' + index">
        <td>{{ index }}</td>
        <td>
          <input class="form-control" @change="updateNode(node)" v-model="node.name.en" type="text" />
        </td>
        <td>
          <input
            class="form-control"
            @change="updateNode(node)"
            v-model="node.radius"
            type="number"
          />
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script setup>
import * as d3 from 'd3';
import { v4 as uuidv4 } from 'uuid';
import { ref, onMounted, onBeforeMount } from 'vue';

//These emits all send up to a parent node, if required
let emit = defineEmits([
  'nodes', //active nodes 
  'links', //active links
  'addNodes', //array of added nodes
  'deleteNodes', //array of deleted nodes
  'selectNodes',  //array of selected nodes
  'deselectNodes', //array of deselected nodes
  'updatedNodes', //array of nodes updated, just a subset
])

//These props allow us to customize the 
const props = defineProps({
  title: { type: String, default: "D3 Graph Creator" },
  description: String,
  nodesSource: {
    type: Array, default: [
      { "id": "jean-michel", "name": { "en": "Jean-Michel", "fr": "" }, "group": 1, "radius": 30 },
      { "id": "alexandra", "name": { "en": "Alexandra", "fr": "" }, "group": 1, "radius": 30 },
      { "id": "hans", "name": { "en": "Hans", "fr": "" }, "group": 1, "radius": 20 },
      { "id": "olafur", "name": { "en": "Ólafur", "fr": "" }, "group": 1, "radius": 15 },
      { "id": "john", "name": { "en": "John", "fr": "" }, "group": 1, "radius": 10 }
    ]
  },
  linksSource: {
    type: Array, default: [
      { "source": "jean-michel", "target": "alexandra", "value": 1, "type": { "en": "Brother of", "fr": "Frère de" } },
      { "source": "jean-michel", "target": "hans", "value": 1, "type": { "en": "Brother Of", "fr": "Frère de" } },
      { "source": "jean-michel", "target": "olafur", "value": 1, "type": { "en": "Employer Of", "fr": "Employeur de" } },
      { "source": "jean-michel", "target": "john", "value": 1, "type": { "en": "Employer Of", "fr": "Employeur de" } },
      { "source": "olafur", "target": "hans", "value": 1, "type": { "en": "Associate Of", "fr": "Associé de" } },
      { "source": "hans", "target": "olafur", "value": 1, "type": { "en": "Associate Of", "fr": "Associé de" } }
    ]
  }
})

var width = 1, height = 1;
var dataset = { nodes: [], links: [] };
let parentId = ref('')
let simulation, link, node = null;
let container, svg = null;

//Active Node and Link Subsets
let selectedNodes = ref([]);
let selectedLinks = ref([]);
let currentNode = ref(null);
let currentLink = ref(null);

onBeforeMount(() => {
  parentId.value = uuidv4();
  dataset.nodes = JSON.parse(JSON.stringify(props.nodesSource));
  dataset.links = JSON.parse(JSON.stringify(props.linksSource));
});

onMounted(() => {
  initialize()
});

function initialize() {
  var parentDiv = document.getElementById(parentId.value);
  width = parentDiv.clientWidth;
  if (!width) width = 500;
  height = parentDiv.clientHeight;
  if (!height) height = 500;
  svg = d3.select(parentDiv).append("svg");
  svg.attr("width", width)
    .attr("height", height)
    .attr("style", "min-height:500px;font-family:sans-serif")
    .attr("xmlns", "http://www.w3.org/2000/svg")
    .attr("xmlns:svg", "http://www.w3.org/2000/svg")
    .attr("xmlns:xlink", "http://www.w3.org/1999/xlink");

  //Establish the container to permit zoom and pan
  container = svg.append("g");

  svg.on("contextmenu", (e, d) => {
    e.preventDefault();
    var nodesFilter = dataset.nodes.filter((node) => { return node.selected })
    nodesFilter.forEach((n, index) => { n.selected = false })
    selectedNodes.value.splice(0, selectedNodes.value.length);
    loadLinks();
    loadNodes();
    restartSimulation();
  });

  //Implement Zoom and Pan
  const handleZoom = (e) => container.attr('transform', e.transform);
  const zoom = d3.zoom().on('zoom', handleZoom);
  svg.call(zoom);

  // Build the arrow definition
  var defs = svg.append("svg:defs")
  var arrow = defs.selectAll("marker")
    .data(["end"])      // Different link/path types can be defined here
    .enter().append("svg:marker")    // This section adds in the arrows
    .attr("id", String)
    .attr("viewBox", "0 -3 6 6")
    .attr("refX", 4)
    .attr("refY", -.3)
    .attr("markerWidth", 6)
    .attr("markerHeight", 6)
    .attr("orient", "auto")
    .append("svg:path")
    .attr("d", "M0,-2.0L5,0L0,2.0")
    .attr("stroke", "#777")
    .attr("stroke-width", .5)
    .attr("fill", "#666")
    ;         // .attr('transform', 'rotate(-10)')

  //Filter for the outside glow
  var filter = defs.append("filter")
    .attr("id", "glow");
  filter.append("feGaussianBlur")
    .attr("stdDeviation", "2")
    .attr("result", "coloredBlur");
  var feMerge = filter.append("feMerge");
  feMerge.append("feMergeNode")
    .attr("in", "coloredBlur");
  feMerge.append("feMergeNode")
    .attr("in", "SourceGraphic");

  var gooey = defs.append("filter")
    //use a unique id to reference again later on
    .attr("id", "gooeyCodeFilter");

  //Append multiple "pieces" to the filter
  gooey.append("feGaussianBlur")
    .attr("in", "SourceGraphic")
    .attr("stdDeviation", "20")
    .attr("color-interpolation-filters", "sRGB")
    .attr("result", "blur");
  gooey.append("feColorMatrix")
    //the class used later to transition the gooey effect
    .attr("class", "blurValues")
    .attr("in", "blur")
    .attr("mode", "matrix")
    .attr("values", "1 0 0 0 0  0 1 0 0 0  0 0 1 0 0  0 0 0 19 -9")
    .attr("result", "gooey");

  //If you want the end shapes to be exactly the same size as without
  //the filter add the feBlend below. However this will result in a
  //less beautiful gooey effect
  gooey.append("feBlend")
    .attr("in", "SourceGraphic")
    .attr("in2", "gooey");

  //Instead of the feBlend, you can do feComposite. This will also
  //place a sharp image on top. But it will result in smaller circles
  // gooey.append("feComposite") 
  // feBlend
  //     .attr("in","SourceGraphic")
  //     .attr("in2","gooey")
  //     .attr("operator","atop");

  //Apply the filter to the group element of all the circles
  // var circleWrapper = svg.append("g")
  ;

  //Configure the Container and SVG
  container = svg.append('g');
  container.append('g').attr('class', 'links');//.style("filter", "url(#gooeyCodeFilter)");;
  container.append('g').attr('class', 'nodes');//.style("filter", "url(#gooeyCodeFilter)");

  simulation = d3.forceSimulation();
  initSimulation();

  link = container.select('.links').selectAll('path');
  loadLinks();

  node = container.select('.nodes').selectAll('.node');
  loadNodes();

  restartSimulation();
}

function initSimulation() {

  console.log("width", width);
  console.log("height", height);

  simulation
    .force('link', d3.forceLink())
    .force('charge', d3.forceManyBody())
    .force('collide', d3.forceCollide())
    .force('center', d3.forceCenter(width / 2, height / 2))
    .force('forceX', d3.forceX().x(width / 2))
    .force('forceY', d3.forceY().y(height / 2));

  simulation.force('link')
    .id((d) => d.id)
    .distance(function (d) {
      return d.radius ? parseInt(d.radius) * 50 : 200
    })
    .iterations(.1)
    .strength(.1);

  simulation.force('collide').radius(function (d) { return d.radius ? parseInt(d.radius) : 2 }).strength(1);
  simulation.force('charge').strength(-100);
}

function loadLinks() {

  //Handle the Links
  container.select('.links')
    .selectAll('.link').remove();

  link = container.select('.links')
    .selectAll('.link')
    .data(dataset.links, (d) => d.id)

  var linkEnter = link.enter().append("path")
    .attr('id', (d, i) => parentId.value + 'textarc' + i)
    .attr("fill", "none")
    .attr("class", 'link')
    .attr("stroke", "#aaa") 
    .attr("stroke-width", 3)
    .attr("marker-end", "url(#end)");

  link = linkEnter.merge(link);


  //Handle the linkTypes
  container.select('.links')
    .selectAll('.linkType').remove();

  var linkType = container.select('.links')
    .selectAll('.linkType')
    .data(dataset.links, (d) => d.id)

  var linkEnter = linkType.enter().append("text")
    .attr("class", 'linkType')
    .attr("fill", "#77a")
    .attr("font-size", ".8em")
    .attr("font-weight", "600")
    .attr('dy', "-10px")
    .append("textPath")
    .attr("xlink:href", (d, i) => '#' + parentId.value + 'textarc' + i)
    .attr('startOffset', '60%')
    .style("text-anchor", "middle")
    .text(function (d) {
      // var dSelected = false;
      if (d.source.selected || d.target.selected) return d?.type?.en ? d.type.en + " \u003E" : "[ ]";
      else return "\u00A0";
    })

  linkType = linkEnter.merge(linkType);

  emit('links', dataset.links)
}

function loadNodes() {
  node = container.select('.nodes')
    .selectAll('.node').remove();

  // 
  node = container.select('.nodes')
    .selectAll('.node')
    .data(dataset.nodes, (d) => d.id)
    .join(
      enter => {
        const nodes = enter.append('g')
          .attr('class', 'node')
          .call(drag(simulation))

        //For Rectanges
        // nodes.append("rect")
        //     .attr("x", d => d.radius ? -d.radius : -10)
        //     .attr("y", d => d.radius ? -d.radius : -10)
        //     .attr("width", d => d.radius ? d.radius * 2 : 20)
        //     .attr("height", d => d.radius ? d.radius * 2 : 20)
        //     .attr("class", "d3circle")
        //     .attr("stroke", "#777")
        //     .attr("stroke-width", 3)
        //     .attr("fill", '#fff');

        nodes.append("circle")
          .attr("stroke", "#777")
          .attr("stroke-width", 3)
          .attr("r", d => d.radius ? parseInt(d.radius) : 20)
          .attr("fill", (d) => {
            if (!d.selected) return '#fff';
            if (d.selected && d.selectedIndex == 0) return '#ffc';
            if (d.selected && d.selectedIndex > 0) return '#fda';
          })
        // .style("filter", "url(#glow)");

        //Underlying text
        nodes.append("text")
          .attr("x", d => parseInt(d.radius) + 2 ? d.radius + 8 : 24)
          .attr("y", "0.31em")
          .text(d => d.name.en)
          .attr("fill", "#fff")
          .attr("stroke-width", 3)
          .attr("stroke", "#fff")
        // .style("filter", "url(#glow)");

        //Overlapping tax
        nodes.append("text")
          .attr("x", d => d.radius ? parseInt(d.radius) + 8 : 24)
          .attr("y", "0.31em")
          .text(d => d.name.en)
          .attr("fill", "#333")
          // .style("filter", "url(#glow)");
          ;



        return nodes;
      },
    );

  emit('nodes', dataset.nodes)
}

function drag(simulation) {

  function dragstarted(event, d) {
    if (!event.active) simulation.alphaTarget(0.3).restart();
    d.fx = d.x;
    d.fy = d.y;
    selectNode(d)

  }

  function dragged(event, d) {
    d.fx = event.x;
    d.fy = event.y;
  }

  function dragended(event, d) {
    if (!event.active) simulation.alphaTarget(0);
    // d.fx = null;
    // d.fy = null;
    // deselectNode(d)
  }

  return d3.drag()
    .on("start", dragstarted)
    .on("drag", dragged)
    .on("end", dragended);
}






function restartSimulation() {
  simulation.nodes(dataset.nodes);
  simulation.force('link').links(dataset.links);
  simulation.alpha(1).restart();
  simulation.on('tick', ticked);
}

function ticked() {

  //Default curves, centre to centre
  //  link.attr("d", function(d) {
  //     var dx = (d.target.x - d.source.x),
  //       dy = (d.target.y - d.source.y),
  //       dr = Math.sqrt(dx * dx + dy * dy);
  //     return "M" + d.source.x + "," + d.source.y + "A" + dr + "," + dr + " 0 0,1 " + d.target.x + "," + d.target.y;
  //   });
  // console.log(link)
  link.attr("d", drawCurve2);
  node.attr('transform', (d) => `translate(${d.x},${d.y})`);
}

function addNode(node) {
  if (!node) node = {};
  if (!node.id) node.id = uuidv4();
  if (!node.instanceId) node.instanceId = uuidv4();
  if (!node.name) node.name = { "en": "New Node", "fr": "Nouveau nœud" };
  if (!node.description) node.descripion = { "en": "New Node Description", "fr": "Description du nouveau nœud" }
  if (!node.type) node.type = 'default';
  if (!node.radius) node.radius = 20;
  if (!node.x) node.x = 0;
  if (!node.y) node.y = 0;
  if (!node.vx) node.vx = 0;
  if (!node.vy) node.vy = 0;
  dataset.nodes.push(node);
  selectNode(node);
  loadNodes();
  restartSimulation();
}

function deleteSelectedNode() {

  for (var a = dataset.links.length - 1; a >= 0; a--) {
    console.log(dataset.links[a].source.selected)
    if (dataset.links[a].source?.selected || dataset.links[a].target?.selected) dataset.links.splice(a, 1);
  }

  for (var a = dataset.nodes.length - 1; a >= 0; a--) {
    if (dataset.nodes[a].selected) dataset.nodes.splice(a, 1);
  }
  selectedNodes.value.splice(0, selectedNodes.value.length)

  loadLinks();
  loadNodes();
  restartSimulation();
}

function updateNode(node) {

  loadNodes();
  restartSimulation();
}


function addLink() {
  if (selectedNodes.value.length >= 2) {
    for (var a = 1; a < selectedNodes.value.length; a++) {
      var link = {};
      link.id = uuidv4();
      link.instanceId = uuidv4();
      link.name = { "en": "New Link", "fr": "Nouveau lien" };
      link.descripion = { "en": "New Link Description", "fr": "Description du nouveau lien" }
      link.source = selectedNodes.value[0].id;
      link.target = selectedNodes.value[a].id;
      link.valence = {};
      console.log(link)
      dataset.links.push(link);
    }
    loadLinks();
    restartSimulation();
  }
}

function selectNode(node) {
  currentNode.value = node;

  var matchNode = selectedNodes.value.findIndex((n) => { return n.id == node.id })
  if (matchNode == -1) {
    node.selected = true;
    node.selectedIndex = selectedNodes.value.length;
    node.fixed = true;
    selectedNodes.value.push(node);
    loadNodes();
    loadLinks();
    restartSimulation();
  }
}

function toggleFixed() {
  if (selectedNodes.value.length) {
    //Release selected fixed
    var matchNode = selectedNodes.value.filter((n) => { return n.fixed })
    matchNode.forEach((node) => {
      node.fixed = false;
      node.fx = null;
      node.fy = null;
    })
  }
  else {
    //Release all fixed
    var matchNode = dataset.nodes.filter((n) => { return n.fixed })
    matchNode.forEach((node) => {
      node.fixed = false;
      node.fx = null;
      node.fy = null;
    })
  }
  loadNodes();
  restartSimulation();
}

function deselectNode(node) {
  currentNode.value = null;
  node.selected = false;
  var matchNode = selectedNodes.value.findIndex((n) => { return n.id == node.id })
  if (matchNode > -1) selectedNodes.value.splice(matchNode, 1)
  loadLinks();
  loadNodes();
  restartSimulation();
}

function drawCurve2(d) {
  var tightness = -3.0;
  if (d.type == "straight")
    tightness = 1000;

  // Places the control point for the Bezier on the bisection of the
  // segment between the source and target points, at a distance
  // equal to half the distance between the points.
  var dx = d.target.x - d.source.x;
  var dy = d.target.y - d.source.y;
  var dr = Math.sqrt(dx * dx + dy * dy);
  var qx = d.source.x + dx / 2.0 - dy / tightness;
  var qy = d.source.y + dy / 2.0 + dx / tightness;

  // Calculates the segment from the control point Q to the target
  // to use it as a direction to wich it will move "node_size" back
  // from the end point, to finish the edge aprox at the edge of the
  // node. Note there will be an angular error due to the segment not
  // having the same direction as the curve at that point.
  var dqx = d.target.x - qx;
  var dqy = d.target.y - qy;
  var qr = Math.sqrt(dqx * dqx + dqy * dqy);

  var offset = d.target.radius + 5;
  var tx = d.target.x - dqx / qr * offset;
  var ty = d.target.y - dqy / qr * offset;

  return "M" + d.source.x + "," + d.source.y + "Q" + qx + "," + qy
    + " " + tx + "," + ty;  // to "node_size" pixels before
  //+ " " + d.target.x + "," + d.target.y; // til target

}

function downloadJson(payload) {
  var dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(payload));
  var downloadAnchorNode = document.createElement('a');
  downloadAnchorNode.setAttribute("href", dataStr);
  downloadAnchorNode.setAttribute("download", "relationships.json");
  document.body.appendChild(downloadAnchorNode); // required for firefox
  downloadAnchorNode.click();
  downloadAnchorNode.remove();
}

function downloadSvg(id) {
  //Set the bounds for the Export
  var bounds = svg.node().getBBox();
  console.log(bounds)
  svg.attr("viewBox", `${bounds.x - 5} ${bounds.y - 5} ${bounds.width + 10} ${bounds.height + 10}`)
  svg.attr("width", bounds.width + 10)
    .attr("height", bounds.height + 10)

  //Export out the SVG
  var svgData = document.getElementById(id).children[0].outerHTML;
  var svgBlob = new Blob(['<?xml version="1.0" encoding="UTF-8" standalone="no"?>', svgData], { type: "image/svg+xml;charset=utf-8" });
  var svgUrl = URL.createObjectURL(svgBlob);
  var downloadLink = document.createElement("a");
  downloadLink.href = svgUrl;
  downloadLink.download = "Symaiosis Diagram.svg";
  document.body.appendChild(downloadLink);
  downloadLink.click();
  document.body.removeChild(downloadLink);

  //Reset the bounds after export
  svg.attr("width", width)
    .attr("height", height)


}

</script>
 
<!-- Important to ensure your styles are scoped for your components to not interfere with global styling -->
<style scoped>
.d3GraphCreatorSvgParent {
  height: 500px;
  min-height: 500px;
  border-radius: 5px;
  border-width: 3px;
  border-color: grey;
  border-style: solid;
}

.d3GraphCreatorMenu {
  height: 40px;
}
.d3GraphCreatorMenuIcon {
  font-size: 1.5em;
  padding: 5px;
  height: 20px;
  margin-top: auto;
  margin-bottom: auto;
}

.d3GraphCreatorMenuIcon:hover {
  opacity: 0.8;
}
</style> 

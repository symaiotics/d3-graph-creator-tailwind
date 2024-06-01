<template>
  <div class="container mx-auto px-4">

    <p v-if="props?.title?.length" class="text-3xl text-gray-700 font-bold" :class="{ 'text-gray-300': props.darkMode }">
      {{ props.title }}
    </p>
    <p v-if="props?.description?.length" class="text-gray-500 text-lg mb-2" :class="{ 'text-white': props.darkMode }">
      {{ props.description }}
    </p>

    <div v-if="props?.showControls">
      <button class="btn btn-blue" @click="addNode()">Add</button>
      <button class="btn btn-blue" @click="deleteSelectedNode()">Delete</button>
      <button class="btn btn-blue" @click="pinAll()">Pin All</button>
      <button class="btn btn-blue" @click="toggleFixed()">Unpin</button>
      <button class="btn btn-blue" @click="addLink()">Link</button>
      <button class="btn btn-blue" @click="downloadSvg(parentId)">Download SVG</button>
      <button class="btn btn-blue" @click="downloadJson(dataset)">Download JSON</button>
    </div>
    <div :id="parentId" class="d3GraphCreatorSvgParent" :class="{ 'bg-slate-800': props.darkMode }"></div>

    <table v-if="props?.showTable" class="w-full text-sm text-left text-gray-500 dark:text-gray-400">
      <thead class="text-xs text-gray-700 uppercase bg-gray-50 dark:bg-gray-700 dark:text-gray-400">
        <tr>
          <th scope="col" class="px-6 py-3">
            Index
          </th>
          <th scope="col" class="px-6 py-3">
            Name
          </th>
          <th scope="col" class="px-6 py-3">
            Radius
          </th>
        </tr>
      </thead>
      <tbody>
        <template v-for="(node, index) in selectedNodes" :key="'selectedNode' + index">
          <tr class="bg-gray-200 border-b dark:bg-gray-900 dark:border-gray-700" :class="{ 'bg-white': index % 2 == 0 }">
            <td class="px-6 py-4">{{ index }}</td>
            <td class="px-6 py-4">
              <input class="inputField" @change="updateNode(node)" @v-on:keyup.enter="updateNode(node)"
                v-model="node.name.en" type="text" />
            </td>
            <td class="px-6 py-4">
              <!-- <input @input="updateNode(node)" type="range" min="10" max="100"
                class="w-full h-2 bg-gray-400 rounded-lg appearance-none cursor-pointer dark:bg-gray-700"
                v-model="node.radius"/> -->
                <!-- Alternate input box-->
                <input class="inputField" @change="updateNode(node)" @v-on:keyup.enter="updateNode(node)" v-model="node.radius" type="number" />
            </td>
          </tr>
        </template>
      </tbody>
    </table>
  </div>
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
  'addLinks', //array of added nodes
  'deleteNodes', //array of deleted nodes
  
  'selectNodes',  //array of last selected nodes
  'selectedNodes',  //array of all selected nodes
  'deselectNodes', //array of deselected nodes
  'updateNodes', //array of nodes updated, just a subset
])

//These props allow us to customize the 
const props = defineProps({
  title: { type: String, default: "D3 Graph Creator" },
  description: { type: String, default: "An interactive graph interface." },

  height: { type: Number, default: 500 },
  glow: { type: Boolean, default: false },
  darkMode: { type: Boolean, default: false },
  showControls: { type: Boolean, default: true },
  showTable: { type: Boolean, default: true },
  useGravity: { type: Boolean, default: true },

  nodesSource: {
    type: Array, default: [
      { "id": "jean-michel", "name": { "en": "Jean-Michel", "fr": "" }, "group": 1, "radius": 30 , fx:500, fy:250, selected:true, selectedIndex:0},
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
  height = props.height;
  svg = d3.select(parentDiv).append("svg");
  svg.attr("width", width)
    .attr("height", height)
    .attr("style", "min-height:500px;font-family:sans-serif")
    .attr("xmlns", "http://www.w3.org/2000/svg")
    .attr("xmlns:svg", "http://www.w3.org/2000/svg")
    .attr("xmlns:xlink", "http://www.w3.org/1999/xlink");

  //Establish the container to permit zoom and pan
  container = svg.append("g");

  //Right click
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
    .attr("stroke", () => { return props.darkMode ? '#ccc' : "#555" })
    .attr("stroke-width", .5)
    .attr("fill", () => { return props.darkMode ? '#ddd' : "#444" })
    ;         // .attr('transform', 'rotate(-10)')

  //Filter for the outside glow for the container
  var filter = defs.append("filter")
    .attr("id", "glow");
  filter.append("feGaussianBlur")
    .attr("stdDeviation", "5")
    .attr("result", "coloredBlur");
  var feMerge = filter.append("feMerge");
  feMerge.append("feMergeNode")
    .attr("in", "coloredBlur");
  feMerge.append("feMergeNode")
    .attr("in", "SourceGraphic");

  // var gooey = defs.append("filter")
  //   //use a unique id to reference again later on
  //   .attr("id", "gooeyCodeFilter");

  // //Append multiple "pieces" to the filter
  // gooey.append("feGaussianBlur")
  //   .attr("in", "SourceGraphic")
  //   .attr("stdDeviation", "20")
  //   .attr("color-interpolation-filters", "sRGB")
  //   .attr("result", "blur");
  // gooey.append("feColorMatrix")
  //   //the class used later to transition the gooey effect
  //   .attr("class", "blurValues")
  //   .attr("in", "blur")
  //   .attr("mode", "matrix")
  //   .attr("values", "1 0 0 0 0  0 1 0 0 0  0 0 1 0 0  0 0 0 19 -9")
  //   .attr("result", "gooey");

  // //If you want the end shapes to be exactly the same size as without
  // //the filter add the feBlend below. However this will result in a
  // //less beautiful gooey effect
  // gooey.append("feBlend")
  //   .attr("in", "SourceGraphic")
  //   .attr("in2", "gooey");

  // //Instead of the feBlend, you can do feComposite. This will also
  // //place a sharp image on top. But it will result in smaller circles
  // gooey.append("feComposite") 
  // // feBlend
  //     .attr("in","SourceGraphic")
  //     .attr("in2","gooey")
  //     .attr("operator","atop");

  //Apply the filter to the group element of all the circles
  // var circleWrapper = svg.append("g")
  ;

  //Configure the Container and SVG
  container = svg.append('g');
  container.append('g').attr('class', 'links').style("filter", "url(#filter)");;
  container.append('g').attr('class', 'nodes').style("filter", "url(#filter)");

  simulation = d3.forceSimulation(dataset.nodes);
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
    .force('collide', d3.forceCollide());

  if(props.useGravity)
  {
    simulation
    .force('center', d3.forceCenter(width / 2, height / 2))
    .force('forceX', d3.forceX().x(width / 2).strength(0.025))
    .force('forceY', d3.forceY().y(height / 2).strength(0.025));
  }

  simulation.force('link')
    .id((d) => d.id)
    .distance(function (d) {
      return parseInt(d.radius) ? parseInt(d.radius) * 50 : 200
    })
    .iterations(.1)
    .strength(.1);

  // simulation.force('collide').radius(function (d) { return parseInt(d.radius) ? parseInt(d.radius) : 2 }).strength(1);
  simulation.force('charge').strength(-200);
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
    .attr("stroke", () => { return props.darkMode ? '#aaa' : "#777" })
    .attr("stroke-width", 3)
    .attr("marker-end", "url(#end)");

  link = linkEnter.merge(link);

  //Handle the linkTypes
  container.select('.links')
    .selectAll('.linkType').remove();

  var linkType = container.select('.links')
    .selectAll('.linkType')
    .data(dataset.links, (d) => d.id)

  var linkEnter = linkType.enter()
    .append("text")
    .attr("class", 'linkType')
    .attr("white-space", "pre")
    .attr("font-size", ".8em")
    .attr("font-weight", "600")
    .attr('dy', "-10px")
    .append("textPath")
    .attr("xlink:href", (d, i) => '#' + parentId.value + 'textarc' + i)
    .attr('startOffset', '60%')
    .style("text-anchor", "middle")
    .text(function (d) {
      if (d.source.selected || d.target.selected) return d?.type?.en ? d.type.en + " \u003E" : "[ ]";
      else return "-";
    })
    .attr("fill", //make fill invisible if not selected
      function (d) {
        return (d?.type?.en && (d.source.selected && d.target.selected)) ? (props.darkMode ? '#fff' : "#444") : "none"; //Make this an OR to show any selected relationships
      }

    )


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
          .attr("r", d => parseInt(d.radius) ? parseInt(d.radius) : 20)
          .attr("fill", (d) => {
            if (!d.selected) return '#fff';
            if (d.selected && d.selectedIndex == 0) return '#ffc';
            if (d.selected && d.selectedIndex > 0) return '#fda';
          })
        // .style("filter", "url(#glow)");

        //Underlying text
        nodes.append("text")
          .attr("x", d => parseInt(d.radius) + 2 ? parseInt(d.radius) + 8 : 24)
          .attr("y", "0.31em")
          .text(d => d.name.en)
          .attr("fill", () => { return props.darkMode ? '#fff' : "#334" })
          .attr("stroke-width", 5)
          .attr("stroke", () => { return props.darkMode ? '#334' : "#fff" })
          .attr("font-weight", "600")
        // .style("filter", "url(#glow)");

        //Overlapping tax
        nodes.append("text")
          .attr("x", d => parseInt(d.radius) ? parseInt(d.radius) + 8 : 24)
          .attr("y", "0.31em")
          .text(d => d.name.en)
          .attr("fill", () => { return props.darkMode ? '#fff' : "#334" })
          .attr("font-weight", "600")

          // .style("filter", "url(#glow)");
          ;

        return nodes;
      },
    );

  emit('nodes', dataset.nodes)
  // console.log(dataset.nodes)
  if (props.glow) container.style("filter", "url(#glow)")

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
    console.log(d)
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
  if (!node.x) node.x = width/2;
  if (!node.y) node.y = height/2;
  if (!node.vx) node.vx = 0;
  if (!node.vy) node.vy = 0;
  dataset.nodes.push(node);
  selectNode(node);
  loadLinks();
  loadNodes();
  restartSimulation();
  emit('addNodes', node)
}

function deleteSelectedNode() {

  var deletedNodes = []
  for (var a = dataset.links.length - 1; a >= 0; a--) {
    console.log(dataset.links[a].source.selected)
    if (dataset.links[a].source?.selected || dataset.links[a].target?.selected) dataset.links.splice(a, 1);
  }

  for (var a = dataset.nodes.length - 1; a >= 0; a--) {
    if (dataset.nodes[a].selected) 
    {
      deletedNodes.push( dataset.nodes[a])
      dataset.nodes.splice(a, 1);
    }
  }
  selectedNodes.value.splice(0, selectedNodes.value.length)

  loadLinks();
  loadNodes();
  restartSimulation();
  emit('deleteNodes',deletedNodes)


}

function updateNode(node) {
  // console.log(node)
  loadNodes();
  restartSimulation();
  emit('updateNodes', [node])
}


function addLink() {
  if (selectedNodes.value.length >= 2) {
    var addedLinks = []
    for (var a = 1; a < selectedNodes.value.length; a++) {
      var link = {};
      link.id = uuidv4();
      link.instanceId = uuidv4();
      link.name = { "en": "New Link", "fr": "Nouveau lien" };
      link.descripion = { "en": "New Link Description", "fr": "Description du nouveau lien" }
      link.source = selectedNodes.value[0];
      link.target = selectedNodes.value[a];
      link.type = { "en": "Associate of", "fr": "Associé de" };
      dataset.links.push(link);
      addedLinks.push(link)
    }
    loadLinks();
    restartSimulation();
    emit('addLinks', addedLinks)
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
    loadLinks();
    loadNodes();
    restartSimulation();
    emit('selectNodes', [node])
    emit('selectedNodes', selectedNodes.value)
  }
}

function pinAll() {
  dataset.nodes.forEach((node) => {
    node.fixed = true;
    node.fx = node.x;
    node.fy = node.y;
  })
  loadNodes();
  restartSimulation();
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
  emit('deselectNodes', [node])
  emit('selectedNodes', selectedNodes.value)
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

  var offset = parseInt(d.target.radius) + 5;
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
.inputField {
  @apply bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500
}

.btn {
  @apply font-bold py-2 px-4 rounded mr-1;
}

.btn-blue {
  @apply bg-blue-500 text-white;
}

.btn-blue:hover {
  @apply bg-blue-700;
}


.d3GraphCreatorSvgParent {
  border-radius: 5px;
  border-width: 3px;
  border-color: grey;
  border-style: solid;
  margin-top: 10px;
  padding-bottom: 0px;
}
</style> 

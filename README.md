# @Symaiotics D3 Graph Creator
This library brings together a featured interface for creating and editing network graph diagrams using the powerful D3.js library.

## Getting Started

Install this into your Vue 3 project.
`npm i @symaiotics/d3-graph-creator-tailwind`

And then register the component within your page or component in Vue.
It is important to bring in the main.css for this component as well.

```
<script setup>
//Import the component and the style sheet used by that component to get its custom styling
import D3GC from '@symaiotics/d3-graph-creator-tailwind'
import '@symaiotics/d3-graph-creator-tailwind/dist/style.css';
</script>

<template>
    <D3GC title="Main Title" :linksSource="links" :nodesSource="nodes" :dark="true" />
</template>
```

## Node and Link Models
These objects are being expanded as the library advances. 

nodeSource array
```

let nodes = ref([
  { id: "building", name: { en: "Building", fr: "Bâtiment" }, group: 1, radius: 30 },
  { id: "beautiful", name: { en: "Beautiful", fr: "Beau" }, group: 1, radius: 30 },
  { id: "nature", name: { en: "Nature", fr: "Nature" }, group: 1, radius: 30 },
  
  { id: "structure", name: { en: "Structure", fr: "Structure" }, group: 1, radius: 20 },
  { id: "edifice", name: { en: "Edifice", fr: "Édifice" }, group: 1, radius: 20 },
  { id: "construction", name: { en: "Construction", fr: "Construction" }, group: 1, radius: 20 },
  
  { id: "attractive", name: { en: "Attractive", fr: "Attrayant" }, group: 1, radius: 20 },
  { id: "pretty", name: { en: "Pretty", fr: "Joli" }, group: 1, radius: 20 },
  { id: "gorgeous", name: { en: "Gorgeous", fr: "Superbe" }, group: 1, radius: 20 },
  
  { id: "environment", name: { en: "Environment", fr: "Environnement" }, group: 1, radius: 20 },
  { id: "wildlife", name: { en: "Wildlife", fr: "Vie sauvage" }, group: 1, radius: 20 },
  { id: "ecosystem", name: { en: "Ecosystem", fr: "Écosystème" }, group: 1, radius: 20 }
]);


```

To be added: type, border, shape...

linkSource array
```
let links = ref([
  { source: "structure", target: "building", value: 1, type: { en: "Synonym of", fr: "Synonyme de" } },
  { source: "edifice", target: "building", value: 1, type: { en: "Synonym of", fr: "Synonyme de" } },
  { source: "construction", target: "building", value: 1, type: { en: "Synonym of", fr: "Synonyme de" } },
  
  { source: "attractive", target: "beautiful", value: 1, type: { en: "Synonym of", fr: "Synonyme de" } },
  { source: "pretty", target: "beautiful", value: 1, type: { en: "Synonym of", fr: "Synonyme de" } },
  { source: "gorgeous", target: "beautiful", value: 1, type: { en: "Synonym of", fr: "Synonyme de" } },
  
  { source: "environment", target: "nature", value: 1, type: { en: "Synonym of", fr: "Synonyme de" } },
  { source: "wildlife", target: "nature", value: 1, type: { en: "Synonym of", fr: "Synonyme de" } },
  { source: "ecosystem", target: "nature", value: 1, type: { en: "Synonym of", fr: "Synonyme de" } }
]);
  
```

Width is adopted by the size of the container. 
## D3.js
This library uses the latest version of D3 and implements a variety of add, delete, link, pin and export tools to make graph creation fast and easy.


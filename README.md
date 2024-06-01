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
  <D3GC title="Main Title" />
</template>
```

## Node and Link Models
These objects are being expanded as the library advances. 

Node object
```{ 
  "id": "jean-michel", 
  "name": { "en": "Jean-Michel", "fr": "" }, 
  "group": 1, 
  "radius": 30 
  }```

To be added: type, border, shape...

Link object
```{ 
  "source": "jean-michel", 
  "target": "alexandra", 
  "value": 1, 
  "type": { "en": "Brother of", "fr": "Frère de" 
  }```

To be added: width, colour, style...

## D3.js
This library uses the latest version of D3 and implements a variety of add, delete, link, pin and export tools to make graph creation fast and easy.


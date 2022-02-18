# @Symaiotics D3 Graph Creator
This library brings together a featured interface for creating and editing network graph diagrams using the powerful D3.js library.

## Getting Started

Install this into your Vue 3 project.
`npm i @symaiotics/d3-graph-creator-gcweb`

And then register the component within your page or component in Vue.
It is important to bring in the style.css for this component as well.

`
<script setup>
//Import the component and the style sheet used by that component to get its custom styling
import D3GC from '@symaiotics/d3-graph-creator-gcweb'
import '@symaiotics/d3-graph-creator-gcweb/dist/style.css';
</script>

<template>
  <D3GC title="Main Title" />
</template>
`

## D3.js
This library uses the latest version of D3 and implements a variety of add, delete, link, pin and export tools to make graph creation fast and easy.

## Styling
This library uses the Canada.ca GCWeb Theme (a customized instance of Bootstrap 3)
https://wet-boew.github.io/GCWeb/index-en.html


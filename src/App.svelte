<svelte:head>
  {@html ashes}
</svelte:head>
<div class="grid" style="grid-template-columns:auto 1fr">
<ListBox class="card p-4 h-screen overflow-y-scroll">
  {#each materials as material,index }
  <ListBoxItem bind:group={selectedMaterial} name="medium" value={index}>{material.name}</ListBoxItem>
  {/each}
</ListBox>
<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<div class="h-screen">
    
    <div bind:this={tabs}>
      <TabGroup >
        <Tab bind:group={selectedShader} name="fragment" value='fragmentShader'>Fragment shader</Tab>
        <Tab bind:group={selectedShader} name="Vertex" value='vertexShader'>Vertex shader</Tab>
        <button
          type="button"
          class="btn variant-filled-surface ml-auto"
          on:click={()=>unpack = !unpack}
        >{unpack?'Fold':'Unfold'} all includes</button>
      </TabGroup>
    </div>
  
  <div on:click|stopPropagation="{click}" class="overflow-y-scroll" style={`height: calc(100% - ${tabs?tabs.clientHeight:0}px)`}>
    <Highlight code="{codeReplaced}" let:highlighted language="{glsl}">
       <LineNumbers {highlighted} highlightedLines="{highlightedLines}" wrapLines></LineNumbers>
    </Highlight>
  </div>
</div>
</div>



<script lang="ts">
import { ListBox, ListBoxItem, Tab, TabGroup } from '@skeletonlabs/skeleton';
import Highlight, { LineNumbers } from "svelte-highlight";
import glsl from "svelte-highlight/languages/glsl";
import ashes from "svelte-highlight/styles/ashes";
import type { MouseEventHandler } from 'svelte/elements';
import { LineBasicMaterial, LineDashedMaterial, Material, Mesh, MeshBasicMaterial, MeshDepthMaterial, MeshLambertMaterial, MeshMatcapMaterial, MeshNormalMaterial, MeshPhongMaterial, MeshPhysicalMaterial, MeshStandardMaterial, MeshToonMaterial, PerspectiveCamera, PointsMaterial, Scene, ShaderChunk, ShaderMaterial, SphereGeometry, WebGLRenderer } from 'three';
import type { WebGLProgramParametersWithUniforms } from 'three/src/Three.js';
import type { Constructor } from 'type-fest';
import "./app.css";
let selectedMaterial = 0
let tabs:HTMLElement|null = null
const materials:Constructor<Material>[] = [
  LineBasicMaterial,
  LineDashedMaterial,
  MeshBasicMaterial,
  MeshDepthMaterial,
  MeshLambertMaterial,
  MeshMatcapMaterial,
  MeshNormalMaterial,
  MeshPhongMaterial,
  MeshPhysicalMaterial,
  MeshStandardMaterial,
  MeshToonMaterial,
  PointsMaterial,
  ShaderMaterial,
]

const getMesh = (Mat:Constructor<Material>)=>{
  switch (Mat){
    default: return new Mesh(new SphereGeometry(1),new Mat())
  }
}

let unpack = false

$:{
  let Mat = materials[selectedMaterial]
  
  class LoggingMaterial extends Mat {
    onBeforeCompile(parameters: WebGLProgramParametersWithUniforms, _renderer: WebGLRenderer): void {
        fragmentShader = parameters.fragmentShader
        vertexShader = parameters.vertexShader
    }
  }
  scene.add(getMesh(LoggingMaterial))
  renderer.render(scene,camera)
}

let selectedShader:'fragmentShader'|'vertexShader' = 'fragmentShader'

const renderer = new WebGLRenderer()
const scene = new Scene()
const camera = new PerspectiveCamera()
let fragmentShader = ''
let vertexShader = ''

let includes = new Set<string>()

const includeRegex = /#include\s*<([^>]+)>/

const click:MouseEventHandler<HTMLElement> = (event)=>{
  if(event.target && 'textContent' in event.target){
    const text = event.target.textContent as string
    if(text.match(includeRegex)){
      if(includes.has(text.replaceAll('\t',''))){
          includes.delete(text)
      }else {
        includes.add(text.replaceAll('\t',''))
      }
      includes = includes
  }
  
  }
}
let codeReplaced = fragmentShader

$: {
codeReplaced = {fragmentShader, vertexShader}[selectedShader]
const allIncludes = fragmentShader.split('\n')
  .filter(x=>x.match(includeRegex))
  .map(x=>x.replace('\t',''))
const selectedIncludes = unpack?allIncludes:includes
for(const include of selectedIncludes){
    const chunk = include.match(includeRegex)
    if(chunk){
      const chunkName = chunk[1] as keyof (typeof ShaderChunk)
      const padding = fragmentShader.split('\n').find(x=>x.includes(include))?.match(/\t/g)?.length??0
      const newChunk = ShaderChunk[chunkName].split('\n').map(l=>new Array(padding).fill('\t').join('')+l).join('\n')

      codeReplaced = codeReplaced.replaceAll(include,include+'\n'+newChunk)
    }

}}
let highlightedLines:number[] = []
$: {
  highlightedLines = codeReplaced.split('\n').reduce((acc,l,i)=>{
    return l.match(includeRegex)?[...acc,i]:acc
  },[] as number[])
}</script><style></style>
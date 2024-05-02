<script lang="ts">
    import type { WebGLProgramParametersWithUniforms } from 'three/src/Three.js';
  import {WebGLRenderer, Scene, PerspectiveCamera, MeshBasicMaterial, Mesh, SphereGeometry, ShaderChunk} from 'three'
  import Highlight, { LineNumbers } from "svelte-highlight";
  import glsl from "svelte-highlight/languages/glsl";
  import ashes from "svelte-highlight/styles/ashes";
  import type { MouseEventHandler } from 'svelte/elements';
  const renderer = new WebGLRenderer()
  const scene = new Scene()
  const camera = new PerspectiveCamera()
  let fragmentShader = ''
  class Basic extends MeshBasicMaterial {
    onBeforeCompile(parameters: WebGLProgramParametersWithUniforms, renderer: WebGLRenderer): void {
        fragmentShader = parameters.fragmentShader
    }
  }
  scene.add(new Mesh(new SphereGeometry(1),new Basic()))
  let includes = new Set<string>()
  renderer.render(scene,camera)
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
		console.log(includes, text.match(includeRegex))
	}
    
    }
  }
  let codeReplaced = fragmentShader
  $: {
  codeReplaced = fragmentShader
  for(const include of includes){
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
  }

</script>
<svelte:head>
  {@html ashes}
</svelte:head>
<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<div on:click|stopPropagation={click}> 
  <Highlight 
    code={codeReplaced}
    let:highlighted
    language={glsl}
  >
     <LineNumbers {highlighted} highlightedLines={highlightedLines} wrapLines  />
  </Highlight>
</div>
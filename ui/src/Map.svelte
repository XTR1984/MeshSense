<script context="module" lang="ts">
  import { writable } from 'svelte/store'
  export let expandedMap = writable(false)
  export let setPositionMode = writable(false)
  import { generateHexer } from '@bdancer/icon-gaga'
  import Style from 'ol/style/Style'
  import Stroke from 'ol/style/Stroke'

  export function getSvgUri(name: string) {
    const hexId = parseInt(name).toString(16).padStart(8, '0')
    const colorId = hexId.slice(-6)
    return 'data:image/svg+xml;utf8,' + encodeURIComponent(generateHexer({ 
      name,
      borderColor: `#${colorId}`
    }))
  }

  export function getIconURL(node: NodeInfo) {
    if (node.position?.latitudeI) {
      if (node?.position?.altitude > 2743) return `${import.meta.env.VITE_PATH || ''}/airplane.svg`
      else return getSvgUri(String(node.num))
      // else return `https://icongaga-api.bytedancer.workers.dev/api/genHexer?name=${node.num}`
    } else {
      return `${import.meta.env.VITE_PATH || ''}/circle-help.svg`
    }
  }
</script>

<script lang="ts">
  import { connectionStatus, myNodeNum, version, type NodeInfo, routeDisplayMode } from 'api/src/vars'
  import { filteredNodes, isInactive, nodeVisibilityMode } from './Nodes.svelte'
  import Card from './lib/Card.svelte'
  import OpenLayersMap from './lib/OpenLayersMap.svelte'
  import { getCoordinates, getNodeById, getNodeName, getNodeNameById, setPosition } from './lib/util'
  import { showConfigModal, showPage } from './SettingsModal.svelte'
  import { newsVisible } from './News.svelte'

  export let ol: OpenLayersMap = undefined

  $: nodesWithCoords = $filteredNodes.filter((n) => !(n.position?.latitudeI == undefined || n.position?.latitudeI == 0) || n.approximatePosition)

  function setRouteMode(mode) {
    console.log('setRouteMode called with:', mode, 'current:', $routeDisplayMode)
    $routeDisplayMode = mode
    // Здесь можно вызвать перерисовку карты
    plotData();
  }

  function plotData() {
    let myNodeCoords = getCoordinates($myNodeNum)

    if ($routeDisplayMode === 'disable') {
      ol.removeLayers(['routes-forward','routes-back']);
    }


    if ($routeDisplayMode === 'all' || $routeDisplayMode === 'forward') {
    if ($routeDisplayMode === 'forward') ol.removeLayers('routes-back');
    ol.plotLines(
      'routes-forward',
      nodesWithCoords
        .filter((n) => (n.trace || n.hopsAway == 0) && $nodeVisibilityMode != 'inactive' && !n.trace?.route?.some((routeNodeId) => isInactive(getNodeById(routeNodeId))))
        .map((n) => {
          let list: any[] = [...(n.trace?.route?.map((traceNode) => getCoordinates(traceNode)) || []), getCoordinates(n)]
          if (myNodeCoords[0] && myNodeCoords[1]) list.unshift(myNodeCoords)
          return list.filter((coords) => !(coords[0] == 0 && coords[1] == 0))
        }),
        new Style({
          stroke: new Stroke({
          color: 'blue',
          width: 4
           })
        })        
    )
    }  

    if ($routeDisplayMode === 'all' || $routeDisplayMode === 'back') {
      if ($routeDisplayMode === 'back') ol.removeLayers('routes-forward');
      ol.plotLines(
    'routes-back',
    nodesWithCoords
      .filter((n) => n.trace?.snrBack?.length > 0 && $nodeVisibilityMode != 'inactive' && !n.trace?.routeBack?.some((routeNodeId) => isInactive(getNodeById(routeNodeId))))
      .map((n) => {
        let routeBackCopy = [...(n.trace?.routeBack || [])];
        let list: any[] = [...(routeBackCopy.reverse().map((traceNode) => getCoordinates(traceNode)) || []), getCoordinates(n)]
        if (myNodeCoords[0] && myNodeCoords[1]) list.unshift(myNodeCoords)
        let result = list.filter((coords) => !(coords[0] == 0 && coords[1] == 0));
        return result
      }),
      new Style({
      stroke: new Stroke({
        color: 'rgba(255, 87, 34, 0.8)',
        width: 4,
        lineDash: [5, 10] 
      })
      })     
    )    
    }

    ol.plotPoints(
      'nodes',
      nodesWithCoords.map((n) => {
        let [lon, lat] = getCoordinates(n)
        return {
          lat,
          lon,
          icon: getIconURL(n),
          description: getNodeName(n)
        }
      })
    )
  }

  $: {
    $myNodeNum, nodesWithCoords
    if (ol) {
      plotData()
    }
  }

  let modalPage = 'Settings'
</script>

<Card title="Map" {...$$restProps}>
  <h2 slot="title" class="rounded-t flex items-center gap-1">
    <div class="mr-2">Map</div>

    <div class="grow">
      <button on:click={() => ($expandedMap = !$expandedMap)} class="btn font-normal text-xs">{$expandedMap ? 'Collapse' : 'Expand'}</button>
          Routes
         <button 
          on:click={() => setRouteMode('all')}
          class="btn font-normal text-xs {$routeDisplayMode === 'all' ? 'btn-active' : ''}"
        >
          All
        </button>
        <button 
          on:click={() => setRouteMode('forward')}
          class="btn font-normal text-xs {$routeDisplayMode === 'forward' ? 'btn-active' : ''}"
        >
          Forward
        </button>
        <button 
          on:click={() => setRouteMode('back')}
          class="btn font-normal text-xs {$routeDisplayMode === 'back' ? 'btn-active' : ''}"
        >
          Back
        </button>
        <button 
          on:click={() => setRouteMode('disable')}
          class="btn font-normal text-xs {$routeDisplayMode === 'disable' ? 'btn-active' : ''}"
        >
          Disable
        </button>


    </div>
    <div class="text-xs text-white/50 pr-2">MeshSense {$version}</div>
    <a href="https://affirmatech.com" target="_blank" rel="noopener" class="text-xs text-white/50 pr-2 font-normal">by Affirmatech</a>
    <a title="Support MeshSense" target="_blank" rel="noopener" class="!text-rose-400 font-bold btn text-sm hover:brightness-110" href="https://purchase.affirmatech.com/?productId=MeshSenseDonation"
      >♥</a
    >
    <button title="What's New?" class="btn btn-sm h-6 grid place-content-center" on:click={() => newsVisible.set(true)}>📰</button>
    <a title="MeshSense Global Map" target="_blank" rel="noopener" class="font-bold btn text-sm hover:brightness-110" href="https://meshsense.affirmatech.com/">🌎</a>
    <button title="Settings" class="btn btn-sm h-6 font-normal grid place-content-center" on:click={() => showPage('Settings')}>⚙</button>
  </h2>
  <OpenLayersMap
    bind:this={ol}
    center={JSON.parse(localStorage.getItem('mapCenter')) ?? getCoordinates($myNodeNum)}
    zoom={JSON.parse(localStorage.getItem('mapZoom'))}
    onMove={(center, zoom) => {
      localStorage.setItem('mapCenter', JSON.stringify(center))
      localStorage.setItem('mapZoom', JSON.stringify(zoom))
    }}
    onClick={(latitude, longitude) => {
      if ($setPositionMode) {
        $setPositionMode = false
        setPosition(latitude, longitude)
      }
    }}
    onDarkModeToggle={plotData}
  ></OpenLayersMap>
  {#if $setPositionMode}
    <div class="absolute select-none top-10 left-10 bg-indigo-600/80 text-white p-3 py-1 rounded-lg">
      Click on a new position for {getNodeNameById($myNodeNum)}
      <button title="Cancel selecting a position" class="btn btn-sm ml-2 font-bold !text-red-200 !from-rose-500 !to-rose-800 rounded-full" on:click={() => ($setPositionMode = false)}>X</button>
    </div>
  {/if}
</Card>

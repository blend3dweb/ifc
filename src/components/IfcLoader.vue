<template>
  <div class="viewer-container">
    <div ref="container" class="viewer"></div>
  </div>
</template>

<script setup>
import * as WEBIFC from "web-ifc";
import * as BUI from "@thatopen/ui";
import Stats from "stats.js";
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";
import { onMounted, ref } from 'vue';

const container = ref(null);
const components = new OBC.Components();

onMounted(async () => {
  const worlds = components.get(OBC.Worlds);
  const world = worlds.create();

  world.scene = new OBC.SimpleScene(components);
  world.renderer = new OBC.SimpleRenderer(components, container.value);
  world.camera = new OBC.SimpleCamera(components);

  components.init();
  world.camera.controls.setLookAt(12, 6, 8, 0, 0, -10);
  world.scene.setup();

  const grids = components.get(OBC.Grids);
  grids.create(world);
  world.scene.three.background = null;

  const fragments = components.get(OBC.FragmentsManager);
  const fragmentIfcLoader = components.get(OBC.IfcLoader);

  await fragmentIfcLoader.setup();

  const excludedCats = [
    WEBIFC.IFCTENDONANCHOR,
    WEBIFC.IFCREINFORCINGBAR,
    WEBIFC.IFCREINFORCINGELEMENT,
  ];

  for (const cat of excludedCats) {
    fragmentIfcLoader.settings.excludedCategories.add(cat);
  }

  fragmentIfcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;

  // Setup Stats
  const stats = new Stats();
  stats.showPanel(2);
  document.body.append(stats.dom);
  stats.dom.style.left = "0px";
  stats.dom.style.zIndex = "unset";
  world.renderer.onBeforeUpdate.add(() => stats.begin());
  world.renderer.onAfterUpdate.add(() => stats.end());

  // Initialize UI
  BUI.Manager.init();

  // Setup UI panel
  const panel = BUI.Component.create(() => {
    return BUI.html`
      <bim-panel active label="IFC Loader Tutorial" class="options-menu">
        <bim-panel-section collapsed label="Controls">
          <bim-panel-section style="padding-top: 12px;">
            <bim-button label="Load IFC" @click="${loadIfc}"></bim-button>  
            <bim-button label="Export fragments" @click="${exportFragments}"></bim-button>  
            <bim-button label="Dispose fragments" @click="${disposeFragments}"></bim-button>
          </bim-panel-section>
        </bim-panel-section>
      </bim-panel>
    `;
  });

  document.body.append(panel);

  // Functions
  async function loadIfc() {
    const file = await fetch("https://thatopen.github.io/engine_components/resources/small.ifc");
    const data = await file.arrayBuffer();
    const buffer = new Uint8Array(data);
    const model = await fragmentIfcLoader.load(buffer);
    model.name = "example";
    world.scene.three.add(model);
  }

  function exportFragments() {
    if (!fragments.groups.size) return;
    const group = Array.from(fragments.groups.values())[0];
    const data = fragments.export(group);
    download(new File([new Blob([data])], "small.frag"));

    const properties = group.getLocalProperties();
    if (properties) {
      download(new File([JSON.stringify(properties)], "small.json"));
    }
  }

  function disposeFragments() {
    fragments.dispose();
  }

  function download(file) {
    const link = document.createElement("a");
    link.href = URL.createObjectURL(file);
    link.download = file.name;
    document.body.appendChild(link);
    link.click();
    link.remove();
  }

  fragments.onFragmentsLoaded.add((model) => {
    console.log(model);
  });

  const dimensions = new OBCF.EdgeMeasurement(components);
  dimensions.world = world;
  dimensions.enabled = true;
  dimensions.previewEnabled = true;
  dimensions.snapDistance = 1;

  


  container.value.addEventListener('mousemove', (event) => {
    if (dimensions && dimensions.enabled && dimensions.previewEnabled) {
        const rect = container.value.getBoundingClientRect();
        const x = ((event.clientX - rect.left) / container.value.clientWidth) * 2 - 1;
        const y = -((event.clientY - rect.top) / container.value.clientHeight) * 2 + 1;
        
        if (typeof dimensions.updatePreview === 'function') {
            dimensions.updatePreview({ x, y });
        }
    }
});

});
</script>

<style>
/* Global styles moved outside scoped block */
body {
    margin: 0;
    padding: 0;
    font-family: "Plus Jakarta Sans", sans-serif;
    overflow: hidden;
}

.full-screen {
    width: 100vw;
    height: 100vh;
    position: relative;
    overflow: hidden;
}

.options-menu {
    position: fixed;
    min-width: unset;
    top: 5px;
    right: 15px;
    max-height: calc(100vh - 10px);
}

.phone-menu-toggler {
    visibility: hidden;
}

@media (max-width: 480px) {
    .options-menu {
        visibility: hidden;
        bottom: 5px;
        left: 5px;
    }

    .options-menu-visible {
        visibility: visible;
    }

    .phone-menu-toggler {
        visibility: visible;
        position: fixed;
        top: 5px;
        right: 5px;
    }
}
</style>

<style scoped>
.viewer-container {
    width: 100%;
    height: 100%;
}
.viewer {
    width: 100%;
    height: 900px;
}
/* .measurement-label {
    background-color: white;
    padding: 5px;
    border-radius: 4px;
    border: 1px solid #666;
    pointer-events: none;
    font-family: sans-serif;
    font-size: 14px;
    font-weight: bold;
    color: #333;
}

.measurement-line {
    stroke: #666;
    stroke-width: 2px;
    stroke-dasharray: 5;
    pointer-events: none;
}

.measurement-endpoint {
    fill: white;
    stroke: #666;
    stroke-width: 2px;
    pointer-events: none;
} */
</style>

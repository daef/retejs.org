<template lang="pug">
.canva(ref="canva")
  client-only
    template(#placeholder)
      .ssr
        .caption Preview needs JS to be enabled
  .container.structures-editor(ref="container")
</template>

<script>
import {
  defineComponent, onMounted, onUnmounted, ref,
} from 'vue';
import {
  createEditor, structuresGraph, structuresSubGraph, methodApplicant, selectNode,
} from '../../shared/editor';

export default defineComponent({
  props: {
    id: {
      type: String,
      required: true,
    },
    pick: {
      type: String,
    },
  },
  setup(props) {
    const canva = ref(null);
    const container = ref(null);
    let resizeHandler = null;
    let instance = null;

    function concealNodes(editor, area, data) {
      const nodes = data.nodes().map((n) => n.id);
      const conns = data.connections().map((c) => c.id);

      editor.getNodes().forEach((n) => {
        const { classList: nodeClassList } = area.nodeViews.get(n.id).element;

        if (nodes.includes(n.id)) {
          nodeClassList.remove('conceal');
        } else {
          nodeClassList.add('conceal');
        }
      });
      editor.getConnections().forEach((c) => {
        const { classList: connectionClassList } = area.connectionViews.get(c.id).element;

        if (conns.includes(c.id)) {
          connectionClassList.remove('conceal');
        } else {
          connectionClassList.add('conceal');
        }
      });
    }

    onMounted(async () => {
      instance = await createEditor(container.value, {
        multiselect: false,
        order: false,
      });
      const { area, editor, resize } = instance;
      const method = methodApplicant(editor, props.id);

      const { nodes } = method.graphType === 'default'
        ? await structuresGraph(instance)
        : await structuresSubGraph(instance);
      resizeHandler = () => resize(canva.value.clientWidth, 900);

      area.area.setDragHandler(null);
      resizeHandler();

      area.addPipe((context) => {
        if (context.type === 'nodepicked') {
          concealNodes(editor, area, method.execute(context.data.id));
        }
        return context;
      });

      if (props.pick) {
        selectNode(nodes[props.pick].id, area);
      } else {
        concealNodes(editor, area, method.execute());
      }
      window.addEventListener('resize', resizeHandler);
    });

    onUnmounted(() => {
      if (instance) instance.area.destroy();
      if (resizeHandler) window.removeEventListener('resize', resizeHandler);
    });

    return {
      canva,
      container,
    };
  },
});
</script>

<style lang="sass" scoped>
@use 'sass:math'
@import '@/assets/styles/media.sass'

$ratio: math.div(15, 45)
$size: 35em

.canva
  max-width: $size
  max-height: $ratio * $size
  width: 65vw
  height: $ratio * 65vw
  +phone
    width: 85vw
    height: $ratio * 85vw
  .ssr
    display: flex
    justify-content: center
    align-items: center
    width: 100%
    height: 100%
</style>

<style lang="sass">
.structures-editor
  .conceal
    opacity: 0.6
</style>

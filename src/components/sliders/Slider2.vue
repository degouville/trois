<template>
  <Renderer ref="renderer" antialias mouse-move>
    <OrthographicCamera ref="camera" :position="{ z: 10 }" />
    <Scene ref="scene" />
  </Renderer>
</template>

<script>
import { Vector2 } from 'three';
import { gsap, Power4 } from 'gsap';
import { lerp, lerpv2 } from '../../tools.js';
import ZoomBlurImage from './ZoomBlurImage.js';
import useTextures from '../../use/useTextures.js';

export default {
  props: {
    images: Array,
    events: { type: Object, default: () => { return { wheel: true, click: true, keyup: true }; } },
  },
  setup() {
    const center = new Vector2();
    const loader = useTextures();

    return {
      loader,
      center,
      progress: 0,
      targetProgress: 0,
    };
  },
  mounted() {
    this.three = this.$refs.renderer.three;

    if (this.images.length < 2) {
      console.error('This slider needs at least 2 images.');
    } else {
      this.loader.loadTextures(this.images, this.init);
    }
  },
  unmounted() {
    this.loader.dispose();
    const domElement = this.three.renderer.domElement;
    domElement.removeEventListener('click', this.onClick);
    domElement.removeEventListener('wheel', this.onWheel);
    document.removeEventListener('keyup', this.onKeyup);
  },
  methods: {
    init() {
      this.initScene();
      gsap.fromTo(this.image1.uStrength,
        {
          value: -2,
        },
        {
          value: 0,
          duration: 2.5,
          ease: Power4.easeOut,
        }
      );

      const domElement = this.three.renderer.domElement;
      if (this.events.click) domElement.addEventListener('click', this.onClick);
      if (this.events.wheel) domElement.addEventListener('wheel', this.onWheel);
      if (this.events.keyup) document.addEventListener('keyup', this.onKeyup);
      this.three.onBeforeRender(this.animate);
      this.three.onAfterResize(this.onResize);
    },
    initScene() {
      const scene = this.$refs.scene.scene;

      this.image1 = new ZoomBlurImage(this.three);
      this.image1.setMap(this.loader.textures[0]);
      this.image2 = new ZoomBlurImage(this.three);
      this.image2.setMap(this.loader.textures[1]);
      this.setImagesProgress(0);

      scene.add(this.image1.mesh);
      scene.add(this.image2.mesh);
    },
    animate() {
      const { mouse } = this.three;
      this.center.copy(mouse).divideScalar(2).addScalar(0.5);
      lerpv2(this.image1.uCenter.value, this.center, 0.1);
      lerpv2(this.image2.uCenter.value, this.center, 0.1);

      this.updateProgress();
    },
    onResize() {
      this.image1.updateUV();
      this.image2.updateUV();
    },
    onWheel(e) {
      // e.preventDefault();
      if (e.deltaY > 0) {
        this.setTargetProgress(this.targetProgress + 1 / 20);
      } else {
        this.setTargetProgress(this.targetProgress - 1 / 20);
      }
    },
    onClick(e) {
      if (e.clientY < this.three.size.height / 2) {
        this.navPrevious();
      } else {
        this.navNext();
      }
    },
    onKeyup(e) {
      if (e.keyCode === 37 || e.keyCode === 38) {
        this.navPrevious();
      } else if (e.keyCode === 39 || e.keyCode === 40) {
        this.navNext();
      }
    },
    navNext() {
      if (Number.isInteger(this.targetProgress)) this.setTargetProgress(this.targetProgress + 1);
      else this.setTargetProgress(Math.ceil(this.targetProgress));
    },
    navPrevious() {
      if (Number.isInteger(this.targetProgress)) this.setTargetProgress(this.targetProgress - 1);
      else this.setTargetProgress(Math.floor(this.targetProgress));
    },
    setTargetProgress(value) {
      this.targetProgress = value;
      if (this.targetProgress < 0) {
        this.progress += this.images.length;
        this.targetProgress += this.images.length;
      }
    },
    updateProgress() {
      const progress1 = lerp(this.progress, this.targetProgress, 0.1);
      const pdiff = progress1 - this.progress;
      if (pdiff === 0) return;

      const p0 = this.progress % 1;
      const p1 = progress1 % 1;
      if ((pdiff > 0 && p1 < p0) || (pdiff < 0 && p0 < p1)) {
        const i = Math.floor(progress1) % this.images.length;
        const j = (i + 1) % this.images.length;
        this.image1.setMap(this.loader.textures[i]);
        this.image2.setMap(this.loader.textures[j]);
      }

      this.progress = progress1;
      this.setImagesProgress(this.progress % 1);
    },
    setImagesProgress(progress) {
      this.image1.uStrength.value = progress;
      this.image2.uStrength.value = -1 + progress;
    },
  },
};
</script>

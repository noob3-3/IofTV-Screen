<template>
  <div class="model-viewer">
    <div ref="modelContainer" class="canvas-container"></div>
  </div>
</template>

<script>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import * as THREE from 'three';
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import UtilVar from "@/config/UtilVar";
let baseUrl = UtilVar.baseUrl

export default {
  methods: {
    showWarning(message) {
      // 使用传入的字符串作为警告消息
      this.$Message.warning(message);
    }
},
  setup() {
    const modelContainer = ref(null);
    let scene, camera, renderer, controls, animationId, mesh, currentMd5;

    const initThreeJS = () => {
      const container = modelContainer.value;
      if (!container) {
        console.error('未找到容器元素');
        return;
      }

      // 初始化场景
      scene = new THREE.Scene();

      // 创建透视相机
      camera = new THREE.PerspectiveCamera(
          75,
          container.clientWidth / container.clientHeight,
          0.1,
          1000
      );

      // 初始化 WebGL 渲染器
      renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
      renderer.setSize(container.clientWidth, container.clientHeight);
      renderer.setClearColor(0x000000, 0);
      container.appendChild(renderer.domElement);

      // 初始化轨道控制器
      controls = new OrbitControls(camera, renderer.domElement);

      // 添加环境光和方向光
      const ambientLight = new THREE.AmbientLight(0x404040, 2);
      scene.add(ambientLight);

      const topLight = new THREE.DirectionalLight(0xffffff, 1);
      topLight.position.set(0, 10, 0).normalize();
      scene.add(topLight);

      // 初始化相机位置
      camera.position.set(5, 5, 5);
      controls.update();

      animate(); // 开始动画循环
    };

    const fetchModelData = async () => {
      try {
        const response = await fetch(baseUrl+'/get-model-info');
        const data = await response.json();

        if (data.md5 !== currentMd5) {
          currentMd5 = data.md5;
          loadModel(data.url);
        }
      } catch (error) {
        console.error('获取模型数据出错:', error);
      }
    };

    const loadModel = (url) => {
      const loader = new STLLoader();
      loader.load(
          url,
          (geometry) => {
            clearScene();
            const material = new THREE.MeshPhongMaterial({
              color: '#1ed5c5',
              side: THREE.DoubleSide,
              specular: '#918b84',
              shininess: 12,
              emissive: new THREE.Color('#1ed5c5'),
            });

            mesh = new THREE.Mesh(geometry, material);

            const scale = 0.7;
            mesh.scale.set(scale, scale, scale);

            const boundingBox = new THREE.Box3().setFromObject(mesh);
            const size = new THREE.Vector3();
            boundingBox.getSize(size);

            const distance = size.length() * 2;
            const modelCenter = new THREE.Vector3();
            boundingBox.getCenter(modelCenter);

            controls.target.copy(modelCenter);

            const cameraPosition = modelCenter.clone().add(new THREE.Vector3(1, 1, 1).normalize().multiplyScalar(distance));
            camera.position.copy(cameraPosition);
            camera.lookAt(modelCenter);

            scene.add(mesh);
          },
          undefined,
          (error) => {
            console.error('模型加载失败:', error);
          }
      );
    };

    const clearScene = () => {
      if (scene && mesh) {
        scene.remove(mesh);
        mesh.geometry.dispose();
        mesh.material.dispose();
        mesh = null;
      }
    };

    const animate = () => {
      if (!renderer || !scene || !camera || !controls) return;

      animationId = requestAnimationFrame(animate);

      if (mesh) {
        mesh.rotation.z += 0.01; // 调整旋转速度
      }

      controls.update();
      renderer.render(scene, camera);
    };

    onMounted(() => {
      initThreeJS();
      fetchModelData();
    });

    onBeforeUnmount(() => {
      if (animationId) {
        cancelAnimationFrame(animationId);
      }
      if (renderer) {
        renderer.dispose();
      }
      clearScene();
      scene = null;
      camera = null;
      controls = null;
      renderer = null;
    });

    return {
      modelContainer,
    };
  },
};
</script>

<style scoped>
.model-viewer {
  .canvas-container {
    width: 100%;
    height: 60vh;
  }
}
</style>

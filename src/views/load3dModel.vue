<template>
  <div class="model-com">
    <!--动态组件-->
    <div class="dynamic-components-div">
      <div id="container" v-loading="isLoading" element-loading-background="rgb(0, 0, 0,.8)"
        element-loading-text="数据加载中..."></div>
      <!--画布的容器-->
    </div>
    <!--切换盒子-->
    <div class="toggle-div">
      <el-button type="primary" v-for="item in modelList"
        :class="[item.modelUrl == cuModel.modelUrl ? 'thisButs' : 'buts']" :key="item.modelUrl"
        @click="toggleModel(item)">
        {{ item.text }}
      </el-button>
    </div>
  </div>
</template>

<script>
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls"; //鼠标控制器
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"; //模型加载器
import roomImg from "./img/panorama/kt.png"; //房间全景图
import beachImg from "./img/panorama/rtbh45hb44t.png"; //沙滩全景图
import right from "./img/space/right.jpg"; //天空盒子
import left from "./img/space/left.jpg";
import top from "./img/space/top.jpg";
import bottom from "./img/space/bottom.jpg";
import front from "./img/space/front.jpg";
import back from "./img/space/back.jpg";
let scene = null; //场景(在vue3中scene与mesh不能在data中定义，必须全局定义)
export default {
  components: {},
  data() {
    return {
      isLoading: "",
      modelList: [
        //模型列表
        {
          modelUrl: "model/ufo.glb",
          text: "ufo",
          cameraPar: [60, 0.01, 1000, 0, 30, 100],
          bg: "sky",
          modePar: [20, 10, -15, 100, 100, 100],
        },
        {
          modelUrl: "model/飞机.glb",
          text: "飞机",
          cameraPar: [60, 0.01, 1000, 0, 50, 100],
          bg: "sky",
          modePar: [0, -100, 0, 10, 10, 10],
        },
        {
          modelUrl: "model/人类头骨.glb", //模型路径
          text: "头骨",
          cameraPar: [60, 0.01, 10000, 0, 5, 10], //前三位为相机参数，后三位为相机位置
          bg: "room", //表示背景类型
          modePar: [2, 2, -3, 10, 10, 10], //前三位为模型位置，后三位为模型缩放
        },
        {
          modelUrl: "model/玫瑰.glb", //模型路径
          text: "玫瑰",
          cameraPar: [60, 0.01, 10000, 0, 0, 10], //前三位为相机参数，后三位为相机位置
          bg: "room", //表示背景类型
          modePar: [0, -3, 0, 100, 100, 100], //前三位为模型位置，后三位为模型缩放
        },
        {
          modelUrl: "model/鲨鱼.glb",
          text: "鲨鱼",
          cameraPar: [60, 0.01, 10000, 0, 0, 25],
          bg: "beach",
          modePar: [0, -5, 0, 50, 50, 50],
        },
      ],
      cuModel: "", //当前选中模型
      camera: null, //相机
      renderer: null, //渲染器
      mouseControls: null, //轨道控制
      pointLight: null, //点光源
      ambientLight: null, //环境光
      anId: null, //动画id
    };
  },
  mounted() {
    //默认选中第一个
    this.toggleModel(this.modelList[0]);
  },
  methods: {
    //点击切换按钮
    async toggleModel(item) {
      this.isLoading = true; //打开加载框
      if (this.camera) {
        //如果上一次存在场景则清除
        await this.destroyCom();
      }
      this.cuModel = item;
      this.init(item); //初始化场景
    },

    //销毁场景
    destroyCom() {
      // //根据动画id停止动画渲染
      cancelAnimationFrame(this.anId);
      // //强制失去上下文
      this.renderer.forceContextLoss();
      this.renderer.dispose();
      scene.clear();
      scene = null;
      this.camera = null;
      this.mouseControls = null;
      document.getElementById("container").innerHTML = null;
      this.renderer = null;
    },

    //初始化场景
    init(item) {
      scene = new THREE.Scene(); //新建场景
      let dom = document.getElementById("container");
      let width = dom.clientWidth; //场景宽度
      let height = dom.clientHeight; //场景高度
      // console.log(width,height);
      let k = width / height; //场景宽高比
      this.camera = new THREE.PerspectiveCamera(
        item.cameraPar[0],
        k,
        item.cameraPar[1],
        item.cameraPar[2]
      ); //透视相机
      this.camera.position.set(
        item.cameraPar[3],
        item.cameraPar[4],
        item.cameraPar[5]
      ); //设置相机位置
      //创建渲染器
      this.renderer = new THREE.WebGLRenderer({
        antialias: true, //抗锯齿
        alpha: true,
      });
      this.renderer.setSize(width, height); //设置渲染区域尺寸
      document
        .getElementById("container")
        .appendChild(this.renderer.domElement); //将画布添加到container中
      let axes = new THREE.AxesHelper(300); //坐标系(辅助开发)
      axes.rotation.x = 0.1;
      // scene.add(axes);//辅助开发坐标
      // this.createUniverse();//宇宙背景
      if (item.bg == "room") {
        this.createPanorama(roomImg); //球体全景
      }
      if (item.bg == "beach") {
        this.createPanorama(beachImg); //球体全景
      }
      if (item.bg == "sky" || item.bg == "") {
        this.createSkyBox(); //天空盒子
      }
      this.createOrbitControls();
      this.createLight();
      this.loadModel(item);
      this.repeatRender();
    },

    //创建天空盒子
    createSkyBox() {
      //加载天空盒子纹理
      let cubeTexture = new THREE.CubeTextureLoader().load([
        right,
        left,
        top,
        bottom,
        front,
        back,
      ]);
      scene.background = cubeTexture; //设置场景背景
    },

    //创建宇宙背景
    createUniverse() {
      let texture = new THREE.TextureLoader().load(BgImg); //加载背景贴图
      scene.background = texture; //设置场景背景
    },

    //场景球体全景
    createPanorama(img) {
      let geometry = new THREE.SphereGeometry(500, 100, 100);
      let material = new THREE.MeshBasicMaterial({
        map: new THREE.TextureLoader().load(img), //导入图片
        color: 0xffffff,
        side: THREE.BackSide,
      });
      let mesh = new THREE.Mesh(geometry, material);
      scene.add(mesh);
    },

    //创建轨道控制
    createOrbitControls() {
      //没有缩放阻尼
      this.mouseControls = new OrbitControls(
        this.camera,
        this.renderer.domElement
      ); //创建控件对象
      this.mouseControls.enablePan = false; //右键平移拖拽
      this.mouseControls.enableZoom = true; //鼠标缩放
      this.mouseControls.minDistance = 0; //相机距离原点的距离范围
      this.mouseControls.maxDistance = 100;
      this.mouseControls.enableDamping = true; //滑动阻尼
      this.mouseControls.dampingFactor = 0.1; //(默认.25)
      this.mouseControls.maxPolarAngle = (Math.PI / 4) * 3; //y旋转角度范围
      this.mouseControls.minPolarAngle = Math.PI / 4;
      this.mouseControls.autoRotate = true; //自转(相机)
      this.mouseControls.autoRotateSpeed = 1; //自转速度
    },

    //创建光源
    createLight() {
      this.ambientLight = new THREE.AmbientLight(0xffffff); //设置环境光
      scene.add(this.ambientLight); //将环境光添加到场景中
      this.pointLight = new THREE.PointLight(0xffffff, 1, 0);
      this.pointLight.position.set(50, 50, 0); //设置点光源位置
      scene.add(this.pointLight); //将点光源添加至场景
    },

    //加载模型
    loadModel(item) {
      //利用Promise加载模型
      let lm = new Promise((resolve, reject) => {
        let loader = new GLTFLoader();
        loader.load(item.modelUrl, (gltf) => {
          resolve(gltf); //返回加载成功的模型
          reject("model加载失败！");
          this.isLoading = false; //关闭加载框
        });
      });
      //加载成功
      lm.then((res) => {
        console.log("model加载成功！");
        res.scene.position.set(
          item.modePar[0],
          item.modePar[1],
          item.modePar[2]
        );
        res.scene.scale.set(item.modePar[3], item.modePar[4], item.modePar[5]);
        scene.add(res.scene);
        this.isLoading = false; //关闭加载框
      });
      //加载失败
      lm.catch((err) => {
        console.log(err);
        this.isLoading = false; //关闭加载框
      });
    },

    //重复渲染
    repeatRender() {
      //请求动画帧，屏幕每刷新一次调用一次，绑定屏幕刷新频率
      this.anId = requestAnimationFrame(this.repeatRender);
      this.mouseControls.update(); //实时更新轨道控制
      //   this.cube.rotation.y += 0.01; //以y为轴心的旋转角度每帧自加0.01
      this.renderer.render(scene, this.camera); //将场景和相机进行渲染
    },
  },
};
</script>
<style scoped lang='scss'>
.model-com {
  height: 100%;
  width: 100%;

  .dynamic-components-div {
    height: 100%;
    width: 100%;

    #container {
      height: 100%;
      width: 100%;
    }
  }

  .toggle-div {
    position: absolute;
    top: 0px;
    margin: 0px 0px 30px 0px;
    background-color: rgba(255, 255, 255, 0.5);
    color: #fff;

    .buts,
    .thisButs {
      cursor: pointer;
      display: block;
      color: #fff;
      margin: 20px;
      padding: 10px 40px;
      box-shadow: 0px 0px 5px black;

      &:hover {
        color: #000;
        box-shadow: 5px 5px 10px black;
        background-color: #ff8282;
      }
    }

    .thisButs {
      color: #000;
      box-shadow: 5px 5px 10px black;
      background-color: #ff8282;
    }
  }
}
</style>
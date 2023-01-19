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
import panoramaImg1 from "@/assets/img/panorama/pImg20.png"; //全景图
import panoramaImg2 from "@/assets/img/panorama/pImg2.png";
import panoramaImg3 from "@/assets/img/panorama/pImg21.png";
import panoramaImg4 from "@/assets/img/panorama/pImg14.png";
import panoramaImg5 from "@/assets/img/panorama/pImg16.png";
import panoramaImg6 from "@/assets/img/panorama/pImg11.png";
import panoramaImg7 from "@/assets/img/panorama/pImg4.png";
import right from "@/assets/img/skyBox/right.jpg"; //天空盒子
import left from "@/assets/img/skyBox/left.jpg";
import top from "@/assets/img/skyBox/top.jpg";
import bottom from "@/assets/img/skyBox/bottom.jpg";
import front from "@/assets/img/skyBox/front.jpg";
import back from "@/assets/img/skyBox/back.jpg";
let scene = null; //场景(在vue3中scene与mesh不能在data中定义，必须全局定义)
export default {
  components: {},
  data() {
    return {
      isLoading: "",
      modelList: [
        //模型列表
        {
          modelUrl: "model/导弹.glb",//模型路径
          text: "导弹",//名称
          bg: panoramaImg1,//全景图
          cameraPar: [60, 0.01, 1000, -50, 30, 100],//前三位为相机参数，后三位为相机位置
          modePar: [0, 0, -15, 50, 50, 50],//前三位为模型位置，后三位为模型缩放
        },
        {
          modelUrl: "model/左轮手枪.glb",
          text: "左轮手枪",
          cameraPar: [60, 0.01, 1000, 0, 50, 100],
          bg: panoramaImg2,
          modePar: [0, 0, 0, .1, .1, .1],
        },
        {
          modelUrl: "model/武士刀.glb",
          text: "武士刀",
          cameraPar: [60, 0.01, 10000, -10, 5, 10],
          bg: panoramaImg3,
          modePar: [2, 0, -5, 20, 20, 20],
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

    //销毁组数据
    clearGroup(group) {
      //清除缓存
      const clearCache = (item) => {
        item.geometry.dispose();//必须对组中的material与geometry进行dispose，清除占用的缓存
        item.material.dispose();
      };
      //移除模型
      const removeObj = (obj) => {
        let arr = obj.children.filter((x) => x);
        arr.forEach((item) => {
          if (item.children.length) {
            removeObj(item);
          } else {
            clearCache(item);
            item.clear();
          }
        });
        obj.clear();
        arr = null;
      };
      removeObj(group);
    },

    //初始化场景
    init(item) {
      scene = new THREE.Scene(); //新建场景
      let dom = document.getElementById("container");
      let width = dom.clientWidth; //场景宽度
      let height = dom.clientHeight; //场景高度
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
      this.createPanorama(item.bg); //球体全景
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
      let lightColor = new THREE.Color(0xffffff);
      let ambient = new THREE.AmbientLight(lightColor); //环境光
      ambient.name = "环境光";
      scene.add(ambient);
      let directionalLight1 = new THREE.DirectionalLight(lightColor);
      directionalLight1.position.set(0, 0, 500);
      scene.add(directionalLight1); //平行光源添加到场景中
      let directionalLight2 = new THREE.DirectionalLight(lightColor);
      directionalLight2.position.set(0, 0, -500);
      scene.add(directionalLight2); //平行光源添加到场景中
      let directionalLight3 = new THREE.DirectionalLight(lightColor);
      directionalLight3.position.set(500, 0, 0);
      scene.add(directionalLight3); //平行光源添加到场景中
      let directionalLight4 = new THREE.DirectionalLight(lightColor);
      directionalLight4.position.set(-500, 0, 0);
      scene.add(directionalLight4); //平行光源添加到场景中
      let directionalLight5 = new THREE.DirectionalLight(lightColor);
      directionalLight5.position.set(0, 500, 0);
      scene.add(directionalLight5); //平行光源添加到场景中
      let directionalLight6 = new THREE.DirectionalLight(lightColor);
      directionalLight6.position.set(0, -500, 0);
      scene.add(directionalLight6); //平行光源添加到场景中
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
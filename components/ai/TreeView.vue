<script setup>

import { onMounted, ref, defineProps, watch } from 'vue';
import * as THREE from 'three';
import { FBXLoader } from 'three/addons/loaders/FBXLoader.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { VRMLoaderPlugin, VRMUtils } from '@pixiv/three-vrm';
import { loadMixamoAnimation } from '~/components/ai/js/loadMixamoAnimation';
import axios from 'axios';
import { ComfyUI } from "~/utils/comfyuiApi";
import { commonUpload, setModalPreview, setActionPreview, uploadJsonAction } from '~/utils/ModelApi';
import Authorization from "~/components/Auth/Authorization.vue"

import { TransformControls } from "three/examples/jsm/controls/TransformControls.js";
import * as THREE_VRM from "@pixiv/three-vrm";
import VRMSkeleton from "../../utils/vrm-skeleton.mjs";
import { InteractionManager } from 'three.interactive';





// 定义接收的 props
const props = defineProps({
    url: String,
    boxWidth: {
        type: Number,
        default: 0,
    },
    boxHeight: {
        type: Number,
        default: 0,
    },
    rate: {
        type: Number,
        default: 0,
    },
    rateItem: {
        type: Object,
        default() {
            return {
                id: 1,
                name: '1:1',
                Width: 1024,
                Height: 1024
            };
        },
    },
    modelPreview: {
        type: Boolean,
        default: false
    },
    modelID: {
        type: Number,
        default: 0
    }
});

const canvasWidth = ref(0);
const canvasHeight = ref(0);
const threeContainer = ref(null);
// 使用 watch 监听 someProp 的变化
watch(() => props.rateItem, (newValue, oldValue) => {

    calcRect(newValue);
    try {
        onWindowResize()
    } catch (error) {

    }



}, { deep: true });


const calcRect = (newValue) => {
    if (props.boxHeight > props.boxWidth) {
        if (newValue.Width >= newValue.Height) {
            canvasWidth.value = props.boxWidth;
            canvasHeight.value = Number((props.boxWidth * (newValue.Height / newValue.Width)).toFixed(0));
        } else {
            canvasHeight.value = props.boxHeight;
            canvasWidth.value = Number((props.boxHeight * (newValue.Width / newValue.Height)).toFixed(0));
        }

    } else {
        if (newValue.Width >= newValue.Height) {
            canvasWidth.value = props.boxWidth;
            canvasHeight.value = Number((props.boxWidth * (newValue.Height / props.rateItem.Width)).toFixed(0));
        } else {
            canvasHeight.value = props.boxHeight;
            canvasWidth.value = Number((props.boxHeight * (newValue.Width / newValue.Height)).toFixed(0));
        }
    }

};


watch(() => canvasWidth.value, (newValue) => {
    try {
        onWindowResize()
    } catch (error) {

    }

});

const emit = defineEmits(['onUploadPreview']);

let scene;
let camera;
let renderer;
let currentVrm = undefined;
let currentFbx = undefined;
let controls;


let skeleton = null;
let transform_controls = null;
let skeletonHolder = null;
let interactionManager = null;
let raycaster = new THREE.Raycaster(); // 射线投射器


const loader = new GLTFLoader();

const isLoading = ref(false);

const params = {
    timeScale: 1.0,
};

let currentAnimationUrl = undefined; // 当前加载的动画文件地址
let currentMixer = undefined; //当前动画播放器
let currentAction = ref(undefined); // 当前播放的动画
const playState = ref(false);

const clock = new THREE.Clock();

//当前动作id
let currentActionId = ref(undefined);

//已经加载得模型列表
let modelList = [];

//是否正在调整骨骼
let isAdjustingSkeleton = false;
//当前高亮得对象
let highlightedObject = null;

onMounted(() => {

    if (!threeContainer.value) return;

    //loadVRM();
    window.addEventListener('resize', onWindowResize); // 添加窗口大小变化监听器

    const time_id = setInterval(() => {
        if (props.boxWidth > 0) {
            initThree();
            clearInterval(time_id);
        }

    }, 100);
});


onUnmounted(() => {
    window.removeEventListener('resize', onWindowResize); // 组件卸载时移除监听器
});

function onWindowResize() {
    // 更新相机和渲染器的大小
    const width = canvasWidth.value;
    const height = canvasHeight.value;
    renderer.setSize(width, height);
    camera.aspect = width / height; // 更新相机的纵横比
    camera.updateProjectionMatrix(); // 更新相机的投影矩阵
}


function setBackground(url) {

    //如果图片是空的就设置为白色背景
    if (url == '') {
        renderer.setClearColor(new THREE.Color(0xffffff));
        scene.background = new THREE.Color(0xffffff);
        return;
    }



    //将传递来得base64图片数据转为url
    const textureLoader = new THREE.TextureLoader();
    textureLoader.load(url, function (texture) {
        //renderer.setClearColor(new THREE.Color(0xffffff));
        texture.format = THREE.RGBAFormat; // 确保支持透明度


        scene.background = texture;
        renderer.render(scene, camera);
    });

}

function initThree() {
    calcRect(props.rateItem);
    const width = canvasWidth.value;
    const height = canvasHeight.value;
    // renderer
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(width, height);
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setClearColor(0xffffff, 1); // 第一个参数是颜色（十六进制或rgb值），第二个参数是透明度（0到1之间）


    threeContainer.value.appendChild(renderer.domElement);

    // camera
    camera = new THREE.PerspectiveCamera(30.0, width / height, 0.1, 20.0);
    camera.position.set(0.0, 1.0, 5.0);

    // camera controls
    controls = new OrbitControls(camera, renderer.domElement);

    // 设置鼠标中键为移动视角的按键
    controls.mouseButtons = {
        LEFT: THREE.MOUSE.ROTATE,
        MIDDLE: THREE.MOUSE.PAN, // 使用鼠标中键进行平移
        RIGHT: THREE.MOUSE.ZOOM
    };

    // 监听键盘事件以实现 WASD 平移
    document.addEventListener('keydown', function (event) {
        switch (event.key) {
            case 'w':
                camera.position.y -= 0.1;
                break;
            case 's':
                camera.position.y += 0.1;
                break;
            case 'a':
                camera.position.x -= 0.1;
                break;
            case 'd':
                camera.position.x += 0.1;
                break;
        }
    });

    controls.screenSpacePanning = true;
    controls.target.set(0.0, 1.0, 0.0);
    controls.update();

    // scene
    scene = new THREE.Scene();

    // light
    const light = new THREE.DirectionalLight(0xffffff, Math.PI);
    light.position.set(1.0, 1.0, 1.0).normalize();
    scene.add(light);

    loader.crossOrigin = 'anonymous';

    loader.register((parser) => {
        return new VRMLoaderPlugin(parser, { autoUpdateHumanBones: true });
    });

    //load('http://111.229.129.250:8989/6634671438455832424.vrm');

    // helpers
    //const gridHelper = new THREE.GridHelper(10, 10);
    //scene.add(gridHelper);


    transform_controls = new TransformControls(camera, renderer.domElement);

    transform_controls.addEventListener('dragging-changed', function (event) {

        controls.enabled = !event.value;

    });

    scene.add(transform_controls);

    interactionManager = new InteractionManager(
        renderer,
        camera,
        renderer.domElement
    );



    clock.start();
    animate();
    const mouse = new THREE.Vector2();





    renderer.domElement.addEventListener('click', (event) => {
        if (isAdjustingSkeleton) return;

        const rect = renderer.domElement.getBoundingClientRect();
        mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
        mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

        raycaster.setFromCamera(mouse, camera);
        let intersects = raycaster.intersectObjects(scene.children, true);

        if (highlightedObject) {
            highlightedObject.traverse((child) => {
                if (child.isMesh && child.userData.originalMaterial) {
                    child.material = child.userData.originalMaterial;
                    delete child.userData.originalMaterial;
                }
            });
            highlightedObject = null;
        }

        let selectedObject = null;
        for (let i = 0; i < intersects.length; i++) {
            let obj = intersects[i].object;

            while (obj) {
                if (obj.userData?.isVrm) {
                    selectedObject = obj;
                    break;
                }
                obj = obj.parent;
            }
        }
        

        console.log(selectedObject, currentVrm);

        if (!selectedObject) return;

        let modelId = selectedObject.userData.modelID;
        console.log(modelId);

        for (let i = 0; i < modelList.length; i++) {
            if (modelList[i].scene.userData.modelID === modelId) {
                console.log(modelList[i]);
                currentVrm = modelList[i];
                if (skeleton) {
                    scene.remove(skeletonHolder);
                }

                skeletonHolder = new THREE.Group();
                scene.add(skeletonHolder);

                skeleton = new VRMSkeleton({
                    vrm: currentVrm,
                    holder: skeletonHolder,
                    interact: true,
                    interactionManager: interactionManager,
                    transform_controls: transform_controls
                });
                skeleton.hide();
            }
        }

        highlightedObject = selectedObject;
        highlightedObject.traverse((child) => {
            if (child.isMesh && !child.userData.originalMaterial) {
                child.userData.originalMaterial = child.material;

                let outlineMaterial = new THREE_VRM.MToonMaterial({
                    outlineWidthFactor: 0.01,
                    outlineWidthMode: "worldCoordinates",
                    outlineColorFactor: new THREE.Color(0x00ff00),
                    side: THREE.BackSide,
                    isOutline: true
                });
                outlineMaterial.transparent = false;
                outlineMaterial.opacity = 1;
                outlineMaterial.depthTest = true;
                outlineMaterial.depthWrite = true;    

                let geometry = child.geometry;
                if (geometry.groups.length === 0) {
                    geometry.clearGroups();
                    geometry.addGroup(0, Infinity, 0);
                    geometry.addGroup(0, Infinity, 1);
                }

                child.material = [child.material, outlineMaterial];
                outlineMaterial.needsUpdate = true;
            }
        });
    });

    renderer.domElement.addEventListener('mousedown', (e) => {
        if (isAdjustingSkeleton) return;

        if (!highlightedObject) return;

        if (!raycast(e)) {

            // 取消高亮
            highlightedObject.traverse((child) => {
                if (child.isMesh && child.userData.originalMaterial) {
                    child.material = child.userData.originalMaterial;
                    delete child.userData.originalMaterial; // 删除原始材质的引用，避免内存泄漏

                    // 寻找child 中userData.isOutline=true得子对象并移除



                }
            });
            highlightedObject = null;
            return;
        }

        isMouseDown = true;
        controls.enabled = false; // 禁用控制器，防止冲突
    });

    // 鼠标抬起事件
    renderer.domElement.addEventListener('mouseup', () => {
        isMouseDown = false;
        controls.enabled = true; // 启用控制器


    });

    window.addEventListener('mouseup', () => {
        isMouseDown = false;
        controls.enabled = true; // 启用控制器

    });

    // 鼠标移动事件
    renderer.domElement.addEventListener("mousemove", (event) => {
        if (isMouseDown && highlightedObject) {
            const rect = renderer.domElement.getBoundingClientRect();

            let mouseX = ((event.clientX - rect.left) / rect.width) * 2 - 1;
            let mouseY = -((event.clientY - rect.top) / rect.height) * 2 + 1;

            const raycaster = new THREE.Raycaster();
            const mouseVector = new THREE.Vector2(mouseX, mouseY);
            raycaster.setFromCamera(mouseVector, camera);

            const cameraDirection = new THREE.Vector3();
            camera.getWorldDirection(cameraDirection);

            let planeNormal;
            if (Math.abs(cameraDirection.y) > Math.abs(cameraDirection.x) && Math.abs(cameraDirection.y) > Math.abs(cameraDirection.z)) {
                planeNormal = new THREE.Vector3(0, 1, 0);
            } else {
                planeNormal = new THREE.Vector3().copy(cameraDirection).setY(0).normalize();
            }

            const plane = new THREE.Plane(planeNormal, -currentVrm.scene.position.dot(planeNormal));
            const targetPoint = new THREE.Vector3();
            raycaster.ray.intersectPlane(plane, targetPoint);

            if (targetPoint) {
                currentVrm.scene.position.lerp(targetPoint, 0.2);
            }
        }
    });



    //监听delete键按下事件，删除当前高亮对象
    window.addEventListener('keydown', (event) => {
        if (highlightedObject && event.code === 'Delete') { // 只有在有高亮对象时才删除
            scene.remove(highlightedObject);
            highlightedObject = null;
        }
    });


}


//射线检测模型
function raycast(event) {
    const mouse = new THREE.Vector2();

    // 获取容器的边界框
    const rect = renderer.domElement.getBoundingClientRect();
    // 计算鼠标相对于容器的位置
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(scene.children, true);
    if (intersects.length > 0) {

        //诸葛查找父对象中得userData.isVrm=true的对象
        let selectedObject = null;

        //循环intersects
        for (let i = 0; i < intersects.length; i++) {
            const intersection = intersects[i];
            //一直往上查找父对象中得userData.isVrm=true的对象 直到找到为止或为null
            let parent = intersection.object.parent;
            while (parent) {
                if (parent.userData && parent.userData.isVrm) {
                    selectedObject = parent;
                    break;
                }
                parent = parent.parent;
            }
        }


        if (!selectedObject) {
            return null; // 退出函数，不进行后续操作
        }


        //获取选中得模型id
        let modelId = selectedObject.userData.modelID;
        console.log(modelId);

        //获取modelList中id对应的模型信息
        var vrm = null;
        for (let i = 0; i < modelList.length; i++) {
            if (modelList[i].scene.userData.modelID === modelId) {
                console.log(modelList[i]);
                vrm = modelList[i];
                if (skeleton) {
                    scene.remove(skeletonHolder);
                }

                skeletonHolder = new THREE.Group();
                scene.add(skeletonHolder);

                skeleton = new VRMSkeleton({
                    vrm: vrm,
                    holder: skeletonHolder,
                    interact: true,
                    interactionManager: interactionManager,
                    transform_controls: transform_controls
                });
                skeleton.hide();
            }
        }

        // 如果没有选中任何
        return vrm;
    }
}



//生成唯一ID
function generateUniqueID() {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) { // 生成一个随机的UUID
        const r = Math.random() * 16 | 0,
            v = c === 'x' ? r : (r & 0x3 | 0x8);
        return v.toString(16);
    });
}



const load = (url) => {
    isLoading.value = true;
    loader.load(
        url,
        async (gltf) => {
            const vrm = gltf.userData.vrm;
            VRMUtils.removeUnnecessaryVertices(gltf.scene);
            VRMUtils.removeUnnecessaryJoints(gltf.scene, { sameBoneCounts: true });

            if (currentFbx) {
                scene.remove(currentFbx);
                VRMUtils.deepDispose(currentFbx);
                currentFbx = undefined;
            }

            if (currentVrm) {
                // scene.remove(currentVrm.scene);
                // VRMUtils.deepDispose(currentVrm.scene);
                // currentVrm = undefined;
            }
            vrm.scene.traverse((obj) => {
                obj.frustumCulled = false;

            });
            currentVrm = vrm;
            modelList.push(vrm)

            scene.add(vrm.scene);
            VRMUtils.rotateVRM0(vrm);
            isLoading.value = false;
            useCustomToast(`切换模型完成`)

            //标记为vrm模型
            vrm.scene.userData.isVrm = true;
            //标记id
            vrm.scene.userData.modelID = generateUniqueID();


            if (skeleton) {
                scene.remove(skeletonHolder);
            }

            skeletonHolder = new THREE.Group();
            scene.add(skeletonHolder);

            skeleton = new VRMSkeleton({
                vrm: vrm,
                holder: skeletonHolder,
                interact: true,
                interactionManager: interactionManager,
                transform_controls: transform_controls
            });
            skeleton.hide();



            if (props.modelPreview == false) {
                console.log('需要上传模型截图');
                let pic = await handleUploadImage();
                if (pic) {
                    console.log('上传模型截图完成', pic);
                    if (props.modelID) {
                        await setModalPreview(props.modelID, pic)
                        emit('onUploadPreview');
                    }
                }
            }
        },
        (progress) => {
            console.log('Loading model...', 100.0 * (progress.loaded / progress.total), '%');
        },
        (error) => {
            console.error(error)
            isLoading.value = false;
        }
    );
}



const loadJson = async (animationUrl, action_id) => {
    currentActionId.value = action_id;
    // 解析 JSON 字符串
    const req = axios.create({});

    try {
        const response = await req.get(animationUrl);
        let myData = response.data;

        //const myData = JSON.parse(data);
        // 遍历 data 对象下的所有属性
        for (const key in myData.data) {
            if (myData.data.hasOwnProperty(key)) {
                updatePose(key, myData.data[key]);
            }
        }

        if (currentMixer) {
            currentMixer.stopAllAction();
            currentMixer = undefined;
        }
    } catch (error) {
        console.error('There has been a problem with your axios operation:', error);
    }
}

// 更新骨骼的函数
function updatePose(bone, PoseData) {
    // 将四元数数据转换为 three.js 的 Quaternion 对象
    const quaternion = new THREE.Quaternion(PoseData.rotation[0], PoseData.rotation[1], PoseData.rotation[2], PoseData.rotation[3]);
    // 获取骨骼节点
    const BoneNode = currentVrm.humanoid.getNormalizedBoneNode(bone)
    if (BoneNode) {
        // 更新骨骼节点的四元数
        // 应用四元数到骨骼
        BoneNode.quaternion.copy(quaternion);

    }
}



// mixamo animation
const loadFBX = (animationUrl, action_id) => {
    currentActionId.value = action_id;
    currentAnimationUrl = animationUrl;
    currentMixer = undefined;
    if (currentVrm) {
        // create AnimationMixer for VRM
        currentMixer = new THREE.AnimationMixer(currentVrm.scene);

        // Load animation
        loadMixamoAnimation(animationUrl, currentVrm).then((clip) => {
            // Apply the loaded animation to mixer and play
            currentAction.value = currentMixer.clipAction(clip);
            currentAction.value.play();
            currentMixer.timeScale = params.timeScale;

            playState.value = true;
        });

    } else {
        currentMixer = new THREE.AnimationMixer(currentFbx);
        loadFbxAnimation(animationUrl);

    }
}


const loadFbxAnimation = (fbxFile) => {
    const loader = new FBXLoader();
    loader.load(fbxFile, (object) => {

        currentAction.value = currentMixer.clipAction(object.animations[0]);
        currentAction.value.play();
        playState.value = true;
        console.log('Animation loaded and playing', playState.value);
    });
}


const loadFbxModel = (animationUrl) => {
    var loader = new FBXLoader();
    isLoading.value = true;
    loader.load(animationUrl, async (object) => {

        if (currentFbx) {
            scene.remove(currentFbx);
            VRMUtils.deepDispose(currentFbx);
            currentFbx = undefined;
        }

        if (currentVrm) {
            scene.remove(currentVrm.scene);
            VRMUtils.deepDispose(currentVrm.scene);
            currentVrm = undefined;
        }

        var box = new THREE.Box3().setFromObject(object);
        var center = box.getCenter(new THREE.Vector3());


        var size = box.getSize(new THREE.Vector3());
        var maxAxis = Math.max(size.x, size.y, size.z);
        object.scale.multiplyScalar(1.0 / maxAxis);


        var box2 = new THREE.Box3().setFromObject(object);
        var center = box2.getCenter(new THREE.Vector3());



        camera.position.x = 0.008329584657080943;
        camera.position.y = 0.5313432487786042; // 假设模型的y轴是高度
        camera.position.z = 3.824631107174858; // 假设模型的x轴是宽度

        scene.add(object);
        // 设置模型位置和大小
        object.position.set(0, 0, 0);

        // 相机聚焦到模型
        camera.lookAt(object.position);

        controls.target.set(0.02449150236624973, 0.6583971038516163, -0.0419121486666639);

        controls.update();


        currentFbx = object;

        isLoading.value = false;
        useCustomToast(`切换模型完成`)

        if (props.modelPreview == false) {
            console.log('需要上传模型截图');
            let pic = await handleUploadImage();
            if (pic) {
                console.log('上传模型截图完成', pic);
                if (props.modelID) {
                    await setModalPreview(props.modelID, pic)
                    emit('onUploadPreview');
                }
            }
        }
    },
        (progress) => console.log('Loading model...', 100.0 * (progress.loaded / progress.total), '%'),
        (error) => {
            console.error(error)
            isLoading.value = false;
        }
    );
}



const animate = () => {
    requestAnimationFrame(animate);

    const deltaTime = clock.getDelta();

    // if animation is loaded
    if (currentMixer) {
        // update the animation
        currentMixer.update(deltaTime);
    }

    if (currentVrm) {

        currentVrm.update(deltaTime);
        if (skeleton) {
            skeleton.update();
        }
    }
    interactionManager.update();
    renderer.render(scene, camera);

};



//上传图片
const handleUploadImage = async () => {

    //隐藏骨骼调整
    skeleton.hide();

    //如果已经有高亮对象，则先取消其高亮
    if (highlightedObject) {
        // 取消高亮
        highlightedObject.traverse((child) => {
            if (child.isMesh && child.userData.originalMaterial) {
                child.material = child.userData.originalMaterial;
                delete child.userData.originalMaterial; // 删除原始材质的引用，避免内存泄漏

                // 寻找child 中userData.isOutline=true得子对象并移除
                child.traverse((subChild) => {
                    if (subChild.userData.isOutline) {
                        subChild.parent.remove(subChild);
                    }
                });



            }
        });
    }

    isAdjustingSkeleton = false;


    // 渲染场景
    renderer.render(scene, camera);
    // 用于存储图片URL
    let imageUrl = '';

    // 捕获Blob数据
    const blobPromise = new Promise((resolve, reject) => {
        renderer.domElement.toBlob((blob) => {
            if (blob) {
                resolve(blob);
            } else {
                reject(new Error('Blob data is null'));
            }
        }, 'image/png');
    });

    try {



        const blob = await blobPromise;
        console.log('Blob:', blob);
        const file = new File([blob], 'screenshot.png', { type: 'image/png' });


        useCustomToast('开始上传文件……');




        await ComfyUI.Initalize(); // 注意方法名大小写，应保持一致

        // 上传文件
        let res = await commonUpload(file)
        console.log('res:', res.data.data.fullurl);
        // const imagePath = await ComfyUI.uploadImage(file);
        if (res && res.data.code === 1) {
            useCustomToast('文件上传完成');
            imageUrl = res.data.data.fullurl;
        } else {
            useSnackbarStore().showErrorMessage('上传服务器错误，请检查服务器OSS配置');
        }
    } catch (error) {
        console.error('处理上传图片时发生错误:', error);
        useSnackbarStore().showErrorMessage('上传图片失败，请重试');
    }
    console.log('Image URL:', imageUrl);
    return imageUrl;
};

// 使用 watch 监听 someProp 的变化
watch(() => props.url, (newValue, oldValue) => {
    // 当 someProp 发生变化时，执行这里的逻辑
    if (isLoading.value === true) {
        return;
    }

    const url = 'http://example.com/path/to/file.vrm';
    if (getSuffix(newValue) === 'fbx') {
        loadFbxModel(newValue)
    } else {
        load(newValue);
    }


});

const getSize = () => {
    //暂停动画
    currentAction.value.paused = !currentAction.value.paused;
    playState.value = !playState.value;
};

//设置动作预览图
const actionPreview = async () => {
    let pic = await handleUploadImage();
    if (pic) {
        //console.log('上传动作截图完成', pic);
        if (currentActionId) {
            await setActionPreview(currentActionId.value, pic)
            emit('onActionPreview');
            useCustomToast('动作预览图上传完成');
        }
    }
}



const changeSkeleton = () => {

    if (!currentVrm) {
        return;
    }

    if (skeleton.visible) {
        isAdjustingSkeleton = false;
        skeleton.hide();
    } else {
        isAdjustingSkeleton = true;
        skeleton.show();
    }
};


const getSkeleton = () => {
    return skeleton.visible;
};


const saveSkeleton = async (name) => {
    if (currentVrm) {
        // 遍历模型上所有骨骼
        const humanoid = currentVrm.humanoid
        console.log(humanoid);


        let pose = {};

        let restPose = humanoid._rawHumanBones.restPose
        for (let key in restPose) {

            let position = humanoid._normalizedHumanBones.original.humanBones[key].node.position
            let rotation = humanoid._normalizedHumanBones.original.humanBones[key].node.quaternion
            //为pose中添加骨骼信息
            pose[key] = { position: [position.x, position.y, position.z], rotation: [rotation._w, rotation._x, rotation._y, rotation._z] };
        }


        //上传数据
        let res = await uploadJsonAction(name, pose);
        console.log(res);
        let aid = res.data.data.id;

        let open = false
        if (skeleton.visible) {
            open = true;

            skeleton.hide();
        }

        jsonActionPreview(aid);

        if (open) {
            skeleton.show();
        }

        // let json = JSON.stringify(pose);
        // saveToFile(json, 'skeleton.json');

    } else {
        //
        useSnackbarStore().showErrorMessage('请先加载模型');
    }
};


//设置动作预览图
const jsonActionPreview = async (aid) => {
    let pic = await handleUploadImage();
    if (pic) {
        //console.log('上传动作截图完成', pic);
        if (currentActionId) {
            await setActionPreview(aid, pic)
            emit('onActionPreview');
            useCustomToast('动作预览图上传完成');
        }
    }
}

const loadSkeletonFromJson = (data, id) => {
    let json = data;
    currentActionId.value = id;

    if (currentVrm) {
        currentMixer = undefined;
        // 遍历模型上所有骨骼
        const humanoid = currentVrm.humanoid;
        //humanoid._normalizedHumanBones.restPose = JSON.parse(json);
        let pose = JSON.parse(json);
        for (let key in pose) {
            //humanoid._normalizedHumanBones[key] = JSON.parse(json)[key]
            var bone = humanoid.getNormalizedBoneNode(key)



            if (bone) {
                console.log(bone, pose[key])

                if (pose[key].position) {

                    bone.position.set(pose[key].position[0], pose[key].position[1], pose[key].position[2]);
                }

                // 检查是否存在旋转数据并更新骨骼旋转
                if (pose[key].rotation) {
                    const quaternion = new THREE.Quaternion(pose[key].rotation[1], pose[key].rotation[2], pose[key].rotation[3], pose[key].rotation[0]);
                    bone.quaternion.copy(quaternion);
                }
            }
            // 更新骨骼节点
            bone.updateMatrixWorld(true);


        }

    }
};


function saveToFile(data, filename) {
    const blob = new Blob([data], { type: 'application/json' });
    const link = document.createElement('a');
    link.href = window.URL.createObjectURL(blob);
    link.download = filename;
    link.click();
}

defineExpose({
    loadFBX,
    loadJson,
    handleUploadImage,
    setBackground,
    changeSkeleton,
    getSkeleton,
    saveSkeleton,
    loadSkeletonFromJson
})

const getSuffix = (url) => {
    const dotIndex = url.lastIndexOf('.');
    if (dotIndex === -1) {
        return ''; // 如果没有找到'.'，返回空字符串
    }
    return url.slice(dotIndex + 1); // 返回'.'之后的部分，即后缀
}

</script>
<template>
    <div ref="threeContainer" :style="'width: ' + canvasWidth + 'px; height: ' + canvasHeight + 'px;'" class="treeView">
        <a-spin size="large" tip="加载模型中..." v-if="isLoading" style="position: absolute;" />

        <div class="btn-group">
            <a-button type="primary" :size="size" @click="getSize()" v-if="currentAction"
                style="display: flex;align-items: center;justify-content: center;">
                <template #icon>
                    <PauseCircleOutlined v-if="!playState" />
                    <PlayCircleOutlined v-else />
                </template>
            </a-button>
            <!--        管理后台-->
            <Authorization role="admin">
                <span style="width: 20px;"></span>
                <a-button type="primary" :size="size" @click="actionPreview()" v-if="currentAction"
                    style="display: flex;align-items: center;justify-content: center;">
                    设置为动作预览图
                </a-button>
            </Authorization>
        </div>
    </div>
</template>

<style scoped lang="scss">
.treeView {
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    color: aliceblue;
}

.btn-group {
    position: absolute;
    bottom: 10px;
    display: flex;
    align-items: center;
    justify-content: center;
}
</style>
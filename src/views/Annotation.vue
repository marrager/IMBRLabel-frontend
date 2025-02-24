<script setup>
import { onMounted, ref, onBeforeUnmount, watch } from "vue";
import { useRoute, useRouter } from "vue-router"; // 引入 useRouter
import CanvasSelect from "canvas-select";

import { addAnnotation, autoAnnotation, fullAutoAnnotation, addAnnotationToImage, exportAnnotations, getTemplates, getImageDetails } from "@/api/annotation.js";
import { ElMessage, ElMessageBox } from "element-plus";
import { FullScreen, Hide, Pointer, Upload, Star } from "@element-plus/icons-vue";

let instance;
const route = useRoute();
const router = useRouter();
const imagePath = ref("");
const imageId = ref(route.query.imageId);
const imageName = ref("");
const previousImageId = ref(null);
const nextImageId = ref(null);

// 初始化已有的标注信息
let outputRef = ref(null);
const selectedShape = ref(null);
const history = ref([]);
const redoStack = ref([]);
const annotations = ref([]); // 用来保存所有标注的数据
const templates = ref([]); // 用来保存模板列表
const templateName = ref(""); // 模板名
const isHidden = ref(false);
// 定义createType，用于追踪当前绘制类型
const createType = ref(0);
const polygonSides = ref(20); // 默认多边形的边数为20

let saveHistoryTimeout = null; // 保存历史记录的防抖定时器

onMounted(() => {
  getImageDetails(imageId.value).then(response => {
    console.log("Response data:", response.data); // 打印响应数据

    imagePath.value = response.data.path;
    imageName.value = response.data.name;
    annotations.value = response.data.annotations;

    previousImageId.value = response.data.previousImageId;
    nextImageId.value = response.data.nextImageId;

    console.log("imagePath.value", imagePath.value);
    console.log("previousImageId.value", previousImageId.value);
    console.log("nextImageId.value", nextImageId.value);

    // 初始化 CanvasSelect 实例
    const fullImagePath = `http://192.168.232.129:8080/images/${imagePath.value}`;
    instance = new CanvasSelect(".container", fullImagePath);
    instance.labelMaxLen = 10;

    // 初始化 annotations，确保它是一个数组
    if (annotations.value && annotations.value.trim() !== "") {
      try {
        // 先解码，再解析 JSON 字符串
        const decodedAnnotations = decodeURIComponent(annotations.value);
        const parsedAnnotations = JSON.parse(decodedAnnotations);
        instance.setData(parsedAnnotations); // 传递标注数据给 CanvasSelect
      } catch (error) {
        console.error("Failed to parse annotations:", error);
        instance.setData([]); // 如果解析失败，使用空数组
      }
    } else {
      instance.setData([]); // 如果 annotations.value 为空，使用空数组
    }

    // 初始化 createType
    createType.value = instance.createType;

    // 获取模板列表
    fetchTemplates();
    // 更新 output 内容
    updateOutput();

    // 监听 createType 变化
    instance.on("typeChange", (type) => {
      createType.value = type;
    });

    instance.on("load", (src) => {
      console.log("image loaded", src);
    });
    instance.on("add", (info) => {
      console.log("add", info);
      triggerSaveHistory();
    });
    instance.on("delete", (info) => {
      console.log("delete", info);
      triggerSaveHistory();
    });
    instance.on("select", (shape) => {
      selectedShape.value = shape;
    });
    instance.on("updated", (result) => {
      annotations.value = result;
      triggerSaveHistory();
      updateOutput(); // 更新 output 内容
    });

    window.addEventListener("keydown", handleKeyDown);

  }).catch(error => {
    console.error("Error fetching image details:", error);
    ElMessage.error("获取图片详情失败");
  });
});

onBeforeUnmount(() => {
  window.removeEventListener("keydown", handleKeyDown);
  clearTimeout(saveHistoryTimeout); // 清除定时器
});

function handleKeyDown(event) {
  if (event.ctrlKey && event.key === "z") {
    undo();
  } else if (event.ctrlKey && event.key === "y") {
    redo();
  } else if (event.ctrlKey && event.shiftKey && event.key === "S") {
    update();
  }
}

// 更新 output 的内容
function updateOutput() {
  if (outputRef.value) {
    outputRef.value.value = JSON.stringify(annotations.value, null, 2);
  }
}

// 防抖机制，只有在停止输入 0.1 秒后才保存历史记录
function triggerSaveHistory() {
  clearTimeout(saveHistoryTimeout);
  saveHistoryTimeout = setTimeout(() => {
    saveHistory();
  }, 100); // 0.1 秒后保存
}

// 保存当前的状态
function saveHistory() {
  const currentState = {
    data: JSON.parse(JSON.stringify(annotations.value)), // 深拷贝保存标注数据
    output: outputRef.value ? outputRef.value : ""
  };
  if (
    history.value.length === 0 ||
    JSON.stringify(history.value[history.value.length - 1]) !==
    JSON.stringify(currentState)
  ) {
    history.value.push(currentState);
    redoStack.value = [];
  }
}

function undo() {
  if (history.value.length > 1) {
    redoStack.value.push(history.value.pop());
    const previousState = history.value[history.value.length - 1];
    annotations.value = JSON.parse(JSON.stringify(previousState.data)); // 恢复标注数据
    instance.setData(annotations.value);
    if (outputRef.value) {
      outputRef.value.value = previousState.output;
    }
  }
}

function redo() {
  annotations.value = []; // 清空标注数据
  instance.setData(annotations.value);
  if (outputRef.value) {
    outputRef.value.value = JSON.stringify(annotations.value, null, 2);
  }
}

function change(type) {
  instance.createType = type;
  createType.value = type;
}

function fitting() {
  instance.fitZoom();
}

function onFocus() {
  isHidden.value = !isHidden.value;
  instance.setFocusMode(!instance.focusMode);
}

function update() {
  addAnnotationToImage(imageId.value, encodeURIComponent(JSON.stringify(annotations.value)));
  ElMessage.success("标注数据已保存");
}

function exportJson() {
  handleExport("json");
}

function exportXml() {
  handleExport("xml");
}

function exportCsv() {
  handleExport("csv");
}

function handleExport(type) {
  exportAnnotations(imageId.value, type).then(result => {
    console.log(result);
    const blob = new Blob([result.data], { type: "text/plain;charset=utf-8" });
    const link = document.createElement("a");
    link.href = URL.createObjectURL(blob);
    console.log(imageName.value);
    link.download = `${imageName.value}.${type}`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  });
}

function saveAsTemplate() {
  if (selectedShape.value) {
    const templateData = {
      name: templateName.value,
      label: selectedShape.value.label,
      strokeStyle: selectedShape.value.strokeStyle,
      fillStyle: selectedShape.value.fillStyle,
      lineWidth: selectedShape.value.lineWidth
    };
    addAnnotation(templateData).then(response => {
      console.log("Template saved:", response);
      fetchTemplates(); // Refresh the template list
    }).catch(error => {
      console.error("Error saving template:", error);
    });
  } else {
    console.error("No shape selected to save as template.");
  }
}

// 获取模板列表
function fetchTemplates() {
  getTemplates().then(response => {
    templates.value = response.data;
  }).catch(error => {
    console.error("Error fetching templates:", error);
  });
}

// 应用模板
function applyTemplate(template) {
  if (selectedShape.value) {
    selectedShape.value.label = template.label;
    selectedShape.value.strokeStyle = template.strokeStyle;
    selectedShape.value.fillStyle = template.fillStyle;
    selectedShape.value.lineWidth = template.lineWidth;
    instance.update();
    triggerSaveHistory();
  } else {
    console.error("No shape selected to apply template.");
  }
}

// 半自动添加标注到图片
function autoAddAnnotation() {
  console.log("Adding annotations to image", annotations.value);
  autoAnnotation(
    encodeURIComponent(JSON.stringify(annotations.value)),
    polygonSides.value, // 传入 polygonSides 参数
    imageId.value
  ).then(response => {
    if (response && response.data) {
      console.log("Full response:", response);

      let data;
      if (typeof response.data === "string") {
        try {
          data = JSON.parse(response.data);
          console.log("Parsed response data:", data);
        } catch (error) {
          console.error("Failed to parse response data:", error);
          ElMessage.error("解析响应数据失败");
          return;
        }
      } else {
        data = response.data;
      }

      if (data) {
        console.log("Annotations added:", data);
        annotations.value = data;
        instance.setData(annotations.value);

        // 计算并输出最后一个标注的坐标点数量
        const lastAnnotation = annotations.value[annotations.value.length - 1];
        let numberOfPoints;
        if (lastAnnotation && lastAnnotation.coor) {
          numberOfPoints = lastAnnotation.coor.length;
        }
        ElMessage.success("标注已成功添加，" + "顶点共" + numberOfPoints + "个");
      }
    }
  }).catch(error => {
    console.error("Error adding annotations:", error);
    ElMessage.error("添加标注失败");
  });
}

// 全自动添加标注到图片
function fullAutoAddAnnotation() {
  console.log("Adding annotations to image", annotations.value);
  const selectedId = selectedShape.value
    ? parseInt(selectedShape.value.label)
    : null;
  fullAutoAnnotation(
    encodeURIComponent(JSON.stringify(annotations.value)),
    polygonSides.value,//传入 polygonSides 参数
    imageId.value,
    selectedId
  )
    .then(response => {
      if (response && response.data) {
        console.log("Full response:", response);

        let data;
        if (typeof response.data === "string") {
          try {
            data = JSON.parse(response.data);
            console.log("Parsed response data:", data);
          } catch (error) {
            console.error("Failed to parse response data:", error);
            ElMessage.error("解析响应数据失败");
            return;
          }
        } else {
          data = response.data;
        }

        if (data) {
          console.log("Annotations added:", data);
          annotations.value = data;
          instance.setData(annotations.value);

          ElMessage.success("标注已成功添加");
        }
      }
    }).catch(error => {
      console.error("Error adding annotations:", error);
      ElMessage.error("添加标注失败");
    });
}

// 处理上一张和下一张的导航
function goToPreviousImage() {
  if (previousImageId.value) {
    ElMessageBox.confirm('您有未保存的更改，是否先保存再切换到上一张？', '提示', {
      confirmButtonText: '保存跳转',
      cancelButtonText: '不保存跳转',
      type: 'warning',
    })
      .then(() => {
        // 用户选择确定，保存更改并跳转
        update().then(() => {
          router.push({ query: { imageId: previousImageId.value } }).then(() => location.reload());
        }).catch(() => {
          // 保存失败，不跳转
          ElMessage.error("保存失败，无法跳转到上一张图片");
        });
      })
      .catch(() => {
        // 用户选择取消，不保存直接跳转
        router.push({ query: { imageId: previousImageId.value } }).then(() => location.reload());
      });
  } else {
    // 如果是第一张图片
    ElMessage.warning("已经是第一张图片！");
  }
}

function goToNextImage() {
  if (nextImageId.value) {
    ElMessageBox.confirm('您有未保存的更改，是否先保存再切换到下一张？', '提示', {
      confirmButtonText: '保存跳转',
      cancelButtonText: '不保存跳转',
      type: 'warning',
    })
      .then(() => {
        // 用户选择确定，保存更改并跳转
        update().then(() => {
          router.push({ query: { imageId: nextImageId.value } }).then(() => location.reload());
        }).catch(() => {
          // 保存失败，不跳转
          ElMessage.error("保存失败，无法跳转到下一张图片");
        });
      })
      .catch(() => {
        // 用户选择取消，不保存直接跳转
        router.push({ query: { imageId: nextImageId.value } }).then(() => location.reload());
      });
  } else {
    // 如果是最后一张图片
    ElMessage.warning("已经是最后一张图片！");
  }
}


function goToImage() {
  router.push('/images');
}


// Watch for changes in label
watch(() => selectedShape.value?.label, (newVal) => {
  if (selectedShape.value) {
    selectedShape.value.label = newVal;
    instance.update();
  }
});

watch(() => selectedShape.value?.strokeStyle, (newVal) => {
  if (selectedShape.value) {
    selectedShape.value.strokeStyle = newVal;
    instance.update();
  }
});

watch(() => selectedShape.value?.fillStyle, (newVal) => {
  if (selectedShape.value) {
    selectedShape.value.fillStyle = newVal;
    instance.update();
  }
});

watch(() => selectedShape.value?.lineWidth, (newVal) => {
  if (selectedShape.value) {
    selectedShape.value.lineWidth = newVal;
    instance.update();
  }
});

// 监听多边形变数变化
watch(polygonSides, (newVal) => {
  instance.polygonSides = newVal;
  instance.update();
  console.log("Polygon sides updated to:", newVal);
});
</script>

<template>
  <div class="common-layout">
    <el-container>
      <el-header>
        <div class="header-buttons" style="border-color: #c0c0c0;">
          <el-button @click="change(1)" :type="createType === 1 ? 'primary' : 'default'">矩形</el-button>
          <el-button @click="change(2)" :type="createType === 2 ? 'primary' : 'default'">自定义多边形</el-button>
          <el-button @click="change(3)" :type="createType === 3 ? 'primary' : 'default'">点</el-button>
          <el-button @click="change(4)" :type="createType === 4 ? 'primary' : 'default'">线</el-button>
          <el-button @click="change(5)" :type="createType === 5 ? 'primary' : 'default'">圆</el-button>
          <el-button :icon="Pointer" @click="change(0)"></el-button>
          <el-button :icon="FullScreen" @click="fitting()"></el-button>
          <el-button :icon="Hide" @click="onFocus()" :type="isHidden ? 'primary' : 'default'"></el-button>
          <el-button :icon="Upload" type="success" @click="update()">保存</el-button>
          <el-button @click="exportJson()">导出为json</el-button>
          <el-button @click="exportXml()">导出为xml</el-button>
          <el-button @click="exportCsv()">导出为csv</el-button>
          <el-button type="danger" @click="autoAddAnnotation()">
            <el-icon>
              <Star />
            </el-icon> 半自动AI
          </el-button>
          <el-input-number v-model="polygonSides" :min="15" label="多边形边数" style="margin-left: 10px;" />
          <el-button type="danger" @click="fullAutoAddAnnotation()">全自动AI</el-button>
        </div>
      </el-header>

      <el-container>
        <el-aside width="100px"
          style="display: flex; flex-direction: column; align-items: center; justify-content: center; height: 100%; margin-left: 100px; margin-top: 30px;">
          <el-button @click="goToPreviousImage" icon="el-icon-arrow-left" size="large"
            style="margin: 10px 0; width: 80px; height: 80px; background-color: #f5f5f5;">
            上一张
          </el-button>
          <el-button @click="goToNextImage" icon="el-icon-arrow-right" size="large"
            style="margin: 10px 0; width: 80px; height: 80px; background-color: #f5f5f5;">
            下一张
          </el-button>
          <el-button @click="goToImage" type="primary" size="large"
            style="margin: 10px 0; width: 80px; height: 80px; background-color: #409eff; color: white;">
            返回
          </el-button>
        </el-aside>


        <el-main>
          <div class="canvas-container">
            <canvas class="container"></canvas>
            <div class="right-panel" v-if="selectedShape">
              <h3>编辑标注属性值</h3>
              <el-form label-width="100px">
                <el-form-item label="标签名称:">
                  <el-input v-model="selectedShape.label" clearable />
                </el-form-item>
                <el-form-item label="边线颜色:">
                  <el-color-picker v-model="selectedShape.strokeStyle" show-alpha />
                </el-form-item>
                <el-form-item label="填充颜色:">
                  <el-color-picker v-model="selectedShape.fillStyle" show-alpha />
                </el-form-item>
                <el-form-item label="边线宽度:">
                  <el-input-number v-model="selectedShape.lineWidth" />
                </el-form-item>
                <el-form-item label="模板名称:">
                  <el-input v-model="templateName" clearable>
                    <template #append>
                      <el-button @click="saveAsTemplate()" :icon="Upload" />
                    </template>
                  </el-input>
                </el-form-item>
              </el-form>

              <h3>模板列表</h3>
              <el-list>
                <el-list-item v-for="template in templates" :key="template.id" @click="applyTemplate(template)">
                  <el-card>{{ template.name }}</el-card>
                </el-list-item>
              </el-list>
            </div>
          </div>
        </el-main>
      </el-container>
    </el-container>
  </div>
</template>

<style scoped>
.common-layout {
  background-color: white;
  min-height: 100vh;
  padding: 0;
}

.header-buttons {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  background-color: #f5f5f5;
}

.canvas-container {
  display: flex;
  justify-content: space-between;
  padding: 20px;
}

.container {
  background-color: #ccc;
  width: 70%;
  height: 800px;
}

.right-panel {
  max-width: 300px;
  margin-left: 20px;
  padding: 10px;
  background-color: #f9f9f9;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
}

.el-form-item {
  margin-bottom: 15px;
}

.el-list-item {
  cursor: pointer;
}

.el-list-item:hover {
  background-color: #f0f0f0;
}
</style>

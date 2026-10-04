# Frontend

## Wireframe
https://www.figma.com/design/GXFM6QdwmtbhbqsYFR85li/5G-ORAN?node-id=0-1&t=pfVb1j7I6rL8sK9J-1

這個目前與實際的網頁有點出入。

## 3D 模型前端校正

在前端套上 `.gltf` 的時候會有些許的誤差(同個模型每次也有可能不同)因此需要讓使用者根據自己的狀況作些微調整
似乎是在從 `.usd` 到`.gltf` 的過程會被放大所以我們需要額外縮放(當然還有模型的大略定位，如經緯度、角度需要紀錄和調整)

### 運作流程

```ts
const response = await $apiClient.project.projectsDetail(validProjectId.value)
projectExists.value = true
// Set lat/lon from API response
projectLat.value = response.data.lat ? Number(response.data.lat) : null
projectLon.value = response.data.lon ? Number(response.data.lon) : null
projectMargin.value = response.data.margin ? Number(response.data.margin) : null
modelLatOffset.value = response.data.lat_offset ? Number(response.data.lat_offset) : null;
modelLonOffset.value = response.data.lon_offset ? Number(response.data.lon_offset) : null;
modelRotateOffset.value = response.data.rotation_offset ? Number(response.data.rotation_offset) : null;
modelScalingOffset.value = response.data.scale ? Number(response.data.scale) : null;
```

先 call api 取得這個project的資訊(座標跟offset)

```ts
// Use projectLat/projectLon for map center
  const mapCenter = computed<[number, number]>(() => {
    if (projectLon.value !== null && projectLat.value !== null) {
      return [projectLon.value, projectLat.value]
    }
    return [141.3501, 43.064] // fallback default
  })
  const mapOffset = computed<[number, number, number, number]>(() => {
    return [
      modelLonOffset.value ?? 0,
      modelLatOffset.value ?? 0,
      modelRotateOffset.value ?? 0,
      modelScalingOffset.value ?? 1
    ]
  });
```

然後用額外的變數存(可以避免用到 null)

```ts

// map 代表 Mapbox GL JS 的地圖物件，負責在網頁上顯示地圖並提供互動功能。你可以用 map 來新增圖層、監聽事件、控制視角等。
// 我們用 Threebox 來加載 .gltf 的物件並渲染在 mapbox 上

map?.addLayer({
          id: THREEBOX_MODEL_LAYER_ID,
          type: 'custom',
          renderingMode: '3d',
          onAdd: function (map, gl) {
            const tb = (window.tb = new Threebox.Threebox(
              map,
              gl,
              { defaultLights: true }
            ));

            const options = {
              obj: 'data:text/plain;base64,' + base64Content,
              type: 'gltf',
              scale: { x: 1, y: 1, z: 1 }, // temp, will update after bounding box
              units: 'meters',
              rotation: { x: 0, y: 0, z: 180 },
              anchor: 'center'
            };
            tb.loadObj(options, (model: any) => {
              console.log('Model loaded:', model);
              model.setCoords(mapCenter.value);
                
              //這裡會開始計算初始的縮放程度
              
              // --- Compute side length of the square model ---
              let boundingBox: any = null;
              const traverseTarget = model.object3d || model;
              let computedSideLength = 1;
              if (traverseTarget && typeof traverseTarget.traverse === 'function') {
                traverseTarget.traverse((child: any) => {
                  if (child.isMesh && child.geometry) {
                    child.geometry.computeBoundingBox();
                    if (!boundingBox) {
                      boundingBox = child.geometry.boundingBox.clone();
                    } else {
                      boundingBox.union(child.geometry.boundingBox);
                    }
                  }
                });
                if (boundingBox) {
                  const size = boundingBox.getSize(new THREE.Vector3());
                  computedSideLength = Math.max(size.x, size.y);
                  console.log('Square side length:', computedSideLength, 'meters');
                }
              }
              // --- End compute side length ---

              // --- Compute scale factor and apply ---
              if (projectMargin.value && computedSideLength > 0) {
                scaleFactor.value = projectMargin.value / computedSideLength /12;
                if (model.object3d && model.object3d.scale && typeof model.object3d.scale.set === 'function') {
                  model.object3d.scale.set(scaleFactor.value, scaleFactor.value, scaleFactor.value);
                }
                if (model.scale && typeof model.scale.set === 'function') {
                  model.scale.set(scaleFactor.value, scaleFactor.value, scaleFactor.value);
                }
                console.log('Applied model scale:', scaleFactor.value);
              }
              // --- End scale ---
              
              // 到這裡會把 scale factor 算出來 
              
              // 下面是再把 offset 套上去的過程
              
              threeboxModel = model;
              if (mapOffset.value[0] && mapOffset.value[1]) {
                const newCoords = [
                  mapCenter.value[0] + mapOffset.value[0],
                  mapCenter.value[1] + mapOffset.value[1]
                ];
                model.setCoords(newCoords);
              }
              if(mapOffset.value[2])model.rotation.z = (mapOffset.value[2]);
              
              // 需要注意，這裡只有 第二個 縮放需要套上 offset (我也不知道為什麼)，另一個套上 scaleFactor 就好
             
              if(mapOffset.value[3]){
                if (threeboxModel.object3d && threeboxModel.object3d.scale && typeof threeboxModel.object3d.scale.set === 'function') {
                  threeboxModel.object3d.scale.set(scaleFactor.value, scaleFactor.value, scaleFactor.value);
                }
                if (threeboxModel.scale && typeof threeboxModel.scale.set === 'function') {
                  threeboxModel.scale.set(scaleFactor.value*mapOffset.value[3], scaleFactor.value*mapOffset.value[3], scaleFactor.value*mapOffset.value[3]);
                }
              }
              tb.add(model);
              modelLoaded.value = true;
              // Add heatmap layer after model is loaded and scaleFactor is set
              addHeatmapLayer();
            });
          },
          render: function () {
            if (window.tb) {
              window.tb.update();
            }
          }
        });
```

這一大段是把 gltf 套上mapbox，詳細直接看裡面的中文註解(照抄就好)

### 使用者互動

```ts
// get current coordinate ** remember this is offset
    let [lon, lat] = [mapOffset.value[0],mapOffset.value[1]];
    let modelRotationZ = mapOffset.value[2];
    let scaleAdjust = mapOffset.value[3] ;

    const step = 0.000025;
    const rotateStep = (Math.PI/36)/5;
    const scaleStep = 0.005;
    
    switch (e.key) {
    case 'ArrowUp':
      lat += step; // north
      break;
    case 'ArrowDown':
      lat -= step; // south
      break;
    case 'ArrowLeft':
      lon -= step; // west
      break;
    case 'ArrowRight':
      lon += step; // east
      break;
    case 'a': // rotate left
      modelRotationZ = (modelRotationZ + rotateStep + 2 * Math.PI) % (2 * Math.PI);
      break;
    case 'd': // rotate right
      modelRotationZ = (modelRotationZ - rotateStep + 2 * Math.PI) % (2 * Math.PI);
      break;
    case '=': // For small keyboards + and = is the same key
      scaleAdjust += scaleStep;
      break;
    case '+':
      scaleAdjust += scaleStep;
      break;
    case '-':
      scaleAdjust -= scaleStep;
      break;
    default:
      return;
    }
    threeboxModel.setCoords([mapCenter.value[0]+lon, mapCenter.value[1]+lat]);

    // rotate
    threeboxModel.rotation.z = (modelRotationZ);
    // rescale
    if (threeboxModel.object3d && threeboxModel.object3d.scale && typeof threeboxModel.object3d.scale.set === 'function') {
      threeboxModel.object3d.scale.set(scaleFactor.value, scaleFactor.value, scaleFactor.value);
    }
    if (threeboxModel.scale && typeof threeboxModel.scale.set === 'function') {
      threeboxModel.scale.set(scaleFactor.value*scaleAdjust, scaleFactor.value*scaleAdjust, scaleFactor.value*scaleAdjust);
    }
    modelLonOffset.value=lon;
    modelLatOffset.value=lat;
    modelRotateOffset.value=modelRotationZ;
    modelScalingOffset.value=scaleAdjust;
  }
```

這裡我們訂互動方式以及每次操作的變量(step)
然後每次操作完都要套一次 offset 才能即時看到變化
我們的變化會繼承讀取到的mapOffset[ ]然後再繼續累加等等
(在先前可以看到把從後端讀取的 offset 整合到同個array)

```ts
mapOffset[0]：模型的經度偏移（lon offset）
mapOffset[1]：模型的緯度偏移（lat offset）
mapOffset[2]：模型的旋轉角度（rotation offset，通常是 z 軸旋轉）
mapOffset[3]：模型的縮放倍率（scale）
```

如果沒儲存，刷新頁面就會不見

```ts
function saveModelEdits() {
    if (modelLonOffset.value !== null && modelLatOffset.value !== null) {
      const correctedMap = {
        lon_offset: mapOffset.value[0],
        lat_offset: mapOffset.value[1],
        rotation_offset: mapOffset.value[2],
        scale: mapOffset.value[3]
      };
      $apiClient.project.mapCorrectionUpdate(Number(projectId.value), correctedMap)
        .then(() => {
          alert('模型變更已儲存並同步到後端！');
          modelEditEnabled.value = false;
        })
        .catch(() => {
          alert('儲存失敗:(((');
        });
    } else {
      alert('儲存失敗:(((');
    }
  }
```

然後就把我們記下的新 offset  存到後端
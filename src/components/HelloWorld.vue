/* eslint-disable no-prototype-builtins */
/* eslint-disable no-prototype-builtins */
<template>
  <div class="main">
    <div ref="vtkContainer" class="vtkContainer"></div>
    <div
      class="controls"
    >
      <div class="close-btn">
        <el-button
          @click="handleRecover"
        >
        recover
        </el-button>
      </div>
    </div>
  </div>
</template>

<script>
import "@kitware/vtk.js/Rendering/Profiles/Geometry";
import vtkFullScreenRenderWindow from "@kitware/vtk.js/Rendering/Misc/FullScreenRenderWindow";
import vtkActor from "@kitware/vtk.js/Rendering/Core/Actor";
import vtkMapper from "@kitware/vtk.js/Rendering/Core/Mapper";
import vtkAxesActor from "@kitware/vtk.js/Rendering/Core/AxesActor";
import vtkCubeAxesActor from "@kitware/vtk.js/Rendering/Core/CubeAxesActor";
import vtkOrientationMarkerWidget from "@kitware/vtk.js/Interaction/Widgets/OrientationMarkerWidget";
import vtkInteractorStyleTrackballCamera from "@kitware/vtk.js/Interaction/Style/InteractorStyleTrackballCamera";


import "@kitware/vtk.js/IO/Core/DataAccessHelper/HttpDataAccessHelper";
import {ModelData} from './data'
import vtkPoints from '@kitware/vtk.js/Common/Core/Points';
import vtkPolyData from '@kitware/vtk.js/Common/DataModel/PolyData';
import vtkCellArray from '@kitware/vtk.js/Common/Core/CellArray';
import {VtkDataTypes} from "@kitware/vtk.js/Common/Core/DataArray/Constants";
// import vtkSurfaceLICMapper from '@kitware/vtk.js/Rendering/Core/SurfaceLICMapper';
// eslint-disable-next-line no-unused-vars
// import vtkPointPicker from '@kitware/vtk.js/Rendering/Core/PointPicker';
// import vtkCellPicker from '@kitware/vtk.js/Rendering/Core/CellPicker';
import vtkPicker from '@kitware/vtk.js/Rendering/Core/Picker';

export default {
  data() {
    return {
      flag: true,
      actor: null,
      fullScreenRenderer: null,
      mapper: null,
      renderer: null,
      camera: null,
      isParallel: true,
      initPosition: null,
      actorPosition: null,
      vtpReader: null,
      vtpWriter: null,
      renderWindow: null,
      resetCamera: null,
      render: null,
      vtkloading: false,
      interactorStyle: null,
      actorColor: null, 
      model:new ModelData()
    };
  },
  mounted() {
    this.init();
    console.log(this.model);
    this.loadData();
  },
  methods: {
    handleColsePanel() {
      this.flag = false;
    },
    init() {
      // const picker =vtkPointPicker.newInstance()
      // const picker =vtkCellPicker.newInstance()
      const picker =vtkPicker.newInstance()
      this.picker=picker
      this.interactorStyle = vtkInteractorStyleTrackballCamera.newInstance({});

      const fullScreenRenderer = vtkFullScreenRenderWindow.newInstance({
        background: [0, 0, 0],
        rootContainer: this.$refs.vtkContainer,
      });

      this.renderer = fullScreenRenderer.getRenderer();
      this.camera = this.renderer.getActiveCamera()
      this.camera.setParallelProjection(true);
      this.initPosition = this.camera.getPosition();
      this.renderWindow = fullScreenRenderer.getRenderWindow();
      fullScreenRenderer.getInteractor().setInteractorStyle(this.interactorStyle);
      this.renderWindow.getInteractor().onRightButtonPress((cellData)=>{
        if(this.renderer !== cellData.pokedRenderer){
          return;
        }
        const pos=cellData.position;
        const point=[pos.x,pos.y,0]
        this.picker.pick(point,fullScreenRenderer.getRenderer())
        console.log(this.picker.getActors());
        if(this.picker.getActors().length > 0){
          console.log(this.picker.getActors());
          // firstPickedActor，i think it is the actor that I selected by right-clicking with my mouse. 
          const firstPickedActor=this.picker.getActors()[0]
          // i hid its edge for the sake of better observation。 But， it is obvious to see that it hides the wrong （rotate it a bit and you can see the problem.）
          // 
          firstPickedActor.getProperty().setEdgeVisibility(false)
          this.renderWindow.render();
        }
      })
      const cubeAxes = vtkCubeAxesActor.newInstance();
      cubeAxes.setCamera(this.renderer.getActiveCamera());
      // eslint-disable-next-line no-unused-vars
      const axes = vtkAxesActor.newInstance({ pickable: true });
      const orientationWidget = vtkOrientationMarkerWidget.newInstance({
        actor: axes,
        // actor: cubeAxes,
        interactor: this.renderWindow.getInteractor(),
      });
      orientationWidget.setEnabled(true);
      orientationWidget.setViewportCorner(
        vtkOrientationMarkerWidget.Corners.BOTTOM_LEFT
      );
      orientationWidget.setViewportSize(0.2); 
      orientationWidget.setMinPixelSize(100);
      orientationWidget.setMaxPixelSize(300);
      this.renderer.resetCamera();
      this.renderer.resetCameraClippingRange();
      this.renderWindow.render();
    },
    loadData() {
      // 
      let zoneFace= this.model.data

      for (let i = 0; i < zoneFace.length; i++) {
        let zone=zoneFace[i];
        let map=zone.faceMap
        let positions=zone.points
        this.insertActor(positions,map)
      }
      this.$nextTick(()=>{
        this.renderer.resetCamera()
        this.renderWindow.render()
      })
    },
    insertActor(facePoint,map){
      let p =vtkPoints.newInstance()
      for (let i = 0; i < facePoint.length; i+=3) {
        p.insertNextPoint(facePoint[i],facePoint[i+1],facePoint[i+2])
        
      }
      let cell = vtkCellArray.newInstance({
        dataType:VtkDataTypes.UNSIGNED_INT
      })
      for (const key in map) {
        let topologySize=Number(key)
        let cellList=map[key]
        let vertsOfNumber =Number(key)
        let outCellList=cellList
        if(topologySize > 3){
          outCellList=this.processCell(cellList)
          vertsOfNumber=key/2
          //  vertsOfNumber=key

        }
        for (let i = 0; i < outCellList.length; i+=vertsOfNumber) {
          let idList=[]
          for (let j = 0; j < vertsOfNumber; j++) {
            idList.push(outCellList[i+j])
          }
          cell.insertNextCell(idList)
          
        }
      }
      let pl=vtkPolyData.newInstance()
      pl.setPoints(p)
      pl.setPolys(cell)
      let polyDataMapper=vtkMapper.newInstance()
      polyDataMapper.setInputData(pl)

      let actor=vtkActor.newInstance()
      actor.setMapper(polyDataMapper)
      actor.setPickable(true)
      actor.getProperty().set({edgeVisibility:1})
      this.renderer.addActor(actor)

    },
    processCell(cellList){
      let cellOfNumber = cellList.length / 2
      let verts =[]
      for (let i = 0,j=0;i < cellOfNumber; i++) {
        verts[i]=cellList[j];
        j=j+2
        
      }
      return verts
    },
    handleRecover(){
      this.renderer.getActors().forEach(actor=>{
        console.log(actor)
        actor.getProperty().setEdgeVisibility(true)
      })
      this.renderWindow.render();
    }
  },
};
</script>

<style scoped>
.controls {
  position: absolute;
  top: 20px;
  left: 20px;
  width: 100px;
  background-color: rgba(255, 255, 255, 1.2);
  padding: 12px;
}
</style>

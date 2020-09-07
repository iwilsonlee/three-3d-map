<template>
  <div id="space">
    <div id="cont"></div>
  </div>
</template>

<script>
import * as THREE from 'three'
import { MapControls } from 'three/examples/jsm/controls/OrbitControls'
import { BufferGeometryUtils } from 'three/examples/jsm/utils/BufferGeometryUtils'
// import { GeometryUtils } from 'three/examples/jsm/utils/GeometryUtils'
import Stats from 'three/examples/jsm/libs/stats.module'
import { Line2 } from 'three/examples/jsm/lines/Line2'
import { LineGeometry } from 'three/examples/jsm/lines/LineGeometry'
import { LineMaterial } from 'three/examples/jsm/lines/LineMaterial'
import * as GEOLIB from 'geolib'

// const api = 'https://gistcdn.githack.com/isjeffcom/a611e99aa888534f67cc2f6273a8d594/raw/9dbb086197c344c860217826c59d8a70d33dcb54/gistfile1.txt'
// const api = '/static/export.txt'
const api = '/static/daxie0907.geojson'

export default {
  name: 'space',
  data () {
    return {
      // Three space
      scene: null,
      camera: null,
      renderer: null,
      controls: null,
      iR: null,
      iR_Road: null,
      iR_Line: null,
      MAT_BUILDING: null,
      MAT_ROAD: null,
      FLAG_ROAD_ANI: true,
      Animated_Line_Speed: 0.004,
      Animated_Line_Distances: [],
      geos_building: [],
      collider_building: [],
      raycaster: null,
      stats: null,
      center: [121.9265, 29.9308]
      // center: [113.3191, 23.1236]
    }
  },

  mounted () {
    this.Awake()

    let that = this

    // when user resize window
    window.addEventListener('resize', onWindowResize, false)

    function onWindowResize () {
      if (that.scene) {
        that.camera.aspect = window.innerWidth / window.innerHeight
        that.camera.updateProjectionMatrix()
        that.renderer.setSize(window.innerWidth, window.innerHeight)
      }
    }

    onWindowResize()
  },

  methods: {
    Awake () {
      // console.log('scene')
      let cont = document.getElementById('cont')

      // Init scene
      this.scene = new THREE.Scene()

      this.scene.background = new THREE.Color(0x222222)

      // Init Camera
      this.camera = new THREE.PerspectiveCamera(25, window.clientWidth / window.clientHeight, 1, 100)
      this.camera.position.set(8, 4, 0)

      // Init group
      this.iR = new THREE.Group()
      this.iR.name = 'Interactive Root'
      this.iR_Road = new THREE.Group()
      this.iR_Road.name = 'Roads'
      this.iR_Line = new THREE.Group()
      this.iR_Line.name = 'Animated Line on Roads'
      this.scene.add(this.iR)
      this.scene.add(this.iR_Road)
      // this.scene.add(this.iR_Line)

      // Init Raycaster
      this.raycaster = new THREE.Raycaster()

      // Init Light
      let light0 = new THREE.AmbientLight(0xfafafa, 0.25)

      let light1 = new THREE.PointLight(0xfafafa, 0.4)
      light1.position.set(200, 90, 40)

      let light2 = new THREE.PointLight(0xfafafa, 0.4)
      light2.position.set(200, 90, -40)

      this.scene.add(light0)
      this.scene.add(light1)
      this.scene.add(light2)

      let gridHelper = new THREE.GridHelper(60, 160, new THREE.Color(0x555555), new THREE.Color(0x333333))
      this.scene.add(gridHelper)

      // let geometry = new THREE.BoxGeometry(1,1,1)
      // let material = new THREE.MeshBasicMaterial({color: 0x00ff00})
      // let mesh = new THREE.Mesh(geometry, material)
      // this.scene.add(mesh)

      // Init renderer
      this.renderer = new THREE.WebGLRenderer({
        antialias: true
      })
      this.renderer.setPixelRatio(window.devicePixelRatio)
      this.renderer.setSize(window.innerWidth, window.innerHeight)

      cont.appendChild(this.renderer.domElement)

      this.controls = new MapControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping = true
      this.controls.dampingFactor = 0.25
      this.controls.screenSpacePanning = false
      this.controls.maxDistance = 800

      this.controls.update()

      this.stats = new Stats()
      cont.appendChild(this.stats.domElement)

      this.MAT_BUILDING = new THREE.MeshPhongMaterial()

      this.Update()

      this.GetGeoJson()
    },
    Update () {
      requestAnimationFrame(this.Update)

      this.renderer.render(this.scene, this.camera)
      this.controls.update()
      this.stats.update()

      if (this.FLAG_ROAD_ANI) {
        this.UpdateAniLines()
      }
    },
    GetGeoJson () {
      fetch(api).then((res) => {
        res.json().then((data) => {
          this.LoadBuildings(data)
        })
      })
    },

    LoadBuildings (data) {
      let features = data.features

      this.MAT_BUILDING = new THREE.MeshPhongMaterial()
      this.MAT_ROAD = new THREE.LineBasicMaterial({ color: 0xf3dd39, linewidth: 15 })
      // this.MAT_ROAD = new THREE.LineBasicMaterial({ color: 0x1B4686 })
      // this.MAT_ROAD = new THREE.LineDashedMaterial({ color: 0x1B4686 })

      for (let i = 0; i < features.length; i++) {
        let fel = features[i]
        if (!fel['properties']) return

        let info = fel.properties

        if (info['building']) {
          this.addBuilding(fel.geometry.coordinates, info, info['building:levels'])
        } else if (info['highway']) {
          if (fel.geometry.type === 'LineString' && info['highway'] !== 'pedestrian' && info['highway'] !== 'footway' && info['highway'] !== 'path') {
            this.addRoad(fel.geometry.coordinates, info)
          }
        }
      }

      let that = this
      this.$nextTick(() => {
        let mergeGeometry = BufferGeometryUtils.mergeBufferGeometries(that.geos_building)
        let mesh = new THREE.Mesh(mergeGeometry, that.MAT_BUILDING)
        that.iR.add(mesh)
      })
    },

    addBuilding (data, info, height = 1) {
      height = height || 1

      let shape, geometry
      let holes = []
      for (let i = 0; i < data.length; i++) {
        let el = data[i]

        if (i === 0) {
          shape = this.genShape(el, this.center)
        } else {
          holes.push(this.genShape(el, this.center))
        }
      }

      for (let i = 0; i < holes.length; i++) {
        shape.holes.push(holes[i])
      }

      geometry = this.genGeometry(shape, {
        curveSegments: 1,
        depth: 0.05 * height,
        bevelEnabled: false
      })

      geometry.rotateX(Math.PI / 2)
      geometry.rotateZ(Math.PI)

      this.geos_building.push(geometry)

      let helper = this.genHelper(geometry)
      if (helper) {
        helper.name = info['name'] ? info['name'] : 'Building'
        helper.info = info
        // this.iR.add(helper)
        this.collider_building.push(helper)
      }
    },
    addRoad (d, info) {
      // Init points array
      let points = []

      // let positions = []
      // let colors = []

      // Loop for all nodes
      for (let i = 0; i < d.length; i++) {
        if (!d[0][1]) return

        let el = d[i]

        // Just in case
        if (!el[0] || !el[1]) return

        let elp = [el[0], el[1]]

        // convert position from the center position
        elp = this.GPSRelativePosition([elp[0], elp[1]], this.center)

        // Draw Line
        // points.push(new THREE.Vector3(elp[0], 0.5, elp[1]))
        points.push(elp[0], 0.5, elp[1])
      }

      // let geometry = new THREE.BufferGeometry().setFromPoints(points)
      // geometry.setFromPoints(points)

      // Adjust geometry rotation
      // geometry.rotateZ(Math.PI)

      // let line = new THREE.Line(geometry, this.MAT_ROAD)

      let lineGeometry = new LineGeometry()
      lineGeometry.setPositions(points)
      lineGeometry.rotateZ(Math.PI)

      // 自定义的材质
      var material = new LineMaterial({
        color: 0xf5c806,
        // 线宽度
        linewidth: 3,
        // vertexColors: true,
        dashed: false
      })
      // 把渲染窗口尺寸分辨率传值给材质LineMaterial的resolution属性
      // resolution属性值会在着色器代码中参与计算
      material.resolution.set(window.innerWidth, window.innerHeight)
      var line2 = new Line2(lineGeometry, material)

      line2.info = info
      line2.computeLineDistances()

      this.iR_Road.add(line2)
      line2.position.set(line2.position.x, 0.5, line2.position.z)

      // if (this.FLAG_ROAD_ANI) {
      //   // Length of the line
      //   let lineLength = lineGeometry.attributes.lineDistance.array[lineGeometry.attributes.lineDistance.count - 1]

      //   if (lineLength > 0.8) {
      //     let aniLine = this.addAnimatedLine(lineGeometry, lineLength)
      //     this.iR_Line.add(aniLine)
      //   }
      // }
    },
    addAnimatedLine (geometry, length) {
      let animatedLine = new THREE.Line(geometry, new THREE.LineDashedMaterial({ color: 0x00FFFF }))
      animatedLine.material.transparent = true
      animatedLine.position.y = 0.5
      animatedLine.material.dashSize = 0
      animatedLine.material.gapSize = 1000

      this.Animated_Line_Distances.push(length)

      return animatedLine
    },
    UpdateAniLines () {
      // If no animated line than do nothing
      if (this.iR_Line.children.length <= 0) return

      for (let i = 0; i < this.iR_Line.children.length; i++) {
        let line = this.iR_Line.children[i]

        let dash = parseInt(line.material.dashSize)
        let length = parseInt(this.Animated_Line_Distances[i])

        if (dash > length) {
          // console.log("b")
          line.material.dashSize = 0
          line.material.opacity = 1
        } else {
          // console.log("a")
          line.material.dashSize += this.Animated_Line_Speed
          line.material.opacity = line.material.opacity > 0 ? line.material.opacity - 0.002 : 0
        }
      }
    },

    genShape (points, center) {
      let shape = new THREE.Shape()

      for (let i = 0; i < points.length; i++) {
        let elp = points[i]
        elp = this.GPSRelativePosition(elp, center)

        if (i === 0) {
          shape.moveTo(elp[0], elp[1])
        } else {
          shape.lineTo(elp[0], elp[1])
        }
      }

      return shape
    },

    genGeometry (shape, settings) {
      let geometry = new THREE.ExtrudeBufferGeometry(shape, settings)
      geometry.computeBoundingBox()

      return geometry
    },

    genHelper (geometry) {
      if (!geometry.boundingBox) {
        geometry.computeBoundingBox()
      }

      let box3 = geometry.boundingBox
      if (!isFinite(box3.max.x)) {
        return false
      }

      let helper = new THREE.Box3Helper(box3, 0xffff00)
      helper.updateMatrixWorld()
      return helper
    },

    FireRaycaster (pointer) {
      this.raycaster.setFromCamera(pointer, this.camera)

      let intersects = this.raycaster.intersectObjects(this.collider_building, true)
      // console.log(intersects)
      if (intersects.length > 0) {
        let selectedObject = intersects[0].object
        return selectedObject
      } else {
        return false
      }
    },

    GPSRelativePosition (objPosi, centerPosi) {
      // Get GPS distance
      let dis = GEOLIB.getDistance(objPosi, centerPosi)

      // Get bearing angle
      let bearing = GEOLIB.getRhumbLineBearing(objPosi, centerPosi)

      // Calculate X by centerPosi.x + distance * cos(rad)
      let x = centerPosi[0] + (dis * Math.cos(bearing * Math.PI / 180))

      // Calculate Y by centerPosi.y + distance * sin(rad)
      let y = centerPosi[1] + (dis * Math.sin(bearing * Math.PI / 180))

      // Reverse X (it work)
      return [-x / 100, y / 100]
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1, h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
